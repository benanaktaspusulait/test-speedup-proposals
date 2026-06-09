# CI/CD and Test Pipeline Speed-Up Approach

## 1. Executive Summary

This document presents a structured approach to improving test execution speed and CI pipeline efficiency across our Java/Maven microservice estate. Many of our services follow a similar pattern: Cucumber BDD tests running sequentially in a single JVM, verbose test logging, wide polling timeouts in E2E harnesses, and no test-level parallelisation. As feature coverage grows, test execution time increases linearly and CI feedback loops become a bottleneck.

The approach defines a reusable set of complementary improvements — from low-risk quick wins deliverable within hours to longer-term structural changes — that can be applied to any service exhibiting these patterns. The goal is faster developer feedback, reduced deployment risk during urgent fixes, and a test architecture that scales with feature growth.

The initial target for this approach is the SNS Command Adaptor (`fdp-cmd-adaptor-sns`), which has 343 Cucumber scenarios across 66 feature files running in ~18s topology test time within a ~13m25s CI stage. The patterns and solutions described here are designed to be transferable to other services with similar test architectures.

> **Child pages:**
> - [Technical Details](./01-technical-details.md) — implementation-level code, configuration examples, and architecture patterns
> - [References](./02-references.md) — links to repositories, pipelines, documentation, and source proposal documents

---

## 2. Problem Statement

The following problems are common across services that use Cucumber/JUnit-based test suites with Maven:

- **Sequential test execution:** Test scenarios run through a single Cucumber runner in a single JVM fork using one thread. Available CPU cores on CI runners are underutilised.
- **Linear growth:** Every new feature file increases pipeline time proportionally. Large feature directories dominate execution time.
- **Slow feedback cycle:** Full CI stages take minutes. Topology/unit tests and E2E integration tests both contribute to long wait times before developers receive feedback.
- **E2E harness inefficiencies:** Common issues include incorrect poll duration units, wide timeout windows, verbose logging, low consumer batch sizes, and uncached readiness checks that add avoidable overhead.
- **Deployment risk:** Long pipelines increase the window of risk during urgent fixes and slow iterative development.
- **No standard parallelisation pattern:** Services independently solve (or don't solve) test parallelisation, leading to inconsistent approaches and duplicated effort.

---

## 3. Goals

- Reduce test execution time significantly (target: 2-3x improvement for topology/unit tests).
- Improve developer feedback cycle for local and CI test runs.
- Reduce E2E integration test overhead on green runs.
- Ensure test execution time does not scale linearly with new feature additions.
- Maintain or improve test coverage and reliability — no reduction in scenario count or assertion quality.
- Define a reusable pattern that can be applied across multiple services.
- Identify quick wins that can be delivered immediately alongside longer-term improvements.
- Make test failures easier to diagnose.

---

## 4. Scope

### In Scope

- Maven/Surefire/Failsafe configuration tuning
- Cucumber runner architecture and parallelisation patterns
- JUnit 5 parallel engine migration path
- Test logging and JVM tuning
- E2E test harness polling, logging, and readiness improvements
- CI pipeline step parallelisation options
- Feature assertion DSL maintainability patterns

### Out of Scope

- Docker cache and base Docker image optimisation
- Test profiler infrastructure
- Docker Compose startup and container readiness orchestration
- Service lifecycle changes (application startup/shutdown)
- Message broker / data store startup optimisation
- Security scan (e.g. Trivy) duration
- Infrastructure-level CI changes beyond test step parallelisation

---

## 5. Approach

The proposals are grouped into three complementary areas. These are general patterns applicable to any service with a similar test architecture. Only one topology/unit test parallelisation model should be active at a time per service.

### 5.1 Test Parallelisation

Three alternative models for running tests in parallel:

- **Multiple Runners + Surefire forkCount** — Split a single Cucumber runner into multiple runners (e.g. one per feature directory) and use Surefire to run them in parallel JVM forks. Best effort-to-impact ratio.
- **CI Matrix by Runner Group** — Run each runner as a separate CI step for better diagnostics and per-group retry.
- **JUnit 5 Platform Engine** — Migrate to JUnit 5 with scenario-level parallelism within a single JVM. Higher potential but requires thread-safety refactoring.

### 5.2 Per-Scenario and Build Optimisation (Quick Wins)

Independent improvements that reduce per-scenario cost and compound with any parallelisation model:

- Reduce test logging verbosity (eliminate synchronous console writes during green runs)
- Cache or inject configuration values instead of per-scenario file parsing
- Make code coverage (JaCoCo) optional for local runs
- JVM startup tuning for test forks
- Maven parallel module build (`-T` flag)
- Feature assertion DSL normalisation (maintainability, not speed)

### 5.3 E2E Test Harness Improvements

Optimisations to E2E integration test steps without changing Docker/container infrastructure:

- Fix poll duration unit bugs (common: seconds vs milliseconds confusion)
- Make poll timeouts configurable via system properties
- Increase consumer batch sizes for high-record-count assertions
- Reduce E2E log volume
- Cache readiness/health checks per test run
- Add per-feature timing baselines for visibility

---

## 6. Proposal Matrix

| Proposal | Description | Value | Risk | Complexity | Effort | MoSCoW | Notes |
|----------|-------------|-------|------|------------|--------|--------|-------|
| Maven Parallel Build (`-T`) | Parallel module builds via Maven reactor | Low | Low | Low | Low (1h) | Could | Small gain on multi-module projects; useful cleanup |
| Multiple Runners + forkCount | Split runners + Surefire parallel forks | High | Low | Medium | Medium (1-2d) | Must | 2-3x likely; primary short-term parallelisation |
| JUnit 5 Parallel Engine | Scenario-level parallelism, single JVM | High | Medium | High | High (3-5d) | Could | 3-5x potential; requires thread-safety refactoring |
| Reduce Test Logging | Set test logging to WARN during CI | High | Low | Low | Low (10min) | Must | Eliminates synchronous console I/O; often the highest standalone quick win |
| Config Value Injection | Inject config values via Spring instead of per-scenario file I/O | Low | Low | Low | Low (30min) | Should | Removes redundant file reads per scenario |
| JaCoCo Skip Profile | Make coverage optional for local runs | Low | Low | Low | Low (30min) | Should | Local developer experience only |
| JVM Tuning for Forks | TieredStopAtLevel=1, ParallelGC | Medium | Low | Low | Low (30min) | Should | Reduces fork startup; compounds with runner split |
| Feature Assertion DSL | Normalise assertion keys in feature files | Low | Low | Medium | Medium (1-2h) | Could | Maintainability; prevents future broad churn |
| CI Matrix by Runner Group | Run each runner as separate CI step | High | Medium | Medium | Medium (1-2d) | Could | Alternative to forkCount; better diagnostics |
| E2E Poll Duration Fix | Fix seconds vs milliseconds bugs | Medium | Low | Low | Low (10min) | Must | Prevents extreme waits on slow/missing records |
| E2E Configurable Polling | Make poll timeout/interval configurable | Medium | Low | Low | Low (30min) | Should | Faster failure detection |
| E2E Consumer Batch Size | Increase max.poll.records | Medium | Low | Low | Low (30min) | Should | Faster high-count record checks |
| E2E Log Reduction | Set E2E test logging to WARN | Medium | Low | Low | Low (30min) | Should | Lower CI log overhead |
| E2E Cached Readiness | Cache readiness health check per run | Low | Low | Low | Low (30min) | Should | Small green-run win |
| E2E Feature Sharding | Parallel E2E runners with isolation | High | Medium | High | High (3-5d) | Could | Only viable with topic/consumer/infrastructure isolation |

---

## 7. Phased Plan

### Phase 1: Low-Risk Quick Wins

**Objective:** Capture immediate gains with near-zero risk. Applicable to any service immediately.

**Proposed changes:**
- Add a `logback-test.xml` to set test logging to WARN
- Fix any poll duration unit bugs in E2E harness code
- Add per-feature E2E timing output (JSON report)
- Apply Maven `-T 1C` parallel reactor flag
- Capture baseline timing matrix before and after each change

**Expected outcome:**
- Reduced log I/O overhead during test execution
- Elimination of potential extreme poll waits in E2E
- Visibility into per-feature E2E timing bottlenecks

**Success criteria:**
- All existing test scenarios pass
- Measurable reduction in test log output volume
- Per-feature timing report generated
- No regression in test reliability

**Dependencies or risks:**
- CI pipeline configuration may be managed centrally (e.g. RepoSync); `-T` flag changes may need platform team coordination
- Minimal risk; all changes are instantly reversible

---

### Phase 2: Medium-Complexity Improvements

**Objective:** Achieve 2-3x test speed-up through parallelisation and harness improvements.

**Proposed changes:**
- Create multiple Cucumber runner classes (one per feature directory or logical group)
- Configure Surefire `forkCount` with `reuseForks=true`
- Remove or rename original single runner class
- Inject configuration values via Spring instead of per-scenario file I/O
- Apply JVM tuning flags for test forks
- Make JaCoCo skippable locally
- E2E: configurable polling, increased batch size, log reduction, cached readiness

**Expected outcome:**
- Test execution time reduced by 2-3x
- E2E integration test step faster on green runs
- Improved local developer iteration speed

**Success criteria:**
- All test scenarios pass across all runner forks
- Test execution time consistently meets target
- E2E scenarios pass with reduced timing
- No increase in flaky test rate

**Dependencies or risks:**
- CI runner must have sufficient CPU and memory for parallel forks
- Spring context startup cost is multiplied by fork count (mitigated by `reuseForks=true`)
- Benchmark different `forkCount` values to find optimal setting per service

---

### Phase 3: Larger Structural Improvements

**Objective:** Further parallelisation and improved CI diagnostics.

**Proposed changes:**
- Split large runner groups into sub-runners based on measured timing
- Evaluate CI Matrix by Runner Group (separate CI steps per runner)
- Feature assertion DSL normalisation (decouple feature files from raw field names)
- Remove static field usage in step definitions (preparation for JUnit 5)

**Expected outcome:**
- More balanced parallelisation (no single group dominates wall time)
- Per-group CI diagnostics and retry capability
- Reduced maintenance cost for future dependency/model upgrades

**Success criteria:**
- No single runner group dominates more than 40% of total wall time
- CI step-level failures are isolated and retriable
- Feature assertion normalisation layer handles both legacy and new keys

**Dependencies or risks:**
- CI Matrix approach requires platform team coordination
- Sub-sharding requires measured scenario timing data

---

### Phase 4: Optional / Future Optimisation

**Objective:** Maximum parallelisation potential for services that need further speed.

**Proposed changes:**
- Migrate to JUnit 5 Platform Engine with Cucumber parallel execution
- Refactor step definitions for full thread safety (ThreadLocal or instance-scoped state)
- Audit shared state (lookup tables, caches) for concurrent safety
- E2E isolated feature sharding (separate topic suffixes or consumer groups per shard)

**Expected outcome:**
- 3-5x test speed-up potential (single JVM, shared Spring context)
- Lower memory footprint than fork-based approach
- E2E gains if isolation is achievable

**Success criteria:**
- All scenarios pass with target parallelism enabled
- No thread-safety regressions
- Multiple consecutive CI runs with zero flaky failures
- Memory usage lower than multi-fork approach

**Dependencies or risks:**
- Medium-High risk; requires thorough thread-safety refactoring
- Static mutable state in step definitions and shared classes is the main barrier
- Staged rollout required (sequential → low parallelism → full parallelism)
- E2E sharding requires infrastructure-level isolation support

---

## 8. Success Criteria

| Metric | Target | Notes |
|--------|--------|-------|
| Test execution time | 2-3x reduction (Phase 2) | Exact baseline varies per service |
| Full build step | Measurable reduction | Dependent on current module structure |
| E2E integration test step | 20-60s faster on green runs | Dependent on current harness overhead |
| Full CI stage | TBC per service | Depends on other pipeline components |
| Test scenario count | No reduction | Coverage must not decrease |
| Flaky test rate | No increase | Measured over 10+ consecutive runs |
| Test coverage | No reduction | JaCoCo or equivalent gate maintained |
| Applicability | Pattern reusable across services | Not a one-off solution |

---

## 9. Risks and Mitigations

| Risk | Impact | Mitigation | Owner / Follow-up |
|------|--------|------------|-------------------|
| CI runner lacks spare CPU for parallel forks | Reduced parallelism | Benchmark forkCount per service; use `0.5C` if worker size varies | TBC |
| Spring context startup multiplied by fork count | Added overhead per fork | `reuseForks=true`; cost paid once per fork | TBC |
| Increased memory usage with multiple forks | OOM in CI | Each fork requires heap allocation; ensure CI runner has sufficient memory | TBC |
| Thread-safety bugs exposed by JUnit 5 migration | Test failures | Staged rollout; run sequential first, then increase parallelism | TBC |
| CI pipeline configuration managed centrally | Blocks direct CI changes | Coordinate with platform team or use override mechanism | TBC |
| Static mutable state in step definitions | Prevents in-process parallelism | Refactor to instance fields; use ThreadLocal where needed | TBC |
| E2E sharding without topic/consumer isolation | Cross-scenario interference | Only shard with isolated infrastructure per shard | TBC |
| Loss of debug logging in CI | Harder failure diagnosis | Override log level via system property when debugging | TBC |
| Inconsistent adoption across services | Fragmented approach | Document patterns as reusable templates; provide reference implementation | TBC |

---

## 10. DACI / Decision Areas

The following areas may benefit from separate DACI-style decision records:

### DACI: Test Parallelisation Model

Three mutually exclusive options exist (Surefire forks, CI matrix, JUnit 5 engine). A decision record would clarify the chosen model per service, the criteria for switching, and ownership.

### DACI: Maven/Surefire/Failsafe Configuration Standards

Changes to build plugins affect all developers and CI. A decision record would document the chosen `forkCount`, JVM flags, and JaCoCo profile behaviour as a team standard.

### DACI: JUnit 5 Migration

This is a larger architectural change with thread-safety implications. A decision record would document the staged rollout plan, acceptance criteria for each stage, and rollback triggers.

### DACI: E2E Harness Improvements

Changes to polling, timeouts, and batch sizes affect E2E reliability. A decision record would document the chosen defaults, configurable parameters, and testing approach.

### DACI: CI Pipeline Structure

Any changes to CI step structure or pipeline configuration require coordination with the platform team. A decision record would document the agreed approach and ownership boundaries.

### DACI: Cross-Service Adoption

If this approach is to be applied across multiple services, a decision record would document prioritisation, rollout order, and which services are candidates.
