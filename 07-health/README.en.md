# health — personal health monitoring

🇷🇺 [Русская версия](README.md)

> **A note before you read.** Russian is my native language, so this translation exists to make
> the material accessible to everyone. The full breadth of what I mean comes across best in the
> [Russian version](README.md).

**57 commits · July–August 2026 · architects Kimi K3 and Qwen 3.8 Max**
The functionality works; ~30% of the plan is delivered.

## ⚠️ Two invariants that matter more than any feature

This project is about medical data, so two constraints here are stricter than any technical one:

**1. Nothing without a doctor.** The service **does not diagnose, does not prescribe or cancel
treatment, and does not set dosages**. Its role is memory, order in the data, trends, lifestyle
prompts and a list of questions for the doctor. Any analytical output is marked "this is not
medical advice" and comes with an explicit "discuss this with your doctor".

**2. Medical data doesn't leave the perimeter without an explicit decision.** Raw data never
reaches git. Medical data is **not sent** to external LLM APIs by default — that's still an open
question. **Tests use synthetic data only**; real exports are never baked into fixtures.

The second point cost the project a feature: a text LLM summary of the health picture would be
the single most useful thing here, but it's on hold until the question of where the data would
travel is settled.

## How I work with the agent

The project is small and moves in bursts: I've been to the doctor — we go through it. I've pulled
an export from somewhere — we look at it and go through that.

Kimi K3 and Qwen 3.8 Max lead here, with Claude as backup. The process doesn't change because of
that: I look at the epic plan to see what's left, give the command, and the architect decides
whether to do it itself or write a work order for the coder. Two of the four closed epics were
done entirely by the coder: document intake and the fitness band import.

One thing sets this project apart from all the others: **the coder never sees real data at all**.
It's an external model, so neither exports nor the contents of reports go into a work order —
only structure and format, with synthetic fixtures. The first run against real data is always
done by the architect, and that isn't up for discussion.

A report after each task, **no merges happen without me**, and a full test run before the merge.

**The rule about comments is the strictest of all my projects here: without the tag, a rake
isn't delivered.** If the agent steps on a rake, a comment with the `AI:` prefix is mandatory,
and that gets checked at acceptance rather than left to its conscience. Plus a docstring "why",
inline on anything non-obvious, and the task tag (`# Э1.2`). The project moves in bursts — went
to the doctor, we go through it — and between those bursts everything not written down in the
code itself is lost.

## The goal

I wanted to build myself a virtual GP that would track my metrics and answer my questions
**without prescribing anything**. This is purely informational; everything else belongs with a
doctor.
Health data lives in three unconnected places: doctors' reports and lab results are on paper and
in PDFs, daily metrics are in Apple Health, sleep and activity are in the fitness band's app.
Taken separately each source is almost useless: you can't see the trend, and at the appointment
you remember the wrong things in the wrong way.

The system brings them together into **a single picture**: it puts the data in order, shows the
trend, tracks progress towards goals and helps prepare for an appointment — including by
producing a list of **questions to ask the doctor**.

## What it does and how it works

```
Apple Health export.xml ─┐
fitness band export     ─┼──► ingest/ ──► normalisation ──► DB ──► services/ ──► API
reports and lab results ─┘   importers    one metric            analytics     JSON
     (PDF, photo)                          dictionary                         + Swagger
```

The three sources are parsed by different importers but mapped onto **a single normalised metric
dictionary** — otherwise a heart rate from Apple and a heart rate from the band can't be compared
and the trend falls apart.

**The import is idempotent.** Loading the same file again doesn't produce duplicates, and that
isn't a convenience, it's a necessity: Apple has a habit of recording the same entry from several
devices.

**Doctors' reports** go through a PDF pipeline: the text layer is taken directly, and if a page
turns out to be a scan, OCR kicks in. A recognition error on one page doesn't take down the whole
document — it's flagged right there in the text.

### Interface

**There's no frontend yet, but one is planned — for metric trends.** For now the only interface
is an HTTP JSON API, with routes grouped by entity: measurements, documents, the health picture,
and a liveness check.

**Swagger here works not as documentation but as a temporary UI.** There are no developers on
this project who'd need the spec — but there is me, and I need to enter a measurement by hand,
open the picture for a period and check that a document was recognised. FastAPI gives you the
interactive page for free, and it's enough right up until trend charts are needed: that's when a
frontend will appear.

The order is deliberate: data and analytics first, the interface only when you genuinely can't
manage without it. I'm on my own here, and every hour not spent on markup goes into what the
project actually exists for.

## Architecture

**Four layers with strict boundaries:**

| Layer | Responsibility |
|---|---|
| `ingest/` | parsing external formats into normalised measurements. Each source has its own module |
| `db/` | SQLAlchemy 2.0 models, engine, sessions. The schema is managed through Alembic |
| `services/` | business logic on top of the DB: deterministic assembly of the "health picture" |
| `api/` | a thin FastAPI layer: routes and schemas, **with no business logic** |

Three decisions worth explaining:

**Streaming parsing instead of loading into memory.** An Apple Health export is hundreds of
megabytes of XML. A naive parse kills the process, so the file is read as a stream, element by
element.

**The database is SQLite, one file.** Over 1M measurements and 18 indexes as of 14.08.2026 — for
a personal system on a single machine that's plenty. Access goes through SQLAlchemy, and the
schema is managed by Alembic.

Tests run on in-memory SQLite: each on a fresh database, a run takes milliseconds and needs
neither a running server nor any cleanup afterwards. The choice was made precisely for the cost
of a run — a full run before a merge is mandatory, and it will only stay mandatory while it stays
cheap.

**The import is decoupled from the web layer.** Importers run as a CLI and don't depend on the
API — meaning you can load data even when the application isn't up.

### Reuse across projects

The PDF pipeline here **wasn't written from scratch** — it was carried over from my investment
project reports, where it's already been proven on hundreds of financial statements: taking the
text layer, OCR as a fallback for scans, fixing mangled filenames, and a single text file with
page markers.

The decision was recorded in a dedicated document, with the **boundary of reuse** written down
explicitly: what was taken as is, and what had to be adapted to the medical specifics. That's a
direct consequence of the rule "reusing module X → first open X and write down what it
hard-codes": without that boundary, someone else's module brings someone else's assumptions with
it.

That portability is exactly what a single working method is for: a pipeline written for financial
statements closed a task in a health project in one pass.

### Data model

| Entity | What it holds |
|---|---|
| **Measurement** | one measurement point: metric, value, unit, time, source |
| **MedicalDocument** | a report or lab result: the file + extracted text + structured findings |
| **HealthSummary** | a snapshot of the overall picture for a period |
| **SleepSession** | sleep with phases |
| **Workout** | a workout |
| **ImportRun** | the import log: what was loaded, when, and how much |

Two more are planned: **Goal** — the owner's goal with a category (ongoing, like weight control;
recurring, like losing weight) — and **Checkin**, a weekly note on how you're feeling. For goals
like posture there's no instrument to measure progress with, so the data source is honestly
labelled "as reported by the owner" — and that's its own field.

## Project structure

```
src/health/
  api/            FastAPI routes and pydantic schemas, no business logic
    routes/       one router per entity
  db/             SQLAlchemy models, engine, sessions
  ingest/         importers:
    apple_health.py   streaming parse of export.xml
    band.py           fitness band export: metrics, sleep with phases, workouts
    documents.py      reports and lab results
    pdf_pipeline.py   the PDF → text pipeline (carried over from reports)
    metrics.py        the single metric dictionary
    import_log.py     import log, idempotency
  services/       business logic: assembling the health picture
  main.py         application factory

migrations/       Alembic
tests/unit/       fast and isolated, no network and no DB file
tests/e2e/        the real application over HTTP, a fresh in-memory DB per test
docs/             product vision, epic plan, architecture, ADRs, practices, work orders
data/             ⛔ raw personal data — never reaches git
```

Everything runs through `make`: install, tests, the local API, the full stack in Docker.

## Roles

- **PO (me)** — product, acceptance, every decision about what may and may not be done with
  medical data.
- **Architects** — **Kimi K3 and Qwen 3.8 Max** lead on this project, plus Claude Opus. Equal and
  interchangeable, one process for all of them.
- **Coder** — DeepSeek v4 Pro in an autonomous loop. Two of the four closed epics were done by it
  entirely: document intake and the fitness band import.

Separately: **the coder doesn't see real data.** It's an external model, so neither exports nor
the contents of reports go into a work order — only structure and format, with synthetic
fixtures. The first run against real data is always done by the architect.

## Principles

1. **Nothing without a doctor.** The service is memory and analytics, not an adviser on
   treatment.
2. **Medical data doesn't leave the perimeter** without a separate, explicit decision.
3. **The import is idempotent** — loading the same file again doesn't produce duplicates.
4. **A single metric dictionary:** the same metric from different sources is mapped onto one
   canonical name, otherwise trends can't be compared.
5. **The source of truth for a document is the file itself**; the database holds the extracted
   text, the structured findings and a link to the file.
6. **Tests on synthetic data**, and no change is handed in without a green run.

## Key mistakes

This project is younger than the others and started **on rules that already existed** — the whole
body of practices was carried over from reports, together with the reasoning behind it. So there
are fewer of the classic traps here: they'd already been paid for on another project. That's
precisely the value of a portable method.

What did come up anyway:

**1. "Just reuse the module" is the most expensive phrase you can put in a work order.**
Carrying the PDF pipeline over looked like copying a file. In reality someone else's module drags
its own assumptions along: paths, operating modes, expectations about the structure of a
document.
**Fix:** the rule "reusing X → first open X and write down the **boundary of reuse**", with the
decision itself recorded as an ADR that explicitly separates "taken as is" from "adapted".
Without that, reuse saves you a day and costs you three.

**2. A naive parse of the Apple export didn't survive a real file.**
The export from Health is hundreds of megabytes of XML. On a test fragment everything worked
beautifully.
**Fix:** streaming, element-by-element parsing. The lesson is general and boring: **the volume of
production data is a requirement too**, and you check against it before delivery, not after.

**3. Duplicate measurements that "shouldn't exist".**
Apple records the same thing from several devices — the phone and the watch. A naive import
honestly entered both records, and the trend broke.
**Fix:** idempotent import as an invariant rather than a feature: loading the same file again is
required to leave the database state unchanged. It's covered by a test.

## What's built and what's next

Four epics are closed: the skeleton (API, DB, Apple Health import, the overall picture), the
coder scaffolding, document intake with the PDF pipeline, and the fitness band import.

Ahead:

- **Structuring the medical data** — parsing lab results into numeric findings with units and
  reference ranges, where the reference is taken **from the lab result itself** and our own
  reference table is only a fallback. An "out of range" flag, metric trends by date, a timeline of
  appointments and prescriptions
- **Analytics and summaries** — trends, anomalies, linking "metric ↔ report" (weight before and
  after a prescription, for example), goals and progress towards them, a check-in journal, and a
  "food: allowed / not allowed / not advisable" mode
- **Preparing for an appointment** — a report for the visit, a PDF export for the doctor,
  reminders
- **A frontend for metric trends** — charts and trends; until then Swagger is enough

## Scale

- **57 commits**, July–August 2026
- **69 automated tests**: unit tests with no network and no DB, plus e2e over HTTP on a fresh
  database
- 7 epics, 4 closed; two of them done by the coder entirely
- Stack: Python 3.12 (uv) · FastAPI · SQLAlchemy 2.0 + Alembic · SQLite ·
  PyMuPDF + tesseract OCR · Docker · pytest
