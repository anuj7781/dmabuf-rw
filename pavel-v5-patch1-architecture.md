# Pavel v5 patch 1: DMA-BUF file-I/O architecture

## Purpose and scope

This note describes patch 1 of Pavel Begunkov's v5 `rw-dmabuf` series:
`dma-buf: introduce initial file I/O infrastructure`.  It is deliberately
limited to the common DMA-BUF layer.  NVMe request construction and io_uring
registration are consumers of this API, not part of its ownership model.

The immediate question is whether an importer can cache a device DMA mapping
while allowing a dynamic exporter to revoke that mapping for migration.

## Objects and ownership

`dma_buf_io_ctx` is the long-lived registration object.  It owns a reference
to the dma-buf and importer-private attachment state, and it serialises map
publication through `ctx->map`.

`dma_buf_io_map` is one generation of device mapping.  Its `percpu_ref` holds
the initial cache reference plus one reference for every in-flight request.
RCU protects readers obtaining a live map.  The importer owns the actual DMA
mapping and implements map, unmap, and final release through `dev_ops`.

The original v5 design adds a software fence to each map generation.  On
invalidation it removes the map from `ctx->map`, publishes that fence in the
DMA-BUF reservation object, kills the map refcount, and later signals the
fence from `system_wq` after all request references have drained.

## Required DMA-BUF contract

Dynamic importers must call `dma_buf_map_attachment()` and
`dma_buf_unmap_attachment()` with the dma-buf reservation lock held.  Exporters
call `dma_buf_invalidate_mappings()` with the same lock held.  The exporter may
allow access after invalidation returns only according to the two-stage
invalidation contract: a reservation fence first stops full access, and the
eventual unmap stops speculative read-and-discard access.

Consequently a map generation must not be unmapped until its request references
are gone, and it must be unmapped before the dynamic importer has completed
revocation.  The common layer must not require its completion path to acquire
`dmabuf->resv` after publishing a fence that another holder may wait on.

## v5 lifecycle

| Phase | State and synchronization |
| --- | --- |
| Register | The target file creates the context and its dynamic DMA-BUF attachment. |
| First I/O | A caller obtains `ctx->map` under RCU or creates a new map under the reservation lock. |
| I/O | A request owns one live `percpu_ref`; the device uses the importer-owned DMA addresses. |
| Invalidate | Exporter holds `dmabuf->resv`; common code unpublishes `ctx->map`, adds a map-drain fence, and kills the map ref. |
| Drain | Last request put queues `dma_buf_io_map_release_work` on `system_wq`. |
| Unmap | The worker signals the published fence, takes `dmabuf->resv`, invokes importer unmap, then frees the map. |
| Release | Another worker drops the last map, waits on reservation fences, releases importer state, drops the dma-buf reference, and frees the context. |

## Synchronous-drain lifecycle (proposed v6 fix)

The proposed replacement removes the importer-owned map fence. It keeps the
DMA-BUF locking rule intact: both map and unmap still run with
`dmabuf->resv` held.

| Phase | State and synchronization |
| --- | --- |
| Register | The target file creates the context and its dynamic DMA-BUF attachment. |
| First I/O | A caller obtains a live `ctx->map` under RCU, or takes `dmabuf->resv`, waits for the selected dependencies, creates the map, and publishes it. |
| I/O | A request owns one live `percpu_ref`; completing the request drops that reference without taking `dmabuf->resv`. |
| Invalidate | Exporter holds `dmabuf->resv`; common code unpublishes `ctx->map` and kills the map ref, preventing any new request from acquiring it. |
| Drain | The invalidating caller waits on a completion. The final request put directly completes it; it does not queue work or signal a reservation fence. |
| Unmap | The same invalidating caller invokes importer unmap, exits the refcount, and frees the map while still holding `dmabuf->resv`. Only then does invalidation return. |
| Release | Release work takes `dmabuf->resv`, runs the same synchronous map drop, unlocks, releases importer state, drops the dma-buf reference, and frees the context. |

This makes revocation stronger than the original two-stage dynamic-importer
contract for this importer: when the invalidation callback returns, no old map
is published or usable, including for speculative read-and-discard access.

## Why this design is difficult

DMA fences are a global dependency mechanism.  Once a fence is visible in a
`dma_resv`, every operation required to signal it is a fence-signalling critical
path.  Kernel DMA-FENCE documentation explicitly permits waiting on a fence
while holding a reservation lock; therefore that signalling path cannot later
need the reservation lock.  The v5 map-release worker does exactly that.

The worker is also scheduled on `system_wq`.  Reclaim may wait on a DMA fence,
while the work item that signals the fence may itself require a worker thread.
Matthew Brost described the resulting saturated-workqueue deadlock from Xe in
the v4 thread.  A code path being asynchronous does not remove this dependency;
it makes the execution context part of the fence protocol.

## v4 to v5 delta

Patch 1 is nearly unchanged between v4 and v5.  v5 adds `map->seg_shift` and a
`WARN_ON_ONCE(!map->seg_shift)` after the importer map callback, for block-layer
splitting.  It does not change the early fence allocation/initialisation, the
outside-lock fence wait, the map-release worker, or the reservation-fence
publication and later signaling model discussed by Christian König and Matthew
Brost after v5 had been posted.

## Primary sources

- `v4-discussion`, Christian König, 2026-08-05 and 2026-08-06.
- `v4-discussion`, Matthew Brost, 2026-08-06.
- `v5-discussion`, Christoph Hellwig, 2026-08-04; Sidong Yang, 2026-08-08.
- `drivers/dma-buf/dma-buf.c`: locking convention and
  `dma_buf_invalidate_mappings()` contract.
- `drivers/dma-buf/dma-fence.c`: signalling critical-path rules.
- Proposed fix: `0001-dma-buf-synchronously-drain-file-I-O-maps-on-invalid.patch`.
