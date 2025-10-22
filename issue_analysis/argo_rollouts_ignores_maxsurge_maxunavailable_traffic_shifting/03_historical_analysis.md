# Historical Analysis: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Git Blame Analysis
### File: `utils/replicaset/canary.go`
- **Lines 337-380:** `CalculateReplicaCountsForTrafficRoutedCanary()` function
  - **Last modified:** c1353e4a44 (Jesse Suen, 2021-09-21)
  - **Commit Message:** feat: support dynamic scaling of stable ReplicaSet as inverse of canary weight (#1430)
  - **Analysis:** Function introduced with traffic routing support but without maxSurge/maxUnavailable logic

- **Lines 94-150:** `CalculateReplicaCountsForBasicCanary()` function  
  - **Last modified:** 0f93f9b82e (Jesse Suen, 2022-01-14) - maxSurge line added
  - **Commit Message:** fix!: improve basic canary approximation accuracy and honor maxSurge (#1759)
  - **Analysis:** maxSurge support added to basic canary but not traffic-routed canary

## Related Commits
| Commit Hash | Date | Author | Message | Relevance |
|-------------|------|--------|---------|-----------|
| c1353e4a44 | 2021-09-21 | Jesse Suen | feat: support dynamic scaling of stable ReplicaSet as inverse of canary weight (#1430) | **HIGH** - Introduced traffic-routed canary without maxSurge support |
| 0f93f9b82e | 2022-01-14 | Jesse Suen | fix!: improve basic canary approximation accuracy and honor maxSurge (#1759) | **HIGH** - Added maxSurge to basic canary but not traffic-routed |
| 7c6a0c519 | 2021-08-26 | Jesse Suen | fix: canary scaledown event could violate maxUnavailable (#1429) | **MEDIUM** - Related maxUnavailable fix for basic canary |

## Correlation with Linked Issues/PRs
- **GitHub Issue #557:** "Support scaleDownDelaySeconds & fast rollbacks with canary strategy"
  - **Analysis:** This issue is linked in the original issue description but appears unrelated to maxSurge/maxUnavailable. It's about rollback behavior, not scaling limits.

- **GitHub Issue #2239:** "Traffic Routing and maxSurge/maxUnavailable"
  - **Analysis:** **CRITICAL** - Zach Aller references this issue explaining why maxSurge/maxUnavailable are intentionally not used with traffic routing. The design prioritizes traffic control over pod scaling limits.

- **Related Issues from Slack Discussion:**
  - **#3284, #3539, #3397:** Multiple issues requesting maxSurge/maxUnavailable support with traffic routing
  - **Analysis:** Indicates this is a common user request, suggesting the current behavior may be seen as a limitation

## Community Discussion Insights (CNCF Slack, Feb 2024)
**Key Findings from Argo Rollouts Maintainers:**

1. **Intentional Design Decision:** Traffic routing was explicitly designed to not use maxSurge/maxUnavailable, unlike standard Kubernetes Deployments.

2. **Rollback Behavior Exception:** During rollbacks with traffic shifting, AR does respect minimum availability by:
   - Requesting maximum pods in the stable set
   - Gradually shifting traffic back as pods become available

3. **Traffic Routing Specifics:** 
   - Only supports `minPodsPerReplicaSet` (not maxSurge/maxUnavailable)
   - Prioritizes traffic control over pod scaling limits
   - Different design philosophy from basic canary deployments

4. **Maintainer Response:** Zach Aller clarified that PDBs (Pod Disruption Budgets) aren't the right analogy - the real issue is maxSurge/maxUnavailable being ignored.

## Previous Resolution Attempts
1. **Traffic Routing Introduction (2021):**
   - **Attempt:** Added `CalculateReplicaCountsForTrafficRoutedCanary()` function
   - **Outcome:** **Intentional** - Traffic routing works but ignores maxSurge/maxUnavailable by design
   - **Why:** Traffic routing prioritizes traffic control over scaling limits (see issue #2239)

2. **Basic Canary maxSurge Fix (2022):**
   - **Attempt:** Improved `CalculateReplicaCountsForBasicCanary()` to honor maxSurge
   - **Outcome:** **Success** for basic canary
   - **Gap identified:** Traffic-routed canary maintains different design philosophy

3. **Multiple User Requests (2023-2024):**
   - **Issues:** #3284, #3539, #3397 all request maxSurge/maxUnavailable support
   - **Outcome:** **No resolution** - Maintainers maintain this is by design
   - **Community Impact:** Users report infrastructure scaling issues

## Current Workarounds and Limitations

#### MinPodsPerReplicaSet as Partial Mitigation
**Purpose:** Introduced to provide minimum pod counts for high availability during traffic routing.

**Implementation:**
```go
func CheckMinPodsPerReplicaSet(rollout *v1alpha1.Rollout, count int32) int32 {
    if rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet != nil && rollout.Spec.Strategy.Canary.TrafficRouting != nil {
        return max(count, *rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet)
    }
    return count
}
```

**Limitations:**
- Only sets minimum floor, doesn't control maximum surge
- Doesn't prevent rapid scaling between traffic weights
- Requires manual tuning per rollout
- Doesn't address user complaints about "very rapid infrastructure scaling"

#### Manual Canary Steps as Current Alternative
**User Counter-Argument:** Without maxSurge/maxUnavailable controls, users are forced to manually add multiple small canary steps to control scaling speed.

**Example Manual Approach:**
```yaml
strategy:
  canary:
    steps:
    - setWeight: 5     # Small increment instead of smooth scaling
    - pause: {duration: 60s}
    - setWeight: 10    # Another small increment
    - pause: {duration: 60s}
    # ... 18 more steps for what should be automatic scaling
```

**Problems with Manual Approach:**
- Creates verbose, hard-to-maintain rollout definitions
- Doesn't adapt to different traffic patterns or service characteristics
- Manual process prone to configuration errors
- Poor user experience compared to standard Kubernetes deployment controls
- Forces users to micro-manage what should be automated scaling behavior

This workaround highlights the fundamental issue: traffic routing should support standard Kubernetes scaling controls rather than requiring manual intervention.

## Key Insights
- **Design Philosophy Difference:** Traffic routing and basic canary have fundamentally different scaling approaches
- **Intentional Behavior:** maxSurge/maxUnavailable omission appears to be by design, not oversight
- **Rollback Exception:** Some minimum availability logic exists during rollbacks
- **Community Demand:** Multiple issues suggest users want this feature despite design intentions
- **Infrastructure Impact:** Users report "very rapid infrastructure scaling" when using traffic routing

### Relative maxSurge/maxUnavailable Proposal Analysis

#### Current Behavior
**Basic Canary:** maxSurge/maxUnavailable are relative to total rollout replicas (standard Kubernetes behavior)
- Can be percentages (e.g., "25%") or absolute numbers (e.g., 3)
- `MaxSurge(rollout)` and `MaxUnavailable(rollout)` calculate absolute values from percentages

**Traffic-Routed Canary:** Settings completely ignored - traffic weights drive scaling directly

#### Proposed Relative Interpretation
**Idea:** Make maxSurge/maxUnavailable relative to traffic weight changes between canary steps, not total replicas.

**Example Scenario:**
```yaml
spec:
  replicas: 10
  strategy:
    canary:
      maxSurge: "25%"        # Currently ignored
      maxUnavailable: "25%"  # Currently ignored
      steps:
      - setWeight: 20        # Canary: 2 replicas, Stable: 8
      - setWeight: 30        # Canary: 3 replicas, Stable: 7 (Δ = +1 canary, -1 stable)
```

**Relative Calculation:**
- Replica delta between steps: +1 canary, -1 stable
- maxSurge "25%" of delta: allow +0.25 extra pods during transition
- maxUnavailable "25%" of delta: allow -0.25 fewer pods during transition

#### Technical Feasibility
**Pros:**
- ✅ Maintains traffic-control-first philosophy
- ✅ Provides scaling control without breaking traffic routing
- ✅ Reuses existing percentage/absolute parsing logic
- ✅ Could work with existing `trafficWeightToReplicas()` function

**Cons:**
- ❌ Requires tracking previous traffic weights between steps
- ❌ Complex state management for step transitions
- ❌ Different behavior from basic canary (confusing for users)
- ❌ May not address "rapid infrastructure scaling" if steps are large
- ❌ Still requires manual step sizing for fine control

**Implementation Sketch:**
```go
func CalculateReplicaCountsForTrafficRoutedCanaryWithLimits(rollout *v1alpha1.Rollout, weights *v1alpha1.TrafficWeights) (int32, int32) {
    // Calculate desired counts from traffic weights
    desiredCanary := trafficWeightToReplicas(rollout.Spec.Replicas, desiredWeight, maxWeight)
    desiredStable := rollout.Spec.Replicas - desiredCanary
    
    // Get previous counts from rollout status or calculate from previous weight
    prevCanary := trafficWeightToReplicas(rollout.Spec.Replicas, previousWeight, maxWeight)
    
    // Calculate replica delta
    delta := desiredCanary - prevCanary
    
    // Apply maxSurge/maxUnavailable to the delta
    maxSurgeAbs := calculateMaxSurgeRelativeToDelta(delta, rollout)
    maxUnavailableAbs := calculateMaxUnavailableRelativeToDelta(delta, rollout)
    
    // Adjust scaling speed based on limits
    actualCanary := applyScalingLimits(prevCanary, desiredCanary, maxSurgeAbs, maxUnavailableAbs)
    actualStable := rollout.Spec.Replicas - actualCanary
    
    return actualCanary, actualStable
}
```

#### Would This Solve the Problem?
**Partial Solution:** Would provide some scaling control but still requires manual step sizing.

**Better Alternative:** Keep absolute interpretation but respect the limits during traffic weight transitions.

**Fundamental Issue:** Traffic routing prioritizes traffic control over pod scaling limits. Any solution needs to balance both priorities.

#### Recommendation
**Relative interpretation is an interesting idea but:**
1. Adds complexity without fully solving manual step problem
2. Creates inconsistent behavior between canary types
3. Better to implement absolute limits with traffic-aware logic
4. Focus on clear validation warnings and documentation as immediate improvements