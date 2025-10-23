# Upstream Analysis: Argo Rollouts Simultaneous 100% Availability Requirement

## Technical Root Cause

**Argo Rollouts Issue**: `RolloutHealthy` function requires 100% pod availability across ALL ReplicaSets simultaneously.

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

**Failure Scenario**: With autoscaling, stable RS may be scaled to 0 pods while canary RS is healthy, making `AvailableReplicas < replicas` permanently true.

## Documented GitHub Issues

**Issue #4294**: "BlueGreen Rollout Stuck in Progressing Blocking new Rollout"
- Rollout stuck after HPA scaling during deployment
- New RS scales fully but never promoted to stable
- Manual interventions fail

**Issue #3316**: "Rollout stuck issue"
- References PRs #3272, #3257, issue #3256
- Occurs with `dynamicStableScale: true` and replica changes
- `FindNewReplicaSet` returns nil due to scaling disruption

**Issue #3256**: "Rollout stuck when changing replicas with dynamicStableScale"
- Reproduction: Enable `dynamicStableScale`, change replicas during rollout
- Result: Permanent stuck state

**Issue #3304**: "BlueGreen Rollbacks get stuck... when setting rollbackWindow configuration"
- Rollback operations fail when availability requirements unmet

## Industry Comparison: Phased vs Simultaneous Availability

### Flagger: Phased Validation Logic

**Configuration** (from Flagger docs + user research):
```yaml
spec:
  analysis:
    primaryReadyThreshold: 100  # 100% threshold
    canaryReadyThreshold: 100   # 100% threshold
```

**Architectural Difference**: Validates canary and primary ReplicaSets separately with phased progression, not simultaneously.

**Source**: [Flagger Documentation](https://docs.flagger.app/usage/how-it-works)

### Kubernetes Deployments: Percentage-Based Availability

**Default Behavior**:
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 25%  # Allows 75% minimum availability
    maxSurge: 25%
```

**Key Difference**: Deployments are "available" with only 75% of pods running, not requiring 100% across all RS simultaneously.

**Source**: [Kubernetes Deployment API](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

## Manual Exploration Required

**Investigate Further**:
- Review all referenced GitHub issues for additional reproduction cases
- Examine Flagger's rollout state machine vs Argo Rollouts' health checks
- Compare with Keptn's evaluation logic
- Analyze how service meshes (Istio/Linkerd) handle phased availability

**Key Question**: Why does Argo Rollouts require simultaneous availability when industry standard is phased validation?

**Sources to Review**:
- Argo Rollouts controller logic in `controller/rollout_controller.go`
- Kubernetes deployment controller availability calculations
- Flagger's canary deployment state transitions