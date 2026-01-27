---
description: Generate pagination appendix for complex or mixed pagination strategies
argument-hint: <api_name_or_url> <research_file_path> [--output <file_path>]
---

# plan_pagination

Generate focused pagination appendix for APIs with complex or mixed pagination strategies.

**Purpose:** Use ONLY when pagination is complex or varies significantly per endpoint. For standard pagination (single_page, simple offset, simple cursor), use `011_plan_dlt_rest_client.md` which includes comprehensive pagination coverage.

**When to use this prompt:**
- Different endpoints use different pagination strategies
- Custom pagination logic not covered by standard dlt paginators
- Complex cursor-based pagination with special handling
- Hybrid pagination (e.g., cursor + offset)
- Pagination with conditional logic or edge cases

**When NOT to use this prompt:**
- All endpoints use same pagination strategy (covered in 011)
- Standard single_page, offset, or cursor patterns (covered in 011)
- Simple page_number pagination (covered in 011)

## Arguments

**Required:**
- `api_name_or_url`: API name (e.g., `imf`, `axiom`) OR URL to API documentation
- `research_file_path`: Path to API research document (e.g., `research/imf_api_research.md`)

**Optional:**
- `--output <file_path>`: Save to (default: `specs/YYYY-MM-DD_013_spec_pagination_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_013_spec_pagination_imf.md`

## Quick Reference

- **Main Spec:** See `011_plan_dlt_rest_client.md` for standard pagination patterns
- **dlt Patterns:** See `00_prime/000_dlt_context.md` for pagination configuration
- **Pagination Types:** single_page, page_number, offset, header_link, json_link, cursor, header_cursor

## Workflow

See `_workflow_template.md` for standard planning workflow.

### Analysis Focus

1. **Confirm complexity justifies appendix:**
   - Do different endpoints use different strategies? → YES, continue
   - Is pagination custom/non-standard? → YES, continue
   - Is this standard single strategy across all? → NO, use 011 instead

2. **Document per-endpoint pagination differences:**
   - Which endpoints use which pagination type
   - Why each endpoint requires specific strategy
   - Parameter differences between endpoints
   - Last page detection strategy per endpoint

3. **Identify edge cases and special handling:**
   - Conditional pagination logic
   - Hybrid pagination patterns
   - API-specific pagination quirks
   - Fallback strategies

## Pagination Appendix Template

**Output Format:** Append to main specification from `011_plan_dlt_rest_client.md`

```markdown
# APPENDIX: {API Name} - Complex Pagination Details

**Note:** This appendix provides detailed guidance for complex pagination scenarios. For standard pagination patterns, refer to the Pagination section in the main specification.

## Pagination Complexity

**Why this requires detailed appendix:**
{Explain what makes pagination complex: mixed strategies, conditional logic, non-standard patterns}

## Per-Endpoint Pagination Strategy

**Summary Table:**
| Endpoint | Paginator Type | Complexity | Notes |
|----------|----------------|------------|-------|
| `/endpoint1` | offset | Standard | Default config |
| `/endpoint2` | cursor + custom | Complex | See details below |
| `/endpoint3` | hybrid | Complex | Uses both offset and cursor |

## Complex Endpoint Details

### {Complex Endpoint Name}

**Why Complex:**
{Explain the complexity: hybrid approach, conditional logic, custom handling}

**Detailed Configuration:**
```python
"{endpoint}": {
    "path": "{path}",
    "paginator": {
        "type": "{type}",
        {detailed_config_params}
    }
}
```

**Special Handling Required:**
{Describe any custom logic, fallbacks, or edge case handling}

**Request Flow:**
1. {Step-by-step pagination flow}
2. {Including any conditional logic}
3. {Fallback strategies}

**Response Parsing:**
```json
{
  {Show actual response structure with annotations}
}
```

**Last Page Detection Strategy:**
- Primary: {method}
- Fallback: {method}
- Edge case: {handling}

## Additional Test Cases

Beyond standard pagination tests in main spec, add these complex scenario tests:

```python
def test_{endpoint}_hybrid_pagination():
    """Test hybrid pagination switches between strategies"""

def test_{endpoint}_conditional_pagination_logic():
    """Test pagination adapts based on response"""

def test_{endpoint}_pagination_fallback():
    """Test fallback when primary pagination fails"""
```

## Implementation Guidance

**Custom Paginator (if needed):**
{If standard dlt paginators don't work, provide guidance on custom implementation}

**Conditional Logic:**
{Provide code examples for conditional pagination behavior}

## Edge Cases & Troubleshooting

**{Edge Case 1}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to fix}

**{Edge Case 2}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to fix}

## Reference

- API Pagination Documentation: {url}
- dlt Custom Paginators: https://dlthub.com/docs/general-usage/http/rest-client#implementing-a-custom-paginator
```

## Validation Checklist

Before generating appendix:
- [ ] Confirmed pagination complexity justifies separate appendix
- [ ] Per-endpoint strategies documented with rationale
- [ ] Complex pagination flow detailed step-by-step
- [ ] Additional test cases beyond main spec identified
- [ ] Edge cases and troubleshooting documented
- [ ] Custom paginator guidance provided (if needed)
- [ ] References to main spec pagination section included

## Output Message

```
Pagination appendix complete: {api_name}
Appended to: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

Complexity: {Mixed strategies / Custom logic / Hybrid}
Complex endpoints: {number}
Additional test cases: {number}

This appendix extends the Pagination section in the main specification with:
- Per-endpoint pagination strategy breakdown
- Complex pagination flow details
- Custom pagination logic (if applicable)
- Additional test scenarios for complex cases
- Edge case handling and troubleshooting
```

## Next Steps

**Check if other appendices are needed:**

| Aspect | Condition | Action |
|--------|-----------|--------|
| **Authentication** | OAuth2 with refresh, multi-step, or custom auth? | → Use `01_plan/012_plan_authentication.md` |
| | Simple API key or Bearer token? | → Skip (already covered) |
| **Incremental** | Compound cursors, interdependencies, or backfill strategies? | → Use `01_plan/014_plan_incremental_load.md` |
| | Simple timestamp or ID cursor? | → Skip (already covered) |
| **Retries** | Response content-based retry, multi-tier limits, or circuit breaker? | → Use `01_plan/015_plan_retries.md` |
| | Standard retry patterns? | → Skip (already covered) |

**If no other appendices needed: Proceed to implementation**

Use prompt: `02_implement/02_implement.md` with the updated spec file.
