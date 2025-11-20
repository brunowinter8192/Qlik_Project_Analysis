# Qlik Syntax: Data Transformations

Data transformation patterns for loading and reshaping data in Qlik Sense.

---

## CROSSTABLE - Pivot to Long Format

Transform pivot tables (months/periods as columns) into long format (row-based).

### Problem

Excel pivot structure with periods as columns must be converted to long format for time-series analysis.

**Excel Structure:**
```
Project ID | Jan | Feb | Mar
P001       | 100 | 200 | 150
P002       | 300 | 250 | 400
```

**Target (Long Format):**
```
Project ID | Month | Value
P001       | Jan   | 100
P001       | Feb   | 200
P001       | Mar   | 150
```

### Basic Syntax

```qlik
CROSSTABLE(MonthField, ValueField, NumberOfFixedColumns)
LOAD
    [Project ID],
    Jan,
    Feb,
    Mar
FROM [lib://DATA/Pivoted_Data.xlsx] (ooxml, embedded labels);
```

**Parameters:**
- `MonthField` - Name of new column containing month/period names
- `ValueField` - Name of new column containing values
- `NumberOfFixedColumns` - Number of ID columns to skip (before pivot columns start)

### CRITICAL: Understanding Parameter 3

**Without Parameter 3 (WRONG):**
```qlik
CROSSTABLE([TempMonth], [TempValue])  // ❌ ERROR!
```
Result: First data column (Sep 25) is interpreted as ID column → missing in result!

**With Parameter 3 (CORRECT):**
```qlik
CROSSTABLE([TempMonth], [TempValue], 1)  // ✅ CORRECT!
```
Result: ID column (Ident/Project ID) is skipped, ALL month columns are transformed

### Real Example from Test10.psql

**Excel Structure:**
```
Ident    | Sep 25 | Oct 25 | Nov 25 | Dec 25
Project1 | 100    | 200    | 300    | 400
Project2 | 150    | 250    | 350    | 450
```

**Code:**
```qlik
Test10_Raw:
CROSSTABLE([TempMonth], [TempValue], 1)  // Skip 1 column (Ident)
LOAD
    [Project ID],
    [Sep 25],
    [Okt 25],
    [Nov 25],
    [Dez 25]
FROM [lib://DATA Transfer:DataFiles/Test10_PLAN_IST*.xlsx]
(ooxml, embedded labels);
```

**Result:**
```
Project ID | TempMonth | TempValue
Project1   | Sep 25    | 100
Project1   | Okt 25    | 200
Project1   | Nov 25    | 300
Project1   | Dez 25    | 400
```

### Parsing Month Names to Period Numbers

After CROSSTABLE, month names need to be parsed into period numbers (1-12).

**Option 1: Match() Function (German Month Names)**
```qlik
Test10_Data:
LOAD
    [Project ID],
    // Parse month from "Sep 25" → 9
    Num(
        Match(Left(TempMonth, 3), 'Jan', 'Feb', 'Mrz', 'Apr', 'Mai', 'Jun',
                                  'Jul', 'Aug', 'Sep', 'Okt', 'Nov', 'Dez'),
        '0'
    ) as [Period],
    // Parse year from "Sep 25" → 2025
    Num('20' & Right(TempMonth, 2), '0') as [Year],
    TempValue as [Revenue]
RESIDENT Test10_Raw;

DROP TABLE Test10_Raw;
```

**Option 2: Mapping Table (Any Month Format)**
```qlik
Map_MonthToPeriod:
MAPPING LOAD * INLINE [
    Month, Period
    Jan, 1
    Feb, 2
    Mar, 3
    Apr, 4
    May, 5
    Jun, 6
    Jul, 7
    Aug, 8
    Sep, 9
    Oct, 10
    Nov, 11
    Dec, 12
];

// Then in LOAD:
ApplyMap('Map_MonthToPeriod', Left(TempMonth, 3), 0) as [Period]
```

### Multiple Fixed Columns

If Excel has multiple ID columns before pivot data:

**Excel Structure:**
```
Project ID | Year | Jan | Feb | Mar
P001       | 2024 | 100 | 200 | 150
P002       | 2024 | 300 | 250 | 400
```

**Code:**
```qlik
CROSSTABLE([Month], [Revenue], 2)  // Skip 2 columns (Project ID, Year)
LOAD
    [Project ID],
    [Year],
    Jan,
    Feb,
    Mar
FROM Source;
```

---

## FOR EACH - Loop Through Multiple Files

Load and concatenate multiple files with identical structure.

### Basic Pattern

```qlik
FOR EACH vFile IN FileList('lib://DATA Transfer:DataFiles/Revenue*')
    IF NoOfRows('PCRR_Data') = 0 OR IsNull(TableNumber('PCRR_Data')) THEN
        PCRR_Data:
        LOAD
            Text([Project ID]) as [PCRR.Project ID],
            Num([Year], '0') as [PCRR.Year],
            Num([Revenue], '#.##0,00') as [PCRR.Revenue]
        FROM [$(vFile)] (ooxml, embedded labels);
    ELSE
        CONCATENATE (PCRR_Data)  // EXPLICIT!
        LOAD
            Text([Project ID]) as [PCRR.Project ID],
            Num([Year], '0') as [PCRR.Year],
            Num([Revenue], '#.##0,00') as [PCRR.Revenue]
        FROM [$(vFile)] (ooxml, embedded labels);
    END IF
NEXT vFile;
```

### Critical Points

1. **Variable Naming:** Use "v" prefix (`vFile`, not `File`)
2. **IF-Check:** Prevents empty table on first iteration
3. **CONCATENATE Explicit:** Always write `CONCATENATE (TableName)`
4. **Identical LOAD Blocks:** Code must be identical in both IF/ELSE branches (Qlik limitation)

### FileList Patterns

```qlik
// All files starting with "Revenue"
FileList('lib://DATA Transfer:DataFiles/Revenue*')

// Files with specific date pattern
FileList('lib://DATA Transfer:DataFiles/Report_202501*.xlsx')

// All Excel files in folder
FileList('lib://DATA Transfer:DataFiles/*.xlsx')
```

### Why IF-Check is Necessary

**Without IF-Check (WRONG):**
```qlik
FOR EACH vFile IN FileList('...')
    CONCATENATE (Data_Table)  // ❌ ERROR on first iteration!
    LOAD * FROM [$(vFile)];
NEXT vFile;
```

First iteration fails because `Data_Table` doesn't exist yet!

**With IF-Check (CORRECT):**
```qlik
FOR EACH vFile IN FileList('...')
    IF NoOfRows('Data_Table') = 0 OR IsNull(TableNumber('Data_Table')) THEN
        Data_Table:  // Create table
        LOAD * FROM [$(vFile)];
    ELSE
        CONCATENATE (Data_Table)  // Append to existing table
        LOAD * FROM [$(vFile)];
    END IF
NEXT vFile;
```

---

## WHERE NOT EXISTS - Avoid Duplicates

Prevent duplicate records when concatenating tables from multiple sources.

### Problem

When creating a master table from multiple sources, duplicates occur if the same combination exists in both sources.

### Pattern

```qlik
// 1. Load base from primary source
Master:
LOAD DISTINCT
    [PCRR.Project ID] as [MST.Project ID],
    [PCRR.Year] as [MST.Year],
    [PCRR.Period] as [MST.Period],
    [PCRR.Project ID] & '|' & [PCRR.Year] & '|' & [PCRR.Period] as [MST.Key]
RESIDENT PCRR_Data;

// 2. Add from secondary source (ONLY NEW combinations)
CONCATENATE (Master)
LOAD DISTINCT
    [Test10.Project ID] as [MST.Project ID],
    [Test10.Year] as [MST.Year],
    [Test10.Period] as [MST.Period],
    [Test10.Project ID] & '|' & [Test10.Year] & '|' & [Test10.Period] as [MST.Key]
RESIDENT Test10_Data_Final
WHERE NOT EXISTS([MST.Key], [Test10.Project ID] & '|' & [Test10.Year] & '|' & [Test10.Period]);
```

### WHERE NOT EXISTS Syntax

```qlik
WHERE NOT EXISTS(FieldName, ValueToCheck)
```

**Returns TRUE when:**
- `ValueToCheck` does NOT exist in `FieldName` (in already loaded data)

**Use Case:**
- Prevent duplicates when concatenating from multiple sources
- Add records only if they don't already exist

### Examples

```qlik
// Simple field check
WHERE NOT EXISTS([Project ID], [New Project ID])

// Composite key check
WHERE NOT EXISTS([MST.Key], [Project ID] & '|' & [Year])

// Multiple conditions
WHERE NOT EXISTS([Project ID], [Project ID])
  AND [Status] = 'Active'
```

---

## CONCATENATE - Explicit Syntax

**CRITICAL:** Always write `CONCATENATE (TableName)` explicitly!

### Why Explicit?

Qlik auto-concatenates tables with identical field lists, but:
- Implicit behavior is unpredictable
- Hard to trace in logs
- Can cause unexpected results

### Pattern

```qlik
// ❌ BAD - Implicit (relies on auto-detection)
Sales_Data:
LOAD * FROM File1.xlsx;

Sales_Data:  // Auto-concatenated (not obvious)
LOAD * FROM File2.xlsx;

// ✅ GOOD - Explicit (clear intent)
Sales_Data:
LOAD * FROM File1.xlsx;

CONCATENATE (Sales_Data)
LOAD * FROM File2.xlsx;
```

### Common Usage

**1. FileList Loop (see FOR EACH above)**

**2. Multiple Sources**
```qlik
// Load from first source
Data_Table:
LOAD * FROM [Source1.xlsx];

// Add from second source
CONCATENATE (Data_Table)
LOAD * FROM [Source2.xlsx];

// Add from third source
CONCATENATE (Data_Table)
LOAD * FROM [Source3.xlsx];
```

**3. Combining RESIDENT Loads**
```qlik
// Load 2024 data
Combined:
LOAD * RESIDENT Source_2024
WHERE [Year] = 2024;

// Add 2025 data
CONCATENATE (Combined)
LOAD * RESIDENT Source_2025
WHERE [Year] = 2025;
```

---

## LEFT JOIN - When to Use

**Preference:** Use MAPPING over JOIN whenever possible (see CLAUDE.md).

**When JOIN is acceptable:**
- Small helper tables (< 1000 rows)
- Joining dimensions to facts (1:1 relationship guaranteed)
- Fields from helper table needed in final RESIDENT LOAD

### JOIN Timing Rule

**CRITICAL:** Fields from helper tables must be joined BEFORE final RESIDENT LOAD!

**Why?**
RESIDENT LOAD creates NEW table → reads only fields that EXIST NOW.

**Wrong Timing:**
```qlik
// 1. Create final table
Final:
LOAD
    *,
    [Helper.Field]  // ❌ ERROR! Doesn't exist yet!
RESIDENT Source;

// 2. Join helper (TOO LATE!)
LEFT JOIN (Final)
LOAD * RESIDENT Helper;
```

**Correct Timing:**
```qlik
// 1. Join helper first
Source_Temp:
LOAD * RESIDENT Source;

LEFT JOIN (Source_Temp)
LOAD * RESIDENT Helper;

// 2. Create final table (helper fields now available)
Final:
LOAD
    *,
    [Helper.Field]  // ✅ Works!
RESIDENT Source_Temp;

DROP TABLE Source_Temp;
```

### JOIN Syntax

```qlik
LEFT JOIN (TargetTable)
LOAD
    [Key Field],
    [Field to Add]
RESIDENT HelperTable;
```

---

## Quick Reference

### CROSSTABLE

```qlik
CROSSTABLE([MonthField], [ValueField], NumberOfFixedColumns)
LOAD
    [ID Column],  // Fixed columns
    [Jan], [Feb], [Mar]  // Pivot columns
FROM Source;
```

### FOR EACH

```qlik
FOR EACH vFile IN FileList('path/pattern*')
    IF NoOfRows('Table') = 0 OR IsNull(TableNumber('Table')) THEN
        Table: LOAD * FROM [$(vFile)];
    ELSE
        CONCATENATE (Table) LOAD * FROM [$(vFile)];
    END IF
NEXT vFile;
```

### WHERE NOT EXISTS

```qlik
WHERE NOT EXISTS([Key Field], [Value to Check])
```

### CONCATENATE

```qlik
CONCATENATE (TargetTable)
LOAD * FROM Source;
```
