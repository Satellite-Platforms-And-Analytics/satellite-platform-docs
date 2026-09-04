# 🛰️ Satellite Intelligence Platform — Master Architecture

> **Status:** Active Development — Phase 0 Complete, Phase 1 Starting
> **Created:** 2026-07-09
> **Solo developer:** Yes
> **Target:** Personal research → commercial product
> **Location:** `02_Projects/Satellite-Platform/`
> **GitHub Org:** Satellite Platforms And Analytics

---

## 1. Vision

Build a real-time satellite intelligence and analytics platform that:
- Tracks every active satellite on a live 3D globe
- Ingests multi-source data (TLE, imagery, metadata, news, regulatory)
- Provides analytics dashboards and custom reports
- Exposes AI-powered natural language query interface
- Scales from personal research tool to commercial SaaS

---

## 2. Repository Structure

**GitHub Org:** `Satellite Platforms And Analytics`

```
satellite-platforms-frontend/      # Next.js + TypeScript + CesiumJS + D3.js
satellite-platform-ingestion/      # Python — TLE, imagery, multi-source data
satellite-platform-infrastructure/ # PostgreSQL schema, Docker, Terraform, CI/CD
satellite-platform-docs/           # Architecture decisions, runbooks, API docs
```

**Local mirrors:**
```
D:/Projects/Satellite-Platform/
├── frontend/          → satellite-platforms-frontend
├── ingestion/         → satellite-platform-ingestion
├── infrastructure/    → satellite-platform-infrastructure
└── docs/              → satellite-platform-docs
```

---

## 3. Full System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      USER & ACCESS LAYER                            │
│  Web Users · Dashboard Users · API Clients · Automated Jobs        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                    PRESENTATION LAYER                               │
│                                                                     │
│   Wix Marketing Site          Next.js Dashboard (Vercel)           │
│   (www.yoursite.com)          (dashboard.yoursite.com)             │
│                               ┌─────────────────────────┐          │
│                               │ 3D Globe   (CesiumJS)   │          │
│                               │ Analytics  (D3.js)      │          │
│                               │ Timeline   (D3.js)      │          │
│                               │ Search     (Global)     │          │
│                               │ Reports    (PDF/Export) │          │
│                               │ Admin      (Panel)      │          │
│                               └─────────────────────────┘          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│              APPLICATION & API LAYER                                │
│                                                                     │
│         API Gateway / Edge Functions (Vercel Edge)                 │
│    Rate Limiting · Auth · Routing · Caching · Security             │
│                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │   Auth   │ │   API    │ │Real-time │ │Visualize │             │
│  │ Service  │ │ Services │ │ Service  │ │ Service  │             │
│  │NextAuth  │ │Satellites│ │WebSocket │ │ CesiumJS │             │
│  │OAuth/SSO │ │Orbits    │ │Live Pos  │ │ D3.js    │             │
│  │RBAC      │ │Launches  │ │Alerts    │ │ Charts   │             │
│  └──────────┘ │Payloads  │ │Conj.Warn │ └──────────┘             │
│               │Companies │ └──────────┘                           │
│               │Countries │                                         │
│               └──────────┘                                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                    DATA & STORAGE LAYER                             │
│                                                                     │
│  PostgreSQL        Redis          Object Storage    Search Engine  │
│  (Primary DB)      (Cache)        (S3/R2)          (OpenSearch)   │
│  · Satellites      · Route cache  · Images/Thumbs  · Full text    │
│  · TLE History     · API cache    · Documents      · Autocomplete │
│  · Launches        · Sessions     · TLE Archives   · Faceted      │
│  · Payloads        · Rate limits  · Satellite imgs                 │
│  · Companies                      · Backups                        │
│  · Countries                                                        │
│  · Ground Stations                                                  │
│                                                                     │
│  TimescaleDB (time series)    Message Queue (RabbitMQ/SQS)        │
│  · Orbital positions          · Ingestion jobs                     │
│  · Telemetry data             · Processing jobs                    │
│  · Conjunction events         · Email jobs                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│              DATA INGESTION & EXTERNAL SOURCES LAYER               │
│                                                                     │
│  TLE/Orbit     Space/Govt    Regulatory    Commercial    News &    │
│  Data          APIs          & Spectrum    & Company     Intel     │
│  · CelesTrak   · NASA APIs   · FCC         · Company     · News    │
│  · Space-Track · NOAA APIs   · ITU         · Operators   · OSINT   │
│  · N2YO        · ESA APIs    · USSF/SDA    · Payloads    · Reports │
│  · Other feeds · USSF data                · Imagery               │
│                                                                     │
│  Satellite Imagery (NEW — from local work)                         │
│  · Copernicus/Sentinel-2  · NASA Earthdata  · USGS EarthExplorer  │
│  · Processing: D:/SatelliteData/ via ingest.py                    │
│                                                                     │
│         ┌─────────────────────────────────────────┐               │
│         │     INGESTION PIPELINE (Python)          │               │
│         │  Fetch → Validate → Normalize →          │               │
│         │  Enrich → Transform → Deduplicate        │               │
│         │  → Schedule/Orchestrate                  │               │
│         └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              AI / INTELLIGENCE LAYER (Phase 6)                     │
│                                                                     │
│  RAG Knowledge Base    LLM Assistant    Trend Forecasting          │
│  Anomaly Detection     Strategic Intel  NLP on Publications        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              INFRASTRUCTURE & DEVOPS                                │
│                                                                     │
│  GitHub Actions CI/CD    Docker Containers    AWS (Phase 5)        │
│  Prometheus/Grafana      ELK Stack logging    Secrets Manager      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Tech Stack

### Frontend (satellite-platforms-frontend)
| Technology | Purpose |
|-----------|---------|
| Next.js 14 + TypeScript | Framework |
| CesiumJS | 3D globe, satellite visualization |
| D3.js | Charts, analytics, timelines |
| Tailwind CSS | Styling (dark space palette) |
| Zustand | State management |
| SWR | Data fetching |
| NextAuth.js | Authentication |

### Backend Ingestion (satellite-platform-ingestion)
| Technology | Purpose |
|-----------|---------|
| Python 3.11 | Primary language |
| sgp4 + skyfield | Orbit propagation |
| rasterio + geopandas | Satellite imagery processing |
| requests + aiohttp | API connectors |
| psycopg2 | PostgreSQL connection |
| SQLAlchemy | ORM |
| FastAPI | API layer (Phase 2+) |
| APScheduler | Job scheduling |
| torchgeo + PyTorch | ML on imagery (Phase 4+) |

### Data Layer
| Technology | Purpose | Phase |
|-----------|---------|-------|
| PostgreSQL (Neon → RDS) | Primary database | 0 |
| TimescaleDB | Orbital positions time series | 5 |
| Redis | Caching, sessions, rate limiting | 5 |
| S3/R2 | Object storage, imagery archive | 5 |
| OpenSearch/Elasticsearch | Full-text search | 5 |
| RabbitMQ/SQS | Message queue | 5 |

### Infrastructure (satellite-platform-infrastructure)
| Technology | Purpose | Phase |
|-----------|---------|-------|
| Vercel | Frontend hosting | 0 |
| Neon.tech | PostgreSQL (free tier) | 0 |
| GitHub Actions | CI/CD | 0 |
| Docker | Containerization | 5 |
| AWS EKS | Orchestration | 5 |
| Terraform | Infrastructure as code | 5 |

---

## 5. Data Sources

### Satellite Tracking (Phase 1 priority)
| Source | Data | Access | Priority |
|--------|------|--------|---------|
| CelesTrak | TLE data — all active satellites | Free, no auth | P0 |
| Space-Track | Official DoD TLE data | Free, registration | P1 |
| N2YO | TLE + pass predictions | Free API | P2 |
| SatNOGS | Satellite metadata | Free | P2 |

### Satellite Imagery (local pipeline — built)
| Source | Data | Access | Status |
|--------|------|--------|--------|
| Copernicus/Sentinel-2 | 10m multispectral | Free, account | ✅ Working |
| NASA Earthdata | Landsat, MODIS, DEM | Free, account | ✅ Account created |
| USGS EarthExplorer | Landsat, historical | Free, account | ✅ Account created |

### Intelligence Data (Phase 2+)
| Source | Data | Access |
|--------|------|--------|
| OpenCorporates | Company data | REST API |
| FCC/ITU | Regulatory/spectrum | Public APIs |
| Launch Library | Launch data | Free API |
| News APIs | Space news | Various |
| arXiv | Research papers | Free |

---

## 6. Local Development Environment

### Python Environments (conda)
| Environment | Purpose | Status |
|-------------|---------|--------|
| satellite-base | Imagery processing, geospatial | ✅ Complete |
| satellite-ml | ML training, torchgeo, PyTorch CUDA | ✅ Complete |

### Local Data Infrastructure
```
D:/SatelliteData/
├── raw/sentinel-2/     ← original .SAFE archives
├── staging/            ← drop zone for new data
├── processed/          ← GeoTIFF analysis-ready files
└── exports/            ← output products

D:/GIS/                 ← QGIS projects and vector data
D:/Models/              ← trained ML model weights
D:/Databases/
├── satellite_platform.db  ← SQLite metadata catalog
└── ingestion_log.csv      ← pipeline audit log
```

### Local Pipeline (built)
- `ingest.py` — Sentinel-2 .SAFE → GeoTIFF conversion + SQLite indexing
- Processes 14 bands per scene in ~110 seconds
- Verified on Colorado Springs scene T13SED

---

## 7. How Local Work Connects To Platform

The local Python pipeline feeds the platform's ingestion layer:

```
Local Work (now)                Platform (later)
─────────────────               ─────────────────
ingest.py                  →   satellite-platform-ingestion/
D:/SatelliteData/          →   S3 Object Storage
SQLite catalog             →   PostgreSQL (Neon → RDS)
QGIS visualization         →   CesiumJS 3D globe
satellite-base env         →   Docker container
satellite-ml models        →   AI/Intelligence Layer
```

---

## 8. Decision Log Index

| ID | Decision | Status |
|----|----------|--------|
| [[AD-001 Primary Language]] | Python | ✅ |
| [[AD-002 Storage Structure]] | raw/processed/exports | ✅ |
| [[AD-003 Database Strategy]] | SQLite → PostgreSQL + CSV | ✅ |
| [[AD-004 JP2 Conversion Strategy]] | Always GeoTIFF on ingest | ✅ |
| [[AD-005 Ingestion Metadata Schema]] | Full SQLite schema | ✅ |
| [[AD-006 Pipeline Logging Strategy]] | SQLite + CSV both | ✅ |
| [[AD-007 First Data Source]] | Copernicus Sentinel-2 | ✅ |
| [[AD-008 ML Framework]] | PyTorch + torchgeo | ✅ |
| AD-009 | Frontend framework | 🔲 Next.js confirmed |
| AD-010 | Database hosting | 🔲 Neon.tech (Phase 0) |
| AD-011 | TLE primary source | 🔲 CelesTrak vs Space-Track |
| AD-012 | Deployment target | 🔲 Vercel (Phase 0) |

---

## 9. Related Notes
- [[Satellite Platform Roadmap]]
- [[Engineering OS Roadmap]]
- [[Satellite Platform MOC]]
- [[Conda Environments]]
- [[Satellite Data Sources]]
- [[Docker Inventory]]
- [[ENB-001 Platform Setup]]
- [[ENB-002 Ingestion Pipeline v1]]
