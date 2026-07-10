# Architecture — firestorm-potential-data

Companion to `firestorm-lightning-data` / `firestorm-goes-fire-data`; same
bridge pattern (GHA cron → slim JSON in repo → frontend `fetch()` from
`raw.githubusercontent.com`). This doc records the non-obvious bits.

## Pipeline

```
NWCG NPSG ArcGIS MapServer  (public, no auth, ~30 MB/day at native res)
  fsapps.nwcg.gov/psp/arcgis/rest/services/npsg/outlooks_forecast/MapServer/{0..6}
        │  per-day query with maxAllowableOffset=0.05 + geometryPrecision=3
        │  serialized with 3s inter-day gap, 4× retry w/ backoff on 400/500/timeout
        ▼
fetch_potential.py  (GHA runner, every 3h)
        │  slim → per-day features [{attrs, rings}] → summarize CRIT/IGN counts
        │  per-day carryover if fresh fetch fails
        ▼
data/potential.json   (~1 MB combined 7 days, committed to repo)
        ▼
FIRESTORM index.html  fetch() → existing `potential` GraphicsLayer + canvas glow
```

## The data product

NWCG Predictive Services publishes **Significant Fire Potential Outlook** as
7 MapServer layers (`0..6` = Day 1..7). Each layer contains ~234 polygons
covering all 9 GACCs across CONUS + AK. Semantics per layer feature:

| field         | values                                                          |
|---------------|-----------------------------------------------------------------|
| `drynesscode` | 0 = not-yet-dry, 1 = normal, 2 = elevated, 3 = high             |
| `type`        | `null` = baseline dryness only, `CRITICAL`, `IGNITION`          |
| `isvalid`     | 1 = active in this geography, 0 = out-of-season/inactive        |
| `gacc_name`   | GACC name (**NOT `gacc`** — see gotcha)                          |
| `nat_code`    | national code (e.g. `NC03B`)                                    |
| `timestampdate` | epoch ms of the forecast start time                            |

Refreshed daily by NICC. During peak season CRITICAL + IGNITION zones appear;
shoulder season is mostly baseline dryness. Both cases are operationally
meaningful — the Day-5 lookahead of IGNITION clusters is the operator's
biggest leading indicator for the coming week.

## Server-side simplification

`maxAllowableOffset=0.05` (in output SR degrees, so ~5 km at CONUS latitudes)
+ `geometryPrecision=3` (coordinate rounding to 3 decimal places, ~110 m).
Measured against native (2026-07-10):

| offset | Day 1 payload | fetch time | features |
|--------|---------------|------------|----------|
| native | 29 MB+       | 30 s+ (timeout) | (partial)|
| 0.02   | 277 KB       | 5.4 s       | 234 / 213 valid / 18 CRIT / 1 IGN |
| **0.05** | **158 KB**    | **3.2 s**    | 234 / 213 valid / 18 CRIT / 1 IGN |
| 0.10   | 107 KB       | 2.7 s       | 234 / 213 valid / 18 CRIT / 1 IGN |
| 0.20   | 79 KB        | 2.5 s       | 234 / 213 valid / 18 CRIT / 1 IGN |

Feature count is identical at every decimation level; only ring vertex count
drops. 0.05 was chosen: GACC boundaries still snap tight, payload is small,
retry budget is comfortable inside the workflow's 8-min timeout.

## Field-name gotcha (2026-07-10)

The correct field for coordination center is `gacc_name`. NWCG's MapServer
accepts `outFields=gacc` without erroring but silently omits it (the field
doesn't exist under that name). Verified via `MapServer/1?f=json` — the
`fields` list contains `gacc_name` (`esriFieldTypeString`).

`fetch_potential.py` uses `gacc_name` and emits BOTH `gacc` and `gacc_name`
into slim JSON so any consumer that read the pre-pipeline field name
continues to work.

## Intermittent HTTP 400

NWCG's MapServer returns `{"error":{"code":400,"message":"Failed to execute
query."}}` under two conditions we've observed:
1. Back-to-back parallel geometry-heavy requests (the pre-pipeline symptom
   — the FIRESTORM frontend fired all 7 days in parallel).
2. Random per-layer flake — a given day's layer occasionally returns 400 for
   ~5-30 min at a time (observed on Day 1 on 2026-07-10, layers 2-7 fine).

Mitigations:
- **Serialize with 3s inter-day delay.** Kills class (1).
- **Retry 4× with exponential backoff.** Recovers most of class (2).
- **Per-day carryover from prior JSON.** Bounds the worst case: if a day
  is still down after all retries, viewers see yesterday's outlook for
  that day, tagged `day_source: "stale-carryover"`.

## Frontend integration (FIRESTORM index.html)

- Existing `gfxLayers.potential` GraphicsLayer + `activeLayers.potential`
  toggle are unchanged.
- `fetchFirePotential` swaps from 7-parallel-ArcGIS-queries to a single
  `fetch()` against `raw.githubusercontent.com/Deasus/firestorm-potential-data/main/data/potential.json`.
- Freshness gate: 26h on `generated_at` (covers a missed publish + weekend).
- Per-day `day_source == 'stale-carryover'` should render an "outlook
  pending" note on that day's scrubber pill so operators know.

## Attribution

NWCG Predictive Services — US Federal Government public domain. No CC license
required; a courtesy label ("NWCG NPSG · Significant Fire Potential") is
already in the popup template.
