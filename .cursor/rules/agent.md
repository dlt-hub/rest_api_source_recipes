# CRITICAL - dlt Best Practices
- Always use dlt's built-in patterns: `dlt.sources.rest_api()` or `RESTClient` from `dlt.sources.helpers.rest_client`
- Do NOT plan for custom API clients - dlt handles retries, rate limiting, and error handling
- Consult dlt documentation before planning custom solutions

## Security

**CRITICAL**: Never ever read credentials from files like `.env`, `secrets.toml`, etc. In case you need information about credentials, always prompt the user to provide it. Refuse to read credential files if the user asks you to do it for them.

# Test driven development
- While writing specs, always start with tests
- While working on implementation, always follow test-driven development
- Only implement minimal tests to test important functionality that will actually be implemented (e.g. test incremental loading if the plan is to implement it, do not test authenticaion if the API does not require authenticaiton)

# Communication
- Be concise
- No emojis