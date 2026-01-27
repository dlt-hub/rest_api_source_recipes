# API Development References

Centralized reference links for dlt pipeline development.

## dlt Documentation

### Core Concepts
- **dlt Overview**: https://dlthub.com/docs/intro
- **dlt Resources**: https://dlthub.com/docs/general-usage/resource
- **dlt Sources**: https://dlthub.com/docs/general-usage/source
- **dlt Pipeline**: https://dlthub.com/docs/general-usage/pipeline

### HTTP & REST API
- **RESTClient**: https://dlthub.com/docs/general-usage/http/rest-client
- **REST API Source**: https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api
- **HTTP Overview**: https://dlthub.com/docs/general-usage/http/overview
- **Custom HTTP Client**: https://dlthub.com/docs/general-usage/http/overview#custom-client

### Authentication
- **Credentials Guide**: https://dlthub.com/docs/general-usage/credentials
- **Complex Credentials**: https://dlthub.com/docs/general-usage/credentials/complex_types
- **OAuth2**: https://dlthub.com/docs/general-usage/credentials/complex_types#oauth2

### Pagination
- **Paginator Reference**: https://dlthub.com/docs/api_reference/dlt/sources/helpers/rest_client/paginators
- **Custom Paginators**: https://dlthub.com/docs/general-usage/http/rest-client#implementing-a-custom-paginator

### Incremental Loading
- **Incremental Loading Guide**: https://dlthub.com/docs/general-usage/incremental-loading
- **Advanced Incremental**: https://dlthub.com/docs/general-usage/incremental-loading#advanced-usage
- **Write Dispositions**: https://dlthub.com/docs/general-usage/resource#write-disposition

### Configuration
- **Config & Secrets**: https://dlthub.com/docs/general-usage/credentials
- **Config Spec**: https://dlthub.com/docs/general-usage/credentials/setup
- **Environment Variables**: https://dlthub.com/docs/general-usage/credentials/setup#environment-variables

### Testing
- **Testing Guide**: https://dlthub.com/docs/general-usage/test
- **Fixtures**: https://dlthub.com/docs/general-usage/test#fixtures

## Common REST API Standards

### SDMX (Statistical Data and Metadata eXchange)
- **SDMX 3.0 Specification**: https://sdmx.org/
- **SDMX REST API**: https://github.com/sdmx-twg/sdmx-rest

### OpenAPI / Swagger
- **OpenAPI Specification**: https://swagger.io/specification/
- **OpenAPI 3.0 Docs**: https://github.com/OAI/OpenAPI-Specification

### JSON:API
- **JSON:API Specification**: https://jsonapi.org/
- **JSON:API Examples**: https://jsonapi.org/examples/

## HTTP Standards

### Status Codes
- **HTTP Status Code Registry**: https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
- **MDN HTTP Status**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status

### Authentication
- **OAuth 2.0 RFC**: https://datatracker.ietf.org/doc/html/rfc6749
- **Bearer Token RFC**: https://datatracker.ietf.org/doc/html/rfc6750
- **HTTP Authentication RFC**: https://datatracker.ietf.org/doc/html/rfc7235

### Headers
- **HTTP Headers**: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers
- **Rate Limiting Headers**: https://datatracker.ietf.org/doc/html/draft-polli-ratelimit-headers

## Python Libraries

### Retry & HTTP
- **urllib3 Retry**: https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry
- **requests**: https://requests.readthedocs.io/
- **httpx**: https://www.python-httpx.org/

### Testing
- **pytest**: https://docs.pytest.org/
- **pytest-mock**: https://pytest-mock.readthedocs.io/
- **responses**: https://github.com/getsentry/responses

## Usage

Reference this file in specification prompts instead of duplicating links:

```markdown
## Reference

See `00_prime/002_api_references.md` for comprehensive dlt and API development references.

API-Specific:
- {API Name} Documentation: {url}
- {API Name} Authentication: {url}
```

## Next Steps

**This is a reference document** - refer back to it when creating specifications or during implementation.

These links are used throughout the workflow:
- Research phase (001): Reference dlt documentation for understanding patterns
- Planning phase (011-015): Reference specific documentation for configuration details
- Implementation phase (02): Reference when implementing specific features

**No action needed** - this file is used by other prompts automatically.
