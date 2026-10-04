# proccputime

This shows the CPU time of every process that exits while tracing, including
short-lived ones, split into user and system time. Each thread's final CPU
time is read at its last context switch (`sched_switch` with
`prev_state == TASK_DEAD`) and summed per process.

Interval-based tools such as `pidstat` and `top` read `/proc` every N seconds.
A process that starts and exits between two samples never shows up, and a
process that lives a few seconds loses the CPU time it used before the first
sample and after the last one. When a workload forks many short-lived helpers
(builds, test runners, shell scripts), `time` tells you the total but not who
used it, and `pidstat` cannot fill the gap. `proccputime` catches each process
at its exit, so none are missed.

## Output

Tracing a Next.js production build (`npm run build`), excerpt:

```
# ./proccputime.bt
Attached 4 probes
Tracing Process CPU-time... Hit Ctrl-C to end.
^C
PID      COMM              UTIME(ms)  STIME(ms)  RTIME(us)
4486     sh                        0          1       1480
4515     git                       1          0       1325
4581     MainThread              531        186     717927
4532     MainThread             1570        445    2015342
4487     next-build (v16         824        611    1435190
4514     sh                        0          4       4483
4525     MainThread             2008        448    2457136
4593     MainThread             1756        430    2186436
4475     npm run build            79         30     110281
4773     sed                       1          0       1811
...
```

Reading the third line:

* `4581` -- the PID, as seen from bpftrace's PID namespace (the number `ps`
  shows).
* `MainThread` -- the process name (`comm` of the thread group leader, up to 15
  characters). This one is a Node.js worker process that does not set a process
  title. There are several of them; each gets its own line because lines are
  keyed by PID, not by name.
* `531` / `186` -- it used 531 ms of user CPU time and 186 ms of system CPU
  time.
* `717927` -- 717,927 us of CPU time in total (`se.sum_exec_runtime`). UTIME
  and STIME are this total split in the same ratio the kernel uses.

Values cover the process's whole life and all of its threads. Lines are not
sorted.

This is the problem the tool was written for. On the same build (12 CPUs),
`pidstat 1` showed only about 20 of the 41 processes the build created, and
2.89 s (15%) of the CPU time reported by `time` could not be attributed to any
process. `proccputime` gives every process its own line, which showed where
that time went: to the Node.js worker processes (`MainThread`). These 13-14
short-lived workers used about 85% of the build's CPU time in total. Each
lived only a few seconds, so `pidstat` missed the parts of their lives that
fell outside its samples; that is where most of the missing 2.89 s went.

## Checking against time(1)

Run the tool in one terminal and `time npm run build` (or any workload) in
another, with background activity stopped, then hit Ctrl-C and sum all printed
lines (no filtering by name). From the run above:

| | proccputime | `time` | Difference |
|---|---|---|---|
| user | 12,584 ms | 12,535 ms | +0.39% |
| sys | 3,717 ms | 3,704 ms | +0.35% |
| user + sys | 16,322 ms | 16,239 ms | +0.51% |

Counting only the build's own lines (`npm run build`, `next-build`,
`MainThread`) gives +0.37% in total, so the result does not depend on which
lines are picked. The cause of the remaining 0.4-0.5% excess is not known yet.

## Notes and caveats

* **Why the last context switch, not `sched_process_exit`.** That tracepoint
  fires in the middle of `do_exit()`, when the CPU time is not final yet: the
  time since the last scheduler update has not been added, and the rest of the
  exit path still runs. For a short-lived `ls`, a median of 78% of the final
  value was still missing there. After the last switch the thread never runs
  again, and `prev` is still valid because the `task_struct` is freed later.
* **Only processes that exited are shown.** Threads are summed per `tgid`, and
  a process is printed once its last thread has exited (`group_dead` in
  `sched_process_exit`). Processes still running at Ctrl-C are not shown.
* **The user/system split is as accurate as the kernel's.** The total is the
  precise `sum_exec_runtime`, divided in the ratio of the tick-sampled `utime`
  and `stime`, as `cputime_adjust()` does for `/proc` and `time`. On kernels
  without `CONFIG_VIRT_CPU_ACCOUNTING_GEN` the split cannot be better than
  that.
* **Overhead scales with context switches**, not with the workload: about
  170 ns per `sched_switch` and 1.4 us per `sched_process_exit` (BPF run time
  from `kernel.bpf_stats_enabled`, a lower bound). For the build above (about
  10,000 switches per second) that is 0.085% of its CPU time.
* For processes already running before tracing started, threads that exited
  before tracing started are missing. Values are lifetime totals, not "CPU time
  used during tracing".
* UTIME and STIME are truncated to milliseconds, so on each line UTIME + STIME
  can be up to about 2 ms less than RTIME.
* If a PID is reused during tracing, the two processes are merged into one
  line.
* The integer arithmetic overflows at about 37 hours of CPU time per process.
* Processes in other PID namespaces (e.g. containers) may be shown with PID 0
  (not verified).
* COMM is the last name the process had. Kernel threads are not counted.
* Requires `CONFIG_DEBUG_INFO_BTF=y`. Tested on Linux 6.18 (WSL2,
  `CONFIG_HZ=250`) with bpftrace v0.27.0.

## USAGE

```
USAGE:
  ./proccputime.bt    # trace until Ctrl-C, then print every process that exited
```
