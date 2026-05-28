# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Codebase Refactoring Assistant · Created: 2026-05-11

## Philosophy

This model uses event sourcing as its architectural backbone: every meaningful action in the system -- from analyzing a file to applying a refactoring to changing a finding's status -- is captured as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimized materialized views (projections) serve the UI and API, while the event store remains the single source of truth.

This approach is particularly well-suited for a codebase refactoring assistant because the domain *inherently requires temporal reasoning*. Core questions like "what was the technical debt score of this file three months ago?", "how has this hotspot evolved over time?", and "what was the state of this transformation before it was modified?" are first-class queries in an event-sourced system but require complex workarounds in a traditional CRUD model. The immutable event log also naturally satisfies NIST SSDF compliance requirements for demonstrating that automated transformations did not introduce security regressions.

The model draws on patterns from Microsoft's Azure Architecture Center (CQRS + Event Sourcing), Martin Fowler's event sourcing documentation, and the AxonIQ framework's aggregate/event/projection pattern. The write side (command processing) is intentionally separated from the read side (query projections), allowing each to scale independently.

**Best for:** Enterprises requiring full audit trails, temporal queries ("what was true on date X?"), compliance reporting (NIST SSDF, SOC 2), and teams building AI analytics over historical code quality trends.

**Trade-offs:**
- (+) Complete, immutable audit trail for every state change -- essential for NIST SSDF and SOC 2 compliance
- (+) Temporal queries are trivial: replay events to any point in time
- (+) AI/ML analytics can process the full event stream to identify patterns (e.g., "refactorings that get reverted")
- (+) CQRS separation allows independent scaling of command and query sides
- (+) Natural fit for distributed/microservices architecture
- (-) Higher complexity: developers must think in events, aggregates, and projections rather than tables
- (-) Eventual consistency between event store and projections requires careful handling
- (-) Event schema evolution (versioning) adds ongoing maintenance burden
- (-) Storage requirements are higher since nothing is deleted, only appended
- (-) Debugging requires tooling to replay event streams and inspect projection state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NIST SP 800-218 (SSDF) | The immutable event store serves as the audit trail for PW (Produce Well-Secured Software) practices; every transformation event records pre/post state |
| ISO/IEC 25010:2023 | Quality characteristic values are derived from event replay, enabling point-in-time quality snapshots |
| ISO 5055 (CISQ) | CWE-based quality measures are events themselves; score changes are tracked as quality_score_computed events |
| SARIF 2.1.0 | Analysis findings are stored as events; SARIF export replays the finding event stream filtered by analysis run |
| SQALE Method | Remediation cost changes are events; the current debt total is a projection that can be queried at any historical point |
| OWASP Top 10:2025 | OWASP category mappings are immutable metadata on finding events |

---

## Event Store Foundation

```sql
-- ============================================================
-- EVENT STORE (append-only, immutable)
-- ============================================================

-- Central event store: all domain events live here
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,   -- 'Repository', 'SourceFile', 'Finding', 'Transformation', 'Campaign'
    aggregate_id    UUID NOT NULL,           -- ID of the aggregate this event belongs to
    event_type      VARCHAR(200) NOT NULL,   -- 'RepositoryRegistered', 'AnalysisCompleted', 'FindingDetected', etc.
    event_version   INTEGER NOT NULL,        -- monotonically increasing per aggregate
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, user context, etc.
    organization_id UUID NOT NULL,           -- tenant scoping
    caused_by       UUID,                    -- event_id of the triggering event (causal chain)
    actor_id        UUID,                    -- user or system that caused this event
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

-- Optimized for replaying a single aggregate's event stream
CREATE INDEX idx_events_aggregate ON events(aggregate_id, event_version);

-- Optimized for querying all events of a type (for building projections)
CREATE INDEX idx_events_type ON events(event_type, occurred_at);

-- Tenant-scoped queries
CREATE INDEX idx_events_org ON events(organization_id, occurred_at);

-- Causal chain queries
CREATE INDEX idx_events_caused_by ON events(caused_by) WHERE caused_by IS NOT NULL;

-- Time-range queries for analytics
CREATE INDEX idx_events_time ON events(occurred_at);

-- Partition by month for manageability at scale
-- In production, this would use PostgreSQL declarative partitioning:
-- CREATE TABLE events (...) PARTITION BY RANGE (occurred_at);

-- ============================================================
-- EVENT SCHEMA REGISTRY
-- ============================================================

-- Tracks event type schemas for evolution/versioning
CREATE TABLE event_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(200) NOT NULL,
    schema_version  INTEGER NOT NULL,
    json_schema     JSONB NOT NULL,          -- JSON Schema for the payload
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_type, schema_version)
);

CREATE INDEX idx_event_schemas_type ON event_schemas(event_type, is_current);
```

### Event Type Catalog

The system defines the following event types, organized by aggregate:

```sql
-- Example: Event type definitions (documented, not stored as tables)
--
-- AGGREGATE: Organization
--   OrganizationCreated        { name, slug, plan_tier }
--   OrganizationSettingsUpdated { settings_diff }
--   MemberAdded                { user_id, role }
--   MemberRoleChanged          { user_id, old_role, new_role }
--   MemberRemoved              { user_id }
--
-- AGGREGATE: Repository
--   RepositoryRegistered       { name, vcs_provider, vcs_url, default_branch }
--   RepositorySettingsUpdated  { settings_diff }
--   RepositoryDeactivated      { reason }
--   RepositoryReactivated      {}
--   BranchDiscovered           { branch_name, head_sha }
--   BranchUpdated              { branch_name, old_sha, new_sha }
--
-- AGGREGATE: AnalysisRun
--   AnalysisRequested          { repository_id, branch, commit_sha, trigger_type }
--   AnalysisStarted            { worker_id }
--   FileAnalyzed               { file_path, language, loc, complexity }
--   FindingDetected            { file_path, rule_key, severity, line, message, cwe_id }
--   FindingResolved            { finding_id, resolution }  -- 'fixed', 'wont_fix', 'false_positive'
--   HotspotIdentified          { file_path, churn_score, complexity_score, hotspot_score, rank }
--   QualityScoreComputed       { characteristic, score, debt_minutes, rating }
--   QualityGateEvaluated       { profile_id, status, conditions[] }
--   AnalysisCompleted          { files_analyzed, findings_count, debt_total_minutes, duration_ms }
--   AnalysisFailed             { error_message, stack_trace }
--
-- AGGREGATE: Transformation
--   TransformationPlanned      { recipe_id, file_path, finding_id, ai_explanation }
--   CharacterizationTestGenerated { test_file, test_content, framework, method }
--   PreRefactorTestExecuted    { total, passed, failed, coverage }
--   CodeTransformed            { diff, code_before, code_after, risk_score }
--   PostRefactorTestExecuted   { total, passed, failed, coverage }
--   TransformationApproved     { approved_by }
--   TransformationApplied      { commit_sha, pr_url }
--   TransformationRejected     { rejected_by, reason }
--   TransformationFailed       { error_message, stage }
--
-- AGGREGATE: Campaign
--   CampaignCreated            { name, description, target_repos[] }
--   CampaignStarted            {}
--   CampaignTransformationAdded { transformation_id }
--   CampaignPaused             { reason }
--   CampaignResumed            {}
--   CampaignCompleted          { summary_stats }
--   CampaignCancelled          { reason }
--
-- AGGREGATE: DebtBudget
--   BudgetAllocated            { repository_id, sprint_name, budget_minutes, period_start, period_end }
--   TaskAssigned               { finding_id, hotspot_id, user_id, estimated_minutes, reason }
--   TaskCompleted              { task_id, actual_minutes }
--   TaskSkipped                { task_id, reason }
--   BudgetAdjusted             { old_budget_minutes, new_budget_minutes, reason }
--
-- AGGREGATE: VCSHistory
--   CommitIngested             { sha, author_email, authored_at, message, files[] }
--   ChurnMetricsComputed       { file_path, window_type, commit_count, churn_score }
--   CouplingDetected           { file_a, file_b, co_change_count, coupling_score }
```

---

## Read-Side Projections (Materialized Views)

The projections below are built by consuming events and maintaining current-state tables. They are *derived* and can be rebuilt from scratch by replaying the event store.

```sql
-- ============================================================
-- PROJECTIONS (derived from events, rebuildable)
-- ============================================================

-- Current state of organizations (rebuilt from Organization events)
CREATE TABLE proj_organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    member_count    INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,           -- last processed event (idempotency)
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Current state of repositories
CREATE TABLE proj_repositories (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    vcs_provider    VARCHAR(50) NOT NULL,
    vcs_url         TEXT NOT NULL,
    default_branch  VARCHAR(255) NOT NULL,
    primary_language VARCHAR(50),
    languages       TEXT[],
    loc_total       INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_analysis_status VARCHAR(50),
    last_analysis_at TIMESTAMPTZ,
    total_findings  INTEGER NOT NULL DEFAULT 0,
    total_debt_minutes INTEGER NOT NULL DEFAULT 0,
    hotspot_count   INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_repos_org ON proj_repositories(organization_id);

-- Current state of source files with latest metrics
CREATE TABLE proj_source_files (
    id              UUID PRIMARY KEY,
    repository_id   UUID NOT NULL,
    file_path       TEXT NOT NULL,
    language        VARCHAR(50),
    loc             INTEGER,
    complexity_score NUMERIC(10,2),
    churn_score     NUMERIC(10,2),           -- latest computed churn
    hotspot_score   NUMERIC(10,2),           -- latest composite hotspot score
    hotspot_rank    INTEGER,
    open_finding_count INTEGER NOT NULL DEFAULT 0,
    debt_minutes    INTEGER NOT NULL DEFAULT 0,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    last_event_id   UUID NOT NULL,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (repository_id, file_path)
);

CREATE INDEX idx_proj_files_repo ON proj_source_files(repository_id);
CREATE INDEX idx_proj_files_hotspot ON proj_source_files(repository_id, hotspot_score DESC NULLS LAST);

-- Current state of findings
CREATE TABLE proj_findings (
    id              UUID PRIMARY KEY,
    analysis_id     UUID NOT NULL,
    repository_id   UUID NOT NULL,
    file_id         UUID NOT NULL,
    file_path       TEXT NOT NULL,
    rule_key        VARCHAR(255) NOT NULL,
    rule_name       VARCHAR(500),
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    line_start      INTEGER,
    message         TEXT NOT NULL,
    cwe_id          INTEGER,
    owasp_id        VARCHAR(10),
    characteristic  VARCHAR(50),
    remediation_minutes INTEGER,
    assigned_to     UUID,
    transformation_id UUID,
    last_event_id   UUID NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_findings_repo ON proj_findings(repository_id);
CREATE INDEX idx_proj_findings_file ON proj_findings(file_id);
CREATE INDEX idx_proj_findings_status ON proj_findings(status);
CREATE INDEX idx_proj_findings_severity ON proj_findings(severity);

-- Current state of transformations
CREATE TABLE proj_transformations (
    id              UUID PRIMARY KEY,
    campaign_id     UUID,
    recipe_id       UUID NOT NULL,
    repository_id   UUID NOT NULL,
    file_id         UUID NOT NULL,
    file_path       TEXT NOT NULL,
    finding_id      UUID,
    status          VARCHAR(50) NOT NULL,
    diff_content    TEXT,
    ai_explanation  TEXT,
    risk_score      NUMERIC(5,2),
    pre_test_passed BOOLEAN,
    post_test_passed BOOLEAN,
    pr_url          TEXT,
    pr_status       VARCHAR(50),
    last_event_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_transforms_campaign ON proj_transformations(campaign_id);
CREATE INDEX idx_proj_transforms_repo ON proj_transformations(repository_id);
CREATE INDEX idx_proj_transforms_status ON proj_transformations(status);

-- Current state of campaigns
CREATE TABLE proj_campaigns (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    created_by      UUID NOT NULL,
    total_transformations INTEGER NOT NULL DEFAULT 0,
    completed_transformations INTEGER NOT NULL DEFAULT 0,
    failed_transformations INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_campaigns_org ON proj_campaigns(organization_id);
```

---

## Time-Series Analytics Projections

```sql
-- ============================================================
-- ANALYTICS PROJECTIONS (time-series from events)
-- ============================================================

-- Daily technical debt snapshot per repository
CREATE TABLE proj_daily_debt_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    total_debt_minutes INTEGER NOT NULL,
    debt_by_characteristic JSONB NOT NULL,
    -- Example: {"maintainability": 4500, "reliability": 1200, "security": 800, "performance": 300}
    finding_counts  JSONB NOT NULL,
    -- Example: {"open": 142, "resolved": 38, "wont_fix": 5}
    hotspot_count   INTEGER NOT NULL,
    sqale_rating    CHAR(1),
    UNIQUE (repository_id, snapshot_date)
);

CREATE INDEX idx_daily_debt_repo_date ON proj_daily_debt_snapshot(repository_id, snapshot_date);

-- Transformation outcome analytics
CREATE TABLE proj_transformation_outcomes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transformation_id UUID NOT NULL,
    repository_id   UUID NOT NULL,
    recipe_type     VARCHAR(50),
    outcome         VARCHAR(50) NOT NULL,    -- 'applied', 'rejected', 'reverted', 'failed'
    risk_score      NUMERIC(5,2),
    test_delta      INTEGER,                 -- change in test count
    coverage_delta  NUMERIC(5,2),            -- change in coverage %
    debt_reduced_minutes INTEGER,
    time_to_review_minutes INTEGER,          -- time from ready_for_review to approved/rejected
    completed_at    TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_transform_outcomes_repo ON proj_transformation_outcomes(repository_id);
CREATE INDEX idx_transform_outcomes_date ON proj_transformation_outcomes(completed_at);

-- Developer activity analytics (for debt task assignment)
CREATE TABLE proj_developer_file_affinity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,
    user_email      VARCHAR(255) NOT NULL,
    file_path       TEXT NOT NULL,
    commit_count    INTEGER NOT NULL,
    last_commit_at  TIMESTAMPTZ NOT NULL,
    affinity_score  NUMERIC(5,4) NOT NULL,   -- 0-1, higher = more familiar
    UNIQUE (repository_id, user_email, file_path)
);

CREATE INDEX idx_dev_affinity_repo_user ON proj_developer_file_affinity(repository_id, user_email);
CREATE INDEX idx_dev_affinity_file ON proj_developer_file_affinity(file_path);
```

---

## Reference Data (Static, Not Event-Sourced)

```sql
-- ============================================================
-- REFERENCE DATA (static, not event-sourced)
-- ============================================================

CREATE TABLE ref_cwe_weaknesses (
    cwe_id          INTEGER PRIMARY KEY,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    parent_cwe_id   INTEGER REFERENCES ref_cwe_weaknesses(cwe_id),
    abstraction     VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'Draft'
);

CREATE TABLE ref_owasp_categories (
    owasp_id        VARCHAR(10) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    version_year    INTEGER NOT NULL
);

CREATE TABLE ref_quality_characteristics (
    code            VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    iso_25010_ref   VARCHAR(100),
    iso_5055_ref    VARCHAR(100)
);

CREATE TABLE ref_detection_rules (
    rule_key        VARCHAR(255) PRIMARY KEY,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    language        VARCHAR(50),
    severity        VARCHAR(20) NOT NULL,
    rule_type       VARCHAR(50) NOT NULL,
    characteristic  VARCHAR(50) REFERENCES ref_quality_characteristics(code),
    cwe_id          INTEGER REFERENCES ref_cwe_weaknesses(cwe_id),
    owasp_id        VARCHAR(10) REFERENCES ref_owasp_categories(owasp_id),
    remediation_minutes INTEGER
);

CREATE TABLE ref_refactoring_recipes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(500) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    recipe_type     VARCHAR(50) NOT NULL,
    source_language VARCHAR(50),
    target_language VARCHAR(50),
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    estimated_risk  VARCHAR(20)
);
```

---

## Command Handlers (Application Layer Pattern)

```sql
-- ============================================================
-- COMMAND PROCESSING (idempotency & deduplication)
-- ============================================================

-- Tracks processed commands for idempotency
CREATE TABLE processed_commands (
    command_id      UUID PRIMARY KEY,
    command_type    VARCHAR(200) NOT NULL,
    aggregate_id    UUID NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    result_event_id UUID NOT NULL REFERENCES events(event_id)
);

CREATE INDEX idx_processed_cmds_type ON processed_commands(command_type, processed_at);

-- Snapshot store for aggregate state (optimization: avoid replaying full event stream)
CREATE TABLE aggregate_snapshots (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version at time of snapshot
    state           JSONB NOT NULL,          -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);

-- Projection checkpoints (track which event each projection has consumed up to)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example: Temporal Query

One of the key advantages of event sourcing -- querying historical state:

```sql
-- "What was the technical debt of repository X on March 1, 2026?"
-- Answer: replay all QualityScoreComputed events for that repo up to March 1

SELECT
    payload->>'characteristic' AS characteristic,
    payload->>'debt_minutes' AS debt_minutes,
    payload->>'rating' AS rating,
    occurred_at
FROM events
WHERE aggregate_type = 'AnalysisRun'
  AND event_type = 'QualityScoreComputed'
  AND organization_id = :org_id
  AND (payload->>'repository_id')::UUID = :repo_id
  AND occurred_at <= '2026-03-01T23:59:59Z'
ORDER BY occurred_at DESC
LIMIT 4;  -- one per characteristic (maintainability, reliability, security, performance)

-- "Show me all state changes for transformation X"
-- Answer: replay the transformation's event stream

SELECT
    event_type,
    payload,
    actor_id,
    occurred_at
FROM events
WHERE aggregate_type = 'Transformation'
  AND aggregate_id = :transformation_id
ORDER BY event_version ASC;
```

---

## Example: Rebuilding a Projection

```sql
-- Rebuild the proj_findings projection from scratch
-- 1. Truncate the projection
TRUNCATE proj_findings;

-- 2. Replay all FindingDetected and FindingResolved events
-- (In practice, this runs in application code, but the logic is:)

-- For each FindingDetected event:
INSERT INTO proj_findings (id, analysis_id, repository_id, file_id, ...)
SELECT
    aggregate_id AS id,
    (payload->>'analysis_id')::UUID,
    (payload->>'repository_id')::UUID,
    ...
FROM events
WHERE event_type = 'FindingDetected'
ORDER BY occurred_at;

-- For each FindingResolved event:
UPDATE proj_findings
SET status = payload->>'resolution',
    resolved_at = occurred_at,
    updated_at = occurred_at
FROM events
WHERE event_type = 'FindingResolved'
  AND proj_findings.id = events.aggregate_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | Core event table + schema registry |
| Command Processing | 3 | Processed commands, snapshots, projection checkpoints |
| Projections (Current State) | 6 | Organizations, repositories, files, findings, transformations, campaigns |
| Projections (Analytics) | 3 | Daily debt snapshots, transformation outcomes, developer affinity |
| Reference Data | 4 | CWE, OWASP, quality characteristics, detection rules, recipes |
| **Total** | **19** | Plus the event store which contains *all* domain data |

---

## Key Design Decisions

1. **Single event table rather than per-aggregate-type tables.** This simplifies the infrastructure (one table to partition, one table to back up) and enables cross-aggregate queries like "all events by user X in the last hour." The `aggregate_type` + `aggregate_id` composite forms the logical partition within the physical table.

2. **JSONB payload rather than typed columns per event.** Each event type has a different structure; modeling every event type as a separate table would create hundreds of tables. JSONB payloads with JSON Schema validation (via `event_schemas` table) provide schema flexibility while maintaining contractual guarantees.

3. **Projections are explicitly marked as derived.** Every projection table is prefixed with `proj_` and includes a `last_event_id` column for idempotent replay. If a projection becomes corrupted or needs to change shape, it can be dropped and rebuilt from the event store.

4. **Aggregate snapshots for performance.** For long-lived aggregates (repositories with thousands of events), replaying the full event stream on every command is expensive. Snapshots capture the aggregate state at a point in time; only events after the snapshot need to be replayed.

5. **Causal chain tracking via `caused_by`.** When a `TransformationApplied` event triggers a `FindingResolved` event, the causal link is preserved. This enables compliance reporting ("this finding was resolved because transformation X was applied") and root-cause analysis.

6. **Reference data is NOT event-sourced.** CWE weaknesses, OWASP categories, and detection rules are externally maintained taxonomies that change infrequently. Storing them as traditional tables avoids the overhead of event sourcing for data that doesn't benefit from temporal querying.

7. **Time-series analytics projections are purpose-built.** The `proj_daily_debt_snapshot` table is optimized for dashboard charts showing debt trends over time. Rather than running expensive event replays for every graph render, the projection maintains pre-computed daily snapshots.

8. **Event partitioning by time (recommended for production).** The events table should use PostgreSQL declarative partitioning by `occurred_at` month. This enables efficient time-range queries and allows old partitions to be archived to cold storage without affecting query performance on recent events.

9. **Multi-tenancy via `organization_id` on every event.** Each event carries its tenant context, enabling row-level security at the event store level. Projections inherit tenant scoping from the events that built them.

10. **No event deletion, ever.** Events are immutable and append-only. "Corrections" are modeled as compensating events (e.g., `FindingReclassified` rather than updating the original `FindingDetected` event). This is the core guarantee that enables full audit trail compliance.
