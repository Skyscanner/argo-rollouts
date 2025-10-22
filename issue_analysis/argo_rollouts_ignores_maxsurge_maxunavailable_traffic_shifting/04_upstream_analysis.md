# Upstream Analysis: Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used

## Search Strategy
**Keywords used:** maxSurge, maxUnavailable, traffic routing, canary, surge limits, #2239, #3284, #3539, #3397
**Date range:** All history (searched git log and codebase references)
**Search method:** Git log grep, codebase references, CHANGELOG analysis, CNCF Slack discussion

## Similar Issues Found
| Issue/PR # | Title | Status | Relevance | Key Insights |
|------------|-------|--------|-----------|--------------|
| #2239 | Traffic Routing and maxSurge/maxUnavailable | Open/Design Discussion | **CRITICAL** - Explains intentional design decision | Zach Aller references this as explaining why maxSurge/maxUnavailable are not used |
| #1759 | fix!: improve basic canary approximation accuracy and honor maxSurge | Closed/Resolved | **HIGH** - Added maxSurge to basic canary | This PR fixed maxSurge for basic canary but did not address traffic-routed canary |
| #3284 | [Request] Support maxSurge/maxUnavailable with traffic routing | Open | **HIGH** - User request for feature | Indicates community demand for this functionality |
| #3539 | maxSurge and maxUnavailable ignored when using traffic routing | Open | **HIGH** - User issue report | Reports infrastructure scaling problems |
| #3397 | maxSurge/maxUnavailable not respected with traffic routing | Open | **HIGH** - User issue report | Another instance of the same problem |
| #1429 | fix: canary scaledown event could violate maxUnavailable | Closed/Resolved | **MEDIUM** - Related maxUnavailable fix | Another maxUnavailable fix that may not have addressed traffic routing |
| #3375 | add unit tests for maxSurge=0, replicas=1 | Closed/Resolved | **LOW** - Test improvements | Added test coverage for edge cases |

## Community Approaches
### Successful Patterns
- **Basic Canary maxSurge Implementation:** The `CalculateReplicaCountsForBasicCanary()` function properly implements maxSurge logic by calculating `maxReplicaCountAllowed = rolloutSpecReplica + maxSurge` and respecting these limits during scaling operations.

### Failed Approaches  
- **Traffic-Routed Canary:** No implementation of maxSurge/maxUnavailable limits in `CalculateReplicaCountsForTrafficRoutedCanary()`, leading to unbounded scaling potential.

### Community Workarounds
- **Rollback Behavior:** During rollbacks, traffic routing does provide some minimum availability protection by requesting max pods in stable set and gradually shifting traffic.

## Maintainer Preferences
Based on CNCF Slack discussion (Feb 2024):
- **Design Philosophy:** Traffic routing intentionally prioritizes traffic control over pod scaling limits
- **Feature Scope:** Traffic routing only supports `minPodsPerReplicaSet`, not maxSurge/maxUnavailable
- **Consistency:** Maintainers acknowledge the inconsistency but maintain it's by design
- **User Impact:** Despite design intentions, users report "very rapid infrastructure scaling" issues

## Ongoing Work
- **Multiple Open Issues:** #3284, #3539, #3397 all request this feature
- **No Active Implementation:** No pending PRs addressing this gap
- **Community Discussion:** Active CNCF Slack discussion indicates this remains a user pain point

## Recommendations for Contribution Strategy
- **Design Decision Challenge:** This appears to be an intentional design decision rather than an oversight
- **Community Demand:** Multiple issues suggest strong user demand despite maintainer stance
- **Implementation Path:** If pursuing, would need to convince maintainers to change design philosophy
- **Alternative Approaches:** Consider documenting workarounds or providing `minPodsPerReplicaSet` guidance
- **High Controversy:** This would require changing fundamental traffic routing design principles

### Validation and User Experience Issues

#### Missing Validation Warning
**Current State:** No validation warning when maxSurge/maxUnavailable are set with traffic routing.

**User Expectation:** Clear validation message stating these settings are ignored with traffic routing.

**Code Location:** `pkg/apis/rollouts/validation/validation.go` - `ValidateRolloutStrategyCanary()` function

**Impact:** Users may unknowingly set maxSurge/maxUnavailable expecting them to work, leading to unexpected scaling behavior.

#### Documentation Gaps
**Current Documentation:** Limited guidance on traffic routing scaling behavior differences.

**User Pain Point:** Lack of clear documentation about when and why maxSurge/maxUnavailable are ignored.

**Recommendation:** Add prominent warnings in:
- API documentation for maxSurge/maxUnavailable fields
- Traffic routing documentation
- Validation error messages