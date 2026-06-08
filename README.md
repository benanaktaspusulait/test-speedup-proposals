# SNS Command Adaptor — Test Execution Speed-Up Proposals

## Purpose

This repository contains a set of proposals to improve the test execution speed of the `fdp-cmd-adaptor-sns` pipeline. The topology tests currently run sequentially and become a bottleneck as more feature files are added.

## Context

The SNS Command Adaptor transforms Safety & Security (SNS) data from CDLz ingest format into POLE domain records via Kafka Streams. The project has **343 test scenarios** spread across **66 Cucumber feature files**, all running through a single test runner in a single JVM with no parallelization.

Current test execution: **~18 seconds** for topology tests alone.  
Current observed CI stage: **~13m25s** total elapsed time.

The current Drone view shows `mvn clean install` at **~01m17s**. Proposals 1-5 primarily optimize that Maven/topology-test slice. Proposal 6 covers E2E test harness runtime. Docker cache, test profiler work, base Docker image work, container startup, and the `Command Adaptor` service lifecycle are treated separately and are intentionally outside this proposal set.

As the mapping coverage grows, more features are added, and this time increases linearly.

## Documents

| Document | Description |
|----------|-------------|
| [01-current-state.md](./01-current-state.md) | Analysis of the current test architecture, bottlenecks, and why tests are slow |
| [02-proposal-maven-parallel-build.md](./02-proposal-maven-parallel-build.md) | Quick win: parallel module builds with Maven `-T` flag |
| [03-proposal-multiple-runners-fork.md](./03-proposal-multiple-runners-fork.md) | Main proposal: split runners + Surefire forkCount for 2-3x likely improvement |
| [04-proposal-junit5-parallel-engine.md](./04-proposal-junit5-parallel-engine.md) | Long-term: migrate to JUnit 5 Platform Engine with scenario-level parallelism |
| [05-proposal-quick-wins.md](./05-proposal-quick-wins.md) | Non-parallel optimizations: test logging, fixture caching, JaCoCo profile, build & JVM tuning |
| [06-proposal-ci-matrix-sharding.md](./06-proposal-ci-matrix-sharding.md) | CI-level alternative: split runner groups into separate parallel Drone steps |
| [07-proposal-e2e-runtime.md](./07-proposal-e2e-runtime.md) | E2E test harness optimizations excluding Docker/container work |

## Summary Comparison

The proposals fall into three categories: **topology-test parallelization**, **per-scenario topology optimization**, and **E2E test harness optimization**. They are complementary, but only one topology parallelization model should be chosen at a time.

### Parallelization

| Proposal | Expected Speed-Up | Risk | Effort | Recommended Phase |
|----------|-------------------|------|--------|-------------------|
| Maven Parallel Build (`-T 4` or `-T 1C`) | Low single-digit overall; ~1s on current reactor | Very low | 1 hour | Immediate, but not the main speed-up |
| Multiple Runners + forkCount | 2-3x likely; up to 3-4x after event split/tuning | Low | 1-2 days | Short-term |
| CI Matrix by Runner | 2-4x topology wall-clock reduction | Low-Medium | 1-2 days | Short-term alternative |
| JUnit 5 Parallel Engine | 3-5x potential | Medium-High | 3-5 days | Long-term |

> Note: Multiple Runners (Proposal 2), JUnit 5 Engine (Proposal 3), and CI Matrix (Proposal 5) are alternative ways to parallelize topology tests. Pick one primary execution model at a time.

### Per-Scenario Optimization (Proposal 4 — stacks with any of the above)

| Optimization | Effect | Risk | Effort |
|--------------|--------|------|--------|
| Reduce test logging to WARN | Removes ~76k synchronous console writes per run | Very low | 10 min |
| Read mapping version from Spring config | Removes 342 redundant file reads | Low | 30 min |
| JaCoCo skip profile (local) | Removes instrumentation overhead locally | Low | 30 min |
| Avoid `clean` locally | Skips code generation + recompilation | None | 0 |
| Test JVM tuning | Faster fork startup | Low | 30 min |

### E2E Runtime Optimization

| Proposal | Expected Speed-Up | Risk | Effort | Scope |
|----------|-------------------|------|--------|-------|
| E2E harness quick fixes | 20-60s on green E2E runs; larger fail-fast gain | Low | 1-2 days | Polling, logging, readiness, report timing |
| Isolated E2E feature sharding | Potentially larger, baseline-dependent | Medium | 3-5 days | Only if topic/consumer/infrastructure isolation is available |

## Recommended Approach

1. **Start with Proposal 4.1** (reduce logging) — near-zero risk, no logic change, likely the single biggest standalone win since the current run produces a 76k-line log
2. **Capture a baseline matrix** before and after each change — current, logging-only, runner split with `forkCount=2/4/6`
3. **Apply Proposal 1** (Maven `-T 1C` or `-T 4`) — zero risk, but expect a small gain only
4. **Implement Proposal 2** (multiple runners + forkCount) as the primary parallelization improvement — best effort-to-impact ratio
5. **Consider Proposal 5** if CI observability or per-group retry/debuggability is more valuable than one Maven test invocation
6. **Apply remaining Proposal 4 items** (mapping version config, JVM tuning) alongside Proposal 2
7. **Plan Proposal 3** (JUnit 5 engine) for a future sprint if even more speed is needed
8. **Apply Proposal 6 quick fixes** for E2E runtime — start with per-feature timing, the RunLog poll unit fix, log reduction, and readiness caching

## Expected Impact on Current CI

On the current **~13m25s** CI stage, expected savings should be read separately because Drone steps overlap. A saving inside one step does not always reduce the total stage by the same amount.

### Individual Estimated Gains

| Area | Proposal | Current | Expected | Step Saving | Full CI Saving |
|------|----------|---------|----------|-------------|----------------|
| Maven reactor | Proposal 1: Maven `-T` | `mvn clean install` ~01m17s | ~01m16s | ~1s | ~0-1s |
| Topology tests | Proposal 4.1: logging reduction | topology ~18s | baseline-dependent | ~2-6s | ~2-6s if Maven is on critical path |
| Topology tests | Proposal 4.2: mapping version from Spring config | topology ~18s | almost unchanged | <1s | <1s |
| Topology tests | Proposal 2: runner split + Surefire forks | topology ~18s | ~7-10s | ~8-11s | ~8-11s if Maven is on critical path |
| Topology tests | Proposal 4.5: JVM tuning with forks | topology after fork split | slightly lower | ~1-3s | ~1-3s if Maven is on critical path |
| Topology tests | Proposal 5: CI matrix by runner | topology ~18s | ~5-9s | ~9-13s | alternative to Proposal 2, not additive |
| Topology tests | Proposal 3: JUnit 5 parallel engine | topology ~18s | ~5-8s | ~10-13s | alternative to Proposal 2/5, not additive |
| E2E harness | Proposal 6 quick fixes | `Integration Tests` ~09m34s | ~08m34s-09m14s | ~20-60s | ~20-60s if Integration Tests is on critical path |
| E2E harness | Proposal 6 isolated sharding | `Integration Tests` ~09m34s | baseline-dependent | potentially +30-180s | only if test execution, not container wait, dominates |

### Combined Safe Path

This is the recommended short-term path: Proposal 1 + Proposal 2 + selected Proposal 4 quick wins + Proposal 6 quick fixes.

| Scope | Current | Expected | Estimated Saving |
|-------|---------|----------|------------------|
| Topology tests inside Maven | ~18s | ~7-10s | ~8-11s |
| `mvn clean install` step | ~01m17s | ~01m02s-01m07s | ~10-15s |
| `Integration Tests` step | ~09m34s | ~08m34s-09m14s | ~20-60s |
| Full CI stage | ~13m25s | ~12m10s-12m55s | ~30-75s |

This is still worth doing because it makes the fast feedback path cheaper and stops topology-test growth from scaling linearly. Larger pipeline gains are expected from the separate Docker cache, test profiler, and base Docker image work, not from the Maven/topology-test proposal set alone.

For E2E specifically, Proposal 6 targets the current **~09m34s Integration Tests** step without changing Docker/container startup. The first expectation is **20-60s** from safer polling, lower log volume, readiness caching, and better batching. Bigger E2E gains need isolated sharding or the separate Docker cache, test profiler, and base Docker image work.

### Final Target After Separate Docker/Test-Profiler Work

If the separate Docker cache, test profiler, and base Docker image work removes most of the `Command Adaptor` build/startup wait, the realistic target is no longer just a 30-75s saving. In that case the pipeline becomes dominated by fixed setup, actual E2E execution, and Trivy.

| Scenario | Expected Full CI Time | Saving vs ~13m25s |
|----------|-----------------------|-------------------|
| Conservative: Docker/cache improved, E2E harness quick fixes only | ~07m00s-08m30s | ~05m00s-06m25s |
| Realistic target: Docker/cache improved + E2E wait/polling cleaned up | ~06m00s-07m30s | ~05m55s-07m25s |
| Stretch: Docker/cache improved + isolated E2E sharding works safely | ~05m00s-06m30s | ~06m55s-08m25s |

The practical lower bound without also optimizing Kafka/Redis startup, aggregator startup, and Trivy is roughly **~05m**. Getting below that likely requires changing the wider CI shape, not only Maven/topology/E2E harness changes.

## How to Read

Each proposal document includes:
- Problem statement
- Proposed solution with code examples
- Step-by-step implementation guide
- Risks and mitigations
- Expected outcomes

## Affected Repository

- **Repository:** `dacc-aws/fdp-cmd-adaptor-sns`
- **Branch:** `develop`
- **Version:** 2.3.2-SNAPSHOT
