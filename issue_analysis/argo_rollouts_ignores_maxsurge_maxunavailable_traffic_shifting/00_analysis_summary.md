# Analysis Summary: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Analysis Document Navigation

- **[Analysis Summary](00_analysis_summary.md)** - Executive overview and key findings
- **[Current State Analysis](01_current_state_analysis.md)** - Technical implementation details and code locations
- **[Relevance Assessment](02_relevance_assessment.md)** - Issue validation and testing evidence
- **[Historical Analysis](03_historical_analysis.md)** - Git blame, commit analysis, and design evolution
- **[Upstream Analysis](04_upstream_analysis.md)** - Community issues, maintainer discussions, and similar patterns
- **[Contribution Epic](05_contribution_epic.md)** - Implementation strategy and risk assessment
- **[Effort Estimation](06_effort_estimation.md)** - Timeline and resource requirements
- **[Relative Interpretation Analysis](07_relative_interpretation_analysis.md)** - Alternative design approaches

---

## Executive Summary
This analysis reveals that the omission of `maxSurge` and `maxUnavailable` support in traffic-routed canary deployments is **an intentional design decision**, not a bug or oversight. The Argo Rollouts maintainers deliberately prioritize traffic control over pod scaling limits, as evidenced by maintainer discussions in CNCF Slack and GitHub issue #2239.

## Community Context: Multiple Similar Issues

**Critical Context:** This issue is not isolated - multiple community members have reported the same problem, indicating widespread user impact and strong demand for resolution.

| Issue # | Title | Status | User Impact | Key Evidence |
|---------|-------|--------|-------------|--------------|
| **#2239** | Traffic Routing and maxSurge/maxUnavailable | Open/Design Discussion | **CRITICAL** - Explains intentional design | Zach Aller (maintainer) references this issue explaining why maxSurge/maxUnavailable are intentionally not used with traffic routing |
| **#3284** | [Request] Support maxSurge/maxUnavailable with traffic routing | Open | **HIGH** - Feature request | Community demand for functionality to control "very rapid infrastructure scaling" |
| **#3539** | maxSurge and maxUnavailable ignored when using traffic routing | Open | **HIGH** - Infrastructure scaling issues | Reports "very rapid infrastructure scaling" causing cost and resource problems |
| **#3397** | maxSurge/maxUnavailable not respected with traffic routing | Open | **HIGH** - Scaling control gaps | Another instance of the same scaling control problem |

**Community Impact Summary:**
- **4+ Open Issues** spanning multiple years indicate persistent user pain points
- **Consistent Problem Description:** "very rapid infrastructure scaling" when using traffic routing
- **Business Impact:** Unexpected infrastructure costs and resource consumption
- **User Demand:** Clear community consensus that maxSurge/maxUnavailable support is needed

## Key Findings

### Functionality Matrix: Basic vs Traffic-Enabled Canaries

| Feature | Basic Canary | Traffic-Enabled Canary | Notes |
|---------|-------------|----------------------|-------|
| **maxSurge/maxUnavailable** | ✅ **Supported** | ⚠️ **Conditional** | Depends on dynamicStableScale; ignored by default for instant rollback |
| **scaleDownDelaySeconds** | ❌ **Validation Error** | ✅ **Supported** | Only available with traffic routing for gradual traffic shifting |
| **dynamicStableScale** | ❌ **Validation Error** | ✅ **Supported** | Dynamically scales stable ReplicaSet to minimize total pods during updates |
| **minPodsPerReplicaSet** | ❌ **Not Applicable** | ✅ **Supported** | Ensures minimum pod floor for high availability in traffic-routed canaries |
| **Traffic Weight Control** | ❌ **Not Applicable** | ✅ **Supported** | Fine-grained traffic distribution (0-100%) between stable and canary |
| **Service Mesh Integration** | ❌ **Not Applicable** | ✅ **Supported** | Istio, Linkerd, ALB, SMI traffic routing providers |
| **Instant Rollback** | ✅ **Supported** | ✅ **Supported** | Traffic routing enables faster rollbacks via traffic shifting |

### DynamicStableScale Impact Analysis

**Critical Finding:** The feasibility of implementing `maxSurge`/`maxUnavailable` for traffic-enabled canaries depends heavily on the `dynamicStableScale` setting.

#### Without DynamicStableScale (Default Behavior)
**Current State:** Stable ReplicaSet remains fully scaled to support instant rollbacks.

**Scaling Dynamics:**
```go
// In CalculateReplicaCountsForTrafficRoutedCanary()
// When !rollout.Spec.Strategy.Canary.DynamicStableScale
return canaryCount, rolloutSpecReplica  // stable = full scale
```

**maxSurge/maxUnavailable Applicability:**
- **maxUnavailable:** ❌ **Not Applicable** - No "unavailable" capacity exists; stable is always at 100%
- **maxSurge:** ⚠️ **Theoretically Possible** - Could limit canary ReplicaSet scale-up, but current implementation ignores it entirely

**Why Ignored:** Design prioritizes traffic control over pod scaling limits for instant rollback capability.

#### With DynamicStableScale Enabled
**Current State:** Stable ReplicaSet scales dynamically based on traffic weights.

**Scaling Dynamics:**
```go
// When rollout.Spec.Strategy.Canary.DynamicStableScale == true
stableCount = trafficWeightToReplicas(rolloutSpecReplica, maxWeight-desiredWeight, maxWeight)
// Total replicas can now vary: canary + stable ≠ rolloutSpecReplica
```

**maxSurge/maxUnavailable Applicability:**
- **maxUnavailable:** ✅ **Fully Applicable** - Both ReplicaSets can scale down below desired traffic-based counts
- **maxSurge:** ✅ **Fully Applicable** - Total replica count across ReplicaSets can exceed rollout spec

**Implementation Feasibility:** Both limits become meaningful when stable scaling is dynamic.

### Technical Gap Confirmed
- **Basic Canary:** Respects maxSurge/maxUnavailable via `CalculateReplicaCountsForBasicCanary()`
- **Traffic-Routed Canary:** Ignores these limits via `CalculateReplicaCountsForTrafficRoutedCanary()`
- **Code Location:** `utils/replicaset/canary.go` (lines 200-300+)

### Intentional Design Decision
- **Maintainer Rationale:** Traffic routing prioritizes traffic control over scaling limits
- **Exception Case:** Rollback scenarios maintain minimum availability (1 pod minimum)
- **Design Philosophy:** Traffic weights drive scaling, not Kubernetes deployment limits

### Community Impact
- **Widespread User Pain Points:** Multiple users report "very rapid infrastructure scaling" causing unexpected costs
- **Strong Feature Demand:** Community clearly wants maxSurge/maxUnavailable support despite maintainer design philosophy
- **Infrastructure Pressure:** Lack of scaling controls puts additional pressure on cluster autoscaling systems

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

## Conclusion
This issue represents a **nuanced design philosophy conflict** with a potential technical solution:

**Traditional View:** Traffic control takes precedence over scaling limits (maintainer perspective)

**Emerging Understanding:** 
- **Without dynamicStableScale:** maxSurge/maxUnavailable remain non-applicable due to instant rollback requirements
- **With dynamicStableScale:** Both limits become technically feasible and could address user scaling concerns

**Key Insight:** The implementation may be viable when `dynamicStableScale=true`, potentially resolving the conflict without challenging core traffic routing principles.

**Community Context:** As detailed in the "Community Context" section above, this is not an isolated issue - 4+ open GitHub issues spanning multiple years indicate persistent user pain points with "very rapid infrastructure scaling" when using traffic routing.

The current workarounds (MinPodsPerReplicaSet, manual canary steps) remain inadequate, but the dynamicStableScale analysis opens a new implementation pathway that respects the existing design philosophy while addressing user infrastructure scaling pain points.

## Next Steps
1. **Immediate:** Add validation warning for unsupported maxSurge/maxUnavailable with traffic routing
2. **Analysis:** Complete dynamicStableScale impact assessment on implementation feasibility
3. **Community:** Engage maintainers in GitHub issue #2239 discussion with new technical insights
4. **Documentation:** Improve MinPodsPerReplicaSet guidance and manual step workarounds
5. **Design Analysis:** See `07_relative_interpretation_analysis.md` for detailed evaluation of making maxSurge/maxUnavailable relative to traffic weight changes
6. **Long-term:** Pursue conditional implementation (dynamicStableScale=true only) if consensus achievable

---
*Analysis completed following in_depth_issue_analysis.prompt.md methodology*
*All 7 analysis steps completed with community context and workaround analysis*
*Relative interpretation design analyzed in separate document*