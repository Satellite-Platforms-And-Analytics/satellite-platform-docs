# Operational audit — 2026-09-10

Companion to `CODE_AUDIT.md`. That document asks whether the code earns its
place; this one asks whether the system that runs it is actually doing what
the schedules claim. Everything below is measured, not assumed. Where a
check could not be run, that is stated rather than skipped.

---

## 1. Scheduled delivery: sub-daily crons run at a quarter of their rate

Seven days of `event=schedule` runs, measured as the gap between
consecutive runs rather than as a ratio of runs to cron slots. The ratio version of this table said `monitor_catalog`
delivered 57% of the time; it does not, it delivers 100%, and the ratio
was wrong because the workflow has only existed for four of the seven
days. Gaps do not have that failure mode.

| Workflow | Cron (UTC) | Nominal gap | Median gap | Worst gap | Runs / 7d |
|---|---|---|---|---|---|
| `propagate.yml` | `17 * * * *` | 1.0h | **3.76h** | 6.09h | 43 |
| `ingest_tle.yml` | `37 */2 * * *` | 2.0h | **4.58h** | 7.84h | 36 |
| `ingest_visibility.yml` | `43 6 * * *` | 24h | **24.06h** | 25.68h | 7 |
| `monitor_catalog.yml` | `19 7 * * *` | 24h | **24.19h** | 25.87h | 4 of 4 possible |
| `enrich_catalog.yml` | `55 6 * * *` | 24h | — | — | never yet, see §2 |

None of the missing runs are failures or cancellations — they were never
created. This is GitHub's documented best-effort scheduling on shared
runners, and `propagate.yml`'s own header already anticipated it ("the
2-hourly ingest_tle.yml was observed landing 8-12h apart on 2026-08-31").

The measurement sharpens that into a rule: **the drop rate scales with
requested frequency, and daily is the frequency where it disappears.**
Both daily jobs landed every single day they existed for, worst case 1.9
hours late. Both sub-daily jobs land at roughly a quarter to a half of
what they ask for. There is no partial credit in between — asking hourly
does not buy a job that runs every two hours instead, it buys one that
runs every four.

One thing the gap measurement hides, visible only once the slot is
compared against the landing time: **every daily job runs about five
hours after the minute its cron asks for.** `ingest_visibility` asks
06:43 and lands ~11:48; `monitor_catalog` asks 07:19 and lands ~12:15;
`enrich_catalog` asks 06:55 and landed 11:52. The gap between runs is a
clean 24h, so nothing is dropped - but the offset is real and consistent.
Anyone waiting for the monitor's issue at 07:19 UTC will conclude it
failed, four hours before it runs.

The practical cadences, as opposed to the configured ones:

- Propagation: median every **3.8 hours**, worst observed 6.1h — not hourly.
- TLE ingest: median every **4.6 hours**, worst observed 7.8h — not every 2.

### Does this break anything?

Checked against each consumer rather than assumed:

- **Visibility predictions.** The stated accuracy requirement is elements
  no more than 1–5 days old at the time of the prediction. A 4.7-hour
  ingest cadence is far inside that. **No impact.**
- **`orbital_positions` history.** 48h retention at ~4h resolution instead
  of 1h. Trend and coverage queries lose resolution, not coverage. Worth
  knowing before anyone reads that table as hourly.
- **New-object detection.** `monitor_catalog` has not missed a day since it
  was created. And it is protected even if it starts to:
  `check_new_objects.py` uses a rolling 30-day window, so a missed day is
  re-examined by the next run rather than lost. A gap would only matter if
  it exceeded 30 days.

**Recommendation: leave the crons alone.** Raising the frequency to
compensate makes the drop rate worse, not the delivery rate better. The
one change worth making is documentary — the hourly claim in
`propagate.yml`'s header and anywhere `orbital_positions` is described as
hourly should say "nominally hourly; observed ~4h".

---

## 2. `enrich_catalog` — first run confirmed

At the time of the audit it had zero runs, which looked alarming and was
not: the workflow was committed at **2026-09-10 02:37 UTC** (`f75f9c8`)
and its first scheduled firing was 06:55 UTC the same day, which had not
yet happened when the audit ran (04:05 UTC).

**Resolved.** It fired once, at **2026-09-10 11:52 UTC**, and succeeded —
its only run, so the schedule took on the first attempt. The five-hour
offset between the 06:55 slot and the 11:52 landing is the pattern
described in §1, not a problem with this workflow.

---

## 3. The monitor's filter works; its silence is still unproven

`report_events.py --only-newsworthy` was added so the monitor stops
opening an issue on days when nothing happened.

At the time of the audit it was untested, and the failure condition was
written down as: *if it opens an issue with 0 notable events, the fix
does not work.* It has now run, and the answer is neither of the two
outcomes that were anticipated.

**It opened issue #11 on 2026-09-10 — correctly.** The digest reads *"5
new catalogue event(s) in the last 24h, 1 notable"* and carries a **New
launches** section: launch `2026-203`, 6 PAYLOAD objects, launched
2026-09-05, first seen 2026-09-07. `new_launch` is not in `ROUTINE_TYPES`,
so `is_newsworthy()` returned True and the issue was right to exist. The
filter let through exactly what it is supposed to let through.

So the mechanism is verified in one direction only. Suppression — the
zero-byte `--out` file that makes the workflow's `[ -s digest.md ]` gate
fail, and no issue appear — has still never happened in production,
because there has not yet been a day with nothing but routine metrics on
it. Issues #7 through #10 all predate the fix; #11 postdates it and had
real news. **The next quiet day is the test**, and it is worth watching
for rather than assuming.

For the record, the earlier framing was wrong: "#11 exists" was treated
as the failure signal, when the failure signal was always "#11 exists
*and* reports nothing but routine metrics". Only the body distinguishes
them.

### A smaller thing the digest shows

Both routine sections list two near-identical bullets — *mean 5.7 days /
226 objects* beside *mean 5.8 days / 221 objects*, and *579 uncatalogued
(+2)* beside *577 (+1)*. That is not a bug: `event_key` for these is
`latency:{today}` and `uncatalogued:{today}`, one row per day, and a 24h
window straddles two of them. It reads as duplication to anyone skimming
the issue. Worth collapsing to the newest row per routine type if the
digest is ever cleaned up; not worth a change on its own.

---

## 4. Two Phase 2 pipelines were writing to the log and nobody was watching

`check_pipeline.py` ran clean — **VERDICT: ALIVE**, newest activity 1.5h
old, `tle_fetch` 3.7h, `propagation` 3.0h, `visibility` 16.3h, all ok, no
future-dated epochs, 18,069 satellites and 343,790 element sets.

But its liveness table listed three pipelines, and there are five.
`catalog_enrich` and `catalog_events` have been writing `ingestion_log`
rows since Phase 2 went in — `catalog_enrich` appears in the last five
entries, at 2026-09-10 02:40 with 17,488 records — and neither was in
`EXPECTED`, so neither was ever checked for staleness.

That is the exact failure mode the table was built for. Its own comment
says it: *"A job that stops is silent; the only evidence is a step that
should have appeared and did not."* An unlisted pipeline cannot produce
that evidence. Had `enrich_catalog.yml`'s schedule silently failed to
take, `check_pipeline.py` would have gone on printing ALIVE indefinitely.

Both are now listed at `(24, 48)` — the same cadence and tolerance as
`visibility`, and comfortably above the 25.9h worst-case gap measured for
daily jobs in §1. The thresholds in that table are now annotated with the
measurements behind them rather than left as round numbers.

---

## 5. `ingest_tle` failures: closed, with a cause

Six failed runs, 2026-09-02 23:44 UTC through 2026-09-04 11:38 UTC, all on
shas `1564f3fb` and `b7514357`, all at the same step — *Run fetcher.py
(write to database)*. Every run since has succeeded.

The logs are not retrievable (the Actions logs endpoint returns 403
without a token), so the cause is inferred rather than read. It fits
exactly: `7a1c1fb` — *"fix(writer): CelesTrak epochs are naive; stop
swallowing env import errors"* — landed **2026-09-04 15:01 UTC**, three
hours after the last failure and before the next success. A naive-datetime
write failure would surface precisely at that step and nowhere earlier.

Treat as resolved. If it recurs, get a token first — inference is not a
log.

### Confirmed from the data, 2026-09-11

The inference above turned out to be checkable without the logs. Row
counts per day in `tle_history`, surfaced by `archive_to_local.py
--status`:

| Date | Element sets ingested |
|---|---|
| 2026-09-01 | 22,950 |
| **2026-09-02** | **361** |
| **2026-09-03** | **7,806** |
| 2026-09-04 | 44,655 |
| 2026-09-05 | 42,280 |

Against a 20,000–47,000 daily norm, those two days are down roughly
**50,000 element sets** — and they are precisely the days the failing runs
span. The write step was failing, so nothing was written; the data says so
as plainly as a log would have.

**This gap is permanent.** CelesTrak serves current elements only, and
Space-Track's GP_History is one request per lifetime, so those sets cannot
be backfilled at any sensible cost. Anything that scores confidence
against historical elements will be thin across 2–3 September. That is
worth knowing now rather than meeting it later as an unexplained dip in a
model.

---

## 6. Two secret-exposure gaps found and closed

Both were latent, neither had fired, and one had already come within a
`git add -A` of firing.

**`satellite-platform-ingestion/.gitignore` named only three env files.**
`.env`, `.env.local`, `.env.*.local`. Every other suffix was un-ignored.
That is how `.env.bak-preconsolidation` — Space-Track password, N2YO key,
Supabase secret and service keys, `DATABASE_URL` — sat in the working tree
visible to `git add -A` on 2026-09-09. Replaced with `.env*` plus
`!.env.example`. Verified: `.env.bak`, `.env.save`,
`.env.bak-preconsolidation` now ignored; `.env.example` still tracked.

**`satellite-platform-infrastructure` had no `.gitignore` at all.** This
is the repository that applies schema migrations, so it is the repository
most likely to acquire a `DATABASE_URL` file. It also had `__pycache__/`
and `apply_migration.py.bak` sitting untracked. Created one covering
`.env*`, Python bytecode, and `*.bak`/`*.orig`.

**Audited history in all four repositories: no `.env` file has ever been
committed to any of them.** The gap was exposure, not disclosure.

---

## 7. What this audit could not check

The database-side checks (`check_pipeline.py`, `check_catalog.py`,
`check_bloat.py`) cannot run from a Claude session. Neither environment
reaches Supabase:

- The bridge shell on the workstation resolves `api.github.com` but not
  `aws-0-us-east-1.pooler.supabase.com`, and its HTTPS proxy returns 403
  for `*.supabase.co` — so the PostgREST fallback is closed too.
- The cloud container resolves the pooler host but has TCP 5432 blocked.

These have to be run from a normal terminal on the workstation. That is
not a defect; it is worth recording so the next audit does not spend the
time rediscovering it.

Still open from earlier work and unchanged by this audit:

- **Database at 481 MB of the 500 MB tier (96%).** Retention levers exist
  (visibility 3→1 day ≈129 MB, `tle_history` 14→7 ≈60 MB,
  `orbital_positions` 48→24h ≈19 MB) but each needs a prune *and* a
  `VACUUM FULL`, and deletion is not cheaply reversible. Deliberate
  decision required; not taken here.
- The `CODE_AUDIT.md` recommendations (requirements.txt, the `src/tracking/`
  fork, WIT coupling, `check2.py`).

---

## Verdict

Nothing is silently broken, and `check_pipeline.py` says ALIVE with every
listed pipeline inside tolerance.

Three findings needed fixing on the spot, and all three are fixed: the two
`.gitignore` gaps in §6, and the two unwatched Phase 2 pipelines in §4.
That last one is the most useful thing this audit turned up, because it
was a hole in the very instrument the others were being checked with — a
health check that reports ALIVE while ignoring two of the five things it
is supposed to watch is worse than no health check, since it is believed.

The one surprise that needs no fixing is §1: sub-daily schedules deliver
at roughly a quarter of their nominal rate, and every consumer that
depends on them tolerates it. Worth having established rather than hoped.

The one thing still unproven is the monitor's ability to stay quiet. Its
filter is now half-verified: on 2026-09-10 it correctly let a genuine new
launch through (§3). Suppression on a quiet day has not been observed
yet, because there has not been one.

Worth recording separately: that same issue is the first time the
new-launch detection has reported a launch in a digest — `2026-203`, six
payloads, first seen two days after launch. That was the capability the
whole Phase 2 monitoring effort was asked for, and it works end to end
without anyone running anything by hand.
