---
layout: post
last_modified_at: 2019-11-24 21:29:14 UTC
---

I'd like to share a couple of ways to use `grep` that I never would have thought to do myself until I came across them.
I'll do this using a couple of hardening guidelines from the CIS CentOS Linux 7 Benchmark as examples.

1. TOC
{:toc}

### Example Use Cases

The CIS CentOS Linux 7 Benchmark is a document published by the Center for Internet Security (CIS) with guidance on hardening CentOS 7.
Towards the end of version 2.2.0 of the document is recommendation 6.1.12,
"Ensure no ungrouped files or directories exist".
It suggests using the following command to check for files and directories that have a group ID that isn't configured on the system:

```bash
df --local -P | awk {'if (NR!=1) print $6'} | xargs -I '{}' find '{}' -xdev -nogroup
```

The next recommendation is 6.1.13, "Audit SUID executables".
It suggests reviewing SUID programs installed on a system with the following command,
which searches each local filesystem that's mounted and prints any SUID files it finds.

```bash
df --local -P | awk {'if (NR!=1) print $6'} | xargs -I '{}' find '{}' -xdev -type f -perm -4000
```

Suppose we want to script these and other tasks from the benchmark.
Let's start by putting the commands into a script and breaking up the pipelines across several lines:

```bash
#!/bin/bash

# 6.1.12 ensure no ungrouped files or directories exist (scored)
df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -nogroup

# 6.1.13 audit suid executables (not scored)
df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000
```

I find this much easier to read,
but I suppose that's down to personal preference so your mileage may vary.

### Check if a Command has Output

Suppose that we want our script to exit with an error if it finds any ungrouped files for requirement 6.1.12.
That way, the admin running the script can fix the group ownership on the files,
and re-run the script to validate that there are no longer any ungrouped files on the system.

Perhaps the most straightforward way to do this is by capturing the pipeline's output in a variable:

```bash
# 6.1.12 ensure no ungrouped files or directories exist (scored)
ungroupedfiles="$(df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -nogroup
)"
if [[ -n "$ungroupedfiles" ]]; then
	echo "$ungroupedfiles"
	echo "CIS 6.1.12: Found ungrouped files or directories. Aborting." >&2
	exit 1
fi
```

This gets the job done, but the user doesn't see any output until after the pipeline completes.
We can give more immediate feedback by checking for output directly in the pipeline with a simple `grep`:

```bash
# 6.1.12 ensure no ungrouped files or directories exist (scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -nogroup \
	| grep .
then
	echo "CIS 6.1.12: Found ungrouped files or directories. Aborting." >&2
	exit 1
fi
```

This way, the user sees the ungrouped files as soon as `find` detects them.
The `grep .` command will match and print almost any line of output.
The one caveat is that it won't match or print empty lines.
This isn't a problem in our case since `find`'s output doesn't contain any empty lines[^1].

Since the return status of a pipeline is the exit status of its last command,
and since grep's exit status indicates whether it found a match,
we can test the whole pipeline directly with our `if` statement to determine if the pipeline had any output or not.

### Exclude Files

After verifying the SUID files that the script outputs for requirement 6.1.13,
you might want to omit some or all of them in future runs of the script so that you don't have to re-evaluate them.
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

With this in place, a nonexistent exclude file is equivalent to an empty one,
and our script behaves the same in both cases.

If we additionally want to exit with an error so that the admin can validate any unignored SUID files,
we can just test the `grep` at the end of the pipeline again.

```bash
# 6.1.13 audit suid executables (not scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 \
	| grep -vFxf "$excludefile"
then
	echo "CIS 6.1.13: Found SUID system executables. Aborting." >&2
	exit 1
fi
```

This can be a bit confusing since grep's `-v` option inverts the sense of matching,
but you can think of the conditional as saying "if grep found a line *not* included in `$excludefile`."

### Final Script

Remember earlier when I said that `find` doesn't have empty lines in its output?
Technically, it can if you have a file on your system with two consecutive newlines in its name.
We can modify our script to handle this edge case by using `find -print0` combined with `grep -z`.
Below is the full script with that change.

TODO: Test `grep -z` on CentOS.

```bash
#!/bin/bash

# 6.1.12 ensure no ungrouped files or directories exist (scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -nogroup -print0 \
	| grep -z .
then
	echo "CIS 6.1.12: Found ungrouped files or directories. Aborting." >&2
	exit 1
fi

# define exclude file
excludefile=/usr/local/etc/suid-executables.exclude
if [[ ! -f "$excludefile" ]]; then
    excludefile=/dev/null
fi

# 6.1.13 audit suid executables (not scored)
if df --local -P \
	| awk '{if (NR!=1) print $6}' \
	| xargs -I '{}' find '{}' -xdev -type f -perm -4000 -print0 \
	| grep -z -vFxf "$excludefile"
then
	echo "CIS 6.1.13: Found SUID system executables. Aborting." >&2
	exit 1
fi
```

### Footnotes

[^1]: Except in certain edge cases. More on this later.

