# TASK: Source inventory — verify every candidate source live

**Status (2026-10-30): paused, waiting on the user for API keys (OpenAQ, INMET).** Research only: no application code, no scaffold, no
infrastructure. The results go into `docs/research/README.md`; questions it raises for the user
go into `docs/architecture.md` §10.

## 1. Current scenario

- `docs/research/README.md` has six candidate rows (NASA FIRMS, INPE BDQueimadas/TerraBrasilis,
  INPE DETER, OpenAQ, Open-Meteo, smoke/aerosol products). **Every cell is `[VERIFY]`.** The
  candidates come from the user's saved ideas list (2026-10-12), written from memory.
- `docs/research/samples/` is empty, so no parser can be tested and no number in
  `docs/architecture.md` is backed by a measurement.
- Blocked on this:
  - the canonical events (architecture §4): which fields exist, their units and time zones;
  - fire identity and the FIRMS ↔ INPE duplicate collapse: whether the two catalogues share
    satellites and how their coordinates and times differ for the same pixel;
  - the join (§5): how dense the air-quality stations are near fire clusters and how often they
    report; whether wind is observed or modelled;
  - the headline idea itself: whether any source lets us show "where the smoke went" without
    computing a plume ourselves (CLAUDE.md invariant 2);
  - volumes (§7, §9): detections per day, requests per hour, bytes per day.

## 2. Planned changes

### 2.1 Research process (per source)

1. Find the official source first (the agency's own site and API docs), then aggregators.
2. Record: exact URL; terms of use and attribution; auth; format and the fields that map to a
   canonical event; `ETag` / `Last-Modified` / `Cache-Control`; whether a conditional request
   returns `304`; any published rate limit; whether it is **observed** or **modelled**.
3. **If a source needs a key or registration: stop and tell the user exactly what to do.**
   No sign-ups on the user's behalf.
4. **Measure, don't trust:**
   - detections per day on a quiet day and on a peak-season day (from published history, not
     a long live poll);
   - latency from satellite overpass (`acq_date`/`acq_time` or equivalent) to availability
     (when the record first appears in the feed). Where only a single snapshot is possible
     here, record the age distribution of the newest records in that snapshot; a proper
     measurement over days goes into a task for the AWS box (CLAUDE.md: no long live jobs in
     the planning session);
   - duplicates between FIRMS and INPE for the same satellite and day;
   - air-quality station count in Brazil, how many reported in the last 24 h, and how many are
     within 50 / 100 / 200 km of the day's fire clusters;
   - bytes per request and per day, raw and gzip.
   - Request scripts live in the session scratchpad, **not** the repo. Any repeated polling is
     no faster than the source updates and capped at a few requests per source.
   - User-Agent: `fire-pulse-research/0.1 (one-off source inventory; contact: repo owner)`.
5. Samples are named `<source>-<YYYYMMDD>-<what>.<ext>` and kept small (trimmed to a few
   hundred rows or one region). Large captures stay local; their URL and capture time are
   recorded.
6. Anything that can't be confirmed stays `[VERIFY: what, where]`, with what was tried.

### 2.2 Candidates

| Source | What to establish |
|---|---|
| NASA FIRMS | satellites and products (MODIS, VIIRS S-NPP / NOAA-20 / NOAA-21, NRT vs URT vs SP); the area and country APIs; update latency; rate limits; the free `MAP_KEY` |
| INPE BDQueimadas / TerraBrasilis | what is actually machine-readable (WFS? CSV? dashboard only?); formats; latency; licence; the "satélite de referência" |
| INPE DETER | what it is and whether it belongs in the fire scope at all |
| OpenAQ | current API version; key required?; Brazilian station coverage; how often stations report |
| Open-Meteo | wind: forecast vs observed; rate limits; licence terms for non-commercial public use |
| Smoke / air quality, other | INPE/CPTEC smoke or air-quality products; CAMS; any other published product better for smoke; flagged **model** vs **observation** |

### 2.3 Documentation

- `docs/research/README.md`: the inventory table (a cell is "yes" only where a sample backs
  it), one section per source, volume numbers, and **"Findings that contradict or change the
  plan" at the top**.
- `docs/architecture.md`: only in Part C (this task writes facts, not design).
- `README.md`: status line.

### 2.4 Alternatives rejected

- **Filling cells from aggregator pages or memory.** Listings go stale; the provider's own
  response is the source of truth.
- **Running a multi-day latency poll from the Mac now.** Rejected: the planning session doesn't
  run long live jobs. A single snapshot gives an age distribution; the real
  overpass-to-availability measurement is written as a task for the AWS box.
- **Registering for keys to "save time".** Rejected by the prompt: the user registers.

## 3. Why

- Unblocks the canonical events, the fire identity, the join design, the phase plan and the
  scaffold task: adapters can't be written against fields nobody has seen.
- Tells us early whether the headline ("see the smoke reach the city") is possible with observed
  data, or only with a labelled model.
- **Cost:** a few hundred requests across all sources. No money.

## 4. Affected files

| File | Change type | Notes |
|---|---|---|
| `docs/tasks/TASK-source-inventory.md` | new | this document |
| `docs/research/README.md` | rewrite | the verified inventory |
| `docs/research/samples/*` | new | small real captures |
| `README.md` | edit | status line |
