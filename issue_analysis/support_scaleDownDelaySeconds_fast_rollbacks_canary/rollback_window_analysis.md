# Rollback Window Logic Analysis

## Overview
Analysis of rollback window inconsistencies observed in production, revealing a controller timing bug that explains why analysis runs are created despite being "within the window".

## Production Context
- **Deployment:** Canary with Istio traffic routing, `rollbackWindow: revisions: 3`
- **Steps:** 6-step canary (setWeight → analysis → setWeight → analysis → setWeight → analysis)
- **Analysis:** Step-based only using `nr-error-rate` template with New Relic metrics
- **Observation:** All rollbacks log "within the window" but analysis runs still created inconsistently

## Root Cause: ReplicaSet Creation Timestamp Logic

### VERIFIED HYPOTHESIS: Rollback Detection Based on ReplicaSet Age vs. Current Stable

**Production Evidence with Actual ReplicaSet Timestamps:**
```
Pod Hash  ReplicaSet Creation Time    Relative Age
675       2025-10-17T08:12:20Z       OLDER (created first)
d99       2025-10-17T11:02:51Z       NEWER (created ~3 hours later)
```

**❌ FLAWED LOGIC IDENTIFIED: Timestamp-Based Rollback Detection**

The current implementation uses ReplicaSet creation timestamps to detect rollbacks, which is fundamentally flawed:

```go
// Current (problematic) logic:
if c.newRS.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) {
    // This assumes creation time = deployment order, which is wrong
}
```

**Why This Approach Could Be Problematic:**
1. **ReplicaSet creation time ≠ deployment progression order**
2. **Same images reuse existing ReplicaSets, keeping original creation timestamps**
3. **Rollback should ideally be based on revision history, not physical creation time**

**More Logical Approach Would Be:**
```
1. Is target pod hash different from current stable? → Potential rollback
2. Is target pod hash in recent N revisions of rollout history? → Within window  
3. If both true → Skip analysis (fast rollback)
```

**Production Data with Revision Context:**
```
ReplicaSet  Revision  Creation Time        Stable Status    Rollback Logic Result
675         427       08:12:20Z (older)    No              427 < 428 → Within window → Skip analysis  
d99         428       11:02:51Z (newer)    Yes (current)   428 = stable → Forward deployment → Run analysis
```

**Rollout Status:** `stableRS: d99d84ff` (revision 428)  
**Rollback Window:** `revisions: 3` (should include 426, 427, 428)

**The Critical Discovery:**
- **Revision logic makes perfect sense:** 427 → 428 is rollback within 3-revision window
- **But code uses timestamps, not revisions** for rollback detection  
- **Behavior works by pure coincidence:** Timestamp order happens to match revision order

**Timestamp Logic (What Code Actually Does):**
```go
if c.newRS.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) {
    // 675 (08:12) < d99 (11:02) = TRUE → Rollback detected
}
```

**Revision Logic (What Should Happen):**
```go
if targetRevision < stableRevision && withinRevisionWindow {
    // 427 < 428 && within 3 revisions = TRUE → Rollback detected  
}
```

**The behavior appears to work correctly, but uses fundamentally wrong logic.**

**Critical Understanding: Stable ReplicaSet Determination**
```go
func GetStableRS(rollout *v1alpha1.Rollout, newRS *appsv1.ReplicaSet, rslist []*appsv1.ReplicaSet) *appsv1.ReplicaSet {
	if rollout.Status.StableRS == "" {
		return nil
	}
	// ✅ Find ReplicaSet matching the pod hash stored in rollout.Status.StableRS
	if newRS != nil && newRS.Labels[v1alpha1.DefaultRolloutUniqueLabelKey] == rollout.Status.StableRS {
		return newRS
	}
	for i := range rslist {
		rs := rslist[i]
		if rs != nil {
			if rs.Labels[v1alpha1.DefaultRolloutUniqueLabelKey] == rollout.Status.StableRS {
				return rs
			}
		}
	}
	return nil
}
```

**How It Actually Works:**
1. **Current stable pod hash** stored in `rollout.Status.StableRS` 
2. **Target deployment** creates/reuses ReplicaSet for new pod hash
3. **Rollback detected** when target ReplicaSet creation timestamp < stable ReplicaSet creation timestamp
4. **ReplicaSet timestamps persist** - they don't change when redeploying same image/pod hash

**Fundamental Design Flaw Identified:**

The rollback window implementation conflates **ReplicaSet creation chronology** with **deployment revision history**, leading to:

1. **Brittle behavior** - Works by accident in some cases, fails in others
2. **Counterintuitive logic** - Timestamps don't represent rollout progression
3. **Unpredictable results** - Same revision can be treated as rollback or forward deployment based on creation timing

**Expected vs. Actual Behavior:**
- **Expected:** "Rollback within 3 revisions" = deploy to any of the last 3 deployed versions
- **Actual:** "Rollback if target ReplicaSet physically created before stable ReplicaSet"

**Why This Matters:**
In environments like Slingshot where deployments can be re-run or images redeployed, the physical ReplicaSet creation time becomes disconnected from the logical deployment sequence, making rollback detection unreliable.

**Analysis of `isRollbackWithinWindow()` Complete Logic:**

```go
func (c *rolloutContext) isRollbackWithinWindow() bool {
	if c.newRS == nil || c.stableRS == nil {
		return false
	}
	// ✅ STEP 1: Detect rollback by comparing ReplicaSet ages
	if c.newRS.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) {
		// ✅ STEP 2: Check if rollback window is configured
		if c.rollout.Spec.RollbackWindow != nil {
			if c.rollout.Spec.RollbackWindow.Revisions > 0 {
				var windowSize int32
				// ✅ STEP 3: Count intermediate ReplicaSets in time window
				for _, rs := range c.allRSs {
					if rs.Annotations != nil && rs.Annotations[v1alpha1.ExperimentNameAnnotationKey] != "" {
						continue // Skip experiment ReplicaSets
					}

					// Count ReplicaSets created between newRS and stableRS
					// Timeline: newRS < rs < stableRS
					if rs.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) &&
						c.newRS.CreationTimestamp.Before(&rs.CreationTimestamp) {
						windowSize = windowSize + 1
					}
				}
				// ✅ STEP 4: Compare window size to configured limit
				if windowSize < c.rollout.Spec.RollbackWindow.Revisions {
					c.log.Infof("Rollback within the window: %d (%v)", windowSize, c.rollout.Spec.RollbackWindow.Revisions)
					return true  // ← ANALYSIS GETS SKIPPED
				}
				c.log.Infof("Rollback outside the window: %d (%v)", windowSize, c.rollout.Spec.RollbackWindow.Revisions)
			}
		}
	}
	return false  // ← ANALYSIS PROCEEDS NORMALLY
}
```

**Complete Picture: Why Production Shows Consistent "Within Window" Logging**

**rollbackWindow: revisions: 3** means:
- Skip analysis if deploying to ReplicaSet with ≤2 intermediate ReplicaSets between target and stable
- Production shows **44 "within window"** occurrences because most rollbacks fall within this 3-revision window
- **0 "outside window"** means no rollbacks exceeded the 3-revision limit

**The Behavior is Actually CORRECT:**
- Code logic works as designed
- Rollback window detection is reliable  
- Analysis skipping happens when intended
- Production correlation pattern shows the feature working properly

## Key Insights - FINAL CORRECTED ANALYSIS

### 1. Rollback Window Logic Works Perfectly
The entire rollback window implementation functions exactly as designed:
- ✅ Rollback detection based on ReplicaSet creation timestamps
- ✅ Window size calculation by counting intermediate ReplicaSets  
- ✅ Analysis skipping when rollback is within configured window
- ✅ Production correlation confirms correct behavior

### 2. No Controller Timing Bug Exists
All previous hypotheses about timing bugs or race conditions were **incorrect**. The controller logic is sound.

### 3. Production Pattern Fully Explained
**Commit 10c69cce (pod hash 675):** Analysis skipped → Rollback to older ReplicaSet within window
**Commit 38f23a1e (pod hash d99):** Analysis runs → Forward deployment to newer ReplicaSet

### 4. **The REAL Issue: No Analysis Deduplication Logic**

The fundamental problem isn't rollback window detection - it's the **lack of analysis deduplication**.

**The Obvious Solution:**
```
If successful analysis already exists for target pod hash:
  → Skip analysis (fast rollback)
Else:
  → Run analysis normally
```

**Current Behavior:**
- Analysis runs are labeled with pod hash (`DefaultRolloutUniqueLabelKey`)
- But controller doesn't check for existing successful analysis runs before creating new ones
- Results in redundant analysis for same pod hash

**Evidence:**
- Line 393 in analysis.go: `stepLabels := analysisutil.StepLabels(*index, podHash, instanceID)`
- Labels include pod hash, but no lookup logic exists to find completed analysis runs by pod hash

### 5. **Why Current Rollback Window Implementation is Accidentally Correct but Fundamentally Flawed**

**The Production Data Reveals:**
- **Revision-based logic would be correct:** 675 (rev 427) → d99 (rev 428) is rollback within 3-revision window
- **Code uses timestamp logic instead:** Works by coincidence when creation timestamps align with revision order
- **Fragile foundation:** Will break when timestamp order != revision order

**Three approaches to consider:**

**1. Current (Timestamp-based):**
```go
// Fragile - works only when timestamps match revision order
if c.newRS.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) && withinTimestampWindow
```

**2. Revision-based (Logical):**
```go  
// Correct - uses actual deployment sequence
if targetRevision < stableRevision && withinRevisionWindow
```

**3. Analysis deduplication (Simplest):**
```go
// Most efficient - avoid redundant work entirely
if successfulAnalysisExistsForPodHash(targetPodHash)
```

**Key Insight:** Production shows the **revision-based approach would work perfectly**, but current timestamp-based implementation is accidentally working due to coincidental ordering.

## Relationship to Enhancement Request

### Current Limitations
1. **Controller timing bug** causes unreliable fast rollback behavior
2. **Analysis termination dependency** introduces variable delays
3. **Canary strategy restriction** prevents using `scaleDownDelaySeconds`

### Enhancement Value - REVISED
The requested `scaleDownDelaySeconds` for canary strategy would:
1. **Eliminate state complexity** - Simple time-based mechanism vs. complex controller state management
2. **Provide deterministic guarantees** - Predictable 30-second window vs. controller timing dependencies  
3. **Remove analysis dependency** - Fast rollback based on ReplicaSet availability, not analysis state
4. **Match blue-green parity** - Consistent fast rollback mechanism across strategies
5. **Reduce race conditions** - ReplicaSet-level mechanism immune to controller reconciliation timing issues

### Why Enhancement is Superior to Current Approach
Even though the rollback window logic works correctly in theory, `scaleDownDelaySeconds` provides:
- **Simpler implementation** - No complex controller state tracking
- **Fewer failure modes** - Time-based mechanism vs. state-dependent logic
- **Better predictability** - Fixed time guarantee vs. variable controller behavior
- **Reduced complexity** - ReplicaSet lifecycle vs. analysis run management