# Historical Analysis: Evolution of Health Check Logic

## Code Evolution

### Early Versions: Basic Health Checks
```go
func RolloutHealthy(rollout *v1alpha1.Rollout) bool {
    return allPodsReady && allStepsExecuted
}
```

### Current: Complex but Binary
```go
func RolloutHealthy(rollout *v1alpha1.Rollout) bool {
    completedStrategy := executedAllSteps && currentRSIsStable && scaleDownOldReplicas
    return newStatus.AvailableReplicas == replicas && completedStrategy  // Still requires 100%
}
```

**Missing Evolution**: No shift to percentage-based logic despite scaling complexity increases.

## Design Decisions

**Original Mindset**: Conservative safety-first approach with 100% availability requirements.

**Trade-offs**: Safety prioritized over efficiency and autoscaling compatibility.

## Community Evolution

**Issue Timeline**:
- 2020-2021: Basic scaling issues
- 2022: Autoscaling conflicts emerge
- 2023: PDB-related deadlocks
- 2024: Large-scale deployment failures

## Manual Exploration Required

**Investigate Further**:
- Review git history of RolloutHealthy function changes
- Examine when autoscaling conflicts first appeared in issues
- Compare evolution with Kubernetes Deployment health logic
- Analyze design decisions in original PRs and issues

**Key Question**: When did the autoscaling incompatibility become apparent in the codebase?