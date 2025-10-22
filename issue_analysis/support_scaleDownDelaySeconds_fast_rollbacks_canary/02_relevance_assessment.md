# Issue Relevance Assessment: Support scaleDownDelaySeconds & fast rollbacks with canary strategy

## Assessment Date
October 17, 2025

## Current Status
- [x] Issue is still present
- [ ] Issue has been resolved
- [ ] Issue is partially resolved
- [ ] Issue is no longer relevant

## Evidence

### 1. Validation Restriction Still Exists
The validation code in `pkg/apis/rollouts/validation/validation.go` (lines 294-295) still explicitly prevents basic canary deployments from using `scaleDownDelaySeconds`:

```go
if canary.TrafficRouting == nil {
    if canary.ScaleDownDelaySeconds != nil {
        allErrs = append(allErrs, field.Invalid(fldPath.Child("scaleDownDelaySeconds"), *canary.ScaleDownDelaySeconds, InvalidCanaryScaleDownDelay))
    }
}
```

The error message `InvalidCanaryScaleDownDelay = "Canary scaleDownDelaySeconds can only be used with traffic routing"` confirms this restriction is intentional and still active.

### 2. Test Cases Confirm Current Behavior
The test `TestCanaryScaleDownDelaySeconds` in `validation_test.go` (lines 896-906) explicitly tests and expects this validation error:
- Basic canary with `scaleDownDelaySeconds` → **validation error**
- Traffic routing canary with `scaleDownDelaySeconds` → **allowed**

### 3. Fast Rollback Gap Still Exists
Analysis of `rollout/bluegreen.go` vs canary implementation shows:

**Blue-Green Fast Rollback (line 114-115):**
```go
if replicasetutil.HasScaleDownDeadline(c.newRS) {
    c.log.Infof("Detected scale down annotation for ReplicaSet '%s' and will skip pause", c.newRS.Name)
    return true
}
```

**Canary Implementation:** No equivalent fast rollback logic that checks for scale down deadline annotations.

### 4. Defaults Implementation Confirms Limited Support
In `utils/defaults/defaults.go` (lines 194-200), canary scale down delay is only applied when traffic routing is enabled:

```go
if rollout.Spec.Strategy.Canary != nil {
    if rollout.Spec.Strategy.Canary.TrafficRouting != nil {
        delaySeconds = DefaultScaleDownDelaySeconds
        if rollout.Spec.Strategy.Canary.ScaleDownDelaySeconds != nil {
            delaySeconds = *rollout.Spec.Strategy.Canary.ScaleDownDelaySeconds
        }
    }
}
```

**Note on the 30s default and operational recommendations**

- The `DefaultScaleDownDelaySeconds` and `DefaultAbortScaleDownDelaySeconds` are set to `30` in
    `utils/defaults/defaults.go`. The proto and experiment type comments recommend 30s as a
    minimum to allow iptables/service-provider propagation after switching selectors. However,
    empirical testing (see `empirical_evidence.md`) and practical rollout timings show 30s is
    frequently too short for human or CI-driven rollback flows. We recommend setting
    `scaleDownDelaySeconds` to 300–600s for production use (example manifest: `examples/rollout-scaleDownDelay-5m.yaml`).

### 5. Feature Parity Gap
Current state comparison:

| Feature | Blue-Green | Canary (Basic) | Canary (Traffic Routing) |
|---------|------------|----------------|--------------------------|
| scaleDownDelaySeconds | ✅ Supported | ❌ Validation Error | ✅ Supported |
| Fast rollback via scale down deadline | ✅ Supported | ❌ Not Implemented | ❌ Not Implemented |
| Rollback window support | ✅ Supported | ✅ Supported | ✅ Supported |

## Testing Results

### Test Scenario 1: Basic Canary with scaleDownDelaySeconds
**Expected:** Validation error  
**Actual:** Validation error occurs as expected  
**Result:** ✅ Confirms issue exists

### Test Scenario 2: Traffic Routing Canary Fast Rollback
**Expected:** Should leverage scale down delay for fast rollback like blue-green  
**Actual:** No fast rollback mechanism implemented for canary strategy  
**Result:** ✅ Confirms gap exists

### Test Scenario 3: Production Empirical Evidence
**Configuration:** Skyscanner production environment with traffic routing canary
- `scaleDownDelaySeconds: 30` (default)
- `rollbackWindow.revisions: 3`
- `revisionHistoryLimit: 2`

**Test 3a: Standard Canary Deployment**
- ✅ Scale down delay working correctly
- ✅ Old ReplicaSet scaled down after exactly 30 seconds
- ✅ Controller logs confirm: "RS has not reached the scaleDownTime"

**Test 3b: Rollback Within Window - Inconsistent Behavior**
- ⚠️ Rollback behavior inconsistent across attempts
- ✅ Some rollbacks show speed improvement (6:58 → 2:11 in best case)
- ❌ Analysis reuse unpredictable - sometimes skipped, sometimes repeated
- ❌ New revisions created even with identical pod template hash
- ❌ Rollback window counting appears revision-based, causing confusion

**Test 3c: Rollback Window Limitations**
- ❌ `rollbackWindow.revisions: 3` may be too restrictive with frequent rollbacks
- ❌ Internal revision counting vs Git commit-based expectations mismatch
- ⚠️ Performance varies: 2-6 minutes depending on analysis reuse

**Test 3d: Fast Rollback Within Scale Down Window**
- ❌ Production deployment pipeline too slow to complete rollback within 30s
- ❌ Would require longer window (e.g., 10 minutes) for practical use
- ❌ Confirms the core issue: no mechanism to leverage scale down delay for fast rollback

**Production Impact:** 
- 30-second default window insufficient for real-world rollback scenarios
- Rollback window behavior inconsistent and confusing
- Performance benefits of rollback window vary unpredictably (2-6 minutes)

## Recent Changes Review

### Search for Related Commits
Searched recent commits for scaleDownDelaySeconds, fast rollback, and canary-related changes:

- No recent changes have addressed the core issue
- Scale down delay functionality exists but is limited to traffic routing scenarios
- Fast rollback logic remains blue-green specific
- Validation restrictions remain unchanged

### Analysis of Current Branch
- Branch: `skyscanner-internal/master`
- The core limitation and feature gap identified in GitHub Issue #557 remains unaddressed
- Infrastructure exists but integration is incomplete

## Conclusion

**The issue is STILL RELEVANT and needs implementation work.**

### Key Findings:
1. **Basic canary restriction persists:** Validation explicitly prevents scaleDownDelaySeconds usage without traffic routing
2. **Fast rollback gap confirmed:** Canary deployments lack the scale down deadline-based fast rollback mechanism that blue-green has
3. **Feature parity missing:** Blue-green strategy offers superior rollback capabilities compared to canary
4. **Infrastructure exists:** Scale down delay mechanism is implemented but not fully leveraged for canary fast rollbacks


Reference: upstream discussion on this behavior is tracked at Issue #557.

### Priority Assessment: HIGH
- This affects deployment strategy choice and rollback capabilities
- Feature parity between deployment strategies is important for user experience
- The infrastructure largely exists, making implementation more feasible
- No recent work has addressed this specific enhancement request
- **Production validation**: Real-world testing confirms the 30s default window is insufficient for practical rollback scenarios
- **Operational impact**: Teams need longer scale down delays (e.g., 10 minutes) and fast rollback integration for effective rollback strategies

### Next Steps Required:
1. Analyze historical attempts and understand why this limitation exists
2. Research upstream community discussions and approaches
3. Design implementation that leverages existing infrastructure
4. Plan comprehensive solution including basic canary support and fast rollback integration

## Upstream references (selected)

- Historical discussion on iptables/service propagation (why 30s minimum): https://github.com/argoproj/argo-rollouts/issues/19#issuecomment-476329960
- Enhancement: Support scaleDownDelaySeconds & fast rollbacks for canary — https://github.com/argoproj/argo-rollouts/issues/557
- AnalysisRun rollback-window fix (PR #3670) — https://github.com/argoproj/argo-rollouts/pull/3670 (commit 243ea917)
- progressDeadlineSeconds vs scaleDownDelaySeconds discussion/fix (Issue #3414 / PR #3417) — https://github.com/argoproj/argo-rollouts/issues/3414 and https://github.com/argoproj/argo-rollouts/pull/3417
- HPA + scaleDownDelaySeconds reported issues (Issue #3848) — https://github.com/argoproj/argo-rollouts/issues/3848
- DynamicStableScale delay patch (PR #4337, not merged) — https://github.com/argoproj/argo-rollouts/pull/4337
- Abort delay docs: see `pkg/apis/rollouts/v1alpha1/generated.proto` and `pkg/apis/rollouts/v1alpha1/experiment_types.go` for the documented 30s default for `abortScaleDownDelaySeconds`.

**Note:** A new ticket C1' (Fast-rollback to known-good revision inside rollbackWindow even when RS scaled to 0) has been added to the contribution epic. This ticket frames the core missing capability: the controller cannot currently fast-track rollback to a prior revision if that revision's ReplicaSet was scaled to 0 and there's no retained evidence. C1' proposes adding conservative checks (for example, a retained successful AnalysisRun for that revision, rollout status evidence, or a recent scale-down-deadline annotation) as signals to allow skipping new AnalysisRuns during rollback and advancing steps safely. If none of those signals exist (GC'd RS or missing AR), the controller should continue to use normal rollback behavior.