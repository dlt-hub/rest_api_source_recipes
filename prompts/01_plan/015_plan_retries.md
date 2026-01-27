---
description: Generate retry appendix for complex error handling and custom retry logic
argument-hint: <api_name> <research_file_path> [--output <file_path>]
---

# plan_retries

Generate focused retry appendix for APIs with complex error handling or custom retry requirements.

**Purpose:** Use ONLY when retry/error handling is complex or requires custom logic beyond dlt defaults. For standard retry patterns (5xx, 429 with exponential backoff), use `011_plan_dlt_rest_client.md` which includes comprehensive retry coverage.

**When to use this prompt:**
- Custom retry logic based on response content (not just status codes)
- Complex rate limiting with multiple tiers
- Endpoint-specific timeout requirements vary significantly
- Custom backoff algorithms
- API-specific error patterns requiring special handling

**When NOT to use this prompt:**
- Standard retry on 5xx, 429 (covered in 011)
- Simple exponential backoff (covered in 011)
- Standard timeout configuration (covered in 011)

**dlt Retry Prioritization:**
1. **rest_api_source configuration** - Simplest (covered in 011)
2. **RESTClient with custom session** - Endpoint-specific (covered in 011)
3. **dlt.requests.Client** - Custom logic (THIS appendix for complex cases)

## Arguments

**Required:**
- `api_name`: API name (e.g., `github`, `stripe`, `pipedrive`)
- `research_file_path`: Path to API research file or scaffold code

**Optional:**
- `--output <file_path>`: Save to (default: `specs/YYYY-MM-DD_015_spec_retries_{api_name}.md`)
  - Pattern: `YYYY-MM-DD_[number]_spec_[topic_name]_[api_name].md`
  - Example: `2026-01-26_015_spec_retries_imf.md`

## Quick Reference

- **Main Spec:** See `011_plan_dlt_rest_client.md` for standard retry patterns
- **dlt Patterns:** See `00_prime/000_dlt_context.md` for retry mechanisms and configuration
- **Prioritization:** rest_api_source → RESTClient → dlt.requests.Client

## Workflow

See `_workflow_template.md` for standard planning workflow.

### Analysis Focus

1. **Confirm complexity justifies appendix:**
   - Is retry logic based on response content (not status)? → YES, continue
   - Is rate limiting multi-tiered or complex? → YES, continue
   - Is custom backoff algorithm needed? → YES, continue
   - Are standard retry patterns sufficient? → NO, use 011 instead

2. **Document custom retry requirements:**
   - Custom retry conditions (beyond standard status codes)
   - Response content-based retry decisions
   - Multi-tier rate limit handling
   - Custom backoff algorithms
   - API-specific error recovery patterns

3. **Identify edge cases and special handling:**
   - Partial success scenarios
   - Quota management across requests
   - Circuit breaker patterns
   - Retry budget management

## Retry Appendix Template

**Output Format:** Append to main specification from `011_plan_dlt_rest_client.md`

```markdown
# APPENDIX: {API Name} - Complex Retry & Error Handling Details

**Note:** This appendix provides detailed guidance for complex retry scenarios. For standard retry patterns, refer to the Retry & Error Handling section in the main specification.

## Retry Complexity

**Why this requires detailed appendix:**
{Explain what makes retry/error handling complex: custom logic, response-based decisions, multi-tier rate limits}

**Complexity Type:**
- [ ] Custom retry based on response content
- [ ] Multi-tier rate limiting
- [ ] Custom backoff algorithm
- [ ] Circuit breaker pattern
- [ ] Retry budget management

## Custom Retry Logic

### Response Content-Based Retry

**Scenario:**
{Describe when to retry based on response body, not just status code}

**Custom Retry Implementation:**
```python
import dlt
from dlt.sources.helpers.requests import Client

class CustomRetryClient(Client):
    def _should_retry(self, response):
        """Custom retry decision based on response content"""
        # Check status code first
        if response.status_code in [429, 500, 502, 503, 504]:
            return True

        # Custom logic: check response body
        if response.status_code == 200:
            data = response.json()
            if data.get("status") == "processing":
                return True  # Retry while still processing
            if data.get("error_code") == "TEMPORARY_ERROR":
                return True  # Retry on temporary errors

        return False
```

### Multi-Tier Rate Limiting

**Rate Limit Tiers:**
| Tier | Limit | Backoff Strategy |
|------|-------|------------------|
| Tier 1 (Warning) | {X} req/{time} | {strategy} |
| Tier 2 (Throttle) | {X} req/{time} | {strategy} |
| Tier 3 (Block) | {X} req/{time} | {strategy} |

**Implementation:**
```python
def custom_backoff(response):
    """Calculate backoff based on rate limit tier"""
    tier = response.headers.get("X-RateLimit-Tier")

    if tier == "warning":
        return 1.0  # Short backoff
    elif tier == "throttle":
        return 60.0  # 1 minute
    elif tier == "block":
        return 3600.0  # 1 hour

    # Default exponential backoff
    return 2 ** attempt_number
```

### Custom Backoff Algorithm

**Algorithm:**
{Describe custom backoff strategy}

**Implementation:**
```python
def custom_backoff_strategy(attempt, response):
    """
    Custom backoff calculation.

    Args:
        attempt: Retry attempt number (1-indexed)
        response: HTTP response object

    Returns:
        float: Seconds to wait before retry
    """
    # Custom logic here
    base_delay = {calculation}
    jitter = {calculation}
    return base_delay + jitter
```

### Circuit Breaker Pattern

**When to use:**
{Describe scenario requiring circuit breaker}

**Configuration:**
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open

    def call(self, func):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func()
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise
```

## Additional Test Cases

Beyond standard retry tests in main spec, add these complex scenario tests:

```python
def test_retry_based_on_response_content():
    """Test custom retry logic examines response body"""

def test_multi_tier_rate_limit_backoff():
    """Test different backoff per rate limit tier"""

def test_custom_backoff_algorithm():
    """Test custom backoff calculation"""

def test_circuit_breaker_opens_on_failures():
    """Test circuit breaker opens after threshold"""

def test_circuit_breaker_half_open_recovery():
    """Test circuit breaker recovery to half-open state"""
```

## Endpoint-Specific Complexity

### {Complex Endpoint Name}

**Why Complex:**
{Explain endpoint-specific retry complexity}

**Custom Configuration:**
```python
"{endpoint}": {
    "path": "{path}",
    "timeout": {"total": {seconds}},
    "retry_config": {
        "max_attempts": {number},
        "custom_logic": custom_retry_function,
        "backoff_strategy": custom_backoff_function
    }
}
```

## Edge Cases & Troubleshooting

**{Edge Case 1: Partial Success}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to handle}

**{Edge Case 2: Quota Exhaustion}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {how to manage}

**{Edge Case 3: Cascading Failures}:**
- Symptom: {what happens}
- Cause: {why}
- Solution: {circuit breaker or other pattern}

## Monitoring Custom Retry Logic

**Metrics to Track:**
- Custom retry trigger count
- Circuit breaker state changes
- Per-tier rate limit hits
- Custom backoff durations
- Success rate after custom retry

**Logging:**
```python
import logging
logger = logging.getLogger(__name__)

def log_custom_retry(attempt, reason, backoff):
    logger.info(
        f"Custom retry triggered",
        extra={
            "attempt": attempt,
            "reason": reason,
            "backoff_seconds": backoff
        }
    )
```

## Reference

- API Error Handling Documentation: {url}
- dlt Custom HTTP Client: https://dlthub.com/docs/general-usage/http/overview#custom-client
```

## Validation Checklist

Before generating appendix:
- [ ] Confirmed retry complexity justifies separate appendix
- [ ] Custom retry logic documented with implementation
- [ ] Multi-tier rate limiting strategy specified (if applicable)
- [ ] Custom backoff algorithm detailed (if applicable)
- [ ] Circuit breaker pattern included (if needed)
- [ ] Additional test cases beyond main spec identified
- [ ] Edge cases and troubleshooting documented
- [ ] Monitoring strategy for custom logic included
- [ ] References to main spec retry section included

## Output Message

```
Retry & error handling appendix complete: {api_name}
Appended to: specs/YYYY-MM-DD_011_spec_dlt_rest_client_{api_name}.md

Complexity: {Custom logic / Multi-tier rate limit / Circuit breaker}
Custom patterns: {number}
Additional test cases: {number}

This appendix extends the Retry & Error Handling section in the main specification with:
- Custom retry logic implementation (if applicable)
- Multi-tier rate limiting strategy (if applicable)
- Custom backoff algorithm (if applicable)
- Circuit breaker pattern (if needed)
- Additional test scenarios for complex cases
- Monitoring strategy for custom logic
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
| **Incremental** | Compound cursors, interdependencies, or backfill strategies? | → Use `01_plan/014_plan_incremental_load.md` |
| | Simple timestamp or ID cursor? | → Skip (already covered) |

**If no other appendices needed: Proceed to implementation**

Use prompt: `02_implement/02_implement.md` with the updated spec file.
