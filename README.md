# Codebase Refactoring Assistant

AI-native refactoring platform combining behavioral code hotspot detection, LLM-driven semantic understanding, and safe autonomous execution — closing the gap between identifying technical debt and acting on it.

## The Problem

Technical debt is measurable and pervasive — yet remains unaddressed:

- **Technical debt consumes 21–40% of enterprise IT budgets** (Deloitte 2026)
- **Existing tools diagnose but don't fix** — SonarQube identifies code smells; developers manually refactor
- **Detection-to-remediation is manual and tedious** — the friction prevents action
- **No tool combines behavioral hotspot detection with safe autonomous execution** — CodeScene identifies where to refactor; OpenRewrite handles how; none bridge both

Current state: organizations accumulate tech debt dashboards while developers drown in SAST alerts they can't address.

## What This Does

### Behavioral Hotspot Detection
- **Combines git churn with complexity metrics** — identifies files developers actually struggle with
- **"High change frequency + high complexity" = ROI-prioritized refactoring target**
- **Predictive defect correlation** — hotspot files have statistically higher defect rates
- **CodeScene's approach (commercial)** brought to open source
- **No open-source alternative exists** — first tool to unify VCS-behavioral analysis with AI

### Semantic Refactoring (AI-Native)
- **LLM understands code intent** — not just pattern matching but context reasoning
- **Generates contextually appropriate refactorings** — explains *why* a pattern is problematic in this codebase
- **Cross-language polyglot transformations** — refactor API contracts and their consumers across Go, Python, TypeScript, Rust, Java simultaneously
- **Rule-based tools (Semgrep, SonarQube) cannot reason about context** — this is the AI gap

### Test-Anchored Safe Execution
- **Generate characterization tests *before* refactoring** — capture current behavior
- **Execute the transformation** with LLM-generated refactoring code
- **Validate test suite afterward** — guarantees safety without human code review
- **No current tool offers end-to-end autonomous refactoring** — all require manual verification

### Continuous Technical Debt Budgeting
- **Actively allocates a "refactoring budget" per sprint** — not passive reporting
- **Assigns specific tasks to developers** based on skill and code familiarity
- **Tracks debt reduction against business KPIs** — quantifies value of refactoring
- **No tool provides this orchestration** — current tools only dashboard

## Key Differentiators

| Feature | This Platform | SonarQube | CodeScene | Moderne/OpenRewrite | GitHub Copilot |
|---------|---|---|---|---|---|
| **VCS hotspot detection** | ✓ | — | ✓ (Commercial) | — | — |
| **Semantic refactoring** | ✓ | — | — | (Rule-based recipes) | ✓ |
| **Test-anchored execution** | ✓ | — | — | — | — |
| **Polyglot transformation** | ✓ | — | — | Java-focused | ✓ (Multi-lang) |
| **Autonomous remediation** | ✓ | — | — | ✓ (Recipe-based) | — |
| **Technical debt budgeting** | ✓ | Dashboards only | — | — | — |

## Market & Opportunity

- **Market size**: Generative AI coding assistants $26–92M (2024–2025) → $92–98M by 2030; broader AI developer tools market 17% CAGR
- **Technical debt TAM**: $5 trillion global IT spend × 21–40% debt = hundreds of billions annually
- **Buyers**: Platform/DevEx teams, engineering managers (legacy modernization), CTOs (regulated industry), security teams
- **Open-source gap**: CodeScene's VCS-behavioral approach has no open-source equivalent; no tool combines behavioral detection + AI execution

## Research Foundation

- **LLM-generated code changes account for 74.45% of code changes across 39 migrations at Google** (arXiv:2504.09691, 2025)
- **Airbnb migrated 3,500 test files in 6 weeks** vs. 1.5 years estimated manually (Airbnb Engineering 2025)
- **Semantic refactoring outperforms syntactic rules** — LLMs reason about intent; rule-based tools cannot (ICSE 2025 workshop)
- **Technical debt research spans ISO 25010, CISQ/ISO 5055, SQALE metrics** — well-established measurement frameworks

## Quick Start

```bash
# Identify hotspots from VCS history
refactor-assistant analyze --repo=/path/to/repo \
  --min-churn=10 \
  --min-complexity=high

# Generate refactoring plan
refactor-assistant plan --hotspot=src/payment/processor.ts \
  --reason="This file changed 40× per sprint, highest coupling"

# Test-anchored execution
refactor-assistant refactor --hotspot=src/payment/processor.ts \
  --generate-tests \
  --apply \
  --validate

# Open PR for review
refactor-assistant pr --create
```

## Target Users

1. **Platform/DevEx Teams** — org-wide code quality standards, reducing release friction
2. **Engineering Managers** — legacy modernization projects (Java → Spring Boot, Python 2 → 3)
3. **CTOs (Regulated Industry)** — must demonstrate NIST SSDF, OWASP, ISO 25010 compliance
4. **Security Engineers** — automate remediation of SAST findings (not just reporting)
5. **Technical Leads** — manage technical debt backlogs at scale

## Related Standards

- ISO/IEC 25010:2023 (SQuaRE) — software quality model with maintainability sub-characteristics
- ISO 5055 (CISQ) — automated measurement of software quality and maintainability
- NIST SP 800-218 (SSDF) — secure software development framework
- OWASP Code Review Guide — security-relevant code patterns
- IEEE 1062 — software acquisition and refactoring as lifecycle activities

---

Built on research from Google's 2025 LLM code migration study, Airbnb's test migration work, and academic literature on semantic refactoring and technical debt measurement. [Read the full research](./research.md) | [Feature roadmap](./features.md)
