# Current State Analysis: maxSurge/maxUnavailable Ignored with Traffic Shifting

## Issue Summary

Traffic-routed canary deployments ignore `maxSurge` and `maxUnavailable` settings, unlike basic canary deployments.

## Critical Code Locations

**Basic Canary**: `utils/replicaset/canary.go:CalculateReplicaCountsForBasicCanary()` - respects maxSurge/maxUnavailable.

**Traffic-Routed Canary**: `utils/replicaset/canary.go:CalculateReplicaCountsForTrafficRoutedCanary()` - ignores these settings.

**Code Flow**:
1. `rolloutCanary()` → `reconcileCanaryReplicaSets()`
2. `reconcileNewReplicaSet()` → `NewRSNewReplicas()`
3. Branches based on traffic routing presence

## Current Behavior

**Basic Canary**: Uses `MaxSurge(rollout)` to limit total replica count.

**Traffic-Routed Canary**: Calculates replicas based solely on traffic weights, can exceed configured limits.

## Expected Behavior

Traffic-routed canaries should respect maxSurge/maxUnavailable to prevent excessive scaling.

## Key Findings

- **Root Cause**: `CalculateReplicaCountsForTrafficRoutedCanary()` lacks maxSurge/maxUnavailable logic
- **Impact**: Unbounded scaling potential in traffic-routed deployments
- **Affected Scenarios**: All canary rollouts with traffic routing (Istio, ALB, SMI)

## Manual Exploration Required

**Investigate Further**:
- Compare code implementations between basic and traffic-routed canary functions
- Review test cases to confirm behavior differences
- Examine how traffic weights drive scaling decisions
- Test scaling behavior with different traffic routing providers

**Key Question**: Why do the two canary implementations have different scaling logic?