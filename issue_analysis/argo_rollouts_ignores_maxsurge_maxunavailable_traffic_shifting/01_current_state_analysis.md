# Current State Analysis: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Issue Summary
When traffic shifting is used in Argo Rollouts canary deployments, the `maxSurge` and `maxUnavailable` settings are ignored. This can impact cluster autoscaling by putting additional pressure on autoscalers like Karpenter to provision new nodes unnecessarily.

## Critical Code Locations
- **File:** `rollout/canary.go`
  - **Function:** `reconcileCanaryReplicaSets()` (line 443)
  - **Purpose:** Main reconciliation logic for canary ReplicaSets

- **File:** `utils/replicaset/replicaset.go`
  - **Function:** `NewRSNewReplicas()` (line 251)
  - **Purpose:** Determines replica count for new ReplicaSet based on strategy

- **File:** `utils/replicaset/canary.go`
  - **Function:** `CalculateReplicaCountsForBasicCanary()` (line 94)
  - **Purpose:** Calculates replica counts for basic canary (respects maxSurge/maxUnavailable)

- **File:** `utils/replicaset/canary.go`
  - **Function:** `CalculateReplicaCountsForTrafficRoutedCanary()` (line 341)
  - **Purpose:** Calculates replica counts for traffic-routed canary (ignores maxSurge/maxUnavailable)

- **File:** `utils/replicaset/replicaset.go`
  - **Function:** `MaxSurge()` (line 451)
  - **Purpose:** Returns maxSurge value for rollout

## Current Behavior
The rollout controller uses different logic for calculating replica counts based on whether traffic routing is enabled:

1. **Basic Canary (TrafficRouting == nil):**
   - Uses `CalculateReplicaCountsForBasicCanary()`
   - Respects `maxSurge` and `maxUnavailable` settings
   - Calculates `maxReplicaCountAllowed = rolloutSpecReplica + maxSurge`
   - Limits total replica count across all ReplicaSets

2. **Traffic-Routed Canary (TrafficRouting != nil):**
   - Uses `CalculateReplicaCountsForTrafficRoutedCanary()`
   - Ignores `maxSurge` and `maxUnavailable` settings
   - Calculates replica counts based solely on traffic weights
   - Can exceed total desired replicas + maxSurge

## Expected Behavior
Traffic-routed canary deployments should also respect `maxSurge` and `maxUnavailable` settings to prevent excessive scaling that impacts cluster autoscaling.

## Code Flow Analysis
1. `rolloutCanary()` calls `reconcileCanaryReplicaSets()`
2. `reconcileCanaryReplicaSets()` calls `reconcileNewReplicaSet()`
3. `reconcileNewReplicaSet()` calls `NewRSNewReplicas()` to get target replica count
4. `NewRSNewReplicas()` branches based on traffic routing:
   - Basic canary: `CalculateReplicaCountsForBasicCanary()` → respects maxSurge
   - Traffic-routed canary: `CalculateReplicaCountsForTrafficRoutedCanary()` → ignores maxSurge

## Key Findings
- **Root Cause:** `CalculateReplicaCountsForTrafficRoutedCanary()` does not implement maxSurge/maxUnavailable logic
- **Impact:** Traffic-routed canaries can scale beyond configured limits
- **Affected Scenarios:** Any canary rollout using traffic routing (Istio, ALB, SMI, etc.)
- **Current Workaround:** None - users must choose between traffic routing and surge control</content>
<parameter name="filePath">/Users/nebojsaprodana/dev/skyscanner/skyscanner-ghec/argo-rollouts/issue_analysis/argo_rollouts_ignores_maxsurge_maxunavailable_traffic_shifting/01_current_state_analysis.md