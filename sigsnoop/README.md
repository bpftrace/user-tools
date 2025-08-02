# sigsnoop

This traces standard and real-time signals generated system wide.
This is done by instrumenting signal/signal_generate tracepoints.

## Output

```
$ sudo ./sigsnoop.bt
Attaching 9 probes...
Tracing signals, hit ctrl-c to end.
TIME     PID      COMM             TPID     SIG
09:42:33 214863   locate           214860   SIGCHLD
09:42:33 214866   locate           214860   SIGCHLD
09:42:33 214862   rpm              214861   SIGCHLD
09:42:33 214868   rpm              214861   SIGCHLD
09:42:33 214869   rpm              214861   SIGCHLD
09:42:33 214870   uname            214861   SIGCHLD
09:42:34 214879   rpm              214874   SIGCHLD
```

The first line shows that pid 214863 (locate thread) delivered a SIGCHLD 
signal to pid 214860 (TPID = the target pid).
