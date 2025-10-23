# Contribution Epic: Simultaneous 100% Availability Requirement

## Problem Statement

Argo Rollouts requires 100% availability across ALL ReplicaSets simultaneously, causing deadlocks in autoscaling environments.

**Code Evidence**: `RolloutHealthy` function demands `AvailableReplicas == totalReplicas`.

## Exploration Directions

### Health Check Logic
- Investigate percentage-based requirements instead of binary logic
- Explore "sufficiently healthy" definitions for different scenarios

### Scaling and Capacity Awareness
- Consider cluster resource accounting in scaling decisions

### Traffic Routing Safety
- Examine traffic routing with partial pod availability
- Explore safety mechanisms for health check and traffic distribution interaction

### API Design Considerations
- Consider new parameters for health and scaling control
- Evaluate backward compatibility approaches

### Testing Approaches
- Validate behavior across scale scenarios
- Test with various autoscaling configurations

## Open Questions

- How should "healthy" be defined for different application types?
- What are safety implications of relaxed health requirements?
- How do service meshes handle partial availability?
- What are performance implications of sophisticated health checking?
- How to ensure smooth migration for existing users?

## Manual Exploration Required

**Investigate Further**:
- Review existing health check implementations in similar tools
- Examine Kubernetes Deployment percentage-based logic
- Analyze service mesh health handling approaches
- Prototype percentage-based health check implementations

**Key Question**: What would a minimum viable percentage-based health check look like?