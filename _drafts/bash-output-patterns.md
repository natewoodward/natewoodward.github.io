---
layout: post
---

Here's a few patterns I use to control a `bash` script's output.

1. TOC
{:toc}

### Redirect All Output to a Log File

Suppose you're writing a script called `myscript` and want it to always send all of its output to a log file rather than to the console.
It would be cumbersome to have to remember to always run `myscript &>myscript.log`,
so instead let's do this:

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

Quick aside about the example above:
`${0##*/}` is identical to `$(basename "$0")` except that the former doesn't fork a new process.

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
So it will always send its output to `$logfile` when run from cron or launched from some other daemon process.
And it will send its output to the terminal if you call it from the command line in most, but not all cases.
Consider what happens in the following examples:

```bash
myscript            # output to terminal
( myscript )        # output to terminal -- launching myscript from a subshell doesn't affect stdout
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
While it's useful in "driver" scripts that we expect to use in a limited number of ways,
for a script that's designed to do one thing and do it well,
we often want to be able to pipe data into and/or out of it in a lot of arbitrary ways.
In that case this pattern becomes an anti-pattern since its behavior can be unintuitive,
and you're better off implementing a flag that changes where the script sends its output.

That said, if you're writing a script where this pattern is useful,
you can tweak it with the patterns below to modify how a script behaves, too.
Rather than send all output to a file when run from cron,
you could instead send all output to syslog or to an email,
for example, when not run in a terminal.

### Duplicate All Output to a Log File

If you want to send all script output to *both* your terminal and a log file, you can do

```bash
# copy all output to logfile
exec &>> >(tee -ia "$logfile")
```

This example uses process substitution to send stdout and stderr to `tee`,
which will append output to `$logfile` and also to stdout.
If you want to overwrite `$logfile` rather than append to it, omit the `-a` flag for `tee`.

#### Limitations of This Pattern

Note that in that, since stderr is redirected to stdout,
if your script had a command like e.g. `echo foo >&2` that sends output to stderr,
that output will show up on your script's stdout instead. I found
[an answer on StackOverflow](https://stackoverflow.com/questions/3173131/redirect-copy-of-stdout-to-log-file-from-within-bash-script-itself)
that tries to work around this limitation,
but in addition to the note about unbuffered `tee` output in that answer,
the script's stdout and stderr are handled by two `tee` processes running asynchronously,
which has some unfortunate drawbacks.
Let's see what happens when we run their example script:

```bash
prompt> myscript 
prompt> foo
bar
```

Both `tee` processes wrote their output *after* the script exited,
so what you see is the shell indicating that it's ready for me to enter a new command by printing `prompt> `,
followed by the first `tee` process printing `foo` after the prompt,
and then the second `tee` process printing `bar`.
Let's try to work around that by adding `sleep 0.001` to the end of the script and running it a couple more times:

```bash
prompt> myscript 
foo
bar
prompt> myscript 
bar
foo
prompt> myscript 
bar
foo
```

Now the order of `foo` and `bar` in the output changes arbitrarily between runs of the script.
Since the `tee` processes run asynchronously, messages on stdout and stderr can appear out of order.
In scripts that I write, I almost always want precise control over how messages appear to the user,
so I consider this solution to be an anti-pattern.
So although the first example I gave for duplicating script output to a file prevents us from meaningfully using stderr,
I find that's a much easier trade-off to make for the kinds of scripts that need this functionality.

### Send All Output to Syslog

To redirect all script output to syslog, you can do something like:

```bash
# redirect all output to syslog
exec &>> >(logger -e -t "$prog" --id=$$ -p local0.info)
```

This sends stdout and stdin to `logger`, which then sends our output to syslog.
The `-e` option to logger prevents empty lines from being logged --
in most cases we probably don't care to store empty log messages.
If you're unfamiliar with the other `logger` flags, you can look them up with `man logger`.

To duplicate script output to syslog rather than simply redirecting it there,
you can combine this pattern with the one for duplicating output to a file:

```bash
# copy all output to syslog
exec &>> >(tee -i >(logger -e -t "$prog" --id=$$ -p local0.info) )
```

### Send All Output in an Email

If your needs are simple,
this can be done similarly to the other examples we've seen involving process substitution.
We can send our output to `sendmail` and let it handle email delivery,
but we need to make sure that the first thing we output are the necessary email headers.

```bash
# redirect all output to sendmail
exec &>> >(sendmail someone@example.com)

# print headers for notification email
echo "Subject: Output of $prog"
echo "From: $(whoami)@$(hostname -f)"
echo
```

In many cases you only want to deliver mail when some part of your script fails, though.
In order to do that, you can write the script's output to a temporary file and email the output at the end of the script.

```bash
# create a temporary file
trap 'rm -f "$tmpfile"' EXIT
tmpfile="$(mktemp --tmpdir "$prog.XXXXXXXXXX")"

# send all output to temporary file
exec &>> "$tmpfile"

# print headers for notification email
echo "Subject: Output of $prog"
echo "From: $(whoami)@$(hostname -f)"
echo

# define a trap command for cleaning up after failure
trapfail='sendmail someone@example.com < "$tmpfile"; rm -f "$tmpfile"' 

# do something that might or might not fail
true || trap "$trapfail" EXIT

# do another thing that might fail
false || trap "$trapfail" EXIT
```

### Output Log File Data Until A Message or Timeout is Reached

It's not unheard of to give a developer access to safely restart a server through `sudo` and a restart script.
On a couple occasions I've needed to do this with a server that takes a long time to start up,
so to give the user feedback as to what's going on while the server starts,
I had the script `tail` the server's log file until some startup message is reached.
In order to prevent the script from hanging indefinitely in case the server never logs the startup message I'm expecting,
I also used a backgrounded `sleep` process to act as a timer.

Here's what that looks like:

```bash
# output log file until pattern or timeout is reached

# define some variables
pattern='Server startup' # extended regex that matches the server's startup message
timeout=300              # the time to wait for the server to start up, in seconds

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

# restart the server
service tomcat restart

# wait until the backgrounded timeout process exits
{
	wait "$timerpid"
} &>/dev/null
```

As you can see, this is a lot of complexity to add to a script,
so I advise looking into other options before resorting to it.

### TODO

* Verify the examples work
* Explain email, log file output more
* Troubleshoot sendmail vs mutt headers
* Cheatsheet
* Common end sections -- Acknowledgements, References, Further Reading

