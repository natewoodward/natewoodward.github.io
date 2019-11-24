---
layout: post
---

I use a `$PROMPT_COMMAND` that prints a yellow smiley face if the last command was successful,
and a blue crying face if the last command failed.
It's helped my scripting skills a lot over the years by making nonzero exit statuses more discoverable.

For example, did you know that `diff` can be used to test for "sameness" in two files?
When run against two files, it returns `0` if they're the same and `1` if they're different.
That said, [`cmp` is better at detecting idential files](https://stackoverflow.com/questions/12900538/fastest-way-to-tell-if-two-files-are-the-same-in-unix-linux).

Anyway, here's my `$PROMPT_COMMAND` in action.

<img src="/img/prompt-command.png" alt="prompt command" title="^_^\nT_T"/>

And the code:

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

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

