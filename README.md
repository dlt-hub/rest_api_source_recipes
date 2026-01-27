# Prompts Structure

This directory contains prompts for developing dlt pipelines using a step-by-step approach.

## Why Step-by-Step?

The prompts follow a sequential structure with the following goals:

1. **Learning**: Each step explains how dlt works, progressing from basic concepts to advanced patterns.
2. **Visibility**: The process is broken down so planning and implementation phases can be seen separately.
3. **Sequencing**: Dependencies are tackled in order, which increases probability of success.

## Development Approaches

### Spec-Driven Development
Prompts define specifications before implementation:
- Requirements are documented upfront
- Implementation has acceptance criteria
- Tests validate behavior

### Test-Driven Development
Prompts follow test-driven development:
- Tests are written first
- Implementation follows test requirements
- For efficiency reasons (since this serves as a learning material) tests are kept minimal via the CLAUDE.md / agents.md / copilot_instructions.md - you can adapt your testing preferences there 
- Changes are validated through tests

## Context Management via Progressive Disclosure

Progressive disclosure presents information gradually as needed. In these prompts:

- Early stages focus on architecture and planning
- Later stages cover implementation details
- Each prompt builds on previous context without repeating it
- Complexity is introduced incrementally

## dlt Best Practices

Prompts follow dlt best practices:
- Use built-in patterns like `dlt.sources.rest_api()` and `RESTClient`
- Use dlt's retry, rate limiting, and error handling
- Follow dlt's resource and source patterns
- Use dlt's incremental loading capabilities

## Quality Assurance

The prompts have been tested and analyzed to avoid duplication in specs and implementation.

## Directory Structure

- `00_prime/`: Initial research and API exploration
- `01_plan/`: Planning prompts for specific features
- `02_implement/`: Implementation prompts (if separate from planning)

## Execution Flow

### Standard Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: Research                                               │
└─────────────────────────────────────────────────────────────────┘
    │
    ├─> 001_research_api.md
    │   Output: research/YYYY-MM-DD_001_research_{api_name}.md
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 2: Main Specification                                     │
└─────────────────────────────────────────────────────────────────┘
    │
    ├─> 011_plan_dlt_rest_client.md
    │   Input: research file from Phase 1
    │   Output: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md
    │
    │   This comprehensive spec includes:
    │   - Authentication configuration
    │   - Pagination strategies per endpoint
    │   - Incremental loading strategies
    │   - Retry and error handling
    │   - Complete test structure
    │   - Task breakdown (9 phases)
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 3: Optional Appendices (ONLY if needed)                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ├─> 012_plan_authentication.md (Optional)
    │   Use ONLY if: OAuth2 with refresh, multi-step auth, custom auth logic
    │   Output: Appends to main spec with detailed auth complexity
    │
    ├─> 013_plan_pagination.md (Optional)
    │   Use ONLY if: Mixed pagination strategies, custom pagination logic
    │   Output: Appends to main spec with per-endpoint pagination details
    │
    ├─> 014_plan_incremental_load.md (Optional)
    │   Use ONLY if: Compound cursors, resource interdependencies, backfill
    │   Output: Appends to main spec with complex state management
    │
    ├─> 015_plan_retries.md (Optional)
    │   Use ONLY if: Custom retry logic, multi-tier rate limits, circuit breaker
    │   Output: Appends to main spec with custom error handling
    │
    v
┌─────────────────────────────────────────────────────────────────┐
│ Phase 4: Implementation                                         │
└─────────────────────────────────────────────────────────────────┘
    │
    └─> Follow main spec (with optional appendices if generated)
        Implement using TDD approach defined in spec
```

### Decision Guide: When to Use Appendix Prompts

**Start with 011 (main spec)** - it covers:
- Standard authentication (API key, Bearer token, Basic auth)
- Standard pagination (single_page, offset, cursor, page_number)
- Standard incremental (timestamp/ID cursor with merge/append)
- Standard retry (5xx, 429 with exponential backoff)

**Use appendix prompts ONLY when:**

| Appendix | When to Use | When NOT to Use |
|----------|-------------|-----------------|
| **012 (auth)** | OAuth2 with refresh, multi-step flows, custom signatures | API key in header, Bearer token, Basic auth |
| **013 (pagination)** | Different strategies per endpoint, hybrid pagination, custom logic | Same strategy all endpoints, standard patterns |
| **014 (incremental)** | Compound cursors, interdependent resources, backfill strategies | Simple timestamp or ID cursor |
| **015 (retries)** | Response content-based retry, multi-tier limits, circuit breaker | Standard HTTP status retry, exponential backoff |

### File Output Structure

```
specs/
├── YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md    # Main spec (always)
│   ├── [APPENDIX sections from 012-015 if generated]    # Appended inline
│
research/
└── YYYY-MM-DD_001_research_{api_name}.md                # Research input
```

**Note:** Appendices are appended to the main spec file, not separate files. This creates a single comprehensive specification document.

## Example Usage

### Simple API (GitHub public endpoints)
```bash
# Step 1: Research
Use prompt: 001_research_api.md with GitHub API docs

# Step 2: Main spec (sufficient for standard patterns)
Use prompt: 011_plan_dlt_rest_client.md with research file

# Step 3: Implement
Follow the main spec - no appendices needed
```

### Complex API (Salesforce with OAuth2)
```bash
# Step 1: Research
Use prompt: 001_research_api.md with Salesforce API docs

# Step 2: Main spec
Use prompt: 011_plan_dlt_rest_client.md with research file

# Step 3: Auth appendix (OAuth2 with refresh is complex)
Use prompt: 012_plan_authentication.md
Output appended to main spec

# Step 4: Implement
Follow the main spec with auth appendix
```

Each prompt builds on the previous ones.