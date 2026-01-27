---
description: Create implementation specification for dlt REST API source using RESTClient
argument-hint: <api_research_file> [--output <spec_file>]
---

# plan_dlt_rest_client

Create test-driven implementation specification for building a dlt REST API source using RESTClient class.

**Purpose:** Generates detailed specification with task breakdown for implementing a dlt source that extracts data from REST APIs using TDD principles and dlt's built-in RESTClient.

**Note:** Use dlt's `RESTClient` from `dlt.sources.helpers.rest_client` - do NOT plan for custom API clients. dlt handles retries, rate limiting, and error handling.

## Arguments

**Required:**
- `api_research_file`: Path to API research document (from `research_api` or scaffold analysis)

**Optional:**
- `--output <spec_file>`: Save to (default: `specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_011_spec_dlt_rest_client_imf.md`

## Quick Reference

- **dlt Patterns:** See `00_prime/000_dlt_context.md` for built-in features, configuration, secrets
- **Auth Classes:** `BearerTokenAuth`, `APIKeyAuth`, `HttpBasicAuth`, `OAuth2ClientCredentials`
- **Pagination Types:** single_page, page_number, offset, header_link, json_link, cursor
- **Write Dispositions:** append (immutable), replace (full refresh), merge (upsert)

## Workflow

### Step 1: Parse and Load

1. Read API research file
2. Extract key information:
   - Base URL and authentication method
   - Endpoints to implement
   - Pagination strategy per endpoint
   - Incremental loading requirements
   - Data structure patterns
3. Ask clarifying questions:
   - Implement all endpoints or subset?
   - Testing requirements?
   - Development environment (local, CI/CD)?
   - Destination preference (duckdb for dev)?
   - Custom transformations needed?
   - Performance requirements?

### Step 2: Analyze Requirements

4. Extract authentication information:
   - Type: API Key, Bearer Token, OAuth2, Basic Auth
   - Location: header, query parameter, body
   - Required parameters and credential structure
   - Map to dlt auth class (`BearerTokenAuth`, `APIKeyAuth`, `HttpBasicAuth`, `OAuth2ClientCredentials`)

5. Analyze pagination per endpoint:
   - Strategy: offset, page, cursor, link, none
   - Query parameters and defaults
   - Response structure and metadata location
   - Map to dlt paginator type (single_page, offset, page_number, cursor, json_link, header_link)

6. Determine incremental loading strategy per endpoint:
   - Data characteristics: immutable vs mutable, size
   - Available cursor fields: timestamps, IDs
   - Write disposition: append, replace, merge
   - Primary keys (for merge disposition)

7. Analyze error handling and retry requirements:
   - Rate limits and retry-after headers
   - Transient vs non-retryable errors
   - Timeout requirements
   - Custom retry logic needed

8. Design high-level resource structure:
   - Resource names and table names
   - Write dispositions (append/replace/merge)
   - Data selectors (JSONPath)
   - Dependencies (parent/child relationships)

### Step 3: Design Tests (TDD)

9. Plan comprehensive test structure:
   - Authentication: valid credentials, missing credentials, invalid credentials
   - Pagination: first page, subsequent pages, last page detection, empty results
   - Incremental: initial load, incremental load with state, state persistence
   - Retry: successful requests, transient errors, rate limits, max retries
   - Data structure: schema validation, data selector extraction
   - Integration: full pipeline, resource selection

10. Define test fixtures and mocks:
    - Mock API responses per endpoint (success, error, pagination scenarios)
    - Sample data structures and expected schemas
    - Mock auth credentials and configuration
    - Error response scenarios (401, 429, 5xx)

### Step 4: Create Specification

11. Design source/resource architecture
12. Document authentication configuration and credential structure
13. Document pagination configuration per endpoint
14. Document incremental loading strategy per endpoint
15. Document retry and error handling configuration
16. Define shared constants (base URL, defaults)
17. Create comprehensive task breakdown

### Step 5: Validate and Output

18. Present summary to user:
    - Number of resources/endpoints
    - Authentication approach
    - Pagination strategies per endpoint
    - Incremental loading endpoints
    - Retry configuration
    - Total tasks across phases
19. Confirm approach with user
20. Generate comprehensive specification document

## Implementation Specification Template

```markdown
# {API Name} - dlt REST Client Implementation Specification

**Date:** {YYYY-MM-DD}
**Based on:** {API Research File Path}

## Summary
{Brief overview: data source, endpoints, key challenges}

## Architecture

**Stack:**
- dlt version: {version}
- Python: {version}
- Destination: {duckdb/bigquery/etc}
- Testing: pytest

**Overview:**
- Source: `{api_name}_source`
- Resources: {N}
- Auth: {type}
- Pagination: {strategies}
- Incremental: {N} endpoints
- Retry: {strategy}

## Constants & Configuration

**Shared Constants:**
```python
# constants.py
BASE_URL = "{base_url}"
DEFAULT_{PARAM} = "{value}"
```

**Configuration Files:**
```toml
# .dlt/secrets.toml (NEVER commit to git)
[sources.{api_name}_source]
{credential_key} = "your_credential_here"

# .dlt/config.toml (optional overrides)
[sources.{api_name}_source]
base_url = "{base_url}"
```

## Authentication

**Method:** {API Key / Bearer Token / OAuth2 / Basic Auth}
**Location:** {Header / Query Parameter}
**dlt Auth Class:** {AuthClass}

**Credential Structure:**
```toml
[sources.{api_name}_source]
{credential_param} = "your_value_here"
```

**Implementation:**
```python
from dlt.sources.helpers.rest_client.auth import {AuthClass}

auth = {AuthClass}(
    {param}=dlt.secrets.value,
    {additional_params}
)
```

**Tests:**
```python
def test_valid_credentials():
    """Test successful authentication with valid credentials"""

def test_missing_credentials_raises_error():
    """Test missing credentials raise ConfigFieldMissingException"""

def test_invalid_credentials():
    """Test 401/403 responses handled correctly"""
```

**Error Handling:**
- 401 Unauthorized: Invalid credentials (non-retryable)
- 403 Forbidden: Insufficient permissions (non-retryable)

**Security:**
- Add `.dlt/secrets.toml` to `.gitignore`
- Never commit credentials
- Use environment variables for CI/CD

## Pagination Strategies

**Decision Matrix:**
| API Behavior | Response Contains | dlt Paginator |
|--------------|-------------------|---------------|
| No pagination | Single page | `single_page` |
| Page numbers | Optional: total_pages | `page_number` |
| Offset/limit | Optional: total | `offset` |
| Link header | Header: Link rel="next" | `header_link` |
| Next URL in JSON | JSON: {"next": "url"} | `json_link` |
| Cursor in JSON | JSON: {"next_cursor": "..."} | `cursor` |

**Per-Endpoint Configuration:**
{For each endpoint with non-default pagination, document:}

### {Endpoint Name} Pagination

**Type:** {offset/cursor/page_number/single_page}
**Parameters:**
- Page size: `{param}` (default: {value}, max: {value})
- Cursor/offset: `{param}`

**Configuration:**
```python
paginator = {PaginatorClass}(
    {param}={value}
)
```

**Last Page Detection:**
- {Strategy: empty results / total count / missing next field}

**Tests:**
```python
def test_{endpoint}_pagination_first_page():
    """Test first page request with correct parameters"""

def test_{endpoint}_pagination_last_page():
    """Test last page detection"""

def test_{endpoint}_pagination_empty_results():
    """Test empty page handling"""
```

## Incremental Loading Strategies

**Decision Matrix:**
| Data Type | Mutability | Size | Strategy |
|-----------|------------|------|----------|
| Events/Logs | Immutable | Any | append + timestamp |
| User Records | Mutable | Small | replace (full refresh) |
| User Records | Mutable | Large | merge + timestamp |
| Transactions | Immutable | Any | append + ID cursor |

**Per-Endpoint Configuration:**
{For each endpoint requiring incremental loading, document:}

### {Endpoint Name} Incremental

**Write Disposition:** {append/replace/merge}
**Primary Key:** {key} (if merge)
**Cursor Field:** {field}
**Initial Value:** {value}
**Query Parameter:** {param}

**Configuration:**
```python
@dlt.resource(
    name="{name}",
    write_disposition="{disposition}",
    primary_key="{key}"  # if merge
)
def get_{name}(
    {cursor_param}: dlt.sources.incremental[{type}] = dlt.sources.incremental(
        "{cursor_field}",
        initial_value="{initial_value}"
    )
):
    params = {"{api_param}": {cursor_param}.last_value}
    for page in client.paginate("{path}", params=params):
        yield page
```

**Tests:**
```python
def test_{endpoint}_initial_load():
    """Test first load with no state"""

def test_{endpoint}_incremental_load():
    """Test incremental load with existing state"""

def test_{endpoint}_state_persistence():
    """Test state is saved and restored correctly"""

def test_{endpoint}_deduplication():
    """Test duplicates handled (merge only)"""
```

## Retry & Error Handling

**Default Retry Configuration:**
```python
{
    "request_max_attempts": 5,
    "request_backoff_factor": 1.0,
    "request_timeout": 60.0,
    "status_forcelist": [429, 500, 502, 503, 504]
}
```

**Rate Limits:**
- Limit: {X} requests per {time unit}
- Header: `{rate_limit_header}`
- Status code: 429 (Too Many Requests)
- Strategy: Exponential backoff with retry-after header respect

**Retryable Errors:**
- 429 (Rate Limit) - exponential backoff
- 500, 502, 503, 504 (Server errors)
- Connection timeouts
- Connection errors

**Non-Retryable Errors:**
- 400 (Bad Request) - invalid request
- 401 (Unauthorized) - invalid credentials
- 403 (Forbidden) - insufficient permissions
- 404 (Not Found) - resource doesn't exist
- 422 (Unprocessable Entity) - validation error

**Endpoint-Specific Timeouts:**
{If any endpoints need custom timeout configuration}
```python
"{endpoint}": {
    "timeout": {"total": {seconds}, "connect": 10.0, "read": {seconds}}
}
```

**Tests:**
```python
def test_successful_request_no_retry():
    """Test successful request completes without retries"""

def test_transient_error_retries():
    """Test 500 error triggers retry"""

def test_rate_limit_backoff():
    """Test 429 triggers exponential backoff"""

def test_max_retries_exceeded():
    """Test max retries exceeded raises error"""

def test_non_retryable_error():
    """Test 4xx errors fail immediately"""
```

## Resources

### {Resource Name}

**Purpose:** {What data this resource extracts}
**Endpoint:** `{path}`
**Write Disposition:** `{append/replace/merge}` (see Incremental Loading section)
**Primary Key:** `{key}` (if merge)
**Pagination:** `{type}` (see Pagination section)
**Data Selector:** `{jsonpath}`
**Incremental:** {Yes/No} - {cursor_field} (see Incremental Loading section)

**Configuration Summary:**
- Auth: See Authentication section
- Pagination: See Pagination section for {endpoint}
- Incremental: See Incremental Loading section for {endpoint}
- Retry: Uses default retry config (see Retry section)

**Tests:**
```python
def test_{name}_data_structure():
    """Verify response schema matches expected structure"""

def test_{name}_data_selector():
    """Verify JSONPath selector extracts correct data"""

def test_{name}_pagination():
    """Verify pagination configuration works"""

def test_{name}_incremental():
    """Verify incremental state management (if applicable)"""
```

**Mock Response:**
```json
{
  "data": [{sample_record}]
}
```

{Repeat for each resource}

## Source

**CRITICAL:** The source `name` parameter MUST match the pipeline's `dataset_name` to avoid schema mismatch issues.

```python
from dlt.sources.helpers.rest_client import RESTClient
import dlt

@dlt.source(name="{api_name}_data")  # MUST match dataset_name in pipeline
def {api_name}_source(
    {credential}: {Type} = dlt.secrets.value,
    base_url: str = dlt.config.value
):
    # Define resources
    return get_{resource1}(), get_{resource2}()
```

**Pipeline Usage:**
```python
pipeline = dlt.pipeline(
    pipeline_name="{api_name}_pipeline",
    destination="duckdb",
    dataset_name="{api_name}_data",  # MUST match @dlt.source(name=...)
    dev_mode=True,
    progress="log"
)
pipeline.run({api_name}_source())
```

**Naming Convention:**
- `pipeline_name`: Identifies the pipeline (e.g., `"{api_name}_pipeline"`)
- `dataset_name`: The database schema where tables are created (e.g., `"{api_name}_data"`)
- `@dlt.source(name=...)`: MUST match `dataset_name` exactly to ensure schema consistency

**Common naming patterns:**
- `{api_name}_data` - For general data (e.g., `imf_sdmx_data`)
- `{api_name}_raw` - For raw/unprocessed data (e.g., `imf_sdmx_raw`)
- `{api_name}_staging` - For staging environments (e.g., `imf_sdmx_staging`)

## Test Structure

**File Organization:**
```
tests/
├── conftest.py              # Shared fixtures (mock responses, auth, client)
├── test_helpers.py          # Unit tests for helper functions (if any)
├── test_authentication.py   # Authentication tests
├── test_resources.py        # Resource-level tests (data structure, selectors)
├── test_pagination.py       # Pagination tests per endpoint
├── test_incremental.py      # Incremental loading and state tests
├── test_retries.py          # Retry and error handling tests
├── test_source.py           # Source integration tests
└── test_pipeline.py         # End-to-end pipeline tests
```

**Shared Fixtures (conftest.py):**
```python
import pytest
import dlt

@pytest.fixture
def mock_{api_name}_response():
    """Mock API response data"""
    return {sample_response}

@pytest.fixture
def mock_auth():
    """Mock authentication for tests"""
    return {auth_config}

@pytest.fixture
def mock_client(mock_auth):
    """Mock RESTClient for tests"""
    # Returns configured mock client
```

**Test Categories:**

1. **Authentication Tests** (`test_authentication.py`):
   - Valid credentials load successfully
   - Missing credentials raise ConfigFieldMissingException
   - Invalid credentials return 401/403
   - Credential format validation

2. **Resource Tests** (`test_resources.py`):
   - Data structure validation
   - JSONPath selector extraction
   - Schema compliance
   - Required fields present

3. **Pagination Tests** (`test_pagination.py`):
   - First page request
   - Subsequent pages
   - Last page detection
   - Empty results handling
   - {Per-endpoint pagination tests}

4. **Incremental Tests** (`test_incremental.py`):
   - Initial load (no state)
   - Incremental load (with state)
   - State persistence
   - Deduplication (merge only)
   - Empty results with state

5. **Retry Tests** (`test_retries.py`):
   - Successful request (no retry)
   - Transient error with retry
   - Rate limit (429) backoff
   - Max retries exceeded
   - Non-retryable errors (4xx)
   - Timeout handling

6. **Source Tests** (`test_source.py`):
   - Source function returns resources
   - Resource selection works
   - Configuration injection
   - Multiple resources coordinate

7. **Pipeline Tests** (`test_pipeline.py`):
   - End-to-end pipeline run
   - Data loads to destination
   - Schema created correctly
   - Incremental runs work
   - Data integrity verified

## Task Breakdown

### Phase 1: Setup
- [ ] Create project structure (see Test Structure section)
- [ ] Set up virtual environment: `python -m venv venv && source venv/bin/activate`
- [ ] Install dlt: `pip install dlt`
- [ ] Install pytest: `pip install pytest pytest-mock`
- [ ] Create `.dlt/secrets.toml.example` (see Authentication section)
- [ ] Create `.dlt/config.toml` (see Constants & Configuration section)
- [ ] Add `.dlt/secrets.toml` to `.gitignore`
- [ ] Create `tests/conftest.py` with shared fixtures

### Phase 2: Authentication (TDD)
- [ ] Write authentication tests (see Authentication section tests)
- [ ] Implement auth class configuration (see Authentication section)
- [ ] Create credential validation
- [ ] Run tests: `pytest tests/test_authentication.py -v`
- [ ] Verify all auth tests pass

### Phase 3: Pagination Configuration (TDD)
- [ ] Write pagination tests per endpoint (see Pagination section tests)
- [ ] Configure paginator per endpoint (see Pagination section)
- [ ] Implement last page detection logic
- [ ] Run tests: `pytest tests/test_pagination.py -v`
- [ ] Verify all pagination tests pass

### Phase 4: Resources - Data Structure (TDD, per resource)
- [ ] Write data structure tests for {resource} (see Resources section tests)
- [ ] Create mock responses for {resource}
- [ ] Implement RESTClient configuration for {resource}
- [ ] Implement data selector (JSONPath)
- [ ] Run tests: `pytest tests/test_resources.py::test_{resource}_data_structure -v`
- [ ] Verify data structure tests pass

### Phase 5: Incremental Loading (TDD, per incremental resource)
- [ ] Write incremental tests for {resource} (see Incremental Loading section tests)
- [ ] Implement write disposition configuration
- [ ] Implement incremental decorator with cursor field
- [ ] Implement state management
- [ ] Run tests: `pytest tests/test_incremental.py::test_{resource}_* -v`
- [ ] Verify incremental tests pass

### Phase 6: Retry & Error Handling (TDD)
- [ ] Write retry tests (see Retry & Error Handling section tests)
- [ ] Configure default retry behavior
- [ ] Configure endpoint-specific timeouts (if needed)
- [ ] Implement error handling for non-retryable errors
- [ ] Run tests: `pytest tests/test_retries.py -v`
- [ ] Verify all retry tests pass

### Phase 7: Source Integration (TDD)
- [ ] Write source-level tests (see Test Structure - Source Tests)
- [ ] Implement @dlt.source wrapper
- [ ] Implement credential injection from dlt.secrets
- [ ] Implement resource factory functions
- [ ] Test resource selection
- [ ] Run tests: `pytest tests/test_source.py -v`
- [ ] Verify source tests pass

### Phase 8: Pipeline Testing (Integration)
- [ ] Write end-to-end pipeline tests (see Test Structure - Pipeline Tests)
- [ ] Test with DuckDB destination (dev_mode=True)
- [ ] Verify schema creation
- [ ] Verify data integrity and completeness
- [ ] Test incremental runs (run twice, verify state)
- [ ] Run tests: `pytest tests/test_pipeline.py -v`
- [ ] Verify all pipeline tests pass

### Phase 9: Documentation
- [ ] Create README with credential setup instructions
- [ ] Document configuration options (.dlt/config.toml)
- [ ] Add usage examples for pipeline
- [ ] Document testing approach
- [ ] Add troubleshooting section
- [ ] Document known limitations

## Acceptance Criteria

1. All tests pass (authentication, pagination, resources, incremental, retries, source, pipeline)
2. All {N} resources extract data successfully
3. Authentication works with proper credential management
4. Pagination handles all pages for all endpoints
5. Incremental loading maintains state correctly (if applicable)
6. Retry logic handles transient errors and rate limits
7. Data loads to destination with correct schema
8. Documentation complete and comprehensive

## Reference

See `00_prime/002_api_references.md` for comprehensive dlt and API development references.

**Key dlt documentation:**
- dlt RESTClient: https://dlthub.com/docs/general-usage/http/rest-client
- dlt REST API Source: https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api

**API-Specific:**
- {API Name} Documentation: {url}
- {API Name} API Reference: {url}
```

## Validation Checklist

Before finalizing specification:
- [ ] All endpoints mapped to resources
- [ ] Authentication method and credential structure documented
- [ ] Pagination strategy specified per endpoint
- [ ] Incremental strategy determined per endpoint
- [ ] Write dispositions chosen appropriately
- [ ] Primary keys identified (for merge disposition)
- [ ] Retry configuration specified
- [ ] Constants and shared configuration defined
- [ ] Comprehensive test structure designed
- [ ] Complete task breakdown created (all 9 phases)
- [ ] All sections of template filled out

## Output Message

```
Comprehensive implementation specification complete: {api_name}
Saved to: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

Summary:
- Resources: {number}
- Auth: {type} via {AuthClass}
- Pagination: {strategies used}
- Incremental: {number} endpoints
- Retry: {strategy}
- Test files: {number}
- Tasks: {number} across 9 phases

This specification includes comprehensive sections for:
- Authentication configuration and tests
- Pagination per endpoint
- Incremental loading strategies
- Retry and error handling
- Complete test structure
- Phase-by-phase implementation tasks
```

## Next Steps

**Evaluate if optional appendices are needed:**

| Check | Condition | Action |
|-------|-----------|--------|
| **Authentication** | OAuth2 with refresh, multi-step flows, or custom signatures? | → Use `01_plan/012_plan_authentication.md` |
| | Simple API key, Bearer token, or Basic auth? | → Skip 012 (already covered in main spec) |
| **Pagination** | Different strategies per endpoint, or custom pagination logic? | → Use `01_plan/013_plan_pagination.md` |
| | Same strategy for all endpoints, or standard patterns? | → Skip 013 (already covered in main spec) |
| **Incremental** | Compound cursors, resource interdependencies, or backfill strategies? | → Use `01_plan/014_plan_incremental_load.md` |
| | Simple timestamp or ID cursor? | → Skip 014 (already covered in main spec) |
| **Retries** | Response content-based retry, multi-tier limits, or circuit breaker? | → Use `01_plan/015_plan_retries.md` |
| | Standard HTTP status retry with exponential backoff? | → Skip 015 (already covered in main spec) |

**RECOMMENDED for most APIs: Proceed directly to implementation**

Use prompt: `02_implement/02_implement.md` with the spec file you just created.

Most APIs don't need appendices - the main spec covers standard patterns comprehensively.
