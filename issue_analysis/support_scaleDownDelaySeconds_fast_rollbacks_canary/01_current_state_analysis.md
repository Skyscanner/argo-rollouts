# Current State Analysis: scaleDownDelaySeconds & Fast Rollbacks

## Issue Summary

**Request**: Add `scaleDownDelaySeconds` support to canary deployments for fast rollback parity with blue-green.

**Current Limitation**: `scaleDownDelaySeconds` only works with traffic-routing canaries, blocked for basic canary by validation.

## Critical Code Locations

**Type Definitions**:
- `pkg/apis/rollouts/v1alpha1/types.go` - ScaleDownDelaySeconds fields for both strategies
- `pkg/apis/rollouts/validation/validation.go#L294-295` - Validation blocking basic canary

**Scale Down Logic**:
- `rollout/canary.go` - scaleDownOldReplicaSetsForCanary function
- `rollout/replicaset.go` - addScaleDownDelay annotation logic

**Default Values**:
- `utils/defaults/defaults.go` - GetScaleDownDelaySecondsOrDefault (returns 30s)

**Rollback Logic**:
- `rollout/sync.go#L902-L996` - isRollbackWithinWindow and shouldFullPromote

## Current Behavior

**Blue-Green**: Full scaleDownDelaySeconds support with fast rollback via scale-down-deadline detection.

**Canary (Traffic Routing)**: scaleDownDelaySeconds supported but no fast rollback mechanism like blue-green.

**Canary (Basic)**: Validation error prevents scaleDownDelaySeconds usage.

## Code Flow Analysis

**Current Canary Scale Down**:
1. `scaleDownOldReplicaSetsForCanary()` called during reconciliation
2. Traffic routing canaries get scale-down-deadline annotations
3. Basic canaries scale down immediately (no delay)

**Fast Rollback Detection**:
1. `shouldFullPromote()` checks rollback window via `isRollbackWithinWindow()`
2. Blue-green additionally checks `HasScaleDownDeadline()` for instant promotion
3. Canary lacks equivalent fast rollback logic

## Empirical Evidence

**Production Testing**:
- 30s defaults insufficient for operator response times
- Rollback window works but requires proper configuration
- Analysis runs terminate in 20-30+ seconds when skipped

## Current Implementation Assessment

**What Works**: Scale down delay annotations, rollback window logic, multiple RS management.

**What Needs Optimization**: Default values (30s → 300-600s), documentation, configuration guidance.

**Potential Enhancements**: Faster analysis termination, improved pause skipping.

## Manual Exploration Required

**Investigate Further**:
- Review validation logic rationale for traffic-routing requirement
- Examine blue-green fast rollback implementation details
- Test rollback window behavior with different configurations
- Analyze analysis run lifecycle during rollbacks

**Key Question**: What prevents scaleDownDelaySeconds from working with basic canary deployments?