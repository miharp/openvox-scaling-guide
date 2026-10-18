---
layout: default
title: "Install the Servers"
nav_order: 2
description: "Install the main server, its certificate authority and a compiler from your workstation with ovadm."
---

From your workstation, one OpenBolt plan installs the main server, its certificate authority, and a compiler that trusts it.

**What you'll learn:**

* Setting up an OpenBolt project with [ovadm](https://github.com/miharp/ovadm)
* Writing an inventory of your machines
* Installing the main server and a compiler with `ovadm::install`
* Why ovadm's CA has a root and an intermediate certificate that last 15 years
* The role ovadm writes into each server's certificate, and what uses it
* Deploying your control repository with r10k, as in [Deploying Code](https://docs.openvoxproject.org/ecosystem/latest/getting_started/code_deployment.html)

This page is an outline so far. The recording below is from the blog post, and will be re-recorded for this page.
{: .tip }

{% include cast.html src="install.cast" title="ovadm installs OpenVox Server, its CA and a compiler" %}
