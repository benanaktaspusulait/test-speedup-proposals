# Proposal 4: Quick Wins (Logging, Fixture Caching, Build Tuning)

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | Significant (logging alone), compounding with other proposals |
| Risk | Very low |
| Effort | Half a day total |
| Reversibility | Instant per change |
| Prerequisites | None |

These are independent, low-risk optimizations that reduce per-scenario and per-build overhead. They are orthogonal to parallelization — they make each scenario cheaper, which compounds with the parallel proposals (1, 2, 3).

---

## 4.1 Reduce Test Logging Verbosity (Highest Impact Quick Win)

### Problem

A single test run produces a **76,888-line log file**. The step definitions log the full content of every record at INFO level:

```java
// IntegrationTestSteps.java
log.info(landingRecord.toString());          // full input record dump per scenario
log.info("Value: {}", testEventRecord);       // full output record dump per assertion
log.info("Value: {}", testPartyRecord);
log.info("Value: {}", testLocationRecord);
// ... and so on for every pole type
```

With 343 scenarios, each producing multiple full-record JSON dumps, this generates an enormous volume of console output. Console I/O is **synchronous and blocking** — every log line blocks the test thread until written. This is pure overhead that adds nothing during normal (green) test runs.

The main module (`cmd-adaptor-sns`) uses Spring Boot logging, so the practical test-time control point is Logback or Spring's `logging.level.*` properties. The E2E module already has a `logback-test.xml`; the topology-test module does not.

### Solution

Add a Logback test configuration file to control the test log level.

Create `cmd-adaptor-sns/src/test/resources/logback-test.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="uk.gov.ho.dacc.fdp.integration.steps" level="WARN"/>
    <logger name="uk.gov.ho.dacc.fdp.assertions" level="WARN"/>
    <logger name="org.apache.kafka" level="WARN"/>
    <logger name="io.confluent" level="WARN"/>

    <root level="WARN">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

### Why This Helps

- Eliminates tens of thousands of synchronous console writes per test run
- Record dumps are only useful when debugging a failure — they add nothing to passing runs
- When a test fails, Cucumber's assertion output and the failure stack trace already provide the needed context

### Trade-off

If developers rely on the INFO dumps to debug locally, override the step package log level for debug sessions:

```bash
mvn -pl cmd-adaptor-sns test -Dlogging.level.uk.gov.ho.dacc.fdp.integration.steps=INFO
```

---

## 4.2 Read Mapping Version from Spring Config

### Problem

The `@Before` hook runs before every scenario (343 times) and reads + parses `application.yml` from the classpath each time, only to extract the mapping version:

```java
@Before
public void beforeScenario() {
    // ... topology setup ...
    try (InputStream inputStream = getClass().getClassLoader().getResourceAsStream("application.yml")) {
        if (inputStream != null) {
            props.load(inputStream);
            mappingVersion = props.getProperty("mapping").replaceAll("(?i)-SNAPSHOT", "");
        }
    } catch (IOException e) {
        mappingVersion = null;
    }
}
```

The mapping version never changes between scenarios. Reading and parsing a file 343 times is wasted I/O.

### Solution

Prefer reading this value from Spring configuration rather than parsing YAML manually. `application.yml` already exposes the filtered value as `build.mapping`.

Option A: inject the value:

```java
@Value("${build.mapping:}")
private String configuredMappingVersion;
```

Then normalize it in `@Before` without opening a file:

```java
mappingVersion = StringUtils.isBlank(configuredMappingVersion)
        ? null
        : configuredMappingVersion.replaceAll("(?i)-SNAPSHOT", "");
```

Option B: if keeping a static cache, initialize it lazily from Spring's `Environment` the first time `@Before` runs:

```java
@Autowired
private Environment environment;

private static volatile String cachedMappingVersion;

@Before
public void beforeScenario() {
    if (cachedMappingVersion == null) {
        String value = environment.getProperty("build.mapping");
        cachedMappingVersion = StringUtils.isBlank(value)
                ? null
                : value.replaceAll("(?i)-SNAPSHOT", "");
    }
    mappingVersion = cachedMappingVersion;

    // ... existing topology setup ...
}
```

Then remove the file-reading block from `beforeScenario()`.

### Note

If migrating to scenario-level parallelism (Proposal 3), keep `mappingVersion` as a read-only static value initialized once — it is safe to share across threads because it is never mutated after initialization.

---

## 4.3 Make JaCoCo Optional for Local Runs

### Problem

The `jacoco-maven-plugin` instruments bytecode to measure code coverage. This instrumentation adds overhead to every test execution, including fast local iteration cycles where coverage is not needed.

### Solution

Wrap JaCoCo in a profile that is active by default (so CI keeps coverage) but can be skipped locally.

In `cmd-adaptor-sns/pom.xml`, add a property and a skip flag:

```xml
<properties>
    <jacoco.skip>false</jacoco.skip>
</properties>

<!-- in the jacoco plugin configuration -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <configuration>
        <skip>${jacoco.skip}</skip>
    </configuration>
    <!-- existing executions -->
</plugin>
```

Then for fast local runs:

```bash
mvn test -Djacoco.skip=true
```

CI continues to run with coverage enabled (default). This keeps coverage gates intact while speeding up the local feedback loop.

---

## 4.4 Avoid `clean` for Local Iterative Runs

### Problem

`mvn clean install` deletes `target/` and rebuilds everything from scratch every time, including:
- Re-running the `fdp-cmd-adaptor-mapping-generator` plugin (code generation in `generate-sources`)
- Recompiling all modules
- Re-copying dependencies

For local development where only test code or feature files changed, this is wasteful.

### Solution

For local iteration, skip `clean` and target only the relevant module:

```bash
# Run only the topology tests in the main module, using incremental compilation
mvn test -pl cmd-adaptor-sns

# Or if only feature files changed (no recompilation needed)
mvn test -pl cmd-adaptor-sns -o   # offline, no dependency checks
```

Maven's incremental compilation reuses already-compiled classes and skips code generation if sources haven't changed.

### Note

This is a **local developer practice**, not a pipeline change. CI must continue to use `mvn clean install` for reproducible, clean builds.

---

## 4.5 Test JVM Startup Tuning

### Problem

Test JVMs are short-lived. The HotSpot C2 JIT compiler optimizes long-running code, but for short test runs the optimization cost outweighs the benefit. This matters more when running multiple forks (Proposal 2), where JVM startup is multiplied.

### Solution

Add JVM tuning flags to the Surefire configuration. This repo already runs JaCoCo's `prepare-agent`, so use the JaCoCo-safe `@{argLine}` form:

```xml
<properties>
    <argLine></argLine>
</properties>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${maven-surefire-plugin.version}</version>
    <configuration>
        <argLine>@{argLine} -XX:TieredStopAtLevel=1 -XX:+UseParallelGC</argLine>
    </configuration>
</plugin>
```

- `-XX:TieredStopAtLevel=1`: Stops JIT at tier 1 (C1 only), skipping expensive C2 compilation — faster startup for short-lived test JVMs
- `-XX:+UseParallelGC`: Uses a throughput-oriented garbage collector suited to batch workloads like test runs

### Note

The `@{argLine}` placeholder is populated by JaCoCo's `prepare-agent` goal. The empty default property keeps non-JaCoCo runs from failing when the placeholder is evaluated.

---

## Combined Impact

These quick wins are cumulative and stack with the parallelization proposals:

| Change | Effect |
|--------|--------|
| 4.1 Logging to WARN | Removes ~76k synchronous console writes per run |
| 4.2 Read mapping version from Spring config | Removes 342 redundant file reads/parses |
| 4.3 JaCoCo profile | Removes instrumentation overhead in local runs |
| 4.4 No `clean` locally | Skips code generation + full recompilation |
| 4.5 JVM tuning | Faster fork startup, especially with Proposal 2 |

## Recommended Order

1. **4.1 (logging)** first — highest impact, zero code logic change, just a properties file
2. **4.2 (mapping version from Spring config)** — small code change, removes per-scenario I/O
3. **4.5 (JVM tuning)** — apply alongside Proposal 2 (forkCount)
4. **4.3 and 4.4** — developer-experience improvements for local iteration

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Losing useful debug logs (4.1) | Medium | Keep step package at INFO, or override via system property when debugging |
| Stale build artifacts (4.4) | Low | This is local-only; CI always uses `clean` |
| argLine conflict with JaCoCo (4.5) | Medium | Use the `@{argLine}` placeholder as shown above |
