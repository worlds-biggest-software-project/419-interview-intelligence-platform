# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Interview Intelligence Platform (419)
> Approach: PostgreSQL with strategic JSONB columns for semi-structured data
> Generated: 2026-05-25

---

## Summary

A hybrid approach that uses PostgreSQL as a single database engine but strategically combines normalized relational tables (for well-defined entities and relationships) with JSONB columns (for semi-structured, evolving, and AI-generated data). This gives the data consistency and referential integrity of a relational model where it matters, while accommodating the inherent flexibility needed for competency frameworks, AI-generated evidence, interview metadata, and integration payloads.

This approach recognizes that an interview intelligence platform has two distinct data categories:
1. **Structured core**: Organisations, users, candidates, applications, jobs, interviews -- these have stable schemas and benefit from strict relational modeling.
2. **Semi-structured periphery**: Competency scoring rubrics, AI-tagged evidence, transcript analysis metadata, integration sync payloads, interviewer coaching signals, and evolving configuration -- these change frequently, vary by customer, and resist rigid table definitions.

PostgreSQL's JSONB support with GIN indexing provides document-store capabilities without requiring a separate database, reducing operational complexity while maintaining transactional guarantees across both structured and semi-structured data.

---

## Design Principles

1. **Use columns for what you query by; use JSONB for what you query within**: If a field is used in WHERE clauses, JOINs, or foreign keys, it gets a dedicated column. If it is display-only, configuration, or varies by customer, it goes in JSONB.
2. **Index JSONB paths that are queried**: GIN indexes on JSONB columns enable efficient containment queries. Computed/generated columns extract frequently-queried JSONB paths into indexed columns.
3. **Validate JSONB with CHECK constraints or application-level JSON Schema**: JSONB flexibility does not mean "anything goes." Use CHECK constraints or application validation to enforce structure within JSONB fields.
4. **Never store foreign keys in JSONB**: Referential integrity only works with real columns. If a relationship matters, model it relationally.

---

## Schema Definitions

### Core Relational Entities (Stable Schema)

```sql
-- Organisations, users, teams: fully relational (same as Suggestion 1)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',   -- Org-level config: timezone, consent defaults, retention policies
    feature_flags   JSONB NOT NULL DEFAULT '{}',   -- Per-org feature toggles
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    preferences     JSONB NOT NULL DEFAULT '{}',   -- UI preferences, notification settings
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
```

### Competency Frameworks (Relational + JSONB Rubrics)

Competency frameworks have a stable structure (framework -> dimensions -> levels) but scoring rubrics, behavioural indicators, and example evidence vary significantly by industry, role type, and customer preference. The rubric content is stored as JSONB.

```sql
CREATE TABLE competency_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    metadata        JSONB NOT NULL DEFAULT '{}',   -- Industry, source, validation status
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE competency_dimensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES competency_frameworks(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    -- JSONB for the rich, variable rubric content
    rubric          JSONB NOT NULL DEFAULT '{}',
    /*
    rubric structure:
    {
        "description": "Ability to break down complex problems...",
        "behavioural_indicators": [
            "Identifies root causes rather than symptoms",
            "Uses structured frameworks to analyze problems",
            "Considers multiple solution approaches"
        ],
        "levels": [
            {
                "level": 1,
                "label": "Limited",
                "description": "Struggles to identify core issues...",
                "example_evidence": "Could not explain their approach to the problem..."
            },
            {
                "level": 5,
                "label": "Exceptional",
                "description": "Demonstrates sophisticated analytical frameworks...",
                "example_evidence": "When presented with the outage scenario, immediately..."
            }
        ],
        "anti_patterns": ["Gives rehearsed STAR answers without depth"],
        "follow_up_prompts": ["Can you walk me through your thought process?"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GIN index for searching within rubrics
CREATE INDEX idx_competency_dimensions_rubric ON competency_dimensions USING GIN (rubric);
```

### Interview Kits (Relational Structure, JSONB Content)

```sql
CREATE TABLE interview_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    job_id          UUID REFERENCES jobs(id),
    name            VARCHAR(255) NOT NULL,
    interview_type  VARCHAR(50) NOT NULL,
    is_template     BOOLEAN NOT NULL DEFAULT false,
    -- JSONB for the full kit structure (sections, questions, time allocations)
    kit_content     JSONB NOT NULL,
    /*
    kit_content structure:
    {
        "estimated_duration_minutes": 60,
        "sections": [
            {
                "id": "sec-1",
                "title": "Introduction & Rapport Building",
                "time_allocation_minutes": 5,
                "instructions": "Put the candidate at ease...",
                "questions": []
            },
            {
                "id": "sec-2",
                "title": "Problem Solving Assessment",
                "time_allocation_minutes": 20,
                "competency_ids": ["dim-101", "dim-102"],
                "questions": [
                    {
                        "id": "q-1",
                        "text": "Tell me about a time you had to solve a complex problem with limited information.",
                        "type": "behavioural",
                        "competency_id": "dim-101",
                        "follow_ups": [
                            "What was the outcome?",
                            "What would you do differently?"
                        ],
                        "scoring_guidance": "Look for structured approach, consideration of alternatives..."
                    }
                ]
            }
        ],
        "closing_instructions": "Ask if the candidate has questions...",
        "industry": "fintech",
        "role_level": "senior"
    }
    */
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Search within kit content
CREATE INDEX idx_interview_kits_content ON interview_kits USING GIN (kit_content);
```

### Jobs, Candidates, Applications (Relational Core)

```sql
CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    department      VARCHAR(255),
    location        VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    hiring_manager_id UUID REFERENCES users(id),
    -- JSONB for variable job metadata synced from ATS
    ats_sync        JSONB NOT NULL DEFAULT '{}',
    /*
    ats_sync structure:
    {
        "provider": "greenhouse",
        "external_id": "gh-12345",
        "last_synced_at": "2026-05-25T10:30:00Z",
        "custom_fields": { ... },
        "pipeline_stages": ["phone_screen", "onsite", "final"]
    }
    */
    -- JSONB for the ideal candidate profile that evolves
    ideal_candidate_profile JSONB NOT NULL DEFAULT '{}',
    /*
    ideal_candidate_profile structure:
    {
        "min_scores": { "problem_solving": 3, "communication": 4 },
        "preferred_signals": ["leadership_experience", "startup_background"],
        "learned_from_hires": 12,
        "last_updated": "2026-05-20T15:00:00Z",
        "confidence": 0.78
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    -- JSONB for variable candidate profile data from ATS
    profile_data    JSONB NOT NULL DEFAULT '{}',
    /*
    profile_data structure:
    {
        "source": "linkedin",
        "current_company": "Acme Corp",
        "current_title": "Senior Engineer",
        "resume_url": "s3://...",
        "ats_ids": {
            "greenhouse": "gh-cand-456",
            "lever": "lev-cand-789"
        },
        "tags": ["referred", "silver_medalist"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    job_id          UUID NOT NULL REFERENCES jobs(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ats_sync        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Interviews (Relational + JSONB for Variable Data)

```sql
CREATE TABLE interviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    kit_id          UUID REFERENCES interview_kits(id),
    interviewer_id  UUID NOT NULL REFERENCES users(id),
    interview_type  VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'scheduled',
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    -- Consent management varies by jurisdiction
    consent         JSONB NOT NULL DEFAULT '{}',
    /*
    consent structure:
    {
        "jurisdiction": "DE",
        "recording_consent": "granted",
        "ai_processing_consent": "granted",
        "consent_mechanism": "verbal_on_recording",
        "consent_timestamp": "2026-05-25T14:00:00Z",
        "retention_policy": "12_months",
        "data_subject_rights_notified": true
    }
    */
    -- Meeting platform details vary
    meeting_details JSONB NOT NULL DEFAULT '{}',
    /*
    meeting_details structure:
    {
        "platform": "zoom",
        "meeting_url": "https://zoom.us/j/...",
        "meeting_id": "123456789",
        "recording_bot_id": "recall-bot-abc",
        "recording_provider": "recall_ai"
    }
    */
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
```

### Transcripts and AI Analysis (JSONB-Heavy)

Transcript segments are stored relationally for efficient sequencing and full-text search, but AI analysis metadata (competency tags, sentiment, compliance flags) is stored as JSONB because the ML pipeline output schema evolves with each model version.

```sql
CREATE TABLE interview_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    storage_url     VARCHAR(1000) NOT NULL,
    format          VARCHAR(20) NOT NULL DEFAULT 'mp4',
    duration_seconds INTEGER,
    file_size_bytes BIGINT,
    encryption_key_id VARCHAR(255),
    retention_expires_at TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- Codec info, resolution, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transcript_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    speaker_label   VARCHAR(50) NOT NULL,
    speaker_user_id UUID REFERENCES users(id),
    start_time_ms   INTEGER NOT NULL,
    end_time_ms     INTEGER NOT NULL,
    text            TEXT NOT NULL,
    confidence      DECIMAL(4,3),
    sort_order      INTEGER NOT NULL,
    -- AI analysis stored as JSONB -- evolves with model versions
    ai_analysis     JSONB NOT NULL DEFAULT '{}',
    /*
    ai_analysis structure:
    {
        "model_version": "transcript-analyzer-v3.2",
        "competency_signals": [
            {
                "dimension_id": "dim-101",
                "evidence_type": "positive",
                "confidence": 0.92,
                "explanation": "Candidate demonstrates structured problem decomposition"
            }
        ],
        "sentiment": { "score": 0.7, "label": "positive" },
        "topics": ["leadership", "incident_response", "team_management"],
        "compliance_check": {
            "flagged": false,
            "checks_passed": ["no_discriminatory_content", "no_personal_questions"]
        },
        "key_phrases": ["led the team", "reduced downtime by 40%"]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Full-text search on transcript text
CREATE INDEX idx_transcript_segments_fts ON transcript_segments
    USING GIN (to_tsvector('english', text));

-- GIN index for AI analysis queries
CREATE INDEX idx_transcript_segments_ai ON transcript_segments USING GIN (ai_analysis);

-- Composite index for interview + sort order
CREATE INDEX idx_transcript_segments_order ON transcript_segments (interview_id, sort_order);
```

### Scoring and Evidence (Hybrid)

```sql
CREATE TABLE interview_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    dimension_id    UUID NOT NULL REFERENCES competency_dimensions(id),
    scorer_id       UUID NOT NULL REFERENCES users(id),
    score           INTEGER NOT NULL,
    notes           TEXT,
    -- JSONB for evidence links and AI-assisted scoring context
    evidence        JSONB NOT NULL DEFAULT '{}',
    /*
    evidence structure:
    {
        "linked_segments": ["seg-201", "seg-205", "seg-210"],
        "ai_suggested_score": 4,
        "ai_confidence": 0.85,
        "ai_rationale": "Strong evidence of structured problem-solving in segments 201 and 205",
        "scoring_rubric_level_matched": 4,
        "time_to_score_seconds": 45
    }
    */
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (interview_id, dimension_id, scorer_id)
);

CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    submitted_by    UUID NOT NULL REFERENCES users(id),
    overall_recommendation VARCHAR(50),
    summary         TEXT,
    -- JSONB for the complete scorecard snapshot
    scorecard_data  JSONB NOT NULL,
    /*
    scorecard_data structure:
    {
        "dimensions": [
            {
                "dimension_id": "dim-101",
                "dimension_name": "Problem Solving",
                "score": 4,
                "evidence_summary": "Strong analytical skills demonstrated...",
                "evidence_count": 3
            }
        ],
        "overall_score": 4.2,
        "strengths": ["Technical depth", "Clear communication"],
        "concerns": ["Limited management experience"],
        "comparison_to_icp": {
            "match_score": 0.82,
            "gaps": ["leadership"]
        }
    }
    */
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Hiring Decisions and Compliance

```sql
CREATE TABLE hiring_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    decision        VARCHAR(50) NOT NULL,
    rationale       TEXT NOT NULL,
    decided_by      UUID NOT NULL REFERENCES users(id),
    -- JSONB for the full decision context snapshot
    decision_context JSONB NOT NULL,
    /*
    decision_context structure:
    {
        "scorecard_ids": ["sc-1", "sc-2", "sc-3"],
        "aggregate_scores": { "problem_solving": 4.2, "communication": 3.8 },
        "interviewer_consensus": "aligned",
        "score_variance": 0.3,
        "compliance_flags_resolved": 0,
        "competing_candidates": 5,
        "rank_position": 1,
        "ai_recommendation": {
            "decision": "hire",
            "confidence": 0.88,
            "model_version": "decision-support-v2.1"
        }
    }
    */
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_flags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    segment_id      UUID REFERENCES transcript_segments(id),
    flag_type       VARCHAR(100) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    -- JSONB for variable flag details
    details         JSONB NOT NULL,
    /*
    details structure:
    {
        "description": "Question about family status detected",
        "detected_text": "Do you have children?",
        "applicable_regulations": ["title_vii", "eeoc_guidelines"],
        "suggested_alternative": "This role requires occasional weekend work. Are you available for that schedule?",
        "ai_model_version": "compliance-detector-v1.4",
        "confidence": 0.95
    }
    */
    resolution      JSONB,  -- { "action": "acknowledged", "resolved_by": "user-123", "notes": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Interviewer Analytics (JSONB Metrics)

```sql
CREATE TABLE interviewer_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    -- JSONB for rich, evolving metrics
    metrics         JSONB NOT NULL,
    /*
    metrics structure:
    {
        "total_interviews": 15,
        "avg_duration_minutes": 52,
        "talk_ratio": 0.35,
        "question_adherence": 0.88,
        "score_calibration": {
            "deviation_from_peers": 0.3,
            "tendency": "slightly_generous"
        },
        "compliance_flags": 0,
        "quality_of_hire_correlation": 0.72,
        "coaching_recommendations": [
            {
                "type": "talk_ratio",
                "message": "Your talk ratio has improved from 45% to 35% over the last quarter.",
                "trend": "improving"
            }
        ],
        "by_competency": {
            "problem_solving": { "avg_score": 3.8, "peer_avg": 3.5, "calibration": 0.92 },
            "communication": { "avg_score": 3.2, "peer_avg": 3.4, "calibration": 0.94 }
        }
    }
    */
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, period_start, period_end)
);
```

### Audit Log and Integration Sync

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID,
    changes         JSONB NOT NULL,    -- { "old": {...}, "new": {...} }
    context         JSONB NOT NULL DEFAULT '{}',  -- IP, user agent, session info
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE integration_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    direction       VARCHAR(10) NOT NULL,  -- 'inbound' or 'outbound'
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID,
    external_id     VARCHAR(255),
    status          VARCHAR(50) NOT NULL,
    -- Full sync payload stored for debugging and audit
    request_payload JSONB,
    response_payload JSONB,
    error_details   JSONB,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (synced_at);
```

---

## Query Examples

### Find all positive evidence for a candidate across interviews

```sql
SELECT
    ts.text,
    ts.start_time_ms,
    ts.end_time_ms,
    cd.name AS competency,
    signal->>'confidence' AS confidence,
    signal->>'explanation' AS explanation
FROM transcript_segments ts
JOIN interviews i ON i.id = ts.interview_id
JOIN applications a ON a.id = i.application_id
CROSS JOIN LATERAL jsonb_array_elements(ts.ai_analysis->'competency_signals') AS signal
JOIN competency_dimensions cd ON cd.id = (signal->>'dimension_id')::uuid
WHERE a.candidate_id = 'cand-123'
  AND signal->>'evidence_type' = 'positive'
ORDER BY (signal->>'confidence')::decimal DESC;
```

### Interviewer calibration comparison

```sql
SELECT
    u.full_name,
    cd.name AS competency,
    AVG(iscore.score) AS avg_score,
    (SELECT AVG(s2.score) FROM interview_scores s2
     WHERE s2.dimension_id = iscore.dimension_id) AS org_avg,
    ia.metrics->'score_calibration'->>'tendency' AS tendency
FROM interview_scores iscore
JOIN users u ON u.id = iscore.scorer_id
JOIN competency_dimensions cd ON cd.id = iscore.dimension_id
LEFT JOIN interviewer_analytics ia ON ia.user_id = iscore.scorer_id
GROUP BY u.full_name, cd.name, ia.metrics->'score_calibration'->>'tendency';
```

---

## Pros

- **Single database engine**: PostgreSQL handles both relational and document workloads, reducing operational complexity. No need for a separate MongoDB, Redis, or Elasticsearch for most use cases.
- **Schema flexibility where needed**: AI analysis outputs, integration payloads, competency rubrics, and coaching recommendations evolve without database migrations. New ML model versions can add fields to JSONB without schema changes.
- **Referential integrity where it matters**: Foreign keys protect the core entity graph (organisations, users, candidates, applications, interviews). Business-critical relationships are enforced at the database level.
- **Transactional consistency**: A single PostgreSQL instance means ACID transactions span both relational and JSONB data. Submitting a score with AI evidence is atomic.
- **Progressive schema hardening**: Fields that start in JSONB and prove stable can be promoted to dedicated columns via generated columns, without breaking existing queries on the JSONB path.
- **GIN indexes for JSONB queries**: PostgreSQL's GIN indexes on JSONB support efficient containment queries (`@>`), path existence checks, and full-text search within JSON documents.
- **Lower team skill requirements**: The team only needs PostgreSQL expertise, not event sourcing, Kafka, or multiple database administration skills.

## Cons

- **JSONB validation gap**: PostgreSQL does not enforce schema within JSONB columns (no native JSON Schema validation). Application-level validation is required to prevent corrupt JSONB payloads from entering the database.
- **JSONB query performance ceiling**: Complex JSONB queries with deep path traversal, array unnesting, and aggregation are slower than equivalent queries on normalized columns. Performance tuning requires careful GIN index design.
- **Mixed paradigm confusion**: Developers must make consistent decisions about what goes in columns vs. JSONB. Without clear guidelines, the schema drifts toward "put everything in JSONB" or "normalize everything," losing the hybrid benefit.
- **Reporting complexity**: Business intelligence tools handle relational data well but may struggle with JSONB paths. Reporting may require views or materialized views that flatten JSONB into columns.
- **Storage overhead**: JSONB stores field names in every row, unlike normalized columns. For high-volume tables like transcript_segments, the ai_analysis JSONB can significantly increase storage requirements.
- **Audit trail limitations**: Unlike the event-sourced approach, JSONB columns are mutable. Historical states must be captured explicitly in the audit_log table rather than being inherent in the data model.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ with JSONB, GIN indexes, and RLS |
| Connection pooler | PgBouncer |
| Migrations | Flyway or golang-migrate (for column changes); application code for JSONB schema evolution |
| JSONB validation | Application-level JSON Schema validation (e.g., ajv for Node.js, jsonschema for Python) |
| Full-text search | PostgreSQL tsvector for transcripts; Elasticsearch only if search requirements exceed PostgreSQL capabilities |
| Object storage | S3/GCS for recordings |
| Caching | Redis for live interview state and hot query results |
| Analytics | PostgreSQL materialized views; consider ClickHouse only at very high scale |

---

## Migration and Scaling Considerations

1. **JSONB schema versioning**: Include a `_schema_version` field in every JSONB column. Application code handles backward compatibility by migrating JSONB content on read (lazy migration) when it encounters an older schema version.

2. **Generated columns for hot paths**: When a JSONB path is queried frequently, create a generated column to extract it into a proper indexed column:
   ```sql
   ALTER TABLE transcript_segments
       ADD COLUMN ai_flagged BOOLEAN GENERATED ALWAYS AS
       ((ai_analysis->'compliance_check'->>'flagged')::boolean) STORED;
   CREATE INDEX idx_segments_flagged ON transcript_segments (ai_flagged) WHERE ai_flagged = true;
   ```

3. **Partition high-volume tables**: `transcript_segments`, `audit_log`, and `integration_sync_log` should be partitioned by time from the start. Use declarative partitioning with monthly partitions for transcript data and yearly for audit logs.

4. **JSONB to columns promotion path**: Establish a process for promoting stable JSONB fields to dedicated columns. When a field has been unchanged for 6+ months and is queried in reports, migrate it to a column with a generated column bridge during transition.

5. **Read replicas for analytics**: Route all analytics queries (interviewer metrics, bias detection, quality-of-hire correlation) to read replicas. JSONB aggregation queries are CPU-intensive and should not compete with transactional writes.

6. **TOAST compression**: PostgreSQL automatically TOASTs large JSONB values. For very large ai_analysis payloads, consider splitting into a separate table with a 1:1 relationship to keep the main transcript_segments table lean for sequential scans.

7. **Upgrade path to event sourcing**: If audit requirements grow beyond what the audit_log table can support, the hybrid model can be extended with an event store table for critical aggregates (hiring decisions, score submissions) without replacing the entire schema. The JSONB columns make the transition easier because they already capture rich state snapshots.
