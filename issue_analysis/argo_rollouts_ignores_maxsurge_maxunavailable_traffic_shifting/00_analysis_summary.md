# Argo Rollouts: maxSurge/maxUnavailable Ignored with Traffic Shifting

## Issue Summary

**Problem**: `maxSurge` and `maxUnavailable` settings ignored in traffic-routed canary deployments.

**Root Cause**: Intentional design decision prioritizing traffic control over scaling limits.

## Documented Issues

| Issue # | Title | Status |
|---------|-------|--------|
| #2239 | Traffic Routing and maxSurge/maxUnavailable | Open/Design Discussion |
| #3284 | [Request] Support maxSurge/maxUnavailable with traffic routing | Open |
| #3539 | maxSurge and maxUnavailable ignored when using traffic routing | Open |
| #3397 | maxSurge/maxUnavailable not respected with traffic routing | Open |

## Technical Details

**Code Location**: `utils/replicaset/canary.go`

**Basic Canary**: `CalculateReplicaCountsForBasicCanary()` respects maxSurge/maxUnavailable.

**Traffic-Routed Canary**: `CalculateReplicaCountsForTrafficRoutedCanary()` ignores these settings.

**Maintainer Rationale** (from CNCF Slack): Traffic routing prioritizes traffic control over pod scaling limits.

## Functionality Matrix

| Feature | Basic Canary | Traffic-Routed Canary |
|---------|-------------|----------------------|
| maxSurge/maxUnavailable | ✅ Supported | ❌ Ignored by design |
| Traffic Weight Control | ❌ N/A | ✅ Supported |
| scaleDownDelaySeconds | ❌ Validation error | ✅ Supported |

## Current Workarounds

**MinPodsPerReplicaSet**: Provides minimum pod floor but doesn't control maximum surge.

**Manual Canary Steps**: Users create many small steps to control scaling speed.

## Design Philosophy Conflict

**Maintainer View**: Traffic control takes precedence over scaling limits.

**User View**: Need scaling controls to prevent "very rapid infrastructure scaling".

## Manual Exploration Required

**Investigate Further**:
- Review GitHub issue #2239 for maintainer design discussion
- Examine CNCF Slack discussions on traffic routing philosophy
- Test behavior differences between basic and traffic-routed canaries
- Analyze impact of dynamicStableScale on scaling limits

**Key Question**: Should traffic routing support both traffic control AND scaling limits?