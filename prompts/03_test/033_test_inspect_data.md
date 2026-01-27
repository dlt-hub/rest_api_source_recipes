---
description: Inspect and explore loaded data with DuckDB and Marimo
argument-hint: <source_name>
---

# test_inspect_data

Explore the actual data loaded by your dlt pipeline using SQL queries, interactive notebooks, and data quality checks.

**Purpose:** Verify data quality, explore data structure, and test incremental loading behavior to ensure your source is production-ready.

**Learning Goals:**
- Query data with SQL using DuckDB CLI
- Use Marimo for interactive data exploration
- Perform data quality checks
- Understand and test incremental loading
- Build confidence in data validation

## Arguments

**Required:**
- `source_name`: Source directory name (e.g., `imf_sdmx_source`)

## Prerequisites

You must have completed:
- `031_test_run_pipeline` - Pipeline ran successfully
- `032_test_inspect_metadata` - Verified schema and loads

Ask user: "What is your pipeline_name and dataset_name? (From your test configuration)"

Wait for input on both values.

## Instructions

### Step 1: Choose Your Exploration Approach

Explain: "There are three ways to explore your data:"

1. **DuckDB CLI** - Direct SQL queries in terminal (fastest, good for quick checks)
2. **Marimo Notebook** - Interactive Python notebook with visualizations (best for learning and exploration)
3. **Both** - Use CLI for quick checks, Marimo for deep dives (recommended!)

Ask user: "Which approach would you like to use?"

Provide options:
- **DuckDB CLI only** - I'm comfortable with SQL and want speed
- **Marimo notebook only** - I prefer visual, interactive tools
- **Both** - Start with CLI, then create notebook (recommended)

Proceed to the appropriate section based on their choice.

---

## Approach A: DuckDB CLI Exploration

### Step A1: Connect to Database

Explain: "DuckDB is an embedded database - no server needed! Your data is in a single `.duckdb` file. We'll connect to it with the DuckDB CLI."

First, verify the file exists:
```bash
ls -lh {pipeline_name}.duckdb
# Should show file size - bigger = more data loaded
```

Ask user: "What size is your .duckdb file?"

Context:
- < 1 MB: Small dataset (expected for 10-100 record tests)
- 1-10 MB: Medium dataset (typical for 100-500 records)
- \> 10 MB: Larger dataset (500-1000+ records or many columns)

Connect to the database:
```bash
duckdb {pipeline_name}.duckdb
```

You should see a DuckDB prompt: `D `

Explain: "You're now in an interactive SQL shell. Any SQL query you type will run against your data!"

### Step A2: List Available Tables

Explain: "Let's see what tables exist in your dataset."

```sql
.tables
```

Ask user: "How many tables do you see? What are their names (excluding `_dlt_` tables)?"

Wait for their answer.

Explain: "Each table typically corresponds to one API endpoint or resource in your source. The `_dlt_` tables are system tables we explored in the metadata step."

### Step A3: Explore Table Schemas

Explain: "Before querying, it's useful to see what columns exist in each table."

```sql
-- Show schema for a specific table
DESCRIBE {dataset_name}.{table_name};

-- Or see all tables and columns at once (DuckDB-specific)
SELECT table_name, column_name, data_type
FROM duckdb_columns()
WHERE schema_name = '{dataset_name}'
AND table_name NOT LIKE '_dlt%'
ORDER BY table_name, column_index;
```

Ask user: "Pick one of your main tables (not _dlt_). What columns does it have?"

Wait for their answer.

**Teaching moment:**

Ask user: "What do the column names tell you about the data? Do they match what you expected from the API documentation?"

### Step A4: Count Records

Explain: "Let's verify how many records were loaded into each table."

```sql
-- Count records in a specific table
SELECT COUNT(*) FROM {dataset_name}.{table_name};

-- Or list all tables (then count each separately)
SELECT table_name
FROM duckdb_tables()
WHERE schema_name = '{dataset_name}'
AND table_name NOT LIKE '_dlt%'
ORDER BY table_name;

-- Note: For row counts of each table, query them individually
-- DuckDB doesn't easily support dynamic row counting in a single query
```

Ask user: "How many records are in your main table? Is this close to the limit you set in the test script?"

**If the count is very different from expected:**
- Much higher: Pagination might be yielding batches, not rows
- Much lower: API might have returned less data than available
- Zero: Data extraction failed for that resource

### Step A5: View Sample Data

Explain: "Let's look at actual data! We'll query a few rows to see what was loaded."

```sql
-- View first 10 rows
SELECT * FROM {dataset_name}.{table_name} LIMIT 10;
```

Ask user: "What do you see in the data? Does it look correct?"

**Guide them to check:**
- Do the values make sense?
- Are there unexpected NULLs?
- Do dates/timestamps look correct?
- Are IDs, names, and other fields populated?

**Follow-up questions:**

Ask user: "Pick one interesting column. What's the range of values?"

Example queries to suggest:
```sql
-- For numeric columns
SELECT MIN(column_name), MAX(column_name), AVG(column_name)
FROM {dataset_name}.{table_name};

-- For text columns
SELECT DISTINCT column_name
FROM {dataset_name}.{table_name}
LIMIT 20;

-- For timestamp columns
SELECT MIN(created_at), MAX(created_at)
FROM {dataset_name}.{table_name};
```

### Step A6: Check Data Quality

Explain: "Data quality checks help catch issues early. Let's run some basic checks."

Ask user: "What's one thing you want to verify about your data?"

Common checks:
1. **NULL check**: Are required fields populated?
2. **Uniqueness**: Are IDs unique?
3. **Date ranges**: Are timestamps reasonable?
4. **Referential integrity**: Do foreign keys match?

Provide queries based on their interest:

**1. NULL check:**
```sql
-- Count NULLs in important columns
SELECT
    COUNT(*) as total_rows,
    SUM(CASE WHEN important_column IS NULL THEN 1 ELSE 0 END) as null_count,
    ROUND(100.0 * SUM(CASE WHEN important_column IS NULL THEN 1 ELSE 0 END) / COUNT(*), 2) as null_percentage
FROM {dataset_name}.{table_name};
```

**2. Uniqueness check:**
```sql
-- Check for duplicate IDs
SELECT id, COUNT(*) as count
FROM {dataset_name}.{table_name}
GROUP BY id
HAVING COUNT(*) > 1;
-- If empty result: all IDs are unique (good!)
```

**3. Date range check:**
```sql
-- Verify timestamps are reasonable
SELECT
    MIN(created_at) as earliest,
    MAX(created_at) as latest,
    MAX(created_at) - MIN(created_at) as date_range
FROM {dataset_name}.{table_name};
```

Ask user: "What did you discover from the data quality checks?"

### Step A7: Exit DuckDB CLI

When done exploring:
```sql
.quit
```

Or press `Ctrl+D`.

Ask user: "Did the CLI exploration help you understand your data? Any surprises?"

Proceed to Step B (Incremental Loading Test) or if they chose "Both approaches," proceed to Approach B (Marimo).

---

## Approach B: Marimo Notebook Exploration

### Step B1: Install Marimo

Explain: "Marimo is a reactive Python notebook - when you change one cell, dependent cells auto-update. It's perfect for data exploration!"

```bash
# Ensure virtual environment is activated
source .venv/bin/activate

# Install marimo and required data dependencies
# numpy and pandas are needed for displaying query results
python -m pip install marimo numpy pandas pyarrow
```

Ask user: "Did Marimo and data dependencies install successfully? (Check for any error messages)"

### Step B2: Create Exploration Notebook

Explain: "We'll create a Marimo notebook pre-configured for exploring your dlt pipeline data."

Create file: `explore_{source_name}_data.py`

```python
import marimo

__generated_with__ = "0.13.9"
app = marimo.App(width="medium", app_title="{source_name} Data Explorer")


@app.cell
def __():
    import marimo as mo
    import dlt
    import duckdb
    import pandas as pd
    import numpy as np
    return dlt, duckdb, mo, np, pd


@app.cell
def _(mo, np, pd):
    # Verify required dependencies
    try:
        mo.callout(
            mo.md(f"""
            **Dependencies Loaded Successfully** ✓

            - marimo: Interactive notebooks
            - dlt: Pipeline management
            - duckdb: SQL database
            - pandas {pd.__version__}: DataFrames for displaying results
            - numpy {np.__version__}: Array operations
            """),
            kind="success"
        )
    except Exception as e:
        mo.callout(
            mo.md(f"""
            **Missing Dependencies**

            Error: {str(e)}

            Install with: `pip install numpy pandas pyarrow marimo`
            """),
            kind="danger"
        )
    return


@app.cell
def _(mo):
    mo.md("""
    # {source_name} Data Explorer

    This notebook helps you explore data loaded by your dlt pipeline.

    **Pipeline**: `{pipeline_name}`
    **Dataset**: `{dataset_name}`
    """)
    return


@app.cell
def _(dlt, mo):
    # Connect to dlt pipeline
    import os
    pipeline = None
    try:
        pipeline = dlt.attach(pipeline_name="{pipeline_name}", pipelines_dir=".")
        mo.callout(
            mo.md(f"""
            **Pipeline Connected Successfully** ✓

            - Pipeline: `{pipeline.pipeline_name}`
            - Destination: `{pipeline.destination.destination_name}`
            - Dataset: `{pipeline.dataset_name}`
            - Working dir: `{os.getcwd()}`
            """),
            kind="success"
        )
    except Exception as e:
        mo.callout(
            mo.md(f"""**Error loading pipeline:** {str(e)}

            Working directory: `{os.getcwd()}`

            Make sure you've run the pipeline first with `031_test_run_pipeline`.
            """),
            kind="danger"
        )
        pipeline = None
    return (pipeline,)


@app.cell
def _(mo, pipeline):
    # Create Ibis connection (for datasources panel)
    ibis_con = None
    if pipeline:
        try:
            ibis_con = pipeline.dataset().ibis(read_only=True)
            mo.callout(
                mo.md("""
                **Ibis Backend Connected** ✓

                Look at the **left sidebar** → **datasources panel** to see your tables!
                You can drag tables into cells to explore them.
                """),
                kind="success"
            )
        except Exception as e:
            mo.callout(
                mo.md(f"**Error creating Ibis connection:** {str(e)}"),
                kind="danger"
            )
            ibis_con = None
    else:
        mo.callout(
            mo.md("**Pipeline not loaded** - Cannot create Ibis connection"),
            kind="warning"
        )
    return (ibis_con,)


@app.cell
def _(duckdb):
    # Direct DuckDB connection (alternative to ibis)
    duckdb_con = duckdb.connect('{pipeline_name}.duckdb', read_only=True)
    return (duckdb_con,)


@app.cell
def _(mo):
    mo.md("""
    ## How to Use This Notebook

    1. **Datasources Panel**: Look at the **left sidebar** to see your tables
    2. **Drag & Drop**: Drag a table from the sidebar into a cell to explore it
    3. **Run SQL Queries**: Use `mo.sql()` in cells below
    4. **Select Connection**: Use `duckdb_con` or `ibis_con` in the dropdown

    ### Example: Run a SQL Query

    ```python
    result = mo.sql(
        "SELECT * FROM {dataset_name}.your_table_name LIMIT 10",
        engine=duckdb_con
    )
    ```

    The result will display as an interactive table!

    ### DuckDB System Functions

    For metadata queries, use DuckDB's built-in functions:
    - `duckdb_tables()` - list tables and schemas
    - `duckdb_columns()` - list columns and types
    - These are more reliable than `information_schema` in DuckDB

    ### Important Note on Dependencies

    This notebook requires numpy and pandas to display query results.
    If you see "Cannot return numpy.ndarray" errors, install: `pip install numpy pandas`
    """)
    return


@app.cell
def _(duckdb_con, mo):
    # List all tables in the dataset using DuckDB system catalog
    tables_list = mo.sql(
        f"""
        SELECT
            table_name,
            estimated_size as approx_row_count
        FROM duckdb_tables()
        WHERE schema_name = '{dataset_name}'
        AND table_name NOT LIKE '_dlt%'
        ORDER BY table_name
        """,
        engine=duckdb_con
    )
    mo.md(f"## Tables in `{dataset_name}` schema")
    tables_list
    return (tables_list,)


@app.cell
def _(mo):
    mo.md("""
    ## Your Exploration Starts Here!

    Add cells below this one to explore your data.

    **Tips:**
    - Use `mo.sql()` to run SQL queries
    - Results are interactive tables you can sort/filter
    - Create visualizations with `mo.ui.altair_chart()` or `mo.ui.plotly()`
    - Add markdown cells with `mo.md()` to document findings
    """)
    return


if __name__ == "__main__":
    app.run()
```

Ask user: "Notebook created! Ready to launch it?"

### Step B3: Launch Marimo

```bash
marimo edit explore_{source_name}_data.py
```

This opens in your browser at `http://localhost:2719`.

Explain: "The notebook will open in your browser. Look for:"
- Pipeline connection success message (green)
- Datasources panel on the LEFT sidebar
- Tables listed with row counts

Ask user: "Do you see the notebook in your browser? What tables are listed?"

Wait for their response.

### Step B4: Explore with Datasources Panel

**Teaching moment:**

Explain: "The datasources panel (left sidebar) is your data catalog. You can:"
- See all tables and their schemas
- Drag tables into cells to preview them
- Click columns to add them to queries

Ask user: "Try dragging one of your tables from the sidebar into a new cell. What happens?"

Wait for their response.

Expected: The table data appears as an interactive grid.

### Step B5: Write SQL Queries in Marimo

Explain: "Let's write a custom SQL query. Add a new cell in Marimo and try this:"

Example query to share:
```python
# Query sample data
sample_data = mo.sql(
    f"SELECT * FROM {dataset_name}.your_table_name LIMIT 10",
    engine=duckdb_con
)
sample_data
```

Ask user: "Create a cell with a SQL query. What question about your data do you want to answer?"

Suggestions:
- "How many records are in each table?"
- "What's the date range of the data?"
- "What are the most common values in column X?"

Wait for them to try a query.

Ask user: "What did you discover?"

### Step B6: Check Data Quality in Notebook

Explain: "Let's add data quality checks as notebook cells."

Guide them to create cells for:

**1. NULL check:**
```python
null_check = mo.sql(
    f"""
    SELECT
        COUNT(*) as total,
        SUM(CASE WHEN column_name IS NULL THEN 1 ELSE 0 END) as nulls
    FROM {dataset_name}.table_name
    """,
    engine=duckdb_con
)
null_check
```

**2. Summary statistics:**
```python
stats = mo.sql(
    f"""
    SELECT
        COUNT(*) as total_records,
        COUNT(DISTINCT id) as unique_ids,
        MIN(created_at) as earliest_date,
        MAX(created_at) as latest_date
    FROM {dataset_name}.table_name
    """,
    engine=duckdb_con
)
stats
```

Ask user: "Add these quality checks to your notebook. What do the results tell you?"

### Step B7: Save and Share Notebook

Explain: "Your notebook is auto-saved as `explore_{source_name}_data.py`. You can:"
- Reopen it anytime with `marimo edit explore_{source_name}_data.py`
- Share it with teammates (it's just a Python file!)
- Version control it with git

Ask user: "When you're done exploring, press Ctrl+C in the terminal to stop the notebook server."

---

## Step C: Test Incremental Loading (Both Approaches)

Ask user: "Does your source support incremental loading? (Loading only new/updated data on subsequent runs)"

Provide options:
- **Yes** - Source has incremental configuration
- **No** - Source loads all data every time (full refresh)
- **Not sure** - Let me check

**If "Not sure":**
Guide them: "Check your source code in `{source_name}/__init__.py` for `dlt.sources.incremental` or parameters like `start_date`, `since`, `last_modified`."

**If "No":**
Skip to Step D (Final Summary).

**If "Yes":**

### Step C1: Understand Incremental Loading

Explain: "Incremental loading is a key dlt feature. Instead of loading all data every run, dlt:"
1. Tracks a cursor (e.g., last timestamp, last ID) in state
2. On next run, only requests data AFTER that cursor
3. Updates the cursor with the latest value

This makes pipelines efficient for large datasets!

Ask user: "What cursor does your source use? (timestamp, ID, sequence number?)"

### Step C2: Check Current State

Explain: "Let's see what cursor value dlt saved after your first run."

```bash
dlt pipeline {pipeline_name} state
```

Or query the state table:
```sql
-- In DuckDB CLI or Marimo
SELECT * FROM {dataset_name}._dlt_pipeline_state;
```

Ask user: "Do you see a cursor value in the state? What field is it tracking?"

Common patterns:
- `"last_timestamp": "2024-01-26T12:00:00Z"`
- `"last_id": 12345`
- `"cursor": "abc123xyz"`

### Step C3: Run Pipeline Again (Incremental Test)

Explain: "Let's run the pipeline a second time. With incremental loading, it should:"
- Load LESS data (only new records since first run)
- Run FASTER (fewer API calls)
- Show incremental behavior in logs

```bash
# Ensure virtual environment is activated
source .venv/bin/activate

# Run the same test script again
python test_{pipeline_name}.py
```

Ask user: "Run the test script again and watch the logs. What do you notice?"

**Expected behaviors:**
- Logs show "Incremental loading starting from [cursor]"
- Fewer records loaded (maybe 0 if no new data)
- Faster execution time
- Load completes successfully

Ask user: "How many records were loaded this time? Compare to your first run."

### Step C4: Verify State Updated

Check the state again:
```bash
dlt pipeline {pipeline_name} state
```

Ask user: "Did the cursor value change? What's the new value?"

Expected: The cursor should now reflect the latest data from the second run.

**Teaching moment:**

Ask user: "What would happen if you deleted the `.dlt/pipelines/{pipeline_name}/` directory and ran the pipeline again?"

Answer: dlt would start from scratch (initial_value), loading all data again. The state tracks "where we left off."

### Step C5: Verify No Duplicates

Query the data to confirm no duplicates:
```sql
-- Check for duplicate IDs (should return 0 rows)
SELECT id, COUNT(*)
FROM {dataset_name}.{table_name}
GROUP BY id
HAVING COUNT(*) > 1;
```

Ask user: "Did you find any duplicates?"

Expected: None (dlt should merge or replace based on primary key).

If duplicates exist: Check your source's `write_disposition` and `primary_key` configuration.

---

## Step D: Final Summary and Reflection

### Celebrate Your Progress!

Ask user: "You've now completed the full testing workflow! How do you feel about your pipeline?"

Provide reflection prompts:
- "What was the most surprising thing you learned about your data?"
- "What data quality issue would you fix first?"
- "Do you feel confident this pipeline is ready for production?"

### Review Checklist

Ask user to confirm:

- [ ] **Data loaded successfully**: Tables have expected row counts
- [ ] **Schema is correct**: Columns and types match expectations
- [ ] **Data quality is acceptable**: No major issues with NULLs, duplicates, or invalid values
- [ ] **Incremental loading works**: (If applicable) Second run loaded only new data
- [ ] **No errors in logs**: Pipeline runs cleanly without warnings

Ask user: "Which of these checkboxes can you mark as complete?"

---

## Next Steps

Ask user: "Based on your exploration, what would you like to do next?"

Provide options:

### Option 1: Fix Issues Found
If they found data quality or schema issues:
- "What specific issue do you want to fix first?"
- Guide them to modify source code, re-run pipeline, and re-test

### Option 2: Increase Test Data Size
If everything looks good with small sample:
- "Try running with a larger limit (e.g., 500 or 1000 records)"
- "More data often reveals edge cases and quality issues"

### Option 3: Iterate on Source Implementation
If they want to add features:
- "What feature would you like to add? (new endpoints, better pagination, error handling?)"
- Refer them back to planning/implementation prompts

### Option 4: Deploy to Production
If pipeline is production-ready:
- "Next step: Deploy your pipeline to a production environment"
- "You'll need to:"
  - Configure production credentials
  - Set up a production destination (not DuckDB)
  - Schedule the pipeline (e.g., Airflow, cron)
  - Monitor loads and set up alerts

### Option 5: Document and Share
- "Create documentation for your source (README, usage examples)"
- "Share findings with your team"
- "Add tests to CI/CD pipeline"

Ask user: "Which path makes the most sense for your project?"

---

## Cleanup (Optional)

Ask user: "Would you like to clean up test files?"

**Files to keep:**
- `test_{pipeline_name}.py` - Reusable test script
- `explore_{source_name}_data.py` - Notebook for future exploration
- `.venv/` - Virtual environment for development

**Files to clean up:**
- `{pipeline_name}.duckdb` - Test database (can be regenerated)
- `.dlt/pipelines/{pipeline_name}/` - Pipeline state (resets incremental loading)

Provide options:
- **Keep everything** - Useful for continued testing
- **Remove .duckdb file** - Start fresh on next run
- **Remove pipeline state** - Test incremental loading from scratch

---

## Troubleshooting

**Issue: "Cannot connect to DuckDB file"**
- File doesn't exist: Run `031_test_run_pipeline` first
- File locked: Close any other DuckDB connections or notebooks
- Permission error: Check file permissions with `ls -la {pipeline_name}.duckdb`

**Issue: "Table not found in queries"**
- Wrong dataset name: Verify with `dlt pipeline {pipeline_name} info`
- Schema prefix missing: Use `{dataset_name}.table_name` not just `table_name`
- Table not created: Check `dlt pipeline {pipeline_name} schema` for table names

**Issue: "information_schema.tables does not exist" in DuckDB**
- DuckDB's `information_schema` may not be available in all contexts
- **Solution**: Use DuckDB-specific system functions instead:
  - `duckdb_tables()` - list all tables
  - `duckdb_columns()` - list all columns
  - `SHOW TABLES` - simple table listing
- These functions are always available and more reliable than `information_schema`

**Issue: "Marimo notebook won't start"**
- Marimo not installed: `pip install marimo`
- Port 2719 in use: Try `marimo edit --port 2720 explore_{source_name}_data.py`
- Browser didn't open: Manually visit `http://localhost:2719`

**Issue: "Cannot return a numpy.ndarray if NumPy is not present"**
- Missing data dependencies: Marimo and DuckDB need numpy and pandas for displaying results
- **Solution**: Install required packages: `pip install numpy pandas pyarrow`
- Restart the notebook after installation: `Ctrl+C` then `marimo edit explore_{source_name}_data.py`
- These packages are essential for converting query results to interactive tables

**Issue: "No data in tables" or "0 row count"**
- Pipeline ran but extracted nothing: Check source code for yield statements
- API returned empty response: Check API authentication and parameters
- Wrong table name: Verify table names in schema

**Issue: "Incremental loading not working"**
- State not saved: Check `.dlt/pipelines/{pipeline_name}/` exists
- Wrong cursor configuration: Verify `cursor_path` in source code
- API doesn't support filtering: Some APIs always return all data

**Issue: "Duplicate data after incremental run"**
- Missing primary key: Source needs `primary_key` configured
- Wrong write disposition: Should be "merge" for incremental, not "append"
- API returns overlapping data: Need to adjust incremental parameters

---

## Learning Resources

Want to learn more?

- **DuckDB SQL**: https://duckdb.org/docs/sql/introduction
- **Marimo docs**: https://docs.marimo.io/
- **dlt incremental loading**: https://dlthub.com/docs/general-usage/incremental-loading
- **Data quality patterns**: https://dlthub.com/docs/general-usage/schema

---

## Final Reflection

Before finishing, ask user:

1. "What's one thing you learned about data testing today?"
2. "If you had to explain incremental loading to a colleague, what would you say?"
3. "What's the most important data quality check for YOUR specific use case?"
4. "On a scale of 1-10, how confident are you that this pipeline is correct?"

Thank them for their thoughtful exploration!

---
