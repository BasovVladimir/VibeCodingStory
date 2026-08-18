# mytrade — algorithmic trading: three engines and a machine for finding edge

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**623 commits · June–August 2026 · architect Claude Opus 5 · review by Fable 5**
The flagship. Right now one robot trades on paper around the clock, the research pipeline is
running, and ahead lies the whole grid of markets and timeframes, plus building live trading
robots.

## What makes this project different from the others

In my other systems you can see the result with your own eyes: the email went out, the report
came together, the checklist was sent. Here there's nothing to look at — you can't *see* whether
a strategy is profitable, you can only prove it. So almost all the time goes into finding
something and then doubting what you've found.

The reason is simple. Try enough variants and a beautiful equity curve will appear, guaranteed —
even on random numbers. Which is why the most valuable thing in this project isn't the
strategies themselves, it's the screening machine: six cheap gates, walk-forward, plateau
instead of a peak, significance deflation across every attempt, a power gate and a stopping
rule. That's mostly what I'm showing here.

The second difference is leveraged money. A mistake in a backtest doesn't spoil a report, it
zeroes an account: with leverage, an imaginary edge usually ends in liquidation. That's where
all the stop rules come from.

## How I work with the agent

A session — or a chain of sessions — starts at the compute node. I take the plan, queue up deep
strategy runs and leave them turning in the background, so something is always being computed
there while I'm busy with something else. A run takes hours, so you have to start it well in
advance.

Then I look at the tasks to see what's left and give the command: "let's do INFRA-4" or "take
26.3.3". The task code leads into the plan and into the strategy doc, so the agent picks up the
context itself — I don't restate it.

After every closed task I ask for a report: what changed, what got fixed, where to look at the
result. **No merges happen without me.** Tests are always a full run before the merge, so I know
for certain nothing came apart.

I might go into a task myself if something there was logically tricky or could blow up half the
platform. Into the logic, not the code: how it works now, what changed in the behaviour, where
it flows next. We've had cases where we carefully built something in one place and something
else broke where we hadn't touched anything in two months. The agent won't see that: its focus
is its own task, while the system as a whole stays with me. Tests cover code well, but logic,
"business value" and breadth of thought are beyond the agent for now...

## The goal

What I want here is multiplying capital. That immediately implies an unusual choice of metric.
Strategies are selected by **Calmar** — return divided by drawdown. The usual pick is Sharpe,
but Sharpe penalises volatility of any kind, including movement upward, and skewed leveraged
strategies need exactly that upward movement. The capital structure is a barbell: a core with a
systematic edge under leverage plus a satellite of convex bets.

The whole architecture grew out of one rule: **any finding is treated as noise until it has
survived the procedure**. Three things follow from it, and they define how the system is built:

- the maths of edge discovery is written once and never copied across strategies — otherwise the
  bar ends up stricter in one corner of the project than in another;
- one strategy codebase serves three modes — backtest, paper, live. Split them into separate
  implementations and the classic "it was profitable in the backtest but somehow not live" is
  guaranteed to show up;
- compute doesn't relax the discipline, it tightens it. The wider the search, the more random
  winners slip through the checks.

## How it works: the screening funnel

A full run means hours of computation, so before it a candidate passes **six cheap gates**, in
increasing order of cost. Fail any of them and the verdict is issued right there; we go no
further.

| Gate | Question | What it kills on |
|---|---|---|
| 1. Applicability | does this cell even exist? | carry without a perpetual contract, a calendar spread without a futures market — there's no cell |
| 2. Structural prior | is there an **economic reason** to expect mean reversion, carry or trend? | cross-sector pairs "cointegrate" with no relationship at all — just two random curves |
| 3. Tradability | can we actually trade both legs? | no shorting, no liquidity, the data source won't give us the data |
| 4. **Data power** | are there enough independent events to judge at all? | the main filter: we count independent opportunities, not bars |
| 5. Probe + oracle | is there anything to catch at all, before any search? | "dead for free" — the ceiling for a perfect trader is below the level worth bothering with |
| 6. Costs | will the gross edge survive commission and spread? | the signal's range is smaller than the cost of a round trip |

Screening has three outcomes:

| Verdict | What it means | What we do |
|---|---|---|
| 🟢 **worth digging** | passed the cheap gates | full run |
| 🔴 **no edge** | we had enough to judge, and there's nothing to catch | stop; the cell is closed, with its boundaries recorded |
| 🟡 **cannot judge** | not enough data, the source is silent, N wasn't reached | postpone: the cell stays and waits for data |

The most expensive mistake in research is confusing red with yellow. A month later "we didn't
have enough data" turns, in memory, into "we looked there, there's nothing", and the direction is
closed forever. That's why a yellow cell physically stays in the matrix along with all of its
gates.

**A full run** is nine steps, each with its own gate: horizon derived from the signal's
half-life → a coarse parameter grid → the metric on the worst out-of-sample fold → plateau
instead of a peak (we read the shape of the surface: a flat centre with slopes is fine, a lone
spike is not) → checking the cloud on new data → significance deflation across every attempt →
the power gate → the stopping rule → the held-out chunk and a forward test on the live stream.

## The oracle: a map of where there's actually meat

A separate machine computes the ceiling of an all-knowing trader: what someone who knows all
future prices in advance would have taken — already net of honest costs. Inside it's dynamic
programming over bars, with no strategy search: a linear pass, daily bars computed in seconds,
minute bars in hours.

The ceilings are laid out in a grid of 64 cells (strategy × asset × timeframe band), and it's
read alongside the status matrix:

- "no edge" **and** a tiny ceiling → closed forever, and for *any* strategy, not just ours;
- "no edge" **and** a fat ceiling → there's meat, our formulation just doesn't capture it → a
  candidate to revisit with a different idea;
- edge found → we compute the capture ratio: what share of the ceiling we actually take.

⚠️ The gate is one-directional, and that was written down before the runs so nobody goes looking
for a convenient interpretation afterwards. "Dead" cancels expensive work. "There's meat" grants
no green light: a random walk is also full of meat in hindsight, there's just no way to take it.
Tuning strategy parameters to the oracle is forbidden — that's a direct route to overfitting.

Three more reading rules were also fixed in advance. The finer the timeframe, the higher the
ceiling (you act more often), so comparing on a "where's it greener" basis is meaningless — you
have to look where costs don't eat the gain. "The best pair out of three hundred" is inflated by
selection in hindsight. There are no universal "meat / thin / dead" thresholds: a cell holds a
number, and the bar is read through the lower bound of what's achievable for that particular
strategy.

## Architecture

One shared machine (~85% of the code) plus thin, swappable signal modules (~15%) and an
allocation policy on top. A new strategy is a module and a config, not a new application.

```
Data → Strategy (signal) → Risk and sizing → Execution → State → Monitoring
```

The same strategy code lives in three modes; only the data source and where execution happens
change:

| Mode | Data | Execution |
|---|---|---|
| **Backtest** | history from storage | cost simulator |
| **Paper trading** | live stream | our own execution simulator |
| **Live** | live stream | a real account |

Paper mode mirrors the backtest engine bar for bar, and there's a dedicated test for that.
Without it, a divergence between backtest and live trading only shows up once money is involved.

**Three shared brains that are never duplicated:** arbitrage maths (cointegration, beta, spread,
z-score, half-life, basis) — every arbitrage strategy calls into it; indicators — everything
directional; search primitives (plateau, deflation, horizon, walk-forward, metrics) — everything
at all.

The key abstraction is the **"leg economics"** object. A single object describes how a position
is computed: a share with lot sizes, a future with a multiplier and initial margin, a perpetual
contract with funding and margin. Thanks to it, one backtest engine computes results for any
instrument: the futures mode slotted in without touching the equities one, and the crypto mode
without touching either.

Strategy engines are independent: a change to one must not break another. Shared building blocks
are touched read-only or additively: a new parameter always has a default, so the behaviour of
existing strategies stays byte-for-byte identical. That requirement is hard and not open to
discussion.

## Regression research: "a strategy as an equation"

A separate machine answers two questions before any trading logic exists: **does the variable
have an effect** on the target event, and **is that effect alive over time** — it's effectively
the factory for all new strategies.

```
hypothesis (object + event + a SMALL, declared menu of variables)
   → feature dataset with no look-ahead
   → partial effects: multifactor, autocorrelation correction, multicollinearity control
        🚦 no effect → stop, cheaply
   → effect-strength curves over time + break tests + half-life of the effect
        🚦 the effect is dead or chaotic → stop
   → the exam: "did it predict forward, continuously?"
        🚦 it didn't → stop, an expensive funnel saved
   → explanation through market regime → candidate mechanic → the full funnel
```

The discipline here is stricter than the computation itself. The menu of variables is ordered
before the run, thresholds are declared in advance, the windows for the strength curves are
fixed, and if verdicts across windows disagree we take the worst one. Otherwise it goes like
this: you pick the "main" variable after the run, based on how pretty the result is — and that's
already curve-fitting, it's just invisible from the outside.

## Infrastructure: three nodes with opposite workloads

| Node | What it does | What matters |
|---|---|---|
| **Compute** | series storage, the collector, backtests, the funnel, parameter search, walk-forward | cores and memory; can be switched off between runs |
| **Trading (RU)** | the live loop: stream → signal → risk → execution | 24/7 uptime, security |
| **Live crypto** | the same code sitting next to the exchange, no database needed | proximity to the exchange, key custody |

The decoupling here is deliberate: the trading node doesn't depend on the compute node. It keeps
local state and pulls data straight from the broker, while the compute node only periodically
hands it the list of what to trade. If the compute node goes down, trading carries on.

Keeping the money safe is requirement number one, above any feature: a trading key with no
withdrawal rights, key-only access, an address restriction, minimal funds while testing. The
first live order from any engine goes through a separate review by the second model.

## Project structure

```
src/mytrade/
  core/          config · models · retries · throttling · compute pool · checkpoints · memory watchdog
  broker/        one venue interface: two Russian brokers + a crypto exchange
  data/          series storage, a collector for 13 timeframes, continuous series, crypto collector
  backtest/      🧠 engine + walk-forward · plateau · deflation · horizon · oracle · run registry
  indicators/    shared indicators
  screening/     the funnel, spread screeners, streaming mode
  research/      regression research: features · effects · strength over time · the exam · regimes
  risk/          sizing, limits, guards, kill switch, growth risk management
  execution/     simulator, live execution, basket-of-pairs accounting
  engine/        runtimes: paper, live, carry; feeds; state
  strategies/    🔴 THE UNIQUE PART: the signal rule and leg assembly

docs/основное-планирование.md      the live matrix: 64 cells, ceilings, batch log
docs/воронка-отсев-и-прогон.md     methodology: 6 gates, 3 verdicts, 9 steps of a run
docs/стратегии/                    one document per strategy: mechanics, sweeps, run logs
docs/регрессионный-ресёрч.md       the research machine
docs/theory/                       the textbook: why this works
docs/вики-аналитики/               the whole path of an idea explained in plain language
```

Three things about this layout aren't obvious:

- **`docs/` weighs more than the code.** Cell statuses, verdicts with dates and boundaries, run
  logs — that's the state of the research, and it's versioned on equal terms with the code. The
  documents have their own rules of upkeep: no strikethroughs or traces of history in the text,
  one status per cell, and a verdict always carries its boundaries of applicability.
- **`вики-аналитики/` is written for me.** The whole path of an idea from hypothesis to live
  trading as a single lecture, with a glossary and analogies. I don't read code — so
  understanding has to live somewhere in readable form.
- **A cell's status can't be edited from memory.** The same verdict appears in several tables at
  once, so when it changes you find every occurrence by search and correct them in a single
  commit.

## Roles

- **PO (me)** — the goal, the risk profile, priorities, launching runs and every decision about
  money. I read and edit the planning documents myself; I don't read code.
- **Architect** — Claude Opus 5: architecture, maths, tasks, runs, deployment, documents.
- **Reviewer** — Fable 5, selectively and where the cost of a mistake is highest: a *positive*
  run verdict before it's recorded, risk maths under leverage, the first live launch of any
  engine. Negative verdicts are recorded by protocol and don't need a separate review.

The escalation rule is simple: if the main model stalls for two iterations in a row, the task
goes to the second one. The asymmetry is deliberate: it's the *good* result that gets sent for
review, not the bad one. A pleasing find is the most likely self-deception, and that's what needs
double-checking.

**IMPORTANT.** There are NO coders on this project. The maths is complex and the risks are high,
so as of 14.08 roughly 95% of the work is done by Fable alone. Opus gets pulled in, but less and
less often, even though it's the one holding the Architect role right now.

## Principles

1. **Edge first, leverage second.** Leverage on an imaginary edge usually ends with a liquidated
   account.
2. **Three verdicts, not two.** "Cannot judge" is its own status: it postpones a cell without
   closing it.
3. **A plateau, not a peak.** A lone spike on the parameter surface is noise. What you need is a
   connected region where the neighbouring parameters are also positive.
4. **Deflation across every attempt.** Each new formulation raises the significance bar for
   everything that gets found in that cell later.
5. **Power is counted in calendar span.** You can accumulate any number of bars while the market
   never once changes regime.
6. **We only sweep what's part of the hypothesis, affects the result, and can't be measured any
   other way.** What's measurable is derived from the data, what's irrelevant is fixed, what
   contradicts the thesis is thrown out.
7. **A verdict always carries a date and boundaries of applicability.** There's no such thing as
   "the strategy doesn't work"; there's "on this instrument, on this timeframe, at these costs,
   with this formulation".
8. **The data collector is untouchable.** Any destructive operation means stop and ask.

## Key mistakes and how we handled them

**1. A test nearly wiped out the collector.**
Storage holds hundreds of millions of bars, accumulated over months on a schedule, and they
can't be restored — some sources simply don't hand out history. And tests work against whichever
database is named in the config. Had I run the test suite on the compute node itself — with a
destructive operation inside — the entire collector would have been gone. We caught it before it
happened.
**Fix:** destructive operations in tests are allowed only against objects with no data or
against temporary tables; any new destructive statement in the code means stop and ask; and the
collector itself has no deletion by design — even delisted instruments stay. The general lesson:
**if only attentiveness prevents a catastrophe, that catastrophe has already happened, it just
hasn't surfaced yet.** And data backups, obviously, are mandatory.

**2. A measurement that was quietly substituted with an approximation.**
Computing the ceilings was supposed to use costs measured from the order book. Because of a bug
in looking up the historical contract, the data wasn't found and the code silently fell back to
an approximate estimate — while the declared mode said "exact". It only came out when we went
through the logs: for 65 objects out of 66 there had been no measurement at all. The numbers had
looked entirely plausible the whole time.
**Fix:** 65 ceilings were discarded and re-measured, and the runs that relied on them were redone.
The bug itself was fixed quickly, but the rule we took from it stayed for good: **a silent
fallback is worse than a crash.** If the exact path is unavailable and an approximate one is
substituted with no explicit marker in the result, you can't trust a single number in the series.

**3. A green verdict on a young instrument.**
The symbol had only existed for a few weeks, but on a minute timeframe it had already produced
thousands of bars. Walk-forward honestly built four folds and the metrics came out good — except
that all four folds covered the same market regime, and there was no information in the sample
at all.
**Fix:** power is counted in calendar span rather than bars, plus a minimum-instrument-age gate
appeared. The "cannot judge" verdict grew out of this too: a small N cuts both ways, so "an edge
on a short history" is yellow as well.

**4. Testing the machine on synthetic data caught a defect in the machine itself.**
Before trusting the research engine with live verdicts, we ran synthetic data through it with a
known answer — data where the effect we're looking for was planted by hand. The scenarios
passed, but one of them exposed a defect in the tail-handling rule. On real data that defect
would have looked like an ordinary result.
**Fix:** that run became a mandatory acceptance step for the machine rather than a one-off check.
The principle reads: **the measuring instrument itself has to be measured against an object with
a known answer**, otherwise it's measuring who knows what.

**5. A run artefact was deleted along with a working file.**
Checkpoints from long runs deleted themselves on successful completion — they were designed as a
restart mechanism and were never meant to be an archive. As a result the per-fold data from a
multi-hour run was lost, and there was nothing left to compare the next one against.
**Fix:** the results of every deep run are now written to a separate file that lives its own
life, and comparability is checked by a parity test — the new path has to produce the same
numbers as the old one, byte for byte. The general conclusion: **a working artefact and a result
artefact have different lifecycles, so they have to be stored separately.**

**6. The gross logic worked while the net was drowned by costs.**
The pair behaved exactly as theory predicts: the spread mean-reverted, every fold confirmed the
logic. But the typical size of the swing turned out to be smaller than the cost of a round
trip — commission plus spread.
**Fix:** the cost gate moved into screening **before** the expensive run, and it counts absolute
values: commission plus spread in ticks, not "percentages by convention". Separately, this grew
into work on measuring real spreads from the order book — you can't take costs from a reference
table, you have to measure them on your own instrument.

## What's built and what's next

**Built:**

- the shared core: a backtest engine for any instrument class, walk-forward, plateau, deflation,
  horizon, metrics, run registry;
- the screening funnel and the 64-cell oracle grid, computed on honest costs;
- the regression machine in full, including the synthetic-data check;
- a series collector across 13 timeframes with continuous series and rollover, plus a crypto
  collector with its own queue;
- three nodes, one broker interface, a risk circuit with a kill switch and a preventive guard;
- **a robot on perpetual-contract funding**: a delta-neutral pair, the significance gate held up
  on re-measurement, and the robot currently runs in paper mode around the clock with alerts.

**Next:**

- **paper gate → live trading at small size** for the finished robot: it needs a stable positive
  result after all costs and a reviewer's verdict on the risk maths;
- a live executor and its plumbing for the crypto node — the technical order check is already
  done;
- a directional single-leg engine on top of the same instrument economics;
- a meta-allocator — distributing capital between strategies, once there are two or three live
  edges;
- running the remaining grid cells and collecting more data where the verdict is "cannot judge".

## Scale

- **623 commits**, June–August 2026 — the flagship project
- **912 automated tests**, including parity tests: the new path has to produce the same numbers
  as the old one
- **64 cells** in the "strategy × asset × band" grid, with ceilings computed on honest costs
- **9 strategies** in the works, each with its own document covering mechanics, sweeps and a run
  log
- **13 timeframes** in the collector — so that half-life-based routing can land in any band, not
  so we can brute-force everything
- Storage — hundreds of millions of bars, the project's irreplaceable asset
- Stack: Python 3.13 (uv) · TimescaleDB · Docker · pytest · gRPC and REST exchange adapters
