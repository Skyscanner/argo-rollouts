# Argo Rollouts: Abort Logic, scaleDownDelaySeconds, Fast Rollbacks, and Sync Behavior (v1.8.3)

Comprehensive technical summary for **traffic-routed canary** (Istio) environments with **inline step AnalysisRuns**, including precise comparison to blue-green strategy.

---

## Current Configuration

| Setting | In Use | Default | Effect |
|----------|---------|----------|--------|
| **`dynamicStableScale`** | ✅ | `false` | Stable RS scales dynamically (e.g., via HPA). Reduces cost but can delay rollback because stable RS must scale up. |
| **`scaleDownDelaySeconds`** | ✅ | **30 s** | Keeps old RS alive after promotion for rollback or mesh convergence. Default favors cost efficiency, not rollback safety. |
| **`abortScaleDownDelaySeconds`** | ❌ | **30 s** | Delay before scaling down canary RS after abort. `0` disables cleanup (pods remain indefinitely). |
| **`progressDeadlineAbort`** | ❌ (`false`) | `false` | If true, rollout aborts automatically on progress timeout; if false, only sets condition and waits for manual action. |
| **`rollbackWindow.revisions`** | ✅ (`3`) | `nil` | Determines how many historical RSs are eligible for reuse. Timestamp-based, not revision-number–based. |

**Code evidence**  
- [utils/defaults/defaults.go#L31](https://github.com/argoproj/argo-rollouts/blob/master/utils/defaults/defaults.go#L31) – default delay constants.  
- [rollout/canary.go#L179-L230](https://github.com/argoproj/argo-rollouts/blob/master/rollout/canary.go#L179-L230) – `HasScaleDownDeadline` logic in canary scaling.  
- [rollout/sync.go#L902-L996](https://github.com/argoproj/argo-rollouts/blob/master/rollout/sync.go#L902-L996) – rollback-window fast-track detection.  
- [pkg/apis/rollouts/v1alpha1/types.go#L1215-L1217](https://github.com/argoproj/argo-rollouts/blob/master/pkg/apis/rollouts/v1alpha1/types.go#L1215-L1217) – `RollbackWindowSpec`.  

---

## Abort Conditions

| Condition | Auto-Abort? | Notes |
|------------|-------------|-------|
| AnalysisRun fails/errors | ✅ | Immediate abort. |
| Progress deadline exceeded + `progressDeadlineAbort: true` | ✅ | Abort triggered by timeout. |
| `progressDeadlineAbort: false` (current) | ❌ | Marks condition; rollout stays paused. |
| Manual abort (CLI/API) | ✅ | Always possible. |
| Pod/RS failure | ❌ | Marks condition only. |
| Traffic routing errors | ❌ | Logged, not aborting. |
| Omitted/skipped AnalysisRun | ❌ | Controller continues. |

Sources: [rollout/sync.go#L345-L360](https://github.com/argoproj/argo-rollouts/blob/master/rollout/sync.go#L345-L360), [rollout/analysis.go#L40-L55](https://github.com/argoproj/argo-rollouts/blob/master/rollout/analysis.go#L40-L55)

---

## scaleDownDelaySeconds and abortScaleDownDelaySeconds

**Purpose**
Hold the previous RS for a defined period after promotion or abort to allow rollback and mesh convergence.

**Behavior**
- `scaleDownDelaySeconds` delays cleanup after promotion.  
- `abortScaleDownDelaySeconds` controls cleanup after abort.  
- 30 s default minimizes cost but is insufficient for real pipelines.  
  **Recommendation:** set 300–600 s (5–10 min).  
- With `dynamicStableScale: true`, delay may be skipped to avoid double scaling.  
- Setting `abortScaleDownDelaySeconds: 0` keeps pods indefinitely for debugging.  
- Traffic routers (e.g., Istio) reroute instantly; delay only affects pod cleanup.

---

## Rollback Windows and Revision Reuse

**Mechanism**
- `rollbackWindow.revisions` limits how many RSs are eligible for reuse.
- Controller compares RS creation timestamps: if within window, rollback is **fast-tracked**.

**Behavior**
- Inside window → all steps completed instantly, inline analyses cancelled.  
- Outside window → normal rollout progression.  
- Pod hashes may match but new RSs still created because revisions increment on reapply.  
- Fast-track logic lives in `shouldFullPromote()` in `rollout/sync.go`.

**Production pattern**
- `rollbackWindow.revisions: 3` is typical; ensures 1–2 min rollback times for near-history RSs.  
- Extending to 5 increases safety with minimal resource overhead.

---

## AnalysisRun Lifecycle and Skip Logic (Traffic-Based Canary)

**Inline Step AnalysisRuns (your configuration)**  
Applies only to canary step-level analysis; pre/post-promotion analyses belong to blue-green.

**Creation and reuse**
- `needsNewAnalysisRun()` creates runs only when none exist, or when the prior run was inconclusive/invalid.  
- AR names include rollout name + pod hash + revision + step index.  
- New revision → new AR name → reuse fails → new AR created.  

**Skip conditions**
1. Rollback within window → all inline ARs skipped/cancelled (fast-track).  
2. Rollout paused/aborted → active ARs cancelled.  
3. New RS not saturated → AR creation deferred until ready.  
4. Pod hash unchanged but revision bumped → controller still creates new AR (different name).  
5. Pre/post promotion skip (`skipPrePromotionAnalysisRun`, `skipPostPromotionAnalysisRun`) apply **only to blue-green**, not to canary.

Retention via history limits controls only garbage collection, not creation logic.

---

## Fast Rollback Reuse Parity (Blue-Green vs. Canary)

| Capability | Blue-Green | Canary (Traffic-Based) | Explanation |
|-------------|-------------|-------------------------|-------------|
| Keep old RS alive via `scaleDownDelaySeconds` | ✅ | ✅ | Both delay RS cleanup for rollback. |
| Trigger fast-promotion (skip pause) on existing RS | ✅ | ❌ | Blue-green skips pause when RS has a scale-down deadline. Canary only delays cleanup, no automatic fast-promotion. |
| Fast rollback via rollback window | ✅ | ✅ | Both support timestamp-based fast-track logic. |
| Use of `HasScaleDownDeadline` | ✅ (promotion + cleanup) | ✅ (cleanup only) | Canary checks it to delay scale-down, not to promote. |
| Typical delay values | 300–1800 s | 30–300 s | Blue-green rollback window much longer for safety. |

**Interpretation:**  
- Blue-green uses the `HasScaleDownDeadline` annotation to detect a previously active RS and **skip the promotion pause**, instantly restoring traffic (true fast rollback).  
- Canary uses the same annotation **only** to keep RS pods alive; fast rollback still depends on `rollbackWindow.revisions`.  
- Hence “partial” fast rollback reuse in canary.

---

## Typical `scaleDownDelaySeconds` Values

| Strategy | Default | Common Range | Reason |
|-----------|----------|---------------|--------|
| **Blue-Green** | 30 s | **300–1800 s (5–30 min)** | Keeps previous RS warm for manual/automated rollback after promotion; rollback safety buffer. |
| **Canary (Traffic-Based)** | 30 s | **30–300 s (0.5–5 min)** | Covers mesh routing convergence and short rollback windows during progressive delivery. |

Blue-green’s window is intentionally an order of magnitude longer: its rollback is post-promotion, while canary rollback occurs incrementally mid-progress.

---

## Recommendations

1. Increase `scaleDownDelaySeconds` to **≥300 s** for practical rollback reuse.  
2. Maintain `rollbackWindow.revisions` at 3–5 for efficient reuse without clutter.  
3. Use `abortScaleDownDelaySeconds: 0` for debugging; 30–120 s for cleanup.  
4. Enable `progressDeadlineAbort` when automatic aborting is desired.  
5. Disable `dynamicStableScale` for faster rollback; enable for HPA efficiency.  
6. Treat `HasScaleDownDeadline` as cleanup control, not promotion logic, in canary.  
7. Consider extending fast-promotion parity to canary (feature gap).

---

## Summary Table

| Mechanism | Purpose | Default | Recommended | Notes |
|------------|----------|----------|--------------|--------|
| `scaleDownDelaySeconds` | Retain old RS for rollback | 30 s | 300–600 s | Longer for realistic rollback windows |
| `abortScaleDownDelaySeconds` | Delay cleanup post-abort | 30 s | 0–120 s | 0 keeps pods indefinitely |
| `rollbackWindow.revisions` | Fast rollback reuse window | 3 | 3–5 | Timestamp-based eligibility |
| `progressDeadlineAbort` | Auto-abort stalled rollout | false | true | Optional safety |
| `dynamicStableScale` | Stable RS autoscaling | false | true/false | Choose based on rollback latency vs. cost |

---

## Behavior During Argo CD Sync Termination

- Sync termination does **not** stop controller reconciliation.  
- Rollout continues based on persisted spec/state.  
- HPA continues adjusting total replicas; controller redistributes between stable/canary.  
- Missing routing/analysis resources may delay AR creation until resync.  
- If rollback target RS is within window → fast-track rollback; inline analyses skipped.

---

**Final Takeaway**

Blue-green and canary both delay RS cleanup for rollback.  
Only blue-green, however, uses `HasScaleDownDeadline` to *skip pause* and promote immediately — canary does not.  
Therefore, canary’s rollback reuse is **partial**: it preserves RS state for rollback but relies entirely on the **rollback window** for fast-tracking.  
Blue-green retains RSs much longer (5–30 min) to enable post-promotion reversions; canary windows are shorter (30–300 s) for routing stabilization.
