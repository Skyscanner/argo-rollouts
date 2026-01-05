---
title: Canary Pre-Warming
authors:
  - '@akashrajput'
sponsors:
  - TBD
creation-date: 2026-01-05
---

# Canary Pre-Warming

Pre-warming enables faster canary rollouts by provisioning pods for the next canary step in parallel with the current step, eliminating pod startup delays between step transitions.

## Summary

Currently, canary rollouts wait for each step to complete before scaling pods for the next step. This introduces delays equal to pod startup time (which can range from seconds to minutes) at each step transition. Pre-warming provisions the pods needed for the next step while the current step is executing, so pods are ready when the step transitions.

This proposal introduces an opt-in `prewarmNextStep` field in the `CanaryStrategy` that enables parallel pod provisioning for the next canary step.

## Motivation

### Current Behavior

Today's canary rollout flow:

```
Step 1 (10% traffic, 1 pod):
├─ Scale canary to 1 pod
├─ Wait for pod to be ready (30s)
├─ Apply traffic routing
├─ Execute analysis/pause
└─ Complete step

Step 2 (25% traffic, 3 pods):
├─ Scale canary to 3 pods       ← Wait for 2 new pods (30s)
├─ Wait for pods to be ready    ← Unnecessary delay!
├─ Apply traffic routing
└─ ...
```

**Problem**: Each step waits for pod startup before proceeding, adding significant time to rollouts.

### Proposed Behavior

With pre-warming enabled:

```
Step 1 (10% traffic, 1 pod):
├─ Scale canary to 1 pod
├─ Wait for pod to be ready
├─ Apply traffic routing (10%)
├─ Pre-warm 2 additional pods   ← Parallel to current step
├─ Execute analysis/pause
└─ Complete step

Step 2 (25% traffic, 3 pods):
├─ Pods already ready!           ← No wait needed
├─ Apply traffic routing (25%)  ← Instant transition
└─ ...
```

**Benefit**: Eliminates pod startup delays between steps, significantly speeding up rollouts.

### Goals

- Reduce total rollout time by eliminating pod startup delays between canary steps
- Provide opt-in mechanism for pre-warming via API field
- Respect cluster resource constraints (maxSurge for basic canary)
- Maintain consistency with traffic-routed canary behavior (natural surge allowed)
- Maintain rollback safety
- Work with both basic canary and traffic-routed canary strategies
- Handle edge cases gracefully (last step, pause steps, abort scenarios)

### Non-Goals

- Pre-warming for blue-green deployments (future work)
- Pre-warming multiple steps ahead (limited to next step only)
- Support with `dynamicStableScale: true` (safety constraint)
- Predictive pre-warming based on historical data

## Proposal

### API Changes

#### Spec Field

Add an opt-in field to `CanaryStrategy`:

```go
type CanaryStrategy struct {
    // ... existing fields ...

    // PrewarmNextStep enables pre-provisioning of pods for the next canary step in parallel
    // with the current step to speed up rollout transitions. When enabled, the controller
    // will calculate the pod requirements for the next step and scale up those pods within
    // maxSurge limits while the current step is still executing.
    //
    // Note: This feature is disabled when DynamicStableScale is enabled, as pre-warming
    // would aggressively scale down stable pods before traffic shifts, reducing rollback capacity.
    //
    // Default: false (disabled)
    // +optional
    PrewarmNextStep bool `json:"prewarmNextStep,omitempty" protobuf:"varint,18,opt,name=prewarmNextStep"`
}
```

#### Status Field

Add status tracking to `CanaryStatus`:

```go
type CanaryStatus struct {
    // ... existing fields ...

    // PrewarmedReplicas tracks the number of replicas that have been pre-provisioned
    // for the next step
    // +optional
    PrewarmedReplicas *int32 `json:"prewarmedReplicas,omitempty" protobuf:"varint,7,opt,name=prewarmedReplicas"`
}
```

### Use Cases

#### Use Case 1: Traffic-Routed Canary with Slow Pods

**Scenario**: Microservice with 30-second pod startup time, using Istio traffic routing

**Configuration**:
```yaml
spec:
  replicas: 10
  strategy:
    canary:
      prewarmNextStep: true
      maxSurge: "50%"
      trafficRouting:
        istio:
          virtualService:
            name: my-service
      steps:
        - setWeight: 10
        - pause: { duration: 2m }
        - setWeight: 25
        - pause: { duration: 2m }
        - setWeight: 50
```

**Behavior**:
- Step 1: 1 canary pod serves 10% traffic, 2 extra pods pre-warmed (total: 3 canary + 10 stable)
- Step 2: Instant transition to 25% traffic (3 pods already ready)
- Pre-warm continues for step 3 (5 pods needed)

**Time Savings**: 30 seconds per step × number of steps

#### Use Case 2: Basic Canary with Moderate Surge

**Scenario**: Basic canary without traffic routing, limited maxSurge

**Configuration**:
```yaml
spec:
  replicas: 10
  strategy:
    canary:
      prewarmNextStep: true
      maxSurge: "30%"
      steps:
        - setWeight: 20
        - pause: { duration: 1m }
        - setWeight: 50
```

**Behavior**:
- Step 1: 2 canary + 8 stable pods, pre-warm within 3 pods surge capacity
- Proportional pre-warming if full next-step requirement exceeds surge

#### Use Case 3: ALB Traffic Routing with High Replicas

**Scenario**: High-traffic service with 20 replicas, AWS ALB

**Configuration**:
```yaml
spec:
  replicas: 20
  strategy:
    canary:
      prewarmNextStep: true
      maxSurge: "40%"
      trafficRouting:
        alb:
          ingress: my-ingress
      steps:
        - setWeight: 5
        - pause: { duration: 1m }
        - setWeight: 10
        - pause: { duration: 1m }
        - setWeight: 20
```

**Behavior**:
- Aggressive pre-warming with 8 pods surge capacity
- Multiple steps benefit from pre-warming
- Faster overall deployment time

### Implementation Details

#### Controller Logic

**File**: `rollout/canary.go`

1. **Pre-warming Check** (`shouldPrewarmNextStep()`):
   - Feature flag enabled
   - NOT using `dynamicStableScale: true`
   - Not paused or aborting
   - Current step requirements met
   - Has next step available

2. **Replica Calculation** (`reconcilePrewarmedReplicas()`):
   - Calculate next step replica requirements
   - Determine additional replicas needed
   - Check available surge capacity
   - Scale proportionally if capacity limited
   - Update status with pre-warmed count

3. **Integration** (`reconcileCanaryReplicaSets()`):
   - Call pre-warming logic when conditions met
   - Clean up on abort/rollback
   - Continue with normal reconciliation

4. **Cleanup** (`syncRolloutStatusCanary()`):
   - Reset pre-warmed counter on step completion
   - Transition seamlessly to next step

#### Replica Calculation Logic

**File**: `utils/replicaset/canary.go`

**New Function**: `CalculateNextStepReplicaCounts()`

```go
func CalculateNextStepReplicaCounts(
    ro *v1alpha1.Rollout,
    newRS, stableRS *appsv1.ReplicaSet,
    weights *v1alpha1.TrafficWeights,
) (canaryCount, stableCount int32, hasNextStep bool) {
    // 1. Get current step index
    // 2. Check if next step exists
    // 3. Skip pause steps, look ahead to actionable step
    // 4. Create temp rollout with next step index
    // 5. Call existing calculation functions
    // 6. Return next step replica requirements
}
```

**Strategy**:
- Reuses existing `CalculateReplicaCountsForBasicCanary()` and `CalculateReplicaCountsForTrafficRoutedCanary()`
- Maintains consistency with current replica calculation logic
- Automatically handles all edge cases (minPodsPerReplicaSet, explicit scales, etc.)

#### MaxSurge Enforcement

Pre-warming behavior differs based on canary type to maintain consistency with normal rollout behavior:

**Basic Canary (no traffic routing)**:
Pre-warming respects `maxSurge` constraints:

```
Available Surge = (spec.replicas + maxSurge) - current total replicas

If (additional replicas needed) > available surge:
    Scale proportionally across canary and stable
```

**Example**:
- Spec: 10 replicas, maxSurge: 2
- Current: 2 canary + 8 stable = 10 total
- Available surge: 10 + 2 - 10 = 2 pods
- Next step needs: 3 additional canary
- Pre-warm: 2 pods only (within surge)

**Traffic-Routed Canary (with traffic routing)**:
Pre-warming allows natural surge without maxSurge constraints:

```
Pre-warm to full next step requirements (no maxSurge check)
```

**Rationale**: Traffic-routed canaries already ignore maxSurge during normal progression (stable RS stays at 100%, canary scales based on traffic weight). Pre-warming maintains this consistency.

**Example**:
- Spec: 10 replicas, maxSurge: 1 (ignored)
- Current: 2 canary + 10 stable = 12 total
- Next step needs: 5 canary + 10 stable = 15 total
- Pre-warm: 3 additional canary (total: 15 pods, 50% surge)

### Edge Cases

#### 1. Last Step
**Behavior**: No next step exists, pre-warming disabled
**Handling**: `CalculateNextStepReplicaCounts()` returns `hasNextStep = false`

#### 2. Pause Steps
**Behavior**: Skip pause steps, look ahead to next actionable step
**Handling**: Loop through steps until non-pause step found

```go
for nextStep.Pause != nil && hasMoreSteps {
    nextStepIndex++
    nextStep = steps[nextStepIndex]
}
```

#### 3. Only Pause Steps Remaining
**Behavior**: No actionable next step, pre-warming disabled
**Handling**: Return `hasNextStep = false` after loop

#### 4. Abort/Rollback
**Behavior**: Immediately clean up pre-warmed pods
**Handling**: Set `prewarmedReplicas = 0`, normal reconciliation scales down excess

#### 5. SetCanaryScale with Explicit Replicas
**Behavior**: Pre-warm to explicit replica count
**Handling**: Existing `GetCanaryReplicasOrWeight()` handles explicit counts

#### 6. DynamicStableScale Enabled
**Behavior**: Pre-warming completely disabled
**Handling**: `shouldPrewarmNextStep()` returns false
**Reason**: Would scale down stable prematurely, reducing rollback capacity

#### 7. MinPodsPerReplicaSet
**Behavior**: Honors minimum pod requirements
**Handling**: Existing `CheckMinPodsPerReplicaSet()` applied automatically

#### 8. Insufficient Surge Capacity

**Behavior**: Only applies to basic canary. Pre-warm proportionally within available capacity.
**Handling**: Scale canary and stable proportionally based on available surge

```go
if totalNeeded > availableSurge {
    ratio = availableSurge / totalNeeded
    additionalCanary = additionalCanary * ratio
    additionalStable = availableSurge - additionalCanary
}
```

**Note**: For traffic-routed canary, this constraint doesn't apply - full pre-warming is allowed regardless of maxSurge (consistent with normal traffic-routed behavior).

### Observability

#### Metrics (Future Work)

Potential metrics to add:

- `argo_rollouts_prewarm_enabled{rollout, namespace}` - Gauge of rollouts with pre-warming
- `argo_rollouts_prewarm_pods_total{rollout, namespace}` - Counter of pre-warmed pods
- `argo_rollouts_prewarm_step_duration{rollout, namespace}` - Histogram of step duration with pre-warming

#### Status Fields

Users can check pre-warming status:

```bash
kubectl get rollout my-rollout -o jsonpath='{.status.canary.prewarmedReplicas}'
```

#### Controller Logs

Pre-warming events logged at INFO level:

```
Pre-warming canary RS from 1 to 3
Scaled pre-warmed replicas for next step
Current step completed, transitioning to pre-warmed next step
No surge capacity available for pre-warming
Cleaning up pre-warmed replicas due to rollback/abort
```

#### Events

Kubernetes events emitted:

- `ScaledReplicaSet` - When pre-warming scales replicas
- `RolloutStepCompleted` - Includes transition to pre-warmed step

### Security Considerations

**Resource Consumption**:
- Pre-warmed pods consume cluster resources (CPU, memory)
- For basic canary: Bounded by `maxSurge` configuration
- For traffic-routed canary: No maxSurge constraint (consistent with normal behavior)
- Users should plan cluster capacity accordingly

**DoS Risk**:
- For basic canary: Bounded by existing `maxSurge` configuration
- For traffic-routed canary: Limited by next step requirements (similar to normal rollout surge)
- Mitigated by existing RBAC and resource quotas
- No new attack vectors introduced

**Privilege Escalation**:
- No additional permissions required
- Uses existing controller service account
- No changes to RBAC model

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Excessive resource usage | Cluster resource exhaustion | Basic canary: bounded by maxSurge. Traffic-routed: natural surge like normal rollout |
| DynamicStableScale conflict | Premature stable scale-down | Automatically disabled when dynamicStableScale=true |
| Abort during pre-warming | Wasted resources | Immediate cleanup on abort/rollback |
| Complex replica calculations | Bugs, edge cases | Reuse existing battle-tested calculation functions |
| User confusion | Misconfiguration | Comprehensive documentation, examples, defaults |

### Alternatives Considered

#### Alternative 1: Always Pre-warm (No Opt-in)

**Rejected**: Could surprise users with unexpected resource usage and behavior changes.

#### Alternative 2: Pre-warm N Steps Ahead

**Deferred**: More complex, limited benefit. Can be added later if needed.

#### Alternative 3: Support with DynamicStableScale

**Rejected**: Safety concern - would scale down stable pods before traffic shifts, reducing rollback capacity. The benefit of faster rollouts doesn't justify the rollback risk.

#### Alternative 4: Separate Pre-warming Controller

**Rejected**: Adds complexity, requires additional deployment. Better to integrate into existing controller.

#### Alternative 5: Blue-Green Pre-warming

**Deferred**: Different use case, requires different implementation. Can be added in future proposal.

## Compatibility

### Backward Compatibility

- ✅ **API**: New fields are optional, default to existing behavior
- ✅ **CRD**: Additive changes only, no breaking changes
- ✅ **Controller**: Feature gated by `prewarmNextStep` flag
- ✅ **Existing Rollouts**: Unaffected, continue working as before

### Forward Compatibility

- ✅ **Future Enhancements**: Design allows for N-step pre-warming
- ✅ **Other Strategies**: Can be extended to blue-green (future)
- ✅ **Metrics**: Status field enables future metrics

## Testing

### Unit Tests

**File**: `utils/replicaset/canary_test.go`

Comprehensive test coverage for `CalculateNextStepReplicaCounts()`:

- ✅ Basic canary - next step increases weight
- ✅ Traffic-routed canary - next step increases weight
- ✅ Last step - no next step available
- ✅ Skip pause steps
- ✅ Only pause steps remaining
- ✅ SetCanaryScale explicit replicas
- ✅ No current step
- ✅ MinPodsPerReplicaSet constraints

### Integration Tests (Future Work)

Recommended integration tests:

1. **E2E Happy Path**: Full rollout with pre-warming enabled
2. **Abort Scenario**: Verify immediate cleanup on abort
3. **MaxSurge Limits**: Verify surge constraints respected
4. **DynamicStableScale**: Verify pre-warming disabled
5. **Traffic Routing**: Verify with Istio/ALB/NGINX
6. **Basic Canary**: Verify without traffic routing

### Manual Testing

Test cases for manual validation:

1. Deploy example rollouts from `examples/prewarm-canary/`
2. Monitor controller logs for pre-warming messages
3. Check `prewarmedReplicas` status field
4. Trigger abort and verify cleanup
5. Test with various `maxSurge` values
6. Verify step transition timing improvements

## Implementation Plan

### Phase 1: Core Implementation ✅ COMPLETED

- [x] API changes (types.go)
- [x] Core logic (canary.go, replicaset/canary.go)
- [x] Unit tests
- [x] CRD generation
- [x] Examples and documentation

### Phase 2: Testing & Refinement

- [ ] Integration tests
- [ ] E2E tests with real clusters
- [ ] Performance benchmarking
- [ ] User documentation updates

### Phase 3: Observability

- [ ] Prometheus metrics
- [ ] Additional controller logs
- [ ] Grafana dashboard examples

### Phase 4: Future Enhancements

- [ ] N-step pre-warming (configurable look-ahead)
- [ ] Blue-green pre-warming
- [ ] Smart pre-warming based on historical data
- [ ] Support with DynamicStableScale (if safe approach found)

## FAQ

### Q: Why not always enable pre-warming?

**A**: Pre-warming consumes additional cluster resources (temporary extra pods). Users should opt-in and configure appropriate `maxSurge` values for their environment.

### Q: Does this work with blue-green deployments?

**A**: Not currently. This proposal focuses on canary deployments. Blue-green pre-warming could be a future enhancement.

### Q: What happens if maxSurge is too small?

**A**: For basic canary (no traffic routing), pre-warming scales proportionally within available surge capacity. If surge is 0, no pre-warming occurs. For traffic-routed canary, maxSurge doesn't apply - pre-warming proceeds to full next step requirements (consistent with how traffic-routed canaries already allow natural surge during normal progression).

### Q: Can I pre-warm multiple steps ahead?

**A**: Not in this initial implementation. The feature is limited to the immediate next step to keep complexity low. This could be added later if needed.

### Q: Why is it disabled with dynamicStableScale?

**A**: Pre-warming with dynamic stable scaling would scale down stable pods before traffic shifts to canary, reducing the capacity available for instant rollback. This compromises rollback safety.

### Q: Does this affect rollback behavior?

**A**: No. On abort/rollback, pre-warmed pods are immediately cleaned up. Rollback proceeds normally.

### Q: What's the performance impact?

**A**: Eliminates pod startup time between steps. For pods with 30s startup: saves 30s per step. For 5 steps: saves 2.5 minutes total rollout time.

### Q: Are there any security concerns?

**A**: No new attack vectors. Pre-warming is bounded by maxSurge (user-configured) and existing RBAC/quotas apply.

## References

- [Argo Rollouts Canary Strategy](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
- [Traffic Management](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/)
- [MaxSurge/MaxUnavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge)
- [Issue #XXXX](https://github.com/argoproj/argo-rollouts/issues/XXXX) - Original feature request (TBD)
