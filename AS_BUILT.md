# As Built vs As Designed

Where this project diverged from its own plan, why, and whether the
divergence was right.

> Last updated **2026-09-04**. Covers Stages 0–5 (built). Update when a
> stage completes and its reality is known.
>
> [`BUILD_GUIDE.md`](./BUILD_GUIDE.md) is the recipe — what you *should*
> do. This file is the honest account of what *was* done, including the
> parts that were wrong. Both are needed: the guide alone reads as though
> it were obvious, and it was not.

**Verdict key:** ✅ right call · ⚠️ worked, but the reasoning was luck ·
❌ mistake · 🔲 still open

---

## 1. Architecture

| Planned | Actually built | Verdict |
|---|---|---|
| Neon.tech Postgres | **Supabase** | ✅ The free tier bundles RLS, auth, real-time, storage and pgvector — all needed by Phases 4–6. Changing later would have been expensive |
| Monorepo | **Four repositories** | ✅ Different deploy targets, different dependency graphs. The frontend never needs GDAL |
| FastAPI backend from Phase 2 | **Deferred indefinitely** | ✅ Supabase's REST layer plus one Next.js route covers the read path. Revisit when attribution matching needs real server logic |
| OpenSearch at Phase 5 | **Postgres FTS** | ✅ A GIN index on `to_tsvector` is sufficient well past Phase 3 |
| Fixed-width TLE parsing | **OMM JSON** | ✅ Forced, and correctly: CelesTrak exhausted 5-digit catalogue numbers on 2026-07-12. Fixed 07-10, two days ahead |
| CesiumJS globe | **d3-geo on canvas** | ✅ Cesium is a WebGL globe with a large bundle and a token requirement, to draw 1,500 dots. Remains an additive upgrade |
| `/api/positions` reading `orbital_positions` | **`/api/tles`, propagated in-browser** | ✅ Measured: server-side 5-minute propagation is 2.3 M rows/day (~the whole tier) and 6.5× the CI budget |

**Nothing in this table was wrong.** The architecture held. Everything that
went wrong was operational.

---

## 2. What was built in the wrong order

| Should have been | What happened | Cost |
|---|---|---|
| Connect the deploy target once there is an app | **Vercel project created 2026-06-11 against an empty repo** | ❌ Framework detected as "Other". Every successful build discarded for **three months**. Diagnosed 09-04 |
| Retention policy written with each unbounded table's writer | `orbital_positions` and `visibility_windows` got one; **`tle_history` did not** | ❌ Reached 514.8 MB of a 597 MB database against a 500 MB tier |
| Every job logs a step from its first run | `fetcher.py` did; **propagator and visibility did not** | ❌ Both ran unattended for days with no trace. A failure in either would have been invisible |
| CI from the first test | 58 tests accumulated with **nothing running them but a human** | ❌ Three real defects were sitting in the repo, found the moment CI existed |
| Batch writes from the start | Per-row `text()` + `executemany` | ❌ Five days of half the TLE fetches silently discarded |

**The pattern:** every one of these is a step that *would have been cheap
at the time* and became expensive precisely because the project went
quiet. None was a hard problem.

---

## 3. Decisions that look wrong and were not

| Looks like | Actually | Verdict |
|---|---|---|
| Two propagation implementations (sgp4 in `propagation/`, skyfield in `visibility/`) | Deliberate. The 2-hourly job must install slim and not download a timescale at runtime; the daily job must match the interactive tool's numbers exactly | ✅ Documented in both modules |
| `main.py` left interactive instead of refactored | Building `visibility/compute.py` alongside it was cheaper and safer than making one 586-line script serve an analyst and a cron | ✅ The analyst tool still works, untouched |
| `upsert_imagery_scene` still per-row | One scene per call. Batching it would be noise | ✅ Comment in the code says not to "fix" it |
| `validate_taxonomy()` warns rather than asserts | A hard assert would kill the 2-hourly ingestion over a data-quality issue | ✅ Non-zero exit in CI, warning at runtime |
| Hourly propagation, not 5-minutely | 5-minute resolution costs the entire tier and 6.5× the CI budget, and shared runners do not fire that often anyway | ✅ The globe propagates client-side instead |

---

## 4. Where the reasoning was luck, not judgement

| What | Why it worked | Why that is uncomfortable |
|---|---|---|
| CelesTrak 6-digit fix landed 07-10, deadline 07-12 | ⚠️ Correct fix, right analysis | Two days of margin on the flagged critical risk of the phase, and then nobody verified it had worked for seven weeks |
| `.env` fallbacks "worked" | ⚠️ Right by accident | They were right only because the working directory happened to be the repo root. `python src/tle/fetcher.py` broke them |
| The globe looked correct on first render | ⚠️ It rendered land, graticule and hundreds of plausible orbits | It was showing the 1,000 *oldest* objects — no Starlink, no ISS, unpopulated GEO belt. Nothing about the picture was wrong; the sample was |

---

## 5. Bugs introduced by fixes

Worth its own section, because it is the least intuitive category.

| The fix | What it broke | Days |
|---|---|---|
| Future-epoch guard (09-01) — reject element sets dated ahead | Compared a **naive** datetime against an **aware** horizon. CelesTrak's OMM `EPOCH` carries no timezone designator. **Every TLE write failed** | 3 |
| Millisecond rounding to dedupe near-identical epochs | Failed on the exact case that motivated it — 376672 and 375808 µs straddle a millisecond boundary and round *apart* | 0 (caught by test) |
| `pytest.ini` scoping collection to `tests/` | Correct, but the same commit's bare `pytest` invocation could not resolve `src` — passed locally, failed in CI | 0 (caught by CI) |

**The first one is the cautionary tale.** The guard was correct in intent
and prevented real junk. It cost three days of ingestion because of a type
mismatch in the feed's timestamps, and it hid because `upsert_satellites`
runs *before* `insert_tle_history` — so the obvious health signal,
`satellites.last_updated`, stayed current throughout.

**Rule derived:** a guard added to a hot write path needs a test built
from **real feed data**, not a constructed example. The test that should
have caught this existed and used `'…376672Z'` — a trailing `Z` that the
feed does not send. The test encoded the assumption rather than the
observation, so test and code agreed with each other and both were wrong.

---

## 6. The recurring shape

Seven instances of one bug, across unrelated subsystems:

| # | Where | The swallowed failure |
|---|---|---|
| 1 | WIT taxonomy | 17 of 18 configured subdomains did not exist; broad fallbacks filled the gap |
| 2 | `.env` loading | Right by luck; the fallback masked a wrong path |
| 3 | CelesTrak groups | `noaa` / `debris` do not exist; HTTP 200 + `Invalid query` body |
| 4 | `check_pipeline.py` | Wrong column name poisoned the transaction, reported as healthy |
| 5 | Sensor registry | `SENSOR_PROFILES` and the `sensors` table drifted; neither noticed |
| 6 | `except ImportError: pass` | Hid the one import failure that mattered |
| 7 | `pytest.skip` on a missing schema | The only schema-drift guard, skipping inside a green build |
| 8 | `apply_migration.py`'s preview | Split SQL on every `;`, including inside string literals and `$$ … $$`. Showed `prune_old_positions()` as five invalid fragments |

**Common form:** configuration naming something the world does not have,
paired with a fallback that makes the absence look like success.

Number 8 is a variant worth naming separately: **the safety feature whose
output looks right and is not.** The runner's own docstring says "a
migration you cannot read before it executes is one you are trusting
rather than reviewing" — and then printed something other than what it
ran. Nothing broke, because execution sends the whole file in one
transaction and never touches the split. But the preview *is* the review
step, so for `001_core_schema.sql` the review has been reading five
invalid fragments in place of a plpgsql function since July.

Fixed with a quote-, comment- and dollar-quote-aware splitter plus
`python apply_migration.py --self-test`, which needs no database.

**Remedy, every time:** make the assumption checkable and fail loudly.
`--check-groups`, `validate_taxonomy()`, `REQUIRE_SCHEMA=1`, the arity
guard in `_bulk_upsert`, the error-rate ceiling in `compute_visibility`.

---

## 7. What the documentation got wrong

| | |
|---|---|
| **Drifted in both directions** | The roadmap understated finished work (all of `writer.py`, unchecked) *and* overstated shipped work (frontend recorded as "deployed to Vercel"; it held three files) |
| **Four competing roadmaps** | Resolved by naming one canonical and archiving the rest |
| **Two phase numberings** | July's notes call environment setup "Phase 1"; the current scheme calls the pipeline Phase 1. Reconciled by a mapping table, still the most confusing thing in the old notes |
| **Seven weeks of silence** | No note saying "pausing here". Work stopped mid-thought with 23 KB uncommitted |

**Rule derived:** checking boxes at the end of a session is cheap;
reconstructing seven weeks later is not. The handoff to your future self
is ten minutes and it is the difference between resuming and re-deriving.

---

## 7b. Data that is present but not usable

Worth its own note, because a populated column reads as a usable one.

**`rcs_size` is a legacy field.** SATCAT publishes radar cross section for
87–91% of on-orbit payloads launched between the 1960s and 2000s, 53.8%
of the 2010s, and **0% of the 2020s** — not one of 15,569. Starlink,
OneWeb and the Chinese megaconstellations are all zero.

The catalogue therefore enriched to 9.6% `rcs_size` coverage, and that
number is correct rather than a failure: it skews to recent launches, and
recent launches have no RCS. No alternative source fixes this; the field
stopped being published.

**Consequence:** any grouping, filtering or detection model keyed on
`rcs_size` is silently a statement about pre-2020 objects. If sensor
detection modelling in the Visibility Tool uses size, it has no data for
the majority of what is currently in orbit.

---

## 8. Still open

| | Since |
|---|---|
| 🔲 PAVE PAWS in `SENSOR_PROFILES` but not the `sensors` table | 09-01 |
| ✅ `test_writer_columns.py` now reads `ADD COLUMN` from every migration — caught `owner_code` on sight | 09-05 |
| 🔲 `check_pipeline.py` does not exit non-zero on a stale pipeline, so it cannot be a canary | 09-04 |
| 🔲 `visibility_windows` retention at 3 days; budget now allows 5 | 09-04 |
| 🔲 `_to_delete/` — 1.5 GB awaiting review | 08-30 |
| 🔲 Clicking a satellite should link to the analysis domain (AD-034) | 09-04 |
| 🔲 `check2.py` at the repo root — unexplained, undocumented | unknown |
| 🔲 `insert_tle_history` uses `ON CONFLICT DO NOTHING` but `_bulk_upsert` returns rows *submitted*, so every `ingestion_log` "written" figure for tle_history counts rows the database silently discarded. Fixable now that `count_affected=True` exists | 09-05 |

---

## 9. If you were starting again

In priority order, the things that would have saved the most time:

1. **Log a step from every job's first run.** Absence of a step is the
   only signal a killed process leaves.
2. **Batch writes from the first writer.** Not an optimisation — a
   correctness issue under a timeout.
3. **Write retention with the writer**, for every unbounded table.
4. **CI from the first test.**
5. **Do not connect a deploy target to an empty repo.**
6. **Verify each step with something that would fail if it silently
   didn't work.** Row counts, not exit codes.
7. **When you stop, hand off to yourself.** Commit, update the status
   block, write down what is next.

Nothing on that list is sophisticated. All seven are cheap on the day and
expensive at a distance, which is exactly why they get skipped.

---

## Related
- [`BUILD_GUIDE.md`](./BUILD_GUIDE.md)
- `Satellite_Platform_And_Analytics_Master_Roadmap.md` (vault) — decision
  log and risk register
- `08_Periodic/` (vault) — the week-by-week history
