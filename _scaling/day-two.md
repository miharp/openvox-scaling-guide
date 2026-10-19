---
layout: default
title: "Day Two"
nav_order: 7
description: "Status checks, deploys on every push, adding a compiler and upgrading, once everything is running."
---

Everything is in place.
This page covers the routine that follows: checking the servers, deploying without logging in, growing the setup, and upgrading it.

**What you'll learn:**

* [Checking every server](#check-every-server)
* [Deploying on every push](#deploy-on-every-push)
* [Adding a compiler](#add-a-compiler)
* [Upgrading OpenVox](#upgrade-openvox)
* [What to check when something goes wrong](#when-something-goes-wrong)

## Prerequisites

Before you start, you'll need:

* Everything from the previous pages: the main server, a compiler, codavox, and an agent
* For a new compiler, a machine with nothing installed, like the ones on [Install the Servers](install.html)

## Check every server

From your workstation, `ovadm::status` runs ovadm's checks against each server you name, and reports its service and version:

```console
$ bolt plan run ovadm::status server_host=puppet.example.com,compiler01.example.com
ovadm 0.7.1
=== puppet ===
  ✓ os_family: RedHat
  ✓ java: java 21.0.12.1
  ✓ port_8140: puppetserver is already listening on 8140
  ✓ firewall: no active firewalld or ufw
  ✓ ntp: NTP synchronised
  ✓ memory: 7488 MB of memory, enough for the 1536 MB heap (-Xmx1536m) in /etc/sysconfig/puppetserver (OpenVox Server needs 1690 MB)
  ✓ service/puppetserver: running
  OpenVox Server: 8.17.0-1.el10
=== compiler01 ===
...
```

The memory check also confirms the sizes from [Size Them](size.html) are in place.

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
In the recording it takes just under three minutes.
codavox then fetches the current version of your code by itself, so the new compiler serves the same code as the others without another deploy:

```console
$ ssh root@puppet 'codavox compilers'
COMPILER                ENVIRONMENT  CODE_ID       COMMIT        LAST POLL
puppet.example.com      production   6d0518b187eb  9ad43c61b446  17s ago
compiler02.example.com  production   6d0518b187eb  9ad43c61b446  17s ago
compiler01.example.com  production   6d0518b187eb  9ad43c61b446  25s ago
```

Finally, size it: run openvox_tune's plan against the main server and every compiler together, as on [Size Them](size.html), and add the new compiler's values to its node data.

## Upgrade OpenVox

Once openvox_server manages your servers, upgrade them through Hiera, so Puppet and ovadm never disagree about the version.
Pin the version you run in `data/common.yaml`, and raise it to upgrade:

```yaml
openvox_server::package_version: '8.17.0'
```

Deploy, then run Puppet on the main server first and the compilers after it.
Each run installs the new `openvox-server` and restarts the service on it.
To move from OpenVox 8 to 9, set `openvox_server::collection: 'openvox9'` as well; [openvox_server's upgrade notes](https://github.com/miharp/puppet-openvox_server#upgrading) cover that move and the agent package.

[`ovadm::upgrade`](https://github.com/miharp/ovadm/blob/main/documentation/upgrade.md) does the same job from your workstation for servers that Puppet doesn't manage.
{: .tip }

## When something goes wrong

* **A compiler serves old code.** `codavox compilers` on the main server shows its `CODE_ID` and `LAST POLL`. A poll that keeps getting older means it can't reach the main server on port 8150, or its `codavox-agent` service is stopped.
* **A new compiler is refused.** codavox only hands code to certificates with the `openvox_compiler` role, or to names in `codavox::publish_allow_certnames`.
* **A server won't start.** `ovadm::status` checks memory against the heap in `JAVA_ARGS`, the port, the firewall and the clock.
* **You need the commit behind a catalog.** `codavox provenance <configuration version>` on the main server names the commit behind any configuration version an agent printed.

## Watch it

The recording checks both servers with `ovadm::status`, then adds a second compiler and shows codavox catching it up.

{% include cast.html src="daytwo.cast" title="ovadm::status, and a second compiler" %}
