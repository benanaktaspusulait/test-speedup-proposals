# Current State Analysis

## Test Architecture Overview

The `fdp-cmd-adaptor-sns` project uses Cucumber BDD tests to validate Kafka Streams topology transformations. These tests verify that SNS input data is correctly mapped into POLE domain records (Party, Object, Location, Event, Service).

### How Topology Tests Work

```
Feature File (e.g. event/CCon2Cons.feature)
    │
    ▼
IntegrationTest.java (single Cucumber runner — JUnit 4)
    │
    ▼
IntegrationTestSteps.java (step definitions)
    │
    ├── @Before: creates TopologyTestDriver (in-memory Kafka)
    ├── Given: builds StreamIngestRecord from feature data table
    ├── When: pipes record into TopologyTestDriver input topic
    ├── Then: reads output topics, counts records
    ├── And: asserts record fields match expected values
    └── @After: closes TopologyTestDriver
```

### Key Characteristics

- **TopologyTestDriver** is an in-memory Kafka Streams test utility — no real broker, no network I/O
- Each scenario is self-contained: fresh driver created in `@Before`, closed in `@After`
- Tests are CPU-bound (serialization, deserialization, mapping logic), not I/O-bound
- Spring context is loaded once and shared across all scenarios

## Current Numbers

| Metric | Value |
|--------|-------|
| Feature files | 66 |
| Test scenarios | 343 |
| Test runner classes | 1 (`IntegrationTest.java`) |
| JVM forks | 1 (default) |
| Threads | 1 (sequential) |
| Execution time | ~18 seconds |
| Feature directories | 6 (event, location, object, party, relationships, service) |

## Why Tests Are Slow

### 1. Single Runner, Single Fork

All 66 feature files are executed through one `IntegrationTest.java`:

```java
@RunWith(Cucumber.class)
@CucumberOptions(
    features = "src/test/resources/features",
    glue = "uk.gov.ho.dacc.fdp.integration.steps",
    tags = "not @ignore"
)
public class IntegrationTest {}
```

Maven Surefire runs this as a single test class in a single JVM fork. There is no `forkCount` or `parallel` configuration.

### 2. No Cucumber Parallel Execution

There is no `junit-platform.properties` or `cucumber.properties` file. The Cucumber JUnit 4 runner does not support parallel scenario execution natively.

### 3. Linear Growth

Every new feature file added to the project increases pipeline time proportionally. With 27 event features alone, the event tests dominate execution time.

### 4. Static Mutable State

The step definitions class uses `static` fields:

```java
protected static StreamIngestRecord streamIngestRecord;
protected static String testId;
protected static String mappingVersion;
static Map<String, ? super Set<? extends SpecificRecordBase>> scenarioCache = new HashMap<>();
private static TestOutputTopic<...> partyOutputTopic;
```

While this doesn't cause bugs in sequential execution, it prevents naive parallelization within a single JVM without refactoring.

### 5. Spring Context Overhead

The `@SpringBootTest` annotation loads the full Spring application context. This happens once per JVM fork (cached), adding ~3-4 seconds to the first scenario. In a single-fork setup this cost is paid once, but it means the per-scenario cost is dominated by topology execution rather than startup.

## Pipeline Context

The topology tests run during `mvn clean install` in the CI pipeline:

```
mvn clean install
├── cmd-adaptor-sns-avro         (build only, no tests)
├── cmd-adaptor-sns-common       (build only, no tests)
├── cmd-adaptor-sns-test-common  (build only, no tests)
├── cmd-adaptor-sns              ← 343 topology tests run here (~18s)
└── cmd-adaptor-sns-integration-tests (skipped in this phase)
```

The current Drone view shows the full CI stage at about **13m25s**, with `mvn clean install` at about **01m17s**. The 18 seconds for topology tests is therefore a meaningful part of the Maven step, but not the dominant part of the full CI stage.

The Docker/cache/test-profiler side of the longer E2E path, especially base Docker image work and the `Command Adaptor` service lifecycle, is handled separately and is intentionally outside this proposal set. The E2E test harness itself is covered by Proposal 6.

## Conclusion

The topology tests are well-suited for parallelization because:
- They are CPU-bound (no network I/O)
- Each scenario is independent (fresh TopologyTestDriver per scenario)
- The feature files are already organized into logical groups (6 directories)
- TopologyTestDriver has no port conflicts across JVM forks

The main barrier is the single-runner architecture and the use of static fields in step definitions. Both are addressable with moderate refactoring effort.
