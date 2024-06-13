# nfcttrace

This traces TCP and UDP entries in nf_conntrack.
This is done by instrumenting kprobe:nf_ct_delete.

## Output

$ sudo ./nfcttrace.bt
Attaching 3 probes...
Show nf_conntrack TCP&UDP entries... Hit Ctrl-C to end.
[nf_ct] proto=tcp status=replied osaddr=10.203.114.10 odaddr=10.203.100.11 osport=34964 odport=6443 rsaddr=10.203.100.11 rdaddr=10.203.114.10 rsport=6443 rdport=34964
[nf_ct] proto=tcp status=replied osaddr=127.0.0.1 odaddr=127.0.0.1 osport=57032 odport=8181 rsaddr=127.0.0.1 rdaddr=127.0.0.1 rsport=8181 rdport=57032
[nf_ct] proto=udp status=replied osaddr=10.203.100.11 odaddr=10.206.130.1 osport=53321 odport=53 rsaddr=10.206.130.1 rdaddr=10.203.100.11 rsport=53 rdport=53321
[nf_ct] proto=tcp status=replied osaddr=10.203.100.11 odaddr=10.203.100.11 osport=43798 odport=10254 rsaddr=10.203.100.11 rdaddr=10.203.100.11 rsport=10254 rdport=43798
[nf_ct] proto=udp status=replied osaddr=10.203.100.11 odaddr=10.206.130.1 osport=56589 odport=53 rsaddr=10.206.130.1 rdaddr=10.203.100.11 rsport=53 rdport=56589
[nf_ct] proto=tcp status=replied osaddr=10.203.114.12 odaddr=10.203.100.11 osport=20590 odport=10256 rsaddr=10.203.100.11 rdaddr=10.203.114.12 rsport=10256 rdport=20590
^C