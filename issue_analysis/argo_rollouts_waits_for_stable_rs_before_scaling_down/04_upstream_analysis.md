# Upstream Analysis: Argo-rollouts waits for stable RS to be stable before scaling it down

## Argo Rollouts Ecosystem

### Related Projects and Dependencies
- **Kubernetes**: Core dependency with evolving deployment controls
- **Istio/Service Mesh**: Traffic routing with health awareness
- **Autoscaling Systems**: Karpenter, Cluster Autoscaler, KEDA
- **Pod Disruption Budgets**: Kubernetes native disruption controls

### Community and Governance
- **CNCF Project**: Part of Cloud Native Computing Foundation
- **Enterprise Adoption**: Used by large organizations with complex deployments
- **Autoscaling Integration**: Growing need for dynamic infrastructure compatibility

## Similar Issues in Related Projects

### Kubernetes Deployment Controller

**Native Kubernetes Deployments** handle this scenario differently:

```yaml
apiVersion: apps/v1
kind: Deployment
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

**Key Differences**:
- **Percentage-Based**: Uses percentages, not absolute pod counts
- **Capacity Aware**: Works within cluster capacity constraints
- **PDB Respect**: Integrates with Pod Disruption Budgets
- **Autoscaling Compatible**: Designed for dynamic cluster environments

**Argo Rollouts Gap**: Doesn't leverage these Kubernetes patterns.

### Progressive Delivery Tools Comparison

#### Flagger
**Health Check Approach**:
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
spec:
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
  healthChecks:
    - name: http-get
      timeout: 5s
      interval: 10s
```

**Key Features**:
- **Percentage-Based**: Uses weight percentages for traffic distribution
- **Health Thresholds**: Configurable success thresholds
- **Autoscaling Aware**: Works with HPA and custom metrics
- **PDB Compatible**: Respects disruption budgets

#### Keptn
**Health Check Logic**:
```yaml
apiVersion: lifecycle.keptn.sh/v1alpha3
kind: KeptnEvaluation
spec:
  evaluationDefinition: my-health-check
  retries: 3
  retryInterval: 10s
```

**Advanced Features**:
- **Custom Health Checks**: Extensible evaluation logic
- **Percentage Metrics**: SLO-based health evaluation
- **Autoscaling Integration**: Works with multiple autoscaling systems
- **Large-Scale Support**: Designed for enterprise-scale deployments

#### Comparison Table

| Tool | Health Check Type | Autoscaling Support | PDB Awareness | Large-Scale Support |
|------|-------------------|-------------------|----------------|-------------------|
| Argo Rollouts | Binary (100%) | Limited | No | Poor |
| Flagger | Percentage/Threshold | Good | Yes | Good |
| Keptn | SLO-Based | Excellent | Yes | Excellent |
| K8s Deployment | Percentage | Basic | Yes | Good |

## Industry Patterns and Best Practices

### Progressive Delivery Best Practices
1. **Percentage-Based Health**: Use minAvailable-style requirements
2. **Capacity Awareness**: Design for cluster resource limitations
3. **PDB Integration**: Respect Pod Disruption Budget constraints
4. **Autoscaling Compatibility**: Work with dynamic infrastructure
5. **Graceful Degradation**: Handle partial failures gracefully

### Cloud-Native Deployment Patterns
- **HPA-Compatible**: Work with Horizontal Pod Autoscalers
- **Cluster Autoscaling**: Compatible with CA and Karpenter
- **Spot Instances**: Handle preemption and capacity loss
- **Multi-Zone**: Survive zone failures and capacity issues

### Kubernetes Deployment Controls Evolution

#### Historical Kubernetes Deployments
- **Early Versions**: Simple rolling updates with basic controls
- **1.10+**: Introduction of maxSurge/maxUnavailable
- **Modern**: Advanced deployment controls and PDB integration

#### Argo Rollouts vs Kubernetes
- **Kubernetes**: Mature percentage-based deployment logic
- **Argo Rollouts**: Still using binary health requirements
- **Gap**: Argo Rollouts doesn't leverage modern K8s deployment patterns

## Technical Standards and Specifications

### Kubernetes Resource Management
- **Pod Disruption Budgets**: Standard for safe pod management
- **Resource Quotas**: Capacity planning and enforcement
- **Priority Classes**: Pod scheduling priorities
- **Taints/Tolerations**: Node selection and constraints

### Service Mesh Health Standards
- **Istio Health Checks**: Percentage-based endpoint health
- **Linkerd Metrics**: Traffic and health metrics
- **Envoy Configuration**: Circuit breaking and health checking

### Autoscaling Standards
- **HPA Algorithms**: Percentage-based scaling decisions
- **Cluster Autoscaler**: Capacity-aware node scaling
- **Karpenter**: Workload-aware instance provisioning

## Community Solutions and Workarounds

### Current Workarounds
1. **Small Deployments**: Limit rollout size to avoid capacity issues
2. **Over-Provisioning**: Maintain excess cluster capacity
3. **Manual Scaling**: Manually adjust RS during stuck rollouts
4. **PDB Modifications**: Temporarily relax PDB constraints
5. **Custom Controllers**: Build custom logic on top of Argo Rollouts

### Community Discussions
- **GitHub Issues**: "Rollouts stuck waiting for pods that can't schedule"
- **Slack/Forums**: "How to make Argo Rollouts work with autoscaling?"
- **Blog Posts**: Workarounds for large-scale Argo Rollouts deployments
- **Enterprise Guides**: Internal documentation for handling scaling issues

### Open Source Extensions
- **Custom Health Checks**: Community-built health check extensions
- **Autoscaling Controllers**: Custom controllers for Argo Rollouts scaling
- **PDB Managers**: Tools to dynamically adjust PDBs during rollouts

## Research and Academic References

### Progressive Delivery Research
- **Google SRE Book**: Service reliability and deployment practices
- **CNCF Whitepapers**: Cloud-native deployment patterns
- **Academic Papers**: Research on deployment efficiency and autoscaling

### Autoscaling Research
- **Cluster Autoscaling Algorithms**: Research on capacity planning
- **Spot Instance Management**: Handling preemptible workloads
- **PDB Optimization**: Research on disruption budget management

### Industry Benchmarks
- **Deployment Performance**: Benchmarks of different rollout tools
- **Autoscaling Efficiency**: Studies on autoscaling system performance
- **Large-Scale Deployments**: Research on very large Kubernetes deployments

## Upstream Dependencies Impact

### Kubernetes Changes
- **Deployment API Evolution**: New deployment controls and features
- **PDB Enhancements**: Improved disruption budget management
- **Autoscaling Improvements**: Better autoscaling algorithms and integration

### Service Mesh Evolution
- **Health Checking**: More sophisticated health check capabilities
- **Traffic Management**: Advanced traffic routing and splitting
- **Observability**: Better monitoring of service health and traffic

### Autoscaling Ecosystem
- **Karpenter**: Workload-aware instance provisioning
- **KEDA**: Event-driven autoscaling
- **Custom Metrics**: More sophisticated scaling metrics

## Future Trends and Predictions

### Industry Direction
1. **Percentage-Based Deployments**: Move away from binary health requirements
2. **AI/ML Integration**: Intelligent autoscaling and deployment decisions
3. **Multi-Cluster**: Cross-cluster deployment orchestration
4. **Edge Computing**: Deployment patterns for edge environments
5. **Serverless Integration**: Compatibility with serverless workloads

### Argo Rollouts Roadmap Alignment
- **Efficiency Improvements**: Better resource utilization
- **Autoscaling Integration**: Enhanced compatibility with autoscaling
- **Large-Scale Support**: Support for very large deployments
- **Advanced Health Checks**: More sophisticated health evaluation

### Competitive Pressure
- **Flagger Adoption**: Growing adoption of percentage-based tools
- **Keptn Integration**: More organizations using SLO-based deployments
- **Kubernetes Native**: Competition from improved native deployment controls

## Upstream Analysis Summary

This upstream analysis reveals that Argo Rollouts is significantly behind industry standards for health checking and autoscaling compatibility. While other progressive delivery tools have adopted percentage-based health checks and autoscaling awareness, Argo Rollouts still uses binary requirements that conflict with modern Kubernetes deployments.

**Key Upstream Insights**:
- **Industry Standard**: Percentage-based health checks are the norm
- **Kubernetes Evolution**: Modern K8s features support flexible deployments
- **Competitive Gap**: Other tools provide better large-scale and autoscaling support
- **Community Need**: Strong demand for improved autoscaling compatibility

The solution must bridge this gap by implementing percentage-based health checking that aligns with Kubernetes best practices and competitor capabilities. The fix should include `minHealthy` parameters and proper integration with `maxSurge`/`maxUnavailable` controls.

**Critical Finding**: This isn't just an Argo Rollouts issue—it's a fundamental architectural limitation that prevents adoption in modern autoscaling environments where other tools already excel.