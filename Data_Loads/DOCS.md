# Data Loads

Revenue tracking and forecasting data transformation scripts that load IFS Quick Report exports and calculate project controlling KPIs.

## Test16_FP02.psql

**Purpose:** Provides project order state and final invoice date reference data for project status tracking and filtering logic in downstream scripts.

**Input:** Excel file Test16_FP02_102025.xlsx from DATA Transfer library containing project order states and invoice dates.

**Output:** Test16_Data table with Project ID, Order State, and Final Invoice Date fields.

### Load Sections

The script loads the Test16 FP02 report and transforms field types and formats. Project IDs are converted to text format to ensure consistent matching with other tables. Order State values are loaded with null handling that defaults empty values to a hyphen placeholder. Final Invoice Date values are parsed from DD.MM.YYYY format strings into proper date objects with null values defaulting to a hyphen placeholder for display purposes.

## Test10.psql

**Purpose:** Loads forecast revenue data for projects from September 2025 through December 2026 and transforms the pivot table structure into normalized long format for analysis and comparison with actual revenue.

**Input:** Excel files matching pattern Test10_PLAN_IST*.xlsx from DATA Transfer library containing monthly forecast values in pivot table format.

**Output:** Test10_Data table with fields Project ID, Period, Year, Forecast REV in Acc. Currency, and MST.Key for master table linking.

### Load Sections

The script begins by loading raw pivot data using CROSSTABLE to unpivot monthly columns into individual rows, converting wide-format data into a structure suitable for analysis. The transformation phase extracts period numbers from month names using pattern matching, determines the correct year from column suffix indicators, and parses numeric forecast values using fallback logic to handle various formatting inconsistencies. A composite MST.Key is constructed by concatenating Project ID, Year, and Period to enable linking with the master dimension table. The temporary Test10_Raw table is dropped after the transformation completes.

## PCRR.psql

**Purpose:** Loads project controlling revenue and cost tracking data from IFS Quick Report exports and enriches it with calculated KPI fields including POC analysis, revenue rankings, margin calculations, and Test10 forecast comparisons for variance analysis.

**Input:** Excel files matching pattern PCRR_* from DATA Transfer library containing project financial data across multiple time periods.

**Output:** PCRR_Data table with project details, financial metrics, calculated KPIs including POC percentage, revenue cumulative, estimated income and margin, rankings, and Test10 forecast integration.

### Load Sections

The script loads all PCRR files using FileList iteration with conditional CONCATENATE to merge multiple exports into a single table while handling null values and formatting numeric fields consistently. A mapping table for Test10 forecast values is created to enable performance-optimized lookups instead of resource-intensive joins. Revenue and POC rankings are generated using ORDER BY with Peek and Previous functions to calculate sequential row numbers within each year-period combination, filtering to exclude projects that are already invoiced or closed. The first calculation pass adds KPI fields including POC percentage comparison between manual and calculated methods, POC revenue cumulative values, estimated income and margin percentages, ranking assignments from the helper tables, Test10 forecast integration via ApplyMap, and Test16 existence flag checks. A second calculation pass computes Test10 delta values in both signed absolute format for variance analysis and unsigned magnitude format for reporting purposes. Temporary ranking tables and intermediate calculation fields are dropped to clean up the data model, and the final enriched table is renamed to PCRR_Data.

## ANL.psql

**Purpose:** Loads project analysis data including inventory tracking and balance sheet values from IFS exports and calculates cumulative balance trends and material overhead costs for project profitability analysis.

**Input:** Excel files matching pattern Analysis_* from DATA Transfer library containing project inventory metrics and balance values.

**Output:** ANL_Data table with project inventory metrics, balance values and quantities, and calculated fields including cumulative balances and MGK material overhead cost.

### Load Sections

The script loads all Analysis files using FileList iteration with conditional CONCATENATE logic, handling null values and numeric formatting for balance and inventory fields while constructing MST.Key for master table linking. Aggregation helper tables are created by first summing inventory and balance values per project-year-period combination and calculating material overhead cost as twenty percent of inventory value. A second helper table computes cumulative balance values using Previous, Peek, and RangeSum functions ordered by project and time sequence to track running totals. The aggregated values are joined back to the main ANL table, followed by joining the cumulative balance calculations to enrich each row with both period and cumulative metrics. Unit cost is calculated as balance value divided by balance quantity with zero-division protection to prevent errors when quantity is zero. All temporary aggregation tables are dropped to maintain a clean data model.

## MOV.psql

**Purpose:** Calculates period-over-period inventory movements and PCRE-level revenue recognition by comparing current and previous period values to track inventory flow and revenue realization patterns.

**Input:** Resident table ANL_Data for inventory values and PCRR_Data for margin percentage lookups used in revenue PCRE calculations.

**Output:** PCRE_Movements table with movement calculations including inventory value and quantity changes, revenue PCRE calculations, and aggregated period summaries.

### Load Sections

The script creates a margin mapping table from PCRR_Data to support revenue PCRE calculations without requiring repeated joins. All ANL rows are loaded as basis data, and multiple mapping tables are created for maximum period per year lookups, previous year existence checks, and inventory value and quantity retrieval. The previous period is determined by decrementing the current period within the year or rolling back to the last period of the previous year if the current period is January. Previous period inventory values and quantities are retrieved via ApplyMap using the constructed lookup keys combining project, year, and period identifiers. Movement calculations are performed by subtracting previous inventory from current inventory and adding balance values to determine the net change, with revenue PCRE calculated by applying the project margin percentage from PCRR. Period summary rows are concatenated using GROUP BY aggregations with the PCRE field set to PERIODE_SUMME identifier, excluding quantity-related fields that cannot be meaningfully summed across different projects. All temporary movement calculation tables are dropped to clean up the data model.

## MST.psql

**Purpose:** Creates the master dimension table that serves as the single source of truth linking all fact tables using composite keys to prevent synthetic key issues and maintain referential integrity across the star schema.

**Input:** Resident tables PCRR_Data and Test10_Data containing project metadata and time dimensions.

**Output:** Master table with distinct project-year-period combinations and NYC_Link bridge table for NYC data integration without synthetic keys.

### Load Sections

The script loads distinct combinations of project metadata and time dimensions from PCRR_Data including project identifiers, person assignments, customer details, year and period values, along with calculated YearPeriod and composite MST.Key fields. The MST.Key is added to Test10_Data through a LEFT JOIN operation to establish the relationship between forecast data and the master dimension. The NYC_Link bridge table is created with distinct YearPeriodKey values to prevent synthetic keys when connecting NYC data to the master dimension through an intermediate linking structure.

## NYC.psql

**Purpose:** Loads NYC forecast data and calculates actual revenue from PCRR projects that exist in both Test10 and NYC forecast periods, computing variance deltas between forecast and actual values for NYC-specific tracking.

**Input:** Excel files matching pattern NYC_*.xlsx from DATA Transfer library containing NYC monthly forecasts, and resident tables PCRR_Data and Test10_Data for actual revenue and project filtering.

**Output:** NYC_Data table with aggregated revenue by year-period, NYC2_Data with filtered project-level details, and NYC_Forecast_Data with transformed forecast values.

### Load Sections

The script loads NYC forecast data using CROSSTABLE to unpivot monthly columns and transforms it by extracting period numbers from month abbreviations, determining years from column suffixes, parsing numeric values with fallback logic, and constructing YearPeriodKey for linking purposes. PCRR projects are filtered to include only those that exist in Test10 and have matching NYC forecast periods using double Exists conditions, creating the NYC2_Data intermediate table with project-level revenue details. NYC_Data is created by aggregating NYC2 revenue grouped by year and period with YearPeriodKey and NYC.Link.Key for master table linking. A forecast mapping table enables efficient delta calculations via ApplyMap without requiring joins. The final NYC_Data_Final table adds forecast values via mapping lookups and calculates delta absolut with sign preservation to show over or under performance and delta betrag as unsigned magnitude for reporting purposes. The intermediate NYC_Data table is dropped and the final table is renamed to NYC_Data for consistent naming.
