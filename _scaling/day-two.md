---
layout: default
title: "Day Two"
nav_order: 7
description: "Status checks, deploys on every push, adding a compiler, backing up the main server, upgrading and extending the CA, once everything is running."
---

Everything is in place.
This page covers the routine that follows: checking the servers, deploying without logging in, growing the setup, backing it up, upgrading it, and keeping its CA from expiring.

**What you'll learn:**

* [Checking every server](#check-every-server)
* [Deploying on every push](#deploy-on-every-push)
* [Adding a compiler](#add-a-compiler)
* [Backing up the main server](#back-up-the-main-server)
* [Upgrading OpenVox](#upgrade-openvox)
* [Extending the CA](#extend-the-ca)
* [What to check when something goes wrong](#when-something-goes-wrong)

## Prerequisites

Before you start, you'll need:

* Everything from the previous pages: the main server, a compiler, codavox, and an agent
* For a new compiler, a machine with nothing installed, like the ones on [Install the Servers](install.html)
* For backups, somewhere off the main server to keep them, readable only by you

## Check every server

From your workstation, `ovadm::status` runs ovadm's checks against each server you name, and reports its service, its version, and the compilers the main server has signed:

```console
$ bolt plan run ovadm::status server_host=puppet.example.com,compiler01.example.com
ovadm 0.14.0
=== puppet ===
  ✓ os_family: RedHat
  ✓ java: java 21.0.12.1
  ✓ port_8140: puppetserver is already listening on 8140
  ✓ firewall: no active firewalld or ufw
  ✓ ntp: NTP synchronised
  ✓ memory: 7488 MB of memory, enough for the 1536 MB heap (-Xmx1536m)
      in /etc/sysconfig/puppetserver (OpenVox Server needs 1690 MB)
  ✓ service/puppetserver: running
  OpenVox Server: 8.17.0-1.el10
  Topology: server puppet.example.com (CA), 1 compiler: compiler01.example.com
  OpenVoxDB: the server sends nothing to OpenVoxDB (no puppetdb.conf)
  ✓ CA certificate CN=Puppet CA: puppet.example.com (intermediate):
      expires on 2041-10-14T16:10:07Z, in 5448 days
  ✓ CA certificate CN=Puppet Root CA: 49c4313b5981be (root):
      expires on 2041-10-14T16:10:06Z, in 5448 days
  ✓ certificate of puppet.example.com:
      expires on 2041-10-14T16:10:08Z, in 5448 days
=== compiler01 ===
...
```

The long lines are wrapped here to fit the page.
The memory check also confirms the sizes from [Size Them](size.html) are in place.
This setup has no OpenVoxDB, which is why the report says so.
The last three lines are the end dates of the CA's certificates and of the server's own, which ovadm reports since 0.14.0; [Extend the CA](#extend-the-ca) is about them.

On the main server, `codavox compilers` shows which version of your code each server runs and when it last checked in, as on [Same Code on Every Server](codavox.html#deploy-a-change).

## Deploy on every push

So far every deploy was `codavox deploy` on the main server.
`codavox::deploy_server` adds two ways to start one from elsewhere, on port 8170 of the main server:

* a **webhook** your git server calls on every push, which deploys the pushed branch as an environment
* a **deploy API** for CI, authenticated with a bearer token

Add it to the main server's role, and give it a webhook secret, an API token, or both, from encrypted Hiera:

```puppet
class role::primary {
  include openvox_server
  include codavox::primary
  include codavox::deploy_server
}
```

```yaml
codavox::deploy_server_secret_file: '/etc/codavox/webhook.secret'
codavox::deploy_server_secret: >
  ENC[PKCS7,...]
```

Then point your control repository's webhook at `https://puppet.example.com:8170/v1/webhook`, with the same secret.
GitHub, GitLab and generic callers are all recognised; [codavox's deploy server documentation](https://github.com/miharp/codavox/blob/main/docs/deploy-server.md) has the details and the API.
Only the main server needs to hear about a push: the compilers fetch each new version themselves.

## Add a compiler

Give the new compiler its node data first.
`ovadm::add_compiler` runs Puppet on it before it finishes, and that run must already find the CA switched off and codavox's publisher:

```yaml
# data/nodes/compiler02.example.com.yaml
openvox_server::ca: false
openvox_server::ca_server: 'puppet.example.com'

codavox::agent_publisher: 'https://puppet.example.com:8150'
```

Its role comes from the `compiler\d+` rule in `manifests/site.pp`.
Commit, push, deploy, then install it from your workstation:

```console
bolt plan run ovadm::add_compiler server_host=puppet.example.com compiler_hosts=compiler02.example.com
```

The plan installs the same version of OpenVox Server the main server runs, has the main server sign the compiler's certificate with the `openvox_compiler` role in it, and runs Puppet, which installs codavox.
In the recording it takes two and a half minutes.
codavox then fetches the current version of your code by itself, so the new compiler serves the same code as the others without another deploy:

```console
$ ssh root@puppet 'codavox compilers'
COMPILER                ENVIRONMENT  CODE_ID       COMMIT        LAST POLL
compiler02.example.com  production   b1dc1ae91e75  57bc106b633a  2s ago
puppet.example.com      production   b1dc1ae91e75  57bc106b633a  4s ago
compiler01.example.com  production   b1dc1ae91e75  57bc106b633a  9s ago
```

Finally, size it: run openvox_tune's plan against the main server and every compiler together, as on [Size Them](size.html), and add the new compiler's values to its node data.

## Back up the main server

Only the main server holds anything you can't get back: its CA.
Every agent and compiler trusts it, and without its private key no new node can join and no certificate can be revoked.
The compilers hold nothing of their own; you just built one from nothing, and codavox caught it up.
A lost agent is added again with `ovadm::add_agent`.

From your workstation:

```console
bolt plan run ovadm::backup server_host=puppet.example.com \
  extra_paths='["/opt/puppetlabs/codavox/state/provenance.jsonl"]'
```

The plan copies the CA, the server's certificates, the rest of `/etc/puppetlabs/` apart from your code, and the commit each environment was deployed from into one archive on the main server.
Then it checks the archive against its own checksums and prints where it is:

```console
puppet: /var/backups/openvox/openvox-backup-puppet.example.com-20261113T191224Z.tar.gz (0.0 MB, 40 files)
puppet: 42 checksums checked, ok
Copy the archive off the host: it holds the CA private key.
```

In the recording it takes about 30 seconds, and the services keep running.

`extra_paths` adds files from outside `/etc/puppetlabs/`.
codavox's provenance log is the one this setup needs: it is what `codavox provenance` reads to trace a configuration version back to its commit, and codavox can't rebuild it.
If r10k reaches your git server with a deploy key, add the key too.

The archive holds the CA's private key.
Copy it off the main server as soon as the plan finishes, to storage only you can read.
{: .warning }

Take a backup after you add a compiler or a batch of agents, and before every upgrade.
A CA restored from a backup doesn't know the certificates it signed after that backup, and each of those nodes has to be added again.
ovadm doesn't run backups on a schedule.

`ovadm::restore` puts an archive back: the configuration after a change that went wrong, a lost CA, or the whole main server rebuilt on a new machine with the same name.
[ovadm's backup and restore documentation](https://github.com/miharp/ovadm/blob/main/documentation/backup_restore.md) walks through each.
On a main server that is still running, put back only what you lost, with `scope='["config"]'` or `scope='["ca"]'`.
A restore without `scope` also puts back the provenance log as it was at backup time, and the deploys recorded since then lose their entries.

## Upgrade OpenVox

`ovadm::upgrade` upgrades the main server first and the compilers after it, from your workstation:

```console
bolt plan run ovadm::upgrade server_host=puppet.example.com \
  compiler_hosts=compiler01.example.com,compiler02.example.com \
  ovox_server_version=8.18.0 \
  backup_extra_paths='["/opt/puppetlabs/codavox/state/provenance.jsonl"]'
```

Before it touches a package, the plan disables the Puppet agent on every server it names and takes a backup of the main server, like the one above.
Copy that archive off the main server too.
The agents stay disabled until the last compiler is ready, so a scheduled run can't apply your code to a server halfway through, and the plan enables them again when it finishes or fails.
This needs ovadm 0.13.0 or later.

The compilers are upgraded together, so plan for a short gap in which they can't serve catalogs.
In the recording the plan takes just under four minutes for the three servers.

Your Hiera data from [Hand Them to Puppet](configure.html) names a collection and no version, so within OpenVox 8 there is nothing to change in your control repository: openvox_server keeps whichever version is installed.

If you pin a version with `openvox_server::package_version`, raise the pin and deploy first, then run the plan straight away.
A pin that still names the old version puts the old package back on the next agent run.
{: .warning }

The collection is a pin of the same kind.
To move from OpenVox 8 to 9, set `openvox_server::collection: 'openvox9'`, deploy, then run the plan with the new version.
[ovadm's upgrade documentation](https://github.com/miharp/ovadm/blob/main/documentation/upgrade.md#when-your-puppet-code-pins-the-version) covers that move, and how to keep a scheduled run from moving a compiler ahead of the main server between the two steps.
[openvox_server's upgrade notes](https://github.com/miharp/puppet-openvox_server#upgrading) cover the agent package.

## Extend the CA

The CA that ovadm created on [Install the Servers](install.html#the-certificate-authority) runs for 15 years, and `ovadm::status` prints its end date on every check.
When less than a year is left, the check becomes a warning.
Past that date every agent run fails at once, so extend the CA soon after the warning appears.

`ovadm::extend_ca` gives the CA's certificates a later end date, from your workstation:

```console
bolt plan run ovadm::extend_ca server_host=puppet.example.com \
  compiler_hosts=compiler01.example.com,compiler02.example.com
```

The CA keeps its key, so every certificate it has signed stays valid, and no agent or compiler is added again.
The new end date is 15 years from the day you run the plan, or as long as `ttl` says.
The CA in the recording is a month old, so there the date moves by a month.

The plan takes a backup of the main server first; copy that archive off the main server too.
Then it restarts OpenVox Server on the main server and on each compiler, one after the other.
In the recording it takes just under three minutes, and the main server is down for less than a minute of that.
This needs ovadm 0.14.0 or later.

Agents fetch the new CA certificates by themselves within a day.

codavox needs nothing done.
From 0.12.0, the version [Same Code on Every Server](codavox.html) installs, it re-reads the CA and its server's certificate when they change on disk.
An older codavox reads them only when its services start: restart `codavox-publish` and `codavox-agent` on the main server and `codavox-agent` on each compiler after the extension, or upgrade it first.

A CA that OpenVox Server created by itself is different: it runs for five years, and the server's own certificate ends on the same day.
[ovadm's documentation](https://github.com/miharp/ovadm/blob/main/documentation/extend_ca.md) covers that CA, one that has already expired, and agents that can no longer fetch the new certificates themselves.

## When something goes wrong

* **A compiler serves old code.** `codavox compilers` on the main server shows its `CODE_ID` and `LAST POLL`. A poll that keeps getting older means it can't reach the main server on port 8150, or its `codavox-agent` service is stopped.
* **A new compiler is refused.** codavox only hands code to certificates with the `openvox_compiler` role, or to names in `codavox::publish_allow_certnames`.
* **A server won't start.** `ovadm::status` checks memory against the heap in `JAVA_ARGS`, the port, the firewall and the clock.
* **You need the commit behind a catalog.** `codavox provenance <configuration version>` on the main server names the commit behind any configuration version an agent printed.
* **The main server is lost.** Agents keep getting catalogs from the compilers, built from the code they already have; deploys and new certificates wait. Build a machine with the same name and address, and restore your latest backup onto it with `ovadm::restore` from ovadm 0.9.0 or later, as [ovadm's documentation](https://github.com/miharp/ovadm/blob/main/documentation/backup_restore.md#rebuilding-the-server-on-a-new-host) describes. The agents and compilers keep their certificates, and codavox seals the same code to the same `code_id`.

## Watch it

The recording checks both servers with `ovadm::status`, adds a second compiler and shows codavox catching it up, backs up the main server and copies the archive off it, upgrades all three servers to 8.18.0, then extends the CA, with codavox left running.
It ends with one more `ovadm::status`, showing the new version and the CA's new end date.

{% include cast.html src="daytwo.cast" title="ovadm::status, a second compiler, a backup, an upgrade, and the CA extended" %}
