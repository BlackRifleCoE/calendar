# Calendar Dimension Tables for Microsoft Fabric

Enterprise-grade calendar and time dimension tables for self-service data platforms, with support for multi-division fiscal year reporting.

## Overview

This solution provides two core dimension tables designed for Microsoft Fabric Warehouse:

- **Dim_Date**: Comprehensive date dimension with enterprise calendar attributes, holidays, and configurable fiscal years for multiple divisions
- **Dim_Time**: Time-of-day dimension with business hour indicators and time period classifications

### Key Features

✅ **Multi-Division Fiscal Year Support**: Configure unique fiscal year start dates for each division
✅ **Group Fiscal Year**: Overarching fiscal year for consolidated reporting
✅ **Enterprise Calendar Attributes**: ISO weeks, quarters, day classifications, and more
✅ **UK Public Holidays**: Automatic bank holiday identification with IsHoliday flag
✅ **Self-Contained Notebooks**: All DDL and logic embedded in Fabric notebooks
✅ **Dynamic Configuration**: Automatic workspace and lakehouse ID detection using notebookutils
✅ **Configurable Date Ranges**: JSON-based configuration for flexibility
✅ **Schedulable Pipeline**: Automated refresh via Fabric Data Pipeline

---

## Architecture

```
Microsoft Fabric Workspace
│
├── Lakehouse (attached to notebooks)
│   └── Files/
│       └── config/
│           └── dimension_config.json
│
├── Notebooks
│   ├── Generate_Dim_Date.ipynb
│   └── Generate_Dim_Time.ipynb
│
├── Warehouse
│   ├── Dim_Date (table)
│   └── Dim_Time (table)
│
└── Data Pipeline
    └── Dimension_Tables_Refresh
```

---

## Table Schemas

### Dim_Date

| Column Category | Columns |
|----------------|---------|
| **Primary Key** | DateKey, Date |
| **Calendar Attributes** | Year, Quarter, Month, Day, YearMonth, MonthName, DayName, DayOfWeek, WeekOfYear, ISOWeekOfYear, etc. |
| **Date Flags** | IsWeekend, IsWeekday, IsLastDayOfMonth, IsLastDayOfQuarter, IsLeapYear |
| **Holidays** | IsHoliday, HolidayName |
| **Group Fiscal** | Group_FiscalYear, Group_FiscalQuarter, Group_FiscalMonth, Group_FiscalYearQuarter, etc. |
| **Division Fiscal** | DIV1_FiscalYear, DIV1_FiscalQuarter... (repeated for each division) |

**Row Count**: ~3,653 rows per year (configurable range)

### Dim_Time

| Column Category | Columns |
|----------------|---------|
| **Primary Key** | TimeKey, Time |
| **Time Components** | Hour24, Hour12, Minute, Second, AMPM |
| **Formatted Time** | TimeDisplay12, TimeDisplay24, HourMinute |
| **Calculations** | MinuteOfDay, SecondOfDay, QuarterHour, HalfHour |
| **Categories** | IsBusinessHour, DayPeriod, HourBucket |

**Row Count**: 1,440 rows (minute granularity) or 86,400 rows (second granularity)

---

## Deployment Guide

### Prerequisites

1. **Microsoft Fabric Workspace** with:
   - Lakehouse (for configuration storage)
   - Warehouse (for dimension tables)
   - Spark compute capacity (for notebook execution)

2. **Permissions**:
   - Contributor access to workspace
   - Write access to Lakehouse and Warehouse

### Step 1: Upload Configuration File

1. Navigate to your Lakehouse in Fabric workspace
2. Go to **Files** section
3. Create folder structure: `config/`
4. Upload `config/dimension_config.json` to `Files/config/`

**Configuration Structure:**

```json
{
  "date_range": {
    "start_year": 2020,
    "end_year": 2030
  },
  "group_fiscal_year": {
    "name": "Group",
    "start_month": 1,
    "start_day": 1
  },
  "divisions": [
    {
      "code": "DIV1",
      "name": "Division One",
      "fiscal_year_start_month": 7,
      "fiscal_year_start_day": 1
    }
    // ... add all 6 divisions
  ],
  "holidays": {
    "enabled": true,
    "country": "UK"
  }
}
```

**Customization Tips:**
- `start_year`/`end_year`: Define the date range (e.g., 10 years of history + 2 years future)
- `start_month`/`start_day`: Set fiscal year start dates (1-12 for month, 1-31 for day)
- `code`: Short division code (e.g., "DIV1", "EMEA", "APAC") - used as column prefix
- `name`: Full division name for documentation

### Step 2: Import Notebooks

1. In Fabric workspace, navigate to **Notebooks**
2. Click **Import** > **Upload from .ipynb file**
3. Upload both notebooks:
   - `notebooks/Generate_Dim_Date.ipynb`
   - `notebooks/Generate_Dim_Time.ipynb`

4. **Attach Lakehouse** to both notebooks:
   - Open each notebook
   - Click **Add lakehouse** (left sidebar)
   - Select your Lakehouse containing the config file

### Step 3: Configure Target Warehouse

Both notebooks need to know which Fabric Warehouse to target for table creation.

**Important Configuration (both notebooks - Cell 2):**
```python
warehouse_name = "YourWarehouse"  # UPDATE THIS: Your Fabric Warehouse name
```

**How it works:**
- Workspace ID and Lakehouse ID are retrieved automatically using `notebookutils.runtime.context`
  - `currentWorkspaceId`: Your Fabric workspace GUID
  - `defaultLakehouseId`: The attached lakehouse GUID
- Tables are created using three-part naming: `{warehouse_name}.dbo.TableName`
- Example: `MyWarehouse.dbo.Dim_Date`

**Additional Configuration:**

**Generate_Dim_Time.ipynb** - Configure time granularity if needed (default: "minute"):
```python
time_granularity = "minute"  # 'minute' or 'second'
```

**Finding Your Warehouse Name:**
- Navigate to your Fabric workspace
- Look in the left navigation under "SQL analytics endpoint" or "Warehouses"
- Use the exact name of your target warehouse

**Note:** The notebooks automatically attach to the default lakehouse for config file access, but create tables in the specified warehouse using SQL commands.

### Step 4: Execute Notebooks

**Initial Load:**

1. Run `Generate_Dim_Date.ipynb` first:
   - Click **Run all** button
   - Monitor execution (typically 2-5 minutes)
   - Verify success in final cells

2. Run `Generate_Dim_Time.ipynb`:
   - Click **Run all** button
   - Verify completion (typically 1-2 minutes)

**Expected Output:**
- ✓ Configuration loaded successfully
- ✓ Tables created in warehouse
- ✓ Data quality checks passed
- ✓ Data loaded to warehouse

**How Warehouse Execution Works:**

The notebooks use Spark SQL with three-part naming to create and populate tables directly in your Fabric Warehouse:

1. **Table Creation**: `CREATE TABLE {warehouse_name}.dbo.Dim_Date (...)`
   - Creates the table in the specified warehouse's `dbo` schema
   - Uses standard SQL DDL syntax

2. **Data Loading**: `saveAsTable(f"{warehouse_name}.dbo.Dim_Date")`
   - Writes DataFrame data directly to the warehouse table
   - Uses Delta format for efficient storage and updates
   - Overwrites existing data on each run

3. **Verification**: `SELECT COUNT(*) FROM {warehouse_name}.dbo.Dim_Date`
   - Queries the warehouse table to verify data loaded successfully

**Note:** The notebooks run in the Lakehouse Spark environment but create tables in the Warehouse using cross-resource SQL commands.

### Step 5: Verify Deployment

Connect to your Warehouse and run validation queries:

```sql
-- Check Dim_Date
SELECT
    MIN(Date) as MinDate,
    MAX(Date) as MaxDate,
    COUNT(*) as TotalRows,
    SUM(CASE WHEN IsHoliday = 1 THEN 1 ELSE 0 END) as HolidayCount
FROM Dim_Date;

-- Sample fiscal year data
SELECT
    Date,
    Year,
    Group_FiscalYear,
    DIV1_FiscalYear,
    DIV2_FiscalYear
FROM Dim_Date
WHERE Day = 1 AND Month = 1
ORDER BY Date DESC;

-- Check Dim_Time
SELECT
    MIN(TimeKey) as MinTime,
    MAX(TimeKey) as MaxTime,
    COUNT(*) as TotalRows,
    SUM(CASE WHEN IsBusinessHour = 1 THEN 1 ELSE 0 END) as BusinessHourCount
FROM Dim_Time;
```

### Step 6: Schedule Refresh (Optional)

1. Import `pipelines/dimension_refresh_pipeline.json` into Fabric
2. Update pipeline configuration:
   - Spark pool references
   - Notification webhook URLs (optional)
   - Schedule (monthly, quarterly, or yearly)
3. Activate the pipeline schedule

---

## Usage Examples

### Joining with Fact Tables

```sql
-- Sales analysis with calendar attributes
SELECT
    d.Year,
    d.MonthName,
    d.Group_FiscalQuarter,
    d.IsWeekend,
    SUM(f.SalesAmount) as TotalSales
FROM FactSales f
INNER JOIN Dim_Date d ON f.DateKey = d.DateKey
WHERE d.Group_FiscalYear = 2025
GROUP BY d.Year, d.MonthName, d.Group_FiscalQuarter, d.IsWeekend;

-- Hourly transaction analysis
SELECT
    t.HourBucket,
    t.DayPeriod,
    t.IsBusinessHour,
    COUNT(*) as TransactionCount,
    AVG(f.Amount) as AvgAmount
FROM FactTransactions f
INNER JOIN Dim_Time t ON f.TimeKey = t.TimeKey
GROUP BY t.HourBucket, t.DayPeriod, t.IsBusinessHour;
```

### Multi-Division Fiscal Reporting

```sql
-- Compare division fiscal quarters
SELECT
    d.Date,
    d.Group_FiscalQuarter as GroupFQ,
    d.DIV1_FiscalQuarter as Division1_FQ,
    d.DIV2_FiscalQuarter as Division2_FQ,
    d.DIV3_FiscalQuarter as Division3_FQ
FROM Dim_Date d
WHERE d.Year = 2025 AND d.Day = 1
ORDER BY d.Date;

-- Division-specific fiscal year sales
SELECT
    d.DIV1_FiscalYear,
    d.DIV1_FiscalQuarter,
    SUM(f.SalesAmount) as TotalSales
FROM FactSales f
INNER JOIN Dim_Date d ON f.DateKey = d.DateKey
WHERE d.DIV1_FiscalYear = 2025
GROUP BY d.DIV1_FiscalYear, d.DIV1_FiscalQuarter;
```

### Holiday and Weekend Analysis

```sql
-- Sales comparison: holidays vs regular days
SELECT
    CASE WHEN IsHoliday = 1 THEN 'Holiday' ELSE 'Regular Day' END as DayType,
    COUNT(DISTINCT d.DateKey) as DayCount,
    SUM(f.SalesAmount) as TotalSales,
    AVG(f.SalesAmount) as AvgSalesPerDay
FROM FactSales f
INNER JOIN Dim_Date d ON f.DateKey = d.DateKey
GROUP BY CASE WHEN IsHoliday = 1 THEN 'Holiday' ELSE 'Regular Day' END;

-- Specific holiday analysis
SELECT
    d.HolidayName,
    d.Date,
    SUM(f.SalesAmount) as TotalSales
FROM FactSales f
INNER JOIN Dim_Date d ON f.DateKey = d.DateKey
WHERE d.IsHoliday = 1 AND d.Year = 2025
GROUP BY d.HolidayName, d.Date
ORDER BY d.Date;
```

---

## Maintenance

### Updating Configuration

To add divisions or change fiscal year dates:

1. Update `dimension_config.json` in Lakehouse Files
2. Re-run `Generate_Dim_Date.ipynb` notebook
3. Tables will be recreated with new structure

### Extending Date Range

To add future years:

1. Update `end_year` in `dimension_config.json`
2. Re-run `Generate_Dim_Date.ipynb`
3. New dates will be appended

### Refresh Schedule Recommendations

| Scenario | Recommended Schedule | Reason |
|----------|---------------------|---------|
| **Historical Only** | Yearly | Date dimensions rarely change |
| **Forward-Looking** | Monthly | Add new future dates regularly |
| **Config Changes** | Manual/On-Demand | After fiscal year adjustments |

---

## Troubleshooting

### Common Issues

**Issue**: Configuration file not found
```
Error: Path does not exist: abfss://...
```
**Solution**:
- Verify config file uploaded to `Files/config/` in Lakehouse
- Ensure Lakehouse is attached to notebook (workspace and lakehouse IDs are retrieved automatically)
- Check that the lakehouse attachment is set as default

**Issue**: Duplicate columns in Dim_Date
```
Error: Duplicate column names
```
**Solution**:
- Ensure division codes in config are unique
- Division codes should be simple identifiers (no spaces or special characters)

**Issue**: Notebook fails with "No module named 'calendar'"
```
ModuleNotFoundError: No module named 'calendar'
```
**Solution**:
- The `calendar` module is Python built-in and should be available
- If issue persists, check Fabric runtime version (should be Runtime 1.2+)

**Issue**: Fiscal year calculations incorrect
**Solution**:
- Verify `fiscal_year_start_month` and `fiscal_year_start_day` in config
- Check sample outputs in notebook quality check section
- For fiscal year starting July 1: month=7, day=1

**Issue**: Table not found in warehouse
```
Error: Table or view not found: YourWarehouse.dbo.Dim_Date
```
**Solution**:
- Update `warehouse_name` in notebook configuration (Cell 2) to match your actual warehouse name
- Verify warehouse exists in your Fabric workspace
- Check you have write permissions to the warehouse
- Ensure warehouse name doesn't contain special characters (use exact name from workspace)

**Issue**: Cannot create table in warehouse
```
Error: CREATE TABLE failed
```
**Solution**:
- Verify you have Contributor or Admin permissions on the warehouse
- Check warehouse is not in read-only mode
- Ensure warehouse has sufficient storage capacity
- Try running the DDL cell separately to see specific error details

### Performance Optimization

For large date ranges (20+ years):
- Consider partitioning Dim_Date by Year
- Use minute granularity for Dim_Time (not second)
- Increase Spark executor size in notebook/pipeline

---

## Technical Details

### Fiscal Year Calculation Logic

```python
# If current date >= fiscal start in current calendar year
# Then FiscalYear = CurrentYear + 1
# Else FiscalYear = CurrentYear

# Example: Fiscal year starts July 1
# Date: 2025-06-30 → FY2025
# Date: 2025-07-01 → FY2026
```

### Holiday Detection

Currently supports UK Public Holidays (Bank Holidays):
- New Year's Day
- Good Friday
- Easter Monday
- Early May Bank Holiday
- Spring Bank Holiday (last Monday in May)
- Summer Bank Holiday (last Monday in August)
- Christmas Day
- Boxing Day

**To add custom holidays**: Modify the `get_uk_holidays()` function in `Generate_Dim_Date.ipynb`

### Time Granularity

| Granularity | Row Count | Use Case |
|-------------|-----------|----------|
| **Minute** | 1,440 | Recommended for most analytics |
| **Second** | 86,400 | High-frequency transaction systems |

---

## Support & Contributions

### Getting Help

For issues or questions:
1. Check Troubleshooting section above
2. Review notebook execution logs in Fabric
3. Verify configuration file structure

### Customization

Common customizations:
- **Add custom holidays**: Modify holiday logic in Dim_Date notebook
- **Additional calendar attributes**: Add columns in data generation section
- **Custom fiscal calculations**: Modify `add_fiscal_year_columns()` function
- **Business hours definition**: Adjust IsBusinessHour logic in Dim_Time

---

## File Structure

```
calendar/
├── config/
│   └── dimension_config.json          # Configuration file (upload to Lakehouse Files)
├── notebooks/
│   ├── Generate_Dim_Date.ipynb        # Self-contained date dimension notebook
│   └── Generate_Dim_Time.ipynb        # Self-contained time dimension notebook
├── pipelines/
│   └── dimension_refresh_pipeline.json # Schedulable pipeline definition
└── README.md                           # This file
```

---

## Version History

**v1.0.0** (2025-12-09)
- Initial release
- Multi-division fiscal year support (6 configurable divisions)
- UK Public Holidays (Bank Holidays) with Easter calculation
- Self-contained Fabric notebooks (all DDL + logic embedded)
- Dynamic workspace and lakehouse ID detection using notebookutils
  - Uses `currentWorkspaceId` and `defaultLakehouseId` from runtime context
- Fabric Warehouse targeting with three-part naming
  - Tables created in specified warehouse: `{warehouse_name}.dbo.TableName`
  - Cross-resource SQL execution from Lakehouse notebooks to Warehouse
- BOOLEAN datatype for compatibility with Fabric Warehouse
- Configurable via JSON (divisions, fiscal years, date ranges, holidays)

---

## License

This solution is provided as-is for use within Microsoft Fabric environments.

---

## Additional Resources

- [Microsoft Fabric Documentation](https://learn.microsoft.com/fabric/)
- [Dimension Table Design Best Practices](https://learn.microsoft.com/analysis-services/tabular-models/)
- [Fiscal Year Reporting in Power BI](https://learn.microsoft.com/power-bi/)
