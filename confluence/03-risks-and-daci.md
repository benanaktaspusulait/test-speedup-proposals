# Risks and DACI

> Child page of CI/CD and Test Pipeline Speed-Up Approach.

| Metadata | Value |
|----------|-------|
| Owner | TBC |
| Status | Draft |
| Created | 2026-06-09 |
| Last updated | 2026-06-09 |
| Last reviewed | 2026-06-09 |
| Labels | `proposal`, `ci-cd`, `test-pipeline`, `daci` |

## Risks and Mitigations

| Risk | Impact | Mitigation | Owner |
| ---- | ------ | ---------- | ----- |
| CI worker lacks CPU/memory | Forked tests may not speed up | Benchmark conservative fork counts | TBC |
| Duplicate scenarios after runner split | Longer run and false confidence | Remove/exclude original all-features runner | TBC |
| Hidden ordering dependency | Flaky or inconsistent results | Compare scenario count and repeat CI runs | TBC |
| Lower logs hide failure detail | Harder investigation | Keep debug log override | TBC |
| JaCoCo `argLine` conflict | Coverage or test execution issue | Use JaCoCo-safe `@{argLine}` | TBC |
| JUnit 5 exposes shared state | Flaky same-JVM parallel runs | Remove static mutable state first | TBC |
| E2E sharding cross-talk | Incorrect E2E results | Shard only with isolated resources | TBC |
| CI matrix overhead | Smaller speed-up than expected | Compare against Surefire forks | TBC |

---

## DACI / Decision Areas

Some changes may need separate DACI-style decision records before implementation.

| Decision Area | Why It May Need DACI | Participants | Status |
| ------------- | -------------------- | ------------ | ------ |
| Surefire / Failsafe config | Affects CI and local execution | Service Team, Platform/CI Team, Tech Lead | TBC |
| Runner split model | Affects ownership and reporting | Service Team, QA, Tech Lead | TBC |
| JUnit 5 migration | Dependency and thread-safety change | Engineering, Architecture, QA | TBC |
| E2E harness changes | Reliability-sensitive defaults | Service Team, QA, Delivery | TBC |
| CI matrix/sharding | Platform and reporting impact | Service Team, Platform/CI Team, Delivery | TBC |
| Docker/cache work | Separate larger pipeline stream | Platform/CI Team, Architecture | Separate / TBC |

