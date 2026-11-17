<!--
SYNC IMPACT REPORT
==================
Version change: (none) → 1.0.0
Change type: Initial ratification (starter constitution)

Principles:
  I.    Trusted & Minimal Supply Chain              (required by org)
  II.   Local-First Development & Testing           (required by org)
  III.  Cloud-Native & Horizontally Scalable        (required by org)
  IV.   Disposability & Resilience                  (required by org)
  V.    Versioned, Contract-First APIs (OpenAPI)    (required by org)
  VI.   Feature Flags: Decouple Release from Enablement (required by org)
  VII.  Test Discipline                             (proposed)
  VIII. Vendor-Neutral Observability                (proposed)
  IX.   Secure by Default                           (proposed)
  X.    Backward-Compatible Change & Data Evolution (proposed)
  XI.   Reproducible Builds & Immutable Artifacts   (proposed)
  XII.  Simplicity & Convention over Invention      (proposed)

Sections:
  - Technology Baseline
  - Decision-Making & Consistency (precedence: MUST > SHOULD > consistency)
  - Development Workflow & Quality Gates
  - Governance

Placeholders to resolve before ratification:
  - [PROJECT_NAME]
  - [ARTIFACT_REPO]              internal Maven/Gradle repository (Artifactory/Nexus)
  - [CONTAINER_REGISTRY]         approved internal container registry
  - [JAVA_LTS]                   company-approved Java LTS version
  - [COVERAGE_THRESHOLD]         minimum coverage gate on new code
  - [FEATURE_FLAG_SYSTEM]        company feature-flag platform, if one exists
  - [DEPLOYMENT_STANDARDS]       link to deployment team's standards/framework
  - [SECURITY_OR_PLATFORM_OWNER], [GOVERNANCE_APPROVERS]
  - [RATIFICATION_DATE]

Templates requiring review after amendment:
  - .specify/templates/plan-template.md   (Constitution Check gates)
  - .specify/templates/spec-template.md   (add "Feature Flag" section — Principle VI)
  - .specify/templates/tasks-template.md  (flag creation, both-state tests, flag removal tasks)
-->

# [PROJECT_NAME] Constitution

This constitution defines the non-negotiable engineering principles for Java /
Spring-based services deployed to Kubernetes. It is intentionally
project-agnostic: individual specifications, plans, and tasks MUST conform to
it, and any deviation MUST be justified explicitly (see Governance).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**
are to be interpreted as described in RFC 2119.

---

## Core Principles

### I. Trusted & Minimal Supply Chain

All third-party and open-source artifacts MUST be obtained exclusively from
company-approved repositories. Nothing is pulled directly from the public
internet at build, test, or run time. The number of dependencies SHOULD be
kept to the minimum necessary.

**Approved sources**

- Gradle dependencies and plugins MUST resolve only through `[ARTIFACT_REPO]`.
  Repositories MUST be declared centrally in `settings.gradle(.kts)`
  (`dependencyResolutionManagement` with
  `RepositoriesMode.FAIL_ON_PROJECT_REPOS`, and `pluginManagement` for
  plugins). Build files MUST NOT declare public repositories (e.g.
  `mavenCentral()`, `gradlePluginPortal()`) directly.
- The Gradle wrapper `distributionUrl` MUST point to an approved internal
  location, not `services.gradle.org`.
- Container base images and Testcontainers images MUST come from
  `[CONTAINER_REGISTRY]` (e.g. via Testcontainers' image name substitution
  or registry-prefix configuration).

**Version control of dependencies**

- Dependency versions MUST be pinned. Dynamic versions (`+`,
  `latest.release`, ranges) MUST NOT be used. `SNAPSHOT` dependencies MUST NOT
  appear in a release build.
- Versions MUST be managed in a Gradle version catalog
  (`gradle/libs.versions.toml`). The Spring Boot BOM SHOULD be the source of
  truth for Spring-managed libraries; overrides MUST be documented with a
  reason.
- Gradle dependency locking SHOULD be enabled so that transitive versions are
  reproducible and changes are visible in review.

**Minimize dependencies (SHOULD)**

- New dependencies SHOULD be added only when they provide meaningful,
  differentiating value that the JDK, Spring Boot, or an existing dependency
  does not already provide. Avoid adding a library for trivial utility
  functions (string helpers, small collection utilities, simple HTTP or JSON
  wrappers already covered by Spring).
- Every new direct dependency MUST be justified in the implementation plan:
  what it does, why existing dependencies are insufficient, and its
  maintenance health (release cadence, CVE history).
- Unused dependencies SHOULD be removed as part of normal work, and
  dependency count SHOULD be reviewed periodically.

**Scanning**

- Every build MUST pass dependency vulnerability and license scanning, and
  SHOULD produce a Software Bill of Materials (SBOM, e.g. CycloneDX).

**Rationale:** Internal repositories give us provenance, license control,
availability when public registries are down, and protection against
dependency-confusion attacks. Every dependency, including its transitive
tree, is a recurring CVE-remediation and upgrade cost. Keeping the
dependency set small and deliberate lowers that cost and the attack surface.

### II. Local-First Development & Testing

A developer MUST be able to build, run, and unit-test the application on their
own workstation without access to a shared environment or a Kubernetes cluster.

- A single documented command (e.g. `./gradlew build` or `./gradlew test`)
  MUST compile the application and run all unit tests locally.
- A single documented command (e.g. `./gradlew bootRun` with a `local`
  Spring profile, optionally with a `compose.yaml`) MUST start the
  application locally.
- Unit tests MUST NOT depend on network access, shared databases, external
  services, wall-clock timing, or test execution order.
- External dependencies needed for local runs or integration tests (databases,
  brokers, caches) SHOULD be provided via Testcontainers or Docker Compose,
  using images from `[CONTAINER_REGISTRY]` (see Principle I).
- Local builds and the CI (TeamCity) build MUST use the same Gradle tasks, so
  that "green locally" reliably predicts "green in CI".
- The `README` MUST document prerequisites (JDK `[JAVA_LTS]`, container
  runtime, internal repository credentials setup) and the commands above. A
  new team member SHOULD be able to go from clone to running app in under 30
  minutes.
- Local configuration MUST NOT require real production credentials.

**Rationale:** Fast, reliable local feedback loops drive developer
productivity and quality more than anything else. If something only runs "in
the cluster", it can't be debugged effectively.

### III. Cloud-Native & Horizontally Scalable

Services MUST be designed to scale out by adding replicas, not by scaling up a
single instance. Services follow Twelve-Factor App practices.

- Application instances MUST be stateless. Session state, shared caches,
  files, and work queues MUST live in backing services (database, distributed
  cache, object storage, message broker), never in pod memory or local disk.
- The service MUST behave correctly with N ≥ 2 replicas running concurrently.
  Scheduled jobs and other singleton work MUST use a coordination mechanism
  (e.g. ShedLock or leader election) so they do not run more than once.
- Configuration MUST be externalized (environment variables, ConfigMaps,
  Spring profiles) and MUST NOT be baked into the image per environment. The
  same image MUST be deployable to every environment.
- Secrets MUST be supplied at runtime by the company-approved secret
  mechanism and MUST NOT be committed to source control or images.
- JVM memory sizing MUST be container-aware (e.g. `-XX:MaxRAMPercentage`)
  rather than hard-coded `-Xmx` values that ignore the pod limit. Resource
  requests/limits MUST be declared as required by `[DEPLOYMENT_STANDARDS]`.
- The service SHOULD expose a meaningful scaling signal (CPU, request rate,
  queue depth) usable for autoscaling.
- Logs MUST be written to stdout/stderr; the platform handles aggregation.

**Rationale:** Kubernetes treats pods as interchangeable units. Statelessness
and externalized configuration are what make autoscaling, rolling updates,
and multi-zone resilience possible.

### IV. Disposability & Resilience

Any pod MUST be able to be started, stopped, evicted, or rescheduled at any
moment without loss of availability or data.

- The application MUST handle `SIGTERM` gracefully: stop accepting new work,
  finish in-flight requests/messages within the termination grace period, and
  release resources (`server.shutdown=graceful` and an appropriate
  `spring.lifecycle.timeout-per-shutdown-phase`).
- Startup MUST be fast and predictable. Long-running initialization MUST NOT
  block readiness indefinitely.
- Services MUST expose distinct liveness and readiness endpoints (Spring Boot
  Actuator health groups) for the platform's probes. Liveness MUST NOT depend
  on downstream systems; readiness MAY reflect the availability of critical
  dependencies.
- Production deployments MUST meet the availability configuration required by
  `[DEPLOYMENT_STANDARDS]` (e.g. minimum replicas, disruption budgets, spread
  across nodes/zones). The application MUST NOT do anything that prevents
  those configurations from working.
- Message consumers and write endpoints MUST be idempotent or otherwise safe
  under redelivery and retry, since a pod may die mid-operation.
- All outbound calls MUST have explicit timeouts. Retries MUST use bounded
  exponential backoff with jitter. Circuit breakers / bulkheads (e.g.
  Resilience4j) SHOULD protect against cascading failure.
- The service MUST degrade gracefully when a non-critical dependency is
  unavailable rather than failing entirely.

**Rationale:** Evictions, node drains, autoscaling, and rolling deployments
are normal, frequent events in Kubernetes. Uptime depends on every pod being
cheap to lose.

### V. Versioned, Contract-First APIs (OpenAPI)

When a service exposes or consumes an HTTP API, that API MUST be documented
with an OpenAPI 3.x specification and MUST be explicitly versioned.

- **Versioning:** API versions MUST be expressed as a major version in the
  URI path (e.g. `/api/v1/orders`, `/api/v2/orders`). Every endpoint MUST
  belong to a version; unversioned endpoints MUST NOT be exposed (operational
  endpoints like Actuator health are excluded).
- Breaking changes MUST be introduced only in a new major version. Each
  OpenAPI document corresponds to one major version.
- When a new version is introduced, the previous version MUST remain available
  for a documented deprecation period. Deprecation MUST be marked in the spec
  (`deprecated: true`) and SHOULD be signalled to clients (e.g. `Deprecation`
  / `Sunset` headers).
- The OpenAPI document is the source of truth for the contract and MUST be
  versioned in the repository alongside the code.
- Contract-first (spec authored before implementation, with interfaces or
  stubs generated from it) is the RECOMMENDED approach. If code-first (e.g.
  springdoc) is used, the generated spec MUST be committed and verified in CI
  so that drift fails the build.
- Specifications MUST include request/response schemas, error responses
  (RFC 9457 Problem Details SHOULD be used), authentication requirements, and
  examples.
- Specs MUST pass linting (e.g. Spectral with the company ruleset) in CI.
  Breaking changes within an existing major version MUST be detected
  automatically in CI (e.g. openapi-diff) and fail the build.
- Asynchronous / event-driven interfaces SHOULD be documented with AsyncAPI
  using the same discipline.

**Rationale:** A machine-readable contract enables client generation,
contract testing, documentation, and safe evolution of APIs across teams.
Explicit path versioning makes the contract visible to every consumer and
lets old and new clients coexist during migration.

### VI. Feature Flags: Decouple Release from Enablement

Deploying code and turning on a feature MUST be separate decisions.

- Every new feature MUST be evaluated for a feature flag. The default is to
  flag it. Each spec MUST include a **Feature Flag** section that either
  defines the flag or states why the feature is not flagged (e.g. purely
  internal refactor, no behavior change, or a technical constraint).
- Every flag MUST be documented in a single place (`[FEATURE_FLAG_SYSTEM]` or,
  if none exists, a flag registry such as `docs/feature-flags.md`) with:
  name, description, owning team/person, default value per environment,
  related spec, date created, and removal criteria.
- New flags MUST default to **off** in production. Code MUST be safe to
  deploy with the flag in either state.
- Both flag states MUST be covered by automated tests.
- Flag evaluation MUST go through a single abstraction (the company flag
  platform's SDK or a small internal interface backed by Spring
  configuration). Flag names MUST NOT be scattered as string literals
  throughout the code.
- Flag state MUST be consistent across replicas and MUST be changeable
  without rebuilding the image. Flags SHOULD be changeable without a
  redeploy.
- Flags are temporary unless explicitly marked as long-lived operational
  toggles (kill switches). Release flags MUST have a removal task created when
  the feature is fully enabled, and SHOULD be removed within a defined period
  after that. Stale flags are tech debt and MUST be tracked.
- Feature flags MUST NOT be used as a substitute for authorization or access
  control.

**Rationale:** Separating release from enablement allows continuous,
low-risk deployment, gradual rollout, instant rollback without a redeploy,
and business-controlled launch timing. Documentation and removal discipline
keeps the flag set from turning into hidden, untested configuration.

### VII. Test Discipline

Every behavior defined in a specification MUST be verified by automated tests.

- Tests MUST be written alongside or before the implementation of each task;
  a task is not complete until its tests pass. Test-first (red-green-refactor)
  is RECOMMENDED for business logic.
- The suite SHOULD follow the test pyramid: many fast unit tests, fewer
  integration tests (Spring slice tests, Testcontainers), and a small number
  of end-to-end tests.
- Acceptance criteria from each spec SHOULD map to at least one automated
  test, traceable by name or tag.
- Feature-flagged behavior MUST be tested in both flag states (Principle VI).
- API contracts (Principle V) SHOULD be verified by contract tests (e.g.
  Spring Cloud Contract, Pact, or schema validation against the OpenAPI spec).
- CI MUST enforce a minimum coverage threshold of `[COVERAGE_THRESHOLD]` on
  new code (e.g. JaCoCo). Coverage is a floor, not a goal; tests MUST assert
  behavior, not merely execute lines.
- Flaky tests MUST be fixed or quarantined with a tracking issue within one
  sprint; they MUST NOT be silently ignored.

**Rationale:** Specs are only valuable if the implementation is provably
faithful to them. Automated tests are the executable form of the spec.

### VIII. Vendor-Neutral Observability

Every service MUST be operable in production without attaching a debugger or
shelling into a pod. Instrumentation MUST use open standards so that the
observability backend can change without code changes.

- The organization's observability platform is Dynatrace (migrating from
  Kibana/Grafana). Application instrumentation MUST use vendor-neutral
  standards: OpenTelemetry for traces, Micrometer with Prometheus/OTLP export
  for metrics, and structured logs. Application code MUST NOT call
  vendor-specific observability APIs directly.
- Logs MUST be structured (JSON) and include timestamp, level, service name,
  and trace/correlation ID. Logs MUST NOT contain secrets, credentials, or
  unmasked PII.
- Services MUST expose metrics including at minimum the RED metrics (rate,
  errors, duration) for inbound and outbound calls, plus JVM metrics.
  Business-relevant metrics SHOULD be added for key domain operations.
- Distributed tracing MUST be enabled (OpenTelemetry / Micrometer Tracing)
  with context propagated (W3C Trace Context) across HTTP and messaging
  boundaries.
- Feature flag state SHOULD be recorded on relevant traces or logs so that
  behavior can be correlated with enablement (Principle VI).
- Actuator endpoints beyond health/info/metrics MUST NOT be publicly exposed.

**Metrics drive health and alerting; logs are for debugging**

- It MUST be possible to determine the service's health from metrics alone,
  without querying logs.
- Alerts and SLOs MUST be based on metrics (including metrics derived from
  traces). Alerts MUST NOT depend on log searches, log pattern matching, or
  log volume.
- Every condition that matters operationally MUST emit a metric: errors by
  type, failed outbound calls, rejected or dead-lettered messages,
  circuit-breaker state, retry exhaustion, queue/backlog depth, and
  business-critical failures. Logging an error is not a substitute for
  counting it. If you would want an alert on it, it MUST be a metric.
- Each service MUST define Service Level Indicators (SLIs) as metric queries
  (e.g. availability = non-5xx responses / total; latency = p95/p99 from
  timer histograms) and SHOULD define SLOs and burn-rate alerts on them.
  Alerts SHOULD be tied to user-facing symptoms rather than internal causes.
- Metric names and tags MUST follow a consistent convention, and tag values
  MUST have bounded cardinality (no user IDs, request IDs, or raw URLs as
  tags).
- Logs are a diagnostic tool, used to investigate an incident after metrics
  have detected and localized it. Correlation through trace IDs SHOULD make it
  possible to move from an alerting metric to the relevant traces and logs.

**Rationale:** In a horizontally scaled, disposable system, any individual pod
may be gone by the time an issue is investigated, so telemetry is the only
durable record of what happened. Metrics are cheap, structured, and
aggregate cleanly across replicas. Log-based alerts are expensive, brittle
(a reworded log message silently breaks an alert), and slow. Detecting with
metrics and diagnosing with logs and traces gives reliable alerting at lower
cost. Open standards (OpenTelemetry, Prometheus) let the current migration
and any future one happen at the platform layer without changing services.

### IX. Secure by Default

Security is a design constraint, not a later phase.

- Containers MUST run as a non-root user and SHOULD be compatible with a
  read-only root filesystem. Images SHOULD use a minimal approved base image
  from `[CONTAINER_REGISTRY]`.
- Services MUST authenticate and authorize every request that is not
  explicitly public (Spring Security); the default posture is deny.
- All external input MUST be validated (Bean Validation at API boundaries).
  Data access MUST use parameterized queries; string-concatenated SQL/JPQL
  MUST NOT be used.
- Services MUST run with least-privilege access to platform and backing
  resources, as defined by `[DEPLOYMENT_STANDARDS]`.
- Secrets scanning MUST run in CI and SHOULD run pre-commit.
- Security findings (critical/high CVEs) MUST be remediated within the
  company's defined SLA. Minimizing dependencies (Principle I) is the primary
  strategy for reducing this burden.

**Rationale:** Defaults determine outcomes. Building least privilege and
input validation into every service eliminates whole classes of
vulnerabilities.

### X. Backward-Compatible Change & Data Evolution

Because rolling deployments run old and new versions side by side, every
change MUST be compatible with the version immediately before it.

- Database schema changes MUST be managed with Liquibase, versioned in source
  control, and applied automatically and repeatably. Changesets MUST NOT be
  edited after they have been applied to any shared environment.
- Schema changes MUST follow expand/contract: additive changes ship first;
  destructive changes (drops, renames, NOT NULL on existing columns) ship
  only after no running version depends on the old shape.
- Migrations MUST be safe to run while the previous application version is
  serving traffic and MUST NOT take long-running locks on large tables
  without a documented plan. Migrations SHOULD include rollback definitions
  where practical.
- Message and event schemas MUST evolve compatibly (tolerant readers; new
  fields optional) or be versioned.
- Feature flags (Principle VI) SHOULD be used to coordinate multi-step
  changes so that new behavior is enabled only after supporting schema and
  contract changes are fully deployed.

**Rationale:** Zero-downtime deployment (Principle IV) is impossible if a
release requires all pods to change at once or breaks the version still
running.

### XI. Reproducible Builds & Immutable Artifacts

What is tested is exactly what is deployed.

- TeamCity is the system of record for builds. It MUST be the only path to an
  artifact that is deployed to a shared or production environment.
- Builds MUST be reproducible from a clean checkout using only the repository
  and approved repositories (Principle I), via the Gradle wrapper.
- Artifacts MUST be built once and promoted through environments unchanged;
  environments differ only by configuration (Principle III).
- Container images MUST be tagged with an immutable identifier (version
  and/or Git SHA). The `latest` tag MUST NOT be used for deployment.
- Packaging, image publishing, and deployment MUST follow
  `[DEPLOYMENT_STANDARDS]` and the company's approved deployment frameworks.
  Projects SHOULD NOT invent their own deployment tooling.

**Rationale:** Immutable, traceable artifacts make rollbacks simple, audits
possible, and "works on staging but not prod" bugs rare. Aligning with the
company's build and deployment platforms keeps services supportable by the
teams that operate them.

### XII. Simplicity & Convention over Invention

Prefer the simplest solution that satisfies the specification.

- Use Spring Boot conventions, starters, and auto-configuration before
  writing custom infrastructure code. Custom frameworks or abstractions MUST
  be justified in the plan.
- Do not build for speculative requirements (YAGNI). Features not in the spec
  MUST NOT be implemented.
- Code SHOULD be organized by feature/domain rather than by technical layer,
  with clear module boundaries.
- Code MUST pass the agreed formatter and static analysis (e.g. Spotless,
  Checkstyle/PMD, SpotBugs or Error Prone, Sonar quality gate) in CI.
- Decisions with long-term impact SHOULD be recorded as Architecture
  Decision Records (ADRs) in the repository.
- Any violation of this principle MUST be recorded in the plan's Complexity
  Tracking section with the simpler alternative that was rejected and why.

**Rationale:** Complexity is the main long-term cost of software. Conventions
make services familiar to every engineer in the organization and easier for
AI agents to generate correctly.

---

## Technology Baseline

These defaults apply unless a plan justifies a deviation.

| Concern                | Standard                                                        |
|------------------------|-----------------------------------------------------------------|
| Language / Runtime     | Java `[JAVA_LTS]` (company-approved LTS)                        |
| Framework              | Spring Boot (current supported release line) + Spring ecosystem |
| Build tool             | Gradle via wrapper, version catalog, resolving from `[ARTIFACT_REPO]` |
| CI                     | TeamCity                                                        |
| Testing                | JUnit 5, AssertJ, Mockito, Spring Boot Test, Testcontainers     |
| API documentation      | OpenAPI 3.x, path-versioned (`/v1`, `/v2`); AsyncAPI for events |
| Database migrations    | Liquibase                                                       |
| Feature flags          | `[FEATURE_FLAG_SYSTEM]` behind a single in-app abstraction      |
| Resilience             | Resilience4j                                                    |
| Observability          | OpenTelemetry + Micrometer (Prometheus/OTLP), JSON logs; Dynatrace backend |
| Deployment             | Per `[DEPLOYMENT_STANDARDS]` and company deployment frameworks  |

---

## Decision-Making & Consistency

Decisions are made using this order of precedence:

1. **MUST / MUST NOT rules are always followed.** Deviation is possible
   only through the formal exception process (see Governance).
2. **SHOULD rules are followed by default.** Deviation is allowed when there
   is a good reason, and that reason MUST be recorded in the plan's
   Complexity Tracking section.
3. **When several options are all reasonable, prefer consistency** with the
   established patterns in the company's other repositories and services:
   libraries, project layout, naming, configuration conventions, error
   formats, API style, testing approach, and so on. Plans SHOULD name the
   existing service or pattern they follow. A familiar codebase is easier to
   review, support, onboard into, and maintain across teams. This applies to
   AI-generated code too.

**Consistency is a tiebreaker, not a trump card.**

> "A foolish consistency is the hobgoblin of little minds." — Ralph Waldo Emerson

- Consistency MUST NOT be used to justify violating a MUST, weakening
  security or correctness, or choosing a clearly worse fit for the problem.
- Existing patterns SHOULD NOT be copied when they are known to be
  problematic, are deprecated, conflict with this constitution, or exist only
  because of constraints that no longer apply. "That's how the other services
  do it" is a valid reason to choose between equals. It is not a valid reason
  to keep a known problem.
- Intentional divergence from an established pattern MUST be recorded (an
  ADR, or the plan's Complexity Tracking section), stating what is different,
  why, and whether it should become the new standard.
- If a divergence proves to be better, it SHOULD be proposed as the new
  convention (through a constitution amendment, shared library, or
  cross-team guidance) so the organization converges on the better approach
  instead of fragmenting.

**Rationale:** Across many microservices, consistency is very valuable: it
lowers cognitive load and support cost. But consistency applied without
judgment locks in past mistakes. Defaulting to consistency, while
documenting and spreading deliberate improvements, lets the ecosystem stay
coherent and still get better over time.

---

## Development Workflow & Quality Gates

- All work follows the Spec Kit flow: `/speckit.constitution` →
  `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` →
  `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement`.
- Specs describe *what* and *why* and MUST NOT prescribe implementation
  details. Each spec MUST include a Feature Flag section (Principle VI).
  Plans describe *how* and MUST pass the Constitution Check below.
- **Constitution Check (plan gate).** Every plan MUST explicitly confirm,
  or justify a deviation from, each of the following:
  1. All dependencies/images resolve from approved repositories; any new
     dependency is justified as differentiating (I)
  2. App and unit tests run locally with documented Gradle commands (II)
  3. Stateless, multi-replica safe, externalized config (III)
  4. Graceful shutdown, health endpoints, idempotency, timeouts (IV)
  5. OpenAPI contract present, path-versioned, and linted for any API (V)
  6. Feature flag defined and documented, or exemption justified (VI)
  7. Test strategy maps to acceptance criteria and covers both flag states (VII)
  8. Metrics, traces, and logs defined using OTel/Micrometer; SLIs and
     alerts defined from metrics only, with every alertable failure emitting
     a metric (VIII)
  9. Security posture: authn/z, non-root, least privilege (IX)
  10. Liquibase changes and contracts are backward compatible (X)
  11. Built by TeamCity, immutable artifact, deployed per company standards (XI)
  12. No unjustified complexity (XII)
  13. Where options were equally valid, the plan follows established company
      patterns, or the divergence is documented with its reasoning
      (Decision-Making & Consistency)
- Task lists MUST include, where applicable: flag creation and registration,
  tests for both flag states, a follow-up flag-removal task, and metrics/SLI
  instrumentation for new behavior.
- All changes MUST go through pull request with at least one approving
  review. Reviewers MUST check constitutional compliance, not only code
  correctness.
- The TeamCity build MUST be green (build, tests, coverage, static analysis,
  dependency and secrets scanning, OpenAPI lint/diff) before merge.
- AI-generated code is held to exactly the same standards as human-written
  code; the human submitting it is accountable for it.

---

## Governance

- This constitution supersedes all other team practices and conventions.
  Where a spec, plan, or task conflicts with it, the constitution wins.
  Where company-wide platform standards (build, deployment, security) are
  stricter, those standards apply.
- **Exceptions** MUST be documented in the plan's Complexity Tracking section
  (what, why, simpler alternative rejected, and an expiry or review date) and
  approved during plan review. Exceptions to Principle I (Trusted & Minimal
  Supply Chain) additionally require approval from
  `[SECURITY_OR_PLATFORM_OWNER]`.
- **Amendments** are proposed via pull request to
  `.specify/memory/constitution.md`, MUST include an updated Sync Impact
  Report, and MUST be approved by `[GOVERNANCE_APPROVERS]`. Dependent
  templates MUST be updated in the same change.
- **Versioning** follows semantic versioning:
  - MAJOR: removal or backward-incompatible redefinition of a principle
  - MINOR: new principle or materially expanded guidance
  - PATCH: clarifications, wording, typo fixes
- Compliance SHOULD be reviewed at least quarterly, and this document SHOULD
  be revisited whenever the platform (Kubernetes, Spring Boot major version,
  Java LTS, observability or deployment tooling) changes significantly.

**Version**: 1.0.0 | **Ratified**: [RATIFICATION_DATE] | **Last Amended**: [RATIFICATION_DATE]
