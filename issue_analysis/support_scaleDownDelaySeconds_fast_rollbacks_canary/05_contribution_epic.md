# Contribution Epic: Support scaleDownDelaySeconds & fast rollbacks with canary strategy

## Epic Overview
This epic focuses on enabling fast rollback capabilities for canary deployments through `scaleDownDelaySeconds` support. The primary goal is to provide feature parity with blue-green deployments for rollback scenarios.

## Possible Exploration Directions

### Chart Value Optimization
- Investigate updating Helm chart defaults for production environments
- Explore environment-specific configuration recommendations
- Consider how existing rollback mechanisms could be better utilized with appropriate timing
- Examine the trade-offs between fast rollbacks and resource efficiency

### Upstream Enhancement Opportunities
- Look into optimizing analysis run termination speed
- Explore improvements to canary pause step skipping during rollbacks
- Consider enhancements to resource cleanup during extended delays
- Investigate better integration between rollback logic and pause mechanisms

### Configuration Strategy
- Think about how to provide sensible defaults for different environments
- Consider validation approaches for rollback window configurations
- Explore documentation approaches for rollback scenarios

## Open Questions for Further Exploration

- What are the appropriate default values for different deployment environments?
- How do rollback windows interact with different types of analysis runs?
- What are the performance implications of different scale down delay settings?
- How can we balance rollback speed with resource efficiency?
- What validation is needed for rollback window configurations?

## Success Criteria
1. Enable fast rollbacks for canary deployments through appropriate configuration
2. Provide clear guidance for production rollback scenarios
3. Maintain compatibility with existing rollback window mechanisms
4. Support environment-specific configuration needs

This epic provides initial directions for exploration while leaving significant room for discovery, configuration optimization, and community input during implementation.

