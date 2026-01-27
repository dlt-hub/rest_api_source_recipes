---
description: Generate incremental loading appendix for complex state management scenarios
argument-hint: <api_name_or_url> <research_file_path> [--output <file_path>]
---

# plan_incremental_load

Generate focused incremental loading appendix for APIs with complex state management requirements.

**Purpose:** Use ONLY when incremental loading has complex state management needs. For standard incremental patterns (simple timestamp or ID cursor), use `011_plan_dlt_rest_client.md` which includes comprehensive incremental loading coverage.

**When to use this prompt:**
- Complex cursor management (compound keys, multiple cursors)
- Mixed write dispositions with interdependencies
- Custom state management logic
- Backfill strategies with incremental loading
- Multi-stage incremental extraction

**When NOT to use this prompt:**
- Simple timestamp-based incremental (covered in 011)
- Simple ID-based incremental (covered in 011)
- Standard merge/append/replace patterns (covered in 011)

## Arguments

**Required:**
- `api_name_or_url`: API name (e.g., `github`, `salesforce`) OR URL to API documentation
- `research_file_path`: Path to API research document or scaffold code

**Optional:**
- `--output <file_path>`: Save to (default: `specs/YYYY-MM-DD_014_spec_incremental_load_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_014_spec_incremental_load_imf.md`

## Quick Reference

- **Main Spec:** See `011_plan_dlt_rest_client.md` for standard incremental patterns
- **dlt Patterns:** See `00_prime/000_dlt_context.md` for incremental loading and write dispositions
- **Write Dispositions:** append (immutable), replace (full refresh), merge (upsert)
- **State Management:** dlt tracks cursor values automatically

## Workflow

See `_workflow_template.md` for standard planning workflow.

### Analysis Focus

1. **Confirm complexity justifies appendix:**
   - Is state management complex (compound cursors, multiple states)? → YES, continue
   - Are there interdependencies between resources? → YES, continue
   - Is custom state logic required? → YES, continue
   - Is this standard timestamp/ID cursor? → NO, use 011 instead

2. **Document complex state management requirements:**
   - Compound cursor keys (multiple fields)
   - Multi-stage incremental extraction
   - Backfill strategies with incremental
   - Cross-resource state dependencies
   - Custom state initialization/reset logic

3. **Identify edge cases and special handling:**
   - State recovery scenarios
   - Partial load handling
   - State corruption recovery
   - Cursor overflow/wraparound

## Incremental Loading Appendix Template

**Output Format:** Append to main specification from `011_plan_dlt_rest_client.md`

```markdown
# APPENDIX: {API Name} - Complex Incremental Loading Details

**Note:** This appendix provides detailed guidance for complex incremental loading scenarios. For standard incremental patterns, refer to the Incremental Loading section in the main specification.

## Incremental Complexity

**Why this requires detailed appendix:**
{Explain what makes incremental loading complex: compound cursors, interdependencies, custom state logic}

## Complex State Management

### {Complex Endpoint Name}

**Why Complex:**
{Explain the complexity: compound keys, multi-stage, backfill, etc.}

**Compound Cursor Configuration:**
{If using multiple cursor fields}
```python
@dlt.resource(
    name="{resource_name}",
    write_disposition="merge",
    primary_key=["field1", "field2"]
)
def get_{resource_name}(
    cursor1: dlt.sources.incremental[str] = dlt.sources.incremental(
        "field1",
        initial_value="{value}"
    ),
    cursor2: dlt.sources.incremental[str] = dlt.sources.incremental(
        "field2",
        initial_value="{value}"
    )
):
    params = {
        "cursor1": cursor1.last_value,
        "cursor2": cursor2.last_value
    }
    for page in client.paginate("/{endpoint}", params=params):
        yield page
```

**State Initialization Strategy:**
{Custom logic for initializing state}

**State Reset Conditions:**
{When/why state needs to be reset}

## Resource Interdependencies

**Dependency Chain:**
{If resources depend on each other's state}

1. {Resource A} loads first, establishes state
2. {Resource B} uses state from A
3. {Resource C} combines states from A and B

**Implementation:**
{Code example showing interdependent resource state management}

## Backfill Strategy

**Backfill Requirements:**
{When/why backfill is needed}

**Backfill Configuration:**
```python
# Initial backfill from specific date
@dlt.resource(
    name="{resource_name}_backfill",
    write_disposition="append"
)
def backfill_{resource_name}():
    # Load historical data without state
    for page in client.paginate("/{endpoint}", params={"from": "{backfill_start}"}):
        yield page

# Then switch to incremental
@dlt.resource(
    name="{resource_name}",
    write_disposition="merge",
    primary_key="id"
)
def get_{resource_name}(
    updated_since: dlt.sources.incremental[str] = dlt.sources.incremental(
        "updated_at",
        initial_value="{current_date}"
    )
):
    params = {"updated_since": updated_since.last_value}
    for page in client.paginate("/{endpoint}", params=params):
        yield page
```

## Additional Test Cases

Beyond standard incremental tests in main spec, add these complex scenario tests:

```python
def test_compound_cursor_state_management():
    """Test multiple cursors tracked correctly"""

def test_resource_interdependency():
    """Test dependent resources coordinate state"""

def test_backfill_then_incremental():
    """Test backfill followed by incremental loads"""

def test_state_recovery_after_failure():
    """Test state recovery from partial load"""

def test_state_reset_conditions():
    """Test state reset when conditions met"""
```

## Edge Cases & Troubleshooting

**{Edge Case 1: State Corruption}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to recover}

**{Edge Case 2: Cursor Overflow}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to handle}

**{Edge Case 3: Partial Load}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to resume}

## State Recovery Procedures

**Manual State Reset:**
```bash
# Reset state for specific resource
dlt pipeline {pipeline_name} reset-state --resource {resource_name}
```

**Inspect Current State:**
```python
import dlt
pipeline = dlt.pipeline(pipeline_name="{pipeline_name}")
state = pipeline.state
print(state["sources"]["{source_name}"]["resources"]["{resource_name}"])
```

## Reference

- API Incremental Documentation: {url}
- dlt Advanced Incremental: https://dlthub.com/docs/general-usage/incremental-loading#advanced-usage
```

## Validation Checklist

Before generating appendix:
- [ ] Confirmed incremental complexity justifies separate appendix
- [ ] Complex state management requirements documented
- [ ] Compound cursor configuration detailed (if applicable)
- [ ] Resource interdependencies mapped (if applicable)
- [ ] Backfill strategy specified (if needed)
- [ ] Additional test cases beyond main spec identified
- [ ] State recovery procedures documented
- [ ] References to main spec incremental section included

## Output Message

```
Incremental loading appendix complete: {api_name}
Appended to: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

Complexity: {Compound cursors / Interdependencies / Backfill / Custom state}
Complex resources: {number}
Additional test cases: {number}

This appendix extends the Incremental Loading section in the main specification with:
- Complex state management details
- Compound cursor configuration (if applicable)
- Resource interdependency handling (if applicable)
- Backfill strategy (if needed)
- Additional test scenarios for complex cases
- State recovery procedures
- Edge case handling and troubleshooting
```

## Next Steps

**Check if other appendices are needed:**

| Aspect | Condition | Action |
|--------|-----------|--------|
| **Authentication** | OAuth2 with refresh, multi-step, or custom auth? | → Use `01_plan/012_plan_authentication.md` |
| | Simple API key or Bearer token? | → Skip (already covered) |
| **Pagination** | Different strategies per endpoint, or custom pagination logic? | → Use `01_plan/013_plan_pagination.md` |
| | Same strategy for all endpoints? | → Skip (already covered) |
| **Retries** | Response content-based retry, multi-tier limits, or circuit breaker? | → Use `01_plan/015_plan_retries.md` |
| | Standard retry patterns? | → Skip (already covered) |

**If no other appendices needed: Proceed to implementation**

Use prompt: `02_implement/02_implement.md` with the updated spec file.
