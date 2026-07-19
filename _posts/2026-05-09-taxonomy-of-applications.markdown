---
layout: post
title:  "Taxonomy of applications"
description: "A practical taxonomy of scripts, interactive apps, services, and jobs based on runtime structure and users."
date:   2026-05-09 18:00:00 +1000
categories: engineering
tags: [engineering, reliability, testing, philosophy]
---



Over the course of a software engineering career, we find ourselves working on applications solving a massive variety of business needs. Yet, when looking for a way to categorize these systems, most existing literature focuses on the business domain or high-level architecture.

What we actually need is a taxonomy based on runtime structure and the number of supported users. From a structural standpoint, the applications we build generally fall into one of four buckets:

* **Script:** An application designed to automate a repeated action. It runs, does its job immediately, and quits.

* **App:** An interactive application that provides a UI (or TUI) to collect user input and perform actions.

* **Service:** A long-running process that accepts requests and performs actions on behalf of multiple users. It has no initial request and simply waits for connections.

* **Job:** A non-interactive application executed periodically by a system, usually to maintain performance or data integrity.


Understanding this taxonomy isn't just an academic exercise. Failing to distinguish between a single-user app and a multi-tenant service is the root cause of some of the most frustrating bugs in our industry.

## The Single-Player Fallacy

Universities and tutorials often teach programming as a single-player game. When junior engineers enter the industry, they bring assumptions developed from writing single-user scripts or running systems locally.

One of the most common failure modes is the assumption that database interactions do not require transaction isolation. In a local script, the world pauses for the user. You can put a record in a database, extract it, or modify it safely. However, in a multi-tenant service, the world keeps spinning. If you put a record into a database, it might no longer be there a millisecond later because another user's transaction has already modified or moved it.

## The In-Memory War Story

The single-player fallacy doesn't just impact database state; it ruins system observability.

I once worked on a deployment with three load-balanced instances. For troubleshooting purposes, an engineer introduced a REST API endpoint with a flag to enable extra debug logging. It was a very basic, naive approach.

The problem? The flag's value was stored in-memory on that specific instance, not persisted to a database. Enabling debugging on one instance did not enable it across the whole service. Because traffic was load-balanced, only a fraction of requests contained the extra debug information. For interactions requiring multiple user queries, the logs fragmented across different instances. We ended up losing information and creating a massive red herring for ourselves while troubleshooting.

<iframe class="interactive-demo" src="/assets/interactive/load_balancer_simulation-v2.html" width="100%" height="800" loading="lazy" title="Interactive load balancer and application state simulation"></iframe>

## The Solution: Teaching Complexity

This is ultimately an educational problem. Our local development environments actively lie to us. We deploy systems from our local machines and troubleshoot them there, completely blind to the fact that the production system has multiple instances and concurrent users.

To fix this, we need to bring local environments closer to the production state. I advocate for running containerized simulations of the production environment locally. You need to run a local database, but you also need to run multiple containerized instances of your service on your machine.

Yes, this creates a higher level of friction. But this is **teaching complexity**. It shatters the single-player illusion right at the developer's desk. The "click" shouldn't happen in the middle of the night while troubleshooting interleaved transactions in production; it should happen on a Tuesday afternoon during local testing.

## The AI Connection: The Modern Multiplier

Understanding your application's taxonomy is even more critical now that AI is part of our toolchain. AI offers great perspectives, but only if you know how to wield it.

* **AI as a Reviewer:** AI assistants are highly context-dependent. If you don't explicitly explain to the AI that you are developing a multi-user service, it will happily validate perfect "single-player" code. Knowing your taxonomy gives you the vocabulary to prompt the AI to look through the lens of transaction isolation and concurrency.

* **AI as a Consumer Simulator:** We can also use AI to enhance our "teaching complexity". Instead of just running multiple instances, we can use AI agents as consumers to run simulations. Imagine unleashing five AI users onto your local app concurrently, watching them step on each other's toes and expose data leaks they weren't supposed to see. It is the ultimate way to trigger the realization that your system is truly multi-user.


## Conclusion

We've had a brief tour of the various applications one can encounter. While this isn't an exhaustive list (embedded engineers would surely scream at me to include their domain), keeping this minimalistic framework in mind is essential for building safe software.

By recognizing whether we are building a script or a service, we can introduce the right teaching complexity into our workflows. We still haven't covered authentication, routing rules, or the wonders of modern platform engineering, but we will save those for a later discussion.
