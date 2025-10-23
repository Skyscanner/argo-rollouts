# Relative maxSurge/maxUnavailable Design Analysis

## Proposal Overview
Make `maxSurge` and `maxUnavailable` relative to traffic weight changes between canary steps, rather than applying to total rollout replicas.

## Current Behavior Analysis

### Basic Canary Scaling
```go
// maxSurge/maxUnavailable are relative to total rollout replicas
maxSurgeAbs := MaxSurge(rollout)        // e.g., 25% of 10 replicas = 2-3 pods
maxUnavailableAbs := MaxUnavailable(rollout)  // e.g., 25% of 10 replicas = 2-3 pods

maxReplicaCountAllowed := rolloutSpecReplica + maxSurgeAbs  // 10 + 2-3 = 12-13
minAvailableReplicaCount := rolloutSpecReplica - maxUnavailableAbs  // 10 - 2-3 = 7-8
```

### Traffic-Routed Canary Scaling
```go
// maxSurge/maxUnavailable completely ignored
canaryCount := trafficWeightToReplicas(rolloutSpecReplica, desiredWeight, maxWeight)
stableCount := rolloutSpecReplica - canaryCount  // or dynamic scaling logic
```

## Proposed Relative Interpretation

### Concept
Instead of limits applying to total replicas, apply them to replica changes caused by traffic weight transitions.

### Example Scenario
```yaml
spec:
  replicas: 10
  strategy:
    canary:
      maxSurge: "25%"        # Relative to step delta, not total replicas
      maxUnavailable: "25%"  # Relative to step delta, not total replicas
      trafficRouting:
        istio: {}
      steps:
      - setWeight: 20        # Canary: 2, Stable: 8
      - pause: {duration: 10s}
      - setWeight: 30        # Canary: 3, Stable: 7 (Δ = +1, -1)
      - pause: {duration: 10s}
      - setWeight: 50        # Canary: 5, Stable: 5 (Δ = +2, -2)
```

### Relative Calculation Logic
```go
func calculateRelativeScalingLimits(rollout *v1alpha1.Rollout, prevWeight, newWeight int32) (maxSurgeDelta, maxUnavailableDelta int32) {
    // Calculate replica change from traffic weight shift
    prevCanary := trafficWeightToReplicas(rollout.Spec.Replicas, prevWeight, maxWeight)
    newCanary := trafficWeightToReplicas(rollout.Spec.Replicas, newWeight, maxWeight)
    replicaDelta := newCanary - prevCanary
    
    // Apply maxSurge/maxUnavailable as percentage of the delta
    maxSurgeAbs := calculatePercentage(MaxSurge(rollout), replicaDelta)
    maxUnavailableAbs := calculatePercentage(MaxUnavailable(rollout), replicaDelta)
    
    return maxSurgeAbs, maxUnavailableAbs
}
```

### Scaling Behavior
- **Step 1→2 (Δ = +1 canary):** maxSurge allows accelerating to +1.25 pods, maxUnavailable allows decelerating to +0.75 pods
- **Step 2→3 (Δ = +2 canary):** maxSurge allows accelerating to +2.5 pods, maxUnavailable allows decelerating to +1.5 pods

## Technical Implementation

### Required Changes
1. **State Tracking:** Store previous traffic weight in rollout status
2. **Delta Calculation:** Compute replica changes between steps
3. **Limit Application:** Apply maxSurge/maxUnavailable to the delta
4. **Transition Logic:** Smooth scaling between desired states

### Code Structure
```go
type TrafficScalingState struct {
    PreviousWeight     int32
    CurrentWeight      int32
    ScalingInProgress  bool
    TargetReplicaDelta int32
}

func (c *rolloutContext) reconcileTrafficRoutedScaling() {
    currentWeight := c.getCurrentSetWeight()
    prevWeight := c.getPreviousWeightFromStatus()
    
    if currentWeight != prevWeight {
        // Weight change detected - apply relative limits
        maxSurgeDelta, maxUnavailableDelta := calculateRelativeLimits(c.rollout, prevWeight, currentWeight)
        c.applyScalingLimits(maxSurgeDelta, maxUnavailableDelta)
    }
}
```

## Pros and Cons Analysis

### Advantages
- ✅ **Traffic-Aware:** Limits scale with traffic changes, not total replicas
- ✅ **Maintains Philosophy:** Still prioritizes traffic control over pod limits
- ✅ **Flexible:** Works with any step size, automatic adaptation
- ✅ **Backward Compatible:** Can be opt-in feature
- ✅ **Addresses User Pain:** Provides control without manual step micro-management

### Disadvantages
- ❌ **Complexity:** Requires state tracking and transition logic
- ❌ **Behavioral Difference:** Works differently from basic canary
- ❌ **Edge Cases:** Handling rollbacks, aborts, dynamic stable scaling
- ❌ **State Management:** Additional status fields and persistence
- ❌ **Testing:** Complex scenarios with weight changes, pauses, aborts

### User Experience Impact
- **Better:** Automatic scaling control without manual step tuning
- **Worse:** Different behavior from basic canary (cognitive load)
- **Unclear:** How limits interact with dynamic stable scaling

## Alternative: Absolute Limits with Traffic Awareness

### Simpler Approach
Keep maxSurge/maxUnavailable as absolute values but respect them during traffic transitions.

```go
func CalculateReplicaCountsForTrafficRoutedCanaryWithLimits(rollout *v1alpha1.Rollout, weights *v1alpha1.TrafficWeights) (int32, int32) {
    // Calculate desired counts from traffic weights
    desiredCanary := trafficWeightToReplicas(rollout.Spec.Replicas, desiredWeight, maxWeight)
    desiredStable := rollout.Spec.Replicas - desiredCanary
    
    // Apply absolute maxSurge/maxUnavailable limits
    maxTotalReplicas := rollout.Spec.Replicas + MaxSurge(rollout)
    minAvailableReplicas := rollout.Spec.Replicas - MaxUnavailable(rollout)
    
    // Adjust scaling to respect limits
    actualCanary, actualStable := applyAbsoluteLimits(desiredCanary, desiredStable, maxTotalReplicas, minAvailableReplicas)
    
    return actualCanary, actualStable
}
```

### Comparison
| Aspect | Relative Approach | Absolute Approach |
|--------|------------------|-------------------|
| Complexity | High | Medium |
| User Familiarity | Low (new behavior) | High (standard Kubernetes) |
| Traffic Integration | Excellent | Good |
| Implementation Risk | High | Medium |
| Addresses User Needs | Partial | Good |

## Recommendation

### Preferred Approach: Absolute Limits First
1. **Implement absolute maxSurge/maxUnavailable** respecting traffic routing
2. **Evaluate effectiveness** for user scaling concerns
3. **Consider relative approach** only if absolute limits prove insufficient

### Rationale
- Absolute limits maintain Kubernetes consistency
- Lower implementation risk and complexity
- Addresses most user scaling concerns
- Relative approach can be added later if needed

### Implementation Priority
1. **Immediate:** Add validation warnings for unsupported settings
2. **Short-term:** Implement absolute limits with traffic awareness
3. **Long-term:** Evaluate relative interpretation if absolute insufficient

# Relative maxSurge/maxUnavailable Interpretation

## Source-Backed Facts

### Current Implementation
- **No relative interpretation exists**: Both basic and traffic-routed canaries use absolute interpretation of maxSurge/maxUnavailable
- **Basic canary**: Limits apply to total rollout replicas (absolute interpretation)
- **Traffic-routed canary**: Limits are completely ignored (no interpretation)

### GitHub Issues Analysis
- **No proposals for relative interpretation**: Issues #3284, #3539, #3397, #2239 request maxSurge/maxUnavailable support but do not propose relative interpretation
- **Focus on absolute limits**: User requests center on respecting standard Kubernetes scaling limits in traffic-routed deployments

### Code Evidence
- `utils/replicaset/canary.go`: `CalculateReplicaCountsForBasicCanary` uses absolute limits
- `utils/replicaset/canary.go`: `CalculateReplicaCountsForTrafficRoutedCanary` ignores limits entirely
- No code implements relative interpretation of maxSurge/maxUnavailable

## Manual Exploration Required

### Technical Questions
- How would relative interpretation work with step-by-step traffic weight changes?
- What edge cases exist with rollbacks, aborts, and dynamic stable scaling?
- How does relative interpretation compare to absolute limits in practice?

### Design Questions
- Should relative interpretation be considered if absolute limits prove insufficient?
- What are the complexity trade-offs between relative and absolute approaches?
- How do other systems handle scaling limits in traffic-based deployments?