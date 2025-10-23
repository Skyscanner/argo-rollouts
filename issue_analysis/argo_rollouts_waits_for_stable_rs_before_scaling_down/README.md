# Argo Rollouts Issue Analysis: Simultaneous 100% Availability Requirement

## Overview

**Core Problem**: Argo Rollouts requires 100% pod availability across ALL ReplicaSets simultaneously, causing deadlocks in autoscaling environments.

**Technical Root Cause**: `RolloutHealthy` function demands `AvailableReplicas == totalReplicas` across all ReplicaSets.

**Impact**: Blocks large-scale deployments with autoscaling, PDBs, and spot instances.

## Documented Issues

| Issue # | Title | Evidence |
|---------|-------|----------|
| #4294 | BlueGreen Rollout Stuck in Progressing | Rollout stuck after HPA scaling |
| #3316 | Rollout stuck issue | References dynamicStableScale conflicts |
| #3256 | Rollout stuck when changing replicas | Permanent stuck state with scaling |
| #3304 | BlueGreen Rollbacks get stuck | Rollback failures with availability requirements |

## Technical Details

**Code Location**: `utils/conditions/conditions.go:RolloutHealthy`

```go
return newStatus.UpdatedReplicas == replicas &&
    newStatus.AvailableReplicas == replicas &&  // ← Requires 100% total availability
    rollout.Status.ObservedGeneration == strconv.Itoa(int(rollout.Generation)) &&
    completedStrategy
```

**Failure Scenario**: With autoscaling, stable RS may be scaled to 0 while canary is healthy, making `AvailableReplicas < replicas` permanent.

## Industry Comparison

**Argo Rollouts**: Requires 100% availability across all ReplicaSets simultaneously.

**Kubernetes Deployments**: Available with 75% of pods (MaxUnavailable: 25%).

**Flagger**: Uses 100% thresholds but validates ReplicaSets separately (phased approach).

## Analysis Files

- `01_current_state_analysis.md` - Technical root cause and code locations
- `02_relevance_assessment.md` - Business impact and priority assessment
- `03_historical_analysis.md` - Evolution of scaling logic and design decisions
- `04_upstream_analysis.md` - Comparison with other tools and external sources
- `05_contribution_epic.md` - Exploration directions for solutions
- `07_final_analysis_summary.md` - Strategic recommendations

## Manual Exploration Required

**Investigate Further**:
- Review all referenced GitHub issues for reproduction cases
- Examine Flagger's phased validation vs Argo Rollouts' simultaneous requirements
- Compare with Kubernetes Deployment availability logic
- Analyze how service meshes handle partial availability

**Key Question**: Why does Argo Rollouts require simultaneous availability when industry standard is phased validation?