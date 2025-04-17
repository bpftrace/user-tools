# sigsnoop

This traces standard and real-time signals generated system wide.
This is done by instrumenting signal/signal_generate tracepoints.

## Output

```
$ sudo ./sigsnoop.bt
Attaching 9 probes...
Tracing signals, hit ctrl-c to end.
TIME     PID      COMM             TPID     SIG
08:25:28 195973   vte-urlencode-c  194760   SIGCHLD
08:25:29 195974   ls               194760   SIGCHLD
08:25:29 195975   vte-urlencode-c  194760   SIGCHLD
08:25:29 1847     Xorg             1847     SIGALRM
08:25:31 0        swapper/3        2594     SIGNAL-34
08:25:31 1847     Xorg             1847     SIGALRM
^CBye.
```

The first line shows that pid 195973 (locate thread) delivered a SIGCHLD
signal to pid 194760 (TPID = the target pid).
