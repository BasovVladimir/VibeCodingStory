# students — an assistant for a full-stack analyst mentor

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**46 commits · June–August 2026 · architect Claude Opus 5**
Working and in use with real students. The core is done; the pipelines around it are planned.

## What makes this project different from the others

Here the agent works first and foremost as **an analyst, not a developer**. Meeting summaries,
reviews of a student's project, marks, development plans — that's its main output, and it's
written by reading the source material, not generated from a template. Code is secondary in this
project: it provides the pipeline, the storage and the checks.

The rule the whole roadmap follows from: **"functionality" ≠ "code"**. A task like "produce the
meeting summary automatically" doesn't exist here and never will — a meeting summary is a
judgement about a living person, not a report. So every epic describes **exactly which routine it
takes off my hands** and states explicitly where the boundary runs: what the code does and what
stays with the agent.

## How I work with the agent

The rhythm here is set by the meetings with students.

After a meeting I drop the video into a folder — from there comes local transcription, speaker
labelling and a draft summary. We go through the draft together: the agent proposes entries in
the personal plan and marks, and I approve them. "Covered" and a mark are different events, and
in purely teaching meetings there are no marks at all. I send the finished text to the student
myself.

The SA review works the same way. The agent reads the source of the student's project and writes
two layers: an internal one for me, and the one that goes to the student. I always proofread the
second — the student receives a review as a fact about their work, and the price of an imprecise
phrasing is high.

Development runs in the background. I come with a task at the level of value, while the decision
"one-off job, methodology or epic" and the split into work orders is made by the architect
itself. The coder spins in an autonomous loop, but the runner neither merges nor pushes: what
comes out of its work is a branch and a report, then the architect decides and I accept. **No
merges happen without me**, and the 50-check gate has to be green.

**The agent comments the code for its own next self.** A docstring "why", inline on anything
non-obvious, an `AI:` tag on new rakes and the work-order task tag — so that a month later
what's clear isn't "what this line does" but why it is the way it is. The project moves in
bursts between meetings with students, and without those marks every new session would start
with archaeology.

## The goal

I train full-stack analysts: a business school, a university, private mentoring. Around every
student a routine piles up that doesn't scale: video recordings of meetings, a personal plan
covering hundreds of topics and questions, a summary of each meeting, a review of their project,
marks, growth areas.
With one student that fits in your head. With several it doesn't.

The system takes over the preparation and the storage, **without taking over the judgement**:
the student and their path stay in the conversation, not in a database.

Video transcription runs **locally on the laptop's GPU** rather than in the cloud: meeting files
are large, and there's no reason to push them outside.

## What it does and how it works

**The meeting cycle** is the main working loop:

```
meeting video  →  local transcription (+ speaker labelling)
   →  draft summary: theory → practice → "Extras" (the canon is carried over verbatim)
   →  agreed with me  →  entries in the personal plan + proposed marks
   →  finished text to the student in Telegram
```

Two things in this cycle are held by rules rather than by code:

- **Marking something "covered" and giving a mark are different events.** "Covered" goes in as
  soon as the topic has been worked through; a mark comes only after a test or an interview, and
  **the agent proposes it while I approve it**. In purely teaching meetings we give no marks at
  all.
- **Stable blocks are carried over verbatim.** The list of links, channels and books at the end
  of a summary is canon; it doesn't get rewritten or "improved".

**The SA review of a student's project comes in two layers, both mandatory:** an internal
reference for me (direct tone, a numeric mark, where the student floundered) and a version to
send the student (warm, informal, with none of the internal meta). The reviewer's stance is "an
analytics lead reviewing a padawan": blockers, major, minor, cosmetic, a competency table,
growth areas. The mark is **calibrated to the stage**: the bar for a junior and the bar for a
graduate about to interview are different things.

What's looked for first is cross-cutting patterns rather than typos: a mismatch between "DB model
↔ API ↔ requirements", a JSON field with no home in the database, an entity with no status under
status-centric logic, endpoint collisions, boilerplate tails left over from someone else's
project.

### Tools

| Tool | What it does |
|---|---|
| **the `mentor` core** | CLI + SQLite: students, meetings, plan topics and their status, artefacts; importers from xlsx and markdown |
| **meeting transcription** | video → text, timecodes, speaker labelling; locally, offline |
| **reading specs from the wiki** | mirroring a student's project locally, read-only; a new student is two lines of config, with no code changes |
| **the gate and the autonomous loop** | executable acceptance criteria + launching the coder from a task manifest |
| **limit consumption measurement** | counts calls, average context and the total spend area across a session transcript |

⚠️ The "mentor / student" roles in speaker labelling are **a guess based on speaking time**, and
that's written down in the reference document: the agent is required to verify it against the
content rather than take it on trust.

## Architecture

**The data is the truth; the code isn't the source.** That's the project's main decision and it's
atypical: normally a database is treated as the system of record, whereas here it's **an index**.

| Layer | What | Property |
|---|---|---|
| **Files** (markdown, xlsx) | meeting summaries, reviews, student plans | human-readable, edited by hand, **the truth** |
| **SQLite** | students, meetings, plan topics and their status, artefacts, registry | **regenerated by import**, kept outside git |

Deleting the database isn't scary: it's rebuilt from the files with one command, which is why
there are no migrations in the project — only idempotent schema creation. And the other way
round: the storage **doesn't replace** the human-readable artefacts, because a meeting summary is
read by a student, not by a program.

**We don't introduce Docker, but we write Docker-ready code.** The reason is specific:
transcription goes through a local GPU engine and document export goes through macOS
applications. Those parts won't travel into a container, so they're isolated behind a thin
wrapper, while the data root and the database path are read from config — so that tests
physically can't reach production data.

**The coder's forbidden zones are enforced by a machine check in the runner**, not by the text of
a work order: diff comparison, a list of untracked files, checksums of the database files. `.env`
and the gate itself are out of reach of the autonomous loop. And the runner **neither merges nor
pushes**: the output of the coder's work is a branch and a report, and the architect makes the
call.

## Project structure

```
mentor/                  the core: CLI (typer) + SQLite storage, importers
tools/                   meeting transcription · speaker labelling · reading specs from the wiki
scripts/                 gate.py (executable criteria) · autonomous loop runners · spend measurement
tests/                   pytest: synthetic fixtures, never touching real data

docs/План и реализация.md          the roadmap: epics, value, executor, status
docs/Методика — студент.md         the plan, meeting summaries, marks, SA review
docs/Подсистемы — справочник.md    how the tools work, file formats, secrets
docs/practices/                    roles and workflow · code and tests · the coder gate · agent lessons
docs/наряды/                       work orders for the coder and acceptance criteria

Студенты/ · Презентации/ · Вики/    ⛔ NOT in git
data/                    the database (⛔ not in git, regenerated by import)
```

Two things about this layout aren't obvious and were done deliberately:

- **The database is in `.gitignore`, and that isn't about size.** It can be restored by import
  while the files can't, so what gets versioned is exactly what a human edits by hand.
- **`.env.example` is tracked, `.env` never is.** Of the secrets here one is genuinely dangerous:
  the access token for the wiki spaces, which carries all of the user's permissions. The coder is
  never shown it.

## Roles

- **PO (me)** — methodology, product, acceptance. I come with a task at the level of value; the
  decision "one-off job / methodology / epic" and the split into work orders is made by the
  architect itself.
- **Architects** — Claude Opus 5, Kimi K3, Qwen 3.8Max, equal and interchangeable. All the
  meaning-level work sits with them here: meeting summaries, reviews, marks, wording.
- **Coder** — DeepSeek in an autonomous loop, in an isolated working copy, never merging into
  `main`. It only gets work that's mechanical and isolated from the data, with a machine check on
  at least ⅔ of the tasks in a batch.

⛔ **At most one subagent per task, and only when explicitly asked for.** If one is genuinely
needed, the return format is compact JSON rather than prose: a subagent's report settles into the
main context and gets dragged through the whole session.

## Principles

1. **The canon isn't composed, it's verified.** A stable block or phrasing is carried over
   verbatim, after checking it against 2–3 sources. The canon is the version that **recurs time
   after time**, not the most recent file: the most recent one may be defective.
2. **When copies disagree, the truth is the one a human edited.** Between generated and
   hand-edited, the hand-edited wins.
3. **Scope doesn't expand to neighbouring artefacts.** "Prepare the summary" means the summary
   file, and nothing else.
4. **Don't pass off unread as verified.** In a review, the reason for a remark has to be a
   checkable fact: "field X isn't in the model" is checkable, "the artefacts aren't consistent" is
   a claim about content that requires reading every section.
5. **The data is the truth; the code isn't the source.** Storage doesn't replace human-readable
   artefacts.
6. **Tests are mandatory and never touch real data** — synthetic fixtures only, and they don't
   reach the network.

## Key mistakes

**1. The agent "improved" the canon — and carried the defective version forward.**
A stable block in the meeting summaries had been word-for-word identical across three summaries
in a row. In the next one it got compressed into one-liners — and that compressed version
travelled onward as "the latest". The reaction was short: "made up the block again, did you?"
**Fix:** the canon is carried over verbatim, and it's defined by **recurrence, not recency**:
before pasting, you open 2–3 earlier artefacts and take the version that repeats. A version
rewritten once is treated as defective and isn't carried forward. The general conclusion: **a
mentor recognises their own phrasing** — one substitution wipes out trust in the whole artefact,
because from then on you have to check every single one.

**2. The work was done from the machine version of a document instead of the human-edited one.**
Material for a student was assembled from a structured export rather than from the final file,
which by then I had already edited by hand: the export listed six items, while the real document
had **one** left. The student would have received material that contradicted their own document.
**Fix:** the source of truth is the artefact a human opened and corrected; the machine version
only reflects a past state. Any work that depends on its contents starts by reading that file,
edits are carried back with a line-by-line comparison, and after manual edits the generator **is
not re-run** — otherwise it wipes out the very thing the work was for.

**3. Scope expanded "while we were at it".**
While preparing a meeting summary, the agent wrote a couple of facts from the transcript into a
neighbouring document. We allowed the edit to stand, but the reaction was: "and who asked you to
edit that file?"
**Fix:** a fact appearing in the source is **not an instruction to apply it**. Anything adjacent
gets written out as a line — "happy to add this" — and only goes in after an explicit yes. The
reason runs deeper than tidiness: a human finalises these artefacts themselves and considers them
theirs, and an edit made without their decision surfaces later — by which point it's too late to
trust the document.

**4. A claim about content, made without reading the content.**
The same class as principle #4, but the price here is higher: a student receives a review as
fact. "The artefacts aren't consistent with each other" sounds like a conclusion, but it requires
reading every section — and if they weren't opened, it's speculation with a judgement inside it.
**Fix:** a review explicitly separates "I checked" from "I assume". If a check wasn't done, that's
what gets written, without phrasing that sounds like a verdict.

**5. Domain traps that fail silently.**
Three different mechanisms with one thing in common: they don't crash, they quietly produce a
wrong result.

- **Names that look identical aren't equal.** macOS creates directories in one Unicode form,
  while the same name arrives from a command argument in another: "ё" is either a single
  character or a letter plus a diacritic. Glob matching, string comparison and database queries
  **silently miss**. Closed off with normalisation at every boundary — including one-off scripts,
  not just the project's code.
- **A fractional mark in an integer column.** 4.5 is truncated to 4 without an error, and an empty
  mark isn't a zero. The type and the semantics of emptiness are specified separately.
- **Formulas in the plan.** Rows with averaging are edited surgically only, and the file has to be
  closed first, otherwise the edit is lost.

## What's built and what's next

**The core is done** — storage, CLI and importers are accepted: the gate's **50 checks are
green**, with **108** tests. The import has been run in on real data: 3 personal plans (222 topics
and 462 questions) and 57 artefacts. Next come the pipelines on top of the core, in priority
order:

- **The meeting pipeline** — drop a video into a folder and get back a transcript, a registered
  meeting and a draft summary **with the canonical block already inserted**. Carrying that block
  over by hand has already produced an error twice — inserting it in code removes a whole class
  of them
- **The student plan** — a diff of "what's proposed to be marked" → confirmation → applied to the
  xlsx with one command, with no risk of wiping the formulas
- **Development plans driven by market data** — take the "position × specialisation" cell from
  the neighbouring vacancy-analytics project, compare it against the student's profile and get
  back a delta and a list of topics. 🔴 The fork "how to fetch the data" is on hold by my
  decision: file export, a read-only API, or a manual extract
- **A checklist against the student's spec** — a structural inventory of the sections, so the
  review can be written on substance instead of being spent on "what's even described here"
- **A bank of test assignments** — the assignment, the reference solution, the student's work, the
  breakdown

Deliberately deferred: student statuses and deadlines (nothing is falling through the cracks for
now, so we keep it as is) and containerisation (we'll come back to it when there's an API or a
move to a server).

## Scale

- **46 commits**, June–August 2026
- **108 tests** and **an executable gate of 50 checks** — both green, run on every delivery
- The roadmap is kept as epics, with the core closed; every epic states explicitly what the code
  does and what stays with the architect
- Storage: students, meetings, plan topics and their status, artefacts
- Stack: Python 3.13 (uv) · SQLite · typer · pydantic · pytest · a local transcription engine on
  the laptop's GPU · openpyxl
