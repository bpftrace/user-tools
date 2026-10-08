# runqslower

This prints run queue waits that are longer than a threshold, one line per
wait: which thread waited, how long it waited, and which thread was using the
CPU just before it got on. "slower" follows the bcc naming (ext4slower,
fileslower): only the slow ones are shown. This is a bpftrace version of the
bcc libbpf-tools/runqslower of the same name.

`runqlat.bt` shows how run queue latency is distributed as a histogram, but
not which thread waited. CPU usage tools such as `top` show time spent on the
CPU, not time spent waiting for it. `runqslower.bt` answers "who waited, for
how long, and who had the CPU".

## Output

```
# ./runqslower.bt 100
Attached 5 probes
Tracing run queue latency higher than 100 us
TIME     COMM             TID         LAT(us) PREV COMM        PREV TID
06:39:20 kworker/11:0     74              244 chronyd          546
06:39:20 containerd       587             168 swapper/0        0
06:39:20 kworker/11:0     74              111 chronyd          546
06:39:20 kworker/2:1      107             125 swapper/2        0
06:39:20 MainThread       1193            630 swapper/1        0
06:39:21 copilot-runtime  1246            145 swapper/2        0
06:39:21 kworker/11:0     74              124 chronyd          546
06:39:21 containerd       661             108 swapper/9        0
06:39:22 copilot-runtime  1246            133 swapper/2        0
06:39:22 copilot-runtime  1246            105 swapper/2        0
^C
```

| Column | Meaning |
|---|---|
| TIME | Time of the event |
| COMM | Name of the thread that finished waiting and got on the CPU |
| TID | Thread ID of that thread |
| LAT(us) | How long it waited in the run queue, in microseconds |
| PREV COMM | Name of the thread that was using the CPU just before |
| PREV TID | Thread ID of that thread |

`swapper/N` with PREV TID 0 is the idle task (one per CPU, all with TID 0).
A line with PREV TID 0 means the thread waited even though the CPU was idle.
In this example, MainThread waited 630 us although CPU 1 was idle
(PREV COMM is swapper/1): waits happen even without CPU contention.

A second example, with two `yes` processes pinned to CPU 0 so that they
compete for one CPU:

```
# taskset -c 0 yes > /dev/null &
# taskset -c 0 yes > /dev/null &
# ./runqslower.bt 1000
Attached 5 probes
Tracing run queue latency higher than 1000 us
TIME     COMM             TID         LAT(us) PREV COMM        PREV TID
07:59:22 yes              6207           4011 yes              6213
07:59:22 yes              6213           3923 yes              6207
07:59:22 yes              6207           4009 yes              6213
07:59:22 yes              6213           3999 yes              6207
07:59:22 yes              6207           4037 kworker/0:0      5976
07:59:22 yes              6213           3983 yes              6207
^C
```

TID and PREV TID swap on every line: the two `yes` take turns on CPU 0, and
while one runs, the other waits. Each wait is about 4 ms because this kernel
has CONFIG_HZ=250 (a 4 ms tick) and the switch happens on a tick. Now and
then a kworker gets in between (PREV COMM is kworker/0:0).

## How it works

- A thread starts waiting in the run queue at one of three points:
  - `tracepoint:sched:sched_wakeup`: a sleeping thread is woken up
  - `tracepoint:sched:sched_wakeup_new`: a new thread is created
  - `tracepoint:sched:sched_switch`: a thread leaves the CPU while still
    runnable
    - `prev_state == TASK_RUNNING`: it gave up the CPU on its own
    - `prev_state == TASK_REPORT_MAX`: it was preempted (the kernel reports
      this value instead of the real state for a preempted task)
- When a thread is switched in, the tool computes and prints
  `[current time] - [time it entered the run queue]`.
- When the thread switched in is the idle task (TID 0), no latency is
  computed: the CPU is going idle, nobody was waiting.

Related tool: `wakesnoop` measures the delay of the wakeup itself
(`sched_waking` to `sched_wakeup`). `runqslower` starts where it ends: from
the wakeup to getting on the CPU.

## Checking against bcc runqslower

Both tools were run at the same time, and two `yes` pinned to CPU 0 were run
for 5 seconds only while both were tracing, so both tools saw the same
events. Threshold 1000 us, counting the `yes` lines. Two runs (run 1 / run 2):

| | bcc runqslower | runqslower.bt |
|---|---|---|
| Lines | 1239 / 1250 | 1239 / 1250 |
| Mean | 4036.4 / 4000.3 us | 4036.3 / 4000.0 us |
| Min | 3644 / 3732 us | 3644 / 3732 us |
| Max | 8010 / 4280 us | 8029 / 4297 us |

- Line by line, the TID sequences matched in every pair in both runs (1239
  and 1250 pairs). The largest difference for a single event was 43 / 69 us.
- The mean difference was -0.11 / -0.33 us: in both runs runqslower.bt was
  slightly lower. That is under 0.01% of a 4 ms wait, which does not matter
  for finding long waits. The cause is not confirmed; a likely one is that
  the two tools' programs run one after the other on the same event and read
  the clock at slightly different times.
- With no load, run 1 recorded the same single event in both tools. In run 2
  only bcc recorded one event, whose PREV COMM was bpftrace itself, most
  likely before bpftrace had finished attaching.

Tested on Linux 6.18 (WSL2, CONFIG_HZ=250, with BTF) and bpftrace v0.27.0.

## Notes and caveats

- TIDs are the kernel's IDs. Inside containers or WSL they may differ from
  what `ps` shows.
- A small threshold prints a lot of lines and adds overhead. With 0, every
  wait is printed.
- The preemption path (`TASK_REPORT_MAX`) never occurred in the test
  environment, whose kernel is built with PREEMPT_NONE, so it is untested.
- A wait exactly equal to the threshold is printed. bcc runqslower prints
  only waits longer than the threshold.
- Running without an argument prints a warning that `$1` is empty. It is
  harmless: the default (10000) is used.

## USAGE

```
# ./runqslower.bt [min_us]

  min_us   Print only waits of at least this many microseconds.
           Default: 10000 (10 ms).

examples:
  ./runqslower.bt          # waits of 10000 us (10 ms) or longer
  ./runqslower.bt 1000     # waits of 1000 us (1 ms) or longer
```
