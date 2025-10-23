# Argo Rollouts Ignores maxSurge/maxUnavailable in Traffic Shifting

## Source-Backed Facts

### Current Implementation
- **Traffic-routed canaries ignore maxSurge/maxUnavailable**: Code in `utils/replicaset/canary.go` shows `CalculateReplicaCountsForTrafficRoutedCanary` does not apply these limits
- **Basic canary respects limits**: `CalculateReplicaCountsForBasicCanary` applies maxSurge/maxUnavailable to total rollout replicas
- **Design decision**: Traffic routing prioritizes traffic control over pod scaling limits (maintainer position from GitHub issues)

### GitHub Issues
- **#3284**: User reports maxSurge/maxUnavailable ignored in traffic-routed deployments
- **#3539**: Request for scaling control in traffic-routed canaries  
- **#3397**: Discussion of scaling behavior differences between canary types
- **#2239**: Related scaling behavior analysis

### Code References
- `rollout/canary.go`: Traffic weight to replica calculations
- `utils/replicaset/canary.go`: Replica count calculation functions
- `utils/conditions/conditions.go`: RolloutHealthy function with availability requirements

## Manual Exploration Required

### Implementation Questions
- How does `dynamicStableScale` setting affect scaling behavior?
- What are the exact replica calculation differences between basic and traffic-routed canaries?
- How does `minPodsPerReplicaSet` interact with traffic routing?

### Design Philosophy Questions  
- Should traffic routing continue to override scaling limits?
- What are the trade-offs between traffic control and scaling control?
- How do other progressive delivery tools handle this conflict?

### User Impact Questions
- What specific scaling problems do users encounter without these limits?
- How effective are current workarounds (manual steps, minPodsPerReplicaSet)?
- What validation warnings should be provided for unsupported configurations?