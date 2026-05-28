# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Codebase Refactoring Assistant · Created: 2026-05-11

## Philosophy

This model uses a hybrid approach: core structural fields are stored as typed relational columns with foreign keys, while variable, language-specific, and extensible data lives in JSONB columns. The result is a schema with fewer tables than a fully normalized model, faster iteration on new features (adding a language-specific metric doesn't require a migration), and the ability to handle the extreme variability inherent in polyglot code analysis.

A codebase refactoring assistant must analyze code across dozens of languages (Python, TypeScript, Java, Go, Rust, C#, PHP, etc.), each with different complexity metrics, AST structures, framework-specific patterns, and refactoring rules. A purely normalized model would either need per-language metric tables (explosion of tables) or a wide table with mostly-NULL language-specific columns. The JSONB hybrid sidesteps this by putting language-invariant fields (file path, LOC, severity) in typed columns and language-variant data (framework version, AST node types, language-specific complexity sub-scores) in JSONB.

This pattern is well-established in production SaaS platforms: PostgreSQL's JSONB with GIN indexes provides indexed access to nested JSON fields, supporting containment queries (`@>`), path queries (`->>`, `#>>`), and partial updates. The approach enables rapid MVP development while retaining the option to "promote" frequently-queried JSONB fields to typed columns as query patterns stabilize.

**Best for:** Teams building an MVP quickly, supporting many programming languages with varying metadata, and wanting to iterate on the data model without frequent migrations.

**Trade-offs:**
- (+) Fewer tables (~18-22) compared to normalized model (~33)
- (+) No schema migration needed when adding language-specific or framework-specific fields
- (+) Polyglot analysis metadata fits naturally in JSONB without per-language tables
- (+) Rapid feature development: new attributes can ship without DDL changes
- (+) GIN indexes on JSONB provide efficient querying of structured metadata
- (-) No foreign key constraints inside JSONB -- referential integrity is application-enforced for nested data
- (-) Complex JSONB queries can be less readable and harder to optimize than relational JOINs
- (-) JSONB storage is less space-efficient than typed columns for frequently-used fields
- (-) Risk of "schema drift" if JSONB structures are not documented and validated at the application layer
- (-) Aggregation queries over JSONB fields (e.g., SUM of a nested numeric) are slower than over typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 25010:2023 | Quality characteristics stored as a typed enum column; sub-characteristic detail in JSONB `quality_detail` field |
| ISO 5055 (CISQ) | Four quality dimensions as typed columns; per-CWE weakness detail in JSONB |
| SQALE Method | Remediation cost as typed INTEGER column; SQALE breakdown (characteristic > sub-characteristic > requirement) in JSONB |
| CWE (MITRE) | `cwe_id` as typed INTEGER column for primary mapping; full CWE hierarchy path in JSONB `cwe_context` |
| SARIF 2.1.0 | Finding `location` field stored as JSONB matching SARIF's `physicalLocation` schema; direct serialization to SARIF |
| OWASP Top 10:2025 | `owasp_id` as typed VARCHAR column; extended OWASP context in JSONB |
| NIST SP 800-218 | Audit entries stored as JSONB-rich rows in `audit_log` table; SSDF practice mapping in JSONB metadata |

---

## Organization & Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATION & USERS
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_quality_gate": "uuid-here",
    --   "vcs_integrations": [{"provider": "github", "org": "acme-corp", "token_ref": "vault:github/acme"}],
    --   "notification_channels": [{"type": "slack", "webhook_url": "..."}],
    --   "debt_budget_defaults": {"weekly_minutes": 480, "auto_assign": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL,
    auth_provider_id VARCHAR(255),
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "github_username": "jdoe",
    --   "preferred_languages": ["typescript", "python"],
    --   "notification_preferences": {"email": true, "slack": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["view_findings", "apply_transformations", "manage_recipes", "admin"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
```

---

## Repositories & Source Files

```sql
-- ============================================================
-- REPOSITORIES & FILES
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    vcs_provider    VARCHAR(50) NOT NULL,
    vcs_url         TEXT NOT NULL,
    default_branch  VARCHAR(255) NOT NULL DEFAULT 'main',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_analyzed_at TIMESTAMPTZ,
    repo_metadata   JSONB NOT NULL DEFAULT '{}',
    -- repo_metadata example:
    -- {
    --   "languages": {"typescript": 45000, "python": 12000, "go": 8000},
    --   "primary_language": "typescript",
    --   "loc_total": 65000,
    --   "framework_versions": {"next": "15.2", "django": "5.1", "go": "1.23"},
    --   "build_systems": ["npm", "pip", "go"],
    --   "test_frameworks": ["jest", "pytest", "go_test"],
    --   "ci_provider": "github_actions"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);
CREATE INDEX idx_repos_metadata_languages ON repositories USING GIN (repo_metadata jsonb_path_ops);

CREATE TABLE source_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,
    language        VARCHAR(50),
    loc             INTEGER,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,

    -- Core metrics (relational columns for frequent queries)
    complexity_score    NUMERIC(10,2),       -- cyclomatic or cognitive complexity
    churn_score         NUMERIC(10,2),       -- normalized churn from VCS history
    hotspot_score       NUMERIC(10,2),       -- composite: churn * complexity
    hotspot_rank        INTEGER,
    open_finding_count  INTEGER NOT NULL DEFAULT 0,
    debt_minutes        INTEGER NOT NULL DEFAULT 0,

    -- Language-specific and extended metrics (JSONB for flexibility)
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- metrics example (TypeScript file):
    -- {
    --   "cyclomatic_complexity": 42,
    --   "cognitive_complexity": 38,
    --   "halstead": {"volume": 1200, "difficulty": 15.3, "effort": 18360},
    --   "maintainability_index": 45.2,
    --   "dependencies": ["react", "next/router", "@stripe/stripe-js"],
    --   "exports": ["PaymentProcessor", "processPayment", "refundPayment"],
    --   "test_coverage": 62.5,
    --   "duplication_blocks": 3,
    --   "framework_patterns": {
    --     "react_hooks": ["useState", "useEffect", "useCallback"],
    --     "next_api_routes": true
    --   }
    -- }
    --
    -- metrics example (Java file):
    -- {
    --   "cyclomatic_complexity": 28,
    --   "cognitive_complexity": 22,
    --   "class_count": 2,
    --   "method_count": 15,
    --   "coupling_between_objects": 8,
    --   "depth_of_inheritance": 3,
    --   "spring_annotations": ["@Service", "@Transactional", "@Autowired"],
    --   "framework_version": "spring-boot:3.2.1",
    --   "test_coverage": 78.0
    -- }

    vcs_stats       JSONB NOT NULL DEFAULT '{}',
    -- vcs_stats example:
    -- {
    --   "total_commits": 147,
    --   "unique_authors": 8,
    --   "last_modified_at": "2026-05-01T14:30:00Z",
    --   "last_author": "jane@acme.com",
    --   "monthly_churn": [
    --     {"month": "2026-04", "commits": 12, "lines_added": 340, "lines_deleted": 120},
    --     {"month": "2026-03", "commits": 8, "lines_added": 210, "lines_deleted": 90}
    --   ],
    --   "top_contributors": [
    --     {"email": "jane@acme.com", "commits": 45, "affinity": 0.85},
    --     {"email": "bob@acme.com", "commits": 28, "affinity": 0.62}
    --   ],
    --   "co_changed_files": [
    --     {"path": "src/payment/types.ts", "coupling_score": 0.82},
    --     {"path": "src/payment/__tests__/processor.test.ts", "coupling_score": 0.95}
    --   ]
    -- }

    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, file_path)
);

CREATE INDEX idx_files_repo ON source_files(repository_id);
CREATE INDEX idx_files_language ON source_files(language);
CREATE INDEX idx_files_hotspot ON source_files(repository_id, hotspot_score DESC NULLS LAST);
CREATE INDEX idx_files_debt ON source_files(repository_id, debt_minutes DESC);
CREATE INDEX idx_files_metrics ON source_files USING GIN (metrics jsonb_path_ops);
CREATE INDEX idx_files_vcs ON source_files USING GIN (vcs_stats jsonb_path_ops);
```

---

## Analysis & Findings

```sql
-- ============================================================
-- ANALYSIS RUNS & FINDINGS
-- ============================================================

CREATE TABLE analysis_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    triggered_by    UUID REFERENCES users(id),
    trigger_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    commit_sha      VARCHAR(40),
    branch          VARCHAR(255),

    -- Summary stats (typed for aggregation)
    files_analyzed  INTEGER,
    findings_total  INTEGER,
    findings_new    INTEGER,
    findings_resolved INTEGER,
    debt_total_minutes INTEGER,

    -- Detailed results (JSONB for flexibility)
    results         JSONB NOT NULL DEFAULT '{}',
    -- results example:
    -- {
    --   "by_severity": {"critical": 3, "major": 18, "minor": 42, "info": 15},
    --   "by_type": {"bug": 5, "vulnerability": 2, "code_smell": 56, "security_hotspot": 15},
    --   "by_language": {"typescript": 52, "python": 18, "go": 8},
    --   "by_characteristic": {
    --     "maintainability": {"issues": 48, "debt_minutes": 3200, "rating": "C"},
    --     "reliability": {"issues": 12, "debt_minutes": 800, "rating": "B"},
    --     "security": {"issues": 8, "debt_minutes": 600, "rating": "B"},
    --     "performance": {"issues": 10, "debt_minutes": 400, "rating": "A"}
    --   },
    --   "quality_gate": {"status": "failed", "conditions": [
    --     {"metric": "new_critical", "operator": "GT", "threshold": 0, "actual": 3, "status": "failed"}
    --   ]},
    --   "hotspots_identified": 12,
    --   "duration_ms": 45000
    -- }

    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_repo ON analysis_runs(repository_id);
CREATE INDEX idx_runs_status ON analysis_runs(status);
CREATE INDEX idx_runs_date ON analysis_runs(created_at DESC);

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,

    -- Core fields (typed columns for frequent filtering and aggregation)
    rule_key        VARCHAR(255) NOT NULL,
    rule_name       VARCHAR(500),
    severity        VARCHAR(20) NOT NULL,
    finding_type    VARCHAR(50) NOT NULL,     -- 'bug', 'vulnerability', 'code_smell', 'security_hotspot'
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    characteristic  VARCHAR(50),              -- ISO 25010 quality characteristic
    cwe_id          INTEGER,
    owasp_id        VARCHAR(10),
    remediation_minutes INTEGER,
    assigned_to     UUID REFERENCES users(id),

    -- Location (JSONB matching SARIF physicalLocation schema)
    location        JSONB NOT NULL,
    -- location example:
    -- {
    --   "file_path": "src/payment/processor.ts",
    --   "region": {
    --     "startLine": 42,
    --     "startColumn": 5,
    --     "endLine": 58,
    --     "endColumn": 2
    --   },
    --   "context_region": {
    --     "startLine": 40,
    --     "endLine": 60,
    --     "snippet": "function processPayment(amount: number) {\n  // ... 18 lines of nested conditionals\n}"
    --   }
    -- }

    -- Extended finding detail (JSONB for language-specific and rule-specific context)
    detail          JSONB NOT NULL DEFAULT '{}',
    -- detail example:
    -- {
    --   "message": "Function 'processPayment' has cognitive complexity of 38 (threshold: 15)",
    --   "ai_explanation": "This function handles payment processing with deeply nested error handling for 5 different payment providers. The complexity stems from...",
    --   "related_findings": ["uuid-1", "uuid-2"],
    --   "data_flow": [
    --     {"step": 1, "location": "line 42", "description": "User input received"},
    --     {"step": 2, "location": "line 48", "description": "Passed to payment API without validation"}
    --   ],
    --   "tags": ["high-churn", "payment-critical", "needs-decomposition"],
    --   "effort_breakdown": {
    --     "analysis": 15, "coding": 60, "testing": 30, "review": 15
    --   }
    -- }

    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_analysis ON findings(analysis_id);
CREATE INDEX idx_findings_file ON findings(source_file_id);
CREATE INDEX idx_findings_repo ON findings(repository_id);
CREATE INDEX idx_findings_status ON findings(status);
CREATE INDEX idx_findings_severity ON findings(severity);
CREATE INDEX idx_findings_type ON findings(finding_type);
CREATE INDEX idx_findings_cwe ON findings(cwe_id) WHERE cwe_id IS NOT NULL;
CREATE INDEX idx_findings_location ON findings USING GIN (location jsonb_path_ops);
CREATE INDEX idx_findings_detail ON findings USING GIN (detail jsonb_path_ops);
```

---

## Refactoring Recipes & Transformations

```sql
-- ============================================================
-- RECIPES, CAMPAIGNS & TRANSFORMATIONS
-- ============================================================

CREATE TABLE refactoring_recipes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name            VARCHAR(500) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    recipe_type     VARCHAR(50) NOT NULL,
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- Recipe definition (JSONB for maximum flexibility)
    definition      JSONB NOT NULL,
    -- definition example (framework migration recipe):
    -- {
    --   "description": "Migrate React class components to functional components with hooks",
    --   "source_language": "typescript",
    --   "applicable_frameworks": ["react@16", "react@17"],
    --   "target_framework": "react@18",
    --   "estimated_risk": "medium",
    --   "addressed_rules": ["ts:S3776", "ts:S1192"],
    --   "preconditions": [
    --     {"type": "file_pattern", "pattern": "**/*.tsx"},
    --     {"type": "import_present", "module": "react", "symbol": "Component"}
    --   ],
    --   "transformation_steps": [
    --     {"step": "extract_state", "description": "Identify this.state usages"},
    --     {"step": "convert_lifecycle", "description": "Map componentDidMount to useEffect"},
    --     {"step": "replace_class", "description": "Convert class to function"}
    --   ],
    --   "test_strategy": "generate_snapshot_tests",
    --   "rollback_strategy": "git_revert"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recipes_org ON refactoring_recipes(organization_id);
CREATE INDEX idx_recipes_type ON refactoring_recipes(recipe_type);
CREATE INDEX idx_recipes_definition ON refactoring_recipes USING GIN (definition jsonb_path_ops);

CREATE TABLE refactoring_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    created_by      UUID NOT NULL REFERENCES users(id),

    -- Campaign configuration and progress (JSONB)
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "target_repos": ["uuid-1", "uuid-2", "uuid-3"],
    --   "recipe_ids": ["uuid-a", "uuid-b"],
    --   "filters": {
    --     "languages": ["typescript"],
    --     "min_hotspot_score": 50,
    --     "severity_threshold": "major"
    --   },
    --   "execution_mode": "batch",
    --   "auto_create_pr": true,
    --   "require_test_pass": true,
    --   "max_concurrent": 5
    -- }

    progress        JSONB NOT NULL DEFAULT '{}',
    -- progress example:
    -- {
    --   "total_transformations": 45,
    --   "completed": 32,
    --   "failed": 3,
    --   "pending": 10,
    --   "debt_reduced_minutes": 2400,
    --   "prs_created": 28,
    --   "prs_merged": 20
    -- }

    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaigns_org ON refactoring_campaigns(organization_id);
CREATE INDEX idx_campaigns_status ON refactoring_campaigns(status);

CREATE TABLE transformations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID REFERENCES refactoring_campaigns(id) ON DELETE SET NULL,
    recipe_id       UUID NOT NULL REFERENCES refactoring_recipes(id),
    source_file_id  UUID NOT NULL REFERENCES source_files(id),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    finding_id      UUID REFERENCES findings(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',

    -- Core transformation data (typed for queries)
    risk_score      NUMERIC(5,2),
    pr_url          TEXT,
    pr_status       VARCHAR(50),

    -- Transformation content and context (JSONB)
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "diff": "--- a/src/payment/processor.ts\n+++ b/src/payment/processor.ts\n@@ -42,18 +42,12 @@...",
    --   "code_before": "class PaymentProcessor extends Component { ...",
    --   "code_after": "function PaymentProcessor() { const [state, setState] = useState(...); ...",
    --   "ai_explanation": "This class component has high cognitive complexity (38) due to...",
    --   "affected_imports": ["react", "@stripe/stripe-js"],
    --   "impact_analysis": {
    --     "downstream_files": ["src/checkout/index.tsx", "src/admin/payments.tsx"],
    --     "api_contract_changes": false,
    --     "type_signature_changes": true
    --   }
    -- }

    -- Test execution results (JSONB)
    test_results    JSONB NOT NULL DEFAULT '{}',
    -- test_results example:
    -- {
    --   "characterization_tests": {
    --     "count": 5,
    --     "framework": "jest",
    --     "generation_method": "ai_generated",
    --     "test_file": "src/payment/__tests__/processor.char.test.ts"
    --   },
    --   "pre_refactor": {
    --     "total": 12, "passed": 12, "failed": 0, "coverage": 78.5,
    --     "duration_ms": 3200, "executed_at": "2026-05-10T10:00:00Z"
    --   },
    --   "post_refactor": {
    --     "total": 12, "passed": 12, "failed": 0, "coverage": 82.1,
    --     "duration_ms": 2800, "executed_at": "2026-05-10T10:02:00Z"
    --   },
    --   "safety_verdict": "pass"
    -- }

    applied_by      UUID REFERENCES users(id),
    applied_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transforms_campaign ON transformations(campaign_id);
CREATE INDEX idx_transforms_repo ON transformations(repository_id);
CREATE INDEX idx_transforms_file ON transformations(source_file_id);
CREATE INDEX idx_transforms_status ON transformations(status);
CREATE INDEX idx_transforms_content ON transformations USING GIN (content jsonb_path_ops);
```

---

## Technical Debt Budgeting & Audit

```sql
-- ============================================================
-- DEBT BUDGETING
-- ============================================================

CREATE TABLE debt_budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    repository_id   UUID REFERENCES repositories(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    budget_minutes  INTEGER NOT NULL,
    spent_minutes   INTEGER NOT NULL DEFAULT 0,

    -- Budget detail and task assignments (JSONB)
    tasks           JSONB NOT NULL DEFAULT '[]',
    -- tasks example:
    -- [
    --   {
    --     "task_id": "uuid-1",
    --     "finding_id": "uuid-f1",
    --     "hotspot_id": "uuid-h1",
    --     "file_path": "src/payment/processor.ts",
    --     "assigned_to": "uuid-user1",
    --     "estimated_minutes": 120,
    --     "actual_minutes": null,
    --     "status": "in_progress",
    --     "assignment_reason": "Most commits (45) to this file in last 90 days"
    --   },
    --   {
    --     "task_id": "uuid-2",
    --     "finding_id": "uuid-f2",
    --     "assigned_to": "uuid-user2",
    --     "estimated_minutes": 60,
    --     "actual_minutes": 45,
    --     "status": "completed",
    --     "completed_at": "2026-05-08T16:00:00Z"
    --   }
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_budgets_org ON debt_budgets(organization_id);
CREATE INDEX idx_budgets_period ON debt_budgets(period_start, period_end);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,                    -- user or system
    action          VARCHAR(200) NOT NULL,   -- 'transformation.applied', 'finding.status_changed', etc.
    resource_type   VARCHAR(100) NOT NULL,   -- 'Transformation', 'Finding', 'Campaign', etc.
    resource_id     UUID NOT NULL,

    -- Action context (JSONB for flexibility)
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example (transformation applied):
    -- {
    --   "repository": "acme-frontend",
    --   "file_path": "src/payment/processor.ts",
    --   "recipe": "react-class-to-hooks",
    --   "pre_test_status": "pass",
    --   "post_test_status": "pass",
    --   "commit_sha": "abc123def",
    --   "pr_url": "https://github.com/acme/frontend/pull/456",
    --   "ssdf_practice": "PW.7",
    --   "risk_score": 35.5,
    --   "debt_reduced_minutes": 120
    -- }

    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_time ON audit_log(occurred_at);
CREATE INDEX idx_audit_context ON audit_log USING GIN (context jsonb_path_ops);
```

---

## Quality Gates & Trend Data

```sql
-- ============================================================
-- QUALITY GATES
-- ============================================================

CREATE TABLE quality_gate_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,

    -- Gate conditions (JSONB instead of separate conditions table)
    conditions      JSONB NOT NULL DEFAULT '[]',
    -- conditions example:
    -- [
    --   {"metric": "new_critical", "operator": "GT", "threshold": 0, "label": "No new critical issues"},
    --   {"metric": "new_major", "operator": "GT", "threshold": 5, "label": "Max 5 new major issues"},
    --   {"metric": "coverage_delta", "operator": "LT", "threshold": -2.0, "label": "Coverage drop < 2%"},
    --   {"metric": "debt_ratio", "operator": "GT", "threshold": 5.0, "label": "Debt ratio < 5%"}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- TREND SNAPSHOTS (for dashboard charts)
-- ============================================================

CREATE TABLE trend_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    snapshot_type   VARCHAR(20) NOT NULL DEFAULT 'daily',  -- 'daily', 'weekly', 'monthly'

    -- All trend data in a single JSONB column
    data            JSONB NOT NULL,
    -- data example:
    -- {
    --   "debt": {
    --     "total_minutes": 5000,
    --     "by_characteristic": {"maintainability": 3200, "reliability": 800, "security": 600, "performance": 400},
    --     "sqale_rating": "C",
    --     "delta_from_previous": -200
    --   },
    --   "findings": {
    --     "open": 78, "resolved_today": 5, "new_today": 3,
    --     "by_severity": {"critical": 2, "major": 18, "minor": 42, "info": 16}
    --   },
    --   "hotspots": {
    --     "count": 12, "top_3": [
    --       {"file": "src/payment/processor.ts", "score": 92.5},
    --       {"file": "src/auth/middleware.ts", "score": 85.1},
    --       {"file": "src/api/routes.ts", "score": 78.3}
    --     ]
    --   },
    --   "transformations": {
    --     "applied_today": 2, "total_applied": 45, "debt_reduced_minutes": 120
    --   },
    --   "coverage": 72.5,
    --   "loc": 65000
    -- }

    UNIQUE (repository_id, snapshot_date, snapshot_type)
);

CREATE INDEX idx_trends_repo_date ON trend_snapshots(repository_id, snapshot_date);
```

---

## Example Queries

### JSONB containment query: Find all TypeScript files using React hooks

```sql
SELECT id, file_path, metrics->>'cognitive_complexity' AS complexity
FROM source_files
WHERE repository_id = :repo_id
  AND language = 'typescript'
  AND metrics @> '{"framework_patterns": {"react_hooks": []}}'
ORDER BY (metrics->>'cognitive_complexity')::NUMERIC DESC;
```

### Find hotspot files with high coupling to a specific file

```sql
SELECT
    sf.file_path,
    sf.hotspot_score,
    coupling->>'coupling_score' AS coupling_score
FROM source_files sf,
     jsonb_array_elements(sf.vcs_stats->'co_changed_files') AS coupling
WHERE sf.repository_id = :repo_id
  AND coupling->>'path' = 'src/payment/processor.ts'
  AND (coupling->>'coupling_score')::NUMERIC > 0.5
ORDER BY sf.hotspot_score DESC;
```

### SARIF export query: Generate SARIF-compatible results

```sql
SELECT jsonb_build_object(
    'ruleId', f.rule_key,
    'level', CASE f.severity
        WHEN 'critical' THEN 'error'
        WHEN 'major' THEN 'warning'
        WHEN 'minor' THEN 'note'
        ELSE 'none'
    END,
    'message', jsonb_build_object('text', f.detail->>'message'),
    'locations', jsonb_build_array(
        jsonb_build_object('physicalLocation', f.location)
    )
) AS sarif_result
FROM findings f
WHERE f.analysis_id = :analysis_id
  AND f.status = 'open';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Users | 3 | Orgs, users, memberships (settings/profile in JSONB) |
| Repositories & Files | 2 | Repos with language metadata in JSONB; files with metrics + VCS stats in JSONB |
| Analysis & Findings | 2 | Runs with summary in JSONB; findings with location + detail in JSONB |
| Recipes & Campaigns | 3 | Recipes with definition JSONB; campaigns with config + progress JSONB; transformations with content + test_results JSONB |
| Debt Budgeting | 1 | Budget with tasks array in JSONB |
| Audit & Compliance | 1 | Audit log with JSONB context |
| Quality Gates & Trends | 2 | Profiles with conditions JSONB; trend snapshots as pure JSONB |
| **Total** | **14** | Compared to ~33 in the normalized model |

---

## Key Design Decisions

1. **Core query fields as typed columns; variable data as JSONB.** The `source_files` table demonstrates this split: `complexity_score`, `churn_score`, `hotspot_score`, and `debt_minutes` are typed columns (fast aggregation, indexing, ORDER BY), while language-specific metrics, VCS history, and framework patterns live in JSONB. This lets `SELECT ... ORDER BY hotspot_score DESC` use a standard B-tree index while language-specific queries use GIN indexes.

2. **Finding `location` field mirrors SARIF's `physicalLocation` schema.** By storing location as JSONB that matches the SARIF structure (`region.startLine`, `region.startColumn`, `context_region.snippet`), SARIF export becomes a direct serialization rather than a mapping transformation. This reduces SARIF export bugs and maintenance.

3. **Campaign progress tracked as JSONB rather than derived from JOINs.** In a normalized model, campaign progress requires joining `refactoring_campaigns` to `transformations` and aggregating. Here, the `progress` JSONB field is updated atomically when transformation statuses change, making dashboard queries O(1) reads.

4. **Debt budget tasks stored as a JSONB array rather than a separate table.** A budget typically has 5-20 tasks per sprint. Storing them as a JSONB array eliminates a table and simplifies the API (one GET for the full budget with tasks). For organizations with hundreds of tasks per budget, this could be promoted to a separate table.

5. **Quality gate conditions as JSONB rather than a separate table.** A quality gate profile typically has 3-8 conditions. Storing them as a JSONB array keeps the model simpler and makes profile CRUD a single-row operation. The conditions schema is validated at the application layer using JSON Schema.

6. **Trend snapshots as a single JSONB `data` column per day.** Dashboard charts need pre-computed daily snapshots of debt, findings, hotspots, and coverage. Storing all trend data in one JSONB column means adding a new metric to the trend dashboard requires only a code change (populate the new field), not a schema migration.

7. **VCS stats embedded in `source_files` rather than separate commit/churn tables.** For the hybrid model, detailed commit history is ingested from git and processed in a pipeline, but only the aggregated results (monthly churn, top contributors, coupling scores) are stored in the database. Raw git data lives in the VCS provider and is re-ingested as needed.

8. **GIN indexes on JSONB columns for containment queries.** The `jsonb_path_ops` GIN index class supports `@>` (contains) and `@?` (jsonpath) operators, enabling efficient queries like "find all files that use React hooks" without extracting JSONB fields into typed columns.

9. **14 tables vs. 33 in the normalized model.** The reduction comes from collapsing junction tables, reference tables, and rarely-queried detail tables into JSONB fields on their parent entities. The trade-off is explicit: some queries that would be simple JOINs in the normalized model require JSONB operators here.

10. **Audit log as a lightweight event stream.** While not a full event-sourcing system (Data Model Suggestion 2), the `audit_log` table captures all significant actions with JSONB context. This provides NIST SSDF compliance and basic temporal querying without the complexity of full event sourcing.
