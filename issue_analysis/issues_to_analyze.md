	
# Argo Rollouts Issues Analysis

| Issue Name | Description | Supporting Evidence / Useful Links | Notes |
|------------|-------------|-----------------------------------|-------|
| Support scaleDownDelaySeconds & fast rollbacks with canary strategy | Currently argo-rollouts only supports fast-track rollback when a canary deployment is in progress. The enhancement requests adding support for keeping the previous version around for scaleDownDelaySeconds (similar to blue-green strategy) to allow fast rollback for canary deployments in case metric checks don't catch regressions. | [GitHub Issue #557](https://github.com/argoproj/argo-rollouts/issues/557) | Blue-green strategy already supports this feature with scaleDownDelaySeconds. This would bring feature parity between deployment strategies and improve rollback capabilities for canary deployments. |
| Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used | When traffic shifting is used, argo-rollouts ignores the maxSurge and maxUnavailable settings, which can impact cluster autoscaling by putting additional pressure on Karpenter to binpack or provide new nodes. | [Support scaleDownDelaySeconds & fast rollbacks with canary strategy](https://github.com/argoproj/argo-rollouts/issues/557) | Can have impact on cluster autoscaling putting additional pressure on karpenter to binpack or provide new nodes. Combined with flaky health checks and aggressive autoscaling that larger services might be unwittingly using, this can lead to long deployment times per cluster. |
| Argo-rollouts waits for stable RS to be stable before scaling it down | When used on a large scale with a cluster autoscaler that can disrupt nodes and evict pods, the canary RS stays scaled-up for a while until the stable RS is fully scaled. This makes sense if the controller scaled down the stable RS during the rollout (using dynamicStableScale), but it doesn't make sense if it didn't. | [GitHub PR #3899](https://github.com/argoproj/argo-rollouts/pull/3899) | This behavior can cause resource inefficiency and increased costs when the stable RS wasn't scaled down during rollout but the canary RS remains scaled up unnecessarily. |

argo-rollouts

argo-rollouts waits for stable RS to be stable before scaling it down

https://github.com/argoproj/argo-rollouts/pull/3899 

 When used on a large scale with a cluster autoscaler that can disrupt nodes and evict pods, the canary RS stays scaled-up for a while until the stable RS is fully scaled. This makes sense if the controller scaled down the stable RS during the rollout (using dynamicStableScale), but it doesn't make sense if it didn't.