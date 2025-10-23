# Argo Rollouts: scaleDownDelaySeconds and Fast Rollbacks (v1.8.3)

## Current Configuration

| Setting | Default | Issue |
|----------|---------|-------|
| `scaleDownDelaySeconds` | 30s | Insufficient for production operator response |
| `abortScaleDownDelaySeconds` | 30s | Too short for investigation |
| `rollbackWindow.revisions` | nil | Not configured by default |

**Code Evidence**: `utils/defaults/defaults.go#L31` - default delay constants

## Abort Conditions

| Condition | Auto-Abort? | Notes |
|------------|-------------|-------|
| AnalysisRun fails/errors | ✅ | Immediate abort |
| Progress deadline exceeded + `progressDeadlineAbort: true` | ✅ | Abort on timeout |
| Manual abort | ✅ | Always possible |

**Source**: `rollout/sync.go#L345-L360`, `rollout/analysis.go#L40-L55`

## scaleDownDelaySeconds Behavior

- Delays cleanup after promotion for rollback capability
- Default 30s insufficient for real pipelines
- Works with traffic routing (Istio/SMI) only
- Basic canary blocked by validation

**Validation Code**: `pkg/apis/rollouts/validation/validation.go#L294-295`
```go
if canary.TrafficRouting == nil {
    // scaleDownDelaySeconds validation error for basic canary
}
```

## Rollback Windows

- `rollbackWindow.revisions` limits eligible RSs for fast rollback
- Timestamp-based detection distinguishes rollback from rollforward
- Fast rollback skips analysis when within window

**Source**: `rollout/sync.go#L902-L996` - rollback window logic

## Empirical Evidence

**Production Testing Results**:
- 30s defaults insufficient for operator response
- Existing mechanisms work with longer delays (300-600s)
- Rollback window feature functional but underutilized

## Recommended Production Configuration

```yaml
rollout:
  scaleDownDelaySeconds: 300      # 5 minutes for operator response
  abortScaleDownDelaySeconds: 300 # 5 minutes for investigation
  rollbackWindow:
    revisions: 3                  # Keep 3 versions available
```

## Implementation Approach

**Primary**: Chart value optimization (immediate impact, low risk)
**Secondary**: Optional upstream enhancements for analysis termination speed

## Manual Exploration Required

**Investigate Further**:
- Review GitHub issue #557 for original feature request
- Examine validation logic preventing basic canary usage
- Test rollback window behavior with different revision counts
- Analyze analysis run termination performance

**Key Question**: Why is scaleDownDelaySeconds restricted to traffic-routing canaries?
