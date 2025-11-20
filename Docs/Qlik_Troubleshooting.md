# Qlik Troubleshooting

Common Qlik Sense errors, their causes, and solutions.

---

## Field Reference Error (Most Common!)

**Error Message:** `Field '...' not found` even though field IS defined in same LOAD.

### Problem

A field CANNOT be used in the SAME LOAD statement where it's defined!

**Why?** Qlik processes all fields in a LOAD in parallel, NOT sequentially!

### Examples of the Error

```qlik
// ❌ WRONG - "Field 'Profit' not found" Error!
LOAD
    [Revenue] - [Costs] as [Profit],
    [Profit] * 0.19 as [Tax]  // ERROR: [Profit] doesn't exist yet!
RESIDENT Source;

// ❌ WRONG - With ApplyMap
LOAD
    ApplyMap('Map_Value', [ID], 0) as [Field A],
    [Field A] + 100 as [Field B]  // ERROR: [Field A] doesn't exist yet!
RESIDENT Source;

// ❌ WRONG - Nested reference
LOAD
    [Revenue] * 0.2 as [Margin],
    [Margin] / [Revenue] as [Margin %],  // ERROR: [Margin] doesn't exist!
    [Margin %] * 100 as [Margin Display]  // ERROR: [Margin %] doesn't exist!
RESIDENT Source;
```

### Solution 1: Repeat Calculation (Simple Cases)

```qlik
// ✅ CORRECT - Repeat formula
LOAD
    [Revenue] - [Costs] as [Profit],
    ([Revenue] - [Costs]) * 0.19 as [Tax]  // Formula repeated
RESIDENT Source;
```

**When to use:** Simple calculations that are easy to repeat.

### Solution 2: Separate LOAD (Complex Cases)

```qlik
// ✅ CORRECT - Two LOAD steps
Temp:
LOAD
    *,
    [Revenue] - [Costs] as [Profit]
RESIDENT Source;

Final:
LOAD
    *,
    [Profit] * 0.19 as [Tax]  // Now [Profit] exists!
RESIDENT Temp;

DROP TABLE Temp;
```

**When to use:**
- Complex calculations
- Multiple dependent fields
- ApplyMap results used in further calculations

### Solution 3: Repeat ApplyMap (Mapping Cases)

```qlik
// ✅ CORRECT - Repeat ApplyMap
LOAD
    ApplyMap('Map_Value', [ID], 0) as [Field A],
    ApplyMap('Map_Value', [ID], 0) + 100 as [Field B]  // ApplyMap repeated
RESIDENT Source;
```

**When to use:** Simple mappings where repeating ApplyMap is cleaner than separate LOAD.

### How to Spot

**Code Review Pattern:**
- Search for: `... as [NewField],` followed by `[NewField]` in same LOAD
- Error message: `Field '...' not found` even though defined → Check this rule!

**Mnemonic:** "Fields exist only in the NEXT LOAD step!"

---

## Lines Fetched Explosion

**Symptom:** Script runs extremely slow, memory usage spikes, Qlik becomes unresponsive.

**Cause:** JOIN creates cartesian product → 1M rows JOIN 100k rows = 100 billion combinations checked!

### Problem Example

```qlik
// ❌ PROBLEM - Lines Fetched Explosion!
Main_Data:
LOAD * FROM [Revenue.xlsx];  // 1 million rows

LEFT JOIN (Main_Data)
LOAD * FROM [Projects.xlsx];  // 100k rows

// Result: Qlik checks 100 billion combinations!
// Lines fetched: >500k (visible in script log)
```

### Solution: Use MAPPING Instead

```qlik
// ✅ CORRECT - Mapping Pattern
Projects_Data:
LOAD * FROM [Projects.xlsx];

// Create mapping immediately
Map_Projects:
MAPPING LOAD
    [Project ID],
    [Project Name]
RESIDENT Projects_Data;

DROP TABLE Projects_Data;

// Use mapping
Main_Data:
LOAD
    *,
    ApplyMap('Map_Projects', [Project ID], '-') as [Project Name]
FROM [Revenue.xlsx];
```

### Performance Comparison

| Method | Lines Fetched | Memory | Speed |
|--------|---------------|---------|-------|
| JOIN (1M x 100k) | >500k | Very high | Very slow |
| MAPPING (1M rows) | ~1M | Normal | Fast |

### When JOIN is Acceptable

- Small helper tables (< 1000 rows)
- 1:1 relationship guaranteed
- Dimension tables (not fact tables)

### How to Spot

**Script Log Check:**
- Look for "Lines fetched" in script execution log
- If > 500k → Investigate for JOIN operations
- Replace with MAPPING pattern

---

## Synthetic Keys

**Symptom:** Qlik creates hidden synthetic keys, data model becomes unpredictable.

**Cause:** Multiple fields with identical names link tables unintentionally.

### Problem Example

```qlik
// ❌ PROBLEM - Synthetic key created!
Table_A:
LOAD
    [ID],
    [Year],
    [Period],
    [Revenue]
FROM [SourceA.xlsx];

Table_B:
LOAD
    [ID],
    [Year],
    [Period],
    [Costs]
FROM [SourceB.xlsx];

// Qlik creates synthetic key on [ID] + [Year] + [Period]
// Hidden, unpredictable, breaks data model!
```

### Solution: Composite Key

```qlik
// ✅ CORRECT - Explicit composite key
Table_A:
LOAD
    [ID],
    [Year],
    [Period],
    [ID] & '|' & [Year] & '|' & [Period] as [Key],  // Explicit!
    [Revenue]
FROM [SourceA.xlsx];

Table_B:
LOAD
    [ID],
    [Year],
    [Period],
    [ID] & '|' & [Year] & '|' & [Period] as [Key],  // Same key!
    [Costs]
FROM [SourceB.xlsx];

// Tables link via [Key] field only
```

### How to Detect

**Data Model Viewer:**
1. Open Data Model Viewer in Qlik Sense
2. Look for `$Syn` tables (synthetic keys)
3. If found → Identify common fields causing the issue

**Prevention:**
- Use unique field names per table (e.g., `[TableA.Field]`)
- Create explicit composite keys
- Review data model regularly

---

## Excel Export Problems

### Problem 1: Scientific Notation "2e+03"

**Symptom:** Year shows as "2e+03" instead of "2024" in Excel.

**Cause:** Wrong format pattern (using `'#'` or `'####'` instead of `'0'`).

```qlik
// ❌ WRONG
Num([Year], '#')         // → "2e+03" in Excel
Num([Year], '####')      // → Scientific notation

// ✅ CORRECT
Num([Year], '0')         // → "2024" in Excel
```

### Problem 2: Leading Zeros Lost

**Symptom:** Project ID "000123" becomes "123" in Excel.

**Cause:** Field interpreted as number, not text.

```qlik
// ❌ WRONG
[Project ID]             // → 123 (leading zeros lost)
Num([Project ID], '0')   // → Still loses leading zeros

// ✅ CORRECT
Text([Project ID])       // → "000123" (preserved)
```

### Problem 3: Date Shows Slashes Instead of Dots

**Symptom:** Date shows "04/05/2023" instead of "04.05.2023".

**Cause:** Missing output format in Date() function.

```qlik
// ❌ WRONG
Date#([Plan Finish], 'DD.MM.YYYY')  // → "04/05/2023" in Excel

// ✅ CORRECT
Date(Date#([Plan Finish], 'DD.MM.YYYY'), 'DD.MM.YYYY')  // → "04.05.2023"
```

### Problem 4: "Custom" Format in Excel

**Symptom:** Excel shows field as "Custom" or "Special" format.

**Cause:** Missing explicit format in Num() or wrong format pattern.

```qlik
// ❌ WRONG
Num([Revenue])           // → "Custom" in Excel
[Revenue]                // → "Custom" in Excel

// ✅ CORRECT
Num([Revenue], '#.##0,00')  // → "Number" in Excel
```

### Quick Fix Reference

| Problem | Cause | Solution |
|---------|-------|----------|
| Scientific notation | `Num([Year], '#')` | `Num([Year], '0')` |
| Leading zeros lost | Number format | `Text([ID])` |
| Slashes in date | Missing Date() format | `Date(Date#(...), 'DD.MM.YYYY')` |
| "Custom" format | No format specified | `Num(..., '#.##0,00')` |
| Date shows "-" | Date# parsing failed | Check source format pattern |

---

## CROSSTABLE Parameter Forgotten

**Symptom:** First month column missing after CROSSTABLE transformation.

**Cause:** Missing or wrong parameter 3 (number of fixed columns).

### Problem Example

```qlik
// Excel structure:
// Project ID | Sep 25 | Oct 25 | Nov 25
// P001       | 100    | 200    | 300

// ❌ WRONG - Parameter missing or 0
CROSSTABLE([Month], [Value])
LOAD
    [Project ID],
    [Sep 25],
    [Oct 25],
    [Nov 25]
FROM [Source.xlsx];

// Result: Sep 25 is missing! (interpreted as ID column)
```

### Solution

```qlik
// ✅ CORRECT - Parameter 1 (skip Project ID column)
CROSSTABLE([Month], [Value], 1)
LOAD
    [Project ID],
    [Sep 25],
    [Oct 25],
    [Nov 25]
FROM [Source.xlsx];

// Result: All three months present (Sep, Oct, Nov)
```

### How to Spot

**After CROSSTABLE, check:**
1. Count distinct months/periods in result
2. Compare with source Excel columns
3. If one month is missing → Check parameter 3

**Rule:** Parameter 3 = Number of ID columns before pivot data starts.

---

## ORDER BY Missing with Peek()

**Symptom:** Cumulative calculations produce wrong results or unpredictable values.

**Cause:** ORDER BY missing when using Peek().

### Problem Example

```qlik
// ❌ WRONG - No ORDER BY with Peek()
LOAD
    *,
    RangeSum(Peek('Cumulative'), [Value]) as [Cumulative]
RESIDENT Source;  // Row order is random!

// Result: Cumulative values are unpredictable
```

### Solution

```qlik
// ✅ CORRECT - ORDER BY defines sequence
LOAD
    *,
    RangeSum(Peek('Cumulative'), [Value]) as [Cumulative]
RESIDENT Source
ORDER BY [Project ID], [Year], [Period];  // CRITICAL!

// Result: Cumulative values are correct
```

### Rule

**Always use ORDER BY when:**
- Using Peek()
- Cumulative calculations
- Row-dependent logic

---

## ApplyMap Returns Wrong Value

**Symptom:** ApplyMap returns default value even though key exists in mapping table.

### Common Causes

**Cause 1: Data Type Mismatch**
```qlik
// Mapping has string key
Map_Data:
MAPPING LOAD
    Text([ID]),  // String: "123"
    [Value]
RESIDENT Source;

// Lookup uses numeric key
ApplyMap('Map_Data', Num([ID]), 0)  // Numeric: 123 → No match!

// ✅ FIX: Match data types
ApplyMap('Map_Data', Text([ID]), 0)  // String: "123" → Match!
```

**Cause 2: Composite Key Mismatch**
```qlik
// Mapping uses '|' separator
[ID] & '|' & [Year] as Key

// Lookup uses '-' separator
ApplyMap('Map', [ID] & '-' & [Year], 0)  // No match!

// ✅ FIX: Use same separator
ApplyMap('Map', [ID] & '|' & [Year], 0)  // Match!
```

**Cause 3: Leading/Trailing Spaces**
```qlik
// ✅ FIX: Trim both sides
Map_Data:
MAPPING LOAD
    Trim([ID]),  // Remove spaces
    [Value]
RESIDENT Source;

ApplyMap('Map_Data', Trim([ID]), 0)  // Both trimmed → Match!
```

---

## NULL Handling Issues

**Symptom:** Calculations produce NULL instead of expected values.

### Common Causes

**Cause 1: NULL in Arithmetic**
```qlik
// NULL + 100 = NULL (not 100!)
[Field A] + 100  // If Field A is NULL → Result is NULL

// ✅ FIX
If(IsNull([Field A]), 0, [Field A]) + 100  // → 100
```

**Cause 2: ApplyMap Default**
```qlik
// Default 0 prevents Alt() fallback
Alt(
    ApplyMap('Map', [Key], 0),  // Returns 0 if not found
    [Fallback Value]             // Never reached!
)

// ✅ FIX: Use Null() as default
Alt(
    ApplyMap('Map', [Key], Null()),  // Returns Null if not found
    [Fallback Value]                  // Now evaluated!
)
```

---

## Quick Diagnostic Checklist

When script fails or produces wrong results:

1. **Field Reference Error?**
   - [ ] Check for `... as [Field],` followed by `[Field]` in same LOAD
   - [ ] Solution: Repeat calculation or separate LOAD

2. **Performance Problem?**
   - [ ] Check Lines fetched in script log (> 500k?)
   - [ ] Solution: Replace JOIN with MAPPING

3. **Data Model Issue?**
   - [ ] Check for `$Syn` tables (synthetic keys)
   - [ ] Solution: Create explicit composite keys

4. **Excel Export Wrong?**
   - [ ] Use `Num(..., '0')` for integers
   - [ ] Use `Text(...)` for IDs with leading zeros
   - [ ] Use `Date(Date#(...), 'DD.MM.YYYY')` for dates

5. **CROSSTABLE Missing Data?**
   - [ ] Check parameter 3 (number of ID columns)
   - [ ] Solution: `CROSSTABLE([M], [V], 1)` if one ID column

6. **Peek() Wrong Results?**
   - [ ] Check for ORDER BY
   - [ ] Solution: Add `ORDER BY [Sort Fields]`

7. **ApplyMap No Match?**
   - [ ] Check data types (Text vs Num)
   - [ ] Check composite key separator
   - [ ] Trim both sides

8. **NULL Problems?**
   - [ ] Check arithmetic with NULL
   - [ ] Check ApplyMap default (use Null() with Alt())
