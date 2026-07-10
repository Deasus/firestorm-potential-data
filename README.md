# firestorm-potential-data

NWCG **7-Day Significant Fire Potential** mirror for [FIRESTORM](https://firestorm-gray.vercel.app).

Fetches every 3h from
`fsapps.nwcg.gov/psp/arcgis/rest/services/npsg/outlooks_forecast/MapServer`,
server-side geometry-simplified (`maxAllowableOffset=0.05°` ≈ 5 km, invisible loss
at national scale), and publishes a slim `data/potential.json` for the FIRESTORM
frontend to consume via `raw.githubusercontent.com`.

## Why the bridge

The NWCG endpoint responds with **~30 MB of native GeoJSON per day** and routinely
returns HTTP 400 or times out beyond 25s. Under the old frontend-direct path
(FIRESTORM `fetchFirePotential` at `AbortSignal.timeout(30000)`), timeouts caught
silently and the map painted "NO OUTLOOK ISSUED · DAY 1" while the country actually
had **18 CRITICAL zones + 1 IGNITION** on Day 1 and **22 IGNITION zones on Day 5**
(verified 2026-07-10).

Server-side decimation preserves 100% of the features (234 total / 213 valid /
CRITICAL + IGNITION counts identical to native) and cuts payload to ~150 KB per
day. Combined all-7-day file lands at **~1 MB**, cold-start ~200 ms on any
network path.

## What's in `data/potential.json`

```jsonc
{
  "generated_at": "2026-07-10T17:36:32Z",
  "source": "fsapps.nwcg.gov/psp/arcgis/rest/services/npsg/outlooks_forecast/MapServer",
  "resolution_deg": 0.05,
  "counts": {
    "total_features": 1383,
    "valid_features": 1278,
    "critical": 28,
    "ignition": 36,
    "by_day": {
      "1": {"total":234,"valid":213,"critical":18,"ignition":1},
      // ...
      "7": {"total":213,"valid":213,"critical":0,"ignition":0}
    }
  },
  "day_source": { "1":"fresh", "2":"fresh", ..., "7":"fresh" },
  "days": {
    "1": [
      { "attrs":{"drynesscode":2,"type":null,"isvalid":1,
                 "gacc":"California North Ops","gacc_name":"California North Ops",
                 "nat_code":"NC03B","timestampdate":1783641600000},
        "rings":[[[lng,lat], ...], ...] },
      // ...
    ],
    "2": [...], ..., "7": [...]
  },
  "attribution": "NWCG Predictive Services · Significant Fire Potential Outlook · public domain"
}
```

`day_source["N"] == "stale-carryover"` means the fresh fetch for that day failed
(NWCG intermittent 400) and the file is holding yesterday's outlook for that day.
The overall file's `generated_at` still reflects the latest run; the frontend
should badge stale-carryover days honestly.

## Resilience

- **Per-day retries** with exponential backoff (4/8/16/32 s).
- **Inter-day 3 s delay** — NWCG returns 400 under back-to-back parallel
  geometry-simplified queries even at modest payload sizes.
- **Per-day carryover** — one day failing = keep yesterday's data for that
  day. All 7 days failing on a cold start = keep the whole prior file.

## Field-name gotcha

The NWCG NPSG field for coordination center is **`gacc_name`, NOT `gacc`**. The
pre-pipeline FIRESTORM code was passing `outFields=gacc`; NWCG accepts the
invalid outField without error but omits it from features. `fetch_potential.py`
uses `gacc_name` and emits BOTH `gacc` and `gacc_name` in the slim JSON so any
frontend that read the old field continues to work.

## No API key. Public domain.

NWCG NPSG is a `.gov` public service, no auth, no egress charge. Data is US
Federal Government public domain (NIFC Predictive Services product).

## Companion pipelines

Same architecture pattern as `firestorm-lightning-data`, `firestorm-goes-fire-data`,
`firestorm-ngfs-data`, `firestorm-cameras`, `firestorm-wind-data`,
`firestorm-hrrr-data`, `firestorm-aircraft-data`, `firestorm-satellite-data`,
`firestorm-aqi-data`, `firestorm-news-data`, `firestorm-imsr-data`,
`firestorm-cams-aerosol-data`.
