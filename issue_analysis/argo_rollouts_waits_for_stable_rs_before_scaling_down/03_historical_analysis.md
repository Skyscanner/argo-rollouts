# Historical Analysis: Argo-rollouts waits for stable RS to be stable before scaling it down

## Issue Timeline

### Early Argo Rollouts (Pre-1.0)
**Context**: Argo Rollouts started as a progressive delivery tool focused on traffic-based canary deployments.

**Design Decisions**:
- **Conservative Health Checks**: Prioritized safety over efficiency
- **Full Availability Requirements**: Assumed deployments could achieve 100% pod availability
- **Simple Scaling Logic**: Basic ReplicaSet management without autoscaling awareness

**Rationale**: Early focus on correctness and safety rather than large-scale performance.

### Introduction of Traffic Routing (v0.8+)
**Changes**: Added support for service mesh integration (Istio, ALB, etc.)

**New Logic**:
```go
// Early traffic routing logic
func shouldWaitForFullAvailability() bool {
    return true  // Always wait for 100% availability
}
```

**Problem Introduced**: Traffic routing made scaling more complex, but health checks remained binary.

### DynamicStableScale Feature (v1.0+)
**Purpose**: Address resource efficiency in traffic-routed canaries by scaling down stable RS during rollout.

**Implementation**:
```go
// rollout/canary.go
if dynamicStableScale {
    scaleDownStableRS()
} else {
    keepStableRSFullScale()
}
```

**Limitation**: Only addressed resource efficiency for traffic-routed canaries, didn't solve the fundamental health checking issue.

### Growing Scale Challenges (v1.2+)
**User Reports**: Issues with large-scale deployments and autoscaling conflicts became more frequent.

**Community Feedback**:
- "Rollouts get stuck waiting for pods that can't be scheduled"
- "Autoscaling doesn't work with Argo Rollouts"
- "PDBs cause deployment deadlocks"

**Response**: Incremental fixes but no fundamental health check redesign.

## Code Evolution

### Health Check Logic Evolution

#### v0.x: Basic Health Checks
```go
func RolloutHealthy(rollout *v1alpha1.Rollout) bool {
    return allPodsReady && allStepsExecuted
}
```

#### v1.0+: Scale-Down Awareness
```go
func RolloutHealthy(rollout *v1alpha1.Rollout) bool {
    return allPodsReady && allStepsExecuted && oldRSFullyScaledDown
}
```

#### Current: Complex but Still Binary
```go
func RolloutHealthy(rollout *v1alpha1.Rollout) bool {
    completedStrategy := executedAllSteps && currentRSIsStable && scaleDownOldReplicas
    return completedStrategy && allConditionsMet
}
```

**Missing Evolution**: No progression toward percentage-based health checks.

### Scaling Logic Development

#### Basic Canary Scaling
```go
// Early: Simple replica distribution
canaryReplicas = totalReplicas * canaryWeight
stableReplicas = totalReplicas - canaryReplicas
```

#### Traffic-Routed Scaling
```go
// Added traffic awareness
canaryReplicas = calculateTrafficBasedReplicas()
stableReplicas = totalReplicas  // Keep full scale for traffic
```

#### DynamicStableScale Addition
```go
// Resource optimization
if dynamicStableScale {
    stableReplicas = totalReplicas - canaryReplicas  // Scale down stable
} else {
    stableReplicas = totalReplicas  // Keep full scale
}
```

**Gap**: Scaling logic evolved but health checks didn't adapt to partial availability scenarios.

## Architectural Decisions

### Conservative Design Philosophy
**Original Mindset**: "Better to be safe than sorry"
- **Safety First**: Prioritized traffic safety and data consistency
- **Simple Logic**: Binary health checks were easier to reason about
- **Kubernetes Assumptions**: Assumed clusters could provide full availability

### Trade-offs Made
- **Safety vs Efficiency**: Chose safety over resource efficiency
- **Simplicity vs Flexibility**: Simple binary logic over complex percentage calculations
- **Static vs Dynamic**: Static health requirements over dynamic capacity awareness

### Missed Opportunities
1. **Percentage-Based Health**: Never implemented minAvailable-style logic
2. **Autoscaling Integration**: No awareness of cluster capacity limitations
3. **PDB Compatibility**: Didn't account for Pod Disruption Budget constraints
4. **Spot Instance Handling**: No graceful handling of pod preemption

## Community Evolution

### User Scaling Patterns
- **Small to Medium**: Early users had smaller deployments where 100% availability was feasible
- **Large-Scale Adoption**: Later users hit scaling limits with existing logic
- **Autoscaling Adoption**: Modern users expect compatibility with dynamic infrastructure

### Issue Reports Timeline
- **2020-2021**: Basic scaling and traffic routing issues
- **2022**: Autoscaling conflicts become more prominent
- **2023**: PDB-related deadlocks reported
- **2024**: Large-scale deployment failures with spot instances

### Feature Requests
- **minAvailable Parameter**: Requests for percentage-based health checks
- **Autoscaling Support**: Better integration with cluster autoscaling
- **PDB Awareness**: Respect for Pod Disruption Budgets
- **Spot Instance Compatibility**: Handling of preemptible workloads

## Technical Debt Accumulation

### Design Debt
- **Binary Health Logic**: All-or-nothing approach doesn't fit modern deployments
- **Capacity Assumptions**: Assumes unlimited cluster capacity
- **Static Requirements**: No adaptation to deployment context

### Implementation Debt
- **Hardcoded Thresholds**: 100% availability requirement is baked into logic
- **Limited State Tracking**: No awareness of cluster capacity or PDBs
- **Testing Gaps**: Few tests for large-scale or autoscaling scenarios

### Integration Debt
- **Kubernetes Features**: Not leveraging modern K8s deployment controls
- **Autoscaling Tools**: Poor integration with Karpenter, CA, etc.
- **Service Mesh**: Traffic routing works but health checks conflict

## Lessons Learned

### Design Patterns
- **Health Check Evolution**: From binary to percentage-based approaches
- **Capacity Awareness**: Modern systems must be capacity-aware
- **Progressive Enhancement**: Start conservative, add flexibility over time

### Technical Lessons
- **Assumption Validation**: Don't assume unlimited cluster resources
- **PDB Integration**: Pod Disruption Budgets are deployment constraints
- **Autoscaling Reality**: Clusters have capacity limits and scaling delays

### Community Lessons
- **User Scaling**: User deployments grow larger over time
- **Infrastructure Evolution**: Autoscaling and spot instances become standard
- **Safety vs Practicality**: Safety requirements must balance with real-world constraints

## Future Implications

### Required Architecture Changes
1. **Percentage-Based Health**: Implement minHealthy-style logic
2. **Capacity Awareness**: Track cluster capacity and autoscaling state
3. **PDB Integration**: Respect Pod Disruption Budget constraints
4. **Dynamic Logic**: Adapt health requirements based on deployment context

### Breaking Changes Needed
- **API Changes**: New minHealthy parameter
- **Logic Overhaul**: Fundamental health check redesign
- **Default Behavior**: Change from 100% to percentage-based requirements

### Backward Compatibility Challenges
- **Existing Deployments**: Must not break current working rollouts
- **Migration Path**: Clear upgrade path for existing users
- **Feature Flagging**: Gradual rollout of new behavior

## Historical Context Summary

This issue represents a classic case of architectural assumptions not keeping pace with infrastructure evolution. Argo Rollouts was designed with conservative, safety-first health checks that worked for early adopters with smaller deployments. However, as Kubernetes deployments grew larger and adopted autoscaling, spot instances, and Pod Disruption Budgets, the binary health requirements became a fundamental blocker.

The evolution shows incremental improvements (like dynamicStableScale) but no fundamental rethinking of health checking logic. The solution requires moving from binary "all healthy" requirements to percentage-based "sufficiently healthy" logic that works with modern autoscaling infrastructure.

**Key Historical Insight**: The issue isn't a bug but a design limitation that became apparent as Kubernetes infrastructure matured toward autoscaling and large-scale deployments.