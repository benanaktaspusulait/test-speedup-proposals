# Technical Details

> Child page of CI/CD and Test Pipeline Speed-Up Approach.
> Keep this page as implementation notes only.

| Metadata | Value |
|----------|-------|
| Owner | TBC |
| Status | Draft |
| Created | 2026-06-09 |
| Last updated | 2026-06-09 |
| Last reviewed | 2026-06-09 |
| Labels | `technical-reference`, `ci-cd`, `test-pipeline` |

> **Implementation Note:** These examples are patterns and reference configurations, not drop-in solutions. Each service must confirm its own baseline, runner structure, CI capacity, and test isolation before applying changes. Do not copy configurations blindly — validate suitability per service.

---

## 1. Baseline Checklist

| Item | Capture |
|------|---------|
| Test command | local and CI command |
| Duration | unit/topology, E2E, full CI stage |
| Coverage | current gate and report location |
| Suite size | feature files, scenarios, runner classes |
| Parallelism | Surefire forks, JUnit threads, CI matrix, or none |
| Bottlenecks | slow groups, log volume, polling waits |

---

## 2. Quick Win Patterns

| Pattern | Implementation Note |
|---------|---------------------|
| Logging reduction | Set noisy test packages and client libraries to `WARN`; keep debug override |
| Maven reactor | Use `mvn clean install -T 1C` only where independent modules exist |
| Timing output | Enable Cucumber JSON/HTML reports |
| Config caching | Inject/cache config instead of reading files per scenario |
| Local JaCoCo skip | Allow local `-Djacoco.skip=true`; keep CI coverage enabled |
| E2E poll units | Use milliseconds values with `Duration.ofMillis(...)` |
| E2E polling config | Make timeout/interval system-property driven |
| E2E batch size | Increase `max.poll.records` where many records are expected |
| Readiness cache | Cache readiness only if valid for the whole run |

Minimal examples:

```xml
<logger name="<step.definition.package>" level="${TEST_STEP_LOG_LEVEL:-WARN}"/>
<logger name="org.apache.kafka" level="WARN"/>
```

```java
consumer.poll(Duration.ofMillis(POLL_DURATION_MS));
Integer.getInteger("e2e.poll.timeout.seconds", 120);
Integer.getInteger("e2e.poll.interval.ms", 250);
```

---

## 3. Fork-Based Parallelism

Split one large Cucumber runner into logical runner groups:

```java
@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/<group>",
        glue = "<step.definition.package>",
        tags = "not @ignore",
        plugin = {"summary", "json:target/cucumber-<group>.json"}
)
public class GroupIntegrationTest {
}
```

Configure Surefire forks:

```xml
<properties>
    <argLine></argLine>
    <test.fork.count>2</test.fork.count>
</properties>

<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <forkCount>${test.fork.count}</forkCount>
        <reuseForks>true</reuseForks>
        <argLine>@{argLine} -XX:TieredStopAtLevel=1 -XX:+UseParallelGC</argLine>
    </configuration>
</plugin>
```

Guardrails:

- remove or exclude the original all-features runner
- start with a conservative fork count
- compare scenario count before/after
- rebalance groups if one dominates wall time

**Failsafe note:** If integration tests are executed by Failsafe rather than Surefire, apply the same fork-count and argLine guardrails to the `maven-failsafe-plugin` configuration. The pattern is identical; only the plugin artifact ID changes.

---

## 4. Hardening Patterns

| Pattern | Why |
|---------|-----|
| Remove static scenario state | Required before safe same-JVM parallelism |
| Balance runner groups | Prevents one group from controlling wall time |
| Assertion aliases | Decouples feature files from internal model paths |
| CI matrix | Useful when group-level diagnostics/retry matter |

Static state example:

```java
// Before
protected static SomeRecord inputRecord;

// After
protected SomeRecord inputRecord;
```

---

## 5. JUnit 5 Option

Use this later if fork-based parallelism is not enough.

Required before enabling parallelism:

- migrate to `cucumber-junit-platform-engine`
- verify sequential JUnit 5 run first
- remove/protect static mutable state
- isolate test data, files, topics, and consumer groups
- start with low parallelism and increase after repeated stable runs

Example:

```properties
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=2
cucumber.plugin=summary, html:target/cucumber-report.html
cucumber.glue=<step.definition.package>
cucumber.features=src/test/resources/features
```

---

## 6. E2E Sharding Guardrails

Do not shard E2E tests unless each shard has isolated:

- topics and consumer groups
- database data or test IDs
- object storage prefixes
- external stub state
- environment variables and correlation IDs

Without isolation, keep E2E tests sequential and focus on polling/logging/readiness improvements.

---

## 7. Sensitive Data Guardrail

- Do not copy secrets, tokens, credentials, `.env` values, or restricted internal URLs into Confluence.
- Use placeholders such as `${SECRET_NAME}` or `<redacted>` if a value must be described.
- Reference sensitive files by path only; do not reproduce their contents.

---

## 8. Applicability Checklist

| Criterion | Yes/No |
|-----------|--------|
| Single large Cucumber/JUnit runner | TBC |
| Multiple feature groups | TBC |
| Independent scenarios | TBC |
| CPU/local-fixture-bound tests | TBC |
| Static mutable step state | TBC |
| Verbose green-run logs | TBC |
| Hardcoded E2E polling waits | TBC |
| CI worker has CPU/memory headroom | TBC |

