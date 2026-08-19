# What This Project Is About

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

This project tells the story of my work as a vibecoder and the path I've taken working alongside
AI agents. It's a living project — I'll be filling it out and expanding it regularly.
You can simply read through it and take whatever is useful, or you can fork it, point your own
agent at it and ask it to find features and details worth borrowing for your project.

I'm a systems analyst and Tech & Area PO in fintech, a lecturer in systems analysis at HSE
University, co-founder of an analysts' school, an investor and a musician.

**I don't write code by hand.** I design systems and make the architecture and product calls,
while AI agents do the implementation. Between April and August 2026 that's produced **ten
systems and more than 2,000 commits** — from algorithmic trading to a production platform that a
client actually uses. Seven of them are described here.

The repositories themselves are private. This one is about the **method**: how the work is set
up, which rules were paid for with failures, and how you verify a result when you don't read
code. It also collects the README and CLAUDE.md files from each project — and inside each
project's README you'll find the system in detail, my path through it, and the mistakes I
collected along the way.

**P.S.** These project descriptions were also written together with Claude. My goal was never a
polished, exhaustive write-up — I just needed to get the idea across.

If you want details, want to see the code, or want an example of how I actually work with
agents — write to me on Telegram: [@Basov_VA](https://t.me/Basov_VA). If you want to see my
professional background, here is [my CV](CV_Basov_Vladimir.pdf) (in Russian).

## THE BASICS — the part that matters most (details further down)

**THE BASICS** — AI writes the code. And code written by AI is what has to check that same AI. To pull that off, you have to learn point 1 and point 2 below.

**1. You have to understand what you're building.** When you're making an application, a
product, or an advanced agent, you don't have to understand the code and its details — thank
you, AI, for that. But you have to understand, a thousand percent, what you want to get, how it
should work, and how you'll know it's the thing you wanted. Hand the wheel to an AI completely
and you won't understand where it takes you.

I have a project called myTrade, where I do
algorithmic trading (details inside the project). I don't understand 100% of the maths or of
the test logic, but I do understand logically how it should work, and I ask the AI questions —
about logic and business, at every step. On top of that, and in parallel, the agent maintains a
wiki for me personally, so that I can see that what it writes in code makes logical and business
sense.

**2. You have to understand how to verify the result and confirm that point 1 actually
happened.** You don't know what's in the code, and the agent makes a great many decisions along
the way. You can't climb inside the agent's head, and you don't need to. Climb inside your own
instead. An agent can hold an enormous amount of content in its head and you will never know
what's really in there. But you *can* write code that checks, algorithmically and
unambiguously, what actually came out. Gates like that (see below) are the foundation of all my
tests. And since the same agent writes that code, to check whether the test itself is written
correctly — see point 1. Yes, there is subagent work, which exists precisely to solve the
"second pair of eyes" problem, but a subagent is the same kind of agent with the same inner
world you'll never see into.

Another example: my reports project, where a large task was
processing PDF financial statements from many issuers. Claude diligently checked everything,
loaded it into the database, moved files into "recognised" — and in the end it turned out that
many files had been moved without any review at all, "just because". There were plenty of checks
for recognition accuracy and a pile of maths on the numbers, but there was no test that checked
Claude's *diligence*. And that's exactly the horror case that surfaced: verification of the work
done, 100%; verification that the work was done at all, 0%.

**3. Context and breadth of thought are yours.** At the start of a project there's no problem.
Once it's big and heavy, only you can see that a change made for the latest feature might touch
something from the very beginning of the project. Only you can "think wide" and see the
whole thing from above. So I'll simply repeat point 1 here — you have to understand what you're
building.

Trust Claude with 100% of the project and you'll walk in circles, building a new piece
at the end while breaking a piece at the start. For a more concrete sense of this, look at the
architecture in myTrade or write to me.

**4. The neural net understands itself better than it understands you.** Every one of my
projects has a mandatory rule about AI comments. Claude — or any other model — leaves comments
addressed to itself, right there in the text, so that it doesn't trip over the same rake twice
and makes fewer mistakes overall. This works better than keeping documentation somewhere and
making notes of that kind off to the side.

There are plenty of ways to optimise development
(graphs, Qoder's wiki for development, and so on) — I'm sure you'll find the one you need. What
I use is comments plus grep on the AI's side within a project. The main thing is to use
something, and ideally from the very start of the project.

## How the work is arranged

At any given moment I have **3–4 projects running autonomously** and **one that needs my full
attention**. When I have time — weekends, say — the autonomous count swells to 5–6 (I try not to
abuse my hardware too much).
An autonomous project isn't a paused one: a coder loop is turning through a work order there,
the gate decides on its own whether to accept a batch, and I step in at acceptance. The project
that needs my attention is the one where something new is being designed — where my head is
actually needed.

That's why most of how I work is built around acceptance. When there are several projects and
one human, only one thing scales: **machine verification instead of reading code**.

**Tools:** Claude Max (Claude Code) and Qoder Pro — Kimi K3 and Qwen 3.8Max work through the
latter. The coder is DeepSeek v4 Pro in an autonomous CLI loop. The three architects are equal
and interchangeable, each with its own memory; the coder has no memory, which is why a work
order has to be self-sufficient. GLM-5.2 used to be a coder here too, but it got made redundant.

## How I work with the agent

Every project page describes this loop with its own specifics, but the skeleton is the same.

I look at the plan to see what's left and give the command by task code. The agent picks up the
context itself: the code leads into the plan, into the work order and into git history, so I
don't have to restate the task. Then it decides whether to do the work itself or write a work
order for the coder — **choosing the executor is always the architect's call**.

After every closed task I ask for a report: what changed, what got fixed, where to look at the
result. **No merges happen without me, in any project.** Before a merge comes a full test run —
not just the tests near the change.

Then I go into the task myself — selectively, where the logic is tricky or where data could get
quietly corrupted. Into the logic, not the code: how it works now, what changed in the
behaviour, where that flows next. Far too many times we've carefully built something in one
place and broken something we hadn't touched in two months. The agent won't catch that — its
focus is its own task, while the system as a whole stays with me.

## The principle everything is built around

If you don't read code, "done" from an executor means nothing. Neither does "tests are green",
nor "I checked everything". So you need a way to accept work without opening the diff.

Hence four pillars.

**1. Separation of roles.** The PO brings a task at the level of value, goal, and business and
user requirements. The architect decides **how** to build it. The coder works to the order and
doesn't improvise on architecture.

**Choosing the executor is always the architect's call**, no exceptions: it decides whether to
do the work itself or hand it to the coder. **Decomposition** we do together with the
architect — except in staffing, where I always do it myself, taking on the product analyst role
as well.

There are three architects and they are **equal**: Claude Opus, Kimi K3, Qwen 3.8Max. One
process for all of them, no personal territories. The coder is DeepSeek in an autonomous loop;
I treat the iterations inside that loop as free.

⛔ **One task = one executor and one status. One batch = one executor.** If two of them do the
work, that's two tasks with different numbers, not one task in two batches. The rule looks
bureaucratic right up until the first violation: a task with two executors has **nowhere to
record a status** after the first acceptance — it's neither done nor not-done, it hangs, and one
session later nobody remembers whose half was closed.

**2. An executable gate instead of prose criteria.** An acceptance criterion isn't "the data is
displayed correctly" — it's a command that prints `✅`/`⛔` and returns an exit code. The gate is
written **before** the work order. A batch without a green gate isn't reviewed by the architect
at all; a red gate is returned without discussion, because the gate output *is* the list of
fixes.

This comes out of two observations: the expensive part isn't writing the task, it's accepting
the result; and green tests from the executor prove nothing — the fixture is written by the same
head that wrote the code.

**3. Data that doesn't lie.** Don't know something? Empty plus a note — never a plausible-looking
value, because an absence is visible and an invention isn't. Every value carries a source and a
verification date. You may not pass off unread as read. In systems about money and health, this
is table stakes.

**4. Tests, tests, and maybe a few more tests.** Every feature and every business function is
covered by tests without exception: automated tests for features, E2E for business flows. A run
after each task and each batch, and before acceptance on my side — a full run of everything plus
E2E.

Plus two rules that save the most: **agent memory holds only state and rules of behaviour**,
with all the substance in git; and **we count the limit, not the money** — consumption equals
context length multiplied by the number of requests, which is why one phase equals one session.

The full version, with the reasoning and the price paid for each rule, is in
[CLAUDE.en.md](CLAUDE.en.md).

## Stack and hardware

I chose the stack together with Claude and keep it deliberately narrow: the fewer distinct
technologies, the more decisions carry over between projects.

**Core stack:** Python 3.11–3.13 (uv) · FastAPI · SQLAlchemy / SQLModel + Alembic ·
Celery + Redis · PostgreSQL, TimescaleDB, SQLite · React 19 + TypeScript + Vite +
Ant Design · Flutter.

**Everything lives in Docker.** Tests: pytest · Vitest · Playwright.

**Hardware — I run the whole fleet myself, together with Claude:**

- **MacBook Pro M5 Pro, 24 GB** — the main working machine. All architect and coder sessions run
  here, and deployments go from here too (P.S. the plan is to move the coder onto the compute
  node so the poor thing can work 24/7)
- **Compute node** — i9-12900K (24 threads), 94 GB RAM, ~5 TB NVMe, Tesla V100 16 GB GPU (the
  Tesla is idle for now — the plan is to run AI on it). Proxmox, Debian, Docker + TimescaleDB.
  The database is ~155–180 GB with over 1 billion rows of quotes (as of 14.08.26). Postgres is
  tuned for the volume, backups are dual-circuit (VM snapshot + `pg_dump`) with a quarterly
  restore test. 99% of it serves heavy maths and data accumulation for myTrade.
- **Trading VPS** — a separate machine for the live crypto robot and nothing else: computation
  and the database stay on the node. Latency to the broker is ~2.5 ms, configuration as code
- **UAT environment** — where the release train gets accepted, living on its own branch;
  staffing only
- **Production environment** — the platform's live circuit: HTTPS, one-command release,
  migrations by procedure; staffing only
- **Telegram bot server** — shared across four projects: systemd timers and Docker, plus a
  registry of who lives there with ports and memory limits, so the neighbours don't break each
  other

Infrastructure is configured **as code** — through scripts and runbooks in the repositories, not
by hand from memory. Addresses, keys and paths are not published in this showcase.

## How to read this repository

| File | What's inside |
|---|---|
| [CLAUDE.en.md](CLAUDE.en.md) | ⭐ **The main thing.** The shared rulebook: roles, gate, data, tests, memory, economics. A working file — drop it in a repository root and it works |
| [AGENTS.md](AGENTS.md) | Entry point for non-Claude agents and a reference for coders |
| project folders | For each one: a description of the system and its own `CLAUDE.md` |

`CLAUDE.md` here is deliberately complete — it shows every rule and the grounds for it. In a
real project, once it's been run in, it's worth trimming to fit: drop what doesn't apply, move
part of it into agent memory. How to do that is written in the file itself.

## Projects

| Project | What it is | Scale | Status | Lead AI executor (architect) |
|---|---|---|---|---|
| [mytrade](01-mytrade/README.en.md) | The flagship — algorithmic trading on MOEX and crypto: three independent engines, a strategy research pipeline with walk-forward and deflation | 623 commits, 06–08.2026 | in active development, first live tests under way | Fable 5 |
| [staffing](02-staffing/README.en.md) | An outstaffing recruitment platform: vacancy parsing, candidate scoring, interview checklists, IT market analytics. Currently expanding in the mirror direction — matching positions for individuals beyond outstaffing | 449 commits, 04–08.2026 | **live production**, at MVP stage with client demos | Opus 5 |
| notifications | A notifications microservice: templates, routing, delivery log. Designed and delivered turnkey — a full production solution | 143 commits, 07.2026 | **rolled out into a commercial circuit**; happy to describe it in person (not for this repository) | Opus 5 |
| [reports](03-reports/README.en.md) | A personal investment system: financial statements, news, prices and portfolio accumulate in a database and turn into decisions about issuers on a fixed cadence | 601 commits, 06–08.2026 | working, development plan in progress | Opus 5 |
| [paperbirds](04-paperbirds/README.en.md) | A co-pilot for my indie band: finding venues and open calls, a contacts database, pitch drafts | 207 commits, 07–08.2026 | functionality works, ~30% of the plan delivered | Qwen 3.8 Max |
| [students](05-students/README.en.md) | An assistant for a full-stack analyst mentor: student plans, meeting breakdowns, CV assembly, SA reviews | 46 commits, 06–08.2026 | working | Opus 5 |
| [scalping](06-scalping/README.en.md) | Claude mentoring me in manual scalping: a pattern-recognition trainer, an automatic trade journal, a morning screener | 142 commits, 07–08.2026 | the core is learning, tools are secondary; ~40% of the plan delivered | Opus 5 |
| [health](07-health/README.en.md) | Personal health monitoring: doctors' reports, Apple Health and a fitness band in a single picture | 57 commits, 07–08.2026 | functionality works, ~30% of the plan delivered | Kimi k3 and Qwen 3.8 Max |

Plus two early-stage projects not described here: a lead generator and CRM for a construction
company's dealer, and a plant-care mobile app in Flutter.

## Caveats

- **Code and data are not published.** This is descriptions and agent working rules only. Need
  details — write to me on Telegram: [@Basov_VA](https://t.me/Basov_VA).
- Project `CLAUDE.md` files are given **as they are, in Russian** — they're working files.
  Sections about infrastructure and certain secrets have been cut, and the cuts are marked.
- Completion percentages are against my own development plan. The functionality may work
  entirely while only a third of the plan is done.
- **Licensed under [MIT](LICENSE).** Take whatever you need: copy it, change it, use it in your
  own projects, commercial ones included. A link back is appreciated, but I don't require one.
  And to be clear: the practices themselves belong to no one — the license only covers the text
  of these files.
