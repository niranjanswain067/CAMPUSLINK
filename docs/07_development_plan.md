<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 07 · Development Plan

</div>

| | |
|---|---|
| **Document** | 07 of 09 — Schedule, sprints, team roles, testing strategy and risks |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Approach](#1-approach)
2. [2. Team Roles (suggested for 4 members)](#2-team-roles-suggested-for-4-members)
3. [3. Phases & Sprints](#3-phases--sprints)
4. [4. Work Breakdown (Epics → Key Tasks)](#4-work-breakdown-epics--key-tasks)
5. [5. Testing Strategy](#5-testing-strategy)
6. [6. Definition of Done](#6-definition-of-done)
7. [7. Compressed Hackathon Timeline (e.g., 48–72 hrs)](#7-compressed-hackathon-timeline-eg-4872-hrs)
8. [8. Risks & Mitigations](#8-risks--mitigations)
9. [9. Milestones & Demo Checkpoints](#9-milestones--demo-checkpoints)
10. [10. Branching & Release](#10-branching--release)

---

## 1. Approach
- **Agile, 1-week sprints** (or 6 compressed phases if hackathon-length — see §7).
- **Vertical slices:** each sprint ends with a demo-able end-to-end flow.
- Priorities (MoSCoW): **Must** = Profiling → Matching → Scheduling → Notification → Offers → Analytics; **Should** = AI mock interview, chatbot, at-risk model; **Could** = PWA push, WhatsApp, LinkedIn; **Won't (v1)** = blockchain, native mobile.

## 2. Team Roles (suggested for 4 members)
| Role | Focus |
|---|---|
| Backend Lead | Django apps, auth, scheduling, offers |
| AI/ML Engineer | Groq services, scoring, matching, prediction, evaluation |
| Frontend Lead | React app, dashboard, scheduler UI |
| Full-stack / QA & DevOps | Seed data, tests, CI/CD, deployment, docs |

(For smaller teams, merge roles and cut "Should/Could" items.)

## 3. Phases & Sprints

### Phase 0 — Foundations (Sprint 0, ~3–4 days)
- Repo, branching, pre-commit, CI.
- Neon project + branches (dev/staging); Groq key; env templates.
- Django project, apps skeleton, custom `User`, JWT auth, RBAC permissions, health endpoint.
- React app shell, routing, auth pages, Axios refresh interceptor, layout.
- **Exit:** login works for each role; protected routes; CI green.

### Phase 1 — Student Profiling & Readiness (Sprint 1)
- Models/migrations: campuses, branches, students, skills, certs, projects, assessments.
- Student profile CRUD + UI; assessment entry (TPO).
- Resume upload + Groq parsing + confirm flow.
- Readiness scoring service + band + breakdown; skill-gap analysis; job-role templates.
- Student dashboard & readiness page.
- **Exit:** a student sees score, band, gaps, prep plan.

### Phase 2 — Drives, Matching & Explainability (Sprint 2)
- Companies, drives, eligibility, drive skills; JD parser (Groq).
- Eligibility filter; fit scoring; ranking; explanation (LLM + template fallback).
- Auto-shortlist, override with audit log.
- Matching UI (ranked table, "Why?" drawer, threshold slider).
- **Exit:** 3+ simulated drives produce ranked, explained shortlists.

### Phase 3 — Scheduling & Conflicts (Sprint 3)
- Venues, panels, rounds; conflict detection service (venue/panel/slot/student).
- GiST exclusion constraint; suggestion engine; publish gate.
- Calendar UI with conflict badges and suggestion panel.
- **Exit:** seeded conflicts detected and resolved via suggestions.

### Phase 4 — Notifications & Offers (Sprint 4)
- Notification templates, in-app + email dispatch, Celery + Beat reminders.
- Offers lifecycle, documents checklist, SHA-256 verification, events timeline.
- Student offers page; TPO pipeline board; recruiter follow-up prompts.
- **Exit:** shortlisting → notification → offer → accept → docs verified → joined.

### Phase 5 — Analytics & Prediction (Sprint 5)
- Aggregation endpoints and materialised views.
- At-risk model training pipeline, scoring job, escalation to mentors.
- Command Dashboard (all widgets in doc 03 §5.5).
- **Exit:** dashboard shows live metrics; at-risk list with factors.

### Phase 6 — AI Extras, Evaluation, Hardening (Sprint 6)
- Chatbot, mock interview, resume improver.
- Evaluation notebook/script: matching and scoring metrics, at-risk AUC, conflict-detection tests.
- Performance (indexes, caching), security review, accessibility pass.
- Deployment, demo data, docs, demo script, slides.
- **Exit:** deployed demo; evaluation report; rehearsed walkthrough.

## 4. Work Breakdown (Epics → Key Tasks)
| Epic | Tasks | Est. (pd) |
|---|---|---|
| E1 Auth & RBAC | custom user, JWT, permissions, campus scoping, UI guards | 3 |
| E2 Student Profile | models, CRUD, resume upload/parse, UI | 5 |
| E3 Readiness | scoring, bands, gap analysis, prep plan, UI | 5 |
| E4 Drives & JD | models, CRUD, JD parse, eligibility, UI | 4 |
| E5 Matching | filter, fit score, embeddings, explanations, UI | 7 |
| E6 Scheduling | models, detection, suggestions, calendar UI | 6 |
| E7 Notifications | templates, Celery, email, in-app, reminders | 4 |
| E8 Offers & Docs | lifecycle, docs, hash, UI | 5 |
| E9 Analytics | endpoints, views, dashboard UI | 6 |
| E10 Prediction | dataset, training, SHAP factors, escalation | 5 |
| E11 AI Assistant | chatbot, mock interview, resume improver | 5 |
| E12 Data & Eval | seed generator, Kaggle prep, metrics | 4 |
| E13 DevOps & QA | CI/CD, tests, deploy, monitoring | 4 |
| **Total** | | **~63 person-days** |

## 5. Testing Strategy
| Level | Tools | Targets |
|---|---|---|
| Unit | pytest | scoring formulas, eligibility rules, conflict detector (edge: touching intervals, zero-length) |
| API | pytest-django + DRF client | permissions matrix, state transitions |
| AI | mocked Groq client + golden JSON samples | schema validity, fallback paths |
| ML | holdout/cross-validation | AUC, precision/recall, calibration |
| Frontend | Vitest + RTL | components, hooks |
| E2E | Playwright | demo flow (login → match → schedule → offer → dashboard) |
| Load | Locust (light) | dashboard & matching endpoints |
Coverage goal: ≥ 70% backend core services.

## 6. Definition of Done
Code reviewed · tests passing · OpenAPI updated · migrations included · no secrets committed · UI has loading/empty/error states · accessibility check done · demo-ready seed data covers the feature.

## 7. Compressed Hackathon Timeline (e.g., 48–72 hrs)
| Block | Deliverable |
|---|---|
| H0–6 | Setup, auth, models, seed script |
| H6–18 | Readiness + skill gap + student UI |
| H18–32 | Drives, JD parsing, matching + explanations |
| H32–42 | Scheduler + conflict detection |
| H42–52 | Notifications + offers + docs |
| H52–62 | Dashboard + at-risk model |
| H62–72 | Evaluation, polish, deployment, demo rehearsal |
Cut-line if behind: drop chatbot/mock interview first, then SHAP factors, then email (keep in-app).

## 8. Risks & Mitigations
| Risk | Impact | Mitigation |
|---|---|---|
| Groq rate limits/outage during demo | High | Cache outputs, template fallbacks, pre-generate demo explanations |
| LLM returns malformed JSON | Med | JSON mode + Pydantic validation + one retry + fallback |
| Neon cold-start latency | Low | Pooled connection, warm-up ping before demo |
| Scope creep | High | MoSCoW, weekly cut review |
| Poor model accuracy on small data | Med | Larger synthetic set with realistic correlations; report honest metrics; use simple interpretable models |
| Prompt injection via resume/JD | Med | Delimit untrusted text, strict output schema, no tool execution from LLM |
| PII exposure to LLM | Med | Strip contact info before sending |
| Embedding model heavy on small host | Low | Precompute in seed; TF-IDF fallback |

## 9. Milestones & Demo Checkpoints
| Milestone | Proof |
|---|---|
| M1 | Role-based login + student readiness page |
| M2 | Drive created from JD with ranked, explained shortlist |
| M3 | Conflict detected and auto-resolved |
| M4 | Offer lifecycle with notifications |
| M5 | Command dashboard + at-risk list |
| M6 | Deployed system + evaluation report + final demo |

## 10. Branching & Release
- `main` (protected, deployable) ← `develop` ← `feature/*`.
- PR template with checklist; squash merge; semantic version tags (`v0.1.0` per milestone).
- Neon branch per environment; migrations applied via CI on deploy.

---

<div align="center">

[← API Specification](06_api_specification.md) &nbsp;·&nbsp; [Setup Guide →](08_setup_guide.md)

<sub>CampusLink · Document 07 of 09 · Version 1.0 · Confidential draft</sub>

</div>
