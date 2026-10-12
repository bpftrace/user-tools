# offcputime

This shows where threads were blocked, and for how long: the total time
spent off the CPU for each process name, kernel stack and user stack.
This is a bpftrace version of the bcc tool of the same name.

CPU profilers show where a program spends time while running. When a
program is slow but uses little CPU, the time goes to waiting instead:
sleeping, locks, disk or network. `offcputime.bt` shows the stacks where
that waiting happened.

`cpudist.bt -- --offcpu` shows how long each off-CPU interval was;
`offcputime.bt` shows where it was.

## Output

### Example: one process (`--pid`)

A python3 loop that sleeps for 10 ms at a time, traced for about 5 seconds:

```
# python3 -c 'import time
while True: time.sleep(0.01)' &
# ./offcputime.bt -- --pid=$!
Tracing off-CPU time of PID 19511... Hit Ctrl-C to end.
^C

@us[python3,
        __schedule+1802
        schedule+43
        do_nanosleep+96
        hrtimer_nanosleep+128
        common_nsleep_timens+82
        __x64_sys_clock_nanosleep+211
        x64_sys_call+3184
        do_syscall_64+110
        entry_SYSCALL_64_after_hwframe+118
,
        __internal_syscall_cancel+126@./nptl/cancellation.c:64
        __GI___clock_nanosleep+50@./time/../sysdeps/unix/sysv/linux/clock_nanosleep.c:71
        0x6f3c68
        _PyEval_EvalFrameDefault+5299
        PyEval_EvalCode+262
        0x6cab01
        0x6c551a
        0x6c52f4
        Py_RunMain+1259
        Py_BytesMain+52
        __libc_start_call_main+129@./csu/../sysdeps/nptl/libc_start_call_main.h:83
        [inlined] call_init@./csu/../csu/libc-start.c:128
        __libc_start_main_impl+136@./csu/../csu/libc-start.c:347
        _start+37
]: 4977071
```

- Each entry is `@us[process name, kernel stack, user stack]: total
  microseconds`. The most recent call is at the top of each stack.
- The kernel stack (before the comma) shows that python3 went to sleep in
  `clock_nanosleep`. The user stack (after the comma) shows that the call
  came from the Python interpreter through libc.
- 4,977,071 us is about 5 seconds: python3 spent almost all of the traced
  time asleep at this one place.
- Addresses such as `0x6f3c68` are functions in the python3 binary that
  have no symbol.

Without `--pid`, every process on the system is shown, one entry per
(process name, kernel stack, user stack). An idle machine printed 279
entries in 7 seconds.

## How it works

- On `sched_switch`, the thread leaving the CPU (prev) has its current
  time, kernel stack and user stack saved, keyed by its TID.
- When the same thread gets back on the CPU (next), the elapsed time is
  added to its (process name, kernel stack, user stack) entry.
- The idle task (TID 0) is skipped. Every CPU's idle task has TID 0, so
  their times cannot be told apart.
- With `--pid`, only threads of that process are recorded. The PID is
  compared with bpftrace's `pid`, which is the PID as seen from bpftrace's
  own PID namespace (the number shown by `ps` and the shell).

## Checking against bcc offcputime

`offcputime.bt` was compared with bcc libbpf-tools/offcputime v0.35.0.
Both tools were started at the same time, and three workloads ran for 5
seconds only while both were tracing: `ping -i 0.2`, a python3 loop
sleeping 10 ms at a time, and `yes` as a control (it never sleeps).

| Workload | bcc offcputime (us) | offcputime.bt (us) |
|---|---|---|
| ping | 5,010,175 / 5,009,951 | 5,010,128 / 5,009,965 |
| python3 | 4,988,446 / 4,994,765 | 4,988,116 / 4,994,863 |
| yes (control) | 1,106 | 1,016 |

- Two runs; ping and python3 matched within 0.01% in both.
- bcc was run with `--stack-storage-size 16384`. With the default (1,024)
  under this load, its stack map filled up and, in 3 of 3 runs, the time
  of ping or python3 was reported under the parent shell's name (`bash`)
  with an empty stack, because its table key had no process name. This is
  fixed in iovisor/bcc#5575. `offcputime.bt` keys by process name, and
  reported all three workloads under their own names in all 5 runs.
- bcc's output had no user stack frames in this environment
  (`failed to get syms`); `offcputime.bt` showed them.

Tested on Linux 6.18 (WSL2, with BTF) and bpftrace v0.27.0.

## Notes and caveats

- **PID namespaces (containers, WSL2):** when bpftrace runs inside a
  non-initial PID namespace, it prints many `get_ns_current_pid_tgid`
  warnings (`retcode: -22`) for threads outside that namespace: from
  `ustack` without `--pid` (516 in 7 seconds on an idle WSL2 machine), and
  from `pid` with `--pid` (440). The results are still correct. Use
  `bpftrace --no-warnings offcputime.bt` to hide them. On a host running
  in the initial PID namespace, no warnings appear.
- Threads with the same name and the same stacks are added together. There
  is no breakdown by PID or TID.
- Off-CPU time runs from leaving the CPU until getting back on, so it
  includes the time spent waiting in the run queue after being woken up.
  `runqslower.bt` shows that part on its own.
- User stacks of interpreted languages show interpreter frames, not lines
  of the script.
- Intervals that started before tracing began, or are still in progress at
  Ctrl-C, are not counted.
- `sched_switch` fires on every context switch, and this tool also walks
  the kernel and user stacks each time a thread leaves the CPU, so it costs
  more on machines that switch a lot. Its overhead was not measured
  separately; `cpudist.bt`, which uses the same tracepoint without stacks,
  added about 0.8 us per context switch in a worst case.

## USAGE

```
# ./offcputime.bt [-- [--pid=PID]]

  --pid=PID   Trace this process only (PID as shown by ps).

examples:
  ./offcputime.bt                          # whole system
  ./offcputime.bt -- --pid=1234            # PID 1234 only
  bpftrace --no-warnings offcputime.bt     # hide PID namespace warnings
```

`--` separates bpftrace's own options from the script's options, and the
`=` in `--pid=PID` is required (`--pid 1234` is rejected).
