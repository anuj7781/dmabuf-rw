# Mapping: `anuj/dmabuf-v6-lifetime` (6 commits) -> `anuj/dmabuf-v6-clean` (19 commits)

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
| 3 | `dma-buf: unwind map creation on missing seg_shift` | new -- v5 bug found while rewriting commit 5's territory: an early `return` left `dmabuf->resv` held |
| 4 | `dma-buf: wait for fences under the reservation lock` | 5 (the "Clear NAK" half) |
| 5 | `dma-buf: unwind incomplete DMA-BUF I/O contexts` | new -- v5 bug found while rewriting commit 5's territory: the ops-completeness check leaked the dma_buf reference |
| 6 | `dma-buf: use separate work items for ctx release and destroy` | 3 (the `ctx->release_work` reinit-while-running half) |
| 7 | `dma-buf: keep target file alive through ctx teardown` | 3 (the `fput()`-before-teardown half) |
| 8 | `nvme-pci: unwind DMA-BUF map creation failures` | new -- v5 bug found while rewriting commit 2's territory: nvme's `->map()` had the same missing-unwind shape as commit 3 above |
| 9 | `dma-buf: pin I/O contexts while releasing their maps` | 1 (the ctx-pin-timing half only: unconditional `refcount_inc()` in `dma_buf_io_init_map()`, replacing the check-then-increment TOCTOU Christian flagged on v4 -- still v5's original single `percpu_ref`, no kref/active split yet) |
| 10 | `dma-buf: split map pointer and DMA-active lifetimes` | 1 (the kref/active split and RCU-delayed free) -- **redesigned**; see below |
| 11 | `dma-buf: do not leak ctx on ctx_release_work's WARN_ON_ONCE paths` | 3 (the teardown-WARN leak half) -- placed after commit 10; see "Ordering fix" below |
| 12 | `dma-buf: signal the drain fence from the active release` | 1 (signal as soon as active users drain instead of making signalling depend on the unmap worker) |
| 13 | `dma-buf: initialise the drain fence after reserving its slot` | 1 (the fence-ordering half: delay `dma_fence_init()` until after `dma_resv_reserve_fences()`, with a synchronous fallback when reservation fails) |
| 14 | `dma-buf: run deferred unmap on a WQ_MEM_RECLAIM workqueue` | 1 (the dedicated workqueue half, landed after signalling was removed from the worker in 12) |
| 15 | `io_uring/rsrc: re-import when the cached dma-buf map is stale` | 3 (the `io_dmabuf_reuse_map()` half) |
| 16 | `nvme-pci: acquire the dma-buf active reference at hardware submission` | 2 + 4 (active-ref funnel, `nvme_iod::flags` widened to `u16` in the same commit that adds `IOD_DMABUF_ACTIVE` rather than as a separate follow-up fixing a bug that commit itself introduced, setup-failure rollback, **and** the io_uring-side removal of the import-time active acquisition -- old commit 2 never did this half, leaving the reference double-acquired until this rewrite) |
| 17 | `nvme-pci: end terminal dma-buf failures instead of requeuing them` | 2 (the `nvme_prep_rq_batch()`/`nvme_queue_rqs()` batch-status half) |
| 18 | `dma-buf: defer percpu_ref_exit() to the map's final release` | new territory, not attempted on the old branch -- with active-ref acquisition decoupled from import (16), a kref-holding request can attempt a fresh `active_tryget()` after `percpu_ref_exit()` has already run, in **both** the FENCED worker and the SYNC fallback path |
| 19 | `dma-buf: annotate the fence signalling critical section` | new -- `dma_fence_begin/end_signalling()` around the release callback, with the commit message explicit that nvme's timeout/reset/PCI-recovery escalation is *not* covered |

## Not carried forward as separate commits

- **`IOD_FLAGS_LAST` sentinel + `BUILD_BUG_ON`** (old commit 4): folded into
  new commit 16. In a correctly-ordered history there is no moment where
  `nvme_iod::flags` is deliberately too narrow, so there is nothing for a
  standalone "widen" commit to fix -- the field is declared `u16` in the same
  commit that introduces the bit needing it.

## Redesign: commit 10's map-ctx lifetime invariant (Codex)

Commits 9-10 were originally a single commit (matching old commit 1's
scope): kref/active split, RCU free, *and* the ctx pin, with the ctx pin
dropped by the unmap worker right after `unmap()` returns. `map->ctx`
could therefore be dangling by the time the RCU free callback ran, so that
version cached `ctx->dev_ops` on the map (`map->dev_ops`) as a workaround
and used that cached pointer instead of dereferencing `map->ctx` in the
free path.

Review (Codex, via `patch-09-map-ctx-lifetime-redesign.md`) proposed a
stronger invariant instead: *a live software reference to a
`dma_buf_io_map` guarantees `map->ctx` remains valid*, not just until
unmap. This removes `map->dev_ops` entirely -- the RCU callback reads
`map->ctx` directly, since the map's own kref-lifetime pin (now held until
the RCU callback itself, not until unmap) keeps it valid by construction.

Verified before implementing:
- No final-release path can free the map before `unmap()` has run:
  the publication kref is still dropped only by the unmap worker (or the
  SYNC fallback), after unmap, unchanged from before.
- The synchronous reserve-failure fallback drops exactly one publication
  kref and leaves the ctx pin for the RCU callback -- no direct
  `dma_buf_io_ctx_put()` call remains in that path.
- After the nvme boundary-move commit (16), any code holding a kref on a
  map can rely on `map->ctx` without a separate pin of its own -- this
  retroactively covers the `map->ctx` dereference in
  `nvme_ns_head_submit_bio()` (the local multipath commit, out of scope
  for this series but noted as a beneficiary).

**Trade-off, stated explicitly in commit 10's message:** this can defer
`ctx->dev_ops->release()` (and with it the target file and `dma_buf`
references) while a stale map software holder remains -- previously that
teardown was gated only on the fence draining (i.e. on unmap), now it is
also gated on every historical map generation's krefs draining. The
number of such holders is bounded (invalidation unpublishes the map from
`ctx->map` before draining it, so no new ones can appear after that
point), but an individual holder's lifetime is not bounded in time. For
the current io_uring consumer this is not a new delay: the same request
already pins the buffer resource node containing `ctx` for its own
lifetime. Other consumers of this generic API inherit the same, stronger
ownership rule and its corresponding release delay.

Split into two commits (9 and 10) rather than landing as one, per the
review's recommendation: the ctx-pin-timing fix (9) is independently
motivated by Christian König's v4 comment and doesn't need the kref/active
split to justify itself -- same granularity principle used elsewhere in
this series.

**Further streamlined after landing:** `dev_ops->free_map()` was removed
entirely, not just the `map->dev_ops` cache. `dma_buf_io_map_free_rcu()`
now does a plain `kfree(map)`. This is sound because every current
consumer (nvme-pci) embeds `struct dma_buf_io_map` as the first member of
its own container struct, so `kfree()` on the base pointer already frees
the whole allocation -- `kmalloc_flex()`/`kfree()` operate on the pointer's
slab metadata, not a driver-declared size. A per-driver `free_map()` hook
would only earn its keep if some future importer's container needed
device-specific cleanup beyond freeing memory (nvme's own `free_map()` was
already just `kfree()`), so it was dropped as unneeded abstraction rather
than kept "for robustness". `nvme_dma_buf_io_ops` no longer has a
`.free_map` entry, and `dma_buf_io_ctx_create()`'s ops-completeness check
no longer requires one.

## Ordering fix: commit 11 was originally placed before commit 10

Earlier still, the ctx-teardown WARN-leak fix (now commit 11) sat at
position 8, grouped with the other ctx-lifetime-hardening commits (6, 7),
*before* the map-lifetime split. That was wrong, caught by review, not by
the build or checkpatch.

`dma_buf_io_ctx_release_work()`'s fix unconditionally falls through to
`dma_buf_io_ctx_put(ctx)` even when its two `WARN_ON_ONCE` checks fire. One
of those checks, `ctx->map` still being non-NULL, is genuinely reachable:
nothing marks `ctx` as closing to new imports, so a concurrent
`dma_buf_io_create_map()` can republish a fresh map between `drop_map()`'s
unlock and the wait returning. Falling through safely in that case depends
on the live map holding its own pin on `ctx`, which only exists once
commit 9 (and, for the full duration, commit 10) has landed. Before that,
falling through to `dma_buf_io_ctx_put()` when a map is still present
could destroy `ctx` while that map's worker still needs it -- the exact
use-after-free class this whole series exists to eliminate, reintroduced
into an intermediate commit by the reordering exercise itself.

Fixed by moving the commit to its current position, after the full
map-lifetime redesign (10), and rewriting its message to state the
dependency explicitly rather than assert "cannot happen" without the
invariant that makes it true. See commit 11's message for the full
reasoning, including the proof that the *other* `WARN_ON_ONCE` (`ret <=
0`) is unreachable for this series' own fence regardless of ordering
(`dma_resv_wait_timeout()` is called with `intr=false` and
`timeout=MAX_SCHEDULE_TIMEOUT`, and the fence never implements a custom
`->wait`, so `dma_fence_default_wait()` cannot return before the fence
signals).

## Known divergence from the old branch (intentional)

- Commit 16 removes the import-time `percpu_ref_get()`/`tryget()` calls
  from `dma_buf_io_get_map()` / `dma_buf_io_create_map()` in the same diff
  that adds nvme's submission-time acquisition. The old branch's commit 2
  never did this -- on `anuj/dmabuf-v6-lifetime`, `map->active` is
  (harmlessly, since nvme's tryget/put still balance) acquired *twice*:
  once at io_uring import and once at nvme submission.
- Commit 10's map-ctx lifetime invariant (above) is strictly stronger than
  anything on the old branch, which never removed `map->dev_ops`. This
  means `drivers/dma-buf/dma-buf-io.c` at this branch's tip is **not**
  expected to be textually or structurally equivalent to
  `anuj/dmabuf-v6-lifetime`'s tip any more -- the redesign is a deliberate
  improvement past what that branch has, not a reordering of the same
  content. See "Verification" below for what was actually checked instead.

## Verification

- All 19 commits build clean at `W=1` (`drivers/dma-buf/dma-buf-io.o`,
  `drivers/nvme/host/pci.o`, `io_uring/rsrc.o`), verified individually via
  `git rebase --exec`, and pass `checkpatch.pl --strict` with 0
  errors/warnings/checks, all re-run after the commit 9/10 redesign and the
  commit 11 reorder.
- Full `make W=1 drivers/dma-buf/dma-buf-io.o drivers/nvme/host/pci.o
  io_uring/rsrc.o` at the tip: clean.
- `include/linux/dma-buf-io.h` differs from `anuj/dmabuf-v6-lifetime` only
  by `struct dma_buf_io_vec`, which belongs to the local vectored-IO
  commit and is absent from this series' v5-only base -- everything else
  (the `dma_buf_io_map` struct shape, the inline accessors) now reflects
  the stronger invariant from commit 10, not the old branch's shape.
- `drivers/nvme/host/pci.c` and `io_uring/rsrc.c`: the funnel/helper
  functions this series authored (`nvme_dmabuf_tryget_active`,
  `nvme_dmabuf_put_active`, `nvme_prep_rq_batch`, `nvme_queue_rqs`,
  `nvme_req_dmabuf_map`, `io_dmabuf_reuse_map`, `io_import_dmabuf`) are
  unaffected by the commit 10 redesign and remain byte-identical or differ
  only by line-wrapping/formatting from the old branch.
