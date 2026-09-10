# satellite-platform-docs

Documentation for the **Satellite Platform & Analytics** project — a
satellite intelligence platform covering orbit and visibility, catalogue
intelligence, industry, funding, launch platforms and strategic analysis.

**Live:** https://satellite-platform-frontend.vercel.app

---

## Start here

| If you want to | Read |
|---|---|
| **Build this from nothing** | [`BUILD_GUIDE.md`](./BUILD_GUIDE.md) — ordered stages, exact commands, a verification at every step, and the fifteen traps that cost this project the most time |
| **Know what actually happened** | [`AS_BUILT.md`](./AS_BUILT.md) — planned vs actual with a verdict on each divergence, including the mistakes |
| **Understand a specific decision** | The master roadmap's Decision Log Index (Obsidian vault) |
| **Follow the history** | `08_Periodic/` in the vault — daily, weekly and monthly records from June 2026 |

Read `BUILD_GUIDE.md` for the recipe and `AS_BUILT.md` for the reasons.
The guide alone reads as though the order were obvious; it was not.

---

## Working rule: never run git through the remote bridge

**Do not run `git` — not even `git status` — against these repositories
from a Claude session's bridge shell.** Git takes `.git/index.lock` and
the bridge mount cannot unlink it, so every invocation strands a lock
file that only a human at the machine can remove. Even a folder-wide
delete grant does not reach inside `.git/`.

This has happened three times: 2026-09-04 twice, and again on 2026-09-05
running `git status` to build a commit list. It was written down after
the first two and repeated anyway, which is why it is here at the front
door rather than in the history.

If a lock is stranded:

```
del D:\Projects\Satellite-Platform\<repo>\.git\index.lock
```

Claude should report file state from `ls` and the filesystem, and hand
over git commands for a human to run.

---

## Checking CI status

GitHub renders pass/fail as an icon with no text equivalent, and the
badge is an SVG. Anything that reads the rendered page — including an
agent fetching it — will report a conclusion it cannot actually see. On
2026-09-05 that produced a confident "CI passed" for a run that had
failed; the API showed `failure` for the same run.

Ask the API, which returns the conclusion as literal text:

```powershell
$r = "https://api.github.com/repos/Satellite-Platforms-And-Analytics/satellite-platform-ingestion/actions/workflows/ci.yml/runs?per_page=8"
(Invoke-RestMethod $r).workflow_runs |
  Select-Object run_number, @{n='sha';e={$_.head_sha.Substring(0,7)}}, conclusion, display_title |
  Format-Table -AutoSize
```

Swap `ci.yml` for `monitor_catalog.yml`, `enrich_catalog.yml`,
`ingest_tle.yml`, `ingest_visibility.yml` or `propagate.yml` for the
others. Unauthenticated calls are rate-limited per IP (60/hour), which is
enough for this and is why a shared or proxied address may get a 403.

---

## The other repositories

| Repo | Contains |
|---|---|
| `satellite-platform-ingestion` | Python pipeline — TLE fetch, propagation, visibility, database writers, workflows, tests |
| `satellite-platform-infrastructure` | `schema/` and the migration runner |
| `satellite-platform-frontend` | Next.js globe, client-side SGP4 |
| `satellite-platform-docs` | this repository |

The canonical plan lives in the Obsidian vault, not here:
`Satellite_Platform_And_Analytics_Master_Roadmap.md`. Where this
repository and the roadmap disagree, **the roadmap wins** and the file
here is stale.

---

## Current state — 2026-09-04

Phases 0 and 1 complete.

| | |
|---|---|
| Catalogue | 18,052 objects, refreshed every 2 hours from CelesTrak |
| Positions | propagated hourly, 48-hour retention |
| Visibility | 3 sensors, computed daily |
| Database | 183 MB of a 500 MB tier, every high-volume table bounded |
| Tests | 58, with 54 running in CI on every push |
| Frontend | live, propagating 1,389 sampled objects in-browser |

Next: Phase 2 — catalogue intelligence (UCS, UNOOSA).

---

## `archive/`

Superseded planning documents, kept for reference. Nothing in there is
current — see [`archive/README.md`](./archive/README.md).
