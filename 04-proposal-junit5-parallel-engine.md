# Proposal 3: JUnit 5 Platform Engine with Cucumber Parallel Execution

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | 3-5x potential after thread-safety refactor |
| Risk | Medium-High |
| Effort | 3-5 days |
| Reversibility | Moderate (dependency and runner changes) |
| Prerequisites | Understanding of ThreadLocal patterns and Cucumber JUnit 5 integration |

## Problem

Proposal 2 (fork-based parallelism) achieves good results but has limitations:
- Each fork loads its own Spring context (~3-4s overhead × 6 forks)
- Memory usage scales linearly with fork count
- The parallelism granularity is at the runner class level, not the scenario level

The ideal solution is to run individual **scenarios in parallel within a single JVM**, sharing one Spring context across all threads. This eliminates redundant context loading and allows finer-grained parallelism. In this repo, the extra risk is shared static topology/lookup state, so this should be treated as a staged migration rather than a quick switch.

## Proposed Solution

Migrate from Cucumber's JUnit 4 runner to the **JUnit 5 Platform Engine**, which natively supports parallel scenario execution within a single JVM.

```
Current (JUnit 4):
┌──────────────────────────────────┐
│ Single JVM                       │
│ Single Thread                    │
│                                  │
│ Scenario 1 → Scenario 2 → ...   │
└──────────────────────────────────┘

Proposed (JUnit 5 Platform Engine):
┌──────────────────────────────────┐
│ Single JVM, Shared Spring Context│
│                                  │
│ Thread 1: Scenario 1, 5, 9...    │
│ Thread 2: Scenario 2, 6, 10...   │
│ Thread 3: Scenario 3, 7, 11...   │
│ Thread 4: Scenario 4, 8, 12...   │
└──────────────────────────────────┘
```

## Implementation

### Step 1: Update Dependencies

In `cmd-adaptor-sns/pom.xml`, replace the Cucumber JUnit 4 dependencies:

```xml
<!-- REMOVE these -->
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
</dependency>

<!-- ADD these -->
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-suite</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <scope>test</scope>
</dependency>
```

**Note:** Keep `cucumber-java` and `cucumber-spring` — they are still needed.

### Step 2: Create Parallel Configuration

Create `cmd-adaptor-sns/src/test/resources/junit-platform.properties`:

```properties
# Enable parallel execution
cucumber.execution.parallel.enabled=true

# Use fixed thread pool
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=6
cucumber.execution.parallel.config.fixed.max-pool-size=8

# Cucumber settings
cucumber.plugin=pretty, summary, html:target/cucumber-report.html
cucumber.glue=uk.gov.ho.dacc.fdp.integration.steps
cucumber.features=src/test/resources/features
cucumber.filter.tags=not @ignore

# Cucumber owns scenario scheduling; JUnit Jupiter parallel settings are not required
```

### Step 3: Replace the Runner Class

Replace `IntegrationTest.java` with a JUnit 5 Suite runner:

```java
package uk.gov.ho.dacc.fdp.integration;

import org.junit.platform.suite.api.*;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = "cucumber.glue", value = "uk.gov.ho.dacc.fdp.integration.steps")
@ConfigurationParameter(key = "cucumber.filter.tags", value = "not @ignore")
public class IntegrationTest {}
```

### Step 4: Remove Remaining JUnit 4 Coupling

The current glue class is annotated with `@RunWith(SpringRunner.class)` and imports JUnit 4 assertions. That is compatible with the current JUnit 4 runner, but it becomes a migration blocker once `cucumber-junit` and `junit-vintage-engine` are removed.

Update `IntegrationTestSteps` as part of the migration:

- remove `@RunWith(SpringRunner.class)` and the `org.junit.runner.RunWith` import
- keep `@CucumberContextConfiguration`, `@SpringBootTest`, `@AutoConfigureMockMvc`, and `@ActiveProfiles("test")`
- replace `org.junit.Assert` / `assertEquals` usage with JUnit Jupiter assertions or AssertJ
- decide whether the `EventListener` implementation is still needed; if it is, register it through `cucumber.plugin` or split it into a dedicated plugin class

### Step 5: Refactor Step Definitions for Thread Safety

This is the most critical step. Since scenarios now run concurrently in the same JVM, shared mutable state must be isolated per thread.

**5a. Replace static fields with scenario-isolated state:**

Do not rely on static fields under same-JVM parallel execution. Convert scenario data, output topics, and caches to instance fields, and verify the Cucumber-Spring object lifecycle. If the glue class is managed as a Spring bean, use scenario scope for mutable per-scenario state.

```java
@Slf4j
@AutoConfigureMockMvc
@ActiveProfiles("test")
@SpringBootTest
@CucumberContextConfiguration
public class IntegrationTestSteps implements EventListener {

    // Scenario-isolated fields
    private StreamIngestRecord streamIngestRecord;
    private String testId;
    private String mappingVersion;
    private Map<String, ? super Set<? extends SpecificRecordBase>> scenarioCache = new HashMap<>();
    private Map<String, String> backgroundDataTemplate;

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, StreamIngestRecord> testInputTopic;
    private TestOutputTopic<PoleV2IdRecord, CmdPartyPoleRecord> partyOutputTopic;
    private TestOutputTopic<PoleV2IdRecord, CmdObjectPoleRecord> objectOutputTopic;
    private TestOutputTopic<PoleV2IdRecord, CmdEventPoleRecord> eventOutputTopic;
    private TestOutputTopic<PoleV2IdRecord, CmdServicePoleRecord> serviceOutputTopic;
    private TestOutputTopic<PoleV2IdRecord, CmdLocationPoleRecord> locationOutputTopic;

    // Spring-managed beans are thread-safe (they are singletons, only read during tests)
    @Autowired
    CmdAdaptorService service;

    @Autowired
    LocalKafkaSerdeConfigTest config;

    @Autowired
    KafkaSerdeConfigTest configTest;

    @Autowired
    KafkaTopics kafkaTopics;

    // ... rest of the class
}
```

**5b. Ensure topology and lookup initialization are thread-safe:**

The `buildTopology()` method uses a `ThreadLocal<StreamsBuilder>`:

```java
// Already exists in the codebase:
CmdAdaptorService.streamsBuilderThreadLocal.set(new StreamsBuilder());
```

This is already thread-safe. Each thread gets its own `StreamsBuilder`. Verify that `buildTopology()` does not write to any shared mutable state.

Also audit `BaseEoriLookup`: it currently keeps `globalKTable` and `keyValueStore` in static fields. That is safe enough for the current sequential runner, but it is not safe to assume under scenario-level parallelism. Either make lookup topology state scenario-local, guard initialization carefully, or prove that it is read-only and compatible with concurrent topologies.

**5c. Update helper methods to be non-static:**

Change methods like `checkPartyRecordsFromTestTopic()`, `checkEventRecordsFromTestTopic()` from `static` to instance methods:

```java
// Before
protected static Set<PartyRecord> checkPartyRecordsFromTestTopic() { ... }

// After
protected Set<PartyRecord> checkPartyRecordsFromTestTopic() { ... }
```

### Step 6: Update Surefire Configuration

Ensure Surefire uses the JUnit Platform provider:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${maven-surefire-plugin.version}</version>
    <configuration>
        <!-- Single fork is sufficient — parallelism is within the JVM -->
        <forkCount>1</forkCount>
    </configuration>
</plugin>
```

## Advantages Over Proposal 2

| Aspect | Proposal 2 (Forks) | Proposal 3 (JUnit 5) |
|--------|--------------------|-----------------------|
| Spring context loads | 6 (one per fork) | 1 (shared) |
| Memory usage | ~3GB (6 × 512MB) | ~800MB |
| Parallelism granularity | Runner class level | Scenario level |
| Startup overhead | ~20s (6 × 3-4s) | ~4s (once) |
| Load balancing | Fixed (uneven feature distribution) | Dynamic (JUnit schedules scenarios) |
| Configuration complexity | Simple | Moderate |

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Thread-safety bugs in step definitions | Medium | Test failures | Thorough refactoring; run tests sequentially first, then enable parallel |
| Thread-safety bugs in generated topology or lookup state | Medium | Test failures | Audit `CmdAdaptorService`, `BaseEoriLookup`, and generated mapping state before enabling parallel |
| Cucumber-Spring integration issues | Low | Setup failures | Well-documented by Cucumber team; many examples available |
| Schema Registry mock conflicts | Low | Serialization errors | Each scenario uses its own TopologyTestDriver which has its own mock registry |
| Report format changes | Low | CI integration issues | Configure `cucumber.plugin` to output standard formats |

## Staged Rollout

To minimize risk, implement in stages:

1. **Stage 1:** Change dependencies and runner class, but keep `cucumber.execution.parallel.enabled=false`
   - Verify all tests pass with JUnit 5 engine in sequential mode
   
2. **Stage 2:** Refactor static fields to instance fields
   - Verify all tests still pass sequentially

3. **Stage 3:** Enable `cucumber.execution.parallel.enabled=true` with `parallelism=2`
   - Run multiple times to check for flaky failures
   
4. **Stage 4:** Increase `parallelism` to 4, then 6
   - Monitor execution time and stability

## Expected Timing

| Phase | Time |
|-------|------|
| Spring context startup | ~4s (once) |
| 343 scenarios ÷ 6 threads | ~3s |
| **Total** | **~5-8s if thread-safety work is complete and logging is reduced** |

## Verification

1. All 343 scenarios pass
2. No `ConcurrentModificationException` or `NullPointerException` in logs
3. Running tests 10 times produces consistent results (no flaky tests)
4. `target/cucumber-report.html` shows correct parallel execution
5. Total test time improves materially without increasing flakiness; exact target should be based on the branch-pipeline baseline

## When to Choose This Over Proposal 2

- When the team has capacity for a larger refactoring effort
- When CI runner memory is constrained (Proposal 3 uses less memory)
- When feature files grow beyond 100+ and even forked parallelism isn't fast enough
- When you want scenario-level load balancing (short and long scenarios distributed optimally)
