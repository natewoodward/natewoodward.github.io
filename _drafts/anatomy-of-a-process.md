---
layout: post
last_modified_at: 2019-11-29 21:44:30 UTC
---

Commands are the basic building blocks of a `bash` script,
and many commands run as an operating system process.
Let's go over how they work and some of the ways we can interact with them from a shell.

I know this is going to seem incredibly basic to a lot of you,
but I recently had a conversation with a friend who,
despite being familiar with processes as viewed in Windows Task Manager,
was a bit fuzzy about how the shell launches and interacts with them.
This article is for people like him who are new to interacting with processes in a *nix environment.

1. TOC
{:toc}

### Topics

* what is a process
  * identified by a pid
  * top, ps, pidof, pgrep, pstree
* what is not a process
  * keywords, builtins, functions, aliases kinda
* arguments
  * main() injection point
  * options, files
* environment variables
  * child processes
* stdin, stdout, stderr
* exit status
* signals
* other ipc: sockets, FIFOs, shared memory, message buses

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

