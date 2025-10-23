# Rollback Window Mechanisms - Technical Documentation

## Overview
Technical documentation of how Argo Rollouts rollback window logic works, including ReplicaSet lifecycle, revision management, and observed behavior.

## ReplicaSet Lifecycle and Pod Hash Management

### ReplicaSet Creation vs. Reuse
**Behavior:** ReplicaSets are reused for the same pod template, not recreated.

```go
// FindNewReplicaSet first tries to find existing RS by pod hash
func FindNewReplicaSet(rollout *v1alpha1.Rollout, rsList []*appsv1.ReplicaSet) *appsv1.ReplicaSet {
    podHash := hash.ComputePodTemplateHash(&rollout.Spec.Template, rollout.Status.CollisionCount)
    if rs := searchRsByHash(rsList, podHash); rs != nil {
        return rs  // REUSES existing ReplicaSet
    }
    // Only creates new RS if no existing one matches pod hash
}

func searchRsByHash(rsList []*appsv1.ReplicaSet, hash string) *appsv1.ReplicaSet {
    for _, rs := range rsList {
        if rs.Labels[v1alpha1.DefaultRolloutUniqueLabelKey] == hash {
            return rs  // Found existing RS with same pod hash
        }
    }
    return nil
}
```

**Facts:**
1. Same image/config generates same pod hash
2. Same pod hash results in ReplicaSet object reuse
3. Creation timestamp remains unchanged when redeploying same configuration
4. Only revision annotation gets updated on existing ReplicaSet

### Revision Update Process
```go
func (c *rolloutContext) syncReplicaSetRevision() (*appsv1.ReplicaSet, error) {
    // Calculate new revision number
    maxOldRevision := replicasetutil.MaxRevision(c.olderRSs)
    newRevision := strconv.FormatInt(maxOldRevision+1, 10)
    
    // Update existing ReplicaSet annotations (including revision)
    rsCopy := c.newRS.DeepCopy()
    annotationsUpdated := annotations.SetNewReplicaSetAnnotations(c.rollout, rsCopy, newRevision, true)
    
    if annotationsUpdated {
        rs, err := c.updateReplicaSet(ctx, rsCopy)  // Updates existing RS
        return rs, nil
    }
}
```

**Result:** Same ReplicaSet object gets updated with new revision number, but `creationTimestamp` remains unchanged.

## Rollback Window Detection Logic

### Current Implementation (Timestamp-Based)
```go
func (c *rolloutContext) isRollbackWithinWindow() bool {
    if c.newRS == nil || c.stableRS == nil {
        return false
    }
    
    // STEP 1: Rollback detection by timestamp comparison
    if c.newRS.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) {
        
        // STEP 2: Window size calculation by timestamp counting
        if c.rollout.Spec.RollbackWindow != nil && c.rollout.Spec.RollbackWindow.Revisions > 0 {
            var windowSize int32
            for _, rs := range c.allRSs {
                // Count ReplicaSets created between newRS and stableRS
                if rs.CreationTimestamp.Before(&c.stableRS.CreationTimestamp) &&
                   c.newRS.CreationTimestamp.Before(&rs.CreationTimestamp) {
                    windowSize = windowSize + 1
                }
            }
            
            // STEP 3: Compare to configured window
            if windowSize < c.rollout.Spec.RollbackWindow.Revisions {
                return true  // Within window → Skip analysis
            }
        }
    }
    return false
}
```

### Key Characteristics
1. **Rollback Detection:** `newRS.CreationTimestamp < stableRS.CreationTimestamp`
2. **Window Counting:** Number of ReplicaSets with timestamps between new and stable
3. **Window Check:** `windowSize < configuredRevisions`

## Observed Production Behavior

### Two-ReplicaSet Flip-Flop Scenario
**Environment:**
- RS1 (675): `creationTimestamp: 08:12:20Z`, `revision: 427`
- RS2 (d99): `creationTimestamp: 11:02:51Z`, `revision: 428`
- Rollback window: `revisions: 3`

**Observed Behavior:**
```
Deploy RS1 (675) when RS2 (d99) is stable:
├─ newRS = RS1 (08:12:20Z)
├─ stableRS = RS2 (11:02:51Z)  
├─ 08:12 < 11:02 = TRUE → Rollback detected
├─ WindowSize = 0 (no RSs between 08:12 and 11:02)
├─ 0 < 3 = TRUE → Within window
└─ Result: Skip analysis

Deploy RS2 (d99) when RS1 (675) is stable:
├─ newRS = RS2 (11:02:51Z)
├─ stableRS = RS1 (08:12:20Z)
├─ 11:02 < 08:12 = FALSE → Not a rollback
└─ Result: Run analysis
```

**Environment Characteristics:**
- Only 2 ReplicaSets exist in environment
- RS1 has older creation timestamp than RS2
- Window size is always 0 (no intermediate ReplicaSets)
- Rollback only triggered when deploying RS1 (older timestamp) over RS2 (newer timestamp)

### Multi-ReplicaSet Scenario (Theoretical)
**Example with 3+ ReplicaSets:**
```
Day 1: Deploy image A → RS_A (hash: abc, created: 09:00, rev: 100)
Day 2: Deploy image B → RS_B (hash: def, created: 10:00, rev: 101) 
Day 3: Deploy image A → RS_A reused (hash: abc, created: 09:00, rev: 102)
```

**Timestamp vs Revision Relationship:**
- RS_A: older timestamp (09:00), newer revision (102)
- RS_B: newer timestamp (10:00), older revision (101)
- Timestamp order: 09:00 < 10:00
- Revision order: 102 > 101

## Current Implementation Facts

### Timestamp vs Revision Characteristics
- **ReplicaSet creation time:** Reflects when pod template was first encountered
- **Revision number:** Reflects actual deployment sequence order
- **Relationship:** Can diverge when same configurations are redeployed at different times

### Analysis Run Behavior
- **Labeling:** Analysis runs are labeled with pod hash values
- **Deduplication:** No logic exists to check for existing successful analysis runs with same pod hash
- **Result:** Same pod hash configurations analyzed multiple times