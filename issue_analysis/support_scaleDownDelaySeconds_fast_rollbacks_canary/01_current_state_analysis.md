# Current State Analysis: Support scaleDownDelaySeconds & fast rollbacks with canary strategy

## Issue Summary
The enhancement request seeks to add support for `scaleDownDelaySeconds` to canary deployments to enable fast rollback capabilities similar to what is available in blue-green deployments. Currently, argo-rollouts only supports fast-track rollback when a canary deployment is in progress. The goal is to keep the previous version around for `scaleDownDelaySeconds` to allow fast rollback for canary deployments in case metric checks don't catch regressions.

## Critical Code Locations

### Core Type Definitions
- **File:** `pkg/apis/rollouts/v1alpha1/types.go`
  - **Structure:** `BlueGreenStrategy` (lines 196-202)
  - **Purpose:** Defines `ScaleDownDelaySeconds` for blue-green deployments
  - **Structure:** `CanaryStrategy` (lines 308-318) 
  - **Purpose:** Defines `ScaleDownDelaySeconds` for canary deployments (only with traffic routing)
  - **Structure:** `RollbackWindowSpec` (lines 1215-1217)
  - **Purpose:** Defines rollback window configuration for fast-track rollbacks

### Scale Down Logic Implementation
- **File:** `rollout/canary.go`
  - **Function:** `scaleDownOldReplicaSetsForCanary` (lines 179-230)
  - **Purpose:** Handles scaling down old replica sets for canary deployments
  - **Function:** `scaleDownDelayHelper` (called within above)
  - **Purpose:** Applies scale down delay logic with annotations

- **File:** `rollout/replicaset.go`
  - **Function:** `addScaleDownDelay` (lines 50+)
  - **Purpose:** Adds scale-down-deadline annotation to ReplicaSets
  - **Function:** `shouldDelayScaleDownOnAbort` (lines 192+)
  - **Purpose:** Determines if scale down should be delayed on abort

### Default Value Handling  
- **File:** `utils/defaults/defaults.go`
  - **Function:** `GetScaleDownDelaySecondsOrDefault` (lines 182-200)
  - **Purpose:** Returns appropriate scale down delay based on deployment strategy
  - **Function:** `GetAbortScaleDownDelaySecondsOrDefault` (lines 213-239)
  - **Purpose:** Handles abort scale down delay logic

### Rollback Window Logic
- **File:** `rollout/sync.go`
  - **Function:** `isRollbackWithinWindow` (lines 902-928)
  - **Purpose:** Determines if a rollback is within the configured window
  - **Function:** `shouldFullPromote` (lines 940-996)
  - **Purpose:** Checks various conditions including rollback window for fast promotion

### Validation Logic
- **File:** `pkg/apis/rollouts/validation/validation.go`
  - **Lines:** 294-295, 301
  - **Purpose:** Validates that canary `ScaleDownDelaySeconds` can only be used with traffic routing

## Current Behavior

### Blue-Green Strategy
Blue-green deployments fully support `scaleDownDelaySeconds`:
- Default value: 30 seconds
- Keeps old ReplicaSet scaled up for the specified duration after promotion
- Enables fast rollback by rolling back to ReplicaSets still within their scale down delay
- Rollback window feature allows fast-track promotion without going through full blue-green process

### Canary Strategy
Canary deployments have **limited** support for `scaleDownDelaySeconds`:
- **With traffic routing:** `ScaleDownDelaySeconds` is supported (default 30 seconds)
- **Basic canary (no traffic routing):** `ScaleDownDelaySeconds` is **NOT supported** (validation error)
- Scale down delay helps with traffic provider re-targeting but doesn't enable the same fast rollback capabilities as blue-green
- Rollback window feature is supported but only works when stable ReplicaSet is still available

### Fast Rollback Mechanisms
Current fast rollback conditions:
1. **Rollback within window:** `isRollbackWithinWindow()` checks if rollback is within configured revision window
2. **Scale down deadline:** For blue-green, checks if target ReplicaSet still has scale down deadline annotation
3. **Active ReplicaSet availability:** Relies on ReplicaSets still being scaled up

## Expected Behavior
The issue requests that canary deployments should:
1. Keep previous version ReplicaSets scaled up for `scaleDownDelaySeconds` duration after promotion
2. Enable fast rollback to ReplicaSets within the scale down delay period
3. Provide feature parity with blue-green strategy for rollback capabilities
4. Potentially work without requiring traffic routing (basic canary support)

## Code Flow Analysis

### Current Canary Scale Down Flow
1. `scaleDownOldReplicaSetsForCanary()` is called during canary reconciliation
2. For traffic routing canaries, `scaleDownDelayHelper()` adds scale down deadline annotations
3. ReplicaSets with scale down deadline are kept running until deadline expires
4. Basic canaries immediately scale down old ReplicaSets without delay

### Current Fast Rollback Flow  
1. `shouldFullPromote()` checks if rollback is within window via `isRollbackWithinWindow()`
2. For blue-green, additional check via `HasScaleDownDeadline(c.newRS)` for fast-track
3. Fast rollback bypasses normal promotion steps and immediately promotes target ReplicaSet

### Gap in Current Implementation
- Canary strategy doesn't leverage scale down delay for fast rollback like blue-green does
- No mechanism to detect and fast-track rollbacks to ReplicaSets within scale down delay for canaries
- Basic canary doesn't support scale down delay at all

## Empirical Evidence from Production

### Real-World Testing Results
Based on production testing with Skyscanner's deployment infrastructure:

#### Test Scenario 1: Standard Canary Deployment
- **Configuration**: Traffic routing canary with default `scaleDownDelaySeconds: 30`
- **Observation**: Old ReplicaSet scaled down after exactly 30 seconds as expected
- **Controller logs**: Confirmed delay behavior with messages like:
  ```
  RS 'test-service-gritpoc-6754fd7678' has not reached the scaleDownTime
  ```
- **Result**: ✅ Scale down delay working correctly for traffic routing canary

#### Test Scenario 2: Rollback Within Window Inconsistencies
- **Configuration**: `rollbackWindow.revisions: 3` with rollback attempts
- **Inconsistent Behavior Observed**:
  - **Test 2a**: Fast rollback (6754fd7678 → d99d84ff) - Duration: 05:26 vs normal 06:58 (only 1.5min savings)
  - **Test 2b**: Same rollback direction created new revision despite same pod hash
  - **Test 2c**: Reverse rollback (d99d84ff → 6754fd7678) - Duration: 02:11 (significant improvement)
  - **Analysis reuse inconsistent**: Sometimes skipped, sometimes repeated even with identical pod hash

#### Test Scenario 3: Rollback Window Limitations
- **Critical Finding**: Rollback window counting appears revision-based, not commit-based
- **Confusion**: New revisions created even when pod template hash is identical
- **Question**: Does `revisions: 3` become too restrictive with frequent rollbacks?
- **Analysis Impact**: Analysis runs sometimes reused, sometimes not, affecting rollback speed

#### Test Scenario 4: Fast Rollback Within Scale Down Window
- **Expectation**: Rollback to ReplicaSet still within scaleDownDelaySeconds window
- **Challenge**: Production deployment pipeline too slow to reproduce within 30s window
- **Workaround**: Would require longer scale down delay (e.g., 10 minutes) for practical testing
- **Result**: ❌ Confirms the issue - window too short for practical fast rollback scenarios

### Production Impact Analysis
1. **30-second window insufficient**: Real deployment pipelines take longer than 30s, making fast rollback impractical
2. **Rollback window behavior inconsistent**: Analysis reuse and timing benefits vary unpredictably
3. **Revision counting complexity**: Rollback window may be counting internal revisions, not Git commits
4. **Scale down delay operational**: Traffic routing canaries correctly implement delay behavior
5. **Missing integration**: No mechanism to leverage scale down delay for fast rollback like blue-green
6. **Performance variance**: Rollback times range from 2-6 minutes depending on analysis reuse

## Key Findings

1. **Infrastructure exists**: Scale down delay mechanism is already implemented and used by canary with traffic routing
2. **Validation gap**: Basic canary is explicitly prevented from using `ScaleDownDelaySeconds` 
3. **Fast rollback gap**: Canary doesn't have equivalent fast rollback logic that blue-green has for ReplicaSets within scale down delay
4. **Feature parity issue**: Blue-green has more comprehensive fast rollback capabilities than canary
5. **Rollback window works**: The rollback window feature is strategy-agnostic and works for both blue-green and canary
6. **Traffic routing dependency**: Current implementation restricts scale down delay to traffic-routing scenarios only
7. **Production validation**: Real-world testing confirms 30s default window is too short for practical fast rollback scenarios
8. **Operational gap**: Current implementation doesn't leverage scale down annotations for canary fast rollback like blue-green does

## Implementation Approach Needed

To implement this feature, the following areas need modification:
1. Enhance canary fast rollback logic to check for scale down deadline similar to blue-green
2. Potentially remove validation restriction for basic canary scale down delay
3. Add logic to detect rollbacks to ReplicaSets within scale down delay window
4. Ensure scale down delay is properly applied during canary promotion
5. Add comprehensive testing for canary fast rollback scenarios