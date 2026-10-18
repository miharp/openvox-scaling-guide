---
layout: default
title: "Size Them"
nav_order: 4
description: "Size each OpenVox server for its hardware with openvox_tune."
---

Out of the box, OpenVox Server uses 2 GB of memory and at most four workers, however big the machine is.
[openvox_tune](https://forge.puppet.com/modules/miharp/openvox_tune) measures each server and recommends a size for it.

**What you'll learn:**

* Running the `openvox_tune::tune` plan against your servers
* Putting its recommendations in each server's node data
* Why the main server gets less once a compiler shares the work
* How `openvox_server` restarts the server within the run

This page is an outline so far. The recording below is from the blog post, and will be re-recorded for this page.
{: .tip }

{% include cast.html src="size.cast" title="openvox_tune sizes the servers and Puppet takes over" %}
