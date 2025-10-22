# Upstream Summary: scaleDownDelaySeconds, rollback window, and dynamicStableScale

## High-priority upstream items

- **[#3669](https://github.com/argoproj/argo-rollouts/issues/3669)** — *Background Analysis runs when rolling back within Rollback Window*  
  - **Status:** Closed (fixed)  
  - **Fix:** [#3670](https://github.com/argoproj/argo-rollouts/pull/3670) — adds `isRollbackWithinWindow()` to skip background analysis during rollback  
  - **Introduced in:** v1.8.0  

- **[#3670](https://github.com/argoproj/argo-rollouts/pull/3670)** — *fix(analysis): Take RollbackWindow into account when Reconciling Analysis Runs*  
  - **Status:** Merged  
  - **Commit:** 243ea91767112a66ac5bc7b0cdefdd7b2173fc33  
  - **Introduced in:** v1.8.0  

- **[#3414](https://github.com/argoproj/argo-rollouts/issues/3414)** — *Rollout in degraded state when progressDeadlineSeconds < scaleDownDelaySeconds*  
  - **Status:** Closed  
  - **Fix:** [#3417](https://github.com/argoproj/argo-rollouts/pull/3417)  
  - **Introduced in:** v1.8.0+ (March 2024)  

- **[#3848](https://github.com/argoproj/argo-rollouts/issues/3848)** — *HPA scaling while in scale down delay window causes perpetual ‘progressing’*  
  - **Status:** Closed  
  - **Introduced in:** v1.8.1+ (mitigated in later patches)  

- **[#4337](https://github.com/argoproj/argo-rollouts/pull/4337)** — *Patch DynamicStableScale to respect scaleDownDelaySeconds at 100% traffic shift*  
  - **Status:** Closed (abandoned, not merged)  
  - **Introduced in:** N/A  

---

## Observations

1. Rollback-window handling for background analysis was fixed in **v1.8.0**.  
2. Interaction between `progressDeadlineSeconds` and `scaleDownDelaySeconds` fixed in **v1.8.0+**.  
3. DynamicStableScale handling of delay semantics remains incomplete — PR #4337 never merged.  
4. HPA + scaleDownDelay edge cases resolved partially by v1.8.x patches.

---

## Recommended follow-ups

- Ensure canary `scaleDownDelaySeconds` mirrors blue-green rollback behavior with routing (VirtualService/DestinationRule).  
- Add tests for:
  - Rollback within window (no AnalysisRun)
  - Delay and abort delay under `dynamicStableScale`
  - HPA and progress deadline interplay

---

## Summary table

| ID | Type | Title | Status | Introduced in | Notes |
|----|------|--------|---------|----------------|--------|
| #3669 | Issue | Background Analysis on rollback | Fixed | v1.8.0 | RollbackWindow respected |
| #3670 | PR | Analysis skip on rollback | Merged | v1.8.0 | Commit 243ea917 |
| #3414 | Issue | Degraded state when progressDeadline < scaleDelay | Fixed | v1.8.0+ | Controller timeout fix |
| #3417 | PR | Fix progressDeadline + delay interaction | Merged | v1.8.0+ | Improves timeout logic |
| #3848 | Issue | HPA stuck progressing | Fixed | v1.8.1+ | Mitigated later |
| #4337 | PR | DynamicStableScale delay patch | Abandoned | N/A | Not merged |

---

## Key takeaways

- v1.8.0 is the baseline for rollback-window correctness.  
- scaleDownDelaySeconds and HPA interactions stabilised in later v1.8.x releases.  
- DynamicStableScale edge cases still exist.  


Reference: see Issue #557 for the upstream discussion that motivated this analysis.
