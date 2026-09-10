# Closing the coverage gap

> Written **2026-09-10**. A decision brief, not a change. Nothing here is
> implemented.

## The gap

| | |
|---|---|
| SATCAT lists in orbit | **35,017** |
| This catalogue holds | **18,069** |
| Missing | **16,948** |

By type, on-orbit: 20,007 payloads, 12,529 debris, 2,426 rocket bodies,
55 unknown. The catalogue is payload-dominated because ingestion fetches
CelesTrak's `active` group plus `analyst` and three named debris events.
**The missing 16,948 are mostly debris and rocket bodies**, and they are
the reason `check_new_objects.py` catches new launches reliably and
misses most new debris.

## The answer that looks obvious, and is wrong

"Doubling the catalogue doubles the database." Working it through from
measured figures:

- `visibility_windows`: 221,788 rows/day for 18,044 objects across three
  sensors — 12.3 windows per object per day, at 215 bytes measured. At
  35,017 objects that is ~431,000 rows/day, ~93 MB/day, **278 MB** at the
  3-day retention against ~143 MB now.
- `tle_history` at 14 days would roughly double, ~101 → ~196 MB.
- `orbital_positions` at 48 hours would roughly double.

That lands near **490 MB against a 500 MB tier** — technically inside it,
with no margin, and it would make every future decision a fight for
space. On those numbers the sensible answer is "no".

**84% of on-orbit objects are LEO (period < 128 min)**, and LEO objects
cross a sensor's field of regard far more often than GEO ones, so the
visibility figure is the one most likely to be understated.

## Why it is wrong

Every expensive consumer already filters itself out.

| Consumer | Query |
|---|---|
| `propagator.load_tles` | `WHERE tle_line1 IS NOT NULL AND tle_line2 IS NOT NULL` |
| `visibility/compute.py` | imports the same `load_tles` |
| frontend `/api/tles` | `.not("tle_line1", "is", null)` |

So a satellite row **with no TLE lines** is invisible to propagation, to
visibility, and to the globe. It costs one row in `satellites` and
nothing in the three tables that actually consume the tier.

That is not a lucky accident — it is the filter added so the propagator
would not choke on rows the fetcher had not yet populated. It happens to
make this expansion nearly free.

## What it would actually cost

Insert the 16,948 on-orbit objects SATCAT knows about and this catalogue
does not, carrying name, designator and SATCAT attributes, with
`tle_line1`/`tle_line2` left NULL.

A satellites row without TLE lines is roughly 200–250 bytes — the two
69-character element-set lines are most of a populated row. With indexes,
**call it 8–12 MB**, against a database last measured at 183 MB.

Confirm before acting, rather than trusting the estimate:

```
python -m src.visibility --sizes
```

That reports the real per-table bytes and the current total.

## What changes, and what does not

**Detection becomes complete.** `check_new_objects.py` would see every
catalogued object in orbit, so a break-up producing debris would appear
as fragmentation instead of being invisible. That was the stated goal —
catching new launches *or any new man-made material* — and it is the half
that does not currently work.

**Tracking does not change.** Propagation, visibility windows and the
globe stay on the objects that have element sets. Nothing gets slower and
no workflow needs a new timeout.

**Enrichment already handles them.** `seed_satcat.py --apply` fills their
attributes on the next daily run, at no extra request cost — it fetches
the whole file anyway.

**The detector will describe its own expansion.** All 16,948 rows would
share one `created_at`, so `classify_arrival` would report them as an
`ingestion_change` — old launches, arriving together, on one day. That is
exactly right, and it is a good check that the classifier means what it
says.

## What it does not solve

These objects would have **no current elements**, so the platform would
know they exist without being able to say where they are. Propagating
them needs Space-Track's `gp` class — one request per hour, a different
policy class from the `gp_history` rule behind the 2026-09-05 incident,
and already implemented in `tle_bulk_seeder.snapshot_daily_gp()`.

That is a second, larger decision: it is the one that would double
`visibility_windows` and `orbital_positions`, and the 490 MB arithmetic
above is what it costs. Worth separating from this one.

## Recommendation

Do the cheap half. Insert the on-orbit objects without elements, get
complete detection coverage for single-digit megabytes, and leave
tracking scope alone.

Deliberately **not** included: the 35,104 objects SATCAT marks `IMP` —
impacted, already re-entered. They are history rather than orbit, they
would double the row count again, and nothing currently asks a question
that needs them. Phase 3's launch-rate analysis might; revisit then.

## Implementation sketch

`seed_satcat.py --insert-missing`, alongside the existing `--apply`:

- Restrict to `ORBIT_TYPE = 'ORB'`.
- Insert only where no satellite row exists, never touching held rows.
- `name` from `OBJECT_NAME` (the column is NOT NULL).
- `source = 'celestrak_satcat'`, full provenance as `--apply` writes.
- Leave `tle_line1`/`tle_line2` NULL — that is the whole mechanism.
- Report inserted vs skipped, and refuse to run without `--apply`
  having succeeded first, so attribution and existence land together.
