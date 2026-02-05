# dlt REST API Source Development Prompts

This directory contains prompts for developing dlt REST API pipelines using a streamlined, test-driven approach.

## Why Step-by-Step?

The prompts follow a sequential structure with the following goals:

1. **Learning**: Each step explains how dlt works, progressing from basic concepts to advanced patterns.
2. **Clarity**: The process is broken down into distinct phases: research & planning, implementation, and testing.
3. **Quality**: Test-driven development ensures each component works correctly before moving forward.

## Development Philosophy

### Test-Driven Development (TDD)
All prompts follow TDD methodology:
- Tests are written first
- Implementation follows test requirements
- Tests are kept minimal but focused on critical functionality
- Each component is validated before moving forward

### Spec-Driven Development
Requirements are documented before implementation:
- API research informs the specification
- Specifications include test structure and acceptance criteria
- Implementation follows the specification exactly

### Progressive Disclosure
Information is presented when needed:
- Phase 0 provides dlt context upfront
- Phase 1 combines research and planning
- Phase 2 focuses on implementation details
- Phase 3 handles end-to-end testing

## dlt Best Practices

All prompts follow dlt best practices:
- Use built-in patterns: `rest_api_source()` and `RESTClient`
- Leverage dlt's retry, rate limiting, and error handling
- Follow dlt's resource and source patterns
- Use dlt's incremental loading capabilities
- Never build custom API clients

## Directory Structure

```
prompts/
├── 00_prime/
│   └── 000_dlt_context.md          # dlt fundamentals and patterns
├── 01_plan/
│   └── 01_research_and_plan.md     # Research API + create spec
├── 02_implement/
│   └── 02_implement.md             # TDD implementation
└── 03_test/
    └── 03_test.md                  # Pipeline testing guide
```

## Execution Flow

### Standard Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 0: Prime (Optional)                                       │
│ Read dlt context before starting                                │
└─────────────────────────────────────────────────────────────────┘
    │
    └─> 00_prime/000_dlt_context.md
        Provides: dlt patterns, auth classes, paginators, decorators
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: Research & Plan                                        │
│ Research API + Generate specification                           │
└─────────────────────────────────────────────────────────────────┘
    │
    └─> 01_plan/01_research_and_plan.md
        Arguments: api_name (e.g., "arxiv", "github")
        Output: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

        This spec includes:
        - API research findings
        - Authentication configuration
        - Endpoint documentation
        - Pagination strategies
        - Incremental loading design
        - Complete test structure
        - TDD task breakdown
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 2: Implement                                              │
│ Follow spec using strict TDD                                    │
└─────────────────────────────────────────────────────────────────┘
    │
    └─> 02_implement/02_implement.md
        Arguments: spec_file_path
        Process: Write test → Run (fail) → Implement → Run (pass)
        Follows task list from spec sequentially
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 3: Test                                                   │
│ Run pipeline end-to-end                                         │
└─────────────────────────────────────────────────────────────────┘
    │
    └─> 03_test/03_test.md
        - Test with limited data first
        - Verify schema and data quality
        - Test incremental loading
        - Validate full pipeline execution
```

### File Output Structure

```
project/
├── specs/
│   └── YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md
├── sources/
│   └── {api_name}_source.py
├── tests/
│   ├── conftest.py
│   ├── test_auth.py (if auth required)
│   ├── test_resources.py
│   ├── test_incremental.py (if applicable)
│   ├── test_source.py
│   └── test_pipeline.py
├── .dlt/
│   ├── config.toml
│   └── secrets.toml.example
└── {api_name}_pipeline.py
```

## Usage Example

### Building a Pipeline for the GitHub API

**Step 0 (Optional): Review dlt context**
```bash
# Read: prompts/00_prime/000_dlt_context.md
# Learn about: RESTClient, auth classes, paginators, decorators
```

**Step 1: Research & Plan**
```bash
# Use prompt: prompts/01_plan/01_research_and_plan.md
# Provide: api_name="github"
# The prompt will:
#   1. Check for existing dltHub verified source
#   2. Research GitHub API documentation
#   3. Ask clarifying questions
#   4. Generate comprehensive specification
# Output: specs/2025-01-15_011_spec_dlt_rest_client_github.md
```

**Step 2: Implement**
```bash
# Use prompt: prompts/02_implement/02_implement.md
# Provide: spec_file_path="specs/2025-01-15_011_spec_dlt_rest_client_github.md"
# The prompt will:
#   1. Extract task list from spec
#   2. Follow TDD: test → fail → implement → pass
#   3. Report progress after each task
#   4. Ask user when spec is unclear
# Output: sources/, tests/, .dlt/, pipeline script
```

**Step 3: Test**
```bash
# Use guide: prompts/03_test/03_test.md
# Process:
#   1. Configure secrets in .dlt/secrets.toml
#   2. Test with limited data (add_limit(10))
#   3. Verify schema and data
#   4. Test incremental loading
#   5. Run full pipeline
```

### Complete Workflow Commands

```bash
# Phase 1: Research & Plan
# → Provide "github" as api_name to prompt 01_research_and_plan.md
# → Review and approve generated specification

# Phase 2: Implement with TDD
# → Provide spec path to prompt 02_implement.md
# → Watch as tests are written and implementation follows

# Phase 3: Test Pipeline
# → Follow guide in 03_test.md
pytest tests/ -v                    # Run all tests
python github_pipeline.py           # Execute pipeline
dlt pipeline github_pipeline show   # Inspect results
```

## Phase Details

### Phase 0: Prime (00_prime/000_dlt_context.md)

**Purpose:** Provide foundational dlt knowledge before starting development.

**Contains:**
- What is dlt and when to use it
- Decision matrix: `rest_api_source()` vs `RESTClient`
- Authentication patterns (API key, Bearer, Basic, OAuth2)
- Pagination classes and examples
- Resource and source decorators
- Secrets management
- Incremental loading patterns
- Pipeline creation and testing

**When to use:** Read before Phase 1 if unfamiliar with dlt, or reference during implementation.

### Phase 1: Research & Plan (01_plan/01_research_and_plan.md)

**Purpose:** Research API and generate complete specification with TDD structure.

**Input:**
- `api_name`: Name of the API (e.g., "arxiv", "github")
- Optional: `--spec-output`, `--destination`

**Process:**
1. Check for existing dltHub verified source
2. Research API documentation for:
   - Base URL, authentication, endpoints
   - Pagination, rate limits, incremental fields
   - Data structure and special considerations
3. Ask clarifying questions
4. Design test-first approach
5. Map API patterns to dlt components
6. Generate specification document

**Output:** `specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md`

**Specification includes:**
- API research findings
- Authentication configuration
- Endpoint documentation
- Resource implementations
- Test structure (conftest.py, test_auth.py, test_resources.py, etc.)
- TDD task breakdown (8 phases)
- Acceptance criteria

### Phase 2: Implement (02_implement/02_implement.md)

**Purpose:** Implement specification following strict TDD methodology.

**Input:**
- `spec_file_path`: Path to specification document

**Critical rules:**
1. Load spec and extract task list
2. TDD only: test first → verify fail → implement → verify pass
3. Follow sequential order, one task at a time
4. Ask user when spec is unclear or information is missing
5. No assumptions about credentials or requirements

**Process for each task:**
1. Write test (red)
2. Run test - should fail
3. Implement minimal code (green)
4. Run test - should pass
5. Report progress
6. Move to next task

**Output:**
- `sources/{api_name}_source.py`
- `tests/` directory with all test files
- `.dlt/config.toml` and `.dlt/secrets.toml.example`
- `{api_name}_pipeline.py`
- `README.md`

**Validation:**
- All spec tests implemented and passing
- All tasks checked off
- Used dlt built-ins (no custom API clients)
- Asked user when needed

### Phase 3: Test (03_test/03_test.md)

**Purpose:** Test the pipeline end-to-end with real API.

**Process:**
1. **Setup:** Create venv, install dependencies
2. **Configure:** Add secrets to `.dlt/secrets.toml`
3. **Limited test:** Run with `add_limit(10)` or `max_pages=1`
4. **Verify:** Check successful load
5. **Full test:** Remove limit and run full pipeline
6. **Inspect:**
   - View schema: `dlt pipeline <name> schema`
   - Query data: `dlt pipeline <name> show`
   - Check counts and data quality
7. **Incremental test:** Run twice, verify state and no duplicates
8. **Validate:** Complete checklist

**Common issues covered:**
- Authentication failures
- Schema changes
- Data quality issues
- Rate limiting

**Output:** Fully tested, production-ready pipeline

## Key Principles

### 1. Use dlt Built-ins
- Always use `rest_api_source()` or `RESTClient`
- Leverage dlt's auth classes: `BearerTokenAuth`, `APIKeyAuth`, `HttpBasicAuth`, `OAuth2ClientCredentials`
- Use built-in paginators: `SinglePagePaginator`, `PageNumberPaginator`, `OffsetPaginator`, `JSONResponsePaginator`, etc.
- Never build custom API clients

### 2. Test-Driven Development
- Write tests first, implementation second
- Tests should fail initially (red)
- Implement minimal code to pass (green)
- Validate each component before moving forward
- Keep tests minimal but focused on critical functionality

### 3. Progressive Complexity
- Start with simplest approach that works
- Only add complexity when needed
- Follow spec exactly, don't over-engineer
- Ask user when requirements are unclear

### 4. Security First
- Never read credentials from files
- Always use `.dlt/secrets.toml` or environment variables
- Keep `.dlt/secrets.toml` in `.gitignore`
- Provide `.dlt/secrets.toml.example` template

### 5. Incremental Development
- Test with limited data first (`add_limit(10)`)
- Verify small sample before full load
- Test incremental loading by running twice
- Validate data quality at each step

## Benefits of This Approach

1. **Clarity:** Each phase has a clear purpose and output
2. **Quality:** TDD ensures correctness at each step
3. **Learning:** Prompts teach dlt patterns progressively
4. **Efficiency:** Streamlined workflow reduces back-and-forth
5. **Reliability:** Specifications prevent misunderstandings
6. **Maintainability:** Well-tested code is easier to update

## Additional Resources

- [dlt Documentation](https://dlthub.com/docs)
- [REST API Source](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api)
- [RESTClient Guide](https://dlthub.com/docs/general-usage/http/rest-client)
- [dlt Verified Sources](https://dlthub.com/docs/dlt-ecosystem/verified-sources/) - Check here first before building custom

## Getting Help

If you encounter issues:
1. Review the relevant phase documentation
2. Check specification for clarity
3. Consult dlt documentation
4. Ask clarifying questions when spec is ambiguous

## Contributing

Improvements welcome:
- Clearer prompt instructions
- Better examples
- Additional patterns
- Bug fixes