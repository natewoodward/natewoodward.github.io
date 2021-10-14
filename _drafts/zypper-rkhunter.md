---
layout: post
last_modified_at: 2021-10-14 21:54:53 UTC
title: Update rkhunter After SUSE Package Updates
---

How to suppress rkhunter false positives due to zypper package updates on openSUSE 15.3.

1. TOC
{:toc}

### Setup

[Install `zypp-plugin-post-commit-actions`](https://github.com/natewoodward/zypp-plugin-post-commit-actions).

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

If all goes well, you'll no longer get false positives like below for files that were changed due to a package update:

    Warning: The file properties have changed:
             File: /bin/systemctl
             Current inode: 133679    Stored inode: 71681
    Warning: The file properties have changed:
             File: /usr/bin/curl
             Current inode: 135284    Stored inode: 72525

### How it Works

Whenever a package is installed, updated, or removed with zypper,
we store the package name in `/var/lib/rkhunter/changed-packages.dat`
with the help of the zipper plugin.

Before the system cron job (`suse.de-rkhunter`) for rkhunter runs,
our `0rkhunter` cron script takes the stored package names
and runs `rkhunter --propupd $package_name` against each one.
Running rkhunter in this way updates the rkhunter database for files included in the RPM package named `$package_name`.
It does so using the file attributes from RPM's database,
not from the the filesystem.

[Credit goes to Danila Vershinin](https://www.getpagespeed.com/server-setup/security/sane-use-of-rkhunter-in-centos-7)
for originally showing how to set this up on CentOS 7.
The only problem I had with his setup is that I couldn't find anything for SUSE that did what
`yum-plugin-post-transaction-actions` does, so I had to write a zypper plugin for that.

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

