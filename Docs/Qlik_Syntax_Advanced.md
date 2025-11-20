# Qlik Syntax: Advanced Topics

Advanced Qlik Sense syntax patterns, edge cases, and optimization techniques.

---

## QUALIFY - Field Name Disambiguation

Automatically prefix field names with table name to avoid naming conflicts.

### Problem

Multiple tables with identical field names (except join keys) create unintended associations.

```qlik
// ❌ PROBLEM - "Name" and "Country" in both tables!
Customers:
LOAD
    ID,
    Name,      // ← Conflict
    Country    // ← Conflict
FROM [Customers.xlsx];

Suppliers:
LOAD
    ID,
    Name,      // ← Conflict
    Country    // ← Conflict
FROM [Suppliers.xlsx];

// Qlik links tables via Name + Country → Data corruption!
```

**Result:** Unwanted synthetic key or unexpected associations between Customers and Suppliers.

### Solution 1: Manual Renaming (Preferred)

```qlik
Customers:
LOAD
    ID as Customer_ID,
    Name as Customer_Name,
    Country as Customer_Country
FROM [Customers.xlsx];

Suppliers:
LOAD
    ID as Supplier_ID,
    Name as Supplier_Name,
    Country as Supplier_Country
FROM [Suppliers.xlsx];
```

**Advantages:**
- Explicit field names (clear in data model)
- Full control over naming
- Easy to trace in expressions

### Solution 2: QUALIFY (When Many Fields)

```qlik
QUALIFY *;  // All fields get table prefix

Customers:
LOAD * FROM [Customers.xlsx];
// Fields: Customers.ID, Customers.Name, Customers.Country

UNQUALIFY ID;  // ID without prefix for linking

Suppliers:
LOAD * FROM [Suppliers.xlsx];
// Fields: ID (no prefix), Suppliers.Name, Suppliers.Country
```

**Result:**
- Tables link only via `ID` field
- All other fields are unique (Customers.Name ≠ Suppliers.Name)

### QUALIFY Variants

```qlik
// Qualify all fields
QUALIFY *;

// Qualify specific fields only
QUALIFY Name, Country, City;

// Unqualify specific fields (allow linking)
UNQUALIFY ID, Date;

// Reset to default (no qualification)
UNQUALIFY *;
```

### Best Practice

**Use manual renaming when:**
- Few tables (< 5)
- Clear naming convention exists
- Fields need descriptive names

**Use QUALIFY when:**
- Many tables with identical structures
- Temporary staging tables
- Prototype/exploratory analysis

**CRITICAL:** Always ensure only intended fields link between tables!

---

## RESIDENT vs FROM

Understanding the difference is crucial for workflow and performance.

### Basic Difference

```qlik
// FROM - Load NEW data from external source
LOAD * FROM [lib://DATA Transfer:DataFiles/Revenue.xlsx]
(ooxml, embedded labels);

// RESIDENT - Transform EXISTING data already in memory
LOAD * RESIDENT Existing_Table;
```

### Workflow Implications

**FROM Loads:**
- **Ask about data volume** (affects JOIN vs MAPPING decision)
- **Ask about column selection** (which fields needed?)
- **Consider performance** (file size, network latency)

**RESIDENT Loads:**
- **DO NOT ask about data volume** (already in memory!)
- **DO NOT ask about column selection** (all fields available)
- **Focus on transformation** (calculations, filtering, aggregations)

### Example: Wrong Questions

```qlik
// ❌ WRONG - Asking about data volume for RESIDENT
"How many rows are in Test10_Data?"  // Data already loaded!

// ✅ CORRECT - Asking about transformation
"What calculations do you need from Test10_Data?"
```

### Performance Considerations

**FROM is slower when:**
- Large files (> 100 MB)
- Network storage (vs local)
- Complex Excel formulas

**RESIDENT is faster for:**
- Filtering subsets
- Adding calculated fields
- Aggregations
- Transformations

### Pattern: Multi-Step Transformation

```qlik
// 1. Load raw data (FROM)
Raw_Data:
LOAD * FROM [Source.xlsx] (ooxml, embedded labels);

// 2. Add calculated fields (RESIDENT)
Enriched_Data:
LOAD
    *,
    [Revenue] * 1.2 as [Revenue with Tax],
    [Costs] / [Revenue] as [Cost Ratio]
RESIDENT Raw_Data;

// 3. Filter and aggregate (RESIDENT)
Final_Data:
LOAD
    [Project ID],
    Sum([Revenue with Tax]) as [Total Revenue]
RESIDENT Enriched_Data
WHERE [Year] = 2025
GROUP BY [Project ID];

DROP TABLES Raw_Data, Enriched_Data;
```

---

## ORDER BY - When Mandatory

ORDER BY is required for operations that depend on row sequence.

### Mandatory for Peek()

```qlik
// ❌ WRONG - No ORDER BY with Peek()
LOAD
    *,
    RangeSum(Peek('Cumulative'), [Value]) as [Cumulative]
RESIDENT Source;  // Row order is unpredictable!

// ✅ CORRECT - ORDER BY defines sequence
LOAD
    *,
    RangeSum(Peek('Cumulative'), [Value]) as [Cumulative]
RESIDENT Source
ORDER BY [Project ID], [Year], [Period];  // CRITICAL!
```

**Why?** Without ORDER BY, Qlik loads rows in arbitrary order. Peek() returns "previous row", but which row is previous if order is random?

### Cumulative Calculations

Any cumulative calculation needs ORDER BY:

```qlik
LOAD
    *,
    // Cumulative sum
    RangeSum(Peek('Total'), [Amount]) as [Total],

    // Running count
    If([ID] = Peek('ID'), Peek('Count') + 1, 1) as [Count],

    // Carry forward previous value
    If(IsNull([Value]), Peek('Value'), [Value]) as [Value]
RESIDENT Source
ORDER BY [Group], [Sequence];  // MANDATORY!
```

### Optional for Other Operations

ORDER BY is NOT required for:
- Simple filters (WHERE)
- Aggregations (GROUP BY)
- ApplyMap()
- Simple calculations

```qlik
// ORDER BY optional (no row-dependent operations)
LOAD
    *,
    [Revenue] * 1.2 as [Revenue with Tax],
    ApplyMap('Map_Info', [ID], '-') as [Description]
RESIDENT Source
WHERE [Year] = 2025;  // ORDER BY not needed
```

---

## DISTINCT vs WHERE NOT EXISTS

Two ways to eliminate duplicates with different use cases.

### DISTINCT - Within Single Table

```qlik
// Load unique combinations
Master:
LOAD DISTINCT
    [Project ID],
    [Year],
    [Period]
RESIDENT Source;
```

**Use when:**
- Single source table
- Removing duplicates within one table
- Simple deduplication

**Performance:** Fast (single-pass operation)

### WHERE NOT EXISTS - Across Multiple Tables

```qlik
// Load base data
Master:
LOAD DISTINCT
    [Project ID],
    [Year]
RESIDENT Source1;

// Add only NEW combinations from Source2
CONCATENATE (Master)
LOAD DISTINCT
    [Project ID],
    [Year]
RESIDENT Source2
WHERE NOT EXISTS([Project ID], [Project ID]);
// OR use composite key:
// WHERE NOT EXISTS([Key], [Project ID] & '|' & [Year]);
```

**Use when:**
- Multiple source tables
- Preventing duplicates during CONCATENATE
- Union with deduplication

**Performance:** Slower (checks existence for each row)

### Comparison

| Operation | DISTINCT | WHERE NOT EXISTS |
|-----------|----------|------------------|
| **Scope** | Within single table | Across tables |
| **Use case** | Simple deduplication | Conditional append |
| **Performance** | Fast | Slower (existence check) |
| **Typical usage** | `LOAD DISTINCT` | `CONCATENATE ... WHERE NOT EXISTS` |

### Best Practice Pattern

```qlik
// Combine both for optimal results
Master:
LOAD DISTINCT  // Remove duplicates in Source1
    [ID], [Name]
RESIDENT Source1;

CONCATENATE (Master)
LOAD DISTINCT  // Remove duplicates in Source2
    [ID], [Name]
RESIDENT Source2
WHERE NOT EXISTS([ID], [ID]);  // Skip if already in Master
```

---

## DROP TABLE - Memory Management

Drop intermediate tables to free memory and clean up data model.

### When to DROP

```qlik
// 1. After creating mapping
Temp_Aggregate:
LOAD ... RESIDENT Source;

Map_Values:
MAPPING LOAD ... RESIDENT Temp_Aggregate;

DROP TABLE Temp_Aggregate;  // No longer needed


// 2. After transformation
Raw_Data:
LOAD * FROM [Source.xlsx];

Clean_Data:
LOAD ... RESIDENT Raw_Data;  // Transform

DROP TABLE Raw_Data;  // Keep only Clean_Data


// 3. After combining sources
Source1:
LOAD * FROM [File1.xlsx];

Source2:
LOAD * FROM [File2.xlsx];

Combined:
LOAD * RESIDENT Source1;

CONCATENATE (Combined)
LOAD * RESIDENT Source2;

DROP TABLES Source1, Source2;  // Keep only Combined
```

### DROP Syntax

```qlik
// Drop single table
DROP TABLE TableName;

// Drop multiple tables
DROP TABLES Table1, Table2, Table3;

// Drop fields (from all tables where they exist)
DROP FIELD FieldName;

// Drop specific fields from specific table
DROP FIELD Field1, Field2 FROM TableName;
```

### What NOT to Drop

**DO NOT drop:**
- Final tables used in frontend
- Mapping tables still in use
- Tables needed for subsequent LOADs

### Memory Impact

Dropping tables:
- Frees RAM
- Reduces .qvf file size
- Improves load performance (fewer tables in model)
- Simplifies data model viewer

---

## Inline Tables

Create small lookup/mapping tables directly in script.

### Basic Syntax

```qlik
Map_Status:
MAPPING LOAD * INLINE [
    Code, Description
    A, Active
    I, Inactive
    P, Pending
    C, Closed
];
```

### Use Cases

**1. Status/Category Mappings**
```qlik
Map_Category:
MAPPING LOAD * INLINE [
    Code, Category
    01, Small
    02, Medium
    03, Large
];

ApplyMap('Map_Category', [Code], 'Unknown') as [Size Category]
```

**2. Month Name to Number**
```qlik
Map_Month:
MAPPING LOAD * INLINE [
    Month, Number
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
```

**3. Test Data**
```qlik
Test_Data:
LOAD * INLINE [
    ID, Name, Amount
    1, Project A, 1000
    2, Project B, 2000
    3, Project C, 1500
];
```

### INLINE Limitations

- **Size:** Max ~1000 rows (guideline)
- **Maintenance:** Hard-coded in script
- **Changes:** Requires script edit + reload

**When to use:**
- Small, stable lookup tables
- Temporary test data
- Simple category mappings

**When NOT to use:**
- Large reference tables (use Excel/DB instead)
- Frequently changing data
- Data that should be user-editable

---

## Quick Reference

### QUALIFY

```qlik
QUALIFY *;           // All fields get table prefix
UNQUALIFY ID, Date;  // Except these
```

### RESIDENT vs FROM

```qlik
FROM [Source.xlsx]   // New data → Ask volume/columns
RESIDENT Table       // Transform → No volume questions
```

### ORDER BY

```qlik
ORDER BY Field1, Field2  // Required for Peek(), cumulative
```

### DISTINCT

```qlik
LOAD DISTINCT ...    // Unique within table
```

### WHERE NOT EXISTS

```qlik
WHERE NOT EXISTS([Key], [Value])  // Unique across tables
```

### DROP

```qlik
DROP TABLE Table1;
DROP TABLES Table1, Table2, Table3;
DROP FIELD FieldName;
```

### INLINE

```qlik
LOAD * INLINE [
    Col1, Col2
    Val1, Val2
];
```
