# References and Evidence

> Child page of [CI/CD and Test Pipeline Speed-Up Approach](./00-main-proposal.md).
> Repository-specific details here are evidence only, not the general solution.

---

## Source Documents

| Document | Purpose |
|----------|---------|
| `01-current-state.md` | current-state analysis |
| `02-proposal-maven-parallel-build.md` | Maven `-T` proposal |
| `03-proposal-multiple-runners-fork.md` | runner split + Surefire forks |
| `04-proposal-junit5-parallel-engine.md` | JUnit 5 migration path |
| `05-proposal-quick-wins.md` | logging, caching, JaCoCo, JVM tuning |
| `06-proposal-ci-matrix-sharding.md` | CI matrix option |
| `07-proposal-e2e-runtime.md` | E2E harness improvements |

TODO: replace local paths with Confluence/Git links when publishing location is confirmed.

---

## Example Baseline Evidence

| Metric | Observed Value |
|--------|----------------|
| Feature files | 66 |
| Scenarios | 343 |
| Cucumber runner classes | 1 |
| JVM forks | 1 |
| Execution model | Sequential |
| Feature groups | 6 logical directories |
| Topology/unit-style test time | ~18s |
| Maven build step | ~1m17s |
| Full CI stage | ~13m25s |
| E2E integration step | ~9m34s |

Each service must capture its own baseline before applying the approach.

---

## Files to Inspect Per Service

| File Type | Check |
|-----------|-------|
| Cucumber runners | one large runner vs grouped runners |
| Step definitions | static state, repeated setup, config reads |
| Maven POMs | Surefire/Failsafe, JaCoCo, JVM args |
| Test resources | features, logging, Cucumber/JUnit config |
| E2E steps | polling, readiness, batch size, logs |
| CI config | commands, matrix support, worker ownership |

---

## External Documentation

| Resource | Link |
|----------|------|
| Maven Surefire Plugin | [https://maven.apache.org/surefire/maven-surefire-plugin/](https://maven.apache.org/surefire/maven-surefire-plugin/) |
| Cucumber JUnit Platform Engine | [https://github.com/cucumber/cucumber-jvm/tree/main/cucumber-junit-platform-engine](https://github.com/cucumber/cucumber-jvm/tree/main/cucumber-junit-platform-engine) |
| JUnit 5 Parallel Execution | [https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution) |
| Logback Configuration | [https://logback.qos.ch/manual/configuration.html](https://logback.qos.ch/manual/configuration.html) |
| Kafka Streams TopologyTestDriver | [https://kafka.apache.org/documentation/streams/developer-guide/testing.html](https://kafka.apache.org/documentation/streams/developer-guide/testing.html) |
| Cucumber Parallel Execution Guide | [https://cucumber.io/docs/guides/parallel-execution/](https://cucumber.io/docs/guides/parallel-execution/) |

---

## Out of Scope Evidence

The source material treats these as separate workstreams:

- Docker cache and base image optimisation
- test-profiler infrastructure
- Docker Compose startup and container readiness orchestration
- service lifecycle changes
- broker/database startup optimisation
- security scanning duration
