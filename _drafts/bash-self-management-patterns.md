---
layout: post
title: Bash Self-Management Patterns
---

Let's look at a few patterns that a `bash` shell script can use to modify its own behavior.
I'll show how a script can drop its own privileges, run with a lower priority,
obtain a lock file, and create a temp file.

1. TOC
{:toc}

### Drop Privileges

Occasionally you might write a script that needs to run as a particular user.
Rather than having to remember to use `su` to switch to that user or `sudo -u $user` to run the script as that user,
you can do the following:

```bash
# global vars
prog="${0##*/}"

# prints an error message to stderr
err() {
	local message="$1"
	shift
	printf "$message\n" "$@"
}

# drop privileges
user=nobody
if (( EUID == 0 )); then
	exec runuser -u "$user" "$0" "$@"
	# or if runuser isn't available on your system:
	#exec su "$user" -c "$(printf %q "$0") $(printf %q "$*")"
elif [[ $(id -un) != "$user" ]]; then
	err "This script must be run as %s or %s" 'root' "$user"
	exit 1
fi
```

Now you can run the script from a root shell or with `sudo`,
safe in the knowledge that it will only run as the `nobody` user.

### Obtain a Lock File

If you want to ensure that multiple instances of the same script can't run at the same time,
you can use the following to make each instance of a script try to obtain a lock file.
Only the first one to obtain the lock will continue running.
The others will exit.

Note that I'm using `$prog` and `err()` from the previous example without declaring them as I did in the previous example.
For the rest of this article, I'll assume that `$prog` and `err()` have already been declared.

```bash
# try to obtain lock
lockfile="${TMPDIR-/tmp}/$prog.lock"
if (set -o noclobber && echo $$>"$lockfile") 2>/dev/null; then
	trap "rm -f '$lockfile'" EXIT
else
	err "Lock %s held by pid %s" "$lockfile" "$(<"$lockfile")"
	exit 1
fi
```

### Create a Temp File

The [pitfalls involved in creating a temporary file](https://www.owasp.org/index.php/Insecure_Temporary_File) are well-known.
To atomically create a temp file, you can use `mktemp`.

```bash
# create a temporary file
trap "rm -f '$tmpfile'" EXIT
tmpfile="$(mktemp --tmpdir "$prog.XXXXXXXXXX")"
```

This will safely create a new temporary file in a directory defined by the `$TMPDIR` environment variable if it's defined,
or `/tmp` if it's not.

### Run With a Lower Priority

If you have a script that uses a lot of CPU,
you can set its niceness with `renice`.
Higher niceness values give a process lower scheduling priority.

```bash
# change niceness of this script
renice 10 -p $$
```

### Generate a Seeded Random Number

Suppose you have a script that runs daily from cron,
and it needs to perform some cleanup task once per week.
If you wanted the task to happen on a day that varies from system to system, you could do

```bash
# clean up once per week
RANDOM=$(( 16#$(hostname -f | md5sum | cut -c1-4) ))
cleanup_day=$(RANDOM % 7)
if (( "$(date +%u)" == "$cleanup_day" )); then
	# do cleanup task
fi
```

Note that this is only a pathological example meant to demonstrate the technique.
It would be simpler to schedule the cleanup task as a separate cron job in this scenario.
But if you need to schedule such a cron job from a configuration management system, for example,
you could use this technique to generate the day of the week
(or the day of the month, the hour, the minute, etc.) to schedule it on.

### TODO

* Test
* Proofread

