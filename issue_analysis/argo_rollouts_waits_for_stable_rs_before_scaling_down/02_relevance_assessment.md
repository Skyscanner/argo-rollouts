# Relevance Assessment: Argo-rollouts waits for stable RS to be stable before scaling it down

## Issue Relevance: CRITICAL

This issue represents a fundamental incompatibility between Argo Rollouts' health checking logic and modern autoscaling Kubernetes deployments. It blocks adoption of progressive delivery for large-scale services and creates operational deadlocks.

## Impact Analysis

### Operational Impact
- **Deployment Deadlocks**: Rollouts get permanently stuck waiting for impossible 100% availability
- **Autoscaling Conflicts**: Prevents effective use of cluster autoscaling and spot instances
- **Resource Inefficiency**: Requires over-provisioning to guarantee rollout completion
- **Operational Complexity**: Manual intervention required for large-scale rollouts
- **Cost Optimization Block**: Prevents cost-effective autoscaling strategies

### Technical Impact
- **Scalability Limits**: Effectively caps rollout size due to health check requirements
- **Automation Breakdown**: CI/CD pipelines fail on large deployments
- **Reliability Issues**: Rollouts become unreliable in dynamic environments
- **PDB Conflicts**: Pod Disruption Budgets create additional constraints

### Affected User Segments
1. **Large-Scale Deployments**: Services with 100+ pods (most affected)
2. **Autoscaling Users**: Teams using Karpenter, Cluster Autoscaler, or similar
3. **Spot Instance Users**: Cost-optimized deployments with spot/preemptible instances
4. **Cloud-Native Teams**: Organizations embracing autoscaling and dynamic infrastructure
5. **Financial Services**: High-reliability requirements with large-scale deployments
6. **E-commerce Platforms**: High-traffic services needing efficient scaling

### Business Value
- **Scalability**: Enables true large-scale progressive delivery
- **Cost Savings**: Supports cost-effective autoscaling strategies
- **Reliability**: Reduces deployment failures and manual interventions
- **Innovation**: Enables advanced deployment patterns for modern applications
- **Competitive Advantage**: Better deployment efficiency vs competitors

## Technical Relevance

### Architecture Impact
- **Health Check Paradigm**: Requires shift from binary to percentage-based health
- **Autoscaling Integration**: Must work with dynamic cluster capacity
- **PDB Compatibility**: Respect Pod Disruption Budget constraints
- **Traffic Routing**: Maintain traffic safety with partial availability

### Code Quality Impact
- **Fundamental Logic Change**: Core health checking logic needs overhaul
- **API Changes**: New parameters for percentage-based health checks
- **Backward Compatibility**: Must preserve existing behavior while adding new options
- **Testing Complexity**: More complex test scenarios for partial availability

### Kubernetes Integration
- **MaxSurge/MaxUnavailable**: Proper integration with existing controls
- **Pod Disruption Budgets**: Respect PDB limitations
- **Cluster Autoscaling**: Work with dynamic capacity changes
- **Spot Instances**: Handle preemption gracefully

## Community Interest

### GitHub Activity
- **Issue Volume**: Multiple reports of rollout deadlocks
- **PR Discussions**: Active development on scaling improvements
- **Community Forums**: Frequent questions about large-scale rollouts
- **Enterprise Users**: High interest from large organizations

### Ecosystem Fit
- **CNCF Alignment**: Supports cloud-native deployment patterns
- **Kubernetes Best Practices**: Aligns with modern K8s deployment strategies
- **Service Mesh Integration**: Works with Istio, Linkerd traffic routing
- **Autoscaling Ecosystem**: Compatible with modern autoscaling tools

## Implementation Feasibility

### Complexity Assessment
- **Technical Complexity**: HIGH - requires fundamental health check redesign
- **API Changes**: MEDIUM - new parameters with backward compatibility
- **Testing Requirements**: HIGH - complex scenarios with partial availability
- **Integration Points**: HIGH - affects multiple components

### Development Effort
- **Analysis**: 2-3 weeks (complex interplay analysis)
- **Design**: 3-4 weeks (percentage-based health logic)
- **Implementation**: 6-8 weeks (core logic changes)
- **Testing**: 4-6 weeks (comprehensive edge case testing)
- **Documentation**: 2-3 weeks (complex new concepts)

### Risk Assessment
- **Technical Risk**: HIGH - fundamental logic changes
- **Compatibility Risk**: MEDIUM - careful backward compatibility
- **Performance Risk**: LOW - health checks are not performance-critical
- **Operational Risk**: HIGH - affects deployment reliability

## Interplay Analysis: MinHealthy, MaxSurge, and MaxUnavailable

### Current Kubernetes Deployment Behavior
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

**Kubernetes Logic**: Allows 25% extra pods (surge) and 25% unavailable pods during rollout.

### Argo Rollouts Current Issue
- **Ignores maxSurge/maxUnavailable**: Demands 100% availability of both RS
- **Double Resource Requirement**: Needs full stable + full canary capacity
- **Autoscaling Conflict**: Cluster can't provide capacity for both simultaneously

### Proposed Solution: MinHealthy Integration

#### Option 1: Independent MinHealthy
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      minHealthy: 80%        # New parameter
      maxSurge: 25%          # Existing
      maxUnavailable: 25%    # Existing
```

**Logic**: Each RS needs 80% healthy pods, respecting surge/unavailable limits.

#### Option 2: MinHealthy as Replacement
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      minHealthy: 80%        # Replaces strict 100% requirement
      # maxSurge/maxUnavailable still apply for pacing
```

**Logic**: MinHealthy becomes the availability requirement, with surge/unavailable controlling rollout speed.

#### Option 3: Contextual MinHealthy
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      minHealthy: 80%
      stableMinHealthy: 90%   # Different requirements for stable vs canary
      maxSurge: 25%
      maxUnavailable: 25%
```

**Logic**: Different health requirements for different RS types.

### Recommended Approach: Option 2
**Rationale**:
- Simplest API change
- Clear semantic meaning
- Works with existing maxSurge/maxUnavailable
- Backward compatible (default 100%)

### Mathematical Interplay

For a rollout with 100 replicas:
- **maxSurge: 25%** = 25 extra pods allowed
- **maxUnavailable: 25%** = 25 pods can be unavailable
- **minHealthy: 80%** = 80 pods must be healthy per RS

**Healthy Deployment Zone**: Between 75-125 pods available (respecting surge limits).

## Priority Recommendation

### CRITICAL PRIORITY
This issue prevents Argo Rollouts from working effectively in modern autoscaling environments. It represents a fundamental architectural limitation that blocks large-scale adoption.

### Implementation Strategy
1. **Introduce minHealthy parameter** with 100% default
2. **Implement percentage-based health checks** in RolloutHealthy
3. **Update scaling logic** to work with partial availability
4. **Add autoscaling awareness** to capacity calculations
5. **Comprehensive testing** for edge cases and large-scale scenarios

### Success Metrics
- **Deployment Success Rate**: >95% success for large-scale rollouts
- **Resource Utilization**: 20-30% reduction in required capacity
- **Autoscaling Compatibility**: Works with spot instances and cluster autoscaling
- **PDB Compliance**: Respects Pod Disruption Budget constraints
- **Rollback Reliability**: Maintains rollback capability with partial availability

### Risk Mitigation
- **Feature Flag**: Gradual rollout with backward compatibility
- **Conservative Defaults**: Start with high minHealthy percentages
- **Extensive Testing**: Test with real large-scale scenarios
- **Monitoring**: Detailed metrics on health check behavior