# Relevance Assessment: Simultaneous 100% Availability Requirement

## Issue Relevance: CRITICAL

**Impact**: Blocks large-scale deployments with autoscaling, PDBs, and spot instances.

## Documented Issues

| Issue # | Title | Evidence |
|---------|-------|----------|
| #4294 | BlueGreen Rollout Stuck in Progressing | HPA scaling conflicts |
| #3316 | Rollout stuck issue | dynamicStableScale conflicts |
| #3256 | Rollout stuck when changing replicas | Permanent stuck state |
| #3304 | BlueGreen Rollbacks get stuck | Rollback failures |

## Affected User Segments

- Large-scale deployments (100+ pods)
- Autoscaling users (HPA, KEDA, Karpenter)
- Spot instance deployments
- Services with Pod Disruption Budgets

## Technical Relevance

**Architecture Impact**: Requires shift from binary to percentage-based health checks.

**Kubernetes Integration**: Must respect MaxUnavailable (25% default) like native Deployments.

## Manual Exploration Required

**Investigate Further**:
- Review GitHub issues for specific user scenarios and reproduction cases
- Examine how Kubernetes Deployments handle availability with MaxUnavailable
- Analyze Flagger's compatibility with autoscaling systems
- Assess impact on enterprise deployment patterns

**Key Question**: What percentage of Argo Rollouts users are affected by this limitation?