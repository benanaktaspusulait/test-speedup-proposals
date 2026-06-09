# CI/CD and Test Pipeline Speed-Up Approach

## 1. Executive Summary

This proposal defines a reusable approach to reduce CI/CD feedback time for Java/Maven services with Cucumber-based tests.

The issue is not one specific class or repository. The common pattern is sequential test execution, noisy test logs, and E2E harness waits that add avoidable delay. The target outcome is faster feedback while keeping scenario coverage, reliability, and failure diagnosis intact.

This page is the short stakeholder summary. Detailed tables and implementation notes are split into child pages.

---

## 2. Page Structure

| Page | Purpose |
|------|---------|
| [Proposal Overview Matrix](./01-proposal-overview-matrix.md) | Value, risk, complexity, effort, and MoSCoW view |
| [Phased Plan](./02-phased-plan.md) | What happens in each phase and why |
| [Risks and DACI](./03-risks-and-daci.md) | Main risks, mitigations, and decision areas |
| [Technical Details](./04-technical-details.md) | Commands, config examples, and implementation guardrails |
| [References and Evidence](./05-references.md) | Source documents, external docs, and example baseline |

---

## 3. Context / Problem Statement

- Tests often run sequentially and underuse CI capacity.
- Pipeline delays slow developer feedback and urgent fixes.
- Large logs make failures harder to review.
- E2E tests may spend avoidable time polling, waiting, or repeating readiness checks.
- Teams need a common approach rather than service-by-service reinvention.

Example measured baseline evidence is listed in [References and Evidence](./05-references.md). Each service must still capture its own baseline before adopting changes.

---

## 4. Objectives

- Reduce test and pipeline execution time where the service shape supports it.
- Improve local and CI feedback loops.
- Keep scenario count, coverage gates, and reliability unchanged.
- Separate quick wins from larger structural changes.
- Make technical decision points visible for review.
- Keep the approach reusable across services.

---

## 5. Scope

### In Scope

- Maven Surefire / Failsafe tuning
- Maven reactor parallelism
- Cucumber runner split and fork-based parallelisation
- JUnit 5 Platform Engine migration path
- test logging, JVM tuning, config caching, optional local JaCoCo skip
- E2E polling, logging, batch-size, readiness, and sharding guardrails
- CI runner-group matrix option

### Out of Scope

- Docker cache, base image, and test-profiler work
- Docker Compose/container startup orchestration
- service lifecycle redesign
- broker/database startup optimisation
- security scan duration
- full CI platform redesign

---

## 6. Approach

Apply the work in phases:

1. Measure the current baseline.
2. Apply low-risk quick wins.
3. Choose one topology-test parallelisation model per service.
4. Harden the test structure if deeper parallelism is needed.

The main execution-model options are:

| Option | When It Fits |
|--------|--------------|
| Multiple runners + Surefire forks | Main near-term option for grouped Cucumber suites |
| CI matrix by runner group | When diagnostics, ownership, or retry behaviour matter |
| JUnit 5 scenario parallelism | Future option after thread-safety work |

Do not enable multiple topology-test parallelisation models at the same time.

---

## 7. Success Summary

Success means shorter feedback loops without reducing test confidence:

- same scenario count
- same CI coverage gate
- no increase in flaky failures
- materially shorter test duration where the service shape supports it
- clear rollback path for each phase
- reusable pattern for future services
