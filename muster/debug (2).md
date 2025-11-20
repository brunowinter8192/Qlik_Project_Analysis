---
description: Systematic debugging with context gathering, debug-specialist subagent and bug documentation
argument-hint: [observation/error-description]
---

## Problem Observation

User observes: $ARGUMENTS

---

## Phase 1: Context Gathering

**CRITICAL:**
- Read the codebase BROADLY to understand the user's problem and effectively prompt the subagent
- If unclear what the user means, ask clarifying questions before prompting the subagent

1. Identify affected modules based on the observation
2. Check bug_fixes/ in root to see if such an issue existed before, it's possible the issue was previously fixed but reappeared due to a new feature
3. Find relevant files in the codebase (max 3-4, focused)
4. Check `logs/` folder if present
5. Debug workspaces: `debug/Agent_1/`, `debug/Agent_2/`, `debug/Agent_3/` (isolated per agent)
6. Gather: Error messages, stack traces, affected functions with File:Line references

---

## Phase 2: Parallel Multi-Agent Debug

**CRITICAL:** Launch 3 debug-specialist agents in parallel to get multiple perspectives.

**MANDATORY:** Each agent gets its own isolated workspace:
- Agent 1 → `debug/Agent_1/`
- Agent 2 → `debug/Agent_2/`
- Agent 3 → `debug/Agent_3/`

Call 3 Task tools **in a single message** (parallel execution):

```json
{
  "tool": "Task",
  "parameters": {
    "description": "debug issue (Agent 1)",
    "subagent_type": "debug-specialist",
    "prompt": "## Problem Description\n<Precise description of what is failing, expected vs actual behavior>\n\n## Investigation Results\n<Findings from Phase 1: error messages, stack traces, affected functions with File:Line references>\n\n## Recommended Starting Points\n<Specific folders and files the subagent should examine first, ordered by relevance>\n\n## CRITICAL: Workspace\nYou MUST create all debug scripts and test files in: debug/Agent_1/\nThis is YOUR isolated workspace. Other agents work in Agent_2/ and Agent_3/"
  }
}
```

Repeat for Agent 2 and Agent 3 with their respective workspace paths.

---

## Phase 2.3: Extract Agent IDs

**CRITICAL:** After launching the 3 agents in parallel, extract their Agent IDs for later resumption.

**Implementation:**
```python
import glob
from pathlib import Path
import time

# Wait briefly for JSONL files to be written
time.sleep(0.5)

# Find Claude projects directory
claude_projects_dir = Path.home() / ".claude" / "projects"
all_agent_files = list(claude_projects_dir.glob("*/agent-*.jsonl"))

# Sort by modification time (newest first)
sorted_agents = sorted(all_agent_files, key=lambda f: f.stat().st_mtime, reverse=True)

# Take the 3 newest (our just-launched agents)
latest_3 = sorted_agents[:3]

# Extract Agent IDs (e.g., "agent-abc123" from "agent-abc123.jsonl")
agent_1_id = latest_3[0].stem  # agent-abc123
agent_2_id = latest_3[1].stem  # agent-def456
agent_3_id = latest_3[2].stem  # agent-ghi789

# Log for reference
print(f"Agent IDs extracted: {agent_1_id}, {agent_2_id}, {agent_3_id}")
```

**Note:** These IDs will be used in Phase 2.4 to resume agents with GO/REDIRECT instructions.

---

## Phase 2.4: Assess Agent Debug Plans

**CRITICAL:** Agents complete Steps 1+2 (Root Cause + Reproduce) and then report with their solution plans.

After Phase 2 agents return with their reproduction results and solution plans, assess their approaches:

### Collect Agent Reports

Each agent provides:
1. **Root Cause Analysis** - What they identified as the problem source (What/Where/Why)
2. **Reproduction Results** - Whether bug was successfully reproduced, key findings from `reproduce_[issue].py`
3. **Solution Plan** - Hypothesized fix and planned solution scripts (describe only, not created yet)

### Assessment Criteria

#### 1. Root Cause Assessment

**Main Agent's Role:**
- ❌ NOT: Deep root cause evaluation (Main Agent doesn't have full context)
- ✅ YES: Catch obviously wrong/absurd root causes

**Intervention Logic:**

```
IF agent has clearly absurd root cause (obviously wrong):
  REDIRECT: "Your root cause analysis seems off. Consider [hint]."

IF all 3 agents agree on SAME root cause AND SAME solution approach:
  REDIRECT: "All agents plan to fix [root cause] with [method X].
             Agent 2: Try fixing it with [method Y] instead
             Agent 3: Try fixing it with [method Z] instead"

OTHERWISE:
  LET AGENTS TEST their different theories
```

**Principle:** Diversify solution approaches when consensus exists, catch extreme outliers.

#### 2. Test Strategy Assessment

**CRITICAL:** Don't let agents fuck around in isolated test environments.

**The Message:**
Testing MUST happen in the **actual production context** with the **actual framework** the project uses.

**RED FLAGS - Agent is jerking off in isolation:**

❌ "I'll test this in a standalone Python script"
❌ "Let me create mock data in debug/"
❌ "I'll validate with json.dumps()"
❌ "Quick test to see if function returns the right value"

**What's ACTUALLY needed:**

✅ Test with the **real framework/stack** the project runs on
✅ Use **actual production imports and structure**
✅ Validate in **full workflow context**, not synthetic bullshit

**Intervention Logic:**

```
IF agent plans to test ONLY in isolation without using the production framework:
  REDIRECT: "Stop testing in isolation. Your fix needs to work in the ACTUAL
             production context. Test with:
             - The actual framework/stack this project uses
             - Real production imports and structure
             - Full workflow context, not mock data"
```

**Why?** Because a fix that works in a vacuum but breaks in production is worthless.

#### 3. Approach Diversity

- Do agents cover different solution strategies?
- If all test same approach: Assign different methods to ensure diversity

#### 4. Plan Quality

- Is the debug plan comprehensive?
- Does it cover reproduction + validation?
- Are planned scripts well-structured?

### Decision & Recommendation Format

For each agent, document:

```
AGENT [X] ASSESSMENT

Root Cause: [Agent's identified root cause]
Root Cause Sanity: ✅ Reasonable / ⚠️ Questionable / ❌ Absurd

Reproduction: ✅ Bug reproduced / ❌ Could not reproduce
Key Findings: [What the reproduction revealed]

Planned Solution: [Brief summary of solution hypothesis]
Planned Scripts: [List of solution scripts agent will create in Phase 2]

SOLUTION STRATEGY:
- Uses Production Framework: ✅/❌
- Uses Real Project Structure: ✅/❌
- Tests in Full Workflow Context: ✅/❌

⚠️ ISSUE: [If testing only in isolation or absurd root cause, describe]
RECOMMENDATION: [GO / REDIRECT]

OVERLAP CHECK:
- With Agent Y: [HIGH/MEDIUM/LOW overlap - explain]
- With Agent Z: [HIGH/MEDIUM/LOW overlap - explain]

QUALITY: [EXCELLENT/GOOD/NEEDS_IMPROVEMENT - explain]

DECISION: GO / REDIRECT / MERGE
REASONING: [Why this decision]

INSTRUCTIONS TO AGENT:
[If GO: "Proceed with your planned solution approach"]
[If REDIRECT for approach: "Try fixing [root cause] with [method Y] instead of [method X]"]
[If REDIRECT for isolation: "Stop testing in isolation. Use the actual production framework and workflow context"]
[If REDIRECT for root cause: "Your reproduction was good but root cause seems off. Consider [hint]"]
[If MERGE: "Collaborate with Agent Y to test [combined approach]"]
```

### Overall Recommendation

After assessing all 3 agents:

```
OVERALL ASSESSMENT

Root Cause Consensus: YES/NO - [explanation]
Reproduction Success: [X/3 agents successfully reproduced the bug]
Solution Approach Diversity: EXCELLENT/GOOD/POOR - [explanation]

SOLUTION STRATEGY ASSESSMENT:
- Agent 1: ✅ Using production framework / ❌ Isolated testing only
- Agent 2: ✅ Using production framework / ❌ Isolated testing only
- Agent 3: ✅ Using production framework / ❌ Isolated testing only

AGENTS TO PROCEED:
- Agent 1: [GO/REDIRECT/MERGE] - [brief instruction]
- Agent 2: [GO/REDIRECT/MERGE] - [brief instruction]
- Agent 3: [GO/REDIRECT/MERGE] - [brief instruction]

EXPECTED OUTCOME:
[What solutions/validations you expect from the 3 agents in Phase 2]
```

**CRITICAL: PRESENT THIS ASSESSMENT TO THE USER**

This is NOT internal analysis. The user MUST see:
1. Each agent's individual assessment (GO/REDIRECT/MERGE decisions with reasoning)
2. The OVERALL ASSESSMENT with all agents' statuses
3. Expected outcomes after redirection

**Format for user presentation:**
- Show all 3 AGENT [X] ASSESSMENT blocks
- Show the OVERALL ASSESSMENT block
- Use clear formatting so user can review your decisions

Do NOT proceed to send continuation instructions until user has seen this report.

### Send Continuation Instructions

Resume each agent with specific instructions based on decisions above. Agents continue from **Step 3 (Develop Solution)** with their assigned approach.

**IMPORTANT:** Agents have already completed:
- Step 1: Root Cause Analysis
- Step 2: Reproduction (created and executed `reproduce_[issue].py`)
- Step 2.5: Reported to Main Agent

Now they proceed with Phase 2:
- Step 3: Develop Solution
- Step 4: Validate Solution
- Step 5: Final Report

**🔥 CRITICAL: Use the extracted agent IDs from Phase 2.3 (HASH ONLY) 🔥**

**REMINDER:**
- Agent IDs are stored as HASH ONLY (e.g., "abc123")
- DO NOT use "agent-" prefix in resume parameter
- ✅ CORRECT: `resume="abc123"`
- ❌ WRONG: `resume="agent-abc123"` (will fail with "No transcript found")

```python
# Resume Agent 1
Task(
    subagent_type="debug-specialist",
    description="Agent 1 - continue",
    resume=agent_1_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 2
Task(
    subagent_type="debug-specialist",
    description="Agent 2 - continue",
    resume=agent_2_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 3
Task(
    subagent_type="debug-specialist",
    description="Agent 3 - continue",
    resume=agent_3_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)
```

**Example of correct usage:**
```python
# If extracted ID is "62b9126b" (correct format from Phase 2.3)
Task(
    subagent_type="debug-specialist",
    description="Agent 1 - continue",
    resume="62b9126b",  # ✅ Correct
    prompt="..."
)

# NOT like this:
# resume="agent-62b9126b"  # ❌ Wrong - will fail
```

---

## Phase 2.5: Agent Results Aggregation

After all 3 agents complete, analyze their solutions:

### Comparison Criteria
1. **Consensus Check**: Do all 3 agents identify the same root cause?
2. **Solution Diversity**: How different are the proposed fixes?
3. **Test Quality**: Which agent wrote the most comprehensive debug scripts?
4. **Root Cause Analysis**: Which agent went deepest in identifying the issue?
5. **CLAUDE.md Compliance**: Which solution follows standards best?

### Assessment Output
Present to yourself (before showing user):

```
AGENT COMPARISON REPORT
=======================

Root Cause Consensus:
- Agent 1: [brief summary]
- Agent 2: [brief summary]
- Agent 3: [brief summary]
- Consensus: [YES/NO - explain]

Proposed Solutions:
- Agent 1: [approach summary]
- Agent 2: [approach summary]
- Agent 3: [approach summary]

Test Coverage:
- Agent 1: [files created, test comprehensiveness]
- Agent 2: [files created, test comprehensiveness]
- Agent 3: [files created, test comprehensiveness]

RECOMMENDED SOLUTION: Agent [X]
REASONING: [Why this solution is best - consider all criteria above]

REJECTED SOLUTIONS:
- Agent [Y]: [Why rejected or inferior]
- Agent [Z]: [Why rejected or inferior]
```

---

## Phase 3: Present Multi-Agent Analysis to User

After completing Phase 2.5 aggregation:

1. **Show the Agent Comparison Report** (from Phase 2.5)
2. **Present your recommended solution** with clear reasoning
3. **Highlight key differences** between the 3 approaches
4. **Show consensus areas** (if all agents agreed on root cause)
5. **WAIT for explicit user confirmation** before proceeding
6. Ask: "Should I implement the recommended fix from Agent [X]?"

**IMPORTANT:** User might choose a different agent's solution than your recommendation. Be prepared to implement their choice.

---

## Phase 4: Implementation and Documentation (after Approval)

### 4.1 Implement Fix
- Apply the proposed fix
- Test the changes

### 4.15 Validate Scraper Quality (If Scraper Modified)

**CRITICAL:** If the fix modified `src/scraper/scrape_url.py`, validate quality impact.

**Detection:**
- Check if `src/scraper/scrape_url.py` was changed in the fix
- If YES: Proceed with validation
- If NO: Skip to Phase 4.2

**Launch tester-specialist agent:**

Use the Task tool to run quality validation:

```json
{
  "tool": "Task",
  "parameters": {
    "description": "validate scraper quality after fix",
    "subagent_type": "tester-specialist",
    "model": "haiku",
    "prompt": "## Context\nBug fix applied to src/scraper/scrape_url.py.\n\n## Fix Description\n[Brief description of what was fixed]\n\n## Task\nRun the scraping suite baseline and comparison to validate the fix maintains or improves content extraction quality across all test domains.\n\n## Required Analysis\n1. Execute baseline suite\n2. Compare with previous iteration\n3. Analyze character/word count changes\n4. Review content diffs for regressions\n5. Assess: PASS (no regressions) / REVIEW (changes need review) / FAIL (regressions detected)\n\n## Required Output\n- Domain-by-domain analysis with concrete evidence\n- Overall assessment (PASS/REVIEW/FAIL)\n- Clear recommendation"
  }
}
```

**Wait for validation report.**

**Decision Gates:**
- **PASS**: Quality maintained or improved → Proceed to Phase 4.2
- **REVIEW**: Moderate changes detected → Present findings to user, wait for approval before Phase 4.2
- **FAIL**: Regressions detected → Return to Phase 4.1, refine the fix based on tester-specialist findings

**If FAIL:**
1. Review tester-specialist report to understand regressions
2. Refine fix in the selected agent's workspace (Agent_1/2/3/)
3. Re-apply fix
4. Re-run Phase 4.15 validation
5. Iterate until PASS or REVIEW

### 4.2 Document Bug Fix
**Location:** `bug_fixes/` (in project root)

**Filename:** `[descriptive-name]_YYYYMMDD_HHMMSS.md`

**Format (CONCISE):**
```markdown
# [Short Bug Title]

**Date:** YYYY-MM-DD HH:MM

## Problem
[How the problem manifested - 2-3 sentences max]

## Root Cause
[What was the root cause - 2-3 sentences max]

## Fix
[How it was fixed - File:Line references]
```

**IMPORTANT:** Documentation must be short and concise - no prose, only facts.
