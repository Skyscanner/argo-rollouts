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

### Success Criteria
1. **Percentage-Based Health**: Support minHealthy parameter for flexible health requirements
2. **Autoscaling Compatibility**: Work with cluster autoscaling and spot instances
3. **PDB Integration**: Respect Pod Disruption Budget constraints
4. **Backward Compatibility**: Preserve existing behavior with sensible defaults
5. **Large-Scale Support**: Successfully deploy services with 100+ pods

## Implementation Strategy

### Phase 1: Analysis and Design (4-6 weeks)

#### Tasks
1. **API Design**: Define minHealthy parameter and integration with existing controls
2. **Health Logic Design**: Design percentage-based health checking algorithms
3. **Capacity Awareness**: Design cluster capacity and autoscaling integration
4. **PDB Integration**: Design Pod Disruption Budget compatibility
5. **Mathematical Modeling**: Model interplay between minHealthy, maxSurge, maxUnavailable
6. **Edge Case Analysis**: Analyze failure scenarios and rollback behavior

#### Deliverables
- **API Specification**: New minHealthy parameter design
- **Algorithm Designs**: Percentage-based health checking logic
- **Mathematical Models**: Capacity and scaling calculations
- **Integration Designs**: PDB and autoscaling compatibility
- **Test Scenarios**: Comprehensive edge case test plans

### Phase 2: Core Implementation (8-10 weeks)

#### Tasks
1. **API Changes**: Add minHealthy to Rollout CRD and types
2. **Health Check Refactor**: Implement percentage-based RolloutHealthy logic
3. **Scaling Logic Updates**: Update canary scaling calculations for partial availability
4. **Capacity Integration**: Add cluster capacity awareness to scaling decisions
5. **PDB Awareness**: Implement PDB constraint checking
6. **Traffic Routing Updates**: Ensure traffic safety with partial pod availability

#### Deliverables
- **CRD Updates**: Extended Rollout API with minHealthy
- **Core Logic**: New percentage-based health checking functions
- **Scaling Algorithms**: Updated replica count calculations
- **Integration Code**: PDB and autoscaling compatibility layers

### Phase 3: Testing and Validation (6-8 weeks)

#### Tasks
1. **Unit Testing**: Comprehensive unit tests for percentage logic
2. **Integration Testing**: End-to-end rollout testing with various configurations
3. **Large-Scale Testing**: Test with 100+ pod deployments
4. **Autoscaling Testing**: Test with Karpenter, Cluster Autoscaler scenarios
5. **PDB Testing**: Test with Pod Disruption Budget constraints
6. **Spot Instance Testing**: Test preemption and capacity loss scenarios
7. **Performance Testing**: Validate performance impact of new logic
8. **Chaos Testing**: Test failure scenarios and recovery

#### Deliverables
- **Test Suites**: Comprehensive automated test coverage
- **Performance Benchmarks**: Performance impact analysis
- **Large-Scale Validation**: Real-world large deployment testing
- **Chaos Test Reports**: Failure scenario analysis and fixes

### Phase 4: Documentation and Release (4-6 weeks)

#### Tasks
1. **User Documentation**: Document minHealthy parameter and usage
2. **Migration Guide**: Guide for upgrading existing rollouts
3. **Best Practices**: Document large-scale deployment patterns
4. **Troubleshooting Guide**: Help for common issues and edge cases
5. **API Documentation**: Update CRD and API references
6. **Release Notes**: Document changes and migration path
7. **Community Communication**: Announce changes and gather feedback

#### Deliverables
- **Documentation Updates**: Complete user and API documentation
- **Migration Guides**: Clear upgrade paths for existing users
- **Best Practice Guides**: Large-scale deployment recommendations
- **Community Materials**: Blog posts, videos, and presentations

## Technical Implementation Details

### API Design

#### MinHealthy Parameter
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      minHealthy: 80%          # New parameter: percentage or absolute count
      maxSurge: 25%            # Existing: controls rollout pacing
      maxUnavailable: 25%      # Existing: controls unavailability
      dynamicStableScale: true # Existing: resource optimization
```

#### Parameter Options
- **Percentage**: "80%" - percentage of desired replicas
- **Absolute**: "50" - absolute number of healthy pods
- **Default**: "100%" - backward compatibility

### Health Check Logic

#### Percentage-Based Health Calculation
```go
func calculateRolloutHealth(rollout *v1alpha1.Rollout, rsList []*appsv1.ReplicaSet) (bool, error) {
    minHealthy := parseMinHealthy(rollout.Spec.Strategy.Canary.MinHealthy)

    for _, rs := range rsList {
        healthyPercent := calculateHealthyPercentage(rs)
        if healthyPercent < minHealthy {
            return false, nil
        }
    }

    // Additional checks for rollout completion
    return checkRolloutCompletion(rollout, rsList)
}
```

#### Healthy Percentage Calculation
```go
func calculateHealthyPercentage(rs *appsv1.ReplicaSet) float64 {
    if rs.Status.Replicas == 0 {
        return 100.0 // Empty RS is considered healthy
    }

    readyPods := rs.Status.ReadyReplicas
    totalPods := rs.Status.Replicas

    return (float64(readyPods) / float64(totalPods)) * 100.0
}
```

### Scaling Logic Updates

#### Capacity-Aware Scaling
```go
func calculateReplicaCountsWithCapacity(rollout *v1alpha1.Rollout, clusterCapacity int32) (canaryCount, stableCount int32) {
    totalDesired := rollout.Spec.Replicas
    canaryWeight := getCanaryWeight(rollout)

    // Calculate base replica counts
    canaryDesired := int32(float64(totalDesired) * canaryWeight)

    // Adjust for cluster capacity and minHealthy
    minHealthyPercent := parseMinHealthy(rollout.Spec.Strategy.Canary.MinHealthy)
    maxCanaryAllowed := int32(float64(clusterCapacity) * minHealthyPercent / 100.0)

    canaryCount = min(canaryDesired, maxCanaryAllowed)

    if rollout.Spec.Strategy.Canary.DynamicStableScale {
        stableCount = totalDesired - canaryCount
    } else {
        stableCount = totalDesired
    }

    return canaryCount, stableCount
}
```

### PDB Integration

#### PDB-Aware Decisions
```go
func checkPDBConstraints(rollout *v1alpha1.Rollout, rs *appsv1.ReplicaSet, desiredReplicas int32) error {
    pdbList, err := getPDBsForRollout(rollout)
    if err != nil {
        return err
    }

    for _, pdb := range pdbList {
        allowedDisruptions := calculateAllowedDisruptions(pdb, rs.Status.Replicas)
        if desiredReplicas < rs.Status.Replicas - allowedDisruptions {
            return fmt.Errorf("PDB %s would be violated", pdb.Name)
        }
    }

    return nil
}
```

### Interplay with MaxSurge/MaxUnavailable

#### Coordinated Logic
```go
func validateScalingParameters(rollout *v1alpha1.Rollout) error {
    minHealthy := parseMinHealthy(rollout.Spec.Strategy.Canary.MinHealthy)
    maxUnavailable := rollout.Spec.Strategy.Canary.MaxUnavailable
    maxSurge := rollout.Spec.Strategy.Canary.MaxSurge

    // Ensure parameters work together
    totalRange := minHealthy + maxUnavailable.Percent
    if totalRange > 100 {
        return fmt.Errorf("minHealthy (%d%%) + maxUnavailable (%d%%) cannot exceed 100%%",
            minHealthy, maxUnavailable.Percent)
    }

    // Ensure surge capacity is available
    if maxSurge.Percent + 100 - minHealthy < 100 {
        return fmt.Errorf("insufficient surge capacity for minHealthy requirement")
    }

    return nil
}
```

## Risk Assessment and Mitigation

### Technical Risks
- **Logic Complexity**: Percentage calculations introduce edge cases
  - *Mitigation*: Comprehensive mathematical modeling and testing
- **Performance Impact**: Additional calculations in health checks
  - *Mitigation*: Performance benchmarking and optimization
- **Race Conditions**: Concurrent scaling and health checking
  - *Mitigation*: Proper locking and state management

### Operational Risks
- **Deployment Failures**: New logic might cause unexpected failures
  - *Mitigation*: Feature flags and gradual rollout
- **Resource Contention**: Health checks competing with scaling operations
  - *Mitigation*: Rate limiting and prioritization
- **PDB Violations**: Incorrect PDB calculations
  - *Mitigation*: Conservative PDB checking and validation

### Compatibility Risks
- **Existing Rollouts**: Breaking changes to current behavior
  - *Mitigation*: Backward compatible defaults (100% minHealthy)
- **Service Mesh**: Traffic routing issues with partial availability
  - *Mitigation*: Extensive integration testing with Istio/ALB
- **Monitoring Tools**: Health check changes affecting monitoring
  - *Mitigation*: Clear communication and migration guides

## Success Metrics

### Quantitative Metrics
- **Deployment Success Rate**: >98% success for large-scale rollouts
- **Resource Utilization**: 30-50% reduction in required cluster capacity
- **PDB Compliance**: 100% PDB constraint satisfaction
- **Autoscaling Events**: 50% reduction in unnecessary scaling events
- **Test Coverage**: >95% code coverage for new logic

### Qualitative Metrics
- **User Feedback**: Positive community response to large-scale support
- **Adoption Rate**: Percentage of users adopting minHealthy parameter
- **Issue Reduction**: Decrease in scaling and health check related issues
- **Documentation Quality**: User satisfaction with new documentation

## Timeline and Milestones

### Phase 1: Analysis and Design (4-6 weeks)
- **Week 1-2**: API and algorithm design
- **Week 3-4**: Mathematical modeling and edge case analysis
- **Week 5-6**: Integration design and test planning

### Phase 2: Core Implementation (8-10 weeks)
- **Week 7-10**: API changes and core logic
- **Week 11-14**: Scaling logic and capacity integration
- **Week 15-16**: PDB integration and traffic routing updates

### Phase 3: Testing and Validation (6-8 weeks)
- **Week 17-20**: Unit and integration testing
- **Week 21-24**: Large-scale and autoscaling testing

### Phase 4: Documentation and Release (4-6 weeks)
- **Week 25-28**: Documentation and migration guides
- **Week 29-30**: Release preparation and community communication

### Total Timeline: 22-30 weeks (5-7 months)

## Dependencies and Prerequisites

### Technical Dependencies
- **Kubernetes Version**: Support for modern PDB and deployment features
- **Go Version**: Compatible with current build requirements
- **Testing Infrastructure**: Large-scale testing environment
- **Service Mesh**: Istio/ALB testing environments

### Team Dependencies
- **Cross-Functional Team**: Backend, testing, and documentation engineers
- **Domain Experts**: Kubernetes, autoscaling, and service mesh expertise
- **Product Management**: Requirements and prioritization
- **DevOps**: Large-scale testing environment setup

### External Dependencies
- **Community Review**: Open source community feedback
- **CNCF Alignment**: Alignment with Cloud Native Computing Foundation
- **Enterprise Users**: Beta testing with large organizations

## Communication Plan

### Internal Communication
- **Weekly Standups**: Progress updates and blocker resolution
- **Bi-Weekly Demos**: Showcase progress and gather feedback
- **Monthly Reviews**: Overall project health and adjustments

### External Communication
- **GitHub Issues**: Regular updates on implementation progress
- **Community Forums**: Discussion of design decisions and trade-offs
- **Blog Posts**: Technical deep dives into new capabilities
- **Webinars**: Live sessions on large-scale rollout patterns

## Contingency Plans

### Schedule Risks
- **Scope Creep**: Additional requirements discovered during implementation
  - *Contingency*: Strict scope control, separate follow-up projects
- **Technical Challenges**: Unexpected complexity in percentage logic
  - *Contingency*: Prototype spikes to validate approaches
- **Testing Delays**: Difficulty setting up large-scale test environments
  - *Contingency*: Start with smaller scale testing, scale up gradually

### Technical Risks
- **Fundamental Issues**: Percentage logic proves incompatible with traffic routing
  - *Contingency*: Fallback to threshold-based approach
- **Performance Problems**: Health checks become too slow for large deployments
  - *Contingency*: Optimize algorithms, add caching
- **Integration Conflicts**: Unexpected issues with service mesh integration
  - *Contingency*: Partner with service mesh communities for solutions

### Resource Risks
- **Team Availability**: Key team members become unavailable
  - *Contingency*: Cross-training and knowledge sharing
- **Infrastructure Issues**: Testing environments become unavailable
  - *Contingency*: Cloud-based testing environments, local alternatives

## Conclusion

This contribution epic outlines a comprehensive approach to solving the fundamental scalability limitations in Argo Rollouts. By implementing percentage-based health checks with `minHealthy` parameters and proper integration with Kubernetes deployment controls, Argo Rollouts can finally support large-scale deployments with autoscaling and spot instances.

The phased approach minimizes risk while delivering significant value to organizations running large-scale progressive delivery. The implementation will position Argo Rollouts as a competitive progressive delivery tool capable of handling modern cloud-native deployment challenges.

**Key Success Factors**:
- Thorough analysis and mathematical modeling
- Comprehensive testing across scale and failure scenarios
- Strong backward compatibility and migration support
- Close collaboration with the community and domain experts