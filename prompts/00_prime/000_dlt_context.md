**CRITICAL: This is ONLY a priming step to provide you with relevant context about dlt. DO NOT take any actions, ask questions, or proceed with any workflow steps after reading this document. Simply acknowledge that you have read and understood the context. WAIT for explicit user instructions before doing anything else.**

---

# Workflow Instructions for AI Assistant

**IMPORTANT: Read this section first before proceeding with any implementation.**

## Step-by-Step Process

When a user wants to build a dlt pipeline, follow this structured workflow:

1. **DO NOT ask multiple questions immediately** after reading this context document
2. **GUIDE the user** to use the prompts in the `prompts/` directory step-by-step
3. **Follow this sequence**:
   - **Step 1**: Use prompts to research and gather API specifications
   - **Step 2**: Create detailed specifications based on research findings
   - **Step 3**: Implement the pipeline based on approved specifications

## Expected Workflow

1. The user provides initial requirements (e.g., "build IMF API pipeline")
2. You guide them to the first research prompt to gather API information
3. After research is complete, guide them to specification creation prompts
4. Only after specs are approved, proceed to implementation prompts
5. Each step builds on the previous one with clear artifacts

## What This Means

- **Don't immediately ask**: "Do you have API documentation? What authentication? Which endpoints?"
- **Instead say**: "Let's start by using the research prompt to gather the API specifications systematically"
- The prompts in this directory are designed to extract information methodically
- The user should progress through prompts sequentially, not jump to implementation

## Context Document Purpose

This document provides reference material about dlt library features, patterns, and best practices. Use it as a lookup reference during the workflow, not as a trigger to ask setup questions.

---

# Part 1: dlt Open Source Library

## Core Concepts

### What is dlt?

dlt is "an open-source Python library that loads data from various, often messy data sources into well-structured datasets." Built ground-up for LLM integration, it enables rapid pipeline development across 5,000+ potential data sources with automatic schema management and evolution.

**Target Users:**
- Python developers building data pipelines
- Data engineers needing flexible ETL/ELT solutions
- Teams requiring portable, non-vendor-locked data infrastructure

**Key Capabilities:**
- Automatic schema inference and evolution
- Incremental loading with state management
- Support for nested/semi-structured data
- 25+ destinations (warehouses, databases, lakes, vectors)
- Deploy anywhere Python runs

**Installation:**
```bash
pip install dlt
```

### Pipelines

Pipelines are the foundational execution framework orchestrating data movement from sources to destinations.

**Creating Pipelines:**
```python
import dlt

pipeline = dlt.pipeline(
    pipeline_name="my_pipeline",
    destination="duckdb",
    dataset_name="my_data"
)
```

**Key Parameters:**
- `pipeline_name`: Identifies pipeline in monitoring; auto-generated from module filename if omitted
- `destination`: Target system (e.g., "duckdb", "bigquery", "postgres")
- `dataset_name`: Logical table grouping (schema in relational databases)
- `pipelines_dir`: Custom location for pipeline artifacts/state
- `dev_mode`: When `True`, appends datetime suffix to dataset names for testing

**Running Pipelines:**
```python
info = pipeline.run(data, table_name="users", write_disposition="append")
```

**run() Parameters:**
- `data`: Accepts sources, resources, generators, or iterables
- `write_disposition`: Controls loading behavior (append/replace/merge)
- `table_name`: Specifies table when not inferrable

**State Management:**
Pipelines store extracted files, load packages, schemas, and state in `~/.dlt/pipelines/<pipeline_name>`. Access via `dlt pipeline info` CLI or programmatically.

**Refresh Options:**
```python
pipeline.run(source(), refresh="drop_sources")   # Remove all tables + wipe state
pipeline.run(source(), refresh="drop_resources")  # Drop specific resource tables
pipeline.run(source(), refresh="drop_data")       # Truncate data, preserve schema
```

**Progress Monitoring:**
```python
pipeline = dlt.pipeline(
    pipeline_name="chess_pipeline",
    destination='duckdb',
    progress="log"  # or "enlighten", "tqdm", "alive_progress"
)
```

**Code Example:**
```python
import dlt

pipeline = dlt.pipeline(
    pipeline_name="my_pipeline",
    destination="duckdb",
    dataset_name="my_data"
)

# Run with different write dispositions
pipeline.run(data, write_disposition="append")  # Default: add new rows
pipeline.run(data, write_disposition="replace")  # Overwrite existing data
pipeline.run(data, write_disposition="merge")    # Merge/upsert based on keys
```

### Resources

Resources are individual data collection units that yield data from generators, APIs, databases, or Python structures. As stated in documentation, "A resource is an (optionally async) function that yields data."

**Decorator Parameters:**
- `name`: Resulting table name (defaults to function name)
- `write_disposition`: Loading strategy—`append` (default), `replace`, or `merge`
- `primary_key`/`merge_key`: Columns for incremental/merge operations
- `columns`: Schema definition via dictionary or Pydantic models
- `table_name`: Dynamic naming using callable functions
- `max_table_nesting`: Controls nested table depth (default: 1000)
- `nested_hints`: Schema configuration for nested objects
- `schema_contract`: Enforces constraints on schema evolution
- `file_format`: Output format (parquet, csv, etc.)
- `parallelized`: Enables parallel extraction in threads

**Basic Pattern:**
```python
@dlt.resource(name='users', write_disposition='replace')
def get_users():
    for user in fetch_users():
        yield user
```

**Parametrized Resources:**
```python
@dlt.resource(name='table_name', write_disposition='replace')
def generate_var_rows(nr):
    for i in range(nr):
        yield {'id': i, 'example_string': 'abc'}

pipeline.run(generate_var_rows(100))
```

**Dynamic Table Dispatch:**
```python
@dlt.resource(table_name=lambda event: event['type'])
def repo_events():
    for event in fetch_events():
        yield event  # Routes to table based on event['type']
```

**Schema Definition Methods:**

*Inline Column Definition:*
```python
@dlt.resource(name="user", columns={"tags": {"data_type": "json"}})
def get_users():
    yield {'id': 1, 'name': 'Alice', 'tags': ['admin', 'user']}
```

*Pydantic Integration:*
```python
from pydantic import BaseModel
from typing import List, Optional

class User(BaseModel):
    id: int
    name: str
    tags: List[str]
    email: Optional[str]

@dlt.resource(name="user", columns=User)
def get_users():
    yield User(id=1, name='Alice', tags=['admin'], email='alice@example.com')
```

**Transformer Pattern:**
```python
@dlt.resource(write_disposition="replace")
def users(limit=None):
    for u in _get_users(limit):
        yield u

@dlt.transformer(data_from=users)
def users_details(user_item):
    for detail in _get_details(user_item["user_id"]):
        yield detail

# Execute with pipe operator
pipeline.run(users(limit=100) | users_details)
```

**Configuration Methods:**

*apply_hints() - Runtime schema adjustment:*
```python
tables = sql_database()
tables.users.apply_hints(
    write_disposition="merge",
    primary_key="user_id",
    incremental=dlt.sources.incremental("updated_at")
)
```

*add_map() - Item transformation:*
```python
def anonymize_user(user_data):
    user_data["user_id"] = _hash_str(user_data["user_id"])
    return user_data

pipeline.run(users().add_map(anonymize_user))
```

*add_filter() - Conditional exclusion:*
```python
pipeline.run(users().add_filter(lambda user: user["user_id"] != "me"))
```

*add_limit() - Data sampling:*
```python
r = dlt.resource(itertools.count(), name="infinity").add_limit(10)
my_resource().add_limit(max_time=10)  # Time-based limiting
my_resource().add_limit(max_items=10, max_time=10)  # Combined limits
```

**Code Example:**
```python
import dlt

@dlt.resource(name='users', write_disposition='merge', primary_key='id')
def get_users():
    for user in fetch_users():
        yield user

# Transformer example
@dlt.transformer(data_from=get_users)
def user_details(user_item):
    details = fetch_details(user_item['id'])
    yield details

# Apply transformations
pipeline.run(get_users().add_filter(lambda u: u['active']).add_limit(100))
```

### Sources

Sources are logical groupings of resources—typically representing a single API or data system. As defined in documentation, "a source is a logical grouping of resources, i.e., endpoints of a single API."

**Declaration:**
```python
@dlt.source
def hubspot(api_key=dlt.secrets.value):
    endpoints = ["companies", "deals", "products"]

    def get_resource(endpoint):
        yield requests.get(url + "/" + endpoint).json()

    for endpoint in endpoints:
        yield dlt.resource(get_resource(endpoint), name=endpoint)
```

**Key Features:**
- Bundle related resources together
- Share authentication and pagination logic
- Attach schemas defining tables and columns
- Control nesting behavior with `max_table_nesting`

**Controlling Nesting:**
```python
@dlt.source(max_table_nesting=2)
def my_source():
    return [resource1(), resource2()]

# Or after instantiation
source = my_source()
source.max_table_nesting = 1
```

**max_table_nesting values:**
- `0`: No nested tables; nested data becomes JSON
- `1`: Only root-level nested tables
- `2-3`: Practical for semi-structured data (MongoDB, APIs)

**Resource Selection:**
```python
# Select specific resources
pipeline.run(source.with_resources("companies", "deals"))

# Access as dictionary/attributes
pipeline.run(source["companies"])
```

**Code Example:**
```python
@dlt.source
def my_api_source(api_key=dlt.secrets.value):
    @dlt.resource
    def users():
        for user in fetch_users(api_key):
            yield user

    @dlt.resource
    def orders():
        for order in fetch_orders(api_key):
            yield order

    return users(), orders()

# Use in pipeline
pipeline.run(my_api_source())
```

### Schema Management

dlt automatically generates schemas during normalization, creating "a content-based hash `version_hash` that is used to detect manual changes to the schema" with numeric versions that increment on updates.

**Key Concepts:**
- **Schema Inference**: Automatic data type detection from values
- **Schema Evolution**: Automatic schema updates for new columns/tables
- **Data Contracts**: Schema enforcement and validation via `schema_contract`
- **Nesting Levels**: Control via `max_table_nesting` parameter

**Data Types:**
- **Numeric**: bigint, double, decimal, wei
- **Text**: text (with precision support)
- **Temporal**: timestamp, date, time (with timezone awareness)
- **Binary**: binary, json
- **Boolean**: bool

**Timestamp Handling:**
Timestamps normalize to UTC timezone-aware format by default. "Naive timestamps are always be considered as UTC, system timezone settings are ignored by dlt". Explicit `timezone` hints on columns are honored.

**Column Hints & Attributes:**

*Key hints:*
- `primary_key`: Unique identifier for merge operations
- `unique`: Uniqueness constraint
- `merge_key`: Merge operation keys

*Nested reference hints:*
- `row_key`: `_dlt_id` for unique row identification
- `parent_key`: `_dlt_parent_id` linking to parent rows
- `root_key`: `_dlt_root_id` connecting nested tables to root

*Performance hints:*
- `partition`, `cluster`, `sort`

*Additional properties:*
- `data_type`, `precision`, `scale`, `nullable`, `is_variant`

**Variant Columns:**
When incompatible types appear for existing columns, dlt creates variant columns. Example: if `id` contains integers and strings, `id__v_text` is auto-generated for string values.

**Nested References:**
```python
# Root tables receive _dlt_id
# Nested tables include _dlt_parent_id (parent reference)
# All nested tables include _dlt_root_id (root table reference)

@dlt.resource(
    nested_hints={
        "purchases": dlt.mark.make_nested_hints(
            columns=[{"name": "price", "data_type": "decimal"}],
            schema_contract={"columns": "freeze"},
        )
    },
)
def customers():
    yield [{"id": 1, "name": "simon", "purchases": [...]}]
```

**Schema Settings:**

*Preferred data types:*
```python
# Columns ending in _timestamp become timestamp type automatically
```

*Column hint rules:*
Apply hints via exact name matching or regex patterns (SimpleRegex format).

*Naming conventions:*
- Default: snake_case, removes non-alphanumeric, nesting with double underscores
- `direct`: Bypasses normalization for destinations accepting arbitrary identifiers

**Applying Hints in Code:**
```python
@dlt.resource(
    name='my_table',
    columns={"my_column": {"data_type": "bool", "nullable": True}}
)
def my_resource():
    for i in range(10):
        yield {'my_column': i % 2 == 0}
```

**Viewing Schemas:**
```python
pipeline.default_schema.to_pretty_yaml()  # Display formatted schema
```

### Incremental Loading

"Incremental loading is the act of loading only new or changed data and not old records that we have already loaded."

**Strategies:**
- **Append-only with cursor**: Track last processed timestamp/ID
- **Merge/upsert**: Update existing records based on keys
- **Replace full dataset**: Complete refresh

**State Management:**
"If we do not keep track of the state of the load (i.e., which increments were loaded), we may encounter issues." dlt maintains state automatically to prevent duplicate loads.

**Code Example:**
```python
@dlt.resource(write_disposition='merge', primary_key='id')
def incremental_users(updated_at=dlt.sources.incremental('updated_at')):
    # Only fetch records after last updated_at
    for user in fetch_since(updated_at.last_value):
        yield user
```

**Decision Framework:**
- **Stateless data** (immutable events): Use append
- **Stateful data requiring history**: Implement SCD2 (Slowly Changing Dimensions Type-2)
- **Stateful data without history**: Use merge if incremental possible; otherwise replace

**Full Refresh:**
Force refresh by passing `write_disposition="replace"` during execution.

### Write Dispositions

dlt offers three write disposition options controlling how data is loaded:

| Disposition | Behavior | Use Case | Merge Keys |
|-------------|----------|----------|------------|
| **append** | Add new rows to table end | Event logs, immutable data, time-series | Not required |
| **replace** | Truncate then load (overwrite all) | Full refresh, snapshots, dimension tables | Not required |
| **merge** | Upsert based on primary_key and merge_key | Incremental updates, CDC, stateful records | Required: `primary_key` |

**Append Example:**
```python
@dlt.resource(write_disposition='append')
def events():
    for event in fetch_events():
        yield event
```

**Replace Example:**
```python
@dlt.resource(write_disposition='replace')
def dimension_table():
    for row in fetch_all():
        yield row
```

**Merge Example:**
```python
@dlt.resource(
    write_disposition='merge',
    primary_key='id',
    merge_key='updated_at'  # Optional: additional merge criteria
)
def users():
    for user in fetch_users():
        yield user
```

### Destinations

dlt supports **25+ destinations** across multiple categories.

**Supported Destinations:**

*Data Warehouses:*
- Google BigQuery, Snowflake, Databricks, Amazon Redshift
- Microsoft SQL Server, Azure Synapse, ClickHouse
- AWS Athena / Glue Catalog

*Databases & Data Lakes:*
- DuckDB, Postgres, Delta, Iceberg
- MotherDuck / DuckLake, Dremio (experimental)
- 30+ additional SQL databases via SQLAlchemy

*Vector Databases:*
- Weaviate, LanceDB, Qdrant

*Specialized:*
- Cloud storage/filesystem (S3, GCS, Azure)
- Custom destinations via decorator-based functions

**Configuration Patterns:**
The platform enables "append, replace, or merge your data" with support for "partitions, clusters, or indexes" and "load directly or via staging."

**Filesystem Destinations:**
Serve dual purposes: staging area for other destinations and quick data lake creation. Leverage fsspec for S3, Google Cloud Storage, Azure Blob Storage abstraction.

**Quality Assurance:**
"All destinations undergo several hundred automated tests every day."

**Extensibility:**
Custom implementations supported, "particularly for reverse ETL components that push data back to REST APIs."

### Data Access & Inspection

**Python Interface:**
```python
# Access loaded data as DataFrame
df = pipeline.dataset().table("users").df()

# Access via Arrow
arrow_table = pipeline.dataset().table("users").arrow()
```

**SQL Interface:**
```python
# Query with SQL
result = pipeline.dataset()("SELECT * FROM users WHERE active = true")
```

### REST API Client (Programmatic Approach)

dlt provides `RESTClient` for programmatic REST API interaction with built-in pagination, authentication, error handling, and retry logic. Use this when you need fine-grained control over API requests or when building custom sources.

#### RESTClient Class

`RESTClient` from `dlt.sources.helpers.rest_client` handles HTTP requests with automatic retries, pagination, and authentication.

**Basic Usage:**
```python
from dlt.sources.helpers.rest_client import RESTClient

client = RESTClient(
    base_url="https://api.example.com",
    headers={"Accept": "application/json"}
)

# Simple GET request
response = client.get("/users")
data = response.json()
```

**Integration with dlt Resources:**
```python
@dlt.resource(name="users", write_disposition="merge", primary_key="id")
def get_users():
    client = RESTClient(base_url="https://api.example.com")

    for page in client.paginate("/users"):
        yield page
```

**Initialization Parameters:**
- `base_url`: API base URL (required)
- `headers`: Default headers for all requests (optional)
- `auth`: Authentication handler instance (optional)
- `paginator`: Paginator instance for handling pagination (optional)
- `data_selector`: JSONPath for extracting data from responses (optional)
- `session`: Custom requests.Session for advanced control (optional)

**Request Methods:**
- `client.get(path, params=None, headers=None)`
- `client.post(path, json=None, data=None, headers=None)`
- `client.put(path, json=None, headers=None)`
- `client.patch(path, json=None, headers=None)`
- `client.delete(path, headers=None)`

#### Authentication Methods

dlt provides authentication classes for common patterns, automatically handling credential injection and token refresh.

**Bearer Token Authentication:**
```python
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth

auth = BearerTokenAuth(token=dlt.secrets["api_token"])

client = RESTClient(
    base_url="https://api.example.com",
    auth=auth
)
```

**API Key Authentication:**
```python
from dlt.sources.helpers.rest_client.auth import APIKeyAuth

# API key in header
auth = APIKeyAuth(
    name="X-API-Key",
    api_key=dlt.secrets["api_key"],
    location="header"
)

# API key in query parameter
auth = APIKeyAuth(
    name="api_key",
    api_key=dlt.secrets["api_key"],
    location="query"
)

client = RESTClient(base_url="https://api.example.com", auth=auth)
```

**HTTP Basic Authentication:**
```python
from dlt.sources.helpers.rest_client.auth import HttpBasicAuth

auth = HttpBasicAuth(
    username=dlt.secrets["username"],
    password=dlt.secrets["password"]
)

client = RESTClient(base_url="https://api.example.com", auth=auth)
```

**OAuth2 Client Credentials:**
```python
from dlt.sources.helpers.rest_client.auth import OAuth2ClientCredentials

auth = OAuth2ClientCredentials(
    access_token_url="https://auth.example.com/oauth/token",
    client_id=dlt.secrets["client_id"],
    client_secret=dlt.secrets["client_secret"],
    scope="read write"  # Optional
)

client = RESTClient(base_url="https://api.example.com", auth=auth)
```

OAuth2ClientCredentials automatically refreshes expired tokens.

**Custom Authentication:**
Extend `AuthConfigBase` for custom authentication logic:
```python
from dlt.sources.helpers.rest_client.auth import AuthConfigBase

class CustomAuth(AuthConfigBase):
    def __call__(self, request):
        # Modify request to add authentication
        request.headers["Authorization"] = f"Custom {self.token}"
        return request

auth = CustomAuth()
client = RESTClient(base_url="https://api.example.com", auth=auth)
```

#### rest_api_source Authentication Configuration

When using the declarative `rest_api_source` approach (not RESTClient), configure authentication through the `auth` dictionary in the configuration.

**Available Auth Types:**

**api_key - API Key Authentication:**
```python
from dlt.sources.rest_api import rest_api_source

source = rest_api_source({
    "client": {
        "base_url": "https://api.example.com",
        "auth": {
            "type": "api_key",
            "name": "X-API-Key",  # Header or query param name
            "api_key": dlt.secrets["api_key"],  # Secret value
            "location": "header"  # "header" or "query"
        }
    },
    "resources": [
        {
            "name": "users",
            "endpoint": {"path": "/users"}
        }
    ]
})
```

**bearer - Bearer Token Authentication:**
```python
source = rest_api_source({
    "client": {
        "base_url": "https://api.example.com",
        "auth": {
            "type": "bearer",
            "token": dlt.secrets["bearer_token"]
        }
    },
    "resources": [...]
})
```

**http_basic - HTTP Basic Authentication:**
```python
source = rest_api_source({
    "client": {
        "base_url": "https://api.example.com",
        "auth": {
            "type": "http_basic",
            "username": dlt.secrets["username"],
            "password": dlt.secrets["password"]
        }
    },
    "resources": [...]
})
```

**oauth2_client_credentials - OAuth2 Client Credentials:**
```python
source = rest_api_source({
    "client": {
        "base_url": "https://api.example.com",
        "auth": {
            "type": "oauth2_client_credentials",
            "token_url": "https://auth.example.com/oauth/token",
            "client_id": dlt.secrets["client_id"],
            "client_secret": dlt.secrets["client_secret"],
            "scopes": ["read", "write"]  # Optional
        }
    },
    "resources": [...]
})
```

**Corresponding .dlt/secrets.toml:**

```toml
# For api_key auth
[sources.my_api]
api_key = "your_api_key_value"

# For bearer auth
[sources.my_api]
bearer_token = "your_bearer_token"

# For http_basic auth
[sources.my_api]
username = "your_username"
password = "your_password"

# For oauth2_client_credentials
[sources.my_api]
client_id = "your_client_id"
client_secret = "your_client_secret"
```

**rest_api_source vs. RESTClient Approach:**

| Feature | rest_api_source (Declarative) | RESTClient (Programmatic) |
|---------|-------------------------------|---------------------------|
| **Configuration** | Dictionary-based | Class-based |
| **Auth Setup** | `auth` dictionary with `type` | Auth class instances |
| **Use Case** | Standard REST APIs | Custom/complex logic |
| **Complexity** | Lower (config-driven) | Higher (code-driven) |
| **Flexibility** | Limited to built-in patterns | Full programmatic control |
| **When to Use** | Rapid prototyping, standard APIs | Custom pagination, error handling |

**When to Use rest_api_source:**
- API follows standard REST patterns
- Authentication is straightforward (API key, Bearer, OAuth)
- Pagination is one of the standard types
- Multiple endpoints with similar structure
- Configuration-driven approach preferred

**When to Use RESTClient:**
- Custom authentication logic required
- Complex error handling or retry strategies
- Dynamic endpoint construction
- Custom pagination not covered by built-in types
- Need fine-grained control over requests

**Complete rest_api_source Example:**

```python
from dlt.sources.rest_api import rest_api_source

@dlt.source
def github_api():
    """GitHub REST API using declarative approach."""

    source = rest_api_source({
        "client": {
            "base_url": "https://api.github.com",
            "auth": {
                "type": "bearer",
                "token": dlt.secrets["github_token"]
            }
        },
        "resource_defaults": {
            "primary_key": "id",
            "write_disposition": "merge"
        },
        "resources": [
            {
                "name": "repositories",
                "endpoint": {
                    "path": "/user/repos",
                    "params": {
                        "per_page": 100
                    },
                    "paginator": {
                        "type": "header_link"
                    }
                }
            },
            {
                "name": "issues",
                "endpoint": {
                    "path": "/repos/{owner}/{repo}/issues",
                    "params": {
                        "state": "open",
                        "per_page": 50
                    }
                }
            }
        ]
    })

    return source

# .dlt/secrets.toml:
# [sources.github_api]
# github_token = "ghp_your_github_token"

pipeline = dlt.pipeline(
    pipeline_name="github_pipeline",
    destination="duckdb",
    dataset_name="github_data"
)

pipeline.run(github_api())
```

#### RESTClient with Credential Classes Integration

Combine RESTClient's programmatic approach with dlt's credential classes for robust authentication and token management.

**Integrating OAuth2Credentials with RESTClient:**

```python
from dlt.sources.credentials import OAuth2Credentials
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth

@dlt.source
def oauth_api_source(credentials: OAuth2Credentials = dlt.secrets.value):
    """Source using OAuth2Credentials with RESTClient."""

    # OAuth2Credentials handles token refresh
    # Create auth using current access_token
    auth = BearerTokenAuth(token=credentials.access_token)

    client = RESTClient(
        base_url="https://api.example.com",
        auth=auth
    )

    @dlt.resource(name="data", write_disposition="merge", primary_key="id")
    def fetch_data():
        for page in client.paginate("/data"):
            # Token automatically refreshed by OAuth2Credentials if expired
            # Update auth if token was refreshed
            if hasattr(credentials, 'access_token'):
                auth.token = credentials.access_token

            yield page

    return fetch_data()

# .dlt/secrets.toml:
# [sources.oauth_api_source.credentials]
# client_id = "your_client_id"
# client_secret = "your_client_secret"
# refresh_token = "your_refresh_token"
```

**Token Refresh in Long-Running Paginated Requests:**

```python
from dlt.sources.credentials import OAuth2Credentials
from dlt.sources.helpers.rest_client import RESTClient

@dlt.source
def long_running_source(credentials: OAuth2Credentials = dlt.secrets.value):
    """Handle token refresh during long pagination."""

    @dlt.resource(name="large_dataset")
    def fetch_large_dataset():
        client = None
        page_count = 0

        for page_num in range(10000):  # Very long pagination
            # Refresh client every 100 pages to pickup new token
            if page_count % 100 == 0 or client is None:
                # Get fresh access token (auto-refreshed if needed)
                from dlt.sources.helpers.rest_client.auth import BearerTokenAuth

                client = RESTClient(
                    base_url="https://api.example.com",
                    auth=BearerTokenAuth(token=credentials.access_token)
                )

            # Fetch page
            response = client.get(f"/data?page={page_num}")
            data = response.json()

            if not data:
                break  # No more data

            yield data
            page_count += 1

    return fetch_large_dataset()
```

**Mixing Custom Auth with Built-in Credential Types:**

```python
from typing import Union
from dlt.sources.credentials import GcpServiceAccountCredentials
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import AuthConfigBase, BearerTokenAuth

class GcpAuthForCustomAPI(AuthConfigBase):
    """Custom auth using GCP credentials for non-Google API."""

    def __init__(self, gcp_credentials: GcpServiceAccountCredentials):
        super().__init__()
        self.gcp_credentials = gcp_credentials

    def __call__(self, request):
        # Get GCP ID token
        from google.oauth2 import service_account
        from google.auth.transport.requests import Request

        credentials = self.gcp_credentials.to_native_credentials()
        credentials.refresh(Request())

        # Use GCP token for custom API
        request.headers["Authorization"] = f"Bearer {credentials.token}"
        return request

@dlt.source
def custom_api_with_gcp_auth(
    credentials: Union[str, GcpServiceAccountCredentials] = dlt.secrets.value
):
    """API that accepts GCP token or simple bearer token."""

    if isinstance(credentials, str):
        # Simple bearer token
        auth = BearerTokenAuth(token=credentials)
    elif isinstance(credentials, GcpServiceAccountCredentials):
        # GCP service account
        auth = GcpAuthForCustomAPI(gcp_credentials=credentials)
    else:
        raise ValueError(f"Unsupported credential type: {type(credentials)}")

    client = RESTClient(
        base_url="https://custom-api.example.com",
        auth=auth
    )

    @dlt.resource
    def data():
        for page in client.paginate("/data"):
            yield page

    return data()
```

**Complete Integration Example:**

```python
from typing import Union
from dlt.sources.credentials import (
    OAuth2Credentials,
    GcpServiceAccountCredentials,
    ConnectionStringCredentials
)
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth, OAuth2ClientCredentials

@dlt.source
def multi_backend_source(
    api_credentials: Union[str, OAuth2Credentials] = dlt.secrets.value,
    db_credentials: ConnectionStringCredentials = dlt.secrets.value
):
    """Source combining API and database backends with credential integration."""

    # Setup API client with OAuth or Bearer
    if isinstance(api_credentials, str):
        api_auth = BearerTokenAuth(token=api_credentials)
    elif isinstance(api_credentials, OAuth2Credentials):
        # Use OAuth2 with automatic refresh
        api_auth = BearerTokenAuth(token=api_credentials.access_token)
    else:
        raise TypeError(f"Unsupported API credential type: {type(api_credentials)}")

    api_client = RESTClient(
        base_url="https://api.example.com",
        auth=api_auth
    )

    @dlt.resource(name="api_data", write_disposition="merge", primary_key="id")
    def fetch_api_data():
        # Fetch from API
        for page in api_client.paginate("/data"):
            yield page

    @dlt.resource(name="db_data", write_disposition="replace")
    def fetch_db_data():
        # Fetch from database using connection string credentials
        import psycopg2

        conn_string = db_credentials.to_native_representation()
        conn = psycopg2.connect(conn_string)
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM reference_data")
        for row in cursor.fetchall():
            yield {
                "id": row[0],
                "value": row[1]
            }

        cursor.close()
        conn.close()

    return fetch_api_data(), fetch_db_data()

# .dlt/secrets.toml:
# [sources.multi_backend_source.api_credentials]
# client_id = "oauth_client_id"
# client_secret = "oauth_secret"
# refresh_token = "refresh_token_value"
#
# [sources.multi_backend_source.db_credentials]
# database = "reference_db"
# username = "readonly_user"
# password = "db_password"
# host = "db.example.com"
# port = 5432
```

**Best Practices:**
- Use credential classes for automatic token refresh
- Recreate RESTClient instances when tokens are refreshed
- Leverage Union types for flexible authentication
- Combine declarative and programmatic approaches as needed
- Test credential refresh logic in long-running pipelines

#### Class-Based Paginators

dlt provides paginator classes for common pagination patterns. Always use class-based paginators for programmatic API interaction.

**JSONLinkPaginator - Next URL in JSON Response:**
```python
from dlt.sources.helpers.rest_client.paginators import JSONLinkPaginator

paginator = JSONLinkPaginator(next_url_path="pagination.next")

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Response structure:
# {
#   "data": [...],
#   "pagination": {"next": "https://api.example.com/users?page=2"}
# }
```

**HeaderLinkPaginator - Link Header Pagination:**
```python
from dlt.sources.helpers.rest_client.paginators import HeaderLinkPaginator

paginator = HeaderLinkPaginator()

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Response header:
# Link: <https://api.example.com/users?page=2>; rel="next"
```

**OffsetPaginator - Offset-Based Pagination:**
```python
from dlt.sources.helpers.rest_client.paginators import OffsetPaginator

paginator = OffsetPaginator(
    limit=100,
    offset=0,
    offset_param="offset",
    limit_param="limit",
    total_path="total"  # JSONPath to total count in response
)

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Requests: /users?offset=0&limit=100, /users?offset=100&limit=100, ...
```

**PageNumberPaginator - Page-Based Pagination:**
```python
from dlt.sources.helpers.rest_client.paginators import PageNumberPaginator

paginator = PageNumberPaginator(
    base_page=1,  # Starting page number (0 or 1)
    page_param="page",
    total_path="total_pages"  # Optional: JSONPath to total pages
)

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Requests: /users?page=1, /users?page=2, ...
```

**JSONResponseCursorPaginator - Cursor in JSON Response:**
```python
from dlt.sources.helpers.rest_client.paginators import JSONResponseCursorPaginator

paginator = JSONResponseCursorPaginator(
    cursor_path="cursors.next",  # JSONPath to cursor in response
    cursor_param="after"  # Query parameter name for cursor
)

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Response structure:
# {
#   "records": [...],
#   "cursors": {"next": "eyJpZCI6MTIzfQ=="}
# }
# Next request: /users?after=eyJpZCI6MTIzfQ==
```

**HeaderCursorPaginator - Cursor in Response Headers:**
```python
from dlt.sources.helpers.rest_client.paginators import HeaderCursorPaginator

paginator = HeaderCursorPaginator(
    cursor_param="after",
    cursor_header_name="X-Next-Cursor"
)

client = RESTClient(
    base_url="https://api.example.com",
    paginator=paginator
)

# Response header: X-Next-Cursor: eyJpZCI6MTIzfQ==
# Next request: /users?after=eyJpZCI6MTIzfQ==
```

**Custom Paginator:**
```python
from dlt.sources.helpers.rest_client.paginators import BasePaginator

class CustomPaginator(BasePaginator):
    def init_request(self, request):
        # Initialize first request (add initial params, headers)
        request.params["page"] = 1
        return request

    def update_state(self, response):
        # Extract pagination info from response
        data = response.json()
        if data.get("has_more"):
            self.next_page = data.get("next_page")
        else:
            self.next_page = None

    def update_request(self, request):
        # Update request with next page info
        if self.next_page:
            request.params["page"] = self.next_page
            return request
        return None  # Stop pagination

paginator = CustomPaginator()
client = RESTClient(base_url="https://api.example.com", paginator=paginator)
```

#### Pagination and Data Selection

**Using paginate() Method:**
The `paginate()` method yields `PageData` objects for each page, providing access to request, response, and extracted data.

```python
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.paginators import JSONLinkPaginator

client = RESTClient(
    base_url="https://api.example.com",
    paginator=JSONLinkPaginator(next_url_path="pagination.next")
)

for page in client.paginate("/users"):
    print(f"Status: {page.response.status_code}")
    print(f"Request URL: {page.request.url}")
    print(f"Data: {page}")  # Extracted data based on data_selector
```

**PageData Object Structure:**
- `page.request`: Prepared request object
- `page.response`: Response object
- `page.paginator`: Paginator instance
- `page.auth`: Authentication handler
- Iteration yields extracted data items

**Data Selection with data_selector:**
Use JSONPath to extract data from nested response structures.

```python
client = RESTClient(
    base_url="https://api.example.com",
    data_selector="data.items"  # Extract from nested structure
)

# Response: {"data": {"items": [...]}}
for page in client.paginate("/users"):
    # page directly yields items from data.items path
    for user in page:
        yield user
```

**Common data_selector Patterns:**
- `"data"`: Extract from `{"data": [...]}`
- `"results"`: Extract from `{"results": [...]}`
- `"data.records"`: Extract from `{"data": {"records": [...]}}`
- `"items"`: Extract from `{"items": [...]}`

**Integration with dlt Resources:**
```python
@dlt.resource(name="users", write_disposition="merge", primary_key="id")
def fetch_users():
    client = RESTClient(
        base_url="https://api.example.com",
        paginator=OffsetPaginator(limit=100),
        data_selector="data.users"
    )

    for page in client.paginate("/users"):
        yield page

@dlt.resource(name="orders", write_disposition="append")
def fetch_orders():
    client = RESTClient(
        base_url="https://api.example.com",
        paginator=JSONResponseCursorPaginator(
            cursor_path="pagination.next_cursor",
            cursor_param="cursor"
        ),
        data_selector="orders"
    )

    for page in client.paginate("/orders"):
        yield page
```

**Shorthand paginate() Function:**
For simple use cases, use the `paginate()` function directly:
```python
from dlt.sources.helpers.rest_client import paginate

for page in paginate(
    "https://api.example.com/users",
    paginator=OffsetPaginator(limit=100),
    auth=BearerTokenAuth(token=dlt.secrets["token"])
):
    yield page
```

#### Error Handling and Retries

RESTClient includes automatic retry logic with exponential backoff for transient errors.

**Default Retry Configuration:**
- `request_timeout`: 60 seconds (connection + read timeout)
- `request_max_attempts`: 5 attempts
- `request_backoff_factor`: 1.0 (exponential backoff multiplier)
- `request_max_retry_delay`: 300 seconds (max wait between retries)

**Custom Retry Configuration via config.toml:**
```toml
[sources.rest_api.config]
request_timeout = 120
request_max_attempts = 10
request_backoff_factor = 2.0
request_max_retry_delay = 600
```

**Retry Behavior:**
- Retries on connection errors, timeouts, and 5xx server errors
- Uses exponential backoff: `backoff_factor * (2 ^ (attempt - 1))`
- Respects `Retry-After` header if present
- Stops retrying on 4xx client errors (except 429 rate limiting)

**Response Hooks for Custom Error Handling:**
```python
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.exceptions import IgnoreResponseException

def handle_rate_limit(response, *args, **kwargs):
    if response.status_code == 429:
        # Custom rate limit handling
        retry_after = int(response.headers.get("Retry-After", 60))
        print(f"Rate limited. Waiting {retry_after} seconds...")
        time.sleep(retry_after)
        # Raise to retry request
        raise IgnoreResponseException()

client = RESTClient(
    base_url="https://api.example.com",
    session=requests.Session()
)
client.session.hooks["response"].append(handle_rate_limit)

for page in client.paginate("/users"):
    yield page
```

**IgnoreResponseException:**
Raise this exception in response hooks to trigger a retry without counting against the max attempts limit.

**Error Debugging Configuration:**
```toml
[sources.rest_api.config]
http_show_error_body = true  # Show response body in error messages
http_max_error_body_length = 1000  # Max chars of error body to display
```

**Error Handling Best Practices:**
```python
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.exceptions import HTTPClientError

@dlt.resource(name="users")
def get_users():
    client = RESTClient(base_url="https://api.example.com")

    try:
        for page in client.paginate("/users"):
            yield page
    except HTTPClientError as e:
        # Log error details
        print(f"HTTP Error: {e.response.status_code}")
        print(f"Response: {e.response.text}")
        raise  # Re-raise to fail the pipeline
```

#### Advanced Features

**Custom Session Management:**
Provide a custom `requests.Session` for advanced control over connections, pooling, and adapters.

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Create session with custom connection pooling
session = requests.Session()
adapter = HTTPAdapter(
    pool_connections=10,
    pool_maxsize=20,
    max_retries=Retry(total=0)  # Disable urllib3 retries (use dlt's)
)
session.mount("https://", adapter)
session.mount("http://", adapter)

client = RESTClient(
    base_url="https://api.example.com",
    session=session
)
```

**URL Sanitization:**
RESTClient automatically sanitizes URLs in logs and error messages to redact sensitive information.

```python
# Sensitive parameters automatically redacted in logs:
# - api_key, apikey, access_token, secret, password, passwd, authorization

client = RESTClient(base_url="https://api.example.com")
response = client.get("/users", params={"api_key": "secret123"})

# Logs show: /users?api_key=***
```

**Debugging Techniques:**

*Enable Debug Logging:*
```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger("dlt.sources.helpers.rest_client")
logger.setLevel(logging.DEBUG)
```

*Inspect PageData Objects:*
```python
for page in client.paginate("/users"):
    print(f"Request: {page.request.method} {page.request.url}")
    print(f"Response Status: {page.response.status_code}")
    print(f"Response Headers: {page.response.headers}")
    print(f"Paginator State: {page.paginator.__dict__}")

    # Process data
    for item in page:
        yield item
```

*Use Response Hooks for Debugging:*
```python
def debug_response(response, *args, **kwargs):
    print(f"URL: {response.url}")
    print(f"Status: {response.status_code}")
    print(f"Headers: {dict(response.headers)}")
    print(f"Body: {response.text[:500]}")  # First 500 chars

client = RESTClient(base_url="https://api.example.com")
client.session.hooks["response"].append(debug_response)
```

**RESTClient vs rest_api_source:**

Use `RESTClient` when:
- You need programmatic control over API requests
- Building custom sources with complex logic
- Implementing custom authentication or pagination
- Fine-grained error handling required
- Dynamic endpoint construction needed

Use `rest_api_source` (declarative) when:
- API follows standard patterns
- Configuration-driven approach preferred
- Rapid prototyping and development
- Multiple endpoints with similar structure
- Leveraging dlt's built-in REST API patterns

**Example: RESTClient Approach**
```python
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth
from dlt.sources.helpers.rest_client.paginators import JSONResponseCursorPaginator

@dlt.source
def github_source(token=dlt.secrets.value):
    @dlt.resource(name="issues", write_disposition="merge", primary_key="id")
    def get_issues(repository):
        client = RESTClient(
            base_url="https://api.github.com",
            auth=BearerTokenAuth(token=token),
            paginator=JSONResponseCursorPaginator(
                cursor_path="pagination.next",
                cursor_param="page"
            )
        )

        for page in client.paginate(f"/repos/{repository}/issues"):
            yield page

    return get_issues("dlt-hub/dlt")

pipeline = dlt.pipeline(
    pipeline_name="github_pipeline",
    destination="duckdb",
    dataset_name="github_data"
)
pipeline.run(github_source())
```

**Example: rest_api_source Approach (Declarative)**
```python
from dlt.sources.rest_api import rest_api_source

source = rest_api_source({
    "client": {
        "base_url": "https://api.github.com",
        "auth": {
            "type": "bearer",
            "token": dlt.secrets["github_token"]
        }
    },
    "resources": [
        {
            "name": "issues",
            "endpoint": {
                "path": "/repos/dlt-hub/dlt/issues",
                "paginator": {
                    "type": "json_link",
                    "next_url_path": "pagination.next"
                }
            },
            "write_disposition": "merge",
            "primary_key": "id"
        }
    ]
})

pipeline.run(source)
```

Both approaches are valid; choose based on your specific requirements for control, complexity, and maintainability.

### Secrets and Credentials Management

dlt provides a comprehensive system for managing credentials and secrets, supporting local development, production deployment, and cloud secret managers.

#### Secret Resolution Priority & Behavior

dlt uses a hierarchical search order when resolving secrets, with environment variables taking highest priority and default parameter values taking lowest priority.

**Resolution Priority (Highest to Lowest):**
1. **Environment Variables** - Highest priority
2. **`.dlt/secrets.toml` file** - Local development secrets
3. **Secure Vaults** - Google Secret Manager, custom providers
4. **Default Parameter Values** - Lowest priority (fallback)

**Progressive Key Elimination:**

When resolving a secret in a decorated function, dlt searches progressively from most specific to least specific paths:

```python
@dlt.source
def github_source(access_token: str = dlt.secrets.value):
    # dlt looks for access_token in this order:
    # 1. sources.github_source.access_token
    # 2. access_token (root level)
    pass
```

**Resolution Flow Example:**

```python
# .dlt/secrets.toml
[sources.github_source]
access_token = "specific_token_for_github"

[sources]
access_token = "general_token_for_all_sources"

# When github_source() is called:
# - Searches: sources.github_source.access_token ✓ (found: "specific_token_for_github")
# - Stops searching (most specific match found)

# If sources.github_source.access_token didn't exist:
# - Searches: sources.access_token ✓ (found: "general_token_for_all_sources")

# If neither existed:
# - Raises ConfigFieldMissingException
```

**Environment Variable Override:**

Environment variables always override TOML files, regardless of specificity:

```bash
# Environment variable takes precedence
export SOURCES__GITHUB_SOURCE__ACCESS_TOKEN="env_token"

# This overrides .dlt/secrets.toml setting:
# [sources.github_source]
# access_token = "toml_token"  # Ignored when env var exists
```

**Key Path Patterns:**

```python
# Source-specific credentials
dlt.secrets["sources.my_api.api_key"]           # Full path
dlt.secrets["sources.my_api.credentials"]       # Credential object

# Destination credentials
dlt.secrets["destination.postgres.password"]    # Nested path
dlt.secrets["destination.bigquery.credentials"] # Complex credentials

# Root-level (shared across sources)
dlt.secrets["api_key"]                          # Fallback for all sources
```

**Resolution Behavior Notes:**
- Secret resolution happens at source/resource instantiation time
- Missing required secrets raise `ConfigFieldMissingException` immediately
- Type coercion failures raise `ConfigValueCannotBeCoercedException`
- Vault providers are queried after TOML files but before defaults
- Once resolved, secrets are cached for the pipeline run duration

#### Local Development: .dlt/secrets.toml

For local pipeline development, store secrets in `.dlt/secrets.toml` file. Structure credentials by matching source and destination names:

```toml
[sources.pipedrive]
pipedrive_api_key = "your_key_here"

[sources.github]
access_token = "ghp_your_token"

[destination.bigquery]
location = "US"

[destination.bigquery.credentials]
project_id = "your_project_id"
private_key = "-----BEGIN PRIVATE KEY-----\n..."
client_email = "service-account@project.iam.gserviceaccount.com"

[destination.postgres]
database = "dlt_data"
username = "loader"
password = "your_password"
host = "localhost"
port = 5432
```

**Important Notes:**
- Keys in TOML files are case-sensitive
- Sections are separated with periods (`.`)
- `.dlt/secrets.toml` should be added to `.gitignore`
- Place `.dlt/` directory in project root or working directory

#### Environment Variables

dlt reads secrets from environment variables using a naming convention with capitalized names and double underscores (`__`) separating sections:

**Conversion Pattern:**
- TOML: `sources.pipedrive.pipedrive_api_key`
- Environment: `SOURCES__PIPEDRIVE__PIPEDRIVE_API_KEY`

**Examples:**
```bash
# Source credentials
export SOURCES__PIPEDRIVE__PIPEDRIVE_API_KEY="your_key"
export SOURCES__GITHUB__ACCESS_TOKEN="ghp_token"

# Destination credentials
export DESTINATION__BIGQUERY__CREDENTIALS__PROJECT_ID="project"
export DESTINATION__BIGQUERY__LOCATION="US"
export DESTINATION__POSTGRES__PASSWORD="password"
```

Environment variables override values in `.dlt/secrets.toml`.

#### Accessing Secrets in Code

dlt provides multiple patterns for accessing secrets in pipeline code.

**Pattern 1: dlt.secrets.value (Recommended with Decorators)**

Use `dlt.secrets.value` as default parameter value in `@dlt.source` or `@dlt.resource` decorated functions. dlt automatically injects the secret:

```python
@dlt.source
def github_source(access_token: str = dlt.secrets.value):
    # access_token automatically resolved from secrets
    @dlt.resource
    def repos():
        headers = {"Authorization": f"Bearer {access_token}"}
        # ... fetch data
        yield data

    return repos()

# Use without passing token explicitly
pipeline.run(github_source())
```

**Secret Resolution:**
- Looks for `sources.github_source.access_token` in secrets
- Falls back to `access_token` at root level
- Raises error if secret not found

**Pattern 2: dlt.secrets["key"] (Dictionary Access)**

Access specific secrets by key path:

```python
@dlt.source
def my_source():
    # Access nested secret
    api_key = dlt.secrets["sources.my_api.api_key"]

    # Access destination credentials
    db_password = dlt.secrets["destination.postgres.password"]

    @dlt.resource
    def data():
        # Use secrets in requests
        yield fetch_data(api_key)

    return data()
```

**Pattern 3: Explicit Resolution (Non-Decorated Functions)**

When calling functions without decorators, explicitly resolve secrets before use:

```python
def create_source():
    # MUST resolve explicitly before creating config
    api_key = dlt.secrets["my_api_key"]

    config = {
        "client": {
            "auth": {
                "type": "api_key",
                "api_key": api_key  # Use resolved value
            }
        }
    }

    return rest_api_source(config)

# Call function
pipeline.run(create_source())
```

#### Error Handling for Credentials

dlt provides specific exceptions and patterns for handling credential-related errors gracefully.

**Common Credential Errors:**

**ConfigFieldMissingException** - Required secret not found:
```python
from dlt.common.configuration.exceptions import ConfigFieldMissingException

@dlt.source
def my_source(api_key: str = dlt.secrets.value):
    try:
        # api_key will raise error if not found
        yield fetch_data(api_key)
    except ConfigFieldMissingException as e:
        print(f"Missing credential: {e.field_name}")
        print(f"Expected at: sources.my_source.{e.field_name}")
        raise

# User-friendly error handling
try:
    pipeline.run(my_source())
except ConfigFieldMissingException:
    print("Please configure credentials in .dlt/secrets.toml:")
    print("[sources.my_source]")
    print("api_key = 'your_key_here'")
```

**ConfigValueCannotBeCoercedException** - Invalid format or type:
```python
from dlt.common.configuration.exceptions import ConfigValueCannotBeCoercedException

@dlt.source
def postgres_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    try:
        conn = credentials.to_native_representation()
    except ConfigValueCannotBeCoercedException as e:
        print(f"Invalid credential format: {e.field_name}")
        print(f"Expected type: {e.expected_type}")
        print(f"Received: {e.value}")
        raise

# Expected format in secrets.toml:
# [sources.postgres_source.credentials]
# database = "db_name"
# username = "user"
# password = "pass"
# host = "localhost"
# port = 5432
```

**Cloud Provider Permission Errors:**
```python
from google.auth.exceptions import DefaultCredentialsError
from botocore.exceptions import NoCredentialsError

@dlt.source
def gcp_source(credentials: GcpServiceAccountCredentials = dlt.secrets.value):
    try:
        client = credentials.to_native_credentials()
    except DefaultCredentialsError:
        print("GCP credentials not found or invalid")
        print("Check GOOGLE_APPLICATION_CREDENTIALS environment variable")
        print("Or provide credentials in .dlt/secrets.toml")
        raise

@dlt.source
def aws_source(credentials: AwsCredentials = dlt.secrets.value):
    try:
        session = credentials.to_native_credentials()
    except NoCredentialsError:
        print("AWS credentials not found")
        print("Check ~/.aws/credentials or environment variables")
        raise
```

**Validation Before Use:**
```python
@dlt.source
def validated_source(api_key: str = dlt.secrets.value):
    # Validate credential format
    if not api_key:
        raise ValueError("api_key cannot be empty")

    if not api_key.startswith("sk_"):
        raise ValueError("api_key must start with 'sk_' prefix")

    if len(api_key) < 32:
        raise ValueError("api_key is too short (minimum 32 characters)")

    # Use validated credential
    @dlt.resource
    def data():
        yield fetch_data(api_key)

    return data()
```

**Graceful Fallbacks:**
```python
from typing import Optional

@dlt.source
def flexible_source(
    api_key: Optional[str] = dlt.secrets.value,
    use_cache: bool = dlt.config.value
):
    # Provide fallback if secret is optional
    if api_key is None:
        print("No API key provided, using cached data")
        @dlt.resource
        def cached_data():
            yield load_from_cache()
        return cached_data()
    else:
        @dlt.resource
        def live_data():
            yield fetch_from_api(api_key)
        return live_data()
```

**Logging Credentials Safely:**
```python
import logging

logger = logging.getLogger(__name__)

@dlt.source
def secure_source(api_key: str = dlt.secrets.value):
    # NEVER log the actual credential
    # ❌ BAD: logger.info(f"Using API key: {api_key}")

    # ✓ GOOD: Log presence without value
    logger.info(f"API key configured: {bool(api_key)}")
    logger.info(f"API key length: {len(api_key)}")
    logger.info(f"API key prefix: {api_key[:4]}***")  # First 4 chars only

    @dlt.resource
    def data():
        yield fetch_data(api_key)

    return data()
```

**Troubleshooting Checklist:**

1. **Secret Not Found Error:**
   - Check `.dlt/secrets.toml` exists in working directory
   - Verify section name matches source/destination name
   - Check for typos in key names (case-sensitive)
   - Try environment variable: `export SOURCES__SOURCE_NAME__KEY="value"`
   - Verify vault provider configuration if using cloud secrets

2. **Invalid Format Error:**
   - Check credential type expectations (string vs. object vs. connection string)
   - Verify JSON syntax if providing credentials as JSON string
   - Ensure TOML structure matches expected credential class
   - Check for missing required fields in credential objects

3. **Permission Denied Errors:**
   - Verify cloud service account has necessary IAM roles
   - Check API key/token hasn't expired
   - Confirm scopes/permissions match API requirements
   - Test credentials independently (e.g., `gcloud auth`, `aws sts get-caller-identity`)

4. **Environment Variable Not Working:**
   - Verify double underscore (`__`) separators in env var name
   - Check case: environment variables must be ALL_CAPS
   - Ensure export statement ran in current shell session
   - Restart application after setting environment variables

5. **Debugging Secret Resolution:**
   ```python
   # Enable debug logging to see secret resolution process
   import logging
   logging.basicConfig(level=logging.DEBUG)
   logger = logging.getLogger("dlt.common.configuration")
   logger.setLevel(logging.DEBUG)

   # dlt will log secret resolution attempts
   pipeline.run(my_source())
   ```

**Pattern 4: Multiple Secrets**

```python
@dlt.source
def database_source(
    host: str = dlt.secrets.value,
    username: str = dlt.secrets.value,
    password: str = dlt.secrets.value,
    database: str = dlt.secrets.value
):
    # All parameters resolved from secrets.toml:
    # [sources.database_source]
    # host = "localhost"
    # username = "user"
    # password = "pass"
    # database = "db"

    connection_string = f"postgresql://{username}:{password}@{host}/{database}"
    # ... use connection
```

#### Programmatic Secret Management

dlt allows setting configuration and secret values programmatically at runtime, which takes precedence over files and environment variables.

**Setting Values at Runtime:**

```python
import dlt

# Set configuration values
dlt.config.set("sources.my_api.base_url", "https://api.example.com")
dlt.config.set("sources.my_api.timeout", 30)

# Set secret values
dlt.secrets.set("sources.my_api.api_key", "runtime_secret_key")
dlt.secrets.set("destination.postgres.password", "runtime_db_password")

# Now run pipeline - uses programmatically set values
@dlt.source
def my_api_source(
    base_url: str = dlt.config.value,
    api_key: str = dlt.secrets.value
):
    # base_url = "https://api.example.com" (from dlt.config.set)
    # api_key = "runtime_secret_key" (from dlt.secrets.set)
    pass

pipeline.run(my_api_source())
```

**Use Cases:**

*Dynamic Credential Configuration:*
```python
def get_credentials_from_vault(environment):
    # Fetch from custom vault/secret service
    return vault_client.get_secret(f"api_key_{environment}")

# Configure dynamically based on environment
environment = os.getenv("ENVIRONMENT", "dev")
api_key = get_credentials_from_vault(environment)

dlt.secrets.set("sources.my_api.api_key", api_key)
dlt.config.set("sources.my_api.base_url", f"https://{environment}.api.example.com")

pipeline.run(my_api_source())
```

*Testing and Mocking:*
```python
import pytest

def test_my_source():
    # Override secrets for testing
    dlt.secrets.set("sources.test_api.api_key", "test_key_12345")
    dlt.config.set("sources.test_api.base_url", "http://localhost:8000")

    # Run source with test credentials
    source = test_api_source()
    pipeline = dlt.pipeline(
        pipeline_name="test_pipeline",
        destination="duckdb",
        dataset_name="test_data",
        dev_mode=True
    )
    result = pipeline.run(source)

    assert result.has_failed_jobs == False
```

*Runtime Adjustments:*
```python
# Multi-tenant application with different credentials per client
def run_pipeline_for_client(client_id, client_api_key):
    # Set client-specific credentials
    dlt.secrets.set(f"sources.api_{client_id}.api_key", client_api_key)
    dlt.config.set(f"sources.api_{client_id}.client_id", client_id)

    @dlt.source(name=f"api_{client_id}")
    def client_source(api_key: str = dlt.secrets.value):
        # Use client-specific API key
        yield fetch_client_data(api_key)

    pipeline = dlt.pipeline(
        pipeline_name=f"client_{client_id}_pipeline",
        destination="bigquery",
        dataset_name=f"client_{client_id}_data"
    )
    pipeline.run(client_source())

# Process multiple clients
for client in clients:
    run_pipeline_for_client(client.id, client.api_key)
```

**Priority Order with Programmatic Values:**

When using `dlt.config.set()` or `dlt.secrets.set()`, these values take **highest priority**, overriding:
- Environment variables
- .dlt/secrets.toml and .dlt/config.toml files
- Vault providers
- Default parameter values

**Resolution Priority (Updated):**
1. **Programmatically set values** (`dlt.secrets.set()`, `dlt.config.set()`) - **Highest**
2. **Environment variables**
3. **`.dlt/secrets.toml` / `.dlt/config.toml` files**
4. **Secure vaults** (Google Secret Manager, custom providers)
5. **Default parameter values** - Lowest

**Complex Credential Objects:**

```python
# Set complex credential objects programmatically
from dlt.sources.credentials import GcpServiceAccountCredentials

credentials_dict = {
    "project_id": "my-project",
    "private_key": "-----BEGIN PRIVATE KEY-----\n...",
    "client_email": "service@project.iam.gserviceaccount.com",
    "token_uri": "https://oauth2.googleapis.com/token"
}

# Set as dictionary - dlt will convert to credential object
dlt.secrets.set("destination.bigquery.credentials", credentials_dict)

# Or set as credential object directly
credentials = GcpServiceAccountCredentials(**credentials_dict)
dlt.secrets.set("destination.bigquery.credentials", credentials)
```

**Clearing Programmatic Values:**

```python
# Remove programmatically set value (falls back to next priority level)
dlt.secrets.set("sources.my_api.api_key", None)
dlt.config.set("sources.my_api.base_url", None)
```

**Best Practices:**
- Use programmatic config for dynamic/runtime scenarios only
- Prefer TOML files for static configuration
- Document programmatic configuration requirements clearly
- Clear sensitive values after use in long-running applications
- Be cautious with precedence - programmatic values override everything

#### Configuration via .dlt/config.toml

Non-sensitive configuration values go in `.dlt/config.toml` (can be version controlled):

```toml
[sources.my_api]
base_url = "https://api.example.com/v1"
timeout = 30
max_retries = 3

[destination.bigquery]
location = "EU"
```

Access config values similarly to secrets:

```python
@dlt.source
def my_api_source(
    base_url: str = dlt.config.value,
    api_key: str = dlt.secrets.value
):
    # base_url from config.toml
    # api_key from secrets.toml
    pass
```

#### Complex Credential Types

dlt provides built-in credential classes for common services, supporting multiple configuration formats.

**ConnectionStringCredentials - Database Connection Strings**

Handles multiple database connection formats:

```toml
# Dictionary form (secrets.toml)
[destination.postgres.credentials]
database = "dlt_data"
username = "loader"
password = "secret"
host = "localhost"
port = 5432

# OR Native connection string form
[destination.postgres]
credentials = "postgresql://loader:secret@localhost:5432/dlt_data"
```

```python
from dlt.sources.credentials import ConnectionStringCredentials

@dlt.source
def postgres_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    # credentials automatically parsed from either format
    conn_string = credentials.to_native_representation()
    # Use with database client
```

**GcpServiceAccountCredentials - Google Cloud Service Accounts**

Accepts service account JSON as string or dictionary:

```toml
# secrets.toml - Option 1: Inline JSON
[destination.bigquery.credentials]
project_id = "my-project"
private_key = "-----BEGIN PRIVATE KEY-----\n..."
client_email = "service@project.iam.gserviceaccount.com"
token_uri = "https://oauth2.googleapis.com/token"

# Option 2: JSON string
[destination.bigquery]
credentials = '{"type": "service_account", "project_id": "my-project", ...}'
```

```python
from dlt.sources.credentials import GcpServiceAccountCredentials

@dlt.source
def bigquery_source(
    credentials: GcpServiceAccountCredentials = dlt.secrets.value
):
    # Automatically converts to Google client credentials
    client = credentials.to_native_credentials()
    # Use with Google API clients
```

Falls back to default credentials from environment (`GOOGLE_APPLICATION_CREDENTIALS`).

**GcpOAuthCredentials - Google OAuth2**

For desktop applications requiring user authorization:

```toml
# secrets.toml
[sources.google_sheets.credentials]
client_id = "your_client_id.apps.googleusercontent.com"
client_secret = "your_client_secret"
refresh_token = "your_refresh_token"
project_id = "your_project"
```

```python
from dlt.sources.credentials import GcpOAuthCredentials

@dlt.source
def google_sheets_source(
    credentials: GcpOAuthCredentials = dlt.secrets.value
):
    # Handles token refresh automatically
    client = credentials.to_native_credentials()
```

Can authenticate via `InstalledAppFlow` when `refresh_token` unavailable.

**AwsCredentials - AWS Authentication**

Manages AWS access keys, session tokens, and profiles:

```toml
# secrets.toml
[destination.s3.credentials]
aws_access_key_id = "AKIAIOSFODNN7EXAMPLE"
aws_secret_access_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
region_name = "us-east-1"

# OR use profile
[destination.s3.credentials]
profile_name = "my-aws-profile"
region_name = "us-east-1"
```

```python
from dlt.sources.credentials import AwsCredentials

@dlt.source
def s3_source(credentials: AwsCredentials = dlt.secrets.value):
    # Converts to boto3 session
    session = credentials.to_native_credentials()
    # Use with AWS services
```

Supports default AWS credential chains (environment variables, `~/.aws/credentials`).

**AzureCredentials - Azure Blob Storage**

Handles Azure authentication with account keys or SAS tokens:

```toml
# secrets.toml - Account key
[destination.azure.credentials]
azure_storage_account_name = "mystorageaccount"
azure_storage_account_key = "your_account_key"

# OR SAS token
[destination.azure.credentials]
azure_storage_account_name = "mystorageaccount"
azure_storage_sas_token = "?sv=2021-01-01&ss=bfqt..."
```

```python
from dlt.sources.credentials import AzureCredentials

@dlt.source
def azure_source(credentials: AzureCredentials = dlt.secrets.value):
    # Converts to adlfs-compatible format
    fs = credentials.to_adlfs_credentials()
```

Uses `DefaultAzureCredential` for environment defaults.

#### Credential Type Conversion Methods

dlt credential classes provide conversion methods to transform credentials into formats required by various client libraries and frameworks.

**to_native_representation() - Connection Strings:**

Converts structured credentials to native connection string format:

```python
from dlt.sources.credentials import ConnectionStringCredentials

credentials = ConnectionStringCredentials(
    database="dlt_data",
    username="loader",
    password="secret",
    host="localhost",
    port=5432
)

# Convert to connection string
conn_string = credentials.to_native_representation()
# Returns: "postgresql://loader:secret@localhost:5432/dlt_data"

# Use with database clients
import psycopg2
conn = psycopg2.connect(conn_string)
```

**to_native_credentials() - Cloud Provider SDKs:**

Converts dlt credentials to native SDK credential objects for GCP and AWS:

```python
# GCP Service Account -> Google Credentials
from dlt.sources.credentials import GcpServiceAccountCredentials

gcp_creds = GcpServiceAccountCredentials(
    project_id="my-project",
    private_key="-----BEGIN PRIVATE KEY-----\n...",
    client_email="service@project.iam.gserviceaccount.com"
)

# Convert to Google credentials object
google_creds = gcp_creds.to_native_credentials()
# Returns: google.auth.credentials.Credentials

# Use with Google client libraries
from google.cloud import bigquery
client = bigquery.Client(credentials=google_creds)

# GCP OAuth -> Google Credentials
from dlt.sources.credentials import GcpOAuthCredentials

oauth_creds = GcpOAuthCredentials(
    client_id="your_client_id.apps.googleusercontent.com",
    client_secret="your_secret",
    refresh_token="your_refresh_token"
)

google_oauth = oauth_creds.to_native_credentials()
# Returns: google.oauth2.credentials.Credentials

# AWS Credentials -> Boto3 Session
from dlt.sources.credentials import AwsCredentials

aws_creds = AwsCredentials(
    aws_access_key_id="AKIAIOSFODNN7EXAMPLE",
    aws_secret_access_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    region_name="us-east-1"
)

# Convert to boto3 session
session = aws_creds.to_native_credentials()
# Returns: boto3.Session

# Use with AWS services
s3_client = session.client('s3')
```

**to_adlfs_credentials() - Azure Filesystem:**

Converts Azure credentials to adlfs (Azure Data Lake FileSystem) format:

```python
from dlt.sources.credentials import AzureCredentials

azure_creds = AzureCredentials(
    azure_storage_account_name="mystorageaccount",
    azure_storage_account_key="your_account_key"
)

# Convert to adlfs credentials dictionary
adlfs_creds = azure_creds.to_adlfs_credentials()
# Returns: {"account_name": "mystorageaccount", "account_key": "your_account_key"}

# Use with adlfs
import adlfs
fs = adlfs.AzureBlobFileSystem(**adlfs_creds)
```

**Conversion Method Reference:**

| Credential Class | Method | Returns | Use With |
|------------------|--------|---------|----------|
| `ConnectionStringCredentials` | `to_native_representation()` | Connection string | psycopg2, SQLAlchemy, pymongo |
| `GcpServiceAccountCredentials` | `to_native_credentials()` | `google.auth.credentials.Credentials` | Google Cloud client libraries |
| `GcpOAuthCredentials` | `to_native_credentials()` | `google.oauth2.credentials.Credentials` | Google API clients (Sheets, Drive) |
| `AwsCredentials` | `to_native_credentials()` | `boto3.Session` | AWS SDK (boto3) |
| `AzureCredentials` | `to_adlfs_credentials()` | Dict | adlfs, Azure Storage SDK |

**Practical Integration Example:**

```python
from dlt.sources.credentials import GcpServiceAccountCredentials
from google.cloud import bigquery

@dlt.source
def bigquery_custom_source(credentials: GcpServiceAccountCredentials = dlt.secrets.value):
    # Convert to native Google credentials
    google_creds = credentials.to_native_credentials()

    # Use with Google BigQuery client
    bq_client = bigquery.Client(credentials=google_creds, project=credentials.project_id)

    @dlt.resource
    def custom_query():
        query = "SELECT * FROM `project.dataset.table`"
        results = bq_client.query(query)
        for row in results:
            yield dict(row)

    return custom_query()
```

**Error Handling in Conversions:**

```python
from dlt.sources.credentials import ConnectionStringCredentials
from dlt.common.configuration.exceptions import ConfigValueCannotBeCoercedException

try:
    credentials = ConnectionStringCredentials(
        database="mydb",
        username="user",
        # Missing required fields: host, password
    )
    conn_string = credentials.to_native_representation()
except ConfigValueCannotBeCoercedException as e:
    print(f"Invalid credential format: {e}")
    # Handle missing required fields
```

#### Credential Parsing and Validation

dlt automatically parses and validates credentials from multiple input formats, providing flexibility and type safety.

**Format Flexibility:**

dlt credential classes accept various input formats and automatically parse them:

**Dictionary Input:**
```python
# .dlt/secrets.toml
[sources.my_api.credentials]
database = "mydb"
username = "user"
password = "pass"
host = "localhost"
port = 5432

# dlt parses as dictionary and converts to ConnectionStringCredentials
@dlt.source
def my_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    # credentials is ConnectionStringCredentials object
    pass
```

**JSON String Input:**
```python
# .dlt/secrets.toml
[sources.my_api]
credentials = '{"database": "mydb", "username": "user", "password": "pass", "host": "localhost", "port": 5432}'

# dlt parses JSON string and converts to credential object
@dlt.source
def my_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    # credentials is ConnectionStringCredentials object
    pass
```

**Native Format Objects:**
```python
# Programmatic configuration with credential object
from dlt.sources.credentials import GcpServiceAccountCredentials

credentials = GcpServiceAccountCredentials(
    project_id="my-project",
    private_key="-----BEGIN PRIVATE KEY-----\n...",
    client_email="service@project.iam.gserviceaccount.com"
)

dlt.secrets.set("sources.my_gcp_source.credentials", credentials)
```

**Connection String Format:**
```python
# .dlt/secrets.toml
[sources.postgres_source]
credentials = "postgresql://user:pass@localhost:5432/mydb"

# dlt parses connection string into structured credentials
@dlt.source
def postgres_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    # credentials.database == "mydb"
    # credentials.username == "user"
    # credentials.host == "localhost"
    pass
```

**Automatic Type Detection:**

dlt uses type hints to determine the expected credential format and validates accordingly:

```python
from dlt.sources.credentials import GcpServiceAccountCredentials, AwsCredentials

@dlt.source
def multi_cloud_source(
    gcp_creds: GcpServiceAccountCredentials = dlt.secrets.value,
    aws_creds: AwsCredentials = dlt.secrets.value
):
    # dlt validates gcp_creds has required GCP fields
    # dlt validates aws_creds has required AWS fields
    pass
```

**Validation Errors:**

**ConfigFieldMissingException** - Missing required credential fields:
```python
# .dlt/secrets.toml - Missing 'password' field
[sources.postgres_source.credentials]
database = "mydb"
username = "user"
host = "localhost"
# password missing!

# Error raised:
# ConfigFieldMissingException: Field 'password' is missing in credentials
```

**ConfigValueCannotBeCoercedException** - Invalid format or type:
```python
# .dlt/secrets.toml - Invalid port type
[sources.postgres_source.credentials]
database = "mydb"
username = "user"
password = "pass"
host = "localhost"
port = "invalid"  # Should be integer

# Error raised:
# ConfigValueCannotBeCoercedException: Cannot convert 'invalid' to int for field 'port'
```

**Format Troubleshooting:**

```python
from dlt.sources.credentials import ConnectionStringCredentials
from dlt.common.configuration.exceptions import ConfigValueCannotBeCoercedException

@dlt.source
def validated_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
    try:
        # Attempt to use credentials
        conn_string = credentials.to_native_representation()
    except ConfigValueCannotBeCoercedException as e:
        print(f"Credential validation failed:")
        print(f"  Field: {e.field_name}")
        print(f"  Expected type: {e.expected_type}")
        print(f"  Received value: {e.value}")
        print(f"\nExpected format in .dlt/secrets.toml:")
        print("[sources.validated_source.credentials]")
        print("database = \"mydb\"")
        print("username = \"user\"")
        print("password = \"pass\"")
        print("host = \"localhost\"")
        print("port = 5432")
        raise
```

**Credential Class Validation Example:**

```python
from dlt.sources.credentials import GcpServiceAccountCredentials
import json

# Valid credential validation
def validate_gcp_credentials(creds_json: str) -> bool:
    try:
        creds_dict = json.loads(creds_json)
        credentials = GcpServiceAccountCredentials.parse_obj(creds_dict)

        # Required fields validated automatically
        assert credentials.project_id is not None
        assert credentials.private_key is not None
        assert credentials.client_email is not None

        return True
    except (json.JSONDecodeError, ConfigValueCannotBeCoercedException):
        return False

# Usage in source
@dlt.source
def safe_gcp_source(credentials: GcpServiceAccountCredentials = dlt.secrets.value):
    # credentials already validated by dlt
    client = credentials.to_native_credentials()
    # Safe to use
```

**OAuth2Credentials - Generic OAuth2**

Base class for OAuth 2.0 flows with custom APIs beyond GCP. Handles automatic token refresh and provides flexible OAuth configuration.

**Configuration:**

```toml
# secrets.toml
[sources.my_oauth_api.credentials]
client_id = "your_client_id"
client_secret = "your_client_secret"
refresh_token = "your_refresh_token"
access_token = "current_access_token"  # Optional: will be refreshed if expired
scopes = ["read", "write", "admin"]
token_expiry = "2024-12-31T23:59:59Z"  # Optional: ISO 8601 format
```

**Fields:**
- `client_id` (required): OAuth application client ID
- `client_secret` (required): OAuth application secret
- `refresh_token` (optional): Long-lived token for obtaining new access tokens
- `access_token` (optional): Current access token (refreshed automatically)
- `scopes` (optional): List of OAuth scopes/permissions
- `token_expiry` (optional): Access token expiration timestamp

**Basic Usage:**

```python
from dlt.sources.credentials import OAuth2Credentials

@dlt.source
def oauth_api_source(credentials: OAuth2Credentials = dlt.secrets.value):
    # Access token automatically refreshed if expired
    access_token = credentials.access_token

    # Use in API requests
    headers = {"Authorization": f"Bearer {access_token}"}
    @dlt.resource
    def data():
        response = requests.get(
            "https://api.example.com/data",
            headers=headers
        )
        yield response.json()

    return data()
```

**Automatic Token Refresh:**

OAuth2Credentials automatically refreshes expired access tokens:

```python
from dlt.sources.credentials import OAuth2Credentials
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth

@dlt.source
def auto_refresh_source(credentials: OAuth2Credentials = dlt.secrets.value):
    # Token refresh handled transparently
    @dlt.resource
    def long_running_resource():
        # Even if access_token expires during execution,
        # it will be refreshed automatically
        for page in range(1000):
            # Each iteration uses fresh token
            headers = {"Authorization": f"Bearer {credentials.access_token}"}
            data = fetch_page(page, headers)
            yield data

    return long_running_resource()
```

**Custom OAuth Token Endpoints:**

For APIs with custom OAuth implementations:

```python
from dlt.sources.credentials import OAuth2Credentials

class CustomOAuthCredentials(OAuth2Credentials):
    token_url: str = "https://custom-api.com/oauth/token"

    def refresh_access_token(self):
        # Custom token refresh logic
        response = requests.post(
            self.token_url,
            data={
                "grant_type": "refresh_token",
                "client_id": self.client_id,
                "client_secret": self.client_secret,
                "refresh_token": self.refresh_token
            }
        )
        token_data = response.json()
        self.access_token = token_data["access_token"]
        # Update expiry if provided
        if "expires_in" in token_data:
            from datetime import datetime, timedelta
            self.token_expiry = datetime.utcnow() + timedelta(seconds=token_data["expires_in"])

@dlt.source
def custom_oauth_source(credentials: CustomOAuthCredentials = dlt.secrets.value):
    # Use custom OAuth implementation
    access_token = credentials.access_token
    # ...
```

**Integration with RESTClient:**

```python
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import OAuth2ClientCredentials

@dlt.source
def rest_oauth_source(
    client_id: str = dlt.secrets.value,
    client_secret: str = dlt.secrets.value
):
    # RESTClient handles OAuth2 client credentials flow
    auth = OAuth2ClientCredentials(
        access_token_url="https://api.example.com/oauth/token",
        client_id=client_id,
        client_secret=client_secret,
        scope="read write"
    )

    client = RESTClient(
        base_url="https://api.example.com",
        auth=auth
    )

    @dlt.resource
    def data():
        # Token refresh handled by RESTClient
        for page in client.paginate("/data"):
            yield page

    return data()
```

**Use Cases:**

- **Custom API OAuth:** APIs with non-standard OAuth2 implementations
- **Multiple Scopes:** Managing different permission levels
- **Long-Running Processes:** Pipelines that run longer than token expiry
- **Token Persistence:** Storing and reusing refresh tokens across runs
- **Multi-User Applications:** Different OAuth credentials per user/tenant

**Comparison: OAuth2Credentials vs. GcpOAuthCredentials:**

| Feature | OAuth2Credentials | GcpOAuthCredentials |
|---------|-------------------|---------------------|
| **Purpose** | Generic OAuth2 APIs | Google-specific OAuth |
| **Token Endpoint** | Custom | Google OAuth endpoints |
| **Scopes** | Flexible | Google API scopes |
| **Native Format** | Access token | Google credentials object |
| **Auto-Refresh** | Yes | Yes (via Google SDK) |
| **Use With** | Custom APIs | Google APIs (Sheets, Drive, etc.) |

**Example: Multiple OAuth APIs:**

```python
@dlt.source
def multi_oauth_source(
    github_creds: OAuth2Credentials = dlt.secrets.value,
    gitlab_creds: OAuth2Credentials = dlt.secrets.value
):
    # Different OAuth credentials for different APIs
    @dlt.resource
    def github_data():
        headers = {"Authorization": f"Bearer {github_creds.access_token}"}
        yield fetch_github_data(headers)

    @dlt.resource
    def gitlab_data():
        headers = {"Authorization": f"Bearer {gitlab_creds.access_token}"}
        yield fetch_gitlab_data(headers)

    return github_data(), gitlab_data()

# .dlt/secrets.toml
# [sources.multi_oauth_source.github_creds]
# client_id = "github_client_id"
# client_secret = "github_secret"
# refresh_token = "github_refresh_token"
#
# [sources.multi_oauth_source.gitlab_creds]
# client_id = "gitlab_client_id"
# client_secret = "gitlab_secret"
# refresh_token = "gitlab_refresh_token"
```

#### Union Types for Multiple Credential Options

Sources can accept multiple credential types using Union types, enabling flexible authentication where users can choose their preferred method.

**Basic Union Type Pattern:**

```python
from typing import Union
from dlt.sources.credentials import (
    ConnectionStringCredentials,
    GcpServiceAccountCredentials
)

@dlt.source
def flexible_source(
    credentials: Union[
        ConnectionStringCredentials,
        GcpServiceAccountCredentials,
        str  # Plain string token
    ] = dlt.secrets.value
):
    # dlt automatically injects correct type based on config
    if isinstance(credentials, str):
        # Handle simple token
        print("Using simple string token")
        api_key = credentials
    elif isinstance(credentials, ConnectionStringCredentials):
        # Handle connection string
        print("Using connection string credentials")
        conn_string = credentials.to_native_representation()
    elif isinstance(credentials, GcpServiceAccountCredentials):
        # Handle GCP credentials
        print("Using GCP service account")
        client = credentials.to_native_credentials()
```

Users provide whichever format suits their needs; dlt handles resolution and type validation.

**Type Validation with isinstance():**

dlt injects the appropriate credential type based on configuration structure, enabling runtime type checking:

```python
from typing import Union
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth, APIKeyAuth

@dlt.source
def multi_auth_source(
    credentials: Union[str, dict] = dlt.secrets.value
):
    # Branching logic based on injected type
    if isinstance(credentials, str):
        # Simple token - use as Bearer
        auth = BearerTokenAuth(token=credentials)
    elif isinstance(credentials, dict):
        # Complex credentials - extract API key
        auth = APIKeyAuth(
            name=credentials.get("header_name", "X-API-Key"),
            api_key=credentials["api_key"],
            location="header"
        )
    else:
        raise ValueError(f"Unexpected credential type: {type(credentials)}")

    # Use auth with RESTClient
    client = RESTClient(base_url="https://api.example.com", auth=auth)
    # ...
```

**Real-World Example: API Key OR Bearer Token OR OAuth:**

```python
from typing import Union
from dlt.sources.credentials import OAuth2Credentials
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth, APIKeyAuth, OAuth2ClientCredentials

@dlt.source
def universal_api_source(
    credentials: Union[
        str,              # Simple API key or bearer token
        OAuth2Credentials, # OAuth2 flow
        dict              # Complex config
    ] = dlt.secrets.value,
    auth_type: str = dlt.config.value  # "api_key", "bearer", or "oauth"
):
    """
    Supports three authentication methods:
    1. Simple string token (API key or bearer)
    2. OAuth2 credentials
    3. Custom dict configuration
    """

    # Determine auth based on type and config
    if isinstance(credentials, str):
        if auth_type == "api_key":
            auth = APIKeyAuth(name="X-API-Key", api_key=credentials, location="header")
        else:  # bearer
            auth = BearerTokenAuth(token=credentials)

    elif isinstance(credentials, OAuth2Credentials):
        auth = BearerTokenAuth(token=credentials.access_token)
        # OAuth token refresh handled automatically

    elif isinstance(credentials, dict):
        # Custom handling for dict-based credentials
        if "api_key" in credentials:
            auth = APIKeyAuth(
                name=credentials.get("key_name", "X-API-Key"),
                api_key=credentials["api_key"],
                location=credentials.get("location", "header")
            )
        elif "bearer_token" in credentials:
            auth = BearerTokenAuth(token=credentials["bearer_token"])
        else:
            raise ValueError("Dict credentials must contain 'api_key' or 'bearer_token'")

    client = RESTClient(base_url="https://api.example.com", auth=auth)

    @dlt.resource
    def data():
        for page in client.paginate("/data"):
            yield page

    return data()
```

**Configuration Options for Union Types:**

Users can provide any of the supported formats:

```toml
# Option 1: Simple string (.dlt/secrets.toml)
[sources.universal_api_source]
credentials = "sk_simple_api_key_12345"
auth_type = "api_key"

# Option 2: OAuth2 credentials
[sources.universal_api_source.credentials]
client_id = "oauth_client_id"
client_secret = "oauth_secret"
refresh_token = "refresh_token_value"
auth_type = "oauth"

# Option 3: Dictionary config
[sources.universal_api_source.credentials]
api_key = "complex_key_12345"
key_name = "Authorization"
location = "header"
auth_type = "api_key"
```

**Type Resolution Logic:**

dlt determines which type to inject based on configuration structure:

```python
# String credentials
credentials = "simple_token"  # → str

# OAuth2 credentials (has OAuth-specific fields)
credentials = {
    "client_id": "...",
    "client_secret": "...",
    "refresh_token": "..."
}  # → OAuth2Credentials

# Connection string (has database fields)
credentials = {
    "database": "db",
    "username": "user",
    "password": "pass",
    "host": "localhost"
}  # → ConnectionStringCredentials

# Generic dict (no matching credential class)
credentials = {
    "api_key": "...",
    "custom_field": "..."
}  # → dict
```

**Advanced Example: Multi-Cloud Credentials:**

```python
from typing import Union
from dlt.sources.credentials import (
    GcpServiceAccountCredentials,
    AwsCredentials,
    AzureCredentials
)

@dlt.source
def multi_cloud_source(
    credentials: Union[
        GcpServiceAccountCredentials,
        AwsCredentials,
        AzureCredentials
    ] = dlt.secrets.value
):
    """
    Works with GCP, AWS, or Azure credentials.
    User provides whichever cloud they're using.
    """

    if isinstance(credentials, GcpServiceAccountCredentials):
        # GCP-specific logic
        from google.cloud import storage
        client = storage.Client(credentials=credentials.to_native_credentials())

        @dlt.resource
        def gcp_data():
            bucket = client.bucket("my-bucket")
            for blob in bucket.list_blobs():
                yield {"name": blob.name, "size": blob.size}

        return gcp_data()

    elif isinstance(credentials, AwsCredentials):
        # AWS-specific logic
        session = credentials.to_native_credentials()
        s3_client = session.client('s3')

        @dlt.resource
        def aws_data():
            response = s3_client.list_objects_v2(Bucket="my-bucket")
            for obj in response.get('Contents', []):
                yield {"name": obj["Key"], "size": obj["Size"]}

        return aws_data()

    elif isinstance(credentials, AzureCredentials):
        # Azure-specific logic
        from azure.storage.blob import BlobServiceClient

        blob_service = BlobServiceClient(
            account_url=f"https://{credentials.azure_storage_account_name}.blob.core.windows.net",
            credential=credentials.azure_storage_account_key
        )

        @dlt.resource
        def azure_data():
            container_client = blob_service.get_container_client("my-container")
            for blob in container_client.list_blobs():
                yield {"name": blob.name, "size": blob.size}

        return azure_data()

# Users configure for their cloud:
# .dlt/secrets.toml for GCP:
# [sources.multi_cloud_source.credentials]
# project_id = "my-gcp-project"
# private_key = "..."
# client_email = "..."

# .dlt/secrets.toml for AWS:
# [sources.multi_cloud_source.credentials]
# aws_access_key_id = "..."
# aws_secret_access_key = "..."
# region_name = "us-east-1"

# .dlt/secrets.toml for Azure:
# [sources.multi_cloud_source.credentials]
# azure_storage_account_name = "..."
# azure_storage_account_key = "..."
```

**Best Practices for Union Types:**

1. **Order matters**: List most specific types first, generic types last
   ```python
   Union[OAuth2Credentials, ConnectionStringCredentials, str]  # ✓ Good
   Union[str, OAuth2Credentials, ConnectionStringCredentials]  # ✗ Risky
   ```

2. **Use isinstance() for type checking**: Never assume type without checking
   ```python
   if isinstance(credentials, OAuth2Credentials):  # ✓ Explicit check
       token = credentials.access_token
   ```

3. **Provide clear documentation**: Tell users which credential formats are accepted
   ```python
   def my_source(credentials: Union[str, OAuth2Credentials] = dlt.secrets.value):
       """
       Accepts credentials in two formats:
       - Simple string: API key or bearer token
       - OAuth2Credentials: For OAuth2 flow

       Examples in .dlt/secrets.toml:
       # Simple token:
       credentials = "your_api_key"

       # OAuth2:
       [credentials]
       client_id = "..."
       client_secret = "..."
       """
   ```

4. **Handle all type branches**: Ensure every Union type has corresponding logic
   ```python
   if isinstance(credentials, TypeA):
       # Handle TypeA
   elif isinstance(credentials, TypeB):
       # Handle TypeB
   else:
       raise TypeError(f"Unexpected credential type: {type(credentials)}")
   ```

**Type Hints and IDE Support:**

Union types provide excellent IDE autocomplete and type checking:

```python
from typing import Union

@dlt.source
def typed_source(
    credentials: Union[str, OAuth2Credentials] = dlt.secrets.value
):
    # IDE knows credentials is either str or OAuth2Credentials
    if isinstance(credentials, OAuth2Credentials):
        # IDE autocompletes OAuth2Credentials methods here
        token = credentials.access_token  # ✓ Autocomplete works
        credentials.refresh_access_token()  # ✓ Autocomplete works
    else:
        # IDE knows credentials is str here
        token = credentials  # ✓ Type inference works
```

#### Google Cloud Secret Manager Integration

dlt can retrieve secrets from Google Cloud Secret Manager with proper IAM permissions, enabling centralized secret management for production deployments.

**Configuration:**

```toml
# .dlt/config.toml
[runtime]
providers = ["google_secrets"]

[runtime.google_secrets]
project_id = "your-gcp-project-id"
list_secrets = true  # Enable listing secrets (recommended)
only_toml_fragments = true  # Use TOML fragments to minimize API calls
```

**Detailed IAM Setup:**

1. **Create or identify a service account:**
   ```bash
   gcloud iam service-accounts create dlt-pipeline-runner \
       --description="Service account for dlt pipelines" \
       --display-name="dlt Pipeline Runner"
   ```

2. **Grant Secret Manager access:**
   ```bash
   # Grant Secret Accessor role (read secrets)
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
       --member="serviceAccount:dlt-pipeline-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
       --role="roles/secretmanager.secretAccessor"

   # Optionally grant Secret Viewer role (list secrets)
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
       --member="serviceAccount:dlt-pipeline-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
       --role="roles/secretmanager.viewer"
   ```

3. **Authenticate the pipeline environment:**
   ```bash
   # Set application default credentials
   export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"

   # Or use gcloud auth for development
   gcloud auth application-default login
   ```

**Secret Naming Conventions:**

dlt expects secrets in Google Secret Manager to follow the same path structure as `.dlt/secrets.toml`:

**TOML Structure:**
```toml
[sources.github_source]
access_token = "ghp_token_value"

[destination.bigquery.credentials]
project_id = "my-project"
```

**Secret Manager Names (Flat Format):**
```
sources__github_source__access_token
destination__bigquery__credentials__project_id
```

**Secret Manager Names (TOML Fragment Format - Recommended):**

Store entire TOML sections as secret values to minimize API calls:

```
Secret Name: sources__github_source
Secret Value:
access_token = "ghp_token_value"

Secret Name: destination__bigquery__credentials
Secret Value:
project_id = "my-project"
private_key = "-----BEGIN PRIVATE KEY-----\n..."
client_email = "service@project.iam.gserviceaccount.com"
```

**Creating Secrets in Secret Manager:**

```bash
# Create secret with TOML fragment
echo 'access_token = "ghp_your_github_token"' | \
    gcloud secrets create sources__github_source__credentials \
    --data-file=- \
    --replication-policy="automatic"

# Create secret with complex credentials
cat > /tmp/bigquery_creds.toml <<EOF
project_id = "my-project"
private_key = "-----BEGIN PRIVATE KEY-----\n..."
client_email = "service@project.iam.gserviceaccount.com"
EOF

gcloud secrets create destination__bigquery__credentials \
    --data-file=/tmp/bigquery_creds.toml \
    --replication-policy="automatic"

# Clean up temp file
rm /tmp/bigquery_creds.toml
```

**list_secrets Flag Explained:**

```toml
[runtime.google_secrets]
list_secrets = true  # Recommended for production
```

- **`true`**: dlt calls `secretmanager.secrets.list()` to discover all available secrets
  - Requires `secretmanager.viewer` or `secretmanager.admin` role
  - Better performance when many secrets exist
  - Recommended for production

- **`false`**: dlt queries secrets individually by expected name
  - Only requires `secretmanager.secretAccessor` role
  - More API calls if many secrets needed
  - Use when list permission unavailable

**only_toml_fragments Flag Explained:**

```toml
[runtime.google_secrets]
only_toml_fragments = true  # Recommended to reduce API calls
```

- **`true`**: Expects secrets to contain TOML fragments (entire sections)
  - Reduces number of Secret Manager API calls
  - More efficient for complex credentials
  - Recommended for production

- **`false`**: Expects individual key-value secrets
  - One secret per credential field
  - More granular but more API calls
  - Use for simple credentials or when sharing secrets across projects

**Caching Behavior:**

Secrets fetched from Google Secret Manager are cached in memory for the duration of the pipeline run:

```python
import dlt

# First access - fetches from Secret Manager
pipeline.run(my_source())  # API call to Secret Manager

# Subsequent accesses - uses cached value
pipeline.run(my_source())  # No API call (cached)

# Process restart required for secret updates
# Changing secret in Secret Manager won't affect running process
```

**Refresh Handling:**

To pick up secret changes without restarting:

```python
# Not recommended - dlt caches automatically
# Secret updates require process restart

# For development/testing, use programmatic override:
if os.getenv("ENVIRONMENT") == "dev":
    # Override with local secrets during development
    dlt.secrets.set("sources.my_api.api_key", os.getenv("DEV_API_KEY"))
else:
    # Production uses Secret Manager (cached)
    pass
```

**Complete Setup Example:**

```bash
# 1. Create service account
gcloud iam service-accounts create dlt-runner \
    --project=my-project

# 2. Grant permissions
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:dlt-runner@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:dlt-runner@my-project.iam.gserviceaccount.com" \
    --role="roles/secretmanager.viewer"

# 3. Create service account key
gcloud iam service-accounts keys create dlt-key.json \
    --iam-account=dlt-runner@my-project.iam.gserviceaccount.com

# 4. Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/dlt-key.json"

# 5. Create secrets
echo 'api_key = "your_api_key_value"' | \
    gcloud secrets create sources__my_api__credentials \
    --data-file=- \
    --replication-policy="automatic"

# 6. Configure dlt
cat > .dlt/config.toml <<EOF
[runtime]
providers = ["google_secrets"]

[runtime.google_secrets]
project_id = "my-project"
list_secrets = true
only_toml_fragments = true
EOF

# 7. Run pipeline - secrets loaded automatically
python my_pipeline.py
```

**Error Handling:**

```python
from google.api_core.exceptions import PermissionDenied, NotFound

@dlt.source
def gcp_secret_source(api_key: str = dlt.secrets.value):
    try:
        # dlt will attempt to fetch from Secret Manager
        @dlt.resource
        def data():
            yield fetch_data(api_key)

        return data()

    except PermissionDenied:
        print("ERROR: Missing Secret Manager permissions")
        print("Required roles:")
        print("  - roles/secretmanager.secretAccessor")
        print("  - roles/secretmanager.viewer (if list_secrets=true)")
        raise

    except NotFound:
        print("ERROR: Secret not found in Secret Manager")
        print("Expected secret name: sources__gcp_secret_source__api_key")
        print("Create secret:")
        print("  echo 'api_key = \"value\"' | gcloud secrets create sources__gcp_secret_source__api_key --data-file=-")
        raise
```

**Best Practices:**

- Use TOML fragments (`only_toml_fragments=true`) to reduce API calls
- Enable secret listing (`list_secrets=true`) when permissions allow
- Use automatic replication policy for high availability
- Implement secret rotation through versioning in Secret Manager
- Use labels in Secret Manager to organize dlt secrets
- Document required secret names in pipeline README
- Test IAM permissions before production deployment

#### Custom Vault Providers

Implement custom secret management by subclassing `VaultDocProvider` to integrate with enterprise secret management systems like HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or internal vaults.

**Complete VaultDocProvider Implementation:**

```python
from dlt.common.configuration.providers import VaultDocProvider
from dlt.common.typing import DictStrAny, StrAny
from typing import Any, Optional, List
import logging

logger = logging.getLogger(__name__)

class HashiCorpVaultProvider(VaultDocProvider):
    """Custom vault provider for HashiCorp Vault integration."""

    def __init__(self, vault_url: str, token: str, mount_point: str = "secret"):
        """
        Initialize HashiCorp Vault provider.

        Args:
            vault_url: Vault server URL (e.g., "https://vault.example.com:8200")
            token: Vault authentication token
            mount_point: KV secrets engine mount point (default: "secret")
        """
        super().__init__()
        self.vault_url = vault_url
        self.token = token
        self.mount_point = mount_point

        # Initialize Vault client
        import hvac
        self.client = hvac.Client(url=vault_url, token=token)

        if not self.client.is_authenticated():
            raise ValueError("Vault authentication failed")

        logger.info(f"Initialized HashiCorp Vault provider at {vault_url}")

    def get_value(self, key: str, *args: Any, **kwargs: Any) -> Any:
        """
        Retrieve a single secret value from Vault.

        Args:
            key: Secret key path (e.g., "sources.github_source.access_token")

        Returns:
            Secret value or None if not found
        """
        # Convert dlt key path to Vault path
        # "sources.github_source.access_token" -> "dlt/sources/github_source"
        vault_path = self._dlt_key_to_vault_path(key)

        try:
            # Read secret from Vault KV v2
            secret_response = self.client.secrets.kv.v2.read_secret_version(
                path=vault_path,
                mount_point=self.mount_point
            )

            secret_data = secret_response['data']['data']

            # Extract specific field from secret
            field_name = key.split('.')[-1]  # Last part of key
            return secret_data.get(field_name)

        except Exception as e:
            logger.debug(f"Secret not found in Vault: {key} ({e})")
            return None

    def get_dict(self, key: str, *args: Any, **kwargs: Any) -> Optional[DictStrAny]:
        """
        Retrieve a dictionary of secrets from Vault.

        Args:
            key: Secret key path (e.g., "sources.github_source")

        Returns:
            Dictionary of secret values or None if not found
        """
        vault_path = self._dlt_key_to_vault_path(key)

        try:
            secret_response = self.client.secrets.kv.v2.read_secret_version(
                path=vault_path,
                mount_point=self.mount_point
            )

            # Return all data from Vault secret
            return secret_response['data']['data']

        except Exception as e:
            logger.debug(f"Secret dict not found in Vault: {key} ({e})")
            return None

    def get_list(self, key: str, *args: Any, **kwargs: Any) -> Optional[List[Any]]:
        """
        Retrieve a list of values from Vault.

        Args:
            key: Secret key path

        Returns:
            List of values or None if not found
        """
        value = self.get_value(key, *args, **kwargs)
        if isinstance(value, list):
            return value
        return None

    def _dlt_key_to_vault_path(self, dlt_key: str) -> str:
        """
        Convert dlt key path to Vault path.

        Examples:
            "sources.github_source.access_token" -> "dlt/sources/github_source"
            "destination.bigquery.credentials" -> "dlt/destination/bigquery"
        """
        parts = dlt_key.split('.')
        # Remove last part (field name) for Vault path
        vault_path = '/'.join(['dlt'] + parts[:-1])
        return vault_path

    @property
    def supports_sections(self) -> bool:
        """Indicates this provider supports hierarchical sections."""
        return True

    @property
    def name(self) -> str:
        """Provider name for logging."""
        return "HashiCorpVault"


# Register provider with dlt
def setup_vault_provider():
    """Register custom Vault provider with dlt."""
    import os
    from dlt.common.configuration import providers

    vault_url = os.getenv("VAULT_ADDR", "https://vault.example.com:8200")
    vault_token = os.getenv("VAULT_TOKEN")

    if vault_token:
        vault_provider = HashiCorpVaultProvider(
            vault_url=vault_url,
            token=vault_token,
            mount_point="secret"
        )
        providers.register_provider(vault_provider)
        logger.info("Registered HashiCorp Vault provider")

# Call during application initialization
setup_vault_provider()
```

**AWS Secrets Manager Provider:**

```python
from dlt.common.configuration.providers import VaultDocProvider
import boto3
import json

class AWSSecretsManagerProvider(VaultDocProvider):
    """Custom vault provider for AWS Secrets Manager."""

    def __init__(self, region_name: str = "us-east-1"):
        super().__init__()
        self.client = boto3.client('secretsmanager', region_name=region_name)

    def get_value(self, key: str, *args, **kwargs):
        """Get single secret value from AWS Secrets Manager."""
        secret_name = self._dlt_key_to_aws_secret_name(key)

        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            secret_data = json.loads(response['SecretString'])

            # Extract field
            field_name = key.split('.')[-1]
            return secret_data.get(field_name)

        except self.client.exceptions.ResourceNotFoundException:
            return None

    def get_dict(self, key: str, *args, **kwargs):
        """Get dictionary of secrets from AWS Secrets Manager."""
        secret_name = self._dlt_key_to_aws_secret_name(key)

        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])

        except self.client.exceptions.ResourceNotFoundException:
            return None

    def _dlt_key_to_aws_secret_name(self, dlt_key: str) -> str:
        """Convert dlt key to AWS secret name."""
        # "sources.github_source.access_token" -> "dlt/sources/github_source"
        parts = dlt_key.split('.')
        return '/'.join(['dlt'] + parts[:-1])

    @property
    def name(self) -> str:
        return "AWSSecretsManager"


# Register
from dlt.common.configuration import providers
providers.register_provider(AWSSecretsManagerProvider(region_name="us-east-1"))
```

**Azure Key Vault Provider:**

```python
from dlt.common.configuration.providers import VaultDocProvider
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import json

class AzureKeyVaultProvider(VaultDocProvider):
    """Custom vault provider for Azure Key Vault."""

    def __init__(self, vault_url: str):
        """
        Args:
            vault_url: Key Vault URL (e.g., "https://my-vault.vault.azure.net/")
        """
        super().__init__()
        credential = DefaultAzureCredential()
        self.client = SecretClient(vault_url=vault_url, credential=credential)

    def get_value(self, key: str, *args, **kwargs):
        """Get secret from Azure Key Vault."""
        # Azure Key Vault requires alphanumeric + hyphens
        secret_name = self._dlt_key_to_azure_name(key)

        try:
            secret = self.client.get_secret(secret_name)
            # Try to parse as JSON first
            try:
                secret_data = json.loads(secret.value)
                field_name = key.split('.')[-1]
                return secret_data.get(field_name)
            except json.JSONDecodeError:
                # Return as plain string
                return secret.value

        except Exception:
            return None

    def _dlt_key_to_azure_name(self, dlt_key: str) -> str:
        """Convert dlt key to Azure Key Vault name (alphanumeric + hyphens only)."""
        # "sources.github_source.access_token" -> "dlt-sources-github-source"
        parts = dlt_key.split('.')
        return '-'.join(['dlt'] + parts[:-1])

    @property
    def name(self) -> str:
        return "AzureKeyVault"


# Register
from dlt.common.configuration import providers
providers.register_provider(
    AzureKeyVaultProvider(vault_url="https://my-vault.vault.azure.net/")
)
```

**Registration Patterns:**

**Early Registration (Recommended):**
```python
# Register before importing pipeline code
from dlt.common.configuration import providers

# Register custom vault
providers.register_provider(MyVaultProvider())

# Now import and run pipeline
import dlt
from my_pipeline import my_source

pipeline.run(my_source())
```

**Context Manager Pattern:**
```python
from contextlib import contextmanager
from dlt.common.configuration import providers

@contextmanager
def vault_context(vault_provider):
    """Context manager for temporary vault registration."""
    # Register
    providers.register_provider(vault_provider)
    try:
        yield
    finally:
        # Cleanup if needed
        pass

# Usage
with vault_context(MyVaultProvider()):
    pipeline.run(my_source())
```

**Lifecycle and Error Recovery:**

```python
class ResilientVaultProvider(VaultDocProvider):
    """Vault provider with retry logic and error recovery."""

    def __init__(self, vault_url: str, token: str, max_retries: int = 3):
        super().__init__()
        self.vault_url = vault_url
        self.token = token
        self.max_retries = max_retries
        self._init_client()

    def _init_client(self):
        """Initialize or reinitialize vault client."""
        import hvac
        self.client = hvac.Client(url=self.vault_url, token=self.token)

    def get_value(self, key: str, *args, **kwargs):
        """Get secret with retry logic."""
        for attempt in range(self.max_retries):
            try:
                vault_path = self._dlt_key_to_vault_path(key)
                secret_response = self.client.secrets.kv.v2.read_secret_version(
                    path=vault_path
                )
                field_name = key.split('.')[-1]
                return secret_response['data']['data'].get(field_name)

            except Exception as e:
                logger.warning(f"Vault access failed (attempt {attempt + 1}/{self.max_retries}): {e}")

                if attempt < self.max_retries - 1:
                    # Reinitialize client and retry
                    self._init_client()
                    continue
                else:
                    # Max retries exceeded
                    logger.error(f"Failed to retrieve secret {key} after {self.max_retries} attempts")
                    return None

    def _dlt_key_to_vault_path(self, dlt_key: str) -> str:
        parts = dlt_key.split('.')
        return '/'.join(['dlt'] + parts[:-1])

    @property
    def name(self) -> str:
        return "ResilientVault"
```

**Testing Custom Providers:**

```python
import pytest
from dlt.common.configuration import providers

def test_custom_vault_provider():
    """Test custom vault provider integration."""

    # Create mock vault provider
    class MockVaultProvider(VaultDocProvider):
        def __init__(self):
            super().__init__()
            self.secrets = {
                "sources.test_source.api_key": "test_key_value"
            }

        def get_value(self, key: str, *args, **kwargs):
            return self.secrets.get(key)

        @property
        def name(self) -> str:
            return "MockVault"

    # Register provider
    providers.register_provider(MockVaultProvider())

    # Test secret resolution
    @dlt.source
    def test_source(api_key: str = dlt.secrets.value):
        assert api_key == "test_key_value"

    # Source should resolve secret from mock vault
    source = test_source()
```

**Best Practices:**

- Implement error handling and retries for network issues
- Use caching for frequently accessed secrets
- Log secret access (without logging values) for audit trails
- Implement proper lifecycle management (connection pooling, cleanup)
- Test provider independently before pipeline integration
- Document secret naming conventions for your vault
- Use environment variables for vault connection details

#### Best Practices

**Security:**

*Basic Security:*
- Never commit `.dlt/secrets.toml` to version control
- Add `.dlt/secrets.toml` to `.gitignore`
- Use environment variables for CI/CD pipelines
- Use least-privilege principle for service accounts

*Secret Rotation:*
- Rotate credentials regularly (recommended: every 90 days)
- Implement automated rotation for long-lived secrets
- Use short-lived tokens when API supports them
- Version secrets in cloud secret managers
- Test new credentials before deactivating old ones
- Document rotation procedures for team

*Audit Logging:*
- Log credential access attempts (without logging values)
- Track which pipelines use which credentials
- Monitor for unauthorized access patterns
- Integrate with SIEM systems for production
- Log credential refresh events

Example audit logging:
```python
import logging

logger = logging.getLogger(__name__)

@dlt.source
def audited_source(api_key: str = dlt.secrets.value):
    # Log credential usage (never log the actual value)
    logger.info(
        "Credential accessed",
        extra={
            "source": "audited_source",
            "credential_type": "api_key",
            "credential_present": bool(api_key),
            "credential_length": len(api_key) if api_key else 0
        }
    )

    @dlt.resource
    def data():
        yield fetch_data(api_key)

    return data()
```

*Principle of Least Privilege:*
- Grant minimal permissions needed for pipeline operation
- Separate credentials for read vs. write operations
- Use dedicated service accounts per pipeline
- Scope API tokens to specific resources when possible
- Review and minimize OAuth scopes regularly

Scope restriction examples:
```python
# ✓ GOOD: Minimal scope
[sources.github_source.credentials]
scopes = ["repo:status", "public_repo"]  # Read-only

# ✗ BAD: Excessive scope
[sources.github_source.credentials]
scopes = ["repo", "admin:org", "delete_repo"]  # Too permissive
```

*Secure Storage:*
- Use encrypted filesystems for local `.dlt` directories
- Implement secrets encryption at rest in custom vaults
- Use hardware security modules (HSMs) for high-security requirements
- Avoid storing secrets in environment variables for long-running processes
- Clear secrets from memory after use in sensitive contexts

**Organization:**
- Keep non-sensitive config in `.dlt/config.toml` (version controlled)
- Keep secrets in `.dlt/secrets.toml` (not version controlled)
- Use consistent naming: `sources.<source_name>.<credential_key>`
- Document required secrets in README

**Development vs Production:**
- Local: Use `.dlt/secrets.toml` for convenience
- CI/CD: Use environment variables
- Cloud: Use cloud secret managers (Google Secret Manager, AWS Secrets Manager)
- Production: Consider `dlt deploy` for automated credential management

**Code Patterns:**
- Prefer `dlt.secrets.value` with decorators (automatic injection)
- Use type hints with credential classes for validation
- Handle missing secrets gracefully with try/except
- Use Union types for flexible authentication options

#### Testing Patterns

dlt provides mechanisms to test pipelines with mock credentials without requiring actual secrets.

**Mock Credentials with Programmatic Configuration:**

```python
import pytest
import dlt

def test_source_with_mock_credentials():
    """Test source using programmatically set mock credentials."""

    # Set mock credentials for testing
    dlt.secrets.set("sources.test_api.api_key", "test_key_12345")
    dlt.config.set("sources.test_api.base_url", "http://localhost:8000")

    @dlt.source
    def test_api_source(
        api_key: str = dlt.secrets.value,
        base_url: str = dlt.config.value
    ):
        # Will receive mock values
        assert api_key == "test_key_12345"
        assert base_url == "http://localhost:8000"

        @dlt.resource
        def data():
            yield [{"id": 1, "value": "test"}]

        return data()

    # Run source with mock credentials
    pipeline = dlt.pipeline(
        pipeline_name="test_pipeline",
        destination="duckdb",
        dataset_name="test_data",
        dev_mode=True
    )

    result = pipeline.run(test_api_source())
    assert result.has_failed_jobs == False
```

**Test-Specific secrets.toml:**

```python
import os
import pytest
import tempfile

@pytest.fixture
def test_secrets_dir():
    """Create temporary directory with test secrets."""
    with tempfile.TemporaryDirectory() as tmpdir:
        # Create test secrets file
        secrets_dir = os.path.join(tmpdir, ".dlt")
        os.makedirs(secrets_dir)

        with open(os.path.join(secrets_dir, "secrets.toml"), "w") as f:
            f.write("""
[sources.my_api]
api_key = "test_api_key_value"

[destination.duckdb]
credentials = "duckdb:///:memory:"
""")

        # Set as dlt secrets directory
        old_cwd = os.getcwd()
        os.chdir(tmpdir)

        yield tmpdir

        # Restore
        os.chdir(old_cwd)

def test_with_test_secrets(test_secrets_dir):
    """Test using test secrets directory."""

    @dlt.source
    def my_api(api_key: str = dlt.secrets.value):
        @dlt.resource
        def data():
            # Use test API key
            assert api_key == "test_api_key_value"
            yield [{"id": 1}]

        return data()

    pipeline = dlt.pipeline(
        pipeline_name="test",
        destination="duckdb",
        dataset_name="test",
        dev_mode=True
    )

    pipeline.run(my_api())
```

**Bypassing Validation in Test Mode:**

```python
from dlt.sources.credentials import ConnectionStringCredentials

def test_source_with_invalid_credentials():
    """Test source behavior with invalid credentials."""

    # Use minimal/mock credentials that wouldn't work in production
    mock_creds = ConnectionStringCredentials()
    mock_creds.database = "test_db"
    mock_creds.username = "test_user"
    mock_creds.password = "test_pass"
    mock_creds.host = "localhost"

    dlt.secrets.set("sources.db_source.credentials", mock_creds)

    @dlt.source
    def db_source(credentials: ConnectionStringCredentials = dlt.secrets.value):
        # Test credential parsing without actual connection
        assert credentials.database == "test_db"
        # Don't actually connect in test

        @dlt.resource
        def data():
            # Mock data instead of real DB query
            yield [{"id": 1, "name": "test"}]

        return data()

    pipeline = dlt.pipeline(
        pipeline_name="test",
        destination="duckdb",
        dev_mode=True
    )

    pipeline.run(db_source())
```

**Mocking External Services:**

```python
import pytest
from unittest.mock import patch, MagicMock

@patch('requests.get')
def test_api_source_with_mocked_requests(mock_get):
    """Test API source with mocked HTTP requests."""

    # Setup mock response
    mock_response = MagicMock()
    mock_response.json.return_value = {"data": [{"id": 1, "value": "test"}]}
    mock_response.status_code = 200
    mock_get.return_value = mock_response

    # Set mock credentials
    dlt.secrets.set("sources.api_source.api_key", "mock_key")

    @dlt.source
    def api_source(api_key: str = dlt.secrets.value):
        import requests

        @dlt.resource
        def data():
            response = requests.get(
                "https://api.example.com/data",
                headers={"Authorization": f"Bearer {api_key}"}
            )
            yield response.json()["data"]

        return data()

    pipeline = dlt.pipeline(
        pipeline_name="test",
        destination="duckdb",
        dev_mode=True
    )

    result = pipeline.run(api_source())

    # Verify mock was called with correct auth
    mock_get.assert_called_once()
    call_args = mock_get.call_args
    assert call_args[1]["headers"]["Authorization"] == "Bearer mock_key"
```

**Testing Credential Validation:**

```python
import pytest
from dlt.common.configuration.exceptions import ConfigFieldMissingException

def test_missing_credentials_raises_error():
    """Test that missing credentials raise appropriate error."""

    @dlt.source
    def strict_source(api_key: str = dlt.secrets.value):
        @dlt.resource
        def data():
            yield [{"id": 1}]

        return data()

    # Don't set any credentials
    # Should raise ConfigFieldMissingException

    with pytest.raises(ConfigFieldMissingException) as exc_info:
        source = strict_source()

    assert "api_key" in str(exc_info.value)
```

**Test Fixtures for Common Credentials:**

```python
import pytest

@pytest.fixture
def mock_api_credentials():
    """Fixture providing mock API credentials."""
    dlt.secrets.set("sources.test_api.api_key", "test_key_12345")
    dlt.secrets.set("sources.test_api.api_secret", "test_secret_67890")
    yield
    # Cleanup
    dlt.secrets.set("sources.test_api.api_key", None)
    dlt.secrets.set("sources.test_api.api_secret", None)

@pytest.fixture
def mock_database_credentials():
    """Fixture providing mock database credentials."""
    from dlt.sources.credentials import ConnectionStringCredentials

    creds = ConnectionStringCredentials(
        database="test_db",
        username="test_user",
        password="test_pass",
        host="localhost",
        port=5432
    )

    dlt.secrets.set("destination.postgres.credentials", creds)
    yield creds
    dlt.secrets.set("destination.postgres.credentials", None)

def test_with_fixtures(mock_api_credentials, mock_database_credentials):
    """Test using credential fixtures."""

    @dlt.source
    def my_source(api_key: str = dlt.secrets.value):
        @dlt.resource
        def data():
            yield [{"id": 1}]

        return data()

    pipeline = dlt.pipeline(
        pipeline_name="test",
        destination="postgres",
        dev_mode=True
    )

    pipeline.run(my_source())
```

#### Multi-Credential Patterns & Hierarchies

Manage multiple sources with different credentials in a single pipeline, with hierarchical organization and credential sharing strategies.

**Multiple Sources with Different Credentials:**

```python
@dlt.source
def github_source(access_token: str = dlt.secrets.value):
    @dlt.resource
    def repos():
        headers = {"Authorization": f"Bearer {access_token}"}
        # Fetch GitHub repos
        yield fetch_github_repos(headers)

    return repos()

@dlt.source
def stripe_source(api_key: str = dlt.secrets.value):
    @dlt.resource
    def customers():
        headers = {"Authorization": f"Bearer {api_key}"}
        # Fetch Stripe customers
        yield fetch_stripe_customers(headers)

    return customers()

# Configure different credentials per source
# .dlt/secrets.toml:
# [sources.github_source]
# access_token = "ghp_github_token"
#
# [sources.stripe_source]
# api_key = "sk_stripe_key"

# Run both sources in one pipeline
pipeline = dlt.pipeline(
    pipeline_name="multi_source_pipeline",
    destination="bigquery",
    dataset_name="combined_data"
)

# Each source gets its own credentials
pipeline.run([
    github_source(),  # Uses sources.github_source credentials
    stripe_source(),  # Uses sources.stripe_source credentials
])
```

**Hierarchical Credential Organization:**

```toml
# .dlt/secrets.toml - Hierarchical structure

# Root-level shared credentials (fallback)
[sources]
default_api_key = "fallback_key_for_all"

# API-specific credentials
[sources.api_v1]
base_url = "https://api-v1.example.com"
api_key = "v1_specific_key"

[sources.api_v2]
base_url = "https://api-v2.example.com"
api_key = "v2_specific_key"

# Environment-specific credentials
[sources.production]
api_key = "prod_key"
database_url = "postgresql://prod-db:5432/data"

[sources.staging]
api_key = "staging_key"
database_url = "postgresql://staging-db:5432/data"
```

**Credential Override Patterns at Resource Level:**

```python
@dlt.source
def multi_account_source(
    primary_api_key: str = dlt.secrets.value,
    secondary_api_key: str = dlt.secrets.value
):
    """Source accessing multiple accounts with different credentials."""

    @dlt.resource(name="primary_account_data")
    def primary_data():
        # Use primary account credentials
        headers = {"Authorization": f"Bearer {primary_api_key}"}
        yield fetch_data("https://api.example.com/primary", headers)

    @dlt.resource(name="secondary_account_data")
    def secondary_data():
        # Use secondary account credentials
        headers = {"Authorization": f"Bearer {secondary_api_key}"}
        yield fetch_data("https://api.example.com/secondary", headers)

    return primary_data(), secondary_data()

# .dlt/secrets.toml:
# [sources.multi_account_source]
# primary_api_key = "primary_account_key"
# secondary_api_key = "secondary_account_key"
```

**Sharing Credentials Across Pipelines:**

```toml
# .dlt/secrets.toml - Shared credentials configuration

# Shared credentials at root level
[shared]
github_token = "ghp_shared_token_for_all_pipelines"
database_password = "shared_db_password"

# Pipeline-specific overrides
[sources.pipeline_a]
github_token = "ghp_pipeline_a_specific_token"  # Override shared

[sources.pipeline_b]
# Uses shared.github_token (no override)
```

```python
@dlt.source
def pipeline_a(github_token: str = dlt.secrets.value):
    # Uses sources.pipeline_a.github_token
    pass

@dlt.source
def pipeline_b(github_token: str = dlt.secrets.value):
    # Falls back to shared.github_token
    pass
```

**Multi-Tenant Credential Management:**

```python
def run_pipeline_for_tenant(tenant_id: str):
    """Run pipeline with tenant-specific credentials."""

    # Load tenant-specific credentials from vault/database
    tenant_creds = load_tenant_credentials(tenant_id)

    # Set programmatically
    dlt.secrets.set(f"sources.tenant_{tenant_id}.api_key", tenant_creds["api_key"])
    dlt.secrets.set(f"sources.tenant_{tenant_id}.api_secret", tenant_creds["api_secret"])

    @dlt.source(name=f"tenant_{tenant_id}")
    def tenant_source(
        api_key: str = dlt.secrets.value,
        api_secret: str = dlt.secrets.value
    ):
        @dlt.resource
        def tenant_data():
            # Fetch data for specific tenant
            yield fetch_tenant_data(tenant_id, api_key, api_secret)

        return tenant_data()

    # Run pipeline for tenant
    pipeline = dlt.pipeline(
        pipeline_name=f"tenant_{tenant_id}_pipeline",
        destination="bigquery",
        dataset_name=f"tenant_{tenant_id}_data"
    )

    pipeline.run(tenant_source())

# Process multiple tenants
for tenant in ["tenant_a", "tenant_b", "tenant_c"]:
    run_pipeline_for_tenant(tenant)
```

**Credential Isolation Best Practices:**

```python
# ✓ GOOD: Isolated credentials per source
@dlt.source
def source_a(api_key_a: str = dlt.secrets.value):
    pass

@dlt.source
def source_b(api_key_b: str = dlt.secrets.value):
    pass

# .dlt/secrets.toml:
# [sources.source_a]
# api_key_a = "key_for_a"
#
# [sources.source_b]
# api_key_b = "key_for_b"

# ✗ BAD: Shared credentials across unrelated sources
@dlt.source
def source_a(shared_api_key: str = dlt.secrets.value):
    pass

@dlt.source
def source_b(shared_api_key: str = dlt.secrets.value):
    pass

# Avoid: Both sources use same credential
# Can cause issues if one source needs different key
```

#### Complete Example

```python
import dlt
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.auth import BearerTokenAuth
from dlt.sources.helpers.rest_client.paginators import OffsetPaginator

@dlt.source
def api_source(
    api_token: str = dlt.secrets.value,
    base_url: str = dlt.config.value,
    items_per_page: int = dlt.config.value
):
    """
    Expects configuration:

    # .dlt/config.toml
    [sources.api_source]
    base_url = "https://api.example.com"
    items_per_page = 100

    # .dlt/secrets.toml
    [sources.api_source]
    api_token = "your_secret_token"

    # OR environment variable:
    export SOURCES__API_SOURCE__API_TOKEN="your_secret_token"
    """

    @dlt.resource(name="users", write_disposition="merge", primary_key="id")
    def get_users():
        client = RESTClient(
            base_url=base_url,
            auth=BearerTokenAuth(token=api_token),
            paginator=OffsetPaginator(limit=items_per_page)
        )

        for page in client.paginate("/users"):
            yield page

    @dlt.resource(name="orders", write_disposition="append")
    def get_orders():
        client = RESTClient(
            base_url=base_url,
            auth=BearerTokenAuth(token=api_token)
        )

        for page in client.paginate("/orders"):
            yield page

    return get_users(), get_orders()

# Use source
pipeline = dlt.pipeline(
    pipeline_name="api_pipeline",
    destination="duckdb",
    dataset_name="api_data"
)

# Credentials automatically injected from secrets
pipeline.run(api_source())
```

## OSS Deployment

**Installation:**
```bash
pip install dlt
# Virtual environment recommended
```

**Deployment Options:**
- **Airflow**: Standard DAG deployment
- **Serverless**: Google Cloud Functions, AWS Lambda
- **GitHub Actions**: CI/CD pipeline execution
- **Custom cloud environments**: Any Python runtime

**How dlt Works (Three-Phase Processing):**

1. **Extract Phase**: Pulls data from sources into load package on disk with unique ID. Users can "supply schema hints to define data types" and control extraction through incremental cursors, parallelization, filtering.

2. **Normalize Phase**: dlt "inspects and normalizes your data and computes a schema corresponding to the input data." Automatically unnests nested structures into child tables and evolves schemas for new columns or type mismatches.

3. **Load Phase**: Executes schema migrations and loads data in parallelized "load jobs" into destination. dlt maintains internal tables tracking schema information, load package history, and state data—enabling incremental state restoration across machines.

---

## Next Steps

**This was a context priming step.** You now have reference information about dlt.

**To build a dlt pipeline, follow the structured workflow:**

1. **Start with research**: Use `00_prime/001_research_api.md` to gather API specifications
2. **Create specification**: Use `01_plan/011_plan_dlt_rest_client.md` with the research document
3. **Optional complexity**: Add appendices (012-015) only if needed for complex scenarios
4. **Implement**: Use `02_implement/02_implement.md` with the specification

**Remember:** Don't jump directly to implementation. Follow the prompts sequentially to ensure comprehensive planning.
