# Qlik Syntax: Core Functions

Essential Qlik Sense functions and their correct syntax patterns.

---

## Match() - Alternative to SQL IN Operator

**Problem:** SQL `IN` operator is NOT supported in Qlik Sense.

**Syntax:**
```qlik
Match(FieldName, Value1, Value2, Value3, ...)
```

### Basic Pattern

```qlik
// ❌ WRONG - IN Operator (Syntax Error!)
WHERE [Period] IN (9, 10, 11, 12)

// ✅ CORRECT - Match() Function
WHERE Match([Period], 9, 10, 11, 12)
```

### Return Value

Match() returns:
- Position of match (1, 2, 3, ...) if found
- 0 if not found
- Works in WHERE clause (0 = false, >0 = true)

### Examples

```qlik
// Match strings
WHERE Match([Project Status], 'Active', 'In Progress', 'Pending')

// Match numbers
WHERE Match([Year], 2023, 2024, 2025)

// Match with variables
WHERE Match([Department], $(vDept1), $(vDept2))

// Match in calculations
If(Match([Status], 'Completed', 'Closed'), 'Done', 'Open') as [Status Category]
```

---

## Peek() - Access Previous Row

**CRITICAL:** String parameter uses NO square brackets!

### Basic Syntax

```qlik
// ❌ WRONG - Square brackets in string
Peek('[FieldName]')

// ✅ CORRECT - No square brackets in string
Peek('FieldName')
```

### Why No Brackets?

The string is already text - square brackets would be interpreted as part of the field name itself.

### Common Usage

```qlik
// Cumulative sum with Peek()
LOAD
    *,
    RangeSum(Peek('Cumulative Revenue'), [Current Revenue]) as [Cumulative Revenue]
RESIDENT Source
ORDER BY [Project ID], [Year], [Period];  // ORDER BY is MANDATORY!
```

**CRITICAL:** ORDER BY is required when using Peek() - it defines which row is "previous"!

### Examples

```qlik
// Reset cumulative on new project
If([Project ID] = Peek('Project ID'),
   RangeSum(Peek('Cumulative'), [Value]),
   [Value]) as [Cumulative]

// Carry forward previous value if NULL
If(IsNull([Current Value]),
   Peek('Field Name'),
   [Current Value]) as [Field Name]

// Calculate delta from previous row
[Revenue] - Peek('Revenue') as [Revenue Change]
```

### Peek() with Calculated Fields

```qlik
// For fields with .C prefix - still no brackets in string!
Peek('PCRR.C.Cumulative POC')
Peek('Test10.C.Forecast YTD')
```

**Rule:** Field names in string form (e.g., `'FieldName'`) NEVER use square brackets, even when the field itself is written with brackets (e.g., `[FieldName]`).

---

## ApplyMap() - Use Mapping Tables

**CRITICAL:** String parameters use NO square brackets!

### Basic Syntax

```qlik
ApplyMap('MappingTableName', KeyField, DefaultValue)
```

### Correct Usage

```qlik
// ❌ WRONG - Brackets in string
ApplyMap('MapName', '[KeyField]')

// ✅ CORRECT - No brackets in string
ApplyMap('Map_ProjectInfo', [Project ID], '-') as [Project Description]
```

### Default Values

Choose appropriate defaults based on field type:
- Numeric fields → `0`
- String fields → `'-'`
- Special case: Use `Null()` when using Alt() for fallback chains

### Examples

```qlik
// Simple mapping with default
ApplyMap('Map_Revenue', [Project ID], 0) as [Total Revenue]

// Composite key mapping
ApplyMap('Map_Forecast',
         [Project ID] & '|' & [Year] & '|' & [Period],
         Null()) as [Forecast Value]

// Mapping with Alt() fallback
Alt(
    ApplyMap('Map_Primary', [ID], Null()),
    ApplyMap('Map_Secondary', [ID], Null()),
    0
) as [Final Value]
```

### Common Mistake: Default 0 with Alt()

```qlik
// ❌ WRONG - Alt() never reaches second parameter!
Alt(
    ApplyMap('Map_Forecast', [Key], 0),  // Returns 0 if not found
    [Actual Value]                        // Never evaluated!
)

// ✅ CORRECT - Use Null() as default
Alt(
    ApplyMap('Map_Forecast', [Key], Null()),  // Returns Null if not found
    [Actual Value]                             // Evaluated when Null
)
```

---

## Alt() - Fallback Chain

Use first non-NULL value from parameter list.

### Basic Syntax

```qlik
Alt(Expression1, Expression2, Expression3, ...)
```

### Behavior

- Evaluates expressions left to right
- Returns first non-NULL value
- Useful for cascading fallbacks

### Examples

```qlik
// Two-level fallback
Alt([Forecast Value], [Actual Value]) as [Best Estimate]

// Three-level fallback with final default
Alt(
    [Forecast Single],
    [Forecast Aggregate],
    0
) as [Final Forecast]

// Time-limited fallback
Alt(
    ApplyMap('Map_Forecast', [Key], Null()),
    If([Year] < 2025 OR ([Year] = 2025 AND [Period] <= 8),
       [Actual Value],
       0)
) as [Forecast with Fallback]
```

### Combining with ApplyMap

```qlik
// Cascade through multiple mapping sources
Alt(
    ApplyMap('Map_Source1', [ID], Null()),  // Try first source
    ApplyMap('Map_Source2', [ID], Null()),  // Try second source
    ApplyMap('Map_Source3', [ID], Null()),  // Try third source
    '-'                                      // Final default
) as [Description]
```

---

## If() vs Alt()

Understanding when to use each:

### If() - Conditional Logic

```qlik
// Use If() for boolean conditions
If([Revenue] > 1000000, 'Large', 'Small') as [Size Category]

// Use If() for complex conditions
If([Year] = 2025 AND [Period] <= 8, [Value], 0) as [Limited Value]
```

### Alt() - NULL Handling

```qlik
// Use Alt() for NULL fallbacks
Alt([Primary], [Secondary], [Tertiary]) as [Best Value]

// NOT ideal for conditions (use If instead)
Alt(
    If([Revenue] > 0, [Revenue], Null()),  // Complicated
    0
)

// Better with If()
If([Revenue] > 0, [Revenue], 0) as [Safe Revenue]
```

---

## Quick Reference

### String Parameters (NO Brackets!)

```qlik
✅ Peek('FieldName')
✅ ApplyMap('MapName', KeyField, Default)
✅ Exists('FieldName', Value)

❌ Peek('[FieldName]')
❌ ApplyMap('[MapName]', '[KeyField]')
```

### Default Values

```qlik
// Numeric
ApplyMap('Map', [Key], 0)

// String
ApplyMap('Map', [Key], '-')

// With Alt() fallback
ApplyMap('Map', [Key], Null())
```

### ORDER BY Requirement

```qlik
// Peek() requires ORDER BY
LOAD
    Peek('FieldName')
RESIDENT Source
ORDER BY [Sort1], [Sort2];  // MANDATORY!

// ApplyMap() does NOT require ORDER BY
LOAD
    ApplyMap('Map', [Key], 0)
RESIDENT Source;  // ORDER BY optional
```
