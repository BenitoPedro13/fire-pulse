# Build guide — Fire Pulse, written by you

> **Status (2026-10-30): the plan for building the project by hand.** You write the code; this
> document tells you what to build, in what order, why each piece exists, and how you'll know
> it works. Every provider fact here comes from `docs/research/README.md` (the inventory) and
> its captured samples. Anything not verified there is marked `[VERIFY]`, as in the rest of the
> repo.

## How to use this guide

- **Work through the milestones in order** (§5). Each one builds on the previous one, and each
  is small enough to finish and commit on its own.
- **Each milestone starts with a task document** (`docs/tasks/TASK-<slug>.md`, CLAUDE.md §1).
  That's not bureaucracy: writing "current scenario / planned changes / why / affected files"
  before coding is the fastest way to find out you don't understand something yet.
- Each milestone has four parts:
  - **Concept**: the idea you need to understand first.
  - **Build**: what to write, file by file, without the code.
  - **Done when**: checks you can run. If you can't run one, the milestone isn't done.
  - **Traps**: mistakes the research already found, so you don't rediscover them.
- **Ask me** to explain a concept, review a diff, point you at docs, or check your reasoning.
  I'll guide rather than write the code unless you ask me to.
- Read §1–§3 once before you start. They're the mental model; the milestones assume it.

---

## 1. The system in one picture

```
             PROVIDERS (we never control them)
 ┌──────────────┬───────────────┬────────────┬──────────────┬─────────────┐
 │ INPE 10-min  │ FIRMS 24h CSV │ INMET      │ OpenAQ v3    │ Open-Meteo  │
 │ (GOES-19 …)  │ (VIIRS/MODIS) │ (wind,obs) │ (PM2.5, obs) │ (wind, AQ:  │
 │              │               │            │              │  MODELLED)  │
 └──────┬───────┴───────┬───────┴─────┬──────┴──────┬───────┴──────┬──────┘
        │ conditional   │ HEAD, then  │             │              │
        │ GET (304)     │ GET if new  │             │              │
        ▼               ▼             ▼             ▼              ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ FETCHER  — one poller per source: schedule, request budget, backoff     │
 │            writes a row to fetch_log for EVERY attempt (ok/304/failed)  │
 └──────────────────────────────┬──────────────────────────────────────────┘
                                │ exact bytes as received
                                ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ RAW LOG (S3, gzip, write-once, kept forever) ← the only source of truth │
 └──────────────────────────────┬──────────────────────────────────────────┘
                                │ ADAPTER per source: parse → canonical events
                                ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STORE (projections — all rebuildable from the raw log)                  │
 │   detections (one row per pixel, deterministic id, ON CONFLICT no-op)   │
 │   weather_obs / air_quality (each tagged observed | modelled)           │
 │   source_status (no fires / fetch failed / no coverage)                 │
 │   city_timeline  ← THE JOIN (recomputed per city+window)                │
 └──────────────────────────────┬──────────────────────────────────────────┘
                                │ precomputed views only
                                ▼
 ┌──────────────────┐     ┌──────────────────────────────────────────────┐
 │ READ API (cached)│────▶│ WEB (Next.js): map + table, "last updated at"│
 └──────────────────┘     └──────────────────────────────────────────────┘
                                │ at the end of the live run
                                ▼
                      STATIC REPLAY on S3 + CloudFront
```

### One detection, end to end

Follow a single fire pixel through the system. Every milestone builds one hop of this trip.

1. **17:45 UTC** — GOES-19 sees a hot pixel in Pará. (Event time.)
2. **~18:05 UTC** — INPE puts it in `focos_10min_20261030_1800.csv` (or a later slot: GOES-19's
   median latency is 20 min, but its p90 is 680 min; inventory §2.1).
3. **18:06 UTC** — our INPE poller asks for that slot with `If-None-Match`. It gets `200` and
   the bytes. (Fetch time.)
4. Those bytes go into the raw log *exactly as received*, under a key derived from their hash.
   A `fetch_log` row records "INPE, slot 1800, 200, N bytes, hash".
5. The INPE adapter parses the CSV row `-7.291600, -38.659710, GOES-19, 2026-10-30 17:45:00`
   into a canonical **Detection** with a deterministic ID.
6. Insert into `detections`. If the ID already exists (we parsed the same file twice), nothing
   changes. That's idempotency.
7. The join notices a new detection near a city with a station and recomputes that city's
   timeline for the affected hours.
8. The read API's precomputed view for "active detections" and "city X" refreshes; the web app
   picks it up on its next query.
9. Two hours later FIRMS publishes the **same pixel** from a polar satellite pass (if one saw it).
   For GOES there's no FIRMS twin; for VIIRS there is, and it collapses onto the same ID
   (finding 1) and adds `confidence` and `frp`.

---

## 2. The concepts that hold it together

Learn these before coding. Each one exists because of a specific finding in the inventory.

### 2.1 Three clocks

Every record has (at least) three times, and mixing them up is the most common bug in systems
like this:

| Clock | Meaning | Example source |
|---|---|---|
| **Event time** | when the satellite saw it / the station measured it | INPE `data`, FIRMS `acq_date`+`acq_time` (both UTC) |
| **Publish time** | when the provider made it available | INPE slot in the file name, FIRMS `Last-Modified` |
| **Fetch time** | when *we* got it | our clock, recorded in `fetch_log` |

- **The map and the join use event time.** "What fires were burning at 14:00" is an event-time
  question.
- **"Last updated at …" uses fetch time.** It tells the visitor how fresh *our* copy is.
- **Latency = publish − event**, and it's what makes data arrive out of order (§2.5).
- **Store everything in UTC.** INPE's directory listings show America/Sao_Paulo times while the
  file names are UTC (inventory §2). Convert only at display time.

### 2.2 The raw log and replay

The raw log is a write-once archive of every provider response, byte for byte. Everything else
(database rows, the join, the replay export) is a **projection**: a function of the raw log.

Why it matters:

- **You can fix a parser bug after the fact.** Change the adapter, wipe the database, replay the
  raw log, and history is corrected. Without the raw log, a bug in week 1 is permanent.
- **The case study survives the backend.** The static replay is just another projection.
- **Tests become honest.** "Replay equals live" is a real test (CLAUDE.md, Tests).

Rules:

- Store what the provider sent, not what you parsed. Don't trim, reformat or re-encode before
  storing. Compress (gzip) the bytes as a whole.
- Key objects by content, e.g. `raw/inpe-10min/2026/10/30/<sha256>.csv.gz`, with fetch
  metadata (URL, status, headers, fetch time) in a sidecar or in `fetch_log`. Same bytes → same
  key → storing twice is a no-op.
- Never overwrite or delete. Write-once is what makes it trustworthy.

### 2.3 Identity and idempotency

Idempotent ingestion means processing the same response twice changes nothing. The mechanism
is a **deterministic identity** per event plus an insert that ignores duplicates.

For detections, the inventory found the identity already (finding 1):

```
detection_id = hash(canonical_satellite, event_time truncated to the minute,
                    round(lat, 3), round(lon, 3))
```

- **Why not INPE's UUID?** INPE duplicates itself: 94 rows on 2026-10-29 repeat the same
  satellite, minute and pixel with a different UUID (inventory §2.2). INPE's UUID is still
  worth keeping as an attribute: it's how events join back to detections (§2.3 of the inventory).
- **Why "canonical satellite"?** The two catalogues name satellites differently. FIRMS writes
  `N`, `N20`, `N21`, `T`, `A`; INPE writes `NPP-375`, `NPP-375D`, `NOAA-20`, `NOAA-21`,
  `TERRA_M-M`, `TERRA_M-T`, `AQUA_M-T`, … (inventory, Method notes). The adapter maps both to one
  vocabulary *before* computing the ID, or nothing matches. This mapping is a small, important,
  well-tested table.
- **Rounding has a boundary problem.** Two coordinates 0.00001° apart can round to different
  3-decimal values if they straddle `…5`. The research measured 99.8–100% matches anyway, so it's
  rare, but write a test with a straddling pair and decide consciously what happens (accept the
  split, or snap to a grid cell and also check neighbours). Write the decision in the task doc.

### 2.4 Adapters and canonical events

An adapter is the only code that knows a provider exists. It turns provider bytes into one of
a few canonical event types, and nothing downstream ever branches on the provider.

| Canonical event | Fields you'll need (first pass) | From |
|---|---|---|
| `Detection` | id, satellite (canonical), event_time, lat, lon, frp?, confidence?, provider_refs | INPE 10-min, INPE daily, FIRMS |
| `WeatherSample` | station_or_grid_id, kind (`observed`/`modelled`), time, lat, lon, wind_speed, wind_from_deg | INMET (observed), Open-Meteo (modelled) |
| `AirQualitySample` | location_id, kind, time, lat, lon, parameter, value, unit | OpenAQ (observed), Open-Meteo CAMS (modelled) |

- `?` means optional: INPE's 10-minute files have only `lat,lon,satelite,data` (inventory §2.1),
  so FRP and confidence arrive later, from FIRMS or INPE's daily file. The canonical event must
  allow "not known yet" (null), never a default of 0.
- **`kind` is a field, not a convention.** Invariant 2: observed and modelled never share a
  colour, label or contract field "without the distinction". Making it a required enum means the
  type system enforces it.
- Field names above are a starting point. Finalising them is architecture §4, and must be done
  against the samples, not this table.

### 2.5 Late and out-of-order data

GOES-19's p90 latency is 680 minutes (inventory §2.1): one point in ten arrives many hours
late. Polar passes arrive 1.5–3 h late. FIRMS and INPE deliver the same pixel at different times.
So the join **must give the same answer no matter what order the inputs arrive in.**

The simple way to guarantee that, given how small the volume is (finding 8):

- The join is a **pure function of the data in a window**: "for city C and hour H, given all
  detections and samples we have, compute the row". No running counters updated per event.
- When a late event lands, **recompute the affected windows** (the city/hour cells it touches).
  The volume is small enough that recomputing a cell from the database is cheap.
- Mark recent windows **provisional** (for example, until 12 h after the window ends, chosen
  from the p90 above). The UI says "may still change".

This avoids watermarks, state stores and stream processors entirely. It's a deliberate choice
(§3, decision C3), and the "late data" test (CLAUDE.md, Tests) proves it.

### 2.6 Missing is not zero

Three different states, which must stay different all the way to the UI (invariant 7):

| State | How you know | Shown as |
|---|---|---|
| **No fires detected** | fetch succeeded, file parsed, zero rows for the area/time | "0 detections" |
| **Fetch failed** | `fetch_log` has a failure (timeout, 5xx) for that slot | "data missing for 14:00–14:10" |
| **No coverage** | no satellite pass / no station within range | "no satellite pass yet" / "no station nearby" |

The only way to tell the first two apart later is the `fetch_log`. Record *every* attempt,
including `304`s and failures. A gap in `fetch_log` is itself a fact ("we were down").

### 2.7 Being a polite client

| Source | Pattern | Budget (planned) |
|---|---|---|
| INPE 10-min | one `GET` per new slot with `If-None-Match`; `304` is cheap | ~144 slots/day + catch-up + re-checks |
| FIRMS 24h files (×4) | `HEAD`, compare `ETag`, `GET` only if changed (FIRMS ignores conditional GETs, finding 9) | 4 HEAD/h = 96/day + ≤ 96 GET/day |
| OpenAQ | `X-API-Key`; ≤ 60/min, ≤ 2,000/h, honour `x-ratelimit-*` | `[VERIFY: once station count is known]` |
| Open-Meteo | several coordinates per call; ≤ 600/min, 5,000/h, 10,000/day | `[VERIFY: how multi-coordinate calls are counted]` |
| INMET | `[VERIFY: token limits]` | — |

Every poller has: a fixed interval no shorter than the source's update interval, a hard per-day
budget it refuses to exceed, exponential backoff on 4xx/5xx, and a descriptive `User-Agent`.

---

## 3. Decisions still open (Part C) — my recommendations

These were going to be Part C of the previous session. Now they're yours. For each: the
question, my recommendation, and the reasoning. When you decide, record it in
`docs/architecture.md` §10 and the stack table in `CLAUDE.md`.

| # | Question | Recommendation | Why |
|---|---|---|---|
| C1 | Ingestion language | **Go** | The repo layout is already "one Go module at the root". `net/http`, `encoding/csv` and `strconv` are in the standard library and locale-independent (the inventory's `awk`/`pt_BR` trap). One static binary per app runs well on a small ARM box. |
| C2 | Broker | **None** for v1 | Peak is ~51,000 detections/day ≈ 0.6/s (finding 8). A broker adds a service to run, monitor and pay memory for on a `t4g.small` (2 GB). The raw log already gives durability and replay. If you want the case study to *argue* this, measure it and write it up rather than adding Redpanda by reflex. |
| C3 | Join / stream processing | **Recompute-per-window in the store** (§2.5) | Order-independent by construction, easy to test, no framework to learn. |
| C4 | Store | **PostgreSQL + PostGIS** | Spatial queries ("detections within 100 km of this city") are one indexed `ST_DWithin` on `geography`. Time-bucketing works with plain `date_trunc`; add TimescaleDB only if a measured query needs it. |
| C5 | Join anchor | **Cities with a station**, not fires | Only 13% of detections are within 50 km of an INMET station (finding 7). A fire-centred join would mostly show "no station". |
| C6 | Smoke framing | "A fire was detected here; this downwind station measured this", with CAMS as a separate **model** layer | Finding 5–6: no observed plume exists. This is the only honest framing (invariant 2). |
| C7 | INPE fire events | **Show INPE's events, labelled "INPE, provisional"**; don't build your own clustering in v1 | Clustering is a project of its own. INPE's events join to detections by UUID. Your own clustering can be a later milestone if you want the challenge. |
| C8 | Map library | **MapLibre GL JS** `[VERIFY: current version and React wrapper in its docs]` | Open source, no token, vector tiles. |
| C9 | API | **Small Go HTTP service** serving precomputed JSON, behind a cache header | Readers never touch ingestion (invariant 6). |

C6 and C7 were the two decisions the previous session flagged for you specifically.

---

## 4. Repository shape (target)

```
fire-pulse/
├── go.mod                       # one Go module for everything backend
├── apps/
│   ├── ingest/main.go           # pollers + adapters → raw log → store
│   ├── replay/main.go           # rebuild projections from the raw log
│   ├── api/main.go              # read-only HTTP API over precomputed views
│   └── web/                     # Next.js app, its own pnpm project
├── packages/
│   ├── rawlog/                  # Put/Get/List; local-dir and S3 implementations
│   ├── fetch/                   # polite HTTP: conditional GET, HEAD-compare, budget, backoff
│   ├── canonical/               # Detection, WeatherSample, AirQualitySample, satellite map, IDs
│   ├── adapters/inpe/           # 10-min, daily, events
│   ├── adapters/firms/
│   ├── adapters/openaq/
│   ├── adapters/openmeteo/
│   ├── adapters/inmet/
│   ├── store/                   # Postgres access, migrations
│   └── join/                    # city timeline computation
├── deploy/                      # docker-compose.yml, Terraform (later)
└── docs/
```

Don't create all of this up front. Each directory arrives with the milestone that needs it.

---

## 5. Milestones

Each milestone ends with a commit (CLAUDE.md §4.1: no `Co-Authored-By` trailer).

### M0 — Keys and tools (no code)

**Build:**
- Register the **OpenAQ** key at <https://explore.openaq.org/register>. Put it in `.env` as
  `OPENAQ_API_KEY=…` (`.env` is git-ignored). Never paste it into chat or a sample.
- Find INMET's token process (`[VERIFY]`: the inventory couldn't). Optional: FIRMS `MAP_KEY`.
- Install Go, Docker, Node + pnpm. Check each tool's current stable version on its own site;
  don't trust versions written anywhere in this repo.

**Then finish the inventory for OpenAQ yourself** (good practice for the adapter later). With
`curl` and the key, find out and record in `docs/research/README.md` §4:
- how many Brazilian locations exist and how many reported PM2.5 in the last 24 h;
- their reporting interval;
- the real shape of the latest-value response (capture one into `samples/`, key removed);
- their distance from the 2026-10-29 detections (as finding 7 did for INMET).

**Done when:** §4 of the inventory has no `[VERIFY]` that the key could resolve, and you've
decided C1–C9 (§3) and recorded them.

**Traps:** a captured sample must not contain the key (check headers you saved). If your shell
locale is `pt_BR`, run number-crunching `awk` under `LC_ALL=C`.

### M1 — Scaffold

**Concept:** generators over hand-written boilerplate (CLAUDE.md §2).

**Build:**
- `go mod init` at the repo root. A `apps/ingest/main.go` that loads config from env and logs
  "starting".
- `deploy/docker-compose.yml` with a PostGIS container (check the `postgis/postgis` image's
  current tags on Docker Hub).
- `.env.example` listing every variable the code reads, with no values.

**Done when:** `go build ./...` and `go vet ./...` pass; `docker compose up` gives a database
you can `psql` into and where `SELECT postgis_full_version();` works.

### M2 — The raw log

**Concept:** §2.2.

**Build:** `packages/rawlog`:
- an interface with `Put(ctx, source string, body []byte, meta FetchMeta) (key string, err)`,
  `Get(ctx, key)` and `List(ctx, source, from, to)`;
- a **local-directory** implementation first (fast to test), an S3 one later (M10);
- the key derived from `sha256(body)`, gzip applied on write;
- `Put` of an existing key returns the key and writes nothing.

**Done when:** tests show: `Put` twice → one object; `Get` returns exactly the original bytes
(compare hashes); `List` returns objects in fetch-time order.

**Traps:** don't key by fetch time alone, or the same response fetched twice becomes two objects.
Don't decompress-and-reparse to "normalise" before storing.

### M3 — Polite fetcher + the INPE 10-minute adapter

**Concept:** §2.1, §2.6, §2.7. This is the "watch it get detected" source (finding 2).

**Build:**
- `packages/fetch`: a client that sends `If-None-Match` / `If-Modified-Since` from the last
  `ETag` / `Last-Modified` it saw, sets a `User-Agent`, enforces a per-day budget, backs off on
  errors, and returns `(status, body, headers)`.
- The INPE poller: compute slot names from the UTC clock
  (`focos_10min_YYYYMMDD_HHMM.csv` under `https://dataserver-coids.inpe.br/queimadas/queimadas/focos/csv/10min/`),
  fetch the newest slot a minute or two after it's due, and **catch up** missed slots (the folder
  keeps ~3.8 days).
- `packages/adapters/inpe`: parse a 10-minute file into `[]canonical.Detection`.
- `packages/canonical`: the `Detection` type, the satellite mapping table, the ID function.

**Done when:**
- The parser test reads `docs/research/samples/inpe-20261030-focos-10min-1910.head200.csv` and
  asserts the count and a few exact values (CLAUDE.md: tests use the real samples, never
  hand-built fixtures).
- The satellite-mapping test covers every satellite name listed in inventory §2.2.
- Running the poller for an hour fetches each slot once, logs `304` on re-checks, and stays under
  its budget.

**Traps:**
- The sample has **leading spaces** in the numbers (`  -7.291600, -38.659710,…`). Trim before
  `strconv.ParseFloat`.
- `data` is `YYYY-MM-DD HH:MM:SS` in UTC with no zone marker. Parse it explicitly as UTC.
- A header-only file is "no fires in this slot", not an error (§2.6).
- Whether INPE ever rewrites a slot is `[VERIFY]` (inventory §2.1). Re-check recent slots with
  conditional GETs for a while; it's cheap.

### M4 — The store and idempotent ingestion

**Concept:** §2.3. Plus integration tests against real infrastructure.

**Build:**
- Migrations (pick a Go migration tool and check its docs): `detections` (PK = `detection_id`,
  `geography(Point)` column with a GiST index, event_time, satellite, nullable `frp`/`confidence`,
  provider refs), `fetch_log`.
- `packages/store`: insert detections with `ON CONFLICT (detection_id) DO NOTHING`, or `DO UPDATE`
  that only **fills nulls** (so FIRMS can add confidence later without overwriting).
- Wire it: poller → raw log → adapter → store.

**Done when:**
- An integration test (testcontainers-go with a PostGIS image) ingests the same sample **twice**
  and asserts the table is byte-identical after the second pass (row count *and* contents).
- After a live hour, `SELECT satellite, count(*) … GROUP BY 1` looks like the inventory's mix.

**Traps:** the "fill nulls" update must be deterministic regardless of which source arrives
first, or the late-data test (M7) will fail. Think about it before writing the SQL.

### M5 — Replay

**Concept:** §2.2. The projection is rebuildable.

**Build:** `apps/replay`: empty the projections, `List` the raw log, run each object through
its adapter in fetch order, insert.

**Done when:** a test ingests samples live, takes a snapshot, wipes, replays from the raw log,
and asserts the snapshot is equal. This test will protect you for the rest of the project.

### M6 — FIRMS adapter

**Concept:** finding 1 (same pixels, exact key) and finding 9 (HEAD before GET).

**Build:**
- A FIRMS poller for the four South America 24h files (URLs in inventory §1.1): `HEAD`, compare
  `ETag`, `GET` only if changed, raw log, adapter.
- Parse VIIRS and MODIS columns (they differ: MODIS `confidence` is 0–100, VIIRS is
  `low/nominal/high`). Combine `acq_date` + `acq_time` (`HHMM`, UTC) into event time.
- Filter to Brazil (FIRMS files are South America). Decide how: a bounding box is crude; a
  Brazil polygon in PostGIS is exact. `[VERIFY: a source for the official boundary, e.g. IBGE]`.

**Done when:**
- Parser tests use both FIRMS samples.
- After ingesting the FIRMS sample and the INPE daily sample for the same day, matching VIIRS
  detections collapse onto one row with `confidence` filled. (You can reproduce the inventory's
  ~100% match as a test.)

**Traps:** the "24h" file spans more than 24 h of acquisitions (inventory §1.1); that's fine
because identity de-duplicates. S-NPP stops on 2026-11-02 (finding 3): its file going stale or
404 is expected, and must not page you or mark FIRMS "failed".

### M7 — Source status and the late-data test

**Concept:** §2.5, §2.6.

**Build:**
- A `source_status` view from `fetch_log`: last success per source, gaps, failures.
- The test CLAUDE.md requires: feed the same set of responses in two different orders (e.g.
  FIRMS-then-INPE vs INPE-then-FIRMS; slots shuffled) and assert identical tables.

**Done when:** that test passes, and you can answer from SQL alone: "for 14:00–14:10 yesterday,
were there no fires, or did we fail to fetch?"

### M8 — Wind and air quality

**Concept:** §2.4 (the `kind` field), C5 (cities with stations).

**Build:**
- A **station catalogue**: INMET stations (`https://apitempo.inmet.gov.br/estacoes/T`, sample in
  `samples/`; 540 `Operante`) and OpenAQ Brazilian locations (from M0). Fetch rarely (daily).
- The set of **anchor cities**: places with an air-quality station (and ideally an INMET station
  nearby). This list is a decision you'll make from M0's numbers.
- Adapters:
  - OpenAQ latest values → `AirQualitySample{kind: observed}`.
  - Open-Meteo forecast wind (`wind_speed_10m`, `wind_direction_10m`, many coordinates per
    call) → `WeatherSample{kind: modelled}`.
  - Open-Meteo air quality (CAMS) → `AirQualitySample{kind: modelled}`.
  - INMET, if you get a token → `WeatherSample{kind: observed}`.
- All of them through the raw log, same as M3.

**Done when:** parser tests on the captured samples; a query lists, per anchor city, its latest
observed PM2.5 and its latest modelled wind, each with `kind` visible.

**Traps:**
- Wind direction is conventionally the direction the wind comes **from**.
  `[VERIFY: Open-Meteo's docs state its convention for wind_direction_10m]`. Getting this
  backwards puts every "downwind" city on the wrong side.
- Open-Meteo units were km/h (inventory §6.1). Store SI (m/s) or state the unit in the field
  name; never leave it implicit.
- Never interpolate a station value to a location without a station and call it observed.

### M9 — The join (the heart of the project)

**Concept:** §2.5. Read this milestone twice.

**The question the join answers**, per anchor city C and hour H:
> Which detections were within R km of C in the preceding N hours, and of those, which were
> **upwind** of C according to the wind at the time? And what did C's air-quality station
> measure in hour H?

**Build:** `packages/join`, writing a `city_timeline` table:
- For city C, hour H: `ST_DWithin(detection.geog, city.geog, R)` and event_time in
  `[H − N, H]`.
- **Upwind test:** bearing from the fire to the city vs the direction the wind blows toward
  (`wind_from + 180`). Within ±θ° → "upwind". R, N and θ are **design parameters**: pick them from
  M0's distances, write them in architecture §5, and show them in the UI.
- Which wind? Observed if an INMET station is within some distance, else modelled, and the row
  says which (`wind_kind`).
- Output per row: counts, total FRP, the measured PM2.5 (or "no measurement"), `wind_kind`,
  `provisional` flag.
- Recompute trigger: when detections or samples land, recompute the (city, hour) cells they can
  affect. A simple first version: every few minutes, recompute all cells whose inputs changed
  since the last run (track with an `ingested_at` column).

**Done when:**
- The late-data test runs through the join (two arrival orders → identical `city_timeline`).
- The replay test includes the join.
- A unit test on bearing/upwind with hand-checkable geometry (a fire due north of a city with
  wind from the north is upwind; from the south isn't).

**Traps:**
- **This never claims causation.** The row says "detections upwind" and "PM2.5 measured", side
  by side. The UI copy must too (C6).
- Do the geometry on `geography`, not degrees. A degree of longitude is ~111 km at the equator
  and less further south.

### M10 — Read API

**Build:** `apps/api`: a few GET endpoints (active detections as GeoJSON for a time range,
anchor cities, one city's timeline, source status). Serve from the precomputed tables, set
`Cache-Control`, cap the response size.

**Done when:** a load test against the API (repeated runs, report the median and p95,
CLAUDE.md) shows ingestion latency unchanged while the API is under load.

### M11 — Web app

**Concept:** the global `~/.claude/CLAUDE.md` frontend rules, unchanged.

**Build:**
- Scaffold `apps/web` with the official `create-next-app` via pnpm (check its current docs).
- Server Components by default; the map is a small `'use client'` leaf.
- TanStack Query with one `queryOptions` factory per endpoint, wrapped in named hooks
  (`useActiveDetections(range)`, `useCityTimeline(cityId)`).
- **A table alternative for every map view** (invariant 8). Build the table *first*: it's
  easier, and it forces you to get the data right before the visuals.
- shadcn/ui + Radix for controls; never colour alone for fire age, confidence, AQ band or
  observed vs modelled (add shape, pattern or text).
- "Last updated at …" from `source_status`; the three missing states rendered distinctly.
- `prefers-reduced-motion` respected in any "new detection" animation.

**Done when:** keyboard-only use reaches every control; an automated a11y check (e.g. axe) has
no violations; the "no fires / fetch failed / no coverage" states each have a screenshot test or
manual check recorded in the task doc.

### M12 — Deploy

**Build:** a `t4g.small` Spot instance running Docker Compose; S3 bucket for the raw log;
secrets in SSM Parameter Store; **an AWS Budgets alarm before anything else**. No NAT, ALB,
RDS or MSK (CLAUDE.md, decisions). Terraform is a reasonable choice; check its AWS provider
docs.

**Done when:** the box runs for 24 h; you can state requests/hour per source, bytes/day into the
raw log, and the measured latency distribution per satellite (the inventory's single snapshot
becomes a real measurement).

**Traps:** Spot instances get reclaimed. The raw log on S3 plus replay means a reclaimed box
loses nothing but a few minutes of polling, which `fetch_log` shows as a gap. Check that's true.

### M13 — Replay export (the ending)

**Build:** export chosen windows (a dramatic week, a quiet day) as static JSON + the web app
built in a mode that reads them, to S3 + CloudFront.

**Done when:** with the backend switched off, the site still works and the replay test proves
the export equals what the live system showed.

---

## 6. Where to look when you're stuck

| Question | Look in |
|---|---|
| What does provider X actually send? | `docs/research/samples/` and inventory section for X |
| Why is the system shaped like this? | §2 of this guide; `CLAUDE.md` invariants |
| What did we decide? | `docs/architecture.md` §10 |
| What's the plan for this piece of work? | your `docs/tasks/TASK-<slug>.md` |
| How does library Y work? | Y's own current docs, never memory (CLAUDE.md §2.0) |

## 7. Glossary

- **Adapter**: provider-specific code turning raw bytes into canonical events.
- **Canonical event**: the one internal shape for a kind of thing, independent of provider.
- **Detection / focus (foco)**: one hot pixel seen by one satellite at one time. Not a fire:
  a fire is many detections.
- **Event (INPE "evento")**: INPE's grouping of detections into one fire, with an outline.
- **FRP**: fire radiative power, in MW. A proxy for fire intensity.
- **Geostationary / polar**: GOES-19 stares at the same region (every 10 min); polar satellites
  (VIIRS, MODIS) pass a few times a day at higher resolution.
- **Idempotent**: doing it twice has the same effect as doing it once.
- **Observed / modelled**: measured by an instrument vs computed by a model. Never mixed.
- **Projection**: anything derived from the raw log; can be deleted and rebuilt.
- **Provisional window**: a join result that may still change because late data can arrive.
