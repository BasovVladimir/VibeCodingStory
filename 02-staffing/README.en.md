# staffing — a recruitment platform with IT market analytics

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**449 commits · April–August 2026 · architect Claude Opus 5**
**Live production.** An MVP with regular client demos (it's a SaaS product), released once a day
on a schedule.

## What makes this project different from the others

Most of the systems in this showcase are mine: I'm the customer, the only user, and the one who
pays for mistakes. Here there's **a real client and external users**, and production is a proper
circuit — HTTPS, a release procedure, migrations in order, and a hotfix lane.

Everything else follows from that: a daily release at a fixed time instead of "committed =
released", a `main` branch that by definition equals production, a branch per task, and
acceptance on a separate environment.

This is also the only project where **I do the decomposition of the business tasks myself**
rather than the architect: here I take on the product analyst role as well, because what I'm
slicing is product value, not technical layers. The architect still handles the technical
slicing of a business task.

It was also my first vibecoding project — it started as a chat with DeepSeek, with me running
the commands from that chat in a terminal.

## How I work with the agent

The rhythm here is set by the release train. Once a day, at a fixed time, a release goes out,
and everything else fits around that schedule.

It all starts with me: I slice a business task into pieces of product value — exactly what the
client gets and in what order. Then the architect slices my brief technically and decides who
gets what. It hands over a batch rather than a whole task: within one work order the first batch
may go to the coder while the second comes back to the architect, once it becomes clear how much
judgement is involved. The coder sits in a separate working copy on shifted ports, never merges
into `uat` or `main`, and doesn't touch the servers at all, not even to read.

Delivery goes only through the gate. I don't look at a batch without a green gate, and a red one
comes back without discussion. After every task I ask for a report: what changed, what got
fixed. **No merges happen without me**, and the merge order is mine too — otherwise the day's
train falls apart.

Then I go to the acceptance environment and click through it by hand. That's exactly how we
found the batch that passed the bar ten out of ten with completely non-working toggles: the test
clicked where it found convenient, while a real user couldn't reach the element at all.

**The agent writes comments for its own next session, and that's a mandatory part of the work.**
In non-obvious places it leaves a comment prefixed `AI:` — what the logic here is and which
rakes have already been stepped on. It's found by grep, so instead of documentation that has to
be maintained separately and goes stale anyway, we get an index of dangerous places right inside
the code. On a project with a release train that pays off fastest: when a hotfix is being made,
the warning is visible right where the edit happens.

## The goal

Outstaff recruitment works like this: client vacancies pour into Telegram channels, while the
recruitment agency has its own database of candidates and CVs. Between those two piles sits a
human matching them by hand: fits, doesn't fit, what to ask at the interview, who to send it to.

The platform takes that over entirely: **a vacancy from a channel becomes a structured record,
candidates are automatically scored for fit, and out comes a checklist of interview questions
that lands in the company's working group in Telegram.**

The whole architecture grew out of one constraint: **this is SaaS, and competing companies live
side by side in it**. Vacancies on the market are shared — everyone sees them. But candidates,
scores, checklists and processing statuses are strictly isolated per company. That isolation
runs not only through API permissions but through delivery too: each company gets its results in
its own Telegram group, one the platform bot was added to by that company itself.

The second goal grew out of the data: **IT market analytics**. A stream of vacancies flows
through the system, and that stream is valuable in itself — what's actually being asked for,
which skills are must-haves, how demand shifts wave by wave.

## What it does and how it works

```
Telegram channels ──▶ listener ──▶ vacancy parsing (LLM) ──▶ vacancies
                                        │
                                        ├──▶ classification: position + specialisation
                                        ├──▶ requirement extraction
                                        └──▶ semantic summary: tasks / experience / domain
                                                     │
candidates (Excel + CVs) ──▶ fit scoring ────────────┘
                                        │
                                        ▼
                          checklist of interview questions
                                        │
                                        ▼
                     the company's Telegram group (strict FIFO)
```

**Heavy model calls were moved into their own queue — that's the main resilience decision.**
There are three queues: a general one (parsing messages and CVs), a "heavy" one for LLM work
(scoring, checklists, requirement extraction), and Telegram delivery with a concurrency of
exactly 1. The story behind it is specific: a burst of scoring across a large candidate base
used to freeze the parsing of incoming vacancies, and messages from different deliveries got
mixed together. Now the first doesn't block the second, and the third guarantees ordering.

**A Telegram group is linked with a one-time code.** A company admin generates a code in the
admin panel, adds the bot to their group and writes a command with that code there; the bot
saves the chat ID through an internal API. The code lives for about fifteen minutes. Isolation
between companies rests on the fact that the bot physically writes only where it was invited —
no configuration required for that.

**Resilience against the external model.** LLM calls have retries with exponential backoff on
rate limits and timeouts, and any computation stuck longer than a set TTL gets picked up by a
watchdog once a minute. Otherwise "still calculating, forever" looks like a breakage to the user.

### Information architecture

| Screen | The question it answers |
|---|---|
| **Vacancies** | what came in from the market, what's already processed, what doesn't fit; filters by position and status |
| **Candidates** | who's in the company's database, their grades, stack, rate, CVs |
| **Scoring and checklists** | how well a candidate fits and what to ask them at the interview |
| **Admin panel** | source channels, incoming messages, Telegram group linking, error log |
| **Dashboard** (in progress) | who to hire · what's burning · where we're losing people · what's happening on the market |

The API is grouped by area: authentication with JWT and invitations, candidates, vacancies,
checklists, the admin panel, and a separate internal circuit for service calls and analytics.
The contract is held in an OpenAPI spec that's regenerated by a command after changes.

## Architecture

**Backend** — FastAPI + SQLAlchemy, PostgreSQL, Celery on Redis, all in Docker. **Frontend** —
React + TypeScript + Vite on Ant Design, state in context and hooks with no Redux. **The
parser** is a separate listener service, **the delivery bot** is another one; both were pulled
out of the monolith because they have their own lifecycle and their own network constraints.

**Multi-tenancy is a cross-cutting invariant, not a feature.** The per-company status of a
vacancy lives in its own table and is created **lazily**, on the first edit: no row means status
"new", with no comment. That saves us from warming up an "all companies × all vacancies" matrix
and makes adding a company free.

**Market analytics is built on the principle "code computes the numbers, the architect writes
the text".** A nightly snapshot lays vacancies and skills out into cells of "date × position ×
specialisation × grade" — that's the single source of truth for the numbers, deterministic and
reproducible. The analysis itself is done with Claude.

On top of that we assemble a **portrait of a cell over a two-week wave, and it consists of two
inseparable layers**:

| Layer | What it is | Who does it |
|---|---|---|
| **Numeric slice** | skill shares (must-have / nice-to-have), grade mix, "N+ years of experience", deltas against the previous wave | code, deterministically |
| **Ideal candidate CV** | the text of a candidate who would cover ~80% of the vacancies in that cell | the architect, from the summaries |

Recomputing the numbers doesn't overwrite the text, and that matters: otherwise every night
would erase the human work. Terms the dictionary doesn't recognise are never lost and never
guessed — they land in an exceptions queue for manual review, and the skills dictionary is
extended by a human decision.

Analytics is exposed outwards through **an internal API secured by a shared secret** — the
neighbouring project about training analysts uses it: student development plans are built from
real market demand.

## Project structure

```
backend/app/
  models.py · schemas.py            DB models and Pydantic schemas
  tasks*.py                         Celery: scoring, checklists, Telegram parsing, delivery
  celery_app.py                     routing tasks across the three queues
  services/                         scoring · CV parser · Telegram delivery · email
  analytics/                        snapshots · portraits · skills dictionary · summaries · topics
  routers/                          the API, including the internal analytics circuit
  specializations.py                specialisation classifier (a second level below position)
backend/migrations/                 Alembic — the schema only moves through migrations

frontend/src/                       pages, components, hooks, auth context
ui-kit/                             the design system (only edited deliberately)
telegram_listener/                  the channel listener
telegram_bot/                       the bot that delivers checklists and links groups
config/nginx/                       production configuration

docs/practices/                     roles and workflow · code and tests · the coder gate · agent lessons
docs/наряды/                        work orders for coders (+ Выполнено/, runs/)
docs/Техдолг.md                     workarounds, functional debt, deferred decisions
docs/Грабли по подсистемам.md       the non-obvious things that already cost us a round of work
Аналитика/                          portrait methodology, frequency analysis, timing summary
RELEASE.md · TESTS.md               release runbook · test coverage catalogue
```

Three things about this layout aren't obvious:

- **`Аналитика/` is a methodology.** It describes trigger procedures: which phrase kicks off a
  wave of portraits, what updates itself, what's done by hand and how often. The document exists
  because half of the analytics is inherently manual.
- **`docs/Техдолг.md` is maintained as a first-class document.** Workarounds are written down the
  moment they're put in, together with the condition for removing them. An unrecorded workaround
  becomes architecture.
- **`docs/наряды/` is kept separate from task codes.** A work order is a brief with executable
  acceptance criteria; it gets written even for tasks the architect does itself.

## Roles

- **PO (me)** — product, priorities, acceptance, merge order. **Decomposition is on me too** —
  the only project with that exception.
- **Architects** — Claude Opus 5, Kimi K3, Qwen 3.8Max, equal, no personal territories:
  architecture, migrations, analytics, the gate and the work order, review, merge and release.
- **Coders** — GLM and DeepSeek in an autonomous CLI loop, in a separate working copy with their
  own environment on shifted ports. They never merge into `uat` or `main` and **never touch the
  servers, not even to read** — the architect puts any needed extracts into the work order.

Coders only get work that's mechanical and isolated, where at least ⅔ of the tasks in a batch
carry a machine check. Architecture, migrations, the access model, analytics and scoring,
taste-driven UI and hard debugging always stay with the architect. And that call is made **per
batch**, not per whole task: within the same work order the first batch may go to the coder while
the second comes back to the architect once it's clear how much judgement is involved.

## Release procedure

This is the only project where a release is a full procedure:

- **`main` equals production by definition** and only moves at release time, once a day
- **`uat` is "the day's train"**: accepted work flows in there, and acceptance happens on a
  separate environment
- **a branch per task**, branched off the fresh head rather than a stale local `main`
- **hotfixes get their own lane** off production, with a forward merge into both branches
- **migrations are applied deliberately, before the application comes up**, and are chained off
  the head of the train; after a release, branches still in progress re-chain their migration
  onto the new head, otherwise you end up with "two heads"

I dropped hourly releases deliberately once there were real clients.

## Principles

1. **The gate is the only delivery interface.** A batch without a green gate isn't reviewed; a
   red gate is returned without discussion. Anything found by hand becomes a new gate check in
   the same pass.
2. **Green tests from the executor prove nothing** if the fixture was written by the same head
   that wrote the code: it simply repeats the same assumptions.
3. **Don't pass off unverified as verified.** Release status is the output of a command, not
   memory; market numbers are a query against the database, not "that's how it was last session".
4. **Interface text is written for the user.** Table names, query limitations and explanations of
   our internals have no place on screen.
5. **Tests run locally**; the environment is only used to check what depends on the environment:
   deployment, migrations, behaviour against the live model.
6. **We save by structuring the work, not by cutting checks.** Review, data reconciliation and
   self-checking are never shortened.
7. **A workaround is written into tech debt the moment it goes in**, together with the condition
   for removing it.

## Key mistakes

**1. Green gate, broken feature.**
A batch with toggle switches passed the bar 10 out of 10: the tests clicked the input element
and got the correct state back. In reality that's a hidden zero-size input underneath a visible
slider, and a real click never reaches it — the linking markup was missing. Completely
non-working toggles passed every check, and I caught them by hand on the acceptance environment.
**Fix:** in a UI check, click **what the user sees**, not what's convenient for the test. That
became a rule for the bar, not a one-off fix.

**2. A gate can't see what it doesn't know how to see.**
Same batch: a race between async saves and silent errors on write requests — the gate was
checking the contract and never looked at either.
**Fix:** checks like that go into the bar **before the batch is handed out**, not after the first
incident. The conclusion is worth more than the case itself: **a gate checks exactly what you put
into it**, so you write it against the risks, not against the happy path.

**3. A symptom mistaken for a cause.**
The error "couldn't compute scoring" looked like a regression from a recent task. In fact it was
the normal behaviour of an aggregate status from an entirely different task: some candidates had
no CV file, so the overall status honestly went to unsuccessful.
**Fix:** before naming a cause — **reproduce it**. A false attribution costs more than having no
answer: it sends you off to rewrite code that works.

**4. A caption under an empty widget damaged the product's reputation.**
Under a metric we had nothing to compute from sat an honest internal caption along the lines of
"we need the vacancy close date". To us that's an explanation; to the client it reads as "they
can't calculate the obvious". **Fix:** if a metric isn't available — a muted placeholder, and
that's it. Internal captions, data sources and query limitations live in the code and the work
order, not on screen. Separately: a caption under a number is obliged to **say something**. If
it's a string of words the reader can't act on, better to write nothing.

**5. Rules were broken because copies had gone stale.**
The limit on the number of scouts was tightened, and then broken again: the agent was reading not
the primary source but a copy of the rule in another document, where the older, looser wording
remained.
**Fix:** when a rule is tightened, fix all of its copies at once; when copies disagree, the
strictest one is the truth. And the conclusion that became common to all my projects: **a
duplicated rule is a future violation**, so rules are referenced by link, not copied.

**6. A branch off a stale head.**
Twice, starting a task tripped over the branch being taken from a local `main` that was a dozen
commits behind, while accepted features waited for the day's release train. Beyond the extra
merge, that gives you **the wrong head for the migration** — and "two heads" in the schema.
**Fix:** always branch off the fresh remote head of the train, and check which branch you're on
**before** the first edit.

## What's built and what's next

Near term (already in progress or scoped out):

- **Analytics dashboard** — four questions instead of tables: who to hire (demand against our
  coverage), what's burning (ageing vacancies), where we're losing people (the recruitment
  funnel), what's happening on the market (the pulse). Part of the API and frontend went to the
  coder; the funnel and acceptance stay with the architect
- **Multiple user roles** — right now a user has exactly one role, so granting access to
  analytics means taking away their main one. We need N roles with additive permissions; it'll be
  done entirely by the architect, since the access model is off limits to coders
- **A second LLM provider as a fallback** — the research is done, both providers are
  API-compatible; implementation hasn't started
- **Expanding in the mirror direction** — matching positions for individuals, beyond outstaffing

## Scale

- **449 commits**, April–August 2026 — the longest-running project in the showcase
- **786 unit and component tests** (backend and frontend) plus **12 end-to-end** against the real
  application in containers; external services are always mocked, tests never reach the network
- **Three environments**: local, acceptance, and live production with HTTPS and a daily release
- **Three background task queues** with different load profiles and separate workers
- The analytics subsystem: nightly snapshots, portraits over two-week waves, a skills dictionary,
  semantic summaries of vacancies, an internal API for the neighbouring project
- Stack: Python 3.11 · FastAPI · SQLAlchemy · Celery + Redis · PostgreSQL 16 · React 19 +
  TypeScript + Vite + Ant Design · Docker · Nginx · pytest · Vitest · Playwright
