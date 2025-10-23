# Exploration Analysis: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Overview
**Issue Type:** Design Philosophy Challenge (not a bug fix)  
**Complexity:** High (requires maintainer consensus and potential design changes)

## Technical Considerations

### Current Design Philosophy
Traffic routing prioritizes traffic control over pod scaling limits. maxSurge/maxUnavailable are intentionally not supported to maintain traffic shifting priority.

### Key Technical Insights
- **Critical Finding:** Implementation feasibility depends on `dynamicStableScale` setting
- **Without dynamicStableScale:** maxSurge/maxUnavailable remain non-applicable (stable always fully scaled)
- **With dynamicStableScale:** Both limits become technically feasible and meaningful

### Implementation Considerations
- Maintain traffic shifting priority over scaling limits
- Ensure rollback minimum availability logic still functions
- Consider interaction with `minPodsPerReplicaSet`
- Preserve dynamic stable scaling behavior
- Evaluate relative percentage interpretation between canary steps

## Community and Design Factors

### User Pain Points
- "Very rapid infrastructure scaling" causing cost and resource problems
- Need for scaling control in traffic-routed deployments
- Manual canary steps as poor workaround for scaling control

### Current Workarounds
- **MinPodsPerReplicaSet:** Provides minimum pod floor but doesn't control surge
- **Manual Canary Steps:** Verbose and hard to maintain
- **Documentation:** Limited guidance on scaling behavior

### Community Reception Assessment
- **High Controversy:** Challenges fundamental traffic routing design decisions
- **Strong User Demand:** Multiple issues (#3284, #3539, #3397) requesting functionality
- **Maintainer Position:** Intentional design decision prioritizing traffic control

## Exploration Directions

### Design Philosophy Evaluation
- Investigate whether current traffic routing design should be revisited
- Assess community consensus on maxSurge/maxUnavailable support
- Analyze dynamicStableScale impact on implementation feasibility

### Technical Approaches
- Examine relative percentage interpretation between canary steps
- Compare absolute vs relative approaches for scaling limits
- Evaluate integration with existing traffic control logic

### Alternative Solutions
- Enhance MinPodsPerReplicaSet documentation and best practices
- Develop infrastructure scaling best practices for traffic routing
- Improve guidance for manual canary step approaches

## Open Questions
- Should the design philosophy be revisited to support maxSurge/maxUnavailable?
- How does dynamicStableScale impact implementation feasibility?
- What are the trade-offs between absolute and relative scaling interpretations?
- Can MinPodsPerReplicaSet be enhanced to better address scaling concerns?
- What validation and warnings should be provided for unsupported configurations?

## Key Insight
MinPodsPerReplicaSet provides some scaling control but doesn't address the core issue of rapid scaling between traffic weights. Manual canary steps highlight the real need for proper scaling controls in traffic-routed deployments.