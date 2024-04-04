# wakesnoop

`wakesnoop` attaches to scheduler tracepoints `sched_waking` and `sched_wakeup`, and
measures delay of wakeup. Measurements exceeding threshold, matching cpu mask
and pid are printed. If no optional arguments are provided, the tool prints all
measurements.

USAGE: wakesnoop.bt \[wakeup delay threshold, us\] \[cpu mask\] \[pid\]

## Output

```
# ./sched/wakesnoop.bt 100
Attaching 4 probes...
Tracing scheduler delay and in microseconds. Ctrl-C to end.
TIME            DELAY(us) CPU     PID COMM
21:49:23.425949       123   7    6124 pulseaudio
21:49:23.534792       116   8   32155 VizCompositorTh
21:49:24.213697       145   5    6124 pulseaudio
21:49:24.315695       113   5    6124 pulseaudio
21:49:24.603702       107   5    6124 pulseaudio
21:49:24.652704       107   5  444683 kworker/u25:3
```
