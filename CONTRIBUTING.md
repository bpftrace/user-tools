# Contributing bpftrace/eBPF tools

If you want to contribute tools to bpftrace's user-tools, please follow the 
guidelines below.

## File Structure for New Tools
- <mytoolname> (directory)
  - README.md
  - <mytoolname>.bt
  
**Example:**
- sigsnoop
  - README.md
  - sigsnoop.bt
  
## Logistics
1. **Create a folder named after the tool**
1. **Add a README.md file in this folder**. Copy the style in 
sigsnoop/README.md: start with an intro sentence, then have examples, and 
finish with the USAGE message. Explain everything: the first example should 
explain what we are seeing, even if this seems obvious. For some people it 
won't be obvious. Also explain why we are running the tool: what problems 
it's solving.
1. **Add your tool in this folder**
1. **Add usage instructions as a comment at the top of the tool**. Think of 
this as a man page. Add details about positional params and how they affect the 
tool's behavior. These details can also go in the README.md.
1. **Make sure your tool's license is correct**. An explicit license is not 
required but if you want one, follow [this guide](README.md#licensing).
1. **Spell check your documentation**. Use a spell checker like aspell to check 
your document quality before committing.
1. **Add an entry to TOOLS.md**. Make sure this list stays alphabetized.
1. **Make a pull request!**

## Development Checklist

_(Adapted from the [bcc version](https://github.com/iovisor/bcc/blob/master/CONTRIBUTING-SCRIPTS.md))._

1. **Research the topic landscape**. Learn the existing tools and metrics 
(incl. from /proc). Determine what real world problems exist and need solving. 
We have too many tools and metrics as it is, we don't need more "I guess that's 
useful" tools, we need more "ah-hah! I couldn't do this before!" tools. 
Consider asking other developers about your idea. Many of us can be found in 
IRC, in the #iovisor or #bpftrace channels on irc.oftc.net.
1. **Create a known workload for testing**. This might involving writing a 10 
line C program, using a micro-benchmark, or just improvising at the shell. If 
you don't know how to create a workload, learn! Figuring this out will provide 
invaluable context and details that you may have otherwise overlooked. 
Sometimes it's easy, and I'm able to just use dd(1) from /dev/urandom or a 
disk device to /dev/null. It lets me set the I/O size, count, and provides 
throughput statistics for cross-checking my tool output. But other times I 
need a micro-benchmark or some C.
1. **Write the tool to solve the problem and no more**. Unix philosophy: do one 
thing and do it well. netstat doesn't have an option to dump packets, 
tcpdump-style. They are two different tools.
1. **Consider bcc for custom output and options**. Need to really customize 
your output? Want to support a variety of command line options? It sounds like 
your tool may be better as a bcc tool, which currently supports these using 
Python (and other) interfaces [bcc](https://github.com/iovisor/bcc).
1. **Check your tool correctly measures your known workload**. If possible, 
run a prime number of events (eg, 23) and check that the numbers match. Try 
other workload variations.
1. **Use other observability tools to perform a cross-check or sanity check**. 
Eg, imagine you write a PCI bus tool that shows current throughput is 28 
Gbytes/sec. How could you sanity test that? Well, what PCI devices are there? 
Disks and network cards? Measure their throughput (iostat, nicstat, sar), and 
check if is in the ballpark of 28 Gbytes/sec (which would include PCI frame 
overheads). Ideally, your numbers match.
1. **Measure the overhead of the tool**. If you are running a micro-benchmark, 
how much slower is it with the tool running. Is more CPU consumed? Try to 
determine the worst case: run the micro-benchmark so that CPU headroom is 
exhausted, and then run the bpftrace tool. Can overhead be lowered?
1. **Test again, and stress test**. You want to discover and fix all the bad 
things before others hit them.
1. **Concise, intuitive, self-explanatory output**. The default output should 
meet the common need concisely. Consider including a startup message that's 
self-explanatory, eg "Tracing block I/O. Output every 1 seconds. Ctrl-C to 
end.". Also, try hard to keep the output less than 80 characters wide, 
especially the default output of the tool. That way, the output not only 
fits on the smallest reasonable terminal, it also fits well in slide decks, 
blog posts, articles, and printed material, all of which help education and 
adoption. Publishers of technical books often have templates they require 
books to conform to: it may not be an option to shrink or narrow the font to 
fit your output.
1. **Check style**. Do you have a consistent convention for indentation, 
variable names, and comment style? You can follow the lead from the other tools.
1. **Test it against the HEAD of bpftrace**. If possible, build [bpftrace](https://github.com/bpftrace/bpftrace/tree/master) 
locally and run your script against the HEAD of the master branch to make sure 
your script works for the latest, unreleased version. If something is broken 
and you believe this is unintentional, please [file an issue](https://github.com/bpftrace/bpftrace/issues).