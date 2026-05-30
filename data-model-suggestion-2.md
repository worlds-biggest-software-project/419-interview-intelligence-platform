# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Interview Intelligence Platform (419)
> Approach: Event sourcing with Command Query Responsibility Segregation
> Generated: 2026-05-25

---

## Summary

An event-sourced architecture where every state change in the interview lifecycle is captured as an immutable event in an append-only event store. The system uses CQRS (Command Query Responsibility Segregation) to separate write operations (commands that produce events) from read operations (projections optimized for specific query patterns).

This approach is particularly well-suited for an interview intelligence platform because:
- **Audit trails are a first-class requirement**, not an afterthought. Every hiring decision, score change, and evidence tag is inherently recorded as an event with full context.
- **Regulatory compliance** (EU AI Act, NYC LL144, GDPR) demands the ability to reconstruct exactly what happened, when, and why at any point in the hiring process.
- **Bias detection requires temporal analysis** -- understanding how scoring patterns change over time, detecting drift, and comparing historical interviewer behaviour demands access to the full event history.
- **Multiple read models** serve different stakeholders: recruiters need candidate comparison views, interviewers need real-time coaching dashboards, compliance officers need audit reports, and ML pipelines need training data.

---

## Architecture Overview

```
                    ┌─────────────┐
                    │   Commands   │
                    │ (Write Side) │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Aggregates  │
                    │  (Domain)    │
                    └──────┬──────┘
                           │ Events
                    ┌──────▼──────┐
                    │ Event Store  │
                    │ (Append-Only)│
                    └──────┬──────┘
                           │ Event Bus
              ┌────────────┼────────────────┐
              │            │                │
       ┌──────▼──────┐ ┌──▼─────────┐ ┌────▼──────┐
       │ Recruiter    │ │ Interviewer │ │ Compliance │
       │ Read Model   │ │ Read Model  │ │ Read Model │
       │ (PostgreSQL)  │ │ (Redis)     │ │ (PostgreSQL)│
       └─────────────┘ └────────────┘ └───────────┘
```

---

## Core Aggregates and Events

### Aggregate: InterviewProcess

The central aggregate representing a candidate's journey through the interview process for a specific job application.

```
Aggregate: InterviewProcess
├── InterviewProcessCreated
├── InterviewScheduled
├── InterviewConsentGranted
├── InterviewConsentDenied
├── InterviewStarted
├── InterviewRecordingStarted
├── TranscriptSegmentCaptured
├── CompetencyEvidenceTagged
├── CompetencyEvidenceVerified
├── ComplianceFlagRaised
├── ComplianceFlagResolved
├── InterviewCompleted
├── InterviewCancelled
├── ScoreSubmitted
├── ScoreRevised
├── ScorecardSubmitted
├── HiringDecisionMade
└── HiringDecisionReversed
```

### Aggregate: CompetencyFramework

Manages the lifecycle of competency frameworks and scoring rubrics.

```
Aggregate: CompetencyFramework
├── FrameworkCreated
├── DimensionAdded
├── DimensionUpdated
├── ScoringLevelDefined
├── FrameworkPublished
├── FrameworkDeprecated
└── FrameworkVersioned
```

### Aggregate: InterviewKit

Manages structured interview guides.

```
Aggregate: InterviewKit
├── KitCreated
├── SectionAdded
├── QuestionAdded
├── QuestionReordered
├── KitPublished
├── KitCloned
└── KitArchived
```

### Aggregate: InterviewerProfile

Tracks interviewer behaviour and effectiveness over time.

```
Aggregate: InterviewerProfile
├── InterviewerOnboarded
├── InterviewConducted
├── TalkRatioRecorded
├── QuestionAdherenceScored
├── CalibrationSessionCompleted
├── CoachingRecommendationGenerated
├── BiasPatternDetected
└── ImprovementMilestoneReached
```

---

## Event Store Schema

The event store uses a PostgreSQL-backed append-only table. Events are the source of truth; all other data is derived.

```sql
-- The single source of truth
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,    -- 'InterviewProcess', 'CompetencyFramework', etc.
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,    -- 'ScoreSubmitted', 'CompetencyEvidenceTagged', etc.
    event_version   INTEGER NOT NULL,         -- Monotonically increasing per aggregate
    payload         JSONB NOT NULL,           -- Event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- user_id, ip_address, correlation_id, causation_id
    organisation_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
) PARTITION BY RANGE (occurred_at);

-- Index for aggregate replay
CREATE INDEX idx_events_aggregate ON events (aggregate_id, event_version);

-- Index for event type subscriptions
CREATE INDEX idx_events_type ON events (event_type, occurred_at);

-- Index for organisation-scoped queries
CREATE INDEX idx_events_org ON events (organisation_id, occurred_at);

-- Yearly partitions
CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

### Example Events

```json
// ScoreSubmitted
{
    "aggregate_type": "InterviewProcess",
    "aggregate_id": "app-123-interview-456",
    "event_type": "ScoreSubmitted",
    "event_version": 14,
    "payload": {
        "interview_id": "int-456",
        "scorer_id": "user-789",
        "dimension_id": "dim-101",
        "dimension_name": "Problem Solving",
        "score": 4,
        "max_score": 5,
        "notes": "Candidate demonstrated strong analytical thinking when discussing the system outage scenario.",
        "evidence_segment_ids": ["seg-201", "seg-205"]
    },
    "metadata": {
        "user_id": "user-789",
        "correlation_id": "corr-abc",
        "ip_address": "192.168.1.100",
        "user_agent": "Mozilla/5.0..."
    }
}

// CompetencyEvidenceTagged
{
    "aggregate_type": "InterviewProcess",
    "aggregate_id": "app-123-interview-456",
    "event_type": "CompetencyEvidenceTagged",
    "event_version": 10,
    "payload": {
        "interview_id": "int-456",
        "dimension_id": "dim-101",
        "segment_id": "seg-201",
        "evidence_text": "When the database went down, I led the incident response by...",
        "evidence_type": "positive",
        "confidence": 0.92,
        "ai_model_version": "competency-tagger-v3.2",
        "is_ai_generated": true
    },
    "metadata": {
        "source": "ml_pipeline",
        "pipeline_run_id": "run-xyz"
    }
}

// HiringDecisionMade
{
    "aggregate_type": "InterviewProcess",
    "aggregate_id": "app-123",
    "event_type": "HiringDecisionMade",
    "event_version": 28,
    "payload": {
        "application_id": "app-123",
        "candidate_id": "cand-456",
        "job_id": "job-789",
        "decision": "hire",
        "rationale": "Strong technical skills, excellent culture fit, consistent positive evidence across all interviewers.",
        "scorecard_ids": ["sc-1", "sc-2", "sc-3"],
        "overall_scores": {
            "problem_solving": 4.2,
            "communication": 3.8,
            "leadership": 4.5
        }
    },
    "metadata": {
        "user_id": "user-hiring-mgr",
        "decision_meeting_id": "meeting-abc"
    }
}
```

---

## Read Models (Projections)

### Projection 1: Recruiter Dashboard (PostgreSQL)

Optimized for candidate comparison, pipeline tracking, and scorecard review.

```sql
-- Denormalized view for recruiter pipeline
CREATE TABLE projection_candidate_pipeline (
    application_id      UUID PRIMARY KEY,
    candidate_id        UUID NOT NULL,
    candidate_name      VARCHAR(255),
    job_id              UUID NOT NULL,
    job_title           VARCHAR(255),
    organisation_id     UUID NOT NULL,
    current_stage       VARCHAR(50),
    interviews_completed INTEGER DEFAULT 0,
    interviews_scheduled INTEGER DEFAULT 0,
    avg_overall_score   DECIMAL(4,2),
    recommendation      VARCHAR(50),   -- Aggregate recommendation
    last_activity_at    TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Denormalized scorecard comparison view
CREATE TABLE projection_scorecard_comparison (
    application_id      UUID NOT NULL,
    interview_id        UUID NOT NULL,
    interviewer_id      UUID NOT NULL,
    interviewer_name    VARCHAR(255),
    dimension_id        UUID NOT NULL,
    dimension_name      VARCHAR(255),
    score               INTEGER,
    evidence_count      INTEGER DEFAULT 0,
    recommendation      VARCHAR(50),
    scored_at           TIMESTAMPTZ,
    PRIMARY KEY (application_id, interview_id, dimension_id)
);
```

### Projection 2: Real-Time Interview View (Redis)

Optimized for live interview coaching and real-time transcript display.

```
Key: interview:{interview_id}:live
Value: {
    "status": "in_progress",
    "interviewer_id": "user-789",
    "kit_id": "kit-123",
    "current_section": 2,
    "talk_ratio": { "interviewer": 0.35, "candidate": 0.65 },
    "questions_asked": 4,
    "questions_remaining": 6,
    "compliance_flags": [],
    "evidence_tagged_count": 3,
    "elapsed_seconds": 1200
}

Key: interview:{interview_id}:transcript:latest
Value: [
    { "speaker": "interviewer", "text": "Tell me about a time...", "ts": 1200100 },
    { "speaker": "candidate", "text": "In my previous role...", "ts": 1200300 }
]

Key: interview:{interview_id}:coaching
Value: [
    { "type": "talk_ratio_warning", "msg": "You have been talking for 60% of the last 5 minutes. Consider asking an open question.", "ts": 1200500 },
    { "type": "question_suggestion", "msg": "Consider probing deeper on the conflict resolution example.", "ts": 1201000 }
]
```

### Projection 3: Compliance Audit View (PostgreSQL)

Optimized for regulatory reporting and audit trail generation.

```sql
-- Immutable decision audit trail
CREATE TABLE projection_decision_audit (
    decision_id         UUID PRIMARY KEY,
    application_id      UUID NOT NULL,
    candidate_id        UUID NOT NULL,
    job_id              UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    decision            VARCHAR(50) NOT NULL,
    decided_by          UUID NOT NULL,
    decided_at          TIMESTAMPTZ NOT NULL,
    rationale           TEXT NOT NULL,
    interview_count     INTEGER,
    avg_score           DECIMAL(4,2),
    score_variance      DECIMAL(4,2),     -- Cross-interviewer variance
    compliance_flags    INTEGER DEFAULT 0,
    evidence_count      INTEGER DEFAULT 0,
    event_ids           UUID[] NOT NULL    -- All events that contributed to this decision
);

-- Bias metrics for LL144 / AI Act reporting
CREATE TABLE projection_bias_metrics (
    organisation_id     UUID NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    job_id              UUID,
    dimension_id        UUID,
    interviewer_id      UUID,
    demographic_group   VARCHAR(100),      -- Anonymised group identifier
    sample_size         INTEGER NOT NULL,
    avg_score           DECIMAL(4,2),
    impact_ratio        DECIMAL(4,3),      -- Adverse impact ratio
    four_fifths_pass    BOOLEAN,           -- Does it pass the 4/5ths rule?
    computed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, period_start, period_end, job_id, dimension_id, interviewer_id, demographic_group)
);
```

### Projection 4: Quality-of-Hire Feedback (PostgreSQL)

Connects interview events to post-hire outcomes.

```sql
CREATE TABLE projection_quality_of_hire (
    application_id      UUID PRIMARY KEY,
    candidate_id        UUID NOT NULL,
    job_id              UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    hire_date           DATE,
    interview_scores    JSONB,             -- { "problem_solving": 4.2, "communication": 3.8 }
    interviewer_ids     UUID[],
    kit_ids             UUID[],
    performance_rating  DECIMAL(3,2),
    retention_months    INTEGER,
    is_still_employed   BOOLEAN,
    prediction_accuracy DECIMAL(4,3),      -- How well did interview scores predict performance?
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Processing Infrastructure

### Event Handlers (Projectors)

```
Event Bus (Kafka / NATS / PostgreSQL LISTEN/NOTIFY)
    │
    ├── RecruiterDashboardProjector
    │     Subscribes to: ScoreSubmitted, ScorecardSubmitted, HiringDecisionMade,
    │                    InterviewCompleted, InterviewScheduled
    │     Writes to: projection_candidate_pipeline, projection_scorecard_comparison
    │
    ├── RealTimeInterviewProjector
    │     Subscribes to: InterviewStarted, TranscriptSegmentCaptured,
    │                    CompetencyEvidenceTagged, ComplianceFlagRaised
    │     Writes to: Redis (interview:{id}:live, interview:{id}:transcript:latest)
    │
    ├── ComplianceProjector
    │     Subscribes to: HiringDecisionMade, ScoreSubmitted, ComplianceFlagRaised,
    │                    ScoreRevised, HiringDecisionReversed
    │     Writes to: projection_decision_audit, projection_bias_metrics
    │
    ├── InterviewerAnalyticsProjector
    │     Subscribes to: InterviewConducted, TalkRatioRecorded,
    │                    QuestionAdherenceScored, ScoreSubmitted
    │     Writes to: projection_interviewer_metrics
    │
    ├── QualityOfHireProjector
    │     Subscribes to: HiringDecisionMade, PostHirePerformanceRecorded
    │     Writes to: projection_quality_of_hire
    │
    └── MLTrainingDataExporter
          Subscribes to: CompetencyEvidenceTagged, ScoreSubmitted,
                         TranscriptSegmentCaptured
          Writes to: S3 (training data lake for competency tagger model retraining)
```

---

## GDPR and Right-to-Erasure Strategy

Event sourcing creates a tension with GDPR's right to erasure (Article 17). The recommended approach is **crypto-shredding**:

1. **Encrypt personal data fields** in event payloads using a per-candidate encryption key
2. **Store encryption keys** in a separate key management service (AWS KMS, HashiCorp Vault)
3. **On erasure request**, delete the candidate's encryption key rather than modifying events
4. **Events become unreadable** for that candidate -- the event structure remains intact but personal data fields decrypt to garbage
5. **Projections** are updated by replaying events, which will skip or anonymise the crypto-shredded candidate

```json
// Event payload with encrypted fields
{
    "event_type": "ScoreSubmitted",
    "payload": {
        "interview_id": "int-456",
        "scorer_id": "user-789",
        "candidate_data_enc": "AES-256-GCM:encrypted_blob...",  // Encrypted with candidate key
        "dimension_id": "dim-101",
        "score": 4,
        "notes_enc": "AES-256-GCM:encrypted_blob..."           // Encrypted with candidate key
    },
    "metadata": {
        "candidate_key_id": "kms:key-cand-123"
    }
}
```

---

## Pros

- **Complete audit trail by design**: Every state change is an immutable event. Regulatory audits (NYC LL144, EU AI Act) can be answered by querying the event store directly. There is no possibility of losing audit history through UPDATE or DELETE operations.
- **Time-travel debugging**: Any past state can be reconstructed by replaying events up to a specific point. "What did the scorecard look like before the score revision?" is a trivial query.
- **Multiple optimized read models**: Recruiters, interviewers, compliance officers, and ML pipelines each get a read model optimized for their access patterns, without compromising the write model.
- **Bias detection on temporal data**: Event history enables analysis like "how did this interviewer's scoring pattern change after bias training?" or "did scores drift over the course of a hiring day?"
- **Natural fit for real-time features**: Transcript segments, coaching signals, and compliance flags are naturally events that flow through the system.
- **Replay and rebuild**: If a projection has a bug, fix the projector code and rebuild the read model from the event store. No data is lost.
- **GDPR compliance via crypto-shredding**: Candidate data can be effectively erased without violating the immutability of the event store.

## Cons

- **Significant complexity increase**: Event sourcing and CQRS require more infrastructure (event store, event bus, multiple projectors, multiple databases) than a simple relational model. The team must understand eventual consistency.
- **Eventual consistency**: Read models are updated asynchronously. A recruiter may submit a score and not see it reflected in the dashboard for a few seconds. This requires careful UX design.
- **Event schema evolution**: As the domain evolves, event schemas change. Upcasting (transforming old event formats to new ones during replay) requires discipline and tooling.
- **Higher operational overhead**: Running Kafka/NATS, maintaining multiple projections, monitoring consumer lag, and handling projection failures adds operational burden.
- **Debugging complexity**: Tracing a bug through events, projections, and multiple databases is harder than querying a single relational database.
- **Steeper learning curve**: Developers unfamiliar with event sourcing and CQRS need significant onboarding time. Hiring for these skills is more competitive.
- **Storage growth**: The event store grows indefinitely (events are never deleted). Snapshot optimisation and archival strategies are needed for long-running aggregates.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL (simple), EventStoreDB (dedicated), or Marten (.NET) |
| Event bus | Apache Kafka (at scale) or NATS JetStream (simpler) |
| Command handling | Application-level aggregate framework (Axon, EventSauce, custom) |
| Read model (operational) | PostgreSQL for durable projections |
| Read model (real-time) | Redis for live interview state |
| Read model (analytics) | ClickHouse or PostgreSQL materialized views |
| Crypto-shredding keys | AWS KMS or HashiCorp Vault |
| Schema registry | Confluent Schema Registry (for Kafka) or custom versioning |
| Monitoring | Consumer lag monitoring via Kafka/NATS metrics; projection health checks |

---

## Migration and Scaling Considerations

1. **Start with PostgreSQL as event store**: For early stages, PostgreSQL with the events table is sufficient. Migrate to EventStoreDB or a dedicated event store only when write throughput exceeds PostgreSQL's capacity (likely 10,000+ events/second sustained).

2. **Event versioning from day one**: Establish an event schema registry and versioning strategy before the first event is written. Changing event formats after production data exists is extremely costly.

3. **Projection rebuild strategy**: Every projector must be able to rebuild its read model from scratch by replaying all events. Test this regularly. A projection that cannot be rebuilt is a ticking time bomb.

4. **Snapshot optimization**: Long-lived aggregates (e.g., an InterviewProcess for a candidate with 20+ interviews) should use periodic snapshots to avoid replaying thousands of events on every command.

5. **Event archival**: Events older than the regulatory retention period can be moved to cold storage (S3/Glacier). The event store should support archival without breaking aggregate integrity.

6. **Gradual adoption**: Consider starting with event sourcing only for the most audit-critical aggregates (InterviewProcess, HiringDecision) and using traditional CRUD for less critical entities (InterviewKit templates, CompetencyFramework definitions). This hybrid approach reduces initial complexity.

7. **Partition by organisation**: For multi-tenant scale, partition the event store by organisation_id. Each organisation's event stream is independent, enabling per-tenant scaling and data isolation.
