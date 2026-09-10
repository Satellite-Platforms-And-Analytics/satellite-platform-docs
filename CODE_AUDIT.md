# Code audit — 2026-09-10

Asked for: does everything work, are there unintended outcomes, and is
there code here that does not belong to this project or its planned
future.

Every number below was measured, not estimated. Where something is a
judgement rather than a measurement it says so.

---

## 1. What the repository is made of

`satellite-platform-ingestion`: **59 Python files, 20,816 lines.**

Reachability was computed by walking the import graph from the real
entry points — the five workflow commands plus every test file.

| | modules | lines | reached |
|---|---|---|---|
| Pipeline (`src/tle`, `src/db`, `src/propagation`, `src/visibility`, `src/catalog`) | 8 | ~3,400 | yes |
| Tests | 11 | ~2,900 | yes |
| **`src/tracking/`** | **22** | **11,460** | **one function** |
| `src/imagery/` | 1 | 722 | no |
| `src/resources.py` | 1 | ~380 | no |
| Root diagnostics | 10 | ~1,277 | no |

---

## 2. `src/tracking/` — 55% of the code, one function used

The entire pipeline touches it at exactly one line:

```
src/visibility/compute.py:201
    from src.tracking.satellite_utils import in_sensor_field_of_regard
```

`satellite_utils.py` is 1,499 lines. The function used is about forty,
pure numpy, with no dependency on the rest of the module. Nothing else in
`src/tracking/` is imported by anything outside `src/tracking/`.

**This is not simply dead code, and should not be deleted on that
basis.** `FORK_ANALYSIS.md` established that this copy is *ahead* of the
Satellite Visibility Tool on API hardening — `SpaceTrackMalformedResponseError`,
`ensure_spacetrack_session`, `spacetrack_session_is_valid`,
`parse_gp_history_json_record`, and the N2YO transaction ceiling exist
here and not in the tool. Deleting it would discard the code written
after the incidents that motivated it.

So it is a **fork artifact**: 11,460 lines vendored for one function,
carrying improvements that belong in the tool.

**Judgement, not measurement:** the resolution is to port the hardening
back to the tool, extract `in_sensor_field_of_regard` into
`src/visibility/`, and then remove the vendored copy. That is a
deliberate piece of work, not a cleanup.

**Cost while it stays:** every workflow installs `numpy` and `requests`
because importing `satellite_utils` pulls them in at module scope. The
monitor workflow installs numpy for a job that only reads the database.

---

## 3. requirements.txt lists nine packages nothing imports

Checked every entry against actual imports:

| Package | Imported by |
|---|---|
| aiohttp | **nothing** |
| geopandas | **nothing** |
| shapely | **nothing** |
| pyproj | **nothing** |
| fiona | **nothing** |
| dask | **nothing** |
| xarray | **nothing** |
| alembic | **nothing** |
| apscheduler | **nothing** |

`ruff` and `mypy` are tools rather than imports and are legitimate.
`rasterio` **is** imported, by `src/imagery/ingest.py`.

**Why this matters more than a tidy-up.** Every one of the five
workflows carries a paragraph explaining that it cannot use
`requirements.txt` because of "the geospatial stack (rasterio,
geopandas, fiona, pyproj) for `src/imagery/`, which needs system GDAL".
Four of those five packages are imported by nothing at all — including
by the imagery module itself, which uses only `rasterio`.

So a repository-wide workaround, repeated in five files, is justified by
a dependency set that largely does not exist. The consequence is five
hand-maintained install lists, and they have already drifted:

| workflow | installs |
|---|---|
| ci | numpy psycopg2 requests sgp4 skyfield sqlalchemy |
| ingest_visibility | numpy psycopg2 requests sgp4 skyfield sqlalchemy |
| enrich_catalog | numpy psycopg2 requests sqlalchemy |
| monitor_catalog | numpy psycopg2 requests sqlalchemy |
| propagate | psycopg2 sgp4 sqlalchemy |

Nothing verifies that any of these matches what the code imports. A
missing package fails at runtime in a scheduled job — which is the
failure mode this project has the worst history with.

---

## 4. Code that is future work, not bloat

Separated deliberately, because the request was about what does not
pertain to the project **or its planned future**.

**`src/imagery/ingest.py` (722 lines)** — Copernicus/Landsat scene
ingestion. Unreached, but GIS was explicitly deferred to the end of the
project rather than dropped. Keep.

**`src/resources.py` (~380 lines)** — reads TLE sources, tracking
databases and launch providers from WIT, a separate project at
`D:\Projects\WIT`. Nothing in the pipeline imports it; only two root
diagnostics do. It plausibly feeds the launch-platforms and industry
domains named as main effort.

**This one needs a decision rather than an assumption.** It is the only
cross-project coupling in the repository: `src/resources.py` inserts
`D:\Projects\WIT` into `sys.path` at import time, and four tests skip
permanently in CI because that database is not there. If WIT is part of
the plan, the coupling should be made explicit and the tests should not
be silently skipping. If it is not, this is the one piece that genuinely
does not belong.

---

## 5. Root diagnostics — one is a leftover

Ten scripts, ~1,277 lines, none reachable from a workflow. Most are
deliberate manual tools and earn their place:

`check_pipeline.py`, `check_catalog.py`, `check_ledger.py`,
`check_new_objects.py`, `check_bloat.py`, `check_tle_history.py`,
`cleanup_tle_history.py`, `report_events.py`.

Two are one-off debugging leftovers:

- **`check2.py`** — flagged "unexplained, undocumented" in `AS_BUILT.md`
  since before this audit. It is a near-duplicate of
  `check_resources.py`, differing only by clearing a private `_cache`
  attribute and printing `km_subdomain`. A debugging variant, kept by
  accident. **Delete.**
- `check_write_speed.py` and `check_writer_roundtrip.py` — written to
  measure the per-row write problem on 2026-09-01, which was fixed.
  Harmless, but they answer a question nobody will ask again.

`verify_paths.py` and `check_resources.py` are environment checks and
worth keeping while the WIT question is open.

---

## 6. Recommended order

1. **Fix `requirements.txt`** — remove the nine unimported packages.
   Cheap, reversible, and it removes the justification for five
   divergent install lists.
2. **Add a test that the workflow install lists cover what the code
   imports.** This is the guard that does not exist, and its absence is
   the highest-risk item here: a scheduled job that cannot import
   something fails unattended.
3. **Delete `check2.py`.**
4. **Decide on WIT** — in the plan and made explicit, or out.
5. **Resolve the `src/tracking/` fork** — port the hardening to the
   tool, extract the one function, remove the copy. Largest, and the
   only one that needs real thought.

Nothing here is urgent in the way the 96%-full database is. This is
about the code, not the running system.
