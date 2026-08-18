# reports — a personal investment system

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**601 commits · June–August 2026 · architect Claude Opus 5**
Working; the development plan continues.

## How I work with the agent

There are two modes here, and they barely overlap.

The first is analytical and runs on a schedule. In the evening I sit down with the agent over
the news that's piled up: it sorts it by issuer, I argue with its interpretations, and together
we set up watches. Once a week a single command refreshes prices, the portfolio and macro data.
Once a quarter there's a big session: retro, issuer review, rebalance. The agent acts as an
investment adviser here, but the decision about money always stays mine.

The second is development. Epics live in the plan, tasks live in work orders, each with its own
code (`# B6`). I look at what's left and give the command. From there the architect decides:
does it itself or writes a work order for the coder. The coder sits in a separate branch and
never merges into `main` — I accept the work, and only on a green gate. A red gate comes back
without discussion: the gate output *is* the list of fixes.

After every closed task I ask for a report — what changed, what got fixed. **No merges happen
without me.** Before a merge, a full test run, not just the tests near the change.

I go in selectively: where the logic is tricky or where data could get quietly corrupted. Into
the logic, not the code — by what rule is this number computed now, where does it come from,
what happens if the source goes silent. Tests catch broken code well, but they never ask "does
this number even make sense?"

**The rules for comments live in a separate practice document** — [код-и-тесты.md](код-и-тесты.md)
("code and tests"); my other projects take it as it is. The point: comments are written FOR THE
NEXT SESSION, because context between sessions is lost and the comment is the only thing left to
stand on. A module docstring "why/problem", inline comments on non-obvious decisions, an `AI:`
prefix on new rakes — `grep -rn "AI:" scripts/` shows every dangerous spot at once, with no
infrastructure. A plan task tag in the comment (`# A1`, `# B6`) ties a line of code to the
reason it appeared at all.

## The goal

I'm a long-term investor in the Russian market. A decision about an issuer is made from
financial statements, news, price and the current portfolio, not from a feel for the market —
and months pass between such decisions, long enough to forget everything.

The system solves exactly that: **it accumulates primary sources and, on a set cadence, turns
them into decisions**. Financial statements, news, prices, the portfolio and macro data go into
a database, and from there they're assembled on schedule into an issuer card, a quarterly review
and a rebalance.

The whole architecture grew out of one constraint: **an investment decision is made from a
table, without opening the primary source**. Which means the cost of an error in the data is
real money, and an "approximately right" value is worse than a missing one. An empty field is
visible immediately, while a wrong number looks exactly like a right one.

## What it does and how it works

The system lives by **rhythms** rather than continuous work:

- **Daily** — incoming news drops into the database through a Telegram bot, plus three automatic
  feeds on timers (posts from an investment social network, the exchange disclosure feed,
  deferred reading). In the evening comes the review ritual: the agent reads what's accumulated,
  sorts it by issuer, and creates "watches" — reminders to look at an event on a particular day.
  **P.S.** — this is done manually to save Claude API tokens. Within the Claude Code
  subscription I ping it in chat in the evening, it goes through the news and we do the evening
  review together. It's also more convenient than plain real-time news analysis.
- **Weekly** — one command updates prices, the portfolio and macro data.
- **On event** — a financial statement is published: the PDF goes through the pipeline and
  becomes rows in the database.
- **Quarterly** — retro, issuer review, rebalance.

**The statements pipeline** is the central piece:

```
PDF → text extraction (+OCR) → recognition by the agent → RAW JSON
    → validation → move the PDF to "Recognised" → link probe
    → build the canonical DB → compute multiples
```

Recognition is done by **an agent**, not a parser: statements are laid out however each company
feels like, and regexes lose that fight. But nobody takes the agent's word for it — the result
goes through a **pyramid of checks**:

0. strict JSON against a schema
1. **accounting identities** — Assets = Liabilities + Equity; Net profit = Pre-tax profit + Tax;
   `9 months − 6 months = Q3`
2. completeness against the industry checklist
3. common sense and double extraction of the key figures

The most valuable step is the identities. They catch what neither the schema nor the eye will:
if the agent mixed up columns or lost a sign, the balance sheet won't balance.

### Information architecture (the dashboard)

A web dashboard on FastAPI + React, brought up in Docker with one command. Four sections —
matching exactly the four questions I ask my money:

| Section | The question it answers |
|---|---|
| **Capital** | How much money there is in total and how it changed over time |
| **Allocation** | How it's spread across categories and where it deviates from target |
| **Portfolio** | What's actually held and what the return is per position |
| **Issuers** | What's happening with the companies and which watches are set |

The API serves data through flat endpoints, one per section; the frontend knows nothing about
SQLite and computes no business logic — all the calculations stay in the scripts and the API.

There's a separate showcase: company cards in Anytype, redrawn from the database by a script.
The truth is always in the database and git; Anytype only displays data on issuers and assets,
including significant corporate events, their news and so on.

## Architecture

**Two data layers, and this is the project's main architectural decision:**

| Layer | Where | Purpose |
|---|---|---|
| **RAW** (as-reported) | JSON files, **in git** | Completeness and truth. Facts exactly as they appear in the report, one report = one file. Immutable |
| **CANONICAL** | SQLite, **not in git** | Comparison and multiples. Metrics mapped onto a single dictionary |

CANONICAL is always rebuilt from RAW with one command. That's why deleting it isn't scary, and
why the project has no database migrations. Anything a rebuild can't restore — prices,
portfolio, news, verdicts, targets — lives in separate persistent tables that survive the
rebuild.

This split gives the property the whole thing was built for: **every number in the system traces
back to a page in an issuer's PDF**. There's a dedicated probe that regularly checks the links
to primary sources haven't gone stale.

**Unifying the metrics.** A company is tagged with an industry, and the industry defines the
expected set of metrics and multiples: a shared core + a "financial / non-financial" split +
sector packs (retail, oil & gas and so on). The same key holds IFRS and RAS side by side, with
IFRS taking priority and RAS as the fallback.

**Sources** are primary only: the issuer's PDF, the exchange's ISS, the Central Bank, a broker
API, Telegram with vetted channels, an investment social network. On top of that, the project
consumes data from the shared node of the neighbouring myTrade project — quotes and bonds aren't
collected twice.

## Project structure

```
data/raw/{TICKER}/{period}_{standard}.json  a validated report (the RAW layer, in git)
data/companies.yaml                 the company registry — master config, edited by hand
data/crypto.yaml · commodities.yaml crypto and commodity registries
data/portfolios.yaml                portfolio registry; target_allocation.yaml — targets
data/pulse_sources.yaml             allow-list of news sources
data/macro.yaml                     macro snapshot (only fetch_macro.py writes to it)
data/descriptions.yaml              descriptions for the showcase cards
data/anytype_map.yaml               ticker → showcase card id
data/archive/                       text extracted from PDFs (traceability)

db/reports.sqlite                   the CANONICAL layer + persistent tables (NOT in git)
schema/report.schema.json           the report's JSON Schema
dictionary/concepts.yaml            canonical metrics (core + sector packs)
dictionary/aliases.csv              raw line name → concept_id
dictionary/rsbu_lines.csv           RAS line codes

scripts/                            the entire pipeline code (62 scripts)
api/                                the dashboard's FastAPI backend
frontend/                           React + Vite frontend (+ e2e Playwright)
tests/                              pytest — green after every change
Промпты/                            the analytical layer: criteria, reviews, rebalancing
docs/                               plan, risk profile, subsystems, practices, work orders
deploy/                             deployment of the news-circuit units

Отчётности/{Ticker}/Новое | Распознано | Важное | Обзоры/    source PDFs (not in git)
```

Three things about this layout aren't obvious and were done deliberately:

- **`data/raw/` is in git while `db/` isn't.** RAW is the truth, so we version it. The database
  is rebuilt from it with one command, so there's no point storing it and nothing to migrate.
- **`dictionary/` is kept separate from the code.** The mapping "report line → canonical metric"
  changes with every new issuer. If it lived in code, changing it would need a developer; in CSV
  it's done by an analyst, which is to say me.
- **`Промпты/` (Prompts) is as much an artefact as the code.** It's the analytical layer: review
  methods, criteria thresholds, decision formats. It's in git, it gets reviewed, it's versioned.

**Script groups** (the source of truth for each is its docstring):

| Group | What it does |
|---|---|
| Statements pipeline | PDF extraction · validation · DB rebuild · multiples · inventory |
| Cards and decisions | company card from the DB · verdict history · quarterly review entry point · showcase sync |
| External data | macro · prices · FX rates · instrument names · portfolio · sync with the neighbouring project's node |
| Portfolios and capital | snapshots · TWR return · allocation of top-ups · rebalancing |
| Bonds | a slice across holdings (YTM, duration, coupons) · market screener · digital financial assets |
| News | receiving bot · article reading · timer-driven feeds · deferred reading |
| Watches | writing reminders into the DB · feeding the notifier bot |
| Video | downloading · transcription · review into the showcase |
| Ritual wrappers | the evening review · "refresh the data" in one command |
| Utilities | the dashboard delivery gate · building a test database |

**Environment:** Python 3.13 through `uv`, with the virtualenv inside the project. The dashboard
comes up with a single `docker compose`. Tests are `pytest -q`, never reach the network, and
fixtures are built on the production schema. Secrets live in `.env` and never reach git.

## Roles

The standard layout for my projects; details are in [CLAUDE.md](CLAUDE.md):

- **PO (me)** — strategy, risk profile, acceptance, every capital decision. I don't read code.
- **Architects** — Claude Opus 5, Kimi K3, Qwen 3.8Max, equal and interchangeable. Here they
  have a second role beyond engineering: **recognition and analysis are the architect's work**,
  while storage and calculation are the scripts' work. **Analysis** means making decisions about
  issuers and assets as objectively as possible, taking my risk profile and investment goals
  into account. In that seat Claude takes the role of an investment adviser, but the final
  decision is mine.
- **Coder** — DeepSeek v4 Pro in an autonomous loop, working in a separate branch, never merging
  into `main`.

The split "analysis is the agent's, computation is the code's" is fundamental here. Anything
that can be computed deterministically is computed by a script: arithmetic isn't the agent's
job, understanding is.

## Principles

1. **Primary sources only.** Numbers are never typed in by hand — not from a web search, not
   "from memory". **No source → the field stays empty, and that's visible.**
2. **Every number traces** back to a file and a page; a probe checks the link is still alive.
3. **RAW is immutable, CANONICAL is rebuilt.** Anything volatile lives only in persistent tables.
4. **Don't pass off unread as read.** If the basis is a headline, say so.
5. **Everything gets checked:** the schema, accounting identities, negative tests.
6. No change is handed in without a green run; tests never reach the network.

## Key mistakes

This is where it hurt.

**1. A link to the primary source went stale silently.**
After validation the PDF moved from the "New" folder to "Recognised", while the path inside the
RAW file stayed as it was. The data looked perfect: numbers in place, schema passing. What broke
was traceability — the one thing the whole project exists for — and the only way to notice was to
open the specific file by hand.
**Fix:** moving the PDF and updating the link became a single operation, and on top of that came
**a dedicated probe** that runs through every link and fails if even one is broken. It currently
holds 737 links. The general lesson: if an invariant can't be checked by a command, it isn't
being upheld — it just hasn't been broken yet.

**2. The agent rejected a news item without reading it.**
It dismissed a "worst stocks" round-up article with the reason "none of our holdings are named
there". All it had read was the teaser. Inside was a held stock with a share issue and weak
dividends — exactly the kind of negative the review exists for.
**Fix:** the rule "a reason for rejection has to be a checkable fact". "Duplicate",
"advertising", "no issuer attached" — those can be checked against the text you already have.
"Our holdings aren't in there" is a claim **about the content**: the agent either opens it or
honestly writes "haven't read it". Round-ups and rankings are always read through: the teaser has
no tickers, but the body almost certainly does.

**3. A rule kept being broken because it lived in three places.**
The limit on the number of subagents was tightened three times and broken again every time — the
agent was working from a stale copy of the rule in another document, without opening the primary
source.
**Fix:** two consequences grew out of this and became common to all my projects. When tightening
a rule, fix all of its copies at once or make one single source of truth; when copies disagree,
the strictest one is the truth. And the main conclusion: **a duplicated rule is a future
violation**, which is why rules are no longer copied across documents but referenced by link.

**4. Documents written across multiple sittings tore at the seams.**
A work order was written over three sittings. Each chunk was internally coherent, and all five
inconsistencies were found exactly between the sittings. It isn't carelessness, it's mixing two
modes: "I wrote down everything I intended" and "I checked that it hangs together" are different
jobs.
**Fix:** a separate self-check pass before the word "done", looking at the seams first. Every
mandatory requirement has to have its own checkable acceptance criterion — anything left only in
prose won't get done.

The same seam exists in actions, not just texts. The invariant was checked before files were
deleted and not checked before they were renamed at the next step, even though it breaks the
same way either way. The probe caught it, not the agent. **Technique: tie the invariant to a
CLASS of operation ("I'm touching the name or path of a production file") rather than to a
specific step, and run it after every sitting.**

**5. A "green check" with no denominator.**
The reconciliation probe reported "0 discrepancies" — and that was true. It just happened to look
inside **one file out of 41**: the rest had no text layer, they were scans. The acceptance
criterion would have gone green even for a completely wrong change. That was the third case in a
row where a green automated check couldn't see the thing the task existed for.
**Fix:** zero violations with no denominator means "everything matched" and "nothing could be
checked" equally well. So before saying "verified", you state **N out of M**; low coverage is a
reason to check the remainder another way, not to close the question. That's also where the
report format "1246 tests, probe 737/737, 0 discrepancies" comes from — you can't write it
without running it.

**6. A ready-made number from your own database still isn't a fact.**
Three cases in one week, and all three nearly cost a wrong decision:

- **Cache instead of the primary source.** The agent named four bond put dates, taking them from
  its own table — neat, ready, sitting right there in the database. Three of the four were wrong,
  in places by six months: that date doesn't come from the exchange, it's computed by a
  heuristic. The decision being made was whether to exit.
- **A field's label instead of its meaning.** The conclusion "multiples are computed on a
  five-day-old price" was drawn from the name of a date column. In reality that field holds **the
  Monday of the week**, while the price in the neighbouring column is **the last close of the
  same week**, i.e. yesterday's. We nearly rebuilt a working mechanism onto daily prices.
- **A hand-written SQL query instead of the canonical figure.** Asset class shares were being
  computed from a capital figure assembled by a custom "latest date" query — and that query
  silently dropped two portfolios whose snapshot was two days old. The base was off by 4%, and
  the decision "accept the overweight" was being made on exactly that. There was no real
  overweight: the share differed from the corridor boundary by hundredths of a percent.

**Fix:** three rules. A number that a decision will be made on gets checked against the primary
source, even if it's sitting ready in the database. If you're drawing a conclusion from a value
in the DB — **open the script that writes it**. Capital, weights and shares are taken with the
canonical command; a custom query is acceptable only as a supplement, and then any discrepancy
has to be explained.

**7. A recorded trap ≠ a planned fix.**
The defect was described in detail in the reference document — and never fixed: it became known
without ever making it into a work order. It surfaced with the question "and where exactly did
you write that down?"
**Fix:** a widespread defect goes into both the document and a task in the same pass. The test for
"widespread" is simple: **a defect in the tooling — the runner, the gate, the environment — is
almost always widespread.** And what gets recorded in the document isn't "what broke", it's how
to tell this defect from a similar one and what to do about it.

## What's built and what's next

**The system is built and working.** The plan holds 21 epics and 143 tasks, of which **126 are
closed** — the statements pipeline, the news circuit, the dashboard, portfolios and returns,
bonds, macro data, the showcase, watch reminders, the "verdict → action" loop, and moving
services onto a shared server.

Not much is left, and most of it isn't development any more, it's **work with the data**:

- **Processing the statements queue** — the biggest current task by volume: a queue of ~150 PDFs
  across 13 companies. It goes one company per session, each on its own branch, closing gaps in
  the quarterly series along the way. This isn't code, it's analysis
- **A full quarterly session** — retro → issuer review → rebalance on half a year of data. The
  first complete cycle across the whole accumulated database, which is what all of this was built
  for
- **The "Retro" prompt** — matching the quarter's verdicts against actual outcomes:
  right / wrong / noise
- **RAS for everyone who has it** — a second numeric layer for issuers whose IFRS comes out with
  a lag
- **Containerising the core** — the only large epic not yet started: right now Docker holds the
  dashboard while the pipeline is still local
- **A bond screener** across the whole market, on top of the yield engine that already exists

## Scale

- **601 commits**, June–August 2026
- **over 1200 automated tests**, green is mandatory before any delivery
- **737 traceability checks** back to the primary source
- 21 epics, 143 tasks — **126 closed**; 62 pipeline scripts
- Sources: issuers' PDFs, the exchange's ISS, the Central Bank, a broker API, Telegram, an
  investment social network, video with transcription
- Stack: Python 3.13 (uv) · SQLite · FastAPI · React + Vite · Docker · pytest · Playwright
