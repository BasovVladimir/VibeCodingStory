# paperbirds — a co-pilot for an indie band

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**207 commits · July–August 2026 · architect Qwen 3.8 Max**
The functionality works; ~30% of the plan is delivered.

## How I work with the agent

I have a few hours a week for this project, and the entire rhythm grew out of that.

The regular part takes five to ten minutes. The collector has pulled in a day's worth of posts,
the router has filtered out the noise, and I run through the "Inbox" in the browser: that's a
venue, that's an open call, that one's a miss. In the evening, in a chat with the architect, I
assign the "fits us" score — I assign it, the model doesn't compute it. The agent puts draft
emails into `outbox/`, and I'm the one who sends them.

Development happens separately and less often. I look at the plan to see what's left and give
the command. The architect then writes the work order, assembles the bar for it, and hands it to
the coder, which spins in an autonomous "coder ↔ gate" loop. I don't look at a batch without a
green gate at all; a red one comes back without discussion.

After a task I ask for a report: what changed, what got fixed. **No merges happen without me.**
Test runs happen only in Docker and only in full: locally the image can turn out to be stale, and
then green means nothing.

One thing I hand to nobody — anything domain-related. The scoring scale, the rules from the
band's profile, the router prompts, the wording of pitches. That needs taste and knowledge of
your own scene, and there's no machine check for it.

## The goal

I play in a local indie band called paperbirds and manage it as well. Every independent band has
the same problem: venues, open calls and festivals are scattered across dozens of Telegram
channels, deadlines burn out unnoticed, and the contacts of art directors live in DMs and get
lost.

The system finds venues and open calls, filters out the obvious misses, keeps deadlines from
slipping, drafts pitches and maintains a contacts database.

**The main asset here isn't the code, it's the database.** The code could be rewritten in an
evening; the collected art director contacts, the venue terms and the history of correspondence
could not. The whole architecture is built around that, which is why data is never invented: no
source means the field stays empty, and that's visible.

## What it does and how it works

A closed path from a Telegram post to a row in the database that a human makes a decision on:

```
TG channel ──fetch──► inbox ──parse──► "Inbox" ──human──► venues / open calls
Telethon   scheduled  raw     DeepSeek  in browser  confirms      the main asset
                              filters noise         and edits
                                                          │
                        filter ──► rule-based cut-offs (city / deadline / genre)
                                                          │
                        candidates ──► evening ritual in chat ──► fit-batch
                                       a HUMAN assigns the score   score + reason
```

Four decisions that explain how it's built:

- **Collection on a schedule.** A channel listener doesn't survive restarts and deployments, and
  it can't see a channel's history at all. So the collector just runs a simple loop: "fetch →
  parse → sleep for an hour". Boring and reliable.
- **The model only filters and extracts fields.** The decision to enter a record is made by a
  human in the browser. The model sorts here; it doesn't decide.
- **A human assigns the "fits us" score.** An evening ritual of 5–10 minutes in a chat with the
  architect. The honest reason: the project has no separate paid API subscription, and pushing
  this through the main one costs more than just assigning the score myself.
- **Fragile integrations live in quarantine** — in a separate `sources/fragile/` folder. One
  external integration falling over doesn't take down the rest of the pipeline.

### Information architecture

React + Vite + TypeScript + Ant Design. Nine screens, grouped by funnel stage:

| Screen | What it's for |
|---|---|
| **Inbox** | raw material after the router: confirm, correct, send to the database or reject |
| **Venues** | the main asset: terms, contacts, the history of the relationship |
| **Open calls** | applications with deadlines and requirements |
| **Deadlines** | what's burning right now |
| **Gigs** | confirmed performances |
| **Todo** | what to do by hand: finish a pitch, send an email |
| **Sources** | managing TG channels: add, check, disable |
| **Console** | the analytics dashboard: funnel, conversion, database health |
| **Admin** | utility operations |

**Nothing** is published outwards: the database holds real personal contacts, and access to the
interface is over an SSH tunnel only (at this stage of development).

## Architecture

**The database is the centre of the system; everything else serves it.** SQLite, with the schema
created idempotently and no migrations. The layers are strictly separated:

| Layer | Responsibility |
|---|---|
| `models.py` | pydantic models for the entities and the reference data |
| `db.py` | schema and idempotent initialisation |
| `store.py` | data access: lists, filters, upsert — **never overwrites the score or the blockers** |
| `api.py` | FastAPI on top of store — the single source for the frontend |
| `cli.py` | everything run by hand or on a schedule |
| `scoring.py` | ⛔ **pure cut-off rules: no network, no database, no models** |
| `sources/fragile/` | quarantine for the fragile parts: post parsing, the "noise / not noise" router |

Two places here deserve special attention.

**`scoring.py` is deliberately sterile.** The cut-off rules for city, deadline and genre reach
nowhere: not the network, not the database, not a model. That's why they test instantly and don't
break because of someone else's failure — and they're the most frequently edited part of the
system.

**`store.upsert` doesn't overwrite what a human entered.** Re-collecting the same post will
update the description and the terms, but won't touch the assigned score or the notes. Otherwise
every collector run would erase the manual work — precisely the work all of this exists for.

## Project structure

```
src/paperbirds/
  models.py            pydantic models: venue, open call, reference data
  db.py                SQLite schema + idempotent initialisation (no migrations)
  store.py             data access: lists, filters, upsert
  api.py               FastAPI — JSON on top of store, the single source for the frontend
  cli.py               typer: everything run by hand or on a schedule
  scoring.py           ⛔ pure cut-off rules — no network, no database, no models
  sources/http.py      a polite HTTP client: ≥2s delay, cache, honest User-Agent
  sources/telegram.py  Telethon user client, scheduled post collection
  sources/fragile/     quarantine: post parsing, the "noise / not noise" router

frontend/src/
  pages/               the nine funnel screens
  components/          cards and forms for venues, open calls, series
  api/                 axios clients, one per entity

tests/                 pytest, external networks mocked · tests/e2e — end-to-end CLI flow
frontend/e2e/          Playwright, the browser comes up in a separate compose file
scripts/gate.py        ⛔ the delivery gate — the only criterion for a batch being ready
scripts/coder_run.py   the autonomous "coder ↔ gate" loop
scripts/coder_epic.py  running a work order from a manifest: one task = one branch
docs/                  plan, band profile, practices, work orders
data/                  the database (not in git) · cache/ — the HTTP client's cache
outbox/                draft emails: the agent puts them there, a human sends them
```

**Everything lives in Docker only** — nothing gets installed on the MacBook.

## Roles

- **PO (me)** — product, the band's profile, every decision about sending something. I assign the
  "fits us" score.
- **Architects** — Qwen 3.8 Max leads on this project, plus Claude Opus and Kimi K3. Equal, one
  process.
- **Coder** — DeepSeek v4 Pro in the autonomous "coder ↔ gate" loop, never merges into `main`.

One rule is specific to this project: **anything domain-related is always mine.** The scoring
scale, the rules from the band's profile, the router prompts, the wording of pitches — none of
that goes to the coder or to automation. A gate can't check it, and explaining it takes longer
and works worse than doing it myself.

## Principles

1. **Don't invent data.** Don't know? Empty plus a "needs checking" note. An invented contact is
   worse than a missing one. **An empty string and whitespace mean "no data", not a value.**
2. **Every record carries a source and a verification date.** Without a source the record is
   useless.
3. **No automatic sending.** The draft goes into `outbox/` and a human sends it. And no
   artificially inflating stream counts.
4. **We don't delete what we filter out**, we mark it with a reason. Otherwise the router quietly
   eats a real open call and we never find out.
5. **Polite scraping:** at least a two-second delay, an honest User-Agent, caching, robots.txt.
6. **Secrets only in `.env`.** The repository is private, and the database holds real people's
   personal contacts.

## Key mistakes

**1. `git add -A` destroyed someone else's work.**
The working copy held uncommitted changes from the coder. `git add -A` swept them into the
agent's commit — and `scoring.py` and `cli.py` were lost for good.
**Fix:** `git status` and a check of the current branch are mandatory before any edit.
Uncommitted changes the current agent didn't make = someone else's work, and `git add -A` is
**forbidden** while they're there. The rule went into agent memory verbatim, because a document
isn't enough here: the mistake is made in the first seconds of a session, before anyone opens the
docs.

**2. A task went "green" without being done.**
If the gate has no section for a specific task, the batch goes green on the general checks —
build, types, tests — while the task itself produced nothing.
**Fix:** **the bar is written BEFORE the work order goes out**, for that specific batch, tagged
with the task. A section for a task that isn't done yet is red in advance, and that's fine: it's
an executable spec, not a breakage.

**3. Tests were running old code.**
`src/` and `tests/` are baked into the image. Without rebuilding the container, a run tests the
previous version — and cheerfully goes green.
**Fix:** rebuilding the image became a mandatory step before a run, and the fact itself went into
the rules: **a run outside Docker doesn't count as delivery.**

**4. A real-time channel listener didn't survive.**
The persistent connection dropped on every deployment and restart, and it can't see a channel's
history at all — so anything missed during downtime was lost forever.
**Fix:** dropping real time in favour of a scheduled loop. Less "clever", but it survives
restarts and picks up what it missed. That's a direct consequence of the "a few hours a week"
constraint: a system that needs babysitting is no use to me.

**5. The router was quietly eating things we needed.**
Rejected records were deleted — so when the router got it wrong, there was no way to find out.
**Fix:** rejection became a mark rather than a deletion. The record stays in the database with
the reason it was filtered out. Now a filter mistake is visible.

**6. The coder was handing in unverified work.**
This particular executor's weak spot is verification discipline: "done" arrived before the
checking did.
**Fix:** the gate as **the only delivery interface**. A batch without green output isn't looked at
at all; a red gate is returned without discussion. It isn't about distrusting the model: checking
simply has to be done by a machine, because a human who doesn't read code can't do it.

## What's built and what's next

Closed epics: the foundation and the band profile, the skeleton, the venues and open calls
database, scoring, source monitoring, data maturity (series and seasons instead of duplicates),
and **the Console** — the analytics dashboard.

In progress and ahead:

- **The funnel: applications, emails, deadlines** — the current epic. Pitch drafts, status
  tracking, a gigs page
- **Authentication and HTTPS** for the interface — a deferred tail from the monitoring epic
- **Reconnaissance** — analysing bands at our level: where they play, we can play; who to put a
  compilation together with
- **A release engine** — planning releases and pitching to playlists
- **Content** — help with running the band's social media
- **A strategist and a Telegram bot** — the whole system in chat, so I don't have to open a
  browser

**The completion criterion for phase one isn't stated in code:** it's **10 live applications
sent**. Until those exist the phase isn't closed, however many epics have gone green.

## Scale

- **207 commits**, July–August 2026
- **328 pytest · 78 Vitest · 6 E2E Playwright**
- The delivery gate is an executable bar and the only criterion for a batch being ready
- Stack: Python + FastAPI + SQLite · React 19 + Vite + TypeScript + Ant Design ·
  Telethon · DeepSeek API · Docker · pytest · Vitest · Playwright
- Production: three services on a shared server, nothing published outwards
