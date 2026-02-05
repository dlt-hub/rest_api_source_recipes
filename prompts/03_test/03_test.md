# Test Pipeline

Test your dlt REST API pipeline to verify it works correctly.

## Setup

1. **Create/activate virtual environment:**
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

2. **Install dependencies:**
```bash
pip install dlt
pip install dlt[duckdb]  # Or your destination: [bigquery], [postgres], etc.
```

3. **Configure secrets in `.dlt/secrets.toml`:**
```toml
[sources.your_api_source]
api_key = "your_actual_api_key_here"
# Add other credentials as needed
```

## Run Pipeline

4. **Start with limited data for testing:**

Use dlt's built-in `add_limit()` to test with a small sample first:

```python
import dlt
from dlt.sources.rest_api import rest_api_source

# Test with limited records first
source = rest_api_source({...})
source.add_limit(10)  # Fetch only 10 records for testing

pipeline = dlt.pipeline(...)
pipeline.run(source)
```

Or if using RESTClient directly:
```python
from dlt.sources.helpers.rest_client import RESTClient

client = RESTClient(...)
# The paginate() method supports max_pages parameter
for page in client.paginate(..., max_pages=1):
    # Process first page only
    pass
```

5. **Execute pipeline script:**
```bash
python your_pipeline_script.py
```

6. **Verify successful load:**
Look for output like:
```
Pipeline your_pipeline load step completed in X.XX seconds
1 load package(s) were loaded to destination duckdb and into dataset your_dataset
```

7. **After successful limited test, remove limit for full load:**

```python
source = rest_api_source({...})
# Remove or comment out: source.add_limit(10)
pipeline.run(source)
```

## Inspect Results

### Check Schema

8. **View pipeline schema:**
```bash
dlt pipeline <pipeline_name> schema
```

This shows:
- Tables created
- Column names and types
- Primary keys
- Write dispositions

### Query Data

9. **For DuckDB destination:**

**Option A - Use dlt CLI:**
```bash
dlt pipeline <pipeline_name> show
```

**Option B - Use DuckDB CLI:**
```bash
duckdb <pipeline_name>.duckdb
```

Then query:
```sql
SHOW TABLES;
SELECT * FROM your_table LIMIT 10;
SELECT COUNT(*) FROM your_table;
```

10. **For other destinations:**
Use destination-specific tools:
- **BigQuery:** Use BigQuery Console or `bq` CLI
- **Postgres:** Use `psql` or database client
- **Snowflake:** Use Snowflake Console or CLI

### Verify Incremental Loading

11. **Test incremental behavior (if applicable):**

**Run 1 - Initial load:**
```bash
python your_pipeline_script.py
# Note record count
```

**Run 2 - Incremental load:**
```bash
python your_pipeline_script.py
# Check if only new/updated records loaded
```

**Query to verify:**
```sql
-- Check state is maintained
SELECT MAX(updated_at) FROM your_table;

-- Check for duplicates (shouldn't exist with merge)
SELECT id, COUNT(*) FROM your_table GROUP BY id HAVING COUNT(*) > 1;
```

## Common Issues

### Authentication Failures

**Symptom:** 401 or 403 errors

**Fix:**
- Verify `.dlt/secrets.toml` has correct credentials
- Check credential format matches specification
- Ensure API key/token is active and has required permissions

### Schema Changes

**Symptom:** "Column X does not exist" or schema evolution warnings

**Fix:**
```bash
# Inspect current schema
dlt pipeline <pipeline_name> schema

# If needed, drop and reload (development only)
dlt pipeline <pipeline_name> drop
python your_pipeline_script.py
```

### Data Quality Issues

**Symptom:** Missing data, incorrect counts, or unexpected values

**Fix:**
- Query destination to inspect actual data
- Check pagination is working (all pages loaded)
- Verify data selector extracts correct fields
- Check incremental cursor values

### Rate Limiting

**Symptom:** 429 errors or slow performance

**Fix:**
- dlt's RESTClient handles retries automatically
- Check API documentation for rate limits
- Add delays if needed (though RESTClient handles this)

## Quick Validation Checklist

- [ ] Test with limited data first (use `add_limit(10)`)
- [ ] Pipeline runs without errors on sample
- [ ] Expected tables created in destination
- [ ] Schema matches expected structure
- [ ] Data values look correct (spot check sample)
- [ ] Run full pipeline without limit
- [ ] Row counts match expectations
- [ ] Primary keys set correctly (for merge)
- [ ] Incremental loading works (run twice, verify state)
- [ ] No duplicate records (for merge disposition)

## Next Steps

Once testing is complete:
- Deploy to production environment
- Set up scheduling (cron, Airflow, etc.)
- Monitor pipeline runs
- Set up alerting for failures
