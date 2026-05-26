# dma-buf read/write fio hang RCA and proposed fix

## Scope

This note analyzes the hung-task trace captured in
`dmabuf_v4_fio_hang_codex_context.md` for branch `pavel/rw-dmabuf-v4`.

No kernel source changes are made here. The patch below is a proposed fix.

## Short RCA

The likely root cause is that the new io_uring dma-buf token invalidation
fence is inserted into the exported dma-buf reservation object as
`DMA_RESV_USAGE_KERNEL`.

That usage class is too strong for this fence. The token fence does not
represent a GPU or memory-manager operation that must block all kernel users of
the BO. It represents io_uring/NVMe importer revocation progress: old block I/O
requests are still holding references to an old DMA mapping, and the importer
will signal the fence when those request references drain.

Because the fence is added as `DMA_RESV_USAGE_KERNEL`, AMDGPU KFD restore sees
it as a memory-management fence and synchronizes with it while restoring BOs.
That makes the KFD restore path wait on fio request lifetime, and the fio
io_uring worker can simultaneously be blocked trying to create a replacement
map by taking the same dma-buf/BO reservation lock.

The result is a lock/fence dependency cycle between:

- io_uring fixed read worker:
  `io_read_fixed() -> io_import_reg_buf() -> io_dmabuf_create_map()`
- NVMe dma-buf importer:
  `nvme_dmabuf_token_map() -> dma_buf_map_attachment()`
- AMDGPU/KFD restore:
  `amdgpu_amdkfd_gpuvm_restore_process_bos() -> drm_exec_prepare_obj()`
  and later `dma_resv_for_each_fence(..., DMA_RESV_USAGE_KERNEL, ...)`
- AMDGPU BO move notification:
  `amdgpu_bo_move_notify() -> dma_buf_invalidate_mappings() ->
  nvme_dmabuf_invalidate_mappings() -> io_dmabuf_token_invalidate_mappings()`

## Evidence in this tree

`lib/io_dmabuf_token.c`:

- `io_dmabuf_create_map()` waits for `DMA_RESV_USAGE_KERNEL` fences before
  mapping, then takes the dma-buf reservation lock and calls the device map op
  (`lib/io_dmabuf_token.c:119-160`).
- `io_dmabuf_drop_map()` adds the token map-drain fence with
  `DMA_RESV_USAGE_KERNEL` (`lib/io_dmabuf_token.c:180-197`).
- `io_dmabuf_token_release_work()` waits for `DMA_RESV_USAGE_KERNEL` fences
  before final token teardown (`lib/io_dmabuf_token.c:206-228`).

`drivers/nvme/host/pci.c`:

- NVMe registers as a dynamic dma-buf importer and forwards invalidation into
  `io_dmabuf_token_invalidate_mappings()` (`drivers/nvme/host/pci.c:2391-2401`).
- `nvme_dmabuf_token_map()` asserts the dma-buf reservation lock is held and
  calls `dma_buf_map_attachment()` (`drivers/nvme/host/pci.c:2403-2464`).
- `nvme_dmabuf_token_unmap()` also runs under that reservation lock and calls
  `dma_buf_unmap_attachment()` (`drivers/nvme/host/pci.c:2466-2477`).

`drivers/dma-buf/dma-buf.c`:

- `dma_buf_map_attachment()` requires the dma-buf reservation lock to be held
  (`drivers/dma-buf/dma-buf.c:1165-1177`).
- Dynamic invalidation requires importers to destroy/recreate cached mappings,
  and after invalidation the caller waits on reservation fences to stop full
  access (`drivers/dma-buf/dma-buf.c:1339-1386`).

`drivers/gpu/drm/amd/amdgpu/amdgpu_object.c`:

- BO movement invalidates exported dma-buf mappings
  (`drivers/gpu/drm/amd/amdgpu/amdgpu_object.c:1275-1277`).

`drivers/gpu/drm/amd/amdgpu/amdgpu_amdkfd_gpuvm.c`:

- KFD restore locks BOs with `drm_exec_prepare_obj()`
  (`drivers/gpu/drm/amd/amdgpu/amdgpu_amdkfd_gpuvm.c:2948-2957`).
- It then imports every `DMA_RESV_USAGE_KERNEL` fence into its restore sync
  object (`drivers/gpu/drm/amd/amdgpu/amdgpu_amdkfd_gpuvm.c:2985-2992`).

`include/linux/dma-resv.h`:

- `DMA_RESV_USAGE_KERNEL` is documented for in-kernel memory management and is
  always waited by drivers before accessing the resource
  (`include/linux/dma-resv.h:71-84`).
- `DMA_RESV_USAGE_BOOKKEEP` is documented for fences that should not
  participate in implicit synchronization, including preemption fences, page
  table updates, TLB flushes, and explicitly synced submissions
  (`include/linux/dma-resv.h:102-116`).

That matches the observed trace:

- `iou-wrk-*` blocks at `ww_mutex_lock()` inside `io_dmabuf_create_map()`.
- `kfd_restore_wq` blocks at `ww_mutex_lock()` inside
  `drm_exec_prepare_obj()`.

## Why `DMA_RESV_USAGE_KERNEL` is wrong here

The io_dmabuf token fence is a revocation/bookkeeping fence for one importer.
It is not a memory move, clear, copy, page-table update, or other operation that
all kernel users must wait on before touching the BO.

Using `DMA_RESV_USAGE_KERNEL` makes unrelated kernel paths, including KFD
restore, depend on outstanding fio/NVMe I/O references. That expands the
dependency graph from "NVMe importer waits for its old map to drain before
remapping" into "AMDGPU memory-management restore waits for fio I/O completion",
which is exactly the wrong direction during eviction/restore.

The token layer still needs to wait for its own old-map fences before creating
a new map or destroying a token. That wait can be done by querying
`DMA_RESV_USAGE_BOOKKEEP`, because `BOOKKEEP` includes all lower usages as
well. The important property is that other kernel users querying
`DMA_RESV_USAGE_KERNEL` will no longer pick up the io_dmabuf map-drain fence.

## Proposed minimal fix

Use `DMA_RESV_USAGE_BOOKKEEP` for io_dmabuf token map-drain fences, and make the
io_dmabuf token remap/release waits use the same usage.

Conceptual patch:

```diff
diff --git a/lib/io_dmabuf_token.c b/lib/io_dmabuf_token.c
index 808b5ad33dbc..xxxxxxxxxxxx 100644
--- a/lib/io_dmabuf_token.c
+++ b/lib/io_dmabuf_token.c
@@
 #include <linux/io_dmabuf_token.h>
 #include <linux/dma-resv.h>
 
+#define IO_DMABUF_FENCE_USAGE DMA_RESV_USAGE_BOOKKEEP
+
 struct io_dmabuf_fence {
        struct dma_fence base;
        spinlock_t lock;
@@
-       ret = dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+       ret = dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
                                    true, MAX_SCHEDULE_TIMEOUT);
@@
-       if (dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+       if (dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
                                  true, 0) < 0) {
@@
        dma_resv_add_fence(dmabuf->resv, &map->fence->base,
-                          DMA_RESV_USAGE_KERNEL);
+                          IO_DMABUF_FENCE_USAGE);
@@
-       ret = dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+       ret = dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
                                    false, MAX_SCHEDULE_TIMEOUT);
```

Expected effect:

- io_dmabuf still serializes its own remap/release against old in-flight I/O.
- `dma_buf_invalidate_mappings()` callers can still wait for the importer fence
  if they wait broadly enough to include bookkeeping fences.
- KFD restore's `DMA_RESV_USAGE_KERNEL` walk no longer waits for fio/NVMe
  importer map-drain fences.
- The observed KFD restore / io_uring worker dependency cycle is removed without
  changing the v4 token object model.

## Additional hardening to consider

The usage-class fix is the minimal change. Two follow-ups would make this path
easier to reason about:

1. Name the usage explicitly in the token layer, for example
   `IO_DMABUF_FENCE_USAGE`, rather than spelling a generic dma-resv usage at
   each call site.
2. Add a short comment near `dma_resv_add_fence()` explaining that the fence is
   importer revocation bookkeeping, not a memory-management fence, and must not
   be waited by unrelated `DMA_RESV_USAGE_KERNEL` users.

## Instrumentation to prove the RCA

Minimal temporary instrumentation:

- Log every `io_dmabuf_drop_map()` fence add with:
  token pointer, dmabuf pointer, fence context/seqno, usage class, current task.
- Log every `io_dmabuf_create_map()` wait start/end and map callback entry/exit.
- In KFD restore, log fences returned by
  `dma_resv_for_each_fence(..., DMA_RESV_USAGE_KERNEL, ...)`, including driver
  name and timeline name.

If the RCA is correct, the pre-fix KFD restore logs will show fences whose
driver/timeline name is `DMABUF token`. After changing the token fences to
`DMA_RESV_USAGE_BOOKKEEP`, those fences should disappear from the KFD restore
`DMA_RESV_USAGE_KERNEL` scan while io_dmabuf remap/release still waits for them.

## Secondary observations

`io_dmabuf_map_release_work()` signals the token fence before taking the
reservation lock for unmap. That is intentional and avoids waiting on the fence
while holding the lock, but it means the reservation fence only proves that old
request references drained, not that `dma_buf_unmap_attachment()` has completed.
This matches the dma-buf invalidation contract for dynamic importers where full
access stops after the fence, while speculative read-and-discard access may
continue until unmap.

I did not find evidence from the inspected code of a missing
`dma_resv_unlock()`, a missing fence signal, or a simple reference leak causing
this hang. The suspicious behavior is the fence usage class and the resulting
over-broad synchronization with KFD restore.
