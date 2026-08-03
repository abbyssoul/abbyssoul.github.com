---
layout: post
title:  "Kinjo: the home lab menu I wanted"
description: "Why I built Kinjo to turn the services advertised by my home lab into a short, useful list of things I can do."
date:   2026-08-03 12:00:00 +1000
categories: engineering
tags: [engineering, plan9, philosophy]
image:
  path: /assets/images/kinjo-home-lab/home-labbing-1200.webp
  width: 1200
  height: 800
  alt: A home-lab operator surrounded by servers, trying to remember which hostname to use
thumbnail: /assets/images/kinjo-home-lab/home-labbing-768.webp
---

{% include responsive-image.html
   small="/assets/images/kinjo-home-lab/home-labbing-768.webp"
   large="/assets/images/kinjo-home-lab/home-labbing-1200.webp"
   alt="A home-lab operator surrounded by servers, trying to remember which hostname to use while a terminal shows a long list of machines"
   loading="eager" %}

My home lab knows exactly what is running on it. I usually do not.

The machines announce their names and services over the local network. The
problem is the gap between that information existing and it being useful to a
human who just wants to get something done.

That gap is why I built [Kinjo](/projects/kinjo/), a terminal UI for discovering
local services and launching actions against them. This is not a guide to its
command files or keybindings—the [project page](/projects/kinjo/) has those
details. This is the story of the problem I wanted to solve, the older ideas
that shaped the design, and why I think a service browser should lead directly
to an action.

## Part I — My home lab remembers more than I do

I run a home lab with a collection of Raspberry Pis, virtual machines and
Docker containers. Every new service begins with good intentions. It gets a
name, an address and, quite often, an odd port number. For a while I remember
all of them.

Then the lab grows.

Was that dashboard on port 3000, 8000 or 8080? Which Raspberry Pi is running
the service I need? Is the old VM still around? I can put everything in a
spreadsheet, a wiki or a carefully maintained SSH config, but then I have
created another inventory that can drift away from reality. The network
already has a more current answer: many of these machines advertise their
services through mDNS and DNS-SD.

Linux can show me those advertisements with Avahi. It works, but its output is
made for inspecting discovery records rather than answering the question in my
head. It gives me plenty of information about interfaces, protocols, service
types and domains. Most of that is useful to a computer, or to a human
debugging the network. It is the wrong information for choosing what to do
next.

What I want is closer to this:

> There are three machines you can SSH into. Two are `raspberrypi-0` and
> `raspberrypi-1`. The third is that VM you forgot about. :)

That is a useful list. More importantly, it should not be the end of the
interaction. Once I have selected the forgotten VM, I want to be inside an SSH
session—not copying its hostname from one command and pasting it into another.

Sometimes the destination is not a shell. It is a web dashboard, a database
client, a remote desktop or something peculiar to one service in my lab. The
list can stay simple, but the next action cannot be hardcoded. Different
services imply different verbs.

That was the actual problem: not discovering more data, but reducing discovered
data to a choice and carrying that choice into the right tool.

## Part II — The idea has ancestors: plumbing and menus

This is not a new class of problem. Operating systems and desktop environments
have been deciding how to connect a thing to an application for a long time.

[Plan 9 plumbing](https://9p.io/wiki/plan9/Using_plumbing/index.html) gives the
idea a wonderfully physical name. Applications send messages to the plumber;
rules inspect those messages and dispatch them to an appropriate destination.
A URL can go to a browser, an image to a viewer, and a compiler error to an
editor positioned at the relevant line. The source does not need to know every
possible receiver. The policy lives in configurable rules between them.

Linux desktop users encounter a narrower version of the same idea through
[`xdg-open`](https://portland.freedesktop.org/doc/xdg-open.html). Give it a file
or URL and it opens the preferred application for that kind of resource. This
is extremely handy when there is a sensible default. It becomes less helpful
when context matters, or when more than one action is reasonable.

There is another family of tools that concentrates on the moment of choice.
[`dmenu`](https://tools.suckless.org/dmenu/) turns arbitrary lines of text into
a small searchable menu and returns the selected line. Rofi expands the same
pattern into an application launcher, window switcher and scriptable selector;
Adam Simpson's [introduction to Rofi](https://adamsimpson.net/writing/getting-started-with-rofi)
is a good demonstration of how a simple popup can become the human-facing part
of a shell workflow.

These tools solve adjacent pieces of my problem:

- Avahi discovers services.
- `dmenu` and Rofi make a list pleasant to select from.
- `xdg-open` delegates a resource to a preferred application.
- Plan 9 plumbing separates recognition and dispatch through rules.

I could glue some of those pieces together with a shell script. For a fixed
workflow, that might even be the right solution. But my home lab is not a fixed
workflow. Services appear and disappear, one advertised service may support
several useful actions, and the same kind of service may need different
handling depending on its name or metadata.

An honourable mention belongs to
[Television](https://alexpasmantier.github.io/television/), or `tv`, a fast
terminal fuzzy finder I find myself using for almost everything. It comes very
close to solving this problem. A custom channel can run a command to produce
items, refresh a changing source and attach actions to the selected result. I
could give it the output of a service-discovery command and get a much better
picker than the raw Avahi output.

But discovery is a live stream rather than a list I want to regenerate
periodically, and the available action depends on the metadata of the selected
service. When several rules match, I want the interface to ask “Which one?” at
that moment. I do not want to remember that one key opens the browser while
another starts SSH. Television solves general-purpose selection beautifully.
Kinjo needs to understand the two things being selected: first a service, then
one of the actions that makes sense for that service.

The design lesson I took from these predecessors was not that Kinjo needed to
copy any one of them. It was that discovery, selection and dispatch are separate
concerns. A useful tool should connect them without welding them together.

## Part III — Kinjo turns discovery into a next action

Kinjo starts with what the network is already saying. It listens for local
DNS-SD and mDNS advertisements and turns them into a compact, filterable list.
Instead of treating every field in a service record as equally important, the
interface keeps the service identity prominent and leaves the details
available when they are actually needed.

So the first half of my SSH example becomes straightforward: start Kinjo and
the three SSH-capable machines appear. I can type part of a name to narrow the
list, move to the forgotten VM and select it.

The second half is where the design becomes useful. Kinjo matches the selected
service against configurable command rules. An SSH service can offer an action
that starts `ssh` with the discovered hostname. A web service can offer an
action that builds its URL from the discovered host and port and passes it to a
browser. If several actions make sense, Kinjo asks which one I want.

This is deliberately a small amount of policy. Kinjo does not need built-in
knowledge of every dashboard, database or remote-access tool I might add to my
lab. It needs to know what the network advertised, which rules match, and how
to substitute the selected service's details into the chosen command. The
configuration carries the part that is personal to my setup.

### The first Kinjo was written in C++

Kinjo was not my first attempt to solve this problem. I started by implementing
it in C++, and learned two useful things.

First, C++ terminal UI libraries are quite usable. The interface was not the
problem. I could draw a list, handle keyboard input and build the interaction I
had in mind without much drama.

The trouble started where the UI met Avahi. A terminal application already has
an event loop: it waits for input, redraws when state changes and reacts to
timers. Service discovery has its own file descriptors, timeouts and callbacks.
For the application to remain responsive, those two worlds need to cooperate
through `poll` or `select`.

The C++ wrapper I tried exposed the discovery operations, but not enough of the
underlying polling interface to integrate cleanly with the rest of the
application. It presented a usable happy path while hiding the exact seam an
interactive program needed.

That left me with a slightly uncharitable—but still useful—lesson about library
design: authors often think about what their library can do, but not how a user
will compose it with everything else in a real application. A service browser
API is only half an API if it assumes it can own the event loop. The underlying
Avahi polling abstraction exists precisely to connect it to other loops; a
wrapper that does not expose that capability makes the easy example possible
and the real program awkward.

I could have written more C++ glue, but integrating somebody else's abstraction
was becoming the project instead of building Kinjo. That pushed me to try Rust
and [Ratatui](https://ratatui.rs/). Ratatui made the terminal interface pleasant
to build, and the pieces composed well enough that Kinjo quickly began to feel
like the tool I had imagined.

It did not make service discovery easy.

### Cross-platform discovery was the harder problem

DNS-SD is a standard, but the ways to access it are not uniform. Linux commonly
uses Avahi, Apple platforms have Bonjour, Windows has its own facilities, and a
pure-Rust implementation can bypass those system services at the cost of owning
more of the protocol behaviour itself.

Many Rust crates I found made publishing a service the main path. Publishing is
important, but Kinjo needs the other direction. It must browse a network whose
contents it does not know in advance, notice service types and instances as
they appear, resolve their hostnames, ports, addresses and TXT records, and
remove them again when they disappear. Those records can arrive separately and
out of order. Discovery is not a function that returns a finished vector; it is
an evolving stream of partial information.

I found an existing crate with an interface close to what I wanted and
implemented the service-discovery half for it. I opened
[the pull request](https://github.com/bzld/dns-sd-native/pull/1), hoping the work
could become part of the original library. At the time of writing, three months
later, the PR is still open. That is a normal risk of open-source
contribution—maintainers do not owe me their timetable—but Kinjo could not
remain blocked on it.

So the discovery work became its own crate:
[`mdns-sd-discovery`](https://crates.io/crates/mdns-sd-discovery/0.3.0). What
began as a missing dependency had turned into a useful project of its own: a
cross-platform, asynchronous interface shaped around browsing rather than only
announcing services.

Then a colleague who knows considerably more about networking than I do gave me
one more piece of advice: do not make Kinjo depend forever on a single discovery
mechanism. Make the backend replaceable.

That suggestion became an important part of the design. Kinjo now consumes one
stream of discovery events regardless of where they came from. An OS-backed
implementation, a direct protocol implementation or a deterministic fake
source can sit behind the same boundary. A future discovery mechanism can be
added without changing the rules engine or the UI.

This is more than defensive abstraction. Cross-platform discovery has enough
environment-specific behaviour that there may never be one universally best
backend. Making it replaceable lets the rest of Kinjo depend on the meaning of
an event—service added, updated or removed—rather than the machinery that found
it.

That separation matters because the useful action is contextual. `_ssh._tcp`
usually means “open a shell” for me, while an HTTP service usually means “open
this page.” Another person may prefer a particular terminal, browser or wrapper
script. Even within my lab, a service name or TXT record may be the detail that
changes what I want to launch.

The result is the interaction I was after from the beginning:

1. See what is available now.
2. Select the thing I recognise—or the VM I had forgotten.
3. Continue directly into the action that makes sense for it.

There is still complexity underneath. Service discovery is asynchronous,
records arrive in pieces, addresses can be ambiguous, and user-defined commands
need to be handled carefully. But that complexity belongs inside the tool. The
user-facing model can remain a list of nearby things and a set of relevant
actions.

## The network should be the inventory

Kinjo began with a very small annoyance: I could not be bothered to remember
the names and odd ports of every machine and service in my home lab. Existing
tools could expose all the raw information, but they stopped just before the
part I cared about—using it.

Plan 9 plumbing showed how rules can separate a message from its destination.
`dmenu` and Rofi showed how little interface is needed to make a generated list
useful. `xdg-open` showed the convenience of delegating a resource to another
application. Kinjo brings those ideas to local service discovery: find what is
there, make it easy to choose, then hand the choice to a configurable action.

Building it reinforced the same lesson from the other direction. The C++
attempt showed what happens when a library hides the integration point its
users need. The search for a Rust discovery crate eventually produced
`mdns-sd-discovery`. A colleague's suggestion turned that dependency into a
replaceable backend rather than a permanent constraint. The useful flexibility
is not only in which command Kinjo launches; it also lies in how Kinjo learns
what is on the network.

The network already maintains a living description of much of my lab. I do not
want to memorise a second, increasingly fictional copy of it. I want to see the
short list, choose the forgotten machine and get on with whatever I opened the
terminal to do.

That is what [Kinjo](/projects/kinjo/) is for.
