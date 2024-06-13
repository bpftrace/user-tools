# user-tools

This repository contains [community contributed tools](TOOLS.md). This is different from
[official bpftrace tools][0] in that we do not intend to be as strict about
requirements.

Thus, we encourage users to contribute anything they find useful for the
community. Motivations could include (but are not limited to):

* Sharing useful functionality
* Sharing novel techniques
* Incubating a new official tool

One intention is that all new official tools begin life in this repository. And
over time we'll develop some criteria for promoting a user tool to an official
tool that bpftrace maintainers will help support over long periods of time.

[0]: https://github.com/bpftrace/bpftrace/blob/master/CONTRIBUTING-TOOLS.md

## [Tools List](TOOLS.md)

## [How to Contribute a New Tool](CONTRIBUTING.md)

If you want to contribute bpftrace tools to this repo, please follow 
[these guidelines](CONTRIBUTING.md).

## Licensing

Depending on which [BPF helpers](kernel-versions.md#helpers) are used by 
bpftrace when compiling your program, a GPL-compatible license is required for 
bpftrace programs/tools.

You can set the license as a comment in your source code (at the top of the 
file). Note: it supports multiple words and quotes are not necessary. **If the 
license is not specified, bpftrace will automatically define the license of the 
program as GPL v2.**

Examples:

- `// SPDX-License-Identifier: GPL-2.0-or-later`
- `// SPDX-License-Identifier: GPL-2.0-or-later OR BSD-2-Clause`

Currently, only these licenses are supported in this repo:

- GPL
- GPL v2
- Dual BSD/GPL
- Dual MIT/GPL
- Dual MPL/GPL

Note: GPL v3 is not supported.