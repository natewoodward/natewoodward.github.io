---
layout: post
last_modified_at: 2022-08-09 18:08:14 UTC
title: Update rkhunter After SUSE Package Updates
---

How to suppress rkhunter false positives due to zypper package updates on openSUSE 15.4.

[Danila Vershinin's blog post](https://www.getpagespeed.com/server-setup/security/sane-use-of-rkhunter-in-centos-7)
explains why one might want to do this and has instructions for CentOS 7.
The only problem I had with his setup is that I couldn't find anything for SUSE that did what
`yum-plugin-post-transaction-actions` does, so I had to write a zypper plugin for that.

1. TOC
{:toc}

### Setup

The daily system cron job to run rkhunter was removed in openSUSE 15.4,
so you will need to set one up yourself.
The files for the cron job that openSUSE 15.3 used are still available,
but were moved under /usr/share,
so we can simply copy those files to set up the job.

    cp /usr/share/fillup-templates/sysconfig.rkhunter /etc/sysconfig/rkhunter
    cp /usr/sharedoc/packages/rkhunter-1.4.6/rkhunter.cron /etc/cron.daily/suse.de-rkhunter
    chmod +x /etc/cron.daily/suse.de-rkhunter

You may also want to edit `/etc/sysconfig/rkhunter` and adjust
`CRON_DB_UPDATE`, `REPORT_EMAIL`, etc.

[Install `zypp-plugin-post-commit-actions`](https://github.com/natewoodward/zypp-plugin-post-commit-actions).

Create `/etc/zypp/post-actions.d/rkhunter.action` with this line:

    *:any:echo $name >> /var/lib/rkhunter/changed-packages.dat

Create a script, `/etc/cron.daily/0rkhunter`, with these contents:

    #!/bin/bash
    
    pkglist=/var/lib/rkhunter/changed-packages.dat
    touch "$pkglist"
    while read pkg; do
        /usr/bin/rkhunter --propupdate "$pkg" &>/dev/null
    done < <(sort -u "$pkglist")
    : > "$pkglist"

Make the script executable with `chmod +x /etc/cron.daily/0rkhunter` .

If all goes well, you'll no longer get false positives like below for files that were changed due to a package being updated, installed, or removed:

    Warning: The file properties have changed:
             File: /bin/systemctl
             Current inode: 133679    Stored inode: 71681
    Warning: The file properties have changed:
             File: /usr/bin/curl
             Current inode: 135284    Stored inode: 72525

### How it Works

Whenever a package is installed, updated, or removed with zypper,
we store the package name in `/var/lib/rkhunter/changed-packages.dat`
with the help of the zypper plugin.

Before the system cron job (`suse.de-rkhunter`) for rkhunter runs,
our `0rkhunter` cron script takes the stored package names
and runs `rkhunter --propupd $pkg` against each one.
Running rkhunter in this way updates the rkhunter database for files included in the RPM package named `$pkg`.
It does so using the file attributes from RPM's database,
not from the the filesystem.

See [Danila Vershinin's blog post](https://www.getpagespeed.com/server-setup/security/sane-use-of-rkhunter-in-centos-7)
for a more thorough explanation of the pieces at play here.

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

