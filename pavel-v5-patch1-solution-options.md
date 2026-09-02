# Pavel v5 patch 1: solution options and recommendation

## Decision

Adopt synchronous map draining during dynamic invalidation.  Remove the
importer-owned reservation fence and the map-release work item.  The exporter
already invokes invalidation with `dmabuf->resv` held, so the common layer can
unpublish the map, kill its request refcount, wait for the last request to
complete, call importer unmap while still holding the lock, and free the map.

This intentionally trades bounded exporter-side invalidation latency for a
simple, contract-compliant lifetime.  It is the strongest option for v6 because
it eliminates rather than attempts to annotate or schedule around the
fence-signalling dependency cycle.

## Candidate designs

### 1. Repair the asynchronous reservation-fence design

This would require exact reserve/init/publish ordering, a fully annotated
fence-signalling path, no reclaim-capable allocation after publication, and a
proven forward-progress execution context.  A dedicated reclaim-capable
workqueue might address only one starvation mode; it does not make it safe for
the required signaler to acquire `dmabuf->resv`, because DMA-FENCE explicitly
allows waiters to hold that lock.  Reject for v6.

### 2. Signal before unmapping and unmap without the reservation lock

Reject.  The DMA-BUF locking convention requires dynamic importers to hold the
reservation lock for `dma_buf_unmap_attachment()`.  The earlier local prototype
`0001-io_dmabuf-drop-resv-lock-requirement-from-map-releas.patch` removed this
lock requirement. It conflicts with the current common DMA-BUF contract and
must not be used.

### 3. Synchronously drain and unmap under the reservation lock

Recommended.  `percpu_ref_kill()` prevents new requests.  Its release callback
only calls `complete()`, which is safe from NVMe completion context and does not
need `dmabuf->resv`.  The invalidating exporter sleeps until outstanding device
requests complete, then calls unmap with the required lock still held.  No
reservation fence is published, so no fence signalling path, workqueue
dependency, or second-stage reservation wait exists.

This is stronger than the dynamic-attachment invalidation contract: after the
callback returns, this importer has neither full nor speculative access left.
The bounded wait is the lifetime of already-issued I/O.  NVMe completion does
not depend on the exporter reservation lock, which is the required property to
validate for every importer adopting the common API.

## Required implementation shape

- Replace `dma_buf_io_map::release_work` and `::fence` with a completion.
- Initialise the completion with the `percpu_ref`; its release callback only
  completes it.
- In `dma_buf_io_drop_map()`, under the reservation lock, clear `ctx->map`,
  kill the refcount, wait for completion, invoke `dev_ops->unmap()`, exit the
  refcount, and free the map.
- Simplify context release to lock, drop the final map synchronously, unlock,
  then release importer state.  It no longer waits on map-drain fences.
- Rework map creation to use one reservation-lock-protected wait protocol and
  remove the pre-lock wait and zero-timeout recheck. The single locked wait
  treats `<= 0` as failure (`-EAGAIN` for a zero return), never as permission
  to map. This resolves Sidong Yang's v5 race.

## Validation requirements

- invalidation versus live I/O at high queue depth;
- unregister/context release versus live I/O;
- map creation racing invalidation;
- allocation and interrupted-wait failures;
- lockdep with a dynamic GPU exporter and forced movement;
- reclaim/workqueue stress proving no required fence signaler remains.
- **Before submission, justify `DMA_RESV_USAGE_WRITE` in map creation with a
  GPU exporter that installs WRITE fences.** Verify the intended
  direction-specific file-I/O coherency contract: whether it must implicitly
  wait for producer writes, or whether userspace is responsible for ordering.

## Source basis

`drivers/dma-buf/dma-buf.c` requires importer map and unmap under the
reservation lock.  `drivers/dma-buf/dma-fence.c` states that code required to
signal a published fence cannot acquire a `dma_resv` lock.  Christian König and
Matthew Brost's v4 replies apply those rules directly to this series.
