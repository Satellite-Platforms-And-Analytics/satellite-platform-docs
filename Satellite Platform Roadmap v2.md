# 🛰️ Satellite Intelligence Platform — Master Roadmap

> **Status:** Phase 1 Starting
> **Created:** 2026-07-09
> **GitHub Org:** Satellite Platforms And Analytics
> **Solo developer:** Yes
> **Related:** [[Satellite Platform Architecture v2]] · [[Engineering OS Roadmap]]

---

## Phase Overview

| Phase | Name | Timeline | Status |
|-------|------|----------|--------|
| 0 | Foundation | Weeks 1-2 | ✅ Complete |
| 1 | Core Satellite Tracking MVP | Weeks 2-4 | 🔄 Starting |
| 2 | Ownership & Country Intelligence | Weeks 4-6 | 🔲 |
| 3 | Payload Intelligence & Analytics | Weeks 7-9 | 🔲 |
| 4 | User Platform | Weeks 10-11 | 🔲 |
| 5 | Production Scale | Weeks 12-13 | 🔲 |
| 6 | AI Intelligence Layer | Future | 🔲 |

---

## Phase 0 — Foundation ✅ COMPLETE

### What Was Built
- ✅ Engineering OS (Obsidian vault, Git backup, templates)
- ✅ Python environments: satellite-base + satellite-ml
- ✅ GPU verified: RTX 3080 Laptop, CUDA 12.4, PyTorch 2.6.0
- ✅ QGIS 3.44 LTR installed and configured
- ✅ Storage structure: D:/SatelliteData, D:/GIS, D:/Models
- ✅ Data source accounts: Copernicus, NASA Earthdata, USGS
- ✅ First Sentinel-2 scene downloaded and verified
- ✅ Ingestion pipeline v1 (ingest.py) — JP2→GeoTIFF, SQLite, CSV
- ✅ GitHub org: Satellite Platforms And Analytics
- ✅ Repos scaffolded: frontend, ingestion, infrastructure, docs

### GitHub Repos State
- satellite-platforms-frontend — empty scaffold
- satellite-platform-ingestion — empty scaffold
- satellite-platform-infrastructure — empty scaffold
- satellite-platform-docs — empty scaffold

---

## Phase 1 — Core Satellite Tracking MVP

**Goal:** Get satellites moving on a 3D globe with real TLE data
**Outcome:** Live satellite visualization demoable in browser
**Repos:** frontend + ingestion + infrastructure

### Epic 1 — Repository Foundation (satellite-platform-infrastructure)

#### Database Schema
- [ ] Write `schema.sql` — core tables:
  ```sql
  satellites, orbital_positions, tle_history,
  ingestion_log, countries, operators, launches
  ```
- [ ] Provision Neon.tech PostgreSQL (free tier)
- [ ] Run schema, verify with psql
- [ ] Document schema in [[AD-010 Database Hosting]]

#### CI/CD (satellite-platforms-frontend + ingestion)
- [ ] `.github/workflows/ci.yml` for frontend: lint, typecheck, build
- [ ] `.github/workflows/ci.yml` for ingestion: ruff, mypy, pytest
- [ ] Connect Vercel to satellite-platforms-frontend
- [ ] Add secrets: DATABASE_URL, VERCEL_TOKEN, CESIUM_TOKEN
- [ ] Verify: git push → auto deploy

### Epic 2 — TLE Ingestion (satellite-platform-ingestion)

#### Fetch TLEs
- [ ] `src/fetcher.py` — pull from CelesTrak groups:
  - active, starlink, gps-ops, glo-ops, geo, stations
- [ ] Parse 3LE format
- [ ] Classify orbit regime from mean motion:
  - LEO < 2000km, MEO 2000-35786km, GEO ~35786km, HEO elliptical
- [ ] Upsert to `satellites` table
- [ ] Archive to `tle_history`
- [ ] Document: [[AD-011 TLE Primary Source]]

#### Orbit Propagation
- [ ] `src/propagator.py`
  - `propagate_single()` → lat/lon/alt/velocity using sgp4 + skyfield
  - `propagate_batch()` — skip TLEs older than 14 days
  - `ground_track()` — one orbital period for path rendering
- [ ] `src/db_writer.py` — bulk insert orbital_positions
- [ ] Prune job — keep only 48h of positions
- [ ] Schedule: TLE refresh every 2hrs, positions every 5min

#### Connect Local ingest.py
- [ ] Port ingest.py into satellite-platform-ingestion/src/imagery/
- [ ] Add imagery_scenes table to schema.sql
- [ ] Connect local D:/SatelliteData/ pipeline to PostgreSQL
- [ ] Replace SQLite catalog with PostgreSQL

### Epic 3 — API Layer (satellite-platforms-frontend)

#### Next.js API Routes
- [ ] `GET /api/positions` — latest positions, filter by regime/country/type
- [ ] `GET /api/stats` — aggregate counts per regime, payload/debris, countries
- [ ] `GET /api/search?q=` — satellite name search
- [ ] `GET /api/satellite/:id` — full detail + 20 recent positions
- [ ] `GET /api/track/:id` — 90-minute ground track
- [ ] Drizzle ORM schema mirroring infrastructure schema.sql

### Epic 4 — CesiumJS Globe (satellite-platforms-frontend)

#### 3D Visualization
- [ ] Register at cesium.com/ion → add NEXT_PUBLIC_CESIUM_TOKEN
- [ ] `src/components/globe/CesiumViewer.tsx` — Earth, atmosphere, terrain
- [ ] `src/components/globe/SatelliteLayer.tsx` — points from /api/positions
  - Color by regime: LEO=blue, MEO=green, GEO=orange, HEO=red
- [ ] `src/components/globe/OrbitLayer.tsx` — ground track via /api/track/:id
- [ ] Click satellite → side panel:
  - NORAD ID, name, country, altitude, velocity, regime
- [ ] Search bar → fly to satellite on globe

#### Basic Search UI
- [ ] `src/components/search/SearchBar.tsx`
- [ ] `src/components/search/SearchResults.tsx`
- [ ] Keyboard navigation, real-time results

### Phase 1 Definition of Done
- [ ] TLEs ingesting automatically every 2 hours
- [ ] Positions updating every 5 minutes
- [ ] Globe shows 1000+ satellites in real time
- [ ] Click any satellite → see name, country, altitude, velocity
- [ ] Search finds satellites by name or NORAD ID
- [ ] CI/CD: push to main → auto deploy to Vercel

---

## Phase 2 — Ownership & Country Intelligence

**Goal:** Enrich satellite data with operator, country, and company intelligence
**Outcome:** Every satellite linked to an operator and country

### Epics
- [ ] Country attribution engine
- [ ] Operator/company database
- [ ] Payload classification
- [ ] Historical launch tracking
- [ ] FastAPI backend service (replaces some Next.js routes)
- [ ] Data APIs for external consumers
- [ ] Manual review queue for ambiguous attributions

### Key Data Sources
- OpenCorporates REST API — company data
- Launch Library 2 API — launch history
- FCC/ITU databases — regulatory/spectrum
- OpenSatelliteData — additional metadata

---

## Phase 3 — Payload Intelligence & Analytics

**Goal:** Analytics dashboards, custom reports, historical analysis
**Outcome:** Interactive D3.js dashboards with drill-down

### Epics
- [ ] D3.js analytics dashboard framework
- [ ] Country statistics (satellites per country, trends)
- [ ] Operator statistics (fleet size, growth)
- [ ] Launch trends (monthly, yearly, by country)
- [ ] Payload trends (type classification over time)
- [ ] Regime distribution charts
- [ ] Conjunction/collision risk display
- [ ] Custom report builder
- [ ] CSV/PDF/Excel export
- [ ] Satellite detail pages (full history, payload info)

---

## Phase 4 — User Platform

**Goal:** Multi-user platform with saved state and alerts
**Outcome:** Users can create accounts, save views, receive alerts

### Epics
- [ ] Authentication (NextAuth.js / Auth0 / Clerk)
- [ ] RBAC: admin, analyst, viewer, guest roles
- [ ] Saved searches and satellite watchlists
- [ ] Dashboard layout persistence per user
- [ ] Alert system: new launches, country activity, conjunction warnings
- [ ] Email notifications (Resend/SendGrid)
- [ ] In-app notification center
- [ ] Admin panel

---

## Phase 5 — Production Scale

**Goal:** Enterprise-grade infrastructure, high availability
**Trigger:** Real usage data justifies operational overhead

### Epics
- [ ] AWS VPC, security groups, subnets
- [ ] RDS PostgreSQL with read replicas
- [ ] TimescaleDB for orbital_positions time series
- [ ] Redis — API caching, sessions, rate limiting
- [ ] Kafka/MSK — event bus for satellite and launch updates
- [ ] OpenSearch — full-text + faceted satellite search
- [ ] Docker + EKS — containerized microservices
- [ ] Prometheus + Grafana observability
- [ ] CloudWatch AWS monitoring
- [ ] Terraform IaC for all infrastructure
- [ ] Load testing, disaster recovery drills

---

## Phase 6 — AI Intelligence Layer

**Goal:** Natural language queries, trend forecasting, strategic reports
**Uses:** satellite-ml environment, PyTorch, torchgeo

### Epics
- [ ] RAG knowledge base — satellite + launch + payload data
- [ ] PDF ingestion pipeline — research papers, reports
- [ ] Embedding pipeline (OpenAI/Voyage embeddings)
- [ ] OpenSearch vector index
- [ ] Natural language query interface
  - "Show all Chinese imaging satellites launched since 2022"
  - Response: D3 chart + CesiumJS map + satellite list
- [ ] Forecasting models (Prophet → LSTM → Transformer)
  - Launch frequency, constellation growth, operator trends
- [ ] Automated strategic intelligence reports
- [ ] Imagery ML integration (torchgeo models on Sentinel-2)
  - Land cover, change detection, object detection

---

## Current Sprint — Phase 1 Start

### Immediate Next Steps (This Week)

**Day 1 — Infrastructure**
1. Clone satellite-platform-infrastructure locally
2. Write schema.sql with core Phase 1 tables
3. Provision Neon.tech PostgreSQL
4. Run schema, verify connection

**Day 2 — Ingestion Setup**
1. Clone satellite-platform-ingestion locally
2. Set up Python environment (link to satellite-base conda)
3. Write fetcher.py for CelesTrak TLE pull
4. Test: fetch → parse → print first 10 satellites

**Day 3 — Frontend Scaffold**
1. Clone satellite-platforms-frontend locally
2. Run: `npx create-next-app@latest . --typescript --tailwind --eslint --app`
3. Install: d3, cesium, zustand, swr
4. Connect to Vercel
5. Verify: push to main → live on Vercel URL

**Day 4 — First API Route**
1. Write `/api/positions` stub
2. Connect to Neon PostgreSQL
3. Return test data
4. Verify from browser

**Day 5 — First Globe**
1. CesiumViewer.tsx — bare Earth
2. SatelliteLayer.tsx — hardcoded test points
3. Push to Vercel — first demoable result

---

## Risk Register

| Risk | Phase | Mitigation |
|------|-------|------------|
| CelesTrak rate limiting | 1 | Cache last-good TLE; Space-Track backup |
| Cesium Ion free tier limits | 1 | Start with D3 geoOrthographic; add Cesium later |
| orbital_positions grows unbounded | 1 | 48h prune job from day one |
| Vercel/Neon free tier limits | 0-4 | Monitor usage; budget for paid tiers |
| JP2 driver issue in satellite-base | Local | Workaround: QGIS gdal_translate (proven) |
| Solo developer burnout | All | Strict phase gating; Definition of Done before next phase |
| Scope creep to Phase 5/6 too early | All | Gate each phase on Definition of Done |
| LLM hallucination in RAG | 6 | Always show underlying data alongside LLM summary |

---

## Related Notes
- [[Satellite Platform Architecture v2]]
- [[Engineering OS Roadmap]]
- [[Satellite Platform MOC]]
- [[ENB-001 Platform Setup]]
- [[ENB-002 Ingestion Pipeline v1]]
- [[Conda Environments]]
- [[Satellite Data Sources]]
