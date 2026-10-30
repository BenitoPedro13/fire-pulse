# Research — sources

Verified facts about every data source, each backed by a real captured response in `samples/`.
**Nothing here may come from memory.** A cell is filled in only after it has been checked
against the live source, with the URL linked and the capture date stated.

**Status (2026-10-30): partly verified** by `docs/tasks/TASK-source-inventory.md`. All captures
ran on 2026-10-30 between 20:07 and 20:40 UTC (a Friday, late in the dry/fire season), with the
User-Agent `fire-pulse-research/0.1 (one-off source inventory; contact: repo owner)`.
**Blocked:** OpenAQ and INMET station data need keys the user has to request (see "Keys" at the
end). Their rows stay partly `[VERIFY]` until then.

> Method warning for whoever repeats these numbers: this Mac's shell locale is `pt_BR`, so
> `awk` reads `-7.29` as `-7` unless it runs under `LC_ALL=C`. The first duplicate and distance
> runs were wrong for that reason and were redone; every number below is from the `LC_ALL=C`
> run. Parsers must be locale-independent (Go's `strconv` is).

## Source inventory

Kind: `observed` = a satellite or ground measurement; `modelled` = forecast, analysis or
reanalysis output.

| Source | Provides | Kind | Source URL | Auth | Update interval (measured) | Latency (measured) | Format | Licence | Verified? |
|---|---|---|---|---|---|---|---|---|---|
| NASA FIRMS, public regional files | active-fire pixels, MODIS + VIIRS (S-NPP, NOAA-20, NOAA-21) | observed | [§1.1](#11-public-regional-files-no-key) | none | hourly (FAQ); file `Last-Modified` 19:49 for a 20:07 fetch | VIIRS NRT ≈ 2 h (newest pixel 17:54, file 19:49); FAQ: "<3 hours" | CSV | NASA open data; cite "NASA FIRMS" | **yes, 2026-10-30** |
| NASA FIRMS, area/country API | same + history (NRT/SP) | observed | [§1.2](#12-area-api-map_key) | **free `MAP_KEY`** | — | — | CSV | same | docs only; needs the key |
| INPE Programa Queimadas, 10-min files | active-fire points, **all satellites incl. GOES-19** | observed | [§2.1](#21-10-minute-files) | none | a new file every 10 min | GOES-19 p50 **20 min**; VIIRS p50 163–180 min; MODIS 79–111 min | CSV (4 columns) | `[VERIFY: licence text]` | **yes, 2026-10-30** |
| INPE Programa Queimadas, daily/monthly files | same, enriched (municipality, state, biome, FRP, UUID) | observed | [§2.2](#22-daily-and-monthly-files) | none | today's file grows during the day; past days fixed | — | CSV | `[VERIFY]` | **yes, 2026-10-30** |
| INPE "Eventos de Fogo" | detections grouped into **fire events** with polygons, area, fronts, duration | derived from observed (INPE's own clustering, "Provisório") | [§2.3](#23-fire-events-eventos-de-fogo) | none | hourly (page); GPKG `Last-Modified` 19:21 | — | GPKG / KML | `[VERIFY]` | **yes, 2026-10-30** |
| INPE "Fumaça AOD469" | daily mean aerosol optical depth | **modelled** (ECMWF CAMS), rescaled ÷3 and capped at 1.0, **although INPE files it under "observado"** | [§2.4](#24-fumaça-aod469-a-model-filed-as-observed) | none | daily, ≈ 2 days late | — | GeoTIFF, 231×230 | `[VERIFY]` | **yes, 2026-10-30** |
| INPE DETER (TerraBrasilis) | deforestation and **burn-scar** polygons, Amazon + Cerrado | observed (CBERS-4 / Amazonia-1 imagery, human-classified) | [§3](#3-inpe-deter) | none | `[VERIFY]` | newest `view_date` 2026-10-23, a week old | WFS (GeoJSON) | `[VERIFY]` | **yes, 2026-10-30** (5-alert sample; the WFS failed on larger requests) |
| OpenAQ v3 | air-quality stations (PM2.5 …) | observed | [§4](#4-openaq) | **free API key (`X-API-Key`)** | `[VERIFY]` | `[VERIFY]` | JSON | no blanket licence; per-provider terms + attribution to OpenAQ | **blocked on the key** |
| INMET automatic stations | **observed wind**, temperature, humidity, rain, hourly | observed | [§5](#5-inmet-automatic-weather-stations) | station list: none; **data: token** | `[VERIFY]` | `[VERIFY]` | JSON | `[VERIFY]` | station list **yes**; data **blocked on the token** |
| Open-Meteo forecast | 10 m wind speed/direction/gusts | **modelled** (ICON / GFS / IFS global; no South American regional model) | [§6.1](#61-forecast-api-wind) | none | `[VERIFY: model run cadence]` | — | JSON | CC BY 4.0, non-commercial free tier | **yes, 2026-10-30** |
| Open-Meteo air quality | PM2.5, AOD, CO, dust … | **modelled** (CAMS global, 0.4°, 3-hourly) | [§6.2](#62-air-quality-api-cams) | none | CAMS global every 12 h (docs) | — | JSON | CC BY 4.0 + CAMS attribution | **yes, 2026-10-30** |

## ⚠ Findings that contradict or change the plan (for the user to decide)

1. **FIRMS and INPE are the same detections for every polar satellite.** On 2026-10-29,
   **100.0%** of INPE's NOAA-20 (4,236), NOAA-21 (6,116), S-NPP (5,112) and Terra (936)
   detections in Brazil, and **99.8%** of Aqua (1,076 of 1,078), appear in FIRMS with the same
   satellite, the same minute and coordinates equal to 3 decimals (≈ 100 m). The "FIRMS ↔ INPE
   duplicate" problem in the plan is therefore **not a fuzzy spatial match: it's an exact key**,
   `(satellite, acq minute, lat/lon rounded)`. What INPE adds that FIRMS doesn't have in its
   files is **GOES-19** (5,032 that day), **METOP-B/C** (522) and **MSG-03** (34), plus
   municipality, state and biome per point. What FIRMS adds is `confidence`, brightness, scan/track
   and `daynight`.
2. **Only INPE gives "watch it get detected".** GOES-19 is geostationary: it looks at Brazil
   every 10 minutes and its points reach INPE's 10-minute files a **median 20 minutes** after
   detection. Polar satellites pass about 4 times a day and reach either catalogue **1.5–3 hours**
   later. FIRMS lists GOES as "Provisional" in its FAQ but has **no GOES source in its API or
   public files** (`[VERIFY: a FIRMS GOES download path]`). The live "it just appeared" moment
   has to come from INPE's GOES-19; FIRMS becomes the enrichment and cross-check for polar passes.
3. **Suomi NPP is ending now.** The FIRMS FAQ banner reads "Delivery of new Suomi NPP data ceases
   on Nov. 2, 2026", three days after this inventory. S-NPP was 22% of INPE's detections on
   2026-10-29. NOAA-20 and NOAA-21 carry the same VIIRS instrument and continue. Adapters must
   treat a satellite disappearing as normal, not as a failure.
4. **INPE already groups detections into fire events.** "Eventos de Fogo" (hourly, GPKG) has
   3,700 active events with an `id_evento`, polygons, area in hectares, number of fronts,
   duration, a fire type (`Incêndio`, `Atividade Antrópica`, `Nova Queima Isolada`, `Possível
   início de incêndio`) and a per-day evolution table. It's labelled **provisional**. We can
   **show** it as INPE's published grouping, or **build** our own clustering; doing both and
   showing where they disagree is possible. This is a decision (architecture §10).
5. **INPE's "observed smoke" is a model.** The only smoke product on INPE's open-data page,
   `fumaca_aod469` under "Risco de Fogo e Meteorologia → observado", is **ECMWF CAMS** output
   (`GRIB_centre: ecmf`, `forecast_reference_time`, `forecast_period 0`), averaged per day,
   **divided by 3 and capped at 1.0** (its own `history` attribute). Under the "smoke is never
   fabricated" rule it must be shown as a model, and the ÷3 rescale means it isn't even raw AOD.
   Open-Meteo serves the unscaled CAMS fields hourly, so INPE's file adds nothing.
6. **There is no observed smoke plume in any source checked.** Observed data near a fire is:
   the detections themselves, ground air-quality stations (OpenAQ, blocked on a key) and ground
   wind (INMET, blocked on a token). Smoke transport exists only as **CAMS model output**. The
   headline "see the smoke reach the city" is honest only as **"a fire was detected here; this
   station downwind measured this"**, with CAMS as a clearly labelled model layer. See
   architecture §10.
7. **Stations are far from fires.** Of 23,072 detections on 2026-10-29, the nearest *operating*
   INMET station (540 of 669) was within 25 km for **3.1%**, within 50 km for **13.3%**, within
   100 km for **54.7%** and within 200 km for **92.4%**. 81% of that day's detections were in the
   Amazon. Air-quality stations are expected to be sparser still in the Amazon
   (`[VERIFY: OpenAQ Brazilian locations, once the key exists]`). A join radius of 25–50 km
   would match almost nothing; the join has to be framed around **cities with a station**, not
   around fires.
8. **Volume swings ≈ 145× between seasons.** All satellites, Brazil: **353** detections on a
   quiet day (2026-04-05; April median 955) and **51,063** on the peak day of September 2026
   (2026-09-21). That's ≈ 8.1 MB/day of INPE CSV at the peak (158 B/row), ≈ 2.2 MB gzip.
   Small: the broker in the Transit Pulse template is not needed for volume (architecture §6).
9. **FIRMS ignores conditional requests.** It sends `ETag` and `Last-Modified` but answered
   `If-None-Match` and `If-Modified-Since` with a full `200` (1.08 MB). A `HEAD` (no body) returns
   the same `ETag`, so the polite pattern is `HEAD`, compare, then `GET`. INPE's Apache returns
   `304` for both.
10. **OpenAQ v3 and INMET need credentials.** OpenAQ v1/v2 are gone (`410`); v3 returns `401`
    without `X-API-Key`. INMET's data endpoints return `204` without a token and "CHAVE
    INVÁLIDA!" with a wrong one. **The user has to register** (see "Keys").
11. **DETER doesn't belong in the live fire scope.** It's deforestation plus burn-scar polygons
    classified from CBERS-4/Amazonia-1 imagery, with the newest `view_date` a week old, and its
    WFS failed on anything beyond a handful of features. At most a later "context" layer
    (architecture §1).
12. **Open-Meteo wind is a model, and its terms need a check.** Free tier: 600/min, 5,000/h,
    10,000/day, non-commercial ("private or non-profit websites … without subscriptions or
    advertising"). A personal portfolio isn't listed either way; the terms also name
    "promotional activities" as commercial. `[VERIFY: email info@open-meteo.com before the live
    run]`. For Brazil "best match" uses global models only (no South American regional model).

---

## 1. NASA FIRMS

### 1.1 Public regional files (no key)

- URLs (all `200`, fetched 2026-10-30 20:07 UTC):
  - `https://firms.modaps.eosdis.nasa.gov/data/active_fire/suomi-npp-viirs-c2/csv/SUOMI_VIIRS_C2_South_America_24h.csv`
    (1,080,246 B, 13,206 rows)
  - `https://firms.modaps.eosdis.nasa.gov/data/active_fire/noaa-20-viirs-c2/csv/J1_VIIRS_C2_South_America_24h.csv`
    (821,168 B, 9,796 rows)
  - `https://firms.modaps.eosdis.nasa.gov/data/active_fire/noaa-21-viirs-c2/csv/J2_VIIRS_C2_South_America_24h.csv`
    (1,180,669 B, 14,070 rows)
  - `https://firms.modaps.eosdis.nasa.gov/data/active_fire/modis-c6.1/csv/MODIS_C6_1_South_America_24h.csv`
    (211,039 B, 2,718 rows)
  - `_48h.csv` and `_7d.csv` variants exist (S-NPP: 1,624,948 B and 5,178,534 B).
- **"24h" covers more than 24 h of acquisitions:** the S-NPP file held pixels from 2026-10-29
  03:47 to 2026-10-30 17:54 UTC (two night and two day passes). FIRMS doesn't define the window
  (`[VERIFY: VIIRS C2 user guide]`). Polling the 24h file and de-duplicating on identity is
  therefore safe; nothing relies on the window being exact.
- Update cadence: "the SHP, KML and CSV files … are updated every 60 minutes" (FIRMS FAQ,
  <https://www.earthdata.nasa.gov/data/tools/firms/faq>).
- Latency: "NRT data available within 3 hours of a satellite observation (on a best-effort
  basis)" (FAQ). Measured once: newest S-NPP pixel 17:54, file `Last-Modified` 19:49 → ≈ 2 h.
  Ultra/Real-Time is US/Canada only. The real distribution is a box measurement (architecture
  §7, `[VERIFY]`).
- Headers (`samples/firms-20261030-active-fire-csv.headers.txt`): `ETag` (= MD5),
  `Last-Modified`, `x-amz-meta-sha256`, **no `Cache-Control`**, no `Content-Encoding` even when
  `gzip` is accepted. **Conditional GETs return `200`.**
- Columns, VIIRS: `latitude,longitude,bright_ti4,scan,track,acq_date,acq_time,satellite,confidence,version,bright_ti5,frp,daynight`.
  MODIS: `brightness`, `bright_t31` instead of the `ti4/ti5` pair, and `confidence` is 0–100.
- Meanings (VIIRS page,
  <https://www.earthdata.nasa.gov/data/instruments/viirs/viirs-i-band-375-m-active-fire-data>):
  `acq_date`/`acq_time` are **UTC** (`HHMM`); `confidence` ∈ {`low`,`nominal`,`high`} (VIIRS);
  the page says low-confidence *night* pixels in the South Atlantic Anomaly (which covers
  Brazil) "have been removed from the NRT data distributed by FIRMS"
  `[VERIFY: the share of low-confidence day pixels in Brazil]`; `frp` is MW; `daynight` `D`/`N`; satellite `N` = S-NPP, `N20`, `N21`;
  MODIS `T`/`A`. `version` was `2.0NRT` (VIIRS) and `6.1NRT` (MODIS) on every row.
- Attribution: cite "NASA FIRMS"; acknowledgement text "We acknowledge the use of … NASA LANCE
  FIRMS … part of the NASA Earth Science Data and Information System (ESDIS)" (FAQ).
- Samples: `firms-20261030-viirs-noaa21-southamerica-24h.head200.csv`,
  `firms-20261030-modis-southamerica-24h.head100.csv`.

### 1.2 Area API (`MAP_KEY`)

- Documented at <https://firms.modaps.eosdis.nasa.gov/api/area/>:
  `/api/area/csv/[MAP_KEY]/[SOURCE]/[west,south,east,north]/[DAY_RANGE 1..5]/[DATE]`.
  Sources: `MODIS_NRT`, `MODIS_SP`, `VIIRS_SNPP_NRT|SP`, `VIIRS_NOAA20_NRT|SP`,
  `VIIRS_NOAA21_NRT|SP`, `LANDSAT_NRT` (US/Canada only). No GOES source.
- Limit: "5000 transactions / 10-minute interval", larger day ranges count as several.
- `/api/country/` answered `400` without parameters; its path format is
  `[VERIFY: with the key]`.
- **Needed for:** history older than 7 days through FIRMS (the replay can use INPE's monthly
  files instead) and `data_availability`. **Not needed for the live adapter**, which reads the
  public files.

## 2. INPE Programa Queimadas

Open-data index: <https://terrabrasilis.dpi.inpe.br/queimadas/portal/pages/secao_downloads/dados-abertos/>.
Files live on `https://dataserver-coids.inpe.br/queimadas/queimadas/` (Apache directory
listings; listing times are **America/Sao_Paulo**, file names are **UTC**). `Access-Control-Allow-Origin: *`.

### 2.1 10-minute files

- `…/focos/csv/10min/focos_10min_YYYYMMDD_HHMM.csv`, one per 10-minute UTC slot, listed ≈ 0–1
  min after the slot. The folder held **553 files (≈ 3.8 days), 4.7 MB in all, ≈ 1.2 MB/day**.
- Columns: **only** `lat,lon,satelite,data` (`data` = detection time, UTC, `YYYY-MM-DD HH:MM:SS`).
  No ID, no FRP, no municipality.
- Conditional GET: **`304`** for both `If-None-Match` and `If-Modified-Since`. A slot's `ETag` was
  unchanged an hour later (`[VERIFY: that slots are never rewritten, over a day on the box]`).
- **Latency** = file slot − detection time, over the 144 files of the last 24 h (26,182 rows,
  no row in two files):

| Satellite | rows | min | p50 | p90 | max (minutes) |
|---|---|---|---|---|---|
| GOES-19 | 5,394 | 20 | **20** | 680 | 1,660 |
| MSG-03 | 70 | 25 | 25 | 25 | 25 |
| TERRA_M-M | 230 | 65 | 79 | 183 | 185 |
| AQUA_M-T | 1,299 | 81 | 83 | 163 | 171 |
| METOP-C | 159 | 97 | 97 | 97 | 97 |
| TERRA_M-T | 332 | 70 | 111 | 111 | 130 |
| NOAA-21 | 7,461 | 115 | 166 | 170 | 180 |
| NOAA-20 | 5,380 | 150 | 163 | 200 | 203 |
| NPP-375 | 3,312 | 119 | 179 | 184 | 220 |
| NPP-375D | 2,063 | 169 | 180 | 182 | 204 |

  GOES-19's p90 of 680 min means about 1 in 10 GOES points arrives many hours late: **late data
  is normal, not an edge case** (architecture §5).
- Sample: `inpe-20261030-focos-10min-1910.head200.csv` (+ `.headers.txt`).

### 2.2 Daily and monthly files

- Daily: `…/focos/csv/diario/Brasil/focos_diario_br_YYYYMMDD.csv` (also `America_Sul/`). The
  folder keeps **32 days**. **Today's file grows during the day** (2026-10-30: 11,609 rows,
  `Last-Modified` 20:06 UTC); a past day's file was last written the next day at ≈ 12:05 UTC.
- Monthly: `…/focos/csv/mensal/Brasil/focos_mensal_br_YYYYMM.csv` from at least 2025-09; the
  current month is rewritten daily (2026-09: 117 MB, 777,214 rows). Annual: `…/anual/`
  (`Brasil_todos_sats`, `Brasil_sat_ref`, …).
- Columns: `id,lat,lon,data_hora_gmt,satelite,municipio,estado,pais,municipio_id,estado_id,pais_id,numero_dias_sem_chuva,precipitacao,risco_fogo,bioma,frp`.
  - `id` is a **version-3 UUID** (name-based, so deterministic) on every row.
  - `numero_dias_sem_chuva`, `precipitacao`, `risco_fogo` are **empty in the daily file**
    (23,072 of 23,072 on 2026-10-29) and **filled in the monthly file** (777,212 of 777,214):
    INPE enriches later. `frp` was empty on 556 rows.
  - **INPE duplicates itself:** 94 rows on 2026-10-29 (88 `TERRA_M-M`, 6 `TERRA_M-T`) repeat the
    same satellite, minute and pixel with latitude 0.00001° apart and **a different UUID**. INPE's
    `id` is therefore not usable as our identity on its own (architecture §4).
- Satellites on 2026-10-29: NOAA-21 6,116; GOES-19 5,032; NOAA-20 4,236; NPP-375D 2,898;
  NPP-375 2,214; AQUA_M-T 1,078; TERRA_M-M 585; TERRA_M-T 351; METOP-C 285; METOP-B 237;
  MSG-03 34; AQUA_M-M 6. `NPP-375` held the day passes (16–18 UTC) and `NPP-375D` the night passes
  (03–05 UTC), with no pixel in both `[VERIFY: INPE's definition of the "D" suffix and of M-M/M-T]`.
- Biomes on 2026-10-29: Amazônia 18,689; Cerrado 2,213; Caatinga 1,909; Mata Atlântica 255;
  Pampa 6.
- Daily counts, all satellites (from the monthly files): see finding 8; September 2026 per day is
  in the table below.

| Day (Sep 2026) | all | VIIRS | GOES-19 | AQUA_M-T |
|---|---|---|---|---|
| 07 (lowest) | 9,778 | 6,526 | 2,277 | 516 |
| 16 | 28,315 | 19,848 | 7,332 | **15** (a gap in the reference satellite) |
| 21 (highest) | 51,063 | 33,925 | 13,789 | 1,641 |
| 30 | 28,022 | 21,293 | 5,256 | 991 |

- Samples: `inpe-20261029-focos-diario-br.sample.csv` (150 rows, the duplicate pair, and rows from
  GOES-19, METOP-B, MSG-03, NPP-375D, AQUA_M-T), `inpe-202609-focos-mensal-br.head50.csv`.

### 2.3 Fire events ("Eventos de Fogo")

- `…/eventos/ativos/eventos_ativos.gpkg` (20,557,824 B) and `.kml` (26 MB);
  `…/eventos/observacao/eventos_observacao.gpkg` (11 MB). The page says "atualizados a cada
  hora" and "Produto em fase de validação — nível de maturidade Provisório".
- GPKG (SQLite, EPSG:4326), `last_change` 2026-10-30T19:21Z, three tables:
  - `eventos_ativos` (3,700): `id_evento`, `tipo_fogo`, `status_nome`, `data_min`, `data_max`,
    `duracao_dias`, `area_ha`, `qtd_frente`, `max_frp`, `total_focos`, `ultimo_foco`,
    `municipios`, `estados`, `desmatamento_pct`, … and a MultiPolygon.
    By type: `Incêndio` 877 (2,149,434 ha), `Atividade Antrópica` 311, `Nova Queima Isolada`
    1,920, `Possível início de incêndio` 592.
  - `evolucao_evento` (13,856): one row per front per day, with polygon, FRP and `delta_area`.
  - `focos_ativos` (21,616): the points, `foco_id` + `id_evento`. **All 21,616 `foco_id`s are
    `id`s from the 2026-10-29 daily CSV**, so events join to detections by INPE's UUID. It held
    only 2026-10-29's points, although `ultimo_foco` reaches 2026-10-30 18:20
    `[VERIFY: what window focos_ativos covers]`.
- Largest event: `860779`, an `Incêndio` in Pará (Mojuí dos Campos, Prainha, Santarém, Uruará),
  2026-08-30 → 2026-10-29, 61 days, **187,338 ha**, 65,075 detections.
- Samples: `inpe-20261030-eventos-ativos.schema.txt`, `.top10.csv`, `-evolucao-860779.csv`,
  `-focos-860779.head50.csv`, `.headers.txt`.

### 2.4 "Fumaça AOD469": a model filed as observed

- `…/riscofogo_meteorologia/observado/fumaca_aod469/media_diaria_aod469_YYYYMMDD.tiff`, 212 KB,
  231×230 float32, published ≈ 2 days after the date.
- Its GDAL metadata (`samples/inpe-20261028-fumaca-aod469.gdal-metadata.xml`) says
  `institution: European Centre for Medium-Range Weather Forecasts`, `forecast_reference_time`
  00/12 UTC, `forecast_period 0`, and the processing history: `cdo daymean`, then
  `aod469 / 3`, then `(aod469 > 1.0) ? 1.0 : aod469`. **Modelled, rescaled, capped.**
- Other INPE meteorology files (observed/forecast fire risk, rain, temperature, humidity, days
  without rain) exist in the same tree; not inspected `[VERIFY: whether "observado" there is
  station-based or model-based]`.

## 3. INPE DETER

- WFS: `https://terrabrasilis.dpi.inpe.br/geoserver/deter-amz/ows` (layers
  `deter-amz:deter_amz`, `deter-amz:updated_date`, `deter-amz:prodes_reference`) and
  `deter-cerrado-nb:deter_cerrado`.
- Fields: `gid, classname, quadrant, path_row, view_date, sensor, satellite, areauckm, uc,
  areamunkm, municipality, mun_geocod, uf, publish_month, geom`.
- The 5 newest alerts (`sortBy=view_date D`): `view_date` 2026-10-23, `CBERS-4`/`AWFI`,
  classes `CICATRIZ_DE_QUEIMADA` (burn scar) and `DESMATAMENTO_CR`.
  Sample: `deter-amz-20261030-latest5.json`.
- Requests for `updated_date`, a 2,000- and a 300-feature page failed with an HTML "Falha
  interna" page or `502`.
- Verdict: deforestation and burn scars a week after the fact; not a live fire source.

## 4. OpenAQ

- `https://api.openaq.org/v3/…` → `401` "A valid API key must be provided in the X-API-Key
  header." `https://api.openaq.org/v2/locations` → `410` "Version 1 and Version 2 API endpoints
  are retired".
- Docs (read 2026-10-30): free key at `explore.openaq.org/register`, **60 requests/min and
  2,000/h** per key, `429` above it, headers `x-ratelimit-used|remaining|limit|reset`
  (<https://docs.openaq.org/using-the-api/rate-limits>). Latest-value endpoints
  `GET /v3/locations/{id}/latest` and `GET /v3/parameters/{id}/latest`
  `[VERIFY: against the API reference with the key]`.
- Public S3 archive `openaq-data-archive` (listing works without credentials):
  `records/csv.gz/locationid=…/year=…/month=…/location-…-YYYYMMDD.csv.gz`, columns
  `location_id, sensor_id, location, datetime, lat, lon, parameter, unit, value`, written
  **72 h after the end of the day** and "may be retroactively patched"
  (<https://docs.openaq.org/aws/about>). Useful for the replay, not for live.
- Terms (<https://docs.openaq.org/about/terms>): no single licence; follow each original
  provider's terms; "Attribution to OpenAQ … is also required"; downloading "strictly prohibited
  unless done through registered and authorized use".
- **Unmeasured until the key exists:** Brazilian locations, how many reported PM2.5 in the last
  24 h, their reporting interval, distance from fire clusters, and which Brazilian networks
  (state agencies, low-cost sensors) they come from.

## 5. INMET automatic weather stations

- `https://apitempo.inmet.gov.br/estacoes/T` (no auth, 261,396 B): **669 automatic stations,
  540 `Operante`, 129 `Pane`**, with `CD_ESTACAO`, latitude, longitude, altitude, state.
  Sample: `inmet-20261030-estacoes-T.head20.json`.
- Data: `https://apitempo.inmet.gov.br/estacao/{start}/{end}/{code}` → `204` (empty);
  `https://apitempo.inmet.gov.br/token/estacao/{start}/{end}/{code}/{token}` → "CHAVE INVÁLIDA!"
  with a placeholder token. **A token is required** `[VERIFY: how INMET issues it and its
  limits; the portal manual URL tried returned 404]`.
- Why it matters: it is the only **observed** wind source found. Distance from the 2026-10-29
  detections: finding 7.

## 6. Open-Meteo

### 6.1 Forecast API (wind)

- `https://api.open-meteo.com/v1/forecast?latitude=-8.76,-23.55&longitude=-63.90,-46.63&hourly=wind_speed_10m,wind_direction_10m,wind_gusts_10m&past_days=1&forecast_days=1&timezone=UTC`
  → `200`, a JSON array, one object per coordinate, hourly, km/h and degrees. **No `ETag`,
  `Last-Modified` or `Cache-Control`.** Sample: `openmeteo-20261030-forecast-wind-2pts.json`.
- Docs (<https://open-meteo.com/en/docs>): several coordinates per call (comma-separated);
  `past_days` 0–92; for Brazil "best match" is "Global models only" (ICON, GFS, IFS).
- Limits (<https://open-meteo.com/en/terms>): < 10,000 calls/day, 5,000/h, 600/min, by IP;
  "more than 10 weather variables or … more than 2 weeks" counts as several calls; how several
  coordinates count is unspecified `[VERIFY]`. Licence CC BY 4.0 with a visible "Weather data by
  Open-Meteo.com" link (<https://open-meteo.com/en/licence>).

### 6.2 Air-quality API (CAMS)

- `https://air-quality-api.open-meteo.com/v1/air-quality?latitude=-8.76&longitude=-63.90&hourly=pm2_5,aerosol_optical_depth,carbon_monoxide&past_days=1&forecast_days=1&timezone=UTC&domains=cams_global`
  → `200`; the grid point snapped to −8.80, −63.90. Sample:
  `openmeteo-20261030-airquality-cams-portovelho.json`.
- Docs (<https://open-meteo.com/en/docs/air-quality-api>): Brazil is covered only by **CAMS
  global, 0.4° (≈ 45 km), 3-hourly, updated every 12 h**, 5-day forecast; `past_days` 0–92.
  "All users … must provide a clear attribution to CAMS".

## Keys the user has to request

| Source | Needed for | Where | Notes |
|---|---|---|---|
| **OpenAQ** | observed PM2.5 at stations, the core of the join | <https://explore.openaq.org/register> | free; 60/min, 2,000/h; one key per person |
| **INMET** | observed wind at 540 stations | `[VERIFY: INMET's token request channel]` | without it, wind is model-only |
| **FIRMS `MAP_KEY`** | history and the area/country API; not needed for live | <https://firms.modaps.eosdis.nasa.gov/api/area/> → "Get MAP_KEY" (email) | free; 5,000 transactions / 10 min |

Keys go into `.env` locally and SSM on the box, never into git (CLAUDE.md §4).

## Samples

All under `samples/`, captured 2026-10-30 unless the name says otherwise, trimmed. Large files
(not committed) and where they came from:

| File | Size | URL | Captured (UTC) |
|---|---|---|---|
| FIRMS S-NPP / NOAA-20 / NOAA-21 / MODIS 24h, South America | 1.08 / 0.82 / 1.18 / 0.21 MB | §1.1 | 20:07:38 |
| INPE daily 2026-10-29, Brasil | 3,350,012 B (899,748 B gzip) | `…/diario/Brasil/focos_diario_br_20261029.csv` | 20:09 |
| INPE 10-min files, last 24 h | 144 files, 1.7 MB | `…/focos/csv/10min/` | 20:10–20:13 |
| INPE monthly 2026-09 / 2026-04 | 117 MB / 4.8 MB | `…/mensal/Brasil/` | 20:25 |
| INPE eventos_ativos.gpkg | 20,557,824 B | §2.3 | 20:15 |
| INPE AOD469 2026-10-28 | 217,386 B | §2.4 | 20:15 |

## Method notes

- Every script ran from the session scratchpad, not the repo. Requests were sequential; the
  only loop (144 INPE 10-minute files) waited 1 s between requests.
- The FIRMS ↔ INPE match key was `(satellite, acquisition minute, lat %.3f, lon %.3f)`; mapping
  `N`→S-NPP (INPE `NPP-375` and `NPP-375D`), `N20`→`NOAA-20`, `N21`→`NOAA-21`, MODIS `T`→`TERRA_*`,
  `A`→`AQUA_*`.
- Station distance uses an equirectangular approximation (111.2 km/°, scaled by cos(latitude)),
  which is accurate to well under 1% at these distances.
- Docs were read through a fetch tool; quotes above are from those pages, with URLs.
