# dma-buf v4 deadlock debugging timeline

## Overview

Pavel's `rw-dmabuf-v4` branch adds dma-buf backed registered buffers to io_uring.
The feature introduces a new `io_dmabuf_token` layer in `lib/io_dmabuf_token.c`
that acts as the dma-buf importer on behalf of NVMe and manages per-request DMA
mappings.

The workload that triggers the hang:

```bash
taskset -c 5 fio --filename=/dev/nvme0n1 --direct=1 --size=4G --rw=randread \
  --numjobs=1 --bs=4K --ioengine=io_uring --sqthread_poll=0 --iodepth=128 \
  --name=uring_dmabuf --gpu_dmabuf=1 --registerfiles=1 \
  --group_reporting --thread
```

GPU dma-buf: AMDGPU exports a BO; NVMe imports it as a dynamic dma-buf
attachment; fio does fixed reads through io_uring using the registered buffer.

---

## Experiment that clarifies the root cause

Replacing `dma_buf_dynamic_attach()` with `dma_buf_attach()` (static attachment)
in the NVMe importer makes the hang disappear.

With static attachment AMDGPU pins the BO in VRAM and never calls
`move_notify` / `dma_buf_invalidate_mappings()`. With dynamic attachment AMDGPU
is free to evict and restore the BO, and the hang only manifests during an
eviction/restore cycle.

**This pins the root cause in the invalidation path, not the initial map path.**

---

## Relevant enum: dma_resv_usage

```c
DMA_RESV_USAGE_KERNEL   = 0   /* memory management, TTM moves, TLB flushes */
DMA_RESV_USAGE_WRITE    = 1   /* GPU write / exclusive access */
DMA_RESV_USAGE_READ     = 2   /* GPU read / shared access */
DMA_RESV_USAGE_BOOKKEEP = 3   /* bookkeeping, no implicit sync */
```

`dma_resv_wait_timeout(resv, usage)` waits for all fences where `fence_usage <=
usage`.  Larger usage = broader wait.

`dma_resv_for_each_fence(&cursor, resv, usage, fence)` iterates fences where
`fence_usage <= usage`.

`amdgpu_sync_resv()` iterates at `DMA_RESV_USAGE_READ` (2) — collects KERNEL,
WRITE, and READ fences. BOOKKEEP (3) fences are invisible to it.

`amdgpu_amdkfd_gpuvm_restore_process_bos()` scans `DMA_RESV_USAGE_KERNEL` (0)
fences after drm_exec locks all BOs — collects only KERNEL fences.

`ttm_bo_wait_ctx()` (called from `ttm_bo_validate()`) uses
`DMA_RESV_USAGE_BOOKKEEP` — waits for all fences on the BO resv.

---

## Deadlock 1 — original v4 patches

**Kernel:** `7.0.0v4+ #42`

**Token fence usage in original code:** `DMA_RESV_USAGE_KERNEL`

### Blocked tasks (dmesg)

```
[ 3564.670482] task:kworker/u80:2  state:D
[ 3564.670538] Workqueue: kfd_restore_wq restore_process_worker [amdgpu]
               __ww_mutex_lock.constprop.0
               ww_mutex_lock
               drm_exec_lock_obj
               drm_exec_prepare_obj
               amdgpu_amdkfd_gpuvm_restore_process_bos

[ 3564.675037] task:iou-wrk-22173  state:D
               __ww_mutex_lock.constprop.0
               ww_mutex_lock
               io_dmabuf_create_map
               io_import_reg_buf
               io_read_fixed
```

### What is happening

```
A. io_uring fixed read calls io_dmabuf_create_map()
   -> dma_resv_lock(dmabuf->resv, NULL)   <-- NULL ww_acquire_ctx

B. While holding resv, calls nvme_dmabuf_token_map()
   -> dma_buf_map_attachment()
   -> amdgpu_dma_buf_map()
   -> ttm_bo_validate()
   -> ttm_bo_wait_ctx(DMA_RESV_USAGE_BOOKKEEP)
   -> dma_fence_default_wait(GPU_FENCE)   <-- BLOCKS here, resv still held

C. GPU fence is waiting for AMDGPU to restore evicted BO memory.
   KFD restore does this via amdgpu_amdkfd_gpuvm_restore_process_bos().

D. KFD restore tries to lock the same BO resv via drm_exec (ww_acquire_ctx).
   drm_exec CANNOT wound the NULL-ctx holder (iou-wrk-A). It sleeps.

E. KFD restore cannot complete; GPU fence never signals; iou-wrk-A never
   unblocks; resv held forever. iou-wrk-22173 and others pile up.
```

### Why the pre-lock wait didn't help

`io_dmabuf_create_map()` did `dma_resv_wait_timeout(resv, DMA_RESV_USAGE_KERNEL)`
before locking. This only drains KERNEL (0) fences. GPU WRITE (1) execution
fences were not drained. `ttm_bo_wait_ctx(BOOKKEEP)` finds those undrained fences
and blocks inside the locked section.

### Additional semantic bug

The token map-drain fence was added as `DMA_RESV_USAGE_KERNEL`.
`amdgpu_amdkfd_gpuvm_restore_process_bos()` scans KERNEL fences after locking
all BOs. It imports the NVMe importer revocation fence into its sync object and
then waits on it. This means KFD restore was waiting for outstanding fio I/O
references to drain before it could restore the BO — the wrong direction.

---

## Fix 1: BOOKKEEP (`beaf686c3bac`)

Changed token fence to `DMA_RESV_USAGE_BOOKKEEP` and widened the pre-lock wait
to `DMA_RESV_USAGE_BOOKKEEP`.

**Effect on semantic bug:** Token fence is now BOOKKEEP (3). KFD's KERNEL (0)
scan no longer picks it up. KFD restore no longer waits on fio I/O drain.

**Effect on pre-lock wait:** Now drains ALL fences including GPU WRITE before
locking resv. `ttm_bo_wait_ctx(BOOKKEEP)` inside `amdgpu_dma_buf_map()` should
find no blocking fences.

---

## Deadlock 2 — after BOOKKEEP fix

**Kernel:** `7.0.0fix-v1+ #46`

**Token fence usage:** `DMA_RESV_USAGE_BOOKKEEP`

### Blocked tasks (dmesg)

```
[246.910040] task:kworker/1:1    state:D
             Workqueue: events io_dmabuf_map_release_work
             __ww_mutex_lock
             ww_mutex_lock
             io_dmabuf_map_release_work   <-- wants resv lock for unmap
             [blocked on mutex owned by kfd_restore_wq worker]

[246.911615] task:kworker/u80:0  state:S
             Workqueue: kfd_restore_wq restore_process_worker [amdgpu]
             dma_fence_default_wait
             dma_fence_wait_timeout
             amdgpu_sync_wait
             amdgpu_vm_cpu_prepare
             amdgpu_vm_update_range
             amdgpu_vm_bo_update
             amdgpu_amdkfd_gpuvm_restore_process_bos
             [holds BO resv locks via drm_exec]

[246.917971] task:kworker/u86:1  state:D
             Workqueue: ttm ttm_bo_delayed_delete [ttm]
             dma_fence_default_wait
             dma_resv_wait_timeout        <-- waits for all fences (BOOKKEEP)

[246.921373] task:iou-wrk-2330   state:D
             __ww_mutex_lock
             io_dmabuf_create_map         <-- wants resv lock
```

### What is happening

The character of the deadlock changed. Now the async `io_dmabuf_map_release_work`
function is the stuck actor, not just the io_uring worker.

`io_dmabuf_map_release_work` was the async function that, after all per-request
refs drained, needed to:
1. Take `dmabuf->resv`
2. Call `dma_buf_unmap_attachment()` (requires resv for dynamic attachments)
3. Signal the token fence

This function cannot get the resv because KFD restore holds it via drm_exec
during its entire BO validation + VM update window.

KFD restore is not stuck getting the resv this time (it has it). It is stuck
inside `amdgpu_sync_wait()` in the VM update phase, waiting for a dma_fence.

### What fence is KFD waiting on?

**`amdgpu_vm_cpu_prepare()` calls `amdgpu_sync_wait(vm->last_update)`** — this
waits for the last VM page-table update GPU command to complete.

That VM update fence depends on `amdgpu_sync_resv()` having collected and waited
on all READ-level (≤2) fences from the BO. The token fence is now BOOKKEEP (3),
so `amdgpu_sync_resv()` (READ scan) does NOT collect it.

However, `amdgpu_sync_test_fence()` returns `true` for any fence with owner
`AMDGPU_FENCE_OWNER_UNDEFINED` before the switch statement, regardless of the
sync mode. The token fence has an UNDEFINED owner. So if it lands in the
READ-or-lower fence bucket (which BOOKKEEP doesn't, but WRITE would), it is
automatically collected.

### TTM delayed delete

`ttm_bo_delayed_delete()` calls `dma_resv_wait_timeout(resv, BOOKKEEP)` —
the broadest possible wait. This DOES pick up the BOOKKEEP token fence and
waits for it to signal. The token fence can only signal after
`io_dmabuf_map_release_work` completes. But `io_dmabuf_map_release_work` can't
get the resv. **This is a clear ABBA deadlock between the async release worker
and TTM/KFD.**

### Honest assessment of Deadlock 2

We are less certain about the exact cycle in Deadlock 2 than in Deadlock 1.
The traces show the symptom clearly but the exact fence KFD is waiting on in
`amdgpu_sync_wait` is not confirmed from the dmesg alone. What is clear:

1. `io_dmabuf_map_release_work` needing the resv lock as a separate async worker
   is fundamentally racy against any path that holds the resv for extended periods.
2. Adding the token fence to the resv (even as BOOKKEEP) creates a dependency
   chain that TTM's delayed delete picks up.
3. The async worker model creates a window where the fence is on the resv but
   its signaler (the worker) cannot proceed.

---

## Fix 2: WRITE fence (`79575a258c93`)

Changed `IO_DMABUF_FENCE_USAGE` from `DMA_RESV_USAGE_BOOKKEEP` to
`DMA_RESV_USAGE_WRITE`.

**Motivation:** BOOKKEEP (3) > READ (2), so `amdgpu_sync_resv()` (READ scan)
does not see the BOOKKEEP token fence. GPU blit commands can start without
waiting for the token fence. If the GPU starts blitting memory that NVMe is
still reading, you get hardware corruption or a stuck MOVE fence.

**What WRITE (1) changes:**
- Token fence is now visible to `amdgpu_sync_resv()` (READ scan picks up ≤2
  fences). GPU blits wait for NVMe I/O to drain before using the memory. This
  is correct for memory safety.
- KFD KERNEL scan (≤0) still does NOT pick up the WRITE (1) token fence.
- Pre-lock wait was also changed to `DMA_RESV_USAGE_WRITE` — drains only
  KERNEL and WRITE fences, not READ.

**Why same trace was still seen:**

With WRITE token fence, `amdgpu_sync_resv()` during the VM update phase picks
it up (1 ≤ 2). KFD restore holds the resv and waits for the WRITE token fence
via `amdgpu_sync_wait`. The token fence signals from `io_dmabuf_map_release_work`
after unmap. But `io_dmabuf_map_release_work` needs the resv lock to perform the
unmap. KFD holds the resv and waits for the fence. The worker has the fence but
waits for the resv.

**ABBA deadlock:**
```
KFD restore:    holds resv → waits for WRITE token fence
release_work:   has token fence → waits for resv
```

This is the same fundamental structure as Deadlock 2. Changing the fence usage
between BOOKKEEP and WRITE does not break the cycle because the cycle is not
about which scan picks up the fence — it is about the async worker competing
with KFD restore for the resv lock.

---

## Fix 3: synchronous invalidation (`000c11d0f04b`)

Completely removed `io_dmabuf_map_release_work` and the token fence. Rewrote
`io_dmabuf_drop_map()` to synchronously drain and unmap inline.

### Core change

```c
static void io_dmabuf_drop_map(struct io_dmabuf_token *token)
{
    dma_resv_assert_held(dmabuf->resv);   /* caller must hold resv */

    map = rcu_dereference_protected(token->map, dma_resv_held(dmabuf->resv));
    if (!map)
        return;
    rcu_assign_pointer(token->map, NULL);

    percpu_ref_kill(&map->refs);
    wait_for_completion(&map->drain);     /* blocks until NVMe I/O drains */

    token->dev_ops->unmap(token, map);    /* resv held — protocol correct */
    percpu_ref_exit(&map->refs);
    kfree(map);
}
```

`io_dmabuf_token_invalidate_mappings()` (called from AMDGPU's `move_notify`)
simply calls `io_dmabuf_drop_map()`. The resv is already held by AMDGPU when
`move_notify` is called, satisfying the `dma_resv_assert_held()`.

### Why this breaks the cycle

- There is no separate async worker competing for the resv.
- There is no token fence on the resv. `amdgpu_sync_resv()` and TTM delayed
  delete have nothing new to wait on.
- AMDGPU calls `move_notify` while holding the resv. We drain NVMe I/O
  synchronously inside that call. AMDGPU can then proceed with the BO move.
- `io_dmabuf_create_map()` (called later by io_uring workers) waits at
  `DMA_RESV_USAGE_WRITE` before locking, draining GPU WRITE fences so that
  `ttm_bo_validate()` inside `amdgpu_dma_buf_map()` doesn't block while the
  resv is held.

### Trade-off

AMDGPU's eviction path blocks synchronously waiting for in-flight NVMe I/O to
drain. With `iodepth=128` and 4K random reads, this could be O(tens of ms) per
eviction. The GPU eviction path is not latency-sensitive in the same way as
the I/O path, so this is likely acceptable, but it is a real cost.

The previous async design was intended to avoid this: add a fence, return from
`move_notify` quickly, let the worker clean up in background. The flaw was that
the worker then needed to re-acquire the resv lock to unmap, which put it in
conflict with KFD restore.

---

## State of understanding

| Question | Confidence |
|---|---|
| What caused Deadlock 1? | High. KERNEL fence picked up by KFD; NULL-ctx holder can't be wounded by drm_exec; ttm_bo_validate blocks with resv held. |
| Why did BOOKKEEP fix Deadlock 1's primary mechanism? | High. KFD KERNEL scan no longer picks up the fence; pre-lock wait now drains GPU WRITE fences. |
| What caused Deadlock 2 (after BOOKKEEP)? | Medium. Async release worker competing for resv against KFD restore is the ABBA cycle. Exact fence KFD is waiting on in amdgpu_sync_wait is not confirmed from the dmesg alone. |
| Why did WRITE fix not help? | Medium-high. WRITE fence is visible to amdgpu_sync_resv; KFD holds resv and waits for WRITE token fence; release worker has fence but needs resv. Same ABBA. |
| Why does static attach work? | High. BO pinned, move_notify never called, invalidate path never triggers, no release worker or token fence race. |
| Is synchronous drain fix correct? | Medium. Eliminates the competing worker and the resv fence. No known remaining cycle. But AMDGPU blocks during eviction — needs test. |
| Are there other cases? | Unknown. io_dmabuf_token_release_work also takes the resv; same race could apply if a token is released while KFD holds resv. Current code does dma_resv_lock(NULL) there. |

---

## Commit history of fixes

```
000c11d0f04b  io_dmabuf: replace async fence drain with synchronous invalidation
79575a258c93  lib/io_dmabuf_token: use DMA_RESV_USAGE_WRITE for token map-drain fences
beaf686c3bac  lib/io_dmabuf_token: use DMA_RESV_USAGE_BOOKKEEP for token map-drain fences
d7beabda16cb  io_uring/rsrc: add dmabuf backed registered buffers  (Pavel's original)
```

The top commit (`000c11d0f04b`) is the one we believe is correct. The two
preceding commits are the earlier failed attempts and are kept in history to
preserve the debugging context.

---

## Open questions before declaring the fix sound

1. **`io_dmabuf_token_release_work`** takes `dma_resv_lock(dmabuf->resv, NULL)`.
   Can KFD restore hold the same resv while `token_release_work` is running?
   If so, the NULL-ctx vs ww_ctx contention from Deadlock 1 could reappear on
   the release path.

2. **NVMe completion context**: `percpu_ref_put` (which triggers `complete()`)
   runs in NVMe interrupt/softirq context. Is `complete()` safe from that
   context while the sleeping wait in `io_dmabuf_drop_map()` holds the
   `ww_mutex`? Yes — `complete()` only signals the waiter and does not acquire
   the resv. The ww_mutex is held by the CPU thread that called `drop_map`,
   not by the completion handler.

3. **`dma_buf_move_notify` re-entrancy**: If AMDGPU calls `move_notify` from a
   context that already holds locks that NVMe completion callbacks need, the
   drain wait could deadlock. This is unlikely for NVMe (completions only need
   their own queue locks) but worth auditing.

4. **Pre-lock wait level in `io_dmabuf_create_map()`**: Currently `WRITE`.
   If AMDGPU adds READ-level GPU read fences to the BO, those would not be
   drained before locking. `ttm_bo_validate()` (BOOKKEEP wait) would then
   block with resv held. Should the pre-lock wait be BOOKKEEP to be safe?
   The argument against: BOOKKEEP includes the token fence itself, which would
   never be drained while we're trying to create a new map (signal comes after
   the map is live). But the token fence is gone from the resv in the
   synchronous design, so a BOOKKEEP pre-lock wait would be safe.
