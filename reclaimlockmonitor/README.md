# reclaimlockmonitor

This traces sleeping locks (mutexes, rw_semaphores, and the write side of
percpu_rw_semaphores) that are held while the holder is pushed into memory
reclaim.

A task can take a lock and then, still holding it, allocate memory. If that
allocation cannot be satisfied immediately, the kernel enters direct reclaim
(global) or memcg reclaim (cgroup limit) inline in the allocation path. Reclaim
can run for hundreds of milliseconds. For that whole time the lock stays held,
and every other task that wants it is blocked -- not because the lock is
genuinely hot, but because the holder wandered into the reclaim path.

This is awkward to diagnose with ordinary lock tooling. A contention profiler
shows you a lock with long waits and a list of waiters, but not *why* the
holder was slow, so the problem reads as lock contention when it is really
memory pressure leaking into a critical section. `reclaimlockmonitor` connects
the two ends: for each lock held across a reclaim episode it reports the total
hold time, how much of that hold was spent in reclaim, whether anyone was
waiting when the lock was released, where the lock was acquired, and where
reclaim was entered.

## Output

Running with no arguments reports every lock that was held across a reclaim
episode. Here a single `dd` is writing to a file inside a cgroup whose
`memory.max` is smaller than the amount of page cache the write dirties:

```
# ./reclaimlockmonitor.bt
Attached 23 probes
>>> RECLAIM_LOCK: rwsem write 0xffff888889adaa20
    hold=2549 us  reclaim=2072 us  reclaim_type=memcg
    pid=441946   tid=441946   process=dd thread=dd
    acquire stack:

        down_write+5
        btrfs_inode_lock+36
        btrfs_buffered_write+108
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    reclaim-entry stack (longest episode):

        try_to_free_mem_cgroup_pages+510
        try_to_free_mem_cgroup_pages+510
        __mem_cgroup_charge+1873
        filemap_add_folio+144
        __filemap_get_folio+745
        prepare_one_folio+76
        btrfs_buffered_write+637
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108
```

Reading that report line by line:

* `rwsem write 0xffff888889adaa20` -- the lock was an `rw_semaphore` taken for
  write, at kernel address `0xffff888889adaa20`. The address is what lets you
  tell "the same lock over and over" from "many different locks"; repeated
  reports at one address mean one object is the bottleneck.
* `hold=2549 us` -- the lock was held for 2549 microseconds in total, measured
  from the acquiring call returning to the matching unlock.
* `reclaim=2072 us` -- of those 2549 us, 2072 us (81%) were spent inside memory
  reclaim. That is the number that matters: the critical section itself was
  short, and reclaim is what made it long.
* `reclaim_type=memcg` -- the reclaim was cgroup-limit reclaim rather than
  global. It would read `global` for system-wide direct reclaim, or `both` if
  the lock happened to be held across one of each.
* `pid=441946 tid=441946 process=dd thread=dd` -- who held it. `pid`/`process`
  are the thread group; `tid`/`thread` are the specific thread, which differ
  for multithreaded programs.
* `acquire stack` -- where the lock was taken. Here `btrfs_inode_lock` from
  `btrfs_buffered_write`: this is the inode lock of the file being written.
* `reclaim-entry stack (longest episode)` -- where reclaim was entered while
  the lock was held. Here the write needed a new page cache folio,
  `filemap_add_folio` tried to charge it to the cgroup, the charge hit
  `memory.max`, and `__mem_cgroup_charge` called into
  `try_to_free_mem_cgroup_pages`. If the lock was held across several reclaim
  episodes, this is the stack of the longest one.

So the whole story is: `dd` grabbed the inode lock to do a buffered write,
needed a page for the page cache, hit the cgroup memory limit, and went off to
reclaim for 2 ms while still holding the inode lock.

That run is not yet a problem, because nothing else wanted that inode. The
interesting case is when something does. Adding `--waiters_only=true` drops
every report where no one was waiting at release time, which is normally the
vast majority of them, and leaves only the locks that actually stalled another
task. With four `dd` processes appending to the *same* file, so they contend
for one inode lock:

```
# ./reclaimlockmonitor.bt -- --waiters_only=true
Attached 23 probes
>>> RECLAIM_LOCK: rwsem write 0xffff8887a2d8d220 [WAITERS]
    hold=554690 us  reclaim=551376 us  reclaim_type=memcg
    pid=412486   tid=412486   process=dd thread=dd
    acquire stack:

        down_write+5
        btrfs_inode_lock+36
        btrfs_buffered_write+108
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108

    reclaim-entry stack (longest episode):

        try_to_free_mem_cgroup_pages+510
        try_to_free_mem_cgroup_pages+510
        __mem_cgroup_charge+1873
        filemap_add_folio+144
        __filemap_get_folio+745
        prepare_one_folio+76
        btrfs_buffered_write+637
        btrfs_do_write_iter+128
        __x64_sys_write+724
        do_syscall_64+84
        entry_SYSCALL_64_after_hwframe+108
```

Same code path, much worse outcome. The `[WAITERS]` tag means at least one
other task was blocked on this lock at the moment it was released. The holder
kept the inode lock for 554 ms and spent 551 ms of that -- 99.4% -- in memcg
reclaim. The other three writers were stuck behind it essentially the entire
time. Nothing here is a slow filesystem or a hot lock; it is one cgroup's
memory limit being paid for inside a critical section that everyone else
needs.

A report can also be tagged `[INFLIGHT]`, which means the lock was still held
when the script was stopped and so was reported by the `END` action instead of
by its unlock. For those, `hold` and `reclaim` are lower bounds -- both were
still growing when tracing ended.

## Reproducing the examples

Both runs above came from writing into a cgroup that is too small to hold the
page cache being dirtied, which forces memcg reclaim in the write path:

```
mkdir /sys/fs/cgroup/rlmtest
echo 268435456 > /sys/fs/cgroup/rlmtest/memory.max
echo 0 > /sys/fs/cgroup/rlmtest/memory.swap.max

# first example: one writer, no contention
bash -c 'echo $$ > /sys/fs/cgroup/rlmtest/cgroup.procs
         dd if=/dev/zero of=/tmp/work.dat bs=1M count=2500'

# second example: four writers appending to one file, contending on its inode
bash -c 'echo $$ > /sys/fs/cgroup/rlmtest/cgroup.procs
         for i in 1 2 3 4; do
             dd if=/dev/zero of=/tmp/shared.dat bs=1M count=1500 \
                conv=notrunc oflag=append &
         done
         wait'
```

Start the tool first and let it finish attaching, then run the workload.

## Notes and caveats

* **Overhead is significant.** Every `mutex_lock`/`down_read`/`down_write` in
  the system is probed, and a kernel stack is captured on each acquisition.
  Use short run windows and do not leave this running on a busy machine.
* **Only the write side of percpu_rw_semaphore is traced.**
  `percpu_down_read()`/`percpu_up_read()` are inlined per-CPU counter
  increments with no symbol to attach to. The write side is the case worth
  watching anyway, since an exclusive writer stalled in reclaim blocks all
  readers and writers.
* **Up to 8 held locks are tracked per task.** Deeper nesting is ignored.
  Unlocks out of acquisition order are handled.
* **Reclaim is only counted when the task is in a memory stall**
  (`in_memstall`), which is what distinguishes an allocation blocked on reclaim
  from background reclaim work.
* A lock reported without `[WAITERS]` delayed nobody at the moment it was
  released; it is worth knowing about as a latent risk, but it is not an active
  stall. Use `--waiters_only=true` to see only the ones that were.
* Requires `CONFIG_DEBUG_INFO_BTF=y`.

## USAGE

```
USAGE:
  ./reclaimlockmonitor.bt                        # all locks seen in reclaim
  ./reclaimlockmonitor.bt -- --waiters_only=true # contended locks only
```
