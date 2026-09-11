# dma-buf I/O lifetime cleanup

Base: `71c75106fd3c` (Pavel's posted v5)

The series is ordered as independent cleanup and teardown fixes, followed by
map lifetime handling, fence publication ordering, stale-map reimport, and
the NVMe submission boundary. The NVMe commit combines active-reference
acquisition with terminal generation-loss and batched-status handling, so no
intermediate commit can requeue that terminal condition indefinitely.

The corrected stack contains 17 commits. The final tree builds with `W=1`,
and all generated patches pass `checkpatch.pl --strict`.
