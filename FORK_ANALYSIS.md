# The Satellite Visibility Tool and `src/tracking/`

> Written **2026-09-05**, after `D:\Projects\Satellite Project` and
> `D:\Projects\Data` were connected to the platform work.

`satellite-platform-ingestion/src/tracking/` began as a copy of the
Satellite Visibility Tool at `D:\Projects\Satellite Project\Satellite
Visibility Tool`. Both have been edited since. This is what actually
diverged, which side is ahead, and the one finding that needs acting on
before anything is merged.

---

## 1. Two compliance ledgers, and neither knew about the other — FIXED 2026-09-05

This matters more than the code drift, and it is not visible from a file
listing.

`api_request_log.py` — byte-identical in both copies — exists because
Space-Track's limits *"apply across ALL usage of your account"*. Its own
docstring names the failure it prevents: two callers each see a stale
cache, each trigger a fetch, and two SATCAT queries go out within ten
minutes. That account has already been suspended once.

The two copies read that ledger from **different files in different
directories**:

| | Tool | `src/tracking/` |
|---|---|---|
| Root | hardcoded `BASE_DIR/data/` | `TLE_DATA_DIR` (env) |
| History cache | `tle_history_cache.db` | `tle_history_cache.sqlite3` |
| SATCAT cache | `satcat_cache.db` | `satcat_cache.sqlite3` |
| Request log | *(via budget db)* | `api_request_log.sqlite3` |
| Also | `gp_history_cache.db`, `catalog_cache.db`, `spacetrack_budget.db` | — |

Different directories **and** different extensions, so they cannot
collide even by accident. Both ledgers exist on disk today:

- `Satellite Visibility Tool\data\tle_history_cache.db` — **262 MB**,
  last written 2026-08-26
- `D:\Projects\Data\TLEs\api_request_log.sqlite3`,
  `satcat_cache.sqlite3`, `satellite_confidence.sqlite3`

**So the guard is real, well-designed, and running twice against one
account with two separate memories.** Whichever copy runs is protected;
the account is not. Nothing has gone wrong yet because the two have not
been run close together — that is luck, not design.

This is the recurring shape from `AS_BUILT.md` §6 in a new place: a
configuration naming something the world does not have (one ledger),
paired with a fallback that makes the absence look like success (each
copy's log looks complete to itself).

### Related, and also wrong today

`satellite-platform-ingestion\.env` sets

    TLE_DATA_DIR=C:\Users\toddl\OneDrive\Data Science Project\Data\TLEs

but the 20 GB of Space-Track bulk archives now live at
`D:\Projects\Data\TLEs`. The pipeline is pointed at the old location.
Whether that path still holds anything is worth checking before the
pointer is moved — if both exist, there is a third ledger.

**Recommendation:** one `TLE_DATA_DIR`, set in both `.env` files, with
one set of filenames. Consolidate before either copy makes another
Space-Track request.

### Resolved, 2026-09-05

Both copies now build every Space-Track path from one `TLE_DATA_DIR`,
set to `D:\Projects\Data\TLEs` in both `.env` files — deliberately not
the OneDrive mirror, because a live SQLite in a synced folder produces a
conflicted copy, which is two ledgers again.

The diagnosis turned out to be worse than "two ledgers". The tool's
`config.py` had the two history caches **crossed**:

- `TLE_HISTORY_CACHE_DB` pointed at `data/tle_history_cache.db`, a
  262 MB file containing `tle_cache.py`'s `tle_history` BLOB schema —
  not `tle_history_cache.py`'s `tle_elements`/`coverage`.
- `GP_HISTORY_CACHE_DB` pointed at a file that did not exist.
- `API_REQUEST_LOG_DB`, `TLE_DATA_DIR` and `SATELLITE_CONFIDENCE_DB`
  were not defined at all, so `seed_tle_history.py` raised
  `ImportError` on import. The tool's bulk-history path had never run.

That 262 MB file is the **gp_history 1/object/lifetime guard**, holding
15,838 objects already spent against the account. It was copied to
`TLE_DATA_DIR/gp_history_cache.sqlite3`; the originals are now
unreferenced and `check_ledger.py` names them until they are removed.

What changed:

| | |
|---|---|
| Tool `config.py` | `TLE_DATA_DIR` block replacing the `BASE_DIR/data` paths; `API_REQUEST_LOG_DB` and `SATELLITE_CONFIDENCE_DB` added; the two history caches un-crossed. Backup at `config.py.bak-preconsolidation` |
| `src/tracking/config.py` | `GP_HISTORY_CACHE_DB`, `CATALOG_CACHE_DB`, `SPACETRACK_BUDGET_DB` added so both copies name the same files |
| Both `.env` | `TLE_DATA_DIR=D:\Projects\Data\TLEs`. Backups at `.env.bak-preconsolidation` |
| New | `tests/test_config_paths.py` (11 tests), `check_ledger.py` |

All eight constants verified to resolve identically from both configs,
and all ten previously-failing tool modules now import.

**What the ledger showed once it could be read as one thing:** seven
SATCAT requests on 2026-08-06 against a documented 1/day limit, four
inside two seconds. `check_ledger.py` reports this and exits non-zero.

---

## 2. The code drift is directional, not ambiguous

The instinct — "the tool is the original, refresh the copy from it" —
would have deleted the account-hardening code. The two sides grew in
different directions, and each is ahead in its own.

### The tool is ahead on analyst-facing work

| File | Tool | `src/tracking/` | What only the tool has |
|---|---|---|---|
| `sensor_select.py` | 1,523 | 573 | user sensor registry, geocoding, timezone detection, multi-sensor prompts |
| `report_utils.py` | 1,459 | 367 | combined multi-sensor sheets, handoff sheets, pass-count summaries |
| `main.py` | 990 | 586 | multi-sensor runs, timezone handling, stacked views |
| `reports.py` | 957 | 530 | `generate_combined_report` and its styling |

Tool-only files, all analyst tooling or Space-Track budgeting:

| File | Lines | Purpose |
|---|---|---|
| `handoff.py` | 452 | handoff chains between sensors, great-circle bearing, chain confidence |
| `spacetrack_budget.py` | 291 | `SpaceTrackBudget`, `BudgetExceeded`, `can_afford`, `acquire` |
| `tle_cache.py` | 262 | `TLEHistoryCache` with `purge_stale`, `size_info` |
| `regenerate_reports.py` | 226 | rebuild reports without re-running the analysis |
| `excel_io.py` | 150 | `check_writable`, lock-file detection and advice |
| `cache_admin.py` | 64 | cache inspection |

`spacetrack_budget.py` is the notable one: it is a *third* rate-limit
mechanism, on the tool side only, and it belongs in the consolidation
decision above rather than being copied blindly.

### `src/tracking/` is ahead on API hardening

| File | Tool | `src/tracking/` | What only the pipeline has |
|---|---|---|---|
| `satellite_utils.py` | 1,119 | 1,499 | `SpaceTrackMalformedResponseError`, `ensure_spacetrack_session`, `spacetrack_session_is_valid`, `parse_gp_history_json_record` |
| `historical_accuracy.py` | 1,732 | 2,171 | `_cached_fetch`, `_ansi_ok` |
| `compare_n2yo.py` | 194 | 245 | the `transactionscount` abort documented in `API_USAGE_POLICY.md` |
| `main.py` | — | — | `_decide_online_or_offline` |

**Every one of those is a guard on an external API.** A refresh from the
tool would remove the malformed-response detection, the session
validation, and the N2YO transaction ceiling — the code written after
the incidents that motivated it.

### Identical in both (11 files)

`api_request_log.py`, `confidence_tabs_report.py`, `grouped_report.py`,
`log_utils.py`, `satcat_cache.py`, `satellite_confidence_db.py`,
`seed_tle_history.py`, `spacetrack_client.py`,
`spacetrack_policy_check.py`, `tle_bulk_seeder.py`, `trial_run.py`.

The compliance core is identical on both sides. Only its *storage
location* differs — which is exactly what makes the split ledger easy to
miss.

### Differ only inside function bodies

`catalog_diff.py`, `config.py`, `run_state.py` — same function set. The
`config.py` difference is the path divergence in §1 plus the tool's
combined-report settings.

---

## 3. What this means for merging

Do not merge wholesale in either direction. Concretely:

1. **Consolidate the ledgers first** (§1). Nothing else is safe to run
   twice until that is done.
2. **Keep `src/tracking/`'s versions** of `satellite_utils.py`,
   `historical_accuracy.py` and `compare_n2yo.py`. They contain guards
   the tool does not.
3. **Keep the tool's versions** of the reporting and sensor-selection
   modules. The pipeline does not use them; the analyst does.
4. **Decide deliberately** about `spacetrack_budget.py` and
   `tle_cache.py` — a third and fourth rate-limit mechanism should be
   adopted or dropped, not left ambient.

A single shared package is the eventual answer. It is not today's work,
and doing it before §1 would only give one wrong ledger a better name.
