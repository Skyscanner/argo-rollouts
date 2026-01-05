# Pre-Warming Canary Rollouts Examples

This directory contains example Rollout manifests demonstrating the `prewarmNextStep` feature for faster canary deployments.

## What is Pre-Warming?

Pre-warming provisions pods for the next canary step in parallel with the current step, eliminating pod startup time between steps. This significantly speeds up rollouts, especially with traffic routing where pods can be ready without receiving traffic.

## Key Benefits

- **Faster Rollouts**: Eliminates pod startup delay between steps
- **Traffic Routing Optimized**: Works best with Istio, ALB, NGINX, etc.
- **Resource Safe**: Respects `maxSurge` limits
- **Rollback Safe**: Automatically disabled when `dynamicStableScale: true`

## Examples

### 1. Traffic-Routed Canary (Recommended)

**File**: `rollout-traffic-routing.yaml`

Best use case for pre-warming. Pods are provisioned ahead of time but only receive traffic based on routing rules.

```yaml
spec:
  replicas: 10
  strategy:
    canary:
      prewarmNextStep: true  # Enable pre-warming
      trafficRouting:
        istio:
          virtualService:
            name: rollout-vsvc
      maxSurge: "50%"  # Allow surge capacity for pre-warming
      steps:
        - setWeight: 10
        - pause: { duration: 2m }
        - setWeight: 25  # Pods pre-warmed while on 10%
```

**Timeline**:
```
Step 1 (10%): 1 canary pod + 10 stable
  → Pre-warms 2 more canary pods (total: 3)
  → Only 10% traffic to canary

Step 2 (25%): Instant transition!
  → 3 canary pods already ready
  → Update routing to 25%
```

### 2. Basic Canary

**File**: `rollout-basic-canary.yaml`

Works with basic canary (no traffic routing), though less impactful since pods immediately serve traffic.

```yaml
spec:
  strategy:
    canary:
      prewarmNextStep: true
      maxSurge: "30%"
      steps:
        - setWeight: 20
        - setWeight: 40  # Pre-warmed during step 1
```

### 3. ALB Traffic Routing

**File**: `rollout-alb-traffic.yaml`

Demonstrates pre-warming with AWS ALB Ingress Controller for high-replica deployments.

```yaml
spec:
  replicas: 20
  strategy:
    canary:
      prewarmNextStep: true
      trafficRouting:
        alb:
          ingress: rollout-ingress
      maxSurge: "40%"  # 8 extra pods allowed
```

## Configuration

### Required Fields

```yaml
strategy:
  canary:
    prewarmNextStep: true  # Enable the feature
    maxSurge: "X%"         # Must allow surge capacity
```

### Compatibility

✅ **Works With**:
- Traffic routing (Istio, ALB, NGINX, etc.) - **RECOMMENDED**
- Basic canary (replica-based)
- `SetWeight` steps
- `SetCanaryScale` steps
- Pause steps (automatically skipped)

❌ **Not Compatible With**:
- `dynamicStableScale: true` (automatically disabled for safety)

### Best Practices

1. **Use with Traffic Routing**: Maximum benefit when traffic is decoupled from replicas
2. **Configure maxSurge**: Allow enough surge capacity for pre-warming
   - Example: 50% surge for aggressive pre-warming
   - Example: 25% surge for conservative approach
3. **Avoid DynamicStableScale**: Pre-warming is disabled with `dynamicStableScale: true`
4. **Monitor Resources**: Pre-warmed pods consume cluster resources

## How It Works

### Traffic-Routed Example

```
Current Step: 10% weight (1 canary pod)
├─ Normal: Wait for step completion
├─ With Pre-warming:
│  ├─ Calculate next step needs 25% (3 pods)
│  ├─ Check maxSurge: 5 pods available
│  ├─ Scale canary to 3 pods immediately
│  └─ Only 10% traffic to canary (1 pod worth)
└─ Next Step: Instant transition to 25%!
```

### Resource Management

Pre-warming respects cluster resources:
- **MaxSurge**: Never exceeds `spec.replicas + maxSurge`
- **Proportional Scaling**: If surge capacity limited, scales proportionally
- **Abort Handling**: Immediately scales down on rollback/abort

## Deployment

```bash
# Deploy example
kubectl apply -f rollout-traffic-routing.yaml

# Trigger update
kubectl argo rollouts set image rollout-prewarm-traffic-routing \
  rollouts-demo=argoproj/rollouts-demo:yellow

# Watch progress
kubectl argo rollouts get rollout rollout-prewarm-traffic-routing --watch

# Check pre-warmed replicas in status
kubectl get rollout rollout-prewarm-traffic-routing -o jsonpath='{.status.canary.prewarmedReplicas}'
```

## Observability

### Status Fields

```yaml
status:
  canary:
    prewarmedReplicas: 2  # Number of pre-warmed pods
```

### Logs

Look for pre-warming messages in controller logs:

```
Pre-warming canary RS from 1 to 3
Scaled pre-warmed replicas for next step
Current step completed, transitioning to pre-warmed next step
```

## Troubleshooting

### Pre-warming not activating?

Check:
1. `prewarmNextStep: true` is set
2. `maxSurge > 0` allows surge capacity
3. NOT using `dynamicStableScale: true`
4. Current step requirements are met
5. Not paused or aborting

### Not seeing speed improvements?

- Works best with traffic routing (Istio, ALB, etc.)
- Basic canary has less benefit since pods immediately serve traffic
- Check if pod startup time is the bottleneck

### Excessive resource usage?

- Reduce `maxSurge` value
- Pre-warmed pods are temporary (only during step transitions)
- Consider if benefits justify resource costs

## Performance Impact

**Typical Improvements** (with traffic routing):

| Scenario | Without Pre-warming | With Pre-warming | Improvement |
|----------|---------------------|------------------|-------------|
| Fast pods (5s startup) | ~5s/step | ~0s/step | **100%** |
| Medium pods (30s startup) | ~30s/step | ~0s/step | **100%** |
| Slow pods (2m startup) | ~2m/step | ~0s/step | **100%** |

**Resource Overhead**:
- Temporary additional pods during each step
- Bounded by `maxSurge` setting
- Example: 10 replicas, 50% surge = max 5 extra pods

## Additional Resources

- [Argo Rollouts Documentation](https://argo-rollouts.readthedocs.io/)
- [Traffic Management](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/)
- [Canary Strategy](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
