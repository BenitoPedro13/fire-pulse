# TASK: Set up the repo (planning only)

**Status (2026-10-30): done.** Written before any other file in the repo.

## 1. Current scenario

`~/Documents/personal/fire-pulse` doesn't exist yet. The project exists only as a planning
prompt (2026-10-30) and an entry in the user's saved ideas list (2026-10-12). Those settled:

- **What it is:** a live, public map of fires in Brazil. Satellite fire detections (INPE
  BDQueimadas, NASA FIRMS) are joined with wind and air-quality data, so a visitor can watch a
  fire get detected and then see what the air did in nearby cities in the hours after.
- **Why:** the next portfolio case study after Renewable Pulse, Apuração 2026 and Transit
  Pulse, still about distributed, data-heavy, real-time systems. The new thing it adds is
  **spatial and temporal joins across several streams**.
- **Decisions the user has made** (they go into `CLAUDE.md` §0 and aren't reopened):
  - real data only;
  - a Transit Pulse-style monorepo (`apps/` + `packages/`, one Go module, a standalone pnpm web
    app; Go or TypeScript, never Python);
  - provider adapters → one canonical event, an immutable raw log, idempotent ingestion,
    rebuildable projections;
  - cheap AWS hosting copied from Renewable Pulse;
  - a short live run that ends in a static replay;
  - the frontend follows the global `~/.claude/CLAUDE.md`;
  - docs in English, no `Co-Authored-By` trailer.
- **The name:** `fire-pulse` matches `renewable-pulse` and `transit-pulse`. The user confirmed
  it on 2026-10-30.

Nothing has been verified against a real source. Every source claim in the ideas list (INPE
BDQueimadas, NASA FIRMS, INPE DETER, OpenAQ, Open-Meteo) came from memory.

The sibling repos share one workflow: a `CLAUDE.md` with §0–§4 and a TL;DR, task documents in
`docs/tasks/`, and verified facts with captured samples in `docs/research/`. Transit Pulse is
the template (`transit-pulse/CLAUDE.md`, `transit-pulse/docs/tasks/TASK-repo-setup.md`).
Apuração is the precedent for a polled feed → an immutable raw log → a replay.

## 2. Planned changes

This task covers the repo, git, `CLAUDE.md` and `docs/` only. It adds no application code, no
scaffold, no infrastructure and no package-manager files.

1. **`git init`** with default branch `main`. No remote; the user decides that separately.
2. **`docs/tasks/TASK-repo-setup.md`**: this document, written first.
3. **`CLAUDE.md`** with Transit Pulse's structure: §0 project context (decisions, a stack table
   with a Status column, how to write, invariants, tests), §1 plan before executing, §2 use
   CLIs (with §2.0), §3 update documentation, §4 project conventions (with §4.1 commits), and a
   TL;DR. **Every stack row is marked planned.** The language split between Go and TypeScript,
   the broker and the store are filled in by Part C from current docs, not from memory.
4. **`README.md`**: what the project is, "Status: planning, no code yet", and links to the
   docs.
5. **`docs/architecture.md`**: a skeleton with the decided points, empty sections for the
   canonical event, the join, the system shape, hosting, the replay ending and retention, and an
   **Open questions** table.
6. **`docs/research/README.md`**: the source inventory as a table, one row per candidate
   source, every cell `[VERIFY]`. Filling it is `TASK-source-inventory.md`.
7. **`docs/research/samples/.gitkeep`**: an empty folder for captured samples.
8. **`.gitignore`**: OS and editor files, `.env*`, and large files under
   `docs/research/samples/`.
9. **One commit**, with a plain message and no `Co-Authored-By` trailer.

Alternatives rejected:

- **Pre-filling the inventory from memory** (for example the FIRMS URL pattern, or OpenAQ's API
  version). A confident wrong URL is worse than a `[VERIFY]`, and the inventory task runs next
  in the same session anyway.
- **Another name** (`queimadas-pulse`, `smoke-pulse`). `fire-pulse` keeps the sibling pattern
  and reads in English, the docs' language. The user confirmed it.
- **Copying Transit Pulse's stack table as decided.** The sources are different (files and
  REST, not a minute-by-minute AVL feed), and the volume is far lower, so the broker and store
  rows have to be argued from this project's measurements, in Part C.

## 3. Why

- **It unblocks `TASK-source-inventory.md`,** which needs a place for samples and an inventory
  table to fill.
- **It puts the decisions in writing** before any code exists, so the implementing session
  builds against a contract, not against this conversation.
- **Cost:** a few documents. No money, no infrastructure.

## 4. Affected files

| File | Change type | Notes |
|---|---|---|
| `docs/tasks/TASK-repo-setup.md` | new | this document |
| `CLAUDE.md` | new | workflow guidelines, Transit Pulse structure, every stack row planned |
| `README.md` | new | short; status "planning, no code yet" |
| `docs/architecture.md` | new | skeleton plus open questions |
| `docs/research/README.md` | new | source inventory, every cell `[VERIFY]` |
| `docs/research/samples/.gitkeep` | new | empty samples folder |
| `.gitignore` | new | OS/editor files, `.env*`, large samples |
