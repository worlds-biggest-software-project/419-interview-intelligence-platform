# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Interview Intelligence Platform (419)
> Approach: Fully normalized relational database with strict referential integrity
> Generated: 2026-05-25

---

## Summary

A traditional normalized relational model using PostgreSQL as the primary data store. All entities are represented as tables with strong foreign key constraints, enforcing referential integrity across the entire hiring workflow. This approach prioritises data consistency, query predictability, and mature tooling for reporting and compliance.

The model follows Third Normal Form (3NF) with strategic denormalization only where query performance demands it (e.g., materialized views for dashboards). It leverages PostgreSQL's robust feature set including row-level security, full-text search, and partitioning for time-series interview data.

---

## Core Entities and Relationships

### Entity-Relationship Overview

```
Organisation ──┬── Team
               ├── User (Interviewer / Hiring Manager / Recruiter)
               ├── CompetencyFramework ── CompetencyDimension
               ├── InterviewKit ── InterviewKitSection ── InterviewQuestion
               └── Job ── JobCompetencyMapping
                    │
                    └── Candidate ── Application
                                        │
                                        └── Interview ──┬── InterviewRecording
                                                        ├── Transcript ── TranscriptSegment
                                                        ├── CompetencyEvidence
                                                        ├── InterviewScore
                                                        └── InterviewNote
                                        │
                                        └── Scorecard ── ScorecardEntry
                                        │
                                        └── HiringDecision ── DecisionEvidence
```

### Schema Definitions

#### Organisation and Users

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL CHECK (role IN ('admin','recruiter','hiring_manager','interviewer','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    PRIMARY KEY (team_id, user_id)
);
```

#### Competency Frameworks

```sql
CREATE TABLE competency_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE competency_dimensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES competency_frameworks(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),  -- e.g., 'technical', 'behavioural', 'leadership'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    scoring_rubric  TEXT,          -- Describes what good/poor evidence looks like
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scoring_levels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id) ON DELETE CASCADE,
    level           INTEGER NOT NULL,  -- e.g., 1-5
    label           VARCHAR(100) NOT NULL,  -- e.g., 'Strong', 'Adequate', 'Weak'
    description     TEXT NOT NULL,     -- What evidence at this level looks like
    UNIQUE (dimension_id, level)
);
```

#### Jobs and Interview Kits

```sql
CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    department      VARCHAR(255),
    location        VARCHAR(255),
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','open','closed','archived')),
    external_ats_id VARCHAR(255),       -- ID in connected ATS (Greenhouse, Lever, etc.)
    external_ats    VARCHAR(50),        -- Which ATS this syncs with
    hiring_manager_id UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_competency_mappings (
    job_id          UUID NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id),
    weight          DECIMAL(3,2) NOT NULL DEFAULT 1.0,  -- Importance weighting
    is_required     BOOLEAN NOT NULL DEFAULT true,
    PRIMARY KEY (job_id, dimension_id)
);

CREATE TABLE interview_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id          UUID REFERENCES jobs(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    interview_type  VARCHAR(50) NOT NULL CHECK (interview_type IN ('screening','technical','behavioural','culture','panel','final')),
    estimated_duration_minutes INTEGER,
    is_template     BOOLEAN NOT NULL DEFAULT false,
    industry        VARCHAR(100),       -- For pre-built industry kits
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE interview_kit_sections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    kit_id          UUID NOT NULL REFERENCES interview_kits(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    time_allocation_minutes INTEGER,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE interview_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES interview_kit_sections(id) ON DELETE CASCADE,
    question_text   TEXT NOT NULL,
    follow_up_prompts TEXT[],            -- Suggested follow-up questions
    dimension_id    UUID REFERENCES competency_dimensions(id),  -- Competency this question assesses
    question_type   VARCHAR(50) DEFAULT 'behavioural',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Candidates and Applications

```sql
CREATE TABLE candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    external_ats_id VARCHAR(255),
    source          VARCHAR(100),       -- Where the candidate came from
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    job_id          UUID NOT NULL REFERENCES jobs(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active','offer','hired','rejected','withdrawn')),
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    external_ats_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Interviews, Recordings, and Transcripts

```sql
CREATE TABLE interviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    kit_id          UUID REFERENCES interview_kits(id),
    interviewer_id  UUID NOT NULL REFERENCES users(id),
    interview_type  VARCHAR(50) NOT NULL,
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    status          VARCHAR(50) NOT NULL DEFAULT 'scheduled' CHECK (status IN ('scheduled','in_progress','completed','cancelled','no_show')),
    meeting_url     VARCHAR(500),
    meeting_platform VARCHAR(50),       -- 'zoom', 'teams', 'google_meet', 'in_person'
    consent_status  VARCHAR(50) DEFAULT 'pending' CHECK (consent_status IN ('pending','granted','denied','not_required')),
    consent_jurisdiction VARCHAR(10),   -- ISO country/region code for consent rules
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Panel interview support
CREATE TABLE interview_panelists (
    interview_id    UUID NOT NULL REFERENCES interviews(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) DEFAULT 'panelist',
    PRIMARY KEY (interview_id, user_id)
);

CREATE TABLE interview_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    storage_url     VARCHAR(1000) NOT NULL,   -- S3/GCS URL to encrypted recording
    format          VARCHAR(20) NOT NULL DEFAULT 'mp4',
    duration_seconds INTEGER,
    file_size_bytes BIGINT,
    encryption_key_id VARCHAR(255),           -- Reference to KMS key for encryption
    retention_expires_at TIMESTAMPTZ,         -- Auto-delete date per retention policy
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    provider        VARCHAR(50) NOT NULL,     -- 'assemblyai', 'deepgram', 'zoom_native', etc.
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    confidence_score DECIMAL(4,3),
    word_count      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transcript_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id   UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    speaker_id      UUID REFERENCES users(id),       -- NULL if candidate
    speaker_label   VARCHAR(50) NOT NULL,             -- 'interviewer', 'candidate', 'panelist_2'
    start_time_ms   INTEGER NOT NULL,
    end_time_ms     INTEGER NOT NULL,
    text            TEXT NOT NULL,
    confidence      DECIMAL(4,3),
    sort_order      INTEGER NOT NULL
);

-- Full-text search index on transcript segments
CREATE INDEX idx_transcript_segments_fts ON transcript_segments
    USING GIN (to_tsvector('english', text));
```

#### Scoring, Evidence, and Bias Detection

```sql
CREATE TABLE interview_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id),
    scorer_id       UUID NOT NULL REFERENCES users(id),
    score           INTEGER NOT NULL,
    notes           TEXT,
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (interview_id, dimension_id, scorer_id)
);

CREATE TABLE competency_evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id),
    segment_id      UUID REFERENCES transcript_segments(id),  -- Linked transcript moment
    evidence_text   TEXT NOT NULL,
    evidence_type   VARCHAR(50) NOT NULL CHECK (evidence_type IN ('positive','negative','neutral')),
    confidence      DECIMAL(4,3),        -- AI confidence score
    is_ai_generated BOOLEAN NOT NULL DEFAULT true,
    verified_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE interview_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    author_id       UUID NOT NULL REFERENCES users(id),
    note_type       VARCHAR(50) NOT NULL DEFAULT 'manual' CHECK (note_type IN ('manual','ai_summary','ai_short_summary','ai_detailed_summary')),
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_flags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    segment_id      UUID REFERENCES transcript_segments(id),
    flag_type       VARCHAR(100) NOT NULL,  -- 'discriminatory_question', 'off_topic', 'legal_risk'
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('info','warning','critical')),
    description     TEXT NOT NULL,
    resolution      TEXT,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Scorecards and Hiring Decisions

```sql
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    overall_recommendation VARCHAR(50) CHECK (overall_recommendation IN ('strong_hire','hire','no_decision','no_hire','strong_no_hire')),
    summary         TEXT,
    submitted_by    UUID NOT NULL REFERENCES users(id),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scorecard_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id) ON DELETE CASCADE,
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id),
    score           INTEGER NOT NULL,
    evidence_summary TEXT,
    UNIQUE (scorecard_id, dimension_id)
);

CREATE TABLE hiring_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    decision        VARCHAR(50) NOT NULL CHECK (decision IN ('hire','reject','hold','withdraw')),
    rationale       TEXT NOT NULL,
    decided_by      UUID NOT NULL REFERENCES users(id),
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE decision_evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id     UUID NOT NULL REFERENCES hiring_decisions(id) ON DELETE CASCADE,
    evidence_id     UUID REFERENCES competency_evidence(id),
    scorecard_id    UUID REFERENCES scorecards(id),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Interviewer Analytics and Bias Metrics

```sql
CREATE TABLE interviewer_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_interviews INTEGER NOT NULL DEFAULT 0,
    avg_talk_ratio  DECIMAL(4,3),       -- Interviewer talk-to-listen ratio
    avg_question_adherence DECIMAL(4,3), -- How closely they followed the kit
    avg_score_calibration DECIMAL(4,3),  -- Deviation from peer scores
    compliance_flags_count INTEGER DEFAULT 0,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, period_start, period_end)
);

CREATE TABLE bias_audit_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    audit_period_start DATE NOT NULL,
    audit_period_end DATE NOT NULL,
    report_type     VARCHAR(50) NOT NULL,  -- 'nyc_ll144', 'eu_ai_act', 'internal'
    findings        TEXT NOT NULL,
    impact_ratios   JSONB,                 -- Stored bias impact ratios by group
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ
);
```

#### ATS Integration and Audit Logging

```sql
CREATE TABLE ats_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,  -- 'greenhouse', 'lever', 'workday', etc.
    credentials_vault_ref VARCHAR(255),    -- Reference to secrets manager
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'active',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create yearly partitions for audit log
CREATE TABLE audit_log_2026 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

#### Quality-of-Hire Feedback Loop

```sql
CREATE TABLE post_hire_outcomes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    hire_date       DATE NOT NULL,
    performance_review_date DATE,
    performance_rating DECIMAL(3,2),     -- Normalised 0.0-5.0
    retention_months INTEGER,
    is_still_employed BOOLEAN,
    external_hris_id VARCHAR(255),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Pros

- **Data integrity**: Foreign key constraints and CHECK constraints enforce business rules at the database level, preventing orphaned records and invalid states
- **Query predictability**: Standard SQL joins make reporting, analytics, and ad-hoc queries straightforward; any reporting tool can connect directly
- **Mature ecosystem**: PostgreSQL has 30+ years of production hardening, extensive tooling (pg_dump, logical replication, pgAdmin), and deep community expertise
- **Compliance-friendly**: Normalized tables with clear audit_log partitioning make it easy to generate regulatory reports (NYC LL144, EU AI Act) and respond to data subject access requests
- **Row-level security**: PostgreSQL RLS enables multi-tenant data isolation at the database level, enforcing that organisations can only see their own data
- **Full-text search**: Built-in tsvector/GIN indexes on transcript_segments reduce the need for a separate search engine for basic transcript search
- **ACID transactions**: Guarantees that complex operations (e.g., submitting a scorecard while recording evidence and updating application status) are atomic

## Cons

- **Schema rigidity**: Every new feature or entity change requires a migration. Competency frameworks, scoring rubrics, and interview kits are semi-structured by nature and may resist rigid table definitions
- **Transcript storage at scale**: Transcript segments can grow to millions of rows quickly (thousands of interviews, each with hundreds of segments). Table partitioning is necessary but adds operational complexity
- **AI feature impedance mismatch**: AI-generated evidence, competency tags, and coaching signals are inherently probabilistic and semi-structured. Forcing them into fixed columns adds friction to the ML pipeline
- **Limited graph queries**: Querying relationships like "which competencies predict quality-of-hire across interviewers and roles" requires complex multi-table joins that are natural in a graph model but awkward in relational SQL
- **Reporting load**: Complex analytics (bias detection, interviewer calibration, quality-of-hire correlation) can create heavy read loads on the primary database; read replicas or materialized views are needed

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ |
| Connection pooler | PgBouncer or Supavisor |
| Migrations | Flyway or golang-migrate |
| Full-text search | PostgreSQL tsvector (basic), Elasticsearch/OpenSearch (advanced) |
| Object storage | S3/GCS for recordings with client-side encryption |
| Secrets management | AWS KMS / HashiCorp Vault for recording encryption keys |
| Reporting | Materialized views + read replica for dashboards |
| Row-level security | PostgreSQL RLS policies per organisation_id |

---

## Migration and Scaling Considerations

1. **Table partitioning**: The `audit_log`, `transcript_segments`, and `competency_evidence` tables should be partitioned by time (monthly or yearly) from day one. Partitioning after data accumulates is significantly more disruptive.

2. **Read replicas**: Interview analytics, bias detection dashboards, and interviewer effectiveness reports should query read replicas to avoid contention with transactional writes during live interviews.

3. **Materialized views**: Pre-compute interviewer metrics, candidate comparison matrices, and bias impact ratios as materialized views refreshed on a schedule (hourly for operational dashboards, daily for compliance reports).

4. **Data retention**: Recording and transcript data must respect jurisdiction-specific retention policies. The `retention_expires_at` column on recordings enables automated deletion jobs. GDPR right-to-erasure requires cascading deletion across candidates, applications, interviews, transcripts, scores, and evidence.

5. **Sharding strategy**: If multi-tenant scale exceeds single-instance PostgreSQL capacity, shard by `organisation_id` using Citus or application-level routing. This is unlikely to be needed before 10,000+ organisations.

6. **Migration from monolith**: This schema supports a monolithic application initially. If microservices are adopted later, the natural service boundaries are: Interview Management, Competency Framework, Candidate/Application (synced from ATS), Transcription Pipeline, Analytics/Reporting.
