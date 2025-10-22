# Contribution Epic: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Epic Overview
**Objective:** Address community demand for maxSurge/maxUnavailable support in traffic-routed canary deployments, despite this being an intentional design decision that prioritizes traffic control over scaling limits.

**Success Criteria:**
- [ ] Community consensus reached on whether to implement maxSurge/maxUnavailable for traffic routing
- [ ] If approved: Traffic-routed canary deployments respect maxSurge and maxUnavailable settings
- [ ] If approved: Scaling behavior consistent between basic and traffic-routed canary strategies
- [ ] If not approved: Clear documentation of design decision and workarounds provided
- [ ] User pain points around "rapid infrastructure scaling" addressed

## Prerequisites
- [ ] Understanding of Argo Rollouts traffic routing design philosophy
- [ ] Familiarity with maxSurge/maxUnavailable Kubernetes concepts
- [ ] Access to test cluster for validation
- [ ] Community discussion and maintainer alignment

## Current Workarounds Analysis

### MinPodsPerReplicaSet as Partial Solution
**How it helps:** `MinPodsPerReplicaSet` provides a minimum floor for pod counts in traffic-routed canaries, preventing complete scale-down during low-traffic phases.

**Code Implementation:**
```go
func CheckMinPodsPerReplicaSet(rollout *v1alpha1.Rollout, count int32) int32 {
    if count == 0 {
        return count
    }
    if rollout.Spec.Strategy.Canary == nil || rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet == nil || rollout.Spec.Strategy.Canary.TrafficRouting == nil {
        return count
    }
    return max(count, *rollout.Spec.Strategy.Canary.MinPodsPerReplicaSet)
}
```

**Limitations:**
- Only sets minimum pods, doesn't control maximum surge
- Doesn't prevent rapid scaling between traffic weights
- Requires manual tuning per rollout

### Manual Canary Steps as Current Alternative
**User Concern:** Without maxSurge/maxUnavailable controls, users must manually add multiple small canary steps to control scaling speed.

**Example Manual Approach:**
```yaml
steps:
- setWeight: 5   # Small increment
- pause: {duration: 60s}
- setWeight: 10  # Another small increment  
- pause: {duration: 60s}
# ... many more steps instead of smooth progression
```

**Problems with Manual Approach:**
- Verbose rollout definitions
- Hard to maintain and tune
- Doesn't adapt to different traffic patterns
- Manual process prone to errors

## Epic Tasks

### Phase 1: Research & Community Engagement
- [ ] **Task 1.1:** Analyze maintainer design rationale from issue #2239
  - **Acceptance Criteria:** Document why maxSurge/maxUnavailable were intentionally excluded
  - **Dependencies:** None
  
- [ ] **Task 1.2:** Survey existing issues (#3284, #3539, #3397) for user impact
  - **Acceptance Criteria:** Quantify infrastructure scaling problems reported by users
  - **Dependencies:** None

- [ ] **Task 1.3:** Evaluate MinPodsPerReplicaSet effectiveness
  - **Acceptance Criteria:** Assess if MinPodsPerReplicaSet adequately addresses scaling concerns
  - **Dependencies:** None

- [ ] **Task 1.4:** Engage maintainers on design philosophy change
  - **Acceptance Criteria:** Open discussion on whether design should be revisited
  - **Dependencies:** Tasks 1.1, 1.2, 1.3

### Phase 2: Design & Technical Approach (Conditional)
- [ ] **Task 2.1:** Design maxSurge integration with traffic routing
  - **Acceptance Criteria:** Define approach that doesn't break traffic control priorities
  - **Dependencies:** Task 1.4 approval
  
- [ ] **Task 2.2:** Evaluate relative percentage interpretation
  - **Acceptance Criteria:** Analyze making maxSurge/maxUnavailable relative to traffic weight changes between steps
  - **Dependencies:** Task 2.1
  
- [ ] **Task 2.3:** Compare absolute vs relative approaches
  - **Acceptance Criteria:** Determine if relative interpretation better addresses user needs than absolute limits
  - **Dependencies:** Task 2.2

- [ ] **Task 2.4:** Prototype implementation approach
  - **Acceptance Criteria:** Working prototype showing chosen scaling limits with traffic routing
  - **Dependencies:** Task 2.3

### Phase 3: Implementation (Conditional)
- [ ] **Task 3.1:** Modify `CalculateReplicaCountsForTrafficRoutedCanary()`
  - **Acceptance Criteria:** Function respects maxSurge limits while maintaining traffic control
  - **Dependencies:** Task 2.4
  
- [ ] **Task 3.2:** Update rollback logic if needed
  - **Acceptance Criteria:** Ensure rollback minimum availability logic still works
  - **Dependencies:** Task 3.1

- [ ] **Task 3.3:** Add validation warning
  - **Acceptance Criteria:** Clear validation message when maxSurge/maxUnavailable set with traffic routing
  - **Dependencies:** Task 3.1

### Phase 4: Testing & Documentation (Conditional)
- [ ] **Task 4.1:** Add comprehensive test coverage
  - **Acceptance Criteria:** Tests cover maxSurge scenarios with traffic routing
  - **Dependencies:** Task 3.1
  
- [ ] **Task 4.2:** Document design decision and workarounds
  - **Acceptance Criteria:** Clear documentation of current behavior and alternatives
  - **Dependencies:** All previous tasks

### Phase 5: Alternative Solutions (If Implementation Rejected)
- [ ] **Task 5.1:** Improve `minPodsPerReplicaSet` documentation
  - **Acceptance Criteria:** Better guidance on using traffic routing's scaling controls
  - **Dependencies:** Task 1.4 rejection
  
- [ ] **Task 5.2:** Create infrastructure scaling best practices
  - **Acceptance Criteria:** Recommendations for avoiding rapid scaling with traffic routing
  - **Dependencies:** Task 5.1

## Risk Assessment
| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|-------------------|
| Maintainers reject design change | High | High | Prepare alternative solutions and documentation |
| Implementation breaks traffic control logic | Medium | High | Extensive testing, phased rollout |
| Community divided on approach | Medium | Medium | Clear RFC process, community voting |
| Performance impact on traffic shifting | Low | Medium | Benchmark tests, performance validation |
| MinPodsPerReplicaSet deemed sufficient | Medium | Medium | Document effectiveness and limitations |

## Technical Approach
**Current Design Philosophy:** Traffic routing prioritizes traffic control over pod scaling limits. maxSurge/maxUnavailable are intentionally not supported.

**Potential Implementation:** If approved, modify `CalculateReplicaCountsForTrafficRoutedCanary()` to apply maxSurge limits while preserving traffic control logic.

**Key Considerations:**
1. Maintain traffic shifting priority over scaling limits
2. Ensure rollback minimum availability logic still functions
3. Consider interaction with `minPodsPerReplicaSet`
4. Preserve dynamic stable scaling behavior
5. Evaluate relative percentage interpretation between canary steps (see `07_relative_interpretation_analysis.md`)

## Contribution Strategy
**Upstream Reception:** High controversy - challenges fundamental traffic routing design decisions.

**Communication Approach:**
- Respect existing design decisions while presenting user pain points
- Reference multiple community issues (#3284, #3539, #3397)
- Highlight manual canary step workaround limitations
- Propose incremental approach that doesn't break existing behavior
- Be prepared for rejection and have alternative solutions ready

**Timeline:** 4-8 weeks for discussion and consensus, plus 2-3 weeks implementation if approved