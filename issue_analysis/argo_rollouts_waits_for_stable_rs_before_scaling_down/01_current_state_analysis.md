# Current State Analysis: Simultaneous 100% Availability Requirement

## Technical Root Cause

**Primary Issue**: `RolloutHealthy` function requires 100% pod availability across ALL ReplicaSets simultaneously.

**Code Evidence**: `utils/conditions/conditions.go:286-312`
```go
func RolloutHealthy(rollout *v1alpha1.Rollout, newStatus *v1alpha1.RolloutStatus) bool {
    // ... strategy completion logic ...
    return newStatus.UpdatedReplicas == replicas &&
        newStatus.AvailableReplicas == replicas &&  // ← Requires 100% total availability
        rollout.Status.ObservedGeneration == strconv.Itoa(int(rollout.Generation)) &&
        completedStrategy
}
```

**Failure Scenario**: With autoscaling, stable RS may be scaled to 0 pods while canary RS is healthy, making `AvailableReplicas < replicas` permanent.

## Impact on Autoscaling Scenarios

### DynamicStableScale = true
- Scales down stable RS during rollout
- Still requires 100% canary pod availability
- Cluster may lack capacity for full canary scale

### DynamicStableScale = false (Default)
- Keeps stable RS at full scale
- Requires both RS at 100% simultaneously
- Double capacity requirement during transition

## Affected Code Locations

**Core Files to Examine**:
- `utils/conditions/conditions.go` - RolloutHealthy function
- `utils/replicaset/canary.go` - Scaling calculations
- `rollout/canary.go` - Rollout progression logic
- `pkg/apis/rollout/v1alpha1/types.go` - CRD definitions

## Manual Exploration Required

**Investigate Further**:
- Review RolloutHealthy function implementation details
- Examine how scaling decisions interact with health checks
- Analyze test cases for autoscaling scenarios
- Compare with Kubernetes Deployment health logic

**Key Question**: How do health checks interact with scaling decisions in different rollout phases?