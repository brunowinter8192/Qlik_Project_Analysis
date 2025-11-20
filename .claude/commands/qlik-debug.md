---
description: Systematic Qlik calculation debugging with context gathering, qlik-debug-specialist subagents, and bug documentation
argument-hint: [calculation-error-description]
---

## Problem Observation

User observes: $ARGUMENTS

---

## Phase 1: Context Gathering

**CRITICAL:**
- Read the Qlik load scripts BROADLY to understand the user's problem and effectively prompt the subagent
- If unclear what the user means, ask clarifying questions before prompting the subagent

1. **Confirm Load Script Location**
   - Which .psql script? (Test10.psql, PCRR.psql, MST.psql)
   - Which section in the script? (HAUPTDATEN-LOAD, MAPPING-TABELLEN, BERECHNUNGEN, etc.)
   - Which field is problematic? (use prefix notation: [Table.Field] or [Table.C.Field])

2. **Confirm Excel Files**
   - Which Excel files are loaded by this script? (Excels/*.xlsx)
   - Are there multiple Excel files with wildcards (e.g., `Revenue_Tracking*.xlsx`)?
   - Which sheets and columns are relevant?

3. **Create Problem Overview**
   - Expected value vs. Actual value
   - Which calculation step is wrong? (mapping? aggregation? transformation?)
   - Are there dependencies on other tables/mappings?

4. **Check Previous Issues**
   - Check Debug/ folder to see if similar issue existed before
   - Check CLAUDE.md for known calculation patterns

5. **Debug Workspaces**
   - `Debug/Agent_1/`, `Debug/Agent_2/`, `Debug/Agent_3/` (isolated per agent)

6. **Gather Context**
   - Field references with File:Line numbers
   - Mapping table dependencies
   - CROSSTABLE transformations
   - ApplyMap() chains
   - Aggregation logic (GROUP BY, Sum, etc.)

---

## Phase 2: Parallel Multi-Agent Debug

**CRITICAL:** Launch 3 qlik-debug-specialist agents in parallel to get multiple perspectives.

**MANDATORY:** Each agent gets its own isolated workspace:
- Agent 1 → `Debug/Agent_1/`
- Agent 2 → `Debug/Agent_2/`
- Agent 3 → `Debug/Agent_3/`

Call 3 Task tools **in a single message** (parallel execution):

```json
{
  "tool": "Task",
  "parameters": {
    "description": "debug Qlik calculation (Agent 1)",
    "subagent_type": "qlik-debug-specialist",
    "prompt": "## Problem Description\n<Precise description of calculation error: expected vs actual value>\n\n## Load Script Context\n<Which .psql file, which section, which field with File:Line references>\n\n## Excel Files Involved\n<List of Excels/*.xlsx files loaded by this script>\n<Relevant sheets and columns>\n\n## Investigation Results\n<Findings from Phase 1: mapping dependencies, calculation logic, field references>\n\n## Recommended Starting Points\n<Specific sections in Data_Loads/*.psql the subagent should examine first>\n<Which mappings to trace>\n<Which aggregations to check>\n\n## CRITICAL: Workspace\nYou MUST create all Python scripts in: Debug/Agent_1/\nThis is YOUR isolated workspace. Other agents work in Agent_2/ and Agent_3/\n\n## CRITICAL: Use ACTUAL Excel Files\nYou MUST load actual Excel files from Excels/ folder.\nNO mock data. NO synthetic examples.\nReproduce the EXACT calculation using the ACTUAL data."
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
2. **Reproduction Results** - Whether calculation was successfully reproduced using ACTUAL Excel files
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

#### 2. Data Source Assessment

**CRITICAL:** Don't let agents fuck around with mock data or synthetic examples.

**The Message:**
Testing MUST happen with **ACTUAL Excel files** from the **Excels/ folder**.

**RED FLAGS - Agent is jerking off with fake data:**

❌ "I'll test this with mock Excel data"
❌ "Let me create sample values in my script"
❌ "I'll hardcode test data to verify the logic"
❌ "Quick test with pd.DataFrame({'col': [1, 2, 3]})"

**What's ACTUALLY needed:**

✅ Load **actual Excel files** from Excels/ folder
✅ Use **exact sheets and columns** from the real data
✅ Reproduce calculation with **real data values**, not synthetic bullshit
✅ Trace through **actual mapping chains** with real lookup values

**Intervention Logic:**

```
IF agent plans to test with mock/synthetic data instead of actual Excel files:
  REDIRECT: "Stop testing with mock data. Your reproduction needs to use the ACTUAL
             Excel files from Excels/ folder. Load:
             - The exact .xlsx files specified in the load script
             - The actual sheets and columns
             - The real data values, not synthetic examples
             - Trace through actual mapping chains with real lookups"
```

**Why?** Because a calculation that works with fake data but fails with real data is worthless.

#### 3. Approach Diversity

- Do agents cover different solution strategies?
- If all test same approach: Assign different methods to ensure diversity

#### 4. Plan Quality

- Is the debug plan comprehensive?
- Does it cover reproduction + validation with multiple Excel files?
- Are planned scripts well-structured?

### Decision & Recommendation Format

For each agent, document:

```
AGENT [X] ASSESSMENT

Root Cause: [Agent's identified root cause]
Root Cause Sanity: ✅ Reasonable / ⚠️ Questionable / ❌ Absurd

Excel Files Used: [List of Excels/*.xlsx files agent loaded]
Data Source Validation: ✅ Used actual Excel files / ❌ Used mock/synthetic data

Reproduction: ✅ Calculation reproduced / ❌ Could not reproduce
Current Result: [What the script currently calculates]
Expected Result: [What it should calculate]
Discrepancy: [Difference]

Planned Solution: [Brief summary of solution hypothesis]
Planned Scripts: [List of solution scripts agent will create in Phase 2]

DATA STRATEGY:
- Uses Actual Excel Files: ✅/❌
- Loads Correct Sheets/Columns: ✅/❌
- Traces Real Mapping Chains: ✅/❌

⚠️ ISSUE: [If using mock data or absurd root cause, describe]
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
[If REDIRECT for data: "Stop testing with mock data. Load actual Excel files from Excels/"]
[If REDIRECT for root cause: "Your reproduction was good but root cause seems off. Consider [hint]"]
[If MERGE: "Collaborate with Agent Y to test [combined approach]"]
```

### Overall Recommendation

After assessing all 3 agents:

```
OVERALL ASSESSMENT

Root Cause Consensus: YES/NO - [explanation]
Reproduction Success: [X/3 agents successfully reproduced the calculation]
Solution Approach Diversity: EXCELLENT/GOOD/POOR - [explanation]

DATA STRATEGY ASSESSMENT:
- Agent 1: ✅ Using actual Excel files / ❌ Mock data only
- Agent 2: ✅ Using actual Excel files / ❌ Mock data only
- Agent 3: ✅ Using actual Excel files / ❌ Mock data only

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
    subagent_type="qlik-debug-specialist",
    description="Agent 1 - continue",
    resume=agent_1_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 2
Task(
    subagent_type="qlik-debug-specialist",
    description="Agent 2 - continue",
    resume=agent_2_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 3
Task(
    subagent_type="qlik-debug-specialist",
    description="Agent 3 - continue",
    resume=agent_3_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)
```

**Example of correct usage:**
```python
# If extracted ID is "62b9126b" (correct format from Phase 2.3)
Task(
    subagent_type="qlik-debug-specialist",
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
3. **Python Script Quality**: Which agent wrote the most comprehensive reproduction scripts?
4. **Root Cause Analysis**: Which agent went deepest in identifying the calculation error?
5. **CLAUDE.md Compliance**: Which solution maintains prefix system and mapping patterns?
6. **Qlik Syntax Correctness**: Which solution translates Python logic to Qlik syntax correctly?

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

Python Script Quality:
- Agent 1: [files created, reproduction comprehensiveness]
- Agent 2: [files created, reproduction comprehensiveness]
- Agent 3: [files created, reproduction comprehensiveness]

Qlik Implementation Plans:
- Agent 1: [how Python → Qlik translation is planned]
- Agent 2: [how Python → Qlik translation is planned]
- Agent 3: [how Python → Qlik translation is planned]

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
5. **Show Qlik implementation plans** from each agent
6. **WAIT for explicit user confirmation** before proceeding
7. Ask: "Should I implement the recommended fix from Agent [X]?"

**IMPORTANT:** User might choose a different agent's solution than your recommendation. Be prepared to implement their choice.

---

## Phase 4: Implementation and Documentation (after Approval - Main Agent Only)

**CRITICAL:** Phase 4 is executed by the **MAIN AGENT**, NOT subagents.

Subagents provided implementation plans in their Step 5 reports.
Main Agent (you) reviews those plans and implements them.

### 4.1 Implement Fix (Main Agent executes)
- Apply the proposed fix to Data_Loads/*.psql
- Verify Qlik syntax correctness
- Maintain prefix system ([Table.Field], [Table.C.Field])
- Update comment numbering if needed
- Test the changes

### 4.2 Document Bug Fix
**Location:** `Debug/bug_fixes/` (in project root Debug folder)

**Filename:** `[descriptive-name]_YYYYMMDD_HHMMSS.md`

**Format (CONCISE):**
```markdown
# [Short Bug Title]

**Date:** YYYY-MM-DD HH:MM

## Problem
[How the calculation error manifested - 2-3 sentences max]
[Expected vs Actual value]

## Root Cause
[What was the root cause - 2-3 sentences max]
[Which mapping/aggregation/transformation was wrong]

## Fix
[How it was fixed - File:Line references]
[Qlik code changes with BEFORE/AFTER]

## Excel Files
[Which Excels/*.xlsx files were used for validation]
```

**IMPORTANT:** Documentation must be short and concise - no prose, only facts.
