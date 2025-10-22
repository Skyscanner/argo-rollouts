# Contribution Epic: Support scaleDownDelaySeconds & fast rollbacks with canary strategy

## Epic Overview
This epic focuses on enabling fast rollback capabilities for canary deployments through `scaleDownDelaySeconds` support. The primary goal is to provide feature parity with blue-green deployments for rollback scenarios.

**Updated Strategic Direction**: **Chart value optimization is the primary solution** with optional upstream contributions for performance improvements.

## Strategic Approach

### Primary Solution: Chart Value Optimization (High Priority)
**Immediate Production Value:**
- Update Helm chart defaults from 30s to 300-600s for production environments
- Provide environment-specific configuration examples
- Enable fast rollbacks through existing mechanisms with appropriate timing
- Zero upstream code changes required

### Secondary Solution: Upstream Contributions (Medium Priority)
**Optional Performance Enhancements:**
- Optimize analysis run termination speed (currently 20-30+ seconds)
- Improve canary pause step skipping logic during fast rollbacks
- Enhance resource cleanup during extended scale down delays

## Technical Implementation Details

### Current Code Architecture
**Scale Down Delay Implementation**:
```go
// rollout/canary.go - Current scale down logic
func scaleDownOldReplicaSetsForCanary(c *rolloutContext) error {
    if c.rollout.Spec.Strategy.Canary.TrafficRouting != nil {
        delaySeconds := defaults.GetScaleDownDelaySecondsOrDefault(c.rollout)
        for _, rs := range oldRSs {
            replicasetutil.AddScaleDownDelay(rs, delaySeconds)
        }
    }
    return nil
}
```

**Validation Restrictions**:
```go
// pkg/apis/rollouts/validation/validation.go
if canary.TrafficRouting == nil {
    if canary.ScaleDownDelaySeconds != nil {
        allErrs = append(allErrs, field.Invalid(fldPath.Child("scaleDownDelaySeconds"), 
            *canary.ScaleDownDelaySeconds, InvalidCanaryScaleDownDelay))
    }
}
```

**Rollback Window Logic**:
```go
// rollout/sync.go - Rollback window detection
func (c *rolloutContext) isRollbackWithinWindow() bool {
    if c.rollout.Spec.RollbackWindow == nil {
        return false
    }
    // Compare ReplicaSet creation timestamps
    return c.newRS.CreationTimestamp.After(c.stableRS.CreationTimestamp)
}
```

## Implementation Strategy

### Phase 1: Chart Optimization (Immediate)
**Deliverables:**
1. **Helm Chart Updates**
   - Update default `scaleDownDelaySeconds` to 300s (5 minutes)
   - Update default `abortScaleDownDelaySeconds` to 300s (5 minutes)
   - Add environment-specific value recommendations

2. **Documentation Excellence**
   - Comprehensive rollback window documentation
   - Production configuration examples
   - Troubleshooting guides for rollback scenarios

3. **Validation Testing**
   - Production environment testing with new values
   - Rollback window validation across scenarios
   - Performance impact assessment

### Phase 2: Upstream Enhancements (Optional)
**Potential Contributions:**
1. **Analysis Run Optimization**
   - Investigate faster termination mechanisms
   - Reduce cleanup time from 20-30+ seconds
   - Optimize resource usage during termination

2. **Canary Pause Logic**
   - Improve pause step skipping during fast rollbacks
   - Enhance rollback detection logic
   - Optimize step progression for rollback scenarios

## Detailed Technical Implementation

### Chart Value Optimization (Primary)
**No Code Changes Required:**
- Leverage existing `scaleDownDelaySeconds` support for traffic-routing canaries
- Utilize existing rollback window logic in `rollout/sync.go`
- Maintain current validation logic (traffic routing requirement)

**Configuration Changes:**
```yaml
# Production recommended values
scaleDownDelaySeconds: 300    # 5 minutes for operator response
abortScaleDownDelaySeconds: 300  # 5 minutes for investigation

# Environment-specific recommendations
development:
  scaleDownDelaySeconds: 60   # 1 minute
  abortScaleDownDelaySeconds: 60

staging:
  scaleDownDelaySeconds: 180  # 3 minutes
  abortScaleDownDelaySeconds: 180

production:
  scaleDownDelaySeconds: 600  # 10 minutes
  abortScaleDownDelaySeconds: 300  # 5 minutes
```

### Upstream Enhancement Opportunities (Secondary)

#### Analysis Run Termination Optimization
**Current State:** Analysis runs take 20-30+ seconds to fully terminate when skipped
**Potential Improvements:**
- Parallel cleanup of analysis resources
- Optimized termination signaling
- Reduced resource hold times

**Code Areas:**
- `experiments/controller.go`: Analysis run lifecycle management
- `analysis/controller.go`: Analysis execution and cleanup
- Integration points in `rollout/canary.go`

**Implementation Approach:**
```go
// Potential enhancement in experiments/controller.go
func (c *Controller) terminateAnalysisRun(ar *v1alpha1.AnalysisRun) error {
    // Current: Sequential cleanup
    // Potential: Parallel cleanup with goroutines
    // Potential: Optimized status updates
}
```

#### Canary Pause Step Optimization
**Current State:** Pause steps may not be optimally skipped during fast rollbacks
**Potential Improvements:**
- Enhanced rollback detection in pause logic
- Improved step progression for rollback scenarios
- Better integration with rollback window logic

**Code Areas:**
- `rollout/canary.go`: Pause step handling
- `rollout/sync.go`: Rollback window integration
- Step progression logic in rollout controller

**Implementation Approach:**
```go
// Potential enhancement in rollout/canary.go
func (c *rolloutContext) shouldSkipPauseForRollback() bool {
    // Current: Basic pause logic
    // Potential: Check rollback window and scale down deadlines
    // Potential: Integrate with fast rollback detection
    return c.isRollbackWithinWindow() || replicasetutil.HasScaleDownDeadline(c.newRS)
}
```

## Risk Assessment

### Chart Optimization (Low Risk)
**Risk Level:** Low
**Impact:** Immediate production benefit
**Complexity:** Configuration changes only
**Testing:** Existing mechanisms, value validation only

### Upstream Contributions (Medium Risk)
**Risk Level:** Medium
**Impact:** Performance improvements
**Complexity:** Code changes across multiple controllers
**Testing:** Requires comprehensive integration testing

## Success Criteria

### Primary Success (Chart Optimization)
- [ ] Helm chart defaults updated with production-appropriate values
- [ ] Documentation provides clear rollback window guidance
- [ ] Production testing validates rollback capabilities
- [ ] Environment-specific configurations documented

### Secondary Success (Upstream Enhancements)
- [ ] Analysis run termination time reduced by 50%
- [ ] Canary pause steps optimally skipped during rollbacks
- [ ] Resource usage optimized during scale down delays

## Timeline and Milestones

### Phase 1: Chart Optimization (2-4 weeks)
- Week 1: Chart value analysis and recommendations
- Week 2: Documentation updates and examples
- Week 3: Testing and validation
- Week 4: Production deployment and monitoring

### Phase 2: Upstream Enhancements (4-8 weeks, Optional)
- Weeks 1-2: Analysis run optimization investigation
- Weeks 3-4: Canary pause logic improvements
- Weeks 5-6: Integration testing
- Weeks 7-8: Upstream contribution preparation

## Dependencies and Prerequisites

### Required for Phase 1
- Access to Helm chart repository
- Production testing environment
- Documentation platform access

### Required for Phase 2 (Optional)
- Argo Rollouts development environment
- Upstream contribution process familiarity
- Comprehensive test suite access

## Conclusion

**Primary Recommendation:** Focus on **chart value optimization** for immediate production value. The existing Argo Rollouts codebase already supports the necessary mechanisms - the issue is inappropriate default values for production environments.

**Secondary Opportunity:** Consider upstream contributions to optimize analysis run termination and canary pause handling, but these are enhancements rather than requirements for the core functionality.

This approach provides **maximum value with minimum risk** while maintaining the option for future performance improvements.

