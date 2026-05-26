# dma-buf v4 fio hang: RCA and fix comparison

## Overview

This document records the root-cause analysis of the hung-task trace from
`dmabuf_v4_fio_hang_codex_context.md` (branch `pavel/rw-dmabuf-v4`), compares
two independent analyses, and arrives at a single agreed fix.

---

## The hung-task traces

Two tasks are shown blocked in `ww_mutex_lock` for more than 122 seconds.

**iou-wrk-22173**
```
io_read_fixed → io_import_reg_buf → io_import_dmabuf
  → io_dmabuf_create_map()                lib/io_dmabuf_token.c:138
      dma_resv_lock(dmabuf->resv, NULL)   ← ww_mutex_lock, blocked
```

**kfd_restore_wq / kworker/u80:2**
```
restore_process_worker → amdgpu_amdkfd_gpuvm_restore_process_bos
  → drm_exec_prepare_obj → drm_exec_lock_obj
      ww_mutex_lock(bo->resv, exec->ww_ctx)  ← blocked
```

Both tasks are trying to **acquire** a `ww_mutex` (a BO reservation lock), not
waiting for a dma_fence. Something holds `dmabuf->resv` for longer than 122
seconds.

---

## Analysis 1 — the third task (primary deadlock mechanism)

### Who holds `dmabuf->resv`

The hung-task excerpt shows two waiters but not the lock holder. The most likely
explanation (consistent with the code paths and the ww_mutex_lock stack in both
traces) is a hidden third task — another io_uring worker (call it iou-wrk-A) —
that has taken `dma_resv_lock(dmabuf->resv, NULL)` at line 138 of
`io_dmabuf_create_map()` and is blocked inside `nvme_dmabuf_token_map()`. This
is inferred; sysrq lock-owner output, lockdep annotations, or a full task dump
would be needed to confirm the holder identity.

```
io_dmabuf_create_map()                     lib/io_dmabuf_token.c:138
  dma_resv_lock(dmabuf->resv, NULL)        ← holds resv (NULL ww_acquire_ctx)
  nvme_dmabuf_token_map()                  drivers/nvme/host/pci.c:2428
    dma_buf_map_attachment(attach, dir)
      amdgpu_dma_buf_map()
        ttm_bo_validate(&bo->tbo, &bo->placement,
                        &ctx)              ctx = { false, false } (no_wait_gpu=false)
          dma_resv_wait_timeout(bo->resv,
              DMA_RESV_USAGE_BOOKKEEP, ...)  ← reads fences under RCU (no re-lock)
            dma_fence_default_wait(F)         ← SLEEPING here
```

`ttm_bo_validate()` calls `ttm_bo_wait_ctx()` (ttm_bo.c:1083) which uses
`DMA_RESV_USAGE_BOOKKEEP` — the broadest usage level, covering all fences
regardless of their registered usage (GPU execution, read, write, management,
etc.). It reads the BO's reservation object fences under RCU without
re-acquiring the mutex and blocks on a fence F that has not yet signalled. This
is normal AMDGPU/TTM behaviour for an exporter that expects a conforming
importer to have drained all relevant reservation fences before entering the
locked section.

### Why kfd_restore_wq cannot break the hold

`amdgpu_amdkfd_gpuvm_restore_process_bos()` uses `drm_exec` with a
`ww_acquire_ctx` to lock every BO in the evicted KFD process. The stack shows it
blocked in `drm_exec_lock_obj` → `ww_mutex_lock`, waiting for some BO's
reservation lock. The most likely (inferred) target is `dmabuf->resv`, held by
iou-wrk-A with a **NULL** `ww_acquire_ctx`. The WW wound-wait protocol only
operates between lockers that both carry a `ww_acquire_ctx`. A drm_exec context
**cannot wound** a NULL-ctx holder, so kfd_restore_wq sleeps indefinitely.

KFD restore or the associated GPU recovery progress is what fence F depends on
to resolve. Until that path completes, iou-wrk-A cannot be unblocked. But
kfd_restore_wq cannot complete because it cannot acquire the resv lock it is
waiting on.

### Why `dma_fence_default_wait` appears in the iou-wrk-22173 trace

```
__ww_mutex_lock.constprop.0+0x7f2/0xe90
? dma_fence_default_wait+0x1c7/0x280      ← '?' = uncertain frame
__ww_mutex_lock_slowpath+0x16/0x30
ww_mutex_lock+0xeb/0x100
io_dmabuf_create_map+0x4b/0x1a0
```

The `?` frame is a return address left on the stack by the completed pre-lock
`dma_resv_wait_timeout` call at line 131. iou-wrk-22173 finished that wait and
is now blocked at `dma_resv_lock()` at line 138. It is not iou-wrk-A.

### The inferred deadlock cycle

```
iou-wrk-A (inferred holder, not in trace)
                    holds dmabuf->resv (NULL ctx)
                    sleeping in ttm_bo_wait_ctx (DMA_RESV_USAGE_BOOKKEEP)
                      → dma_fence_default_wait(F)

kfd_restore_wq      blocked in drm_exec_lock_obj waiting for a BO resv lock
                    (most likely dmabuf->resv, inferred from the path)
                    GPU/KFD recovery progress is needed to signal fence F,
                    but that progress cannot complete while the resv is held

iou-wrk-22173       blocked in dma_resv_lock waiting for dmabuf->resv
                    (secondary victim, not part of the cycle itself)
```

### Why the pre-lock fence wait at line 131 did not prevent this

```c
/* lib/io_dmabuf_token.c:131 */
ret = dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
                            true, MAX_SCHEDULE_TIMEOUT);
```

`DMA_RESV_USAGE_KERNEL` only waits for fences at that usage level. In the
`dma_resv_usage` enum:

```
DMA_RESV_USAGE_KERNEL   = 0   (highest priority)
DMA_RESV_USAGE_WRITE    = 1
DMA_RESV_USAGE_READ     = 2
DMA_RESV_USAGE_BOOKKEEP = 3   (lowest priority, broadest wait)
```

`dma_resv_wait_timeout(resv, usage)` waits for fences whose usage value is ≤
the given usage. Waiting for KERNEL (0) only waits for usage-0 fences. It does
**not** wait for any fences at WRITE, READ, or BOOKKEEP levels. But
`ttm_bo_wait_ctx()` (ttm_bo.c:1083) uses `DMA_RESV_USAGE_BOOKKEEP` — the
broadest possible wait — covering every fence on the BO's resv regardless of
usage. So when iou-wrk-A enters `amdgpu_dma_buf_map()` → `ttm_bo_validate()`,
fences at any usage level that have not yet signalled can block it. The pre-lock
wait at line 131 never drained those fences. The block occurs **while
`dmabuf->resv` is held**.

---

## Analysis 2 — wrong fence usage class (semantic bug)

The NVMe token map-drain fence is added to `dmabuf->resv` as
`DMA_RESV_USAGE_KERNEL`:

```c
/* lib/io_dmabuf_token.c:191 */
dma_resv_add_fence(dmabuf->resv, &map->fence->base, DMA_RESV_USAGE_KERNEL);
```

`DMA_RESV_USAGE_KERNEL` is documented for in-kernel memory management operations
(TTM moves, page-table updates, TLB flushes). The NVMe token fence represents
**importer revocation bookkeeping**: old NVMe/io_uring requests are draining
their references to an old DMA mapping. It is not a memory-management operation.

In `amdgpu_amdkfd_gpuvm_restore_process_bos()`, after locking all BOs via
`drm_exec`, the code collects `DMA_RESV_USAGE_KERNEL` fences from each BO:

```c
/* amdgpu_amdkfd_gpuvm.c:2985 */
dma_resv_for_each_fence(&cursor, bo->tbo.base.resv,
                        DMA_RESV_USAGE_KERNEL, fence) {
    ret = amdgpu_sync_fence(&sync_obj, fence, GFP_KERNEL);
```

This loop runs after the `drm_exec_until_all_locked` acquisition loop has
completed, while drm_exec still holds all BO resv locks (they are not released
until `drm_exec_fini()`). If the `dmabuf` BO is in the KFD process's `kfd_bo_list`, KFD
restore picks up the NVMe token fence here and adds it to its sync object.
KFD restore then waits on that sync object as part of its BO validation pass,
creating an unwanted dependency on outstanding fio/NVMe I/O lifetime.

`DMA_RESV_USAGE_BOOKKEEP` is the correct usage for this fence. It is
documented for fences that should not participate in implicit synchronization
and that do not need to be waited by unrelated kernel users.

---

## Why both analyses describe the same root cause

The two problems interact and share the same fix.

**The core design gap** is that `io_dmabuf_create_map()` acquires `dmabuf->resv`
with a NULL `ww_acquire_ctx` and then calls into the exporter's `map_dma_buf()`
without having drained all fences the exporter's implementation may need to wait
on. The exporter (`amdgpu_dma_buf_map()`) is conforming to the dma-buf dynamic
attachment contract — it expects the caller to hold the resv and may call
`ttm_bo_validate()` to move or validate the BO. `ttm_bo_validate()` with
`no_wait_gpu=false` calls `ttm_bo_wait_ctx()`, which waits with
`DMA_RESV_USAGE_BOOKKEEP` (the broadest level, covering KERNEL, WRITE, READ,
and BOOKKEEP fences). Because the pre-lock wait in `io_dmabuf_create_map()`
only drained `DMA_RESV_USAGE_KERNEL` fences, any GPU execution or other
higher-priority reservation fences were not cleared before the resv was
locked, and `ttm_bo_wait_ctx()` blocks with the resv held.

The NVMe token fence being `DMA_RESV_USAGE_KERNEL` compounds the problem: it
inflates the dependency set that AMDGPU's memory-management paths wait on (the
KERNEL fence scan in kfd_restore picks it up), and it is also the only fence
that the existing pre-lock wait drains — so the code was never actually draining
the relevant GPU execution fences even in theory.

**Changing the exporter is the wrong direction.** The exporter (`amdgpu_dma_buf_map`)
is not buggy. Asking it to use `no_wait_gpu=true` to accommodate a new importer's
locking model is unjustifiable: it would change semantics for every importer and
every code path that calls `dma_buf_map_attachment()` on AMDGPU BOs. The importer
must be written to work with conforming exporters as they exist.

---

## The correct fix — importer side only

Use `DMA_RESV_USAGE_BOOKKEEP` everywhere the token layer interacts with the resv.

`DMA_RESV_USAGE_BOOKKEEP` (value 3) is the broadest wait: `dma_resv_wait_timeout`
with BOOKKEEP waits for **all** fences regardless of usage level, including GPU
execution fences at `DMA_RESV_USAGE_WRITE`. Widening the pre-lock wait from
KERNEL to BOOKKEEP therefore also drains GPU execution fences before the resv is
locked. When `amdgpu_dma_buf_map()` → `ttm_bo_validate()` runs inside the lock,
it finds no unsignaled WRITE fences and returns without blocking. The hold on
`dmabuf->resv` becomes short-lived, and the WW deadlock cannot form.

```diff
diff --git a/lib/io_dmabuf_token.c b/lib/io_dmabuf_token.c
--- a/lib/io_dmabuf_token.c
+++ b/lib/io_dmabuf_token.c
@@ -7,6 +7,8 @@
 #include <linux/io_dmabuf_token.h>
 #include <linux/dma-resv.h>

+/* Importer revocation bookkeeping; must not be waited by unrelated kernel users */
+#define IO_DMABUF_FENCE_USAGE DMA_RESV_USAGE_BOOKKEEP
+
 struct io_dmabuf_fence {
 	struct dma_fence base;
 	spinlock_t lock;
@@ -125,14 +127,14 @@ struct io_dmabuf_map *io_dmabuf_create_map(struct io_dmabuf_token *token)
 retry:
 	/*
 	 * ->dmabuf_map() will be calling dma_buf_map_attachment(), for which
-	 * we'll need to wait for fences. Do a bit nicer and try to wait
-	 * without the resv lock first.
+	 * we need to wait for all fences (including GPU execution fences at
+	 * WRITE usage) so that ttm_bo_validate() does not block while we hold
+	 * the resv. Wait without the lock first to avoid a NULL-ctx deadlock
+	 * with drm_exec paths (e.g. KFD restore) that also need this resv.
 	 */
-	ret = dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+	ret = dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
 				    true, MAX_SCHEDULE_TIMEOUT);
 	if (!ret)
 		ret = -EAGAIN;
@@ -142,7 +144,7 @@ struct io_dmabuf_map *io_dmabuf_create_map(struct io_dmabuf_token *token)
 	if (map) {
 		ret = 0;
 		goto out;
 	}
-	if (dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+	if (dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
 				  true, 0) < 0) {
 		dma_resv_unlock(dmabuf->resv);
 		goto retry;
@@ -190,7 +192,7 @@ static void io_dmabuf_drop_map(struct io_dmabuf_token *token)
 	}

 	dma_resv_add_fence(dmabuf->resv, &map->fence->base,
-			   DMA_RESV_USAGE_KERNEL);
+			   IO_DMABUF_FENCE_USAGE);
 	/*
 	 * Delay destruction until all inflight requests using the map are
 	 * gone. It'll also signal the fence then.
@@ -218,7 +220,7 @@ static void io_dmabuf_token_release_work(struct work_struct *work)
 	dma_resv_unlock(dmabuf->resv);

 	/* Wait until all maps are destroyed. */
-	ret = dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_KERNEL,
+	ret = dma_resv_wait_timeout(dmabuf->resv, IO_DMABUF_FENCE_USAGE,
 				    false, MAX_SCHEDULE_TIMEOUT);
```

No changes to any exporter.

---

## How the fix prevents the deadlock

**Step 1 — pre-lock wait now drains GPU execution fences**

`dma_resv_wait_timeout(dmabuf->resv, DMA_RESV_USAGE_BOOKKEEP, ...)` waits for
fences at all usage levels (0 through 3). All existing reservation fences —
including GPU execution fences — are drained before the resv is locked. When
`amdgpu_dma_buf_map()` → `ttm_bo_wait_ctx()` runs inside the lock, it should
not block on those already-drained fences. `dmabuf->resv` is held for the
duration of the map operation, which may still do allocation, DMA mapping, or
other driver work, but should not block waiting on pre-existing reservation
fences.

**Step 2 — KFD restore no longer depends on fio I/O lifetime**

The token map-drain fence is now `DMA_RESV_USAGE_BOOKKEEP`. The KERNEL-fence
scan in `amdgpu_amdkfd_gpuvm_restore_process_bos()` only collects fences at
usage ≤ KERNEL (0). A BOOKKEEP fence (usage 3) is invisible to that scan.
kfd_restore_wq no longer imports the NVMe importer revocation fence into its
sync object and no longer waits on outstanding fio I/O references as part of
BO restore.

**Step 3 — the inner retry check remains correct**

The non-blocking inner check at line 145 (timeout=0) catches any fence that
arrived in the race window between the pre-lock drain completing and the resv
being acquired. With BOOKKEEP it catches GPU execution fences that appeared
during that window, forces a retry, and the pre-lock drain on the next iteration
clears them. This is a pre-existing mechanism that already handled the KERNEL
fence race; it continues to work correctly with BOOKKEEP.

---

## Why Analysis 1 initially suggested exporter changes

Analysis 1 correctly identified the hidden third task and the GPU-fence-wait
blocking mechanism, and traced the problem to `ttm_bo_validate()` with
`no_wait_gpu=false` running while `dmabuf->resv` is held. The initial proposed
fix (using `no_wait_gpu=true` in `amdgpu_dma_buf_map()`) was incorrect because:

1. The exporter is not buggy. `amdgpu_dma_buf_map()` is conforming to the
   dma-buf dynamic attachment contract. Modifying exporter logic to accommodate
   a new importer's locking model is unjustifiable and sets a bad precedent.

2. The importer is responsible for presenting a clean resv state to the exporter.
   The fix belongs entirely in `lib/io_dmabuf_token.c`: widen the pre-lock fence
   drain to include GPU execution fences. This is what the BOOKKEEP change does.

Analysis 2's fix (`DMA_RESV_USAGE_BOOKKEEP`) is the correct minimal fix for
the identified lock/fence cycle. Its primary motivation was the semantic wrongness of KERNEL for an
importer revocation fence, but as a side effect it also widens the pre-lock
drain to cover GPU execution fences, which is exactly what is needed to prevent
`ttm_bo_validate()` from blocking while the resv is held.

---

## Summary

| Question | Answer |
|---|---|
| What holds `dmabuf->resv` for 122 s? | Likely hidden iou-wrk-A blocked in `ttm_bo_wait_ctx()` waiting on an unsignaled reservation fence |
| Why can't kfd_restore_wq wound it? | iou-wrk-A uses NULL `ww_acquire_ctx`; WW cannot wound NULL-ctx holders |
| Why does the pre-lock wait not help? | Line 131 only waits for KERNEL fences; GPU WRITE fences are not drained |
| What is the semantic bug? | Token fence registered as KERNEL; KFD restore's KERNEL scan picks it up |
| Should the exporter be changed? | No. The exporter is conforming. The importer must drain the right fences. |
| What is the fix? | Change `IO_DMABUF_FENCE_USAGE` to `DMA_RESV_USAGE_BOOKKEEP` in `lib/io_dmabuf_token.c` |
| Does the fix change any exporter? | No. Only `lib/io_dmabuf_token.c` is touched. |
