# 03-01 - Implement Exchange Rates HTTP Methods in Node struct

# Links
- [PRD](../../prd/vtcpd-test-suite/03-exchange-rates-testing.md)

# Description
Implement 6 new HTTP methods in pkg/testsuite/node.go for exchange rates operations. These methods will provide HTTP client functionality to interact with vtcpd-cli exchange rates endpoints, following existing Node struct patterns for error handling and response parsing.

# Requirements and DOD
- Add 6 new methods to Node struct in pkg/testsuite/node.go:
  - `SetExchangeRate(t *testing.T, equivalentFrom, equivalentTo, realRate string, minAmount, maxAmount *string)` - sets rate using real decimal format
  - `SetExchangeRateNative(t *testing.T, equivalentFrom, equivalentTo, value string, shift int16, minAmount, maxAmount *string)` - sets rate using native value+shift format
  - `GetExchangeRate(t *testing.T, equivalentFrom, equivalentTo string) *RateItem` - retrieves specific exchange rate
  - `ListExchangeRates(t *testing.T) *RatesListResponse` - lists all exchange rates
  - `DeleteExchangeRate(t *testing.T, equivalentFrom, equivalentTo string)` - deletes specific exchange rate
  - `ClearExchangeRates(t *testing.T)` - clears all exchange rates
- All methods must follow existing Node method patterns:
  - Accept `t *testing.T` parameter as first argument
  - Handle HTTP errors internally using `t.Fatalf()` - no error return values
  - Validate HTTP status codes within methods
  - Parse JSON responses and return appropriate Go structs
- Methods must interact with vtcpd-cli HTTP endpoints:
  - POST `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` for SetExchangeRate/SetExchangeRateNative
  - GET `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` for GetExchangeRate
  - GET `/api/v1/node/rates/` for ListExchangeRates
  - DELETE `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` for DeleteExchangeRate
  - DELETE `/api/v1/node/rates/` for ClearExchangeRates
- Response structures must use existing types from vtcpd-cli:
  - RateItem struct for individual rate data
  - RatesListResponse struct for list operations
  - Handle empty data responses for set/delete/clear operations
- HTTP request construction must follow existing patterns in node.go
- JSON parsing must handle all fields defined in RateItem struct
- Error messages must be descriptive and indicate which operation failed

# Implementation Plan
1. **Analyze existing Node methods**: Study patterns in existing methods like CreateTransactionCheckStatus, GetMaxFlow, SetSettlementLine to understand:
   - HTTP client usage patterns
   - URL construction with path parameters and query parameters
   - Request header setting
   - Response parsing and error handling
   - Use of t.Fatalf() for error reporting

2. **Define response struct imports**: Ensure proper imports for RateItem and response structs:
   - Import or define RateItem struct locally if needed
   - Define RatesListResponse struct if not available
   - Handle JSON unmarshaling for all response types

3. **Implement SetExchangeRate method**:
   - Construct POST URL with path parameters: `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/`
   - Add query parameter `real_rate` with provided value
   - Add optional `min_exchange_amount` and `max_exchange_amount` if provided
   - Send POST request with appropriate headers
   - Check for HTTP 200 status, call t.Fatalf() on errors
   - Parse empty data response (success confirmation)

4. **Implement SetExchangeRateNative method**:
   - Similar to SetExchangeRate but use `value` and `shift` query parameters
   - Convert int16 shift to string for query parameter
   - Handle same optional min/max amounts

5. **Implement GetExchangeRate method**:
   - Construct GET URL with path parameters
   - Send GET request
   - Parse response containing single RateItem in data.rate field
   - Return pointer to RateItem

6. **Implement ListExchangeRates method**:
   - Construct GET URL: `/api/v1/node/rates/`
   - Parse response containing count and rates array
   - Return pointer to RatesListResponse struct

7. **Implement DeleteExchangeRate method**:
   - Construct DELETE URL with path parameters
   - Handle empty data response
   - Use t.Fatalf() for HTTP errors

8. **Implement ClearExchangeRates method**:
   - Construct DELETE URL: `/api/v1/node/rates/`
   - Handle empty data response
   - Use t.Fatalf() for HTTP errors

9. **Add necessary struct definitions** (if not imported):
   - Define RateItem struct matching vtcpd-cli implementation
   - Define RatesListResponse struct
   - Ensure proper JSON tags for all fields

10. **Test method signatures and basic functionality**:
    - Verify all methods compile correctly
    - Ensure method signatures match PRD specifications
    - Test basic HTTP request construction (can be verified via demo)

# Test Plan
## Objective
Verify that all 6 HTTP methods are correctly implemented and can successfully make HTTP requests to vtcpd-cli endpoints.

## Test Scope
- Method signature validation
- HTTP request construction
- Basic error handling
- Response parsing (structure validation)

## Test Environment
- Single vtcpd node with exchange rates functionality enabled
- vtcpd-cli with exchange rates endpoints available
- Go test environment

## Key Test Scenarios
1. **Method Compilation**: All methods compile without errors
2. **HTTP Request Construction**: URLs are correctly formed with path parameters
3. **Query Parameter Handling**: Parameters are correctly added to requests
4. **Response Structure**: JSON parsing works for expected response formats
5. **Error Handling**: HTTP errors trigger t.Fatalf() as expected

## Success Criteria
- All 6 methods are implemented and compile successfully
- Methods follow existing Node struct patterns
- Basic HTTP request/response cycle works for each method
- Error handling is consistent with existing methods

# Verification and Validation

## Architecture integrity
- Methods follow established Node struct patterns and conventions
- HTTP client usage is consistent with existing implementations
- No breaking changes to existing Node functionality
- Response struct definitions align with vtcpd-cli API specifications

## Security
- No additional security considerations beyond existing HTTP client usage
- Uses existing HTTP client configuration and security patterns
- No direct external network access beyond established vtcpd-cli communication

## Performance
- HTTP methods have minimal overhead beyond network requests
- Response parsing is efficient and doesn't introduce memory leaks
- Methods don't block unnecessarily or introduce performance regressions

## Scalability
- Methods scale with existing Node struct usage patterns
- No resource leaks or unbounded memory usage
- HTTP client reuse follows existing patterns

## Reliability
- Error handling is comprehensive and fails fast with clear messages
- HTTP status codes are properly validated
- JSON parsing handles malformed responses gracefully
- Methods are deterministic and don't introduce flaky behavior

## Maintainability
- Code follows existing conventions and is easy to understand
- Method names and signatures are clear and consistent
- Error messages are descriptive and helpful for debugging
- Implementation can be easily extended for future exchange rates features

## Cost
- Implementation has minimal resource overhead
- No additional dependencies or external services required
- Development effort is proportional to functionality provided

## Compliance
- Follows Go coding standards and project conventions
- Adheres to existing HTTP client patterns and security practices
- Compatible with existing test infrastructure and CI/CD pipelines

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify existing Node struct methods or functionality
- Do not change existing HTTP client configuration or patterns
- Follow existing error handling patterns using t.Fatalf() - do not return errors
- Use existing response struct definitions from vtcpd-cli where possible
