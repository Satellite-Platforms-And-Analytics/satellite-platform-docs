# Build Guide

How to build this platform from nothing, in the order that works — written
after building it once the wrong way.

> **Status:** Stages 0–2 are built and running. Stage 3 onward is planned,
> not built, and is marked as such. Last updated **2026-09-04**.
>
> **This is a living document.** Every stage that gets built should have
> its section rewritten from *plan* to *recipe*, with the commands that
> actually worked and the traps actually hit. See
> [Maintaining this document](#maintaining-this-document).

For what happened versus what should have happened, see
[`AS_BUILT.md`](./AS_BUILT.md). This file is the recipe; that one is the
post-mortem.

---

## The one principle

**Every step ends with a verification that would fail if the step
silently didn't work.**

Not "it ran without error" — this project lost five days to a job that
logged `fetch success` while writing nothing, and three more to a build
that printed `✓ Compiled successfully` and was then discarded. Both
"succeeded". Neither did anything.

Each step below has a **Verify** line. Do not skip it. It is the whole
difference between this taking a week and taking three months.

---

## Before you start

### Accounts
| Service | Why | Cost |
|---|---|---|
| GitHub | Repos + Actions (2,000 min/month free) | $0 |
| Supabase | Postgres + RLS + auth + storage | $0 (500 MB) |
| Vercel | Frontend hosting | $0 (Hobby) |
| CelesTrak | TLE source — no account needed | $0 |
| Space-Track | Authoritative catalogue; account required | $0 |

### Local tooling
- **Python 3.11+.** Not 3.9 — modern `dataclass(slots=True)` needs 3.10+,
  and building a venv from an old interpreter fails confusingly.
- **Node 20+** for the frontend (Next.js 16 requires it).
- **Git.**
- QGIS and the geospatial stack only if you are doing imagery. You are
  not, yet — see Stage 6.

---

## Stage 0 — Foundation

### 0.1 Repositories

Four, not a monorepo — they deploy to different places and have different
dependency graphs.

```
satellite-platform-ingestion       Python pipeline
satellite-platform-infrastructure  schema + migrations
satellite-platform-frontend        Next.js
satellite-platform-docs            this guide
```

**Do:** create all four with a README. Add a `.gitattributes` to each
immediately:

```gitattributes
* text=auto eol=lf
```

**Verify:** `git add` a file and confirm no "LF will be replaced by CRLF"
warning.

> **Trap.** Without `.gitattributes`, Git on Windows rewrites working
> copies to CRLF and whole files show as modified when nothing changed.
> Phantom modifications hide real ones. Costs nothing on day one; annoying
> to retrofit.

### 0.2 Supabase

**Do:** create the project. Copy the **pooled** connection string (port
6543, not 5432) — Actions runners need the pooler.

**Verify:** connect and run `SELECT 1`.

### 0.3 Local environment

```bash
python3.11 -m venv .venv        # name the interpreter explicitly
.venv/Scripts/activate          # Windows;  source .venv/bin/activate elsewhere
pip install -r requirements.txt
```

**Verify:** `python -c "import sgp4, skyfield, sqlalchemy, psycopg2; print('ok')"`

> **Trap.** Never build a venv on top of an existing one — it inherits a
> stale `site-packages` and fails in ways that point nowhere. Delete the
> directory first, or use `--clear`.

### 0.4 Do NOT connect a deploy target yet

**This is the single most expensive mistake in this project's history.**

A Vercel project created against an empty repo detects no framework, sets
the preset to "Other" with an output directory of `public/`, and then
**silently discards every successful build** — for three months, in our
case. The build compiles, the log says so, and the deployment is thrown
away by a post-build check.

Connect Vercel at **Stage 5**, when there is a Next.js app to detect. And
commit `vercel.json` regardless:

```json
{ "framework": "nextjs" }
```

---

## Stage 1 — Schema

### 1.1 Core tables

`infrastructure/schema/001_core_schema.sql`. Eight tables:

| Table | Holds | Growth |
|---|---|---|
| `countries` | reference | static |
| `satellites` | current catalogue, one row per object | bounded (~18k) |
| `tle_history` | every element set ever seen | **unbounded** |
| `orbital_positions` | propagated lat/lon/alt | **unbounded** |
| `sensors` | ground sites | static |
| `visibility_windows` | which sensor sees what, when | **unbounded** |
| `imagery_scenes` | scene metadata | slow |
| `ingestion_log` | one row per pipeline step | slow |

**Every table marked unbounded needs a retention policy written at the
same time as its writer.** Not later. See 1.7.

### 1.2 Grants — the step everyone misses

RLS policies are **half** of public read access:

- **GRANT** decides whether a role may touch the table at all.
- **POLICY** decides which rows it sees once it may.

Postgres refuses before consulting any policy. Ship both:

```sql
GRANT USAGE ON SCHEMA public TO anon;
GRANT SELECT ON TABLE satellites TO anon;   -- and each public table
```

Then revoke what Supabase's defaults hand out:

```sql
REVOKE TRUNCATE, TRIGGER, REFERENCES ON ALL TABLES IN SCHEMA public FROM anon;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    REVOKE TRUNCATE, TRIGGER, REFERENCES ON TABLES FROM anon;
```

> **Why:** Supabase grants ALL on new public tables to `anon`. **RLS does
> not apply to TRUNCATE** — a role that can empty a table is not
> constrained by policies about which rows it may read.

**Verify:**
```sql
SELECT table_name, privilege_type FROM information_schema.role_table_grants
 WHERE grantee = 'anon' AND table_schema = 'public';
```
`anon` should hold SELECT and nothing else.

### 1.3 A migration runner

Write it before the second migration, not after:

```bash
python apply_migration.py 002_public_read_grants.sql --dry-run
python apply_migration.py 002_public_read_grants.sql
```

Print every statement before executing, run the file in one transaction,
print the resulting state afterwards.

---

## Stage 2 — Ingestion

### 2.1 Environment loading

One module, `src/env.py`, with a `bootstrap()` that puts the repo root on
`sys.path` **then** loads `.env`. Call it from every entry point.

> **Trap.** The obvious form fails silently:
> ```python
> try:
>     from src.env import load_env; load_env()
> except ImportError:
>     pass                      # <-- swallows the only case that matters
> ```
> `python path/to/file.py` puts that file's directory on `sys.path`, not
> the repo root, so the import raises and `.env` is never read. The run
> dies later claiming `DATABASE_URL` is unset, on a machine where it is
> set. **Always invoke as `python -m src.x`.**

### 2.2 The TLE fetcher

CelesTrak, `gp.php?FORMAT=json` — **not** the fixed-width TLE endpoint,
which cannot represent 6-digit catalogue numbers. Re-derive TLE lines
through sgp4's exporter with Alpha-5 encoding.

**Group names must be validated.** CelesTrak answers an unknown group with
**HTTP 200 and an `Invalid query` body**, so status codes cannot detect
it. Our config named `noaa` and `debris`; neither exists, and the platform
tracked no debris at all for months.

Fetch only groups holding objects `active` excludes:
`active`, `analyst`, `cosmos-2251-debris`, `iridium-33-debris`,
`cosmos-1408-debris`.

**Respect the source.** 3-second spacing, a persistent fetch log honouring
a 2-hour re-fetch window, and any bulk probing gated behind `--force`.
Write a `docs/API_USAGE_POLICY.md` and keep it current.

**Verify:** `python -m src.tle.fetcher --check-groups` — every configured
group resolves.

### 2.3 The writer — batch it from the start

```python
from psycopg2.extras import execute_values
execute_values(cur, "INSERT INTO t (...) VALUES %s ON CONFLICT ...", rows)
```

> **Trap, and the most expensive one here.** **SQLAlchemy cannot batch a
> `text()` statement.** It falls through to `cursor.executemany()` — a
> per-row loop. 16,600 rows became 16,600 round trips at 24/sec: 11.6
> minutes against a 15-minute Actions timeout, clearing the bar about half
> the time. **697 s → 11.8 s** after conversion.

**Timestamps:** CelesTrak's OMM `EPOCH` has **no timezone designator**.
`datetime.fromisoformat` returns a naive datetime; comparing it to an
aware one raises `TypeError`. Attach UTC on parse — OMM epochs are UTC by
definition.

**Verify:** write a batch and time it. Under 20 seconds for ~17k rows.

### 2.4 Propagation

sgp4 alone — no skyfield in the 2-hourly path, because it pulls numpy and
downloads a timescale file at runtime. TEME → ECEF → WGS84 explicitly
(Vallado GMST, Bowring iteration).

Skip element sets older than 14 days and any dated more than a day in the
future.

**Verify:** compare against Skyfield on a handful of objects across
LEO/SSO/GEO. Ours agrees to **44 m**. If yours is kilometres out, the
GMST or the geodetic conversion is wrong.

### 2.5 Visibility

Build a **separate module**, do not retrofit `--headless` onto an
interactive tool. An analyst tool with prompts and an unattended job with
parameters are different programs that happen to share maths.

Read sensors from the **database**, not a Python dict — they drift.

Use skyfield here (unlike 2.4) so results match the interactive tool
exactly; `load.timescale(builtin=True)` avoids the runtime download.

**Verify:** field-of-regard results must be a strict subset of a
co-located full-sky sensor's.

### 2.6 Guard against your own bugs

Do not write `except Exception: continue` around a per-item loop. A defect
in the computation then makes every item "fail", the job exits 0, and the
table stays empty.

```python
if attempted and errors / attempted > MAX_ERROR_FRACTION:
    raise RuntimeError(...)   # bad data is expected; 100% failure is a bug
```

Also: **sgp4 parses prose into a 1949 epoch** rather than raising. Without
a structural check on the lines, a catalogue of garbage looks like a
catalogue of old-but-valid element sets.

### 2.7 Retention — before you need it

```python
prune_old_positions(hours=48)
prune_old_visibility_windows(days=3)
prune_old_tle_history(days=14)
```

> **Trap.** A **DELETE does not return space to the filesystem.** 1.2 M
> rows deleted, size unchanged, because Postgres marks rows dead and keeps
> the pages — and Supabase bills on the file. Run `VACUUM FULL` after a
> large prune. It cannot run inside a transaction, and
> `engine.raw_connection().autocommit = True` does **not** work (that sets
> an attribute on SQLAlchemy's pool proxy). Use
> `execution_options(isolation_level="AUTOCOMMIT")`.

**Verify:** `pg_total_relation_size` before and after. Measure bytes per
row; do not estimate them. Three estimates here were wrong, one by 10x.

### 2.8 Observability — every job logs a step

```python
run_id = new_run_id()
log_step(run_id, pipeline="tle_fetch", step="write_db",
         status="success", records_processed=n, duration_s=elapsed)
```

> **Why this is not optional.** A GitHub Actions timeout **kills the
> process** — no exception, so no `failed` row is ever written. The
> failure produces silence. **The absence of a step is the only signal,
> which requires there to be a step.** Two of our jobs ran unattended for
> days with no logging; a bug in either would have left no trace at all.

**Verify:** a liveness check per pipeline against its own cadence, not one
global freshness rule.

---

## Stage 3 — Scheduling

Three workflows, each on an **offbeat minute** — top-of-the-hour jobs
compete with everyone else's and are likelier to be delayed or dropped.
Ours were landing 8–12 hours apart on `0 */2`.

| Workflow | Cron | Timeout |
|---|---|---|
| `ingest_tle.yml` | `37 */2 * * *` | 15 min |
| `propagate.yml` | `17 * * * *` | 20 min |
| `ingest_visibility.yml` | `43 6 * * *` | 20 min |

**Set every timeout from a measured runtime.** Ours: visibility measured
71 s, timeout 20 min. A timeout that fires is invisible.

Install dependencies per workflow, not from `requirements.txt` — that file
carries the GDAL-heavy geospatial stack, and a rasterio build failure
should not take down TLE ingestion.

**Verify:** after the first scheduled run, `ingestion_log` has rows from
it.

---

## Stage 4 — CI

```yaml
on: [push, pull_request]
```

**Make skipped guards fail.** A test that skips when its input is missing
is right on a laptop and wrong in CI, where it means the guard silently
never ran inside a green build. Pass `REQUIRE_SCHEMA=1` (or equivalent)
and turn the skip into a failure.

**Scope collection.** `pytest.ini`:

```ini
testpaths = tests
pythonpath = .
addopts = -ra
```

> **Two traps, both cost a CI run each.** A root-level diagnostic named
> `test_*.py` gets imported by pytest whether or not it is a test — name
> operational scripts `check_*` / `verify_*`. And **`python -m pytest`
> puts the working directory on `sys.path`; bare `pytest` does not**, so a
> suite can pass locally and fail in CI on the same commit. `pythonpath = .`
> fixes both invocations.

**Verify:** CI is green *and* the test count matches what you expect. Ours
runs 54 of 58 — the 4 skips are deliberate and named in the log.

---

## Stage 5 — Frontend

Only now connect Vercel.

**Ship element sets, not positions.** Server-side propagation at 5-minute
resolution for 18,000 objects is 2.3 M rows/day — roughly the entire
500 MB tier — and ~12,960 Actions minutes against a 2,000 minute
allowance. A TLE is ~140 bytes and stays usable for days: send those and
run SGP4 in the browser.

- **Canvas, not SVG.** 1,500 satellites is 1,500 DOM nodes per frame.
- **Server route, not a direct browser→Supabase call.** Keeps the query
  shape controlled, lets one edge cache serve everyone, trims the payload.
- **`ORDER BY` before `LIMIT`**, and sample across the catalogue.
  `norad_id` is issued in launch order, so `ORDER BY norad_id LIMIT n` is
  not a sample — it is the *n oldest objects*. Ours rendered a convincing
  globe with no Starlink, no ISS and an unpopulated GEO belt.
- **PostgREST caps responses at 1,000 rows** and does not say so. Page
  with `.range()`.
- **`Cache-Control: s-maxage=...` gives browsers nothing.** Without
  `max-age=0` they apply heuristic caching and serve stale data after a
  fix ships.
- **Set `maxDuration`.** A cold serverless function doing ~19 round trips
  will exceed a 10-second default and 504 with no application error.
- **Fail loudly on missing config.** Our page renders and names both
  missing environment variables. That is the difference between a
  five-minute fix and an afternoon.

**Verify:** load the production URL, check the object count matches what
the API reports, and check the console is clean.

---

## Stage 6 — Catalog intelligence · NOT YET BUILT

Planned. See [[Sprint Plan 2026-09-05]].

1. `004_catalog_fields.sql` — operator, country, purpose, launch, mass,
   plus `data_source` and `source_confidence`.
2. UCS Satellite Database and UNOOSA registry into `data/seed/`,
   **downloaded by hand and committed**. No scrapers.
3. A matcher: NORAD id → international designator → name, with confidence
   recorded per row. Prefer an unmatched satellite to a wrongly attributed
   one.

> `.gitignore` blanket-ignoring `*.csv` will silently swallow seed data.
> Un-ignore `data/seed/` explicitly and verify with `git check-ignore -q`.

## Stages 7+ · NOT YET BUILT

Industry intelligence → funding and finance → launch platforms →
analysis. In that order. GIS/imagery ingestion is deliberately deferred to
the end (AD-035).

---

## Verification toolkit

Build these early; they pay for themselves immediately.

| Script | Answers |
|---|---|
| `verify_paths.py` | does every configured path resolve? |
| `check_pipeline.py` | row counts, freshness, per-pipeline liveness |
| `check_write_speed.py` | are writes completing, or timing out? |
| `check_tle_history.py` | real churn, or defeated deduplication? |
| `cleanup_tle_history.py` | prune + `VACUUM FULL` |
| `python -m src.visibility --sizes` | the tier budget, measured |

---

## The traps, collected

1. Framework detected against an empty repo — discards every build.
2. SQLAlchemy `text()` cannot batch — silent timeouts.
3. An Actions timeout kills the process — no `failed` row, ever.
4. CelesTrak returns HTTP 200 for unknown groups.
5. OMM `EPOCH` carries no timezone designator.
6. RLS policies without GRANTs — permission denied.
7. RLS does not apply to TRUNCATE.
8. DELETE does not reclaim disk; `VACUUM FULL` does.
9. `except ImportError: pass` hides the one failure that matters.
10. `python -m pytest` ≠ `pytest` on `sys.path`.
11. A file named `test_*.py` gets imported by pytest regardless.
12. `ORDER BY norad_id LIMIT n` is not a sample.
13. PostgREST silently caps at 1,000 rows.
14. `s-maxage` alone tells browsers nothing.
15. A skipped guard inside a green build is invisible.

Every one of these was hit here. None was hypothetical.

---

## Maintaining this document

**When you build a stage:** rewrite its section from plan to recipe. Put
in the commands that actually worked, and every trap actually hit — a trap
you hit and did not record will be hit again.

**When you hit a trap:** add it to the collected list with one line on how
it presented, not just what it was. "Permission denied for table X" is
what you will search for at 11pm.

**When a decision changes:** record it in the master roadmap's Decision
Log and update the affected step here. This document should never
disagree with the roadmap; if it does, the roadmap wins and this file is
stale.

**Review cadence:** at each monthly review, read the not-yet-built stages
and ask whether they still describe the plan.

---

## Related
- [`AS_BUILT.md`](./AS_BUILT.md) — what was actually done vs what should
  have been
- `Satellite_Platform_And_Analytics_Master_Roadmap.md` (vault) — canonical
  plan and decision log
- `08_Periodic/README.md` (vault) — the project's history week by week
