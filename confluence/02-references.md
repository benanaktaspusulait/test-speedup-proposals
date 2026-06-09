# References

> This is a child page of [CI/CD and Test Pipeline Speed-Up Approach](./00-main-proposal.md).

---

## Initial Target Service

| Attribute | Value |
|-----------|-------|
| Repository | `dacc-aws/fdp-cmd-adaptor-sns` |
| Branch | `develop` |
| Version | 2.3.2-SNAPSHOT |

---

## Source Proposal Documents

These documents contain the original detailed analysis for the SNS Command Adaptor and were used to produce this approach:

| Document | Description |
|----------|-------------|
| `01-current-state.md` | Analysis of the current test architecture, bottlenecks, and why tests are slow |
| `02-proposal-maven-parallel-build.md` | Quick win: parallel module builds with Maven `-T` flag |
| `03-proposal-multiple-runners-fork.md` | Main proposal: split runners + Surefire forkCount for 2-3x improvement |
| `04-proposal-junit5-parallel-engine.md` | Long-term: migrate to JUnit 5 Platform Engine with scenario-level parallelism |
| `05-proposal-quick-wins.md` | Non-parallel optimisations: logging, fixture caching, JaCoCo profile, JVM tuning, feature assertion DSL |
| `06-proposal-ci-matrix-sharding.md` | CI-level alternative: split runner groups into separate parallel CI steps |
| `07-proposal-e2e-runtime.md` | E2E test harness optimisations excluding Docker/container work |

TODO: verify link — these documents live in a GitLab repository. Add absolute GitLab links once the repository URL and branch are confirmed for external readers.

---

## Key Files (SNS Command Adaptor)

| File / Path | Purpose |
|-------------|---------|
| `cmd-adaptor-sns/src/test/java/.../IntegrationTest.java` | Current single Cucumber runner (topology tests) |
| `cmd-adaptor-sns/src/test/java/.../steps/IntegrationTestSteps.java` | Step definitions and test setup |
| `cmd-adaptor-sns/pom.xml` | Module POM with Surefire/JaCoCo configuration |
| `cmd-adaptor-sns-integration-tests/src/test/java/.../IntegrationTest.java` | E2E Cucumber runner |
| `cmd-adaptor-sns-integration-tests/src/test/java/.../steps/SnsSteps.java` | E2E step definitions |
| CI pipeline configuration | Managed centrally (TODO: verify link) |

---

## CI Pipeline Baseline (SNS Command Adaptor)

| Step | Observed Time | Notes |
|------|---------------|-------|
| Full CI stage | ~13m25s | Total elapsed |
| `mvn clean install` | ~1m17s | Includes topology tests (~18s) |
| Docker build + startup | ~10m50s | Out of scope for this approach |
| E2E Integration Tests | ~9m34s | 8 scenarios, 7 feature files |

---

## External Documentation

| Resource | Link |
|----------|------|
| Maven Surefire Plugin | [https://maven.apache.org/surefire/maven-surefire-plugin/](https://maven.apache.org/surefire/maven-surefire-plugin/) |
| Cucumber JUnit Platform Engine | [https://github.com/cucumber/cucumber-jvm/tree/main/cucumber-junit-platform-engine](https://github.com/cucumber/cucumber-jvm/tree/main/cucumber-junit-platform-engine) |
| JUnit 5 Parallel Execution | [https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution) |
| Logback Configuration | [https://logback.qos.ch/manual/configuration.html](https://logback.qos.ch/manual/configuration.html) |
| TopologyTestDriver (Kafka Streams) | [https://kafka.apache.org/documentation/streams/developer-guide/testing.html](https://kafka.apache.org/documentation/streams/developer-guide/testing.html) |
| Cucumber Parallel Execution Guide | [https://cucumber.io/docs/guides/parallel-execution/](https://cucumber.io/docs/guides/parallel-execution/) |

---

## Related Work (Out of Scope)

The following are handled separately and are not part of this approach:

- Docker cache and base Docker image optimisation
- Test profiler infrastructure
- Docker Compose startup and container readiness orchestration
- Service lifecycle changes (application startup/shutdown)
- Message broker / data store startup optimisation
- Security scanning (e.g. Trivy) duration

---

## Candidate Services for Future Application

Once validated on the initial target, this approach can be assessed against other services using the Applicability Checklist in the [Technical Details](./01-technical-details.md) page. Potential candidates should be prioritised based on:

1. Current CI pipeline duration
2. Test scenario count and growth rate
3. Similarity to the initial target architecture
4. Team capacity for implementation
