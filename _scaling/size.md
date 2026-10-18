---
layout: default
title: "Size Them"
nav_order: 4
description: "Size each OpenVox server for its hardware with openvox_tune."
---

Out of the box, OpenVox Server uses 2 GB of memory and at most four workers, however big the machine is.
Each worker, a JRuby instance, compiles one catalog at a time, so a large compiler with the defaults leaves most of its CPUs and memory idle.
[openvox_tune](https://forge.puppet.com/modules/miharp/openvox_tune) measures each server and recommends a size for it.

**What you'll learn:**

* [Running openvox_tune's plan against your servers](#get-a-recommendation)
* [Putting its recommendations in each server's node data](#put-the-values-in-node-data)
* [Applying them with Puppet](#apply-them)

## Prerequisites

Before you start, you'll need:

* Both servers managed by openvox_server, as in [Hand Them to Puppet](configure.html)
* openvox_tune in your OpenBolt project: `bolt module add miharp-openvox_tune`

## Get a recommendation

Run the plan against both servers together, from your workstation:

```console
bolt plan run openvox_tune::tune --targets puppet.example.com,compiler01.example.com hiera=openvox_server
```

`hiera=openvox_server` prints the recommendations as Hiera data for openvox_server.
For the machines in the recording, a main server with 4 CPUs and 8 GB and a compiler with 8 CPUs and 16 GB:

```yaml
# For puppet.example.com:
openvox_server::max_active_instances: 2
openvox_server::heap_mb: 1536
openvox_server::reserved_code_cache_mb: 512
# For compiler01.example.com:
openvox_server::max_active_instances: 7
openvox_server::heap_mb: 4096
openvox_server::reserved_code_cache_mb: 1024
```

The compiler goes from four workers and 2 GB to seven workers and 4 GB.
The plan leaves a quarter of each machine's memory to the operating system and gives each worker 512 MB.

The main server goes *down*, from four workers to two and from 2 GB to 1.5 GB.
Run against both servers, the plan knows the main server has a compiler beside it, and the compiler does the compiling.

## Put the values in node data

Each server's values go in its node data, next to what is already there:

```yaml
# data/nodes/compiler01.example.com.yaml
openvox_server::ca: false
openvox_server::ca_server: 'puppet.example.com'

# OpenVox Server sized for this machine by openvox_tune's plan.
openvox_server::max_active_instances: 7
openvox_server::heap_mb: 4096
openvox_server::reserved_code_cache_mb: 1024
```

```yaml
# data/nodes/puppet.example.com.yaml
# OpenVox Server sized for this machine by openvox_tune's plan.
openvox_server::max_active_instances: 2
openvox_server::heap_mb: 1536
openvox_server::reserved_code_cache_mb: 512
```

## Apply them

Commit, push, deploy with r10k, and run Puppet on each server:

```console
ssh root@compiler01 'puppet agent -t'
ssh root@puppet 'puppet agent -t'
```

Each run changes the memory settings in `JAVA_ARGS` and the number of workers, then restarts OpenVox Server before the run ends, so the server already runs at its new size when the run finishes.

## Watch it

{% include cast.html src="size.cast" title="openvox_tune sizes the servers and Puppet applies it" %}

Next, [Same Code on Every Server](codavox.html) gets your code onto the compiler.
