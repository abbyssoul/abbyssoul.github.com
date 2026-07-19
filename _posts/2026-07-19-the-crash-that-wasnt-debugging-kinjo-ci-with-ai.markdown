---
layout: post
title:  "The crash that wasn't: debugging Kinjo's CI with AI"
date:   2026-07-19 12:00:00 +1000
categories: engineering
tags: [engineering, reliability, testing]
---

![Debugging with AI](/assets/images/the-crash-that-wasnt/the_kinjo_failure_that_wasnt.png)
A CI smoke test told me [Kinjo](/projects/kinjo/) had crashed. Kinjo had actually exited successfully.

That distinction took a few hours, several AI sessions, a handful of discarded
hypotheses, and a small crash-capture toolkit to uncover.

I was updating dependencies in [Kinjo](/projects/kinjo/),
my side project for browsing local DNS-SD and mDNS services from a terminal UI.
Its CI suite includes a smoke test that starts [Kinjo](/projects/kinjo/) inside tmux, sends it some
keystrokes, and checks that it quits cleanly. Intermittently, the test instead
reported:

```text
drive-tui: pane exited via a signal (no exit code)
```

It looked like a classic application crash. There was only one problem: I could
not reproduce it locally. Not after more than 250 runs, not under AddressSanitizer,
and not while restricting the machine to a single CPU core to encourage a timing
failure.

This is a story about debugging a problem that only existed in CI, but it is also
about what productive AI-assisted engineering actually looks like. AI made each
hypothesis cheap to test. It did not supply the decisive intuition. Most
importantly, the work spent investigating the false crash did not disappear: it
left [Kinjo](/projects/kinjo/) with much better machinery for diagnosing the next real one.

## AI made the experimental loop fast

The most useful part of working with AI was not a single brilliant answer. It was
the speed of the guess-test loop.

We could form a hypothesis, change the test harness, push an experiment through
CI, inspect the evidence, and move on. We captured the TUI's stderr separately.
We changed the driver to distinguish a normal exit from a signal death. We
bisected dependency changes. We enabled core dumps and arranged to collect gdb
backtraces, kernel messages, journal entries, and application logs after a
failure.

Several of those experiments ruled things out rather than finding the cause. A
suspected dependency was not responsible. Reverting other dependencies changed
how often the failure appeared but did not remove it, which was a useful clue in
itself: we were probably looking at a delicate timing or layout effect, not a bad
code path in one particular crate.

This is where AI felt genuinely enabling. Preparing every variation by hand would
have taken much longer. The machine did not eliminate the investigative work; it
compressed the mechanical time between an idea and evidence about that idea.

![Debugging with AI](/assets/images/the-crash-that-wasnt/guess_try_loop.png)

## The decisive guess was a human one

Because the failure would not reproduce on my local machine, despite many attempts, I eventually wondered
whether the `tmux` server itself was different.

That was not a random implementation detail. The smoke test needs a real
pseudo-terminal to drive a terminal UI, and `tmux` is part of that environment. If
`tmux` behaves differently, then the test behaves differently — even when Kinjo is
the same binary.

The version check exposed exactly that difference:

- GitHub Actions was using the Ubuntu package of tmux 3.4.
- My development machine was using tmux 3.5a.

AI then helped turn the suspicion into a controlled experiment. On the same
Ubuntu 24.04 runner, we looped a narrow smoke test twelve times with each `tmux`
version:

| Environment | Result |
|---|---:|
| tmux 3.4 from the Ubuntu package | 8 failures out of 12 runs |
| tmux 3.5a built from source | 0 failures out of 12 runs |

Same runner, same application, same test, different `tmux`. The failure tracked
the tmux version. That also explained why hundreds of local attempts had told me
nothing: my local environment did not contain the component that produced the
failure.

This is the balance I want to preserve when talking about AI-assisted debugging.
AI was excellent at validating hypotheses and rapidly building the experiments
needed to challenge them. _I_ was still instrumental in choosing which difference
to investigate. The useful unit was neither "human intuition" nor "AI output" in
isolation; it was the speed with which the two could turn a hunch into a result.

## The process that died was not Kinjo

Finding the version correlation explained where to look, but the most satisfying
piece of evidence came from a small foreground-preserving launcher we added to
the harness.

Normally, Kinjo under a smoke test _is_ the tmux pane process. That makes the pane's fate and the
application's fate look like the same thing. The launcher separated them. It
became the pane process, started Kinjo as a child in the terminal foreground,
and recorded how both processes ended.

On a failing run it recorded that the Kinjo child exited with status 0. The
launcher—the tmux pane process—was the process that received a signal.

In other words, Kinjo had handled the quit command and shut down correctly! Phew. 
`tmux` 3.4 then signalled the pane process during teardown. In the original test, where
Kinjo itself occupied that role, the signal raced its clean exit and masqueraded
as an application crash.

We never pinned down the exact signal, and at that point we did not need to. It
did not produce a core dump, there was no kernel segfault or out-of-memory event,
and adding more signal handling changed the timing enough to make the failure
disappear. The controlled tmux-version experiment and the child exit status were
enough to establish the cause.

The CI fix was straightforward: build `tmux` 3.5a for the smoke-test job and cache
it. The supposedly crashing test became deterministic and green.

## AI debugging still has a bill

It would be easy to turn this into a story about AI making debugging effortless.
That would not be honest, and just hype. The investigation was not effortless. It was still a lot of work.

The investigation consumed a non-trivial number of tokens and spanned several
sessions. We maintained a handover document so that each new session inherited
the hypotheses, experiments, branch state, and conclusions from the previous
one. There was still waiting for CI, interpreting ambiguous results, and deciding
what to try next.

But conventional debugging has a bill too. Every experiment has to be prepared,
reviewed, run, and interpreted by somebody. On a commercial project those are
billable engineering hours; on a side project they are evenings and attention.
An engineer manually building every diagnostic variation could easily have cost
more than the tokens, especially for a failure they could never reproduce on
their own machine.

That does not mean token cost should be ignored. It means the comparison should
be honest. The value was not that the investigation was free. The value was that
_AI_ let me run a much wider and faster sequence of experiments with the limited
human time I had available. My own time.

## The diagnostic work became part of the product

The best outcome is not merely that CI is green again.

To find this failure, [Kinjo](/projects/kinjo/) gained a reusable crash-capture toolkit: a launcher
that distinguishes the application's exit from its terminal wrapper, a way to
enable and collect core dumps, automatic gdb backtraces, stderr capture, and a
failure report that gathers the relevant system evidence without hiding the
original test result.

None of that is specific to `tmux` 3.4. The same mechanism can be switched on for a
future CI flake, and the ideas can support user-facing bug reports when [Kinjo](/projects/kinjo/)
fails in an environment I do not control. Instead of asking someone to describe
"it crashed" from memory, the project can collect evidence that answers much
better questions: Did [Kinjo](/projects/kinjo/) receive a signal? Did it exit normally? Was there a
core dump? What did stderr contain? What did the operating system observe?

The original failure was a false accusation against the application. Chasing it
made the application easier to diagnose when a real failure arrives.

## A bug hunt can produce more than a fix

A CI-only failure is often described as hard to reproduce. In this case it was
impossible to reproduce locally because the relevant difference was not in
[Kinjo](/projects/kinjo/) at all; it was in the `tmux` server around it.

AI made this investigation practical by shortening the path from hypothesis to
experiment. Human context and intuition still supplied the question that broke
the case open. The process cost real time and real tokens, just as a traditional
bug hunt costs real engineering hours. What made it worthwhile was both the
speed of learning and what remained afterwards.

[Kinjo](/projects/kinjo/) did not have a crash. It now has a better crash-reporting mechanism.

That is a win.
