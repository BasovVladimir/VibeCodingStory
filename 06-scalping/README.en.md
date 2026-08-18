# scalping — mentoring in manual scalping on the Moscow Exchange

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**142 commits · July–August 2026 · architect Claude Opus 5**
The core is learning and review; the tools and the code-writing are secondary. ~40% of the plan
is delivered.

## What makes this project different from the others

In every other project the agent is an executor: it designs and builds a system and I accept the
work. Here it's **a mentor**. The tools in `tools/` are props for the learning: if it turns out
tomorrow that a quiz in chat teaches better than the trainer does, the trainer goes into the
archive.

## How I work with the agent

Here the agent works as a mentor, and the day is built around lessons rather than tasks.

At six in the morning the screener sends me, on its own, what's worth scalping today in Telegram.
The working window at the order book runs from four until half past five in the afternoon: one
instrument, one setup, 1 contract, with the panel next to me. During that time I write nothing
at all — the panel keeps the log itself. After the session I sit down with the agent to go
through it: the trade journal against the recorded order book, with discipline assessed
separately from P&L. In the evening it's "let's go read the news": the agent works through the
tape and the macro data and puts together a plan for tomorrow.

A lesson itself looks like a quiz in chat. The agent pulls frames out of the recording, shows
them without the outcomes, I answer, and then comes the breakdown with the actual numbers. And
here the agent has a right it has in none of my other projects: it holds discipline, position
size and module gates firmly, and arguing about them is pointless. The urge to jump a gate is
exactly the impulse that costs money in scalping.

The code side runs in the background. I set the priority and the run mode, while the decision
"do it myself or hand it to the coder" is made by the architect on its own, per batch, and it
doesn't come to me with that. A report after each task, **no merges happen without me**, and runs
happen only in the container and only in full.

I go in where a tool touches market numbers. Contract codes, thresholds, the semantics of
direction in the tape — precisely the things a coder will write plausibly and wrongly, and the
gate will happily let through.

## The goal

I'm a long-term investor and in parallel I'm building an algorithmic trading robot (MyTrade).
Manual scalping is a third, separate skill. Analytical strength barely helps here, and that was
the first thing I had to accept.

The goal of the project is to learn to trade intraday by hand, consistently: the classics of
order book and tape reading, plus trading the news. Everything else is derived from two
constraints.

**Constraint one — nerves.** Broadly, the overall goal of the project is to embed the trading
skill — to trade on reaction, without deliberating, but still make the right decisions. At a
large size, fear and greed take the controls and the reflex doesn't get recorded, which is why
you can only learn on small account sizes. Hence the rule **"one instrument, one setup, 1
contract"**: emotion is removed **through position size**. And hence the main consequence for the
product: **the P&L of the first months is noise** — the product of this project is discipline and
per-setup statistics, not profit.

**Constraint two — costs.** Over a trade horizon of a few points, commission and spread eat more
than bad forecasts do. The round-trip cost on the contract I trade is **1.17 points**, the spread
wanders between **0.10 and 0.55 points**, and the break-even win rate after commission is
**≈61%**. So any idea is first tested against "will it survive the costs" and only then against
"is it a pretty idea". Everything we tried mechanically got eaten by costs.

## How it works

**A triple loop, deliberately separated in time:**

| Loop | What | When |
|---|---|---|
| **Recognition** | replay and quiz over a recorded order book, run by a script at the start of trading — hundreds of repetitions with no money involved | outside market hours |
| **Execution** | live trades with 1 contract, **on instinct**, with no analysis in the moment | during the session |
| **Correction** | a cold review from the trade journal and the recorded order book | after the session |

⛔ **Hot execution and cold review are never mixed.** Analysing in the moment breaks the core
logic of training the skill: explicit control destroys the very reflex we're wiring in. The
practical consequence that came out of this — **during trading hours I write nothing at all**:
the panel keeps the metrics log itself, and observations get talked through after the session
closes.

**A day looks like this:**

```
06:00  the morning screener sends "what to scalp today" to Telegram
       (range, turnover, initial margin, affordability for the account, catalysts)
       ↓
16:00–17:30  the working window at the order book: 1 contract, one setup, panel alongside
       (regime · flow · walls · breadth — the log writes itself)
       ↓
evening  "let's go read the news" → the agent works through the tape and macro, picks out
       what's event-driven, creates events and reminders → a plan for tomorrow
       ↓
after the session  review: the trade journal against the recorded book, discipline separate from P&L
```

**The lesson format is "a quiz in chat".** The agent pulls frames out of the recording (order
book + tape + delta), shows them **without the outcomes**, I answer "it holds / it gets eaten"
and "up / down / flat", and then comes the breakdown with the actual figures: what happened next
and by how many points. A score of "2 out of 6" is broadly normal, given that a mechanical signal
on the same data produces 42%.

**The evening plan is a playbook of entries.** A plan that says "don't trade today" is one I
reject: I'm going to open positions anyway, since I'm used to learning through action and
mistakes (theory through practice). So for every item on the watchlist we write a table of
scenarios: "outcome with numbers → interpretation → reaction → which button to press". "Don't
trade" remains only where there's liquidity chaos: a spike, the first 15 minutes after a
headline, a thin book.

## Architecture

**Two tracks of a different nature live in one repository**, and that's the project's main
decision:

| Track | What it is | Who runs it |
|---|---|---|
| **Learning** (modules 0–7 with gates) | theory, homework, quizzes, reviews, setup passports | the mentor-architect, entirely |
| **Code epics** | tools that take the routine out of learning | the architect + the coder under a work order |

The boundary is held by a rule: **teaching, reviews, setup formulas and thresholds are the
architect's alone.** The coder receives the analysis **already finished, right there in the work
order**, and doesn't invent thresholds.

**We don't accumulate market data ourselves.** The L2 book and the tape arrive as slices from the
neighbouring myTrade project under a read-only contract: it writes them forward, we consume them.
The reason is hard: **there is no history of L2** — only forward recording, so a second collector
would mean a second source of truth with gaps and a permanent question of "which one is right".

**The panel is read-only on principle.** It answers "should I be trading right now, and which way
is the pressure", but **gives no entry signals and sends no orders**

**Every market value carries a source** (MOEX ISS / T-Invest / an L2 slice / a calculation). A
number with no origin can't be checked, and a nice story about the market is more persuasive than
data, which makes it more dangerous. If we don't know, it's empty with a note.

## Tools instead of screens

There are no interfaces here — it's all CLI, all run in a container. Each tool answers one
question in the learning cycle:

| Tool | The question it answers | Status |
|---|---|---|
| **brief** — the morning screener | "what's worth scalping at all today": range, turnover, initial margin, affordability for the account, catalysts | 77 tests, live runs, weekdays 06:00 to Telegram |
| **panel** — a live microstructure panel | "should I be trading now and which way is the pressure": regime, window delta, absorption, persistent walls, index breadth | 112 tests, a bar of 79 checks + 13 live ones |
| **l2lab** — trainer and review | "what am I seeing in this frame" and "what actually happened in my trade": replay, quiz, post-mortem, flow, oracle | 27 tests, data kept outside git |
| **tjournal** — automatic trade journal | "how do I actually trade": FIFO round trips → win rate, PF, expectancy, R | 5 tests |
| **scalpcal** — an events and reminders hub | "what do I need to remember and when": events, individual reminders, digest, actions | 144 tests, in Docker on the shared bot server |

Two tools live on the shared bot server on timers (the morning brief and the notifier); the rest
run locally. The notifier is **cross-project**: neighbouring projects order deliveries through a
contract, and we own it.

⚠️ Thresholds in these tools are set **from measurement**. A recent example: the "dead
instruments" cut-off computed the spread in ticks and threw out my main contract every day — its
median spread is 5 ticks (3,774 samples from a live log), while on the neighbouring instrument
the same amount of money fits inside 1 tick. The threshold was rewritten in terms of round-trip
costs.

## Project structure

```
PLAN.md                  the compass: the learning track (modules + gates) and the code epics
CLAUDE.md · AGENTS.md    project rules for any architect · entry point for non-Claude agents
TESTS.md                 live coverage: what's covered, how to run it, where the gaps are

docs/метод-обучения.md      HOW to teach this particular student — reflexes, not erudition
docs/журнал-обучения.md     where we are now: progress, diagnoses, session reviews
docs/торговая-механика.md   costs, reading the order book, measurements instead of opinions
docs/вечерний-ритуал.md     the "news → events → plan for tomorrow" procedure
docs/скальперский-трек-и-робот.md   how practice turns into hypotheses and verdicts
docs/инструменты-справочник.md      how the tools, servers and sources work
docs/practices/             roles and workflow · code and tests · the coder gate · agent lessons
docs/наряды/                work orders for the coder by epic (+ выполнено/, runs/)

tools/                   scalpcal · brief · l2lab · tjournal · panel (each with its own tests/)
scripts/                 the autonomous loop's scaffolding: gate.py · coder_run.py · coder_epic.py
deploy/srv/              production plumbing for the notifier: image, compose, units, smoke tests
setups/                  setup passports — pre-registration of hypotheses
calendar/                event snapshots (the truth is the database on the server)
```

## Roles

- **PO (me)** — student and customer at the same time: product, risk, acceptance, every decision
  about money and size. I don't read code.
- **Architects** — Claude Opus 5, Kimi K3, Qwen 3.8Max, equal and interchangeable. Here they have
  **a double role**: mentor (teaching, quizzes, reviews) and engineer (epics, work orders, gates,
  review).
- **Coder** — DeepSeek v4 Pro in an autonomous CLI loop, working in an isolated copy, never
  merging into `main`. It's brought in **selectively, for code epics**.

**The limit of collegiality is specific to this project.** In the engineering loop it's like
everywhere else: the architect gives a recommendation with reasoning and the decision is joint.
In the mentoring loop there's a limit — **discipline, position size and module gates aren't
"recommended" by the architect, they're enforced**. The urge to jump a gate is exactly the
impulse that costs money in scalping.

What follows for splitting the work: a task goes to the coder only if it's mechanical and
verifiable by execution. A setup formula, thresholds, interpreting data — not there. **The
"coder or myself" decision is made by the architect per batch and isn't escalated to me**; what
comes to me is priority and run mode.

## Principles

1. **Don't invent numbers.** Don't know? Empty, with a note. An invented level, margin figure or
   win rate is worse than a missing one. **Numbers always beat a nice story.**
2. **Every market value carries its source.** A number with no origin can't be checked.
3. **No automated trading.** The tools prepare and highlight — the decision and the click are
   always the human's. The panel gives no entry signals, and that's enforced by a gate check.
4. **One instrument, one setup, 1 contract.** Size is about neurology, not about money.
5. **Measurement instead of opinion.** Any "the market usually does this" is either a number with
   a source or a note saying "hypothesis, unconfirmed". A percentage without a base rate means
   nothing.
6. **Everything in Docker; no change is handed in without a green run in the container.** Tests
   don't reach the network and use synthetic data; if you added tests, update `TESTS.md` — that's
   part of acceptance.
7. **Green tests ≠ a working tool.** Any layer that talks to the network or someone else's
   database gets poked live at acceptance.

## Key mistakes and how we handled them

**1. Green tests on a dead tool.**
The coder wrote a solid exchange API layer — 19 green tests — but mixed up the exchange section:
`stock` instead of `futures`. The response came back syntactically valid, with **zero rows
instead of 481**. The fixture mocks the network and physically cannot see a wrong endpoint: green
test, dead screener. It was only caught by a live request at acceptance.
**Fix:** accepting any work order involving an external source = a green gate **plus** a separate
live run, and what gets checked isn't "a response arrived" but **whether it adds up**: the number
of rows, the spread of values, familiar tickers. "Not empty" can be faked by a stub; adding up
can't.

**2. Domain knowledge beats syntax.**
Same class of problem from the other side: the affordability formula was clean, but the
instrument in it was named wrong — the contract codes were mixed up (`MIX` instead of `MXU6`).
Plausible code passes a review "by syntax" and fails on meaning: the coder is strong on structure
and weak on domain knowledge.
**Fix:** fields, instrument codes, the semantics of direction in the tape and time zones are
checked by the architect **against reality, not against plausibility**. That's exactly the part
of the work that can't be delegated — it *is* the expertise.

**3. A green run doesn't prove coverage.**
Four work orders took the main tool from **0 to 144 tests**. The counter looks convincing, but
the author of the tests and the author of the code are the same head with the same assumptions: a
test that doesn't fail on broken code checks nothing.
**Fix:** accepting a testing work order became **mutation-based** — the code gets broken in 4–5
places and we confirm the tests fail, plus a run with the network disabled wherever it's mocked.
A side effect of that same pass: an end-to-end test uncovered two **pre-existing** CLI bugs that
nobody had seen.

**4. A work order isn't "done" until the architect has run its acceptance criteria itself.**
In the work order for migrating a service, the criterion looked flawless — and physically didn't
work: a command in docker compose **replaces** the image's command, a different flag was needed,
and the deployment directory wasn't copied into the image at all, so there was nothing to run "in
Docker" with. On top of that the install script computed the project root one level too high.
Both defects were found by **reproducing the layout locally in five minutes**.
**Fix:** a plausible-looking command in the "acceptance criteria" section that nobody ever ran is
defective work. The coder has none of the architect's memory and no right to improvise: it will
hit the wall and stop. And the holes appear **at the seams** — a work order is written over
several sittings, the logic within each chunk is intact, and it's the join that tears: a
requirement written in prose that never made it into the criteria.

**5. A broad `pkill` killed someone else's work — A VERY IMPORTANT LESSON FOR OTHER PROJECTS.**
While stopping its own coder run, the agent executed `pkill -9 -f "kilo run --auto"` without a
path — and killed **a coder loop in a neighbouring project**, started by a different session. The
data survived by luck: that loop had only just started. The first command in the same sequence
had used a full path — meaning the rule was known and got broken "in passing".
**Fix:** stop processes **by PID only**, obtained from `pgrep` and read with your own eyes; broad
patterns are forbidden, and for Docker there's no `prune`/`stop` without an explicit container
name.
⚠️ The tail of the same story: the runner kills its own process while **the grandchild survives**
— an orphaned agent spent two hours editing the working copy while a review was running against
it, and restored a mutation that had already been rolled back. So after stopping a loop we check
that the coder is actually dead.

## Development plans

**The priority is the learning track.** But there are code changes too:

- **The panel** — calibrating thresholds on live sessions and wiring it into the ritual; the
  core, the metrics and the live layer are already accepted
- **The evening ritual → events** — this closes the pipeline: news and macro become events that
  feed the morning screener and the notifier
- **A dynamic account size in the screener** — affordability computed from the real account
  balance pulled from the neighbouring investment system, instead of a hard-coded figure
- **Level alerts** and **a forward order log** — the second one will let me review not only what
  was executed but also "what I intended to do and then cancelled": the API doesn't provide
  cancelled orders historically
- **Validating setups with data** — separating winners from losers by context; the naive
  threshold has already been disproved, and what's needed is a statistically meaningful number of
  observations
- **Tech debt:** end-to-end tests for two older tools that currently only have unit tests

## Scale

- **142 commits**, July–August 2026
- **428 automated tests** across seven suites, all run in a container; the counters in `TESTS.md`
  are checked by the gate — if they disagree with the actual run, the gate goes red
- **12 business epics and 5 pieces of tech debt**, of which 5 epics and 3 debt items are closed
- **5 tools**, two of which run on the server on timers
- The learning track: 8 modules with formal transition gates, setup passports, a review journal
- Data: the L2 book and tape from the neighbouring project under a contract (13,979 book
  snapshots and 106,025 prints in the first slice), the exchange ISS, a broker API, a news
  database
- Stack: Python 3.12 (uv) · SQLite · Docker · pytest · tkinter · stdlib by default
