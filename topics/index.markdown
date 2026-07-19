---
layout: page
title: Topics
permalink: /topics/
description: Curated paths through the recurring ideas in this blog.
---

<section id="reliability" class="topic-section">
  <p class="eyebrow">Reliability &amp; testing</p>
  <h2>Make failures understandable—and change less frightening</h2>
  <p>
    These essays look at error models, automated testing, runtime assumptions,
    and the support structures that let engineers change systems with confidence.
  </p>
  <div class="curated-list">
    <a href="{% post_url 2026-07-19-the-crash-that-wasnt-debugging-kinjo-ci-with-ai %}">
      <strong>The crash that wasn't</strong>
      <span>Debugging a CI-only failure with AI and turning diagnostics into product capability.</span>
    </a>
    <a href="{% post_url 2020-11-03-no-time-for-testing %}">
      <strong>No time for testing?</strong>
      <span>Why testing is part of delivery, not optional work after implementation.</span>
    </a>
    <a href="{% post_url 2019-10-01-thinking-about-error %}">
      <strong>How to think about errors in your code</strong>
      <span>A practical model for deciding what failure means across applications and services.</span>
    </a>
    <a href="{% post_url 2026-05-09-taxonomy-of-applications %}">
      <strong>A taxonomy of applications</strong>
      <span>Scripts, apps, services, and jobs—and why their runtime shape changes engineering decisions.</span>
    </a>
  </div>
</section>

<section id="distributed-systems" class="topic-section">
  <p class="eyebrow">Distributed systems</p>
  <h2>Understand why the machines need each other</h2>
  <p>
    Start with the economic motivation for distribution, then move into communication,
    resource naming, and the abstractions that make remote systems feel local.
  </p>
  <div class="curated-list">
    <a href="{% post_url 2020-06-06-whys-of-distributed-system %}">
      <strong>Why distributed systems are useful</strong>
      <span>The problems that justify distribution in time, space, or both.</span>
    </a>
    <a href="{% post_url 2019-11-25-vfs-for-fun-and-profit %}">
      <strong>Virtual filesystem: fun and profit</strong>
      <span>Using paths as a general interface to local and remote resources.</span>
    </a>
    <a href="{% post_url 2019-10-30-libstyxe-great-refactoring %}">
      <strong>libstyxe: Great Refactoring</strong>
      <span>Redesigning a 9P protocol library for extensibility.</span>
    </a>
  </div>
</section>

<section id="building-in-public" class="topic-section">
  <p class="eyebrow">Building in public</p>
  <h2>Learn through the projects themselves</h2>
  <p>
    Side projects are where architectural ideas meet awkward environments,
    dependency constraints, users, packaging, and the occasional false crash.
  </p>
  <div class="curated-list">
    <a href="{{ '/projects/kinjo/' | relative_url }}">
      <strong>Kinjo</strong>
      <span>A terminal UI for discovering and acting on local DNS-SD and mDNS services.</span>
    </a>
    <a href="{% post_url 2026-07-19-the-crash-that-wasnt-debugging-kinjo-ci-with-ai %}">
      <strong>Debugging Kinjo's CI with AI</strong>
      <span>A concrete account of human intuition, fast experiments, and durable tooling.</span>
    </a>
    <a href="{% post_url 2019-10-07-libsolace-philosophy %}">
      <strong>libsolace: library philosophy</strong>
      <span>Capturing repeatedly useful engineering primitives in a focused C++ library.</span>
    </a>
  </div>
</section>
