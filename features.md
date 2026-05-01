# Codebase Refactoring Assistant — Feature & Functionality Survey

> Candidate #16 · Researched: 2026-04-22

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| SonarQube / SonarSource | Open source (Community) + Commercial | GNU LGPLv3 (Community); proprietary (Developer/Enterprise) | https://sonarqube.org |
| CodeScene | Commercial SaaS + on-prem | Proprietary | https://codescene.com |
| Moderne / OpenRewrite | Open source (OpenRewrite) + Commercial | Apache-2.0 (OpenRewrite); proprietary (Moderne) | https://moderne.io / https://openrewrite.org |
| Sourcegraph Cody | Open source (core) + Commercial | Apache-2.0 (core); proprietary (Enterprise) | https://sourcegraph.com/cody |
| GitHub Copilot Workspace | Commercial SaaS | Proprietary | https://github.com/features/copilot |
| Cursor | Commercial | Proprietary | https://cursor.com |
| JetBrains AI Assistant | Commercial (bundled with IDE) | Proprietary | https://jetbrains.com/ai |
| Qodo | Commercial SaaS | Proprietary | https://qodo.ai |
| IBM watsonx Code Assistant | Commercial | Proprietary | https://ibm.com/products/watsonx-code-assistant |
| Semgrep | Open source + Commercial | GPL-2.0 (OSS engine); proprietary (Code SaaS) | https://semgrep.dev |

---

## Feature Analysis by Solution

### SonarQube / SonarSource

**Core features**
- Static analysis across 35+ languages with 6,500+ rules detecting code smells, bugs, and security hotspots
- Technical debt quantification: time-to-fix estimates aggregated at file, component, and project level
- Quality Gates: configurable pass/fail thresholds (max debt ratio, minimum coverage, zero critical issues)
- Duplication detection: identifies copy-pasted code blocks across the codebase
- Trend dashboards: debt accumulation over time, new issues per release, leak period analysis
- SonarLint IDE integration: real-time feedback during development before CI
- Security-specific rules: OWASP Top 10, CWE, and SANS Top 25 mapping

**Differentiating features**
- Longest track record in the category (20+ years); most widely deployed technical debt measurement tool
- Quality Gate concept — the "definition of done" for code quality — is a well-understood pattern that engineering managers buy against

**UX patterns**
- Project dashboard with overall health score, issue counts by severity, and coverage percentage
- Issue list with filter/sort by type, severity, language, age, and assignee
- PR decoration: inline comment on the PR with issue count delta vs. base branch

**Integration points**
- GitHub, GitLab, Bitbucket, Azure DevOps PR decoration
- Jenkins, GitHub Actions, CircleCI CI plugins
- SonarLint for VS Code, IntelliJ, Eclipse, Visual Studio

**Known gaps**
- Rule-based only: cannot understand semantic intent or business logic; no cross-file reasoning
- Does not generate refactored code; identifies issues only
- Alert fatigue: large codebases generate thousands of issues with no prioritisation by ROI
- Community Edition lacks branch analysis and PR decoration (paid tiers required)

**Licence / IP notes**
- Community Edition: GNU LGPLv3. Dynamic linking in proprietary products is permitted; static linking requires legal review.

---

### CodeScene

**Core features**
- Behavioural code analysis: combines git VCS history (change frequency, author coupling) with static complexity metrics
- Hotspot identification: files with both high complexity and high churn — where refactoring yields the highest ROI
- Team coupling analysis: which files are always changed together, revealing hidden dependencies
- Code health trend: tracks complexity and coupling over time to show whether debt is increasing or decreasing
- Predictive defect analysis: hotspot files have statistically higher defect rates; prioritises them for review
- Architecture analysis: service dependency maps inferred from code coupling patterns

**Differentiating features**
- VCS-behavioural approach is unique: CodeScene is the only tool that identifies where developers actually struggle (high churn = friction), not just where the code is theoretically complex
- Predictive defect correlation: academic backing from multiple empirical studies linking code hotspots to defect density

**UX patterns**
- Interactive "city map" visualisation: code as a landscape with hotspot files as red, large buildings
- Trend sparklines per file show debt trajectory (improving, stable, degrading)
- Team analysis heatmaps showing author and temporal coupling across file clusters

**Integration points**
- GitHub, GitLab, Bitbucket, Azure DevOps VCS integration
- Jira for linking hotspot files to backlog tickets
- CI pipeline integration for quality gate enforcement

**Known gaps**
- Analysis only: does not generate refactored code or automated fixes
- EUR 18/author/month pricing becomes significant at 100+ contributor organisations
- Less actionable for greenfield codebases with short git history

**Licence / IP notes**
- Proprietary SaaS.

---

### Moderne / OpenRewrite

**Core features**
- OpenRewrite: open-source automated refactoring engine using a lossless semantic tree (LST) for safe code transformation
- Recipe catalog: 1,000+ pre-built recipes for framework migrations (Spring Boot 2→3, JUnit 4→5, Log4j→SLF4J), dependency upgrades, and security fixes
- Batch transformations: apply a recipe across 100M+ lines of code simultaneously without manual intervention
- Moderne SaaS: multi-repository orchestration, recipe management, and organisation-wide transformation visibility
- Custom recipe authoring: write recipes in Java or YAML to encode team-specific refactoring patterns
- Diff review workflow: all transformations produce a reviewable diff before merging

**Differentiating features**
- Scale of transformation is unique: Google's 2025 study reported 74.45% of code changes LLM-generated across 39 migrations; Airbnb migrated 3,500 files in 6 weeks. No other tool operates safely at this scale.
- LST (lossless semantic tree) preserves comments, formatting, and whitespace through transformations — the foundation for safe, automated bulk refactoring

**UX patterns**
- Recipe marketplace: browse and apply pre-built transformations from the Moderne UI
- Organisation-wide recipe run: apply a recipe across all repositories in the organisation and track progress in a single dashboard
- PR generation: Moderne opens pull requests with the transformed code for human review

**Integration points**
- GitHub, GitLab, Bitbucket for multi-repo PR generation
- Maven, Gradle build system integration
- GitHub Actions, Jenkins for CI-triggered recipe execution
- Moderne DX Edition for self-hosted enterprise deployment

**Known gaps**
- Java/JVM-centric: OSS recipe catalog is strongest for Java; Go, Python, Rust, TypeScript recipes are sparse
- Custom recipe authoring requires Java development skills
- Moderne commercial pricing requires sales engagement; no public pricing

**Licence / IP notes**
- OpenRewrite: Apache-2.0. Unrestricted commercial use. Moderne: proprietary.

---

### Semgrep

**Core features**
- Open-source static analysis and custom rule engine using a pattern-matching DSL
- Fast multi-language scanning: pattern matching against ASTs without parsing overhead
- `fix` field in rules: rules can include automated fix code, enabling auto-remediation alongside detection
- Supply-chain analysis (Semgrep Supply Chain): SCA alongside SAST in one tool
- Rule registry: 4,000+ community and Semgrep-authored rules covering security, performance, and code quality
- CI/CD integration with SARIF output for GitHub Advanced Security

**Differentiating features**
- `fix` patterns in rules enable automated code transformation, not just detection — comparable to a lightweight OpenRewrite recipe system for security patterns
- Custom rule authoring in a readable DSL (not Java) makes it accessible to security engineers without compiler background

**UX patterns**
- Web-based Semgrep Playground for testing rules against sample code without setup
- `semgrep --autofix` applies fix patterns automatically in CI
- VS Code extension with real-time rule evaluation

**Integration points**
- GitHub, GitLab, Bitbucket, Azure DevOps via CLI and native integrations
- SARIF output for GitHub Advanced Security
- Semgrep Code (managed cloud tier) for team rule management and PR decoration

**Known gaps**
- GPL-2.0 engine creates embedding friction for proprietary products
- Pattern-based only: cannot understand semantic intent or cross-file data flows without dataflow extensions
- Fix patterns are limited to local transformations; cannot orchestrate multi-file refactoring campaigns

**Licence / IP notes**
- GPL-2.0 (OSS engine). Products embedding or distributing the Semgrep engine must be GPL-2.0 compatible or obtain a commercial licence. Semgrep Code SaaS: proprietary.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Any viable solution in this space must provide:

- Static analysis across at least Python, TypeScript/JavaScript, Java, Go, and C# with rule-based issue detection
- Technical debt quantification with time-to-fix estimates and severity classification
- PR integration: inline issue annotations and quality gate enforcement on pull requests
- GitHub, GitLab, Bitbucket CI/CD integration
- Trend dashboards showing debt accumulation vs. reduction over time
- Duplicate code detection across the codebase
- Automated fix generation for at least a subset of detected issues

### Differentiating Features

Capabilities that provide competitive advantage:

- **VCS-behavioural hotspot analysis**: combining git churn history with complexity metrics to identify where refactoring yields highest ROI (CodeScene — no open-source equivalent)
- **Safe bulk code transformation**: applying refactoring recipes across hundreds of repositories simultaneously without manual review of each file (Moderne/OpenRewrite)
- **Test-anchored safe execution**: generating characterisation tests before refactoring, then validating the test suite after transformation to guarantee safety
- **Cross-repository polyglot transformation**: refactoring API contracts and their consumers across language boundaries in a single campaign
- **Semantic refactoring vs. syntactic rules**: AI-powered understanding of *why* a pattern is problematic in the specific business context of the calling code

### Underserved Areas / Opportunities

- **VCS-behavioural hotspot detection in open-source tooling**: CodeScene's approach is commercially validated but has no open-source equivalent. Combining git churn analysis with AI-powered transformation planning is unaddressed.
- **Polyglot monorepo transformation**: OpenRewrite is Java-excellent; no tool handles mixed-language codebases (Go + TypeScript + Python) with comparable recipe coverage.
- **Test-anchored autonomous refactoring**: AI generates characterisation tests, executes the transformation, and validates safety — no tool offers this end-to-end.
- **Continuous technical debt budgeting**: tools produce reports but none actively allocate a "refactoring budget" per sprint, assign tasks based on developer context familiarity, and track debt reduction against business KPIs.

### AI-Augmentation Candidates

- **Semantic refactoring from intent**: LLM understands *why* a pattern is problematic in the specific calling context — coupling, business logic violations, downstream impact — generating refactorings that are contextually appropriate rather than mechanically correct.
- **Hotspot prioritisation with LLM-generated transformation plan**: combining VCS churn data with LLM analysis to produce "this file is changed 40× per sprint; here is a concrete refactoring plan with risk assessment and test coverage requirements."
- **Cross-language polyglot transformation**: LLM understands idiomatic patterns across Go, Python, TypeScript, Rust, and Java simultaneously, enabling migration campaigns that cross language boundaries.
- **Test-anchored safe execution**: LLM generates characterisation tests before refactoring, executes the transformation, and validates the resulting test suite — providing the safety guarantee enterprises require for autonomous operation.

---

## Legal & IP Summary

Key licence considerations:

- **SonarQube Community (GNU LGPLv3)**: dynamic linking in proprietary products is permitted; static linking requires legal review.
- **OpenRewrite (Apache-2.0)**: unrestricted commercial use and embedding.
- **Semgrep (GPL-2.0)**: embedding the OSS engine in a proprietary distributed product requires GPL-2.0 compliance or a commercial licence from Semgrep. Interface via subprocess/CLI to avoid propagation.
- **Sourcegraph core (Apache-2.0)**: unrestricted commercial use.

Technical debt quantification methodologies (CISQ/ISO 5055, SQALE) are open standards. VCS-behavioural hotspot analysis is documented in academic literature (Adam Tornhill's published research) with no known active patents. LLM-based automated code refactoring is a generic AI application with no known specific patent encumbrances as of April 2026.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Static analysis across Python, TypeScript/JavaScript, Java, and Go with rule-based issue detection and severity classification
- Technical debt quantification with time-to-fix estimates and trend dashboards
- VCS-behavioural hotspot analysis: identify files with high churn + high complexity for ROI-prioritised refactoring
- Automated fix generation for detected issues with diff preview before applying
- PR integration with GitHub and GitLab for inline annotations and quality gate enforcement

**Should-have (v1.1)**
- AI-powered semantic refactoring: LLM generates contextually appropriate refactorings explaining *why* a pattern is problematic in the specific calling context
- Test-anchored execution: generate characterisation tests before applying refactoring, validate suite after
- Batch transformation campaign: apply a refactoring recipe across multiple files or repositories simultaneously
- Cross-file impact analysis: identify all call sites and usages affected by a proposed refactoring before execution

**Nice-to-have (backlog)**
- Polyglot monorepo transformation: cross-language refactoring campaigns spanning Go, TypeScript, Python, and Java
- Continuous technical debt budgeting: per-sprint refactoring allocation with developer assignment and KPI tracking
- Team coupling analysis: identify hidden dependencies from git co-change patterns
- NIST SSDF compliance reporting: audit trail demonstrating that automated transformations did not introduce security regressions
