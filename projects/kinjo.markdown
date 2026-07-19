---
layout: page
title: Kinjo
permalink: /projects/kinjo/
description: >-
  Kinjo is a terminal UI to browse local DNS-SD / Bonjour / mDNS services,
  filter them, and run configurable actions against a selected service.
---

**Browse local DNS-SD services, filter them, and run custom actions from a terminal UI.**

[![GitHub Release](https://img.shields.io/github/v/release/abbyssoul/kinjo?display_name=tag&color=%23a6a)](https://github.com/abbyssoul/kinjo/releases)
[![Crates.io](https://img.shields.io/crates/v/kinjo.svg)](https://crates.io/crates/kinjo)
[![docs.rs](https://img.shields.io/docsrs/kinjo)](https://docs.rs/kinjo)
[![License: MIT](https://img.shields.io/github/license/abbyssoul/kinjo)](https://github.com/abbyssoul/kinjo/blob/main/LICENSE)

![Kinjo screenshot](https://github.com/user-attachments/assets/319acde6-a3d0-4cb1-aefc-12088bc67328)

## What's it for?

Avahi, the common Linux implementation of Bonjour / mDNS / DNS-SD, allows
services to be published and discovered on a local network. Kinjo lets you
browse discovered services, filter and group them, and launch configured
actions for a selected service — all without leaving the terminal.

```sh
kinjo
```

For development or a quick look without a running Avahi setup:

```sh
kinjo --fake-discovery
```

## Install

**Homebrew** (macOS and Linux):

```sh
brew install abbyssoul/abyss/kinjo
```

**Debian / Ubuntu**, from the latest [GitHub release](https://github.com/abbyssoul/kinjo/releases):

```sh
sudo apt install ./kinjo_*_amd64.deb
```

**Cargo**, from [crates.io](https://crates.io/crates/kinjo):

```sh
cargo install kinjo
```

**From source**:

```sh
git clone https://github.com/abbyssoul/kinjo.git
cd kinjo
cargo install --path .
```

## How it works

Kinjo has three deliberately decoupled parts: a discovery backend that finds
DNS-SD/mDNS records on the network, a small rules engine that matches
services against user-defined commands, and a terminal UI that ties them
together. Each part sits behind a trait, so it can be swapped or extended
without touching the other two.

### Two discovery backends, and room for more

Discovery is pluggable:

- `mdns-sd` (default) — a pure-Rust backend built on the `mdns-sd-discovery`
  crate. A single browser enumerates every service type on the link via the
  native DNS-SD meta-query, so it needs nothing beyond network access.
- `zeroconf` — talks to the system Avahi daemon on Linux via `zeroconf-tokio`.
  It's opt-in (`cargo install kinjo --features zeroconf`) since it needs the
  Avahi client headers to build, but it's handy if you already run `avahi-daemon`
  and want Kinjo to use it directly.

Both backends implement the same `Discovery` trait and emit the same `Entry`
type, which is the only contract the rest of the app depends on. That seam is
intentional — a static file, an SSDP/UPnP browser, or any other service
source could be dropped in as a third backend without changing the rules
engine or the UI.

### Configurable keybindings

The defaults follow Vim-style conventions (`j`/`k` to move, `/` to filter,
`enter` to run an action, `?` for help — see the full list in the
[README](https://github.com/abbyssoul/kinjo#ui)), but every built-in UI
command can be rebound. Overrides live in a TOML file at
`$XDG_CONFIG_HOME/kinjo/keybindings.toml` (falling back to
`~/.config/kinjo/keybindings.toml`), so you can move to arrow keys, Emacs
bindings, or whatever fits your muscle memory. See
[docs/keybindings.md](https://github.com/abbyssoul/kinjo/blob/main/docs/keybindings.md)
for the complete reference and examples.

### Extending it through commands

The part of Kinjo that matters most is the command system. Discovered
services carry structured fields — name, type, domain, hostname, address,
port, and TXT record values — and a command file matches on those fields to
decide what actions are available for a given service. Nothing about the
match logic or the action it triggers is hardcoded into the app: it's all
data, read from `$XDG_CONFIG_HOME/kinjo/commands/*.toml` and reloaded live on
`SIGHUP`, no restart needed.

A command file is a match predicate plus a shell command template:

```toml
[metadata]
name = "ssh"
description = "SSH into a service"
requirements = ["ssh"]

[match.service_type]
equals = "_ssh._tcp"

[action]
description = "SSH into the selected service"
command = "ssh {hostname}"
mode = "execute"
```

Predicates support `equals`, `contains`, and `regex` against any service
field (including `txt.<key>`), and the same fields interpolate into the
command line, so `xdg-open http://{hostname}:{port}` or `ssh {hostname}` are
a few lines away. Multiple commands can match the same service — Kinjo shows
a picker when they do, and asks which instance to use if the fields it needs
are ambiguous. Full syntax, overlay rules, and more examples are in
[docs/actions.md](https://github.com/abbyssoul/kinjo/blob/main/docs/actions.md).

Because actions are just commands matched against whatever your network
already advertises, Kinjo works well as a jumping-off point for home labs:
today it's mostly SSH and web UIs, but the same mechanism could just as
easily launch a VNC viewer against a discovered `_rfb._tcp` service, open a
`psql`/`mysql` client against a discovered database instance, tail logs on a
matching host, or trigger any other local tool — without a single line of
Rust. If your home network already broadcasts its services over mDNS, Kinjo
can already be their launcher; it's mostly a matter of writing the right
command files.

## Try it and tell me what you think

Kinjo is young and actively developed, and the command system is deliberately
open-ended — if you run a home lab with a pile of self-hosted services, I'd
genuinely like to hear what actions you end up writing, what's awkward about
the command format, and what discovery backend you wish existed. Grab it with
`kinjo --fake-discovery` to poke around without any setup, then point it at
your real network:

```sh
kinjo
```

Bug reports, feature requests, and command file examples are all welcome on
the [issue tracker](https://github.com/abbyssoul/kinjo/issues) — and if you
build something cool with it, I'd love to know.

## Behind the build

Kinjo also gives me a place to write about engineering in the messier, more
interesting real world. In
[The crash that wasn't: debugging Kinjo's CI with AI]({% post_url 2026-07-19-the-crash-that-wasnt-debugging-kinjo-ci-with-ai %}),
I trace a CI-only failure to a difference between tmux 3.4 and 3.5a—and explain
how the diagnostic harness built during the investigation became the basis for
better user-facing bug reports.

## Links

- [Source on GitHub](https://github.com/abbyssoul/kinjo)
- [Crate on crates.io](https://crates.io/crates/kinjo)
- [Issue tracker](https://github.com/abbyssoul/kinjo/issues)
- License: [MIT](https://github.com/abbyssoul/kinjo/blob/main/LICENSE)
