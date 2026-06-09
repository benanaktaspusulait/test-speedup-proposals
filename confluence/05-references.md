# References and Evidence

> Child page of CI/CD and Test Pipeline Speed-Up Approach.
> Repository-specific details here are evidence only, not the general solution.

| Metadata | Value |
|----------|-------|
| Owner | TBC |
| Status | Draft |
| Created | 2026-06-09 |
| Last updated | 2026-06-09 |
| Last reviewed | 2026-06-09 |
| Labels | `proposal`, `ci-cd`, `test-pipeline` |

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

