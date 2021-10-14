---
layout: post
last_modified_at: 2019-11-25 01:07:03 UTC
---

How to suppress rkhunter false positives due to zypper package updates on openSUSE 15.3.

1. TOC
{:toc}

### Introduction

Whenever a package is installed, updated, or removed with zypper,
we can store the package name with the help of a zipper plugin I wrote called
`zypp-plugin-post-commit-actions`.

We can also set up a daily cron job that takes those stored package names
and runs `rkhunter --propupd $package_name`.
Running rkhunter in this way updates the rkhunter database for files included in the RPM package named `$package_name`,
so if we set up the cron job to run before rkhunter's daily run,
we will no longer get false positives like below for files that were changed due to a package update:

    Warning: The file properties have changed:
             File: /bin/systemctl
             Current inode: 133679    Stored inode: 71681
    Warning: The file properties have changed:
             File: /usr/bin/curl
             Current inode: 135284    Stored inode: 72525

Credit goes to Danila Vershinin for originally showing how to set this up on CentOS 7:
https://www.getpagespeed.com/server-setup/security/sane-use-of-rkhunter-in-centos-7

The problem I had with his setup is that I couldn't find anything for SUSE that did what
`yum-plugin-post-transaction-actions` does, so I had to write a zypper plugin for that.

### Setup

Install `zypp-plugin-post-commit-actions`: https://github.com/natewoodward/zypp-plugin-post-commit-actions

Create `/etc/zypp/post-actions.d/rkhunter.action` with this line:

    *:any:echo $name >> /var/lib/rkhunter/changed-packages.dat

Create a script, `/etc/cron.daily/0rkhunter`, with these contents:

    #!/bin/bash
    
    pkglist=/var/lib/rkhunter/changed-packages.dat
    touch "$pkglist"
    while read pkg; do
        /usr/bin/rkhunter --propupdate "$pkg" &>/dev/null
    done < "$pkglist"
    : > "$pkglist"

Make the script executable with `chmod +x /etc/cron.daily/0rkhunter` .

By naming this script `0rkhunter`,
we ensure that it runs before the daily rkhunter cron job,
`suse.de-rkhunter` .

If all goes well, you shouldn't see false positives due to package updates anymore.

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

