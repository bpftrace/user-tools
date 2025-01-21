# hrtimersnoop

hrtimersnoop attaches to the hrtimer_expire_entry and hrtimer_expire_exit
tracepoints, measuring both the delay in timer expiration and the duration of
timer handler execution. Measurements exceeding at least one threshold are
printed.

USAGE: hrtimersnoop.bt \[delay threshold in microseconds\] \[duration threshold in microseconds\]

## Output

```
# ./hrtimersnoop.bt 100 10
Attaching 4 probes...
Tracing high-resolution timer delay and duration in microseconds. Ctrl-C to end.
TIME            HANDLER                           DELAY DURATION
11:37:32.804113 tick_sched_timer                      0       22
11:37:32.804148 tick_sched_timer                     45       13
11:37:32.804150 tick_sched_timer                     43       15
11:37:32.804152 tick_sched_timer                     45       16
11:37:32.804153 tick_sched_timer                     43       14
11:37:32.805093 hrtimer_wakeup                       86       10
11:37:32.805148 intel_uncore_fw_release_timer       966        1
```
