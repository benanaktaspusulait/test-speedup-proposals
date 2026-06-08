# Proposal 1: Maven Parallel Module Build

## Summary

| Attribute | Value |
|-----------|-------|
| Expected Speed-Up | Low single-digit overall; ~1s on the current reactor |
| Risk | Very low |
| Effort | 1 hour |
| Reversibility | Instant (remove one flag) |
| Prerequisites | None |

## Problem

The Maven build executes modules sequentially by default:

```
cmd-adaptor-sns-avro         → 5.2s
cmd-adaptor-sns-common       → 0.5s
cmd-adaptor-sns-test-common  → 0.5s
cmd-adaptor-sns              → 38.9s (includes tests)
cmd-adaptor-sns-integration-tests → skipped
─────────────────────────────────────
Total sequential:              ~45s module build time
```

Only part of the early reactor can run in parallel. `cmd-adaptor-sns-avro` and `cmd-adaptor-sns-common` can overlap, but `cmd-adaptor-sns-test-common` depends on `cmd-adaptor-sns-common`, and the main `cmd-adaptor-sns` module still waits for all upstream modules.

## Proposed Solution

Add the `-T` (thread count) flag to the Maven build command. This tells Maven to build independent modules in parallel.

### Change Required

In `.drone.star` (or wherever the build command is invoked):

```python
# Before
'mvn clean install'

# After
'mvn clean install -T 4'
```

The `-T 4` flag means "use up to 4 threads for module builds". Maven will automatically determine the dependency graph and parallelize where possible.

### What Happens With `-T 4`

```
Thread 1: cmd-adaptor-sns-avro         (5.2s)
Thread 2: cmd-adaptor-sns-common       (0.5s) → done early, thread reused
          cmd-adaptor-sns-test-common  (0.5s, after common)
                    │
                    ▼ (all 3 complete)
Thread 1: cmd-adaptor-sns              (38.9s — depends on the above three)
                    │
                    ▼
Thread 1: cmd-adaptor-sns-integration-tests (skipped)
```

The early modules partially overlap (~5.2s instead of ~6.2s sequentially based on the current log). The main module still waits for all upstream modules, so this is a safe reactor-level improvement, not the main topology-test speed-up.

### Alternative: `-T 1C`

```python
'mvn clean install -T 1C'
```

This means "1 thread per available CPU core". On a 4-core CI runner, this is equivalent to `-T 4`. This scales automatically with the CI runner's resources.

## Impact Analysis

| Phase | Before | After | Savings |
|-------|--------|-------|---------|
| Pre-test module builds | ~6.2s | ~5.2s | ~1s |
| Main module (with tests) | ~38.9s | ~38.9s | 0s |
| Total build | ~59.8s | ~58-59s | ~1s |

The savings come primarily from overlapping the independent module builds. The topology tests in `cmd-adaptor-sns` still run sequentially within that module.

**Note:** The actual savings depend on CI runner CPU availability. If the runner is already CPU-constrained, parallel builds may not help significantly. Treat `-T` as a low-risk cleanup rather than the primary optimization.

## Risks and Mitigations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Build instability from parallel execution | Very low | Maven's reactor handles dependency ordering correctly |
| CI runner resource contention | Low | Use `-T 1C` to auto-scale with available cores |
| Log output interleaving | Low | Only affects readability; use `-B` (batch mode) for cleaner output |

## Implementation Steps

1. Modify the build command in `.drone.star` (line with `'mvn clean install'`)
2. Test on a feature branch
3. Verify build logs show parallel module execution
4. Merge

## Important Note

The `.drone.star` file is managed centrally by RepoSync. Changes must be coordinated with the platform team, or applied via a RepoSync override mechanism if available.

## Verification

After applying, check the build log for lines like:

```
[INFO] Using the MultiThreadedBuilder implementation with a thread count of 4
```

And confirm that `cmd-adaptor-sns-avro`, `cmd-adaptor-sns-common`, and `cmd-adaptor-sns-test-common` show overlapping timestamps.
