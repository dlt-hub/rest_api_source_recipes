---
description: Generate authentication appendix for complex auth scenarios
argument-hint: <api_name> --research <file_path> [--output <file_path>]
---

# plan_authentication

Generate focused authentication appendix for complex authentication scenarios beyond basic API key/Bearer token patterns.

**Purpose:** Use ONLY when authentication is complex (OAuth2, custom headers, multi-step auth, token refresh). For standard auth (API key, Bearer token), use `011_plan_dlt_rest_client.md` which includes comprehensive auth coverage.

**When to use this prompt:**
- OAuth2 with token refresh
- Multi-step authentication flows
- Custom authentication headers/logic
- Token exchange patterns
- API-specific auth complexities

**When NOT to use this prompt:**
- Simple API key in header (covered in 011)
- Bearer token authentication (covered in 011)
- Basic HTTP auth (covered in 011)

## Arguments

**Required:**
- `api_name`: API name (e.g., `github`, `stripe`, `hubspot`)
- `--research <file_path>`: Path to API research document or scaffold source code

**Optional:**
- `--output <file_path>`: Save to (default: `specs/YYYY-MM-DD_012_spec_authentication_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_012_spec_authentication_imf.md`

## Quick Reference

- **Main Spec:** See `011_plan_dlt_rest_client.md` for standard authentication patterns
- **dlt Patterns:** See `00_prime/000_dlt_context.md` for auth classes, configuration, secrets
- **Auth Classes:** `BearerTokenAuth`, `APIKeyAuth`, `HttpBasicAuth`, `OAuth2ClientCredentials`

## Workflow

See `_workflow_template.md` for standard planning workflow.

### Analysis Focus

1. **Confirm complexity justifies appendix:**
   - Is this OAuth2 with refresh? → YES, continue
   - Is this multi-step auth? → YES, continue
   - Is this custom auth logic? → YES, continue
   - Is this simple API key/Bearer token? → NO, use 011 instead

2. **Extract complex auth details:**
   - OAuth2: scopes, token endpoint, refresh mechanism
   - Multi-step: authentication sequence, dependencies
   - Custom: headers, signing, token generation logic
   - Token management: expiration, refresh, storage

3. **Document edge cases and special handling:**
   - Token refresh timing and strategy
   - Error recovery for auth failures
   - Credential rotation requirements
   - API-specific authentication quirks

## Authentication Appendix Template

**Output Format:** Append to main specification from `011_plan_dlt_rest_client.md`

```markdown
# APPENDIX: {API Name} - Complex Authentication Details

**Note:** This appendix provides detailed guidance for complex authentication. For standard auth patterns, refer to the Authentication section in the main specification.

## Authentication Complexity

**Why this requires detailed appendix:**
{Explain what makes this auth complex: OAuth2 refresh, multi-step, custom logic, etc.}

## Detailed Authentication Flow

{Describe step-by-step authentication process}

### OAuth2 Configuration (if applicable)

**Token Endpoint:** `{url}`
**Scopes:** `{scopes}`
**Grant Type:** `{client_credentials/authorization_code/etc}`

**Configuration:**
```toml
[sources.{api_name}_source]
client_id = "your_client_id"
client_secret = "your_client_secret"
{additional_params} = "values"
```

**Token Refresh Strategy:**
- Refresh trigger: {condition}
- Refresh mechanism: {how}
- Token storage: {where}

### Custom Authentication Logic (if applicable)

**Custom Headers Required:**
```python
{
    "X-Custom-Header": "value",
    "X-Signature": "computed_signature"
}
```

**Signature/Token Generation:**
{Describe how to generate tokens, signatures, or custom auth values}

## Additional Test Cases

Beyond standard auth tests in main spec, add these complex scenario tests:

```python
def test_token_refresh():
    """Test OAuth2 token refresh on expiration"""

def test_multi_step_auth_sequence():
    """Test multi-step authentication completes successfully"""

def test_custom_signature_generation():
    """Test custom auth signature computed correctly"""
```

## Implementation Guidance

**OAuth2 with Refresh:**
```python
from dlt.sources.helpers.rest_client.auth import OAuth2ClientCredentials

auth = OAuth2ClientCredentials(
    access_token_url="{token_url}",
    client_id=dlt.secrets.value,
    client_secret=dlt.secrets.value,
    {additional_params}
)
```

**Custom Auth Class:**
{If standard dlt auth classes don't work, provide guidance on custom implementation}

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

- API Auth Documentation: {url}
- dlt OAuth2: https://dlthub.com/docs/general-usage/credentials/complex_types#oauth2
```

## Validation Checklist

Before generating appendix:
- [ ] Confirmed auth complexity justifies separate appendix
- [ ] Complex auth flow documented step-by-step
- [ ] OAuth2 refresh strategy specified (if applicable)
- [ ] Custom auth logic detailed (if applicable)
- [ ] Additional test cases beyond main spec identified
- [ ] Edge cases and troubleshooting documented
- [ ] References to main spec authentication section included

## Output Message

```
Authentication appendix complete: {api_name}
Appended to: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

Complexity: {OAuth2 with refresh / Multi-step / Custom logic}
Additional test cases: {number}
Edge cases documented: {number}

This appendix extends the Authentication section in the main specification with:
- Detailed complex authentication flow
- OAuth2 token refresh strategy (if applicable)
- Custom authentication logic (if applicable)
- Additional test scenarios for complex cases
- Edge case handling and troubleshooting
```

## Next Steps

**Check if other appendices are needed:**

| Aspect | Condition | Action |
|--------|-----------|--------|
| **Pagination** | Different strategies per endpoint, or custom pagination logic? | → Use `01_plan/013_plan_pagination.md` |
| | Same strategy for all endpoints? | → Skip (already covered) |
| **Incremental** | Compound cursors, interdependencies, or backfill strategies? | → Use `01_plan/014_plan_incremental_load.md` |
| | Simple timestamp or ID cursor? | → Skip (already covered) |
| **Retries** | Response content-based retry, multi-tier limits, or circuit breaker? | → Use `01_plan/015_plan_retries.md` |
| | Standard retry patterns? | → Skip (already covered) |

**If no other appendices needed: Proceed to implementation**

Use prompt: `02_implement/02_implement.md` with the updated spec file.
