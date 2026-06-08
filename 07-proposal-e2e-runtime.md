# Proposal 6: E2E Runtime Optimization

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | 20-60s from quick harness fixes; larger gains only with isolated E2E sharding |
| Risk | Low for harness fixes; Medium for sharding |
| Effort | 1-2 days for quick fixes; 3-5 days for sharding |
| Reversibility | Easy for harness fixes |
| Prerequisites | Current E2E baseline with per-feature timing |

## Scope

This proposal intentionally excludes Docker cache, test profiler work, base Docker image work, Docker image build, Docker Compose startup, container readiness orchestration, and infrastructure changes. Those are handled separately.

The scope here is only the E2E test harness after the environment is available:

- Cucumber/Failsafe execution in `cmd-adaptor-sns-integration-tests`
- polling and timeout behavior in `SnsSteps`
- E2E log volume
- optional feature-level sharding, only if each shard can run with isolated topic suffixes or isolated infrastructure

## Current State

The current Drone view shows:

| Step | Observed Time |
|------|---------------|
| Full CI stage | ~13m25s |
| `mvn clean install` | ~01m17s |
| `Command Adaptor` | ~10m50s |
| `Integration Tests` | ~09m34s |

The E2E module has **7 feature files** and **8 scenarios**:

| Feature | Scenarios |
|---------|-----------|
| `E2E_Event.feature` | 1 |
| `E2E_Event-roro-unacc.feature` | 1 |
| `E2E_Location.feature` | 1 |
| `E2E_Object.feature` | 2 |
| `E2E_Party.feature` | 1 |
| `E2E_Service.feature` | 1 |
| `RunLog.feature` | 1 |

All E2E scenarios currently run through one JUnit 4 Cucumber runner:

```java
@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features",
        glue = "uk.gov.ho.dacc.fdp.steps",
        plugin = {"pretty", "summary", "uk.gov.ho.dacc.fdp.steps.SnsSteps", "html:target/cucumber.html"},
        tags = "not @ignore"
)
public class IntegrationTest {}
```

## Key Findings

### 1. RunLog Poll Duration Uses Seconds Instead of Milliseconds

`SnsSteps.pollForRunlogRecords()` uses:

```java
kafkaConsumerRunlogCmd.poll(Duration.ofSeconds(POLL_DURATION_MS));
```

`POLL_DURATION_MS` is `500`, so this means **500 seconds**, not 500 milliseconds. In a green run this may return quickly if records are already available, but on a slow or missing RunLog record it can turn into a very large wait.

This should be changed to:

```java
kafkaConsumerRunlogCmd.poll(Duration.ofMillis(POLL_DURATION_MS));
```

### 2. Polling Timeouts Are Very Wide

The E2E polling loop allows up to:

```java
MAX_RETRIES_GET_CONSUMER_RECORDS = 500
POLL_DURATION_MS = 500
```

That is up to **250 seconds per assertion path**. This is useful as a defensive upper bound, but it makes slow failures expensive and hides where time is being spent.

### 3. E2E Logs Are INFO by Default

`logback-test.xml` sets root logging to INFO, and `SnsSteps` logs every poll attempt and several full payloads. This is useful while developing a failing E2E test, but it is noisy and expensive for normal CI.

### 4. Feature-Level Parallelism Needs Isolation

The E2E step code keeps shared static producers, consumers, topic names, `testId`, and `scenarioCache`. It also uses one consumer group:

```java
consumerConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "e2e-testing");
```

Do not run these scenarios in parallel in the same JVM or against the same topic suffix. If E2E sharding is needed, each shard must use an isolated `FDP_APP_KAFKA_TOPIC_SUFFIX`, isolated consumer group, or separate infrastructure.

## Proposed Quick Fixes

### 6.1 Add Per-Feature Timing Baseline

Add JSON output so each scenario duration can be compared before and after changes:

```java
plugin = {
        "pretty",
        "summary",
        "json:target/cucumber-e2e.json",
        "uk.gov.ho.dacc.fdp.steps.SnsSteps",
        "html:target/cucumber.html"
}
```

Use the JSON report to identify whether the slowest time is in Event, RunLog, object snapshots, or readiness waits.

### 6.2 Fix RunLog Poll Unit

Change `Duration.ofSeconds(POLL_DURATION_MS)` to `Duration.ofMillis(POLL_DURATION_MS)`.

Also add an explicit failure message when the timeout is exhausted:

```java
assertEquals("RunLog records not emitted before timeout", number, runlogRecords.size());
```

### 6.3 Make Poll Timeouts Configurable

Keep CI conservative, but avoid hardcoding all values:

```java
private static final int POLL_TIMEOUT_SECONDS =
        Integer.getInteger("e2e.poll.timeout.seconds", 120);
private static final int POLL_INTERVAL_MS =
        Integer.getInteger("e2e.poll.interval.ms", 250);
```

Then calculate attempts from timeout and interval. This makes local tuning and CI experimentation possible without code changes.

### 6.4 Increase Consumer Batch Size

Current config uses:

```java
consumerConfig.put("max.poll.records", 1);
```

For scenarios expecting many records, such as 56 Event snapshots, this forces many small polls. Benchmark increasing this to `25` or `50`:

```java
consumerConfig.put("max.poll.records", "50");
```

Keep filtering by `testId`, but let Kafka return more records per poll.

### 6.5 Reduce E2E Test Log Volume

Change E2E test logging to WARN by default and enable `SnsSteps` DEBUG/INFO only when troubleshooting:

```xml
<logger name="uk.gov.ho.dacc.fdp.steps" level="${E2E_STEP_LOG_LEVEL:-WARN}"/>
<logger name="org.apache.kafka" level="WARN"/>
<logger name="io.confluent" level="WARN"/>
<root level="WARN">
    <appender-ref ref="STDOUT"/>
</root>
```

If environment-variable substitution is not available in the active Logback version, use a fixed WARN default and a local-only override file.

### 6.6 Cache Readiness Once Per Test Run

Several features call:

```gherkin
When Readiness health check is completed
```

Keep the step for readability, but make it a cached guard:

```java
private static final AtomicBoolean readinessConfirmed = new AtomicBoolean(false);
```

If readiness is already confirmed, return immediately. This avoids repeated HTTP polling and keeps the first failure point clear.

## Optional: E2E Feature Sharding

After quick fixes, consider splitting E2E runners by feature group:

```text
E2EEventIntegrationTest
E2EObjectIntegrationTest
E2EPartyIntegrationTest
E2ELocationIntegrationTest
E2EServiceIntegrationTest
E2ERunLogIntegrationTest
```

Only run these in parallel if each shard has isolation. Safe isolation options:

- separate `FDP_APP_KAFKA_TOPIC_SUFFIX` per shard
- separate consumer group per shard
- separate Docker Compose project or separate infrastructure per shard

Without isolation, use these runner classes only for targeted debugging, not parallel CI.

## Expected Impact

| Change | Expected Impact |
|--------|-----------------|
| Per-feature timing JSON | No direct speed-up; makes bottlenecks visible |
| RunLog poll unit fix | Prevents extreme slow/failure waits |
| Configurable polling + smaller interval | Faster slow-path detection; likely seconds to tens of seconds |
| Larger consumer batch size | Potentially faster high-count Event/Object snapshot checks |
| E2E logging to WARN | Lower CI log overhead and easier diagnostics |
| Cached readiness | Small green-run win; clearer readiness failures |
| Isolated feature sharding | Larger gain, but only if isolation is available |

For the current **~09m34s Integration Tests** step, the quick harness fixes should be expected to save roughly **20-60 seconds** in green CI runs and much more on slow/failing runs. Any larger reduction requires isolated E2E sharding or the separate Docker cache, test profiler, and base Docker image work.

## Verification

1. Capture current per-feature timings from `target/cucumber-e2e.json`.
2. Apply 6.2-6.6 one at a time.
3. Re-run the E2E container with the same profile:

   ```bash
   mvn -Pci-snapshot install
   ```

4. Confirm all 8 scenarios still pass.
5. Compare `Integration Tests` wall-clock time and per-feature timings.
6. Confirm failure cases fail faster and with clearer messages.
