# slableaktracker

Tracks outstanding slab object allocations within a single slab. Useful for tracking down the source of slab leaks.
On slab object allocation, creates a record of allocation information and deletes that record when the object is freed. Will terminate early if 65536 allocations are recorded.

## Output

```
# ./slableaktracker.bt dentry
Attaching 4 probes...
Starting slab user tracking for slab dentry.
^C@remaining_allocs[1804, bpftrace, 
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1b4a4bd __d_alloc+45
	ffffffffb1b4d67e d_alloc_pseudo+14
	ffffffffb1b29a4a alloc_file_pseudo+106
	ffffffffb1ba124b __anon_inode_getfile+123
	ffffffffb19cbd17 __do_sys_perf_event_open+2279
	ffffffffb254387c do_syscall_64+124
	ffffffffb140012f entry_SYSCALL_64_after_hwframe+118
]: 1
@remaining_allocs[1813, find, 
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1b4a4bd __d_alloc+45
	ffffffffb1b4d7fd d_alloc_parallel+61
	ffffffffb1bf2498 proc_fill_cache+232
	ffffffffb1bf3363 proc_pid_readdir+339
	ffffffffb1b45a0a iterate_dir+170
	ffffffffb1b4607b __x64_sys_getdents64+123
	ffffffffb254387c do_syscall_64+124
	ffffffffb140012f entry_SYSCALL_64_after_hwframe+118
]: 4
...
@remaining_allocs[1813, find, 
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1a96ecf kmem_cache_alloc_lru_noprof+655
	ffffffffb1b4a4bd __d_alloc+45
	ffffffffb1b4d7fd d_alloc_parallel+61
	ffffffffb1bf2498 proc_fill_cache+232
	ffffffffb1bf293b proc_map_files_readdir+1035
	ffffffffb1b45a0a iterate_dir+170
	ffffffffb1b4607b __x64_sys_getdents64+123
	ffffffffb254387c do_syscall_64+124
	ffffffffb140012f entry_SYSCALL_64_after_hwframe+118
]: 592
Remaining dentry object count: 1486
```

Each "stanza" is the count of unique combinations of the process and pid the allocation occurred in and the code path taken to allocate the object. For example, the last stanza indicates 592 `dentry` slab objects were allocated by the `find` command with PID 1813. It allocated that many because the code path needed to iterate a directory in `/proc` and store the dentry object found. A total of 1486 dentry slab objects were not freed before the script terminated.
