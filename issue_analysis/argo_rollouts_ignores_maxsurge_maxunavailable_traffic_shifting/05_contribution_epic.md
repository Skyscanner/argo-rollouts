# Contribution Epic: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Epic Overview
**Objective:** Address community demand for maxSurge/maxUnavailable support in traffic-routed canary deployments, despite this being an intentional design decision that prioritizes traffic control over scaling limits.

## Current Workarounds Analysis

### MinPodsPerReplicaSet as Partial Solution
**How it helps:** `MinPodsPerReplicaSet` provides a minimum floor for pod counts in traffic-routed canaries, preventing complete scale-down during low-traffic phases.

**Code Implementation:**
```go
func CheckMinPodsPerReplicaSet(rollout *v1alpha1.Rollout, count int32) int32 {
    if count == 0 {
        return count
    }
    if rollout.Spec.Strategy.Canary == nil || rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet == nil || rollout.Spec.Strategy.Canary.TrafficRouting == nil {
        return count
    }
    return max(count, *rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet)
}
```

**Limitations:**
- Only sets minimum pods, doesn't control maximum surge
- Doesn't prevent rapid scaling between traffic weights
- Requires manual tuning per rollout

### Manual Canary Steps as Current Alternative
**User Concern:** Without maxSurge/maxUnavailable controls, users must manually add multiple small canary steps to control scaling speed.

**Example Manual Approach:**
```yaml
steps:
- setWeight: 5   # Small increment
- pause: {duration: 60s}
- setWeight: 10  # Another small increment
- pause: {duration: 60s}
# ... many more steps instead of smooth progression
```

**Problems with Manual Approach:**
- Verbose rollout definitions
- Hard to maintain and tune
- Doesn't adapt to different traffic patterns
- Manual process prone to errors

## Possible Exploration Directions

### Design Philosophy Evaluation
- Investigate whether the current traffic routing design philosophy should be revisited
- Explore community consensus on maxSurge/maxUnavailable support
- Assess the impact of dynamicStableScale on implementation feasibility

### Technical Implementation Approaches
- Examine relative percentage interpretation between canary steps
- Compare absolute vs relative approaches for scaling limits
- Evaluate integration with existing traffic control logic

### Workaround Enhancement
- Improve MinPodsPerReplicaSet documentation and best practices
- Create infrastructure scaling best practices for traffic routing
- Develop better guidance for manual canary step approaches

## Open Questions for Further Exploration

- Should the design philosophy be revisited to support maxSurge/maxUnavailable?
- How does dynamicStableScale impact implementation feasibility?
- What are the trade-offs between absolute and relative scaling interpretations?
- Can MinPodsPerReplicaSet be enhanced to better address scaling concerns?
- What validation and warnings should be provided for unsupported configurations?

## Success Criteria
1. Clear understanding of whether maxSurge/maxUnavailable support is feasible and desirable
2. Improved documentation and guidance for current workarounds
3. Community consensus on design direction
4. Enhanced user experience for scaling control in traffic-routed deployments

This epic provides initial directions for exploration while leaving significant room for discovery, community discussion, and design iteration.