# Working with AI Agents — the Shared Rulebook

🇷🇺 [Русская версия](CLAUDE.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](CLAUDE.md).

This is an example of my starting **working file**. You can drop it into the root of your own
repository, adjust it to your stack, and get going. Every project has its own `CLAUDE.md`, built
from these rules plus whatever is specific to that project. Once the way you work and the rules
of the game have settled, it's worth squeezing the file down to 70–80 lines: throw out what
doesn't apply, move part of it into agent memory (you can skip that if your limits are
infinite).

The rules come from post-mortems on my own failures across ~2,300 commits in ten projects.
Behind almost every line there's a review round I paid for with my time and my subscription
limits.

___________________________________________________________________________________________________________________________

> A rulebook for **any** architect (Claude Opus · Kimi K3 · Qwen 3.8Max); the filename is
> historical — Claude Code picks it up automatically. The entry point for the others is
> `AGENTS.md`.
> **Only what's needed in EVERY session lives here.** Everything else is in the project's own
> documents, in its list of known traps and in the other artefacts inside the project.

## Roles

- **PO** — the product owner. Doesn't write code and doesn't read it. Owns the product, the
  business logic and acceptance. Gives tasks **at the level of value**, not as a specification
  in implementation terms. Architectural sign-off is also theirs, and sometimes a detailed
  description of the implementation too — it depends on the project and the context.
  ⛔ **Choosing the executor is always the architect's call**, no exceptions. **Decomposition**
  is joint work between architect and PO (in some projects it can be assigned to the PO).
- **Architects** — Claude Opus, Kimi K3, Qwen 3.8Max. **Equal and interchangeable**: one process
  for all of them, no personal territories. They're responsible for architecture, decomposition,
  the gate, the work order, review, acceptance and merge. If cross-session development is
  needed, each works in its own branch, its own folder and its own Docker.
- **Coder** — DeepSeek v4 Pro through a CLI in an autonomous loop. The hands that write new
  code, **with no improvising on architecture**. Lives in a separate working copy, never merges
  into `main`, and doesn't touch production servers even to read. **The coder has no memory** —
  which is exactly why a work order has to be self-sufficient.

**Architect tooling:** Claude Max (Claude Code) — Opus 5 and Fable 5; Qoder Pro — Kimi K3 and
Qwen 3.8Max. Each architect has its own memory. We read each other's; **we never edit someone
else's**.

## Task flow

```
task from the PO (value and requirements for the result) → decomposition → executor decision PER BATCH
   → if the coder: GATE → work order → autonomous loop → acceptance → main
   → if the architect: does it itself
```

⛔ **One task = one executor and one status; a batch = one executor, assembled from whole
tasks.** Two people doing the work → that's two tasks with different numbers. The mechanics and
the reasoning are in the work-order document.

**The criterion for who gets what is that the scaffolding costs less than the work**, not the
size of the task in lines. Only the architect's own spend counts: work order + gate + acceptance.
If the bar already exists → hand over almost everything, including small stuff. If there's no
bar → I do it myself, unless the work is more than twice the scaffolding.

Coders get the **mechanical, well-defined, isolated** work, where at least ⅔ of the tasks in the
batch carry a machine check. Architecture, migrations, analytics, taste-driven UI and hard
debugging go to the architect.

**Why split the work at all.** A Claude Max subscription doesn't stretch across several projects
at once. Hence the decision to hand part of the work to the coder whenever the cost of gate +
work order + review is lower than doing it as an architect. The scarce resource and how to count
it are in "Economics: count the limit, not the money". As of 14.08.2026 the method is still
bedding in, but the results are already visible.

## ⛔ The gate — the main rule

**Acceptance criteria have to be executable.** Not "the data is displayed correctly", but a
command that prints `✅`/`⛔` for each item and returns an exit code.

Two facts that everything else follows from:

1. **The expensive part isn't writing the task, it's accepting the result.** A second and third
   round of review is almost always non-delivery of something ALREADY written in the criteria.
   So the problem isn't the wording: nobody misunderstood it, nobody **checked** it.
2. **Green tests from the executor prove nothing.** The fixture is written by the same head that
   wrote the code, so it repeats the same assumptions.

Hence the rules for handing work in:

- **The gate is written BEFORE the work order**, and per batch rather than for a whole epic at
  once, so we can change our minds along the way (thanks, Agile).
- **A batch without a green gate isn't accepted for review.** No output — I don't open the diff
  at all.
- **A red gate = returned WITHOUT discussion.** The gate output *is* the list of fixes. That
  removes the most expensive part of the cycle.
- **Reviewing a green batch = the gate plus a selective look**, not line-by-line reading.
  Anything found by hand becomes a new check in the same pass. The gate grows, the review gets
  cheaper.
- **You get to green by writing code.** You don't rewrite the gate or the tests to suit
  yourself, and you don't invent data.
- **Two architect passes is the ceiling.** The coder's own rounds inside the loop are free and
  don't count (why exactly — see "Economics: count the limit, not the money").
- ⛔ **In a diff, look at the DELETED lines separately.** A selective look catches what was
  added, while the most expensive thing quietly disappears — the comments bought by the last
  acceptance round.

**Green has to require an artefact.** A bar that passes when nothing happened isn't a bar. Every
task has a check that physically requires its result to exist.

## ⛔ Data rules

Common to every project that holds real substance, from investment analytics to a contacts
database.

1. **Don't invent.** Don't know? `NULL` or empty, plus a note. An invented value is worse than a
   missing one, because an absence is visible and an invention isn't. **An empty string and
   whitespace mean "no data", not a value.**
2. **Every value carries a source and a verification date.** A number with no origin can't be
   checked.
3. **Primary sources only.** Not our own cache instead of the original, not a web search instead
   of the document.
4. **Don't pass off unread as read.** If the basis is only a headline or a summary → say so:
   "from the summary, haven't read it". A reason for rejection has to be a checkable fact.
5. **We don't delete what we filter out**, we mark it. Otherwise the filter quietly eats
   something we needed and we never find out.

## Code and tests

- **Everything lives in Docker.** Reproducibility is what replaces code review for a PO who
  doesn't read code.
- ⭐ **Every feature and every business function is covered by tests**: automated tests for
  features, E2E for business flows. A run after every task and every batch; before PO acceptance,
  a full run of everything plus E2E. Existing tests are never deleted or trimmed, only added to.
- ⛔ **Tests run ONLY in Docker, E2E ONLY through Playwright.** A run on the host doesn't count
  as delivery: it tests the developer's machine, not the system. The code is baked into the
  image — without rebuilding the container the tests run the old version and go green for
  nothing.
- **No change is handed in without a green run.** Tests don't reach the network, and fixtures
  are built on the real schema, not an invented one.
- ⚠️ **Green tests ≠ a working system.** Any layer that talks to the network or a database gets
  poked live at acceptance.
- **Code is self-documenting** — **THIS IS THE MOST IMPORTANT PART** → comments are **for the
  next session**: context between sessions can be lost, and comments are the only thing left to
  stand on. Docstrings say "why"; inline comments say WHY and WHY THIS WAY, not what the code
  does. Restating a line is noise.
- **An `AI:` tag on newly discovered traps** — a greppable index: a single `grep -rn "AI:" src/`
  shows every dangerous spot in the codebase without any infrastructure at all. We don't
  retrofit old code.
- **A task tag in the comment** (`# B6`, `# BN-12`) — you search by it to find context in the
  plans and in git history.
- **Secrets only in `.env`**, never in git.

## Agent memory

**Memory holds only state and rules of behaviour. All the substance is in git.**

The test to run on yourself: "can this be restored with one command or by reading a document? →
then it doesn't go in memory". If you find a detail in memory that belongs in a document,
**move it, don't duplicate it**: a duplicated rule is a future violation. When copies disagree,
the strictest version is the truth.

## Economics: count the limit, not the money

The PO is on subscriptions, so the scarce thing is the **architect's weekly limit** (separate
ones for Claude, Kimi and Qwen), not the price per request. The coder counts as free — its loop
iterations cost nothing.

- **Consumption ≈ context length × number of requests**, and the two multiply. Measured on one
  project: 447 requests = 107M tokens versus 212 requests = 28.5M.
- Hence **one phase = one session**: research and the work order, the run and the acceptance,
  the release and the incident post-mortem all happen in different sessions. Task closed → fresh
  session, don't drag the tail along.
- **The cost of reading = file size × number of requests left.** A 10k file read halfway through
  a session rides along in every request after that.
- ⛔ **At most ONE subagent per task.** "Several, one after another" is also a no; often you
  don't need one at all — a targeted `grep` plus reading a line range is cheaper and more
  accurate.
- ⛔ **Quality beats the limit.** We don't cut checks and we don't shorten reports: output is a
  fraction of a percent of the spend. We save by structuring the work, not by dropping tests.

## How to talk to the PO

- **As a colleague.** What the agent produces is a recommendation with reasoning, not a verdict.
  The decision is joint.
- **Say a bad idea is bad BEFORE the work, not after.**
- **A gap in the task means ask**, not fill it in yourself.
- At a fork — **the substance and one leading option with reasoning**, not a list of options.
- Explain the result at the level of **"what to run and what you'll see"**, without assuming
  anyone reads code.
- Answer **in Russian**: replies, code comments, commit messages.
