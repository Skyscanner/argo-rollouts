# Argo Rollouts: Abort Logic, scaleDownDelaySeconds, Fast Rollbacks, and Analysis Behavior (v1.8.3)

Concise, code-verified summary for **traffic-routed canary (Istio)** deployments with **inline step AnalysisRuns**.

---

## Core Settings and References

| Setting | Default | Behavior | Code Reference |
|----------|----------|-----------|----------------|
| `dynamicStableScale` | `false` | Stable RS scales dynamically (e.g., via HPA). Saves cost but slows rollback as stable RS must scale back up. | [rollout/canary.go#L179-L230](https://github.com/argoproj/argo-rollouts/blob/master/rollout/canary.go#L179-L230) |
| `scaleDownDelaySeconds` | 30 s | Keeps old RS alive after promotion for rollback or routing stability. Extend to 300–600 s for practical rollback. | [utils/defaults/defaults.go#L182-L200](https://github.com/argoproj/argo-rollouts/blob/master/utils/defaults/defaults.go#L182-L200) |
| `abortScaleDownDelaySeconds` | 30 s | Delay cleanup after abort. `0` disables cleanup (pods remain indefinitely). | [utils/defaults/defaults.go#L213-L239](https://github.com/argoproj/argo-rollouts/blob/master/utils/defaults/defaults.go#L213-L239) |
| `progressDeadlineAbort` | `false` | Auto-abort when rollout exceeds `progressDeadlineSeconds`. | [rollout/sync.go#L345-L360](https://github.com/argoproj/argo-rollouts/blob/master/rollout/sync.go#L345-L360) |
| `rollbackWindow.revisions` | 3 | How many past RSs are eligible for fast-track rollback (timestamp-based). | [rollout/sync.go#L902-L996](https://github.com/argoproj/argo-rollouts/blob/master/rollout/sync.go#L902-L996) |

---

## Abort Logic

| Trigger | Auto-Abort? | Description |
|----------|-------------|-------------|
| AnalysisRun fails/errors | ✅ | Immediate abort. |
| Progress deadline exceeded + `progressDeadlineAbort: true` | ✅ | Timeout abort. |
| Manual abort (CLI/API) | ✅ | Always supported. |
| Pod/RS failure | ❌ | Flags condition only. |
| Traffic routing errors | ❌ | Logged only. |

---

## Delay and Cleanup Behavior

- `scaleDownDelaySeconds` delays cleanup of old RS after promotion.  
- `abortScaleDownDelaySeconds` applies after abort.  
- Default (30 s) is too short for production.  
- **Recommended:** 300–600 s for safe rollback reuse.  
  
Note on the 30s default:

- The 30s default is defined in `utils/defaults/defaults.go` as `DefaultScaleDownDelaySeconds = 30` and
  `DefaultAbortScaleDownDelaySeconds = 30`. It originates as a minimum window to allow
  iptables/service-provider propagation after switching service selectors (see comments
  in `pkg/apis/rollouts/v1alpha1/experiment_types.go` and `generated.proto`). In practice
  30s is a historical minimum and not sufficient for operator-driven rollback in many
  production environments. See `examples/rollout-scaleDownDelay-5m.yaml` for a recommended
  starting point.
- `dynamicStableScale:true` skips delay to prevent dual scaling.  
- Routing systems (Istio/ALB) reroute instantly; delay affects pod cleanup only.  
- Logic verified in [rollout/canary.go#L179-L230](https://github.com/argoproj/argo-rollouts/blob/master/rollout/canary.go#L179-L230).

---

## ReplicaSet Reuse Logic

**Source:** [utils/replicaset/replicaset.go](https://github.com/argoproj/argo-rollouts/blob/master/utils/replicaset/replicaset.go)

- Controller reuses existing RS when **pod-template-hash** matches desired spec.  
- Matching RS is scaled up per step/weight.  
- If RS was scaled to 0 (after delay expiry), it is reused and scaled back up.  
- Only deleted RSs (GC, history limit) trigger new creation.  
- Revisions increment, but RS identity is hash-based.

✅ **Conclusion:** Reapply same pod spec → existing RS reused (no new RS created).

---

## Rollback Window Fast-Track

**Source:** [rollout/sync.go#L902-L996](https://github.com/argoproj/argo-rollouts/blob/master/rollout/sync.go#L902-L996)

- `rollbackWindow.revisions` defines eligible RSs for reuse.  
- Controller checks if target RS predates stable RS but is within the window.  
- Inside window:  
  - Steps marked complete.  
  - Inline analyses skipped.  
  - Rollback executes immediately.  
- Outside window → normal rollout progression resumes.

---

## AnalysisRun Creation and Reuse Logic

**Sources:**  
- [rollout/analysis.go](https://github.com/argoproj/argo-rollouts/blob/master/rollout/analysis.go) (`needsNewAnalysisRun()`, `reconcileStepBasedAnalysisRun()`)  
- [analysis/util/](https://github.com/argoproj/argo-rollouts/tree/master/analysis/util) (deterministic name generation)

**Deterministic Naming Pattern:**  
`<rollout-name>-<pod-template-hash>-<revision>[-stepX]`

**Observed Behavior:**
- Controller links AnalysisRuns to both pod hash and revision.  
- When **revision changes**, even if **pod-template-hash** is identical, a new deterministic name is computed.  
- Controller finds no existing AR with that name and creates a new one.  
- Successful historical ARs for the same pod hash are ignored because their names include old revision.  
- Only `rollbackWindow` fast-tracks skip new AnalysisRun creation.  

**Result (explicit):**  
➡️ If **revision is new**, **AnalysisRuns are repeated** even when pod spec and pod-template-hash are unchanged.  
This happens because name mismatch triggers new run creation in `needsNewAnalysisRun()`.  
✅ **Exception:** if rollback target RS is within rollback window, analyses are skipped entirely.

---

## Canary vs. Blue-Green Fast-Rollback Comparison

| Capability | Blue-Green | Canary (Traffic-Based) | Notes |
|-------------|-------------|-------------------------|-------|
| Retain old RS (`scaleDownDelaySeconds`) | ✅ | ✅ | Both hold RS for rollback. |
| Auto-promote via `HasScaleDownDeadline` | ✅ | ❌ | Blue-Green auto-promotes; canary only delays cleanup. |
| Fast rollback via rollback window | ✅ | ✅ | Timestamp-based fast-track. |
| RS reuse on same pod hash | ✅ | ✅ | Controller reuses and scales matching RS. |
| Analyses reused on same hash + new revision | ❌ | ❌ | Revision bump always causes new AnalysisRuns. |
| Typical delay ranges | 300–1800 s | 30–300 s | Blue-Green keeps longer rollback safety window. |

---

## Typical scaleDownDelaySeconds Values

| Strategy | Default | Common Range | Purpose |
|-----------|----------|---------------|----------|
| **Blue-Green** | 30 s | **300–1800 s (5–30 min)** | Keeps prior RS alive for post-promotion rollback. |
| **Canary (Traffic-Based)** | 30 s | **30–300 s (0.5–5 min)** | Allows service mesh to converge safely. |

---

## Recommendations

1. Set `scaleDownDelaySeconds` ≥ 300 s for effective rollback reuse.  
2. Keep `rollbackWindow.revisions` = 3–5 for safety.  
3. Use `abortScaleDownDelaySeconds: 0` for debugging.  
4. Enable `progressDeadlineAbort` for autonomous rollback safety.  
5. Disable `dynamicStableScale` for faster rollbacks.  
6. Understand that **new revision always triggers new AnalysisRun**, even for identical specs; only rollback-window fast-tracks skip analysis creation.

## Upstream evidence and discussion (selected)

- Issue #19 discussion (historical): iptables/service-propagation rationale mentioned — https://github.com/argoproj/argo-rollouts/issues/19#issuecomment-476329960
- Issue/PR #3669 / #3670: Fix and PR that added rollback-window awareness to AnalysisRun reconciliation — https://github.com/argoproj/argo-rollouts/issues/3669 and https://github.com/argoproj/argo-rollouts/pull/3670 (commit 243ea917)
- Issue #3414 and PR #3417: progressDeadlineSeconds vs scaleDownDelaySeconds interaction — https://github.com/argoproj/argo-rollouts/issues/3414 and https://github.com/argoproj/argo-rollouts/pull/3417
- Issue #3848: HPA + scaleDownDelaySeconds interaction and mitigations — https://github.com/argoproj/argo-rollouts/issues/3848
- PR #4337: DynamicStableScale patch to respect scaleDownDelaySeconds at 100% traffic shift (not merged) — https://github.com/argoproj/argo-rollouts/pull/4337
- Related enhancement request: Issue #557 — support scaleDownDelaySeconds & fast rollbacks for Canary strategy — https://github.com/argoproj/argo-rollouts/issues/557
- Notes on Abort delay default (`abortScaleDownDelaySeconds` is 30s): see proto/type docs — `pkg/apis/rollouts/v1alpha1/generated.proto` and `pkg/apis/rollouts/v1alpha1/experiment_types.go`


Reference: see Issue #557 for the upstream discussion and clarifying comments around fast rollback to previously validated revisions.

---

## Behavior During Argo CD Sync Termination

- Terminating sync does not stop rollout reconciliation.  
- Controller continues scaling, weight distribution, and traffic management.  
- HPA continues operating.  
- Missing routing or analysis resources delay analysis creation until re-sync.  
- If RS with same pod hash exists → reused and scaled up (no new RS).  
- If revision changes (new sync/commit) → AnalysisRuns are recreated even if pod spec is identical.

---

## What specifically landed in upstream v1.8.3

- commit 243ea91767112a66ac5bc7b0cdefdd7b2173fc33 (PR #3670) — AnalysisRun rollback-window fix
  - Summary: Ensures Background AnalysisRuns are not launched when a rollout performs a fast rollback to an active ReplicaSet that is within the `rollbackWindow`.
  - Files/areas touched: analysis reconciliation (`rollout/analysis.go`) and tests.

- commit 3db9784288537e6c294e7baa1d18948a0d8595de (PR #4221) — Restarter + reconciliation safety
  - Summary: Pod restarter now returns number of restarted pods and the canary reconciliation short-circuits when restarts occurred so availability counts used for scale decisions are accurate; prevents scaling down in the same reconciliation that would risk downtime.
  - Files changed: `rollout/restart.go`, `rollout/canary.go`, `rollout/bluegreen.go`.

- commit aa6d28781d2a462b9c9e72f74dbde80447045dea (PR #4299) — abort scenario when canary/stable service not present
  - Summary: Fixes an abort/reconcile edge-case when rollout's canary/stable services are not provided; reduces spurious aborts when traffic routing config is partially absent.
  - Files changed: `rollout/trafficrouting/istio/istio.go`.

- commit 406f6bfb6a9e9a53361c83131ef0f399f40b00f8 (PR #4055) — VirtualService patching robustness
  - Summary: Patches VirtualService logic to work when only one named route exists — reduces regressions/downtime during traffic shifts.
  - Files changed: `rollout/trafficrouting/istio/istio.go`.

Note: These v1.8.3 patches improve safety around scale-down/traffic shifts and AnalysisRuns for fast rollbacks, but do not change the deterministic AnalysisRun naming behavior (AnalysisRun names are still revision-bound and a new revision normally triggers a new AnalysisRun unless rollback-window fast-path applies).

---

## Final Summary

- **ReplicaSets** reuse based on **pod-template-hash**, not revision.  
- **AnalysisRuns** are revision-bound: new revision → new AR, even with same pod spec.  
- **Rollback window** is the only condition that suppresses new AnalysisRun creation.  
- Blue-Green uses `HasScaleDownDeadline` for immediate promotion; Canary uses it only for cleanup.  
- Default 30 s windows are too short; increase to 300–600 s for real rollback capability.
