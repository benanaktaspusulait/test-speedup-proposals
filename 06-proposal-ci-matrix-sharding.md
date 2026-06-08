# Proposal 5: CI Matrix by Runner Group

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | 2-4x topology-test wall-clock reduction |
| Risk | Low to Medium |
| Effort | 1-2 days |
| Reversibility | Easy if implemented as separate CI steps |
| Prerequisites | Runner classes from Proposal 2, RepoSync/Drone support |

## Problem

Proposal 2 uses Maven Surefire forks inside one Maven invocation. That is simple, but all parallelism is still coordinated by one Maven process and one CI step. When one runner group fails, the build log can still be noisy, and tuning individual groups is awkward.

The current feature distribution is uneven:

| Group | Feature Files |
|-------|---------------|
| event | 27 |
| location | 13 |
| party | 13 |
| object | 8 |
| service | 4 |
| relationships | 1 |

The event group is likely to dominate wall time even after runner splitting.

## Proposed Solution

Split the Cucumber suite into runner classes as in Proposal 2, but run those runners as separate CI jobs or Drone steps:

```bash
mvn -pl cmd-adaptor-sns -Dtest=EventIntegrationTest test
mvn -pl cmd-adaptor-sns -Dtest=PartyIntegrationTest test
mvn -pl cmd-adaptor-sns -Dtest=LocationIntegrationTest test
mvn -pl cmd-adaptor-sns -Dtest=ObjectIntegrationTest test
mvn -pl cmd-adaptor-sns -Dtest=ServiceIntegrationTest test
mvn -pl cmd-adaptor-sns -Dtest=RelationshipsIntegrationTest test
```

If `event/` remains the long pole, split it into two or three event runners:

```text
EventAssociationIntegrationTest
EventMovementIntegrationTest
EventMiscIntegrationTest
```

The exact split should be based on measured scenario timing, not only feature-file count.

## Why This Helps

- CI can run independent runner groups at the same time.
- Failures are easier to locate because each group has its own CI step.
- Heavy groups can be split without changing the execution model for all tests.
- Step-level retry, log retention, and timing metrics become more useful.

## Trade-Offs

| Trade-Off | Impact | Mitigation |
|-----------|--------|------------|
| Repeated Maven startup per CI step | Can add overhead | Use dependency cache and keep steps module-scoped |
| Repeated Spring context startup per runner group | Expected | Similar to Surefire fork approach; benchmark against Proposal 2 |
| RepoSync ownership of `.drone.star` | May block direct edits | Coordinate with the RepoSync/platform team |
| More CI configuration | Slightly more maintenance | Generate runner-step definitions consistently |

## Implementation Sketch

1. Create runner classes from Proposal 2.
2. Keep a local single-command path for developers:

   ```bash
   mvn -pl cmd-adaptor-sns test
   ```

3. Add CI steps that run one runner or shard each.
4. Publish Surefire reports per step.
5. Compare wall-clock time against Proposal 2 with `forkCount=2/4/6`.

## When To Choose This

Choose this over Surefire-only fork parallelism when:

- CI diagnostics and per-group ownership matter.
- The event group needs custom sharding.
- CI workers have enough capacity to run multiple Maven steps concurrently.
- RepoSync can support parallel test steps cleanly.

Choose Proposal 2 instead when:

- You want the smallest code and CI change.
- The team prefers a single Maven test invocation.
- CI dependency caching is weak and repeated Maven startup would be expensive.

## Verification

After implementation:

1. The sum of all runner groups still reports 343 scenarios.
2. Every feature directory is covered by exactly one runner or shard.
3. A deliberately failing scenario fails only its own CI step.
4. Total CI wall time improves versus the current single runner.
5. A sequential fallback command remains available for local reproduction.
