# Draft private note to Pavel

Subject: rw-dmabuf v6 / patch-1 lifetime work

Hi Pavel,

I have been revisiting the v5 DMA-BUF file-I/O infrastructure, especially the
late v4 feedback from Christian and Matthew that arrived after v5 was posted.
Do you have a plan or approximate timeline for a v6, and are you already
reworking patch 1?

I traced the map/invalidation lifecycle against the current DMA-BUF locking and
DMA-FENCE rules.  My current conclusion is that the hard problem is not just
the fence allocation ordering: once the map-drain fence is published, the
required `system_wq` release path becomes part of its signalling path, but it
also has to acquire `dmabuf->resv` for dynamic-importer unmap.  Fence waiters
are allowed to hold that lock, so this creates the cycle Christian and Matthew
were warning about.

I have a small alternative direction prepared: make dynamic invalidation
synchronously kill and drain the map refcount while the exporter holds the
reservation lock, then unmap under that lock.  That removes the importer fence
and release worker entirely.  It trades bounded invalidation latency for a
much simpler DMA-BUF lifetime and avoids the fence/reclaim dependency.

I am documenting the reasoning and can share a tested v5-based patch if that
is useful.  Before posting anything wider, I would value knowing whether that
direction matches what you had in mind for v6 or whether you are pursuing a
different model.

Thanks,
Anuj
