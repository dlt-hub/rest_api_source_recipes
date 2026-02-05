# Research API & Create Implementation Specification

Research external API documentation and generate a test-driven implementation specification for building a dlt REST API pipeline.

**Purpose:** Research an API and create a detailed implementation spec with TDD task breakdown.

## Arguments

**Required:**
- `api_name`: Name of the API to research and build (e.g., "arxiv", "github", "stripe")

**Optional:**
- `--spec-output`: Path for spec doc (default: `specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md`)
- `--destination`: Target destination (default: "duckdb" for development)

## Quick Reference

**dlt Components:**
- **Auth Classes:** `BearerTokenAuth`, `APIKeyAuth`, `HttpBasicAuth`, `OAuth2ClientCredentials`
- **Paginators:** `SinglePagePaginator`, `PageNumberPaginator`, `OffsetPaginator`, `JSONResponsePaginator`, `HeaderLinkPaginator`, `JSONResponseCursorPaginator`
- **Write Dispositions:** `append` (immutable logs/events), `replace` (full refresh), `merge` (upsert with primary key)

## Workflow

### Phase 1: Check for Existing Solution

1. **Search for dltHub verified source:**
   - Query: "dlthub {api_name} verified source"
   - If exists: Inform user they can use the scaffold instead
   - If not: Proceed with research

### Phase 2: API Research

2. **Extract key information from API documentation:**
   - **Base URL**: API base endpoint
   - **Authentication**: Method (API key/Bearer/OAuth/Basic), location (header/query/body), required credentials
   - **Endpoints**: Available GET endpoints to ingest
   - **Pagination**: Type per endpoint (offset/page/cursor/link/none), parameters, last page detection, max page size
   - **Rate Limits**: Requests per minute/hour, rate limit headers, backoff strategy
   - **Incremental Fields**: Timestamp fields (`updated_at`, `created_at`) or ID fields for cursor-based loading
   - **Data Structure**: Response format (JSON/XML/other), data location path (e.g., `data`, `results`), nested structures, field types
   - **Special Considerations**: API quirks, non-standard patterns, error handling

### Phase 3: Clarifying Questions

3. **Ask user to clarify:**
   - Implement all endpoints or subset?
   - Destination preference (duckdb for dev, other for production)?
   - Custom transformations needed (data flattening, field renaming)?
   - Specific date ranges or filters for initial load?

### Phase 4: Design Implementation (TDD)

4. **Design test-first approach:**
   - Authentication tests (valid/missing/invalid credentials)
   - Resource tests per endpoint (data structure, pagination, data selector)
   - Incremental loading tests (initial load, incremental load, state persistence)
   - Integration tests (full pipeline, schema validation)

5. **Map API patterns to dlt components:**
   - Choose appropriate auth class
   - Select paginator per endpoint
   - Design incremental loading strategy
   - Determine write dispositions and primary keys
   - Plan data transformations (if needed for XML, nested data, etc.)

6. **Output specification document:** `specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md`

## Specification Document Template

```markdown
# {API Name} - dlt REST Client Implementation Specification

**Date:** {YYYY-MM-DD}
**API Documentation:** {Documentation URL}

## API Research

### Overview

{Brief description of the API and what data it provides}

### Base URL

```
{base_url}
```

### Authentication

- **Method:** {API Key / Bearer Token / OAuth2 / HTTP Basic / None}
- **Location:** {Header / Query Parameter / Body}
- **Credential Name:** `{credential_field}` (e.g., `api_key`, `token`)
- **dlt Auth Class:** `{AuthClass}`

**Example Configuration:**
```python
from dlt.sources.helpers.rest_client.auth import {AuthClass}

auth = {AuthClass}({param}=dlt.secrets.value)
```

### Endpoints

#### {Endpoint 1 Name}

- **Path:** `{/endpoint/path}`
- **Method:** GET
- **Purpose:** {What data it returns}
- **Pagination:**
  - Type: {offset / page / cursor / link / none}
  - Parameters: `{param_names}` (e.g., `offset`, `limit` or `page`, `per_page`)
  - Max page size: {number}
  - Total count location: `{json_path}` (e.g., `meta.total`)
- **Data Selector:** `{json_path}` (e.g., `data`, `results`, `items`)
- **Response Format:** {JSON / XML / Other}
- **Incremental Loading:**
  - Field: `{field_name}` (e.g., `updated_at`, `created_at`, `id`)
  - Type: {timestamp / cursor / id}
  - Query Parameter: `{param_name}` (e.g., `since`, `updated_after`)

**Example Response:**
```json
{
  "data": [...],
  "pagination": {
    "total": 100,
    "page": 1,
    "per_page": 20
  }
}
```

[Repeat for each endpoint]

### Rate Limiting

- **Limits:** {X} requests per {minute/hour/day}
- **Headers:** `{header_names}` (e.g., `X-RateLimit-Remaining`, `X-RateLimit-Reset`)
- **Strategy:** {Exponential backoff / Fixed delay / Respect headers}
- **Recommended Delay:** {seconds} between requests

### Special Considerations

{API quirks, non-standard patterns, edge cases, version requirements, data structure notes, XML handling, etc.}

## Summary

{Brief overview: N endpoints, auth method, pagination strategies, N incremental resources}

## Architecture

**Stack:**
- dlt, Python, pytest
- Destination: {destination}

**Overview:**
- Source: `{api_name}_source`
- Resources: {N}
- Auth: {auth_type}
- Pagination: {list_paginator_types}
- Incremental: {N} endpoints with {timestamp/cursor} loading

**Special Components:**
{List any custom components needed: response hooks, custom paginators, transformations}

## File Structure

```
{api_name}_pipeline/
├── sources/
│   └── {api_name}_source.py          # Main implementation
├── tests/
│   ├── conftest.py                   # Shared fixtures
│   ├── test_client.py                # RESTClient initialization (if no auth)
│   ├── test_auth.py                  # Authentication tests (if auth required)
│   ├── test_resources.py             # Resource & pagination tests
│   ├── test_incremental.py           # Incremental loading tests (if applicable)
│   ├── test_source.py                # Source integration tests
│   └── test_pipeline.py              # End-to-end tests
├── .dlt/
│   ├── config.toml                   # Configuration
│   └── secrets.toml.example          # Credential template
├── {api_name}_pipeline.py            # Pipeline runner
└── README.md                         # Documentation
```

## Configuration

**Secrets (.dlt/secrets.toml):**
```toml
[sources.{api_name}_data]
{credential_key} = "your_credential_here"
```

**Config (.dlt/config.toml):**
```toml
[sources.{api_name}_data]
base_url = "{base_url}"
# Add any other config parameters

[runtime]
log_level = "INFO"
```

## Authentication

**Method:** {API Key / Bearer Token / OAuth2 / HTTP Basic / None}
**dlt Auth Class:** `{AuthClass}`

**Implementation:**
```python
from dlt.sources.helpers.rest_client.auth import {AuthClass}

auth = {AuthClass}({param}=dlt.secrets.value)
client = RESTClient(base_url=base_url, auth=auth)
```

**Tests:**
- Valid credentials succeed
- Missing credentials raise `ConfigFieldMissingException`
- Invalid credentials handled gracefully (401/403)

{If no auth: explain that no authentication is required and no test_auth.py needed}

## Resources

### {Resource 1 Name}

**Endpoint:** `{/path}`
**Write Disposition:** `{append/replace/merge}`
**Primary Key:** `"{field}"` (if merge)
**Data Selector:** `{json_path}`

**Pagination:**
```python
from dlt.sources.helpers.rest_client.paginators import {PaginatorClass}

paginator = {PaginatorClass}(
    {params}
)
```

**Incremental Loading (if applicable):**
```python
@dlt.resource(
    name="{name}",
    write_disposition="{disposition}",
    primary_key="{key}"
)
def get_{name}(
    client: RESTClient,
    {cursor}: dlt.sources.incremental[{type}] = dlt.sources.incremental(
        "{field}",
        initial_value="{value}"
    )
) -> Iterator[dict]:
    params = {"{api_param}": {cursor}.last_value}

    for page in client.paginate(
        "{path}",
        params=params,
        paginator=paginator
    ):
        # Extract data from response
        data = page.get("{data_path}", [])

        # Handle single item vs array
        if isinstance(data, dict):
            data = [data]

        # Apply transformations if needed
        for item in data:
            yield transform_{name}(item)  # Or just: yield item
```

**Tests:**
- Data structure validation
- Pagination fetches all pages
- Data selector extracts correct path
- Incremental loading maintains state (if applicable)
- Transformations work correctly (if applicable)

[Repeat for each resource]

## Source Implementation

```python
from dlt.sources.helpers.rest_client import RESTClient
import dlt

@dlt.source(name="{api_name}_data")
def {api_name}_source(
    {credential}: {Type} = dlt.secrets.value,  # Omit if no auth
    base_url: str = dlt.config.value
):
    """
    {API Name} source.

    Args:
        {credential}: {Description of credential}
        base_url: API base URL

    Returns:
        List of resources
    """
    # Initialize client with auth
    auth = {AuthClass}({param}={credential})
    client = RESTClient(base_url=base_url, auth=auth)

    # Return resources
    return [
        get_{resource1}(client),
        get_{resource2}(client),
    ]
```

**Pipeline Usage:**
```python
import dlt
from sources.{api_name}_source import {api_name}_source

pipeline = dlt.pipeline(
    pipeline_name="{api_name}_pipeline",
    destination="duckdb",
    dataset_name="{api_name}_data"
)

load_info = pipeline.run({api_name}_source())
print(load_info)
```

## Test Structure

### Test Files Overview

1. **conftest.py** - Shared fixtures
   - Mock API responses for each endpoint
   - Expected transformed data structures
   - Reusable test helpers

2. **test_client.py or test_auth.py** - Client initialization
   - If auth: Test authentication (valid/missing/invalid)
   - If no auth: Test RESTClient initialization

3. **test_resources.py** - Resource behavior
   - Resource structure (name, write_disposition, primary_key)
   - Pagination handling
   - Data extraction and transformation
   - Single item vs array handling

4. **test_incremental.py** - Incremental loading (if applicable)
   - Initial load behavior
   - Incremental load with cursor
   - State persistence
   - Merge disposition with primary key

5. **test_source.py** - Source integration
   - Source initialization with config
   - Returns correct resources
   - Config/secrets injection works

6. **test_pipeline.py** - End-to-end
   - Full pipeline run with mocked responses
   - Table creation in destination
   - Schema validation
   - Data count verification

### Example Test Structure

```python
# conftest.py
import pytest

@pytest.fixture
def mock_{api_name}_response():
    """Mock API response."""
    return {
        "{data_path}": [
            {"id": 1, "name": "Item 1"},
            {"id": 2, "name": "Item 2"}
        ],
        "pagination": {"total": 2}
    }

@pytest.fixture
def expected_records():
    """Expected transformed records."""
    return [
        {"id": 1, "name": "Item 1"},
        {"id": 2, "name": "Item 2"}
    ]

# test_resources.py
def test_{resource}_extraction(mock_{api_name}_response, expected_records):
    """Test resource extracts data correctly."""
    from sources.{api_name}_source import get_{resource}
    from unittest.mock import Mock

    mock_client = Mock()
    mock_client.paginate.return_value = [mock_{api_name}_response]

    resource = get_{resource}(client=mock_client)
    results = list(resource)

    assert results == expected_records

# test_pipeline.py
def test_pipeline_initial_load(temp_dir, mock_{api_name}_response):
    """Test full pipeline run."""
    from sources.{api_name}_source import {api_name}_source
    from unittest.mock import patch, Mock

    with patch('sources.{api_name}_source.RESTClient') as mock_rest:
        mock_client = Mock()
        mock_client.paginate.return_value = [mock_{api_name}_response]
        mock_rest.return_value = mock_client

        pipeline = dlt.pipeline(
            pipeline_name="test_{api_name}",
            destination="duckdb",
            dataset_name="{api_name}_test",
            pipelines_dir=temp_dir
        )

        load_info = pipeline.run({api_name}_source())
        assert not load_info.has_failed_jobs

        # Verify data loaded
        with pipeline.sql_client() as client:
            result = client.execute_query("SELECT COUNT(*) FROM {resource}").fetchone()
            assert result[0] > 0
```

## Task Breakdown (TDD Implementation Order)

### Phase 1: Project Setup
- [ ] Create directory structure (sources/, tests/, .dlt/)
- [ ] Install dependencies: `pip install dlt[{destination}] pytest pytest-mock {other_deps}`
- [ ] Create `.dlt/config.toml` with base configuration
- [ ] Create `.dlt/secrets.toml.example` with credential template
- [ ] Add `.dlt/secrets.toml`, `.dlt/*.db`, `.dlt/pipelines/` to `.gitignore`
- [ ] Create `tests/conftest.py` with mock responses

### Phase 2: Authentication (if required)
- [ ] Write `tests/test_auth.py` (valid/missing/invalid credentials)
- [ ] Run tests (expect failures)
- [ ] Implement authentication setup in source
- [ ] Verify: `pytest tests/test_auth.py -v` (expect pass)

{If no auth: Skip to Phase 3 and test RESTClient initialization in test_client.py}

### Phase 3: Resources & Pagination
For each resource:
- [ ] Write resource tests in `tests/test_resources.py`:
  - Resource structure (name, write_disposition, primary_key)
  - Data extraction from mock response
  - Pagination behavior
  - Data selector path
  - Single item vs array handling
- [ ] Run tests (expect failures)
- [ ] Implement resource function with:
  - `@dlt.resource` decorator
  - RESTClient pagination
  - Appropriate paginator
  - Data selector
  - Transformations (if needed)
- [ ] Verify: `pytest tests/test_resources.py::test_{resource}_* -v` (expect pass)

### Phase 4: Incremental Loading (if applicable)
For each incremental resource:
- [ ] Write incremental tests in `tests/test_incremental.py`:
  - Initial load behavior
  - Incremental cursor tracking
  - Merge disposition prevents duplicates
- [ ] Run tests (expect failures)
- [ ] Update resource with:
  - `write_disposition="merge"`
  - `primary_key="{field}"`
  - `dlt.sources.incremental()` decorator
  - Query parameter with cursor value
- [ ] Verify: `pytest tests/test_incremental.py -v` (expect pass)

### Phase 5: Source & Integration
- [ ] Write `tests/test_source.py`:
  - Source initialization with config/secrets
  - Returns correct resources
  - Config injection works
- [ ] Run tests (expect failures)
- [ ] Implement `@dlt.source` wrapper:
  - Parameter injection from config/secrets
  - RESTClient initialization
  - Return list of resources
- [ ] Verify: `pytest tests/test_source.py -v` (expect pass)

### Phase 6: End-to-End Pipeline Tests
- [ ] Write `tests/test_pipeline.py`:
  - Full pipeline run with mocked RESTClient
  - Table creation
  - Data count verification
  - Schema validation
- [ ] Run tests (expect failures)
- [ ] Create `{api_name}_pipeline.py` runner script
- [ ] Fix any integration issues
- [ ] Verify: `pytest tests/ -v` (all tests pass)

### Phase 7: Real Pipeline Execution
- [ ] Configure real credentials in `.dlt/secrets.toml`
- [ ] Run pipeline with small test query (limit scope)
- [ ] Inspect loaded data:
  - `dlt pipeline {api_name}_pipeline show`
  - Query destination: `SELECT * FROM {table} LIMIT 10`
- [ ] Test incremental load (run twice, verify second run behavior)
- [ ] Validate data quality and schema

### Phase 8: Documentation
- [ ] Create README.md with:
  - Setup instructions
  - Configuration guide
  - Usage examples
  - Testing approach
- [ ] Add inline code comments
- [ ] Document API-specific patterns or quirks
- [ ] Add troubleshooting section

## Acceptance Criteria

- [ ] All tests pass (`pytest tests/ -v`)
- [ ] All resources extract data successfully with correct schema
- [ ] Authentication works (or no auth correctly configured)
- [ ] Pagination fetches all pages correctly
- [ ] Incremental loading maintains state and prevents duplicates (if applicable)
- [ ] Data loads to destination with correct schema
- [ ] Pipeline can be run multiple times (incremental verification)
- [ ] Rate limiting respected (if applicable)
- [ ] Documentation complete and accurate
- [ ] Edge cases handled (single items, missing fields, empty responses)

## Special Implementation Patterns

{Add any special sections needed for this API}

### XML Response Handling (if applicable)

If API returns XML instead of JSON:

```python
import xmltodict
from dlt.common import json as dlt_json

def parse_xml_response(response, *args, **kwargs):
    """Custom response hook to convert XML to JSON."""
    parsed = xmltodict.parse(response.content)
    response._content = dlt_json.dumps(parsed).encode("utf-8")
    response.headers["Content-Type"] = "application/json"
    return response

client = RESTClient(
    base_url=base_url,
    hooks={"response": [parse_xml_response]}
)
```

### Custom Paginator (if needed)

If API has non-standard pagination:

```python
from dlt.sources.helpers.rest_client.paginators import BasePaginator

class CustomPaginator(BasePaginator):
    def __init__(self, ...):
        super().__init__()
        # Initialize state

    def update_state(self, response, data=None):
        # Extract next page info from response
        # Set self._has_next_page
        pass

    def update_request(self, request):
        # Modify request with pagination params
        return request
```

### Rate Limiting (if needed)

If API requires delays between requests:

```python
import time

class RateLimitedPaginator(OffsetPaginator):
    def __init__(self, delay_seconds=3, **kwargs):
        super().__init__(**kwargs)
        self._delay = delay_seconds
        self._last_request_time = None

    def update_request(self, request):
        if self._last_request_time:
            elapsed = time.time() - self._last_request_time
            if elapsed < self._delay:
                time.sleep(self._delay - elapsed)

        self._last_request_time = time.time()
        return super().update_request(request)
```

### Data Transformations (if needed)

If API returns nested structures that need flattening:

```python
def flatten_{resource}(item: dict) -> dict:
    """Flatten nested structure."""
    return {
        "id": item["id"],
        "field1": item["nested"]["field1"],
        "field2": item.get("optional_field"),
        # Handle arrays
        "list_field": [x["value"] for x in item.get("array_field", [])]
    }

@dlt.resource(...)
def get_{resource}(client):
    for page in client.paginate(...):
        data = page.get("data", [])
        for item in data:
            yield flatten_{resource}(item)
```

## Dependencies

```bash
pip install dlt[{destination}] pytest pytest-mock {additional_deps}
```

{List any special dependencies: xmltodict for XML parsing, etc.}

## Key Principles

- **Test-driven development**: Write tests first, implement after
- **Use dlt's built-in patterns**: RESTClient, paginators, auth classes
- **Keep it simple**: Only implement what's needed, avoid over-engineering
- **Minimal mocking**: Mock at RESTClient level, not internal functions
- **Incremental verification**: Test each component as you build
- **Handle edge cases**: Single items vs arrays, missing fields, empty responses

## Next Steps

After completing specification and implementation:
1. Run full test suite: `pytest tests/ -v`
2. Execute pipeline with real credentials
3. Verify data quality in destination
4. Test incremental loading behavior
5. Proceed to: **`02_implement/02_implement.md`** for detailed implementation guidance
```

## Output Files

This prompt generates one file:

**Specification Document**: `specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md`
- API research findings
- Endpoint documentation
- Authentication and rate limiting details
- Implementation architecture
- TDD test structure
- Task breakdown
- Code examples and patterns

## Usage Example

```
User: "I want to build a pipeline for the GitHub API"