---
layout: post
title: Monitoring Java Memory Usage with Jstat and Check MK
last_modified_at: 2019-11-28 17:49:59 UTC
---

I recently wrote a script called
[`check_jvm_memory`](https://github.com/natewoodward/code-snippets/tree/master/check-mk)
to monitor JVM memory usage with `jstat` and Check MK.
Let's go over how to use it.

1. TOC
{:toc}

### Requirements

The script runs as a [local check](https://checkmk.com/cms_localchecks.html),
so it requires the Check MK agent.

For obvious reasons, it requires that you have a running Java process that you want to monitor.

It requires `pidof` unless you configure the [`PidCommand`](#pidcommand) option.

It also requires `perl`,
and a version of `jstat` that's compatible with the Java process you're monitoring.

### Quick Setup

Grab
[`check_jvm_memory`](https://raw.githubusercontent.com/natewoodward/code-snippets/master/check-mk/check_jvm_memory)
from GitHub and drop it in the
[Check MK script directory](https://checkmk.com/cms_localchecks.html#folder)
on the host you want to monitor.
The path to the script directory varies depending on how the Check MK agent was set up.
On my systems, it's `/usr/share/check-mk-agent/local`.

If everything's set up right,
running the script should give you output that starts with a `P`, similar to this:

    P JVM_MEMORY.SurvivorSpace1 SurvivorSpace1=0.000000% SurvivorSpace1 0.0% (0.00 / 25.56 MB)
    P JVM_MEMORY.OldGen OldGen=26.603203% OldGen 26.6% (340.52 / 1280.00 MB)
    P JVM_MEMORY.SurvivorSpace0 SurvivorSpace0=1.325642% SurvivorSpace0 1.3% (0.34 / 25.56 MB)
    P JVM_MEMORY.EdenSpace EdenSpace=13.898290% EdenSpace 13.9% (28.47 / 204.88 MB)
    P JVM_MEMORY.MetaSpace MetaSpace=98.491438% MetaSpace 98.5% (249.63 / 253.45 MB)
    P JVM_MEMORY.CompressedClassSpace CompressedClassSpace=98.215967% CompressedClassSpace 98.2% (32.61 / 33.20 MB)

Once Check MK inventories the host,
you should be able to add the new service checks through WATO in the usual way.

### Install

I prefer to install the script to `/usr/local/bin/check_jvm_memory`,
then symlink it from the Check MK script directory using a name to indicate the service it checks.
For example, to monitor Tomcat I symlink the script from `/usr/share/check-mk-agent/local/300/check_tomcat_memory`.
Using the `300` subdirectory causes Check MK to run the script every 300 seconds (5 minutes),
and the `check_tomcat_memory` filename makes the script produce this output:

    P TOMCAT_MEMORY.SurvivorSpace1 SurvivorSpace1=0.000000% SurvivorSpace1 0.0% (0.00 / 25.56 MB)
    P TOMCAT_MEMORY.OldGen OldGen=26.603203% OldGen 26.6% (340.52 / 1280.00 MB)
    P TOMCAT_MEMORY.SurvivorSpace0 SurvivorSpace0=1.325642% SurvivorSpace0 1.3% (0.34 / 25.56 MB)
    P TOMCAT_MEMORY.EdenSpace EdenSpace=25.776531% EdenSpace 25.8% (52.81 / 204.88 MB)
    P TOMCAT_MEMORY.MetaSpace MetaSpace=98.491438% MetaSpace 98.5% (249.63 / 253.45 MB)
    P TOMCAT_MEMORY.CompressedClassSpace CompressedClassSpace=98.215967% CompressedClassSpace 98.2% (32.61 / 33.20 MB)

Since the script is called as `check_tomcat_memory` and not `check_jvm_memory`,
the service check names start with `TOMCAT_MEMORY` instead of `JVM_MEMORY`.
This behavior can be overriden by creating a configuration file at
`/etc/check-mk-agent/check_tomcat_memory.cfg`,
and including the [`Name`](#name) option described below.

I recommend setting a [`PidCommand`](#pidcommand) so that the script will always find the correct Java process to monitor if there are multiple Java processes running on the system.
You might also want to configure warning and critical thresholds for OldGen as described in the [`Threshold`](#threshold) section,
and for PermGen as well on Java 7 and under.

### Configure

`check_jvm_memory` looks for its configuration in the `/etc/check-mk-agent` directory.
The name of the config file is the name of the script plus a `.cfg` extension.
There's an example
[`check_jvm_memory.cfg`](https://raw.githubusercontent.com/natewoodward/code-snippets/master/check-mk/check_jvm_memory.cfg)
on GitHub.

Configuration options are described below.

#### Name

This lets you set a service check name that's not derived from the script's filename.
For instance, here's the script's output with `Name: TomcatMemory` set:

    P TomcatMemory.SurvivorSpace1 SurvivorSpace1=0.000000% SurvivorSpace1 0.0% (0.00 / 25.56 MB)
    P TomcatMemory.OldGen OldGen=26.603203% OldGen 26.6% (340.52 / 1280.00 MB)
    P TomcatMemory.SurvivorSpace0 SurvivorSpace0=1.325642% SurvivorSpace0 1.3% (0.34 / 25.56 MB)
    P TomcatMemory.EdenSpace EdenSpace=45.458359% EdenSpace 45.5% (93.13 / 204.88 MB)
    P TomcatMemory.MetaSpace MetaSpace=98.491438% MetaSpace 98.5% (249.63 / 253.45 MB)
    P TomcatMemory.CompressedClassSpace CompressedClassSpace=98.215967% CompressedClassSpace 98.2% (32.61 / 33.20 MB)

Note that `Name` can't contain spaces, or else Check MK won't be able to parse the script's output.

#### PidCommand

`check_jvm_memory` locates the process ID (pid) of the Java process it's monitoring using a command specified by the `PidCommand` option.
`check_jvm_memory` will look at the first line of output from this command,
find the first integer, and use that as the pid to pass to `jstat`.

The default `PidCommand` is `pidof java`.

This option is especially useful on systems that have multiple Java processes running on them.
On such systems, you can monitor each process with a differently-named `check_jvm_memory` script,
then configure a different `PidCommand` for each of those scripts.

Here are a few example commands you could use with this configuration option.
In every case, the script would monitor process ID `9170` since that's the first integer on the first line of output.

    root@beemo:~# pidof java
    9170 4978 4443
    root@beemo:~# service tomcat status
    tomcat (pid 9170) is running...                            [  OK  ]
    root@beemo:~# cat /var/run/tomcat.pid
    9170

#### Threshold

This option is used to set the threshold for each service check.
The value you set is used as the `;warn;crit;min;max` values for
[Check MK's metrics](https://checkmk.com/cms_localchecks.html#perfdata).
Read Check MK's documentation for a complete description of how metrics work.

To explain by way of example, suppose you want Check MK to put the OldGen and PermGen service checks into Warning state if they go over 98%,
and Critical state if they go over 99%.
You can do that by configuring these thresholds:

    Threshold OldGen:  ;98;99
    Threshold PermGen: ;98;99

#### Label

The `Label` config option lets you set a label that's used in the service name, metrics (a.k.a. perfdata), and status detail (a.k.a. description) of the check's output.
Most people won't ever need to use this configuration option.

The `Label` option is best explained by describing how `check_jvm_memory` does its thing.
Internally, it gets its data using a `jstat` command similar to the one shown below.
The output is a bit confusing, but to use a couple examples,
the `S0U` here indicates how much of **S**urvivor Space #**0** is **U**sed (in KB).
Similarly, `S0C` is **S**urvivor Space #**0**'s total **C**apacity.

    woody@beemo:~# jstat -gc 9170
     S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
    26176.0 26176.0 347.0   0.0   209792.0 178909.4 1310720.0   348693.5  259532.0 255616.8 33996.0 33389.5     84    3.658   5      2.046    5.703

After parsing that data, `check_jvm_memory` looks at every field header ending with a `U`,
looks up the matching header that ends in `C`,
and uses the two values for those columns to compute the percent usage of that area of memory.

If there were no labels set,
the script would report that `S0` has a usage of `1.33%` in the example above,
which isn't very helpful -- what the heck is `S0`?
To make the output more human friendly,
you can configure a label for `S0` so that the script reports `1.33%` usage for `SurvivorSpace0` instead:

    Label S0:  SurvivorSpace0

Internally, the script has default labels for every area of memory in Java 7 and Java 8,
so you shouldn't need to configure any labels.
But if you don't like the labels that `check_jvm_memory` uses,
or if other versions of Java ever add new areas of memory,
you can configure how they appear in the script's output using the `Label` configuration option.

Note that any `Threshold`s you have configured need to match the `Label`s you set.
Also note that `Label`s can't contain spaces, or else Check MK won't be able to parse the script's output.

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

