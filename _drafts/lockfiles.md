---
layout: post
last_modified_at: 2019-11-25 01:07:03 UTC
---

Locking mechanisms are hard.
Let's look at some different ways to obtain a lock from a `bash` script.

1. TOC
{:toc}

### Options

lockfiles
* set -o noclobber && echo $$ >"$lockfile"
* mktemp
* shlock
* lockfile (procmail)
* flock (util-linux)
* don't use lock files

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

