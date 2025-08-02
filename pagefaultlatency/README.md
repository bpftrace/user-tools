# pagefaultlatency

Pagefaults are a common exception in Linux kernel. Their handling efficiency
significantly impacts the performance of programs with frequent memory accesses.
This tool helps assess pagefault latency when executing specific programs.

`pagefaultlatency.bt` traces pagefaults by instrumenting `kprobe:handle_mm_fault`.
Its output shows:
- The number of pagefaults generated and the latency of each pagefault
- The average pagefault latency
- A pagefault latency histogram

USAGE: `pagefaultlatency.bt [thread_name]`

## Output

1. Open console no.1

```shell
$ sudo ./pagefaultlatency.bt ls
Attaching 4 probes...
proc=ls
```

2. Open console no.2

```shell
$ ls -l
```

3. The console no.1 outputs the average pagefault time. Press Ctrl+C to output
the latency histogram.

```shell
[000] latency=11637 ns
[001] latency=5921 ns
[002] latency=3471 ns
...
[408] latency=1429 ns
[409] latency=2041 ns
[410] latency=1429 ns
^CAverage page fault time for comm[ps]: 2765 ns [cnt=411]

@usecs:
[0]                   45 |@@@@@@@@@@@@                                        |
[1]                  191 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[2, 4)               126 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                  |
[4, 8)                37 |@@@@@@@@@@                                          |
[8, 16)               10 |@@                                                  |
[16, 32)               2 |                                                    |
```
