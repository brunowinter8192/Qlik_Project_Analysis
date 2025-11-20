# Qlik Syntax: Excel Export Formatting

Ensure clean data types in Excel exports - no "Custom" or "Special" formats!

---

## Overview

**Problem:** Qlik Sense doesn't always transfer correct data types to Excel. Without explicit formatting, fields appear as "Custom" or "Special" format in Excel.

**Solution:** Explicit type formatting in Load Script for every data type.

**Goal:** Clean Excel export with proper types:
- Integer fields → Excel Number format
- Decimal fields → Excel Number format with decimals
- Text fields → Excel Text format
- Date fields → Excel Date format

---

## Integer (Ganzzahlen) - Year, Period

**CRITICAL:** ONLY `Num(..., '0')` works reliably for Excel!

### Correct Pattern

```qlik
// ✅ CORRECT - Excel recognizes as "Number"
Num([Year], '0') as [PCRR.Year],
Num([Period], '0') as [PCRR.Period]
```

**Excel Display:** 2024, 2025, 1, 2, 3, ..., 12

### Common Mistakes

```qlik
// ❌ WRONG - All lead to problems in Excel!

Num([Year])              // → "Custom" format
Num([Year], '#')         // → Scientific notation "2e+03"
Num([Year], '####')      // → "Special" format
Floor([Year])            // → "Custom" format
[Year]                   // → "Custom" format
```

### Why Other Patterns Fail

| Pattern | Result in Excel | Problem |
|---------|----------------|---------|
| `Num([Year])` | "Custom" | No format specified |
| `Num([Year], '#')` | "2e+03" | Scientific notation |
| `Num([Year], '####')` | "Special" | Excel interprets as custom |
| `Floor([Year])` | "Custom" | Function result, no format |
| `[Year]` | "Custom" | No type specified |

**Rule:** For integers, always use `Num([Field], '0')` - nothing else!

---

## Decimal Numbers - Revenue, Costs, Margin

German format with thousand separator (dot) and decimal separator (comma).

### Basic Pattern

```qlik
// German format: 1.234,56
Num([Revenue], '#.##0,00') as [PCRR.Revenue],
Num([Costs], '#.##0,00') as [PCRR.Costs],
Num([Margin], '#.##0,00') as [PCRR.Margin]
```

**Excel Display:** 1.234,56 | 0,00 | -234,78

### Format Pattern Explained

```qlik
Num([Field], '#.##0,00')
```

**Pattern components:**
- `#` = Optional digit (appears only if present)
- `.` = Thousand separator (due to `SET ThousandSep='.'`)
- `0` = Mandatory digit (always appears)
- `,` = Decimal separator (due to `SET DecimalSep=','`)
- `00` = Two decimal places (mandatory)

### Format Variations

```qlik
// With two decimal places (default)
Num([Revenue], '#.##0,00')      // 1.234,56

// Without decimal places
Num([Count], '#.##0')           // 1.234

// Without thousand separator
Num([Small], '0,00')            // 1234,56

// Four decimal places (percentages)
Num([Margin %], '#.##0,0000')   // 12,3456
```

### Examples

| Value | Pattern | Excel Display |
|-------|---------|---------------|
| 1234.56 | `'#.##0,00'` | 1.234,56 |
| 0 | `'#.##0,00'` | 0,00 |
| 1000000 | `'#.##0,00'` | 1.000.000,00 |
| 12.3 | `'#.##0,00'` | 12,30 |
| -234.78 | `'#.##0,00'` | -234,78 |

---

## Text/String - Project ID, Codes

Force string type to prevent numeric interpretation (and loss of leading zeros).

### Basic Pattern

```qlik
// Force string type
Text([Project ID]) as [PCRR.Project ID],
Text([WBS Element]) as [PCRR.WBS Element],
Text([Code]) as [PCRR.Code]
```

### Why Text() is Critical

**Problem:** Numeric IDs lose leading zeros when interpreted as numbers.

**Example:**
```
Source: "000123456"
Without Text(): 123456  ❌
With Text(): "000123456"  ✅
```

### When to Use Text()

Use `Text()` for:
- IDs with leading zeros
- Codes that look numeric (ZIP codes, phone numbers)
- Mixed alphanumeric codes
- Any field where leading/trailing zeros matter

### Pattern with NULL Handling

```qlik
// String with NULL fallback to '-'
If(Len(Trim([Project Description])) > 0 AND NOT IsNull([Project Description]),
   [Project Description],
   '-') as [PCRR.Project Description]
```

---

## Date - Plan Finish, Start Date

**CRITICAL:** Two steps required - parse AND format!

### Basic Pattern

```qlik
// Parse (Date#) AND format (Date)
If(Len(Trim([Plan Finish])) > 0 AND NOT IsNull([Plan Finish]),
   Date(Date#([Plan Finish], 'DD.MM.YYYY'), 'DD.MM.YYYY'),
   '-') as [PCRR.Plan Finish]
```

**Excel Display:** 04.05.2023 (with dots, not slashes!)

### Two Steps Explained

**Step 1: Parse with Date#()**
```qlik
Date#([Plan Finish], 'DD.MM.YYYY')
```
Converts string to internal date number.

**Step 2: Format with Date()**
```qlik
Date(..., 'DD.MM.YYYY')
```
Formats date with dots (German format), not slashes.

### Different Source Formats

```qlik
// Source: DD.MM.YYYY (German)
Date(Date#([Start Date], 'DD.MM.YYYY'), 'DD.MM.YYYY')

// Source: YYYY-MM-DD (ISO)
Date(Date#([Start Date], 'YYYY-MM-DD'), 'DD.MM.YYYY')

// Source: MM/DD/YYYY (US)
Date(Date#([Start Date], 'MM/DD/YYYY'), 'DD.MM.YYYY')

// Source: DD/MM/YYYY (UK)
Date(Date#([Start Date], 'DD/MM/YYYY'), 'DD.MM.YYYY')
```

### Common Mistakes

```qlik
// ❌ WRONG - Only parsing, no output format
Date#([Plan Finish], 'DD.MM.YYYY')
// Result: Slashes in Excel (04/05/2023)

// ❌ WRONG - Only formatting, no parsing
Date([Plan Finish], 'DD.MM.YYYY')
// Result: NULL if source is string

// ✅ CORRECT - Both steps
Date(Date#([Plan Finish], 'DD.MM.YYYY'), 'DD.MM.YYYY')
// Result: 04.05.2023 in Excel
```

---

## Complete Example

Full pattern with all data types and NULL handling.

```qlik
SET ThousandSep='.';
SET DecimalSep=',';

PCRR_Data:
LOAD
    // 1. Integer (Ganzzahlen)
    Num([Year], '0') as [PCRR.Year],
    Num([Period], '0') as [PCRR.Period],

    // 2. Text/String (IDs, Codes)
    Text([Project ID]) as [PCRR.Project ID],
    If(Len(Trim([Project Description])) > 0 AND NOT IsNull([Project Description]),
       [Project Description],
       '-') as [PCRR.Project Description],

    // 3. Date
    If(Len(Trim([Plan Finish])) > 0 AND NOT IsNull([Plan Finish]),
       Date(Date#([Plan Finish], 'DD.MM.YYYY'), 'DD.MM.YYYY'),
       '-') as [PCRR.Plan Finish],

    // 4. Decimal with NULL handling
    Num(If(Len(Trim([Revenue])) > 0 AND NOT IsNull([Revenue]),
           [Revenue], 0), '#.##0,00') as [PCRR.Revenue],

    Num(If(Len(Trim([Costs])) > 0 AND NOT IsNull([Costs]),
           [Costs], 0), '#.##0,00') as [PCRR.Costs]

FROM [lib://DATA Transfer:DataFiles/PCRR*.xlsx]
(ooxml, embedded labels);
```

---

## Format Pattern Reference

Quick reference for all common patterns.

```qlik
// Integer (Ganzzahlen)
Num([Field], '0')                // 2024, 9, 100

// Decimal (Deutsch)
Num([Field], '#.##0,00')         // 1.234,56
Num([Field], '#.##0')            // 1.234 (no decimals)
Num([Field], '0,00')             // 1234,56 (no thousand sep)

// Date (Deutsch)
Date(Date#([Field], 'DD.MM.YYYY'), 'DD.MM.YYYY')  // 04.05.2023
Date(Date#([Field], 'YYYY-MM-DD'), 'DD.MM.YYYY')  // ISO → German
Date(Date#([Field], 'MM/DD/YYYY'), 'DD.MM.YYYY')  // US → German

// Text (Force string)
Text([Field])                    // Prevents numeric interpretation
```

---

## Troubleshooting Excel Export

Common problems and solutions.

| Problem | Cause | Solution |
|---------|-------|----------|
| Year/Period as "Custom" | Wrong format pattern | Use `Num([Year], '0')` |
| Date shows slashes instead of dots | Missing output format | Use `Date(Date#(...), 'DD.MM.YYYY')` |
| Plan Finish shows only "-" | Date not parsed | Use `Date#([Plan Finish], 'DD.MM.YYYY')` first |
| Project ID loses leading zeros | Interpreted as number | Use `Text([Project ID])` |
| Scientific notation "2e+03" | Pattern too short | Use `Num([Year], '0')` not `'#'` or `'####'` |
| Revenue shows as integer | Missing decimal pattern | Use `Num([Revenue], '#.##0,00')` |
| NULL values as blank | Missing NULL handling | Use `If(IsNull(...), 0, ...)` for numeric |
| German decimal separator not working | Missing SET statement | Add `SET DecimalSep=','` at script start |

---

## Time-Limited Fallback Logic

Advanced pattern: Fallback values with temporal constraints.

### Problem

Forecast values should only be used from a certain point in time. Before that, Actual values serve as fallback.

**Requirements:**
- Until August 2025: No Forecast → Use Actual
- From September 2025: No Forecast → 0 (NOT Actual!)

### Wrong Implementation

```qlik
// ❌ PROBLEM: Always fills with Actual!
Alt(
   ApplyMap('Map_Forecast', [Key], Null()),
   [Actual Revenue],  // ← Used even after Sept 2025!
   0
)
```

### Correct Implementation

```qlik
Num(
    Alt(
       ApplyMap('Map_Forecast', [Key], Null()),
       // Time-limited fallback
       If(([Year] < 2025) OR ([Year] = 2025 AND [Period] <= 8),
          [Actual Revenue],  // Only until August 2025
          0)                  // From September 2025
    ),
    '#.##0,00'
) as [Forecast Single]
```

### Logic Table

| Year | Period | Forecast? | Actual | Result | Reason |
|------|--------|-----------|--------|--------|--------|
| 2024 | 12 | ❌ | 1000€ | **1000€** | Actual fallback (before Sept 2025) |
| 2025 | 8 | ❌ | 500€ | **500€** | Actual fallback (before Sept 2025) |
| 2025 | 9 | ❌ | 500€ | **0€** | No fallback anymore! |
| 2025 | 9 | ✅ | 500€ | **Forecast** | Forecast available |
| 2025 | 10 | ❌ | 300€ | **0€** | No fallback anymore! |

---

## Checklist: Excel Export

Use this checklist before and after coding.

### Before Coding

- [ ] Which fields are integers? → `Num(..., '0')`
- [ ] Which fields are decimals? → `Num(..., '#.##0,00')`
- [ ] Which fields are IDs/codes? → `Text(...)`
- [ ] Which fields are dates? → `Date(Date#(...), 'DD.MM.YYYY')`
- [ ] Do I need time-limited fallbacks? → `Alt() + If()`

### After Load

- [ ] Test Excel export
- [ ] Check column formats in Excel (Number? Text? Date?)
- [ ] No "Custom" or "Special" formats?
- [ ] Dates with dots instead of slashes?
- [ ] No leading zeros lost in IDs?
- [ ] Decimal separator is comma?
- [ ] Thousand separator is dot?

---

## SET Statements Required

Always include at script start:

```qlik
SET ThousandSep='.';
SET DecimalSep=',';
```

**Why?** These settings control the format pattern interpretation:
- `.` in pattern becomes thousand separator
- `,` in pattern becomes decimal separator

Without these, patterns like `'#.##0,00'` won't work correctly!

---

## Quick Test Pattern

Use this minimal example to test Excel export formatting:

```qlik
SET ThousandSep='.';
SET DecimalSep=',';

Test_Export:
LOAD * INLINE [
    Year, Period, Revenue, Project_ID, Plan_Finish
    2024, 1, 1234.56, "000123", "04.05.2024"
    2025, 12, 1000000, "999888", "31.12.2025"
];

Test_Export_Formatted:
LOAD
    Num([Year], '0') as [Year],
    Num([Period], '0') as [Period],
    Num([Revenue], '#.##0,00') as [Revenue],
    Text([Project_ID]) as [Project ID],
    Date(Date#([Plan_Finish], 'DD.MM.YYYY'), 'DD.MM.YYYY') as [Plan Finish]
RESIDENT Test_Export;

DROP TABLE Test_Export;
```

**Expected Excel Result:**
- Year: 2024, 2025 (Number format)
- Period: 1, 12 (Number format)
- Revenue: 1.234,56 | 1.000.000,00 (Number format with 2 decimals)
- Project ID: "000123", "999888" (Text format, leading zeros preserved)
- Plan Finish: 04.05.2024, 31.12.2025 (Date format with dots)
