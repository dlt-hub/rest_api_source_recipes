# dlt Context for REST API Source Development

## What is dlt?

dlt (data load tool) is a Python library for building data pipelines. It handles schema inference, evolution, loading to destinations, and provides built-in patterns for REST APIs.

**Installation:**
```bash
pip install dlt
```

## REST API Patterns: Decision Matrix

dlt provides two approaches for REST APIs:

| Use Case | Approach | When to Use |
|----------|----------|-------------|
| **Standard REST APIs** | `rest_api_source()` | API with predictable structure, standard pagination, simple auth |
| **Complex/Custom APIs** | `RESTClient` | Custom pagination logic, complex authentication flows, non-standard responses |

### rest_api_source() - Declarative Approach

```python
from dlt.sources.rest_api import rest_api_source

source = rest_api_source({
    "client": {
        "base_url": "https://api.example.com",
        "auth": {
            "type": "bearer",
            "token": dlt.secrets.value
        }
    },
    "resources": [
        {
            "name": "users",
            "endpoint": {
                "path": "users",
                "paginator": "json_link",
                "data_selector": "data"
            }
        }
    ]
})
```

### RESTClient - Programmatic Approach

```python
from dlt.sources.helpers.rest_client import RESTClient, paginate
from dlt.sources.helpers.rest_client.paginators import JSONResponsePaginator

@dlt.resource(write_disposition="replace")
def users():
    client = RESTClient(base_url="https://api.example.com")

    for page in client.paginate(
        "/users",
        paginator=JSONResponsePaginator(next_url_path="pagination.next")
    ):
        yield page
```

## Authentication Patterns

### API Key (Query Parameter)
```python
# In rest_api_source config
"auth": {
    "type": "api_key",
    "name": "api_key",
    "api_key": dlt.secrets.value,
    "location": "query"
}

# With RESTClient
client = RESTClient(
    base_url="https://api.example.com",
    params={"api_key": dlt.secrets["api_key"]}
)
```

### Bearer Token
```python
# In rest_api_source config
"auth": {
    "type": "bearer",
    "token": dlt.secrets.value
}

# With RESTClient
client = RESTClient(
    base_url="https://api.example.com",
    headers={"Authorization": f"Bearer {dlt.secrets['token']}"}
)
```

### HTTP Basic Authentication
```python
# In rest_api_source config
"auth": {
    "type": "http_basic",
    "username": dlt.secrets.value,
    "password": dlt.secrets.value
}

# With RESTClient
from requests.auth import HTTPBasicAuth
client = RESTClient(
    base_url="https://api.example.com",
    auth=HTTPBasicAuth(dlt.secrets["username"], dlt.secrets["password"])
)
```

### OAuth 2.0 Client Credentials
```python
# In rest_api_source config
"auth": {
    "type": "oauth2_client_credentials",
    "access_token_url": "https://api.example.com/oauth/token",
    "client_id": dlt.secrets.value,
    "client_secret": dlt.secrets.value
}

# With RESTClient - use rest_api_source for OAuth
```

## Pagination Classes

### JSONLinkPaginator
Follows `next` links in JSON responses.

```python
# Declarative
"paginator": "json_link"  # Default path: "next"

# Custom path
"paginator": {
    "type": "json_link",
    "next_url_path": "pagination.next_page"
}

# Programmatic
from dlt.sources.helpers.rest_client.paginators import JSONResponsePaginator
paginator = JSONResponsePaginator(next_url_path="links.next")
```

### OffsetPaginator
Uses offset/limit parameters.

```python
# Declarative
"paginator": {
    "type": "offset",
    "limit": 100,
    "offset_param": "offset",
    "limit_param": "limit"
}

# Programmatic
from dlt.sources.helpers.rest_client.paginators import OffsetPaginator
paginator = OffsetPaginator(limit=100)
```

### PageNumberPaginator
Uses page numbers.

```python
# Declarative
"paginator": {
    "type": "page_number",
    "page_param": "page",
    "page_size": 50
}

# Programmatic
from dlt.sources.helpers.rest_client.paginators import PageNumberPaginator
paginator = PageNumberPaginator(page_size=50)
```

### JSONResponseCursorPaginator
Uses cursor-based pagination from response.

```python
# Declarative
"paginator": {
    "type": "cursor",
    "cursor_path": "pagination.cursor",
    "cursor_param": "cursor"
}

# Programmatic
from dlt.sources.helpers.rest_client.paginators import JSONResponseCursorPaginator
paginator = JSONResponseCursorPaginator(cursor_path="meta.next_cursor")
```

## Resource and Source Decorators

### Basic Resource
```python
import dlt

@dlt.resource(write_disposition="replace")
def my_resource():
    yield [{"id": 1, "name": "Item 1"}]
```

### Resource with Primary Key
```python
@dlt.resource(
    write_disposition="merge",
    primary_key="id"
)
def users():
    yield [{"id": 1, "email": "user@example.com"}]
```

### Source with Multiple Resources
```python
@dlt.source
def my_api_source():
    return [users(), posts(), comments()]
```

### Write Dispositions

| Disposition | Behavior | Use Case |
|-------------|----------|----------|
| `replace` | Truncate and reload all data | Full snapshots, dimension tables |
| `append` | Add new records without deduplication | Immutable logs, events |
| `merge` | Upsert based on primary key | Updating records, CDC |

## Secrets Management

### Using dlt.secrets.value
```python
# Access secrets from .dlt/secrets.toml
api_key = dlt.secrets["api_key"]
token = dlt.secrets["sources.my_api.token"]
```

### .dlt/secrets.toml Structure
```toml
# Root level secrets
api_key = "your_api_key_here"

# Source-specific secrets
[sources.my_api]
base_url = "https://api.example.com"
api_key = "source_specific_key"

# Nested secrets
[sources.my_api.credentials]
client_id = "your_client_id"
client_secret = "your_client_secret"
```

### Environment Variables
dlt automatically reads from environment variables:
```bash
# Maps to dlt.secrets["api_key"]
SOURCES__MY_API__API_KEY=your_key

# Maps to dlt.secrets["sources"]["my_api"]["token"]
SOURCES__MY_API__TOKEN=your_token
```

## Incremental Loading

### Basic Incremental Loading
```python
import dlt
from dlt.sources.helpers.rest_client import RESTClient

@dlt.resource(
    write_disposition="append",
    primary_key="id"
)
def events(
    updated_at=dlt.sources.incremental("updated_at", initial_value="2024-01-01T00:00:00Z")
):
    client = RESTClient(base_url="https://api.example.com")

    for page in client.paginate(
        "/events",
        params={"updated_since": updated_at.last_value}
    ):
        yield page
```

### Incremental with Merge
```python
@dlt.resource(
    write_disposition="merge",
    primary_key="id"
)
def users(
    updated_at=dlt.sources.incremental("updated_at")
):
    # Fetches only records updated since last run
    # Merges based on primary_key
    ...
```

### Key Patterns
- Use `append` for immutable data (logs, events)
- Use `merge` for mutable data (users, orders)
- Specify `initial_value` for first run
- Use `last_value` to get the last processed value
- Common incremental fields: `updated_at`, `modified_date`, `id`

## Creating and Running Pipelines

### Basic Pipeline
```python
import dlt
from my_source import my_api_source

# Create pipeline
pipeline = dlt.pipeline(
    pipeline_name="my_pipeline",
    destination="duckdb",
    dataset_name="my_dataset"
)

# Run pipeline
info = pipeline.run(my_api_source())
print(info)
```

### Common Destinations
- `duckdb` - Local analytics, testing
- `bigquery` - Google Cloud warehouse
- `snowflake` - Cloud data warehouse
- `postgres` - PostgreSQL database
- `redshift` - AWS data warehouse

### Pipeline Configuration
Specify in `.dlt/config.toml`:
```toml
[pipeline]
pipeline_name = "my_pipeline"
destination = "duckdb"
dataset_name = "my_dataset"
```

## Testing REST API Sources

### Local Testing with DuckDB
```python
# Use DuckDB for fast local testing
pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data"
)

# Run and inspect
info = pipeline.run(source)
print(f"Loaded {info.load_packages[0].state} rows")
```

### Inspecting Data
```python
# Get DuckDB connection
conn = pipeline.sql_client()

# Query loaded data
result = conn.execute("SELECT COUNT(*) FROM users").fetchone()
print(f"Total users: {result[0]}")
```

## Common Patterns Summary

1. **Start Simple**: Use `rest_api_source()` for standard APIs
2. **Use RESTClient**: When you need custom logic
3. **Incremental Loading**: Always specify incremental fields for large datasets
4. **Write Dispositions**: Choose based on data mutability
5. **Secrets**: Always use `.dlt/secrets.toml` or environment variables
6. **Test Locally**: Use DuckDB destination for fast iteration
7. **Primary Keys**: Required for `merge` disposition

## Additional Resources

- dlt Documentation: https://dlthub.com/docs
- REST API Source: https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api
- RESTClient: https://dlthub.com/docs/general-usage/http/rest-client
