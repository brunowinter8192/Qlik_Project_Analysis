---
description: Systematic Qlik feature implementation with context gathering, qlik-feature-specialist subagents, and documentation
argument-hint: [feature-description]
---

## Feature Request

User requests: $ARGUMENTS

---

## Phase 1: Context Gathering

**CRITICAL:**
- Read the Qlik load scripts BROADLY to understand where this feature should be implemented
- Analyze existing calculations to find similar patterns
- If unclear what the user means, ask clarifying questions before prompting the subagent

1. **Understand Feature Requirements**
   - What calculation/field needs to be added?
   - What business purpose does it serve?
   - What's the expected output format?
   - Any specific formulas or logic requirements?

2. **Analyze Existing Load Scripts**
   - Which .psql scripts have similar calculations? (Test10/PCRR/MST)
   - Which sections contain related logic? (MAPPING/BERECHNUNGEN/HILFSTABELLEN)
   - Are there existing fields this feature depends on?
   - What prefix should be used? ([Table.C.NewField])

3. **Identify Excel Files**
   - Which Excel files contain the source data? (Excels/*.xlsx)
   - Are there multiple Excel files with wildcards?
   - Which sheets and columns are relevant?
   - Is the data already loaded or needs new HAUPTDATEN-LOAD?

4. **Check CLAUDE.md Compliance**
   - Review prefix system requirements
   - Review section ordering (SET-STATEMENTS → HAUPTDATEN → MAPPING → BERECHNUNGEN)
   - Review comment numbering patterns (// 4.X depends on 4.Y)
   - Check Data_Loads/DOCS.md for documentation standards

5. **Testing Workspaces**
   - `Testing/Agent_1/`, `Testing/Agent_2/`, `Testing/Agent_3/` (isolated per agent)

6. **Gather Context**
   - Similar calculations with File:Line numbers
   - Potential mapping dependencies
   - Aggregation patterns
   - Possible implementation locations

---

## Phase 2: Parallel Multi-Agent Feature Development

**CRITICAL:** Launch 3 qlik-feature-specialist agents in parallel to get multiple implementation perspectives.

**MANDATORY:** Each agent gets its own isolated workspace:
- Agent 1 → `Testing/Agent_1/`
- Agent 2 → `Testing/Agent_2/`
- Agent 3 → `Testing/Agent_3/`

Call 3 Task tools **in a single message** (parallel execution):

```json
{
  "tool": "Task",
  "parameters": {
    "description": "implement Qlik feature (Agent 1)",
    "subagent_type": "qlik-feature-specialist",
    "prompt": "## Feature Description\n<Precise description of what needs to be implemented>\n<Expected output format and business purpose>\n\n## Existing Load Scripts Context\n<Which .psql files exist, what they do, similar calculations with File:Line references>\n\n## Excel Files Involved\n<List of Excels/*.xlsx files that contain relevant data>\n<Relevant sheets and columns>\n\n## Investigation Results\n<Findings from Phase 1: similar patterns, potential dependencies, recommended sections>\n\n## Recommended Starting Points\n<Specific sections in Data_Loads/*.psql the subagent should examine first>\n<Which existing fields could be used>\n<Which mappings to consider>\n\n## CRITICAL: Workspace\nYou MUST create all Python scripts in: Testing/Agent_1/\nThis is YOUR isolated workspace. Other agents work in Agent_2/ and Agent_3/\n\n## CRITICAL: Use ACTUAL Excel Files\nYou MUST load actual Excel files from Excels/ folder.\nNO mock data. NO synthetic examples.\nPrototype the feature using the ACTUAL data."
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

## Phase 2.4: Assess Agent Implementation Plans

**CRITICAL:** Agents complete Steps 1+2 (Location Analysis + Prototype) and then report with their implementation plans.

After Phase 2 agents return with their prototype results and implementation plans, assess their approaches:

### Collect Agent Reports

Each agent provides:
1. **Implementation Location Analysis** - Where they propose to implement (Which script? Which section?)
2. **Prototype Results** - Python prototype using ACTUAL Excel files, sample outputs
3. **Implementation Plan** - Qlik syntax approach and planned validation (describe only, not created yet)

### Assessment Criteria

#### 1. Location Assessment

**Main Agent's Role:**
- ❌ NOT: Deep implementation evaluation (Main Agent doesn't have full context)
- ✅ YES: Catch obviously wrong location choices

**Intervention Logic:**

```
IF agent proposes clearly wrong location (violates section order or dependencies):
  REDIRECT: "Your location choice seems problematic. Consider [hint]."

IF all 3 agents agree on SAME location AND SAME implementation approach:
  REDIRECT: "All agents plan to implement in [location] with [method X].
             Agent 2: Try implementing it in [alternative section] instead
             Agent 3: Try implementing it with [method Y] instead"

OTHERWISE:
  LET AGENTS TEST their different approaches
```

**Principle:** Diversify implementation approaches when consensus exists, catch extreme outliers.

#### 2. Data Source Assessment

**CRITICAL:** Don't let agents fuck around with mock data or synthetic examples.

**The Message:**
Prototyping MUST happen with **ACTUAL Excel files** from the **Excels/ folder**.

**RED FLAGS - Agent is jerking off with fake data:**

❌ "I'll test this with mock Excel data"
❌ "Let me create sample values in my script"
❌ "I'll hardcode test data to verify the logic"
❌ "Quick test with pd.DataFrame({'col': [1, 2, 3]})"

**What's ACTUALLY needed:**

✅ Load **actual Excel files** from Excels/ folder
✅ Use **exact sheets and columns** from the real data
✅ Prototype feature with **real data values**, not synthetic bullshit
✅ Validate against **actual data ranges** and values

**Intervention Logic:**

```
IF agent plans to prototype with mock/synthetic data instead of actual Excel files:
  REDIRECT: "Stop testing with mock data. Your prototype needs to use the ACTUAL
             Excel files from Excels/ folder. Load:
             - The exact .xlsx files containing relevant data
             - The actual sheets and columns
             - The real data values, not synthetic examples
             - Validate with actual data ranges and edge cases"
```

**Why?** Because a feature that works with fake data but fails with real data is worthless.

#### 3. Approach Diversity

- Do agents cover different implementation strategies?
- Do they propose different locations or sections?
- If all test same approach: Assign different methods to ensure diversity

#### 4. Plan Quality

- Is the implementation plan comprehensive?
- Does it cover prototype + Qlik syntax + validation?
- Are planned scripts well-structured?
- Is CLAUDE.md compliance addressed?

### Decision & Recommendation Format

For each agent, document:

```
AGENT [X] ASSESSMENT

Implementation Location: [Agent's proposed load script and section]
Location Sanity: ✅ Reasonable / ⚠️ Questionable / ❌ Wrong

Excel Files Used: [List of Excels/*.xlsx files agent loaded]
Data Source Validation: ✅ Used actual Excel files / ❌ Used mock/synthetic data

Prototype: ✅ Feature prototyped successfully / ❌ Issues encountered
Prototype Logic: [How the feature is calculated]
Sample Output: [What results the prototype produces]

Dependencies Identified:
- Required Fields: [List]
- Required Mappings: [List]
- Load Order: [After which scripts/sections?]

Prefix Naming: [Table.C.ProposedFieldName]
Prefix Compliance: ✅ Follows convention / ❌ Violates convention

Planned Implementation: [Brief summary of Qlik syntax approach]
Planned Scripts: [List of implementation scripts agent will create in Phase 2]

DATA STRATEGY:
- Uses Actual Excel Files: ✅/❌
- Loads Correct Sheets/Columns: ✅/❌
- Validates Real Data Ranges: ✅/❌

⚠️ ISSUE: [If using mock data or wrong location, describe]
RECOMMENDATION: [GO / REDIRECT]

OVERLAP CHECK:
- With Agent Y: [HIGH/MEDIUM/LOW overlap - explain]
- With Agent Z: [HIGH/MEDIUM/LOW overlap - explain]

QUALITY: [EXCELLENT/GOOD/NEEDS_IMPROVEMENT - explain]

DECISION: GO / REDIRECT / MERGE
REASONING: [Why this decision]

INSTRUCTIONS TO AGENT:
[If GO: "Proceed with your planned implementation approach"]
[If REDIRECT for location: "Try implementing in [alternative section] instead"]
[If REDIRECT for approach: "Try implementing with [method Y] instead of [method X]"]
[If REDIRECT for data: "Stop testing with mock data. Load actual Excel files from Excels/"]
[If MERGE: "Collaborate with Agent Y to test [combined approach]"]
```

### Overall Recommendation

After assessing all 3 agents:

```
OVERALL ASSESSMENT

Location Consensus: YES/NO - [explanation]
Prototype Success: [X/3 agents successfully prototyped the feature]
Implementation Approach Diversity: EXCELLENT/GOOD/POOR - [explanation]

DATA STRATEGY ASSESSMENT:
- Agent 1: ✅ Using actual Excel files / ❌ Mock data only
- Agent 2: ✅ Using actual Excel files / ❌ Mock data only
- Agent 3: ✅ Using actual Excel files / ❌ Mock data only

AGENTS TO PROCEED:
- Agent 1: [GO/REDIRECT/MERGE] - [brief instruction]
- Agent 2: [GO/REDIRECT/MERGE] - [brief instruction]
- Agent 3: [GO/REDIRECT/MERGE] - [brief instruction]

EXPECTED OUTCOME:
[What implementations/validations you expect from the 3 agents in Phase 2]
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
- Step 1: Implementation Location Analysis
- Step 2: Prototype in Python (created and executed `prototype_feature.py`)
- Step 2.5: Reported to Main Agent

Now they proceed with Phase 2:
- Step 3: Develop Solution (Qlik syntax implementation)
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
    subagent_type="qlik-feature-specialist",
    description="Agent 1 - continue",
    resume=agent_1_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 2
Task(
    subagent_type="qlik-feature-specialist",
    description="Agent 2 - continue",
    resume=agent_2_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)

# Resume Agent 3
Task(
    subagent_type="qlik-feature-specialist",
    description="Agent 3 - continue",
    resume=agent_3_id,  # ← Use extracted ID (HASH ONLY, no "agent-" prefix)
    prompt="## [GO/REDIRECT] - INSTRUCTIONS\n[Your assessment and instructions]"
)
```

**Example of correct usage:**
```python
# If extracted ID is "62b9126b" (correct format from Phase 2.3)
Task(
    subagent_type="qlik-feature-specialist",
    description="Agent 1 - continue",
    resume="62b9126b",  # ✅ Correct
    prompt="..."
)

# NOT like this:
# resume="agent-62b9126b"  # ❌ Wrong - will fail
```

---

## Phase 2.5: Agent Results Aggregation

After all 3 agents complete, analyze their implementations:

### Comparison Criteria
1. **Location Consensus**: Do all 3 agents propose the same implementation location?
2. **Implementation Diversity**: How different are the Qlik syntax approaches?
3. **Python Script Quality**: Which agent wrote the most comprehensive prototype?
4. **Location Justification**: Which agent best explained their location choice?
5. **CLAUDE.md Compliance**: Which solution best maintains prefix system and section order?
6. **Qlik Syntax Correctness**: Which solution has the most robust Qlik implementation?

### Assessment Output
Present to yourself (before showing user):

```
AGENT COMPARISON REPORT
=======================

Location Proposals:
- Agent 1: [load script / section]
- Agent 2: [load script / section]
- Agent 3: [load script / section]
- Consensus: [YES/NO - explain]

Implementation Approaches:
- Agent 1: [approach summary]
- Agent 2: [approach summary]
- Agent 3: [approach summary]

Python Prototype Quality:
- Agent 1: [files created, prototype comprehensiveness]
- Agent 2: [files created, prototype comprehensiveness]
- Agent 3: [files created, prototype comprehensiveness]

Qlik Implementation Quality:
- Agent 1: [syntax correctness, completeness, CLAUDE.md compliance]
- Agent 2: [syntax correctness, completeness, CLAUDE.md compliance]
- Agent 3: [syntax correctness, completeness, CLAUDE.md compliance]

RECOMMENDED IMPLEMENTATION: Agent [X]
REASONING: [Why this implementation is best - consider all criteria above]

REJECTED IMPLEMENTATIONS:
- Agent [Y]: [Why rejected or inferior]
- Agent [Z]: [Why rejected or inferior]
```

---

## Phase 3: Present Multi-Agent Analysis to User

After completing Phase 2.5 aggregation:

1. **Show the Agent Comparison Report** (from Phase 2.5)
2. **Present your recommended implementation** with clear reasoning
3. **Highlight key differences** between the 3 approaches
4. **Show location choices** and why one is preferred
5. **Show Qlik syntax previews** from each agent
6. **WAIT for explicit user confirmation** before proceeding
7. Ask: "Should I implement the recommended solution from Agent [X]?"

**IMPORTANT:** User might choose a different agent's solution than your recommendation. Be prepared to implement their choice.

---

## Phase 4: Implementation (after Approval - Main Agent Only)

**CRITICAL:** Phase 4 is executed by the **MAIN AGENT**, NOT subagents.

Subagents provided implementation plans in their Step 5 reports.
Main Agent (you) reviews those plans and implements them.

### 4.1 Implement Feature (Main Agent executes)
- Apply the approved implementation to Data_Loads/*.psql
- Verify Qlik syntax correctness
- Maintain prefix system ([Table.Field], [Table.C.Field])
- Update comment numbering with dependency annotations
- Ensure section ordering is correct
- Test the changes

---

## Phase 5: Documentation Update (with Approval Gate)

**CRITICAL:** Documentation changes require explicit user approval.

### 5.1 Assess Documentation Impact

Analyze what documentation needs updating:

**Data_Loads/DOCS.md:**
- Does the new feature need to be documented?
- Which load script section descriptions need updates?
- Are new calculated fields explained?

**README.md (if applicable):**
- Does this feature change key metrics tracked?
- Does it affect dashboard overview?
- Does it change data model?

**CLAUDE.md (rarely):**
- Does this feature introduce new patterns?
- Does it affect coding standards?

### 5.2 Present Documentation Plan to User

**APPROVAL GATE:** Wait for explicit user confirmation before updating documentation.

Present:
```
DOCUMENTATION UPDATE PLAN
=========================

Files Requiring Updates:
- Data_Loads/DOCS.md: [What sections need updates]
- README.md: [If applicable - what needs updating]
- CLAUDE.md: [If applicable - what needs updating]

Proposed Changes:
[Show concrete documentation additions/modifications]

AWAITING USER APPROVAL
```

### 5.3 Update Documentation (after Approval)

Only proceed after user explicitly approves.

**Data_Loads/DOCS.md Updates:**
- Add descriptions for new load sections
- Document new calculated fields
- Explain dependencies
- Use prose format (no bullet lists)

**Example:**
```markdown
## PCRR.psql

**Purpose:** Processes Project Cost and Revenue Recognition data...

### Load Sections

The BERECHNUNGEN section calculates derived metrics including the new year-over-year
growth percentage field. This field depends on both current year and prior year revenue
mappings from Test10 and compares them to produce a percentage growth rate. The
calculation handles null values by returning zero when prior year data is unavailable.
```

---

## Phase 6: Completion Summary

Present final summary to user:

```
FEATURE IMPLEMENTATION COMPLETE
================================

Feature Implemented: [Feature name]
Location: [Data_Loads/script.psql / Section]
New Field: [Table.C.NewFieldName]

Changes Made:
- [File:Line] - Added new calculation
- [File:Line] - Updated mapping table
- [File:Line] - Added comment numbering

Documentation Updated:
- Data_Loads/DOCS.md - [What was documented]
- README.md - [If applicable]

CLAUDE.md Compliance: ✅ PASS
- Prefix System: ✅ Maintained
- Section Order: ✅ Correct
- Comment Numbering: ✅ Updated with dependencies
- Mapping Pattern: ✅ Followed

Testing Status:
- Python Prototype: ✅ Validated with actual Excel files
- Edge Cases: ✅ Tested (nulls, zeros, missing data)
- Multiple Files: ✅ Tested with all available Excels

Next Steps:
1. Test in Qlik Sense Data Load Editor
2. Verify calculation results match Python prototype
3. Review dashboard visualizations
4. Validate with user acceptance testing
```
