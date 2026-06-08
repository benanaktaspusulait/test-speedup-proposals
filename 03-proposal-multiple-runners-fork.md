# Proposal 2: Multiple Test Runners with Surefire Fork Parallelization

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | 2-3x likely; up to 3-4x after event split/tuning |
| Risk | Low |
| Effort | 1-2 days |
| Reversibility | Easy (revert runner classes and surefire config) |
| Prerequisites | None beyond standard Maven/JUnit knowledge |

## Problem

All 343 test scenarios run through a single `IntegrationTest.java` Cucumber runner in a single JVM process. The CI runner typically has 4+ CPU cores available, but only one is used for test execution.

```
Current: 1 runner × 1 fork × 1 thread = sequential execution

┌──────────────────────────────────────────────────────┐
│ JVM Fork 1                                           │
│                                                      │
│ IntegrationTest.java                                 │
│   → event/CCon2Cons.feature                          │
│   → event/CCon2Cons-Multiple.feature                 │
│   → event/CNot2Cons.feature                          │
│   → ... (27 event features)                          │
│   → location/CCons-Add.feature                       │
│   → ... (13 location features)                       │
│   → ... (all 66 files sequentially)                  │
└──────────────────────────────────────────────────────┘
```

## Proposed Solution

Split the single Cucumber runner into separate runner classes, then configure Maven Surefire to run them in parallel JVM forks.

Start with **6 runner classes** (one per feature directory), then benchmark whether the `event/` group needs to be split further. The event directory has 27 feature files, so it is likely to become the long pole.

```
Proposed: 6 runners × 6 forks = parallel execution

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ JVM Fork 1      │  │ JVM Fork 2      │  │ JVM Fork 3      │
│                 │  │                 │  │                 │
│ EventIntTest    │  │ PartyIntTest    │  │ LocationIntTest │
│  → 27 features  │  │  → 13 features  │  │  → 13 features  │
└─────────────────┘  └─────────────────┘  └─────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ JVM Fork 4      │  │ JVM Fork 5      │  │ JVM Fork 6      │
│                 │  │                 │  │                 │
│ ObjectIntTest   │  │ ServiceIntTest  │  │ RelationshipsIT │
│  → 8 features   │  │  → 4 features   │  │  → 1 feature    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

Each fork runs in its own JVM, so there are no shared-state conflicts.

Keep the existing `plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps"` entry in each runner unless the `EventListener` behavior is split into a dedicated plugin class. In the current code, `IntegrationTestSteps` is both glue and a Cucumber plugin.

## Implementation

### Step 1: Create Runner Classes

Create 6 new files in `cmd-adaptor-sns/src/test/java/uk/gov/ho/dacc/fdp/integration/`:

**EventIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/event",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class EventIntegrationTest {}
```

**PartyIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/party",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class PartyIntegrationTest {}
```

**LocationIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/location",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class LocationIntegrationTest {}
```

**ObjectIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/object",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class ObjectIntegrationTest {}
```

**ServiceIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/service",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class ServiceIntegrationTest {}
```

**RelationshipsIntegrationTest.java**
```java
package uk.gov.ho.dacc.fdp.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/relationships",
        glue = "uk.gov.ho.dacc.fdp.integration.steps",
        plugin = "uk.gov.ho.dacc.fdp.integration.steps.IntegrationTestSteps",
        tags = "not @ignore"
)
public class RelationshipsIntegrationTest {}
```

### Step 2: Remove the Original Runner

Delete or rename `IntegrationTest.java` to prevent duplicate execution. All scenarios are now covered by the 6 new runners.

### Step 3: Configure Maven Surefire

Add the following to `cmd-adaptor-sns/pom.xml` inside the `<build><plugins>` section:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${maven-surefire-plugin.version}</version>
    <configuration>
        <forkCount>4</forkCount>
        <reuseForks>true</reuseForks>
    </configuration>
</plugin>
```

**Configuration explained:**
- `forkCount=4`: conservative starting point for CI runners; benchmark `2`, `4`, and `6`
- `reuseForks=true`: JVM forks are reused across test classes (reduces JVM startup overhead)

Alternative for runner-size-based tuning:

```xml
<forkCount>0.5C</forkCount>
```

This uses half a fork per available CPU core. Prefer a fixed value if the CI worker size is known and stable.

### Step 4: Remove Static Keywords (Recommended Cleanup)

While not strictly necessary (each fork is a separate JVM), removing `static` from shared fields in `IntegrationTestSteps.java` is best practice and prepares the code for future in-process parallelization:

```java
// Change these from static to instance fields:
protected StreamIngestRecord streamIngestRecord;       // was: protected static
protected String testId;                               // was: protected static
protected String mappingVersion;                       // was: protected static
Map<String, ? super Set<? extends SpecificRecordBase>> scenarioCache = new HashMap<>();  // was: static

private TestOutputTopic<PoleV2IdRecord, CmdPartyPoleRecord> partyOutputTopic;      // was: private static
private TestOutputTopic<PoleV2IdRecord, CmdObjectPoleRecord> objectOutputTopic;    // was: private static
private TestOutputTopic<PoleV2IdRecord, CmdEventPoleRecord> eventOutputTopic;      // was: private static
private TestOutputTopic<PoleV2IdRecord, CmdServicePoleRecord> serviceOutputTopic;  // was: private static
private TestOutputTopic<PoleV2IdRecord, CmdLocationPoleRecord> locationOutputTopic; // was: private static
```

Also update any methods that reference these fields (e.g., `checkPartyRecordsFromTestTopic()`, `checkEventRecordsFromTestTopic()`) to be non-static.

## Why This Is Safe

1. **Fork isolation:** Each fork runs in its own JVM process. Memory is not shared between forks. Even if fields remain `static`, there is no cross-fork interference.

2. **Scenario isolation:** The `@Before` hook creates a fresh `TopologyTestDriver` for every scenario. No state leaks between scenarios.

3. **No external resources:** TopologyTestDriver is fully in-memory. No ports, no files, no databases to conflict between forks.

4. **Spring context caching:** Each fork loads its own Spring context once. With `reuseForks=true`, this cost is paid once per fork instead of once per scenario. Benchmark the fork count because too many forks can duplicate startup enough to reduce the gain.

## Expected Timing

| Metric | Before | After |
|--------|--------|-------|
| Event tests (27 features) | ~7s | ~7s plus one fork's startup; largest fork determines total |
| Party tests (13 features) | ~4s | runs in parallel |
| Location tests (13 features) | ~3.5s | runs in parallel |
| Object tests (8 features) | ~2s | runs in parallel |
| Service tests (4 features) | ~1s | runs in parallel |
| Relationships tests (1 feature) | ~0.5s | runs in parallel |
| **Total wall time** | **~18s** | **~7-10s initially; lower if event is split and logging is reduced** |

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| CI runner lacks spare CPU | Medium | Reduced parallelism or slower forks | Benchmark `forkCount=2/4/6`; use `0.5C` only if worker size varies |
| Spring context startup per fork | Certain | +3-4s per fork overhead | `reuseForks=true` ensures this happens only once per fork |
| Increased memory usage | Low | OOM in CI | Each fork needs ~512MB heap; ensure CI runner has ≥4GB |
| Flaky test exposed by timing | Very low | Test failure | Investigate and fix — likely indicates a pre-existing bug |
| Report aggregation issues | Low | Unclear test reports | Surefire automatically aggregates reports from all forks |

## Verification

After implementation, verify:

1. `mvn -pl cmd-adaptor-sns test` produces the same number of passing tests (343)
2. Build log shows multiple forks:
   ```
   [INFO] Fork 1: ... EventIntegrationTest
   [INFO] Fork 2: ... PartyIntegrationTest
   [INFO] Fork 3: ... LocationIntegrationTest
   ```
3. Total test phase time is reduced from ~18s to roughly ~7-10s initially, with lower times expected after event sharding and logging reduction
4. All Surefire reports are generated in `target/surefire-reports/`

## Future Considerations

- If event tests dominate the wall time, split them into sub-runners (e.g., `EventAssociationIntegrationTest`, `EventMovementIntegrationTest`)
- The `forkCount` can be tuned based on CI runner capacity
- This approach is a good stepping stone toward a future JUnit 5 migration, but do not run forked class parallelism and same-JVM Cucumber scenario parallelism at the same time
