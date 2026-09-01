# Pavel v5 patch 1: feedback disposition

## Summary

The v4 feedback arrived after v5 was sent.  It is not a collection of cosmetic
cleanups: the lock/fence feedback identifies a contract violation in v5's
asynchronous invalidation design.  `seg_shift` is the only material patch-1
change in v5, so all lifecycle feedback below must be considered open unless a
later, unpublished respin changes the design.

## Feedback matrix

| Reviewer and point | v5 status | Disposition |
| --- | --- | --- |
| Christian: use embedded DMA-FENCE lock | Still uses wrapper with a separate spinlock. | Minor cleanup; use the current DMA-FENCE facility if the fence design survives. |
| Christian: direct refcount warning/use `kref` | Same direct `refcount_t` pattern. | Style/robustness review item; not the principal deadlock. |
| Christian: fence signalling has strict rules | Worker signals a reservation-published fence. | Still applicable and fundamental. |
| Christian: wait under `dma_resv` rather than before it | v5 retains outside-lock wait plus zero-timeout recheck. | Still applicable.  The design must use one coherent lock/wait protocol. |
| Christian: reserve fence slot before creating/initing fence | v5 creates/initialises at map allocation, before invalidation calls `dma_resv_reserve_fences()`. | Still applicable.  Publication must enter a no-allocation signalling-critical region. |
| Matthew: `system_wq` may deadlock through reclaim/fence waiting | v5 queues the only required signaler to `system_wq`. | Still applicable and independently corroborated by local v4 hang work. |
| Christoph: explain `seg_shift` | v5 adds it but the comment remains insufficient. | Documentation follow-up; unrelated to lifetime correctness. |
| Sidong: zero-timeout retry must include `0` | v5 tests `< 0`. | Still applicable: a pending fence after the initial wait returns `0`; mapping otherwise proceeds. |

## Christian's core rule in context

For a fence published through `dma_resv`, the safe sequence is:

1. prepare the operation and all fallible allocations;
2. hold the reservation lock and reserve a fence slot;
3. create/init the fence;
4. publish it with `dma_resv_add_fence()`;
5. perform no reclaim-capable allocation until that fence signals.

This is not merely an allocation-order preference.  Reservation-fence readers
are RCU-protected, so the signalling critical path begins at publication, even
before the reservation lock is released.  `dma_fence_begin_signalling()` is a
lockdep annotation, not a mechanism that makes an arbitrary workqueue path
safe.

## Semantics not settled by the original feedback

The map-creation dependency class must be chosen from the actual I/O coherency
contract, not from the old map-drain fence.  A DMA-BUF file-I/O request needs
producer writes complete before device access; that is at least
`DMA_RESV_USAGE_WRITE`.  The implementation must verify the direction-specific
contract and any exporter map callback that waits more broadly.  Removing the
importer drain fence separates this question from revocation lifetime.

## References

The full quoted discussion is archived in `v4-discussion` near lines 5822,
6327, 6468, and 7002.  The v5 patch and Sidong's follow-up are in
`v5-discussion` near lines 175 and 4459.
