# Technical Details

> This is a child page of [CI/CD and Test Pipeline Speed-Up Approach](./00-main-proposal.md).
> It contains implementation-level patterns, configuration examples, and architecture guidance applicable to any Java/Maven/Cucumber service.

---

## Typical Test Architecture (Pattern)

Services following the Cucumber BDD + Kafka Streams topology test pattern typically have this structure:

```
Feature File (e.g. event/SomeMapping.feature)
    │
    ▼
IntegrationTest.java (single Cucumber runner — JUnit 4)
    │
    ▼
StepDefinitions.java (glue code)
    │
    ├── @Before: creates TopologyTestDriver or test fixture
    ├── Given: builds input record from feature data table
    ├── When: drives record through topology or service under test
    ├── Then: reads output, counts records
    ├── And: asserts fields match expected values
    └── @After: cleans up test fixture
```

**Common characteristics:**
- Test driver is in-memory (no real broker, no network I/O)
- Each scenario is self-contained (fresh fixture per scenario)
- Tests are CPU-bound (serialisation, deserialisation, mapping logic)
- Spring context loaded once and shared across all scenarios in a single fork
- Static mutable fields in step definitions prevent naive in-process parallelisation

---

## Initial Target: SNS Command Adaptor

The first service to apply this approach is `fdp-cmd-adaptor-sns`. Its current state provides a concrete reference for the general patterns below.

| Metric | Value |
|--------|-------|
| Feature files | 66 |
| Test scenarios | 343 |
| Test runner classes | 1 |
| JVM forks | 1 (default) |
| Threads | 1 (sequential) |
| Topology test time | ~18 seconds |
| Feature directories | 6 (event, location, object, party, relationships, service) |
| Full CI stage | ~13m25s |
| Maven build step | ~1m17s |
| E2E integration step | ~9m34s |

---

## Phase 1: Quick Win Patterns

### Pattern: Test Logging Reduction

Create a `logback-test.xml` in `src/test/resources/` of the module under test:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Suppress verbose test step logging -->
    <logger name="${SERVICE_STEP_PACKAGE}" level="WARN"/>
    <logger name="org.apache.kafka" level="WARN"/>
    <logger name="io.confluent" level="WARN"/>

    <root level="WARN">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

Replace `${SERVICE_STEP_PACKAGE}` with the actual package of your step definitions (e.g. `uk.gov.ho.dacc.fdp.integration.steps`).

To restore verbose logging for local debugging:

```bash
mvn test -Dlogging.level.your.step.package=INFO
```

### Pattern: Maven Parallel Reactor

In your CI pipeline build command:

```bash
# Before
mvn clean install

# After
mvn clean install -T 1C
```

`-T 1C` means 1 thread per available CPU core. Only independent modules benefit; the main test module still runs after its upstream dependencies complete.

### Pattern: E2E Poll Duration Fix

A common bug across E2E harness code:

```java
// Bug: POLL_DURATION_MS is e.g. 500, so this means 500 SECONDS
consumer.poll(Duration.ofSeconds(POLL_DURATION_MS));

// Fix: use the correct unit
consumer.poll(Duration.ofMillis(POLL_DURATION_MS));
```

### Pattern: Per-Feature Timing Output

Add JSON report output to Cucumber runner plugin configuration:

```java
plugin = {
        "pretty",
        "summary",
        "json:target/cucumber-e2e.json",
        "html:target/cucumber.html"
}
```

Use the JSON report to identify which features are slowest and where time is being spent.

---

## Phase 2: Parallelisation Patterns

### Pattern: Runner Split by Feature Directory

Create one runner class per logical feature group. Example for a service with 6 feature directories:

```java
@RunWith(Cucumber.class)
@CucumberOptions(
        features = "src/test/resources/features/{group}",
        glue = "{your.step.package}",
        tags = "not @ignore"
)
public class {Group}IntegrationTest {}
```

Repeat for each feature directory. Remove the original single runner to prevent duplicate execution.

### Pattern: Surefire Fork Configuration

```xml
<properties>
    <argLine></argLine>
</properties>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${maven-surefire-plugin.version}</version>
    <configuration>
        <forkCount>4</forkCount>
        <reuseForks>true</reuseForks>
        <argLine>@{argLine} -XX:TieredStopAtLevel=1 -XX:+UseParallelGC</argLine>
    </configuration>
</plugin>
```

| Setting | Purpose |
|---------|---------|
| `forkCount=4` | Conservative starting point; benchmark 2, 4, 6 per service |
| `reuseForks=true` | JVM forks reused across test classes (reduces startup overhead) |
| `-XX:TieredStopAtLevel=1` | Stops JIT at C1 only — faster startup for short-lived test JVMs |
| `-XX:+UseParallelGC` | Throughput-oriented GC suited to batch workloads |
| `@{argLine}` | Placeholder populated by JaCoCo's `prepare-agent` goal |

### Pattern: Config Value Injection

Replace per-scenario file parsing with Spring injection:

```java
// Before: file I/O repeated per scenario
@Before
public void beforeScenario() {
    try (InputStream is = getClass().getClassLoader().getResourceAsStream("application.yml")) {
        props.load(is);
        configValue = props.getProperty("some.key");
    }
}

// After: injected once by Spring
@Value("${some.key:}")
private String configValue;

@Before
public void beforeScenario() {
    // Use configValue directly — no file I/O
}
```

### Pattern: JaCoCo Skip Profile

```xml
<properties>
    <jacoco.skip>false</jacoco.skip>
</properties>

<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <skip>${jacoco.skip}</skip>
    </configuration>
</plugin>
```

Local usage: `mvn test -Djacoco.skip=true`

CI continues with coverage enabled (default).

### Pattern: E2E Configurable Polling

```java
private static final int POLL_TIMEOUT_SECONDS =
        Integer.getInteger("e2e.poll.timeout.seconds", 120);
private static final int POLL_INTERVAL_MS =
        Integer.getInteger("e2e.poll.interval.ms", 250);
```

### Pattern: E2E Consumer Batch Size

```java
// Before: forces many small polls for high-record-count assertions
consumerConfig.put("max.poll.records", 1);

// After: allows Kafka to return more records per poll
consumerConfig.put("max.poll.records", "50");
```

### Pattern: E2E Cached Readiness

```java
private static final AtomicBoolean readinessConfirmed = new AtomicBoolean(false);

// In the readiness step:
if (readinessConfirmed.get()) {
    return; // already confirmed this run
}
// ... existing readiness polling ...
readinessConfirmed.set(true);
```

### Pattern: E2E Log Reduction

```xml
<logger name="${E2E_STEP_PACKAGE}" level="${E2E_STEP_LOG_LEVEL:-WARN}"/>
<logger name="org.apache.kafka" level="WARN"/>
<logger name="io.confluent" level="WARN"/>
<root level="WARN">
    <appender-ref ref="STDOUT"/>
</root>
```

---

## Phase 3: Structural Improvement Patterns

### Pattern: Static Field Removal

Convert shared mutable fields from `static` to instance in step definition classes:

```java
// Before (prevents in-process parallelisation)
protected static SomeRecord inputRecord;
protected static String testId;
static Map<String, Object> scenarioCache = new HashMap<>();

// After (safe for fork-based parallelism; prerequisite for JUnit 5)
protected SomeRecord inputRecord;
protected String testId;
Map<String, Object> scenarioCache = new HashMap<>();
```

Also update helper methods from `static` to instance.

### Pattern: Feature Assertion DSL Normalisation

Introduce a mapping layer between feature-file assertion keys and internal field paths:

```java
private static final Map<String, String> FIELD_ALIASES = Map.of(
        "semanticAlias", "Internal.path.to.field",
        "legacy.path.to.field", "Internal.path.to.field"
);

private static Map<String, String> normalizeExpectedFields(Map<String, String> fields) {
    return fields.entrySet().stream()
            .collect(Collectors.toMap(
                    entry -> FIELD_ALIASES.getOrDefault(entry.getKey(), entry.getKey()),
                    Map.Entry::getValue,
                    (left, right) -> right,
                    LinkedHashMap::new
            ));
}
```

This decouples feature files from internal model field names and prevents broad feature-file churn when underlying dependencies change.

### Pattern: CI Matrix by Runner Group

Run each runner as a separate CI step for per-group diagnostics:

```bash
mvn -pl ${MODULE} -Dtest=${Runner}IntegrationTest test
```

Only use this if CI diagnostics, per-group retry, or per-group ownership are more valuable than a single Maven test invocation.

---

## Phase 4: JUnit 5 Platform Engine Pattern

### Dependency Changes

```xml
<!-- REMOVE -->
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit</artifactId>
    <scope>test</scope>
</dependency>

<!-- ADD -->
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

### Parallel Configuration

Create `src/test/resources/junit-platform.properties`:

```properties
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=6
cucumber.execution.parallel.config.fixed.max-pool-size=8
cucumber.plugin=pretty, summary, html:target/cucumber-report.html
cucumber.glue={your.step.package}
cucumber.features=src/test/resources/features
cucumber.filter.tags=not @ignore
```

### JUnit 5 Suite Runner

Replace all runner classes with a single Suite:

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = "cucumber.glue", value = "{your.step.package}")
@ConfigurationParameter(key = "cucumber.filter.tags", value = "not @ignore")
public class IntegrationTest {}
```

### Thread-Safety Requirements

For in-process parallel execution, all scenario state must be instance-scoped:

```java
@CucumberContextConfiguration
@SpringBootTest
public class StepDefinitions {

    // Instance fields — isolated per scenario
    private SomeRecord inputRecord;
    private String testId;
    private Map<String, Object> scenarioCache = new HashMap<>();

    // Spring-managed beans (singletons, read-only during tests — thread-safe)
    @Autowired SomeService service;
    @Autowired SomeConfig config;
}
```

Additional audits:
- Any shared lookup tables or caches must be verified for concurrent safety
- ThreadLocal usage in service classes should be verified (often already safe)
- Helper methods must become instance methods

### Surefire for JUnit 5

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <forkCount>1</forkCount>
    </configuration>
</plugin>
```

Single fork is sufficient — parallelism is managed within the JVM by JUnit 5.

### Staged Rollout

1. Change dependencies and runner; keep `parallel.enabled=false` → verify all tests pass sequentially
2. Refactor static fields to instance fields → verify sequentially
3. Enable `parallel.enabled=true` with `parallelism=2` → run multiple times for flaky detection
4. Increase to target parallelism → monitor stability

---

## Expected Impact (SNS Command Adaptor Reference)

| Area | Current | Expected After Phase 2 | Estimated Saving |
|------|---------|------------------------|------------------|
| Topology tests | ~18s | ~7-10s | ~8-11s |
| Maven build step | ~1m17s | ~1m02s-1m07s | ~10-15s |
| E2E Integration step | ~9m34s | ~8m34s-9m14s | ~20-60s |
| Full CI stage | ~13m25s | ~12m10s-12m55s | ~30-75s |

These numbers are specific to the SNS Command Adaptor and serve as a reference. Actual impact per service will vary based on test count, feature distribution, current CI structure, and harness implementation.

---

## Applicability Checklist

Use this checklist to assess whether a service is a candidate for this approach:

| Criterion | Indicator |
|-----------|-----------|
| Single Cucumber runner | One `@RunWith(Cucumber.class)` test class covering all features |
| Sequential execution | No `forkCount`, no parallel configuration |
| Multiple feature directories | Features organised into logical groups (by domain, entity type, etc.) |
| CPU-bound tests | In-memory test drivers, no real network I/O during test execution |
| Independent scenarios | Each scenario creates/destroys its own test fixture |
| Verbose test logging | Large log output per test run (thousands of lines) |
| E2E polling harness | Kafka/HTTP polling loops with hardcoded timeouts |
| Growing feature count | Test time increasing as coverage expands |

Services matching 3+ criteria are good candidates. Services matching 5+ criteria are high-priority candidates.
