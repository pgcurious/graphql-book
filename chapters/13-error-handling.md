# Chapter 13: Error Handling - Partial Failures in Graphs

**Status:** Detailed Outline (To Be Expanded)

---

## Outline

### Part 1: GraphQL Error Model
- Errors array in response
- Partial success is valid
- Error paths and locations
- Extensions for custom data

### Part 2: Error Types and Codes
- Validation errors
- Execution errors
- Authorization errors
- Custom error types

### Part 3: Throwing Errors in Resolvers
- Exception handling
- Error propagation
- Null vs error
- When to throw vs return null

### Part 4: Structured Error Responses
```json
{
  "errors": [{
    "message": "Resource not found",
    "locations": [{"line": 2, "column": 3}],
    "path": ["user", "posts", 0],
    "extensions": {
      "code": "NOT_FOUND",
      "resourceType": "Post",
      "resourceId": "123"
    }
  }],
  "data": {
    "user": null
  }
}
```

### Part 5: Error Masking and Security
- Hiding internal errors from clients
- Development vs production error messages
- Logging sensitive errors server-side
- Error sanitization

### Part 6: Null Propagation
- Nullable fields and error handling
- Non-null fields and bubbling
- Stopping null propagation
- List error handling

### Part 7: Retry Logic and Circuit Breakers
- Transient failure handling
- Exponential backoff
- Circuit breaker pattern
- Fallback values

### Java Implementation Topics:
- Exception mappers
- GraphQLError interface
- Custom error classes
- Logging integration
- Sentry/error tracking integration

### Best Practices:
- Error code standardization
- Client-friendly error messages
- Actionable error information
- Error documentation

---

**Next:** [Chapter 14: Batching and Performance Optimization ’](14-batching-performance.md)
