---
description: Iterative feature implementation with exploration, planning, and approval gates
argument-hint: [feature-description]
---

## Feature Request

User wants to implement: $ARGUMENTS

---

## Phase 1: Exploration

**CRITICAL:**
- Use explore-specialist to understand the codebase broadly
- If unclear what the user means, ask clarifying questions before proceeding
- Focus on finding: existing patterns, relevant modules, similar features, architectural conventions

Launch the Task tool with explore-specialist:

```json
{
  "tool": "Task",
  "parameters": {
    "description": "explore codebase for feature context",
    "subagent_type": "explore-specialist",
    "model": "haiku",
    "prompt": "## Feature to Implement\n$ARGUMENTS\n\n## Exploration Goals\n1. Find existing modules that handle similar functionality\n2. Identify architectural patterns used in the codebase\n3. Locate where this feature would logically fit (new module vs extend existing)\n4. Check for relevant dependencies or cross-module interactions\n5. Review CLAUDE.md and src/DOCS.md for project structure\n\n## Output Required\n- Relevant files with File:Line references\n- Similar patterns already in use\n- Recommended location for implementation\n- Architectural considerations"
  }
}
```

---

## Phase 2: Location & Architecture Proposal

After receiving the exploration report:

### 2.1 Analyze Findings
Based on exploration results, determine:
- Should this be a new module in src/ or extend existing module?
- Which existing modules will be affected?
- What cross-module dependencies are needed?

### 2.2 Present Proposal to User

**Location Recommendation:**
```
IMPLEMENTATION LOCATION
=======================

Recommended Approach: [New Module | Extend Existing]

[If New Module:]
- File: src/[module_name].py
- Reason: [Why new module is justified]
- Integrates with: [List affected modules with File:Line]

[If Extend Existing:]
- File: src/[existing_module].py
- Functions to add: [List new function signatures]
- Reason: [Why extending is better than new module]

Affected Files:
- src/[file1].py:[line] - [what changes]
- src/[file2].py:[line] - [what changes]
- workflow.py:[line] - [what changes if entry point affected]
```

**Architectural Approach:**
- Follows existing patterns: [describe patterns from exploration]
- CLAUDE.md compliance: [3-section structure, logging to src/logs/, etc.]
- Cross-module interaction: [how modules will call each other]

### 2.3 Wait for User Approval

**CRITICAL:** WAIT for explicit user confirmation before proceeding.

Ask: "Should I proceed with this location and approach?"

User might want to:
- Choose different location
- Adjust architectural approach
- Ask questions about the proposal

---

## Phase 3: Implementation Planning

**Only after Phase 2 approval.**

### 3.1 Create Detailed Implementation Plan

Break down the feature into specific tasks following CLAUDE.md structure:

```
IMPLEMENTATION PLAN
===================

[If New Module: src/[module_name].py]

INFRASTRUCTURE Section:
- Import statements: [list required imports]
- Constants: [list any constants needed]
- Logging setup: src/logs/[module_name].log

ORCHESTRATOR Section:
- Function: [orchestrator_function_name]([parameters])
- Calls in sequence:
  1. [function_1]() - [purpose]
  2. [function_2]() - [purpose]
  3. [function_3]() - [purpose]

FUNCTIONS Section:
1. [function_1_name]([params]) -> [return_type]
   Purpose: [what it does]
   Logic: [brief description]

2. [function_2_name]([params]) -> [return_type]
   Purpose: [what it does]
   Logic: [brief description]

3. [function_3_name]([params]) -> [return_type]
   Purpose: [what it does]
   Logic: [brief description]

[If Extending Existing Module:]

Modifications to src/[existing_module].py:

INFRASTRUCTURE Updates:
- Add imports: [list new imports]
- Add constants: [list new constants]

ORCHESTRATOR Updates:
- [Modify existing orchestrator OR keep unchanged]

FUNCTIONS to Add:
[Same function breakdown as above]

---

Cross-Module Changes:
- src/[module1].py:[line] - [change description]
- workflow.py:[line] - [change description if needed]

---

Debug/Test Strategy:
- Create: src/debug/test_[feature_name].py
- Test cases: [list key test scenarios]

---

CLAUDE.md Compliance Checklist:
✓ All code in src/ directory
✓ 3-section structure (INFRASTRUCTURE, ORCHESTRATOR, FUNCTIONS)
✓ No emojis in production code
✓ Logging to src/logs/[module].log instead of console prints
✓ Function header comments (1 line, WHAT not HOW)
✓ No inline comments inside function bodies
✓ Cross-module imports with comments: "# From [module].py: [purpose]"
```

### 3.2 Wait for User Approval

**CRITICAL:** WAIT for explicit user confirmation before implementing.

Present the plan and ask: "Should I implement this feature according to the plan?"

User might want to:
- Adjust function breakdown
- Change implementation details
- Add/remove functionality
- Modify cross-module interactions

---

## Phase 4: Implementation

**Only after Phase 3 approval.**

### 4.1 Execute Implementation

Follow the approved plan step-by-step:

1. **Create/Modify Module File(s)**
   - Write INFRASTRUCTURE section first
   - Write ORCHESTRATOR section (calls only, no logic)
   - Write FUNCTIONS section (ordered by call sequence)
   - Include function header comments (1 line each)
   - Add cross-module import comments where needed

2. **Update Cross-Module Dependencies**
   - Modify affected files (orchestrators that need to call new functions)
   - Add imports with proper comments
   - Update workflow.py if entry point affected

3. **Create Debug/Test Script**
   - File: src/debug/test_[feature_name].py
   - Include test cases from plan
   - Use emojis allowed in debug scripts

4. **Test the Implementation**
   - Run debug script to verify functionality
   - Check that feature works as expected
   - Fix any issues found

### 4.5 Validate Scraper Quality (If Scraper Modified)

**CRITICAL:** If the feature modified `src/scraper/scrape_url.py`, validate quality impact.

**Detection:**
- Check if `src/scraper/scrape_url.py` was changed in the implementation
- If YES: Proceed with validation
- If NO: Skip to Phase 4.2

**Launch tester-specialist agent:**

Use the Task tool to run quality validation:

```json
{
  "tool": "Task",
  "parameters": {
    "description": "validate scraper quality after feature",
    "subagent_type": "tester-specialist",
    "model": "haiku",
    "prompt": "## Context\nNew feature implemented in src/scraper/scrape_url.py.\n\n## Feature Description\n[Brief description of what feature was added]\n\n## Task\nRun the scraping suite baseline and comparison to validate the feature improves content extraction quality and doesn't introduce regressions in other domains.\n\n## Required Analysis\n1. Execute baseline suite\n2. Compare with previous iteration\n3. Verify improvements appear in expected domains\n4. Check for unintended side effects in other domains\n5. Analyze character/word count changes\n6. Review content diffs\n7. Assess: EXPECTED (feature works as intended) / UNEXPECTED (side effects) / MAJOR (>20% changes)\n\n## Required Output\n- Domain-by-domain analysis with concrete evidence\n- Feature improvement verification\n- Regression detection\n- Overall assessment (EXPECTED/UNEXPECTED/MAJOR)\n- Clear recommendation"
  }
}
```

**Wait for validation report.**

**Decision Gates:**
- **EXPECTED**: Feature improvements confirmed, no unexpected regressions → Proceed to Phase 4.2
- **UNEXPECTED**: Side effects detected → Present findings to user, may need refinement
- **MAJOR**: Large changes (>20%) → Review with user to ensure intended

**If UNEXPECTED or requires refinement:**
1. Review tester-specialist report to understand issues
2. Refine implementation based on findings
3. Re-test with debug script
4. Re-run Phase 4.5 validation
5. Iterate until EXPECTED or user approves

### 4.2 Report Implementation Status

After implementation:
```
IMPLEMENTATION COMPLETE
=======================

Files Created/Modified:
✓ src/[module].py - [brief description]
✓ src/[other_module].py:[lines] - [what changed]
✓ src/debug/test_[feature].py - [test coverage]

Test Results:
[Output from running debug script]

Next Step: Documentation Update
```

---

## Phase 5: Documentation Update

### 5.1 Assess Documentation Impact

Analyze what documentation needs updating:

**src/DOCS.md:**
- If new module created: Add ## [module_name].py section with all functions
- If extended existing: Update existing ## [module_name].py section with new functions
- Follow prose description format (no bullet lists for function descriptions)

**README.md:**
- Update only if workflow entry point changed
- Update Basic Usage section if feature affects how user runs workflow.py

### 5.2 Present Documentation Changes

Show proposed documentation updates to user with format:

```
DOCUMENTATION UPDATES REQUIRED
===============================

src/DOCS.md:
[Show exactly what will be added/modified]

README.md:
[UPDATE - show changes] OR [NO UPDATE - explain why not]
```

### 5.3 Wait for User Approval

**CRITICAL:** WAIT for explicit user confirmation before updating docs.

Ask: "Should I update the documentation as shown above?"

### 5.4 Execute Documentation Updates

After approval:
- Update src/DOCS.md
- Update README.md if needed
- Ensure proper formatting (prose, no emojis, File:Line references)

---

## Phase 6: Completion Summary

Present final summary:

```
FEATURE IMPLEMENTATION SUMMARY
==============================

Feature: [feature description]

Implementation:
✓ Location: src/[module].py
✓ Functions added: [count]
✓ Cross-module changes: [count files]
✓ Test coverage: src/debug/test_[feature].py

Documentation:
✓ src/DOCS.md updated
✓ README.md [updated/unchanged]

CLAUDE.md Compliance: ✓ All standards followed

Next Steps:
- Review the implementation
- Run workflow.py to test integration
- Commit changes when satisfied
```

---

**IMPORTANT NOTES:**

1. **Approval Gates:** This command has 3 explicit approval gates:
   - After Phase 2 (location/approach)
   - After Phase 3 (implementation plan)
   - After Phase 5.3 (documentation)

2. **Iterative Process:** User can request changes at any approval gate. Be flexible.

3. **CLAUDE.md Compliance:** Every implementation MUST follow project standards strictly.

4. **Cross-Module Awareness:** Always consider how new feature integrates with existing modules.

5. **Documentation Sync:** Keep DOCS.md in sync with code changes immediately.
