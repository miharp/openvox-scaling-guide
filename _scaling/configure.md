---
layout: default
title: "Hand Them to Puppet"
nav_order: 3
description: "Hand the servers ovadm installed to Puppet with the openvox_server module."
---

Once installed, the servers are nodes like any other, and the Puppet way to look after a node is with Puppet.
This page gives each server a role in your control repository and checks that Puppet's first run agrees with everything ovadm set up.

**What you'll learn:**

* [Adding the modules to your Puppetfile](#add-the-modules)
* [Giving each server a role](#give-each-server-a-role)
* [Switching the CA off on compilers](#switch-the-ca-off-on-compilers)
* [Running Puppet on the servers](#run-puppet)

## Prerequisites

Before you start, you'll need:

* The main server and compiler from [Install the Servers](install.html), with your control repository deployed to the main server

## Add the modules

[openvox_server](https://forge.puppet.com/modules/miharp/openvox_server) manages OpenVox Server: the package, the CA settings, the web server settings, compiler mode and the service.
It is written to pick up exactly where ovadm stops, but ovadm doesn't mind what you use; your own code works too.

Add it to your control repository's `Puppetfile`, with the two modules the next pages use and their dependencies:

```ruby
mod 'miharp/openvox_server', '0.1.1'
mod 'miharp/openvox_agent',  '0.2.0'
mod 'miharp/openvox_tune',   '0.1.0'
mod 'puppetlabs/hocon',      '2.0.0'
mod 'puppetlabs/inifile',    '6.5.0'
mod 'puppetlabs/stdlib',     '9.7.0'
```

Tell openvox_server which package collection ovadm installed from, in `data/common.yaml`.
Without a version, it keeps the version ovadm installed:

```yaml
openvox_server::collection: 'openvox8'
```

## Give each server a role

Each node gets its role class in `manifests/site.pp`, by name:

```puppet
node default {
  include role::agent
}

node 'puppet.example.com' {
  include role::primary
}

node /^compiler\d+\.example\.com$/ {
  include role::compiler
}
```

Both server roles are a single line for now:

```puppet
# site-modules/role/manifests/primary.pp
class role::primary {
  include openvox_server
}
```

```puppet
# site-modules/role/manifests/compiler.pp
class role::compiler {
  include openvox_server
}
```

openvox_server also manages the agent on each server through [openvox_agent](https://forge.puppet.com/modules/miharp/openvox_agent), so don't include `openvox_agent` in these roles yourself.
{: .tip }

## Switch the CA off on compilers

A compiler is the same module with its CA switched off.
That goes in the compiler's node data, the file the template's `hiera.yaml` reads for each node by its certificate name:

```yaml
# data/nodes/compiler01.example.com.yaml
openvox_server::ca: false
openvox_server::ca_server: 'puppet.example.com'
```

With a second compiler, give it a file of its own with the same two lines.

## Run Puppet

Commit, push, deploy with r10k, then run Puppet on each server:

```console
ssh root@compiler01 'puppet agent -t'
ssh root@puppet 'puppet agent -t'
```

The runs should change nothing.
openvox_server agrees with everything ovadm set up, and from now on it is in charge whenever Puppet runs on the servers.

The first run on each server downloads the modules' plugins, which prints a line per file. The recording does that out of view.
{: .tip }

## Watch it

{% include cast.html src="handover.cast" title="Puppet takes over the servers ovadm installed" %}

Nothing runs Puppet on a schedule yet: ovadm ran the agent once, and the recordings run Puppet by hand.
[Add Agents](agents.html) turns on the agent service everywhere.
