# Phased Plan

> Child page of CI/CD and Test Pipeline Speed-Up Approach.

| Metadata | Value |
|----------|-------|
| Owner | TBC |
| Status | Draft |
| Created | 2026-06-09 |
| Last updated | 2026-06-09 |
| Last reviewed | 2026-06-09 |
| Labels | `proposal`, `ci-cd`, `test-pipeline` |

## Phase Summary

| Phase | Purpose | Candidate Changes | Exit Criteria |
|-------|---------|-------------------|---------------|
| Phase 1: Low-risk quick wins | Prove baseline and remove obvious overhead | timing reports, log reduction, Maven `-T` assessment where module graph supports it, E2E poll fix | same scenario count, smaller logs, timings captured |
| Phase 2: Medium-complexity improvements | Achieve main near-term speed-up | runner split, Surefire forks, JVM tuning, config caching, E2E harness tuning | all groups pass, no duplicates, no flaky increase |
| Phase 3: Structural improvements | Improve balance and diagnostics | rebalance groups, remove static state, assertion aliases, CI matrix assessment | clearer failures, balanced groups |
| Phase 4: Optional future improvements | Unlock deeper parallelism if needed | JUnit 5 parallelism, E2E sharding with isolation | stable repeated runs, no cross-test interference |

---

## Phase 1: Low-Risk Quick Wins

**Objective:** establish baseline and remove low-risk overhead.

**Expected outcome:** smaller logs, clearer timing evidence, and fewer avoidable E2E waits.

**Risks / dependencies:** CI command ownership is `TBC`; exact gain depends on service baseline.

---

## Phase 2: Medium-Complexity Improvements

**Objective:** introduce the main near-term parallelisation model while keeping rollback simple.

**Expected outcome:** topology/unit-style tests improve where the suite can be split into balanced runner groups; E2E green runs spend less time in avoidable harness overhead.

**Risks / dependencies:** fork count must fit CI CPU/memory; each fork may load its own Spring context; the original all-features runner must be removed or excluded.

---

## Phase 3: Structural Improvements

**Objective:** improve long-term maintainability and prepare for deeper parallelism.

**Expected outcome:** better runner balance, clearer failure ownership, less static scenario state, and less coupling between feature files and internal model paths.

**Risks / dependencies:** CI matrix changes may require platform coordination; assertion aliasing must not change test meaning.

---

## Phase 4: Optional Future Improvements

**Objective:** unlock deeper parallelism only where the service needs it.

**Expected outcome:** higher parallelism may be possible after thread-safety work; E2E sharding may help only with safe isolation.

**Risks / dependencies:** JUnit 5 can expose hidden shared state; E2E sharding is unsafe without isolated topics, consumer groups, data, or infrastructure.
