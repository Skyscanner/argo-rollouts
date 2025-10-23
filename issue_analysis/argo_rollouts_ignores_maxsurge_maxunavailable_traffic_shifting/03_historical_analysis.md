# Historical Analysis: maxSurge/maxUnavailable Evolution

## Key Commits

| Commit | Date | Message | Impact |
|--------|------|---------|--------|
| c1353e4a44 | 2021-09-21 | feat: support dynamic scaling of stable ReplicaSet (#1430) | Introduced traffic-routed canary without maxSurge support |
| 0f93f9b82e | 2022-01-14 | fix!: improve basic canary and honor maxSurge (#1759) | Added maxSurge to basic canary only |

## Design Philosophy

**Traffic Routing**: Intentionally prioritizes traffic control over pod scaling limits (maintainer stance from issue #2239).

**Basic Canary**: Respects Kubernetes deployment scaling controls.

## Community Issues

**Multiple Requests**: Issues #3284, #3539, #3397 all request maxSurge/maxUnavailable support despite maintainer design.

**User Impact**: Reports of "very rapid infrastructure scaling" causing cost and resource issues.

## Current Workarounds

**MinPodsPerReplicaSet**: Provides minimum pod floor but doesn't control maximum surge.

**Manual Steps**: Users create many small canary steps to control scaling speed.

## Manual Exploration Required

**Investigate Further**:
- Review commit c1353e4a44 that introduced traffic routing without scaling limits
- Examine CNCF Slack discussions on design philosophy
- Analyze evolution of basic vs traffic-routed canary implementations
- Review user issue reports for scaling problems

**Key Question**: When did the design philosophy divergence between canary types occur?