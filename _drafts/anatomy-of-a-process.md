---
layout: post
last_modified_at: 2019-11-28 17:08:01 UTC
---

Commands are the basic building blocks of a `bash` script,
and many commands run as an operating system process.
Let's go over how they work and some of the ways we can interact with them.

I know this is going to seem incredibly basic to a lot of you,
but I recently had a conversation with a friend who was a Windows administrator of many years.
He dabbled in Linux here and there and he wanted to know more about some of the shell constructs he had been using.
As the conversation progressed it became clear that although he had interacted with processes for years with Task Manager and such,
the way the shell launched and interacted with processes was a bit fuzzy to him.
This article is for people like him who, though they may know what processes are,
are new to interacting with them in a Linux environment.

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

