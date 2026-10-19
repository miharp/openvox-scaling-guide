---
layout: default
title: "Add Agents"
nav_order: 6
description: "Add agents that get their catalogs from a compiler, and run Puppet on a schedule."
---

With the same code on every server, the nodes you actually want to manage can get their catalogs from the compiler.
This page adds one with ovadm, then turns on the agent service so Puppet runs on a schedule everywhere.

**What you'll learn:**

* [Adding an agent pointed at a compiler](#add-an-agent)
* [Turning on the agent service on every node](#run-puppet-on-a-schedule)

## Prerequisites

Before you start, you'll need:

* The servers from [Same Code on Every Server](codavox.html), with codavox running
* A machine with nothing installed, reachable from your workstation over SSH and listed in your OpenBolt inventory
* Port 8140 open from the agent to the compiler and to the main server

## Add an agent

From your workstation:

```console
bolt plan run ovadm::add_agent server_host=puppet.example.com agent_hosts=agent01.example.com agent_server=compiler01.example.com
```

`agent_server` is where the agent gets its catalogs: here the compiler.
Leave it out and the agent uses the main server.
Name several agents in `agent_hosts`, separated by commas.

The plan adds the OpenVox package repository, installs the agent, and points it at `agent_server`.
Then it runs the agent once to send a certificate request, signs it on the main server, and runs the agent again, so the new node has applied its first catalog when the plan finishes.

That catalog came from the compiler, and its configuration version is the one codavox deployed:

```console
$ ssh root@agent01 'puppet agent -t'
Notice: Requesting catalog from compiler01.example.com:8140 (192.168.57.14)
Notice: Catalog compiled by compiler01.example.com
Info: Applying configuration version 'production-821a20908b17'
Notice: Applied catalog in 0.06 seconds
```

## Run Puppet on a schedule

Nothing runs Puppet on a schedule yet: ovadm ran the agent once, and every run so far was started by hand.
[openvox_agent](https://forge.puppet.com/modules/miharp/openvox_agent) can keep the agent service running.

openvox_server already declares openvox_agent on the servers, so only your other nodes need it in their role:

```puppet
# site-modules/role/manifests/agent.pp
class role::agent {
  include openvox_agent

  file { '/etc/motd':
    ensure => file,
    source => 'puppet:///modules/role/motd',
  }
}
```

Then turn the service on for every node in `data/common.yaml`:

```yaml
openvox_agent::service_ensure: 'running'
openvox_agent::service_enable: true
```

Left unset, openvox_agent leaves the service as it found it.
{: .tip }

Commit, push, and deploy with codavox:

```console
ssh root@puppet 'codavox deploy production --wait'
```

Each node's next Puppet run starts the service and enables it at boot, and from then on Puppet runs every 30 minutes on its own:

```console
$ ssh root@agent01 'puppet agent -t'
Notice: Catalog compiled by compiler01.example.com
Info: Applying configuration version 'production-d71e15a49470'
Notice: /Stage[main]/Openvox_agent::Service/Service[puppet]/ensure: ensure changed 'stopped' to 'running'
```

## Watch it

{% include cast.html src="agents.cast" title="ovadm adds an agent, and Puppet starts running on a schedule" %}
