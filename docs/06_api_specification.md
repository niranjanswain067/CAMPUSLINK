<div align="center">

# CampusLink

**AI-Powered Campus-to-Corporate Placement Management & Analytics Platform**

### 06 · API Specification

</div>

| | |
|---|---|
| **Document** | 06 of 09 — REST endpoints, roles, payloads and error formats |
| **Version** | 1.0 |
| **Date** | 1 November 2026 |
| **Status** | Draft for review |
| **Technology** | React · Django (DRF) · Neon DB (PostgreSQL) · Groq API · JWT |

---

## Table of Contents

1. [1. Conventions](#1-conventions)
2. [2. Authentication](#2-authentication)
3. [3. Students & Profile](#3-students--profile)
4. [4. Readiness & Skill Gap](#4-readiness--skill-gap)
5. [5. Companies & Recruiters](#5-companies--recruiters)
6. [6. Drives](#6-drives)
7. [7. Matching](#7-matching)
8. [8. Scheduling](#8-scheduling)
9. [9. Offers & Documents](#9-offers--documents)
10. [10. Notifications](#10-notifications)
11. [11. Analytics & Prediction](#11-analytics--prediction)
12. [12. AI Assistant](#12-ai-assistant)
13. [13. Admin & System](#13-admin--system)
14. [14. Webhooks / Events (internal)](#14-webhooks--events-internal)
15. [15. Versioning & Compatibility](#15-versioning--compatibility)

---

**Base URL:** `/api/v1/` · **Format:** JSON · **Auth:** `Authorization: Bearer <access_token>`
**Docs:** auto-generated OpenAPI at `/api/schema/`, Swagger UI at `/api/docs/` (drf-spectacular).

## 1. Conventions

### 1.1 Roles
`ADMIN`, `TPO`, `STUDENT`, `RECRUITER`, `MENTOR` — each endpoint lists allowed roles. All data is campus-scoped (Admin sees all).

### 1.2 Pagination, Filtering, Sorting
`?page=1&page_size=20&ordering=-created_at&search=term` + resource-specific filters. Response envelope for lists:
```json
{ "count": 120, "next": "…?page=2", "previous": null, "results": [ ... ] }
```

### 1.3 Error Format
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Invalid input", "details": { "cgpa": ["Must be between 0 and 10"] } } }
```
| HTTP | Code | Meaning |
|---|---|---|
| 400 | VALIDATION_ERROR | Bad input |
| 401 | NOT_AUTHENTICATED / TOKEN_EXPIRED | Missing/expired JWT |
| 403 | PERMISSION_DENIED | Role/campus violation |
| 404 | NOT_FOUND | |
| 409 | CONFLICT | Scheduling or state conflict |
| 429 | RATE_LIMITED | |
| 502/503 | AI_UNAVAILABLE | Groq failure (fallback result usually returned instead) |

---

## 2. Authentication
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/auth/register/` | Public (students self-register with campus code; others by Admin) | Create account |
| POST | `/auth/login/` | Public | Obtain tokens |
| POST | `/auth/refresh/` | Public | Rotate refresh, new access |
| POST | `/auth/logout/` | Auth | Blacklist refresh token |
| GET | `/auth/me/` | Auth | Current user + role + campus |
| POST | `/auth/password/change/` | Auth | Change password |
| POST | `/auth/password/reset/` | Public | Send reset email |
| POST | `/auth/password/reset/confirm/` | Public | Complete reset |

**POST `/auth/login/`**
```json
// request
{ "email": "student@demo.edu", "password": "••••••••" }
// 200
{ "access": "eyJ…", "refresh": "eyJ…",
  "user": { "id": 12, "email": "student@demo.edu", "full_name": "Asha Rao", "role": "STUDENT", "campus_id": 1 } }
```
JWT claims: `user_id, role, campus_id, exp, jti`.

---

## 3. Students & Profile
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/students/` | TPO, ADMIN, MENTOR | List with filters: `branch, grad_year, status, band, min_cgpa, skill` |
| GET | `/students/{id}/` | TPO, MENTOR, owner | Full profile |
| GET/PATCH | `/students/me/` | STUDENT | Own profile |
| POST | `/students/me/resume/` | STUDENT | Upload resume (multipart) |
| POST | `/students/me/resume/parse/` | STUDENT | AI-parse resume → proposed fields (not saved until confirmed) |
| POST | `/students/me/resume/parse/confirm/` | STUDENT | Save parsed data |
| GET/POST | `/students/{id}/skills/` | owner, TPO | Skills list/add |
| PATCH/DELETE | `/students/{id}/skills/{skill_id}/` | owner, TPO | Update proficiency / remove |
| GET/POST | `/students/{id}/certifications/` | owner, TPO | |
| GET/POST | `/students/{id}/projects/` | owner, TPO | |
| GET/POST | `/students/{id}/assessments/` | TPO (write), owner (read) | Add aptitude/communication/soft-skill scores |
| POST | `/students/import/` | TPO | Bulk CSV import |
| GET | `/skills/` | Auth | Skill catalogue (`?q=`) |

**POST `/students/me/resume/parse/`** → 200
```json
{ "extracted": {
    "skills": [{"name":"Python","proficiency":4},{"name":"React","proficiency":3}],
    "projects": [{"title":"Smart Attendance","tech_stack":["OpenCV","Django"],"description":"…"}],
    "certifications": [{"name":"AWS Cloud Practitioner","issuer":"AWS"}],
    "summary": "…" },
  "ai_fallback": false }
```

---

## 4. Readiness & Skill Gap
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/students/{id}/readiness/` | owner, TPO, MENTOR | Latest score + breakdown |
| POST | `/students/{id}/readiness/recompute/` | owner, TPO | Recompute now |
| GET | `/students/{id}/readiness/history/` | owner, TPO | Score trend |
| GET | `/students/{id}/skill-gap/?role=Backend%20Developer` | owner, TPO | Gap vs. target role |
| POST | `/students/{id}/prep-plan/` | owner | AI prep plan from gaps |
| GET | `/job-roles/` | Auth | Role templates (required skills) |

**GET `/students/12/readiness/`** → 200
```json
{ "student_id": 12, "score": 72.4, "band": "READY",
  "breakdown": {
    "academics":   {"value": 78, "weight": 0.25, "contribution": 19.5},
    "technical":   {"value": 70, "weight": 0.25, "contribution": 17.5},
    "aptitude":    {"value": 74, "weight": 0.15, "contribution": 11.1},
    "mock_interview": {"value": 65, "weight": 0.15, "contribution": 9.75},
    "soft_skills": {"value": 72, "weight": 0.10, "contribution": 7.2},
    "projects_certs": {"value": 74, "weight": 0.10, "contribution": 7.4} },
  "top_gaps": ["Cloud technologies", "Mock interview performance"],
  "model_version": "readiness-v1", "computed_at": "2026-11-01T10:00:00Z" }
```

**GET `/students/12/skill-gap/?role=Cloud%20Engineer`** → 200
```json
{ "role": "Cloud Engineer", "coverage_pct": 55,
  "gaps": [ {"skill":"AWS","required":4,"current":1,"gap":3,"resources":["AWS Cloud Practitioner course"]},
            {"skill":"Docker","required":3,"current":0,"gap":3,"resources":[…]} ],
  "strengths": ["Python","Linux"] }
```

---

## 5. Companies & Recruiters
| Method | Endpoint | Roles |
|---|---|---|
| GET/POST | `/companies/` | TPO, ADMIN (write); all auth (read) |
| GET/PATCH | `/companies/{id}/` | TPO, ADMIN |
| GET | `/companies/{id}/history/` | TPO — past drives, hires, avg CTC, repeat-hiring |
| GET/POST | `/recruiters/` | TPO, ADMIN |

---

## 6. Drives
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/drives/` | All (students see eligible/public) | Filters: `status, company, date_from, date_to, mine` |
| POST | `/drives/` | TPO, RECRUITER | Create (DRAFT) |
| GET/PATCH | `/drives/{id}/` | TPO, owner recruiter | |
| POST | `/drives/{id}/parse-jd/` | TPO, RECRUITER | AI JD → structured requirements |
| PUT | `/drives/{id}/eligibility/` | TPO, RECRUITER | Set branches, CGPA, backlogs |
| PUT | `/drives/{id}/skills/` | TPO, RECRUITER | Required/preferred skills |
| POST | `/drives/{id}/publish/` | TPO | Publish (blocked if blocking conflicts exist) |
| POST | `/drives/{id}/cancel/` | TPO | |
| GET | `/drives/{id}/eligible-students/` | TPO | Hard-filter result |
| POST | `/drives/{id}/apply/` | STUDENT | Apply |
| GET | `/students/me/drives/` | STUDENT | Eligible/applied with fit summary |

**POST `/drives/5/parse-jd/`** → 200
```json
{ "parsed_jd": {
    "role_category": "Backend",
    "required_skills": ["Python","Django","SQL"],
    "preferred_skills": ["Docker","AWS"],
    "experience_level": "Fresher",
    "responsibilities": ["…"],
    "suggested_eligibility": {"min_cgpa": 7.0, "branches": ["CSE","IT"]} },
  "ai_fallback": false }
```

---

## 7. Matching
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/drives/{id}/match/` | TPO, RECRUITER | Run matching (async if > 200 students → returns `task_id`) |
| GET | `/drives/{id}/matches/` | TPO, RECRUITER | Ranked list. Filters: `decision, min_fit, branch` |
| GET | `/drives/{id}/matches/{student_id}/` | TPO, RECRUITER, student(own) | Full explanation |
| POST | `/drives/{id}/shortlist/auto/` | TPO | Auto-shortlist ≥ threshold |
| POST | `/drives/{id}/shortlist/override/` | TPO | `{student_id, decision, reason}` (reason required, audited) |
| PATCH | `/drives/{id}/threshold/` | TPO | Change fit threshold |
| GET | `/drives/{id}/matches/export/` | TPO | CSV |
| GET | `/tasks/{task_id}/` | Auth | Async task status |

**GET `/drives/5/matches/12/`** → 200
```json
{ "drive_id": 5, "student_id": 12, "eligible": true, "fit_score": 58.3,
  "fit_band": "DEVELOPING", "decision": "BELOW_THRESHOLD", "rank": 41,
  "factors": { "skill_match": 0.45, "semantic": 0.62, "proj_cert": 0.5, "interview": 0.4, "readiness": 0.72 },
  "matched_skills": ["Python","SQL"], "missing_skills": ["AWS","Docker"],
  "explanation": "Below Threshold: The student's CGPA meets the eligibility criteria, but the required skill set shows a gap in cloud technologies and the mock-interview score is below the recruiter's expected benchmark.",
  "explanation_source": "LLM", "model_version": "fit-v1" }
```

---

## 8. Scheduling
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET/POST | `/venues/` | TPO | |
| GET/POST | `/panels/` | TPO | |
| GET/POST | `/drives/{id}/rounds/` | TPO, RECRUITER (own) | Create/list rounds |
| PATCH/DELETE | `/rounds/{id}/` | TPO | Move/edit (re-runs conflict detection) |
| GET | `/schedule/calendar/?from=&to=&view=venue\|panel\|drive` | TPO | Calendar data |
| POST | `/schedule/check/` | TPO | Dry-run conflict check for a proposed round |
| GET | `/schedule/conflicts/` | TPO | All open conflicts |
| POST | `/rounds/{id}/suggest/` | TPO | Alternative slots/venues/panels |
| POST | `/rounds/{id}/apply-suggestion/` | TPO | Apply chosen suggestion |

**POST `/schedule/check/`**
```json
// request
{ "drive_id": 5, "round_type": "TECHNICAL", "start_at": "2026-11-10T10:00:00+05:30",
  "end_at": "2026-11-10T13:00:00+05:30", "venue_id": 2, "panel_id": 3 }
// 200
{ "has_conflicts": true,
  "conflicts": [
    {"type":"VENUE_DOUBLE_BOOKING","severity":"BLOCKING","entities":{"venue_id":2,"round_id":31},
     "message":"Seminar Hall A is already booked by TCS Round 1 (10:30–12:30)."},
    {"type":"STUDENT_OVERLAP","severity":"WARNING","entities":{"student_ids":[12,18,44]},
     "message":"3 shortlisted students are also in Infosys Technical (11:00–12:00)."} ] }
```
Conflict types: `VENUE_DOUBLE_BOOKING`, `PANEL_DOUBLE_BOOKING`, `SLOT_OVERLAP`, `STUDENT_OVERLAP`, `CAPACITY_EXCEEDED`, `RESOURCE_UNAVAILABLE`.

**POST `/rounds/31/suggest/`** → 200
```json
{ "suggestions": [
   {"rank":1,"start_at":"2026-11-10T14:00:00+05:30","end_at":"…","venue_id":2,"panel_id":3,"conflicts":0,"shift_minutes":240,"reason":"Same venue and panel, no student overlap"},
   {"rank":2,"venue_id":4,"start_at":"…(same time)","conflicts":0,"shift_minutes":0,"reason":"Different venue (capacity 120), same time"} ] }
```

---

## 9. Offers & Documents
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/offers/` | TPO, RECRUITER (own), STUDENT (own) | Filters: `status, company, branch, type` |
| POST | `/offers/` | TPO, RECRUITER | Issue offer (`application_id, ctc, role_title, joining_date, …`) |
| GET/PATCH | `/offers/{id}/` | TPO, RECRUITER | |
| POST | `/offers/{id}/letter/` | TPO, RECRUITER | Upload letter → SHA-256 stored |
| GET | `/offers/{id}/verify/?hash=` | Auth | Compare hash |
| POST | `/offers/{id}/respond/` | STUDENT | `{action: ACCEPT\|DEFER\|DECLINE, note, deferred_until}` |
| POST | `/offers/{id}/withdraw/` | RECRUITER, TPO | |
| POST | `/offers/{id}/mark-joined/` | TPO | |
| GET | `/offers/{id}/events/` | TPO, owner | Timeline |
| GET/POST | `/offers/{id}/documents/` | owner, TPO | Checklist |
| PATCH | `/offers/{id}/documents/{doc_id}/` | owner (submit), TPO (verify/reject) | |
| POST | `/offers/{id}/remind/` | TPO | Send reminder now |
| GET | `/offers/pending-followups/` | TPO, RECRUITER | Pending offers needing follow-up |

Valid status transitions: `ISSUED→ACCEPTED|DEFERRED|DECLINED|WITHDRAWN`, `DEFERRED→ACCEPTED|DECLINED|WITHDRAWN`, `ACCEPTED→JOINED|WITHDRAWN`. Others → `409 CONFLICT`.

---

## 10. Notifications
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/notifications/` | Auth | Own (`?unread=true`) |
| POST | `/notifications/{id}/read/` | Auth | |
| POST | `/notifications/read-all/` | Auth | |
| GET/PUT | `/notifications/preferences/` | Auth | |
| POST | `/notifications/broadcast/` | TPO | Drive announcement to filtered audience |
| GET/POST/PATCH | `/notification-templates/` | TPO, ADMIN | |
| GET | `/notifications/log/` | TPO | Delivery log |

---

## 11. Analytics & Prediction
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/analytics/summary/` | TPO, ADMIN | KPI cards (registered, ready, active drives, offers made/accepted/pending, avg/highest CTC) |
| GET | `/analytics/conversion/branch/` | TPO | Branch-wise conversion |
| GET | `/analytics/conversion/skill/` | TPO | Skill-wise conversion |
| GET | `/analytics/packages/trends/` | TPO | Avg/median/highest by year/recruiter/branch |
| GET | `/analytics/recruiters/engagement/` | TPO | Pipeline + repeat hiring |
| GET | `/analytics/funnel/` | TPO | Applied → Shortlisted → Selected → Offered → Accepted → Joined |
| GET | `/analytics/documents/summary/` | TPO | Documentation/joining status |
| GET | `/analytics/readiness/distribution/` | TPO | Band distribution |
| GET | `/analytics/at-risk/` | TPO, MENTOR | At-risk list with probabilities and factors |
| POST | `/analytics/at-risk/recompute/` | TPO | Trigger prediction run |
| POST | `/analytics/at-risk/{student_id}/escalate/` | TPO | Create escalation to mentor |
| GET | `/analytics/models/` | ADMIN, TPO | Model metrics (AUC, precision, recall) |
| GET | `/analytics/campuses/compare/` | ADMIN | Multi-campus aggregation |

Common filters: `campus, branch, year, from, to`.

**GET `/analytics/summary/`** → 200
```json
{ "students_registered": 300, "placement_ready": 142, "active_drives": 3, "upcoming_drives": 2,
  "offers": {"made": 96, "accepted": 71, "pending": 18, "declined": 7},
  "package": {"average": 7.8, "median": 6.5, "highest": 32.0},
  "at_risk_count": 27, "documents": {"pending": 22, "verified": 49}, "joined": 12 }
```

**GET `/analytics/at-risk/`** → 200
```json
{ "results": [ { "student_id": 77, "name": "R. Kumar", "branch": "MECH", "probability": 0.81, "risk_level": "HIGH",
   "top_factors": [ {"factor":"CGPA below drive cut-offs","impact":0.32}, {"factor":"No applications in 30 days","impact":0.21}, {"factor":"Low mock-interview score","impact":0.17} ],
   "suggested_action": "Assign mentor and schedule 2 mock interviews" } ] }
```

---

## 12. AI Assistant
| Method | Endpoint | Roles | Description |
|---|---|---|---|
| POST | `/ai/chat/` | STUDENT, TPO | FAQ/eligibility chatbot `{session_id?, message}` |
| GET | `/ai/chat/{session_id}/` | owner | History |
| POST | `/ai/mock-interview/start/` | STUDENT | `{role, difficulty, num_questions}` → questions |
| POST | `/ai/mock-interview/answer/` | STUDENT | `{question_id, answer}` → feedback & scores |
| POST | `/ai/mock-interview/finish/` | STUDENT | Summary + saves an `assessments` row |
| POST | `/ai/resume/improve/` | STUDENT | Suggestions / rewritten bullets |
| POST | `/ai/explain-score/` | Auth | Plain-language explanation of a score |

**POST `/ai/chat/`**
```json
// request
{ "session_id": "b0c1…", "message": "Am I eligible for the Infosys drive?" }
// 200
{ "session_id": "b0c1…", "reply": "Yes. You meet the CGPA (8.1 ≥ 6.5) and branch criteria. Your fit score is 74 (Ready). Apply before 7 Nov, 5 PM.",
  "sources": [{"type":"drive","id":5}], "ai_fallback": false }
```

AI endpoints use `ScopedRateThrottle` (`ai`: 20/min/user).

---

## 13. Admin & System
| Method | Endpoint | Roles |
|---|---|---|
| GET/POST | `/campuses/` | ADMIN |
| GET/POST/PATCH | `/users/` | ADMIN (TPO for own campus students) |
| GET | `/audit-logs/` | ADMIN, TPO |
| GET | `/health/` | Public — `{status:"ok", db:"ok", redis:"ok"}` |
| POST | `/admin/seed-demo/` | ADMIN (dev only) |

---

## 14. Webhooks / Events (internal)
Domain events (Django signals → Celery): `application.shortlisted`, `round.scheduled`, `round.conflict_detected`, `offer.status_changed`, `document.due_soon`, `student.at_risk`. Each maps to a notification template.

## 15. Versioning & Compatibility
URL versioning (`/v1/`); additive changes only within a version; deprecations announced via `Deprecation` header.

---

<div align="center">

[← Database Schema](05_database_schema.md) &nbsp;·&nbsp; [Development Plan →](07_development_plan.md)

<sub>CampusLink · Document 06 of 09 · Version 1.0 · Confidential draft</sub>

</div>
