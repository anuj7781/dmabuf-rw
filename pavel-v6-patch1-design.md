# dma-buf-io.c: proposed v6 design

Supersedes the sync-drain recommendation in `pavel-v5-patch1-solution-options.md`
and the `~/.claude/plans/glistening-fluttering-cherny.md` plan. Both contain
conclusions retracted during analysis; see "Retractions" at the end.

All file:line citations are against `pavel/rw-dmabuf-v5` unless noted.

## Implementation status

First-cut prototype on branch `anuj/dmabuf-v6-lifetime`
(worktree `/tmp/dmabuf-v6-lifetime`, based on `pavel/rw-dmabuf-v5`), two commits:

1. `dma-buf: split map lifetime into a software kref and an active percpu_ref`
   -- §3.1-§3.2.4: the kref/active split, RCU-safe destruction via a cached
   `dev_ops` pointer (map->ctx can be freed before the last software ref
   drops), fence-init ordering fix, `release_mode` branch for the
   `dma_resv_reserve_fences()` failure fallback.
2. `nvme-pci: acquire the dma-buf active reference at hardware submission`
   -- §3.2.2, §3.2.5: the `nvme_rq_setup_dmabuf()` tryget funnel,
   `IOD_DMABUF_ACTIVE` enforcement (`nvme_req_dmabuf_map()` asserts it),
   local rollback on every setup-failure exit, `nvme_prep_rq_batch()` status
   preservation so generation loss ends the request instead of looping through
   `requeue_list`, and the `BLK_STS_IOERR` terminal-status choice (documented
   as an `-ESTALE` placeholder) that also suppresses transparent io_uring
   reissue by construction (`io_rw_should_reissue()` only fires on `-EAGAIN`).

Both object files build clean (`W=1`, no warnings) against a config with
`CONFIG_DMA_SHARED_BUFFER=y`, `CONFIG_BLK_DEV_NVME=m`. checkpatch clean on the
diff. Full vmlinux link (for cross-module `EXPORT_SYMBOL` resolution) run
separately; see commit log / session notes for the result.

**PROTOTYPE SCOPE, stated in commit 1's message**: this implements mapping
revocation and lifetime only. The §5 WRITE-fence blocker is deliberately
untouched -- `dma_buf_io_create_map()` still waits `DMA_RESV_USAGE_KERNEL`
only, matching v5. Do not read that as an implicit resolution of §5; it is
still gated on Christian's answer.

**Not yet done** (§7 open items, still open): the `-EAGAIN`-equivalent reissue
trace end-to-end, the reissue-path liveness check against `ctx->map`
(`io_import_dmabuf()`'s `REQ_F_DROP_DMABUF` fast path), short-I/O semantics
across split requests, and lockdep/stress validation. None of these are design
questions anymore -- they're verification work against the code above.

---

## 1. Constraints we must not break

These are settled. Any design that violates one is wrong regardless of how
otherwise attractive it is.

| # | Rule | Source |
| --- | --- | --- |
| R1 | No pinning. Long-term imports by new importers must use dynamic attachment. | Christian König + Christoph Hellwig, v1 (`cover.1751035820.git.asml.silence@gmail.com`) |
| R2 | Fence ordering: allocate object → prepare incl. **all** allocations → `dma_resv_reserve_fences()` → `dma_fence_init()` → `dma_resv_add_fence()` → unlock. | Christian, `v4-discussion:7050` |
| R3 | After `dma_fence_init()`, no GFP_KERNEL/GFP_IO/GFP_FS **on the path required to signal the fence** until it signals. | Christian, `v4-discussion:7065` |
| R4 | Nothing on the fence-signalling path may acquire `dma_resv`. Waiters are permitted to hold it while waiting. | `Documentation/driver-api/dma-buf`, DMA-FENCE rules |
| R5 | Signal-before-unmap is **legal**. Stage 1 (fence) stops full access; stage 2 (unmap) stops speculative read-and-discard. Do not "fix" this ordering. | `dma_buf_invalidate_mappings()` kdoc, `drivers/dma-buf/dma-buf.c` |
| R6 | Unmap must occur within **bounded time** after the invalidate callback. | same kdoc |
| R7 | Fence signalling must not depend on reclaim, on workqueue availability, or on userspace task scheduling. | R3 + Matthew Brost (v4) + empirical, see §2.3 |
| R8 | Map creation waits `DMA_RESV_USAGE_KERNEL`. Do **not** upgrade to `WRITE`. | matches `ib_umem_dmabuf_map_pages()`; see §5 |
| R9 | No per-I/O fences in `dma_resv`. | `dma_resv` is a ww_mutex (per-I/O lock at Mi-IOPS); `dma_resv_add_fence()` replaces same-context fences, assuming a serial timeline that out-of-order NVMe completion violates |
| R10 | The map struct must stay allocated while software holds a pointer to it. | `bio_split_io_at_dmabuf()` dereferences `bio->bi_dmabuf_map->seg_shift`, `block/blk-merge.c:329` |
| R11 | Strip `Co-Authored-By` before any upstream-oriented posting. | project convention |

---

## 2. What is actually wrong in v5

Three **independent** defects. Fixing any one does not fix the others.

### 2.1 The map ref covers reclaim-capable code

The active reference is taken at io_uring import time and released at request
cleanup:

```
io_uring/rsrc.c:1328   io_import_dmabuf():   map = dma_buf_io_get_map(&db->ctx);
io_uring/rsrc.c:1303   io_drop_dmabuf_node(): dma_buf_io_map_drop(req->dmabuf_map);
```

The window spans `bio_alloc`, `blk_mq_get_tag`, request setup, and bio
splitting — all reclaim-capable. A thread holding the ref can enter direct
reclaim, reclaim can wait on the map fence, and the fence waits on that thread.
This is Christian's stated objection (R3) and it is not auditable: the window
covers the whole block layer and would have to *stay* audited forever.

### 2.2 Fence ordering

`dma_fence_init()` runs at map creation; `dma_resv_reserve_fences()` runs at map
destruction. Violates R2. Christian on the fix: *"Yeah that is a good start."*

### 2.3 On IOPOLL and reissue, the fence signals from io_uring task-work

Not in the v4 review. **Scope correction:** this does *not* apply to every
completion. Ordinary async completion is fine — `io_complete_rw()` drops the ref
*before* queueing task-work:

```c
/* io_uring/rw.c:589 */
static void io_complete_rw(struct kiocb *kiocb, long res)
{
	io_req_drop_dmabuf(req);              /* ← before task-work */
	...
	__io_req_task_work_add(req, IOU_F_TWQ_LAZY_WAKE);
}
```

But `io_complete_rw_iopoll()` (`io_uring/rw.c:602`) does **not** drop at all. It
only sets `REQ_F_REISSUE` on `-EAGAIN` and stores `iopoll_completed`; the drop is
deferred to `io_free_batch_list()`:

```
io_uring/io_uring.c:1117   io_req_drop_dmabuf(req);
io_uring/io_uring.c:1094   if (req->flags & REQ_F_REISSUE) { ... continue; }  ← skips it
```

That runs in io_uring **task-work**, which under `DEFER_TASKRUN` executes only
when the userspace task is woken or enters the kernel. So on the IOPOLL and
reissue paths a `dma_fence` that memory reclaim can wait on is gated on userspace
scheduling — an unbounded dependency, violating R7. IOPOLL + `DEFER_TASKRUN` is
exactly the configuration this feature targets.

This is empirically confirmed by
`0001-debug-trace-map-kill-I-O-drain-fence-signal-lazy-wak.patch`: for a stuck
map, *"all I/O completed pre task-work"* fires and `will_wake=1`, but *"fence
signalled"* never does.

The same `REQ_F_REISSUE` branch re-queues a completed request without dropping
the map ref. This is **not** a leak — `io_import_dmabuf()` deliberately reuses
the retained reference:

```c
if (req->flags & REQ_F_DROP_DMABUF) {
	map = req->dmabuf_map;
	goto init_iter;          /* reuses the existing ref by design */
}
```

It is correct in v5, where the retained ref also blocks unmap. It becomes a
problem under §3, where the active ref is released at completion: that `goto`
skips any liveness check, so a reissued request rebuilds against a possibly
killed map, fails `tryget` at `queue_rq`, and reissues again — a livelock. See
§7.

---

## 3. The design

Keep the map-wide drain fence. It is the contract-sanctioned mechanism for an
importer that cannot stop device access immediately (`include/linux/dma-buf.h:409`:
*"Dynamic importers should set fences for any access that they can't disable
immediately from their invalidate_mappings callback"*). NVMe cannot abort a
submitted command, so a fence is required.

### 3.1 Two-level map refcount

The single refcount is doing two unrelated jobs. Split it:

| Reference | Tracks | Held from → to | Blocks unmap? |
| --- | --- | --- | --- |
| `struct kref refs` | software holds a pointer | `io_import_dmabuf()` → `io_drop_dmabuf_node()` | no |
| `struct percpu_ref active` | **hardware is doing DMA** | `nvme_queue_rq` → NVMe completion | yes |

Only `active` is in the fence's dependency set, because only `active` is bounded
by hardware rather than by scheduling. `kref` satisfies R10.

**This is only safe if preparation code never touches the DMA mapping** — the
`kref` keeps the struct allocated, but the mapping itself is torn down when
`active` hits zero. The code currently satisfies this:

- The generic `struct dma_buf_io_map` holds no DMA addresses. They live in the
  driver container `struct nvme_dmabuf_map` (`drivers/nvme/host/pci.c:475`):
  `sgt`, `prp_array`, `prp_array_dma`, `dma_list[]`.
- Reaching them requires `to_nvme_dmabuf_map()`. All call sites are inside
  `queue_rq` or completion (`pci.c:962,984,1016,1413,1453`) or the unmap callback
  (`pci.c:3122`). None are in preparation.
- Outside NVMe there is exactly one map dereference in the tree:
  `block/blk-merge.c:329`, reading `seg_shift` — a plain `unsigned` in the
  generic struct, safe to read post-unmap while the struct is allocated.
- `bio_iov_iter_set()` (`block/bio.c:1189`) copies no addresses into the bio,
  only `(map pointer, bi_offset, bi_size)`.

**Invariant (must be enforced, currently only accidental):**

> Mapping-dependent state may only be accessed while holding an `active`
> reference. The generic `struct dma_buf_io_map` must contain no
> mapping-dependent state.

Enforcement. Documentation alone is not sufficient — a later driver change would
silently reintroduce use-after-unmap. Make it structural:

1. Add `IOD_DMABUF_ACTIVE` to `enum nvme_iod_flags`. Set it when
   `percpu_ref_tryget_live()` succeeds, clear it on put.
2. Replace `to_nvme_dmabuf_map(map)` with a **request-scoped** accessor
   `nvme_req_dmabuf_map(req)` that asserts the flag:

   ```c
   static struct nvme_dmabuf_map *nvme_req_dmabuf_map(struct request *req)
   {
   	struct nvme_iod *iod = blk_mq_rq_to_pdu(req);

   	WARN_ON_ONCE(!(iod->flags & IOD_DMABUF_ACTIVE));
   	return container_of(req->bio->bi_dmabuf_map,
   			    struct nvme_dmabuf_map, base);
   }
   ```

   This is an always-on flag test, not a debug-only check, and it makes the
   accessor impossible to call without a request in hand — so generic and block
   code physically cannot reach driver DMA state.
3. Keep one raw container accessor for the teardown callback (`pci.c:3122`),
   which by definition runs at `active == 0`. Name it distinctly
   (`nvme_dmabuf_map_from_base()`) so the two uses cannot be confused.
4. State the invariant in the `dma-buf-io.h` kerneldoc next to the refcount split.

Without this the design is one careless patch away from a use-after-unmap, which
is the substance of Codex's objection even though the race does not exist today.
The alternative — gating unmap on `prep users == 0 && active users == 0` — is
safe but strictly worse: it delays stage 2 behind threads that can sit in
reclaim, for no benefit given the invariant above.

### 3.2 Lifecycle

```
map create (dma_resv held):
    dma_resv_wait_timeout(..., DMA_RESV_USAGE_KERNEL, ...)   [R8]
    dev_ops->map()
    percpu_ref_init(&map->active)
    kzalloc(fence memory)                    ← memory only; NOT dma_fence_init [R2]
    rcu_assign_pointer(ctx->map, map)

io_import_dmabuf (no dma_resv):
    kref_get(&map->refs)                 ← exactly where v5 takes its active ref
    req->dmabuf_map = map
    iov_iter_dmabuf_map(iter, ..., map, ...)

bio build:
    bio->bi_dmabuf_map = map             ← covered by the request's kref
    ... bio_alloc / tag / split: allocates freely, holds NO active ref ...

nvme_queue_rq:
    percpu_ref_tryget_live(&bio->bi_dmabuf_map->active)
        fail → terminal generation-loss error, no transparent reissue (§3.2.5)
    read DMA addresses, build PRP/SGL      ← GFP_ATOMIC only (pci.c:1063,1477,...)
    ring doorbell

invalidate_mappings (exporter holds dma_resv):
    rcu_assign_pointer(ctx->map, NULL)
    dma_resv_reserve_fences(dmabuf->resv, 1)          [R2 step 3]
    dma_fence_init(preallocated)                      [R2 step 4]
    dma_resv_add_fence(..., DMA_RESV_USAGE_KERNEL)    [R2 step 5]
    percpu_ref_kill(&map->active)
    return                                            ← non-blocking

NVMe completion (IRQ/softirq):
    percpu_ref_put(&map->active)
    last put → dma_fence_signal()      ← irq-safe, no dma_resv [R4], stage 1 done
              queue_work(dma_buf_io_wq)

cleanup worker (WQ_MEM_RECLAIM):
    dma_resv_lock → dev_ops->unmap() → dma_resv_unlock   ← stage 2 [R5], bounded [R6]
    percpu_ref_exit
    kref_put(&map->refs)                 ← worker holds one kref while running

io_drop_dmabuf_node:
    kref_put(&map->refs)                 ← last put frees the struct
```

### 3.2.1 Teardown is two independent events

This is the part Codex flagged as unspecified. There are two terminal conditions
and they are deliberately decoupled:

| Event | Trigger | Effect |
| --- | --- | --- |
| `active` → 0 | last submitted request completes | signal fence (stage 1); queue worker → `dev_ops->unmap()` (stage 2). **The DMA mapping is gone.** |
| `refs` → 0 | last software pointer holder drops it | `kfree(map)`. **The C object is gone.** |

**They cannot occur in either order** — an earlier draft of this document said
they could, which was wrong. The references nest: the active window is contained
in the request's `kref` window, and the ctx's publication `kref` outlives unmap.
So `refs == 0` implies `active == 0` *and* unmap already complete. A free can
never precede an unmap; only the reverse is reachable (a lingering importer
holding a software reference after the mapping is gone).

Since that ordering is guaranteed, `percpu_ref_exit()` belongs on the `refs == 0`
side, not the unmap side — it is the only point at which its "no longer in
active use" contract actually holds. See `dma_buf_io_map_free_rcu()`.

Answering Codex's list directly:

- **Where is the first preparation reference acquired?** `io_import_dmabuf()`,
  replacing v5's `dma_buf_io_get_map()`.
- **How is the import→first-bio interval covered?** By that same reference; it is
  held for the whole request, not per bio.
- **How is it transferred across split bios and requeues?** It isn't. One `kref`
  per io_uring request dominates every derived bio — `__bio_clone()` copies the
  pointer (`block/bio.c:866`) — and survives requeue, because the request
  outlives all of them.
- **Which callback performs unmap?** The `WQ_MEM_RECLAIM` worker queued from the
  `percpu_ref` release callback.
- **How does cleanup avoid unmapping while a preparation reference remains?** It
  does not need to. Preparation never reads the mapping (§3.1), so unmapping
  underneath a preparation reference is safe by the stated invariant. Gating
  unmap on preparation references would be *less* safe: those references are held
  by threads that can sit in direct reclaim, so an exporter awaiting stage-2
  cessation would wait on reclaim — a potential cycle, and a violation of R6,
  which the current split satisfies.

### 3.2.2 Active-reference ownership

The active reference is the most correctness-critical object in the series: a
missed put wedges the fence forever, a double put is a use-after-free. Ownership
rule:

> Acquire immediately before the first access to driver map state — no earlier,
> no later. Every exit after that point must reach **exactly one** put, whether
> by completion, cancellation, or setup-failure rollback.

Note the acquisition point cannot be deferred until "after all fallible setup",
because the fallible setup *is* what reads the map: `nvme_pci_dmabuf_sgl_nents()`
(`pci.c:1536`) dereferences it before any descriptor construction. Deferring
would read driver DMA state without an active reference — the exact
use-after-unmap §3.1 prevents.

**Partially verified pairing in `nvme_prep_rq()`:**

```c
ret = nvme_setup_cmd(...);      /* fallible, runs BEFORE acquisition — no ref  */
ret = nvme_map_data(req);       /* dmabuf setup + tryget live here             */
if (ret) goto out_free_cmd;     /*   ← does NOT reach the unmap path           */
ret = nvme_map_metadata(req);
if (ret) goto out_unmap_data;   /*   ← nvme_unmap_data() →                     */
                                /*     nvme_rq_clean_dmabuf_map() → put. OK.   */
```

So the `nvme_map_metadata()` failure path is already correctly paired, and
`nvme_setup_cmd()` needs nothing. The remaining gap is failures *inside*
`nvme_rq_setup_dmabuf()`, which exit via `out_free_cmd` and never reach the
unmap path — rollback for those is local to the function being written.

Implement through three helpers so every exit can be audited mechanically:

```c
nvme_dmabuf_tryget_active(req);      /* sets IOD_DMABUF_ACTIVE on success */
nvme_dmabuf_put_active(req);         /* no-op unless the bit is set       */
nvme_dmabuf_cleanup_failed_setup(req);
```

`IOD_DMABUF_ACTIVE` — the same bit §3.1 uses to gate the map accessor — also
gates the put, and must be set immediately on a successful `tryget_live()`,
before any descriptor or metadata setup. This carries most of the audit burden:

- **At-most-once is free.** `nvme_dmabuf_put_active()` tests the bit, so it is
  safe to call unconditionally from any cleanup or cancellation path regardless
  of how far setup progressed. Double puts become structurally impossible rather
  than something to prove.
- **At-least-once becomes a runtime assertion.** `WARN_ON_ONCE()` on
  `IOD_DMABUF_ACTIVE` still being set when the request is freed or recycled
  catches every missed put under test, instead of requiring proof by inspection.

**The assertion has a trap.** `nvme_prep_rq()` opens with `iod->flags = 0`, so a
requeued request silently clears `IOD_DMABUF_ACTIVE` — erasing the evidence of a
leaked reference before any teardown assertion can see it. Assert **before** that
line:

```c
static blk_status_t nvme_prep_rq(struct request *req)
{
	struct nvme_iod *iod = blk_mq_rq_to_pdu(req);

	WARN_ON_ONCE(iod->flags & IOD_DMABUF_ACTIVE);   /* leaked from a prior pass */
	iod->flags = 0;
```

Without this the leak detector is defeated by exactly the path — requeue — that
is most likely to leak.

Together these reduce the §7 pairing work from "prove every path is exactly
right" to "call the helper on every plausible exit, and let the assertions find
what was missed." Given the number of paths involved — batch requeue, timeout,
abort, reset, controller removal — that difference is what makes the audit
tractable.

**Verified blk-mq behaviour that constrains this:**

- A `tryget` failure is **not** a requeue. `blk_mq_dispatch_rq_list()`
  (`block/blk-mq.c:2122`) requeues only `BLK_STS_RESOURCE` and
  `BLK_STS_DEV_RESOURCE`; every other status falls to
  `default: blk_mq_end_request(rq, ret)`. Generation loss must end the request,
  not requeue it.
- **`BLK_STS_AGAIN` must not be used for this.** `include/linux/blk_types.h:113`:
  *"BLK_STS_AGAIN should only be returned if RQF_NOWAIT is set and the bio would
  block."* A blocking dma-buf request rejected for generation loss satisfies
  neither condition. See §3.2.5.
- `BLK_STS_RESOURCE` **is** a requeue, and the requeued request runs prep
  **again**. So an active reference retained across requeue becomes a double
  take with one leak. Rollback before returning `BLK_STS_RESOURCE` is mandatory.

**The batch path cannot express our failure mode, and must be fixed.**
`nvme_prep_rq_batch()` returns `bool` — `return nvme_prep_rq(req) == BLK_STS_OK;`
— discarding the status, and `nvme_queue_rqs()` puts every failure on
`requeue_list`. A `tryget` failure therefore becomes a requeue rather than
`-EAGAIN`.

An earlier draft claimed this still terminates because the requeued request falls
back to the status-preserving single-request path. **That is not guaranteed** —
the same request can return through `queue_rqs()` again with the same dead map,
and the map never comes back. Do not rely on the fallback. Fix it directly:

- **Preferred:** change the batch helper to preserve `blk_status_t`, complete
  generation-loss requests via `blk_mq_end_request()`, and requeue only
  `BLK_STS_RESOURCE` / `BLK_STS_DEV_RESOURCE`.
- **Alternative:** exclude dma-buf requests from `queue_rqs()` entirely and issue
  them through the single-request path. Simpler, but gives up batching for the
  workload this feature exists to accelerate.

Two further consequences:

1. The rollback must happen inside `nvme_prep_rq()`, not in any completion path,
   because a batch-path failure never reaches completion.
2. `nvme_submit_cmds()` must be called only for requests whose `tryget` **and**
   mapping setup both succeeded — which holds today, since only
   `nvme_prep_rq_batch() == true` requests reach `submit_list`.

**Audit list.** Every return path from these must be checked for exactly one
put: `nvme_prep_rq()`, `nvme_prep_rq_batch()`, `nvme_queue_rq()`,
`nvme_queue_rqs()`, timeout/abort, controller reset and removal, normal
completion.

### 3.2.5 Generation loss is terminal, not retryable

Two independent findings converge on the same decision, which simplifies things:

- `BLK_STS_AGAIN` is contractually unavailable (`blk_types.h:113` — RQF_NOWAIT
  only).
- Transparent reissue after partial submission re-issues already-transferred
  ranges (§7), which is unsafe duplicate I/O.

So: **a request rejected at `queue_rq` for generation loss completes terminally
and is not reissued.** Userspace resubmits.

Implementation note on the status. `BLK_STS_IOERR` is the smallest change, but
it is not free — userspace sees `-EIO` for a recoverable condition, which
applications reasonably treat as device failure. A distinct status mapping to
something like `-ESTALE` ("the mapping you hold is stale") describes the
condition honestly. Since the errno is user-visible, settle it with Pavel in the
same exchange as the reissue-policy question — they are one decision, not two.

### 3.2.3 RCU-delayed map destruction

Map lookup is RCU-based (`percpu_ref_tryget_live_rcu()` today). Converting the
software reference to a `kref` is insufficient on its own: an RCU reader can have
loaded `ctx->map` but not yet taken a reference when the final `kref_put()` runs,
and an immediate `kfree()` frees the struct underneath it.

Required:

- Add `struct rcu_head` to `struct dma_buf_io_map`. **The grace period must
  defer freeing the containing allocation, not just the embedded base.**
  `struct nvme_dmabuf_map` embeds `base` as its first member, so `kfree_rcu()` on
  the base address happens to free the container correctly today — but that is a
  layout accident and must not be relied on. Prefer `call_rcu()` with a callback
  that invokes a driver `dev_ops->free_map()`, letting the driver free its own
  container; that stays correct regardless of member ordering.
- The initial `kref` is the **publication/core reference**: taken when the map is
  published to `ctx->map`, dropped by the unmap worker. It must not be acquired
  opportunistically after the last request reference could have vanished.
- Unpublish with `rcu_assign_pointer(ctx->map, NULL)` before killing `active`, so
  no new reader can find it.

### 3.2.4 `dma_resv_reserve_fences()` failure has no error return

`invalidate_mappings()` returns `void`, but `dma_resv_reserve_fences()` can fail
with `-ENOMEM`. §3.2 specifies only the success path. v5 has a fallback here and
v6 still needs one.

The late active boundary makes a synchronous fallback defensible, because the
`active` release path takes no `dma_resv` and cannot reclaim:

```
under dma_resv, reserve_fences() fails:
    rcu_assign_pointer(ctx->map, NULL)
    percpu_ref_kill(&map->active)
    wait_for_completion(&map->drain)     ← preallocated at map create
    dev_ops->unmap()                     ← still holding dma_resv
    release map
```

No fence is initialised or published on this path, so R2/R3/R4 do not apply to
it. **The `percpu_ref` release callback must therefore signal the completion
directly**, not queue unmap work — queued work would need the same `dma_resv`
the failing caller already holds. In the normal path the same callback signals
the fence and queues the worker; the callback must branch on whether a fence was
published.

**The mode must be recorded in the map before `percpu_ref_kill()`**, not decided
inside the release callback. The callback can become runnable the instant the
kill happens, so the branch input has to be established first:

```c
map->release_mode = FENCED_RELEASE | SYNCHRONOUS_DRAIN;   /* set BEFORE kill */
percpu_ref_kill(&map->active);
```

```
release callback:
    FENCED_RELEASE:     dma_fence_signal(); queue_work(unmap)
    SYNCHRONOUS_DRAIN:  complete(&map->drain)
```

This is the one place the rejected synchronous-drain shape is correct: it is an
exceptional, allocation-failure-only path, not the steady state.

### 3.3 Why this satisfies R3

R3 forbids reclaim-capable allocation **on the path required to signal the
fence**. After `dma_fence_init()`, that path is exactly: already-submitted
requests → device → IRQ → `percpu_ref_put` → `dma_fence_signal`. Every step is
hardware or `GFP_ATOMIC`.

A thread sitting between bio-build and `queue_rq` when the fence is published
*may* allocate GFP_KERNEL and *may* enter reclaim. That is not a violation,
because that thread is not required to signal the fence — its `tryget` will fail
and it will retry. This is the crux of the design: the fix is not to forbid
allocation, it is to keep allocating threads out of the fence's dependency set.

### 3.4 What the ref move fixes

Moving acquire to `nvme_queue_rq` and release to the NVMe completion collapses
the ref window to one hardware round trip with no software scheduling inside it,
which resolves all three defects of §2 at once:

- §2.1 reclaim cycle — no allocating thread holds an active ref.
- §2.3 task-work dependency — the ref drops in IRQ, not `io_free_batch_list()`.
It does **not** fix the `REQ_F_REISSUE` path, which needs separate work (§7).

---

## 4. Rule-compliance check

| Rule | How the design satisfies it |
| --- | --- |
| R1 | Dynamic attachment retained; no `dma_buf_pin()` anywhere. |
| R2 | `reserve_fences` → `dma_fence_init` → `add_fence`, all under `dma_resv` in invalidate. Fence *memory* preallocated at map create; only `init` is ordered. |
| R3 | Post-`init` signalling path is device + IRQ + `GFP_ATOMIC`. See §3.3. |
| R4 | `dma_fence_signal()` runs in IRQ and takes no lock. Unmap takes `dma_resv` but runs **after** signal, so it is not on the signalling path. |
| R5 | Signal (stage 1) precedes unmap (stage 2), as the kdoc prescribes. Between them no I/O is in flight, so NVMe trivially satisfies "no writes". |
| R6 | Unmap is queued on signal to a dedicated `WQ_MEM_RECLAIM \| WQ_UNBOUND` workqueue. Boundedness is **conditional**, not automatic — `WQ_MEM_RECLAIM` addresses starvation only. All four must hold: (a) the worker never waits on the fence it follows, and the fence is already signalled before it takes `dma_resv`; (b) the rescuer thread means reclaim cannot starve it; (c) `dma_resv` is held only for bounded periods by exporters doing migration; (d) no teardown worker holding `dma_resv` waits on this unmap worker. Context and attachment lifetime must extend through unmap. |
| R7 | Signalling depends only on NVMe completion — no task-work, no `system_wq`, no reclaim. Boundedness must hold on **every** terminal path, not just normal completion: normal IRQ completion; timeout → `nvme_timeout()` → abort or `nvme_dev_disable()`; controller reset and removal → `nvme_cancel_tagset()` → `blk_mq_tagset_busy_iter(nvme_cancel_request)` force-completes every outstanding request, then `blk_mq_tagset_wait_completed_request()` (`drivers/nvme/host/core.c:561`); requeue → request re-enters `queue_rq`, so the ref is put and retaken. Each of these must be confirmed to drop `active` exactly once — see §7. |
| R8 | `DMA_RESV_USAGE_KERNEL` retained at map creation. |
| R9 | One fence per map generation, published only at invalidation. No per-I/O fences. |
| R10 | Outer `kref` keeps the struct alive across block-layer transit. |
| R11 | To be stripped at posting time. |

---

## 5. BLOCKER: WRITE fences vs. explicit ordering

**This is unresolved and gates implementation. It needs Christian's agreement
before any code is written.**

`include/linux/dma-buf.h:400`, under an explicit `DYNAMIC IMPORTER RULES:`
heading:

> - Dynamic importers **must obey the write fences and wait for them to signal
>   before allowing access** to the buffer's underlying storage through the
>   device.

`DMA_RESV_USAGE_KERNEL` does not include WRITE fences (`KERNEL<WRITE<READ<BOOKKEEP`,
and asking for one class returns only lower ones). So R8 as stated — wait
`KERNEL` only, delegate data ordering to userspace — **contradicts a documented
"must"** for dynamic importers. Documenting explicit synchronization in the
io_uring uAPI does not override the DMA-BUF contract; only the DMA-BUF
maintainer can grant that.

The RDMA precedent (§5.1) is evidence about practice, not a licence. It cannot
justify violating an explicit rule in the importer contract.

Three possible resolutions, in preference order:

1. **Ask Christian for an explicit opt-out.** If granted, it must land as a
   documented exception or an attachment capability in DMA-BUF itself — not as
   prose in io_uring documentation.
2. **Obey the rule.** Wait for WRITE dependencies before each access and close
   the wait-to-submit race under `dma_resv`. Given R9 (no per-I/O fences), this
   implies a coarse batch or barrier interface, not a per-request wait. The
   map-creation-only wait is insufficient either way.
3. **Reject:** quietly changing the map-creation wait to WRITE. A GPU WRITE fence
   installed after map creation still races every request using the cached map,
   so this buys the appearance of conformance without the substance.

Until this is settled, §5.1 below describes the *intended* contract, not an
agreed one.

## 5.1 Intended uAPI contract: explicit ordering

R9 means we cannot provide implicit GPU↔NVMe data ordering. The map-creation
`KERNEL` wait answers *"is it safe to establish this mapping"* — not *"has the
producer finished writing"*. A GPU can install a WRITE fence after the map is
cached, and subsequent I/Os on that cached map will not see it.

This must be documented, not left implicit:

> Userspace is responsible for ordering GPU and storage operations on a
> registered dma-buf. Overlapping GPU and NVMe access to the same buffer
> produces undefined **contents**; it never produces undefined **behavior** —
> the mapping and the backing pages remain valid throughout.

This matches RDMA (which publishes no fences at all — `git grep dma_resv_add_fence
drivers/infiniband/` is empty) and matches cuFile/GPUDirect Storage practice,
where the application synchronizes its stream before issuing I/O.

Do **not** upgrade the wait to `DMA_RESV_USAGE_WRITE`. That buys implicit
ordering at one arbitrary instant and abandons it for every cached-map I/O
afterwards — it would make the code look like it does implicit sync while the
race stays open.

---

## 6. Implementation steps

1. **`include/linux/dma-buf-io.h`** — split the refcount: add `struct kref refs`,
   rename the existing `percpu_ref` to `active`. Document `seg_shift`
   (Christoph's v5 request): power-of-2 shift encoding the device DMA segment
   size, used by `bio_split_io_at_dmabuf()`, must be set by the driver's `map()`.
2. **`drivers/dma-buf/dma-buf-io.c`** — move `dma_fence_init()` into
   `dma_buf_io_drop_map()` after `dma_resv_reserve_fences()`; preallocate fence
   memory in `dma_buf_io_init_map()`. Signal directly from the `percpu_ref`
   release callback; queue only unmap. Add the `WQ_MEM_RECLAIM` workqueue.
3. **`drivers/dma-buf/dma-buf-io.c`** — fix the zero-timeout recheck in
   `dma_buf_io_create_map()` (Sidong Yang): `dma_resv_wait_timeout(..., 0)`
   returning `0` means "not signalled", currently treated as success at line 148.
4. **`drivers/nvme/host/pci.c`** — the acquire and release points are both single
   funnels, verified:

   - **Acquire:** first statement of `nvme_rq_setup_dmabuf()` (`pci.c:1527`),
     *before* the `use_sgl == SGL_UNSUPPORTED` branch. This dominates every
     reader of driver map state on the submit side —
     `nvme_pci_dmabuf_sgl_nents()` (called at `:1536`),
     `nvme_rq_setup_dmabuf_sgl()`, `nvme_rq_setup_dmabuf_map()`, and
     `nvme_dmabuf_map_sync_for_device()`. Acquiring merely "before the doorbell"
     would be too late; `nvme_pci_dmabuf_sgl_nents()` reads the map first.
   - **Release:** last statement of `nvme_rq_clean_dmabuf_map()` (`pci.c:1000`),
     *after* `nvme_dmabuf_map_sync_for_cpu()`, which still reads map state.

   Every path from a successful `tryget` must reach exactly one put. Verified
   paired: normal completion, and `nvme_map_metadata()` failure (via
   `out_unmap_data`). Needs new local rollback: the `BLK_STS_IOERR` returns
   inside `nvme_rq_setup_dmabuf_*` (e.g. `pci.c:1524`), which exit through
   `out_free_cmd` and never reach the unmap path. Still unverified:
   `BLK_STS_RESOURCE` requeue, `blk_mq_requeue_request()`, timeout/abort, and
   controller reset or removal — see §3.2.2 and §7.
5. **`io_uring/rsrc.c`** — convert the existing get/put in `io_import_dmabuf()` /
   `io_drop_dmabuf_node()` from the active `percpu_ref` to the `kref`. Same call
   sites, same lifetime; only the reference class changes. Do **not** move them
   to bio build — the iterator returned by import already carries the map
   pointer, so the interval between import and first bio construction must be
   covered.
6. **uAPI docs** — the §5 contract.

---

## 7. Open items

**These are not implementation hygiene — they determine whether the architecture
is correct.** If the two ownership rules below do not survive the reissue, batch,
timeout and cancellation paths, the design changes rather than merely the code.
Do not present it as settled until each is verified against a running kernel.

```
active ref:  acquired immediately before first driver-map access;
             released exactly once, by completion, cancellation, or rollback.

software kref: held from io_import_dmabuf() until the io_uring request and every
             derived bio are finished.
```

- **Reissue must re-import against the current map.** `io_import_dmabuf()`'s
  `REQ_F_DROP_DMABUF` fast path reuses `req->dmabuf_map` without a liveness
  check (`io_uring/rsrc.c:1332`). Under §3 that livelocks against a killed map.
  The fast path must verify the map is still `ctx->map` and, if not, drop the
  `kref` and re-import. This is a prerequisite, not an optimisation.
- **NVMe recovery-progress audit (fence forward-progress bound).** The drain
  fence cannot signal until the last `active` reference drops, so every path
  that completes an outstanding request is on the fence's signalling path and
  must reach completion without a GFP_KERNEL allocation. Verified safe:
  all three nvme workqueues are `WQ_UNBOUND | WQ_MEM_RECLAIM`
  (`core.c:5482`), kblockd is `WQ_MEM_RECLAIM | WQ_HIGHPRI`, the abort
  request uses `BLK_MQ_REQ_NOWAIT`, and the terminal
  `nvme_dev_disable()` → `nvme_cancel_tagset()` / `nvme_reap_pending_cqes()`
  force-completes every request allocation-free.

  Two paths remain unbounded and need follow-up:

  1. `case NVME_CTRL_RESETTING: return BLK_EH_RESET_TIMER;` — timeouts
     re-arm indefinitely while a reset is in flight, and `nvme_reset_work`
     allocates GFP_KERNEL (queue setup, tagset, IRQ vectors). Structural
     cycle: reclaim → our fence → request completion → reset work →
     GFP_KERNEL → reclaim.
  2. `if (pci_channel_offline(pdev)) return BLK_EH_RESET_TIMER;` — bounded
     only by AER/EEH recovery, which is not a bound we control.

  Status: structurally present, **not reproduced**. No concrete reclaim path
  waiting on this dmabuf's fences has been traced. Scope is an audit of these
  two paths, not a demonstrated lifetime bug — the lifetime plumbing itself is
  internally consistent.

  Candidate fix if the audit confirms it matters: give the drain its own bound
  rather than inheriting NVMe's — arm a delayed work at `percpu_ref_kill()`
  time and, on expiry, invoke a new `dev_ops->force_drain()` backed by the
  driver's existing allocation-free forced-completion primitive. Note
  `nvme_dev_disable(dev, false)` is *not* usable as-is: it calls
  `nvme_delete_io_queues()`, which allocates admin requests. The exact
  primitive (likely `nvme_cancel_tagset()` directly, or the `dead`/shutdown
  variant) still needs tracing.

  Gating question for Christian, alongside §5: what does dma-buf require of an
  importer whose hardware cannot be made to complete in bounded time? If the
  answer is "you must be able to force-revoke", the above is mandatory; if it
  is "the device timeout is the accepted bound", document the two holes and
  move on.

  Note on annotation scope: `dma_fence_begin_signalling()` currently wraps only
  `dma_buf_io_map_active_release()`. Extending it to nvme's shared completion
  path is *not* the fix — those paths are shared with all non-dmabuf I/O and
  are not ours to annotate. Keep the annotation on the dma-buf-specific funnels.

- **Retry propagation.** `BLK_STS_AGAIN` → `-EAGAIN` → io_uring reissue. The
  O_DIRECT reissue path plausibly covers it but has not been traced. Needs a
  retry bound so a GPU thrashing migration cannot spin a request indefinitely.
- **Put pairing on every non-completion exit.** See §3.2.2 for the ownership rule
  and the audit list. Confirmed hazard: requeued requests re-run prep, so a
  retained reference double-takes. Still unverified: whether a request failing
  setup routes through `nvme_rq_clean_dmabuf_map()` at all, and whether
  timeout/reset cancellation reaches it. This is the most correctness-critical
  item in the series.
- **BLOCKER: transparent reissue is unsafe after partial submission.** One direct
  I/O produces several requests. If invalidation lands between their `queue_rq()`
  calls, request A submits and transfers data while request B fails `tryget`.
  `blkdev_dio` records the first bio error and returns that errno; it does not
  produce a reliable completed-prefix byte count. So the whole operation
  completes `-EAGAIN`, io_uring restores the iterator and reissues **everything**
  — re-issuing A's already-transferred range. This is not ordinary short-I/O
  handling.

  The hazard is **unsafe duplicate I/O**, not necessarily data corruption.
  Repeating a write to a conventional namespace is often content-idempotent. But
  it violates exactly-once semantics, and becomes genuinely unsafe with
  concurrent writers to the same LBA range, zoned namespaces (zone append and
  write-pointer advance are not idempotent), or any other externally observable
  effect. That framing is the one to use on the list — it does not depend on
  proving a corruption case.

  Resolution: do **not** auto-reissue an operation rejected at this boundary.
  Complete it to userspace with a distinct, non-auto-reissued error and require
  resubmission. If `-EAGAIN` is wanted as the uAPI result, io_uring needs to
  distinguish "retry from nonblocking submission" from "dma-buf generation
  revoked after partial submission."

  The alternative — one active lease spanning the whole logical I/O — reinstates
  the reclaim window unless every bio, split and request is fully prepared before
  acquisition, which §3.2.2 shows is not achievable.
- **Staggered `queue_rq` across split fragments.** All fragments share one map
  (`__bio_clone`, `block/bio.c:866`), so they cannot straddle generations — but
  they reach `queue_rq` at different times, so invalidation between them gives a
  partially-submitted I/O needing retry. Generic short-I/O semantics; needs
  specifying.
- **Lockdep validation.** `CONFIG_PROVE_LOCKING` with a dynamic GPU exporter,
  forced migration concurrent with I/O submission under memory pressure. This is
  the experiment that would catch a residual reclaim path blocking on
  `dma_resv`.

---

## 8. Out of scope

- NVMe passthrough and multipath (not yet supported; deferred by decision).
- Per-I/O READ/WRITE fences — see R9 and §5. If a user needing implicit ordering
  appears, the answer is a coarse explicit barrier op covering a batch, not
  per-I/O fences in `dma_resv`. Worth one sentence in the cover letter as future
  work.

---

## 9. Retractions

Recorded so they are not reintroduced:

- **"Signal before unmap is wrong."** False — R5. This appeared as "Issue 3" in
  the old plan. A grep of `v4-discussion` finds no such objection from Christian;
  it was not his feedback.
- **"Eliminate the fence, drain synchronously."** Not required, and worse: it
  blocks the exporter's eviction path for the duration of an I/O and forgoes the
  contract's intended revoke model.
- **"Use `DMA_RESV_USAGE_WRITE` at map creation."** No — R8, §5.
- **"Pin the buffer / offer a pinned mode."** NAK'd in v1 — R1. RDMA's pinned
  path is grandfathered legacy, not a template for new importers.
- **"The ref-boundary move is a small delta."** It is not. It is the hard part,
  and without the two-level refcount it is a use-after-free.
- **"`REQ_F_REISSUE` leaks the map reference."** False. The retention is
  deliberate and `io_import_dmabuf()` reuses it. The real problem is the missing
  liveness check on that path under the §3 design — see §2.3 and §7.
