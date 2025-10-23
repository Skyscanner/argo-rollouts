# Relevance Assessment: scaleDownDelaySeconds & Fast Rollbacks

## Issue Relevance: HIGH

**Impact**: 30-second defaults insufficient for production operator response times during rollbacks.

## Technical Validation

**Validation Still Active**: `pkg/apis/rollouts/validation/validation.go#L294-295` blocks basic canary usage.

**Test Evidence**: `TestCanaryScaleDownDelaySeconds` confirms validation error for basic canary.

**Fast Rollback Gap**: Blue-green uses `HasScaleDownDeadline()` for instant promotion; canary lacks equivalent.

## Production Testing Results

**Configuration Tested**:
- `scaleDownDelaySeconds: 30` (default)
- `rollbackWindow.revisions: 3`

**Findings**:
- Scale down delay works correctly
- 30s window insufficient for real pipelines
- Rollback window behavior inconsistent
- Analysis termination takes 20-30+ seconds

## Impact Analysis

**Operational Impact**: Rollback failures due to insufficient delay windows require manual intervention.

**Technical Impact**: Existing mechanisms work well with appropriate values; issue is configuration optimization.

**Affected Segments**: Production teams, platform teams, DevOps engineers, SREs.

## Implementation Feasibility

**Primary Approach**: Chart value optimization (low risk, immediate impact).

**Success Criteria**: >95% rollbacks complete within delay window, clear production guidance.

## Manual Exploration Required

**Investigate Further**:
- Review GitHub issue #557 for user requirements
- Examine production rollback scenarios and timing needs
- Analyze rollback window vs scaleDownDelaySeconds trade-offs
- Test configuration changes in staging environments

**Key Question**: What delay values work for different team sizes and incident response processes?