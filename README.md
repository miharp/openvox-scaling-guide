# Scaling OpenVox

A hands-on guide to going from one OpenVox server to a main server with
compilers: ovadm, openvox_server, openvox_tune, openvox_agent and codavox,
with a terminal recording on every page.

Built with [jekyll-vitepress-theme](https://github.com/crmne/jekyll-vitepress-theme)
1.10.1, the theme and version docs.openvoxproject.org uses, so the pages in
`_scaling/` can move to `docs/_ecosystem_latest/scaling/` in
[openvox-docs](https://github.com/OpenVoxProject/openvox-docs) unchanged.
Recordings are asciicast files in `assets/casts/`, embedded with
`{% include cast.html src="..." title="..." %}`.

```console
bundle install
bundle exec jekyll serve
```
