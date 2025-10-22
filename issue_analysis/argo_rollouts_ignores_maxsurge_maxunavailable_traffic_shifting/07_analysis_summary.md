# Analysis Summary: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Executive Summary
This analysis reveals that the omission of `maxSurge` and `maxUnavailable` support in traffic-routed canary deployments is **an intentional design decision**, not a bug or oversight. The Argo Rollouts maintainers deliberately prioritize traffic control over pod scaling limits, as evidenced by maintainer discussions in CNCF Slack and GitHub issue #2239.

## Key Findings

### Technical Gap Confirmed
- **Basic Canary:** Respects maxSurge/maxUnavailable via `CalculateReplicaCountsForBasicCanary()`
- **Traffic-Routed Canary:** Ignores these limits via `CalculateReplicaCountsForTrafficRoutedCanary()`
- **Code Location:** `utils/replicaset/canary.go` (lines 200-300+)

### Intentional Design Decision
- **Maintainer Rationale:** Traffic routing prioritizes traffic control over scaling limits
- **Exception Case:** Rollback scenarios maintain minimum availability (1 pod minimum)
- **Design Philosophy:** Traffic weights drive scaling, not Kubernetes deployment limits

### Community Impact
- **Multiple Issues:** #3284, #3539, #3397 report "very rapid infrastructure scaling"
- **User Pain Points:** Unexpected cost and resource consumption
- **Demand Exists:** Community clearly wants maxSurge/maxUnavailable support

### Current Workarounds Analysis

#### MinPodsPerReplicaSet Limitations
**Purpose:** Provides minimum pod floor for high availability in traffic-routed canaries.

**How it helps:**
```go
func CheckMinPodsPerReplicaSet(rollout *v1alpha1.Rollout, count int32) int32 {
    return max(count, *rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet)
}
```

**Limitations:**
- Only prevents scale-down below minimum, doesn't control scale-up/surge
- Doesn't prevent rapid scaling between traffic weight changes
- Requires manual tuning per rollout
- Doesn't address the core "rapid infrastructure scaling" complaint

#### Manual Canary Steps Workaround
**Current Alternative:** Users must manually create many small canary steps to control scaling speed.

**Example Problem:**
```yaml
steps:
- setWeight: 5   # Small increment to control scaling
- pause: {duration: 60s}
- setWeight: 10  # Another small increment
- pause: {duration: 60s}
# ... 18 more steps instead of smooth progression
```

**Issues:**
- Verbose and hard to maintain rollout definitions
- Doesn't adapt to different traffic patterns
- Manual process prone to configuration errors
- Poor user experience compared to standard Kubernetes controls

### Relative maxSurge/maxUnavailable Design Alternative
**User Proposal:** Make maxSurge/maxUnavailable relative to traffic weight changes between canary steps rather than total replicas.

**Analysis:** Interesting concept but adds complexity:
- Would require tracking previous traffic weights
- Creates different behavior from basic canary
- Still requires manual step sizing for fine control
- May not fully address "rapid infrastructure scaling" concerns

**Recommendation:** Consider absolute limits first, evaluate relative approach if absolute proves insufficient.

## Analysis Evolution
Initial analysis treated this as a technical oversight. However, CNCF Slack context revealed maintainer design philosophy, fundamentally changing the issue characterization from "bug to fix" to "design decision to potentially challenge."

## Contribution Strategy
**High Controversy:** Challenging fundamental traffic routing design decisions.

### Recommended Approach
1. **Immediate:** Add validation warning when maxSurge/maxUnavailable set with traffic routing
2. **Short-term:** Improve MinPodsPerReplicaSet documentation and best practices
3. **Long-term:** Pursue design change only with strong community consensus

### Effort Estimate
- **Design Discussion:** 2-3 weeks (required)
- **Implementation (if approved):** 2-3 weeks
- **Documentation Alternative:** 1-2 weeks
- **Total Range:** 3-8 weeks depending on maintainer consensus

## Conclusion
This issue represents a **design philosophy conflict** between:
- **Maintainer View:** Traffic control takes precedence over scaling limits
- **User View:** Infrastructure costs and resource management are critical

The current workarounds (MinPodsPerReplicaSet, manual canary steps) are inadequate solutions that highlight the real need for proper scaling controls. However, challenging this fundamental design decision requires strong community consensus and may face maintainer resistance.

## Next Steps
1. **Immediate:** Add validation warning for unsupported maxSurge/maxUnavailable with traffic routing
2. **Community:** Engage maintainers in GitHub issue #2239 discussion
3. **Documentation:** Improve MinPodsPerReplicaSet guidance and manual step workarounds
4. **Design Analysis:** See `08_relative_interpretation_analysis.md` for detailed evaluation of making maxSurge/maxUnavailable relative to traffic weight changes
5. **Long-term:** Pursue implementation only if consensus achievable

---
*Analysis completed following in_depth_issue_analysis.prompt.md methodology*
*All 7 analysis steps completed with community context and workaround analysis*
*Relative interpretation design analyzed in separate document*