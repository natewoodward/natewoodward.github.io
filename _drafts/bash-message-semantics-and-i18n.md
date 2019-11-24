---
layout: post
last_modified_at: 2019-11-24 15:21:26 UTC 
title: Bash Message Semantics and i18n
---

Description of article.

1. TOC
{:toc}

### Section

Section contents.

```bash
#!/bin/bash

# prints an info message to stdout
msg() {
	local message="$1"
	shift
	printf "$message\n" "$@"
}

# prints a warning message to stderr
warn() {
	local message="$1"
	shift
	printf "$message\n" "$@" >&2
}

# prints an error message to stderr
err() {
	local message="$1"
	shift
	printf "$message\n" "$@" >&2
}
```

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

