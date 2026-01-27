---
description: Overview of local testing workflow for dlt pipelines
---

# Test Ingestion Locally - Overview

This directory contains prompts for testing dlt ingestion pipelines locally with DuckDB.

**Purpose:** Validate pipeline implementation through an interactive, educational testing workflow that covers environment setup, execution, metadata inspection, and data exploration.

## Testing Workflow

The testing process is split into three sequential prompts to create an interactive learning experience:

### 1. Run Pipeline (`031_test_run_pipeline.md`)

**Prepare environment and execute pipeline**

- Validate environment and check authentication requirements
- Setup virtual environment with uv
- Configure pipeline parameters (pipeline_name, dataset_name)
- Choose test data size (10, 100, 500, or 1000 records)
- Create and run test script
- Verify successful execution

**Next Step:** Inspect pipeline metadata

**Usage:**
```
/031_test_run_pipeline <source_name>
```

### 2. Inspect Metadata (`032_test_inspect_metadata.md`)

**Explore pipeline metadata, schema, and load history**

- Use dlt CLI commands (info, loads, schema)
- Understand dlt's metadata tracking
- View schema and table structures
- Check load history and status
- Optional: Launch interactive dlt dashboard
- Learn the difference between metadata and data

**Next Step:** Inspect loaded data

**Usage:**
```
/032_test_inspect_metadata <source_name>
```

### 3. Inspect Data (`033_test_inspect_data.md`)

**Query and validate loaded data**

- Choose exploration approach (DuckDB CLI, Marimo notebook, or both)
- Query data with SQL
- Perform data quality checks (NULLs, duplicates, ranges)
- Create interactive exploration notebooks
- Test incremental loading behavior
- Decide on next steps (fix issues, deploy, iterate)

**Next Step:** Fix issues, increase test size, or deploy to production

**Usage:**
```
/033_test_inspect_data <source_name>
```

## Why Three Prompts?

**Interactive Learning:** Each prompt asks questions and waits for user input, creating an engaging educational experience.

**Clear Progression:** Users move through distinct phases:
1. Setup → Run → Verify success
2. Understand structure → Check metadata → Verify loads
3. Explore data → Check quality → Test incremental loading

**Manageable Scope:** Each prompt focuses on one aspect of testing, avoiding overwhelming users with too many steps at once.

**Flexible Workflow:** Users can stop after any prompt and resume later, or repeat prompts as needed during development.

## Learning Goals

By completing all three prompts, users will:

- ✓ Understand dlt's local testing workflow
- ✓ Know how to configure and run pipelines
- ✓ Learn what metadata dlt tracks and why it matters
- ✓ Be able to query and validate data with SQL
- ✓ Understand incremental loading and state management
- ✓ Build confidence in data quality validation
- ✓ Know when a pipeline is production-ready

## Prerequisites

Before starting:
- Source code implemented in `{source_name}/` directory
- API credentials configured in `.dlt/secrets.toml` (if required)
- `uv` package manager installed
- `duckdb` CLI installed (optional, for direct database access)

## Quick Start

For a complete testing session:

```bash
# 1. Run the pipeline
/031_test_run_pipeline imf_sdmx_source

# 2. Inspect metadata
/032_test_inspect_metadata imf_sdmx_source

# 3. Inspect data
/033_test_inspect_data imf_sdmx_source
```

## Tips for Best Experience

**Follow the order:** The prompts build on each other - complete them sequentially on your first time.

**Take your time:** Each prompt includes reflection questions - use them to deepen understanding.

**Experiment:** After completing the workflow once, try different test sizes, configurations, and queries.

**Save your work:** Test scripts and Marimo notebooks can be reused throughout development.

**Ask questions:** The prompts are designed to be interactive - engage with the questions and exercises.

## Related Prompts

- **Planning:** `/01_plan/` - Create implementation specifications
- **Implementation:** `/02_implement/` - Build dlt sources (future)
- **Deployment:** (future) - Deploy to production environments

---
