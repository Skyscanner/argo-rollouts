# In-Depth Issue Analysis Prompt

This prompt guides an AI agent through a comprehensive analysis of Argo Rollouts issues to determine their current state, historical context, and create actionable contribution plans.

## Overview

You are tasked with performing a deep technical analysis of a single Argo Rollouts issue to:
1. Understand the current state of the codebase
2. Assess if the issue is still relevant 
3. Investigate historical attempts at resolution
4. Create a comprehensive contribution plan
5. Provide accurate effort estimations

## Prerequisites

Before starting, ensure you have:
- Access to the Argo Rollouts codebase
- The specific issue details from `issue_analysis/issues_to_analyze.md`
- Understanding of the fibonacci point scale from `issue_analysis/point_scale.md`

## Analysis Process

### Step 1: Issue Selection and Setup
**Prompt the user:** "Which issue from the issues_to_analyze.md table would you like me to analyze? Please provide the issue name or select from:
1. Support scaleDownDelaySeconds & fast rollbacks with canary strategy
2. Argo-rollouts ignores maxSurge and maxUnavailable when traffic shifting is used  
3. Argo-rollouts waits for stable RS to be stable before scaling it down"

**Action:** Create issue-specific analysis folder `issue_analysis/<issue_name_sanitized>/`

**Wait for user approval before proceeding to Step 2**

### Step 2: Current Codebase Analysis
**Objectives:**
- Analyze the current implementation related to the issue
- Identify critical code paths and files
- Document current behavior vs expected behavior

**Tasks:**
1. Search codebase for relevant keywords, functions, and structures
2. Identify the main files and functions involved
3. Trace the code flow related to the issue
4. Document current implementation in `<issue_folder>/01_current_state_analysis.md`

**Output format for current_state_analysis.md:**
```markdown
# Current State Analysis: [Issue Name]

## Issue Summary
[Brief description of the issue]

## Critical Code Locations
- **File:** `path/to/file.go`
  - **Function/Struct:** `FunctionName`
  - **Lines:** X-Y
  - **Purpose:** [What this code does]

## Current Behavior
[Detailed description of how the system currently behaves]

## Expected Behavior
[What the issue requests to be changed]

## Code Flow Analysis
[Step-by-step flow of how the current implementation works]

## Key Findings
- [Finding 1]
- [Finding 2]
```

**Prompt the user:** "Current state analysis complete. Please review `<issue_folder>/01_current_state_analysis.md`. Should I proceed to assess if this issue is still relevant?"

**Wait for user approval before proceeding to Step 3**

### Step 3: Issue Relevance Assessment
**Objectives:**
- Determine if the issue is still a problem in the current codebase
- Check if any recent changes have addressed the issue
- Validate the issue against current code behavior

**Tasks:**
1. Compare current implementation with issue description
2. Test scenarios described in the issue (if applicable)
3. Check recent commits that might have addressed the issue
4. Document findings in `<issue_folder>/02_relevance_assessment.md`

**Output format for relevance_assessment.md:**
```markdown
# Issue Relevance Assessment: [Issue Name]

## Assessment Date
[Current date]

## Current Status
- [ ] Issue is still present
- [ ] Issue has been resolved
- [ ] Issue is partially resolved
- [ ] Issue is no longer relevant

## Evidence
[Detailed explanation of why the issue is/isn't still relevant]

## Testing Results
[If applicable, results of testing the scenarios described in the issue]

## Recent Changes Review
[Analysis of recent commits that might have impacted this issue]

## Conclusion
[Clear statement on whether this issue needs work]
```

**Prompt the user:** "Relevance assessment complete. The issue is [STILL RELEVANT/RESOLVED/PARTIALLY RESOLVED]. Should I proceed with historical analysis?"

**Wait for user approval before proceeding to Step 4**

### Step 4: Historical Analysis (Git Blame & Correlation)
**Objectives:**
- Use git blame to understand what changes were made to critical lines
- Correlate blame results with linked GitHub issues/PRs
- Identify previous attempts at resolution

**Tasks:**
1. Run git blame on critical files identified in Step 2
2. Analyze commit history related to the issue
3. Cross-reference with linked GitHub issues/PRs
4. Document in `<issue_folder>/03_historical_analysis.md`

**Commands to use:**
```bash
git blame <critical_file>
git log --grep="<issue_keywords>" --oneline
git log -p --follow <critical_file>
```

**Output format for historical_analysis.md:**
```markdown
# Historical Analysis: [Issue Name]

## Git Blame Analysis
### File: [filename]
- **Line X-Y:** Last modified by [commit_hash] ([author], [date])
- **Commit Message:** [message]
- **Analysis:** [What this change was trying to achieve]

## Related Commits
| Commit Hash | Date | Author | Message | Relevance |
|-------------|------|--------|---------|-----------|
| abc123 | 2024-01-01 | Author | Message | High/Medium/Low |

## Correlation with Linked Issues/PRs
- **GitHub Issue #557:** [Analysis of how it relates]
- **GitHub PR #3899:** [Analysis of what was attempted]

## Previous Resolution Attempts
1. **Attempt 1:** [Description]
   - **Outcome:** [Success/Failure/Partial]
   - **Why it didn't fully resolve:** [Explanation]

## Key Insights
- [Insight 1]
- [Insight 2]
```

**Prompt the user:** "Historical analysis complete. Found [X] previous attempts at resolution. Should I proceed to search for similar upstream issues?"

**Wait for user approval before proceeding to Step 5**

### Step 5: Upstream Issue Discovery
**Objectives:**
- Find similar issues/PRs in upstream Argo Rollouts repository
- Identify patterns and approaches used by the community
- Discover any ongoing work or discussions

**Tasks:**
1. Search upstream GitHub repository for similar issues
2. Analyze successful and unsuccessful attempts
3. Identify maintainer preferences and patterns
4. Document in `<issue_folder>/04_upstream_analysis.md`

**Search strategies:**
- Keywords from the issue description
- Code function/variable names
- Error messages or behaviors
- Related concepts (e.g., "canary", "rollback", "maxSurge")

**Output format for upstream_analysis.md:**
```markdown
# Upstream Analysis: [Issue Name]

## Search Strategy
**Keywords used:** [list of search terms]
**Date range:** [if applicable]

## Similar Issues Found
| Issue/PR # | Title | Status | Relevance | Key Insights |
|------------|-------|--------|-----------|--------------|
| #123 | Title | Open/Closed | High/Medium/Low | [Insights] |

## Community Approaches
### Successful Patterns
- [Pattern 1 with example]
- [Pattern 2 with example]

### Failed Approaches
- [Approach 1 and why it failed]
- [Approach 2 and why it failed]

## Maintainer Preferences
- [Preference 1 based on PR reviews]
- [Preference 2 based on comments]

## Ongoing Work
- [Any current PRs or discussions related to this issue]

## Recommendations for Contribution Strategy
- [Strategic recommendation 1]
- [Strategic recommendation 2]
```

**Prompt the user:** "Upstream analysis complete. Found [X] similar issues and identified [Y] key patterns. Should I proceed to create the contribution epic?"

**Wait for user approval before proceeding to Step 6**

### Step 6: Epic Creation
**Objectives:**
- Create a comprehensive epic with detailed tasks
- Break down the work into logical, manageable pieces
- Ensure tasks align with upstream contribution requirements

**Tasks:**
1. Define the epic scope and objectives
2. Break down into specific tasks
3. Identify dependencies and prerequisites
4. Document in `<issue_folder>/05_contribution_epic.md`

**Output format for contribution_epic.md:**
```markdown
# Contribution Epic: [Issue Name]

## Epic Overview
**Objective:** [High-level goal]
**Success Criteria:** 
- [ ] [Criterion 1]
- [ ] [Criterion 2]

## Prerequisites
- [ ] [Prerequisite 1]
- [ ] [Prerequisite 2]

## Epic Tasks

### Phase 1: Research & Design
- [ ] **Task 1.1:** [Detailed task description]
  - **Acceptance Criteria:** [Specific criteria]
  - **Dependencies:** [Any dependencies]
  
- [ ] **Task 1.2:** [Detailed task description]
  - **Acceptance Criteria:** [Specific criteria]
  - **Dependencies:** [Any dependencies]

### Phase 2: Implementation
- [ ] **Task 2.1:** [Detailed task description]
  - **Acceptance Criteria:** [Specific criteria]
  - **Dependencies:** [Any dependencies]

### Phase 3: Testing & Documentation
- [ ] **Task 3.1:** [Detailed task description]
  - **Acceptance Criteria:** [Specific criteria]
  - **Dependencies:** [Any dependencies]

### Phase 4: Contribution & Review
- [ ] **Task 4.1:** [Detailed task description]
  - **Acceptance Criteria:** [Specific criteria]
  - **Dependencies:** [Any dependencies]

## Risk Assessment
| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|-------------------|
| Risk 1 | High/Medium/Low | High/Medium/Low | [Strategy] |

## Technical Approach
[Detailed technical approach based on analysis]

## Contribution Strategy
[How to approach the upstream community based on upstream analysis]
```

**Prompt the user:** "Epic created with [X] tasks across [Y] phases. Please review the epic structure. Should I proceed with fibonacci estimation?"

**Wait for user approval before proceeding to Step 7**

### Step 7: Fibonacci Estimation
**Objectives:**
- Estimate each task using fibonacci sequence (1, 2, 3, 5, 8, 13)
- Provide reasoning for each estimation
- Follow the point scale guidelines from point_scale.md

**Reference the point scale:**
- **1 point:** Very small, quick tasks (< few hours)
- **2 points:** Small tasks (< 1 day)  
- **3 points:** Medium tasks (1-2 days)
- **5 points:** Larger tasks (2-4 days)
- **8 points:** Large tasks (1 week, high risk)
- **13 points:** Epic-level work that needs breakdown

**Tasks:**
1. Estimate each task individually
2. Provide reasoning for each estimate
3. Calculate total epic estimate
4. Document in `<issue_folder>/06_effort_estimation.md`

**Output format for effort_estimation.md:**
```markdown
# Effort Estimation: [Issue Name]

## Estimation Guidelines Used
Based on point_scale.md:
- 1 point: < few hours
- 2 points: < 1 day
- 3 points: 1-2 days
- 5 points: 2-4 days
- 8 points: ~1 week (high risk)
- 13 points: Needs breakdown

## Task Estimations

### Phase 1: Research & Design
- **Task 1.1:** [Task name] - **[X] points**
  - **Reasoning:** [Detailed reasoning for estimate]
  - **Assumptions:** [Any assumptions made]

- **Task 1.2:** [Task name] - **[X] points**  
  - **Reasoning:** [Detailed reasoning for estimate]
  - **Assumptions:** [Any assumptions made]

### Phase 2: Implementation
[Continue for all tasks...]

## Summary
| Phase | Tasks | Total Points |
|-------|-------|--------------|
| Phase 1 | X | Y points |
| Phase 2 | X | Y points |
| Phase 3 | X | Y points |
| Phase 4 | X | Y points |
| **Total** | **X** | **Y points** |

## Estimation Confidence
- **High Confidence:** [List tasks with high confidence]
- **Medium Confidence:** [List tasks with medium confidence]  
- **Low Confidence:** [List tasks requiring more research]

## Recommendations
- [Recommendation for high-risk tasks]
- [Suggestion for breaking down large estimates]
- [Timeline recommendations based on point scale guidelines]
```

**Prompt the user:** "Estimation complete. Epic totals [X] points across [Y] tasks. This suggests approximately [Z] weeks of work. Please review all analysis files. Is there anything you'd like me to revise or expand on?"

## Final Deliverables

At the completion of this analysis, you will have created:

1. `<issue_folder>/01_current_state_analysis.md` - Technical analysis of current implementation
2. `<issue_folder>/02_relevance_assessment.md` - Current status and relevance  
3. `<issue_folder>/03_historical_analysis.md` - Git blame and historical context
4. `<issue_folder>/04_upstream_analysis.md` - Community patterns and approaches
5. `<issue_folder>/05_contribution_epic.md` - Detailed epic and task breakdown
6. `<issue_folder>/06_effort_estimation.md` - Fibonacci-based effort estimates

## Usage Instructions

1. Start by asking the agent to follow this prompt
2. Provide the specific issue you want analyzed
3. Review and approve each step before proceeding
4. All analysis will be documented in markdown files for easy reference
5. Use the final epic and estimates for sprint planning and contribution strategy

## Quality Checklist

Before concluding the analysis, ensure:
- [ ] All critical code paths have been identified
- [ ] Historical context is thoroughly documented  
- [ ] Upstream patterns have been researched
- [ ] Epic tasks are specific and actionable
- [ ] Estimates follow fibonacci scale guidelines
- [ ] All markdown files are complete and well-formatted