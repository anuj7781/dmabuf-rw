# Commit-message rewrite notes

This RFC series contains patches 1-16 from `anuj/dmabuf-v6-clean-fixed`.
The lockdep annotation patch was intentionally left out so the discussion can
focus on the lifetime and invalidation design.

The messages were rewritten using Pavel's v5 series as the style reference.
Straightforward fixes use one connected paragraph. Changes that introduce a
new lifetime rule use a problem paragraph followed by a solution paragraph.
Review-history names, debugging narratives, implementation diaries and claims
about later commits were removed.

## Per-patch changes

1. `dma-buf: document dma_buf_io_map::seg_shift`
   Explains what the field communicates to the block layer instead of merely
   saying that a comment was added.

2. `dma-buf: drop the dma_buf_io_fence wrapper struct`
   Explains that the wrapper and private lock are unnecessary because
   `dma_fence` already contains the required lock.

3. `dma-buf: unwind map creation on missing seg_shift`
   States the invalid-map condition and the resources that must be unwound.

4. `dma-buf: wait for fences under the reservation lock`
   Connects the locking change to the requirement that map creation observe a
   stable fence set.

5. `dma-buf: unwind incomplete DMA-BUF I/O contexts`
   Explains why importer-private state may exist before common validation and
   therefore needs explicit cleanup.

6. `dma-buf: use separate work items for ctx release and destroy`
   Describes the work-structure reinitialization race and the two-work-item
   solution without the earlier call-path walkthrough.

7. `dma-buf: keep target file alive through ctx teardown`
   Relates the file reference directly to the attachment's device lifetime
   during deferred detach.

8. `nvme-pci: unwind DMA-BUF map creation failures`
   Describes the NVMe validation failure and its matching driver cleanup.

9. `dma-buf: pin I/O contexts for map teardown`
   Spells out the exact race: the final context reference can be dropped
   between the release worker's refcount check and increment. It then explains
   why acquiring the pin during map initialization closes that window.

10. `dma-buf: split map pointer and DMA-active lifetimes`
    Uses two paragraphs: the first explains why software ownership and active
    DMA ownership differ; the second describes the kref, percpu_ref and RCU
    roles.

11. `dma-buf: signal the drain fence from the active release`
    Explains why the last active DMA reference is the correct fence-signalling
    point and separates it from deferred unmap.

12. `dma-buf: initialise the drain fence after reserving its slot`
    Uses one paragraph for the required reserve/init/publish ordering and one
    for the synchronous fallback when reservation fails.

13. `dma-buf: run deferred unmap on a WQ_MEM_RECLAIM workqueue`
    Removes reviewer attribution and states the reclaim-forward-progress
    concern and dedicated-workqueue solution directly.

14. `io_uring/rsrc: re-import when the cached dma-buf map is stale`
    Uses one paragraph to describe how a request retains an old generation and
    a second to describe dropping it and importing the current generation.

15. `nvme-pci: acquire dma-buf active references at submission`
    Defines the reclaim cycle that requires moving the reference, identifies
    the first driver DMA-map access as the new boundary, and records that only
    non-reclaiming allocations occur until the reference is released. It also
    explains setup-failure balancing and terminal batch handling. Code comments
    were reduced to API constraints and assertions that are not obvious from
    the control flow.

16. `dma-buf: defer percpu_ref_exit() to the map's final release`
    Explains why a stale software holder can still attempt an active tryget and
    why percpu-ref storage must survive until final map release.

All messages are wrapped for kernel style and retain matching
`Anuj Gupta <anuj20.g@samsung.com>` author and Signed-off-by identities.
