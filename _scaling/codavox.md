---
layout: default
title: "Same Code on Every Server"
nav_order: 5
description: "Get the same version of your code onto every OpenVox server with codavox, and label each catalog with it."
---

Your main server and your compiler both compile catalogs, so both need the same version of your code.
r10k alone can't promise that.
[codavox](https://github.com/miharp/codavox) can: it deploys once on the main server, hands that exact version to every compiler, and labels each catalog with the version it was built from.

**What you'll learn:**

* [Why r10k alone is not enough](#why-r10k-alone-is-not-enough)
* [How codavox works](#how-codavox-works)
* [Adding codavox to your control repository](#add-codavox-to-your-control-repository)
* [Installing it with Puppet](#install-it-with-puppet)
* [Deploying a change and checking every server](#deploy-a-change)
* [Tracing a catalog back to its commit](#trace-a-catalog-to-its-commit)

## Prerequisites

Before you start, you'll need:

* The main server and compiler from the previous pages, managed by openvox_server and sized by [Size Them](size.html)
* Port 8150 open from each compiler to the main server

## Why r10k alone is not enough

r10k, the usual tool for deploying a control repository, changes the code in place, on each server separately.
That leaves three gaps:

* A compiler that is down when you push misses the deploy and keeps the old code until the next one.
* A catalog compiled in the middle of a deploy can mix old and new code.
* Nothing tells you which server runs which version.

## How codavox works

1. You deploy on the main server with `codavox deploy production`.
   codavox runs r10k there, once, exactly as you do now.
2. It takes a fingerprint of the result, the `code_id`, and packs that version into an ordinary `.tar.gz` file.
3. Each compiler, and the main server itself, polls for a new version every 30 seconds, downloads it, checks the fingerprint, and switches to it in one step.
   A server never serves a half-copied version, and a compiler that was down catches up when it comes back.
   Agents need nothing new: they keep asking their server for catalogs as before.
4. Every catalog records the `code_id` it was built from, so agents fetch the files they manage from your modules out of that same version.

Compilers prove who they are with the Puppet certificates they already have, so there are no extra passwords.
codavox hands your code only to the compilers you allow: by the role ovadm wrote into their certificates, or by name.

## Add codavox to your control repository

The [codavox module](https://forge.puppet.com/modules/miharp/codavox) installs and configures everything.
The whole change to the control repository in the recording is [one commit of 25 lines](https://github.com/miharp/openvox-demo-control/commit/c628d446b73458313b108ffed536880bba00d380).

Add the module and its two dependencies to the `Puppetfile`:

```ruby
# codavox: code distribution to the compilers, and static catalogs.
mod 'miharp/codavox',                  '0.6.3'
mod 'puppetlabs/puppet_authorization', '1.0.1'
mod 'puppetlabs/concat',               '10.1.0'
```

Settings for every server go in `data/common.yaml`.
`basedir` is where r10k deploys, and `publish_allow_roles` names the compilers that may download your code, by the role in their certificates:

```yaml
codavox::package_ensure: '0.11.0'
codavox::basedir: '/etc/puppetlabs/code/environments'
codavox::publish_allow_roles:
  - 'openvox_compiler'
```

If your compilers' certificates have no role, list them by name with `codavox::publish_allow_certnames` instead.
{: .tip }

Each server's node data says where to download code from.
The main server downloads from itself:

```yaml
# data/nodes/puppet.example.com.yaml
codavox::agent_publisher: "https://%{trusted.certname}:8150"
```

```yaml
# data/nodes/compiler01.example.com.yaml
codavox::agent_publisher: 'https://puppet.example.com:8150'
```

Three lines in the roles put it to work:

```puppet
# site-modules/role/manifests/primary.pp
class role::primary {
  include openvox_server
  include codavox::primary
}
```

```puppet
# site-modules/role/manifests/compiler.pp
class role::compiler {
  include openvox_server
  include codavox::agent
  include codavox::server
}
```

Finally, let codavox name the catalogs it serves.
Add this to `scripts/config_version.sh`, after its argument check:

```shell
# codavox serves this environment from a sealed tree with no .git or
# .r10k-deploy.json, so let codavox label the catalog: <environment>-<code_id>.
if [ "$1" = /opt/puppetlabs/codavox/environments ]; then
  exec /usr/bin/codavox-config-version "$1" "$2"
fi
```

Without it, catalogs keep r10k's label.

## Install it with Puppet

Deploy the change with r10k one last time, then run Puppet on both servers:

```console
ssh root@puppet 'r10k deploy environment --modules'
ssh root@puppet 'puppet agent -t'
ssh root@compiler01 'puppet agent -t'
```

The first run installs codavox on both servers, and codavox deploys its first version.
The compiler switches to it in that same run.
The main server keeps compiling from r10k's copy until the first version has arrived, so it is never left with an empty code directory.
It switches on its next run, whose own catalog was compiled just before the switch and still has r10k's label.
The run after that has codavox's label:

```console
$ ssh root@puppet 'puppet agent -t'
Info: Applying configuration version 'puppet-production-c628d446b73'
...
$ ssh root@puppet 'puppet agent -t'
Info: Applying configuration version 'production-991bc15eb41b'
```

## Deploy a change

From now on, deploy with codavox instead of r10k:

```console
ssh root@puppet 'codavox deploy production --wait'
```

Then check where every server stands:

```console
$ ssh root@puppet 'codavox compilers'
COMPILER                ENVIRONMENT  CODE_ID       COMMIT        LAST POLL
puppet.example.com      production   821a20908b17  06a6d57df357  6s ago
compiler01.example.com  production   821a20908b17  06a6d57df357  19s ago
```

Both servers run the same version of `production`, built from the commit beside it.
`LAST POLL` is how long ago each one last checked in.
The `code_id` is a fingerprint of the code alone, so the same code gives the same `code_id` on any machine, on any day.

## Trace a catalog to its commit

Every catalog is labelled with its environment and the first 12 characters of its `code_id`, and every Puppet run prints that label, as the main server's run did above.

Each catalog with a `code_id` is a [static catalog](https://docs.openvoxproject.org/openvox/latest/static_catalogs.html).
OpenVox Server supports static catalogs but has nothing to supply the version, so out of the box it does without.
With codavox, the files agents manage from your modules come from the same version as their catalog, so a deploy in the middle of a run can't hand an agent a file its catalog never saw.
And because the catalog already describes each file, agents stop asking the server about every one of them on every run: fewer requests for the compilers, and quicker runs.

On the main server, the label leads back to git:

```console
$ ssh root@puppet 'codavox provenance production-821a20908b17'
06a6d57df357a8897b536c5c6bd79d7a35ccaa25        deployed 2026-10-18 16:24:28 +0000      sealed 2026-10-18T16:24:29Z
```

## Watch it

The recording adds codavox to the main server and compiler, then deploys a change to `/etc/motd`, a file agents get from a module, and traces its label back to the commit.

{% include cast.html src="codavox.cast" title="codavox gets the same code onto every server" %}

## A single server

None of this needs a compiler.
On a single server, `include codavox::primary` alone gives it the same one-step switch, static catalogs and catalog labels, and adding compilers later changes nothing on it.
