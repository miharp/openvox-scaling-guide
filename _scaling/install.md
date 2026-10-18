---
layout: default
title: "Install the Servers"
nav_order: 2
description: "Install the main server, its certificate authority and a compiler from your workstation with ovadm."
---

From your workstation, one OpenBolt plan installs the main server, its certificate authority (CA), and a compiler that trusts it.
Then r10k deploys your control repository to the main server, as it would on a single server.

**What you'll learn:**

* [Setting up an OpenBolt project with ovadm](#set-up-an-openbolt-project)
* [Describing your machines in an inventory](#write-an-inventory)
* [Installing the main server and a compiler](#install-the-main-server-and-a-compiler)
* [What makes ovadm's CA different](#the-certificate-authority)
* [The role ovadm writes into each server's certificate](#roles-in-the-certificates)
* [Deploying your control repository](#deploy-your-control-repository)

## Prerequisites

Before you start, you'll need:

* The machines and the workstation from the [overview](index.html)
* At least 2,253 MB of memory on each server. OpenVox Server won't start on less than 1.1 times its 2 GB heap, and ovadm's checks stop before installing on less.
* Port 8140 open to the main server from the compiler and the agents, and to the compiler from the agents. ovadm doesn't manage the host firewall.

All commands on this page run on your workstation.

## Set up an OpenBolt project

[ovadm](https://github.com/miharp/ovadm) is a module for OpenBolt, which runs commands on other machines over SSH.
It does for OpenVox what [peadm](https://github.com/puppetlabs/puppetlabs-peadm) does for Puppet Enterprise.
Create a project and add ovadm to it:

```console
mkdir scaling && cd scaling
bolt project init scaling
bolt module add miharp-ovadm
```

OpenBolt writes ovadm into the project's `Puppetfile` and installs it.

## Write an inventory

The inventory lists the machines and how to reach them.
Save it as `inventory.yaml` in the project:

```yaml
targets:
  - puppet.example.com
  - compiler01.example.com
  - agent01.example.com
config:
  ssh:
    user: root
```

If you log in as a user who can use `sudo`, set `user` to that user and add `run-as: root`.

The recording reaches the machines over a private network, so its inventory names each one and gives its address in `uri`.
{: .tip }

## Install the main server and a compiler

```console
bolt plan run ovadm::install server_host=puppet.example.com compiler_hosts=compiler01.example.com
```

The plan checks each machine first: the operating system, Java, port 8140, the firewall, the clock, and memory.
Then it adds the OpenVox package repository, installs OpenVox Server, creates the CA, and starts the server.
Last, it installs each compiler, has the main server sign its certificate, and switches its own CA off.

Name more compilers in `compiler_hosts`, separated by commas, or add them later with `ovadm::add_compiler`, as [Day Two](day-two.html) shows.

## The certificate authority

The CA signs the certificate of every machine Puppet manages.
ovadm creates it the way the Puppet Enterprise installer does: a root certificate and an intermediate signing certificate, both valid for 15 years.

Left to itself, OpenVox Server creates a single CA certificate that expires after five years, on the same day as your first agents' certificates.
Nobody writes that date down, and five years later every agent stops trusting the server at the same moment.

## Roles in the certificates

ovadm writes a role into each server's certificate: `openvox_server` for the main server and `openvox_compiler` for compilers.
You can see it with `puppet ssl show`:

```console
$ ssh root@compiler01 'puppet ssl show | grep -A1 "Role Name"'
            Puppet Node Role Name:
                ..openvox_compiler
```

The two dots are bytes from how the certificate stores the text.
codavox uses this role on [Same Code on Every Server](codavox.html) to decide which machines may download your code.
It is only a label in the certificate, separate from the role classes in your control repository that decide what Puppet applies to a node.

## Deploy your control repository

Set up r10k on the main server as [Deploying Code](https://docs.openvoxproject.org/ecosystem/latest/getting_started/code_deployment.html) describes, and deploy:

```console
ssh root@puppet 'r10k deploy environment --modules'
```

Only the main server has your code now.
The compiler doesn't need any until it compiles catalogs for agents, and [Same Code on Every Server](codavox.html) gets it there.

## Watch it

The recording starts from three machines with nothing installed.

{% include cast.html src="install.cast" title="ovadm installs the main server, its CA and a compiler" %}

ovadm's job is done.
It installs and then gets out of the way: nothing it does runs on a schedule.
On the [next page](configure.html), Puppet takes over.
