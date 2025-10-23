# Contribution Epic: scaleDownDelaySeconds & Fast Rollbacks

## Epic Overview

**Problem**: 30-second scaleDownDelaySeconds defaults insufficient for production rollback scenarios.

**Solution Focus**: Chart value optimization rather than major code changes.

## Exploration Directions

### Chart Value Optimization
- Update Helm chart defaults for production environments (300-600s delays)
- Provide environment-specific configuration recommendations
- Document rollback window best practices

### Optional Upstream Enhancements
- Optimize analysis run termination speed (current: 20-30+ seconds)
- Improve canary pause step skipping during rollbacks
- Enhance resource cleanup during extended delays

### Configuration Strategy
- Provide sensible defaults for dev/staging/production
- Add validation for rollback window configurations
- Create production-ready configuration examples

## Open Questions

- What delay values work for different team sizes?
- How do rollback windows interact with analysis runs?
- What are performance implications of extended delays?
- How to balance rollback speed with resource efficiency?

## Success Criteria

1. Enable reliable rollbacks through appropriate configuration
2. Provide clear production guidance and examples
3. Maintain compatibility with existing mechanisms
4. Support environment-specific needs

## Manual Exploration Required

**Investigate Further**:
- Test different delay values in staging environments
- Analyze rollback window behavior with various revision counts
- Measure analysis run termination performance
- Document production configuration patterns

**Key Question**: What configuration provides optimal rollback reliability for different deployment scales?

