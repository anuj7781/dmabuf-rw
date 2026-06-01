# dmabuf read/write v4: post-fix hung-task / deadlock context for Codex

## Goal for Codex

Please inspect Pavel's `rw-dmabuf-v4` kernel tree plus my local fix and diagnose the remaining hang/deadlock seen while running fio with `gpu_dmabuf=1`.

The issue still reproduces after my attempted fix. The dmesg below is a partial screenshot transcription, but it captures the important blocked tasks and call traces.

Kernel branch under test:

```text
https://github.com/isilence/linux/commits/rw-dmabuf-v4/
```

Kernel shown in dmesg:

```text
Not tainted 7.0.0fix-v1+ #46
```

## fio command

```bash
taskset -c 5 fio --filename=/dev/nvme0n1 --direct=1 --size=4G --rw=randread \
  --numjobs=1 --bs=4K --ioengine=io_uring --sqthread_poll=0 --iodepth=128 \
  --name=uring_dmabuf --gpu_dmabuf=1 --registerfiles=1 \
  --group_reporting --thread
```

## High-level symptom

Multiple tasks are blocked for more than 122 seconds. The important actors are:

1. `io_dmabuf_map_release_work` workers stuck in `ww_mutex_lock()`.
2. `kfd_restore_wq restore_process_worker [amdgpu]` stuck while waiting on a DMA fence inside AMDGPU VM restore/update path.
3. fio `iou-wrk-*` workers stuck in `io_dmabuf_create_map()` while importing the registered dmabuf for `io_read_fixed()`.
4. `ttm_bo_delayed_delete [ttm]` also blocked waiting on dma-resv/dma-fence.

This strongly suggests a dma-buf reservation object / ww_mutex / fence ordering issue involving:

```text
io_dmabuf_create_map()
io_dmabuf_map_release_work()
AMDGPU KFD restore_process_worker
TTM delayed BO delete
dma_resv / dma_fence / ww_mutex locking
```

## Partial dmesg transcription from latest hang

### 1. `io_dmabuf_map_release_work` blocked on ww_mutex

```text
[  246.910040] INFO: task kworker/1:1:183 blocked for more than 122 seconds.
[  246.910092]       Not tainted 7.0.0fix-v1+ #46
[  246.910111] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[  246.910133] task:kworker/1:1     state:D stack:0     pid:183  tgid:183  ppid:2  task_flags:0x4208060 flags:0x00080000
[  246.910148] Workqueue: events io_dmabuf_map_release_work
[  246.910164] Call Trace:
[  246.910172]  <TASK>
[  246.910180]  __schedule+0x462/0x19e0
[  246.910193]  schedule+0x27/0xf0
[  246.910199]  schedule_preempt_disabled+0x15/0x30
[  246.910204]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[  246.910212]  ? sysvec_reschedule_ipi+0x67/0x120
[  246.910220]  ? preempt_schedule+0x59/0x70
[  246.910223]  __ww_mutex_lock_slowpath+0x16/0x30
[  246.910238]  ww_mutex_lock+0xeb/0x100
[  246.910240]  io_dmabuf_map_release_work+0x5c/0x1b0
[  246.910249]  process_one_work+0x19c/0x420
[  246.910260]  worker_thread+0x1a8/0x330
[  246.910269]  ? __pfx_worker_thread+0x10/0x10
[  246.910277]  kthread+0xfb/0x140
[  246.910283]  ? __pfx_kthread+0x10/0x10
[  246.910288]  ret_from_fork+0x2a3/0x360
[  246.910293]  ? __pfx_kthread+0x10/0x10
[  246.910303]  ret_from_fork_asm+0x1a/0x30
[  246.910317]  </TASK>
```

The kernel reports this worker is blocked on a mutex likely owned by the AMDGPU KFD restore worker:

```text
[  246.910331] INFO: task kworker/1:1:183 is blocked on a mutex likely owned by task kworker/u80:0:12.
[  246.910344] task:kworker/u80:0 state:S stack:5 pid:12 tgid:12 ppid:2 task_flags:0x4208860 flags:0x00080000
[  246.910356] Workqueue: kfd_restore_wq restore_process_worker [amdgpu]
```

### 2. AMDGPU KFD restore worker waiting on DMA fence

```text
[  246.911615] Call Trace:
[  246.911621]  <TASK>
[  246.911627]  __schedule+0x462/0x19e0
[  246.911638]  ? wakeup_preempt_fair+0x1f3/0x3d0
[  246.911641]  ? wake_up_state+0x96/0xa0
[  246.911644]  schedule+0x27/0xf0
[  246.911649]  schedule_timeout+0x104/0x110
[  246.911660]  dma_fence_default_wait+0x169/0x280
[  246.911667]  ? wake_up_process+0x15/0x30
[  246.911679]  ? __pfx_dma_fence_default_wait_cb+0x10/0x10
[  246.911688]  dma_fence_wait_timeout+0x64/0x1c0
[  246.911696]  amdgpu_sync_wait+0x68/0xb0 [amdgpu]
[  246.912547]  amdgpu_vm_cpu_prepare+0x1b/0x30 [amdgpu]
[  246.913465]  amdgpu_vm_update_range+0x1e0/0x940 [amdgpu]
[  246.914275]  amdgpu_vm_bo_update+0x327/0x7b0 [amdgpu]
[  246.915079]  update_gpuvm_ptes+0x176/0x4c0 [amdgpu]
[  246.916023]  ? amdgpu_gmc_get_pde_for_bo+0x80/0xc0 [amdgpu]
[  246.916397]  amdgpu_amdkfd_gpuvm_restore_process_bos+0x435/0x9c0 [amdgpu]
[  246.917650]  restore_process_helper+0x28/0xb0 [amdgpu]
[  246.917931]  restore_process_worker+0x2b/0xf0 [amdgpu]
[  246.917871]  process_one_work+0x19c/0x420
[  246.917880]  worker_thread+0x1a8/0x330
[  246.917888]  ? __pfx_worker_thread+0x10/0x10
[  246.917894]  kthread+0xfb/0x140
[  246.917899]  ? __pfx_kthread+0x10/0x10
[  246.917904]  ret_from_fork+0x2a3/0x360
[  246.917908]  ? __pfx_kthread+0x10/0x10
[  246.917914]  ret_from_fork_asm+0x1a/0x30
[  246.917929]  </TASK>
```

### 3. TTM delayed BO delete also waiting on dma_resv / dma_fence

```text
[  246.917971] INFO: task kworker/u86:1:1482 blocked for more than 122 seconds.
[  246.917984]       Not tainted 7.0.0fix-v1+ #46
[  246.917997] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[  246.918004] task:kworker/u86:1 state:D stack:0 pid:1482 tgid:1482 ppid:2 task_flags:0x4208060 flags:0x00080000
[  246.918016] Workqueue: ttm ttm_bo_delayed_delete [ttm]
[  246.918023] Call Trace:
[  246.918027]  <TASK>
[  246.918031]  __schedule+0x462/0x19e0
[  246.918037]  schedule+0x27/0xf0
[  246.918043]  schedule_timeout+0x104/0x110
[  246.918051]  dma_fence_default_wait+0x169/0x280
[  246.918062]  ? __pfx_dma_fence_default_wait_cb+0x10/0x10
[  246.918069]  dma_fence_wait_timeout+0x64/0x1c0
[  246.918076]  dma_resv_wait_timeout+0x85/0x110
[  246.918083]  ttm_bo_delayed_delete+0x35/0xc0 [ttm]
[  246.918091]  process_one_work+0x19c/0x420
[  246.918097]  worker_thread+0x1a8/0x330
[  246.918103]  ? __pfx_worker_thread+0x10/0x10
[  246.918114]  kthread+0xfb/0x140
[  246.918120]  ? __pfx_kthread+0x10/0x10
[  246.918133]  ret_from_fork+0x2a3/0x360
[  246.918148]  ? __pfx_kthread+0x10/0x10
[  246.918149]  ret_from_fork_asm+0x1a/0x30
[  246.918154]  </TASK>
```

### 4. Another `io_dmabuf_map_release_work` worker blocked on same pattern

```text
[  246.918061] INFO: task kworker/12:3:1639 blocked for more than 122 seconds.
[  246.918062]       Not tainted 7.0.0fix-v1+ #46
[  246.918079] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[  246.918084] task:kworker/12:3 state:D stack:0 pid:1639 tgid:1639 ppid:2 task_flags:0x4208060 flags:0x00080000
[  246.918089] Workqueue: events io_dmabuf_map_release_work
[  246.918090] Call Trace:
[  246.918091]  <TASK>
[  246.918094]  __schedule+0x462/0x19e0
[  246.918096]  schedule+0x27/0xf0
[  246.918098]  ? kick_pool+0x8c/0x1b0
[  246.918101]  schedule_preempt_disabled+0x15/0x30
[  246.918107]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[  246.918110]  ? preempt_schedule+0x59/0x70
[  246.918117]  __ww_mutex_lock_slowpath+0x16/0x30
[  246.918121]  ww_mutex_lock+0xeb/0x100
[  246.918124]  io_dmabuf_map_release_work+0x5c/0x1b0
[  246.918128]  process_one_work+0x19c/0x420
[  246.918132]  worker_thread+0x1a8/0x330
[  246.918136]  kthread+0xfb/0x140
[  246.918140]  ret_from_fork+0x2a3/0x360
[  246.918149]  ret_from_fork_asm+0x1a/0x30
[  246.918149]  </TASK>
```

Again, the kernel says it is blocked on a mutex likely owned by the AMDGPU KFD restore worker:

```text
[  246.918152] INFO: task kworker/12:3:1639 is blocked on a mutex likely owned by task kworker/u80:0:12.
[  246.918163] task:kworker/u80:0 state:S stack:0 pid:12 tgid:12 ppid:2 task_flags:0x4208860 flags:0x00080000
[  246.918657] Workqueue: kfd_restore_wq restore_process_worker [amdgpu]
```

### 5. fio io-wq worker blocked inside `io_dmabuf_create_map()`

```text
[  246.921373] INFO: task iou-wrk-2330:2378 blocked for more than 122 seconds.
[  246.921389]       Not tainted 7.0.0fix-v1+ #46
[  246.921392] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[  246.921398] task:iou-wrk-2330 state:D stack:0 pid:2378 tgid:2294 ppid:2227 task_flags:0x404050 flags:0x00080000
[  246.921399] Call Trace:
[  246.921399]  <TASK>
[  246.921398]  __schedule+0x462/0x19e0
[  246.921402]  ? wake_up_state+0x10/0x20
[  246.921403]  ? task_work_add+0xd7/0x110
[  246.921405]  ? io_queue_worker_create+0xb5/0x150
[  246.921405]  schedule+0x27/0xf0
[  246.921406]  schedule_preempt_disabled+0x15/0x30
[  246.921407]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[  246.921409]  ? dma_fence_default_wait+0x1c7/0x280
[  246.921410]  __ww_mutex_lock_slowpath+0x16/0x30
[  246.921407]  ww_mutex_lock+0xeb/0x100
[  246.921413]  io_dmabuf_create_map+0x4e/0x180
[  246.921418]  io_import_reg_buf+0x36d/0x3e0
[  246.921419]  io_read_fixed+0x64/0xb0
[  246.921417]  io_issue_sqe+0x43/0x1b0
[  246.921418]  io_issue_sqe+0x3f/0x550
[  246.921419]  ? lock_timer_base+0x81/0xc0
[  246.921420]  io_wq_submit_work+0x106/0x370
[  246.921423]  io_worker_handle_work+0x141/0x550
[  246.921424]  ? __pfx_process_timeout+0x10/0x10
[  246.921425]  io_wq_worker+0xfd/0x3a0
[  246.921427]  ? finish_task_switch.isra.0+0xcd/0x3d0
[  246.921429]  ? __pfx_io_wq_worker+0x10/0x10
[  246.921430]  ret_from_fork+0x2a3/0x360
[  246.921431]  ? __pfx_io_wq_worker+0x10/0x10
[  246.921433]  ret_from_fork_asm+0x1a/0x30
[  246.921435]  </TASK>
```

### 6. Another fio io-wq worker blocked in same path

```text
[  246.923122] INFO: task iou-wrk-2345:2379 blocked for more than 122 seconds.
[  246.923126]       Not tainted 7.0.0fix-v1+ #46
[  246.923129] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[  246.923133] task:iou-wrk-2345 state:D stack:0 pid:2379 tgid:2294 ppid:2227 task_flags:0x404050 flags:0x00080000
[  246.923136] Call Trace:
[  246.923135]  <TASK>
[  246.923136]  __schedule+0x462/0x19e0
[  246.923138]  ? sysvec_call_function_single+0x57/0xc0
[  246.923138]  schedule+0x27/0xf0
[  246.923139]  schedule_preempt_disabled+0x15/0x30
[  246.923139]  __ww_mutex_lock.constprop.0+0x7f2/0xe90
[  246.923141]  ? dma_fence_default_wait+0x1c7/0x280
[  246.923142]  __ww_mutex_lock_slowpath+0x16/0x30
[  246.923143]  ww_mutex_lock+0xeb/0x100
[  246.923144]  io_dmabuf_create_map+0x4e/0x180
[  246.923146]  io_import_reg_buf+0x36d/0x3e0
[  246.923147]  io_read_fixed+0x64/0xb0
[  246.923148]  io_issue_sqe+0x43/0x1b0
[  246.923149]  io_issue_sqe+0x3f/0x550
[  246.923150]  ? lock_timer_base+0x81/0xc0
[  246.923152]  io_wq_submit_work+0x106/0x370
[  246.923153]  io_worker_handle_work+0x141/0x550
[  246.923154]  io_wq_worker+0xfd/0x3a0
[  246.923155]  ? finish_task_switch.isra.0+0xcd/0x3d0
[  246.923158]  ? __pfx_io_wq_worker+0x10/0x10
[  246.923159]  ret_from_fork+0x2a3/0x360
[  246.923160]  ? __pfx_io_wq_worker+0x10/0x10
[  246.923161]  ret_from_fork_asm+0x1a/0x30
[  246.923162]  </TASK>
```

## Why this looks suspicious

The post-fix hang is not just an fio worker stuck in `io_dmabuf_create_map()`. Now the release worker is also blocked:

```text
io_dmabuf_map_release_work
  -> ww_mutex_lock
```

At the same time, AMDGPU KFD restore is waiting on a fence:

```text
restore_process_worker [amdgpu]
  -> amdgpu_sync_wait
  -> dma_fence_wait_timeout
```

And TTM delayed delete is waiting on dma-resv fences:

```text
ttm_bo_delayed_delete
  -> dma_resv_wait_timeout
```

The likely cycle to investigate:

```text
A. io_dmabuf_map_release_work wants dmabuf->resv / BO reservation lock
B. kfd_restore_wq appears to own or hold a related ww_mutex/reservation lock
C. kfd_restore_wq waits on a dma_fence
D. that fence may be the io_dmabuf map invalidation/release fence
E. fio io-wq workers try to create/recreate a map and block on the same ww_mutex/resv path
```

## Code paths to inspect

Please inspect these functions and their exact locking/fence order:

```text
lib/io_dmabuf_token.c:
  io_dmabuf_create_map()
  io_dmabuf_map_release_work()
  io_dmabuf_drop_map()
  io_dmabuf_token_invalidate_mappings()
  io_dmabuf_token_release_work()

drivers/nvme/host/pci.c:
  nvme_dmabuf_token_map()
  nvme_dmabuf_token_unmap()
  nvme_dmabuf_invalidate_mappings()
  nvme_init_dmabuf_token()

AMDGPU / KFD paths:
  amdgpu_amdkfd_gpuvm_restore_process_bos()
  restore_process_worker()
  amdgpu_vm_cpu_prepare()
  amdgpu_vm_update_range()
  amdgpu_sync_wait()
  drm_exec_prepare_obj()
  drm_exec_lock_obj()

TTM path:
  ttm_bo_delayed_delete()
```

## Specific questions for Codex

1. Does `io_dmabuf_map_release_work()` take `dmabuf->resv` and then call the driver `unmap()` while the reservation lock is held?
2. Can the NVMe/DMABUF map release fence be waited on by AMDGPU KFD restore while KFD is still holding the BO reservation lock that `io_dmabuf_map_release_work()` needs?
3. Is there an ABBA lock ordering issue between:

```text
dmabuf->resv / BO ww_mutex
DMA fence wait
io_dmabuf_map refs/fence release
AMDGPU KFD restore locks
```

4. Should `io_dmabuf_map_release_work()` signal the fence before attempting operations that may require the exporter BO reservation lock, or would that violate DMA safety?
5. Should unmap be split into:

```text
stop accepting new users / kill map refs
wait for active IO refs to drain
signal dmabuf fence
then perform heavy exporter/driver unmap cleanup without blocking fence waiters
```

6. Is the release worker using the same workqueue as something that can be blocked by the fence wait path?
7. Are we using `dma_resv_wait_timeout()` with the right usage class: `DMA_RESV_USAGE_KERNEL` vs `DMA_RESV_USAGE_BOOKKEEP`?
8. Is it wrong to hold `dmabuf->resv` while calling into `token->dev_ops->unmap()` or while doing `dma_buf_unmap_attachment*()` for AMDGPU exported buffers?

## Useful instrumentation to add

Add logs with task name/pid and timestamps around:

```c
io_dmabuf_create_map():
  before/after dma_resv_lock()
  before/after dma_resv_wait_timeout()
  before/after token->dev_ops->map()
  before rcu_assign_pointer(token->map, map)

io_dmabuf_drop_map():
  before/after dma_resv_reserve_fences()
  before/after dma_resv_add_fence()
  before/after percpu_ref_kill()

io_dmabuf_map_release_work():
  entry
  before/after dma_resv_lock()
  before/after token->dev_ops->unmap()
  before/after dma_fence_signal()

nvme_dmabuf_invalidate_mappings():
  entry + token pointer + current map pointer

AMDGPU exporter callbacks if reachable:
  map_attachment
  unmap_attachment
  invalidate_mappings / move_notify
```

Also dump the fence context/seqno for the dmabuf token fence so we can see if AMDGPU/TTM is waiting on exactly the fence created by `io_dmabuf_drop_map()`.

## Current hypothesis

The remaining bug after my fix is likely not in NVMe I/O completion. It looks like a lock/fence ordering bug in the dynamic dma-buf mapping invalidation/release path:

```text
io_dmabuf_map_release_work() cannot complete because it is blocked on BO/dmabuf reservation lock;
AMDGPU KFD restore or TTM is waiting on a dma-resv fence that requires io_dmabuf_map_release_work() to finish/signals;
fio io-wq workers then pile up trying to create a new dmabuf map.
```

