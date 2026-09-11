# dma-buf I/O lifetime cleanup

Base: `71c75106fd3c` (Pavel's posted v5)

The 15 commits are ordered as follows:

1. Documentation and wrapper cleanup.
2. Map/context error unwinding and reservation-lock ordering.
3. Context teardown lifetime fixes.
4. Map software and DMA-active lifetime split.
5. Stale-map reimport in io_uring.
6. NVMe submission-time active references and terminal batch status.
7. Fence publication ordering, signalling annotation, and deferred active
   reference teardown.

The NVMe submission commit contains the active-reference move, rollback paths,
generation-loss status, and batched-request status propagation atomically.
The final tree was built with `W=1`; all generated patches pass
`checkpatch.pl --strict`.
