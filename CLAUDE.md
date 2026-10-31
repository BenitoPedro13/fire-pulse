# Workflow Guidelines — Fire Pulse

> Ported from the `transit-pulse` workflow (itself from `apuracao-2026` / `renewable-pulse` /
> `flora`): plan before you touch anything, lean on official CLIs and generators, treat
> documentation as part of the deliverable. Section 0 is project-specific; sections 1–4 are
> the portable rules.
>
> **The philosophy in one line:** plan before you write, lean on existing tooling while you
> work, and treat documentation as part of the deliverable when you finish.

---

## 0. Project context — Fire Pulse

A live, public map of **fires in Brazil**, built from real public data. Satellite fire
detections are joined in space and time with wind and air-quality data, so a visitor can watch
a fire get detected and then see what the air did in nearby cities in the hours after.

It's a portfolio case study in **distributed, data-heavy real-time systems**, after
`renewable-pulse` (polled energy feeds → broker → TimescaleDB), `apuracao-2026` (polled TSE
feed → immutable S3 log → replay) and `transit-pulse` (one adapter per operator → canonical
event → map + time machine). What this one adds is **spatial and temporal joins across several
streams** that arrive at different rates, with different latencies, and out of order.

- every source gets its own adapter that emits one canonical event;
- the raw log of provider responses is immutable;
- ingestion is idempotent, and projections (including every join) can be rebuilt;
- the map always has a table alternative.

**Status (2026-10-30): planning, no code.** Read in this order:

1. `docs/research/README.md`: the verified source inventory. Its "Findings that contradict or
   change the plan" section comes first.
2. `docs/architecture.md`: scope, coverage tiers, the canonical events, the join, the system
   shape, hosting and cost, the replay-first ending, retention and open questions.
3. `docs/research/ux-direction.md`: what a visitor can do and see.
4. `docs/build-guide.md`: the concepts and the milestone order. **The user writes the code
   themselves (2026-10-30)**; Claude guides, explains and reviews, and writes code only when
   asked.

A sibling project (floods and rivers: ANA, CEMADEN) may come later. Lessons should carry over,
but nothing here is designed for it.

### Decisions already made (user, 2026-10-30). Don't reopen them

- **Real data only.** No simulated or synthetic data of any kind. The user rejected simulation
  in renewable-pulse and the same rule holds here.
- **Smoke is never fabricated.** "Where the smoke goes" is shown only as (a) observed wind and
  observed air quality, or (b) a published model product, clearly labelled as a model. A plume
  we compute ourselves and present as observed is out.
- **Provider adapters → one canonical event** per kind of thing (a detection, a weather
  observation, an air-quality measurement). Nothing after the adapter knows which provider an
  event came from.
- **An immutable raw log, idempotent ingestion, rebuildable projections.**
- **Monorepo like Transit Pulse:** `apps/` (deployables) + `packages/` (shared code), one Go
  module at the root for ingestion, one standalone pnpm project for the web app. pnpm, never
  npm. Backend in Go or TypeScript, **never Python**.
- **Hosting on AWS, as cheap as possible,** copying renewable-pulse / transit-pulse: one
  `t4g.small` Spot instance running Docker Compose. No NAT Gateway, ALB, managed Kafka or RDS.
  A Budgets alarm on day one. The Free-plan account is shared with the sibling projects and
  closes around 2027-03-27, or earlier if the credits run out.
- **A short live run, then a static replay** on S3 + CloudFront, so the case study works with
  the backend off.
- **The frontend follows the global `~/.claude/CLAUDE.md`, unchanged:** Server Components by
  default, TanStack Query, every hook abstracted, shadcn/Radix, WCAG 2.2 AA.
- **Docs in English. No `Co-Authored-By` trailer** on any commit.

### Stack

Every row is **planned**: no code exists. Version numbers are a snapshot, not a pin: check each
tool's current docs before installing or upgrading (§2.0).

| Layer | Choice | Status |
|---|---|---|
| Ingestion | One adapter per source → canonical event. Language `[decided in Part C]` | planned |
| Raw log | S3, compressed, kept forever | planned |
| Broker | `[decided in Part C]` | planned |
| Join / stream processing | `[decided in Part C]` | planned |
| Store | `[decided in Part C]` | planned |
| API | read-only, precomputed views | planned |
| Web | Next.js (App Router) + TanStack Query, pnpm; map library `[decided in Part C]` | planned |
| Replay | static export of chosen windows on S3 + CloudFront | planned |
| Hosting | AWS, one `t4g.small` Spot, Docker Compose, Budgets alarm | planned |
| Frontend rules | the user's global `~/.claude/CLAUDE.md`, unchanged | applies |

### How to write in this repo

- **Never invent a provider field, URL, endpoint or behaviour.** That covers NASA FIRMS, INPE
  (BDQueimadas, TerraBrasilis, DETER, CPTEC), OpenAQ, Open-Meteo, CAMS and every other source.
  - Write `[VERIFY: what to check and where]` inline instead.
  - Resolve it against a real response captured into `docs/research/samples/` before any code
    depends on it.
- **Be specific to the point of discomfort:** exact URLs, exact poll intervals, exact numbers.
  - No acceptance criterion may rest on "works" or "fast enough".
  - State detections per day, requests per hour, the p95 latency and the bytes per day.
- **Sources over memory.** Anything about a provider, a library or a hosting platform is
  checked against the live source, and the source is linked.
- **Observed, modelled and missing are different words.** Never write "smoke" for an air-quality
  reading, or "observed" for a model field.

### Invariants — never break these without changing the spec first

1. **Real data only.**
   - Every detection, wind value and air-quality value traces back to a real provider response.
   - Nothing is synthetic, simulated or interpolated and then shown as observed.
   - A gap (an unrecorded period, a failed fetch) is shown as missing, never smoothed over.
2. **Observed ≠ modelled.**
   - Satellite detections and station measurements are **observed**. Forecast/reanalysis wind
     and any smoke or aerosol model are **modelled**. The two never share a colour, a label or
     a contract field without the distinction.
   - We never compute a plume, a trajectory or a dispersion and present it as observed.
3. **Raw responses are immutable.**
   - Every provider response we fetch is stored as received, compressed, in S3, and kept
     forever.
   - Everything else is a projection that can be rebuilt from that raw log: database rows,
     merged fire identities, joins, aggregates and the replay export.
4. **Ingestion is idempotent.**
   - Fetching or processing the same provider response twice changes nothing.
   - Every event has a deterministic identity (architecture §4).
5. **Be a polite client to every provider.**
   - Poll no faster than the source updates.
   - Use conditional requests where the provider supports them.
   - Keep a hard request budget per source (per minute and per day) and back off on 4xx/5xx.
   - Never proxy our readers straight through to a provider.
6. **Readers never touch the ingestion path.**
   - Viewer traffic is served from precomputed or cached views.
   - A traffic spike must not slow ingestion; an ingestion stall shows "last updated at …".
7. **Missing is not zero.**
   - "No fires detected", "fetch failed" and "no sensor coverage" are three different states,
     in contracts and in the UI.
   - "No satellite pass yet" is not "no fires".
8. **WCAG 2.2 AA.**
   - The map is never the only way to read a result; a table or list alternative is required.
   - Colour is never the only carrier of meaning (fire age, confidence, air-quality band,
     observed vs modelled).
   - Motion respects `prefers-reduced-motion`.

### Tests

- **Integration tests against real infrastructure** (testcontainers or equivalent), never a
  mocked database or broker.
- **Parsers are tested against the real captured samples** in `docs/research/samples/`, never
  hand-built fixtures.
- **Idempotency is a test:** ingest the same provider response twice and assert nothing changed.
- **Replay is a test:** rebuild a projection (including the join) from the raw log and assert it
  equals the live projection.
- **Late data is a test:** feed the same set of responses in two different arrival orders and
  assert the join's output is identical.
- **Performance claims need medians of repeated runs**, never a single measurement.

---

## 1. Plan before executing — write a task document first

**Rule:** Before editing or creating **any** code file, write a task document at
`docs/tasks/TASK-<slug>.md`. No exceptions for "small" changes.

Required sections, in order:

1. **Current scenario**: how it works today, and what's missing or blocked.
2. **Planned changes**: file by file, plus alternatives considered and rejected.
3. **Why**: what it unblocks and what it costs.
4. **Affected files**: a table of `File | Change type | Notes`.

Write the document, summarize it in 2–3 lines, and wait for alignment on anything significant
before writing code. Keep it in sync if the plan changes. The task document is the contract.

## 2. Use CLIs, generators, and SDKs — don't write everything by hand

### 2.0 Assume your framework knowledge is outdated — check first, every time

1. Go to the tool's own current docs first: the chosen framework, the map libraries, the
   broker, the database and its geo extension, Terraform, and each provider's API docs.
2. Use the official CLI or generator to scaffold, never a hand-written copy of what it
   produces.
3. Take the current stable major as authoritative over anything written here, and update this
   file's stack table to match (§3).
4. One package manager per language, decided at scaffold time, never mixed.

Hand-writing is for what no generator covers: the adapters, the canonical events, the
projections, the join, the replay export and the map layers. Match the style of the surrounding
code.

## 3. Update documentation after executing

A task isn't done until every doc it affects is updated:

- **`CLAUDE.md`**: if the stack, architecture or an invariant changes.
- **`docs/architecture.md`**: if a `[VERIFY]` resolves, an open question closes, or scope
  changes.
- **`docs/research/`**: if a new fact about a provider is observed. Update the inventory row and
  add the captured sample.
- **`.env.example`** (once code exists): every environment variable the code reads.
- **`README.md`**: the status line, setup and scripts.

When unsure, grep for the thing you changed across these files.

## 4. Project conventions

A single repo laid out as `apps/` (deployables) + `packages/` (shared code), with one Go module
at the root and one standalone pnpm project (`apps/web/`). `apps/` and `packages/` are
directories, not modules or workspaces. Today it holds only documentation:

```
fire-pulse/
├── CLAUDE.md
├── README.md
├── .gitignore
└── docs/
    ├── architecture.md
    ├── build-guide.md      # how it works + milestones, for building it by hand
    ├── research/           # source inventory + samples/ (real captured provider responses)
    └── tasks/              # task documents (§1)
```

Each Go binary arrives with its task as `apps/<name>/main.go`; shared Go code goes in
`packages/<name>/`. A root pnpm workspace is added only when a second JS package exists.

- **Samples:** keep small, representative captures in git (one response, a trimmed CSV). Large
  files stay out (see `.gitignore`); record their source URL and capture time instead.
- **Secrets:** API keys (the FIRMS `MAP_KEY`, an OpenAQ key) live in `.env` locally and in SSM
  Parameter Store on the box. Never in git, never in a sample, never in a doc.

### 4.1 Commit conventions

- **Commit automatically once a task document's work is complete and verified** (build, lint
  and tests passing, where they exist). Don't wait to be asked each time.
- **Never run destructive git commands without asking.** Force-push, `reset --hard`, branch
  deletion and history rewrites need explicit confirmation.
- **No `Co-Authored-By` trailer.** User preference for every repo.

---

## TL;DR

| Phase | Rule | Output |
|---|---|---|
| **Research** | Verify every source live, never from memory | `docs/research/` + real samples |
| **Before** | Write a task document first | `docs/tasks/TASK-<slug>.md`: current scenario, planned changes, why, affected files |
| **During** | Official CLIs and generators; `[VERIFY: …]` for anything unconfirmed | canonical, reproducible output |
| **After** | Update `CLAUDE.md` / `docs/architecture.md` / `docs/research/` / `README.md`, then commit | docs in sync, a commit with no `Co-Authored-By` |

**Never broken:** real data only, observed ≠ modelled (no fabricated smoke), raw responses
immutable and replayable, idempotent ingestion, polite to every provider, readers isolated from
ingestion, missing ≠ zero (no fires / fetch failed / no coverage), WCAG 2.2 AA.
