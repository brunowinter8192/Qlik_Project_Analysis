---
description: Assess if documentation needs updating after debug/feature workflows. Checks README, Data_Loads/DOCS, Docs/Qlik_Syntax_*, and CLAUDE.md against session changes.
argument-hint: [optional: specific documentation concern]
---

## Context

User invoked: `/qlik-check-docs $ARGUMENTS`

**Purpose:** Determine if documentation needs updating after code changes in debug or feature implementation workflows.

**Philosophy:** Documentation updates are OPTIONAL - only when "WHAT" changes, not "HOW". When uncertain, ASK THE USER with recommendation.

---

## COMPREHENSIVE REVIEW PROTOCOL

### Phase 1: Analyze Changes

**List all files modified in this session:**
- Which .psql scripts in Data_Loads/ were changed?
- What was the nature of changes in each script?
- Were new fields added? Were calculations modified? Were mappings changed?

**Identify WHAT vs HOW:**
- **WHAT changed:** Function purpose, script responsibility, data flow, field interface
- **HOW changed:** Implementation details, bug fixes, performance optimizations, refactoring

**IMPORTANT:** Only WHAT changes trigger documentation updates. HOW changes do NOT.

---

### Phase 2: Impact Assessment

Assess impact on 4 documentation concerns:

#### 2.1 README.md Impact Assessment

**Check if these changed:**
1. Were new data sources added to Excels/?
2. Did load execution order change (Test10 → PCRR → MST)?
3. Did business metrics or KPIs tracked change?
4. Did dashboard functionality or visualizations change?
5. Did project purpose or target audience change?

**Document Decision:**
```
FILE: README.md
CHANGE: [Brief description of what changed in code]
SECTION: [Which README section would be affected]
DECISION: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
REASON: [Explain why WHAT changed or why only HOW changed]
ACTION: [If update required, what specifically to update]
RECOMMENDATION: [If uncertain, formulate question + YES/NO recommendation]
```

---

#### 2.2 Data_Loads/DOCS.md Impact Assessment

**Check if these changed:**
1. Was a new .psql script added?
2. Did script PURPOSE change (WHAT it does, not HOW it does it)?
3. Did INPUT files (Excel sources) or OUTPUT tables change?
4. Did load section flow change (new MAPPING? new HILFSTABELLE?)?
5. Did helper table responsibilities change?
6. Did mapping table usage patterns change?

**Document Decision:**
```
FILE: Data_Loads/DOCS.md
CHANGE: [Brief description of what changed in code]
SECTION: [Which script documentation: Test10.psql / PCRR.psql / MST.psql]
DECISION: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
REASON: [Explain why WHAT changed or why only HOW changed]
ACTION: [If update required, what specifically to update]
RECOMMENDATION: [If uncertain, formulate question + YES/NO recommendation]
```

---

#### 2.3 Docs/Qlik_Syntax_*.md Impact Assessment

**CRITICAL:** This is unique to Qlik - check if code changes relate to documented syntax patterns.

**Identify which syntax file relates to the change:**

**Docs/Qlik_Syntax_Core_Functions.md:**
- ApplyMap() usage patterns
- Match() instead of IN
- Peek() field references
- Alt() fallback patterns
- Combinations: Alt() + ApplyMap(), nested functions

**Docs/Qlik_Syntax_Transformations.md:**
- CROSSTABLE unpivot patterns
- FileList() for multiple Excel files
- CONCATENATE with NoConcatenate
- Duplicate prevention patterns

**Docs/Qlik_Syntax_Advanced.md:**
- QUALIFY for field name conflicts
- ORDER BY requirements
- DISTINCT vs WHERE NOT EXISTS
- Data volume considerations (FROM vs RESIDENT)

**Docs/Qlik_Syntax_Excel_Export.md:**
- Num() formatting patterns (Year, decimals, IDs)
- Text() formatting patterns (leading zeros)
- Date formatting patterns

**Docs/Qlik_Troubleshooting.md:**
- "Field not found" errors
- Synthetic keys
- ApplyMap default value issues
- Performance issues (Lines Fetched)

**For EACH relevant syntax file, document decision:**
```
FILE: Docs/Qlik_Syntax_[specific].md
CHANGE: [What syntax pattern was used in code]
SECTION: [Which section in the syntax file]
DECISION: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
REASON: [Is this a NEW pattern? Edge case discovered? Rule clarified?]
ACTION: [If update required, what example/rule to add]
RECOMMENDATION: [If uncertain, formulate question + YES/NO recommendation]
```

**Examples of when to update Docs/Qlik_Syntax_*.md:**
- ✅ Discovered Alt() + ApplyMap() pattern not documented
- ✅ Found solution to "Field not found" edge case
- ✅ Learned new Num() formatting pattern for Excel export
- ✅ Discovered ORDER BY requirement for Peek() in mapping
- ❌ Used existing ApplyMap() pattern as documented
- ❌ Fixed bug in existing calculation (HOW, not WHAT)

---

#### 2.4 CLAUDE.md Impact Assessment

**Check if these changed:**
1. Was a new structural pattern established (new section type)?
2. Was a new compliance rule added (prefix system extension)?
3. Did comment numbering pattern change?
4. Did section ordering rules change?
5. Were new coding standards established?

**Document Decision:**
```
FILE: CLAUDE.md
CHANGE: [Brief description of what changed in code]
SECTION: [Which CLAUDE.md section: PREFIX SYSTEM / COMMENT NUMBERING / etc.]
DECISION: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
REASON: [Explain why WHAT changed or why only HOW changed]
ACTION: [If update required, what specifically to update]
RECOMMENDATION: [If uncertain, formulate question + YES/NO recommendation]
```

---

### Phase 3: Consolidated Recommendation

Present a single consolidated overview to the user.

**Format:**

```markdown
DOCUMENTATION ASSESSMENT
========================

SESSION CHANGES SUMMARY:
[Brief bullet list of what changed in this session]

IMPACT ASSESSMENT:

README.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

Data_Loads/DOCS.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ Section: [Which script]
└─ [Brief reasoning]

Docs/Qlik_Syntax_Core_Functions.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ Section: [If applicable]
└─ [Brief reasoning]

Docs/Qlik_Syntax_Transformations.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

Docs/Qlik_Syntax_Advanced.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

Docs/Qlik_Syntax_Excel_Export.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

Docs/Qlik_Troubleshooting.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

CLAUDE.md: UPDATE REQUIRED / NO UPDATE / UNCERTAIN
└─ [Brief reasoning]

───────────────────────────────────────────────

CONSOLIDATED RECOMMENDATION:

Files requiring updates:
- [List files that need UPDATE REQUIRED]

Files recommended for updates:
- [List files that are UNCERTAIN with recommendations]

QUESTION (if any UNCERTAIN):
[Formulate specific question about uncertain updates]

RECOMMENDATION: [YES/NO for each uncertain file with brief reasoning]

───────────────────────────────────────────────

Proceed with documentation updates?
```

**IMPORTANT:**
- Only present ONE consolidated recommendation, not per-file recommendations
- If all are "NO UPDATE", state clearly and ask if user wants to review anyway
- If some are UNCERTAIN, formulate a SINGLE question covering all uncertainties

---

### Phase 4: Wait for User Approval

**DO NOT proceed to documentation updates without explicit user approval.**

Wait for user response:
- "Yes" / "Proceed" / "Update" → Go to Phase 5
- "No" / "Skip" → End workflow
- Specific guidance → Adjust plan and confirm again

---

### Phase 5: Execute Updates (after Approval)

Only execute if user approved in Phase 4.

For each file marked UPDATE REQUIRED or approved UNCERTAIN:

**5.1 Read Current Documentation**
- Read the current state of the documentation file

**5.2 Identify Exact Location**
- Find the exact section that needs updating
- Identify line numbers if possible

**5.3 Propose Specific Changes**
- Show BEFORE (current text)
- Show AFTER (proposed updated text)
- Explain reasoning for each change

**5.4 Apply Updates**
- Use Edit tool to update documentation
- Maintain documentation style:
  - README.md: Project-specific information, business focus
  - Data_Loads/DOCS.md: Prose format, no bullet lists
  - Docs/Qlik_Syntax_*.md: Code examples with explanations
  - CLAUDE.md: Technical standards, concise rules

**5.5 Verify Updates**
- Read updated documentation to verify changes applied correctly
- Check formatting is consistent

---

### Phase 6: Summary

Present summary of completed updates:

```markdown
DOCUMENTATION UPDATES COMPLETE
==============================

Updated Files:
- README.md: [What was updated]
- Data_Loads/DOCS.md: [What was updated]
- Docs/Qlik_Syntax_[specific].md: [What was updated]
- CLAUDE.md: [What was updated]

Skipped Files (no update needed):
- [List files that didn't need updates]

All documentation is now synchronized with code changes.
```

---

## Decision Framework Reference

Use this framework to determine UPDATE REQUIRED vs NO UPDATE:

### UPDATE REQUIRED (WHAT changed)

**README.md:**
- New data source added
- Load execution order changed
- Business metrics tracked changed
- Dashboard functionality changed
- Project purpose changed

**Data_Loads/DOCS.md:**
- New .psql script added
- Script PURPOSE changed (WHAT it does)
- INPUT/OUTPUT contracts changed
- Load section flow changed
- Mapping/Helper table responsibilities changed

**Docs/Qlik_Syntax_*.md:**
- New Qlik function usage pattern discovered
- Edge case solution found
- Syntax rule clarified or corrected
- New troubleshooting pattern discovered
- New example needed to clarify usage

**CLAUDE.md:**
- New structural pattern established
- New compliance rule added
- Prefix system extended
- Comment numbering pattern changed
- Section ordering rules changed

### NO UPDATE (HOW changed)

**All Documentation:**
- Bug fixes in existing calculations
- Performance optimizations
- Code refactoring
- Implementation details changed
- Internal variable naming changed
- Comment wording improved

### UNCERTAIN

**When to be uncertain:**
- Edge case discovered but unclear if it's common enough to document
- Pattern used might be worth documenting but unsure if it's a best practice
- Syntax clarification might be helpful but unsure if it's significant
- Change is borderline between WHAT and HOW

**When uncertain, ALWAYS:**
1. Formulate specific question
2. Provide YES/NO recommendation with reasoning
3. Let user decide

---

## Usage Examples

**After debug workflow:**
```bash
/qlik-debug "PCRR.C.Forecast Single shows wrong value"
[debug workflow completes with fix]
/qlik-check-docs
```

**After feature workflow:**
```bash
/qlik-feature "Add year-over-year growth percentage field"
[feature workflow completes with implementation]
/qlik-check-docs
```

**With specific concern:**
```bash
/qlik-check-docs "Check if ApplyMap pattern should be documented"
```

---

## Important Reminders

- **Documentation is OPTIONAL** - Only update when WHAT changes, not HOW
- **When uncertain, ASK** - Provide recommendation, let user decide
- **One consolidated recommendation** - Not per-file mini-recommendations
- **Wait for approval** - Never update documentation without user confirmation
- **Maintain style** - Follow existing documentation patterns and tone
