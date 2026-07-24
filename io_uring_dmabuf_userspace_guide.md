# io_uring dmabuf I/O: Userspace Guide

## What this is

This interface lets an io_uring application do direct block I/O **into and out
of a dmabuf** — memory that is often *device memory* exported by another driver
(a GPU's VRAM, an accelerator's on-device RAM, a NIC, or a plain memfd wrapped
by udmabuf). Reads and writes to a storage device (e.g. an NVMe SSD) land
straight in that memory.

The point is to **remove the CPU bounce buffer**. Traditionally, to move data
from an NVMe drive into GPU memory you would:

```
NVMe --DMA--> system RAM (bounce buffer) --copy--> GPU VRAM
```

With this interface, when the platform supports PCIe peer-to-peer (P2P) DMA, the
storage device DMAs directly to/from the exporter's memory:

```
NVMe --DMA (P2P)--> GPU VRAM
```

No staging buffer, no CPU copy, no extra RAM bandwidth. When P2P is not
available the same code still works correctly — the exporter falls back to
placing the buffer in system memory — you just lose the direct-to-device
benefit. Correctness never depends on P2P; only the data path does.

---

## The big picture

There are three actors:

1. **The exporter** — the driver that owns the target memory and exports it as a
   dmabuf fd (e.g. amdgpu exporting VRAM, or `/dev/udmabuf` wrapping a memfd).
2. **The importer / target device** — the block device that will DMA to that
   memory. Today this is an NVMe device (`drivers/nvme/host/pci.c` implements the
   token callback). The device must be opened `O_DIRECT`.
3. **io_uring** — the glue. You register the dmabuf as a *registered buffer*
   bound to a specific target file, then issue normal `READ_FIXED` / `WRITE_FIXED`
   requests against that file.

Internally, registration creates a long-lived **dmabuf token**: a persistent DMA
mapping of the exporter's memory for the target device. This mapping is built
once at registration and reused for every I/O, so you don't pay the
attach/map/unmap cost per request. The exporter can still move or reclaim its
memory underneath you (see *Invalidation* below); io_uring and the driver
cooperate via dma_fences to keep that safe.

---

## Requirements

- Kernel built with `CONFIG_DMABUF_TOKEN=y`.
- A block driver that implements the `create_dmabuf_token` blk-mq op. Currently
  **NVMe (nvme-pci)** only.
- The target block device opened with **`O_DIRECT`** — buffered I/O is rejected
  (`-EINVAL` at registration).
- dmabuf size **≤ 1 GiB** (`SZ_1G`). Larger dmabufs are rejected with `-EINVAL`.
- For the zero-copy P2P data path specifically: `CONFIG_PCI_P2PDMA=y` **and** a
  PCIe topology where the storage device and the exporter can reach each other
  (ideally under a common PCIe switch). Without this the buffer is served from
  system memory instead — still correct, just not device-to-device.

---

## Step by step

### 1. Get a dmabuf fd

This comes from whatever exports your target memory. For a GPU you'd use the
driver's export ioctl (e.g. amdgpu GEM → `DMA_BUF_IOCTL` export). For testing,
`/dev/udmabuf` can wrap a sealed memfd:

```c
/* create a dmabuf backing 'size' bytes (udmabuf test path) */
int memfd = memfd_create("buf", MFD_ALLOW_SEALING);
fcntl(memfd, F_ADD_SEALS, F_SEAL_SHRINK);
ftruncate(memfd, size);

struct udmabuf_create create = { .memfd = memfd, .offset = 0, .size = size };
int dmabuf_fd = ioctl(udmabuf_devfd, UDMABUF_CREATE, &create);

/* optional: a CPU mapping so you can fill/verify the buffer from userspace */
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, dmabuf_fd, 0);
```

For real device memory (GPU VRAM), you generally **cannot** `mmap` it usefully
on the CPU — the whole point is that only the device touches it.

### 2. Open the target device O_DIRECT

```c
int dev_fd = open("/dev/nvme0n1", O_DIRECT | O_RDWR);
```

### 3. Set up a ring and a registered-buffer table

The dmabuf is installed into a slot of the ring's registered-buffer table. Create
the table first (sparse is fine):

```c
struct io_uring ring;
io_uring_queue_init(64, &ring, 0);
io_uring_register_buffers_sparse(&ring, 1);   /* 1 slot */
```

### 4. Register the dmabuf into a buffer slot

Registration uses the **extended** buffer-update descriptor
(`struct io_uring_regbuf_desc`) with type `IO_REGBUF_TYPE_DMABUF`. You bind the
dmabuf to a specific **target file** here — the dmabuf token is created against
that file's device, and requests using this buffer *must* be issued against the
same file.

```c
struct io_uring_regbuf_desc rd;
memset(&rd, 0, sizeof(rd));
rd.type      = IO_REGBUF_TYPE_DMABUF;
rd.dmabuf_fd = dmabuf_fd;   /* the exporter's dmabuf */
rd.target_fd = dev_fd;      /* the O_DIRECT block device */
/* rd.uaddr and rd.size MUST be 0 for dmabuf — size comes from the dmabuf */

struct io_uring_rsrc_update2 up;
memset(&up, 0, sizeof(up));
up.data = (unsigned long)&rd;
up.nr   = 1;
up.offset = 0;                            /* buffer table index to fill */
up.resv = IORING_RSRC_UPDATE_EXTENDED;    /* select extended desc format */

int ret = io_uring_register(ring.ring_fd, IORING_REGISTER_BUFFERS_UPDATE,
                            &up, sizeof(up));
/* ret == number of descriptors processed (1 on success) */
```

At this point the kernel has:
- fetched the dmabuf and the target file,
- created a persistent dmabuf token (long-term DMA mapping) for the device,
- installed a registered buffer of size = `dmabuf->size` in table slot 0.

### 5. Issue READ_FIXED / WRITE_FIXED

Use the ordinary fixed-buffer read/write helpers **against the target file**.
The buffer index selects your dmabuf slot.

> **Key addressing detail.** For a dmabuf registered buffer the "buffer address"
> argument is **not** a pointer — it is a **byte offset into the dmabuf**. The
> valid range is `[0, dmabuf->size)`. The kernel sets the buffer's base to 0, so
> whatever value you pass is used directly as the offset within the dmabuf.

```c
/* read dev at file offset `f_off`, into the dmabuf at byte offset `buf_off` */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, dev_fd,
                         (void *)buf_off,   /* offset INTO the dmabuf */
                         len,               /* bytes */
                         f_off,             /* device offset */
                         0 /* buf_index */);
io_uring_submit(&ring);
```

Write is symmetric with `io_uring_prep_write_fixed()`. Completions arrive as
normal CQEs; `cqe->res` is the byte count (or a negative errno).

---

## Handling spurious -EAGAIN (important)

A dmabuf-backed request can complete with **`-EAGAIN` even on a correctly
submitted request**. This is expected and is part of the design: it happens when
the exporter has invalidated the mapping (see below) and the request needs to be
re-driven. **You must be prepared to reissue the request unchanged when you see
`-EAGAIN`.** Treat it as "retry", not "error".

A minimal completion handler:

```c
if (cqe->res == -EAGAIN) {
    /* resubmit the same read/write_fixed request as-is */
} else if (cqe->res < 0) {
    /* real error */
} else {
    /* cqe->res bytes transferred */
}
```

---

## Invalidation: what happens under the hood

The exporter does not give up ownership of its memory. It may need to **move**
the buffer (e.g. migrate GPU VRAM ↔ system memory) or **reclaim** it under
pressure. When it does, it *invalidates* the current mapping:

1. The exporter signals invalidation on the dmabuf.
2. The io_uring token mapping is torn down; a dma_fence tracks in-flight I/O so
   the exporter waits until outstanding DMA against the old mapping drains before
   it actually moves the memory.
3. New requests transparently build a fresh mapping. Requests that raced the
   invalidation surface as `-EAGAIN` — which is why you must reissue.

You don't manage any of this directly; the contract you must honor is simply:
**register once, reissue on `-EAGAIN`, and keep the dmabuf fd and target fd alive
for as long as the buffer is registered.**

---

## Constraints & gotchas

- **One buffer ↔ one file binding.** The token is created for `target_fd` at
  registration. Issuing a request that uses this buffer against a *different*
  file fails. Register the same dmabuf separately per device if you need it on
  more than one.
- **`uaddr`/`size` must be zero** in the descriptor for dmabuf type; the size is
  taken from the dmabuf itself. Non-zero → `-EINVAL`.
- **Offsets, not pointers.** As above, the read/write "address" is an offset in
  `[0, dmabuf->size)`. Out-of-range → `-EFAULT`.
- **1 GiB cap** on dmabuf size.
- **O_DIRECT only**, and only on devices whose driver implements the token op
  (NVMe today). Non-supporting files → `-EOPNOTSUPP`/`-EINVAL` at registration.
- **Lifetime.** Keep `dmabuf_fd` and `target_fd` open while registered. Tearing
  down the ring / unregistering the buffer releases the token and drops the
  device DMA mapping.
- **P2P is an optimization, not a correctness requirement.** If P2P is
  unavailable, the exporter serves the buffer from system-reachable memory and
  everything still works — you simply don't get the direct device-to-device
  path. Whether you actually get P2P depends on `CONFIG_PCI_P2PDMA` *and* the
  physical PCIe topology.

---

## End-to-end skeleton

```c
/* 1. dmabuf fd from exporter (GPU export ioctl, or udmabuf for testing) */
int dmabuf_fd = /* ... */;

/* 2. target device, O_DIRECT */
int dev_fd = open("/dev/nvme0n1", O_DIRECT | O_RDWR);

/* 3. ring + buffer table */
struct io_uring ring;
io_uring_queue_init(64, &ring, 0);
io_uring_register_buffers_sparse(&ring, 1);

/* 4. register dmabuf -> slot 0, bound to dev_fd */
struct io_uring_regbuf_desc rd = {
    .type = IO_REGBUF_TYPE_DMABUF,
    .dmabuf_fd = dmabuf_fd,
    .target_fd = dev_fd,
};
struct io_uring_rsrc_update2 up = {
    .data = (unsigned long)&rd,
    .nr = 1,
    .offset = 0,
    .resv = IORING_RSRC_UPDATE_EXTENDED,
};
io_uring_register(ring.ring_fd, IORING_REGISTER_BUFFERS_UPDATE, &up, sizeof(up));

/* 5. read 4K from device offset 0 into dmabuf offset 0 */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, dev_fd, (void *)0, 4096, 0, /*buf_index*/0);
io_uring_submit(&ring);

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
if (cqe->res == -EAGAIN) {
    /* reissue the same request */
} else if (cqe->res < 0) {
    /* error: -cqe->res */
} else {
    /* cqe->res bytes now in the dmabuf */
}
io_uring_cqe_seen(&ring, cqe);
```

---

## Reference: the interface surface

**Registration** — `io_uring_register(IORING_REGISTER_BUFFERS_UPDATE)` with
`struct io_uring_rsrc_update2{ .resv = IORING_RSRC_UPDATE_EXTENDED }` pointing at
one or more `struct io_uring_regbuf_desc`:

```c
enum io_uring_regbuf_type {
    IO_REGBUF_TYPE_EMPTY,
    IO_REGBUF_TYPE_UADDR,
    IO_REGBUF_TYPE_DMABUF,   /* <- this feature */
};

struct io_uring_regbuf_desc {
    __u32 type;        /* IO_REGBUF_TYPE_DMABUF */
    __u32 flags;
    __u64 size;        /* must be 0 for dmabuf */
    __u64 uaddr;       /* must be 0 for dmabuf */
    __s32 dmabuf_fd;   /* exporter's dmabuf */
    __s32 target_fd;   /* O_DIRECT block device to bind to */
    __u64 __resv[6];   /* must be zero */
};
```

**I/O** — standard `IORING_OP_READ_FIXED` / `IORING_OP_WRITE_FIXED`
(`io_uring_prep_read_fixed` / `io_uring_prep_write_fixed`), issued against
`target_fd`, with the "address" being the offset into the dmabuf and `buf_index`
selecting the registered slot.

**Working example** — `test/rw-dmabuf.c` in liburing exercises registration,
single and parallel `READ_FIXED`, `WRITE_FIXED`, invalid-registration rejection,
and data verification against a udmabuf.
