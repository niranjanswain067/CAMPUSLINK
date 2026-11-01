<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 01 · Project Requirements Document

</div>

| | |
|---|---|
| **Document** | 01 of 09 — Scope, users, functional and non-functional requirements, acceptance criteria |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [CampusLink — AI-Powered Campus-to-Corporate Placement Management & Analytics Platform](#campuslink--ai-powered-campus-to-corporate-placement-management--analytics-platform)
2. [1. Purpose](#1-purpose)
3. [2. Problem Summary](#2-problem-summary)
4. [3. Goals & Success Metrics](#3-goals--success-metrics)
5. [4. User Roles](#4-user-roles)
6. [5. Functional Requirements](#5-functional-requirements)
7. [6. Non-Functional Requirements](#6-non-functional-requirements)
8. [7. Constraints & Assumptions](#7-constraints--assumptions)
9. [8. Minimum Deliverables Mapping](#8-minimum-deliverables-mapping)
10. [9. Out of Scope (v1)](#9-out-of-scope-v1)
11. [10. Acceptance Criteria (Demo Flow)](#10-acceptance-criteria-demo-flow)

---

## CampusLink — AI-Powered Campus-to-Corporate Placement Management & Analytics Platform

**Domain:** EdTech + AI / Placement & Career Services Automation
**Stack:** React · Django (DRF) · Neon (PostgreSQL) · Groq API · JWT

---

## 1. Purpose
Digitise and intelligently automate the full placement lifecycle: **Profiling → Matching → Scheduling → Notification → Offer Tracking → Analytics**. The system must go beyond record-keeping and provide scoring, explainable matching, conflict-free scheduling and predictive insights.

## 2. Problem Summary
- Placement cells use spreadsheets, email and WhatsApp, causing duplicated effort, missed deadlines and inconsistent communication.
- Students have no single readiness profile, skill-gap analysis or personalised guidance.
- Recruiters receive poorly matched candidate pools.
- Drives clash on dates, venues and shortlisted students.
- Offers and joining statuses are tracked manually; analytics are absent.

## 3. Goals & Success Metrics
| Goal | Metric / Target |
|---|---|
| Accurate readiness scoring | Score bands agree with placement outcome (AUC ≥ 0.75 on simulated data) |
| Good matching | Precision@10 ≥ 0.80, NDCG@10 ≥ 0.85 vs. rule-based ground truth |
| Zero scheduling clashes | 100% of published drives pass conflict check |
| Explainability | Every shortlist decision has a human-readable reason |
| Automation | ≥ 90% of notifications sent without manual action |
| Early risk detection | At-risk model recall ≥ 0.70 for unplaced students |

## 4. User Roles
| Role | Description |
|---|---|
| **Admin** | Manages campuses, users, global settings |
| **Placement Officer (TPO)** | Runs drives, shortlists, tracks offers, views dashboards |
| **Student** | Builds profile, sees readiness score, skill gaps, eligible drives, applies |
| **Recruiter** | Posts JDs, sees ranked candidate pool, schedules interviews, issues offers |
| **Mentor/Faculty** | Receives escalations for at-risk students, records guidance |

## 5. Functional Requirements

### FR-1 Authentication & Access (JWT)
- FR-1.1 Register/login with email + password; JWT access (15 min) + refresh (7 days) tokens.
- FR-1.2 Role-based access control on every endpoint.
- FR-1.3 Token refresh, logout (refresh-token blacklist), password reset.
- FR-1.4 Multi-campus scoping: every record belongs to a campus; users only see their campus (Admin sees all).

### FR-2 Student Readiness Profiling
- FR-2.1 Capture academics (branch, CGPA, backlogs, 10th/12th %), skills, certifications, projects, internships.
- FR-2.2 Resume upload (PDF/DOCX) and **AI resume parsing** (Groq) into structured profile fields.
- FR-2.3 Record aptitude, mock-interview, communication and soft-skill assessment scores.
- FR-2.4 Compute **Readiness Score (0–100)** with band: *Not Ready → Developing → Ready → Highly Employable*.
- FR-2.5 Show score breakdown and top improvement areas.
- FR-2.6 **Skill-gap analysis** against any target job role.
- FR-2.7 Personalised preparation plan (Groq-generated, grounded in the student's gaps).

### FR-3 Recruiter Requirement Matching
- FR-3.1 Recruiter/TPO creates a drive with JD text and eligibility (branches, min CGPA, max backlogs, required skills).
- FR-3.2 **JD parsing** with Groq into structured required/preferred skills, experience level, role category.
- FR-3.3 Hard eligibility filter (branch, CGPA, backlogs, year).
- FR-3.4 **Fit Score (0–100)** per student–drive pair from skills, semantic JD similarity, projects/certs, interview history, readiness.
- FR-3.5 Ranked candidate list with per-candidate explanation (matched skills, gaps, threshold failures).
- FR-3.6 Auto-shortlist at a configurable threshold; manual override with audit log.

### FR-4 Drive Scheduling & Conflict Management
- FR-4.1 Create drives with date, slot, venue, panel and rounds.
- FR-4.2 Detect: same-slot overlapping drives, venue double-booking, panel double-booking, student in simultaneous drives.
- FR-4.3 Show conflicts with severity and reason before publishing.
- FR-4.4 **Auto-resolve**: suggest alternative slots/venues/panels ranked by least disruption.
- FR-4.5 Resolve student conflicts by priority rules (e.g., higher-fit drive first, or reschedule the lower-priority round).

### FR-5 Offer & Documentation Tracking
- FR-5.1 Offer record: CTC, role, location, type (full-time / PPO / internship conversion), joining date.
- FR-5.2 Status flow: *Issued → Accepted / Deferred / Declined / Withdrawn → Joined*.
- FR-5.3 Document checklist per offer (offer letter, bond, ID, marksheets, verification); status per document.
- FR-5.4 Offer-letter upload with SHA-256 hash for tamper checking.
- FR-5.5 Recruiter follow-up prompts for pending offers.

### FR-6 Analytics & Predictive Insights
- FR-6.1 Branch-wise and skill-wise conversion rates.
- FR-6.2 Package trends (average, median, highest) by recruiter, branch, year.
- FR-6.3 **At-risk prediction**: probability a student remains unplaced, with top contributing factors.
- FR-6.4 Recruiter engagement and repeat-hiring patterns.
- FR-6.5 Offer/documentation/joining funnel.

### FR-7 Communication & Notifications
- FR-7.1 Event-driven notifications: shortlisting, interview schedule, document deadlines, offer updates, drive announcements.
- FR-7.2 Channels: in-app (required), email (required), optional WhatsApp/SMS hook.
- FR-7.3 Automated reminders for pending documents and deadlines.
- FR-7.4 Escalation of at-risk students to mentors.
- FR-7.5 User notification preferences and delivery log.

### FR-8 Placement Command Dashboard
Interactive dashboard with: registered vs. placement-ready students, active/upcoming drives, offers (made/accepted/pending), branch/skill conversion, package trends, at-risk list, recruiter pipeline, documentation/joining summary. Filters: campus, branch, year, date range.

### FR-9 AI Innovation Features (Groq-powered)
- Student FAQ / eligibility **chatbot** (grounded in drives and student data).
- **Mock-interview** question generation and answer scoring (communication, relevance, structure).
- **Resume / profile builder** assistance.
- Explainable shortlist reasons (LLM narrates deterministic score factors; LLM never decides the score alone).

## 6. Non-Functional Requirements
| Category | Requirement |
|---|---|
| Performance | API p95 < 500 ms (non-AI); dashboard load < 2 s; AI calls < 5 s |
| Scalability | Support 50 campuses, 100k students, stateless API for horizontal scaling |
| Security | JWT, RBAC, bcrypt/PBKDF2 hashing, HTTPS, CORS allow-list, rate limiting, input validation, audit log |
| Privacy | Student data restricted by role/campus; PII not sent to Groq beyond what the task needs (strip phone/email/address) |
| Reliability | Graceful fallback when Groq is unavailable (rule-based explanations) |
| Explainability | Every score has stored factor breakdown |
| Usability | Responsive UI; WCAG-AA contrast; works on mobile browsers |
| Observability | Structured logs, error tracking, health endpoint |

## 7. Constraints & Assumptions
- Database is **Neon serverless PostgreSQL** (use pooled connection string; `pgvector` extension available).
- Groq API provides LLM inference only (no embedding endpoint) → semantic similarity uses a local embedding model or TF-IDF fallback.
- Demo uses **simulated data** (Faker) plus the public Kaggle *Campus Recruitment* dataset for model training.
- Prototype scope: web app. Mobile app, blockchain verification and LinkedIn integration are stretch goals.

## 8. Minimum Deliverables Mapping
| # | Deliverable | Where |
|---|---|---|
| 1 | Working prototype | Full stack |
| 2 | Readiness scoring | FR-2, doc 09 §Readiness |
| 3 | Matching for ≥3 simulated drives | Seed script creates 4 drives |
| 4 | Conflict-aware scheduling | FR-4 |
| 5 | Explainable shortlisting | FR-3.5, FR-9 |
| 6 | Dashboard | FR-8 |
| 7 | Offer & document tracking | FR-5 |
| 8 | Architecture | doc 02 |
| 9 | Models/algorithms | doc 09 |
| 10 | Simulated dataset demo | doc 08 seed step |
| 11 | Accuracy evaluation | doc 09 §Evaluation |
| 12 | Scalability/deployment | doc 02 §Deployment |

## 9. Out of Scope (v1)
Payments, video-interview hosting, native mobile apps, blockchain, LinkedIn/job-portal APIs (documented as future work).

## 10. Acceptance Criteria (Demo Flow)
1. Student logs in, uploads resume, sees readiness score and skill gaps.
2. TPO creates drive from JD; system parses JD and ranks students with explanations.
3. Scheduler flags a venue/panel/student conflict and proposes a fix.
4. Shortlisted students receive notifications.
5. Recruiter issues offer; student accepts; documents tracked to "Joined".
6. Dashboard reflects all of the above and shows at-risk students.

---

<div align="center">

[System Architecture →](02_system_architecture.md)

<sub>CampusLink · Document 01 of 09 · Version 1.0 · Confidential draft</sub>

</div>
