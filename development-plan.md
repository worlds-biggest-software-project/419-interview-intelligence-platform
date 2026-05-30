# Interview Intelligence Platform — Phased Development Plan

> Project: 419-interview-intelligence-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation roadmap. The database design is based on **Data Model Suggestion 3 (Hybrid Relational + JSONB)** — it provides the referential integrity and SQL tooling of a normalised relational model (Suggestion 1) for the structured core, while accommodating the semi-structured, AI-generated, and customer-varying data (competency rubrics, AI evidence, coaching signals, integration payloads) in JSONB columns, without the operational overhead of event sourcing (Suggestion 2) or polyglot graph persistence (Suggestion 4). An append-only `audit_log` and immutable decision records deliver the regulatory audit trail that the event-sourced approach would provide, at a fraction of the complexity.

---

## Core Requirements (synthesis)

**What it does.** An AI-native, open-source platform that records and transcribes job interviews, tags candidate statements as evidence against competency frameworks, enforces structured scoring, detects bias across interviewers and demographic groups, flags legally problematic questions in real time, and produces auditable hiring decisions — closing the loop to post-hire performance. It is the only open, inspectable alternative to BrightHire/HireVue/Metaview, positioned for organisations facing EU AI Act, NYC LL144, and Colorado SB 24-205 audit obligations.

**Primary users.** Recruiters (run pipelines, push to ATS), hiring managers (review scorecards, make decisions), interviewers (use kits, score, receive coaching), TA leaders / compliance officers (bias audits, defensibility).

**Key differentiators.** Open-source and auditable bias detection (vs. black-box incumbents); first-class compliance API surface (LL144 impact-ratio export, candidate notifications, GDPR erasure); SMB-friendly self-hostable deployment; quality-of-hire closed loop; real-time interviewer coaching.

**MVP feature set** (from features.md "Must-have"): recording + transcription with diarisation; structured interview kit builder; AI note/summary generation; candidate scorecard with evidence tagging; ATS integration (Greenhouse, Lever, Workday via unified layer); comparative candidate evaluation; audit logs + compliance documentation; real-time compliance flagging of discriminatory questions.

**Post-MVP** (v1.1 / backlog): AI interview-guide generation from JDs; bias detection across demographic groups; interviewer effectiveness analytics; multi-modal (async video) interviews; ICP learning; calibration-session coaching; HRIS quality-of-hire loop; candidate transparency; panel consistency; industry kit templates; QoH predictive model.

**Deployment model.** Self-hosted, cloud, and hybrid — Docker Compose for self-host, same image deployable to managed cloud. Data-residency / sovereignty configurable.

**Integration surface.** Recording (Recall.ai for unified Zoom/Teams/Meet bot capture); transcription (AssemblyAI async for accuracy, Deepgram streaming for live coaching); ATS (Merge.dev unified API for MVP breadth, direct Greenhouse Harvest as reference direct integration); LLM provider (abstracted — Anthropic/OpenAI/local); HRIS (Merge.dev HRIS for QoH); SSO (OIDC/SAML); object storage (S3-compatible).

**Standards compliance.** WebVTT transcripts; OpenAPI 3.1 + JSON Schema 2020-12 for the public API; OAuth 2.0 / OIDC / SAML 2.0 / SCIM 2.0 for auth; RFC 7807 problem-details errors; HMAC-SHA256 webhook verification; NIST AI RMF MEASURE 2.11 for bias metrics; EEOC four-fifths rule for impact ratios; ISO/IEC 42001 + 27001 + 27701 alignment; MCP server for AI-agent context access.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The platform is LLM- and ML-heavy (competency tagging, bias detection, summary generation, QoH modelling). Python has first-class SDKs for every dependency we need (AssemblyAI, Deepgram, Anthropic, Merge, Recall.ai) and the richest data/ML ecosystem (pandas, scikit-learn, numpy) for the bias and analytics work. |
| API framework | **FastAPI** | Generates OpenAPI 3.1 (JSON Schema 2020-12) automatically — a hard requirement from standards.md for ATS partner integrations. Native async suits the I/O-bound integration workload. Pydantic v2 models double as request/response schemas and validation. |
| Data validation | **Pydantic v2** | Enforces JSON Schema on the JSONB-stored semi-structured data (rubrics, evidence, coaching signals) per Suggestion 3's "validate JSONB" principle, and defines wire formats. |
| Database | **PostgreSQL 16** | Per Suggestion 3: relational core + JSONB periphery in one engine. Gives FK integrity, RLS multi-tenancy, `tsvector` transcript search, table partitioning for audit/transcript volume, and `pgvector` for semantic transcript/evidence search — without a second datastore. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature async ORM; Alembic gives versioned, reviewable migrations (compliance needs a migration history). |
| Vector search | **pgvector** | Embeddings for semantic transcript search and "find similar evidence"; avoids a separate vector DB at MVP scale. |
| Task queue | **Celery + Redis** | Transcription, AI tagging, summary generation, ATS sync, and bias-report generation are long-running and webhook-triggered. Celery handles retries/backoff; Redis doubles as broker and live-interview state cache (Suggestion 2's Redis read-model idea, reused minimally). |
| Real-time transport | **WebSockets (FastAPI) + Redis pub/sub** | Live transcript stream, talk-ratio, and coaching prompts pushed to the interviewer UI during interviews. |
| Recording capture | **Recall.ai API** | standards.md identifies it as the most practical path to unified Zoom/Teams/Meet/in-person bot recording with managed consent — avoids maintaining three meeting SDKs. Abstracted behind a `RecordingProvider` interface. |
| Transcription | **AssemblyAI (async) + Deepgram (streaming)** | AssemblyAI for highest-accuracy post-interview transcripts + diarisation + PII redaction; Deepgram Nova-3 streaming (~300ms) for live coaching. Both behind a `TranscriptionProvider` interface. |
| LLM access | **Provider-abstracted LLM client** (Anthropic default; OpenAI + local/Ollama supported) | AI-native features need model flexibility and self-host requires a local option. Single `LLMClient` interface; model + provider configurable per deployment. |
| ATS / HRIS integration | **Merge.dev unified API (primary) + direct Greenhouse Harvest (reference)** | Merge covers Greenhouse, Lever, Workday, Ashby, iCIMS, SmartRecruiters with one normalised schema (BrightHire's production pattern). One direct Harvest integration proves the direct path and validates the abstraction. |
| Frontend | **Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui** | Dashboard-heavy product (kit builder, live interview view, scorecard comparison, bias dashboard). Server components for data-heavy pages; client components for the live interview WebSocket UI. |
| Auth | **OIDC + SAML 2.0 + SCIM 2.0**; sessions via JWT (RFC 7519) | Enterprise SSO is mandatory (standards.md). `authlib` for OIDC/OAuth, `python3-saml` for SAML, custom SCIM 2.0 endpoints. |
| Containerisation | **Docker + Docker Compose** | Self-hosted is a primary deployment mode; Compose bundles API, worker, Postgres, Redis, and frontend for one-command self-host. |
| Object storage | **S3-compatible (boto3)** — AWS S3 / MinIO | Recordings + transcripts; MinIO in Compose for self-host, S3 for cloud. Client-side AES-256-GCM envelope encryption with per-recording keys. |
| Secrets / KMS | **Pluggable: env + HashiCorp Vault / AWS KMS** | Recording encryption keys and ATS credentials; KMS-backed crypto-shredding for GDPR erasure of recordings. |
| Testing | **pytest + pytest-asyncio + httpx + testcontainers** | Unit + integration; testcontainers spins real Postgres/Redis for integration tests. Frontend: Vitest + Playwright. |
| Code quality | **ruff (lint+format), mypy (strict), pyright** | Fast lint/format; strict typing on a compliance-sensitive codebase. Frontend: ESLint + Prettier + tsc. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, reproducible installs. |
| MCP server | **Python MCP SDK** | Exposes transcripts, scorecards, competency frameworks, and kit-generation as AI-agent context sources (standards.md). |

### Project Structure

```
interview-intelligence-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile                      # API + worker image
├── docker-compose.yml              # api, worker, postgres, redis, minio, frontend
├── docker-compose.override.yml     # local dev overrides
├── .env.example
├── alembic.ini
├── README.md
├── openapi/                        # exported OpenAPI 3.1 spec (CI artifact)
│   └── openapi.json
├── migrations/                     # Alembic migrations
│   └── versions/
├── src/
│   └── iip/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router registration
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py             # SQLAlchemy declarative base, session
│       │   ├── models/             # ORM models (one file per aggregate area)
│       │   │   ├── org.py          # organisations, users, teams, team_members
│       │   │   ├── competency.py   # frameworks, dimensions, scoring_levels
│       │   │   ├── kit.py          # interview_kits, sections, questions
│       │   │   ├── candidate.py    # candidates, applications, jobs
│       │   │   ├── interview.py    # interviews, panelists, recordings, transcripts, segments
│       │   │   ├── scoring.py      # scores, evidence, notes, compliance_flags
│       │   │   ├── decision.py     # scorecards, entries, hiring_decisions, decision_evidence
│       │   │   ├── analytics.py    # interviewer_metrics, bias_audit_reports, post_hire_outcomes
│       │   │   └── integration.py  # ats_integrations, audit_log, webhook_events
│       │   └── rls.py              # row-level-security policy helpers
│       ├── schemas/                # Pydantic request/response + JSONB validation models
│       ├── api/
│       │   ├── deps.py             # auth, db session, current-org dependencies
│       │   ├── errors.py           # RFC 7807 problem-details handlers
│       │   └── routes/             # one router per resource (kits, interviews, scorecards…)
│       ├── auth/                   # oidc.py, saml.py, scim.py, jwt.py, rbac.py
│       ├── services/               # business logic (kit_service, scoring_service, …)
│       ├── integrations/
│       │   ├── recording/          # base.py (RecordingProvider), recall.py
│       │   ├── transcription/      # base.py, assemblyai.py, deepgram.py
│       │   ├── ats/                # base.py (ATSProvider), merge.py, greenhouse.py
│       │   ├── hris/               # base.py, merge.py
│       │   └── llm/                # base.py (LLMClient), anthropic.py, openai.py, ollama.py
│       ├── ai/
│       │   ├── prompts/            # versioned prompt templates
│       │   ├── competency_tagger.py
│       │   ├── summariser.py
│       │   ├── compliance_detector.py
│       │   ├── kit_generator.py
│       │   └── coach.py
│       ├── analytics/
│       │   ├── interviewer_metrics.py
│       │   ├── bias.py             # four-fifths, impact ratios, NIST AI RMF metrics
│       │   └── quality_of_hire.py
│       ├── realtime/               # ws.py (WebSocket), live_state.py (Redis)
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/              # transcription, tagging, summary, ats_sync, bias_report
│       ├── compliance/             # consent.py (jurisdiction rules), erasure.py, ll144.py
│       ├── storage/                # s3.py, encryption.py (envelope/crypto-shred)
│       └── mcp/                    # server.py (MCP tools)
├── frontend/
│   ├── package.json
│   ├── app/                        # Next.js App Router
│   │   ├── (dashboard)/            # kits, candidates, interviews, scorecards, bias, settings
│   │   └── interview/[id]/live/    # live interview WebSocket UI
│   ├── components/                 # shadcn/ui-based components
│   └── lib/                        # api client (generated from OpenAPI), ws client
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                   # sample VTT, transcripts, ATS payloads, JDs
└── scripts/
    └── seed.py                     # demo org, kits, candidates for local dev
```

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish the runnable skeleton: FastAPI app, config, database connection, multi-tenant data layer base, RFC 7807 error handling, Docker Compose, and CI. After this phase the app boots, connects to Postgres/Redis, serves a health check and an empty OpenAPI spec, and `docker compose up` brings the full stack online. Everything else builds on this.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Initialise the Python package, dependency management, linting, typing, and CI.

**Design**:
- `pyproject.toml` with `uv`, dependencies pinned: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]>=2.0`, `asyncpg`, `alembic`, `pydantic>=2`, `pydantic-settings`, `celery`, `redis`, `pgvector`, `boto3`, `httpx`, `python-jose`/`authlib`, dev: `pytest`, `pytest-asyncio`, `httpx`, `testcontainers`, `ruff`, `mypy`.
- `ruff.toml` (line length 100, select E,F,I,UP,B), `mypy` strict in `pyproject.toml`.
- `config.py` using `pydantic_settings.BaseSettings`:

```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    s3_endpoint: str | None = None
    s3_bucket: str = "iip-recordings"
    llm_provider: Literal["anthropic", "openai", "ollama"] = "anthropic"
    llm_model: str = "claude-sonnet-4-6"
    recording_provider: Literal["recall"] = "recall"
    transcription_async_provider: Literal["assemblyai"] = "assemblyai"
    transcription_stream_provider: Literal["deepgram"] = "deepgram"
    ats_provider: Literal["merge", "greenhouse"] = "merge"
    jwt_secret: str
    jwt_ttl_seconds: int = 3600
    default_recording_retention_days: int = 365
    model_config = SettingsConfigDict(env_prefix="IIP_", env_file=".env")
```

- GitHub Actions workflow: `ruff check`, `ruff format --check`, `mypy src`, `pytest`, export OpenAPI to `openapi/openapi.json` as artifact.

**Testing**:
- `Unit: Settings loads from env with IIP_ prefix → fields populated, defaults applied`
- `Unit: missing required IIP_DATABASE_URL → ValidationError naming the field`
- `CI: ruff/mypy/pytest run green on empty skeleton`

#### 1.2 — App factory, health check, RFC 7807 errors

**What**: FastAPI app factory with health endpoint and standardised problem-details error responses.

**Design**:
- `main.py`: `create_app() -> FastAPI`; registers exception handlers, routers, CORS, OpenAPI metadata (`title`, `version`, `openapi_version="3.1.0"`).
- `api/errors.py`: RFC 7807 model and handlers:

```python
class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    errors: list[dict] | None = None  # validation field errors

class AppError(Exception):
    def __init__(self, status: int, title: str, detail: str | None = None, type_: str = "about:blank"): ...
```

- Handlers for `AppError`, `RequestValidationError` (→ 422 with `errors[]`), and unhandled `Exception` (→ 500, log with correlation id). Response content type `application/problem+json`.
- `GET /healthz` → `{"status":"ok","db":<bool>,"redis":<bool>}`.

**Testing**:
- `Unit: GET /healthz with DB+Redis up → 200, both true`
- `Integration (mocked): DB down → /healthz db:false, status still 200`
- `Unit: raising AppError(404,"Not Found") → application/problem+json body with status 404`
- `Unit: invalid request body → 422 problem+json with errors[] listing field`

#### 1.3 — Database layer, base models, RLS, Alembic

**What**: Async SQLAlchemy session management, declarative base with common columns, row-level-security helpers, and the initial Alembic migration creating `organisations`, `users`, `teams`, `team_members`.

**Design**:
- `db/base.py`: async engine, `async_sessionmaker`, `Base(DeclarativeBase)` with mixin providing `id: Mapped[UUID]`, `created_at`, `updated_at` (server defaults).
- Models in `db/models/org.py` mirror Suggestion 1's `organisations`, `users` (role CHECK in {admin,recruiter,hiring_manager,interviewer,viewer}), `teams`, `team_members`. `organisations.settings JSONB` and `users` extended attributes stored relationally; org config (timezone, consent defaults, retention) in `settings` JSONB validated by a Pydantic `OrgSettings` model (Suggestion 3).
- `db/rls.py`: helper to `SET app.current_org_id` per request transaction; migration emits `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + policy `USING (organisation_id = current_setting('app.current_org_id')::uuid)` on every tenant table.
- Alembic env configured for async engine; first migration `0001_org_users`.

**Testing**:
- `Integration (testcontainers Postgres): run migrations → tables exist with expected columns/constraints`
- `Integration: insert user with role 'banana' → CHECK constraint violation`
- `Integration: RLS — set org A, query users → only org A rows visible; org B rows hidden`
- `Unit: OrgSettings JSONB validation rejects unknown retention policy value`

#### 1.4 — Docker Compose stack

**What**: One-command local stack.

**Design**:
- `Dockerfile`: multi-stage (uv build → slim runtime), single image runs API (`uvicorn`) or worker (`celery`) by command.
- `docker-compose.yml`: services `api`, `worker`, `postgres:16` (with pgvector init), `redis:7`, `minio`, `frontend`; healthchecks; `api` depends_on postgres/redis healthy; runs `alembic upgrade head` on start.
- `.env.example` with all `IIP_*` vars.

**Testing**:
- `E2E (CI, optional): docker compose up → /healthz returns 200 with db:true, redis:true within 60s`
- `Integration: migrations applied automatically on api container start`

---

## Phase 2: Identity, Auth & Multi-Tenancy

### Purpose
Add authentication, RBAC, and enterprise identity so every subsequent endpoint can be access-controlled and tenant-scoped. Delivers OIDC/SAML SSO, SCIM provisioning, JWT sessions, and role-based authorisation — prerequisites named in standards.md for enterprise procurement and for OWASP A01 (broken access control).

### Tasks

#### 2.1 — JWT sessions & auth dependencies

**What**: Issue/verify JWT sessions and provide FastAPI dependencies for current user + org.

**Design**:
- `auth/jwt.py`: `create_token(user_id, org_id, role) -> str`, `decode_token(str) -> Claims` (RFC 7519; HS256 with `jwt_secret`, `exp`, `iss="iip"`).
- `api/deps.py`: `get_current_claims` (Bearer header), `get_current_user`, `get_current_org`, `require_role(*roles)` dependency factory. Sets `app.current_org_id` on the session for RLS.

```python
class Claims(BaseModel):
    sub: UUID            # user id
    org: UUID
    role: Role
    exp: int
```

**Testing**:
- `Unit: create_token then decode_token → round-trips claims`
- `Unit: expired token → AppError 401`
- `Unit: tampered signature → AppError 401`
- `Integration: endpoint with require_role('admin'); recruiter token → 403 problem+json`

#### 2.2 — OIDC & SAML SSO

**What**: Enterprise SSO login via OIDC and SAML 2.0.

**Design**:
- `auth/oidc.py` using `authlib`: `/auth/oidc/login` (redirect), `/auth/oidc/callback` (exchange code, verify ID token, upsert user by email within org mapped from issuer/domain, issue platform JWT). OpenID Connect Core 1.0.
- `auth/saml.py` using `python3-saml`: `/auth/saml/metadata`, `/auth/saml/acs` (assertion consumer). Per-org IdP config stored in `organisations.settings.saml`.
- Just-in-time user provisioning with default role from org config.

**Testing**:
- `Integration (mocked IdP): OIDC callback with valid ID token → JWT issued, user upserted`
- `Integration (mocked): OIDC callback with mismatched nonce → 401`
- `Unit: SAML assertion with valid signature → claims extracted; tampered → rejected`

#### 2.3 — SCIM 2.0 provisioning & RBAC matrix

**What**: SCIM endpoints for automated user lifecycle and a documented RBAC permission matrix.

**Design**:
- `auth/scim.py`: `GET/POST /scim/v2/Users`, `GET/PATCH/DELETE /scim/v2/Users/{id}`, `/Groups` (RFC 7643/7644 schemas). Bearer token per org (provisioning secret). PATCH supports `active=false` → deactivate.
- `auth/rbac.py`: capability matrix mapping `(role, action)` → bool (e.g., `hiring_manager` may `decision.create`; `interviewer` may not). Enforced in services.

**Testing**:
- `Integration: SCIM POST /Users → user created, role mapped from group`
- `Integration: SCIM PATCH active=false → user.is_active=false, JWTs rejected thereafter`
- `Unit: rbac.can('interviewer','decision.create') → False; ('hiring_manager','decision.create') → True`

---

## Phase 3: Competency Frameworks & Interview Kits (Core Domain)

### Purpose
Build the structured-interviewing backbone that the entire product depends on: competency frameworks with scoring rubrics, and interview kits (sections, questions, time allocations) mapped to competencies. This is the "structure" half of the value proposition and a prerequisite for scoring, evidence tagging, and bias detection. Ships the kit-builder API.

### Tasks

#### 3.1 — Competency framework model & API

**What**: CRUD for competency frameworks, dimensions, and scoring levels with versioning.

**Design**:
- Models from Suggestion 1: `competency_frameworks` (versioned, `is_published`), `competency_dimensions` (category, `scoring_rubric`), `scoring_levels` (level 1–N, label, description). Per Suggestion 3, the rubric's structured "good/poor evidence" descriptors live in a JSONB `rubric_detail` validated by:

```python
class RubricLevel(BaseModel):
    level: int
    label: str
    description: str
    positive_indicators: list[str]
    negative_indicators: list[str]

class RubricDetail(BaseModel):
    levels: list[RubricLevel]  # ascending, contiguous
```

- Routes: `POST/GET/PATCH /frameworks`, `/frameworks/{id}/dimensions`, publish endpoint freezes a version (creating immutable snapshot for audit). Published frameworks are referenced by jobs/kits; editing a published framework forks a new version.
- Aligns with ISO 10667-2 (validity/fairness of assessment design): rubrics are explicit and versioned.

**Testing**:
- `Unit: RubricDetail with non-contiguous levels → ValidationError`
- `Integration: create framework + dimensions → GET returns nested structure`
- `Integration: publish framework then PATCH dimension → new version created, v1 immutable`
- `Integration: RLS — org B cannot read org A framework`

#### 3.2 — Interview kit builder model & API

**What**: CRUD for interview kits, sections, and questions, mapped to competency dimensions.

**Design**:
- Models from Suggestion 1: `interview_kits` (interview_type, estimated_duration, `is_template`, `industry`), `interview_kit_sections` (time_allocation, sort_order), `interview_questions` (question_text, `follow_up_prompts` array, `dimension_id`, question_type, sort_order).
- Routes: `POST/GET/PATCH/DELETE /kits`, nested section/question management, `POST /kits/{id}/clone` (for templates), `GET /kits?is_template=true&industry=...`.
- Validation: total section time should be ~equal to `estimated_duration` (warning, not error); every question optionally maps to a published dimension.

**Testing**:
- `Integration: create kit with 2 sections, 3 questions each → GET returns ordered nested tree`
- `Integration: clone template kit → new kit, is_template=false, deep-copied sections/questions`
- `Unit: question mapped to unpublished dimension → 422`
- `Integration: reorder questions via PATCH sort_order → GET reflects new order`

#### 3.3 — Jobs, candidates, applications

**What**: The hiring entities that interviews attach to.

**Design**:
- Models from Suggestion 1: `jobs` (status, `external_ats_id`/`external_ats`, hiring_manager), `job_competency_mappings` (weight, is_required), `candidates` (PII fields, `external_ats_id`), `applications` (status state machine `active→offer→hired|rejected|withdrawn`).
- Routes: CRUD for jobs and candidates; `POST /jobs/{id}/competencies` to attach weighted dimensions; `POST /applications` linking candidate↔job.
- Candidate PII fields flagged for GDPR/CCPA handling (Phase 9 erasure).

**Testing**:
- `Integration: create job, attach 3 weighted competencies → GET returns mappings`
- `Unit: application status transition active→hired allowed; hired→active rejected`
- `Integration: create candidate + application → linked correctly, org-scoped`

---

## Phase 4: Recording & Transcription Pipeline

### Purpose
Capture interviews and turn them into searchable, diarised, time-coded transcripts — the raw material for all AI analysis. Introduces the provider abstractions, consent management, encrypted storage, the async worker pipeline, and WebVTT interchange. After this phase an interview can be scheduled, recorded via a meeting bot, and transcribed.

### Tasks

#### 4.1 — Interview lifecycle & consent management

**What**: Interview entity with scheduling, status state machine, panel support, and jurisdiction-aware consent.

**Design**:
- Models from Suggestion 1: `interviews` (status `scheduled→in_progress→completed|cancelled|no_show`, `meeting_url`, `meeting_platform`, `consent_status`, `consent_jurisdiction`), `interview_panelists`.
- `compliance/consent.py`: `consent_rules(jurisdiction) -> ConsentRule` encoding two-party vs one-party recording consent (e.g., IL/CA two-party, EU explicit consent, GDPR Art. 6/9, BIPA opt-in for IL voiceprints). Recording cannot start until `consent_status='granted'` when the rule requires it.

```python
class ConsentRule(BaseModel):
    jurisdiction: str
    recording_requires_consent: bool
    biometric_opt_in_required: bool   # BIPA / GDPR Art.9
    notice_text_key: str
```

- Routes: `POST /interviews`, `POST /interviews/{id}/consent`, status transition endpoints.

**Testing**:
- `Unit: consent_rules('IL') → recording_requires_consent=True, biometric_opt_in_required=True`
- `Integration: start recording without consent in two-party jurisdiction → 409 problem+json`
- `Unit: interview status no_show→completed → invalid transition error`

#### 4.2 — Recording provider abstraction (Recall.ai)

**What**: Join/record meetings via a pluggable provider, defaulting to Recall.ai.

**Design**:
- `integrations/recording/base.py`:

```python
class RecordingProvider(Protocol):
    async def start_bot(self, meeting_url: str, interview_id: UUID) -> str: ...  # returns bot_id
    async def stop_bot(self, bot_id: str) -> None: ...
    async def get_recording(self, bot_id: str) -> RecordingArtifact: ...  # mp4 url + vtt url
```

- `recall.py` implements via Recall REST; stores `interview_recordings` (storage_url, format mp4, duration, `encryption_key_id`, `retention_expires_at` = now + org retention).
- Recall webhook `POST /webhooks/recall` (verify signature) → on `recording.done` enqueue transcription task.

**Testing**:
- `Integration (mocked Recall): start_bot → bot_id stored on interview`
- `Integration (mocked): recall webhook recording.done → transcription task enqueued`
- `Unit: webhook with bad signature → 401, no task enqueued`

#### 4.3 — Encrypted recording storage

**What**: Store recordings in S3-compatible storage with envelope encryption and retention.

**Design**:
- `storage/s3.py` (boto3, MinIO/S3), `storage/encryption.py`: per-recording AES-256-GCM data key wrapped by KMS master key (`encryption_key_id`). Enables crypto-shredding (delete data key → recording unreadable) for GDPR erasure.
- Retention sweeper Celery beat task deletes recordings past `retention_expires_at`.

**Testing**:
- `Unit: encrypt then decrypt blob with same key → original bytes`
- `Unit: decrypt with deleted/rotated key → DecryptionError (crypto-shred verified)`
- `Integration (MinIO testcontainer): upload + retrieve encrypted object round-trips`

#### 4.4 — Transcription pipeline (AssemblyAI async)

**What**: Worker task producing a diarised, WebVTT transcript persisted as segments.

**Design**:
- `integrations/transcription/base.py`:

```python
class TranscriptionProvider(Protocol):
    async def transcribe(self, audio_url: str, *, diarise: bool, redact_pii: bool) -> TranscriptResult: ...

class TranscriptSegmentDTO(BaseModel):
    speaker_label: str
    start_ms: int
    end_ms: int
    text: str
    confidence: float
```

- `assemblyai.py` implements with diarisation + PII redaction policy (CCPA/BIPA-sensitive). Worker `tasks/transcription.py` persists `transcripts` + `transcript_segments` (Suggestion 1), maps speaker labels to interviewer/candidate/panelist, and writes a WebVTT artifact (W3C WebVTT) to S3 for interchange. `tsvector` GIN index + `pgvector` embedding per segment for search.
- On completion enqueues competency-tagging, summary, and compliance-detection tasks (Phases 5–6).

**Testing**:
- `Integration (mocked AssemblyAI, fixture audio): transcribe → N segments persisted, speakers labelled`
- `Unit: TranscriptResult → WebVTT output matches W3C cue format (fixture .vtt comparison)`
- `Integration: full-text search 'incident response' → returns matching segment`
- `Integration: PII redaction on → SSN-like token replaced in stored text`

#### 4.5 — Transcript search & retrieval API

**What**: Endpoints to fetch transcripts and search across them.

**Design**:
- `GET /interviews/{id}/transcript` (segments + WebVTT link), `GET /search/transcripts?q=...&job_id=...` (tsvector ranked), `GET /search/semantic?q=...` (pgvector cosine over segment embeddings).

**Testing**:
- `Integration: keyword search returns ranked segments with interview context`
- `Integration: semantic search 'handled a crisis' surfaces 'led the incident response' segment`

---

## Phase 5: AI Analysis — Evidence Tagging, Summaries, Scoring

### Purpose
The heart of the product: turn transcripts into competency evidence, structured scorecards, and AI summaries. Introduces the abstracted LLM client, versioned prompts, and the scoring domain. After this phase a completed interview yields tagged evidence linked to transcript moments, an AI summary, and a structured scorecard the interviewer can review and submit.

### Tasks

#### 5.1 — LLM client abstraction & prompt management

**What**: Provider-agnostic LLM client with versioned, testable prompt templates.

**Design**:
- `integrations/llm/base.py`:

```python
class LLMClient(Protocol):
    async def complete_json(self, *, system: str, user: str, schema: type[BaseModel], model: str) -> BaseModel: ...
```

- Implementations `anthropic.py`, `openai.py`, `ollama.py`; structured-output enforced against a Pydantic schema (tool/JSON mode). `ai/prompts/` holds versioned templates with a `version` string recorded on every AI artifact (for EU AI Act technical documentation / reproducibility).

**Testing**:
- `Unit (mocked provider): complete_json returns schema-valid object`
- `Unit: provider returns invalid JSON → retried once then AppError 502`
- `Unit: prompt template renders with interview context variables`

#### 5.2 — Competency evidence tagging

**What**: AI tags transcript segments as positive/negative/neutral evidence for each mapped competency.

**Design**:
- `ai/competency_tagger.py`: for each job competency + its rubric, prompt the LLM with relevant transcript segments and `RubricDetail` indicators. Output schema:

```python
class EvidenceTag(BaseModel):
    dimension_id: UUID
    segment_id: UUID
    evidence_text: str
    evidence_type: Literal["positive","negative","neutral"]
    confidence: float
    rationale: str

class TaggingResult(BaseModel):
    tags: list[EvidenceTag]
    model_version: str
```

- Persists `competency_evidence` (Suggestion 1) with `is_ai_generated=true`, `confidence`, and `verified_by` nullable. Prompt grounded in IO-psychology rubric indicators (the AI-native differentiator from research.md). System prompt instructs: cite only segments present, never infer protected characteristics.
- Worker task triggered post-transcription.

**Testing**:
- `Integration (mocked LLM, fixture transcript+framework): tagging → evidence rows linked to real segment_ids`
- `Unit: tag referencing non-existent segment → rejected before persist`
- `Unit: confidence outside [0,1] → ValidationError`
- `Integration: human verifies evidence → verified_by set, is_ai_generated unchanged (audit)`

#### 5.3 — AI interview summaries

**What**: Generate short / medium / long summaries from the transcript.

**Design**:
- `ai/summariser.py`: produces three `interview_notes` rows (note_type `ai_short_summary`/`ai_summary`/`ai_detailed_summary`, Suggestion 1). Summaries organised by competency coverage and topic. Worker task post-transcription.

**Testing**:
- `Integration (mocked LLM): three summary notes persisted with correct note_type`
- `Unit: empty transcript → graceful "insufficient content" note, no crash`

#### 5.4 — Scoring API & evidence-linked scorecards

**What**: Interviewers score competencies (assisted by AI evidence); scores and scorecards persist immutably once submitted.

**Design**:
- Models from Suggestion 1: `interview_scores` (unique per interview+dimension+scorer), `scorecards` (overall_recommendation enum), `scorecard_entries`. `scoring_service` validates score ∈ scoring_levels range; links evidence to entries.
- Routes: `POST /interviews/{id}/scores`, `POST /applications/{id}/scorecards` (one per interviewer), `GET /interviews/{id}/evidence` (AI-suggested evidence to assist scoring). Submitting a scorecard writes an immutable record + audit_log entry; revisions create a new version (never overwrite — supports defensibility).

**Testing**:
- `Integration: submit score within rubric range → persisted; out-of-range → 422`
- `Integration: duplicate score same interview+dimension+scorer → 409`
- `Integration: submit scorecard → immutable; second submit → new version, audit_log records both`
- `Unit: scorecard with overall_recommendation 'maybe' → 422 (not in enum)`

---

## Phase 6: Real-Time Coaching & Compliance Flagging

### Purpose
Differentiating, AI-native capability: during the live interview, stream the transcript, compute talk-to-listen ratio and kit adherence, push microcoaching prompts, and flag potentially discriminatory or off-topic questions in real time. Delivers the "helps interviewers, not monitors them" experience and the EEOC/compliance guardrail that incumbents underdeliver.

### Tasks

#### 6.1 — Live transcript streaming (Deepgram) & WebSocket

**What**: Stream low-latency transcript to the interviewer UI during the interview.

**Design**:
- `integrations/transcription/deepgram.py` streaming (Nova-3, ~300ms). `realtime/ws.py`: `WS /interviews/{id}/live` authenticated by JWT; pushes transcript deltas. `realtime/live_state.py` keeps live state in Redis (Suggestion 2 Projection 2 shape): `interview:{id}:live` (talk_ratio, questions_asked/remaining, current_section, elapsed), `interview:{id}:transcript:latest`, `interview:{id}:coaching`.

**Testing**:
- `Integration (mocked Deepgram stream): transcript deltas pushed to connected WS client`
- `Unit: live_state updates talk_ratio as segments arrive`
- `Unit: WS connection without valid JWT → closed 4401`

#### 6.2 — Talk ratio, kit adherence & microcoaching

**What**: Compute interviewer behaviour metrics live and emit coaching prompts.

**Design**:
- `ai/coach.py`: rolling 5-min talk ratio from diarised stream; compares spoken content to kit questions (embedding match) to track adherence and `questions_remaining`. Emits coaching events:

```python
class CoachingPrompt(BaseModel):
    type: Literal["talk_ratio_warning","question_suggestion","time_warning","missed_competency"]
    message: str
    ts_ms: int
```

- Thresholds configurable (e.g., talk_ratio > 0.6 over 5 min → warning). Prompts pushed via WS + cached in Redis.

**Testing**:
- `Unit: interviewer talks 65% over window → talk_ratio_warning emitted`
- `Unit: section nearing time allocation → time_warning emitted`
- `Integration: all kit competencies covered → no missed_competency prompt; one uncovered near end → prompt`

#### 6.3 — Real-time compliance / discriminatory-question detection

**What**: Flag legally problematic or off-topic questions live and post-hoc.

**Design**:
- `ai/compliance_detector.py`: classifies interviewer utterances against EEOC-protected categories (questions implicating age, family/pregnancy, national origin, disability, religion, etc.) and off-topic. Produces `compliance_flags` (Suggestion 1: flag_type, severity info/warning/critical, description, segment_id). Live: critical flags pushed to WS immediately ("This question may relate to family status — consider rephrasing"). Post-interview: full pass over transcript persists all flags. Grounded in EEOC Uniform Guidelines (29 CFR 1607).

**Testing**:
- `Unit (mocked LLM): "Do you have kids?" → discriminatory_question flag, severity critical`
- `Unit: on-topic competency question → no flag`
- `Integration: post-interview pass persists flags linked to segments`
- `Integration: resolve flag → resolved_by/resolved_at set, original flag retained (audit)`

---

## Phase 7: ATS & HRIS Integration

### Purpose
Interview intelligence only creates value when connected to where hiring decisions are made. Implements the ATS abstraction (Merge.dev unified + direct Greenhouse Harvest reference), bidirectional sync of jobs/candidates/applications and outbound scorecards, and the HRIS link that later powers quality-of-hire. Webhook ingestion keeps data fresh.

### Tasks

#### 7.1 — ATS provider abstraction

**What**: Normalised read/write interface over ATS platforms.

**Design**:
- `integrations/ats/base.py`:

```python
class ATSProvider(Protocol):
    async def list_jobs(self, since: datetime | None) -> list[ATSJob]: ...
    async def list_candidates(self, since: datetime | None) -> list[ATSCandidate]: ...
    async def list_applications(self, since: datetime | None) -> list[ATSApplication]: ...
    async def push_scorecard(self, application_ext_id: str, scorecard: ATSScorecard) -> str: ...
```

- Normalised DTOs follow Merge's ATS object shapes (Candidates, Applications, Jobs, ScheduledInterview, Scorecards). `ats_integrations` row stores `provider`, `credentials_vault_ref`, `sync_status`, `last_synced_at`.

**Testing**:
- `Unit: ATSJob DTO parses Merge job payload fixture`
- `Unit: push_scorecard maps internal scorecard → ATSScorecard correctly`

#### 7.2 — Merge.dev unified integration + direct Greenhouse Harvest

**What**: Concrete providers for breadth (Merge) and a direct reference (Greenhouse).

**Design**:
- `integrations/ats/merge.py`: per-linked-account API key; normalised pagination. `greenhouse.py`: Harvest v3, Basic auth (RFC 7617, API key as username), HMAC-SHA256 webhook verification (`X-Grnhse-Signature`). Both map external entities to local `jobs`/`candidates`/`applications` keyed by `external_ats_id`+`external_ats`.

**Testing**:
- `Integration (mocked Merge): list_candidates paginated → all candidates upserted`
- `Integration (mocked Greenhouse): Harvest job payload → local job upserted with external ids`
- `Unit: Greenhouse webhook valid HMAC → accepted; invalid → 401`

#### 7.3 — Sync engine & webhook ingestion

**What**: Scheduled + webhook-driven bidirectional sync.

**Design**:
- `workers/tasks/ats_sync.py`: incremental pull since `last_synced_at`; conflict policy = ATS wins for candidate/job fields, platform owns interview/score data. Outbound: on scorecard submit, enqueue `push_scorecard`. `POST /webhooks/ats/{provider}` validates signature, upserts changed entities. `webhook_events` table records receipt for idempotency + audit.

**Testing**:
- `Integration: incremental sync only pulls records changed since last_synced_at`
- `Integration: scorecard submit → push_scorecard task enqueued`
- `Integration: duplicate webhook (same event id) → processed once (idempotent)`

---

## Phase 8: Analytics — Bias Detection, Interviewer Effectiveness, Comparison

### Purpose
Turn accumulated interview data into the platform's signature insights: cross-interviewer scoring consistency, demographic bias metrics computed transparently (the open-source differentiator), interviewer effectiveness analytics, and candidate comparison matrices. These power the dashboards and the regulatory bias reports of Phase 9.

### Tasks

#### 8.1 — Candidate comparison & scoring consistency

**What**: Side-by-side candidate comparison and cross-interviewer divergence detection.

**Design**:
- `analytics/interviewer_metrics.py` and comparison service producing (read-model style, Suggestion 2 Projection 1) a comparison matrix: per application, per dimension, each interviewer's score + evidence count + variance. Flags divergence beyond a threshold (e.g., std-dev > configurable) for calibration discussion.
- Routes: `GET /applications/{id}/comparison`, `GET /jobs/{id}/candidates/ranking` (weighted by `job_competency_mappings`).

**Testing**:
- `Unit: scores [4,4,1] for a dimension → divergence flagged`
- `Integration: ranking weights competencies per job mapping correctly`
- `Integration: comparison matrix groups by dimension across interviewers`

#### 8.2 — Bias detection (four-fifths, impact ratios, NIST AI RMF)

**What**: Compute adverse-impact metrics across demographic groups transparently.

**Design**:
- `analytics/bias.py`: implements EEOC four-fifths rule and impact ratios (selection-rate ratio of each group vs. highest-rate group), score-distribution comparisons by group/interviewer/role, recency bias detection (time-of-day score drift). Aligns with NIST AI RMF MEASURE 2.11. Demographic data is **separately stored, opt-in, anonymised group identifiers** — never used by scoring AI (enforced by 5.2 prompt rules). Outputs match Suggestion 2's `projection_bias_metrics` shape (sample_size, avg_score, impact_ratio, four_fifths_pass).

```python
def four_fifths(selection_rates: dict[str, float]) -> dict[str, BiasResult]:
    """Returns per-group impact_ratio vs. max-rate group and pass/fail (>= 0.8)."""
```

- Root-cause hook: for failing dimensions, LLM analyses associated questions/evidence and recommends rewrites (AI-augmentation candidate from features.md).

**Testing**:
- `Unit: selection rates {A:0.50,B:0.30} → B impact_ratio 0.6, four_fifths_pass False`
- `Unit: equal rates → all pass, ratio 1.0`
- `Unit: insufficient sample size (<N) → result marked low_confidence, no false flag`
- `Integration (mocked LLM): failing dimension → root-cause recommendation generated`

#### 8.3 — Interviewer effectiveness analytics

**What**: Aggregate interviewer metrics over time.

**Design**:
- Materialised view / scheduled task populating `interviewer_metrics` (Suggestion 1): total_interviews, avg_talk_ratio, avg_question_adherence, avg_score_calibration (deviation from peers), compliance_flags_count, per period. `GET /users/{id}/effectiveness?from=&to=` and org-level leaderboards.

**Testing**:
- `Integration: compute metrics for interviewer over period → matches hand-calculated fixture`
- `Unit: calibration = mean deviation from peer scores on shared candidates`

---

## Phase 9: Compliance, Audit & Regulatory Workflows

### Purpose
Make compliance a first-class API surface, not policy prose (standards.md). Delivers the partitioned audit log surfacing, NYC LL144 bias-audit export + candidate notifications, EU AI Act technical documentation export, and GDPR/CCPA/BIPA data-subject workflows including crypto-shredding erasure. This is a core differentiator for the regulated buyers the README targets.

### Tasks

#### 9.1 — Audit log & immutable decision trail

**What**: Append-only audit log and exportable decision audit trail.

**Design**:
- `audit_log` (Suggestion 1, partitioned by year) written on every state-changing action via a service-layer decorator capturing actor, action, entity, old/new values, IP, user agent. Immutable decision audit view (Suggestion 2 Projection 3 shape): per decision, interview_count, avg_score, cross-interviewer variance, compliance_flags, contributing record ids.
- Routes: `GET /audit?entity=&from=&to=`, `GET /applications/{id}/decision-trail`.

**Testing**:
- `Integration: submit scorecard → audit_log row with old/new values + actor`
- `Integration: decision-trail export → includes all contributing scores/evidence/flags`
- `Unit: audit_log has no UPDATE/DELETE path exposed (append-only enforced)`

#### 9.2 — NYC LL144 & EU AI Act reporting

**What**: Regulatory report generation and candidate notification workflows.

**Design**:
- `compliance/ll144.py`: generates annual bias-audit summary (impact ratios by race/ethnicity/sex from `bias.py`), persists `bias_audit_reports` (report_type `nyc_ll144`), exports publishable summary (the LL144 careers-page artifact). Candidate notification: `compliance/notifications` schedules ≥10-business-day AEDT notices for NYC candidates.
- EU AI Act: technical-documentation export bundling model versions (from AI artifacts), data-governance notes, bias metrics, human-oversight records — produced as a downloadable package.
- Routes: `POST /compliance/reports/ll144`, `GET /compliance/reports/{id}`, `POST /compliance/notifications/aedt`.

**Testing**:
- `Integration: generate LL144 report → impact ratios per group, four-fifths pass/fail, publishable summary`
- `Unit: NYC candidate created → AEDT notice scheduled 10 business days out`
- `Integration: EU AI Act export bundles model_version of every AI artifact for the application`

#### 9.3 — GDPR/CCPA/BIPA data-subject workflows

**What**: Access, deletion (crypto-shred), and consent-record workflows.

**Design**:
- `compliance/erasure.py`: on erasure request, delete the candidate's recording data keys (crypto-shred, Phase 4.3), null PII columns, and replace transcript PII with tombstones, retaining non-personal audit structure. `GET /candidates/{id}/data-export` (DSAR) returns all personal data. Consent records (BIPA written consent, retention schedule) tracked per candidate.

**Testing**:
- `Integration: erasure request → recording undecryptable, PII columns nulled, audit_log retains action`
- `Integration: DSAR export includes transcripts, scores, evidence for the candidate`
- `Unit: BIPA candidate without written consent → recording blocked`

---

## Phase 10: AI Kit Generation, ICP & Quality-of-Hire Loop

### Purpose
Deliver the closed-loop, AI-native advantages that no incumbent fully provides: generate interview kits from job descriptions, learn an Ideal Candidate Profile from outcomes, and connect interview assessments to post-hire performance to prove (and continuously recalibrate) predictive validity. This is the long-term defensibility moat.

### Tasks

#### 10.1 — AI interview-kit generation from job descriptions

**What**: Generate a structured kit (sections, questions, competency mapping, rubrics) from a JD.

**Design**:
- `ai/kit_generator.py`: LLM analyses JD → proposes competency mapping (linking to existing published dimensions, suggesting new ones), generates calibrated questions with follow-ups and time allocations. Also flags biased/exclusionary JD language (BrightHire-style inclusive-JD check). Output validated to the kit schema (Phase 3.2); created as a draft kit for human review (EU AI Act human-oversight).
- Route: `POST /kits/generate` `{job_id, job_description}`.

**Testing**:
- `Integration (mocked LLM, fixture JD): generates kit with sections, questions, dimension mappings`
- `Unit: generated kit validates against kit schema`
- `Unit: biased JD phrase → flagged in response`

#### 10.2 — Ideal Candidate Profile (ICP) learning

**What**: Evolve a per-role competency profile from hiring outcomes.

**Design**:
- ICP stored as JSONB on the job (Suggestion 3): target score distribution per competency, refined as applications reach `hired` and accumulate outcomes. `analytics` recomputes ICP weights from successful hires.
- Route: `GET /jobs/{id}/icp`, recompute task triggered on outcome updates.

**Testing**:
- `Integration: after 5 hires, ICP weights shift toward their dominant competency profile`
- `Unit: ICP recompute with no hires → returns seed profile, no error`

#### 10.3 — Quality-of-hire feedback loop & predictive validity

**What**: Connect interview scores to post-hire outcomes and measure predictive accuracy.

**Design**:
- `post_hire_outcomes` (Suggestion 1) populated via HRIS sync (Merge HRIS): performance_rating, retention_months, is_still_employed. `analytics/quality_of_hire.py` correlates interview scores ↔ outcomes (per Suggestion 2 `projection_quality_of_hire`), reports `prediction_accuracy` per competency, and feeds 8.2/10.2 (which competencies/questions actually predict success). A scheduled recalibration job updates competency weights.
- Routes: `GET /jobs/{id}/quality-of-hire`, `GET /analytics/predictive-validity`.

**Testing**:
- `Integration (mocked HRIS): outcomes synced → post_hire_outcomes populated`
- `Unit: correlation of scores vs. performance computed correctly on fixture dataset`
- `Integration: high-predictive competency surfaces in predictive-validity report`

---

## Phase 11: Frontend Application

### Purpose
Provide the recruiter/hiring-manager/interviewer UI: kit builder, candidate pipeline, the live-interview coaching view, scorecard comparison, bias dashboard, and compliance/settings. The product is dashboard-heavy; this phase makes every prior API usable by non-developers. Can begin in parallel once the relevant APIs exist.

### Tasks

#### 11.1 — App shell, auth, generated API client

**What**: Next.js shell with SSO login, role-aware nav, and a typed API client generated from OpenAPI.

**Design**:
- Generate `frontend/lib/api` from `openapi/openapi.json` (openapi-typescript). Auth flow stores JWT; middleware guards routes by role. shadcn/ui layout, Tailwind theme.

**Testing**:
- `E2E (Playwright, mocked API): SSO login → dashboard; unauthenticated → redirect to login`
- `Unit: nav hides admin-only items for interviewer role`

#### 11.2 — Kit builder, pipeline, scorecard & bias dashboards

**What**: Core management screens.

**Design**:
- Kit builder (drag-order sections/questions, competency mapping). Candidate pipeline + comparison matrix. Scorecard submission with AI evidence panel. Bias dashboard (impact-ratio charts, four-fifths pass/fail). Compliance/settings (consent, retention, integrations, SSO/SCIM).

**Testing**:
- `E2E (mocked API): build kit → POST payload matches schema`
- `E2E: scorecard screen shows AI evidence and submits scores`
- `E2E: bias dashboard renders impact ratios from fixture report`

#### 11.3 — Live interview coaching view

**What**: Real-time interview screen consuming the WebSocket.

**Design**:
- Connects `WS /interviews/{id}/live`; renders live transcript, talk-ratio gauge, kit progress, coaching prompts, and compliance alerts. Interviewer scores inline as evidence arrives.

**Testing**:
- `E2E (mock WS server): transcript deltas render; talk_ratio_warning shows alert`
- `E2E: critical compliance flag → prominent inline warning`

---

## Phase 12: MCP Server, Hardening & Release

### Purpose
Expose the platform to AI agents via MCP, complete security/compliance hardening against OWASP Top 10 and the ISO/NIST frameworks, finalise the OpenAPI spec and deployment artifacts, and prepare a releasable, self-hostable product.

### Tasks

#### 12.1 — MCP server

**What**: Expose interview data and kit generation as MCP tools.

**Design**:
- `mcp/server.py` (Python MCP SDK) tools: `get_transcript`, `query_scorecards`, `get_competency_framework`, `generate_interview_kit`. Auth-scoped to org via API token; read tools respect RLS; write tools (kit gen) create drafts for human review.

**Testing**:
- `Integration (MCP client): get_transcript returns segments for authorised org only`
- `Unit: cross-org access via MCP token → denied`

#### 12.2 — Security & compliance hardening

**What**: OWASP Top 10 mitigations, rate limiting, secrets handling, audit completeness.

**Design**:
- Verify access control on every route (A01); parameterised queries via ORM (A03); secrets only via Vault/KMS (A02/A05); rate limiting + lockout; security headers; dependency scanning in CI. Map controls to ISO 27001/27701 and NIST AI RMF; produce a SOC 2 control matrix doc.

**Testing**:
- `Integration: cross-org access attempt on every resource → 403/404 (RLS) — parametrised test over all routes`
- `Integration: SQL-injection-style input in search → safely parameterised, no error leak`
- `Integration: error responses never leak stack traces (problem+json only)`

#### 12.3 — OpenAPI finalisation, deployment & docs

**What**: Ship the spec, containers, and self-host docs.

**Design**:
- Validate exported `openapi.json` against OpenAPI 3.1; publish SDK-ready spec for ATS partners. Finalise Compose + production Helm/compose notes for cloud/hybrid. Seed script + getting-started docs.

**Testing**:
- `CI: openapi.json validates against OpenAPI 3.1 schema`
- `E2E (optional): fresh docker compose up + seed → login, create kit, run mocked interview end-to-end`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton          ─── required by everything
    │
Phase 2: Identity, Auth, Multi-Tenancy  ─── requires 1
    │
Phase 3: Competency Frameworks & Kits   ─── requires 2
    │
Phase 4: Recording & Transcription      ─── requires 3
    │
Phase 5: AI Analysis (evidence/scoring) ─── requires 4
    ├── Phase 6: Real-Time Coaching & Compliance ─── requires 4,5
    ├── Phase 7: ATS & HRIS Integration          ─── requires 3 (parallel with 5/6)
    └── Phase 8: Analytics (bias/effectiveness)  ─── requires 5,7
         │
Phase 9: Compliance & Regulatory        ─── requires 5,8
    │
Phase 10: AI Kit Gen, ICP, QoH Loop     ─── requires 7,8 (10.3 needs 7 HRIS)
    │
Phase 11: Frontend                      ─── each screen requires its API; shell after 2,
    │                                       can develop in parallel from Phase 3 onward
Phase 12: MCP, Hardening & Release      ─── requires all
```

**Parallelism opportunities:**
- Phases **6, 7** can be developed concurrently once Phases 4–5 land (6 needs the AI/transcript layer; 7 only needs Phase 3 entities).
- **Phase 11 (frontend)** proceeds in parallel from Phase 3 onward — each screen built as soon as its backend API exists (shell needs only Phase 2).
- **Phase 8** depends on both 5 (scores/evidence) and 7 (synced applications) before bias/effectiveness analytics are meaningful.
- Within Phase 5, evidence tagging (5.2), summaries (5.3), and scoring (5.4) are independent after the LLM client (5.1).

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`); frontend tests pass (Vitest/Playwright) where applicable.
3. Linting and formatting pass (`ruff check`, `ruff format --check`; ESLint/Prettier for frontend).
4. Strict type checking passes (`mypy src`; `tsc --noEmit` for frontend).
5. `docker compose up` succeeds and `/healthz` is green with the phase's services.
6. Alembic migration(s) for new tables/columns are created, reversible, and applied automatically on startup.
7. The feature works end-to-end against the running stack (real Postgres/Redis/MinIO; external providers mocked unless a real-integration test is explicitly marked).
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec and validate against it.
9. New configuration options are added to `.env.example` and documented.
10. Every state-changing action writes an `audit_log` entry (from Phase 9 onward, verified by test).
11. RLS isolation is verified for any new tenant-scoped table (cross-org access test passes).
12. Error responses conform to RFC 7807 (`application/problem+json`).
