# Scaling OpenVox

A hands-on guide to going from one OpenVox server to a main server with
compilers, published at <https://mikeharp.com/openvox-scaling-guide/>.

It starts where the OpenVox docs'
[Deploying Code](https://docs.openvoxproject.org/ecosystem/latest/getting_started/code_deployment.html)
page ends: you have a control repository, and r10k has deployed it to a
server. Seven pages later you have a main server with a long-lived certificate
authority, a compiler beside it, the same version of your code on both, agents
that get their catalogs from the compiler, and the routine for keeping it all
running. Every page has a terminal recording of its commands running on fresh
machines.

## The pages

| Page | What it does |
| ---- | ------------ |
| [Overview](_scaling/index.md) | What you'll build, the machines, and the tools |
| [Install the Servers](_scaling/install.md) | ovadm installs the main server, its CA and a compiler from your workstation |
| [Hand Them to Puppet](_scaling/configure.md) | The openvox_server module takes over the servers |
| [Size Them](_scaling/size.md) | openvox_tune sizes each server for its hardware |
| [Same Code on Every Server](_scaling/codavox.md) | codavox gets one version of your code onto every server and records it in each catalog |
| [Add Agents](_scaling/agents.md) | ovadm adds agents pointed at the compiler; openvox_agent runs Puppet on a schedule |
| [Day Two](_scaling/day-two.md) | Status, deploys on every push, a second compiler, backups and upgrades |

## The tools

The guide uses two tools and three Puppet modules, all at version 0 and so far
run only on test machines like the ones in the recordings:

* [ovadm](https://github.com/miharp/ovadm), an OpenBolt module that installs the servers and adds agents
* [codavox](https://github.com/miharp/codavox), which distributes code to the compilers
* [openvox_server](https://forge.puppet.com/modules/miharp/openvox_server),
  [openvox_tune](https://forge.puppet.com/modules/miharp/openvox_tune) and
  [openvox_agent](https://forge.puppet.com/modules/miharp/openvox_agent),
  which keep the servers and agents configured with Puppet

The control repository the recordings deploy is
[openvox-demo-control](https://github.com/miharp/openvox-demo-control), made
from the OpenVox control repository template. Each page's change to it is one
commit, linked from the page.

## The recordings

The recordings in `assets/casts/` are [asciinema](https://asciinema.org/)
casts, 96 columns by 28 rows, played in the pages by asciinema-player. Each one
was taken on cloud VMs of the sizes the overview lists, in one chain: the state
at the end of one page is the start of the next, kept as a snapshot so a single
page can be retaken on the same starting point when a tool changes. The text on
each page quotes the output of the take it shows, so when a recording is
retaken the page is checked against it.

The scripts that drive the lab and the takes are not in this repository.

## Building the site

The site is Jekyll with
[jekyll-vitepress-theme](https://github.com/crmne/jekyll-vitepress-theme),
the theme and version <https://docs.openvoxproject.org> uses. Ruby comes from
`.ruby-version`.

```console
bundle install
bundle exec jekyll serve
```

Pushing `main` builds and deploys the site to GitHub Pages through
`.github/workflows/pages.yml`. Pull requests build without deploying.

How the pieces fit:

* `_scaling/` holds the pages, one collection in the order of their `nav_order`.
* `assets/casts/` holds the recordings. A page embeds one with
  `{% include cast.html src="install.cast" title="..." %}`; `_includes/cast.html`
  is the slot, and `_includes/jekyll_vitepress/head_end.html` loads the player
  and starts it on each page load, including the theme's Turbo page swaps.
* `_data/navigation.yml` and `_data/sidebar.yml` are the theme's menus.

## Relation to the OpenVox docs

The pages use the same theme, layout and conventions as the OpenVox docs, so
they can move to `docs/_ecosystem_8x/scaling/` in
[openvox-docs](https://github.com/OpenVoxProject/openvox-docs) with a nav entry
and the player lines from `head_end.html`. Until the tools are past version 0
the guide lives here, listed with the other community material on the docs'
[training page](https://docs.openvoxproject.org/ecosystem/latest/training.html).

## Contributing

Corrections and questions are welcome as issues or pull requests. A change to
what a page says a command does should come with the output it is based on, so
the page and its recording stay in agreement. By contributing you agree to
license your contribution under the terms below.

## License

The guide, its pages and recordings, is licensed under the
[Creative Commons Attribution-ShareAlike 4.0 International License](LICENSE)
(CC BY-SA 4.0).

The code and configuration examples in the pages, such as the Puppetfile
entries, Hiera data and manifests, are additionally dedicated to the public
domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/), so
you can copy them into your own control repository without attribution.
