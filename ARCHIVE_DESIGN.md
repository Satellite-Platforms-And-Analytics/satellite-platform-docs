# Cloud as serving layer, workstation as warehouse

**Status: proposed, not implemented. Read before I build it — this
changes prune semantics on a live pipeline, and a mistake here loses
data that cannot be re-fetched.**

---

## Why

`satellite-platform-frontend` has one API route, `app/api/tles/route.ts`,
and it reads one table: `satellites`, columns `norad_id, name,
tle_line1, tle_line2, orbit_regime`. Nothing else in the app touches the
database. No module in `src/tracking/` mentions `DATABASE_URL` at all —
the Visibility Tool runs entirely from local files.

Every read of the three large tables, across the whole codebase:

| Table | Share of 481 MB | Reads |
|---|---|---|
| `visibility_windows` | 258 MB (54%) | one `count(*)` in `compute.py` |
| `orbital_positions` | 214k rows | **none** — written and pruned only |
| `tle_history` | 344k rows | `max(fetched_at)`, counts, sizes — diagnostics |

So a 500 MB tier is 96% full of data nothing consumes yet. It is not
waste — it is the history the dashboard will want — but it is a warehouse
living inside a serving layer. Meanwhile `D:` has **1.3 TB free**: 2,700×
the room, on the machine the work already happens on.

The obstacle is that the pipelines run in GitHub Actions, which can reach
Supabase and cannot reach `D:`. So something has to move the data, and
the cloud must not delete anything before that something has run.

---

## The shape

```
GitHub Actions  ──writes──▶  Supabase          ──archives──▶   D:\Databases\satellite\archive\
(unchanged)                  (serving layer)    (workstation)   (Parquet, unlimited history)
                                  │                                      ▲
                                  └──prunes only up to the watermark─────┘
```

Supabase keeps, permanently: `satellites`, `sensors`, `countries`,
`catalog_events`, `ingestion_log`, `archive_watermark`. All small, all
read by the frontend or the monitor.

Supabase keeps, transiently: `visibility_windows`, `orbital_positions`,
`tle_history` — only until archived, plus a safety margin.

---

## The watermark, and why the obvious version is wrong

The naive design is "archive nightly, prune to 1 day". It loses data the
first time the workstation is off for two nights, and it loses it
silently. The prune has to be subordinate to the archive, not scheduled
alongside it.

```sql
CREATE TABLE archive_watermark (
    table_name       TEXT PRIMARY KEY,
    archived_through TIMESTAMPTZ NOT NULL,
    row_count        BIGINT,
    archive_host     TEXT,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`archived_through` means: *every row whose time column is ≤ this value is
on disk at `D:`.* The workstation advances it only after a successful
write and a verified row count.

The time column differs per table, which is a real wrinkle rather than a
detail:

| Table | Time column | Type |
|---|---|---|
| `tle_history` | `epoch` | `timestamptz` |
| `orbital_positions` | `timestamp` | `timestamptz` |
| `visibility_windows` | `analysis_date` | `date` |

`visibility_windows` stores its watermark as that date at 00:00 UTC, and
prunes on `analysis_date` as it does today, so a run still leaves whole
rather than half-deleted.

### Prune becomes three-way

```
soft_cutoff  = now() - retention          # what we would like to keep
watermark    = archive_watermark.archived_through
hard_cutoff  = now() - max_retention      # backstop

DELETE WHERE time_col < LEAST(soft_cutoff, watermark)   -- normal path
DELETE WHERE time_col < hard_cutoff                     -- backstop, loud
```

- **Normal:** the archive is current, `watermark` is recent, and the
  prune behaves exactly as it does now.
- **Workstation off:** `watermark` stops advancing, the effective cutoff
  stops moving, and the table grows instead of losing rows. Nothing is
  lost.
- **Workstation off for a long time:** the backstop fires. Unarchived
  rows are deleted, because the alternative is filling the tier and
  breaking ingestion for everything. It writes a `catalog_events` row and
  logs at ERROR, so it appears in the next monitor digest. **This is the
  one path that loses data, and it must never be silent.**

`max_retention` should be set from how long the tier can absorb the
growth, not picked round. That needs the measurement below.

---

## Archive format

`D:\Databases\satellite\archive\<table>\<YYYY-MM-DD>.parquet`, one file
per table per day, append-only, never rewritten.

Parquet rather than CSV because the consumer is a future dashboard doing
trend queries, and DuckDB reads a directory of Parquet files as one table
with no server to run. Compression should take the 258 MB of
`visibility_windows` to somewhere in the tens of MB — worth measuring on
the first real run rather than promising.

Needs `pandas` + `pyarrow`. `pandas` is already declared in
`requirements-tracking.txt`; `pyarrow` is declared nowhere, so this adds
`requirements-archive.txt` — and `tests/test_requirements.py` will fail
until it does, which is the drift test doing its job on its first real
change.

---

## What I still need before building

Two things, both measurements, because the backstop interval and the
prune retention should come from the growth rate rather than from me
picking numbers that sound reasonable:

```
cd D:\Projects\Satellite-Platform\satellite-platform-ingestion
python check_bloat.py
```

That gives current per-table sizes and dead-tuple share. The second is
growth per day per table, which I can derive from row counts and the
known daily write volumes already recorded in `writer.py`
(`visibility_windows` at 221,788 rows/day measured 2026-09-01;
`tle_history` at ~30,000 rows/day ≈ 10 MB).

---

## Order of work, once agreed

1. `007_archive_watermark.sql` — the table, RLS, grants. Cloud-side only,
   changes no behaviour.
2. `archive_to_local.py` — workstation script: read new rows above the
   watermark, write Parquet to `D:`, verify the count, advance the
   watermark. Safe to run repeatedly; does not delete anything.
3. Run it for several days with the prunes untouched, and confirm the
   archive matches the cloud row-for-row.
4. **Only then** change the prune call sites to respect the watermark,
   and shorten retention.
5. `VACUUM FULL` to actually return the space — a `DELETE` marks rows
   dead without giving the filesystem anything back, and Supabase bills
   on what is on disk.

Steps 1–3 are reversible and lose nothing. Step 4 is the one that needs
step 3 to have actually passed, not to have been assumed.
