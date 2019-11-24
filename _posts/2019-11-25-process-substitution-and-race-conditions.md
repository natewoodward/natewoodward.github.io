---
layout: post
last_modified_at: 2019-11-24 19:17:19 UTC
---

In a recent article, I showed how to
[print a script's output to both a log file and stdout]({{ site.url }}/blog/2019/11/22/bash-output-patterns#duplicate-all-output-to-a-log-file).
Some of you might be wondering why I used a `{ group command; }` piped into `tee` for this,
rather than `>(process substition)`. Let me explain.

The solution I offered in that post has a downside in that it redirects stderr to stdout.
[This StackOverflow answer](https://stackoverflow.com/a/11886837)
tries to get around that limitation by using separate `tee` processes to handle stdout and stderr,
but it introduces a new problem.
Let's see what happens when we run their example script:

    ^_^ woody@beemo:~$ myscript 
    ^_^ woody@beemo:~$ foo
    bar

It might look like I typed `foo` into the terminal above,
but what actually happened is `myscript` exited,
the shell printed `$PS1` to indicate it was ready for me to type another command on stdin,
then the first `tee` process from `myscript` printed the string "foo" on stdout after the prompt,
and finally the second `tee` process printed "bar" on stderr on the next line.
This is because commands inside the `>( )` of a process substition run asynchronously,
so it's possible for the aynchronously-running `tee` processes to continue running and printing their output after our script exists.
It's a classic race condition.

We can use information from
[another StackOverflow answer](https://unix.stackexchange.com/a/524844)
to address the race condition by forcing our script to wait for the `tee` processes to end.
Here's what that looks like:

```bash
#!/bin/bash

# global vars
prog="${0##*/}"
syncfifo="${TMPDIR-/tmp}/$prog.pipe"

# create fifo
mkfifo "$syncfifo" || exit 1
trap "rm '$syncfifo'" EXIT

# output to stdout and log file
exec  > >(tee -ia "$prog.log";     echo >"$syncfifo")
exec 2> >(tee -ia "$prog.log" >&2; echo >"$syncfifo")

# your script here
echo foo
echo bar >&2

# close stdout and stderr so that the tee processes will exit
exec >&-
exec 2>&-

# wait for tee to exit
read line < "$syncfifo"
read line < "$syncfifo"
```

Seems pretty complicated, but if it works it works, right?
Unfortunately, if you run this script enough times, you'll see output like this:

    ^_^ woody@beemo:~$ myscript 
    foo
    bar
    ^_^ woody@beemo:~$ myscript 
    bar
    foo
    ^_^ woody@beemo:~$ myscript 
    foo
    bar
    ^_^ woody@beemo:~$ 

Here we've exposed another problem.
In addition to the race condition between our script and the `tee` processes,
there's also a race condition between the two `tee` processes,
so sometimes it prints `"foo\nbar"` and other times it prints `"bar\nfoo"`.

[The accepted answer](https://stackoverflow.com/a/3403786) from the first SO link only uses one `tee` process,
so it doesn't have that problem.
But it *does* have the same race condition between the script and `tee` that we discussed before.
We can solve that problem similarly to how we solved it above for the script with two `tee` processes, like so:

```bash
#!/bin/bash

# global vars
prog="${0##*/}"
syncfifo="${TMPDIR-/tmp}/$prog.pipe"

# create fifo
mkfifo "$syncfifo" || exit 1
trap "rm '$syncfifo'" EXIT

# output to stdout and log file
exec &> >(tee -ia "$prog.log";     echo >"$syncfifo")

# your script here
echo foo
echo bar >&2

# close stdout and stderr so that the tee process will exit
exec >&-
exec 2>&-

# wait for tee to exit
read line < "$syncfifo"
```

But ultimately, that solution adds a lot of complexity to our script and no benefit compared to just doing this:

```bash
#!/bin/bash

# global vars
prog="${0##*/}"

{

# your script here
echo foo
echo bar >&2

} 2>&1 | tee -a "prog.log"
```

So that's my preferred solution.

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->


