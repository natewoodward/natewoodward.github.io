---
layout: post
last_modified_at: 2019-11-25 01:07:03 UTC
---

Learning the `bash` shell's interactive features will make you more efficient at interacting with your system.
These are the ones I use the most.

### Tab Completion

This should almost go without saying,
but I know some people who still don't use tab completion to I figure it bears repeating.

*description of how tab completion works*

*using tab completion instead of ls*

*using the concept of "work directories" to avoid excessive cd'ing*

### Prompt Command

I use a `$PROMPT_COMMAND` that prints a smiley face if the last command was successful,
and a crying face if the last command failed.
It's helped my scripting skills a lot over the years by making nonzero exit statuses more discoverable.

For example, did you know that `diff` can be used to test for the "sameness" of files?
I sure didn't until my prompt starting crying about two files being different.
When run against two files, `diff` returns zero if they're the same and nonzero if they're different.
That said, [`cmp` is better at detecting if files are identical](https://stackoverflow.com/questions/12900538/fastest-way-to-tell-if-two-files-are-the-same-in-unix-linux).

Anyway, here's my `$PROMPT_COMMAND` in action.

<img src="/img/prompt-command.png" alt="prompt command" title="^_^\nT_T"/>

And the code from `~/.bashrc`:

```bash
prompt-command() {
	if (( $? )); then
		local face='\[\e[0;36m\]T_T\[\e[0m\]'
	else
		local face='\[\e[0;33m\]^_^\[\e[0m\]'
	fi

	local me host dir prompt_color end_color

	me='\u'
	host='\h'
	dir='\W'
	prompt_color='\[\e[1;32m\]'
	end_color='\[\e[0m\]'
	export PS1="${face} ${prompt_color}${me}@${host}${end_color}:${dir}\\$ "
}
export PROMPT_COMMAND="prompt-command"
```

### Shortcuts

#### Copying and Pasting

Ctrl+U, Ctrl+W, Alt+Backspace, Ctrl+Y

#### Screen Control

Ctrl+L, Ctrl+S, Ctrl+Q, Shift+PgUp, Shift+PgDn

#### Others

Ctrl+R

Ctrl+Z plus `bg`, `fg`

### History Expansion

`!*`, `!!`, not so much `!!:gs/search/replace` or `^search^replace`

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

