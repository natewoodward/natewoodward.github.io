---
layout: post
---

Here's a few patterns I use to send a `bash` script's output to a log file,
to syslog, or as an email notification.

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

The `exec &>> "$logfile"` here will redirect your script's stdout and stderr,
appending all of your script's output to `$logfile` without overwriting its existing contents.
You can use `&>` instead of `&>>` to overwrite `$logfile`,
effectively replacing its contents with your script's output.

<!--
Quick aside about the declaration of `$prog` above:
`${0##*/}` is identical to `$(basename "$0")` except that the former doesn't fork a new process.
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
Consider what happens in that case if you were to run the examples above. Try testing it out in a shell if you're unsure.

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

<!--
If you want to overwrite `$logfile` rather than append to it, omit the `-a` flag in the `tee` command.
-->

Note that, since both stderr and stdout are piped into `tee`,
if your script had a command like e.g. `echo foo >&2` that sends output to stderr,
that output will show up on your script's stdout instead.

### Send All Output to Syslog

To redirect all script output to syslog, you can do something like:

```bash
# redirect all output to syslog
exec &> >(logger -e -t "$prog" --id=$$ -p local0.info)
# with older versions of util-linux:
#exec &> >(grep -v '^$' | logger -t "$prog" -p local0.info)
```

This uses process substitution to send stdout and stdin to `logger`, which sends our output to syslog.
The `-e` option to logger prevents empty lines from being logged --
in most cases we probably don't care to store empty log messages.
The `--id` option logs our script's process id.
If you're unfamiliar with the other `logger` flags, you can look them up with `man logger`.

On systems with older versions of util-linux, `logger` might not have the `-e` and `--id` flags.
In that case we can do that next best thing, using `grep` to suppress empty lines,
and just living without the PID in our syslog messages.
You can use `-i` to pass the logger process's PID instead,
but I worry that could be a point of confusion for anyone reading the logs who's not familiar with the script,
so I don't do that.

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
[`mailfail`](https://raw.githubusercontent.com/natewoodward/code-snippets/master/bin/mailfail),
and it's
[available on GitHub](https://github.com/natewoodward/code-snippets/blob/master/bin/mailfail).

To use the script, you call it with a comma-delimited list of email addresses and the command you want to run, like so:

```bash
mailfail someone@example.com,somebody@example.com /some/command --that -might fail
```

For more detailed usage info, run `mailfail --help`.

You might need to inspect the command's output on occasion if system mail isn't configured correctly,
or if you just need to inspect the output of a successful command.
So `mailfail` always logs the script's output to a file in addition to sending an email notification on failure.

For cron jobs, another way you can approach this problem is to configure cron to send out emails if a job has any output,
and design your scripts so that they never output anything during a successful run.
<!--
If you already have a lot of scripts that weren't designed that way,
changing them to accomodate that setup can be a lot of work, though.
-->

### Output Log File Data Until A Message or Timeout is Reached

It's not unheard of to give a developer access to restart a service through `sudo` and a restart script.
On a couple occasions I've needed to do this with a service that takes a long time to start up,
so to give the user feedback as to what's going on while the service starts,
I had the script `tail` the service's log file until some startup message is reached.
In order to prevent the script from hanging indefinitely in case the service never logs the startup message I'm expecting,
I used a combination of `sleep`, `wait`, and `kill`.

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

Let's unpack what's going on here:

1. First we define some variables. No surprises here.
1. Then we start a `sleep` process in the background. Other child processes of our script will end if this process terminates.
1. Set a trap to `kill` the sleep process if our script exits early. This ensures our child processes are cleaned up sooner rather than later.
1. Then we launch a pipeline in the background.
   * First, we `tail` the log file, passing the following options to tail:
     * `-F`: Follow the log file by name, rather than by file descriptor. That way if the service rotates the logfile when it restarts, we'll find the new log file and continue printing messages from it.
     * `-n0`: Follow the log file starting where it currently ends. We don't need the context of the previous 10 log messages.
     * `--pid $timerpid`: If the sleep process exits, so does the tail process. So we'll stop outputting the log file after `$timeout` seconds at most, or sooner if the sleep process is killed early.
   * We pipe tail's output to a `sed` command that prints the output until it reaches the service's startup message. Then it exits, and
   * We kill the sleep process.
   * Also, we run this whole pipeline in the background.
1. In other words, the `tail` pipeline and `sleep` process interact to print the service's log file until either the startup message is printed, or a timeout is reached, whichever happens first. And all of this happens in the background, because once we're done setting this up we need to:
1. Restart the service.
1. Then, we wait until the sleep process is killed or exits.

As you can see, this is a lot of complexity to add to a script,
so I recommend looking into other options before resorting to it.

<!--
### Footnotes

[^1]: More specifically, `${0##*/}` is a type of parameter expansion,
      and parameter expansions typically run faster than process forks like `$(basename "$0")`.
Forking a process over and over in a tight loop can add up pretty quickly and bog a script down,
and you never know when a script might be used in *another* script's tight loop,
so I try to avoid forking when possible.
On the other hand, using `basename` is a lot easier to read,
so overall I think there's a case to be made for using either of them.
On the other *other* hand, you only need `${0##*/}` explained to you once before you grok it,
so my personal preference is that it's objectively better.

[^2]: There are
      [ways around this](https://stackoverflow.com/a/11886837)
but
[they're](https://unix.stackexchange.com/questions/524811/not-being-set-to-the-pid-of-a-process-substitution-used-with-an-exte)
all
[terrible](https://unix.stackexchange.com/questions/388519/bash-wait-for-process-in-process-substitution-even-if-command-is-invalid).
If you're not sure what I mean by this,
try making the SO answer in the first link work in a sane way, without any race conditions or relying on undocumented `bash` behavior.

[^3]: When I say that `tail` is "first" here,
    I only mean that it's the first command in the pipeline when read top to bottom, left to right.
As many of you are well aware, processes in the same pipeline run concurrently.
-->
