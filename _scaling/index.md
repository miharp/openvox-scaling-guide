---
layout: default
title: "Overview"
nav_order: 1
description: "Go from one OpenVox server to a main server with compilers: what you'll build and what you need."
---

One OpenVox server goes a long way, but it compiles every catalog itself, and sooner or later it can't keep up.
The answer is more servers: a main server that runs the certificate authority (CA), and compilers beside it that share the work of compiling catalogs.

This guide builds that setup step by step.
It starts where [Deploying Code](https://docs.openvoxproject.org/ecosystem/latest/getting_started/code_deployment.html) ends: you know what a control repository is, and r10k has deployed one to a server.

**What you'll build:**

* [A main server with a long-lived CA, and a compiler that trusts it](install.html), installed from your workstation
* [Both servers configured by Puppet](configure.html) from then on
* [Each server sized for its hardware](size.html)
* [The same version of your code on every server](codavox.html), with each catalog recording which version it was built from
* [Agents that get their catalogs from the compiler](agents.html) and run Puppet on a schedule
* [The routine after that](day-two.html): deploys, status, upgrades, and adding a compiler later

Every page has a short recording of its commands running on fresh machines.

## The machines

| Machine | Role | Size in the recordings |
| ------- | ---- | ---------------------- |
| Your workstation | Runs OpenBolt, which installs everything over SSH | Your laptop |
| `puppet.example.com` | Main server: the CA, compiles catalogs, hands code to the compilers | 4 CPUs, 8 GB |
| `compiler01.example.com` | Compiler: compiles catalogs for agents | 8 CPUs, 16 GB |
| `agent01.example.com` | A node you manage | 2 CPUs, 4 GB |

## Prerequisites

Before you start, you'll need:

* Three Linux machines with nothing installed. The recordings use AlmaLinux 9, the only platform this guide has been tested on so far; ovadm itself also supports [several others](https://github.com/miharp/ovadm#requirements).
* SSH from your workstation to each machine, as root or as a user who can use `sudo`
* Names that resolve on every machine, such as `puppet.example.com`
* A control repository made from the [OpenVox control repository template](https://github.com/OpenVoxProject/control-repo-template), on a git server the main server can reach
* [OpenBolt](https://docs.openvoxproject.org/openbolt/latest/) on your workstation

## The tools

* **[ovadm](https://github.com/miharp/ovadm)**, an OpenBolt module, installs the servers and adds agents, then gets out of the way.
* **[openvox_server](https://forge.puppet.com/modules/miharp/openvox_server)**, **[openvox_tune](https://forge.puppet.com/modules/miharp/openvox_tune)** and **[openvox_agent](https://forge.puppet.com/modules/miharp/openvox_agent)** keep the servers and agents configured with Puppet. You can use your own code instead.
* **[codavox](https://github.com/miharp/codavox)** gets the same version of your code onto every server.

All of them are young, at version 0, and have so far run only on test machines like the ones in the recordings.
{: .warning }
