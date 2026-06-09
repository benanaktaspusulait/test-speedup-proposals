# CI/CD and Test Pipeline Speed-Up Approach

| Metadata | Value |
|----------|-------|
| Owner | TBC |
| Initial contributor | Benan Aktas |
| Reviewers | Service team, Platform/CI team, QA, Delivery/ETO representatives |
| Status | Draft |
| Created | 2026-06-09 |
| Last updated | 2026-06-09 |
| Last reviewed | 2026-06-09 |
| Labels | `proposal`, `ci-cd`, `test-pipeline` |

## 1. Executive Summary

This proposal defines a reusable approach to reduce CI/CD feedback time for Java/Maven services with Cucumber-based tests.

The issue is not one specific class or repository. The common pattern is sequential test execution, noisy test logs, and E2E harness waits that add avoidable delay. The target outcome is faster feedback while keeping scenario coverage, reliability, and failure diagnosis intact.

This page is the short stakeholder summary. Detailed tables and implementation notes are split into child pages.

---

## Decision / Review Needed

This proposal is shared for review and alignment. The requested feedback is:

- Confirm whether this structure is suitable for wider reuse across services.
- Confirm which quick-win items should be prioritised first.
- Identify any proposals that need a separate DACI review before implementation.
- Agree the right owner or team for the next step.

---

## 2. Page Structure

| Page | Purpose |
|------|---------|
| Proposal Overview Matrix | Value, risk, complexity, effort, and MoSCoW view |
| Phased Plan | What happens in each phase and why |
| Risks and DACI | Main risks, mitigations, and decision areas |
| Technical Details | Commands, config examples, and implementation guardrails |
| References and Evidence | Source documents, external docs, and example baseline |

> TODO: replace page names above with Confluence child page links after publishing.

---

## 3. Context / Problem Statement

- Tests often run sequentially and underuse CI capacity.
- Pipeline delays slow developer feedback and urgent fixes.
- Large logs make failures harder to review.
- E2E tests may spend avoidable time polling, waiting, or repeating readiness checks.
- Teams need a common approach rather than service-by-service reinvention.

Example measured baseline evidence is listed in the References and Evidence child page. Each service must capture its own baseline before adopting changes.

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
- measurable reduction in test duration (target confirmed per service after baseline capture)
- clear rollback path for each phase
- reusable pattern for future services

This approach is reusable, but implementation must be validated per service.

Feedback or questions? Comment on the Confluence page or contact the page owner once assigned.
