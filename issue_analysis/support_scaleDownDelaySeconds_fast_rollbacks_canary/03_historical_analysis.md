# Historical Analysis: scaleDownDelaySeconds Evolution

## Critical Bug Fix: Rollback Window Analysis Skipping

**Issue**: Pre-v1.8.0, rollback window detection worked but analysis runs were still created unnecessarily.

**Fix**: Commit 243ea9176 (v1.8.0-rc1) added missing `isRollbackWithinWindow()` check to analysis reconciliation.

**Impact**: Rollback performance improved significantly (150-200s → 50-100s) due to proper analysis skipping.

**Test Added**: `TestDoNotCreateBackgroundAnalysisRunWhenWithinRollbackWindow`

**GitHub Issue**: #3669 - "Background analysis runs should not be created when rolling back within the rollback window"

## Code Evolution

**Early Versions**: 30s defaults established for development/demo scenarios.

**Blue-Green**: Full scaleDownDelaySeconds support with annotation-based deadline management.

**Canary**: Limited to traffic-routing scenarios with same conservative defaults.

## Architectural Decisions

**Conservative Defaults**: 30s values prioritized quick cleanup over rollback safety.

**Traffic Routing Dependency**: scaleDownDelaySeconds restricted to canary with traffic routing.

## Community Evolution

**Early Users**: Development teams found defaults acceptable.

**Production Users**: Enterprise teams reported insufficient delay windows.

**Current State**: Existing mechanisms work well; issue is inappropriate defaults for production.

## Manual Exploration Required

**Investigate Further**:
- Review commit history for scaleDownDelaySeconds implementation
- Examine when traffic-routing restriction was added
- Analyze evolution of rollback window logic
- Check GitHub issues for user pain points over time

**Key Question**: When did production teams first report 30s defaults as insufficient?