---
layout: post
title: Check MK Guide
last_modified_at: 2019-11-29 21:44:30 UTC
---

Check MK is a monitoring solution with a rule-based configuration,
support for distributed monitoring,
and a simple interface for writing custom check scripts.
This guide will show you how to install and configure it on CentOS with the Open Monitoring Distribution (OMD).
<!-- Along the way we'll configure authentication with Active Directory (AD), -->

1. TOC
{:toc}

### Outline

* Install OMD
* Configure stuff
* AD Authentication
* Distributed Monitoring
* Configuration with WATO
  * Describe rule-based vs. template/ad-hoc configuration
  * Add a host
  * Process Discovery
  * Memory Thresholds
  * Disable a Service
  * Direct checks
  * Notifications
* Administrative tasks
  * Acknowledgements
  * Downtime
  * New services

<!--
### Footnotes

[^1]: Credit goes to <user> for <whatever reasons>.
-->

