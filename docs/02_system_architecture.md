<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 02 · System Architecture

</div>

| | |
|---|---|
| **Document** | 02 of 09 — Components, data flows, AI layer, security, scalability and deployment |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Architectural Style](#1-architectural-style)
2. [2. High-Level Diagram](#2-high-level-diagram)
3. [3. Components](#3-components)
4. [4. Key Flows](#4-key-flows)
5. [5. Security Architecture](#5-security-architecture)
6. [6. Scalability & Multi-Campus Strategy](#6-scalability--multi-campus-strategy)
7. [7. Deployment Architecture](#7-deployment-architecture)
8. [8. Failure Handling](#8-failure-handling)
9. [9. Observability](#9-observability)

---

## 1. Architectural Style
**Modular monolith** (Django) + **React SPA**, with an async worker for notifications and heavy AI/ML jobs. Chosen for fast hackathon delivery while keeping clear module boundaries that can later be split into services.

## 2. High-Level Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      React SPA (Vite)                        │
│  Student Portal │ TPO Console │ Recruiter Portal │ Dashboard │
└───────────────┬──────────────────────────────────────────────┘
                │ HTTPS  (JSON, Authorization: Bearer <JWT>)
┌───────────────▼──────────────────────────────────────────────┐
│                 Django + Django REST Framework               │
│ ┌────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐  │
│ │ accounts│ │ students│ │recruiters│ │  drives  │ │ offers  │  │
│ │ (JWT)   │ │ profile │ │  & JDs   │ │scheduling│ │ & docs  │  │
│ └────────┘ └─────────┘ └─────────┘ └──────────┘ └─────────┘  │
│ ┌────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐              │
│ │matching│ │readiness│ │analytics│ │notifica- │              │
│ │ engine │ │ scoring │ │& predict│ │  tions   │              │
│ └───┬────┘ └────┬────┘ └────┬────┘ └────┬─────┘              │
│     └───────────┴───┬───────┴───────────┘                    │
│               ┌─────▼──────┐                                 │
│               │ ai_services │  Groq client, prompts, guards  │
│               └─────┬──────┘                                 │
└─────────────────────┼────────────────────────────────────────┘
      │               │                     │
┌─────▼─────┐   ┌─────▼──────┐      ┌───────▼────────┐
│ Neon       │   │ Groq API   │      │ Celery worker  │
│ PostgreSQL │   │ (LLM)      │      │ + Redis broker │
│ (+pgvector)│   └────────────┘      │ email/reminders│
└───────────┘                        │ ML retraining  │
                                     └────────────────┘
        Object storage (resumes, offer letters, docs): S3-compatible / local media in dev
```

## 3. Components

### 3.1 Frontend (React)
- Vite + React 18, React Router, TanStack Query (server state), Axios (JWT interceptor with silent refresh), Zustand (auth/UI state), Recharts (charts), Tailwind CSS.
- Route guards by role. Feature folders: `auth`, `student`, `tpo`, `recruiter`, `dashboard`, `shared`.

### 3.2 Backend (Django apps)
| App | Responsibility |
|---|---|
| `accounts` | Users, roles, JWT auth (SimpleJWT), campus scoping |
| `students` | Profiles, skills, certs, projects, assessments, resume upload |
| `recruiters` | Companies, contacts, historical hiring |
| `drives` | Drives, JDs, eligibility, rounds, venues, panels, applications |
| `scheduling` | Conflict detection and auto-resolution service |
| `matching` | Eligibility filter, fit scoring, ranking, explanations |
| `readiness` | Readiness scoring, skill-gap analysis, prep plans |
| `offers` | Offers, documents, joining status |
| `notifications` | Templates, in-app/email dispatch, reminders, escalations |
| `analytics` | Aggregations, at-risk prediction, trend endpoints |
| `ai_services` | Groq client wrapper, prompt templates, JSON-schema validation, caching, fallback |

### 3.3 AI / ML Layer
| Capability | Technique |
|---|---|
| Resume parsing, JD parsing | Groq LLM with JSON-mode structured output + schema validation |
| Readiness score | Weighted rule model (transparent) |
| Fit score | Hybrid: hard rules + skill overlap + semantic similarity + readiness |
| Semantic similarity | Sentence-Transformers `all-MiniLM-L6-v2` (CPU) stored in `pgvector`; TF-IDF cosine fallback |
| At-risk prediction | scikit-learn Gradient Boosting / Logistic Regression + SHAP-style top factors |
| Explanations | Deterministic factor list → Groq narrates in natural language (fallback: template text) |
| Chatbot, mock interview, prep plan | Groq chat with retrieval of the student's/drive data as context |

**Guardrail principle:** the LLM extracts and explains; **numbers and decisions come from deterministic code or trained models**, so results are reproducible and auditable.

### 3.4 Async Layer
Celery + Redis (Upstash free tier in cloud) for: sending emails, scheduled reminders (Celery Beat), nightly at-risk recompute, bulk matching, embedding generation.

### 3.5 Data Layer
- **Neon PostgreSQL**: relational data, JSONB for score breakdowns, `pgvector` for embeddings.
- Use Neon's **pooled** connection string (`-pooler` host) for the app; direct string for migrations.
- File storage: S3-compatible bucket (Cloudflare R2 / S3); local `media/` in development.

## 4. Key Flows

### 4.1 Authentication
1. `POST /auth/login` → `{access, refresh}`.
2. Axios attaches `Authorization: Bearer <access>`.
3. On 401, interceptor calls `/auth/refresh`; on failure, redirect to login.
4. Logout blacklists refresh token.

### 4.2 Matching a Drive
1. TPO posts JD → `ai_services.parse_jd` → structured requirements saved.
2. `matching.run(drive_id)`: hard filter → compute features → fit score → rank → store `MatchResult` with factor JSON.
3. `ai_services.explain` creates narrative per candidate (batched, cached).
4. Auto-shortlist above threshold → `notifications` event fired.

### 4.3 Conflict-Aware Scheduling
1. On create/update of a drive round → `scheduling.detect(round)` checks time-interval overlaps for venue, panel members and shortlisted students.
2. Conflicts returned as list `{type, severity, entities, message}`.
3. `scheduling.suggest(round)` enumerates candidate slots, scores by (no conflicts, minimal shift, student/panel preference) and returns top 3.
4. Publish is blocked while any *blocking* conflict exists.

### 4.4 Offer Lifecycle
`Issued → (Accepted | Deferred | Declined | Withdrawn) → Documents Verified → Joined`. Each transition writes an `OfferEvent` and triggers a notification. Celery Beat sends reminders for pending docs/offers.

### 4.5 At-Risk Escalation
Nightly job scores all unplaced students → probability ≥ threshold → create `Escalation` → notify mentor → mentor logs action.

## 5. Security Architecture
- JWT (HS256 via SimpleJWT), short-lived access token, rotating refresh with blacklist.
- DRF permission classes: `IsAdmin`, `IsTPO`, `IsStudentOwner`, `IsRecruiterOfDrive`; campus-scoped querysets.
- Secrets in environment variables; `.env` never committed.
- CORS allow-list, CSRF not needed for pure-JWT API (no cookies), DRF throttling (login, AI endpoints).
- File uploads: type/size validation, random filenames, virus scan hook (future).
- PII minimisation before Groq calls; AI outputs validated against schemas; prompt-injection mitigation (resume/JD text treated as data, delimited, never as instructions).
- Audit log for shortlist overrides, offer changes, role changes.

## 6. Scalability & Multi-Campus Strategy
- **Tenancy:** shared database, `campus_id` column on all tenant tables with indexed filtering; optional Postgres Row-Level Security later.
- **Stateless API** behind a load balancer; scale Django instances horizontally.
- **Neon autoscaling** compute and read replicas for analytics queries.
- **Caching:** Redis for dashboard aggregates (TTL 60 s) and AI explanation cache.
- **Batch jobs:** matching for large drives chunked in Celery tasks.
- **Multi-cell analytics:** Admin-level aggregation endpoints across campuses; materialised views for heavy rollups.
- **Embeddings:** computed once per profile/JD change, indexed with pgvector HNSW.

## 7. Deployment Architecture
| Component | Recommended Host |
|---|---|
| React build | Vercel / Netlify / Cloudflare Pages |
| Django API + Celery | Render / Railway / Fly.io (Docker) |
| Database | Neon (managed) |
| Redis | Upstash / Render Redis |
| Files | Cloudflare R2 / S3 |

**CI/CD:** GitHub Actions → lint + tests → build Docker image → deploy. Environments: `dev`, `staging`, `prod` (separate Neon branches — Neon branching is useful for staging).

## 8. Failure Handling
| Failure | Behaviour |
|---|---|
| Groq timeout / rate limit | Retry with backoff (2×), then rule-based fallback explanation; mark `ai_fallback=true` |
| Invalid LLM JSON | Re-prompt once with schema; else fail safely and flag for manual entry |
| Email provider down | Celery retry; in-app notification still delivered |
| Neon cold start | Short retry on connection; pooled connection reduces impact |

## 9. Observability
Structured JSON logs, request IDs, Sentry for errors, `/api/health/` endpoint, Celery task metrics, AI call latency/cost counters.

---

<div align="center">

[← Project Requirements Document](01_project_requirements.md) &nbsp;·&nbsp; [UI Design →](03_ui_design.md)

<sub>CampusLink · Document 02 of 09 · Version 1.0 · Confidential draft</sub>

</div>
