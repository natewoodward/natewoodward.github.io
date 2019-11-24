---
layout: post
last_modified_at: 2019-11-24 15:21:26 UTC 
---

I'd like to share a couple of non-obvious ways to use `grep` that I've come across over the years.
I'll do this using a hardening guideline from the CIS CentOS Linux 7 Benchmark as an example.

I'm sure that for some readers out there, this is super simple stuff,
but for me I never considered grepping like this until I stumbled across other people doing it.

1. TOC
{:toc}

### Example Use Case

The CIS CentOS Linux 7 Benchmark is a document published by the Center for Internet Security (CIS) with guidance on hardening CentOS 7.
Towards the end of version 2.2.0 of the document is recommendation 6.1.13, "Audit SUID executables".
It suggests reviewing SUID programs installed on a system with the following command,
which searches each local filesystem that's mounted and prints any SUID files it finds.

```bash
df --local -P | awk {'if (NR!=1) print $6'} | xargs -I '{}' find '{}' -xdev -type f -perm -4000
```

Suppose we want to script this and other tasks from the benchmark.
Let's start by putting the command into a script and breaking up each part of the pipeline so that it's a bit easier to read:

```bash
#!/bin/bash

# 6.1.13 audit suid executables (not scored)
df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000
```

### Exclude Files

After verifying the SUID files that the script outputs,
you might want to omit some or all of them from the output of future runs of the script so that you don't have to re-evaluate them.
In that case, it would be useful to have an exclude file that we can list such files in.

Let's implement that with a `grep` command tacked onto the end of the pipeline:

```bash
# define exclude file
excludefile=/usr/local/etc/suid-executables.exclude

# 6.1.13 audit suid executables (not scored)
df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 \
	| grep -vFxf "$excludefile"
```

The `grep` command we added uses the entries in our exclude file (`-f`) as strings (`-F`) to match whole lines (`-x`) in the pipeline's output,
and prints any lines that don't match (`-v`).
In other words, it will print anything except what we list in the exclude file.

If we wanted to give the user more control over what's excluded,
we could use some variation of `grep -vEf` instead,
which would let them use extended regular expressions to specify filename patterns.
Then they could exclude entire directories,
or files with a particular extension, for example.
But the downside of doing things this way is that the user would have to escape special characters like periods (`.`) when they occur in file paths.

One problem with the implementation of our script is that if `suid-executables.exclude` doesn't exist,
our script will error out with

```
grep: /usr/local/etc/suid-executables.exclude: No such file or directory
xargs: find: terminated by signal 13
```

Let's fix that by using `/dev/null` as our exclude file if `suid-executables.exclude` doesn't exist.

```bash
# define exclude file
excludefile=/usr/local/etc/suid-executables.exclude
if [[ ! -f "$excludefile" ]]; then
    excludefile=/dev/null
fi
```

### Check if a Command has Output

Suppose we want our script to take some action based on whether it finds any SUID files that aren't defined in our exclude file.
One way we can do this is by capturing the pipeline's output in a variable:

```bash
# 6.1.13 audit suid executables (not scored)
suidfiles="$(df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 -print \
	| grep -vFxf "$excludefile"
)"
if [[ -n "$suidfiles" ]]; then
	echo "$suidfiles"
	echo "CIS 6.1.13: Found SUID system executables. Aborting." >&2
	exit 1
fi
```

This gets the job done, but the user doesn't see any output until after the pipeline completes.
We can give more immediate feedback by checking for output directly in the pipeline with a simple `grep`:

```bash
# 6.1.13 audit suid executables (not scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 -print \
	| grep -vFxf "$excludefile" \
	| grep .
then
	echo "CIS 6.1.13: Found SUID system executables. Aborting." >&2
	exit 1
fi
```

This way, the user sees the SUID files as soon as `find` detects them.
The `grep .` command will print every non-empty line it sees.
Its exit status is 1 if it recieves any non-newline characters and 0 otherwise.
Since the return status of a pipeline is the exit status of its last command,
we can test the whole pipeline directly with our `if` statement to determine if the pipeline had any output or not.

### Final Script

Here's the full script after the two improvements we've made with grep:

```bash
#!/bin/bash

# define exclude file
excludefile=/usr/local/etc/suid-executables.exclude
if [[ ! -f "$excludefile" ]]; then
    excludefile=/dev/null
fi

# 6.1.13 audit suid executables (not scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 -print \
	| grep -vFxf "$excludefile" \
	| grep .
then
	echo "CIS 6.1.13: Found SUID system executables. Aborting." >&2
	exit 1
fi
```

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

