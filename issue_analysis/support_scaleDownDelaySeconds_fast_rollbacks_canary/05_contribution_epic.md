# Contribution Epic: Support scaleDownDelaySeconds & fast rollbacks for Canary strategy (focused)


Goal

- Enable robust fast rollback behavior for traffic-routed Canary deployments by using existing `scaleDownDelaySeconds` / `scale-down-deadline` annotations (Blue-Green parity for traffic-routed canaries only).
- Keep changes minimal, testable, and limited to traffic-routed canaries (no work for basic canary or dynamicStableScale).

Constraints / assumptions

- Only traffic-routed canaries are in scope (canaries that use `TrafficRouting` such as Istio, SMI, ALB, etc.).
- We will NOT change or depend on `dynamicStableScale`.
- We will NOT touch validation for basic canary; existing validation can remain as-is.
- Keep behavioral changes small and guarded by unit and e2e tests.

High-level design summary

1. Allow `scaleDownDelaySeconds` to be set on basic canary (validation change).
2. When a canary promotion occurs (weight reaches desired promotion or final step), annotate the previous (old) ReplicaSet(s) with a `scale-down-deadline` (the existing annotation and helpers are already present) so the old RS is retained for the configured duration.
3. Extend the canary reconciliation fast-rollback logic so that when rolling back to a ReplicaSet that: (a) is within the `scale-down-deadline` OR (b) has a live `scale-down-deadline` annotation OR (c) is within the configured `rollbackWindow`, then the controller will: skip step-based AnalysisRuns, mark steps as complete, and fast-track the rollback (same intent as blue-green fast rollback behavior).
4. Ensure `scaleDownDelaySeconds` semantics (abort vs normal flow) are respected using existing `addScaleDownDelay`/`HasScaleDownDeadline` helpers.
5. Provide tests and docs. Recommend a production default recommendation (e.g., 300s) in docs, but do not change code defaults automatically.

Deliverables (PR-sized tasks)

<!-- Tasks A and B intentionally omitted: out of scope (we only handle traffic-routed canaries). -->

Task C — Fast rollback semantics for Canary using the scale-down-deadline (breakdown)

We split Task C into small tickets (C1..C6) that are individually reviewable and testable.

C1 — Add predicate helpers and unit tests
- What: Add small helper funcs that encapsulate "is this RS eligible for scale-down-deadline fast-rollback?" and unit tests.
- Files: `rollout/replicaset_helpers.go` (new small file or add to `rollout/canary.go`), tests in `rollout/replicaset_helpers_test.go`
- Details:
  - Implement `IsReplicaSetWithinScaleDownDeadline(rs *appsv1.ReplicaSet) bool` and `IsReplicaSetEligibleForFastRollback(ro *v1alpha1.Rollout, rs *appsv1.ReplicaSet) bool`.
  - Cover edge cases: missing annotation, expired annotation, annotation present but replicas=0.
- Acceptance criteria: unit tests for helper functions pass.
- Estimate: 3

C2 — Skip background AnalysisRuns when target RS eligible
- What: Modify `rollout/analysis.go` reconcile logic to treat eligible RSs as conditions to skip background AnalysisRuns (add the helper check). Add unit tests.
- Files: `rollout/analysis.go`, `analysis/*_test.go`
- Acceptance criteria: when `IsReplicaSetEligibleForFastRollback()` is true, no background AnalysisRun is created; existing ARs are canceled appropriately.
- Estimate: 5

C3 — Skip step AnalysisRuns + mark steps complete for fast rollback path
- What: When a rollback to an eligible RS occurs during step reconciliation, mark steps as completed (advance `CurrentStepIndex`) and ensure step-specific AnalysisRuns are not created.
- Files: `rollout/canary.go` (`completedCurrentCanaryStep`, `reconcileCanaryPause`, `syncRolloutStatusCanary`), `rollout/sync.go` where step skipping decisions are made.
- Acceptance criteria: step AnalysisRuns not created; `CurrentStepIndex` is advanced to final for fast rollback; unit tests show rollback completes without AnalysisRun creation.
- Estimate: 8

C4 — Fast rollback end-to-end unit tests (rollouts)
- What: Add integration-style unit tests simulating a Rollout that triggers rollback to an eligible RS and assert timings, no AnalysisRuns created, and status transitions.
- Files: `rollout/canary_test.go` (new cases)
- Acceptance criteria: tests assert that rollback path uses fast-flow and controller sets `CurrentStepIndex` to end and `PromoteFull` as needed.
- Estimate: 5

C5 — Avoid regressions: ensure pause/abort interplay preserved
- What: Add tests and small guards to avoid skipping AnalysisRuns when pause/abort/ControllerPause conditions should prevent fast rollback.
- Files: `rollout/analysis.go`, `rollout/canary.go`, tests
- Acceptance criteria: controller still respects `pauseContext.IsAborted()` and any abort causes cancel/cleanup behavior, not fast rollback.
- Estimate: 3

C6 — Integration/optional e2e manifest and guide
- What: Provide a minimal e2e manifest and runbook steps to reproduce fast rollback in a local Kind/minikube or CI with an extended `scaleDownDelaySeconds` (e.g., 600s) to validate behavior end-to-end.
- Files: `test/fixtures/canary-fast-rollback.yaml`, `issue_analysis/.../howto_fast_rollback.md`
- Acceptance criteria: documented steps for local validation; optional CI job can be added later.
- Estimate: 3

Task D — AnalysisRun handling: reuse vs skip decisions (breakdown)

Split D into small tasks D1..D4.

D1 — Audit `needsNewAnalysisRun()` call-sites and decision graph
- What: Produce a small PR that documents all call sites where `needsNewAnalysisRun()` and AR creation logic is invoked and add unit tests skeletons for the decision branches.
- Files: `rollout/analysis.go`, test scaffolding
- Acceptance criteria: PR documents call sites and adds test skeletons; no behavior change.
- Estimate: 2

D2 — Implement conservative skip rule for step & background AR creation
- What: Modify `needsNewAnalysisRun()` to early-return false (skip creation) when `IsReplicaSetEligibleForFastRollback()` is true and rollback path is detected. Keep existing name generation unchanged.
- Files: `rollout/analysis.go`
- Acceptance criteria: When eligible, no new AR is created; unit tests validate both eligible and non-eligible cases.
- Estimate: 5

D3 — Ensure historic ARs are not erroneously reused: cancellation vs reuse policy
- What: Add logic so that when skipping AR creation we do not accidentally cancel ongoing valid ARs of a different purpose; adjust cancel semantics only for ARs tied to the target RS.
- Files: `rollout/analysis.go`, tests
- Acceptance criteria: Only ARs associated with the rolled back steps are canceled; unrelated ARs (e.g., longer-running background analyses) are preserved.
- Estimate: 5

D4 — Add targeted unit tests for edge cases and sequencing
- What: Add tests for race conditions: overlapping create/cancel flows, repeated rapid rollbacks, interactions with `rollbackWindow.revisions`.
- Files: `analysis/*_test.go`, `rollout/*_test.go`
- Acceptance criteria: Tests pass and show stable decisioning across sequences.
- Estimate: 5


Task E — Tests: unit + integration harness + minimal e2e guidance
- What: Create test coverage across these flows; include a small e2e recipe that runs in CI or locally with extended `scaleDownDelaySeconds` (e.g., 600s) to validate fast rollback in a synthetic environment.
- Files:
  - `rollout/*_test.go`, `analysis/*_test.go`, `rollout/trafficrouting/istio/*_test.go`
  - `test/fixtures/` or `examples/` manifests demonstrating basic canary with `scaleDownDelaySeconds` and rollback commands
- Acceptance criteria:
  - Unit tests exercising all new code paths pass locally and in CI.
  - A documented demo manifest reproduces the fast-rollback behavior when run locally against a kind/minikube cluster or in CI with a short timeout using a larger delay value.
- Estimate: 8-13

Task F — Documentation and recommended defaults
- What: Add docs explaining the basic-canary `scaleDownDelaySeconds` option, recommended operational values (recommend 300–600s in production), and examples showing how to trigger fast rollback.
- Files:
  - `docs/features/canary.md` (or existing docs location), README additions, and update `issue_analysis/*` with a short How-To
- Acceptance criteria:
  - Clear docs merged and linked from epic/PR
- Estimate: 3

Task G — Optional: CI e2e job (low priority)
- What: Small CI job that runs the demo manifest in a disposable cluster to validate fast rollback automatically (flakiness risk). Adds a gating step for PRs that touch rollout reconciliation.
- Estimate: 5

Fibonacci effort summary (per task)
- Task A: 3
- Task B: 8
- Task C: 8–13 (pick 13 if you want conservative planning)
- Task D: 5
- Task E: 8–13 (pick 13 conservative)
- Task F: 3
- Task G: 5 (optional)

Planned ordering / minimal cut

1. Task A (validation) — small, unblock other work
2. Task B (apply annotation logic to basic canary) — core behavior
3. Task C (fast rollback behavior for canary) — fast-track; depends on B
4. Task D (AnalysisRun creation rules) — make skip stable and safe
5. Task E (tests) — landing test coverage in parallel with code PRs
6. Task F (docs) — finalize PRs
7. Task G (optional CI e2e) — polish and long-term safety

Acceptance criteria for the epic

- A rollout configured with basic canary + `scaleDownDelaySeconds: 600` will retain the previous ReplicaSet for 10 minutes after promotion and will fast-rollback to it (skipping AnalysisRuns) if the operator triggers a rollback while the scale-down deadline is valid.
- Existing canary with traffic routing continues to work unchanged.
- No impact to `dynamicStableScale` users (we do not change or depend on that feature).
- Tests demonstrate behavior deterministically in unit test and as a documented local e2e run.

Risk analysis

- Changing validation exposes a new, potentially surprising option for basic canary users — mitigate via docs and examples.
- Scheduling/cluster autoscaler interactions: keeping old RS alive may increase resource usage; recommend operator explicitly set desired delays and monitor.
- Analysis-run creation logic must be conservative to avoid suppressing legitimate analyses; keep skip rules narrow (only when scale-down annotation present or within rollbackWindow).

Files likely to change (summary)

- `pkg/apis/rollouts/validation/validation.go`
- `rollout/canary.go`
- `rollout/analysis.go`
- `rollout/sync.go`
- `utils/replicaset/replicaset.go` (verify existing helpers)
- `rollout/*_test.go`, `analysis/*_test.go`, `rollout/trafficrouting/istio/*_test.go`
- Documentation files under `docs/` and `issue_analysis/` (update examples and recommendations)

Findings about the 30s default

- The repository defines the default as `DefaultScaleDownDelaySeconds = 30` and
  `DefaultAbortScaleDownDelaySeconds = 30` in `utils/defaults/defaults.go`. These are
  used by the `GetScaleDownDelaySecondsOrDefault` and
  `GetAbortScaleDownDelaySecondsOrDefault` helpers.
- The API comments in `pkg/apis/rollouts/v1alpha1/generated.proto` and
  `pkg/apis/rollouts/v1alpha1/experiment_types.go` document the 30s default and
  explicitly mention that the delay is intended to allow service/iptables or
  service-provider propagation after switching selectors (IPTables / routing
  convergence is cited historically).  The experiment types file even recommends
  "a minimum of 30 seconds" and links to discussion in the upstream issue history.
- Practical implication: 30s appears to be a historical minimum to allow network
  convergence, not an operational recommendation for rollbacks in production.
  For human operator-driven rollback or slower pipelines, 30s is usually too
  short.

Recommendation (docs action)

- Do not change controller defaults in this patch series. Instead, update docs
  and the examples to recommend a larger `scaleDownDelaySeconds` for production
  use (we recommend 300–600s depending on team tolerance for resources).
- Add an example manifest showing a 5 minute (`300`) `scaleDownDelaySeconds` and
  brief guidance on when to use `abortScaleDownDelaySeconds: 0` for debugging.


Next actions (I will take if you confirm)

- Convert the above tasks into PR branches and create the first minimal PR for Task A (validation) and Task B (scaleDown annotation behavior) with unit tests for the core happy-path. I can open draft PRs or produce local patches for review.
- After you confirm, I will also produce the Fibonacci effort roll-up (total estimated story points) and break down Task C/D/E into more granular subtasks and tests.

Guiding goal (derived from upstream discussion — see Issue #557):

Enable fast rollback to a previously validated revision inside the configured `rollbackWindow` even when its ReplicaSet has been scaled to 0, provided the controller can conservatively verify the revision was previously healthy (for example via a retained successful AnalysisRun, rollout status evidence, or a recent scale-down-deadline annotation).



Draft GitHub question / comment (paste into https://github.com/argoproj/argo-rollouts/issues/557)

```
Short question: For traffic-routed canaries, can the controller fast-track rollback to a revision inside `rollbackWindow` even if that ReplicaSet is scaled to 0, provided we can verify the revision was previously healthy (e.g., retained successful AnalysisRun, rollout.Status evidence, or a recent scale-down-deadline annotation)?

If yes, we propose a conservative helper-based approach that skips creating new AnalysisRuns and advances steps only when one of those signals is present; otherwise fallback to normal behavior. Happy to open a small, well-tested PR implementing this.
```

New ticket: C1' — Fast-rollback to known-good revision inside rollbackWindow even when RS scaled to 0
- What: Add helpers and logic so the controller may fast-track rollback to a revision inside `rollbackWindow` if the revision is "known good" via a retained successful AnalysisRun or other rollout status evidence, even when the ReplicaSet is scaled to 0.
- Acceptance: Given revision R inside rollbackWindow and evidence of prior success, rollback to R should skip new AnalysisRuns and advance steps; if R is GC'd, fallback to normal behavior.
- Files: `rollout/replicaset_helpers.go`, `rollout/analysis.go`, `rollout/canary.go`, tests
- Estimate: 5

---

*Prepared after ingesting all files in `issue_analysis/support_scaleDownDelaySeconds_fast_rollbacks_canary/`.*

