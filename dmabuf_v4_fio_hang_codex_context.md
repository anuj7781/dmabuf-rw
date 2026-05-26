# dma-buf read/write fio hang - Codex debugging context

## Purpose

This document captures the fio command and the dmesg hung-task output from Alok's run. The goal is to feed this context to Codex so it can inspect Pavel's `rw-dmabuf-v4` kernel tree and diagnose the possible deadlock / lock-order issue.

## Kernel tree under test

Repository / branch:

```text
https://github.com/isilence/linux/commits/rw-dmabuf-v4/
```

Kernel shown in dmesg:

```text
7.0.0v4+ #42
```

## fio command

Transcribed from screenshot. Please verify the command line from shell history if exact reproduction is required.

```bash
taskset -c 5 fio \
  --filename=/dev/nvme0n1 \
  --direct=1 \
  --size=4G \
  --rw=randread \
  --numjobs=1 \
  --bs=4K \
  --ioengine=io_uring \
  --sqthread_poll=0 \
  --iodepth=128 \
  --name=uring_dmabuf \
  --gpu_dmabuf=1 \
  --registerfiles=1 \
  --group_reporting \
  --thread
```

Single-line form:

```bash
taskset -c 5 fio --filename=/dev/nvme0n1 --direct=1 --size=4G --rw=randread --numjobs=1 --bs=4K --ioengine=io_uring --sqthread_poll=0 --iodepth=128 --name=uring_dmabuf --gpu_dmabuf=1 --registerfiles=1 --group_reporting --thread
```

## dmesg / hung task output

```text
[Alok Rathore (alok)] 05-25-2026 17:41

[ 3564.670482]        Not tainted 7.0.0v4+ #42
[ 3564.670501] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[ 3564.670523] task:kworker/u80:2    state:D stack:0     pid:21390 tgid:21390 ppid:2      task_flags:0x4208060 flags:0x00080000
[ 3564.670538] Workqueue: kfd_restore_wq restore_process_worker [amdgpu]
[ 3564.671774] Call Trace:
[ 3564.671779]  <TASK>
[ 3564.671786]  __schedule+0x462/0x19e0
[ 3564.671798]  ? sysvec_apic_timer_interrupt+0x57/0xc0
[ 3564.671808]  schedule+0x27/0xf0
[ 3564.671814]  schedule_preempt_disabled+0x15/0x30
[ 3564.671820]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[ 3564.671830]  __ww_mutex_lock_slowpath+0x16/0x30
[ 3564.671836]  ww_mutex_lock+0xeb/0x100
[ 3564.671847]  drm_exec_lock_obj+0x46/0x210 [drm_exec]
[ 3564.671854]  drm_exec_prepare_obj+0x20/0x60 [drm_exec]
[ 3564.671861]  amdgpu_amdkfd_gpuvm_restore_process_bos+0x106/0x9c0 [amdgpu]
[ 3564.673082]  ? dequeue_entities+0x198/0x1030
[ 3564.673094]  ? psi_group_change+0x201/0x4e0
[ 3564.673107]  restore_process_helper+0x28/0xb0 [amdgpu]
[ 3564.674048]  restore_process_worker+0x2b/0xf0 [amdgpu]
[ 3564.674912]  process_one_work+0x19c/0x420
[ 3564.674921]  worker_thread+0x1a8/0x330
[ 3564.674929]  ? __pfx_worker_thread+0x10/0x10
[ 3564.674936]  kthread+0xfb/0x140
[ 3564.674941]  ? __pfx_kthread+0x10/0x10
[ 3564.674946]  ret_from_fork+0x2a3/0x360
[ 3564.674954]  ? __pfx_kthread+0x10/0x10
[ 3564.674958]  ret_from_fork_asm+0x1a/0x30
[ 3564.674965]  </TASK>

[ 3564.675037] INFO: task iou-wrk-22173:22202 blocked for more than 122 seconds.
[ 3564.675078]        Not tainted 7.0.0v4+ #42
[ 3564.675094] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[ 3564.675117] task:iou-wrk-22173   state:D stack:0     pid:22202 tgid:22170 ppid:2186     task_flags:0x404050 flags:0x00080000
[ 3564.675127] Call Trace:
[ 3564.675130]  <TASK>
[ 3564.675134]  __schedule+0x462/0x19e0
[ 3564.675143]  schedule+0x27/0xf0
[ 3564.675149]  schedule_preempt_disabled+0x15/0x30
[ 3564.675154]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[ 3564.675168]  ? dma_fence_default_wait+0x1c7/0x280
[ 3564.675181]  __ww_mutex_lock_slowpath+0x16/0x30
[ 3564.675189]  ww_mutex_lock+0xeb/0x100
[ 3564.675197]  io_dmabuf_create_map+0x4b/0x1a0
[ 3564.675208]  io_import_reg_buf+0x3da/0x440
[ 3564.675218]  io_read_fixed+0x64/0xb0
[ 3564.675224]  io_issue_sqe+0x43/0x1b0
[ 3564.675232]  io_issue_sqe+0x3f/0x550
[ 3564.675239]  io_wq_submit_work+0x106/0x370
[ 3564.675245]  io_worker_handle_work+0x141/0x550
[ 3564.675251]  ? __x64_sys_execve+0x20/0x70
[ 3564.675258]  io_wq_worker+0xfd/0x3a0
[ 3564.675263]  ? hrtick_cond_restart+0x62/0x80
[ 3564.675271]  ? finish_task_switch.isra.0+0xcd/0x3d0
[ 3564.675279]  ? __pfx_io_wq_worker+0x10/0x10
[ 3564.675284]  ret_from_fork+0x2a3/0x360
[ 3564.675290]  ? __pfx_io_wq_worker+0x10/0x10
[ 3564.675295]  ret_from_fork_asm+0x1a/0x30
[ 3564.675299]  RIP: 0033:0x0
[ 3564.675305]  RSP: 002b:0000000000000000 EFLAGS: 00000246 ORIG_RAX: 00000000000001aa
[ 3564.675312]  RAX: 0000000000000000 RBX: 0000000000000000 RCX: 00006017a93ba6b6
[ 3564.675316]  RDX: 0000000000000001 RSI: 0000000000000000 RDI: 0000000000000006
[ 3564.675319]  RBP: 0000000000000001 R08: 0000000000000000 R09: 0000000000000000
[ 3564.675321]  R10: 0000000000000001 R11: 0000000000000246
```

## Immediate observation from the traces

The fio/io_uring worker is blocked while importing a registered dmabuf for fixed read I/O:

```text
io_read_fixed
  io_import_reg_buf
    io_dmabuf_create_map
      ww_mutex_lock
```

At the same time, an AMDGPU KFD restore worker is blocked while restoring process BOs:

```text
kfd_restore_wq restore_process_worker [amdgpu]
  amdgpu_amdkfd_gpuvm_restore_process_bos
    drm_exec_prepare_obj
      drm_exec_lock_obj
        ww_mutex_lock
```

This suggests the issue may be around dma-buf / dma-resv / ww_mutex ordering between the v4 dmabuf mapping path and AMDGPU KFD BO restore path.

## Suggested Codex task

Please inspect Pavel's `rw-dmabuf-v4` branch and diagnose the hang using the traces above.

Focus on whether there is a lock-order cycle or invalidation/fence wait cycle involving:

- `io_dmabuf_create_map()`
- `io_import_reg_buf()`
- `io_read_fixed()`
- dma-buf reservation lock / `dma_resv_lock()` / `dma_resv_wait_timeout()`
- `dma_fence_default_wait()`
- v4 dmabuf token map/invalidate/release code
- NVMe dmabuf map/token implementation
- AMDGPU dma-buf exporter map path
- AMDGPU KFD restore path, especially `amdgpu_amdkfd_gpuvm_restore_process_bos()` and `drm_exec_prepare_obj()`

Questions to answer:

1. Is `io_dmabuf_create_map()` taking a dma-buf/BO reservation lock and then calling into a path that can take other AMDGPU BO reservation locks?
2. Can KFD restore hold or wait for BO locks in a conflicting order with `io_dmabuf_create_map()`?
3. Is `dma_resv_wait_timeout()` being called while holding a lock that prevents the waited fence from being signaled?
4. Is the v4 invalidation path adding a fence that waits for outstanding request refs, while the worker needed to drop those refs is blocked behind AMDGPU BO restore?
5. Does the `invalidate_mappings` / revoke handling in v4 need to be adjusted for AMDGPU dynamic dma-buf attachments?
6. Is there a missing `dma_resv_unlock()`, missing fence signal, reference leak, or an incorrect usage class (`DMA_RESV_USAGE_KERNEL` vs `DMA_RESV_USAGE_BOOKKEEP`) in the v4 token code?
7. What minimal instrumentation would prove the deadlock cycle?
8. What minimal patch would avoid the deadlock while keeping the v4 design intact?
```

## Reproduction notes

- Workload: single fio job, 4K random read, `iodepth=128`, direct I/O, io_uring fixed read, GPU dma-buf enabled.
- Target block device: `/dev/nvme0n1`.
- CPU pinning: `taskset -c 5`.
- dmesg shows two blocked contexts: `kfd_restore_wq` and `iou-wrk-*`.
