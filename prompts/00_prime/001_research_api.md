---
description: Research external API documentation to extract dlt pipeline requirements
argument-hint: <api_name_or_url> [--output <file_path>]
---

# research_api

Research external API documentation for a data source and extract all information required to build a dlt REST API ingestion pipeline.

**Purpose:** This generates a research document that informs the implementation of a dlt REST API source. Use this when dltHub scaffolds do NOT exist for the API.

**Note:** If a dltHub scaffold exists for this API, you can skip this research step and leverage the scaffold instead.

## Arguments

**Required:**
- `api_name_or_url`: API name (e.g., `imf`, `axiom`) OR direct URL to API documentation

**Optional:**
- `--output <file_path>`: Save to (default: `research/YYYY-MM-DD_001_research_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_001_research_[api_name].md`
  - Example: `2026-01-26_001_research_imf.md`

## Quick Reference: dlt Best Practices

- Use dlt's built-in `rest_api_source()` - do NOT plan for custom API clients
- Check for existing dltHub scaffolds first - they eliminate the need for this research
- Document pagination/auth patterns compatible with dlt's built-in utilities
- Verify actual API responses when possible - docs may be outdated

## Workflow

### Step 1: Understand Requirements

1. Parse `api_name_or_url` argument (error if missing)

2. Ask user clarifying questions:
   - Which specific endpoints/resources to ingest?
   - Expected data volume and update frequency?
   - Authentication constraints?
   - Primary use case?
   - Incremental loading requirements?

### Step 2: Quick Scaffold Check

3. Search for dltHub scaffold:
   - WebSearch: "dlthub {api_name} verified source"
   - If found: Inform user that a scaffold exists (they can skip this research and use the scaffold directly)
   - If not found: Proceed to external API research

### Step 3: Research External API Documentation

4. **Locate API documentation:**
   - Use URL provided by user, or search for official docs
   - Verify documentation is publicly accessible

5. **Verify actual API responses when possible:**
   - Test requests to confirm response format (row vs columnar)
   - Verify field names, nesting, pagination structure
   - Note discrepancies between docs and actual responses

6. **Extract key API information:**

   **Authentication:**
   - Methods: API key, Bearer token, OAuth, Basic Auth
   - Credential location: headers, query params, body
   - Scopes/permissions required

   **Pagination (check per endpoint - strategies may differ):**
   - Type: offset, page, cursor, link header, or none
   - Parameters: limit/offset, page/per_page, cursor/next_cursor, etc.
   - Last page detection: empty results, total count, etc.

   **Rate Limiting:**
   - Limits: requests per second/minute/hour
   - Headers: X-RateLimit-*, Retry-After
   - Recommended backoff strategy

   **Endpoints & Resources:**
   - Base URL(s)
   - Available GET endpoints
   - Resource hierarchies (parent/child relationships)
   - Query parameters (filters, sorting)

   **Data Structure:**
   - Response format: JSON, XML, etc.
   - Data location in response (e.g., `data`, `results`, `items`)
   - Field types and datetime formats
   - Nested objects and arrays

   **Incremental Loading:**
   - Timestamp fields: `created_at`, `updated_at`, etc.
   - Query parameters: `since`, `after`, etc.
   - ID fields for cursor-based incremental

   **Error Handling:**
   - HTTP status codes and meanings
   - Error response structure
   - Retry guidance

7. **Recommend dlt configuration:**
   - Write disposition per resource: `append`, `replace`, or `merge`
   - Primary keys for merge resources
   - Incremental strategy: timestamp-based, cursor-based, or full refresh

### Step 4: Validate & Create Output

8. Present findings to user:
   - Summarize key information
   - List endpoints and recommendations
   - Confirm before writing output file

9. **Create output document:**
   - Default path: `research/YYYY-MM-DD_001_research_{api_name}.md`
   - Or use `--output` path if provided
   - Create directory if needed

## Output Document Structure

```markdown
# {API Name} - API Research

Date: {YYYY-MM-DD}
Source: {API Documentation URL}

## Summary
[Brief overview of the API and its purpose]

## Authentication
- **Method:** [API Key / Bearer Token / OAuth / Basic Auth]
- **Location:** [Header / Query Parameter]
- **Example:**
```python
"auth": {
    "type": "...",
    ...
}
```

## Endpoints

### {Endpoint Name 1}
- **Path:** `/path/to/endpoint`
- **Method:** GET
- **Purpose:** [What data this returns]
- **Query Parameters:**
  - `param1`: Description
  - `param2`: Description

**Pagination:**
- Type: [offset / page / cursor / link / none]
- Parameters: [e.g., limit/offset, cursor, page/per_page]
- Last page detection: [e.g., empty results, total field]

**Sample Response:**
```json
{
  "data": [...],
  "pagination": {...}
}
```

**Data Selector:** `data` (JSONPath to the array of items)

**Incremental Loading:**
- Strategy: [Timestamp-based / Cursor-based / Full refresh]
- Timestamp field: `updated_at` (ISO 8601)
- Query parameter: `since` or `updated_after`

**dlt Recommendations:**
- Write disposition: `merge`
- Primary key: `["id"]`
- Incremental cursor: `updated_at`

[Repeat for each endpoint]

## Rate Limiting
- **Limits:** X requests per minute
- **Headers:** `X-RateLimit-Remaining`, `Retry-After`
- **Strategy:** Exponential backoff on 429 responses

## Error Handling
- **401:** Unauthorized - check credentials
- **403:** Forbidden - check permissions/scopes
- **429:** Rate limit - backoff and retry
- **500/502/503:** Server error - retry with backoff

## Additional Notes
[API quirks, special considerations, edge cases]

## Reference Links
- [API Documentation URL]
- [Authentication Guide URL]
```

## Validation Checklist

Before finalizing the research document:
- [ ] Checked for dltHub scaffolds first
- [ ] Extracted authentication details (method, location, scopes)
- [ ] Documented pagination per endpoint (strategies may differ across endpoints)
- [ ] Identified incremental loading strategy (timestamp/cursor/full refresh)
- [ ] Included sample responses with actual API data
- [ ] Documented rate limits and retry strategy
- [ ] Documented error codes and error handling
- [ ] Provided dlt configuration recommendations (write disposition, primary keys)
- [ ] Verified actual API responses when possible (docs may be outdated)
- [ ] Noted API quirks, edge cases, and special considerations

## Key Principles

- **Check for scaffolds first:** If a dltHub scaffold exists, you can skip this research and use the scaffold directly
- **Be specific:** Include exact parameter names, field names, formats
- **Verify responses:** Test actual API responses when possible - docs may be outdated
- **Check pagination per endpoint:** Different endpoints may use different pagination strategies
- **Think dlt:** Frame findings in terms of dlt patterns (rest_api_source, RESTClient, paginators, incremental)
- **Document edge cases:** Note API quirks, rate limits, and special behaviors
- **Get user confirmation:** Ask clarifying questions and validate findings before writing output

## Output

Save research document to specified path in the `/research` folder and confirm with:

```
Research complete: {api_name}
Saved to: research/YYYY-MM-DD_001_research_{api_name}.md

Summary:
- Endpoints: {number}
- Auth method: {type}
- Pagination strategies: {types}
- Incremental endpoints: {number}
```

## Next Steps

**RECOMMENDED: Proceed to main specification**

Use prompt: `01_plan/011_plan_dlt_rest_client.md` with the research file you just created.

The main spec prompt (011) covers:
- Standard authentication (API key, Bearer token, Basic auth)
- Standard pagination (single_page, offset, cursor, page_number)
- Standard incremental loading (timestamp/ID cursor)
- Standard retry logic (5xx, 429 with exponential backoff)

This is sufficient for most APIs. Optional appendix prompts (012-015) are only needed for complex scenarios.
