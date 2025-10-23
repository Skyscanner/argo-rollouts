# Relevance Assessment: maxSurge/maxUnavailable Ignored with Traffic Shifting

## Issue Relevance: HIGH

**Status**: Still present in current codebase.

## Technical Evidence

**Basic Canary**: `CalculateReplicaCountsForBasicCanary()` uses `MaxSurge(rollout)` to respect limits.

**Traffic-Routed Canary**: `CalculateReplicaCountsForTrafficRoutedCanary()` ignores maxSurge/maxUnavailable.

**Test Evidence**: Code review confirms traffic-routed tests don't validate surge behavior.

## Impact Analysis

**Operational Impact**: Excessive scaling affects cluster autoscaling efficiency and cost.

**Affected Users**: All using traffic-routed canaries (Istio, ALB, SMI) with configured scaling limits.

## Manual Exploration Required

**Investigate Further**:
- Review recent commits for any changes to scaling logic
- Test behavior with different traffic routing configurations
- Analyze impact on cluster autoscaling systems
- Examine user reports of scaling issues

**Key Question**: How does this issue affect real-world deployment costs and performance?