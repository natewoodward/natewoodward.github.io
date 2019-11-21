---
layout: post
---

Here's a few patterns I use to control a `bash` script's output.

1. TOC
{:toc}

### Redirect All Output to a Log File

Suppose you're writing a script called `myscript` and want to send its output to a log file rather than to the console whenever it's run.
Running `myscript &>>myscript.log` every time would be cumbersome and easy to forget,
so let's do this instead:

```bash
# some setup we'll use in other examples
prog="${0##*/}"
logfile="$prog.log"

# send all output to log file
exec &>> "$logfile"
# alternatively:
#exec &> "$logfile"
```

The `exec &>> "$logfile"` here will redirect stdout and stderr of your script,
appending all of your script's output to `$logfile` without overwriting its existing contents.
You can use `&>` instead of `&>>` to overwrite `$logfile`,
effectively replacing its contents with your script's output.

Quick aside about the declaration of `$prog` above:
`${0##*/}` is identical to `$(basename "$0")` except that the former doesn't fork a new process.
<!--
More specifically, `${0##*/}` is a type of parameter expansion,
and parameter expansions typically run faster than process forks.
Forking a process over and over in a tight loop can add up pretty quickly and bog a script down,
so I try to avoid it when possible.
On the other hand, using `basename` is a lot easier to read,
so overall I think there's a case to be made for using either of them.
-->

### Conditionally Send Output Elsewhere

Let's look at one way we can extend the previous example.
Suppose you want your script to output to the terminal when you call it normally from the command line,
but when it's run from cron you want it to output to a log file instead.
In that case, you can do the following.

Note that I'm using `$logfile` from the previous example without declaring it as I did in the previous example.
For the rest of this article, I'll assume that `$prog` and `$logfile` have already been declared.

```bash
# check if stdout isn't a terminal
if [[ ! -t 1 ]]; then
	# send all output to log file
	exec &>> "$logfile"
fi
```

Just like it says in the comments,
this checks if stdout is a terminal and sends all output to `$logfile` if it's not.
So it will always send its output to `$logfile` when run from cron or launched from another daemon process.
And it will send its output to the terminal if you call it from the command line in most, but not all cases.
Consider what happens in the following examples:

```bash
myscript            # output to terminal
( myscript )        # output to terminal -- subshells don't affect stdout
echo | myscript     # output to terminal -- stdin is a pipe, but stdout is still a terminal
myscript 2> foo.txt # output to terminal -- stderr is a file, but stdout is still a terminal
myscript < foo.txt  # output to terminal -- stdin is a file
myscript > foo.txt  # output to $logfile -- stdout is a file
myscript | grep .   # output to $logfile -- stdout is a pipe
```

As you can see by reading the comments or by testing the commands yourself from a shell,
it's stdout that matters when it comes to reasoning about how the command will function.
If that behavior isn't what's desired,
you can try using `-t 0` instead of `-t 1` to detect whether stdin is a terminal.
Consider what happens in that case, and try testing it out in a shell if you're unsure.

Ultimately, using `[[ -t $fd ]]` in this way is a heuristic for determining whether a human is likely to see our output or not,
so it's important to understand its limitations.
While it's useful in scripts that we expect to call in simple ways,
for a script that's designed to do one thing and do it well,
we often want to be able to pipe data into and/or out of it in a lot of arbitrary ways.
In that case this pattern becomes an anti-pattern since its behavior can be unintuitive,
and you're better off implementing an option that changes where the script sends its output.

### Duplicate All Output to a Log File

If you want to send all script output to *both* your terminal and a log file,
you can wrap your script in `{ }` and pipe all of its output into `tee` like so:

```bash
# copy all script output to log file
{

# your script here

} 2>&1 | tee -a "$logfile"
```

If you want to overwrite `$logfile` rather than append to it, omit the `-a` flag in the `tee` command.

Note that, since both stderr and stdout are piped into `tee`,
if your script had a command like e.g. `echo foo >&2` that sends output to stderr,
that output will show up on your script's stdout instead.

### Send All Output to Syslog

To redirect all script output to syslog, you can do something like:

```bash
# redirect all output to syslog
exec &> >(logger -e -t "$prog" --id=$$ -p local0.info)
# with older versions of util-linux:
#exec &> >(grep -v '^$' | logger -t "$prog" -i -p local0.info)
```

This uses process substitution to send stdout and stdin to `logger`, which sends our output to syslog.
The `-e` option to logger prevents empty lines from being logged --
in most cases we probably don't care to store empty log messages.
The `--id` option logs our script's process id.
If you're unfamiliar with the other `logger` flags, you can look them up with `man logger`.

On systems with older versions of util-linux, `logger` might not have the `-e` and `--id` flags.
In that case we can do that next best thing, using `grep` to suppress empty lines
and `-i` to use the `logger` process's PID instead of our script's PID.

### Send All Output in an Email

For simple scripts,
this can be done with process substitution.
We'll just redirect our output to `sendmail`,
making sure that the first thing we output is the necessary email headers.

```bash
# redirect all output to sendmail
exec &> >(sendmail -t)

# print headers for notification email
cat <<-EOF
	Subject: Output of $prog
	To: somebody@example.com
	From: $(whoami)@$(hostname -f)

EOF
```

In many cases you only want to deliver mail when your script fails, though.
In order to do that, I've started using a wrapper script on systems that I'm responsible for.
The script is called
[`mailfail`](https://raw.githubusercontent.com/natewoodward/code-snippets/master/bin/mailfail), and you can
[view it on GitHub](https://github.com/natewoodward/code-snippets/blob/master/bin/mailfail).

To use it, you call it with a comma-delimited list of email addresses and the command you want to run, like so:

```bash
mailfail someone@example.com,somebody@example.com /some/command --that -might fail
```

For more detailed usage info, run `mailfail --help`.

You might need to inspect the command's output on occasion if system mail isn't configured correctly or
if you just need to inspect the output of a successful command that wasn't emailed.
So `mailfail` always logs the script's output to a file in addition to sending an email notification on failure.

For cron jobs, another way you can approach this problem is to configure cron to send out emails if a job has any output,
and design your scripts so that they never output anything during a successful run.
If you already have a lot of scripts that weren't designed that way,
it can be a lot of work to implement, though.

### Output Log File Data Until A Message or Timeout is Reached

It's not unheard of to give a developer access to restart a service through `sudo` and a restart script.
On a couple occasions I've needed to do this with a service that takes a long time to start up,
so to give the user feedback as to what's going on while the service starts,
I had the script `tail` the service's log file until some startup message is reached.
In order to prevent the script from hanging indefinitely in case the service never logs the startup message I'm expecting,
I also used a backgrounded `sleep` process to act as a timer.

Here's what that looks like:

```bash
# output log file until pattern or timeout is reached

# define some variables
pattern='Server startup' # regex that matches the service's startup message
timeout=300              # number of seconds to wait for service to start

# launch a process in the background that will time out eventually
sleep "$timeout" &
timerpid=$!

# kill the timeout process if we exit abnormally
trap "kill '$timerpid' 2>/dev/null" EXIT

# output logfile until we see $pattern
tail -Fn0 --pid "$timerpid" "$logfile" 2>/dev/null | {
	sed -r "/$pattern/ q"
	kill "$timerpid" 2>/dev/null
} &

# restart the service
service tomcat restart

# wait until the backgrounded timeout process exits
{
	wait "$timerpid"
} &>/dev/null
```

As you can see, this is a lot of complexity to add to a script,
so I recommend looking into other options before resorting to it.

### TODO

* Verify the examples work
* Move `mailfail` code to gist/github
* Make `mailfail` work when not run as root
* Explain service restart snippet better
* Cheatsheet?
* Common end sections -- Acknowledgements, References, Further Reading

<!--
### References
### Further Reading
### Acknowledgements
-->

