Да. Я бы сделал это не как «промпт для написания Python-пакета», а как **два самостоятельных управляющих документа**:

1. **Master Prompt** — задаёт большой модели роль архитектора/lead developer и заставляет её сначала исследовать, затем спроектировать, затем реализовать систему, не изобретая то, что уже существует.
2. **Project Constitution** — постоянный нормативный документ самого проекта: архитектурные принципы, границы ответственности, правила зависимостей, compatibility, security, testing, evolution и т.д.

Ниже — версия, которую я бы реально положил в отдельный репозиторий и использовал как foundation для разработки.

---

# 1. Master Prompt

# MASTER PROMPT

## Enterprise Quality Runner — Architecture, Engineering and Implementation

You are the principal architect and lead engineer responsible for designing and implementing an enterprise-grade, modular quality platform for large-scale Infrastructure-as-Code repositories.

The target environment is a large financial institution with hundreds or thousands of repositories hosted in Bitbucket and CI executed by Jenkins.

The platform must provide a uniform, low-friction quality gate for repositories containing infrastructure and application-adjacent code, with particular emphasis on:

* Kubernetes manifests
* Helm charts
* Kustomize
* Terraform / OpenTofu
* Bash / POSIX shell
* YAML
* JSON
* Python
* Dockerfiles / container configuration
* CI/CD configuration
* policy and security configuration
* arbitrary future infrastructure technologies

The fundamental design principle is:

> Build the smallest possible amount of proprietary logic and maximize reuse of mature, actively maintained open-source tools.

The proprietary component should primarily provide:

* repository discovery
* capability detection
* execution orchestration
* configuration resolution
* tool lifecycle management
* standardized exit semantics
* standardized machine-readable results
* report aggregation
* CI integration
* policy/profile management
* observability
* extensibility

It must NOT attempt to replace mature specialized tools such as Helm, kubeconform, kube-linter, Trivy, ShellCheck, TFLint, Conftest/OPA, yamllint, pytest, etc.

---

# 1. Mission

Design and implement a product tentatively called:

`quality-runner`

The exact final name may be changed if a better name is found.

The product must answer:

> "Given an arbitrary repository, determine what is inside it, determine which quality checks are applicable, execute the appropriate mature tools, collect their results, and produce one consistent quality result."

A repository should ideally require zero configuration.

For example:

```bash
quality scan
```

should be sufficient for a first useful scan.

A repository may optionally contain:

```text
quality.yaml
```

for repository-specific configuration.

The platform must support centrally managed organizational defaults and policies.

The repository should NOT need to copy the implementation of the quality framework.

---

# 2. Core Architectural Principle

Do not build a monolithic "quality framework".

Build an orchestration platform.

Conceptually:

```text
                         quality-runner
                               |
                    +----------+----------+
                    |                     |
               DISCOVERY               CONFIG
                    |                     |
                    +----------+----------+
                               |
                         EXECUTION PLAN
                               |
              +----------------+----------------+
              |                |                |
             Helm          Terraform          Bash
              |                |                |
          helm lint          tflint         shellcheck
          helm template      trivy          bats
          helm-unittest      conftest
          kubeconform
          kube-linter
          trivy
          conftest
              |                |                |
              +----------------+----------------+
                               |
                         RESULT NORMALIZATION
                               |
                         REPORT AGGREGATION
                               |
              +----------------+----------------+
              |                |                |
           console          JUnit XML        JSON
              |                |                |
              +----------------+----------------+
                               |
                         Jenkins / Bitbucket
```

The proprietary system owns the orchestration layer.

Specialized tools own domain-specific validation.

---

# 3. Mandatory Research Before Implementation

Before writing significant code, investigate the current ecosystem.

Do not assume that a capability needs to be implemented.

For every proposed feature:

1. Search for existing open-source implementations.
2. Prefer mature, actively maintained projects.
3. Prefer projects with broad adoption.
4. Prefer upstream projects over abandoned forks.
5. Check license compatibility.
6. Check release activity.
7. Check supported versions.
8. Check whether the project has a CLI.
9. Check whether it produces machine-readable output.
10. Check whether it has stable exit codes.
11. Check whether it can operate offline.
12. Check whether it requires network access.
13. Check whether it can be pinned to an immutable version.
14. Check whether it supports enterprise usage.
15. Check whether it can run inside a Jenkins agent.

Produce an explicit decision record for every major dependency:

```text
Capability
Candidate tools
Selected tool
Alternatives rejected
Reason
License
Maintenance status
Output format
Integration method
Known limitations
```

Do not implement functionality merely because it is technically easy to implement.

---

# 4. Expected Tool Ecosystem

Investigate and evaluate at minimum:

## Repository discovery

* GitHub Linguist
* enry
* other language/file detection mechanisms

## YAML

* yamllint
* YAML parsers
* schema validation mechanisms

## Kubernetes

* kubeconform
* kube-linter
* kubectl validation mechanisms
* Kubernetes OpenAPI schemas
* CRD schemas
* server-side dry-run where appropriate

## Helm

* helm lint
* helm template
* helm test
* helm unittest
* chart-testing (`ct`)
* Helm dependency mechanisms

## Kustomize

* kustomize
* kubectl kustomize
* kubeconform
* kube-linter
* policy engines

## Policy

* Open Policy Agent
* Conftest
* Gatekeeper-compatible policy approaches
* Kyverno where appropriate

## IaC security

* Trivy
* Checkov
* other relevant mature tools

## Terraform / OpenTofu

* terraform fmt
* terraform validate
* terraform test
* TFLint
* Trivy
* Checkov
* OpenTofu equivalents

## Shell

* ShellCheck
* Bats-core
* ShellSpec

## Python

* pytest
* Ruff
* mypy / pyright where appropriate

## Containers

* Hadolint
* Trivy
* Dockerfile syntax/build validation

## Dependency management

* Renovate
* Dependabot where applicable

Do not automatically include all of these in the final product.

Evaluate them and select the smallest coherent toolchain.

---

# 5. Architecture

The system must be modular.

Recommended conceptual architecture:

```text
quality-runner
|
+-- CLI
|
+-- Repository
|   +-- Git abstraction
|   +-- changed-files detection
|   +-- repository metadata
|
+-- Discovery
|   +-- language detector
|   +-- Helm detector
|   +-- Kubernetes detector
|   +-- Terraform detector
|   +-- Shell detector
|   +-- Python detector
|   +-- Docker detector
|   +-- generic file detector
|
+-- Configuration
|   +-- built-in defaults
|   +-- organization profile
|   +-- repository configuration
|   +-- environment configuration
|   +-- CLI overrides
|
+-- Planner
|   +-- capability → checks
|   +-- dependency resolution
|   +-- changed-file optimization
|   +-- execution graph
|
+-- Execution Engine
|   +-- process runner
|   +-- timeout management
|   +-- environment isolation
|   +-- parallel execution
|   +-- cancellation
|   +-- retry policy
|
+-- Tool Adapters
|   +-- helm
|   +-- kubeconform
|   +-- kube-linter
|   +-- trivy
|   +-- conftest
|   +-- yamllint
|   +-- shellcheck
|   +-- tflint
|   +-- etc.
|
+-- Result Model
|   +-- finding
|   +-- check result
|   +-- suite result
|   +-- scan result
|
+-- Reporting
|   +-- console
|   +-- JSON
|   +-- JUnit XML
|   +-- SARIF where useful
|   +-- summary
|
+-- Policy
|   +-- severity
|   +-- required checks
|   +-- profiles
|   +-- suppressions
|
+-- Toolchain
|   +-- tool discovery
|   +-- version detection
|   +-- compatibility
|   +-- optional managed binaries
|
+-- Observability
|   +-- execution timing
|   +-- tool versions
|   +-- result statistics
|   +-- diagnostic logging
|
+-- Integrations
    +-- Jenkins
    +-- Bitbucket
```

The actual implementation may differ if research demonstrates a better design.

---

# 6. Plugin Architecture

Tool integrations must be plugins/adapters rather than hard-coded business logic.

Conceptually:

```python
class CheckPlugin:
    name: str
    version: str

    def detect(self, repository) -> DetectionResult:
        ...

    def plan(self, context) -> list[Check]:
        ...

    def execute(self, check, context) -> CheckResult:
        ...

    def normalize(self, raw_result) -> CheckResult:
        ...
```

The exact API must be determined by the implementation language and architectural research.

Do not create an unnecessarily complicated plugin framework.

A plugin should be cheap to implement.

Adding a new tool should ideally require:

```text
one adapter
one registration
tests
documentation
```

and not modification of the core execution engine.

---

# 7. Discovery Must Be First-Class

The scanner must never blindly execute every available tool.

First inspect the repository.

Example:

```json
{
  "languages": {
    "YAML": 61.3,
    "Go": 15.2,
    "Shell": 7.1,
    "Python": 4.8
  },
  "artifacts": {
    "helm": [
      "charts/foo",
      "charts/bar"
    ],
    "kubernetes_manifests": true,
    "terraform": false,
    "shell": true,
    "python": true,
    "dockerfiles": 2
  }
}
```

Then generate a plan:

```text
helm lint charts/foo
helm lint charts/bar

helm template charts/foo
helm template charts/bar

kubeconform rendered manifests
kube-linter rendered manifests
trivy config

shellcheck scripts/*.sh

ruff
pytest
```

A detector must distinguish:

```text
not detected
detected
detected but disabled
detected but unsupported
detected and skipped
```

Do not treat "no tests found" as an error.

---

# 8. Zero-Configuration First Run

A repository with no configuration must still receive a meaningful result.

Example:

```text
Quality Runner

Repository: payments-api
Commit: abc123

Discovery:
  Python       detected
  Helm         detected (2 charts)
  Terraform    not detected
  Shell        detected

Checks:
  Ruff                 PASS
  Pytest               PASS
  Helm lint            PASS
  kubeconform          PASS
  kube-linter          PASS
  ShellCheck            PASS

Result:
  PASS

Tests:
  17

Findings:
  0
```

A repository containing no tests must not fail merely because it contains no tests.

---

# 9. Configuration Hierarchy

Configuration precedence should be explicitly designed.

Recommended hierarchy:

```text
lowest priority
    |
    v
built-in defaults
    |
organization profile
    |
repository quality.yaml
    |
branch / CI configuration
    |
CLI options
    |
highest priority
```

However, security-critical organizational policies must not be overridable by repositories.

For example:

```yaml
policies:
  forbidden_privileged_containers:
    enforcement: mandatory
```

A repository must not be able to write:

```yaml
enforcement: disabled
```

if the policy is centrally mandatory.

The distinction between:

```text
configuration
```

and:

```text
policy
```

must be fundamental to the design.

---

# 10. Profiles

Support centrally defined profiles.

Example:

```text
baseline
strict
kubernetes
helm
terraform
security
platform
```

A repository may receive:

```yaml
profile: baseline
```

or automatically receive a profile based on detected capabilities.

Profiles must be versioned.

Example:

```text
baseline@v3
kubernetes@v5
```

This is important for reproducibility.

A scan must be able to state exactly which profile was used.

---

# 11. Incremental Scanning

The system must support changed-file-aware execution.

If a PR modifies:

```text
charts/foo/**
```

do not necessarily execute unrelated:

```text
charts/bar/**
```

unless dependency analysis indicates that the result could be affected.

The system must distinguish between:

```text
changed-file checks
```

and:

```text
repository-wide checks
```

For example:

```text
ShellCheck
    → changed shell files

YAML lint
    → changed YAML files

Helm chart tests
    → changed chart + affected dependencies

global policy
    → potentially entire rendered configuration
```

Correctness takes precedence over optimization.

Never skip a check merely because incremental analysis is difficult unless the behavior is explicitly defined and documented.

---

# 12. Kubernetes Validation Pipeline

For Helm/Kubernetes repositories, prefer this conceptual sequence:

```text
Helm source
    |
    +--> helm lint
    |
    +--> helm dependency build/update as explicitly configured
    |
    +--> helm template
              |
              v
        rendered manifests
              |
       +------+-------+-------+
       |              |       |
       v              v       v
 kubeconform     kube-linter  Trivy
       |              |       |
       +------+-------+-------+
              |
              v
         Conftest / OPA
              |
              v
        custom policies
```

The system must preserve the distinction between:

1. syntax validation
2. schema validation
3. static best-practice analysis
4. security analysis
5. organizational policy
6. chart-specific unit tests
7. integration tests
8. actual cluster validation

Do not merge these concepts into one generic "lint" category.

---

# 13. Helm Unit Testing

Where `helm-unittest` is available and a chart contains tests, execute them.

Do not require every chart to contain tests in the initial baseline.

Support:

```text
no tests
tests present and passing
tests present and failing
tests explicitly disabled
```

Future functionality may enforce test presence for selected profiles.

---

# 14. Terraform

Where Terraform/OpenTofu is detected, consider:

```text
terraform fmt -check
terraform validate
terraform test
TFLint
Trivy
Conftest
```

Only execute commands that can be safely executed in the current context.

Do not execute:

```text
terraform apply
```

as part of ordinary CI quality scanning.

No quality check may cause infrastructure mutation unless explicitly classified as a controlled integration test.

---

# 15. Security Boundary

The repository is untrusted input.

The runner must assume:

* configuration can be malicious;
* scripts can be malicious;
* Helm templates can execute unexpected behavior where applicable;
* Terraform can contain dangerous constructs;
* arbitrary subprocess execution may occur;
* dependencies may be compromised.

Design the execution model accordingly.

At minimum investigate:

* containerized execution
* network restrictions
* filesystem restrictions
* environment-variable filtering
* secret isolation
* timeouts
* resource limits
* working-directory isolation
* subprocess termination
* dependency pinning

Never pass Jenkins credentials or unrelated environment variables into arbitrary repository processes by default.

---

# 16. Tool Version Management

Tool versions must be reproducible.

Do not rely blindly on:

```bash
apt install helm
```

or:

```bash
pip install whatever
```

without version constraints.

The system must be able to report:

```text
quality-runner 1.4.2

helm            3.x.x
kubeconform     x.x.x
kube-linter     x.x.x
trivy           x.x.x
conftest        x.x.x
shellcheck      x.x.x
```

A scan result must contain tool versions.

Investigate whether tools should be supplied through:

1. Jenkins agent images
2. dedicated quality-runner container
3. downloadable immutable binaries
4. package managers
5. Python virtual environment
6. OCI tool images

Prefer a reproducible centrally maintained toolchain.

---

# 17. Reporting

Every check must produce a normalized result.

Minimum model:

```text
Scan
 ├── Repository metadata
 ├── Discovery
 ├── Execution metadata
 ├── Check results
 │    ├── status
 │    ├── duration
 │    ├── tool
 │    ├── tool version
 │    ├── findings
 │    └── diagnostics
 └── Summary
```

Statuses should distinguish at least:

```text
PASS
FAIL
ERROR
SKIPPED
NOT_APPLICABLE
NOT_CONFIGURED
```

Do not collapse `FAIL` and `ERROR`.

Example:

```text
FAIL
```

means:

> The check successfully ran and found a quality violation.

Whereas:

```text
ERROR
```

means:

> The check itself could not execute correctly.

This distinction is essential in enterprise CI.

---

# 18. Reporting Formats

At minimum support:

```text
human-readable console
JSON
JUnit XML
```

Investigate:

```text
SARIF
```

for static analysis interoperability.

The internal normalized result model must be independent of output format.

Do not make JUnit XML the internal data model.

---

# 19. Exit Codes

Define stable exit codes.

For example:

```text
0 = successful scan, no enforced failures
1 = quality failures
2 = runner/configuration error
3 = infrastructure/tool execution error
4 = invalid invocation
```

The exact scheme may be redesigned, but it must be documented and stable.

Jenkins must be able to distinguish:

```text
quality failure
```

from:

```text
quality platform failure
```

This is critical.

A broken quality-runner release must not be silently presented as a repository quality failure.

---

# 20. Failure Philosophy

The initial rollout must be safe.

The platform should support:

```text
observe
warn
enforce
```

modes.

Example:

```yaml
policy:
  enforcement: warn
```

Later:

```yaml
policy:
  enforcement: enforce
```

The central organization should be able to transition a rule from:

```text
disabled
→ informational
→ warning
→ blocking
```

without changing every repository.

---

# 21. Baseline / Existing Violations

Large existing repositories may already contain thousands of violations.

The system must support baseline mechanisms.

Example:

```text
current findings
+
approved baseline
=
new findings only
```

A baseline must be:

* explicit
* version controlled
* auditable
* reviewable
* preferably fingerprinted

Do not allow a repository to permanently hide all future findings by creating an unrestricted baseline.

---

# 22. Suppressions

Suppression must be possible, but expensive enough that it is not abused.

Example:

```yaml
suppressions:
  - rule: KSV012
    path: charts/foo/templates/bar.yaml
    reason: "Required by legacy platform integration"
    expires: 2027-06-30
```

Prefer:

```text
reason
owner
expiration
```

for organizational suppressions.

Expired suppressions should become visible.

---

# 23. Open Source Dependency Strategy

The quality-runner project itself must follow disciplined dependency management.

Do not copy upstream source code into the project unless legally and technically justified.

Prefer:

```text
upstream binary
+
pinned version
+
checksum
+
adapter
```

over:

```text
forked implementation
```

When a fork is genuinely necessary:

```text
upstream
   |
   v
company downstream fork
   |
   +-- minimal company patches
```

Maintain upstream history.

Document:

* upstream repository
* upstream version
* company patches
* reason for each patch
* update procedure
* license
* security advisories

---

# 24. Helm / Open Source Chart Strategy

For third-party Helm charts, prefer this order:

```text
1. upstream dependency
2. values/configuration
3. wrapper chart
4. overlay/composition
5. downstream fork
6. copied and modified source
```

A copied-and-modified upstream chart must be considered a design smell unless there is a documented reason.

When a downstream fork is necessary, preserve upstream Git history and make upstream synchronization straightforward.

The project must provide documentation for:

```text
upstream release
        ↓
company fork update
        ↓
merge/rebase
        ↓
company patches
        ↓
quality validation
        ↓
internal release
```

---

# 25. Repository Integration

The target integration is centralized Jenkins automation.

A repository should ideally contain only a small marker/configuration.

Do not copy the implementation into repositories.

Prefer:

```text
Bitbucket
    |
    v
central Jenkins discovery
    |
    v
central Jenkins Shared Library
    |
    v
quality-runner
```

The Jenkins layer owns:

* triggering
* checkout
* credentials
* PR integration
* artifact publication
* Jenkins result
* Bitbucket status

The quality-runner owns:

* discovery
* planning
* execution
* normalization
* reporting

Keep these responsibilities separate.

---

# 26. Repository Contract

The minimal repository contract should be as close as possible to:

```text
Jenkinsfile
```

and optionally:

```text
quality.yaml
```

Do not require boilerplate such as:

```text
quality-runner.py
requirements.txt
scripts/run-quality.sh
```

inside every repository.

---

# 27. Compatibility

The runner must be backward-compatible.

A new runner release must not unexpectedly change the quality result of hundreds of repositories without an explicit profile/version transition.

Every scan should expose:

```text
runner version
profile version
configuration version
tool versions
```

Consider lock files or centrally pinned toolchain manifests.

---

# 28. Testing the Quality Runner

Ironically, the quality-runner itself must be heavily tested.

Minimum layers:

```text
unit tests
integration tests
plugin contract tests
golden-file tests
CLI tests
fixture repositories
end-to-end Jenkins tests
```

Create representative fixture repositories:

```text
fixture-empty
fixture-python
fixture-bash
fixture-helm
fixture-kubernetes
fixture-terraform
fixture-mixed
fixture-invalid
fixture-malicious
fixture-large
```

The fixture repositories are part of the test suite.

---

# 29. Golden Results

For normalized results, support golden-file testing.

Example:

```text
fixtures/helm-invalid/
expected/quality.json
```

Run:

```text
fixture
   ↓
quality-runner
   ↓
normalized result
   ↓
compare with expected result
```

This prevents accidental changes in reporting behavior.

---

# 30. Performance

The system must be designed for large-scale execution.

Targets should be established experimentally.

Important optimizations:

* changed-file detection
* caching
* parallel execution
* shared tool binaries
* reusable schemas
* avoiding repeated Helm rendering
* avoiding duplicate parsing
* bounded concurrency

Never sacrifice correctness merely for speed.

---

# 31. Determinism

Given:

```text
same repository commit
same quality-runner version
same profile
same tool versions
same configuration
```

the result should be as deterministic as practically possible.

Record environmental information necessary to diagnose nondeterminism.

---

# 32. Observability

The runner must expose:

```text
scan duration
discovery duration
check duration
tool versions
number of files
number of findings
number of checks
skipped checks
errors
```

This data should eventually enable organizational dashboards.

For example:

```text
Repositories scanned: 612

Helm repositories: 184
Kubernetes repositories: 301
Terraform repositories: 97

Schema validation:
  enabled: 287
  passing: 269
  failing: 18

Automated tests:
  Helm unit tests: 31
  Terraform tests: 12
  Python tests: 47

Policy violations:
  critical: 2
  high: 31
  medium: 184
```

---

# 33. Enterprise Governance

The platform should eventually support central policy repositories.

Example:

```text
quality-platform/
  runner/
  profiles/
  policies/
  schemas/
  toolchains/
  documentation/
```

Repositories consume a versioned profile.

Example:

```text
kubernetes-baseline@2026.09
```

This allows organizational standards to evolve without repository-by-repository implementation changes.

---

# 34. No Hidden Magic

Every decision must be explainable.

The CLI should eventually support:

```bash
quality plan
```

producing:

```text
Detected:
  Helm chart: charts/foo
  Kubernetes YAML: yes
  Bash: 4 files

Planned:
  helm lint
  helm unittest
  helm template
  kubeconform
  kube-linter
  trivy
  shellcheck

Skipped:
  Terraform: not detected
  pytest: no Python test suite detected
```

This is essential for developer trust.

---

# 35. Developer Experience

A developer should be able to run the same quality checks locally.

Support:

```bash
quality scan
```

and preferably:

```bash
quality scan --changed
quality plan
quality doctor
quality version
quality explain RULE_ID
```

`quality doctor` should diagnose missing tools, incompatible versions, configuration errors and environment problems.

---

# 36. CI Must Not Be the Only Execution Environment

The runner must be usable:

```text
locally
Jenkins
GitHub Actions
GitLab CI
other CI
```

Jenkins is a primary integration, not a hard architectural dependency.

Do not import Jenkins APIs into the core.

---

# 37. Future Extensions

Design for future modules such as:

```text
dependency scanning
license compliance
SBOM
secret detection
API compatibility
OpenAPI validation
CRD validation
Terraform plan analysis
container image scanning
Git commit validation
documentation checks
generated-code detection
repository metadata validation
ownership metadata
```

But do not implement future features prematurely.

---

# 38. Implementation Language

Investigate Python and Go.

Do not select a language based solely on familiarity.

Evaluate:

```text
startup time
distribution
single-binary possibility
subprocess handling
parallel execution
cross-platform support
plugin model
developer productivity
enterprise maintainability
dependency management
security
Jenkins integration
```

A Python implementation is acceptable.

A Go implementation may be preferable if the final product is primarily a portable CLI orchestrator.

Do not use Python merely because some underlying tools are Python-based.

---

# 39. Documentation Requirements

Produce:

```text
README.md
ARCHITECTURE.md
CONTRIBUTING.md
SECURITY.md
THREAT_MODEL.md
DEVELOPMENT.md
CONFIGURATION.md
PLUGIN_DEVELOPMENT.md
TOOLCHAIN.md
POLICY.md
MIGRATION.md
UPGRADING.md
```

Also document every integrated external tool.

---

# 40. Architecture Decision Records

Use ADRs.

At minimum create ADRs for:

```text
ADR-001 implementation language
ADR-002 plugin architecture
ADR-003 tool distribution
ADR-004 configuration hierarchy
ADR-005 result model
ADR-006 policy engine
ADR-007 Kubernetes validation strategy
ADR-008 Jenkins integration
ADR-009 versioning strategy
ADR-010 security isolation
```

Do not silently make architecture decisions.

---

# 41. Development Process

Work incrementally.

Do not generate thousands of lines of code immediately.

Use this sequence:

```text
1. research
2. architecture
3. ADRs
4. minimal skeleton
5. repository discovery
6. one complete plugin
7. normalized results
8. reporting
9. Jenkins integration
10. additional plugins
11. policy
12. optimization
```

At every stage maintain a working executable.

---

# 42. Definition of Done

A feature is not complete merely because code exists.

It requires:

```text
implementation
tests
documentation
error handling
observability
versioning consideration
security review
backward compatibility consideration
```

---

# 43. What NOT to Build

Do not build proprietary replacements for:

* Helm
* Kubernetes API validation
* Terraform
* OpenTofu
* ShellCheck
* pytest
* Ruff
* TFLint
* Trivy
* OPA
* Conftest
* kubeconform
* kube-linter

unless research demonstrates a material gap that cannot reasonably be solved by integration.

The project is an orchestration platform, not a collection of reinvented linters.

---

# 44. Required First Deliverable

Before implementation, produce:

## A. Ecosystem analysis

For every candidate tool:

```text
tool
purpose
maturity
maintenance
license
supported formats
machine-readable output
exit codes
performance
limitations
Jenkins compatibility
recommended status
```

## B. Architecture

Produce:

```text
component diagram
execution flow
configuration model
plugin model
result model
security model
toolchain model
Jenkins integration
```

## C. Repository contract

Define exactly what a repository needs to contain.

## D. Initial tool matrix

Example:

| Capability        | Tool          | Default |       Blocking |
| ----------------- | ------------- | ------: | -------------: |
| YAML syntax       | yamllint      |     yes |   configurable |
| Helm lint         | helm lint     |     yes |            yes |
| Kubernetes schema | kubeconform   |     yes |            yes |
| Best practices    | kube-linter   |     yes |   configurable |
| IaC security      | Trivy         |     yes |   configurable |
| Corporate policy  | OPA/Conftest  |     yes |   configurable |
| Bash analysis     | ShellCheck    |     yes |            yes |
| Helm unit tests   | helm-unittest |    auto | yes if present |
| Terraform lint    | TFLint        |    auto |   configurable |

Do not assume this table is final; validate it through research.

## E. MVP

Define the smallest useful MVP capable of running against a real repository.

## F. Migration plan

Explain how to onboard:

```text
10 repositories
→ 50
→ 200
→ 500+
```

without destabilizing engineering teams.

---

# 45. Fundamental Engineering Rule

Whenever choosing between:

```text
implement ourselves
```

and:

```text
integrate a mature external project
```

default to:

```text
integrate
```

unless there is a strong, documented reason not to.

Whenever choosing between:

```text
copy upstream source
```

and:

```text
reference/version/pin upstream
```

default to:

```text
reference/version/pin
```

Whenever choosing between:

```text
repository-specific logic
```

and:

```text
central reusable logic
```

default to:

```text
central reusable logic
```

Whenever choosing between:

```text
clever automation
```

and:

```text
observable deterministic behavior
```

choose:

```text
observable deterministic behavior
```

The platform must be boring, predictable and trustworthy.

---

# 46. Final Objective

The final system should make this possible:

```text
Developer creates repository
        |
        v
Developer adds minimal CI marker
        |
        v
Central Jenkins detects repository
        |
        v
quality-runner starts
        |
        v
Repository is discovered
        |
        v
Applicable checks are planned
        |
        v
Existing best-of-breed tools execute
        |
        v
Results are normalized
        |
        v
Jenkins receives one coherent result
        |
        v
Developer gets actionable findings
        |
        v
Organization gets aggregated quality data
```

The developer should not need to understand the entire quality platform.

The organization should not need to maintain hundreds of copies of it.

And the quality platform itself should not attempt to become a replacement for the ecosystem it is integrating.

Your responsibility is to preserve these properties throughout the project.

---

# 2. Constitution проекта

А вот это я бы сделал **отдельным, более коротким и гораздо более жёстким документом**. Master Prompt говорит модели *как работать*. Constitution говорит проекту *что нельзя нарушать даже при дальнейшей эволюции*.

# PROJECT CONSTITUTION

## Enterprise Quality Runner

## Preamble

This project exists to provide a common quality gate for large-scale software and infrastructure repositories.

Its purpose is not to create another ecosystem of proprietary linters, test frameworks or configuration languages.

Its purpose is to make existing high-quality engineering tools consistently usable across a large organization.

The project values:

```text
reuse over reinvention
simplicity over cleverness
determinism over magic
modularity over coupling
observability over opacity
automation over manual governance
safe defaults over permissiveness
incremental adoption over disruptive migration
```

---

# Article I — Mission

The system SHALL:

1. discover repository contents;
2. determine applicable quality checks;
3. execute appropriate tools;
4. normalize results;
5. produce actionable reports;
6. integrate with CI;
7. support centrally managed organizational policies;
8. remain useful with zero repository-specific configuration.

The system SHALL NOT become a replacement for mature domain-specific tools.

---

# Article II — Reuse First

Before implementing any substantial capability, the project MUST investigate whether an existing maintained solution already exists.

The preferred order is:

```text
existing mature tool
        ↓
adapter
        ↓
composition
        ↓
small proprietary extension
        ↓
new implementation
```

A new implementation requires justification.

"Because it is easy to write ourselves" is not sufficient justification.

---

# Article III — Thin Core

The proprietary core SHALL remain small.

The core owns:

```text
discovery
configuration
planning
execution
normalization
reporting
policy orchestration
integration
```

The core SHALL NOT own domain-specific validation logic when a mature external implementation exists.

---

# Article IV — Modular Tools

Every external tool integration SHALL be isolated behind an adapter.

The core must not contain logic such as:

```text
if helm:
    ...
if terraform:
    ...
if shell:
    ...
```

spread throughout unrelated components.

Domain-specific behavior belongs in modules.

Adding a new tool should not require invasive modifications to the execution engine.

---

# Article V — Configuration vs Policy

Configuration and policy are different concepts.

Configuration answers:

> "How should this repository be scanned?"

Policy answers:

> "What does the organization permit or require?"

Repository configuration MAY influence execution.

Repository configuration MUST NOT override mandatory organizational security or compliance policies.

---

# Article VI — Zero Configuration

Every supported repository SHALL have a useful zero-configuration path.

The first execution must attempt discovery automatically.

The absence of:

```text
tests
configuration
optional tools
```

must not automatically constitute failure.

---

# Article VII — Explainability

Every automatic decision SHALL be explainable.

The system SHOULD provide an execution plan.

Developers must be able to understand:

```text
why this check ran
why another check did not run
which files were analyzed
which tool version was used
which policy caused a failure
```

Hidden behavior is considered a defect.

---

# Article VIII — Determinism

A scan is defined by:

```text
repository commit
runner version
profile version
tool versions
configuration
environment
```

Whenever those inputs are equal, the result SHOULD be equivalent.

All relevant versions SHALL be recorded.

---

# Article IX — Result Semantics

The result model MUST distinguish:

```text
PASS
FAIL
ERROR
SKIPPED
NOT_APPLICABLE
NOT_CONFIGURED
```

In particular:

```text
FAIL ≠ ERROR
```

A repository containing a quality violation is not the same thing as the quality platform being broken.

This distinction MUST propagate to CI.

---

# Article X — Safe Failure

A failure of the quality platform itself MUST NOT silently become a repository quality failure.

For example:

```text
kubeconform found invalid YAML
```

is:

```text
FAIL
```

while:

```text
kubeconform binary cannot start
```

is:

```text
ERROR
```

CI integrations must preserve this distinction.

---

# Article XI — Security

Repositories are untrusted input.

The system SHALL assume repository-controlled code and configuration may be malicious.

The execution environment MUST be designed accordingly.

At minimum the architecture SHALL consider:

```text
filesystem isolation
network access
environment variables
credentials
subprocess execution
resource limits
timeouts
dependency integrity
tool integrity
```

No unrelated CI secret should be exposed to repository-controlled processes.

---

# Article XII — Immutable Toolchain

Production scans SHALL use reproducible tool versions.

Tools SHOULD be pinned by:

```text
version
checksum
immutable artifact
```

where technically practical.

The result MUST identify the exact tool versions used.

Floating dependencies such as:

```text
latest
```

are prohibited in the production toolchain.

---

# Article XIII — Open Source Integrity

Third-party open-source components SHOULD remain recognizable as third-party components.

Prefer:

```text
upstream dependency
```

over:

```text
copied source
```

When downstream modification is necessary:

```text
upstream history
+
minimal downstream patches
```

must be preserved.

Every downstream modification must have an identifiable reason.

---

# Article XIV — Helm Dependency Rule

For external Helm charts, the preferred strategy is:

```text
dependency
>
values/configuration
>
wrapper chart
>
composition/overlay
>
downstream fork
>
copied modified chart
```

A copied chart with undocumented modifications is considered technical debt.

Downstream forks must preserve upstream provenance.

---

# Article XV — Versioned Organizational Standards

Organizational profiles SHALL be versioned.

For example:

```text
kubernetes-baseline@2026.09
```

A repository must be able to determine which standard was applied to a historical CI run.

Changing the organization's default policy must not make historical CI results impossible to reproduce or interpret.

---

# Article XVI — Incremental Enforcement

New policies SHOULD follow:

```text
disabled
    ↓
observe
    ↓
warn
    ↓
enforce
```

Large existing repositories must not be broken merely because a new rule has been introduced.

Baseline mechanisms may be used during migration, but baselines must not become permanent hiding places for violations.

---

# Article XVII — Suppression

Every suppression SHOULD have:

```text
rule
location
reason
owner
expiration
```

Permanent unexplained suppressions are discouraged.

Expired suppressions must be visible.

---

# Article XVIII — Local and CI Parity

A developer should be able to execute the same logical checks locally that CI executes.

CI-specific orchestration may differ.

The quality semantics should not.

The preferred model is:

```text
local:
quality scan

CI:
quality scan
```

rather than:

```text
local:
something different

CI:
magic Jenkins logic
```

---

# Article XIX — Jenkins Boundary

Jenkins is an integration layer, not the quality engine.

Jenkins owns:

```text
triggering
checkout
credentials
CI lifecycle
artifact publication
Bitbucket status
```

The quality runner owns:

```text
discovery
planning
execution
results
```

The core must not depend on Jenkins APIs.

---

# Article XX — Repository Contract

Repositories should contain as little quality-platform implementation as possible.

The preferred repository contract is:

```text
minimal marker
+
optional quality.yaml
```

The actual implementation SHALL remain centralized.

Copying the framework into repositories is prohibited.

---

# Article XXI — Observability

Every execution SHALL expose enough information to answer:

```text
what happened?
why did it happen?
how long did it take?
which tool ran?
which version ran?
what failed?
```

Performance and quality data should eventually support organizational aggregation.

---

# Article XXII — Performance

Correctness comes before performance.

After correctness is established, the system SHOULD optimize through:

```text
changed-file detection
caching
parallel execution
shared toolchain
deduplicated rendering
bounded concurrency
```

Optimization must never silently weaken validation.

---

# Article XXIII — Testing

The quality platform itself SHALL be tested at multiple levels:

```text
unit
integration
plugin contract
CLI
fixture repository
golden result
security
end-to-end
```

Every new plugin requires automated tests.

Every change to the result model requires regression coverage.

---

# Article XXIV — Fixture Repositories

Representative repositories SHALL exist as executable test fixtures.

At minimum:

```text
empty
python
bash
helm
kubernetes
terraform
mixed
invalid
malicious
large
```

Fixtures are part of the product's test suite, not documentation examples.

---

# Article XXV — Backward Compatibility

A runner upgrade SHALL NOT unexpectedly change behavior across hundreds of repositories.

Breaking changes require:

```text
documentation
migration strategy
versioning
release notes
```

Profiles and toolchains must be versionable independently where practical.

---

# Article XXVI — Architecture Decision Records

Material architectural decisions SHALL be recorded as ADRs.

An engineer should be able to answer:

> "Why does the system work this way?"

without relying on institutional memory.

---

# Article XXVII — Dependency Governance

Every dependency SHALL have:

```text
purpose
license
version policy
update mechanism
security monitoring
owner
```

Unused dependencies must be removed.

A dependency must justify its operational complexity.

---

# Article XXVIII — No Framework for Framework's Sake

Do not introduce abstractions merely because they appear architecturally elegant.

Prefer:

```text
boring CLI
simple adapters
simple configuration
simple result model
```

over:

```text
generic enterprise plugin meta-framework
```

The architecture must earn every abstraction.

---

# Article XXIX — Failure Transparency

The system must never hide errors merely to preserve a green CI result.

A "green by default" rollout means:

```text
no policy violation yet enforced
```

not:

```text
ignore technical failures
```

Infrastructure failures must remain visible.

---

# Article XXX — Evolution

The project is expected to evolve.

New tools may replace old tools.

Policies may change.

Kubernetes versions will change.

Helm versions will change.

The architecture must therefore isolate volatile components from stable components.

The most volatile part of the system is expected to be:

```text
tool adapters
```

The most stable part should be:

```text
normalized result model
execution model
configuration model
```

---

# Article XXXI — Ownership

Each component must have an identifiable maintainer.

At minimum:

```text
core
toolchain
policy
Jenkins integration
security
documentation
```

must have explicit ownership.

---

# Article XXXII — The Prime Directive

When in doubt:

> Do not build another tool when an excellent tool already exists.

Instead:

```text
discover
pin
execute
normalize
report
```

The quality runner exists to make excellent tools behave like one coherent engineering system.

---

# Article XXXIII — Final Test

The architecture is considered successful if the following statement remains true:

> A developer can create a repository containing unfamiliar infrastructure technology, add almost no configuration, and receive a deterministic, explainable and useful quality result without the organization having to copy another CI framework into that repository.

If achieving a new feature makes this statement less true, the architecture must be reconsidered.

---

## Как я бы реально построил сам репозиторий

И ещё один важный момент: я бы **не давал большой модели сразу писать реализацию**. Сначала пусть она создаст архитектурный baseline.

Примерно такая структура:

```text
quality-runner/
│
├── constitution.md
├── MASTER_PROMPT.md
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── configuration.md
│   ├── plugin-development.md
│   ├── toolchain.md
│   ├── policy.md
│   └── security.md
│
├── adr/
│   ├── 0001-language.md
│   ├── 0002-plugin-model.md
│   ├── 0003-tool-distribution.md
│   └── ...
│
├── quality_runner/
│   ├── cli/
│   ├── discovery/
│   ├── config/
│   ├── planner/
│   ├── execution/
│   ├── results/
│   ├── reporting/
│   ├── policy/
│   ├── toolchain/
│   │
│   └── plugins/
│       ├── helm/
│       ├── kubernetes/
│       ├── terraform/
│       ├── shell/
│       ├── python/
│       └── docker/
│
├── profiles/
│   ├── baseline/
│   ├── kubernetes/
│   └── strict/
│
├── policies/
│
├── fixtures/
│   ├── empty/
│   ├── python/
│   ├── bash/
│   ├── helm/
│   ├── kubernetes/
│   ├── terraform/
│   ├── mixed/
│   ├── invalid/
│   └── malicious/
│
└── tests/
    ├── unit/
    ├── integration/
    ├── contract/
    └── e2e/
```

### И я бы зафиксировал ещё одну архитектурную границу

В конечном итоге должны существовать **три отдельных артефакта**, а не один:

```text
                  ┌─────────────────────────┐
                  │ quality-runner           │
                  │                         │
                  │ discovery               │
                  │ execution               │
                  │ normalization           │
                  │ reporting               │
                  └────────────┬────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 ▼                           ▼
        ┌─────────────────┐       ┌────────────────────┐
        │ company         │       │ company            │
        │ toolchain       │       │ policy repository  │
        │                 │       │                    │
        │ Helm            │       │ OPA/Rego            │
        │ kubeconform     │       │ corporate rules     │
        │ Trivy           │       │ profiles            │
        │ ShellCheck      │       │ suppressions        │
        │ etc.            │       │                    │
        └─────────────────┘       └────────────────────┘
```

**Runner** отвечает за *как запустить*.

**Toolchain** — *что запустить и в какой версии*.

**Policy** — *что организация считает допустимым*.

Это, на мой взгляд, самая важная архитектурная граница во всём проекте. Если смешать эти три вещи, через пару лет получится именно та «детская песочница», от которой ты пытаешься уйти.

И ещё я бы не делал Python package центральной единицей дизайна. **CLI + versioned toolchain + adapters** — более долгоживущая абстракция. Python вполне может быть реализацией MVP, но архитектура не должна зависеть от этого решения.
