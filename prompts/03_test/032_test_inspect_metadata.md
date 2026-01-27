---
description: Inspect dlt pipeline metadata, schema, and load history
argument-hint: <source_name>
---

# test_inspect_metadata

Explore your dlt pipeline's metadata, schema, and load history using dlt's built-in CLI tools and interactive dashboard.

**Purpose:** Learn what dlt tracks about your pipeline, understand the schema it created, and verify loads completed successfully through both command-line tools and visual exploration.

**Learning Goals:**
- Understand dlt's pipeline metadata (state, schema, loads)
- Learn to use dlt CLI commands for inspection
- **Master the interactive dlt dashboard for visual exploration (CRITICAL)**
- Recognize the difference between pipeline metadata and actual data
- Build confidence in understanding dlt's internal tracking

## Arguments

**Required:**
- `source_name`: Source directory name (e.g., `imf_sdmx_source`)

## Prerequisites

You must have already completed `031_test_run_pipeline` and have:
- A `.duckdb` file in your directory
- The pipeline_name used in your test

Ask user: "What is your pipeline_name? (This is what you used in the test script, e.g., `imf_sdmx_pipeline`)"

Wait for input.

## Instructions

### Step 1: Understand Pipeline Metadata

Explain: "dlt tracks metadata about every pipeline run. This includes:"
- **Pipeline info**: Configuration, destination, dataset name
- **Schema**: Tables created, columns, data types, constraints
- **Loads**: When data was loaded, how much, which load IDs
- **State**: For incremental sources, the last cursor/checkpoint

Ask user: "Think of metadata as the 'control panel' for your pipeline. It tells you what happened, but not the actual data. Ready to explore?"

### Step 2: View Pipeline Info

Explain: "Let's start with the basic pipeline information. This shows you the pipeline configuration and current state."

```bash
dlt pipeline {pipeline_name} info
```

Ask user: "Please run the command above. What do you see?"

**Guide them to look for:**
- **Pipeline name**: Should match what you used
- **Destination**: Should show `duckdb`
- **Dataset name**: The schema you configured
- **Pipeline location**: Where dlt stores pipeline files (`.dlt/pipelines/`)
- **Last run**: Timestamp of your test run

Ask user: "Do all these values look correct based on your test configuration?"

Provide options:
- **Yes, looks correct** - Great, let's continue!
- **Something looks wrong** - Let's discuss what's unexpected
- **I don't understand a field** - Ask me about it

If they have questions, explain:
- **Pipeline location**: dlt stores state in `.dlt/pipelines/{pipeline_name}/`. This tracks incremental loading state.
- **Destination**: Where the data goes (`duckdb` = local database file)
- **Dataset name**: The schema/namespace - tables are created as `{dataset_name}.{table_name}`

### Step 2.5: Verify Schema Name Consistency (CRITICAL)

**IMPORTANT CHECK:** Before inspecting loads, verify there's no schema mismatch.

Explain: "Let's make sure the source schema name matches the dataset name. A mismatch here causes dashboard errors where data exists but queries fail."

```bash
# Check what dlt thinks the schema name is
dlt pipeline {pipeline_name} info | grep -i "schema"
```

Ask user: "What schema name is reported? Does it match your dataset_name from the test script?"

**Expected:** Schema name should match the dataset_name you configured.

**If they DON'T match:**

Explain: "⚠️  SCHEMA MISMATCH DETECTED! This is the root cause of dashboard query errors."

Common scenario:
- Source has: `@dlt.source` (no name parameter) → schema name = function name (e.g., `imf_sdmx_source`)
- Pipeline has: `dataset_name="imf_sdmx_source_raw"` → data stored in schema `imf_sdmx_source_raw`
- Result: Data in wrong schema, queries fail

Ask user: "Would you like me to fix this mismatch?"

Provide options:
- **Yes, fix the source code** - Update `@dlt.source(name="{dataset_name}")`
- **Yes, re-run with correct dataset** - Use matching dataset_name in test script
- **No, I'll investigate first** - Let's continue inspection

**If "Yes, fix the source code":**
- Update the source decorator
- Explain: "You'll need to re-run the pipeline with `--dev-mode` to reset the schema"

**If they MATCH:**

Confirm: "✓ Schema name matches dataset name. No mismatch issues!"

### Step 3: View Load History

Explain: "Every time you run a pipeline, dlt creates a 'load'. Each load has a unique ID and tracks what was loaded. Let's see your load history."

```bash
dlt pipeline {pipeline_name} loads
```

Ask user: "Run the command above. How many loads do you see?"

**Guide them to look for:**
- **Load IDs**: Unique timestamps like `1234567890.123456`
- **Load status**: Should show `completed` or `loaded`
- **Package IDs**: Unique identifiers for each batch of data
- **Table names**: Which tables were loaded

Ask user: "Do you see load ID(s) with status 'completed'? How many loads are there?"

Expected answer: Should be 1 load if they just ran the pipeline once.

**Follow-up questions for learning:**

Ask user: "What do you think will happen if you run the test pipeline again? Will you see:"
- A) The same single load
- B) A second load added to the list
- C) The first load gets replaced

Wait for their answer.

Explain: "The answer is B! Each pipeline run creates a new load. dlt tracks the full history. This is useful for debugging and auditing."

### Step 4: View Schema

Explain: "Now let's look at the schema - the structure of tables and columns that dlt created. This is crucial for understanding what data you can query."

```bash
dlt pipeline {pipeline_name} schema
```

Ask user: "Run the command above. The output will be long - that's okay! What do you notice?"

**Guide them to look for:**

1. **Tables section**: List of all tables created
   - Ask: "How many tables do you see? What are their names?"
   - Explain: "Most sources create multiple tables - one per API endpoint/resource, plus dlt system tables (starting with `_dlt_`)"

2. **Column definitions**: For each table, the columns and their types
   - Ask: "Pick one of your tables (not a `_dlt_` table). What columns does it have?"
   - Ask: "What data types do you see? (e.g., text, bigint, timestamp, bool)"

3. **Special annotations**: Look for keys, nullable, etc.
   - Ask: "Do you see any columns marked with `primary_key: true` or `nullable: false`?"
   - Explain: "dlt can infer schema from your data, or you can define it explicitly in your source"

4. **System tables**: Tables starting with `_dlt_`
   - Explain: "dlt creates system tables like `_dlt_loads` (load tracking) and `_dlt_version` (schema versioning). These are used internally."

**Interactive learning moment:**

Ask user: "Why do you think dlt tracks schema? What problems does this solve?"

Wait for their answer.

Explain: "Schema tracking enables:"
- **Schema evolution**: If API adds new fields, dlt can detect and migrate
- **Data validation**: Ensures loaded data matches expected types
- **Documentation**: Auto-generated schema serves as API documentation
- **Reproducibility**: You can see exactly what structure was used at any point in time

### Step 5: Understand dlt System Tables

Ask user: "Would you like to understand what the `_dlt_` system tables do?"

Provide options:
- **Yes, explain them** - I want to learn about dlt internals
- **No, skip for now** - Just tell me what I need to know

**If "Yes, explain them":**

Explain each system table:

**`_dlt_version`**: Tracks schema versions
- When schema changes (new columns, type changes), dlt increments version
- Helps with schema migrations and evolution
- You'll rarely need to query this directly

**`_dlt_loads`**: Tracks load operations
- One row per load (same info as `dlt pipeline {pipeline_name} loads`)
- Includes status, timestamps, row counts per table
- Useful for monitoring and debugging

**`_dlt_pipeline_state`**: Stores incremental loading state
- For incremental sources, tracks the last cursor/checkpoint
- Example: Last timestamp processed, last ID seen
- This is how dlt knows "where it left off"

Ask user: "Which of these do you think you'll use most during development?"

Most developers use `_dlt_loads` (to check if loads succeeded) and `_dlt_pipeline_state` (to debug incremental loading).

### Step 6: CRITICAL - Launch Interactive Dashboard

**IMPORTANT:** The dlt interactive dashboard is the BEST way to understand your pipeline metadata. It provides visual, interactive exploration that makes learning much easier than CLI output.

Explain: "dlt has a built-in web dashboard powered by Streamlit. This dashboard gives you a visual, interactive way to explore:"
- Schema with searchable tables and columns
- Load history with drill-down details
- Pipeline state visualization
- Sample data previews

**This is not just a nice-to-have - it's a powerful learning tool that will help you understand dlt's internals!**

Ask user: "Let's launch the dlt dashboard to explore your pipeline visually. Ready?"

**CRITICAL: Strongly encourage users to say YES. If they hesitate, explain the benefits:**
- "The dashboard makes it 10x easier to understand complex schemas"
- "You can see relationships between tables visually"
- "It's the fastest way to verify your pipeline structure"
- "Most dlt developers use this regularly during development"

Provide options:
- **Yes, show me!** - Launch the dashboard (RECOMMENDED)
- **Maybe later** - Skip for now (but we'll revisit this)

**If "Yes, show me!" (strongly recommended):**

```bash
dlt pipeline {pipeline_name} show
```

This will open a browser at `http://localhost:8501` (Streamlit app).

**Guide them through the dashboard - take your time here:**

1. **Overview tab**:
   - Ask: "What pipeline info do you see? Does the dataset name match your config?"
   - Point out: Recent load summary, pipeline health status

2. **Schema tab** (MOST IMPORTANT):
   - Ask: "Click on the Schema tab. How many tables do you see?"
   - Ask: "Expand one of your main tables. How is this view different from the CLI output?"
   - Point out: Collapsible tree view, data types, nullable indicators, key markers

3. **Data tab**:
   - Ask: "Click on the Data tab. Can you preview data right in the dashboard?"
   - Point out: Sample rows from each table, useful for quick validation

4. **Loads tab**:
   - Ask: "Check the Loads tab. Can you see the timestamp of your test run?"
   - Point out: Detailed load information, row counts per table

Ask user: "Take 2-3 minutes to explore the dashboard. Click through the tabs, expand tables in the schema viewer. What's your favorite feature?"

Wait for their response.

**Teaching moment:**

Ask user: "How does the visual dashboard compare to CLI commands for understanding your pipeline?"

Expected insight: Dashboard is easier for exploration, CLI is faster for quick checks.

Explain: "During development, you'll likely use both - CLI for quick verification, dashboard for deep exploration and debugging."

Ask user: "Ready to close the dashboard and continue? (Press Ctrl+C in the terminal)"

**If "Maybe later":**

Emphasize: "I strongly recommend taking 5 minutes to explore the dashboard after we finish. It will give you a much better understanding of what dlt created."

Note: "You can launch it anytime with `dlt pipeline {pipeline_name} show`"

Ask user: "Let's make a note to come back to this. Will you commit to exploring the dashboard before moving to data inspection?"

### Step 7: Compare Metadata vs. Data

**Teaching moment:**

Explain: "It's important to understand the difference between:"
- **Metadata** (what we just explored): Information *about* your pipeline
- **Data** (what we'll explore next): The actual records loaded from the API

Ask user: "Can you explain the difference in your own words?"

Wait for their answer.

Clarify:
- **Metadata = "How many tables?", "What columns?", "When did it load?"**
- **Data = "What's in row 5?", "How many users from California?", "What's the average price?"**

Ask user: "Which one do you think is more useful for debugging API issues? Metadata or data?"

Answer: Usually metadata first (to see if load succeeded, check table names), then data (to verify quality).

### Step 8: Check for Issues

Ask user: "Based on what you saw in the metadata, do you notice any issues?"

**Common things to check:**

1. **Expected tables created?**
   - Ask: "Did you expect these table names? Do they match your API endpoints/resources?"

2. **Schema looks reasonable?**
   - Ask: "Do the column types make sense? Any surprises?"

3. **Loads completed successfully?**
   - Ask: "Are all loads showing 'completed' status?"

4. **System tables present?**
   - Ask: "Do you see the `_dlt_` system tables? (You should)"

Provide options:
- **Everything looks good** - Ready to inspect data!
- **I see issues** - Let's troubleshoot
- **Unsure** - Help me understand if this is correct

**If "I see issues":**

Ask them to describe the issue and guide accordingly:
- Missing tables: Source might not be yielding that resource
- Wrong column types: Source might need type hints
- Failed loads: Check error logs in `.dlt/` directory
- No system tables: Pipeline didn't complete initialization

---

## Summary

After completing this prompt, you should understand:

- [x] How to use dlt CLI commands (`info`, `loads`, `schema`)
- [x] What metadata dlt tracks about your pipeline
- [x] The difference between pipeline metadata and actual data
- [x] How to verify pipeline runs succeeded
- [x] Where dlt stores state (`.dlt/pipelines/` directory)
- [x] **How to use the interactive dlt dashboard (CRITICAL)**

**You now understand what dlt created! 📊**

**CHECKPOINT:** Before proceeding, verify you explored the interactive dashboard.

Ask user: "Did you launch and explore the dlt dashboard (`dlt pipeline {pipeline_name} show`)?"

Provide options:
- **Yes, I explored it** - Great! Ready to move on
- **No, I skipped it** - Please go back to Step 6 and explore it now

**If "No, I skipped it":**

Strongly emphasize: "The dashboard is not optional for learning! It provides visual understanding that's essential for working with dlt. Please take 5 minutes to:"
1. Launch the dashboard: `dlt pipeline {pipeline_name} show`
2. Explore all 4 tabs (Overview, Schema, Data, Loads)
3. Expand at least 2-3 tables in the Schema tab
4. Look at sample data in the Data tab

Ask: "Ready to launch the dashboard now?"

Wait until they confirm they've explored it before proceeding.

---

## Next Step: Inspect Loaded Data

Ask user: "Now that you've verified the pipeline structure AND explored the dashboard, would you like to inspect the actual data loaded?"

Explain: "Data inspection lets you:"
- Query tables with SQL
- Verify data quality (nulls, types, ranges)
- Explore relationships between tables
- Test incremental loading behavior
- Create visualizations or reports

**CRITICAL:** If incremental loading is implemented in the pipeline, always ask the user to test it by re-running the pipeline after the first load and validating that no duplicate data got loaded

Provide options:
- **Yes, let's inspect data** - Run `/032_test_inspect_data {source_name}` (or reference the correct next prompt number)
- **Not yet** - I want to run the pipeline again first
- **I found issues** - Help me fix the metadata problems first

**If "Yes, let's inspect data":**
Proceed to next prompt: `033_test_inspect_data.md`

**If "Not yet":**
Ask: "What would you like to change? (sample size, configuration, source code?)"

**If "I found issues":**
Guide them through troubleshooting based on what they found.

---

## Troubleshooting

**Issue: "Command not found: dlt"**
- Virtual environment not activated: `source .venv/bin/activate`
- dlt not installed: `uv pip install "dlt[duckdb]"`

**Issue: "Pipeline not found"**
- Wrong pipeline name: Check the name you used in test script
- Pipeline not run yet: Go back and run `031_test_run_pipeline`
- Wrong directory: Run from the same directory as your test script

**Issue: "No loads shown"**
- Pipeline might have failed: Check for error messages in terminal
- Wrong pipeline name: Verify the name matches your test script
- No data loaded: Source might have returned 0 records

**Issue: "Schema is empty or has only system tables"**
- Source didn't yield any data: Check API connectivity
- Source function not called correctly: Verify import in test script
- Record limit too restrictive: Try increasing the limit

**Issue: "Dashboard won't open"**
- Port 8501 already in use: Close other Streamlit apps or use `dlt pipeline {pipeline_name} show --port 8502`
- Browser didn't auto-open: Manually visit `http://localhost:8501`
- Permission error: Check firewall settings

**Issue: "Dashboard shows 'Table does not exist' errors when querying"**
- **Root cause:** Schema name mismatch between source and dataset
- Check: Run `dlt pipeline {pipeline_name} info` and compare schema name to dataset_name
- Solution 1: Add `@dlt.source(name="{dataset_name}")` to match dataset
- Solution 2: Change `dataset_name` in pipeline to match source schema name
- Solution 3: Query with correct schema: `SELECT * FROM {actual_schema}.table_name`
- **Prevention:** Always specify `name` parameter in `@dlt.source` decorator to match `dataset_name`

---

## Reflection Questions

Before moving to data inspection, reflect:

1. "What surprised you most about what dlt tracks?"
2. "How could metadata be useful when deploying to production?"
3. "If your pipeline fails in production, which metadata would you check first?"
4. "What's the difference between a 'load' and a 'pipeline run' in dlt?"
5. **"How did the interactive dashboard change your understanding of the pipeline compared to CLI output alone?"** (CRITICAL reflection)

These questions help cement understanding before moving to data exploration.

**Special emphasis:** If you haven't used the dashboard yet, you're missing out on one of dlt's most powerful learning tools. Go back to Step 6 now!

---
