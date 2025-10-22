# Current State Analysis: Argo-rollouts waits for stable RS to be stable before scaling it down

## Issue Description

**Updated Understanding**: The issue is more fundamental than initially analyzed. Argo Rollouts requires BOTH canary AND stable ReplicaSets to be fully ready and available before considering a rollout complete. This creates an impossible situation for large-scale services with 100s of pods, Pod Disruption Budgets (PDBs), and autoscaling clusters backed by spot instances.

**Core Problem**: The health checking logic expects 100% availability of all pods in both ReplicaSets, which conflicts with:
- **Pod Disruption Budgets**: Prevent simultaneous pod disruptions
- **Spot Instances**: Can be preempted causing pod unavailability
- **Cluster Autoscaling**: May not have capacity for full pod counts
- **Large Scale**: 100+ pods make 100% availability unrealistic

## Technical Root Cause

### Health Checking Logic
The issue lies in multiple places where Argo Rollouts expects full pod availability:

#### 1. RolloutHealthy Condition (Primary Issue)
```go
// utils/conditions/conditions.go
func RolloutHealthy(rollout *v1alpha1.Rollout, rsList []*appsv1.ReplicaSet) bool {
    // ... existing logic ...
    scaleDownOldReplicas := allOldRSFullyScaledAndReady(rsList)
    completedStrategy := executedAllSteps && currentRSIsStable && scaleDownOldReplicas
    // ...
}
```

**Problem**: `scaleDownOldReplicas` requires old RS to be fully scaled down AND fully ready, which means all pods must be running and available.

#### 2. ReplicaSet Stability Checks
```go
// Multiple locations check for full RS readiness
func isReplicaSetStable(rs *appsv1.ReplicaSet) bool {
    return rs.Status.ReadyReplicas == rs.Status.Replicas
}
```

**Problem**: Requires 100% of pods to be ready, incompatible with autoscaling and PDBs.

#### 3. Scale Down Logic
```go
// utils/replicaset/canary.go
func CalculateReplicaCountsForTrafficRoutedCanary(...) {
    // Logic assumes full availability of stable RS
    // when dynamicStableScale=false
}
```

**Problem**: Even when stable RS isn't scaled, the health check still waits for full readiness.

## Impact on Autoscaling Scenarios

### DynamicStableScale = true
- **Current Behavior**: Scales down stable RS during rollout, then waits for canary to be fully ready
- **Problem**: Still requires 100% canary pod availability
- **Autoscaling Conflict**: Cluster may not have capacity for full canary scale

### DynamicStableScale = false (Default)
- **Current Behavior**: Keeps stable RS at full scale, waits for canary to be fully ready
- **Problem**: Requires both RS to be 100% available simultaneously
- **Autoscaling Conflict**: Double the resource requirement during transition

## Interplay with Kubernetes Deployment Controls

### MaxSurge and MaxUnavailable
Kubernetes Deployments use these controls to manage rollout pacing:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

**Current Argo Rollouts Issue**: Ignores these controls and demands 100% availability of both RS.

### Proposed Solution: MinHealthy Percentage
Introduce a `minHealthy` parameter (e.g., 80%) that allows rollouts to proceed when sufficient pods are healthy:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      minHealthy: 80%  # New parameter
      maxSurge: 25%
      maxUnavailable: 25%
```

## Technical Implementation Requirements

### 1. Health Check Refactoring
Replace binary (all-or-nothing) health checks with percentage-based logic:

```go
func isRolloutHealthyWithPercentage(rollout *v1alpha1.Rollout, rsList []*appsv1.ReplicaSet) bool {
    minHealthyPercent := parseMinHealthy(rollout.Spec.Strategy.Canary.MinHealthy)

    for _, rs := range rsList {
        healthyPercent := calculateHealthyPercentage(rs)
        if healthyPercent < minHealthyPercent {
            return false
        }
    }
    return true
}
```

### 2. Scale Down Logic Updates
Modify scale-down decisions to work with percentage-based health:

```go
func shouldScaleDownOldRS(rollout *v1alpha1.Rollout, oldRS, newRS *appsv1.ReplicaSet) bool {
    // Consider minHealthy instead of requiring full scale-down
    oldHealthyPercent := calculateHealthyPercentage(oldRS)
    newHealthyPercent := calculateHealthyPercentage(newRS)

    return oldHealthyPercent >= minHealthy && newHealthyPercent >= minHealthy
}
```

### 3. Autoscaling Integration
Account for cluster autoscaling capacity limitations:

```go
func calculateEffectiveReplicaCount(desired int32, clusterCapacity int32, minHealthy float64) int32 {
    // Don't demand more than cluster can provide
    maxAvailable := int32(float64(clusterCapacity) * minHealthy)
    return min(desired, maxAvailable)
}
```

## Affected Code Locations

### Core Files to Modify
1. **`utils/conditions/conditions.go`** - RolloutHealthy function
2. **`utils/replicaset/canary.go`** - Scaling calculations
3. **`rollout/canary.go`** - Rollout progression logic
4. **`pkg/apis/rollout/v1alpha1/types.go`** - Add minHealthy field

### Test Files to Update
1. **`utils/conditions/conditions_test.go`**
2. **`utils/replicaset/canary_test.go`**
3. **`rollout/canary_test.go`**

## Edge Cases and Considerations

### Large-Scale Deployment Challenges
- **Pod Disruption Budgets**: Must respect PDB constraints
- **Spot Instance Preemption**: Handle pod loss gracefully
- **Cluster Autoscaling Delays**: Account for scaling lag time
- **Network Partitioning**: Handle partial cluster failures

### Rollback Scenarios
- **Failed Rollout**: Must handle percentage-based rollback
- **Resource Constraints**: Rollback when cluster can't support full scale
- **PDB Conflicts**: Rollback when PDBs prevent scaling

### Traffic Routing Integration
- **Service Mesh**: Ensure traffic routing works with partial availability
- **Load Balancing**: Handle uneven pod distribution
- **Health Checks**: Align with service mesh health checking

## Current Workarounds

### Existing Mitigations
1. **Smaller Rollouts**: Use smaller replica counts to avoid scaling issues
2. **Manual Intervention**: Manually scale RS when stuck
3. **Disable PDBs**: Temporarily remove PDBs (dangerous)
4. **Over-provisioning**: Maintain excess cluster capacity

### Limitations of Workarounds
- **Scalability**: Don't work for large-scale deployments
- **Automation**: Manual intervention defeats CI/CD automation
- **Safety**: Disabling PDBs reduces deployment safety
- **Cost**: Over-provisioning increases infrastructure costs

## Summary

The issue is more complex than initially analyzed. It's not just about scale-down timing, but about fundamental incompatibility between Argo Rollouts' strict health requirements and the realities of large-scale autoscaling deployments. The solution requires implementing percentage-based health checking that respects `maxSurge`/`maxUnavailable` limits and works with cluster autoscaling constraints.

**Key Technical Challenge**: Replace binary health checks with percentage-based logic while maintaining rollout safety and traffic routing integrity.