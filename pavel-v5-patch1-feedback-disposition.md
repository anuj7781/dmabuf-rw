# Pavel v5 patch 1: feedback disposition

## Summary

The v4 feedback arrived after v5 was sent.  It is not a collection of cosmetic
cleanups: the lock/fence feedback identifies a contract violation in v5's
asynchronous invalidation design. `seg_shift` is the only material patch-1
change in v5. The disposition below also records what the proposed
synchronous-drain v6 fix resolves, rather than leaving resolved fence-specific
items looking open.

## Feedback matrix

| Reviewer and point | v5 status | Disposition |
| --- | --- | --- |
| Christian: use embedded DMA-FENCE lock | Still uses wrapper with a separate spinlock. | Moot in the proposed v6 fix: synchronous draining eliminates the map fence. |
| Christian: direct refcount warning/use `kref` | Same direct `refcount_t` pattern. | Resolved in the proposed v6 fix: context lifetime uses `kref`. |
| Christian: fence signalling has strict rules | Worker signals a reservation-published fence. | Resolved in the proposed v6 fix: no map fence is published or signalled. |
| Christian: wait under `dma_resv` rather than before it | v5 retains outside-lock wait plus zero-timeout recheck. | Resolved in the proposed v6 fix: `dma_buf_io_create_map()` waits while holding the reservation lock. |
| Christian: reserve fence slot before creating/initing fence | v5 creates/initialises at map allocation, before invalidation calls `dma_resv_reserve_fences()`. | Moot in the proposed v6 fix: there is no reservation-fence publication. |
| Matthew: `system_wq` may deadlock through reclaim/fence waiting | v5 queues the only required signaler to `system_wq`. | Resolved in the proposed v6 fix: the final request put completes a waiter directly. |
| Christoph: explain `seg_shift` | v5 adds it but the comment remains insufficient. | Resolved in the proposed v6 fix with a map-callback contract comment. |
| Sidong: zero-timeout retry must include `0` | v5 tests `< 0`. | Resolved in the proposed v6 fix: the zero-timeout recheck is gone; the single locked wait treats `<= 0` as failure. |

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

## Remaining correctness question: map-creation dependency class

The proposed implementation waits at `DMA_RESV_USAGE_WRITE` before creating a
map. That is correct only if the file-I/O coherency contract promises implicit
waiting for producer WRITE fences before the target device accesses the buffer.
It must be justified with a real GPU exporter and direction-specific I/O tests;
some GPUDirect-style users may instead require userspace to order GPU and
storage work explicitly. This is independent of the resolved map-lifetime
problem and is the remaining architectural question before proposing v6.

## References

The full quoted discussion is archived in `v4-discussion` near lines 5822,
6327, 6468, and 7002.  The v5 patch and Sidong's follow-up are in
`v5-discussion` near lines 175 and 4459.
