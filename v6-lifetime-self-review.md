# v6 map-lifetime series: adversarial review against DMA-BUF / DMA-fence rules

Reviewing `anuj/dmabuf-v6-lifetime` (3 commits over `pavel/rw-dmabuf-v5`) the way
Christian König reviewed v4: against the documented contracts, not against
whether it happens to work. Every finding below is verified against the tree,
with the citation.

Severity: P1 = must fix before posting, P2 = will be caught in review,
P3 = open items from v4 still not addressed.

---

## P1-1. RETRACTED — `DMA_RESV_USAGE_KERNEL` is *not* a known regression

**Everything in this section is withdrawn. It is preserved only so the error is
not silently repeated.**

The claim below was that switching the four `DMA_RESV_USAGE_KERNEL` sites to
`BOOKKEEP` was an already-diagnosed, already-fixed issue that this series had
regressed. That is false, and it misrepresents the project's own debugging
history. From `dmabuf_v4_debugging_timeline.md`:

- `KERNEL` → Deadlock 1
- `BOOKKEEP` (`beaf686c3bac`) → **Deadlock 2** — the BOOKKEEP patch did not fix
  anything; it produced a second deadlock
- `WRITE` (`79575a258c93`) → still deadlocked
- synchronous invalidation (`000c11d0f04b`) → resolved

The usage class was never the fix. The v4 deadlock was resolved by dropping the
map in the correct place in io_uring, which has nothing to do with the usage
class. Citing `0001-lib-io_dmabuf_token-use-DMA_RESV_USAGE_BOOKKEEP-for-.patch`
as "the fix that already existed" was presenting a superseded intermediate patch
as proven.

The usage class may still be worth revisiting on its own merits — an importer's
I/O-drain fence arguably is not kernel memory management — but that is an open
design question for Christian, not a regression, and it must not be argued from
the discredited BOOKKEEP patch.

<details>
<summary>Original (wrong) text, retained for the record</summary>

### [RETRACTED] `DMA_RESV_USAGE_KERNEL` is wrong, and this is a REGRESSION

Four sites in `drivers/dma-buf/dma-buf-io.c` (225, 242, 354, 385) use
`DMA_RESV_USAGE_KERNEL`. Carried forward from v5 without question.

The usage class is documented (`include/linux/dma-resv.h`):

> @DMA_RESV_USAGE_KERNEL: For in kernel memory management only.
> This should only be used for things like copying or clearing memory with a
> DMA hardware engine for the purpose of kernel memory management.
> Drivers *always* must wait for those fences before accessing the resource

An importer's I/O-drain fence is not kernel memory management. It is the
importer's own bookkeeping about when its cached mapping stops being used.
Publishing it at KERNEL over-claims the highest priority class in the system
and forces *every* KERNEL-priority walker to block on unrelated NVMe I/O.

This is not a theoretical objection. **This project already diagnosed this as a
real, reproduced deadlock and already wrote the fix**, which is sitting in this
same directory: `0001-lib-io_dmabuf_token-use-DMA_RESV_USAGE_BOOKKEEP-for-.patch`.
From its commit message:

> Using DMA_RESV_USAGE_KERNEL for this fence is wrong: it causes any kernel
> path that walks reservation fences at KERNEL priority -- most notably the
> AMDGPU KFD restore path in amdgpu_amdkfd_gpuvm_restore_process_bos() -- to
> block on outstanding fio/NVMe I/O lifetime. At the same time, io_uring
> workers may need to create a new map via io_dmabuf_create_map(), which also
> contends on the dma-buf reservation lock used by KFD restore. This can form
> a lock/fence dependency cycle.

Note also `ib_umem_dmabuf_map_pages()` waits `DMA_RESV_USAGE_KERNEL` — so an
unrelated RDMA importer establishing a mapping would block on our NVMe I/O
draining. That coupling is nonsense.

**Fix:** all four sites to `DMA_RESV_USAGE_BOOKKEEP`, per the existing patch.
BOOKKEEP is documented as "No implicit sync... preemption fences, page table
updates" — bookkeeping the exporter's memory manager still honours (TTM's
`ttm_bo_wait_ctx()` waits BOOKKEEP) but which does not drag unrelated
importers into our I/O lifetime. This also *widens* the `create_map` waits
correctly, since BOOKKEEP returns all fences.

This one is the reason to be paranoid rather than pleased: the analysis was
already done, the fix already existed in the tree, and it was silently undone
by rebuilding the same code path from the v5 baseline.

</details>

## P1-2. [FIXED] `percpu_ref_exit()` is called while the ref is still reachable

`dma_buf_io_map_release_work()` does `percpu_ref_exit(&map->active)` and then
`kref_put()`. But `percpu_ref_exit`'s own contract
(`lib/percpu-refcount.c:120`) is:

> The caller is responsible for ensuring that @ref is no longer in active use.

We cannot ensure that. The whole point of the kref/active split is that a
request can hold *only* a software kref while sitting in the block layer.
Concretely:

```
req imports map N               -> holds kref, no active ref
exporter invalidates            -> percpu_ref_kill(N)
other in-flight I/O drains      -> active hits 0
release worker runs             -> unmap, percpu_ref_exit(&N->active)
req finally reaches queue_rq    -> dma_buf_io_map_active_tryget(&N->active)
                                   ...on an exited percpu_ref
```

Reachable, not exotic — it is the ordinary "request was queued behind
something slow while a migration happened" case.

It is benign *today* only by implementation accident:
`__percpu_ref_exit()` sets `ref->percpu_count_ptr = __PERCPU_REF_ATOMIC_DEAD`
(`lib/percpu-refcount.c:116`), and `percpu_ref_tryget_live_rcu()` tests
`!(ref->percpu_count_ptr & __PERCPU_REF_DEAD)` *before* it would dereference
`ref->data` — which `percpu_ref_exit()` has already set to NULL. So the tryget
returns false instead of NULL-dereferencing. Change either of those two
implementation details upstream and this becomes an oops.

**Fix:** move `percpu_ref_exit()` out of the unmap worker and into the map's
final free path (the `->free_map()` RCU callback), where by construction no
kref holder remains. Same for the synchronous-drain fallback in
`dma_buf_io_drop_map()`. The invariant then becomes checkable: *you may call
`active_tryget()` on any map you hold a kref to* — which is exactly the
contract the header claims.

---

## P2-1. One fence context shared across all map generations

`ctx->fence_ctx = dma_fence_context_alloc(1)` once per ctx (`:419`); each map's
fence uses that context with `atomic_inc_return(&ctx->fence_seq)` (`:351-352`).

`dma_resv_add_fence()` (`drivers/dma-buf/dma-resv.c`) will *silently replace*
an existing fence:

```c
if ((old->context == fence->context && old_usage >= usage &&
     dma_fence_is_later_or_same(fence, old)) || dma_fence_is_signaled(old)) {
        dma_resv_list_set(fobj, i, fence, usage);
        dma_fence_put(old);
```

So if generation N's fence were ever still unsignaled when generation N+1's
fence is added, N's fence is dropped from the reservation object. The exporter
then stops waiting for in-flight DMA on the *old* mapping and is free to move
memory underneath it. That is straightforward memory corruption.

It is currently unreachable, because `dma_buf_io_create_map()` waits for the
outstanding fence before creating a replacement map. But that is an invariant
enforced in a *different function*, undocumented at the `dma_resv_add_fence()`
site, and silently load-bearing. Anyone touching the create_map wait
(including the P1-1 usage change) is one edit away from a corruption bug with
no warning.

**Fix:** allocate the fence context per map, not per ctx. Costs one u64 and
makes the replacement structurally impossible rather than incidentally
avoided.

## P2-2. [PARTIALLY FIXED] No `dma_fence_begin_signalling()` / `dma_fence_end_signalling()`

These annotations exist precisely so lockdep can prove a fence's signalling
path takes no locks that could deadlock against memory reclaim
(`drivers/dma-buf/dma-fence.c:292`).

The central claim of this whole series is that the active-ref window is
allocation-free and therefore safe as a fence signalling path. That claim is
currently made only in commit messages and comments. Annotate it and lockdep
will check it on every boot with `CONFIG_PROVE_LOCKING`, including on the
paths nobody thought to test (timeout, abort, controller reset).

Without this, the argument for the design rests on my having read the NVMe
submission path correctly. With it, the kernel enforces it.

**Status:** the annotation now wraps `dma_buf_io_map_active_release()`, which is
the cheap end of the signalling path and was never the part in doubt. The
expensive end — the NVMe funnels that must run for the last `active` reference
to drop — is deliberately *not* annotated: those paths are shared with all
non-dmabuf I/O and are not ours to place inside a fence signalling section.
The residual exposure is tracked as the recovery-progress audit in the design
doc's §7, not as an annotation gap.

## P2-3. [FIXED] Error paths leak `ctx` and its `dma_buf` reference

`dma_buf_io_ctx_release_work()`:

```c
if (WARN_ON_ONCE(ret <= 0))
        return;
if (WARN_ON_ONCE(rcu_dereference_protected(ctx->map, true)))
        return;
if (refcount_dec_and_test(&ctx->refs))
        ...
```

Both early returns skip the final decrement, permanently leaking `ctx`, the
`dma_buf` reference it holds, and the driver's attachment. `dma_buf_io_ctx_destroy_work()`
has the same shape.

A WARN is a "this shouldn't happen" marker, not a licence to leak the object.
These should unwind, or at minimum still drop the reference.

---

## P3. v4 review items from Christian that are *still* not addressed

These were raised on v4, acknowledged, and have now survived two more
iterations. They will be raised a third time.

**P3-1. `struct dma_buf_io_fence` wrapper is unnecessary.** Christian, v4:

> Upstream has changed to allow embedding the spinlock into the dma_fence, so
> this structure here is most likely not necessary any more.

Verified present in this tree — `struct dma_fence` now has:

```c
union {
        spinlock_t *extern_lock;   /* "(deprecated)" per the kdoc */
        spinlock_t inline_lock;
};
```

and `dma_fence_init()` (`drivers/dma-buf/dma-fence.c:1080-1082`) uses
`inline_lock` when passed a NULL lock. The external lock is *explicitly
documented as deprecated*. So the wrapper struct and its `spinlock_t lock`
should both go; `map->fence` becomes a plain `struct dma_fence *` initialised
with a NULL lock argument.

**Thread status: agreed but not delivered.** Pavel's reply to this comment
(`v4-discussion:6347`) was a one-word "ok". He accepted it and then never made
the change — v5's `dma-buf-io.c` is byte-identical here, and so is ours. That
makes this a real outstanding item rather than a speculative one.

**P3-2. RETRACTED — `ctx->refs` as `refcount_t` is a settled question.**

This section previously claimed Christian's question had "never been answered"
and that we should either convert or have a reason ready. Both halves are
wrong; the thread answers it.

Christian, v4 (`v4-discussion:5956`):

> And why are you using refcount directly instead of kref?

Pavel, same thread (`v4-discussion:6374`):

> Not sure it'd make much difference here.

Christian replied to that message on 2026-08-06 07:29 — the **last message in
the thread** — and did not raise it again. He picked up the `kfree(map)` point
and gave the fence-ordering protocol; kref went unmentioned.

Note also that the kref question was bundled with the TOCTOU objection ("Stuff
like that is usually illegal" on `refcount_read() == 0` followed by
`refcount_inc()`). That was the substantive half and it is already fixed: we
take an unconditional pin in `dma_buf_io_init_map()`. What remains is a style
question the author declined and the maintainer dropped.

**Do not convert it.** Doing so would re-litigate a settled point and would
override the series author's stated preference on his own code. The only thing
worth keeping is awareness that `map->refs` is a `kref` while `ctx->refs` is a
`refcount_t` — if that inconsistency is ever raised, this is the history.

Method note: the original claim came from reading Christian's message without
checking whether it had been answered. Same error as the retracted P1-1 above.
Before citing any maintainer comment as outstanding, read to the end of the
thread.

---

## Lower priority / worth knowing

**RCU grace period under `dma_resv` in the sync-drain fallback.**
`percpu_ref_kill()` defers the switch to atomic mode via `call_rcu()`, so
`wait_for_completion(&map->drain)` in `dma_buf_io_drop_map()` blocks for at
least a full RCU grace period *while holding the exporter's reservation lock*,
even when zero I/O is in flight. Exceptional path (only on
`dma_resv_reserve_fences()` failure, i.e. under memory pressure) — but memory
pressure is exactly when an exporter is trying to evict, so it stalls the
thing that is already struggling. Acceptable, worth a comment so it is not
rediscovered as a mystery latency spike.

---

## Out of scope for this series: `map->ctx` lifetime

A software `kref` holder may outlive both unmap and ctx destruction, so
`map->ctx` must not be dereferenced without an active reference. The core
handles this (hence the cached `map->dev_ops`).

`nvme_ns_head_submit_bio()` violates it — `multipath.c:549` reads
`bio->bi_dmabuf_map->ctx` in the *routing* path, before the active tryget:

```c
ns = nvme_ns_head_find_pinned_path(head,
		nvme_dmabuf_pinned_ctrl(bio->bi_dmabuf_map->ctx));
```

**This is not a defect in this series.** Pavel's posted v5
(`b36c780ae003..71c75106fd3c`) does not touch `multipath.c` at all; the
dereference comes from our own local commits `088c3d700c40` /
`1351fc3e3a91`. Nothing in the posted scope dereferences `map->ctx` outside
the active window, and the design is self-consistent without change.

Recorded here so it is fixed on the multipath patch before that is posted.
It also means the "hold the ctx pin until the final map kref" restructuring
is a simplification, not a repair — do not justify it as a bug fix.

## What this review did not cover

The WRITE-fence contract question (design doc §5) remains deliberately out of
scope and is still the gating item for the series as a whole. Nothing above
changes that; these are all defects *within* the revocation/lifetime design,
which is the part that was supposed to be finished.
