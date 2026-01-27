# Plan Phase Prompt Template

Template structure for all plan phase prompts in the dlt pipeline development workflow.

## Overview

This template defines the structure for planning prompts. The main planning prompt is `011_plan_dlt_rest_client.md`, which generates comprehensive specifications. Specialized prompts (012-015) are used only for complex scenarios and generate appendices.

## Critical Rules

### Planning Only - No Implementation

1. Generate specifications only - never write implementation code
2. Output to `specs/YYYY-MM-DD_{number}_spec_{topic}_{api_name}.md` following naming pattern
3. No Python execution, no pipeline creation, no API calls
4. Focus on architecture, design decisions, and test specifications

### Required Components

1. **Test-Driven Development**
   - Test specifications before implementation specifications
   - Given/when/then format with mock responses
   - Reference shared test structure from main spec

2. **Generic API Compatibility**
   - Work for ANY REST API (use placeholders: `{api_name}`, `{endpoint_name}`)
   - Support multiple strategies per API

3. **Implementation Phases**
   - Follow 9-phase structure from 011 (see Implementation Phase Structure below)
   - Sequential, testable tasks with checkboxes
   - Group by phase: Setup, Auth, Pagination, Resources, Incremental, Retry, Source, Pipeline, Docs

## Prompt Structure

```markdown
---
description: {Brief description}
argument-hint: <required_args> [--optional <args>]
---

# {Prompt Name}

{One paragraph purpose}

## Arguments

**Required:**
- `{arg_name}`: {Description}

**Optional:**
- `--output <file_path>`: Save to (default: `specs/YYYY-MM-DD_{number}_spec_{topic}_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_013_spec_pagination_imf_sdmx.md`

## Quick Reference

- {Key dlt pattern 1}
- {Key dlt pattern 2}
- {Key dlt pattern 3}

## Workflow

### Step 1: Parse and Load
1. Validate required arguments
2. Load research/context files
3. Ask clarifying questions

### Step 2: Analyze Requirements
4. Identify {key aspects}
5. Map to dlt patterns

### Step 3: Design Tests (TDD)
6. Create test cases (happy path, edge cases, errors)
7. Design mock responses

### Step 4: Create Specification
8. Design implementation approach
9. Document configuration
10. Write specification document

### Step 5: Validate and Output
11. Validate completeness
12. Present to user for confirmation
13. Generate specification and task list

## Output
```

## Specification Template Structure

### For Main Specification (011)

See `011_plan_dlt_rest_client.md` for the comprehensive specification template. Main specs include:

1. **Summary** - Overview and architecture
2. **Constants & Configuration** - Shared values and config files
3. **Authentication** - Full auth setup with tests
4. **Pagination Strategies** - Per-endpoint pagination with decision matrix
5. **Incremental Loading Strategies** - Per-endpoint write disposition and cursors
6. **Retry & Error Handling** - Default retry config and error patterns
7. **Resources** - Per-resource details referencing above sections
8. **Source** - dlt.source wrapper with usage examples
9. **Test Structure** - Complete test file organization and fixtures
10. **Task Breakdown** - 9 phases with specific tasks

### For Appendix Specifications (012-015)

Appendices extend the main spec with complex scenario details:

```markdown
# APPENDIX: {API Name} - {Topic} Details

**Note:** This appendix extends the {Topic} section in the main specification.

## Complexity Rationale
{Why this requires detailed appendix}

## Detailed Configuration/Implementation
{Complex patterns, custom logic, special cases}

## Additional Test Cases
{Tests beyond main spec for complex scenarios}

## Edge Cases & Troubleshooting
{Complex edge cases and solutions}

## Reference
{API-specific documentation links}
```

## Implementation Phase Structure

All main specifications follow this 9-phase breakdown:

### Phase 1: Setup
- Project structure, virtual environment, dependencies
- Configuration files (.dlt/secrets.toml, .dlt/config.toml)
- Test setup (pytest, conftest.py with fixtures)

### Phase 2: Authentication (TDD)
- Write authentication tests
- Implement auth class configuration
- Verify credential management

### Phase 3: Pagination Configuration (TDD)
- Write pagination tests per endpoint
- Configure paginator per endpoint
- Verify last page detection

### Phase 4: Resources - Data Structure (TDD)
- Write data structure tests per resource
- Implement RESTClient configuration
- Verify data selectors and schema

### Phase 5: Incremental Loading (TDD)
- Write incremental tests per resource
- Implement write disposition and cursor
- Verify state management

### Phase 6: Retry & Error Handling (TDD)
- Write retry tests
- Configure retry behavior
- Verify error handling

### Phase 7: Source Integration (TDD)
- Write source-level tests
- Implement @dlt.source wrapper
- Verify resource coordination

### Phase 8: Pipeline Testing (Integration)
- Write end-to-end tests
- Test with destination (DuckDB dev_mode)
- Verify data integrity and incremental runs

### Phase 9: Documentation
- README with setup instructions
- Usage examples
- Troubleshooting guide

## Validation Checklist

Before finalizing:
- [ ] Planning only (no implementation code)
- [ ] Output to `specs/YYYY-MM-DD_{number}_spec_{topic}_{api_name}.md` (following naming pattern)
- [ ] TDD test specifications included
- [ ] Task breakdown included
- [ ] Generic (works for any API)
- [ ] Uses dlt built-in features

## Output Message

```
{Topic} specification complete: {api_name}
Saved to: specs/YYYY-MM-DD_{number}_spec_{topic}_{api_name}.md

Summary:
- Test cases: {number}
- Tasks: {number} across {N} phases

Next: Review and proceed to implementation phase
```
