# Proposal Overview Matrix

> Child page of [CI/CD and Test Pipeline Speed-Up Approach](./00-main-proposal.md).

This page gives the decision view: value, risk, complexity, effort, and MoSCoW rating.

| Proposal | Description | Value | Risk | Complexity | Effort | MoSCoW | Notes |
| -------- | ----------- | ----- | ---- | ---------- | ------ | ------ | ----- |
| Baseline timing/report output | Capture timings, scenario count, feature timing, and reports before changes | High | Low | Low | Low | Must | Needed before any decision |
| Reduce test logging | Lower noisy green-run logs with debug override | High | Low | Low | Low | Must | Quick win |
| E2E poll duration fix | Correct milliseconds/seconds polling mistakes | Medium | Low | Low | Low | Must | Avoids accidental long waits |
| Multiple runners + Surefire forks | Split large Cucumber runner into runner groups and run JVM forks | High | Medium | Medium | Medium | Must | Main near-term speed-up model |
| JVM tuning for forks | Add short-lived JVM flags for forked tests | Medium | Low | Low | Low | Should | Use with forked tests |
| Config injection/caching | Avoid repeated per-scenario config reads | Medium | Low | Low | Low | Should | Apply only where repeated reads exist |
| Optional local JaCoCo skip | Allow faster local runs while keeping CI coverage enabled | Low | Low | Low | Low | Should | Local feedback only |
| E2E polling/batch/log/readiness tuning | Reduce avoidable E2E harness overhead | Medium | Low | Low | Low | Should | Keeps assertions unchanged |
| Maven parallel reactor | Use Maven `-T` for independent modules | Low | Low | Low | Low | Could | Gain depends on module graph |
| CI matrix by runner group | Run runner groups as separate CI steps | High | Medium | Medium | Medium | Could | Alternative to Surefire forks |
| JUnit 5 Platform Engine | Enable future scenario-level parallelism | High | High | High | High | Could | Needs thread-safety work |
| E2E feature sharding | Parallelise E2E feature groups | High | High | High | High | Could | Only with safe resource isolation |

## Reading the Matrix

- **Must** items are the initial focus.
- **Should** items are low-risk improvements that compound with the main approach.
- **Could** items need service-specific evidence or technical review.
- Exact impact is `TBC` per service baseline unless listed in [References and Evidence](./05-references.md).
