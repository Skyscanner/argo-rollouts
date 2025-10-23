# Contribution Epic: Argo-rollouts waits for stable RS to be stable before scaling it down

## Epic Overview

### Problem Statement
Argo Rollouts requires 100% availability of both canary AND stable ReplicaSets, creating impossible demands for large-scale deployments with Pod Disruption Budgets, autoscaling clusters, and spot instances. This causes deployment deadlocks and prevents effective progressive delivery at scale.

### Business Value
- **Scalability**: Enable progressive delivery for services with 100+ pods
- **Cost Optimization**: Support autoscaling and spot instance deployments
- **Reliability**: Reduce deployment failures and manual interventions
- **Automation**: Restore CI/CD pipeline reliability for large deployments
- **Competitive Advantage**: Match capabilities of Flagger, Keptn, and modern deployment tools

## Possible Exploration Directions

### Health Check Logic
- Investigate percentage-based health requirements instead of binary all-or-nothing logic
- Explore how to define "sufficiently healthy" vs "fully healthy" for different deployment scenarios
- Consider the trade-offs between safety and flexibility in health checking

### Scaling and Capacity Awareness
- Consider how scaling decisions could account for available cluster resources

### Traffic Routing Safety
- Examine how traffic routing decisions could work with partial pod availability
- Explore safety mechanisms to ensure traffic is only routed to sufficiently healthy pods
- Consider the interaction between health checks and traffic distribution logic

### API Design Considerations
- Think about how new parameters might be introduced to control health and scaling behavior
- Consider backward compatibility and migration paths for existing users
- Explore how to make these controls intuitive for different deployment patterns

### Testing Approaches
- Consider how to validate behavior across different scale scenarios
- Think about testing with various autoscaling configurations
- Explore chaos engineering approaches for failure scenario validation

## Open Questions for Further Exploration

- How should "healthy" be defined for different types of applications?
- What are the safety implications of relaxing health requirements?
- How do different service mesh implementations handle partial pod availability?
- What are the performance implications of more sophisticated health checking?
- How can we ensure smooth migration for existing Argo Rollouts users?
- What additional integration points exist with cloud provider services?