# Data Model Suggestion 4: Competency Knowledge Graph + Relational Core

> Project: Interview Intelligence Platform (419)
> Approach: Graph database for competency/evidence relationships with PostgreSQL for transactional data
> Generated: 2026-05-25

---

## Summary

A polyglot persistence architecture that pairs PostgreSQL (for transactional data: organisations, users, candidates, applications, interviews) with a property graph database -- Neo4j or Apache AGE (PostgreSQL extension) -- for the competency knowledge graph. The graph models the rich interconnected relationships between competencies, evidence, interviewers, candidates, roles, and outcomes that are central to the platform's value proposition.

This approach is motivated by the observation that the most valuable features of an interview intelligence platform -- bias detection across interviewers, competency-to-outcome correlation, interviewer calibration, ideal candidate profile learning, and cross-role skill bridging -- are fundamentally graph traversal problems. In a relational model, these queries require 5-8 table joins and complex CTEs. In a graph model, they are natural path traversals.

The platform's domain naturally forms an ontology:
- **Competencies** relate to other competencies (hierarchy, prerequisites, complementary skills)
- **Evidence** connects candidates to competencies through specific interview moments
- **Interviewers** have scoring patterns across competencies, roles, and time periods
- **Roles** require competency profiles that evolve based on hiring outcomes
- **Outcomes** (quality-of-hire) feed back to validate which competency assessments actually predict job success

This is a domain where connected data analysis creates the primary competitive advantage.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Interview Mgmt│  │ Scoring &    │  │ Analytics &      │  │
│  │ Service       │  │ Evidence Svc │  │ Intelligence Svc │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │             │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────────▼─────────┐  │
│  │ PostgreSQL   │  │ PostgreSQL   │  │ Neo4j / AGE      │  │
│  │ (Transactional│  │ (Scores &   │  │ (Knowledge Graph)│  │
│  │  Core)       │  │  Transcripts)│  │                  │  │
│  └──────────────┘  └──────┬───────┘  └────────▲─────────┘  │
│                           │    Sync            │             │
│                           └────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

---

## PostgreSQL: Transactional Core

The relational database handles the operational CRUD for entities with well-defined schemas and transactional requirements.

```sql
-- Same core tables as Suggestion 1/3 for:
-- organisations, users, teams, team_members
-- candidates, applications, jobs
-- interviews, interview_panelists
-- interview_recordings
-- transcript_segments (with full-text search index)
-- audit_log (partitioned)
-- ats_integrations

-- Interview kits stored relationally
CREATE TABLE interview_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    job_id          UUID REFERENCES jobs(id),
    name            VARCHAR(255) NOT NULL,
    interview_type  VARCHAR(50) NOT NULL,
    estimated_duration_minutes INTEGER,
    is_template     BOOLEAN NOT NULL DEFAULT false,
    industry        VARCHAR(100),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Scores are written to PostgreSQL first (source of truth for transactions)
CREATE TABLE interview_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id    UUID NOT NULL REFERENCES interviews(id),
    dimension_id    UUID NOT NULL,  -- References graph node ID
    scorer_id       UUID NOT NULL REFERENCES users(id),
    score           INTEGER NOT NULL,
    notes           TEXT,
    evidence_segment_ids UUID[],
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_to_graph BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (interview_id, dimension_id, scorer_id)
);

-- Hiring decisions with full context
CREATE TABLE hiring_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    decision        VARCHAR(50) NOT NULL,
    rationale       TEXT NOT NULL,
    decided_by      UUID NOT NULL REFERENCES users(id),
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_to_graph BOOLEAN NOT NULL DEFAULT false
);

-- Post-hire outcomes for quality-of-hire feedback
CREATE TABLE post_hire_outcomes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    hire_date       DATE NOT NULL,
    performance_rating DECIMAL(3,2),
    retention_months INTEGER,
    is_still_employed BOOLEAN,
    synced_to_graph BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Neo4j / AGE: Competency Knowledge Graph

The graph database models the interconnected relationships that drive the platform's intelligence features.

### Node Types

```cypher
// Competency Framework nodes
(:CompetencyFramework {
    id: "fw-001",
    name: "Engineering Leadership Framework",
    organisation_id: "org-123",
    version: 3,
    is_published: true
})

(:Competency {
    id: "dim-101",
    name: "Problem Solving",
    category: "technical",
    description: "Ability to decompose complex problems...",
    behavioural_indicators: ["Identifies root causes", "Uses structured frameworks"],
    framework_id: "fw-001"
})

(:ScoringLevel {
    id: "sl-101-4",
    level: 4,
    label: "Strong",
    description: "Demonstrates sophisticated analytical frameworks...",
    example_evidence: "When presented with the outage scenario..."
})

// People nodes
(:Interviewer {
    id: "user-789",
    name: "Sarah Chen",
    organisation_id: "org-123",
    role: "hiring_manager",
    total_interviews: 87
})

(:Candidate {
    id: "cand-456",
    name: "Alex Rodriguez",
    organisation_id: "org-123"
})

// Work nodes
(:Job {
    id: "job-789",
    title: "Senior Backend Engineer",
    department: "Engineering",
    organisation_id: "org-123"
})

(:Interview {
    id: "int-456",
    type: "technical",
    date: "2026-05-20",
    duration_seconds: 3600
})

// Evidence nodes
(:Evidence {
    id: "ev-201",
    text: "When the database went down, I led the incident response...",
    evidence_type: "positive",
    ai_confidence: 0.92,
    segment_id: "seg-201",
    start_time_ms: 120000,
    end_time_ms: 145000
})

// Outcome nodes
(:HireOutcome {
    id: "outcome-123",
    decision: "hire",
    hire_date: "2026-06-01",
    performance_rating: 4.2,
    retention_months: 12,
    is_still_employed: true
})
```

### Relationship Types

```cypher
// Competency structure
(:CompetencyFramework)-[:CONTAINS]->(:Competency)
(:Competency)-[:HAS_LEVEL]->(:ScoringLevel)
(:Competency)-[:PREREQUISITE_FOR]->(:Competency)
(:Competency)-[:COMPLEMENTARY_TO]->(:Competency)
(:Competency)-[:PARENT_OF]->(:Competency)  // Hierarchy

// Job requirements
(:Job)-[:REQUIRES {weight: 1.0, is_critical: true}]->(:Competency)

// Interview relationships
(:Candidate)-[:APPLIED_FOR]->(:Job)
(:Candidate)-[:PARTICIPATED_IN]->(:Interview)
(:Interviewer)-[:CONDUCTED]->(:Interview)
(:Interview)-[:USED_KIT {kit_id: "kit-123"}]->(:Job)
(:Interview)-[:ASSESSED]->(:Competency)

// Evidence and scoring (the critical intelligence layer)
(:Interview)-[:PRODUCED]->(:Evidence)
(:Evidence)-[:DEMONSTRATES {
    score: 4,
    scorer_id: "user-789",
    confidence: 0.92
}]->(:Competency)

(:Interviewer)-[:SCORED {
    score: 4,
    interview_id: "int-456",
    scored_at: "2026-05-20T15:30:00Z"
}]->(:Competency)

// Outcome feedback loop
(:Candidate)-[:RESULTED_IN]->(:HireOutcome)
(:HireOutcome)-[:VALIDATES {
    prediction_accuracy: 0.85
}]->(:Competency)  // Did this competency score predict success?

// Bias detection relationships
(:Interviewer)-[:SCORING_PATTERN {
    period: "2026-Q2",
    avg_score: 3.8,
    std_dev: 0.6,
    sample_size: 15
}]->(:Competency)
```

### Graph Data Model Diagram

```
                    ┌──────────────────┐
                    │ CompetencyFramework│
                    └────────┬─────────┘
                             │ CONTAINS
                    ┌────────▼─────────┐
              ┌─────│   Competency     │─────┐
              │     └────────┬─────────┘     │
              │              │ HAS_LEVEL     │ PREREQUISITE_FOR
              │     ┌────────▼─────────┐     │
              │     │  ScoringLevel    │     │
              │     └──────────────────┘     │
              │                              │
    REQUIRES  │                              │
              │                              │
         ┌────▼───┐                   ┌──────▼─────┐
         │  Job   │                   │ Competency  │
         └────┬───┘                   └────────────┘
              │
    APPLIED_FOR
              │
    ┌─────────▼───────┐          ┌──────────────┐
    │    Candidate     │──────────│  Interviewer  │
    └─────────┬───────┘          └──────┬───────┘
              │ PARTICIPATED_IN         │ CONDUCTED
              │                         │
         ┌────▼─────────────────────────▼────┐
         │            Interview               │
         └────────────┬──────────────────────┘
                      │ PRODUCED
              ┌───────▼────────┐
              │    Evidence     │
              └───────┬────────┘
                      │ DEMONSTRATES
              ┌───────▼────────┐
              │   Competency   │
              └───────┬────────┘
                      │ VALIDATES
              ┌───────▼────────┐
              │  HireOutcome   │
              └────────────────┘
```

---

## Key Graph Queries

### 1. Quality-of-Hire Feedback: Which competency scores predict job success?

```cypher
MATCH (c:Candidate)-[:PARTICIPATED_IN]->(i:Interview)-[:PRODUCED]->(e:Evidence)
      -[:DEMONSTRATES]->(comp:Competency),
      (c)-[:RESULTED_IN]->(outcome:HireOutcome)
WHERE outcome.is_still_employed = true
  AND outcome.performance_rating >= 4.0
WITH comp, AVG(e.score) AS avg_score, COUNT(DISTINCT c) AS sample_size,
     AVG(outcome.performance_rating) AS avg_performance
WHERE sample_size >= 10
RETURN comp.name, avg_score, sample_size, avg_performance,
       corr(avg_score, avg_performance) AS prediction_strength
ORDER BY prediction_strength DESC;
```

### 2. Bias Detection: Interviewer scoring patterns vs. peers

```cypher
MATCH (interviewer:Interviewer)-[s:SCORED]->(comp:Competency)
WITH interviewer, comp,
     AVG(s.score) AS interviewer_avg,
     STDEV(s.score) AS interviewer_stdev,
     COUNT(s) AS sample_size
MATCH (peer:Interviewer)-[ps:SCORED]->(comp)
WHERE peer <> interviewer
WITH interviewer, comp, interviewer_avg, interviewer_stdev, sample_size,
     AVG(ps.score) AS peer_avg, STDEV(ps.score) AS peer_stdev
WHERE sample_size >= 5
  AND ABS(interviewer_avg - peer_avg) > peer_stdev
RETURN interviewer.name, comp.name,
       interviewer_avg, peer_avg,
       interviewer_avg - peer_avg AS deviation,
       sample_size
ORDER BY ABS(deviation) DESC;
```

### 3. Cross-Role Skill Bridging: Candidate is not ideal for target role but matches another

```cypher
MATCH (c:Candidate {id: "cand-456"})-[:PARTICIPATED_IN]->(i:Interview)
      -[:PRODUCED]->(e:Evidence)-[:DEMONSTRATES]->(comp:Competency)
WITH c, comp, AVG(e.score) AS avg_score
// Find the candidate's competency profile
WITH c, COLLECT({competency: comp, score: avg_score}) AS profile

// Find jobs that match the candidate's strengths
UNWIND profile AS p
MATCH (j:Job)-[r:REQUIRES]->(p.competency)
WHERE p.score >= 3
WITH c, j, COUNT(p) AS matching_competencies,
     SIZE([(j)-[:REQUIRES]->() | 1]) AS total_required,
     AVG(p.score) AS avg_matching_score
WHERE matching_competencies >= total_required * 0.7
RETURN j.title, j.department,
       matching_competencies, total_required,
       avg_matching_score
ORDER BY avg_matching_score DESC;
```

### 4. Competency Pathway Analysis: What skills cluster together in successful hires?

```cypher
MATCH (c:Candidate)-[:RESULTED_IN]->(outcome:HireOutcome)
WHERE outcome.performance_rating >= 4.0
MATCH (c)-[:PARTICIPATED_IN]->(i:Interview)-[:PRODUCED]->(e:Evidence)
      -[:DEMONSTRATES]->(comp:Competency)
WHERE e.evidence_type = 'positive'
WITH c, COLLECT(DISTINCT comp.name) AS competencies
UNWIND competencies AS c1
UNWIND competencies AS c2
WHERE c1 < c2
RETURN c1, c2, COUNT(*) AS co_occurrence
ORDER BY co_occurrence DESC
LIMIT 20;
```

### 5. Interviewer Calibration: How consistent is an interviewer vs. eventual hire outcomes?

```cypher
MATCH (interviewer:Interviewer {id: "user-789"})-[:CONDUCTED]->(i:Interview)
      <-[:PARTICIPATED_IN]-(c:Candidate)-[:RESULTED_IN]->(outcome:HireOutcome)
MATCH (i)-[:PRODUCED]->(e:Evidence)-[:DEMONSTRATES]->(comp:Competency)
WITH interviewer, comp,
     AVG(e.score) AS interview_score,
     AVG(outcome.performance_rating) AS actual_performance,
     COUNT(DISTINCT c) AS sample_size
WHERE sample_size >= 5
RETURN comp.name,
       interview_score,
       actual_performance,
       ABS(interview_score - actual_performance) AS prediction_error,
       sample_size
ORDER BY prediction_error DESC;
```

---

## Data Synchronisation Strategy

The graph is a derived read model, synchronized from PostgreSQL (the transactional source of truth).

```
PostgreSQL (Source of Truth)
    │
    │  Change Data Capture (Debezium / LISTEN/NOTIFY)
    │
    ▼
Sync Service
    │
    │  Transform relational records to graph nodes/relationships
    │
    ▼
Neo4j / AGE (Knowledge Graph)
    │
    │  Cypher queries for intelligence features
    │
    ▼
Analytics & Intelligence API
```

### Sync Rules

| PostgreSQL Event | Graph Operation |
|-----------------|-----------------|
| New interview_score inserted | Create/update SCORED relationship, create Evidence node |
| New transcript_segment with competency tags | Create Evidence node, DEMONSTRATES relationship |
| Hiring decision made | Create HireOutcome node, RESULTED_IN relationship |
| Post-hire outcome recorded | Update HireOutcome node, create/update VALIDATES relationships |
| New competency_dimension created | Create Competency node, CONTAINS relationship |
| Interview completed | Update Interview node properties |

### Apache AGE Alternative

For teams that want graph capabilities without operating a separate Neo4j instance, Apache AGE provides graph query capabilities as a PostgreSQL extension:

```sql
-- Using Apache AGE within PostgreSQL
SELECT * FROM cypher('interview_graph', $$
    MATCH (interviewer:Interviewer)-[s:SCORED]->(comp:Competency)
    WITH interviewer, comp, AVG(s.score) AS avg_score
    RETURN interviewer.name, comp.name, avg_score
    ORDER BY avg_score DESC
$$) AS (interviewer_name agtype, competency_name agtype, avg_score agtype);
```

This eliminates the operational overhead of a separate database while still enabling graph queries. The trade-off is that AGE is less mature than Neo4j and lacks some advanced graph algorithms (e.g., community detection, centrality measures).

---

## Pros

- **Natural query model for intelligence features**: Bias detection, competency correlation, quality-of-hire prediction, and cross-role skill bridging are path traversal problems that are 10-100x simpler to express in Cypher than in SQL with multiple joins
- **Competency ontology modeling**: Competencies naturally form hierarchies, prerequisites, and complementary relationships. A graph captures these semantic relationships natively -- "Problem Solving PREREQUISITE_FOR System Design" is a first-class concept
- **Emergent pattern discovery**: Graph algorithms (community detection, PageRank, shortest path) can discover patterns invisible in relational data: competency clusters that predict success, interviewer networks that reinforce bias, role families with transferable skills
- **Ideal Candidate Profile as graph matching**: The ICP is naturally expressed as a subgraph pattern. Candidate matching is graph similarity scoring, which is more expressive than vector similarity or weighted scoring
- **Explainable AI**: Graph paths provide natural explanations: "This candidate was recommended because they demonstrated strong Problem Solving (score 4.2 across 3 interviews) and Problem Solving has a 0.85 correlation with job success in Senior Backend Engineer roles based on 23 previous hires"
- **Quality-of-hire closed loop**: The VALIDATES relationship directly connects interview evidence to post-hire outcomes, enabling the feedback loop that no incumbent currently delivers

## Cons

- **Polyglot persistence complexity**: Operating two database systems (PostgreSQL + Neo4j) doubles operational overhead: backups, monitoring, upgrades, security patching, and team expertise requirements
- **Data synchronisation challenges**: Keeping the graph in sync with PostgreSQL requires reliable change data capture. Sync lag, failures, and data inconsistencies must be handled carefully
- **Neo4j licensing cost**: Neo4j Enterprise (required for clustering, security, and performance features) has significant licensing costs. The community edition has limitations on database size and clustering
- **Smaller talent pool**: Fewer developers have graph database experience compared to PostgreSQL. Training and hiring costs are higher
- **Write scalability**: Neo4j's write performance is lower than PostgreSQL for high-throughput transactional workloads. Graph is used as a read-optimized model, not for primary writes
- **Apache AGE trade-offs**: Using AGE avoids a separate database but sacrifices Neo4j's mature graph algorithms, visualisation tools, and ecosystem
- **Overkill for small deployments**: For an MVP or small organisations with few interviews, the relational model handles all queries adequately. Graph advantages emerge at scale (thousands of interviews, dozens of interviewers, quality-of-hire data across many hires)

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Transactional database | PostgreSQL 16+ |
| Graph database | Neo4j 5.x (production) or Apache AGE 1.5+ (simpler ops) |
| Change data capture | Debezium (Kafka-based) or PostgreSQL LISTEN/NOTIFY (simpler) |
| Graph visualisation | Neo4j Bloom (for internal analysis) or custom D3.js/vis.js (for app UI) |
| Graph algorithms | Neo4j Graph Data Science library (community detection, centrality, similarity) |
| Object storage | S3/GCS for recordings |
| Search | Elasticsearch for transcript full-text search |
| Caching | Redis for live interview state |

---

## Migration and Scaling Considerations

1. **Start relational, add graph later**: Build the MVP with PostgreSQL only (Suggestion 1 or 3). Add the graph layer when intelligence features become the priority and sufficient data exists to make graph queries valuable (typically after 1,000+ interviews with scoring data).

2. **Apache AGE as stepping stone**: Start with AGE (PostgreSQL extension) to validate that graph queries add value. Migrate to Neo4j only if AGE's limitations (missing graph algorithms, less mature tooling) become blocking.

3. **Graph as read model**: Never write directly to the graph from application code. All mutations go through PostgreSQL. The graph is a materialized view optimized for analytical queries. This makes it safe to rebuild the graph from PostgreSQL at any time.

4. **Incremental graph population**: Start by modelling only the core intelligence triangle (Competency -- Evidence -- Outcome). Add Interviewer patterns, Job requirements, and Candidate profiles as intelligence features are built.

5. **Graph partitioning by organisation**: In a multi-tenant deployment, use separate graph databases (Neo4j) or labelled subgraphs (AGE) per organisation to ensure data isolation and enable per-tenant scaling.

6. **Cold data archival**: Archive graph data for candidates and interviews older than the retention period. Keep aggregated relationships (Interviewer scoring patterns, Competency-Outcome correlations) even after individual evidence nodes are archived.

7. **Performance benchmarking**: Before committing to the graph approach, benchmark the 5 key intelligence queries (listed above) against equivalent PostgreSQL CTEs on realistic data volumes. If the relational model handles them adequately at projected scale, the graph layer adds complexity without sufficient benefit.
