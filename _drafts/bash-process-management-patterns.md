---
layout: post
---

Let's look at a few patterns I use to manage processes in `bash`.

```bash
#!/bin/bash

# global vars
prog="${0##*/}"

# print error message to stderr
err() {
	local msg="$1"
	shift
	printf "$msg\n" "$@"
}

# drop privileges
user=nobody
if (( EUID == 0 )); then
	exec runuser -u "$user" "$0" "$@"
	#exec su "$user" -c "$(printf %q "$0") $(printf %q "$*")"
if [[ $(id -u) != "$user" ]]; then
	err "This script must be run as %s or %s" 'root' "$user"
	exit 1
fi

# try to obtain lock
lockfile="${TMPDIR-/tmp}/$prog.lock"
if (set -o noclobber && echo $$>"$lockfile") 2>/dev/null; then
	trap "rm -f '$lockfile'" EXIT
else
	err "Lock %s held by pid %s" "$lockfile" "$(<"$lockfile")"
	exit 1
fi

# create a temporary file
trap "rm -f '$tmpfile'" EXIT
tmpfile="$(mktemp --tmpdir "$prog.XXXXXXXXXX")"

# change niceness of this script
renice 10 -p $$

## generate a system-specific random number
# pick a day of the week to prune backups and do an integrity check during the 6pm backup
#
# we want our integrity checks to be spread out throughout the work week,
# so make each system pick a number that's consistent between runs of this script,
# but different from system to system
#
# to do this, we use the system's hostname to seed $RANDOM and use it to pick a weekday
RANDOM=$(( 16#$(hostname -f | md5sum | cut -c-4) ))
integrityday=$((RANDOM % 5 + 1))
# format as "day-hour" for comparison later
#
#        %u     day of week (1..7); 1 is Monday
#        %k     hour ( 0..23)
now="$(date +"%u-%k")"
integrityhour="$integrityday-18"
```

