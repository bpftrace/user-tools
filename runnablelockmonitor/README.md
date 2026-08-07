# runnablelockmonitor

This traces sleeping locks (mutexes, rw_semaphores, and the write side of
percpu_rw_semaphores) that are held while the holder is not running on a CPU,
and splits that off-CPU time into two bands with separate stacks.

When a lock is held for a long time, the interesting question is usually not
"what was the holder computing" but "was the holder even running". A holder can
stop running for two very different reasons:

* **RUNNABLE** -- it is still on the run queue, ready to go, but not executing.
  It lost the CPU to preemption, to CPU contention from other tasks, or to
  `cpu.max` throttling. Nothing is wrong with the critical section; the holder
  simply cannot get scheduled.
* **SLEEPING** -- it left the run queue because it blocked on something: a
  nested lock, RCU, memory reclaim, I/O. Here the critical section really is
  waiting on an event, and the block-site stack says which one.

The distinction matters because the fixes are opposite. A RUNNABLE stall is a
scheduling or capacity problem -- the code is fine, the CPU allocation is not.
A SLEEPING stall is a code problem -- something blocking is being done inside a
critical section. Ordinary lock-contention tooling reports neither; it shows
long waits and leaves you to guess.

`runnablelockmonitor` reports, for each lock that spent time off-CPU while
held: total hold time, the accumulated and largest-episode time in each band,
the stack where the lock was acquired, the stack where the holder was preempted
(RUNNABLE) and the stack where it blocked (SLEEPING), plus whether anyone was
waiting.

## Output

By default a lock is reported when either band exceeds 5000 us. Here four `dd`
processes append to the same file from inside a cgroup confined to 2 CPUs, so
they contend on one inode lock and cannot always get a CPU:

```
# ./runnablelockmonitor.bt
Attached 21 probes
>>> OFFCPU_LOCK: rwsem write 0xffff8901fa7c6a20
    hold=15407 us  [WAITERS]
    runnable=14900 us (max episode 14900 us)
    sleeping=0 us (max episode 0 us)
    pid=1501555  tid=1501555  process=dd thread=dd
    allowed_cpus=2 of 316 system cpus
    acquire stack:

        down_write+5
        btrfs_inode_lock+36
        btrfs_buffered_write+108
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    largest runnable-episode (preempt) stack:

        __traceiter_sched_switch+87
        __traceiter_sched_switch+87
        __schedule+2554
        __cond_resched+39
        btrfs_buffered_write+1217
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108
```

Reading that report line by line:

* `rwsem write 0xffff8901fa7c6a20` -- an `rw_semaphore` taken for write, at
  that kernel address. Repeated reports at one address mean a single object is
  the bottleneck; a changing address means many different locks.
* `hold=15407 us` -- the lock was held for 15.4 ms end to end.
* `[WAITERS]` -- at least one other task was blocked on this lock when it was
  released. Without this tag the lock delayed nobody.
* `runnable=14900 us (max episode 14900 us)` -- 14.9 ms of the 15.4 ms hold was
  spent on the run queue *not executing*. The two numbers being equal means it
  was one single 14.9 ms episode rather than many small ones.
* `sleeping=0 us` -- the holder never blocked. It was ready to run the whole
  time.
* `allowed_cpus=2 of 316 system cpus` -- the holder was restricted to 2 CPUs
  out of 316. This is the key context: a large RUNNABLE band with a small
  `allowed_cpus` is affinity- or cgroup-limited scheduling, not system-wide CPU
  starvation.
* `acquire stack` -- where the lock was taken: the inode lock of the file being
  written.
* `largest runnable-episode (preempt) stack` -- where the holder was preempted.
  `__cond_resched` inside `btrfs_buffered_write`: the write path voluntarily
  offered the CPU up mid-write, and with only 2 CPUs for four writers plus
  other load it did not get one back for 14.9 ms.

So: the holder was not slow and was not blocked. It gave up the CPU inside a
critical section and then could not get scheduled again, while another writer
sat waiting on the inode lock for essentially the entire time. No amount of
filesystem or lock tuning helps here -- the cgroup needs more CPU.

### Both bands at once

When a lock accumulates time in both bands, each gets its own stack, and the
two usually tell different stories:

```
>>> OFFCPU_LOCK: rwsem write 0xffff8901fa7c6a20
    hold=56639324 us  [WAITERS]
    runnable=18700151 us (max episode 15282 us)
    sleeping=31688233 us (max episode 208982 us)
    pid=1501553  tid=1501553  process=dd thread=dd
    allowed_cpus=2 of 316 system cpus
    acquire stack:

        down_write+5
        btrfs_inode_lock+36
        btrfs_buffered_write+108
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    largest runnable-episode (preempt) stack:

        __traceiter_sched_switch+87
        __traceiter_sched_switch+87
        __schedule+2554
        __cond_resched+39
        shrink_node+741
        do_try_to_free_pages+197
        try_to_free_mem_cgroup_pages+331
        __mem_cgroup_charge+1873
        filemap_add_folio+144
        __filemap_get_folio+745
        prepare_one_folio+76
        btrfs_buffered_write+637
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    largest sleep-episode (block-site) stack:

        __traceiter_sched_switch+87
        __traceiter_sched_switch+87
        __schedule+2554
        schedule+67
        schedule_timeout+121
        io_schedule_timeout+72
        balance_dirty_pages_ratelimited_flags+2229
        btrfs_buffered_write+605
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108
```

This holder kept the inode lock for 56 seconds: 18.7 s runnable-but-not-running
spread over many short episodes (largest 15 ms), and 31.7 s asleep in a few long
ones (largest 209 ms). The two stacks name two independent problems in the same
critical section. The RUNNABLE stack shows it was preempted inside memcg reclaim
(`shrink_node` under `try_to_free_mem_cgroup_pages`), so the write hit the
cgroup memory limit. The SLEEPING stack shows it also blocked in
`balance_dirty_pages_ratelimited_flags` -> `io_schedule_timeout`, dirty-page
throttling waiting on writeback. Meanwhile `[WAITERS]` means other writers were
queued behind the inode lock through all of it.

### Throttled holders

A RUNNABLE episode that began while the holder's CFS hierarchy was throttled by
`cpu.max` is tagged `[THROTTLED]` and its episode stack is labeled `throttle`
instead of `preempt`:

```
>>> OFFCPU_LOCK: rwsem write 0xffff8905a5820678
    hold=101750 us  [WAITERS]  [THROTTLED]
    runnable=97865 us (max episode 97865 us)
    sleeping=0 us (max episode 0 us)
    pid=1533956  tid=1533956  process=bash thread=bash
    allowed_cpus=2 of 316 system cpus
    acquire stack:

        down_write_killable+5
        copy_process+5203
        kernel_clone+137
        __x64_sys_clone+197
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    largest runnable-episode (throttle) stack:

        __traceiter_sched_switch+87
        __traceiter_sched_switch+87
        __schedule+2554
        __cond_resched+39
        copy_page_range+2503
        copy_process+6114
        kernel_clone+137
        __x64_sys_clone+197
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108
```

`bash` forked, took `mmap_lock` for write in `copy_process`, and ran out of
`cpu.max` quota partway through `copy_page_range`. It was thrown off the CPU
for 97 ms still holding the lock, with another thread waiting. This is the
worst version of the RUNNABLE case: throttling is self-inflicted by
configuration, and it stopped a task in the middle of a critical section.

Note that this tag only appears on kernels that throttle inline. Kernels that
defer `cpu.max` throttling to exit-to-userspace (they have
`throttle_cfs_rq_work`) never throttle a holder mid-critical-section, so the
tool detects that symbol and suppresses the tag rather than reporting something
that cannot happen.

A report can also carry `[INFLIGHT]`, meaning the lock was still held when the
script stopped and so was reported by the `END` action. For those, all the time
figures are lower bounds -- they were still growing when tracing ended.

## Reproducing the examples

The first two came from four writers appending to one file inside a cgroup
confined to 2 CPUs and a small memory limit, with spinners added to make those
CPUs genuinely scarce:

```
mkdir /sys/fs/cgroup/rlmtest
echo "0-1" > /sys/fs/cgroup/rlmtest/cpuset.cpus
echo 268435456 > /sys/fs/cgroup/rlmtest/memory.max
echo 0 > /sys/fs/cgroup/rlmtest/memory.swap.max

bash -c 'echo $$ > /sys/fs/cgroup/rlmtest/cgroup.procs
         for i in 1 2 3 4; do
             dd if=/dev/zero of=/tmp/shared.dat bs=1M count=800 \
                conv=notrunc oflag=append &
         done
         for i in 1 2 3 4 5 6; do (while :; do :; done) & done
         sleep 24; kill $(jobs -p)'
```

The throttled example added a quota to the same cgroup:

```
echo "10000 100000" > /sys/fs/cgroup/rlmtest/cpu.max   # 10% of one CPU
```

Start the tool first and let it finish attaching, then run the workload.

## Notes and caveats

* **Overhead is significant.** `sched_switch` fires constantly, every
  `mutex_lock`/`down_read`/`down_write` in the system is probed, and a kernel
  stack is captured on each acquisition. Use short run windows and do not leave
  this running on a busy machine.
* **The default thresholds are 5000 us for each band**, and a lock is reported
  if *either* band crosses its own threshold. Lower them with
  `--min_runnable_us` / `--min_sleep_us` to see shorter episodes, at the cost of
  a lot more output.
* **`allowed_cpus` is captured at acquisition time** and is the size of the
  holder's CPU affinity mask. Compare it against the system CPU count in the
  same line to tell affinity-limited scheduling from global CPU contention.
* **Only the write side of percpu_rw_semaphore is traced.**
  `percpu_down_read()`/`percpu_up_read()` are inlined per-CPU counter
  increments with no symbol to attach to. An exclusive writer that loses the
  CPU while holding the lock blocks all readers and writers.
* **Up to 8 held locks are tracked per task.** Deeper nesting is ignored.
  Unlocks out of acquisition order are handled.
* A lock reported without `[WAITERS]` delayed nobody at the moment it was
  released. Use `--waiters_only=true` to see only the ones that did.
* Requires `CONFIG_DEBUG_INFO_BTF=y`.

## USAGE

```
USAGE:
  ./runnablelockmonitor.bt                          # both bands >= 5000 us
  ./runnablelockmonitor.bt -- --waiters_only=true   # contended locks only
  ./runnablelockmonitor.bt -- --min_runnable_us=500 # runnable >= 500 us
  ./runnablelockmonitor.bt -- --min_sleep_us=500    # sleeping >= 500 us
A lock is reported if EITHER band crosses its threshold.
```
