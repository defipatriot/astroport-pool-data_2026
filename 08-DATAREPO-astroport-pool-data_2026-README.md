# astroport-pool-data_2026

Daily Astroport pool snapshots for the Terra Alliance DAO (TLA). Pool list is rediscovered from the TLA gauge controller every run, so this captures whatever pools the DAO is currently voting on.

Companion cron script lives at [`defipatriot/cron-scripts/astroport`](https://github.com/defipatriot/cron-scripts/tree/main/astroport).

---

## Directory layout

```
astroport-pool-data_2026/
├── README.md                              ← you are here
├── astroport/                             ← rich per-epoch JSON
│   ├── astroport-epoch-184.json
│   ├── astroport-epoch-185.json
│   └── ...
└── data/
    ├── daily/                             ← one CSV per day
    │   ├── 2026-05-12.csv
    │   ├── 2026-05-13.csv
    │   └── ...
    ├── weekly-avg/                        ← rollup per TLA epoch
    │   ├── 2026-epoch-184.csv
    │   ├── 2026-epoch-185.csv
    │   └── ...
    └── monthly-avg/                       ← rollup per calendar month
        ├── 2026-05.csv
        └── ...
```

### Why JSON + CSV side-by-side

- **JSON** preserves all per-pool detail: 30 days of per-epoch history, sample counts, deprecation reasons, fetch errors. Use it for analytics, time-series work, or rebuilding a CSV with different aggregation rules.
- **CSV** is a flat one-row-per-pool snapshot, joinable in any spreadsheet tool or pandas DataFrame. Use it for quick reports, dashboards, or "show me the top pools by TVL today."

### Daily JSON behavior

The daily cron writes to `astroport/astroport-epoch-{N}.json` — **same filename for the entire epoch**. So each day during epoch 184, the file gets overwritten with fresher data (more samples per epoch as the week progresses). When epoch 185 starts, a new file is created. This means:

- You always have ≤1 file per epoch in `astroport/`
- The file always contains the **most current** view of that epoch
- Daily CSVs in `data/daily/` preserve the day-by-day series

---

## Schema (JSON)

See [`cron-scripts/astroport/README.md`](https://github.com/defipatriot/cron-scripts/tree/main/astroport#field-reference--json-output) for the complete schema. Top-level fields:

```jsonc
{
  "schemaVersion": 1,
  "capturedAt":     "2026-05-12T23:59:00.000Z",
  "period":         184,
  "runMode":        "daily",
  "stats":          { "ok": 9, "deprecated": 11, "failed": 0, "total": 20 },
  "pools":          [ /* per-pool details */ ]
}
```

## Schema (CSV)

See [`cron-scripts/astroport/README.md`](https://github.com/defipatriot/cron-scripts/tree/main/astroport#field-reference--csv-output) for column-by-column definitions.

---

## Pool coverage

The cron tracks **all Astroport LP pools registered in the TLA gauge controller** (across all 4 buckets: stable, project, bluechip, single). Single-sided staking gauges (xASTRO, ampCAPA staking, etc.) are skipped — those aren't LP pools and Astroport doesn't index them in the chart endpoints.

Skeleton Swap pools are captured separately by [`ss-pool-data_2026`](https://github.com/defipatriot/ss-pool-data_2026).

### Why some pools show as `deprecated: true`

The TLA gauge controller doesn't garbage-collect old pool addresses. When a pool migrates (e.g. Astroport redeploys a pool contract due to a code upgrade), the old address can stay listed in the gauge config. Astroport's indexer drops these old addresses, returning "Pool not found" to chart queries.

The cron detects this pattern and flags the row with `deprecated: true`. The pool name still appears in the data (sometimes alongside a live pool with the same name on the new address) so analysts can see what's voted-but-dead.

---

## Capture timing

- **Daily:** every day at 23:59 UTC
- **Weekly rollup:** Monday at 00:05 UTC (covers previous Mon→Sun epoch)
- **Monthly rollup:** 1st of each month at 00:10 UTC

### TLA epoch alignment

`epoch` numbers in the CSV/JSON match TLA's convention: `EPOCH_START = 2022-10-31T00:00:00Z`, `EPOCH_DURATION = 7 days`. Same numbering as [`votion-data_2026`](https://github.com/defipatriot/votion-data_2026), [`ss-pool-data_2026`](https://github.com/defipatriot/ss-pool-data_2026), and `tla-tool_ext.html`.

---

## Data source

- **Pool discovery:** Terra LCD (`terra-rest.publicnode.com`, fallback `terra.publicnode.com`) querying TLA gauge controller `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj`
- **Chart data:** Astroport TRPC at `app.astroport.fi/api/trpc/charts.{liquidity,volume}` — no auth, no CORS issues server-side, ~5s response time per pool

---

## Available history

This repo was initialized empty in May 2026. Historical data before that doesn't exist here — Astroport's chart API only serves recent windows (last ~30 days), so historical pool TVL/volume going further back would need a different source (Phoenix-1 chain indexer scrape, etc.).
