<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 04 · Technology Stack

</div>

| | |
|---|---|
| **Document** | 04 of 09 — Technologies, libraries, versions and rationale |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Mandated Stack](#1-mandated-stack)
2. [2. Frontend](#2-frontend)
3. [3. Backend](#3-backend)
4. [4. Database — Neon](#4-database--neon)
5. [5. AI Layer](#5-ai-layer)
6. [6. Authentication & Security](#6-authentication--security)
7. [7. DevOps & Tooling](#7-devops--tooling)
8. [8. Data Sources for Demo](#8-data-sources-for-demo)
9. [9. Optional / Stretch](#9-optional--stretch)
10. [10. Version Pins (suggested starting point)](#10-version-pins-suggested-starting-point)

---

## 1. Mandated Stack
| Layer | Technology | Notes |
|---|---|---|
| Frontend | **React** | SPA built with Vite |
| Backend | **Django** | Django REST Framework API |
| Database | **Neon DB** | Serverless PostgreSQL |
| AI | **Groq API** | LLM inference (fast Llama-family models) |
| Authentication | **JWT** | `djangorestframework-simplejwt` |

## 2. Frontend
| Purpose | Choice | Reason |
|---|---|---|
| Build tool | Vite 5 | Fast dev/build |
| Language | TypeScript | Safer API contracts |
| UI | React 18 | Required |
| Routing | React Router v6 | Role-guarded routes |
| Server state | TanStack Query | Caching, retries, pagination |
| Client state | Zustand | Tiny store for auth/UI |
| HTTP | Axios | Interceptors for JWT refresh |
| Styling | Tailwind CSS + shadcn/ui | Rapid, consistent UI |
| Charts | Recharts | Dashboard visuals |
| Tables | TanStack Table | Sorting/filter/pagination |
| Calendar | FullCalendar (resource timeline) | Scheduler view |
| Forms | React Hook Form + Zod | Validation |
| Icons | Lucide | |
| Testing | Vitest + React Testing Library, Playwright (E2E) | |

## 3. Backend
| Purpose | Choice |
|---|---|
| Language | Python 3.11+ |
| Framework | Django 5.x + Django REST Framework |
| Auth | `djangorestframework-simplejwt` (access/refresh, rotation, blacklist) |
| DB driver | `psycopg[binary]` (v3) |
| DB URL parsing | `dj-database-url` |
| Config | `django-environ` / `python-dotenv` |
| CORS | `django-cors-headers` |
| Filtering | `django-filter` |
| API docs | `drf-spectacular` (OpenAPI + Swagger UI) |
| Async | Celery 5 + Redis; Celery Beat for schedules |
| File handling | `django-storages` + boto3 (S3/R2); resume text via `pdfplumber`, `python-docx` |
| Server | Gunicorn (prod), Uvicorn optional |
| Static | WhiteNoise |
| Testing | pytest, pytest-django, factory-boy, Faker |
| Quality | Ruff, Black, mypy (optional), pre-commit |

## 4. Database — Neon
- Serverless PostgreSQL 16, branching (dev/staging/prod branches), autoscaling, scale-to-zero.
- Use **pooled connection string** (`...-pooler...`) for the app and **direct** string for migrations.
- Extensions: `pgvector` (embeddings), `pg_trgm` (fuzzy search), `uuid-ossp`/`pgcrypto`.
- Required params: `sslmode=require`; set `CONN_MAX_AGE=0` (or small) with pooler to avoid stale connections.

## 5. AI Layer
### 5.1 Groq API
| Use | Model (configurable via env) | Why |
|---|---|---|
| Resume/JD parsing, explanations, chatbot, prep plans, mock interview | `llama-3.3-70b-versatile` | Quality for structured extraction |
| Quick/low-cost tasks (short narratives, classification) | `llama-3.1-8b-instant` | Speed and cost |

> Model names change over time; keep them in env vars (`GROQ_MODEL_PRIMARY`, `GROQ_MODEL_FAST`) and confirm availability in the Groq console before the demo.

- SDK: `groq` Python package. Use **JSON mode** (`response_format={"type":"json_object"}`) and validate with Pydantic.
- Temperature 0–0.2 for extraction, 0.5–0.7 for coaching text.
- Retries with exponential backoff, request timeout 20 s, response cache keyed by input hash.

### 5.2 Classical ML
| Need | Library |
|---|---|
| At-risk / placement prediction | scikit-learn (Logistic Regression, Gradient Boosting), `shap` for factors |
| Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`, runs on CPU) + pgvector |
| Fallback similarity | scikit-learn TF-IDF + cosine |
| Data handling | pandas, numpy |
| Scheduling optimisation | Interval logic in pure Python; optional `ortools` CP-SAT for larger instances |

> **Why local embeddings:** Groq does not provide an embeddings endpoint, so semantic matching uses a small local model (or TF-IDF fallback) while Groq handles language tasks.

## 6. Authentication & Security
- JWT access (15 min) + refresh (7 days, rotating, blacklisted on logout).
- Custom `User` model with `role` and `campus`.
- DRF permissions per role, campus-scoped querysets.
- Throttling: `AnonRateThrottle`, `UserRateThrottle`, stricter scope for AI endpoints.
- Password validators, HTTPS, security headers (`SECURE_*`), CORS allow-list.
- Store the access token **in memory** and refresh token in an `HttpOnly` cookie if you add a cookie flow later; for the prototype, memory + `localStorage` refresh is acceptable but note XSS risk.

## 7. DevOps & Tooling
| Area | Choice |
|---|---|
| Version control | Git + GitHub, trunk-based with feature branches |
| CI | GitHub Actions (lint, test, build) |
| Containers | Docker, docker-compose for local (Django + Redis; DB is Neon) |
| Hosting | Frontend: Vercel/Netlify · API: Render/Railway/Fly.io · Redis: Upstash · Files: Cloudflare R2 |
| Monitoring | Sentry, UptimeRobot on `/api/health/` |
| Secrets | Platform env vars; `.env` locally (git-ignored) |

## 8. Data Sources for Demo
- **Synthetic generator** (Faker + custom rules) for students, recruiters, drives, assessments, historical placements.
- **Kaggle "Campus Recruitment" dataset** for baseline placement-outcome modelling.
- Reproducible seed with fixed random seed.

## 9. Optional / Stretch
| Feature | Tech |
|---|---|
| WhatsApp/SMS | Twilio / Meta Cloud API |
| Mobile alerts | PWA + Web Push |
| LinkedIn / job portals | Official APIs (subject to access) |
| Offer verification | SHA-256 hash (v1), blockchain anchoring (future) |
| Vector search | pgvector HNSW index |

## 10. Version Pins (suggested starting point)
```
python >=3.11
django >=5.0,<5.2
djangorestframework >=3.15
djangorestframework-simplejwt >=5.3
psycopg[binary] >=3.1
celery >=5.3
redis >=5.0
groq >=0.9
scikit-learn >=1.4
sentence-transformers >=2.7
pgvector >=0.2
node >=20
react ^18
vite ^5
```

---

<div align="center">

[← UI Design](03_ui_design.md) &nbsp;·&nbsp; [Database Schema →](05_database_schema.md)

<sub>CampusLink · Document 04 of 09 · Version 1.0 · Confidential draft</sub>

</div>
