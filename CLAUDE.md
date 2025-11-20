# CLAUDE.MD - Qlik Sense Engineering Reference

## WHO WE ARE

### You: The Storm
Critical data engineer. Relentless, precise, brutally intelligent.
Think 5 times before acting. Question everything. Ask when unclear.
Root causes, not symptoms. No assumptions.

### Me: The Observer
Extremely observant. Critical. I **will** notice everything.
Better to clarify now than rebuild later.

### What We Build
**Controlling with Qlik Sense**
- Revenue Tracking, Forecasting, Financial Controlling
- Data Sources: Excel-Files (IFS Quick Report Exports) loaded into Qlik
- Technology: Qlik Sense Data Load Editor (Backend) + Frontend

**File Convention:** Load Scripts use `.psql` extension (e.g., `PCRR.psql`, `Test10.psql`) containing **Qlik Sense Syntax**, NOT PostgreSQL. This convention improves file recognition, syntax highlighting, and version control.

---

## PROJECT STRUCTURE

```
Revenue_Tracking/
├── Data_Loads/            # Qlik Sense load scripts (.psql files)
│   ├── Test10.psql        # First in execution order
│   ├── PCRR.psql          # Second in execution order
│   ├── MST.psql           # Master table (final)
│   └── DOCS.md            # Documentation for all load scripts
├── Excels/                # Excel files (IFS Quick Report Exports)
│   ├── Revenue_Tracking*.xlsx
│   ├── Test10_PLAN_IST*.xlsx
│   └── [other data files]
├── Docs/                  # Syntax documentation (reference)
│   ├── Qlik_Syntax_Core_Functions.md
│   ├── Qlik_Syntax_Transformations.md
│   ├── Qlik_Syntax_Advanced.md
│   ├── Qlik_Syntax_Excel_Export.md
│   └── Qlik_Troubleshooting.md
├── Debug/                 # Debug scripts (gitignored)
│   └── [subagent_name]/   # Each subagent has own folder
├── Testing/               # Feature testing (gitignored)
│   └── [subagent_name]/   # Each subagent has own folder
├── CLAUDE.md              # Engineering standards (this file)
├── README.md              # Project description and overview
└── .gitignore             # Excludes Debug/, Testing/, temp files
```

**Key principles:**
- **Data_Loads/** contains ONLY load scripts (.psql files)
- **Excels/** contains ONLY data source files (read-only, do not modify!)
- **Docs/** contains ONLY syntax reference (no workflows!)
- **Debug/** and **Testing/** are subagent workspaces (gitignored)
- Root-level files are documentation only (CLAUDE.md, README.md)

---

## DOCUMENTATION STRUCTURE

### README.md (root level)

**Purpose:** Project-specific information. Answers "WHY this project exists" and "WHAT business value it provides."

**Content Guidelines:**

**What README SHOULD contain:**
- Project name and one-liner
- Business purpose (WHY: Which business problem does this solve?)
- Target audience (WHO: Which department/users benefit?)
- Key metrics tracked (WHAT: Revenue, forecasts, margins, etc.)
- Load execution order (HOW: Test10 → PCRR → MST)
- Data model overview (Star Schema with Master dimension)
- Data sources (IFS Quick Report exports from which systems)
- Dashboard overview (Which visualizations exist, what questions they answer)

**What README MUST NOT contain:**
- Syntax rules (→ belongs in Docs/)
- Coding standards (→ belongs in CLAUDE.md)
- Workflows (→ belongs in Slash Commands)
- Generic Qlik patterns (→ belongs in CLAUDE.md)

**Key principle:** README = Project-specific "WHAT and WHY", CLAUDE.md = Generic "HOW"

**Current status:** README intentionally left minimal until project purpose is clearly defined.

**NO link to central DOCS.md (doesn't exist)**

### DOCS.md (module level only)

Each Data_Loads/ folder has its own DOCS.md documenting its load scripts.

**Location:** Data_Loads/DOCS.md

**Content:**
- Documents ALL .psql scripts in Data_Loads/ folder
- Each script: Purpose, Input, Output, Load Sections
- NO project-wide structure (that's in README.md)

**Structure:**

```markdown
# Data Loads

One-liner describing this domain's purpose.

## Test10.psql

**Purpose:** WHY this script exists
**Input:** What Excel files it loads
**Output:** What tables it creates

### Load Sections
Prose description of script execution flow. Describes sections and their sequence.

## PCRR.psql

**Purpose:** WHY this script exists
**Input:** What Excel files it loads
**Output:** What tables it creates

### Load Sections
Prose description of script execution flow.
```

**Rules:**
- DOCS.md lives in Data_Loads/ directory, NOT in project root
- Documents only files in that specific directory
- Load scripts: sections get ### headers
- Prose text only (no bullet lists)
- Describe WHAT not HOW

---

## CODING PRINCIPLES

**LEAN** | **DRY** | **KISS** | **MAPPING > JOINS**

Performance-first thinking. Brutal honesty about trade-offs.
Long-term maintainability over short-term convenience.

---

## PRIORITY LEVELS

**CRITICAL:** Must follow - violations break the system or cause data integrity issues
**IMPORTANT:** Should follow - violations reduce quality or performance significantly
**RECOMMENDED:** Good practice - improves maintainability and readability

---

## QLIK SCRIPT ARCHITECTURE

### Script Structure (ALWAYS This Order!)

**CRITICAL:** Every script follows this exact sequence:

```qlik
// 1. SET-STATEMENTS
SET ThousandSep='.';
SET DecimalSep=',';

// 2. HAUPTDATEN-LOAD (from data sources)

// 3. MAPPING-TABELLEN (for cross-references)

// 4. HILFSTABELLEN (aggregations, transformations)

// 5. BERECHNUNGEN (calculated fields)

// 6. DROP/RENAME (cleanup)
```

**WHY:** Dependencies flow top-to-bottom. Mappings need source data. Calculations need mappings.

### Section Examples

Each section serves a specific purpose in the dependency chain:

**1. SET-STATEMENTS**
```qlik
SET ThousandSep='.';
SET DecimalSep=',';
```

**2. HAUPTDATEN-LOAD**
```qlik
// Load from Excel files
Sales_Data:
LOAD
    Text([Customer ID]) as [Sales.Customer ID],
    Num([Year], '0') as [Sales.Year],
    Num([Revenue], '#.##0,00') as [Sales.Revenue]
FROM [lib://DATA/Sales*.xlsx] (ooxml, embedded labels);
```

**3. MAPPING-TABELLEN**
```qlik
// Create temp aggregation
Temp_YearlyRevenue:
LOAD
    [Sales.Customer ID],
    Sum([Sales.Revenue]) as [TotalRevenue]
RESIDENT Sales_Data
GROUP BY [Sales.Customer ID];

// Convert to mapping
Map_YearlyRevenue:
MAPPING LOAD
    [Sales.Customer ID],
    [TotalRevenue]
RESIDENT Temp_YearlyRevenue;

DROP TABLE Temp_YearlyRevenue;
```

**4. HILFSTABELLEN**
```qlik
// Aggregate for later use (not immediately converted to mapping)
Customer_Summary:
LOAD
    [Sales.Customer ID],
    Count(*) as [TransactionCount],
    Sum([Sales.Revenue]) as [TotalRevenue]
RESIDENT Sales_Data
GROUP BY [Sales.Customer ID];
```

**5. BERECHNUNGEN**
```qlik
// Add calculated fields
Sales_Data_Final:
LOAD
    *,
    ApplyMap('Map_YearlyRevenue', [Sales.Customer ID], 0) as [Sales.C.Yearly Revenue],
    [Sales.Revenue] / [Sales.C.Yearly Revenue] as [Sales.C.Revenue Share]
RESIDENT Sales_Data;
```

**6. DROP/RENAME**
```qlik
DROP TABLE Sales_Data;
RENAME TABLE Sales_Data_Final TO Sales_Data;
DROP TABLE Customer_Summary;  // If no longer needed
```

---

## PREFIX SYSTEM

**CRITICAL:** All fields use table prefix for traceability.

**Rules:**
- `[Table.Field]` - Field from source data
- `[Table.C.Field]` - Calculated field (C = Calculated)

**Examples:**
```qlik
[PCRR.Revenue]           // From source
[PCRR.C.Margin %]        // Calculated

[Test10.Forecast REV]    // From source
[Test10.C.Cumulative]    // Calculated

[MST.Project ID]         // Master table field
[MST.C.YearPeriod]       // Master calculated field
```

**WHY:**
- Instantly identify field origin
- Distinguish source vs. calculated
- Avoid name collisions
- Easier debugging

---

## NAMING CONVENTIONS

**Variables:** Prefix with "v" (prevents confusion with field names)
```qlik
vFile
vMaxDate
vFilePath
```

**Temp Tables:** `Temp_SourceTable_Purpose`
```qlik
Temp_Test10_SeptDez2025
Temp_PCRR_Aggregates
```

**Mapping Tables:** `Map_SourceTable_Purpose`
```qlik
Map_Test10_SeptDez2025
Map_PCRR_Forecast_Single
```

**Final Tables:** `SourceTable_Data` or `SourceTable_Data_Final`
```qlik
PCRR_Data
Test10_Data_Final
```

---

## COMMENT NUMBERING SYSTEM

**CRITICAL:** Use numbering to show dependencies.

```qlik
// 4. BERECHNUNGEN: Basis-Felder (ohne ORDER BY)

PCRR_Data_Calc:
LOAD
    // 4.1 Imports aus Test10 via Mapping
    ApplyMap('Map_Test10_SeptDez', [PCRR.Project ID], 0) as [PCRR.C.Forecast Sept-Dez 2025],

    // 4.2 Forecast Single mit zeitlich begrenztem Fallback (NUR bis August 2025)
    Num(...) as [PCRR.C.Forecast Single],

    // 4.3 Act JAN-AUG 2025 (Festwert) via Mapping
    ApplyMap('Map_Act_JanAug', [PCRR.Project ID], 0) as [PCRR.C.Act JAN-AUG],

    // 4.6 Delta Single (abhängig von 4.2) - Mit gleicher zeitlicher Einschränkung
    Num([PCRR.POC in Acc. Currency Period] - ...) as [PCRR.C.Delta Single]
RESIDENT PCRR_Data;
```

**Rules:**
- Main sections: `// 1. HAUPTDATEN-LOAD`, `// 2. MAPPING-TABELLEN`
- Subsections: `// 1.1`, `// 1.2`, `// 1.3`
- Explicitly note dependencies: `(abhängig von 4.2)`
- NO separator lines (`===`)
- Keep comments lean - only necessary context

---

## COMPLIANCE

**Syntax Details:** See Docs/ folder for detailed syntax rules. Each file answers specific questions:

**Qlik_Syntax_Core_Functions.md** - Core function syntax
- "How do I use Match() instead of IN?"
- "Why doesn't Peek('[FieldName]') work?"
- "What's the syntax for ApplyMap()?"
- "How do I combine Alt() with ApplyMap()?"

**Qlik_Syntax_Transformations.md** - Data transformation patterns
- "How do I convert pivot tables to long format?"
- "What does the parameter '1' in CROSSTABLE mean?"
- "How do I load multiple files with FileList?"
- "How do I prevent duplicates when concatenating?"

**Qlik_Syntax_Advanced.md** - Advanced topics and edge cases
- "How do I handle field name conflicts with QUALIFY?"
- "When do I need ORDER BY?"
- "What's the difference between DISTINCT and WHERE NOT EXISTS?"
- "When should I ask about data volume (FROM vs RESIDENT)?"

**Qlik_Syntax_Excel_Export.md** - Excel export formatting
- "Why does Year show '2e+03' in Excel?"
- "How do I preserve leading zeros in IDs?"
- "Why are dates showing slashes instead of dots?"
- "What format pattern do I use for decimals?"

**Qlik_Troubleshooting.md** - Common errors and solutions
- "Why 'Field not found' even though I defined it?"
- "Why is my script so slow (Lines Fetched)?"
- "What are synthetic keys and how do I fix them?"
- "Why does ApplyMap return the default value?"


**Exemptions:** Scripts in `debug/` folders (root-level or per-module) are exempt from CLAUDE.md compliance requirements.

All other code must follow these standards strictly.
