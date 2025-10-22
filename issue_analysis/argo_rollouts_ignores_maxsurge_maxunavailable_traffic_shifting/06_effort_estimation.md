# Effort Estimation: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Overview
**Issue Type:** Design Philosophy Challenge (not a bug fix)  
**Complexity:** High (requires maintainer consensus and potential design changes)  
**Estimated Total Effort:** 4-8 weeks (highly variable based on community discussion)

## Effort Breakdown

### Phase 1: Research & Community Engagement (2-3 weeks)
**Effort:** 2-3 weeks  
**Rationale:** Understanding maintainer rationale and building community consensus is the most time-consuming part. This involves:
- Deep analysis of design decisions from issue #2239
- Surveying multiple related issues (#3284, #3539, #3397)
- Evaluating MinPodsPerReplicaSet effectiveness as current workaround
- Analyzing dynamicStableScale impact on maxSurge/maxUnavailable feasibility
- Assessing manual canary steps as alternative scaling control

**Key Activities:**
- Code archaeology to understand traffic routing design decisions
- Community engagement and maintainer discussions
- Analysis of MinPodsPerReplicaSet limitations
- Assessment of whether maxSurge/maxUnavailable can be supported when dynamicStableScale is enabled
- Documentation of manual workaround pain points

### Phase 2: Design & Technical Approach (1-2 weeks, conditional)
**Effort:** 1-2 weeks  
**Rationale:** Only applicable if maintainers approve design change. Requires evaluating multiple approaches (absolute vs relative interpretation).

**Key Activities:**
- Design maxSurge integration that doesn't break traffic routing logic
- Evaluate relative percentage interpretation between canary steps
- Compare absolute vs relative approaches for user needs
- Risk assessment and mitigation planning

### Phase 3: Implementation (2-3 weeks, conditional)
**Effort:** 2-3 weeks  
**Rationale:** Straightforward implementation once design is approved, but requires careful testing to avoid breaking traffic control logic.

**Key Activities:**
- Modify `CalculateReplicaCountsForTrafficRoutedCanary()` function
- Update rollback logic if needed
- Add validation warning for unsupported maxSurge/maxUnavailable with traffic routing
- Comprehensive testing

### Phase 4: Testing & Documentation (1-2 weeks, conditional)
**Effort:** 1-2 weeks  
**Rationale:** Standard testing and documentation effort, but conditional on implementation approval.

**Key Activities:**
- Add test coverage for maxSurge scenarios with traffic routing
- Update documentation
- Performance validation

### Phase 5: Alternative Solutions (1-2 weeks, if implementation rejected)
**Effort:** 1-2 weeks  
**Rationale:** If maintainers reject the design change, focus shifts to documentation and workarounds. This is now more likely given MinPodsPerReplicaSet provides some relief.

**Key Activities:**
- Improve `minPodsPerReplicaSet` documentation and best practices
- Create infrastructure scaling best practices for traffic routing
- Document manual canary step approaches with pros/cons

## Risk Factors Affecting Effort

### High Probability Risks
- **Maintainers reject design change:** If maintainers reject the design change, effort shifts to documentation (reduces total effort by 50-70%)
- **MinPodsPerReplicaSet deemed sufficient:** Community may accept current workarounds, reducing need for changes
- **Community divided on approach:** Extended discussions could add 2-4 weeks

### Medium Probability Risks
- **Implementation Challenges:** Breaking traffic control logic during implementation
- **Performance Impact:** Traffic shifting performance degradation
- **Backward Compatibility:** Ensuring existing behavior isn't broken

## Resource Requirements

### Skills Needed
- **Primary:** Go programming, Kubernetes concepts, Argo Rollouts architecture
- **Secondary:** Community engagement, technical writing, RFC processes
- **Tertiary:** Traffic routing systems (Istio, ALB, SMI)

### Environment Requirements
- Kubernetes cluster for testing
- Access to Argo Rollouts development environment
- Community forum access (GitHub, Slack)

## Success Probability Assessment

### Best Case Scenario (20% probability)
- Maintainers open to design discussion
- Community consensus reached quickly
- Implementation approved and completed in 4-6 weeks

### Most Likely Scenario (60% probability)
- Extended community discussions (2-4 weeks)
- Design change rejected, focus on documentation and MinPodsPerReplicaSet improvements
- Total effort: 3-5 weeks

### Worst Case Scenario (20% probability)
- Maintainers strongly oppose design change
- Community divided, no consensus reached
- Effort limited to documentation improvements (1-2 weeks)

## Recommendation
**Proceed with Caution:** This issue challenges fundamental Argo Rollouts design decisions. Consider starting with community engagement before investing significant development effort. The most valuable contribution might be:

1. **Immediate Impact:** Improve MinPodsPerReplicaSet documentation and best practices
2. **Medium-term:** Add validation warnings when maxSurge/maxUnavailable are set with traffic routing
3. **Long-term:** Pursue design change only if community consensus is achievable

**Key Insight:** MinPodsPerReplicaSet provides some scaling control but doesn't address the core issue of rapid scaling between traffic weights. Manual canary steps are a poor workaround that highlights the real need for proper scaling controls.