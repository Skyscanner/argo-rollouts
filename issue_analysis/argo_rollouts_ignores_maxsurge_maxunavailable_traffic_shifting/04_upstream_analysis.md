# Upstream Analysis: maxSurge/maxUnavailable with Traffic Shifting

## Documented Issues

| Issue # | Title | Status | Key Insight |
|---------|-------|--------|-------------|
| #2239 | Traffic Routing and maxSurge/maxUnavailable | Open | Zach Aller explains intentional design decision |
| #1759 | improve basic canary and honor maxSurge | Closed | Added maxSurge to basic canary only |
| #3284 | Support maxSurge/maxUnavailable with traffic routing | Open | Community feature request |
| #3539 | maxSurge and maxUnavailable ignored | Open | User reports scaling issues |
| #3397 | maxSurge/maxUnavailable not respected | Open | Another scaling issue report |

## Community Context

**Maintainer Stance**: Traffic routing intentionally prioritizes traffic control over scaling limits.

**User Demand**: Multiple issues report "very rapid infrastructure scaling" problems.

**Design Conflict**: Traffic control vs pod scaling limits creates fundamental tension.

## Implementation Patterns

**Basic Canary**: Properly implements maxSurge via `maxReplicaCountAllowed = rolloutSpecReplica + maxSurge`.

**Traffic-Routed Canary**: No maxSurge/maxUnavailable logic, scales based on traffic weights only.

## Manual Exploration Required

**Investigate Further**:
- Review maintainer discussions in issue #2239
- Examine CNCF Slack threads on traffic routing design
- Compare with other progressive delivery tools
- Analyze user cost impact reports

**Key Question**: Is the design philosophy conflict resolvable, or is this a fundamental limitation?