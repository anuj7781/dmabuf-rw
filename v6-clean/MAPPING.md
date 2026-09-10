# Mapping: `anuj/dmabuf-v6-lifetime` (6 commits) -> `anuj/dmabuf-v6-clean` (17 commits)

Old branch, discovery-ordered:

1. `dma-buf: split map lifetime into a software kref and an active percpu_ref`
2. `nvme-pci: acquire the dma-buf active reference at hardware submission`
3. `dma-buf, io_uring: fix ctx UAF and stale-map reuse found in review`
4. `nvme-pci: widen nvme_iod::flags so IOD_DMABUF_ACTIVE fits`
5. `dma-buf: fix ctx teardown races and honour the wait-under-resv rule`
6. `dma-buf: drop the dma_buf_io_fence wrapper struct`

New branch, `anuj/dmabuf-v6-clean`, based directly on Pavel's posted v5
head (`71c75106fd3c`) rather than on old commit 1:

| # | New commit | Derived from old commit(s) |
|---|---|---|
| 1 | `dma-buf: document dma_buf_io_map::seg_shift` | new -- Christoph's unanswered v5 review comment, not previously addressed |
| 2 | `dma-buf: drop the dma_buf_io_fence wrapper struct` | 6 |
| 3 | `dma-buf: unlock the reservation on the seg_shift driver-bug path` | new -- v5 bug found while rewriting commit 5's territory: an early `return` left `dmabuf->resv` held |
| 4 | `dma-buf: wait for fences under the reservation lock` | 5 (the "Clear NAK" half) |
| 5 | `dma-buf: do not leak the dma_buf on the incomplete-ops path` | new -- v5 bug found while rewriting commit 5's territory: the ops-completeness check leaked the dma_buf reference |
| 6 | `dma-buf: use separate work items for ctx release and destroy` | 3 (the `ctx->release_work` reinit-while-running half) |
| 7 | `dma-buf: pin the target file for the ctx lifetime` | 3 (the `fput()`-before-teardown half) |
| 8 | `dma-buf: do not leak ctx on ctx_release_work's WARN_ON_ONCE paths` | 3 (the teardown-WARN leak half) |
| 9 | `nvme-pci: fix map and sgt leak on the seg_shift driver-bug path` | new -- v5 bug found while rewriting commit 2's territory: nvme's `->map()` had the same missing-unwind shape as commit 3 above |
| 10 | `dma-buf: split map lifetime into software and active references` | 1 (kref/active split, RCU free, cached `dev_ops`, per-map ctx pin -- **without** the fence-ordering or workqueue changes 1 originally bundled) |
| 11 | `dma-buf: initialise the drain fence after reserving its slot` | 1 (the fence-ordering half: `dma_fence_init()` moved past `dma_resv_reserve_fences()`, `release_mode`, the sync-drain fallback, and the signal moved out of the worker) |
| 12 | `dma-buf: run deferred unmap on a WQ_MEM_RECLAIM workqueue` | 1 (the dedicated workqueue half, landed *after* 11 -- the original commit's title claimed this made signalling safe, which was never true; that is commit 11's doing) |
| 13 | `io_uring/rsrc: re-import when the cached dma-buf map is stale` | 3 (the `io_dmabuf_reuse_map()` half) |
| 14 | `nvme-pci: acquire the dma-buf active reference at hardware submission` | 2 + 4 (active-ref funnel, `nvme_iod::flags` widened to `u16` in the same commit that adds `IOD_DMABUF_ACTIVE` rather than as a separate follow-up fixing a bug that commit itself introduced, setup-failure rollback, **and** the io_uring-side removal of the import-time active acquisition -- old commit 2 never did this half, leaving the reference double-acquired until this rewrite) |
| 15 | `nvme-pci: end terminal dma-buf failures instead of requeuing them` | 2 (the `nvme_prep_rq_batch()`/`nvme_queue_rqs()` batch-status half) |
| 16 | `dma-buf: defer percpu_ref_exit() to the map's final release` | new territory, not attempted on the old branch -- found while verifying commit 14 above: with active-ref acquisition decoupled from import (14), a kref-holding request can attempt a fresh `active_tryget()` after `percpu_ref_exit()` has already run, in **both** the FENCED worker and the SYNC fallback path |
| 17 | `dma-buf: annotate the fence signalling critical section` | new -- `dma_fence_begin/end_signalling()` around the release callback, with the commit message explicit that nvme's timeout/reset/PCI-recovery escalation is *not* covered |

## Not carried forward as separate commits

- **`IOD_FLAGS_LAST` sentinel + `BUILD_BUG_ON`** (old commit 4): folded into
  new commit 14. In a correctly-ordered history there is no moment where
  `nvme_iod::flags` is deliberately too narrow, so there is nothing for a
  standalone "widen" commit to fix -- the field is declared `u16` in the same
  commit that introduces the bit needing it.

## Known divergence from the old branch (intentional)

New commit 14 removes the import-time `percpu_ref_get()`/`tryget()` calls
from `dma_buf_io_get_map()` / `dma_buf_io_create_map()` in the same diff
that adds nvme's submission-time acquisition. The old branch's commit 2
never did this -- on `anuj/dmabuf-v6-lifetime`, `map->active` is
(harmlessly, since nvme's tryget/put still balance) acquired *twice*: once
at io_uring import and once at nvme submission. This was caught during the
per-commit equivalence review of this rewrite, not before.

## Verification

- All 17 commits build clean at `W=1` (`drivers/dma-buf/dma-buf-io.o`,
  `drivers/nvme/host/pci.o`, `io_uring/rsrc.o`), verified individually via
  `git rebase --exec`.
- All 17 commits pass `checkpatch.pl --strict` with 0 errors/warnings/checks.
- `drivers/dma-buf/dma-buf-io.c` at the tip of `anuj/dmabuf-v6-clean` is
  functionally equivalent to the tip of `anuj/dmabuf-v6-lifetime`: every
  textual difference is comment rewording, a harmless redundant `= NULL`
  the old branch had, or three `EXPORT_SYMBOL_NS_GPL`s confirmed
  unnecessary (matches v5's own unexported originals -- only built-in
  io_uring code calls those functions, verified by symbol reference).
- `include/linux/dma-buf-io.h` differs only by `struct dma_buf_io_vec`,
  which belongs to the local vectored-IO commit and is absent from this
  series' v5-only base.
- `drivers/nvme/host/pci.c` and `io_uring/rsrc.c`: the specific
  funnel/helper functions this series authored
  (`nvme_dmabuf_tryget_active`, `nvme_dmabuf_put_active`,
  `nvme_prep_rq_batch`, `nvme_queue_rqs`, `nvme_req_dmabuf_map`,
  `io_dmabuf_reuse_map`, `io_import_dmabuf`) are byte-identical or differ
  only by line-wrapping/formatting between the two branches.
