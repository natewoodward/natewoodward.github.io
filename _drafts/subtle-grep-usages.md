---
layout: post
---

I'd like to share a couple of non-obvious ways to use `grep` that I've come across over the years.
I'll do this using some benchmarks from the CIS CentOS 7 Linux hardening guidelines as examples.

1. TOC
{:toc}

### Exclude Files

```bash
# implement an exclude file where the user can configure files to ignore
excludefile="/usr/local/etc/rpm_integrity.exclude"
if [[ ! -f "$excludefile" ]]; then
    excludefile=/dev/null
fi

# print files that fail an rpm integrity check,
# ignoring config files and user-excluded files
rpm -Va --nomtime --nosize --nomd5 --nolinkto --noghost \
    | grep -vFf "$excludefile" \
    | awk '$2 != "c" {print}'
```

### Check if a Command has Output

