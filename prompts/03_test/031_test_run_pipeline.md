---
description: Prepare environment and run dlt pipeline locally with DuckDB
argument-hint: <source_name>
---

# test_run_pipeline

Prepare your local environment and execute a dlt ingestion pipeline with DuckDB for testing.

**Purpose:** Sets up the testing environment, configures the pipeline, and runs it with a limited dataset to validate the implementation works correctly.

**Learning Goals:**
- Understand dlt's virtual environment requirements
- Learn about pipeline configuration (pipeline_name, dataset_name)
- Experience running a dlt pipeline locally
- Practice limiting data for efficient testing

## Arguments

**Required:**
- `source_name`: Source directory name (e.g., `imf_sdmx_source`)

## Instructions

### Step 1: Validate Environment

**A. Check source directory:**

Ask user: "Let's verify your source directory exists. What is the name of your source directory?"

Once provided, check:
```bash
ls -la {source_name}/
# Should contain: __init__.py, README.md
```

If directory doesn't exist or is missing files, stop and inform the user to fix the source structure first.

**B. Understand authentication requirements:**

Ask user: "Does your API require authentication (API keys, tokens, OAuth, etc.)?"

Provide options:
- **Yes** - API requires authentication
- **No** - API is public/open, no authentication needed

**If YES (authentication required):**

Explain: "Authentication credentials in dlt are stored in `.dlt/secrets.toml`. This file should never be committed to git (it's in .gitignore). It keeps your API keys and tokens secure."

Ask user: "Have you already configured your credentials in `.dlt/secrets.toml`?"

**CRITICAL**: Never read `.dlt/secrets.toml` or any credential files. Always ask the user to verify.

If they say NO:
- Guide them to create/update `.dlt/secrets.toml`
- Explain what credentials they need (based on source documentation)
- Ask them to confirm when configured
- Wait for confirmation before proceeding

If they say YES:
- Ask: "Can you confirm the credential keys match what your source expects? Check your source's `__init__.py` for required secret names."
- Wait for confirmation

**If NO (no authentication):**

Explain: "Great! Public APIs don't require credentials. Your source will work without any `.dlt/secrets.toml` configuration for authentication. You can proceed directly to setup."

### Step 2: Setup Virtual Environment with uv

Explain: "dlt requires a clean Python environment. We'll use `uv` (ultra-fast Python package manager) to create an isolated virtual environment. This prevents dependency conflicts with other projects."

Ask user: "Do you already have a virtual environment activated, or should we create a new one?"

Provide options:
- **Create new** - Set up fresh virtual environment
- **Use existing** - Already have .venv activated
- **Not sure** - Show me how to check

**If "Create new":**

```bash
# Create virtual environment with uv
uv venv

# Activate the virtual environment
source .venv/bin/activate
```

Explain: "The `uv venv` command creates a `.venv` directory with an isolated Python environment. The `source .venv/bin/activate` command activates it - you'll see `(.venv)` in your terminal prompt."

**If "Use existing":**

Verify it's activated:
```bash
which python
# Should show path containing .venv
```

**If "Not sure":**

```bash
# Check if venv is active
which python
# If output contains '.venv', you're good
# If not, create and activate: uv venv && source .venv/bin/activate
```

**Install dependencies:**

Explain: "Now we'll install dlt with DuckDB support. DuckDB is a fast, embedded database perfect for local testing - no server required!"

```bash
# Install dlt with DuckDB support
uv pip install "dlt[duckdb]"

# Install source-specific dependencies (if pyproject.toml exists)
if [ -f pyproject.toml ]; then
    uv pip install -e "."
fi
```

**Verify installation:**

```bash
# Check dlt is installed
python -c "import dlt; print(f'dlt version: {dlt.__version__}')"
```

Ask user: "Did you see the dlt version printed? If yes, we're ready to proceed!"

### Step 3: Configure Pipeline Parameters

Explain: "Every dlt pipeline needs two key identifiers: a pipeline_name (identifies this pipeline's state and data) and a dataset_name (the schema/namespace for tables in the database)."

**A. Find the pipeline name:**

Ask user: "What is the `pipeline_name` defined in your source code?"

Guide them: "Look in `{source_name}/__init__.py` for calls to `dlt.pipeline()` or check your source's README. The pipeline_name is typically something like `'{source_name}_pipeline'` or `'{api_name}'`."

Wait for user input.

Validate: If they're unsure, offer to help them find it:
```bash
# Search for pipeline_name in source code
grep -r "pipeline_name" {source_name}/
```

**B. Choose the dataset name:**

Ask user: "What dataset name should we use for the tables?"

Provide options:
- **Use default** - `{source_name}_raw` (recommended)
- **Custom name** - Specify your own (e.g., `test_data`, `{api_name}_staging`)

Explain: "The dataset_name becomes the schema in the database. Tables will be created as `{dataset_name}.table_name`. Using `_raw` suffix is a common pattern for raw extracted data."

**CRITICAL - Check Source Name:**

Ask user: "Let's verify the source name parameter. Look in `{source_name}/__init__.py` for the `@dlt.source` decorator. Does it specify a `name` parameter?"

```bash
grep -A 1 "@dlt.source" {source_name}/__init__.py
```

Expected output: `@dlt.source(name="something")`

**If NO name parameter found (shows just `@dlt.source`):**

Explain: "Your source doesn't specify a name, which means dlt will use the function name as the schema name. This can cause a mismatch with the dataset_name and create dashboard errors."

Ask user: "Would you like me to fix this by adding the name parameter to match your dataset_name?"

Provide options:
- **Yes, fix it** - Update the source decorator
- **No, I'll handle it** - User will fix manually

If "Yes, fix it":
- Update `@dlt.source` to `@dlt.source(name="{dataset_name}")`
- Explain: "This ensures the source schema name matches the pipeline dataset name, preventing metadata mismatches"

**If name parameter exists:**

Verify: "The source name is `{found_name}`. This MUST match your dataset_name."

If they don't match:
- Warn: "⚠️  MISMATCH DETECTED: Source name `{found_name}` doesn't match dataset name `{chosen_dataset_name}`"
- Explain: "This will cause the dashboard and queries to fail because data will be in one schema but dlt will look in another"
- Ask: "Should we use `{found_name}` as the dataset_name to match the source?"

### Step 4: Choose Test Data Size

Explain: "For local testing, we limit the data volume to keep things fast and manageable. You'll use dlt's `add_limit()` function to control how many records to ingest."

Ask user: "How many records would you like to ingest for this test run?"

Provide options:
- **10 records** - Quick validation (2-3 seconds, perfect for testing if pipeline runs)
- **100 records** - Standard testing (5-10 seconds, good for exploring data structure)
- **500 records** - Extended testing (20-30 seconds, better sample for data quality checks)
- **1000 records** - Maximum (1-2 minutes, comprehensive local dataset)

Explain the trade-off: "Fewer records = faster iteration for debugging. More records = better data quality validation. Start small (10-100) until everything works, then increase."

Wait for user selection.

### Step 5: Create Test Script

Explain: "We'll create a Python script that configures and runs your pipeline. This script can be reused for testing throughout development."

Create file: `test_{pipeline_name}.py` in the root directory:

```python
import dlt
import sys
from pathlib import Path

# Add source directory to path for imports
sys.path.insert(0, str(Path(__file__).parent))

from {source_name} import {source_name}_source

# Configuration
RECORD_LIMIT = {record_limit}  # {selected_option_description}
PIPELINE_NAME = "{pipeline_name}"
DATASET_NAME = "{dataset_name}"

print(f"=" * 60)
print(f"Testing dlt pipeline: {PIPELINE_NAME}")
print(f"Dataset: {DATASET_NAME}")
print(f"Record limit: {RECORD_LIMIT}")
print(f"=" * 60)

# Create pipeline with DuckDB destination
pipeline = dlt.pipeline(
    pipeline_name=PIPELINE_NAME,
    destination="duckdb",
    dataset_name=DATASET_NAME,
    progress="log"  # Shows progress bar during loading
)

# Run the source with record limit
print(f"\nInitializing {source_name}_source()...")
data = {source_name}_source()

# Apply limit to all resources
print(f"Applying limit of {RECORD_LIMIT} records...")
data = data.add_limit(RECORD_LIMIT, count_rows=True)

# Execute pipeline
print(f"\nRunning pipeline...\n")
info = pipeline.run(data)

# Summary
print(f"\n" + "=" * 60)
print(f"Pipeline completed successfully!")
print(f"=" * 60)
print(f"Loads: {len(info.loads_ids)}")
print(f"Dataset: {DATASET_NAME}")
print(f"DuckDB file: {PIPELINE_NAME}.duckdb")
print(f"\nRecords were limited to maximum of {RECORD_LIMIT} for local testing")
print(f"=" * 60)
```

Ask user: "I've created the test script. Ready to review it?"

Show them the script and explain key parts:
- **Line 8**: Imports your source function
- **Line 11-13**: Configuration you just chose
- **Line 17-22**: Creates a dlt pipeline pointing to DuckDB
- **Line 25-29**: Initializes your source and applies the record limit
- **Line 32**: Executes the pipeline (this is where the magic happens!)

### Step 6: Run the Pipeline

Ask user: "Ready to run your first local pipeline test?"

Explain: "This will execute your source, extract data from the API, and load it into a local DuckDB database file. You'll see progress logs and a summary when complete."

```bash
# Ensure virtual environment is activated
source .venv/bin/activate

# Run the pipeline
python test_{pipeline_name}.py
```

Wait for execution.

### Step 7: Verify Success

After execution, ask user: "What happened? Did you see:"

**Success indicators to look for:**
- [ ] "Initializing {source_name}_source()..." message
- [ ] "Applying limit of X records..." message
- [ ] "Running pipeline..." followed by progress logs
- [ ] "Pipeline completed successfully!" message
- [ ] "Loads: 1" (or more)
- [ ] "DuckDB file: {pipeline_name}.duckdb" message
- [ ] No red error messages

Ask user: "Did you see all of these success indicators?"

Provide options:
- **Yes, all good!** - Pipeline ran successfully
- **Saw errors** - Let's troubleshoot
- **Unsure** - Let me check

**If "Yes, all good!":**

Verify the DuckDB file was created:
```bash
ls -lh {pipeline_name}.duckdb
# Should show a file with size > 0 bytes
```

Ask user: "Can you confirm the .duckdb file exists and has a size greater than 0 bytes?"

**If "Saw errors":**

Ask user: "What error message did you see? Please share the last 10-20 lines of output."

Common issues:
- **Import errors**: Virtual environment not activated or dependencies missing
- **Authentication errors**: Credentials missing or incorrect in `.dlt/secrets.toml`
- **API errors**: Rate limiting, invalid endpoints, or API down
- **Timeout errors**: API too slow, increase timeout or reduce limit

Guide them through the specific error.

**If "Unsure":**

Help them check:
```bash
# Check if DuckDB file exists
ls -lh *.duckdb 2>/dev/null || echo "No .duckdb files found"

# Check dlt logs (if any errors occurred)
ls -la .dlt/
```

### Step 8: Celebrate and Reflect

If successful, congratulate them!

Ask user: "How did that feel? What did you learn from running the pipeline?"

Reflection questions:
- "Did the pipeline run faster or slower than you expected?"
- "How many records were actually loaded? (Check the logs)"
- "What surprised you about the process?"

---

## Summary

After completing this prompt, you should have:

- [x] Virtual environment setup with dlt and dependencies installed
- [x] Test script: `test_{pipeline_name}.py`
- [x] DuckDB file: `{pipeline_name}.duckdb` with limited test data
- [x] Verified the pipeline runs without errors

**You now have a working local pipeline! 🎉**

---

## Next Step: Inspect Pipeline Metadata

Ask user: "Now that your pipeline has run successfully, would you like to inspect the pipeline metadata? This will show you:"
- Pipeline run history
- Schema (tables and columns created)
- Load information (when data was loaded, how much)

Provide options:
- **Yes, let's inspect metadata** - Run `/031_test_inspect_metadata {source_name}`
- **Not yet** - I want to run the pipeline again with different settings
- **Skip to data inspection** - Jump directly to exploring the loaded data

Explain: "Inspecting metadata helps you understand what dlt created in the database. It's like looking at the 'receipt' for your data load. This is especially useful for debugging schema issues."

---

## Troubleshooting

**Issue: "Virtual environment activation failed"**
- Check if `uv` is installed: `which uv`
- If not installed: `pip install uv` or `brew install uv` (macOS)
- Try creating manually: `python -m venv .venv`

**Issue: "Import error - cannot import {source_name}_source"**
- Verify `{source_name}/__init__.py` exists
- Check if virtual environment is activated: `which python` should show `.venv`
- Check the source exports the function: `grep "def.*_source" {source_name}/__init__.py`

**Issue: "Authentication failed" or "401/403 errors"**
- If authentication required: Verify `.dlt/secrets.toml` has correct credentials
- Check credential key names match source code expectations
- For public APIs: This shouldn't happen - might be API rate limiting

**Issue: "No data loaded" or "Loaded 0 records"**
- Check API endpoint is accessible: Try the URL in a browser or with `curl`
- Review source code for filtering that might exclude all data
- Try increasing record limit - some sources have pagination issues with small limits
- Check for API rate limiting or temporary downtime

**Issue: "Too much data loaded" (more than limit)**
- Source might be yielding batches: Check if `count_rows=True` is in `add_limit()`
- Some sources count "pages" not "records" - this is a source implementation issue
- For now, just note the actual count for data inspection

**Issue: "Pipeline too slow"**
- Reduce record limit (try 10 records)
- Check if API has rate limiting (add delays in source if needed)
- Some APIs are just slow - be patient!

---
