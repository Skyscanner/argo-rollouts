# Issue Relevance Assessment: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Assessment Date
22 October 2025

## Current Status
- [x] Issue is still present
- [ ] Issue has been resolved
- [ ] Issue is partially resolved
- [ ] Issue is no longer relevant

## Evidence
The issue remains present in the current codebase. Analysis of the code shows that:

1. **Basic Canary Logic** (`CalculateReplicaCountsForBasicCanary`):
   - Uses `MaxSurge(rollout)` function to respect surge limits
   - Calculates `maxReplicaCountAllowed = rolloutSpecReplica + maxSurge`
   - Limits total replica count across all ReplicaSets

2. **Traffic-Routed Canary Logic** (`CalculateReplicaCountsForTrafficRoutedCanary`):
   - Does NOT use `MaxSurge()` or `maxUnavailable` settings
   - Calculates replica counts based solely on traffic weights
   - Can exceed `rolloutSpecReplica + maxSurge` limits

## Testing Results
Code review confirms the issue exists:

- `CalculateReplicaCountsForTrafficRoutedCanary()` in `utils/replicaset/canary.go` (line 341) has no reference to maxSurge/maxUnavailable
- `CalculateReplicaCountsForBasicCanary()` in the same file (line 94) properly uses `MaxSurge(rollout)` 
- Test cases in `canary_test.go` with `trafficRouting` set do not validate maxSurge behavior

## Recent Changes Review
No recent commits appear to have addressed this issue. The core logic in `CalculateReplicaCountsForTrafficRoutedCanary` remains unchanged and still ignores surge limits.

## Conclusion
This issue is **STILL RELEVANT** and affects all Argo Rollouts users who:
- Use canary deployments with traffic routing (Istio, ALB, SMI, etc.)
- Configure `maxSurge` or `maxUnavailable` settings
- Rely on cluster autoscaling to manage node provisioning

The issue can cause excessive scaling that impacts cluster autoscaling efficiency and cost.</content>
<parameter name="filePath">/Users/nebojsaprodana/dev/skyscanner/skyscanner-ghec/argo-rollouts/issue_analysis/argo_rollouts_ignores_maxsurge_maxunavailable_traffic_shifting/02_relevance_assessment.md