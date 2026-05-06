# Standards & API Reference

> Project: Codebase Refactoring Assistant · Generated: 2026-05-04

## Industry Standards & Specifications

### Software Quality Standards

**ISO/IEC 25010:2023 — Product Quality Model**
- URL: <https://www.iso.org/standard/78176.html>
- The 2023 Edition 2 of SQuaRE's product quality model. Defines nine top-level characteristics (functional suitability, performance efficiency, compatibility, usability, reliability, security, maintainability, portability, and — new in 2023 — safety) each subdivided into subcharacteristics. Directly relevant as the canonical framework for defining and measuring the refactoring assistant's quality outcomes, particularly the _maintainability_ characteristic (modifiability, reusability, analysability, testability).
- Overview/reference: <https://iso25000.com/en/iso-25000-standards/iso-25010>

**ISO/IEC 25023:2016 — Measurement of System and Software Product Quality**
- URL: <https://www.iso.org/standard/35747.html>
- Companion to ISO/IEC 25010; defines concrete quality measures (metrics) for quantitatively evaluating the characteristics defined in 25010. Intended to be used alongside 25010 for quality assurance and improvement during or post development. Useful for defining measurable thresholds for technical-debt scoring and refactoring impact assessment.

**ISO/IEC 9126 (Legacy — withdrawn)**
- URL: <https://www.iso.org/standard/22749.html>
- The predecessor to ISO/IEC 25010, withdrawn in 2011. Defined six characteristics: functionality, reliability, usability, efficiency, maintainability, portability. Many existing tools and literature still reference 9126; the refactoring assistant should map its outputs to both 9126 (for legacy integrations) and 25010 (for current compliance).

**ISO/IEC/IEEE 42010:2022 — Architecture Description**
- URL: <https://www.iso.org/standard/74393.html>
- Defines requirements for expressing architecture descriptions of systems, software, and enterprises. Relevant when the refactoring assistant produces or consumes architectural views and viewpoints as part of large-scale structural refactors. The 2022 edition supersedes the 2011 version.
- IEEE mirror: <https://standards.ieee.org/ieee/42010/6846/>

---

### Language & Editor Protocol Standards

**Language Server Protocol (LSP) — Version 3.17 (current stable) / 3.18 (in development)**
- Official site: <https://microsoft.github.io/language-server-protocol/>
- Specification 3.17: <https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/>
- GitHub: <https://github.com/microsoft/language-server-protocol>
- The foundational open protocol — originally developed for VS Code, now a W3C-adjacent community standard supported by Microsoft, Red Hat, Codenvy, and dozens of editors — governing how editors and IDEs communicate with language servers. Defines JSON-RPC messages for: code completion, go-to-definition, find references, rename, formatting, and crucially **code actions** (the hook used by refactoring tools). Any refactoring assistant with editor integration must implement LSP's `textDocument/codeAction` and `workspace/applyEdit` methods.
- Microsoft Learn overview: <https://learn.microsoft.com/en-us/visualstudio/extensibility/language-server-protocol>

**Debug Adapter Protocol (DAP)**
- Official site: <https://microsoft.github.io/debug-adapter-protocol/>
- Specification: <https://microsoft.github.io/debug-adapter-protocol/specification.html>
- GitHub: <https://github.com/microsoft/debug-adapter-protocol>
- Companion to LSP; defines the JSON-based (not JSON-RPC) wire format between IDEs and debuggers. While primarily a debugging protocol, DAP is relevant for safe refactoring: the ability to set breakpoints and inspect execution state before/after a transformation validates that refactors are behaviour-preserving. Uses a JSON schema as its machine-readable spec.

---

### Code Analysis & Security Standards

**MISRA C:2023 / MISRA C++:2023**
- Overview: <https://ldra.com/misra/>
- Perforce overview: <https://www.perforce.com/resources/qac/misra-c-cpp>
- Coding guidelines for safety-critical C and C++ systems (automotive, aerospace, medical). MISRA C:2023 targets C90/C99/C11/C18; MISRA C++:2023 covers C++17. Each rule is classified Mandatory, Required, or Advisory, and is enforceable through static analysis. A refactoring assistant targeting embedded/safety domains must understand MISRA rules so it does not introduce non-compliant constructs during transformation.

**SEI CERT Coding Standards (C, C++, Java, others)**
- Main hub: <https://wiki.sei.cmu.edu/confluence/display/seccode/SEI+CERT+Coding+Standards>
- CERT C confluence: <https://wiki.sei.cmu.edu/confluence/display/c>
- GitHub pages: <https://cmu-sei.github.io/secure-coding-standards/>
- Maintained by Carnegie Mellon's Software Engineering Institute. Provides rules and recommendations for C, C++, Java, Android, and Perl to eliminate root causes of buffer overflows, format-string bugs, integer overflow, etc. The 2016 edition covers C11. Directly relevant for ensuring refactoring transformations do not introduce security weaknesses.

**CWE — Common Weakness Enumeration (MITRE)**
- Main site: <https://cwe.mitre.org/>
- CWE Top 25 (2025): <https://cwe.mitre.org/top25/archive/2025/2025_cwe_top25.html>
- Sponsored by CISA/DHS, maintained by MITRE. A community-developed formal list of over 600 software weakness categories (buffer overflows, path traversal, race conditions, XSS, hard-coded credentials, etc.), used as: a common language for describing weaknesses; a measuring stick for SAST tools; and a baseline for identification, mitigation, and prevention. Every SAST tool — including a refactoring assistant with security analysis — must map findings to CWE IDs.

**OWASP Top 10:2025**
- Main site: <https://owasp.org/www-project-top-ten/>
- 2025 edition: <https://owasp.org/Top10/2025/en/>
- The OWASP Top 10 is the de-facto awareness standard for web application security risks, updated to 2025 with two new categories (Software Supply Chain Failures; Mishandling of Exceptional Conditions) and one consolidation (SSRF merged into Broken Access Control). Referenced by ISO 27001 Annex A.8, SOC 2 CC8.1, and PCI DSS Requirement 6. A refactoring assistant with security remediation features should detect and fix OWASP Top 10 vulnerabilities.

**NIST SP 800-218 — Secure Software Development Framework (SSDF) v1.1**
- NIST CSRC page: <https://csrc.nist.gov/pubs/sp/800/218/final>
- SSDF project: <https://csrc.nist.gov/projects/ssdf>
- PDF: <https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-218.pdf>
- CISA resource: <https://www.cisa.gov/resources-tools/resources/nist-sp-800-218-secure-software-development-framework-v11-recommendations-mitigating-risk-software>
- Defines four practice groups: Prepare the Organisation (PO), Protect the Software (PS), Produce Well-Secured Software (PW), and Respond to Vulnerabilities (RV). Outcome-oriented rather than prescriptive; widely adopted for US federal procurement compliance. A refactoring assistant contributes to PW and RV practices.

**NIST SP 800-218A — Secure Software Development Practices for Generative AI (2024)**
- NIST CSRC page: <https://csrc.nist.gov/pubs/sp/800/218/a/final>
- PDF: <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-218A.pdf>
- Finalized July 2024. Augments SP 800-218 with practices specific to AI model development throughout the SDLC — directly applicable when the refactoring assistant uses LLMs to suggest or execute transformations. Provides the only current government guidance on AI-assisted code development security.

---

### AST & Program Transformation Standards

**Tree-sitter — Incremental Parsing Library**
- Official docs: <https://tree-sitter.github.io/tree-sitter/>
- Using Parsers guide: <https://tree-sitter.github.io/tree-sitter/using-parsers/>
- GitHub org: <https://github.com/tree-sitter>
- Main repo: <https://github.com/tree-sitter/tree-sitter>
- De-facto standard for language-agnostic incremental parsing in editor tooling (used by Neovim, GitHub, Zed, and many LSP servers). Grammars are written in JavaScript (.g4-style) and compiled to efficient C parsers. Produces Concrete Syntax Trees (CSTs), not ASTs; the CST retains all whitespace/comment tokens, making it suitable for format-preserving refactoring. Official bindings: C, Rust, Python, JavaScript (Node.js + Wasm), Java, Go, Swift, C#, Kotlin, Haskell, Zig. Supports 305+ language grammars via the language-pack.

**ANTLR 4 — ANother Tool for Language Recognition**
- Official site: <https://www.antlr.org/>
- GitHub: <https://github.com/antlr/antlr4>
- A widely used parser generator that accepts Extended Backus–Naur Form (EBNF) grammars (.g4 files) and generates parsers in Java, C#, Python, JavaScript, Go, Swift, and others. Produces parse trees which can be walked or transformed to produce ASTs. Used as the grammar backbone for many static analysis and refactoring tools; grammars for hundreds of languages are published in the antlr/grammars-v4 repository.

**Eclipse JDT AST (Java DOM/AST)**
- API reference: <https://help.eclipse.org/latest/topic/org.eclipse.jdt.doc.isv/reference/api/org/eclipse/jdt/core/dom/>
- Article: <https://www.eclipse.org/articles/Article-JavaCodeManipulation_AST/>
- Vogella tutorial: <https://www.vogella.com/tutorials/EclipseJDT/article.html>
- The Java-specific AST API within Eclipse JDT Core. Principal classes are `AST`, `ASTNode`, and `ASTParser`. The API fully covers the Java Language Specification through JLS25 (Java SE 25). Used by OpenRewrite, Checkstyle, SpotBugs, and virtually every Java refactoring tool. The dominant AST representation for Java program transformation; provides resolved type binding information for semantic refactors.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- Swagger (OAI): <https://swagger.io/specification/>
- Spec 3.1.0: <https://spec.openapis.org/oas/v3.1.0.html>
- Spec 3.2.0: <https://spec.openapis.org/oas/v3.2.0.html>
- OAI GitHub: <https://github.com/OAI/OpenAPI-Specification>
- The standard for describing RESTful APIs in YAML or JSON. Version 3.1 is a superset of JSON Schema Draft 2020-12, enabling complete JSON Schema support for request/response schemas. Any refactoring assistant exposing a REST API for external integrations should publish an OpenAPI 3.1+ document as its contract.

**JSON Schema — Draft 2020-12**
- Used via OpenAPI 3.1 and independently for configuration file validation. The canonical way to define and validate structured data formats such as the refactoring assistant's rule/recipe config files, finding report schemas, and transformation descriptors.
- JSON Schema spec: <https://json-schema.org/>

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749**
- IETF datatracker: <https://datatracker.ietf.org/doc/html/rfc6749>
- RFC Editor: <https://www.rfc-editor.org/rfc/rfc6749>
- oauth.net: <https://oauth.net/2/>
- The foundational authorization framework used by virtually every product in this space (SonarCloud, Sourcegraph, GitHub Copilot, CodeClimate). Defines roles, grant types, and token exchange. RFC 9700 supersedes earlier security guidance; prefer Authorization Code + PKCE for all client flows; avoid implicit and ROPC flows.

**OpenID Connect Core 1.0**
- Spec: <https://openid.net/specs/openid-connect-core-1_0.html>
- How it works: <https://openid.net/developers/how-connect-works/>
- Identity layer built on OAuth 2.0 that standardises how identity tokens (JWTs) are issued and verified. Used wherever the refactoring assistant needs to verify which developer is requesting a transformation or accessing findings.

**RFC 6750 — OAuth 2.0 Bearer Token**
- IETF: <https://www.ietf.org/rfc/rfc6750.txt>
- Defines the `Authorization: Bearer <token>` header scheme used by SonarQube, Sourcegraph, CodeClimate, and Semgrep APIs. The standard HTTP authentication pattern for the refactoring assistant's own API.

---

## Similar Products — Developer Documentation & APIs

### SonarQube / SonarCloud

- **Description:** SonarQube (self-hosted) and SonarCloud (SaaS) are the dominant code quality and technical-debt platforms. They analyse 30+ languages for bugs, code smells, duplications, coverage gaps, and security vulnerabilities, assigning a Technical Debt ratio and quality gate scores. The Web API exposes all analysis results programmatically.
- **API Documentation:**
  - SonarQube Server (latest): <https://docs.sonarsource.com/sonarqube-server/latest/extension-guide/web-api/>
  - SonarQube Server 2025.1 LTA: <https://docs.sonarsource.com/sonarqube-server/2025.1/extension-guide/web-api>
  - SonarCloud: <https://docs.sonarsource.com/sonarqube-cloud/appendices/web-api>
  - Live API explorer (SonarCloud): <https://sonarcloud.io/web_api>
  - Live API explorer (next SonarQube): <https://next.sonarqube.com/sonarqube/web_api>
- **SDKs/Libraries:** No official SDK; API is consumed directly via HTTP. Community clients exist in Python (`python-sonarqube-api`), Java (SonarQube Scanner), and JavaScript.
- **Developer Guide:** <https://docs.sonarsource.com/sonarqube-server/latest/extension-guide/web-api/>
- **Standards:** REST/JSON (Web API v1); Web API v2 in progress. No OpenAPI document published.
- **Authentication:** Bearer token via `Authorization: Bearer <token>` header. Tokens generated in the SonarQube/SonarCloud UI (user-scoped or project-scoped). Site-admin tokens available for CI/CD automation. Response includes `SonarQube-Authentication-Token-Expiration` header.
- **Plugin/Extension Model:** SonarQube supports custom plugins (Java) deployed into the `extensions/plugins/` directory. Plugin API covered in the extension guide. The SonarQube Marketplace lists certified plugins: <https://docs.sonarsource.com/sonarqube-server/latest/instance-administration/marketplace/>

---

### CodeClimate / Qlty

- **Description:** CodeClimate Quality (now rebranded as **Qlty**) provides automated code review for maintainability (complexity, duplication, style) and test coverage. Qlty is the modernised CLI and cloud successor, supporting 60+ linter plugins via a TOML-based plugin definition model instead of Docker images. The REST API exposes quality metrics, issues, and coverage data.
- **API Documentation:** <https://developer.codeclimate.com/>
- **SDKs/Libraries:** No official SDK. API is REST/JSON-API spec compliant. Qlty CLI (Rust-based, open-source): <https://github.com/qltysh/qlty>
- **Developer Guide:** <https://docs.codeclimate.com/> (legacy); <https://docs.qlty.sh/> (Qlty)
- **Standards:** REST, JSON API specification. SARIF output supported for findings interoperability.
- **Authentication:** API token. `Authorization: Token token={TOKEN}` header. Tokens generated in the user profile settings on codeclimate.com. Rate limit: 5,000 requests per token per hour.
- **Plugin/Extension Model:** Qlty uses TOML plugin definition files plus optional Rust output parsers for custom linters. Building a CodeClimate engine (Docker-based, legacy): <https://docs.codeclimate.com/docs/building-a-code-climate-engine>. Available plugin list: <https://docs.codeclimate.com/docs/list-of-engines>

---

### Sourcegraph (Batch Changes + Cody AI)

- **Description:** Sourcegraph is a code intelligence platform offering universal code search, batch large-scale automated code changes (Batch Changes), and Cody — an AI coding assistant with codebase-aware context. Batch Changes uses a declarative YAML spec and a GraphQL API to create, track, and manage pull requests across many repositories simultaneously.
- **API Documentation:**
  - GraphQL API docs: <https://sourcegraph.com/docs/api/graphql>
  - Batch Changes FAQ: <https://sourcegraph.com/docs/batch-changes/faq>
  - Batch Changes via GraphQL blog: <https://sourcegraph.com/blog/using-batch-changes-via-graphql-api>
  - Help Centre (creating/executing via API): <https://help.sourcegraph.com/hc/en-us/articles/26226121931917-Creating-and-Executing-Batch-Changes-via-the-GraphQL-API>
  - Cody docs: <https://sourcegraph.com/docs/cody>
- **SDKs/Libraries:** No official SDK. GraphQL consumed directly. Built-in API console at `https://<instance>/api/console`.
- **Developer Guide:** <https://sourcegraph.com/docs>
- **Standards:** GraphQL (primary API); authentication follows OAuth 2.0 / token patterns.
- **Authentication:** Access tokens (`Authorization: token YOUR_TOKEN`) or OAuth bearer tokens (`Authorization: Bearer YOUR_OAUTH_TOKEN`). Site admins can create sudo-scoped tokens. Service accounts recommended for CI/CD. Auth and authorisation docs: <https://sourcegraph.com/docs/admin/config/authorization-and-authentication>
- **Plugin/Extension Model:** Cody is available as VS Code and JetBrains extensions; VS Code Marketplace: <https://marketplace.visualstudio.com/items?itemName=sourcegraph.cody-ai>. Sourcegraph also has a GitHub integration and CLI (`src`).

---

### GitHub Copilot / Copilot Workspace

- **Description:** GitHub Copilot is an AI pair-programmer offering inline code completion, chat, and — via Copilot Workspace (GitHub Next) — a full agentic task-planning and multi-file refactoring mode. Copilot Workspace takes a natural-language task, plans changes across a codebase, writes code, and opens a pull request. Copilot SDK enables third-party integrations.
- **API Documentation:**
  - GitHub Copilot docs: <https://docs.github.com/en/copilot>
  - Features overview: <https://docs.github.com/en/copilot/get-started/features>
  - REST API for Copilot user management: <https://docs.github.com/en/rest/copilot/copilot-user-management>
  - Copilot Workspace (GitHub Next): <https://githubnext.com/projects/copilot-workspace/>
  - Copilot SDK GitHub: <https://github.com/github/copilot-sdk>
- **SDKs/Libraries:** Copilot SDK (multi-platform, Node.js/TypeScript primary): <https://github.com/github/copilot-sdk>. `@types/jscodeshift` for TypeScript codemods. GitHub CLI extension `gh copilot`.
- **Developer Guide:**
  - Authenticating with Copilot SDK: <https://docs.github.com/en/copilot/how-tos/copilot-sdk/authenticate-copilot-sdk/authenticate-copilot-sdk>
  - Using GitHub OAuth with Copilot SDK: <https://docs.github.com/en/copilot/how-tos/copilot-sdk/set-up-copilot-sdk/github-oauth>
- **Standards:** REST (GitHub REST API); OAuth 2.0 for SDK auth; GitHub Apps / OAuth Apps.
- **Authentication:** OAuth 2.0 via GitHub OAuth App or GitHub App. Users authorise the OAuth App; the resulting user access token (`gho_` or `ghu_` prefix) is passed to the SDK. Enterprise supports SAML SSO and EMU.
- **Plugin/Extension Model:** VS Code extension marketplace; JetBrains plugin; Copilot Extensions (GitHub Apps that extend the `@copilot` chat interface). Awesome Copilot repo: <https://github.com/github/awesome-copilot>

---

### Semgrep

- **Description:** Semgrep is an open-source, lightweight static analysis engine supporting 30+ languages. Rules are authored in YAML and use pattern-matching syntax that resembles the target source code. Semgrep AppSec Platform adds managed scanning, supply-chain analysis, secrets detection, and an API for findings, CI integration, and AI-assisted triage/remediation.
- **API Documentation:**
  - Semgrep API: <https://semgrep.dev/docs/semgrep-appsec-platform/semgrep-api>
  - Docs home: <https://semgrep.dev/docs/>
  - Rule syntax: <https://semgrep.dev/docs/writing-rules/rule-syntax>
  - Writing rules overview: <https://semgrep.dev/docs/writing-rules/overview>
- **SDKs/Libraries:** Python package on PyPI: <https://pypi.org/project/semgrep/>. GitHub: <https://github.com/semgrep/semgrep>. Community rules: <https://github.com/semgrep/semgrep-rules>
- **Developer Guide:** <https://semgrep.dev/docs/introduction>; Semgrep Code overview: <https://semgrep.dev/docs/semgrep-code/overview>
- **Standards:** YAML rule format (Semgrep schema); SARIF output for findings interchange; REST API for platform integration.
- **Authentication:** API token stored in `~/.semgrep/settings.yml`. Bearer token used for HTTP requests to the Semgrep AppSec Platform API. Rules can include HTTP validators using Bearer tokens for custom authentication checks.
- **Plugin/Extension Model:** Semgrep Editor (web IDE for rule authoring); VS Code extension; pre-commit hooks; CI/CD integrations (GitHub Actions, GitLab CI, Jenkins, CircleCI). Custom rules are YAML files conforming to the Semgrep rule schema.

---

### OpenRewrite

- **Description:** OpenRewrite is an open-source automated refactoring framework (Apache 2.0), maintained by Moderne. It uses a Lossless Semantic Tree (LST) representation and a recipe/visitor pattern to apply safe, large-scale transformations to Java, Kotlin, Groovy, XML, YAML, JSON, and more. Popular recipes include Spring Boot 2.x→3.x migration, JUnit 4→JUnit 5, Log4j→SLF4J, and Java version upgrades.
- **API Documentation:**
  - Docs home: <https://docs.openrewrite.org/>
  - Recipes concept: <https://docs.openrewrite.org/concepts-and-explanations/recipes>
  - Writing a Java refactoring recipe: <https://docs.openrewrite.org/authoring-recipes/writing-a-java-refactoring-recipe>
  - Popular recipe guides: <https://docs.openrewrite.org/popular-recipe-guides>
  - Maven plugin config: <https://docs.openrewrite.org/reference/rewrite-maven-plugin>
  - Gradle plugin config: <https://docs.openrewrite.org/reference/gradle-plugin-configuration>
- **SDKs/Libraries:** Maven plugin: `org.openrewrite.maven:rewrite-maven-plugin`. Gradle plugin: `org.openrewrite:plugin`. Core library: `org.openrewrite:rewrite-core`. GitHub: <https://github.com/openrewrite/rewrite>
- **Developer Guide:** Recipe development environment: <https://docs.openrewrite.org/authoring-recipes/recipe-development-environment>; Community recipes: <https://docs.openrewrite.org/reference/community-recipes>
- **Standards:** Recipes are Java classes extending `org.openrewrite.Recipe`; YAML recipes for declarative composition. No REST API exposed; invoked via Maven/Gradle plugins or Moderne platform.
- **Authentication:** Maven/Gradle credential configuration for accessing private recipe repositories (standard Maven repository authentication). Moderne SaaS platform uses token-based authentication for recipe execution at scale.
- **Plugin/Extension Model:** Recipes are the extension unit — each is a self-contained Java class with `getVisitor()` returning a `TreeVisitor`. Recipes are distributed as Maven/Gradle dependencies. The `@Option` annotation and `RuleDefinition` provide documentation and discoverability. Moderne marketplace lists published recipe modules.

---

### Facebook jscodeshift

- **Description:** jscodeshift is Facebook's JavaScript/TypeScript codemod toolkit. It runs transform scripts ("codemods") over multiple JS/TS files, using `recast` for AST-to-AST transformation while preserving the original code style. It provides a jQuery-like fluent API over `ast-types` for navigating and mutating the AST.
- **API Documentation:**
  - GitHub: <https://github.com/facebook/jscodeshift>
  - Wiki / docs: <https://github.com/facebook/jscodeshift/wiki/jscodeshift-Documentation>
  - Introduction (jscodeshift.com): <https://jscodeshift.com/overview/introduction>
  - npm: <https://www.npmjs.com/package/jscodeshift>
- **SDKs/Libraries:** `jscodeshift` npm package. TypeScript types: `@types/jscodeshift` (`npm i -D @types/jscodeshift`). Dependencies: `recast` (AST printer preserving style), `ast-types` (AST node definitions and traversal).
- **Developer Guide:** <https://github.com/facebook/jscodeshift/wiki>; custom transform tutorial: <https://www.skovy.dev/blog/jscodeshift-custom-transform>
- **Standards:** Based on ESTree / Mozilla AST standard (via `ast-types`). Codemods are CommonJS or ES modules exporting a `transform(fileInfo, api, options)` function.
- **Authentication:** N/A — local CLI tool; no network API.
- **Plugin/Extension Model:** Transforms are plain JS/TS files passed as the transform argument to the `jscodeshift` CLI runner. No formal plugin registry; community codemods are distributed via npm or GitHub.

---

### Rector (PHP)

- **Description:** Rector performs instant automated PHP upgrades and refactoring, supporting PHP 5.3 through 8.5 and major frameworks (Symfony, PHPUnit, Doctrine, Laravel). Each transformation is a "Rector rule" — a PHP class extending `AbstractRector` that inspects and rewrites PHP Parser AST nodes.
- **API Documentation:**
  - Docs home: <https://getrector.com/documentation>
  - Custom rule: <https://getrector.com/documentation/custom-rule>
  - Config configuration: <https://getrector.com/documentation/config-configuration>
  - Set Lists: <https://getrector.com/documentation/set-lists>
  - Rules overview: <https://getrector.com/documentation/rules-overview>
  - Writing tests for custom rules: <https://getrector.com/documentation/writing-tests-for-custom-rule>
- **SDKs/Libraries:** Composer package: `rector/rector`. GitHub: <https://github.com/rectorphp/rector>. Packagist: `rectorphp/rector`.
- **Developer Guide:** Integration into new project: <https://getrector.com/documentation/integration-to-new-project>; PHP version features: <https://getrector.com/documentation/php-version-features>
- **Standards:** Rules are PHP classes conforming to Rector's rule interface. Configuration via `rector.php` using `RectorConfig` fluent API (`withPaths()`, `withPhpSets()`, `withRules()`, `withSets()`). Uses nikic/PHP-Parser AST internally.
- **Authentication:** N/A — local CLI tool and Composer dependency; no network API.
- **Plugin/Extension Model:** Custom rules are PHP classes extending `Rector\Rector\AbstractRector`, implementing `getNodeTypes()`, `refactor()`, and `getRuleDefinition()`. Distributed as Composer packages. Rule sets (`SetList` constants) bundle related rules for batch application.

---

### Roslyn (.NET Compiler Platform)

- **Description:** Roslyn is Microsoft's open-source C# and Visual Basic compiler platform providing rich code analysis APIs. It exposes full syntax trees, semantic models (type resolution, data-flow analysis), and a workspace API for multi-project solutions. Third parties build Roslyn Analyzers (diagnostics) and Code Fix Providers (automated repairs) distributed as NuGet packages, and surfaced in Visual Studio / VS Code as light-bulb suggestions.
- **API Documentation:**
  - Microsoft Learn: <https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/>
  - Syntax analysis quickstart: <https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/get-started/syntax-analysis>
  - Compiler API model: <https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/compiler-api-model>
  - Roslyn analyzers overview (VS): <https://learn.microsoft.com/en-us/visualstudio/code-quality/roslyn-analyzers-overview>
  - Getting started (VS extensibility): <https://learn.microsoft.com/en-us/visualstudio/extensibility/getting-started-with-roslyn-analyzers>
- **SDKs/Libraries:** NuGet: `Microsoft.CodeAnalysis` (core); `Microsoft.CodeAnalysis.CSharp` (C# APIs); `Microsoft.CodeAnalysis.Workspaces.Common` (solution/project model). GitHub: <https://github.com/dotnet/roslyn>. API docs source: <https://github.com/dotnet/roslyn-api-docs>
- **Developer Guide:** Roslyn overview (GitHub wiki): <https://github.com/dotnet/roslyn/blob/main/docs/wiki/Roslyn-Overview.md>
- **Standards:** Analyzers must implement `DiagnosticAnalyzer`; code fixes must implement `CodeFixProvider`. Distributed in NuGet packages under `analyzers/dotnet/cs/`. IDE integration follows LSP for VS Code; native API for Visual Studio VSIX.
- **Authentication:** N/A — compiler SDK; no network API. NuGet feeds use standard NuGet authentication.
- **Plugin/Extension Model:** Roslyn Analyzers + Code Fix Providers are the plugin unit. Packaged as NuGet packages (`analyzers\dotnet\cs\` folder convention). Can also be embedded in the .NET SDK for automatic distribution. API authors can ship domain-specific analysis as part of their own NuGet libraries. Tools built on Roslyn include Roslynator, StyleCop, and many Microsoft platform analyzers.

---

### tree-sitter

- **Description:** tree-sitter is an incremental, error-tolerant parser generator and parsing library, now the dominant parsing substrate for editor tooling (Neovim, GitHub's code navigation, Zed, Helix). It generates efficient C parsers from JavaScript grammar definitions, producing Concrete Syntax Trees (CSTs) that are updated in place as source files are edited — making it ideal for refactoring tools that need to parse on every keystroke without a full reparse.
- **API Documentation:**
  - Docs home: <https://tree-sitter.github.io/tree-sitter/>
  - Using Parsers guide: <https://tree-sitter.github.io/tree-sitter/using-parsers/>
  - GitHub org: <https://github.com/tree-sitter>
  - Main repo: <https://github.com/tree-sitter/tree-sitter>
  - Rust crate: <https://docs.rs/tree-sitter>
- **SDKs/Libraries:** Official runtime bindings: Rust (`tree-sitter` crate), Python (`tree-sitter` pip package), JavaScript/Node.js, Wasm, Java (JDK 8+ and JDK 22+ variants), C#, Go, Swift, Kotlin, Haskell, Zig. Language-pack (305+ grammars): <https://github.com/kreuzberg-dev/tree-sitter-language-pack>; PyPI: <https://pypi.org/project/tree-sitter-language-pack/>
- **Developer Guide:** <https://tree-sitter.github.io/tree-sitter/>; incremental parsing explainer: <https://tomassetti.me/incremental-parsing-using-tree-sitter/>
- **Standards:** No formal interchange standard for CSTs; tree-sitter defines its own node type JSON schema per grammar. The C API (documented in `tree_sitter/api.h`) is the canonical interface. Grammars published in upstream org follow naming convention `tree-sitter-<language>`.
- **Authentication:** N/A — embedded library; no network API.
- **Plugin/Extension Model:** Grammars are npm packages (e.g., `tree-sitter-javascript`) with a compiled C parser and a `grammar.js`. Editors load grammars at runtime via the tree-sitter library binding. Custom queries (S-expression patterns) are used for syntax highlighting, code folding, and refactoring pattern matching.

---

## Notes

### Gaps and Evolving Areas

1. **No universal AST interchange format.** There is no standard wire format for exchanging abstract syntax trees between tools. Each ecosystem (tree-sitter CSTs, Eclipse JDT ASTs, Roslyn SyntaxTrees, ANTLR parse trees, OpenRewrite LSTs) uses its own representation. Emerging efforts like the OASIS Code Analysis Results Interchange Format (SARIF, ISO/IEC 5711-1 in progress) address *findings*, not *trees*.

2. **SARIF for findings interoperability.** SARIF (Static Analysis Results Interchange Format) — Microsoft/OASIS standard, GitHub Actions natively consumes it — is the practical interchange format for static analysis *results* (not yet an ISO standard but widely adopted). A refactoring assistant should emit SARIF for compatibility with GitHub Advanced Security, VS Code Problems pane, and Azure DevOps.
   - SARIF 2.1.0 spec: <https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html>
   - OASIS TC: <https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=sarif>

3. **AI-assisted refactoring not yet standardised.** As of 2026, there is no formal standard or specification governing how AI models should propose, validate, or apply code transformations. NIST SP 800-218A (2024) is the closest normative guidance but addresses AI model *development* security rather than AI-generated code quality. Emerging best practices (OWASP LLM Top 10, various enterprise governance frameworks) are still maturing.

4. **LSP 3.18 in development.** Version 3.18 of LSP is under active development at <https://github.com/microsoft/language-server-protocol/blob/gh-pages/_specifications/lsp/3.18/specification.md> and may introduce new refactoring-related capabilities before this project ships.

5. **ISO/IEC 25010:2023 adoption lag.** Many existing tools (SonarQube, CodeClimate) still map metrics to the 2011 edition of ISO/IEC 25010. The 2023 revision (adding "safety" as a ninth characteristic) has not yet been widely adopted by tool vendors as of 2026.

6. **OpenRewrite expanding beyond JVM.** OpenRewrite's language support is expanding beyond Java/Kotlin/Groovy to Python and other languages. The recipe API is Java-centric; a language-agnostic recipe specification does not yet exist.

7. **Rector lacks a published API/REST endpoint.** Rector is entirely a local CLI/library tool with no HTTP API. Integration into a refactoring assistant would require either subprocess invocation or implementing the rule pattern in PHP natively.

8. **jscodeshift maintenance status.** As of 2026, jscodeshift's primary maintainer activity has slowed. Alternatives such as ts-morph (TypeScript-first) and Codemod.com (managed platform) are gaining traction for TypeScript-heavy codebases. The refactoring assistant should evaluate both.
