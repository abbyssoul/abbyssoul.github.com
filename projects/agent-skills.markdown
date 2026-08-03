---
layout: page
title: Agent Skills
permalink: /projects/agent-skills/
description: >-
  A version-controlled collection of reusable AI agent skills and specialist
  agent personas for practical, repeatable AI-enabled engineering.
---

**Versioning the behaviours and workflows I use to engineer with AI.**

[![Source on GitHub](https://img.shields.io/badge/source-GitHub-181717?logo=github)](https://github.com/abbyssoul/agent-skills)
[![License: Apache-2.0](https://img.shields.io/github/license/abbyssoul/agent-skills)](https://github.com/abbyssoul/agent-skills/blob/main/LICENSE)

![A collection of specialised AI agents](https://raw.githubusercontent.com/abbyssoul/agent-skills/main/assets/agents-skills.png)

## Why I built it

An AI coding tool can generate code. The harder and more interesting problem is
making the collaboration consistent: when should the agent ask questions before
acting? How should it diagnose a failure? Which perspective should challenge an
architecture, and what should the result look like?

Those decisions are engineering knowledge. I built Agent Skills to keep that
knowledge as ordinary, inspectable files rather than losing it in one-off chat
sessions or hiding it inside a large framework. The collection lives in Git, so
the instructions can be reviewed, refined, shared across projects, and rolled
back when an experiment makes them worse.

## Skills and agents solve different problems

The repository contains two related kinds of reusable behaviour:

- **Skills** describe focused workflows that an agent loads when a task matches.
  The current collection includes `idea-interview`, for discovering and
  sharpening a point of view, and `blog-writer`, for turning the resulting
  material into a standalone technical article.
- **Agents** establish a specialist role, its boundaries, and its expected
  output. They include an architectural critic, product and technical product
  owners, a reliability engineer, a troubleshooter, a podcast host, and a
  precision reasoning persona.

The distinction matters. A role answers _who should think about this?_ A skill
answers _what repeatable method should they follow?_ Keeping the two small and
composable makes them useful beyond the project where an idea first emerged.

One workflow used to develop this blog is deliberately simple:

```text
rough idea
  → idea-interview
  → sharpened thesis, examples, and objections
  → blog-writer
  → Jekyll article
```

The value is not the diagram. It is that each transition has explicit
expectations, so the human remains involved in the decisions while the agent
handles more of the mechanical work.

## AI-enabled engineering is more than prompting

I treat agent behaviour as part of the engineering environment. Useful
instructions need clear triggers, narrow responsibilities, and boundaries that
stop a specialist from silently becoming a general-purpose decision maker.
They also need maintenance: prompts drift, assumptions leak out of their
original projects, and a workflow that sounds good on paper may be awkward in
real use.

Agent Skills is where I work on those problems in public. It reflects the same
principles I apply to software: make responsibilities explicit, prefer small
composable pieces, keep the source readable, and use version control to preserve
the reasoning behind change.

## Plain files, linked into local tools

The implementation stays intentionally close to the metal:

```text
.
├── agents/     # specialist personas
├── skills/     # reusable workflows, one SKILL.md per skill
└── scripts/    # local linking and discovery helpers
```

The linker creates symlinks into supported runtime directories that already
exist, including `~/.claude/skills`, `~/.claude/agents`, `~/.agents/skills`, and
`~/.config/opencode/agents`. Because the links point back to the repository, an
edit is immediately available to the local tools without copying prompt files
between them.

To try the collection:

```sh
git clone https://github.com/abbyssoul/agent-skills.git
cd agent-skills
./scripts/link-skills.sh
./scripts/list-skills.sh
```

The repository also includes `./scripts/unlink-skills.sh` to remove symlinks
that point back to the checkout.

## Links

- [Source and setup instructions](https://github.com/abbyssoul/agent-skills)
- [Skills](https://github.com/abbyssoul/agent-skills/tree/main/skills)
- [Agents](https://github.com/abbyssoul/agent-skills/tree/main/agents)
- License: [Apache-2.0](https://github.com/abbyssoul/agent-skills/blob/main/LICENSE)
