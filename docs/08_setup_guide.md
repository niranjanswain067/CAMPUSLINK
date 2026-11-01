<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 08 · Setup Guide

</div>

| | |
|---|---|
| **Document** | 08 of 09 — Local setup, configuration, deployment and troubleshooting |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Prerequisites](#1-prerequisites)
2. [2. Repository Layout](#2-repository-layout)
3. [3. Neon Database Setup](#3-neon-database-setup)
4. [4. Backend Setup](#4-backend-setup)
5. [5. Frontend Setup](#5-frontend-setup)
6. [6. Demo Accounts (created by `seed_demo`)](#6-demo-accounts-created-by-seed_demo)
7. [7. Verify the Install](#7-verify-the-install)
8. [8. Docker Compose (optional)](#8-docker-compose-optional)
9. [9. Deployment Quick Guide](#9-deployment-quick-guide)
10. [10. Troubleshooting](#10-troubleshooting)

---

Get CampusLink running locally in ~30 minutes.

## 1. Prerequisites
| Tool | Version | Check |
|---|---|---|
| Python | 3.11+ | `python --version` |
| Node.js | 20+ | `node -v` |
| Git | any | `git --version` |
| Redis | 7+ (local or Upstash URL) | `redis-cli ping` |
| Docker (optional) | latest | `docker -v` |

Accounts needed (all have free tiers):
1. **Neon** — https://neon.tech
2. **Groq** — https://console.groq.com
3. (Optional) SMTP provider — Gmail app password / Mailtrap for dev email

## 2. Repository Layout
```
campuslink/
├── backend/
│   ├── manage.py
│   ├── config/            # settings, urls, celery
│   ├── apps/
│   │   ├── accounts/ students/ recruiters/ drives/ scheduling/
│   │   ├── matching/ readiness/ offers/ notifications/ analytics/ ai_services/
│   ├── ml/                # training scripts, saved models
│   ├── data/              # kaggle csv, seed configs
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── src/  index.html  vite.config.ts  package.json
│   └── .env.example
├── docs/                  # the 9 .md files
├── docker-compose.yml
└── README.md
```

## 3. Neon Database Setup
1. Create a Neon project (region closest to you) → note the default branch.
2. Create branches: `dev` (local work), later `staging`, `prod`.
3. Copy two connection strings from the dashboard:
   - **Pooled** (host contains `-pooler`) → app runtime.
   - **Direct** → migrations.
4. In the Neon SQL editor, run:
```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

## 4. Backend Setup
```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env               # then edit values
```

### 4.1 `requirements.txt` (core)
```
Django>=5.0,<5.2
djangorestframework>=3.15
djangorestframework-simplejwt>=5.3
django-cors-headers
django-filter
django-environ
dj-database-url
drf-spectacular
psycopg[binary]>=3.1
pgvector
celery>=5.3
redis>=5.0
django-celery-beat
django-storages
boto3
whitenoise
gunicorn
groq
pydantic>=2
pdfplumber
python-docx
scikit-learn
pandas
numpy
joblib
shap
sentence-transformers
Faker
pytest
pytest-django
factory-boy
ruff
```

### 4.2 `.env` (backend)
```
DJANGO_SECRET_KEY=change-me-to-a-long-random-string
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=http://localhost:5173

# Neon
DATABASE_URL=postgresql://USER:PASSWORD@ep-xxxx-pooler.REGION.aws.neon.tech/neondb?sslmode=require
DATABASE_URL_DIRECT=postgresql://USER:PASSWORD@ep-xxxx.REGION.aws.neon.tech/neondb?sslmode=require

# JWT
JWT_ACCESS_MINUTES=15
JWT_REFRESH_DAYS=7

# Groq
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxx
GROQ_MODEL_PRIMARY=llama-3.3-70b-versatile
GROQ_MODEL_FAST=llama-3.1-8b-instant
GROQ_TIMEOUT_SECONDS=20

# Celery / Redis
REDIS_URL=redis://localhost:6379/0

# Email (dev: console backend)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
DEFAULT_FROM_EMAIL=placements@campuslink.local

# Storage (dev: local media)
USE_S3=False
```

### 4.3 Key settings (excerpt)
```python
# config/settings.py
import environ, dj_database_url
from datetime import timedelta
env = environ.Env()

DATABASES = {"default": dj_database_url.parse(env("DATABASE_URL"), conn_max_age=0, ssl_require=True)}

INSTALLED_APPS += [
    "rest_framework", "rest_framework_simplejwt.token_blacklist",
    "corsheaders", "django_filters", "drf_spectacular", "django_celery_beat",
    "apps.accounts", "apps.students", "apps.recruiters", "apps.drives", "apps.scheduling",
    "apps.matching", "apps.readiness", "apps.offers", "apps.notifications",
    "apps.analytics", "apps.ai_services",
]
AUTH_USER_MODEL = "accounts.User"

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": ["rest_framework_simplejwt.authentication.JWTAuthentication"],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_FILTER_BACKENDS": ["django_filters.rest_framework.DjangoFilterBackend",
                                "rest_framework.filters.SearchFilter", "rest_framework.filters.OrderingFilter"],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.UserRateThrottle", "rest_framework.throttling.AnonRateThrottle"],
    "DEFAULT_THROTTLE_RATES": {"user": "1000/hour", "anon": "100/hour", "ai": "20/min", "login": "10/min"},
}
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=env.int("JWT_ACCESS_MINUTES", 15)),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=env.int("JWT_REFRESH_DAYS", 7)),
    "ROTATE_REFRESH_TOKENS": True, "BLACKLIST_AFTER_ROTATION": True,
}
```

### 4.4 Migrate, seed, run
```bash
# use the DIRECT url for migrations
DATABASE_URL=$DATABASE_URL_DIRECT python manage.py migrate
python manage.py createsuperuser
python manage.py seed_demo --students 300 --seed 42   # synthetic data + 4 demo drives
python manage.py train_models                          # at-risk model + readiness calibration
python manage.py runserver                             # http://localhost:8000
```
Swagger UI: http://localhost:8000/api/docs/

### 4.5 Background workers (new terminals, venv active)
```bash
celery -A config worker -l info
celery -A config beat -l info
```

## 5. Frontend Setup
```bash
cd frontend
npm create vite@latest . -- --template react-ts     # skip if repo already scaffolded
npm install
npm install axios @tanstack/react-query @tanstack/react-table react-router-dom zustand \
            react-hook-form zod @hookform/resolvers recharts lucide-react \
            @fullcalendar/react @fullcalendar/daygrid @fullcalendar/timegrid @fullcalendar/resource-timeline @fullcalendar/interaction
npm install -D tailwindcss postcss autoprefixer vitest @testing-library/react playwright
npx tailwindcss init -p
```

`frontend/.env`
```
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

Axios JWT interceptor (sketch):
```ts
// src/api/client.ts
import axios from "axios";
import { useAuth } from "../store/auth";

export const api = axios.create({ baseURL: import.meta.env.VITE_API_BASE_URL });

api.interceptors.request.use((cfg) => {
  const t = useAuth.getState().access;
  if (t) cfg.headers.Authorization = `Bearer ${t}`;
  return cfg;
});

let refreshing: Promise<string> | null = null;
api.interceptors.response.use(r => r, async (err) => {
  const orig = err.config;
  if (err.response?.status === 401 && !orig._retry) {
    orig._retry = true;
    refreshing ??= useAuth.getState().refresh().finally(() => (refreshing = null));
    const token = await refreshing;
    orig.headers.Authorization = `Bearer ${token}`;
    return api(orig);
  }
  return Promise.reject(err);
});
```

Run:
```bash
npm run dev      # http://localhost:5173
```

## 6. Demo Accounts (created by `seed_demo`)
| Role | Email | Password |
|---|---|---|
| Admin | admin@campuslink.demo | Admin@123 |
| TPO | tpo@campuslink.demo | Tpo@12345 |
| Student | student1@campuslink.demo | Student@123 |
| Recruiter | recruiter@campuslink.demo | Recruit@123 |
| Mentor | mentor@campuslink.demo | Mentor@123 |
> Change or disable these in any non-demo deployment.

## 7. Verify the Install
| Check | Expected |
|---|---|
| `GET /api/v1/health/` | `{"status":"ok","db":"ok","redis":"ok"}` |
| Login as student → Readiness page | Score + band + breakdown |
| Login as TPO → Drive 1 → Run matching | Ranked list with explanations |
| TPO → Scheduler | At least one seeded conflict displayed |
| Groq test: `python manage.py check_groq` | Prints a short model reply |

## 8. Docker Compose (optional)
```yaml
services:
  redis:
    image: redis:7
    ports: ["6379:6379"]
  api:
    build: ./backend
    env_file: ./backend/.env
    command: gunicorn config.wsgi:application -b 0.0.0.0:8000
    ports: ["8000:8000"]
    depends_on: [redis]
  worker:
    build: ./backend
    env_file: ./backend/.env
    command: celery -A config worker -l info
    depends_on: [redis]
  beat:
    build: ./backend
    env_file: ./backend/.env
    command: celery -A config beat -l info
    depends_on: [redis]
  web:
    build: ./frontend
    ports: ["5173:80"]
```
(Database is Neon, so no Postgres container is needed.)

## 9. Deployment Quick Guide
1. **Neon:** create `prod` branch; copy pooled + direct URLs.
2. **API (Render/Railway):** Docker deploy; set env vars (`DJANGO_DEBUG=False`, real `DJANGO_SECRET_KEY`, `ALLOWED_HOSTS`, `CORS_ALLOWED_ORIGINS`, Neon, Groq, Redis, SMTP, S3). Release command: `python manage.py migrate` (with direct URL) and `collectstatic`.
3. **Workers:** deploy `worker` and `beat` as separate services.
4. **Frontend (Vercel/Netlify):** build `npm run build`, set `VITE_API_BASE_URL` to the API URL.
5. **Smoke test:** health endpoint → login → matching → offer.

## 10. Troubleshooting
| Symptom | Fix |
|---|---|
| `SSL connection required` | Ensure `?sslmode=require` in URL |
| Random "connection closed" errors | Use pooled URL, `conn_max_age=0`, retry on cold start |
| `extension "vector" is not available` | Run `CREATE EXTENSION vector;` on the correct Neon branch/db |
| CORS error in browser | Add frontend origin to `CORS_ALLOWED_ORIGINS` |
| 401 loop on frontend | Check system clock, refresh token blacklist, interceptor `_retry` flag |
| Groq 429 | Lower concurrency, enable caching, use fast model; fallback explanations still work |
| Groq returns non-JSON | Confirm `response_format={"type":"json_object"}` and "JSON" appears in prompt |
| Slow first embedding | Model downloads once (~90 MB); pre-download in Docker build |
| Celery tasks not running | Check `REDIS_URL`, worker running, `CELERY_BROKER_URL` set in settings |

---

<div align="center">

[← Development Plan](07_development_plan.md) &nbsp;·&nbsp; [Implementation Plan →](09_implementation_plan.md)

<sub>CampusLink · Document 08 of 09 · Version 1.0 · Confidential draft</sub>

</div>
