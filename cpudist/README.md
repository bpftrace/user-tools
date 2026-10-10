# cpudist

This shows, as a histogram, how long a thread runs each time it gets on a
CPU: from being switched in until being switched out (on-CPU time). With
`--offcpu`, it shows the opposite: from being switched out until getting
back on (off-CPU time). This is a bpftrace version of the bcc tool of the
same name.

CPU usage tools such as `top` show only the total, so 10% CPU could be many
short runs or a few long ones. `cpudist.bt` shows the distribution of each
run, which tells them apart. For on-CPU time, a thread that sleeps and wakes
up quickly makes a peak at short times, and threads competing for a CPU make
a peak at the length of a time slice, where they are preempted.

`runqslower.bt` uses the same `sched_switch` tracepoint to show time spent
waiting for a CPU; `cpudist.bt` shows time spent running on it.

## Output

### Example 1: no options (one histogram for the whole system)

Five seconds on an idle machine:

```
# ./cpudist.bt
Tracing on-CPU time... Hit Ctrl-C to end.
^C

@usecs:
[4, 8)                 4 |@                                                   |
[8, 16)               65 |@@@@@@@@@@@@@@@@@                                   |
[16, 32)             195 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[32, 64)             180 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    |
[64, 128)            101 |@@@@@@@@@@@@@@@@@@@@@@@@@@                          |
[128, 256)            72 |@@@@@@@@@@@@@@@@@@@                                 |
[256, 512)            44 |@@@@@@@@@@@                                         |
[512, 1K)              8 |@@                                                  |
[1K, 2K)               1 |                                                    |
[2K, 4K)               0 |                                                    |
[4K, 8K)               0 |                                                    |
[8K, 16K)              0 |                                                    |
[16K, 32K)             1 |                                                    |
```

- Each line is one bucket. `[16, 32)` means 195 runs lasted at least 16 us
  and less than 32 us. The `@` bar on the right shows that count.
- Each bucket is twice as wide as the one before (4, 8, 16, 32, ...), so
  short times are shown in fine detail and long times coarsely.
- Here almost every run ended in less than 1 ms. On an idle machine,
  threads run briefly and soon go back to sleep by themselves.
- The idle task (TID 0), which runs when a CPU has nothing to do, is not
  counted (see Notes).

### Example 2: `--per_thread` (one histogram per thread)

Two `yes` processes pinned to CPU 0, competing for one CPU:

```
# taskset -c 0 yes > /dev/null &
# taskset -c 0 yes > /dev/null &
# ./cpudist.bt -- --per_thread
Tracing on-CPU time per thread... Hit Ctrl-C to end.
^C
...
@usecs_by_thread[yes, 1880]:
[2K, 4K)             400 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@usecs_by_thread[yes, 1881]:
[2K, 4K)             400 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

- `[yes, 1880]` is the thread named `yes` with TID 1880.
- Every run of both threads is in `[2K, 4K)`. This kernel has a 4 ms tick
  (CONFIG_HZ=250), and the two threads take turns on each tick, so each
  run is about 4 ms (4000 us is below 4096, so it falls in `[2K, 4K)`).
- Other threads were printed too; only the two `yes` are shown here.

### Example 3: `--offcpu` (time off the CPU)

A python3 loop that sleeps for 10 ms at a time:

```
# python3 -c 'import time
while True: time.sleep(0.01)' &
# ./cpudist.bt -- --offcpu --per_thread
Tracing off-CPU time per thread... Hit Ctrl-C to end.
^C
...
@usecs_by_thread[python3, 1944]:
[8K, 16K)            312 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

- The 10 ms (10000 us) sleep shows up as is, in `[8K, 16K)`.
- Without `--offcpu`, the same python3 runs for about 16-64 us each time
  (case A in "Checking against bcc cpudist" below). Together they show a
  thread that sleeps most of the time and, once woken, finishes its work
  almost at once.

## How it works

- On `sched_switch` (the moment one thread on a CPU is replaced by
  another), the tool records a timestamp and computes a duration.
  - On-CPU (default): it records the time when a thread is switched in
    (next). When that thread is switched out (prev), the difference from
    the current time is its running time.
  - Off-CPU (`--offcpu`): it records the time when a thread is switched out
    (prev). When that thread is switched in again (next), the difference
    from the current time is its time off the CPU.
  - In other words, on-CPU and off-CPU only swap the start and the end of
    the interval (next and prev).
- It does not matter why a thread left the CPU (preempted, or went to sleep
  by itself). Every switch-out is counted.
- No timestamp is recorded for the idle task (TID 0), so no duration is
  computed for it.

## Checking against bcc cpudist

`cpudist.bt` was compared with bcc libbpf-tools/cpudist v0.35.0.

Both tools were started at the same time, and a workload was run for 5
seconds only while both were tracing, so both tools saw the same events
(measured separately, the workload would differ from run to run and the
numbers could not be compared). Three cases: a thread that sleeps right
away (A), threads competing for a CPU (B), and no workload (C).

| Case | bcc cpudist (libbpf-tools) | cpudist.bt |
|---|---|---|
| A: python3 sleeping 10 ms at a time | 2 on-CPU runs of python3 | 487 runs (mostly 16-64 us) |
| B: two `yes` on CPU 0 | 588 / 587 runs | 588 / 587 runs |
| C: no workload | 2 names (one of them idle) | 110 names (idle: 0) |

- **B (switched out by preemption): the counts matched exactly.** The two
  `yes` are preempted each time their time slice runs out. On this path
  both tools measure the same thing.
- **A (switched out by going to sleep): 2 versus 487.** python3 wakes up
  about 480 times in 5 seconds and goes back to sleep right away each time.
  bcc libbpf-tools/cpudist counts a run only if the thread was still
  runnable when it was switched out, that is, preempted
  (`get_task_state(prev) == TASK_RUNNING` in `cpudist.bpf.c`), so it drops
  every run that ended in sleep.
- **Why `cpudist.bt` counts every run:** what users want to know is how
  long a thread runs each time, and that length does not depend on why it
  left the CPU. bcc tools/cpudist.py removed the same condition in 2021
  (iovisor/bcc#3456). `cpudist.bt` counts the same way as cpudist.py after
  that fix.
- In C, bcc libbpf-tools/cpudist dropped every ordinary process that went
  to sleep and kept only the idle task, while `cpudist.bt` skipped the idle
  task and showed 110 ordinary processes. cpudist.py also skips the idle
  task by default (iovisor/bcc#3924).

Off-CPU (`--offcpu`): the same 10 ms sleeping python3 was measured for 5
seconds with each tool (one after the other, not at the same time).
`cpudist.bt` counted 482 in `[8K, 16K)` and bcc `-O` counted 481 in
`8192 -> 16383`, matching the known 10 ms. The condition above is not used
for off-CPU time in bcc libbpf-tools/cpudist, so both tools counted the
sleeps here.

Tested on Linux 6.18 (WSL2, CONFIG_HZ=250, with BTF) and bpftrace v0.27.0.

## Notes and caveats

- The idle task (TID 0) is not counted. Time when a CPU has nothing to do
  is not time a thread spent running, and since every CPU's idle task has
  TID 0, mixing them in would also make the CPUs impossible to tell apart.
- TIDs are thread IDs. There is no per-process mode (bcc `-P`).
- For a thread that was already on (or off) the CPU when tracing started,
  the first interval is not counted. Intervals still in progress at Ctrl-C
  are not counted either.
- The 4 ms in Example 2 comes from this machine's tick (CONFIG_HZ=250).
  With a different tick or scheduler settings, the peak moves.
- `sched_switch` fires on every context switch, so the tool costs more on
  machines that switch a lot. In the worst case, two processes bouncing one
  byte through a pipe (a workload that does nothing but switch), pinned to
  CPU 0 for 5 seconds, made 676,026 round trips without the tool, 553,407
  with it, and 672,170 without it again: **about 18% fewer** (the two runs
  without the tool differed by 0.6%). That is about 0.8 us added per
  context switch. Ordinary workloads switch far less often, so the impact
  is much smaller.
- `--per_thread` output gets long when there are many short-lived threads
  (90-110 histograms even on an idle machine).

## USAGE

```
# ./cpudist.bt [-- [--offcpu] [--per_thread]]

  --offcpu       Measure off-CPU time (from leaving the CPU until getting
                 back on) instead of on-CPU time.
  --per_thread   Print one histogram per thread (COMM, TID) instead of one
                 for the whole system.

examples:
  ./cpudist.bt                              # on-CPU, whole system
  ./cpudist.bt -- --per_thread              # on-CPU, per thread
  ./cpudist.bt -- --offcpu                  # off-CPU, whole system
  ./cpudist.bt -- --offcpu --per_thread     # off-CPU, per thread
```

`--` separates bpftrace's own options from the script's options. Without
it, `--offcpu` and `--per_thread` are taken as bpftrace options.
