# Contribution Epic: maxSurge/maxUnavailable with Traffic Shifting

## Epic Overview

**Problem**: maxSurge/maxUnavailable ignored in traffic-routed canaries due to intentional design prioritizing traffic control over scaling limits.

**Challenge**: Requires either changing maintainer design philosophy or accepting fundamental limitation.

## Exploration Directions

### Design Philosophy Evaluation
- Assess whether traffic routing should support both traffic control AND scaling limits
- Evaluate community consensus from multiple open issues
- Analyze impact of dynamicStableScale on implementation feasibility

### Technical Approaches
- Compare absolute vs relative interpretations of scaling limits
- Examine integration with existing traffic weight logic
- Evaluate validation warnings for unsupported configurations

### Workaround Enhancement
- Improve MinPodsPerReplicaSet documentation
- Provide infrastructure scaling best practices
- Enhance guidance for manual canary step approaches

## Open Questions

- Should design philosophy be revisited for scaling limits?
- How does dynamicStableScale affect implementation?
- What are trade-offs between absolute and relative approaches?
- Can MinPodsPerReplicaSet better address scaling concerns?

## Success Criteria

1. Clear understanding of design philosophy conflict
2. Improved documentation for current workarounds
3. Community consensus on implementation direction
4. Enhanced user experience for scaling control

## Manual Exploration Required

**Investigate Further**:
- Engage maintainers in issue #2239 discussion
- Survey community sentiment across multiple issues
- Prototype scaling limit implementations
- Test impact on traffic routing behavior

**Key Question**: Is there a technical solution that respects both traffic control and scaling limits?