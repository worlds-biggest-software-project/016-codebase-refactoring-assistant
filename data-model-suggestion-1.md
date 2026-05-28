# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Codebase Refactoring Assistant · Created: 2026-05-11

## Philosophy

This model follows classical normalized relational design where every domain concept gets its own table with explicit foreign key relationships. The schema is designed for maximum data integrity and complex cross-entity analytical queries -- answering questions like "which repositories have the most unfixed critical findings in files with the highest churn, broken down by CWE category and assigned developer?"

The approach mirrors how mature static analysis platforms like SonarQube structure their data internally: separate tables for projects, components (files), issues, metrics, and quality profiles, with junction tables for many-to-many relationships. It aligns closely with the SQALE quality model's hierarchical structure (characteristics > sub-characteristics > requirements) and ISO 5055's four quality dimensions.

This is the most "traditional" schema and will be the most familiar to teams with SQL expertise. It trades flexibility (adding a new field requires a migration) for query power, referential integrity, and the ability to enforce business rules at the database level.

**Best for:** Teams prioritizing data integrity, complex analytical reporting, and long-term maintainability over rapid schema evolution.

**Trade-offs:**
- (+) Strong referential integrity -- impossible to have orphaned findings or metrics
- (+) Complex analytical queries are natural with JOINs across normalized tables
- (+) Well-suited for compliance environments requiring provable data consistency
- (+) Familiar to most backend engineers; extensive PostgreSQL tooling
- (-) Higher table count (~40-50 tables) increases migration complexity
- (-) Adding language-specific or framework-specific fields requires schema changes
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Multi-language/polyglot analysis generates wide variation in file-level attributes that normalized tables handle awkwardly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 25010:2023 (SQuaRE) | Quality characteristics (maintainability, reliability, security, etc.) modeled as an enum/reference table; every finding and metric maps to a characteristic |
| ISO 5055 (CISQ) | Four quality dimensions (Security, Reliability, Performance Efficiency, Maintainability) stored as top-level categories; CWE weakness mappings per finding |
| SQALE Method | Three-level hierarchy (characteristic > sub-characteristic > requirement) modeled as three related reference tables; remediation cost stored per finding |
| CWE (MITRE) | Hierarchical weakness taxonomy stored as a self-referential table with parent_cwe_id; each finding references a CWE ID |
| SARIF 2.1.0 | Finding structure aligns with SARIF result/location/rule model; export functions map directly to SARIF JSON schema |
| OWASP Top 10:2025 | OWASP category stored as a reference table; findings tagged with applicable OWASP category |
| NIST SP 800-218 (SSDF) | Transformation audit records map to PW (Produce Well-Secured Software) and RV (Respond to Vulnerabilities) practices |

---

## Multi-Tenancy & Organization

```sql
-- ============================================================
-- ORGANIZATION & TENANCY
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- 'free', 'team', 'enterprise'
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL,  -- 'github', 'gitlab', 'google', 'saml'
    auth_provider_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'owner', 'admin', 'member', 'viewer'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
```

---

## Repository & Source Code Management

```sql
-- ============================================================
-- REPOSITORIES & SOURCE FILES
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    vcs_provider    VARCHAR(50) NOT NULL,  -- 'github', 'gitlab', 'bitbucket', 'azure_devops'
    vcs_url         TEXT NOT NULL,
    default_branch  VARCHAR(255) NOT NULL DEFAULT 'main',
    primary_language VARCHAR(50),
    languages       TEXT[],                -- array of detected languages
    loc_total       INTEGER,               -- lines of code
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_analyzed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_repositories_org ON repositories(organization_id);
CREATE INDEX idx_repositories_active ON repositories(organization_id, is_active);

CREATE TABLE source_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,          -- e.g., 'src/payment/processor.ts'
    language        VARCHAR(50),            -- detected language
    loc             INTEGER,                -- lines of code
    complexity_score NUMERIC(10,2),         -- cyclomatic or cognitive complexity
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, file_path)
);

CREATE INDEX idx_source_files_repo ON source_files(repository_id);
CREATE INDEX idx_source_files_language ON source_files(language);
CREATE INDEX idx_source_files_complexity ON source_files(repository_id, complexity_score DESC NULLS LAST);

-- Tracks branches and their analysis state
CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    head_commit_sha VARCHAR(40),
    last_analyzed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, name)
);

CREATE INDEX idx_branches_repo ON branches(repository_id);
```

---

## VCS History & Behavioral Analysis

```sql
-- ============================================================
-- VCS HISTORY (for behavioral hotspot detection)
-- ============================================================

CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    sha             VARCHAR(40) NOT NULL,
    author_email    VARCHAR(255) NOT NULL,
    author_name     VARCHAR(255),
    authored_at     TIMESTAMPTZ NOT NULL,
    message         TEXT,
    lines_added     INTEGER,
    lines_deleted   INTEGER,
    files_changed   INTEGER,
    UNIQUE (repository_id, sha)
);

CREATE INDEX idx_commits_repo_date ON commits(repository_id, authored_at DESC);
CREATE INDEX idx_commits_author ON commits(repository_id, author_email);

-- Per-file changes within each commit
CREATE TABLE commit_file_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    change_type     VARCHAR(20) NOT NULL,  -- 'added', 'modified', 'deleted', 'renamed'
    lines_added     INTEGER NOT NULL DEFAULT 0,
    lines_deleted   INTEGER NOT NULL DEFAULT 0,
    old_path        TEXT                    -- for renames
);

CREATE INDEX idx_commit_files_commit ON commit_file_changes(commit_id);
CREATE INDEX idx_commit_files_file ON commit_file_changes(source_file_id);

-- Pre-computed churn metrics per file per time window
CREATE TABLE file_churn_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    window_start    DATE NOT NULL,
    window_end      DATE NOT NULL,
    window_type     VARCHAR(20) NOT NULL,  -- 'weekly', 'monthly', 'quarterly'
    commit_count    INTEGER NOT NULL DEFAULT 0,
    unique_authors  INTEGER NOT NULL DEFAULT 0,
    lines_added     INTEGER NOT NULL DEFAULT 0,
    lines_deleted   INTEGER NOT NULL DEFAULT 0,
    churn_score     NUMERIC(10,2),          -- normalized churn metric
    UNIQUE (source_file_id, window_start, window_type)
);

CREATE INDEX idx_churn_file ON file_churn_metrics(source_file_id);
CREATE INDEX idx_churn_window ON file_churn_metrics(window_start, window_type);

-- Temporal coupling: files that change together
CREATE TABLE file_coupling (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    file_a_id       UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    file_b_id       UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    co_change_count INTEGER NOT NULL,       -- number of commits both files changed in
    coupling_score  NUMERIC(5,4),           -- 0.0000 to 1.0000
    window_start    DATE NOT NULL,
    window_end      DATE NOT NULL,
    UNIQUE (file_a_id, file_b_id, window_start),
    CHECK (file_a_id < file_b_id)           -- prevent duplicates
);

CREATE INDEX idx_coupling_repo ON file_coupling(repository_id);
CREATE INDEX idx_coupling_score ON file_coupling(coupling_score DESC);
```

---

## Hotspot & Technical Debt Analysis

```sql
-- ============================================================
-- HOTSPOT DETECTION & TECHNICAL DEBT
-- ============================================================

-- Hotspot = high churn + high complexity
CREATE TABLE hotspots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    analysis_id     UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    churn_score     NUMERIC(10,2) NOT NULL,
    complexity_score NUMERIC(10,2) NOT NULL,
    hotspot_score   NUMERIC(10,2) NOT NULL,  -- composite score
    defect_probability NUMERIC(5,4),         -- predicted defect likelihood (0-1)
    priority_rank   INTEGER NOT NULL,        -- 1 = highest priority
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hotspots_repo ON hotspots(repository_id);
CREATE INDEX idx_hotspots_analysis ON hotspots(analysis_id);
CREATE INDEX idx_hotspots_score ON hotspots(hotspot_score DESC);

-- Quality characteristics reference (ISO 25010 / SQALE)
CREATE TABLE quality_characteristics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- 'maintainability', 'reliability', 'security', 'performance_efficiency'
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    iso_25010_ref   VARCHAR(100),                 -- ISO 25010 reference code
    iso_5055_ref    VARCHAR(100),                 -- ISO 5055 mapping
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE quality_sub_characteristics (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    characteristic_id       UUID NOT NULL REFERENCES quality_characteristics(id),
    code                    VARCHAR(50) NOT NULL UNIQUE,  -- 'modularity', 'reusability', 'analysability', etc.
    name                    VARCHAR(255) NOT NULL,
    description             TEXT,
    sort_order              INTEGER NOT NULL DEFAULT 0
);

-- Technical debt per file, computed per analysis run
CREATE TABLE technical_debt_scores (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_file_id          UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    analysis_id             UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    characteristic_id       UUID NOT NULL REFERENCES quality_characteristics(id),
    debt_minutes            INTEGER NOT NULL,       -- remediation cost in minutes (SQALE)
    severity                VARCHAR(20) NOT NULL,   -- 'critical', 'major', 'minor', 'info'
    issue_count             INTEGER NOT NULL DEFAULT 0,
    sqale_rating            CHAR(1),                -- 'A' through 'E'
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_file_id, analysis_id, characteristic_id)
);

CREATE INDEX idx_debt_scores_file ON technical_debt_scores(source_file_id);
CREATE INDEX idx_debt_scores_analysis ON technical_debt_scores(analysis_id);
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
    branch_id       UUID REFERENCES branches(id),
    triggered_by    UUID REFERENCES users(id),
    trigger_type    VARCHAR(50) NOT NULL,   -- 'manual', 'push', 'pull_request', 'scheduled'
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'running', 'completed', 'failed'
    commit_sha      VARCHAR(40),
    files_analyzed  INTEGER,
    findings_count  INTEGER,
    debt_total_minutes INTEGER,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analysis_runs_repo ON analysis_runs(repository_id);
CREATE INDEX idx_analysis_runs_status ON analysis_runs(status);
CREATE INDEX idx_analysis_runs_date ON analysis_runs(created_at DESC);

-- CWE weakness taxonomy (self-referential hierarchy)
CREATE TABLE cwe_weaknesses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cwe_id          INTEGER NOT NULL UNIQUE,        -- e.g., 79
    name            VARCHAR(500) NOT NULL,           -- e.g., 'Improper Neutralization of Input During Web Page Generation'
    description     TEXT,
    parent_cwe_id   INTEGER REFERENCES cwe_weaknesses(cwe_id),  -- hierarchy
    abstraction     VARCHAR(20),                     -- 'Class', 'Base', 'Variant', 'Compound'
    status          VARCHAR(20) NOT NULL DEFAULT 'Draft'
);

CREATE INDEX idx_cwe_parent ON cwe_weaknesses(parent_cwe_id);

-- OWASP Top 10 categories
CREATE TABLE owasp_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owasp_id        VARCHAR(10) NOT NULL UNIQUE,    -- e.g., 'A01:2025'
    name            VARCHAR(255) NOT NULL,
    version_year    INTEGER NOT NULL                 -- 2021, 2025
);

-- Detection rules (like SonarQube rules or Semgrep rules)
CREATE TABLE detection_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),  -- NULL = built-in rules
    rule_key        VARCHAR(255) NOT NULL,           -- e.g., 'ts:S1192' or custom key
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    language        VARCHAR(50),                     -- target language, NULL = language-agnostic
    severity        VARCHAR(20) NOT NULL,            -- 'blocker', 'critical', 'major', 'minor', 'info'
    rule_type       VARCHAR(50) NOT NULL,            -- 'bug', 'vulnerability', 'code_smell', 'security_hotspot'
    characteristic_id UUID REFERENCES quality_characteristics(id),
    sub_characteristic_id UUID REFERENCES quality_sub_characteristics(id),
    cwe_id          INTEGER REFERENCES cwe_weaknesses(cwe_id),
    owasp_id        VARCHAR(10) REFERENCES owasp_categories(owasp_id),
    remediation_minutes INTEGER,                     -- estimated fix time
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rules_org ON detection_rules(organization_id);
CREATE INDEX idx_rules_language ON detection_rules(language);
CREATE INDEX idx_rules_type ON detection_rules(rule_type);

-- Individual findings (issues detected in source code)
CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    source_file_id  UUID NOT NULL REFERENCES source_files(id) ON DELETE CASCADE,
    rule_id         UUID NOT NULL REFERENCES detection_rules(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',  -- 'open', 'confirmed', 'resolved', 'wont_fix', 'false_positive'
    severity        VARCHAR(20) NOT NULL,
    line_start      INTEGER,
    line_end        INTEGER,
    column_start    INTEGER,
    column_end      INTEGER,
    message         TEXT NOT NULL,                   -- human-readable description
    code_snippet    TEXT,                             -- affected code fragment
    remediation_minutes INTEGER,
    assigned_to     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_analysis ON findings(analysis_id);
CREATE INDEX idx_findings_file ON findings(source_file_id);
CREATE INDEX idx_findings_rule ON findings(rule_id);
CREATE INDEX idx_findings_status ON findings(status);
CREATE INDEX idx_findings_severity ON findings(severity);
CREATE INDEX idx_findings_assigned ON findings(assigned_to) WHERE assigned_to IS NOT NULL;
```

---

## Refactoring Recipes & Transformations

```sql
-- ============================================================
-- REFACTORING RECIPES & EXECUTION
-- ============================================================

-- Recipe = a reusable refactoring pattern (like OpenRewrite recipes)
CREATE TABLE refactoring_recipes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),  -- NULL = built-in recipes
    name            VARCHAR(500) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    recipe_type     VARCHAR(50) NOT NULL,            -- 'framework_migration', 'dependency_upgrade', 'security_fix', 'code_quality', 'custom'
    source_language VARCHAR(50),
    target_language VARCHAR(50),                     -- for cross-language migrations
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    estimated_risk  VARCHAR(20),                     -- 'low', 'medium', 'high'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recipes_org ON refactoring_recipes(organization_id);
CREATE INDEX idx_recipes_type ON refactoring_recipes(recipe_type);

-- Rules that a recipe addresses
CREATE TABLE recipe_rule_mappings (
    recipe_id       UUID NOT NULL REFERENCES refactoring_recipes(id) ON DELETE CASCADE,
    rule_id         UUID NOT NULL REFERENCES detection_rules(id) ON DELETE CASCADE,
    PRIMARY KEY (recipe_id, rule_id)
);

-- Refactoring campaign = a batch application of recipes across repos
CREATE TABLE refactoring_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'planning', 'in_progress', 'paused', 'completed', 'cancelled'
    created_by      UUID NOT NULL REFERENCES users(id),
    target_repos    UUID[],                          -- array of repository IDs
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaigns_org ON refactoring_campaigns(organization_id);
CREATE INDEX idx_campaigns_status ON refactoring_campaigns(status);

-- Individual transformation (a recipe applied to a specific file)
CREATE TABLE transformations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID REFERENCES refactoring_campaigns(id) ON DELETE SET NULL,
    recipe_id       UUID NOT NULL REFERENCES refactoring_recipes(id),
    source_file_id  UUID NOT NULL REFERENCES source_files(id),
    finding_id      UUID REFERENCES findings(id),    -- the finding this addresses
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'generating', 'testing', 'ready_for_review', 'approved', 'applied', 'failed', 'rejected'
    diff_content    TEXT,                             -- unified diff of the transformation
    code_before     TEXT,                             -- original code snippet
    code_after      TEXT,                             -- transformed code snippet
    ai_explanation  TEXT,                             -- LLM explanation of why this refactoring is appropriate
    risk_score      NUMERIC(5,2),                    -- 0-100 risk assessment
    applied_by      UUID REFERENCES users(id),
    applied_at      TIMESTAMPTZ,
    pr_url          TEXT,                             -- link to generated PR
    pr_status       VARCHAR(50),                     -- 'open', 'merged', 'closed'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transformations_campaign ON transformations(campaign_id);
CREATE INDEX idx_transformations_recipe ON transformations(recipe_id);
CREATE INDEX idx_transformations_file ON transformations(source_file_id);
CREATE INDEX idx_transformations_status ON transformations(status);
```

---

## Test-Anchored Safe Execution

```sql
-- ============================================================
-- TEST-ANCHORED EXECUTION
-- ============================================================

-- Characterization tests generated before refactoring
CREATE TABLE characterization_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transformation_id UUID NOT NULL REFERENCES transformations(id) ON DELETE CASCADE,
    test_file_path  TEXT NOT NULL,
    test_content    TEXT NOT NULL,
    test_framework  VARCHAR(50),             -- 'jest', 'pytest', 'junit', 'go_test'
    generation_method VARCHAR(50) NOT NULL,  -- 'ai_generated', 'mutation_based', 'property_based'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_char_tests_transformation ON characterization_tests(transformation_id);

-- Test execution results (before and after refactoring)
CREATE TABLE test_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transformation_id UUID NOT NULL REFERENCES transformations(id) ON DELETE CASCADE,
    execution_phase VARCHAR(20) NOT NULL,    -- 'pre_refactor', 'post_refactor'
    total_tests     INTEGER NOT NULL,
    passed          INTEGER NOT NULL,
    failed          INTEGER NOT NULL,
    skipped         INTEGER NOT NULL DEFAULT 0,
    duration_ms     INTEGER,
    coverage_percent NUMERIC(5,2),
    is_passing      BOOLEAN NOT NULL,
    log_output      TEXT,
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_test_exec_transformation ON test_executions(transformation_id);
```

---

## Technical Debt Budgeting

```sql
-- ============================================================
-- TECHNICAL DEBT BUDGETING
-- ============================================================

-- Sprint/iteration debt budget allocation
CREATE TABLE debt_budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    repository_id   UUID REFERENCES repositories(id),  -- NULL = org-wide budget
    sprint_name     VARCHAR(255),
    budget_minutes  INTEGER NOT NULL,        -- allocated refactoring time in minutes
    spent_minutes   INTEGER NOT NULL DEFAULT 0,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_debt_budgets_org ON debt_budgets(organization_id);
CREATE INDEX idx_debt_budgets_period ON debt_budgets(period_start, period_end);

-- Task assignments for debt reduction
CREATE TABLE debt_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id       UUID NOT NULL REFERENCES debt_budgets(id) ON DELETE CASCADE,
    finding_id      UUID REFERENCES findings(id),
    hotspot_id      UUID REFERENCES hotspots(id),
    assigned_to     UUID REFERENCES users(id),
    estimated_minutes INTEGER NOT NULL,
    actual_minutes  INTEGER,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'completed', 'skipped'
    assignment_reason TEXT,                  -- why this developer was chosen (e.g., "most commits to this file")
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_debt_tasks_budget ON debt_tasks(budget_id);
CREATE INDEX idx_debt_tasks_assigned ON debt_tasks(assigned_to);
CREATE INDEX idx_debt_tasks_status ON debt_tasks(status);
```

---

## Quality Gates & PR Integration

```sql
-- ============================================================
-- QUALITY GATES & PR INTEGRATION
-- ============================================================

CREATE TABLE quality_gate_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quality_gate_conditions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id      UUID NOT NULL REFERENCES quality_gate_profiles(id) ON DELETE CASCADE,
    metric          VARCHAR(100) NOT NULL,   -- 'new_issues', 'new_critical', 'coverage_delta', 'debt_ratio'
    operator        VARCHAR(10) NOT NULL,    -- 'GT', 'LT', 'GTE', 'LTE', 'EQ'
    threshold       NUMERIC(10,2) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE quality_gate_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    profile_id      UUID NOT NULL REFERENCES quality_gate_profiles(id),
    overall_status  VARCHAR(20) NOT NULL,    -- 'passed', 'failed', 'warning'
    pr_number       INTEGER,
    pr_url          TEXT,
    pr_comment_id   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qg_results_analysis ON quality_gate_results(analysis_id);

-- Individual condition results for a quality gate evaluation
CREATE TABLE quality_gate_condition_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gate_result_id  UUID NOT NULL REFERENCES quality_gate_results(id) ON DELETE CASCADE,
    condition_id    UUID NOT NULL REFERENCES quality_gate_conditions(id),
    actual_value    NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL     -- 'passed', 'failed'
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Tenancy | 3 | Multi-tenant foundation with RBAC |
| Repository & Source Code | 3 | Repos, files, branches |
| VCS History & Behavior | 4 | Commits, file changes, churn metrics, coupling |
| Hotspots & Technical Debt | 4 | Hotspots, quality characteristics, sub-characteristics, debt scores |
| Analysis & Findings | 6 | Runs, CWE, OWASP, rules, findings |
| Refactoring & Execution | 5 | Recipes, mappings, campaigns, transformations |
| Test-Anchored Execution | 2 | Characterization tests, test executions |
| Debt Budgeting | 2 | Budgets, tasks |
| Quality Gates | 4 | Profiles, conditions, results, condition results |
| **Total** | **33** | |

---

## Key Design Decisions

1. **CWE weaknesses stored as a self-referential hierarchy** rather than a flat list, enabling queries like "all findings in the Injection category and its descendants" using recursive CTEs. This mirrors MITRE's actual CWE structure (View > Category > Weakness > Variant).

2. **Churn metrics pre-computed per time window** (`file_churn_metrics`) rather than calculated on the fly from `commits` and `commit_file_changes`. This trades storage for query performance -- hotspot detection queries millions of commit records; pre-aggregation makes the dashboard responsive.

3. **Findings separate from transformations** with a nullable `finding_id` on `transformations`. Not every transformation addresses a specific finding (some are proactive recipe applications), and not every finding gets a transformation (some are marked `wont_fix`).

4. **Remediation cost in minutes (SQALE model)** stored per finding and per rule. This enables the technical debt budget system to work in concrete time units rather than abstract scores, making it actionable for sprint planning.

5. **Quality characteristics as reference tables** aligned with ISO 25010 and ISO 5055 rather than hardcoded enums. This allows the system to support both ISO 25010:2023 (nine characteristics) and legacy ISO 9126 (six characteristics) simultaneously.

6. **Multi-tenant via `organization_id` foreign keys** using shared-schema pattern with application-level filtering. This is simpler than schema-per-tenant for a code analysis tool where cross-org queries are rare but org-scoped queries are constant.

7. **Test executions track pre- and post-refactor phases** separately. The `execution_phase` column enables a direct comparison of test results before and after a transformation -- the core safety guarantee of test-anchored execution.

8. **File coupling stored as undirected pairs** with a CHECK constraint (`file_a_id < file_b_id`) to prevent duplicate entries for the A-B / B-A relationship. The `coupling_score` (0-1) is the Jaccard similarity of co-change frequency.

9. **Campaign model separates planning from execution.** A `refactoring_campaign` groups transformations across repos and tracks overall progress, while individual `transformations` track file-level execution -- mirroring how Moderne orchestrates multi-repo recipe runs.

10. **SARIF export is a view, not a table.** The `findings` table structure (file, line/column, rule, message, severity) directly maps to SARIF's `result` + `location` + `reportingDescriptor` objects, so SARIF export is a query transformation rather than stored data.
