<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 05 · Database Schema

</div>

| | |
|---|---|
| **Document** | 05 of 09 — Neon PostgreSQL tables, constraints, indexes and views |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Conventions](#1-conventions)
2. [2. ER Overview](#2-er-overview)
3. [3. Tables](#3-tables)
4. [4. Key Constraints & Rules](#4-key-constraints--rules)
5. [5. Indexing Summary](#5-indexing-summary)
6. [6. Analytics Views (read-optimised)](#6-analytics-views-read-optimised)
7. [7. Migration & Seeding Notes](#7-migration--seeding-notes)

---

## 1. Conventions
- Primary keys: `BIGINT` identity (Django default `BigAutoField`) — or UUID where noted.
- Every tenant table has `campus_id` (FK) and `created_at`, `updated_at`.
- Soft facts that vary (score breakdowns, parsed JDs) use `JSONB`.
- Embeddings use `vector(384)` (MiniLM) via `pgvector`.
- Enums are stored as `VARCHAR` with Django `choices` (easy migrations).

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

## 2. ER Overview

```
Campus ─┬─< User >─┬─ StudentProfile ─┬─< StudentSkill >─ Skill
        │          │                  ├─< Certification
        │          │                  ├─< Project
        │          │                  ├─< Assessment
        │          │                  └─1 ReadinessScore (history)
        │          ├─ RecruiterContact ─ Company
        │          └─ MentorAssignment
        ├─< Venue
        ├─< Panel >─< PanelMember (User)
        └─< Drive ─┬─ DriveEligibility
                   ├─< DriveSkill >─ Skill
                   ├─< DriveRound ─ (Venue, Panel)
                   ├─< Application ─ MatchResult
                   └─< Offer ─┬─< OfferDocument
                              └─< OfferEvent
Notification, NotificationTemplate, Escalation, AtRiskPrediction, AuditLog, ChatMessage
```

## 3. Tables

### 3.1 Tenancy & Users
**campuses**
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| name | varchar(200) | |
| code | varchar(20) UNIQUE | |
| city, state | varchar | |
| is_active | bool | |

**users** (custom `AbstractBaseUser`)
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| email | varchar(254) UNIQUE | login |
| password | varchar(128) | hashed |
| full_name | varchar(150) | |
| role | varchar(20) | `ADMIN, TPO, STUDENT, RECRUITER, MENTOR` |
| campus_id | FK campuses NULL | NULL for global admin/recruiters |
| phone | varchar(20) | |
| is_active, is_staff | bool | |
| last_login, created_at | timestamptz | |

Index: `(campus_id, role)`.

Token blacklist tables are created by `rest_framework_simplejwt.token_blacklist`.

### 3.2 Students
**branches**: `id, campus_id, name, code`

**skills**
| Column | Type |
|---|---|
| id | bigint PK |
| name | varchar(100) UNIQUE |
| category | varchar(50) — `LANGUAGE, FRAMEWORK, CLOUD, DATA, SOFT, TOOL, DOMAIN` |
| embedding | vector(384) NULL |

**student_profiles**
| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | 1-1 FK users | |
| campus_id | FK | |
| roll_number | varchar(30) | UNIQUE per campus |
| branch_id | FK branches | |
| graduation_year | int | |
| cgpa | numeric(4,2) | 0–10 |
| backlogs_current | smallint | |
| backlogs_history | smallint | |
| tenth_pct, twelfth_pct | numeric(5,2) | |
| resume_file | varchar | storage path |
| resume_text | text | extracted |
| profile_summary | text | |
| profile_embedding | vector(384) NULL | |
| target_role | varchar(100) | |
| placement_status | varchar(20) | `UNPLACED, IN_PROCESS, PLACED, OPTED_OUT` |
| mentor_id | FK users NULL | |
| created_at, updated_at | timestamptz | |

Indexes: `(campus_id, branch_id, graduation_year)`, `(placement_status)`, HNSW on `profile_embedding`.

**student_skills**
| Column | Type |
|---|---|
| id | PK |
| student_id | FK |
| skill_id | FK |
| proficiency | smallint (1–5) |
| source | varchar — `RESUME, SELF, ASSESSMENT, CERT` |
| UNIQUE (student_id, skill_id) | |

**certifications**: `id, student_id, name, issuer, issued_on, credential_url, skill_id NULL, verified bool`

**projects**: `id, student_id, title, description, tech_stack JSONB, url, start_date, end_date, embedding vector(384) NULL`

**internships**: `id, student_id, company, role, start_date, end_date, description`

**assessments**
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| student_id | FK | |
| type | varchar | `APTITUDE, MOCK_INTERVIEW, COMMUNICATION, SOFT_SKILL, TECHNICAL` |
| score | numeric(5,2) | normalised 0–100 |
| max_score | numeric(6,2) | |
| details | JSONB | per-section scores, AI feedback |
| taken_on | date | |
| source | varchar | `MANUAL, AI_MOCK, IMPORT` |

Index: `(student_id, type, taken_on DESC)`.

**readiness_scores** (history)
| Column | Type |
|---|---|
| id | PK |
| student_id | FK |
| score | numeric(5,2) |
| band | varchar — `NOT_READY, DEVELOPING, READY, HIGHLY_EMPLOYABLE` |
| breakdown | JSONB — `{academics: {value, weight, contribution}, ...}` |
| top_gaps | JSONB — list of improvement areas |
| model_version | varchar |
| computed_at | timestamptz |

Index: `(student_id, computed_at DESC)`.

### 3.3 Recruiters & Drives
**companies**: `id, name, industry, website, tier varchar (DREAM/SUPER/NORMAL), hq_location, created_at`

**recruiter_contacts**: `id, user_id FK, company_id FK, designation`

**venues**: `id, campus_id, name, capacity int, resources JSONB`

**panels**: `id, campus_id, name`; **panel_members**: `id, panel_id, user_id (recruiter/faculty), name` 

**drives**
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| campus_id | FK | |
| company_id | FK | |
| title | varchar(200) | |
| job_role | varchar(120) | |
| job_description | text | raw JD |
| parsed_jd | JSONB | AI-structured |
| jd_embedding | vector(384) NULL | |
| ctc_min, ctc_max | numeric(10,2) | LPA |
| location | varchar | |
| drive_type | varchar | `FULL_TIME, INTERNSHIP, PPO` |
| status | varchar | `DRAFT, SCHEDULED, ACTIVE, COMPLETED, CANCELLED` |
| application_deadline | timestamptz | |
| fit_threshold | numeric(5,2) | default 60 |
| created_by | FK users | |

**drive_eligibility**: `id, drive_id (1-1), allowed_branches (M2M branches), min_cgpa numeric, max_backlogs smallint, min_tenth_pct, min_twelfth_pct, graduation_year, extra_rules JSONB`

**drive_skills**: `id, drive_id, skill_id, importance (REQUIRED/PREFERRED), min_proficiency smallint, weight numeric`

**drive_rounds**
| Column | Type |
|---|---|
| id | PK |
| drive_id | FK |
| round_no | smallint |
| round_type | varchar — `APTITUDE, TECHNICAL, GD, HR, OA` |
| start_at, end_at | timestamptz |
| venue_id | FK NULL |
| panel_id | FK NULL |
| status | varchar — `PLANNED, CONFIRMED, DONE` |
| UNIQUE (drive_id, round_no) | |

Indexes: `(venue_id, start_at, end_at)`, `(panel_id, start_at, end_at)`, GiST range index:
```sql
ALTER TABLE drive_rounds ADD CONSTRAINT no_venue_overlap
EXCLUDE USING gist (venue_id WITH =, tstzrange(start_at, end_at) WITH &&)
WHERE (venue_id IS NOT NULL AND status <> 'CANCELLED');
-- requires: CREATE EXTENSION btree_gist;
```
(DB-level safety net; the service layer still returns friendly conflict messages.)

**applications**
| Column | Type |
|---|---|
| id | PK |
| drive_id, student_id | FK, UNIQUE together |
| status | varchar — `ELIGIBLE, APPLIED, SHORTLISTED, REJECTED, IN_ROUND, SELECTED, WITHDRAWN` |
| current_round | smallint NULL |
| applied_at | timestamptz |
| decision_source | varchar — `AUTO, MANUAL` |
| decision_reason | text |

**match_results**
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| drive_id, student_id | FK, UNIQUE together | |
| eligible | bool | hard-filter result |
| ineligible_reasons | JSONB | e.g. `["CGPA 6.2 < 7.0"]` |
| fit_score | numeric(5,2) | |
| fit_band | varchar | |
| factors | JSONB | `{skill_match, semantic, proj_cert, interview, readiness}` |
| matched_skills, missing_skills | JSONB | |
| rank | int | |
| decision | varchar | `SHORTLISTED, BELOW_THRESHOLD, INELIGIBLE` |
| explanation | text | LLM narrative |
| explanation_source | varchar | `LLM, TEMPLATE` |
| model_version | varchar | |
| computed_at | timestamptz | |

Index: `(drive_id, rank)`, `(student_id)`.

### 3.4 Offers & Documents
**offers**
| Column | Type |
|---|---|
| id | PK |
| application_id | 1-1 FK |
| drive_id, student_id, company_id | FK |
| offer_type | `FULL_TIME, PPO, INTERN_CONVERSION, INTERNSHIP` |
| role_title | varchar |
| ctc | numeric(10,2) (LPA) |
| joining_bonus | numeric(10,2) NULL |
| location | varchar |
| joining_date | date NULL |
| status | `ISSUED, ACCEPTED, DEFERRED, DECLINED, WITHDRAWN, JOINED` |
| bond_months | smallint NULL |
| offer_letter_file | varchar |
| offer_letter_sha256 | char(64) |
| issued_at, responded_at, response_deadline | timestamptz |

**offer_documents**: `id, offer_id, doc_type (OFFER_LETTER, BOND, ID_PROOF, MARKSHEETS, BACKGROUND_CHECK, MEDICAL, OTHER), status (PENDING, SUBMITTED, VERIFIED, REJECTED), file, due_date, verified_by FK NULL, remarks, updated_at`

**offer_events** (audit trail): `id, offer_id, from_status, to_status, actor_id, note, created_at`

### 3.5 Analytics & Prediction
**at_risk_predictions**: `id, student_id, probability numeric(4,3), risk_level (LOW/MEDIUM/HIGH), top_factors JSONB, model_version, computed_at`

**escalations**: `id, student_id, mentor_id, reason, status (OPEN, IN_PROGRESS, RESOLVED), notes, created_at, resolved_at`

**historical_placements** (training/analytics): `id, campus_id, student_ref, branch, grad_year, cgpa, skills JSONB, company_id, ctc, outcome (PLACED/NOT_PLACED), source (SEED/KAGGLE/LIVE)`

**ml_models**: `id, name, version, metrics JSONB, artifact_path, trained_at, is_active`

### 3.6 Communication
**notification_templates**: `id, code UNIQUE, channel (INAPP/EMAIL/SMS), subject, body (with {{placeholders}}), campus_id NULL`

**notifications**
| Column | Type |
|---|---|
| id | PK |
| user_id | FK |
| event | varchar — `SHORTLISTED, INTERVIEW_SCHEDULED, DOC_DUE, OFFER_UPDATE, DRIVE_ANNOUNCEMENT, AT_RISK, CONFLICT` |
| title, message | varchar / text |
| payload | JSONB (deep-link ids) |
| channel | varchar |
| status | `PENDING, SENT, FAILED, READ` |
| scheduled_for, sent_at, read_at | timestamptz |

Index: `(user_id, status, created_at DESC)`.

**notification_preferences**: `user_id, channel, event, enabled`

**chat_messages**: `id, user_id, session_id uuid, role (user/assistant), content, tokens_used, created_at`

**audit_logs**: `id, actor_id, action, entity_type, entity_id, before JSONB, after JSONB, ip, created_at`

## 4. Key Constraints & Rules
| Rule | Enforcement |
|---|---|
| One application per student per drive | UNIQUE (drive_id, student_id) |
| No venue overlap | GiST exclusion constraint + service check |
| CGPA 0–10 | CHECK constraint |
| Offer must belong to a selected application | Service validation |
| Users scoped by campus | Queryset filter + (optional) RLS |

## 5. Indexing Summary
- B-tree: foreign keys, `(campus_id, …)` composites, status columns.
- GiST: round time ranges.
- HNSW (pgvector): `student_profiles.profile_embedding`, `drives.jd_embedding` using `vector_cosine_ops`.
- GIN: `parsed_jd`, `match_results.factors` (only if queried by key).
- `pg_trgm` GIN on `students` name / `skills.name` for search.

## 6. Analytics Views (read-optimised)
```sql
CREATE MATERIALIZED VIEW mv_branch_conversion AS
SELECT sp.campus_id, b.name AS branch, sp.graduation_year,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE sp.placement_status='PLACED') AS placed
FROM student_profiles sp JOIN branches b ON b.id = sp.branch_id
GROUP BY sp.campus_id, b.name, sp.graduation_year;
```
Refresh via Celery Beat (nightly) or on-demand. Similar views: `mv_skill_conversion`, `mv_package_trends`, `mv_offer_funnel`.

## 7. Migration & Seeding Notes
- Django migrations manage all tables; raw SQL for extensions, GiST constraint and materialised views goes in `RunSQL` migrations.
- Seed command `python manage.py seed_demo` creates: 2 campuses, ~300 students, 25 companies, 4 drives (with deliberate conflicts), assessments, offers in each status, and historical placements.

---

<div align="center">

[← Technology Stack](04_technology_stack.md) &nbsp;·&nbsp; [API Specification →](06_api_specification.md)

<sub>CampusLink · Document 05 of 09 · Version 1.0 · Confidential draft</sub>

</div>
