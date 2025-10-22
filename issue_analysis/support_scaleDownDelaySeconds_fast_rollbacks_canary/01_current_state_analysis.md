# Current State Analysis: Support scaleDownDelaySeconds & fast rollbacks with canary strategy

## Issue Summary
The enhancement request seeks to add support for `scaleDownDelaySeconds` to canary deployments to enable fast rollback capabilities similar to what is available in blue-green deployments. Currently, argo-rollouts only supports fast-track rollback when a canary deployment is in progress. The goal is to keep the previous version around for `scaleDownDelaySeconds` to allow fast rollback for canary deployments in case metric checks don't catch regressions.

**Updated Conclusion**: Based on empirical evidence, the optimal approach is **chart value optimization** rather than major upstream code changes. The existing implementation already supports the necessary mechanisms, but default values need production tuning. However, **upstream contributions could improve analysis run termination speed and canary pause step skipping**.

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

**Key Findings from Image Analysis:**
- `scaledown_1h_fastrollback_within_window.png`: Shows successful fast rollback within extended windows
- `scaledown_1h_fastrollback_within_window_multiple_rs_active_expected.png`: Confirms robust multiple RS management
- `scaledown_1h_fastrollback_within_window_ar_fastterminated.png`: Shows analysis run termination timing (20-30+ seconds)
- `scaledown_1h_fastrollforward_within_window_slow.png`: Confirms rollforward is intentionally slow (timestamp-based detection)
- `scaledown_1h_multiple_rs_on_rollback.png`: Validates rollback window mechanism

**Critical Insights:**
1. **Rollback vs Rollforward**: Fast rollback only works for true rollbacks (older RS timestamps), rollforward follows normal progression
2. **Analysis Run Termination**: Takes 20-30+ seconds for full cleanup when skipped
3. **Existing Mechanisms Work**: Scale down delay and rollback window logic function correctly
4. **Value Problem**: 30s defaults are insufficient for production operator response times

## Current Implementation Assessment

### What Works Well (Keep Existing Code)
**Robust Mechanisms:**
- Scale down delay annotations work correctly (`addScaleDownDelay`)
- Rollback window logic is sound (`isRollbackWithinWindow`)
- Multiple ReplicaSet management handles complex scenarios
- Timestamp-based rollback detection is accurate
- Traffic routing integration functions properly

### What Needs Optimization (Chart Values)
**Value Adjustments Needed:**
- `scaleDownDelaySeconds`: 30s → 300-600s (5-10 minutes for operator response)
- `abortScaleDownDelaySeconds`: 30s → 300s (5 minutes for investigation)
- Environment-specific recommendations (dev: 60-120s, staging: 180-300s, production: 300-600s)

### Potential Upstream Contributions
**Optional Enhancements:**
- **Analysis Run Termination**: Optimize cleanup time from 20-30+ seconds to faster termination
- **Canary Pause Skipping**: Improve logic for skipping pause steps during fast rollbacks
- **Resource Optimization**: Better cleanup of unused resources during extended delays

## Recommended Implementation Strategy

### Primary Approach: Chart Value Optimization
**Immediate Impact, Low Risk:**
1. Update Helm chart defaults to production-appropriate values
2. Provide comprehensive documentation and examples
3. Enable environment-specific configurations
4. Maintain backward compatibility

### Secondary Approach: Targeted Upstream Contributions
**Optional Enhancements:**
1. **Analysis Run Cleanup**: Investigate faster termination mechanisms
2. **Pause Step Optimization**: Improve canary pause skipping logic
3. **Resource Management**: Optimize resource usage during delays

### Implementation Priority
1. **Chart Optimization** (High Priority - Immediate Value)
2. **Documentation Excellence** (High Priority - Adoption Enablement)  
3. **Upstream Enhancements** (Medium Priority - Future Optimization)

## Conclusion

The existing Argo Rollouts implementation provides robust scale down delay and rollback window mechanisms. The primary issue is **inappropriate default values** for production environments. **Chart value optimization provides immediate production value** with minimal risk.

**Upstream contributions could further improve analysis run termination speed and canary pause step handling**, but these are secondary to the core value optimization needed for production reliability.