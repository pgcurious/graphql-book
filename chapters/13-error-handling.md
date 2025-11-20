# Chapter 13: Error Handling - Partial Failures in Graphs

> "In REST, requests either succeed or fail. In GraphQL, they can do both simultaneously. This changes everything."

---

## The Paradigm Shift

In traditional REST APIs, a request has exactly two outcomes:

```http
GET /api/users/123
✅ 200 OK → Success, here's your data
❌ 404 Not Found → Failure, no data

GET /api/users/123/posts
✅ 200 OK → Success, all posts
❌ 500 Internal Server Error → Failure, no data
```

**Clear binary: success XOR failure.**

But in GraphQL, a single query can request multiple pieces of data:

```graphql
query {
  user(id: "123") {    # Might succeed
    name
    email
    posts {            # Might fail!
      title
      comments {       # Might partially fail!
        text
        author {       # Might fail!
          name
        }
      }
    }
  }
}
```

What if:
- User exists ✅
- User's posts query fails ❌ (database timeout)
- Some comments load ✅, others fail ❌
- Some comment authors load ✅, others fail ❌

**Do we:**
1. **Return nothing** (treat any failure as total failure)?
2. **Return what we can** (partial success)?
3. **Hide errors** (return partial data, pretend everything worked)?

GraphQL chooses **option 2: return partial data + explicit errors**.

This is a **fundamental design decision** that makes GraphQL powerful but requires a different error handling approach.

---

## Part 1: The GraphQL Error Model

### The Dual-Channel Response

Every GraphQL response has **two channels**:

```json
{
  "data": {
    // The data that succeeded
  },
  "errors": [
    // The things that failed
  ]
}
```

**Both can exist simultaneously.**

### Example: Partial Success

**Query:**

```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
  settings {
    theme
  }
}
```

**Response (partial failure):**

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": null
    },
    "settings": {
      "theme": "dark"
    }
  },
  "errors": [
    {
      "message": "Database timeout while fetching posts",
      "locations": [{ "line": 5, "column": 5 }],
      "path": ["user", "posts"],
      "extensions": {
        "code": "DATABASE_TIMEOUT",
        "timeout": 5000
      }
    }
  ]
}
```

**What happened:**
- ✅ User name and email loaded successfully
- ❌ Posts failed (database timeout)
- ✅ Settings loaded successfully

**Result:** Client gets 80% of requested data + explicit error about the 20% that failed.

### Anatomy of a GraphQL Error

```java
public class GraphQLError {
    private String message;              // Human-readable error message
    private List<SourceLocation> locations;  // Where in query error occurred
    private List<Object> path;           // Path to field that failed
    private Map<String, Object> extensions;  // Custom data (error codes, etc.)
}
```

**Example:**

```json
{
  "message": "Post not found",
  "locations": [
    {
      "line": 3,
      "column": 5
    }
  ],
  "path": ["user", "posts", 2],  // user.posts[2] failed
  "extensions": {
    "code": "NOT_FOUND",
    "resourceType": "Post",
    "resourceId": "456",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

**The `path` field** is crucial—it tells the client **exactly which field** failed:
- `["user"]` → Top-level user query failed
- `["user", "posts"]` → user.posts field failed
- `["user", "posts", 2]` → The 3rd post (index 2) failed
- `["user", "posts", 2, "author"]` → The author of the 3rd post failed

---

## Part 2: Error Types and Codes

### Standard Error Categories

```java
public enum ErrorCode {
    // Validation errors (400-like)
    GRAPHQL_VALIDATION_FAILED,
    INVALID_SYNTAX,
    INVALID_TYPE,
    UNKNOWN_FIELD,

    // Authentication/Authorization (401/403-like)
    UNAUTHENTICATED,
    FORBIDDEN,
    INSUFFICIENT_PERMISSIONS,

    // Not Found (404-like)
    NOT_FOUND,
    RESOURCE_NOT_FOUND,

    // Server errors (500-like)
    INTERNAL_SERVER_ERROR,
    DATABASE_ERROR,
    EXTERNAL_API_ERROR,

    // Business logic errors
    VALIDATION_ERROR,
    BUSINESS_RULE_VIOLATION,

    // Performance errors
    TIMEOUT,
    RATE_LIMIT_EXCEEDED,
    QUERY_TOO_COMPLEX
}
```

### Implementing Custom Error Types

```java
public abstract class BaseGraphQLError extends GraphQLException implements GraphQLError {
    private final ErrorCode code;
    private final Map<String, Object> extensions;

    protected BaseGraphQLError(String message, ErrorCode code) {
        super(message);
        this.code = code;
        this.extensions = new HashMap<>();
        this.extensions.put("code", code.name());
    }

    @Override
    public List<SourceLocation> getLocations() {
        return null;  // Framework fills this in
    }

    @Override
    public ErrorClassification getErrorType() {
        return ErrorClassification.errorClassification(code.name());
    }

    @Override
    public Map<String, Object> getExtensions() {
        return extensions;
    }

    protected void addExtension(String key, Object value) {
        extensions.put(key, value);
    }
}
```

**Concrete error types:**

```java
public class ResourceNotFoundException extends BaseGraphQLError {
    public ResourceNotFoundException(String resourceType, String resourceId) {
        super(
            String.format("%s with id '%s' not found", resourceType, resourceId),
            ErrorCode.NOT_FOUND
        );
        addExtension("resourceType", resourceType);
        addExtension("resourceId", resourceId);
    }
}

public class UnauthorizedException extends BaseGraphQLError {
    public UnauthorizedException(String resource) {
        super(
            String.format("Unauthorized to access %s", resource),
            ErrorCode.FORBIDDEN
        );
        addExtension("resource", resource);
    }
}

public class ValidationException extends BaseGraphQLError {
    public ValidationException(String field, String reason) {
        super(
            String.format("Validation failed for field '%s': %s", field, reason),
            ErrorCode.VALIDATION_ERROR
        );
        addExtension("field", field);
        addExtension("reason", reason);
    }
}

public class DatabaseTimeoutException extends BaseGraphQLError {
    public DatabaseTimeoutException(int timeoutMs) {
        super(
            String.format("Database query timed out after %dms", timeoutMs),
            ErrorCode.TIMEOUT
        );
        addExtension("timeoutMs", timeoutMs);
    }
}
```

---

## Part 3: Throwing Errors in Resolvers

### Basic Error Throwing

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {
    @Autowired
    private UserRepository userRepository;

    public User user(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id.toString()));
    }
}
```

**Response when user not found:**

```json
{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User with id '123' not found",
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND",
        "resourceType": "User",
        "resourceId": "123"
      }
    }
  ]
}
```

### When to Throw vs When to Return Null

**Guidelines:**

```java
@Component
public class UserResolver {

    // ❌ DON'T: Return null for required data
    public User getUser(Long id) {
        User user = userRepository.findById(id).orElse(null);
        return user;  // Returns null, but schema says User! (non-null)
        // This will cause GraphQL to throw a generic "field error"
    }

    // ✅ DO: Throw for required data that's missing
    public User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id.toString()));
        // Clear, specific error with context
    }

    // ✅ DO: Return null for optional data
    public String nickname(User user) {
        return user.getNickname();  // null is fine, schema says String (nullable)
    }

    // ✅ DO: Return empty collections instead of null
    public List<Post> posts(User user) {
        List<Post> posts = postRepository.findByAuthorId(user.getId());
        return posts != null ? posts : Collections.emptyList();
        // Never return null for lists, return empty list
    }
}
```

**Rule of thumb:**
- **Non-null field (`User!`)**: Throw exception if you can't provide it
- **Nullable field (`User`)**: Return null if appropriate
- **Lists (`[Post!]!`)**: Return empty list, not null
- **Business logic violations**: Throw descriptive exceptions

---

## Part 4: Null Propagation and Bubbling

### How Null Propagates

When a **non-null field** fails, the null "bubbles up" to the nearest nullable ancestor.

**Schema:**

```graphql
type Query {
  user(id: ID!): User    # Nullable
}

type User {
  id: ID!                # Non-null
  name: String!          # Non-null
  email: String!         # Non-null
  posts: [Post!]!        # Non-null list of non-null posts
}

type Post {
  id: ID!                # Non-null
  title: String!         # Non-null
  author: User!          # Non-null
}
```

**Case 1: Nullable field fails**

```graphql
query {
  user(id: "123") {  # Nullable User
    name
  }
}
```

If `user` resolver throws exception:

```json
{
  "data": {
    "user": null     // Stopped here (user is nullable)
  },
  "errors": [{ "message": "User not found", "path": ["user"] }]
}
```

**Case 2: Non-null field fails**

```graphql
query {
  user(id: "123") {
    name   # Non-null String!
    email  # Non-null String!
  }
}
```

If `email` resolver throws exception:

```json
{
  "data": {
    "user": null     // Bubbled up! (email is non-null, so user becomes null)
  },
  "errors": [{
    "message": "Email not found",
    "path": ["user", "email"]
  }]
}
```

**Why `user` became null:**
- `email` field is `String!` (non-null)
- Can't return a User with `email: null` (violates schema)
- Null bubbles to nearest nullable ancestor: `user` field
- `user` is nullable, so it becomes `null`

**Case 3: Multiple failures in a list**

```graphql
query {
  user(id: "123") {
    posts {  # Non-null list [Post!]!
      title
      author {  # Non-null User!
        name
      }
    }
  }
}
```

If post author lookup fails for one post:

```json
{
  "data": {
    "user": null    // Entire user became null!
  },
  "errors": [{
    "message": "Author not found",
    "path": ["user", "posts", 2, "author"]
  }]
}
```

**Why the entire `user` became null:**
1. `posts[2].author` failed
2. `author` is `User!` (non-null)
3. Can't have a Post without an author
4. `posts[2]` becomes null
5. But `posts` is `[Post!]!` (non-null list of non-null posts)
6. Can't have `null` in the list
7. `posts` becomes null
8. But `posts` is non-null `[Post!]!`
9. Bubbles to `user`, which is nullable
10. **Result: entire user is null**

### Preventing Excessive Null Propagation

**Strategy 1: Make fields nullable where appropriate**

```graphql
type User {
  id: ID!
  name: String!
  email: String        # Nullable - if email fails, only email is null
  posts: [Post!]       # Nullable list - if posts fail, only posts is null
}
```

Now if `posts` fails:

```json
{
  "data": {
    "user": {
      "id": "123",
      "name": "Alice",
      "email": "alice@example.com",
      "posts": null    // Only posts is null, rest of user is intact
    }
  },
  "errors": [{
    "message": "Posts query failed",
    "path": ["user", "posts"]
  }]
}
```

**Strategy 2: Catch errors in resolvers**

```java
@Component
public class UserResolver implements GraphQLResolver<User> {
    @Autowired
    private PostService postService;

    public List<Post> posts(User user) {
        try {
            return postService.getPostsByAuthorId(user.getId());
        } catch (DatabaseException e) {
            // Log error for debugging
            logger.error("Failed to load posts for user {}", user.getId(), e);

            // Return empty list instead of propagating exception
            return Collections.emptyList();
        }
    }
}
```

**Trade-off:** Silent failure vs explicit error. Choose based on criticality.

**Strategy 3: Use nullable wrappers for risky operations**

```graphql
type User {
  id: ID!
  name: String!

  # Instead of failing entire query if posts fail:
  postsResult: PostsResult!  # Non-null wrapper, but allows error state
}

type PostsResult {
  posts: [Post!]      # Nullable
  error: String       # Error message if posts failed
}
```

```java
public PostsResult postsResult(User user) {
    try {
        List<Post> posts = postService.getPostsByAuthorId(user.getId());
        return new PostsResult(posts, null);
    } catch (Exception e) {
        return new PostsResult(null, "Failed to load posts: " + e.getMessage());
    }
}
```

**Response:**

```json
{
  "data": {
    "user": {
      "id": "123",
      "name": "Alice",
      "postsResult": {
        "posts": null,
        "error": "Failed to load posts: Database timeout"
      }
    }
  }
}
```

**Benefit:** Client gets explicit error info without the error bubbling up.

---

## Part 5: Error Masking and Security

### The Security Risk

**DON'T expose internal errors to clients:**

```java
public User user(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new RuntimeException(
            "Query failed: SELECT * FROM users WHERE id = " + id +
            " - Connection to database 'prod-db-master-01.internal:5432' refused"
        ));
}
```

**Response exposes sensitive info:**

```json
{
  "errors": [{
    "message": "Query failed: SELECT * FROM users WHERE id = 123 - Connection to database 'prod-db-master-01.internal:5432' refused",
    "path": ["user"]
  }]
}
```

**Attacker now knows:**
- Your database technology (PostgreSQL, port 5432)
- Internal hostname structure
- You're running a "master" database
- SQL query structure

### Solution: Error Sanitization

```java
@Component
public class ErrorSanitizer implements DataFetcherExceptionHandler {
    private static final Logger logger = LoggerFactory.getLogger(ErrorSanitizer.class);

    @Value("${graphql.error.detailed:false}")
    private boolean showDetailedErrors;  // false in production, true in dev

    @Override
    public DataFetcherExceptionHandlerResult onException(
            DataFetcherExceptionHandlerParameters handlerParameters) {

        Throwable exception = handlerParameters.getException();
        SourceLocation sourceLocation = handlerParameters.getSourceLocation();
        ResultPath path = handlerParameters.getPath();

        GraphQLError error;

        if (exception instanceof BaseGraphQLError) {
            // Our custom errors are safe to show
            error = (GraphQLError) exception;

        } else {
            // Unexpected exception - sanitize it

            // Log full details server-side
            logger.error("Unexpected GraphQL error at path {}: {}",
                        path, exception.getMessage(), exception);

            // Return sanitized error to client
            if (showDetailedErrors) {
                // Development: show real error
                error = GraphqlErrorBuilder.newError()
                    .message(exception.getMessage())
                    .location(sourceLocation)
                    .path(path)
                    .extensions(Map.of(
                        "code", "INTERNAL_SERVER_ERROR",
                        "exception", exception.getClass().getSimpleName()
                    ))
                    .build();
            } else {
                // Production: hide details
                error = GraphqlErrorBuilder.newError()
                    .message("Internal server error")
                    .location(sourceLocation)
                    .path(path)
                    .extensions(Map.of(
                        "code", "INTERNAL_SERVER_ERROR",
                        "errorId", UUID.randomUUID().toString()  // For log correlation
                    ))
                    .build();
            }
        }

        return DataFetcherExceptionHandlerResult.newResult()
            .error(error)
            .build();
    }
}
```

**Production response (sanitized):**

```json
{
  "errors": [{
    "message": "Internal server error",
    "path": ["user"],
    "extensions": {
      "code": "INTERNAL_SERVER_ERROR",
      "errorId": "a7b3c4d5-e6f7-8901-2345-6789abcdef01"
    }
  }]
}
```

**Server logs:**

```
ERROR [ErrorSanitizer] Unexpected GraphQL error at path [user]:
Connection to database 'prod-db-master-01.internal:5432' refused
errorId=a7b3c4d5-e6f7-8901-2345-6789abcdef01
java.sql.SQLException: Connection refused
    at org.postgresql.jdbc.PgConnection.connect(...)
    ...
```

**Benefits:**
- Client sees generic error
- Support team can correlate with logs using `errorId`
- No sensitive information leaked

---

## Part 6: Handling Errors from External Services

### The Challenge

Your GraphQL server often calls external APIs, which can fail:

```java
@Component
public class PostResolver implements GraphQLResolver<Post> {

    @Autowired
    private RestTemplate restTemplate;

    public UserStats stats(Post post) {
        // Call external analytics API
        String url = "https://analytics-api.example.com/posts/" + post.getId() + "/stats";

        try {
            return restTemplate.getForObject(url, UserStats.class);
        } catch (HttpClientErrorException e) {
            // 4xx error from external API
            throw new ExternalApiException("Analytics API returned error: " + e.getStatusCode());
        } catch (HttpServerErrorException e) {
            // 5xx error from external API
            throw new ExternalApiException("Analytics API is down");
        } catch (ResourceAccessException e) {
            // Timeout or network error
            throw new ExternalApiException("Analytics API timeout");
        }
    }
}
```

**Custom exception:**

```java
public class ExternalApiException extends BaseGraphQLError {
    public ExternalApiException(String message) {
        super(message, ErrorCode.EXTERNAL_API_ERROR);
    }
}
```

### Circuit Breaker Pattern

Prevent cascading failures when external service is down:

```java
@Service
public class AnalyticsService {
    private static final int FAILURE_THRESHOLD = 5;
    private static final long TIMEOUT_MS = 60000;  // 1 minute

    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicLong circuitOpenedAt = new AtomicLong(0);
    private final AtomicBoolean circuitOpen = new AtomicBoolean(false);

    @Autowired
    private RestTemplate restTemplate;

    public Optional<UserStats> getUserStats(Long postId) {
        // Check if circuit is open
        if (circuitOpen.get()) {
            long now = System.currentTimeMillis();
            if (now - circuitOpenedAt.get() < TIMEOUT_MS) {
                // Circuit still open, fail fast
                logger.warn("Circuit breaker open, skipping analytics API call");
                return Optional.empty();
            } else {
                // Try to close circuit
                logger.info("Attempting to close circuit breaker");
                circuitOpen.set(false);
                failureCount.set(0);
            }
        }

        try {
            UserStats stats = restTemplate.getForObject(
                "https://analytics-api.example.com/posts/" + postId + "/stats",
                UserStats.class
            );

            // Success! Reset failure count
            failureCount.set(0);
            return Optional.of(stats);

        } catch (Exception e) {
            // Failure! Increment counter
            int failures = failureCount.incrementAndGet();

            if (failures >= FAILURE_THRESHOLD) {
                // Open the circuit
                logger.error("Circuit breaker opened after {} failures", failures);
                circuitOpen.set(true);
                circuitOpenedAt.set(System.currentTimeMillis());
            }

            return Optional.empty();
        }
    }
}
```

**Resolver using circuit breaker:**

```java
@Component
public class PostResolver implements GraphQLResolver<Post> {
    @Autowired
    private AnalyticsService analyticsService;

    public UserStats stats(Post post) {
        return analyticsService.getUserStats(post.getId())
            .orElse(new UserStats(0, 0, 0));  // Fallback to empty stats
    }
}
```

**Benefits:**
- **Fail fast** when service is down (don't waste time on timeouts)
- **Graceful degradation** (return empty stats instead of error)
- **Automatic recovery** (circuit closes after timeout period)

---

## Part 7: Validation Errors

### Input Validation

```java
public class CreatePostInput {
    private String title;
    private String content;
    private Long authorId;

    public void validate() {
        List<String> errors = new ArrayList<>();

        if (title == null || title.trim().isEmpty()) {
            errors.add("Title is required");
        } else if (title.length() > 200) {
            errors.add("Title must be 200 characters or less");
        }

        if (content == null || content.trim().isEmpty()) {
            errors.add("Content is required");
        } else if (content.length() > 10000) {
            errors.add("Content must be 10,000 characters or less");
        }

        if (authorId == null || authorId <= 0) {
            errors.add("Valid author ID is required");
        }

        if (!errors.isEmpty()) {
            throw new ValidationException("Input validation failed", errors);
        }
    }
}
```

**Custom validation exception:**

```java
public class ValidationException extends BaseGraphQLError {
    public ValidationException(String message, List<String> validationErrors) {
        super(message, ErrorCode.VALIDATION_ERROR);
        addExtension("validationErrors", validationErrors);
    }
}
```

**Usage:**

```java
@Component
public class MutationResolver implements GraphQLMutationResolver {

    public Post createPost(CreatePostInput input) {
        // Validate input
        input.validate();

        // If we get here, input is valid
        return postService.createPost(input);
    }
}
```

**Error response:**

```json
{
  "data": {
    "createPost": null
  },
  "errors": [{
    "message": "Input validation failed",
    "path": ["createPost"],
    "extensions": {
      "code": "VALIDATION_ERROR",
      "validationErrors": [
        "Title is required",
        "Content must be 10,000 characters or less"
      ]
    }
  }]
}
```

### Bean Validation Integration

Use JSR-303 annotations:

```java
public class CreatePostInput {
    @NotBlank(message = "Title is required")
    @Size(max = 200, message = "Title must be 200 characters or less")
    private String title;

    @NotBlank(message = "Content is required")
    @Size(max = 10000, message = "Content must be 10,000 characters or less")
    private String content;

    @NotNull(message = "Author ID is required")
    @Positive(message = "Author ID must be positive")
    private Long authorId;

    // Getters and setters
}
```

**Validation interceptor:**

```java
@Component
public class ValidationInstrumentation extends SimpleInstrumentation {
    private final Validator validator;

    @Autowired
    public ValidationInstrumentation(Validator validator) {
        this.validator = validator;
    }

    @Override
    public DataFetcher<?> instrumentDataFetcher(
            DataFetcher<?> dataFetcher,
            InstrumentationFieldFetchParameters parameters) {

        return (DataFetcher<Object>) environment -> {
            // Validate all input arguments
            Map<String, Object> arguments = environment.getArguments();

            for (Object arg : arguments.values()) {
                if (arg != null) {
                    Set<ConstraintViolation<Object>> violations = validator.validate(arg);

                    if (!violations.isEmpty()) {
                        List<String> errors = violations.stream()
                            .map(ConstraintViolation::getMessage)
                            .collect(Collectors.toList());

                        throw new ValidationException("Validation failed", errors);
                    }
                }
            }

            // Validation passed, execute resolver
            return dataFetcher.get(environment);
        };
    }
}
```

---

## Part 8: Partial Errors in Lists

### The Challenge

```graphql
query {
  posts {
    id
    title
    author {  # What if some authors fail to load?
      name
    }
  }
}
```

If we have 100 posts, but 3 authors fail to load:
- **Option A:** Fail entire query (lose all 100 posts)
- **Option B:** Return 97 successful posts + 3 errors

**GraphQL chooses Option B** (partial success).

### Handling with Try-Catch in Resolvers

```java
@Component
public class PostResolver implements GraphQLResolver<Post> {
    @Autowired
    private UserService userService;

    public User author(Post post) {
        try {
            return userService.getUserById(post.getAuthorId());
        } catch (UserNotFoundException e) {
            // Log the error
            logger.warn("Author {} not found for post {}",
                       post.getAuthorId(), post.getId());

            // Re-throw as GraphQL error
            throw new ResourceNotFoundException("User", post.getAuthorId().toString());
        }
    }
}
```

**Response:**

```json
{
  "data": {
    "posts": [
      { "id": "1", "title": "Post 1", "author": { "name": "Alice" } },
      { "id": "2", "title": "Post 2", "author": { "name": "Bob" } },
      { "id": "3", "title": "Post 3", "author": null },  // Failed
      { "id": "4", "title": "Post 4", "author": { "name": "Charlie" } }
    ]
  },
  "errors": [
    {
      "message": "User with id '999' not found",
      "path": ["posts", 2, "author"],
      "extensions": {
        "code": "NOT_FOUND",
        "resourceType": "User",
        "resourceId": "999"
      }
    }
  ]
}
```

**Result:** Client gets 97 successful posts + clear error about the 3 failures.

---

## Part 9: Best Practices

### 1. Use Structured Error Codes

**DON'T:**

```json
{
  "errors": [{
    "message": "Something went wrong"
  }]
}
```

**DO:**

```json
{
  "errors": [{
    "message": "User with id '123' not found",
    "extensions": {
      "code": "NOT_FOUND",
      "resourceType": "User",
      "resourceId": "123"
    }
  }]
}
```

Clients can handle errors programmatically:

```typescript
// Client code
if (error.extensions.code === 'NOT_FOUND') {
  showNotFoundUI();
} else if (error.extensions.code === 'UNAUTHENTICATED') {
  redirectToLogin();
} else if (error.extensions.code === 'VALIDATION_ERROR') {
  showValidationErrors(error.extensions.validationErrors);
}
```

### 2. Include Actionable Information

**DON'T:**

```json
{
  "message": "Forbidden"
}
```

**DO:**

```json
{
  "message": "You don't have permission to delete this post",
  "extensions": {
    "code": "FORBIDDEN",
    "resource": "Post",
    "resourceId": "456",
    "requiredPermission": "DELETE_POST",
    "userPermissions": ["READ_POST", "CREATE_POST"]
  }
}
```

User knows exactly what they need to do (get DELETE_POST permission).

### 3. Log Errors with Context

```java
@Component
public class ErrorLoggingInstrumentation extends SimpleInstrumentation {

    @Override
    public CompletableFuture<ExecutionResult> instrumentExecutionResult(
            ExecutionResult executionResult,
            InstrumentationExecutionParameters parameters) {

        if (executionResult.getErrors() != null && !executionResult.getErrors().isEmpty()) {
            String query = parameters.getQuery();
            Map<String, Object> variables = parameters.getVariables();

            logger.error("GraphQL query completed with {} errors. Query: {}, Variables: {}",
                        executionResult.getErrors().size(),
                        query,
                        variables);

            executionResult.getErrors().forEach(error -> {
                logger.error("  Error at path {}: {} ({})",
                            error.getPath(),
                            error.getMessage(),
                            error.getExtensions().get("code"));
            });
        }

        return CompletableFuture.completedFuture(executionResult);
    }
}
```

### 4. Make Fields Nullable by Default

**DON'T make everything non-null:**

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  phone: String!        # Non-null - if phone lookup fails, entire user fails
  address: Address!     # Non-null - if address lookup fails, entire user fails
  posts: [Post!]!       # Non-null - if posts query fails, entire user fails
}
```

**DO make fields nullable unless truly required:**

```graphql
type User {
  id: ID!               # Actually required
  name: String!         # Actually required
  email: String         # Nullable - if email lookup fails, only email is null
  phone: String         # Nullable
  address: Address      # Nullable
  posts: [Post!]        # Nullable list - if posts query fails, only posts is null
}
```

### 5. Provide Fallback Values

```java
@Component
public class UserResolver implements GraphQLResolver<User> {

    public List<Post> posts(User user) {
        try {
            return postService.getPostsByAuthorId(user.getId());
        } catch (Exception e) {
            logger.error("Failed to load posts for user {}", user.getId(), e);
            return Collections.emptyList();  // Fallback to empty list
        }
    }

    public UserStats stats(User user) {
        try {
            return statsService.getUserStats(user.getId());
        } catch (Exception e) {
            logger.error("Failed to load stats for user {}", user.getId(), e);
            return UserStats.empty();  // Fallback to zero stats
        }
    }
}
```

**Trade-off:** Silent failure (client doesn't know there was an error) vs graceful degradation (client gets partial data).

Choose based on:
- **Critical data:** Throw error (e.g., payment information)
- **Nice-to-have data:** Return fallback (e.g., analytics, recommendations)

---

## Summary: Error Handling Philosophy

### In REST
```
Request → Success (200) XOR Failure (4xx/5xx)
All or nothing.
```

### In GraphQL
```
Request → Partial Success + Partial Failure
Data + Errors can coexist.
```

**Key Principles:**

1. **Embrace partial success** - return what you can
2. **Be explicit about failures** - structured error codes
3. **Protect sensitive information** - sanitize internal errors
4. **Make errors actionable** - include context for debugging
5. **Design for resilience** - nullable fields, fallback values, circuit breakers
6. **Log comprehensively** - server-side logging for debugging
7. **Validate early** - clear validation error messages

**Remember:** In GraphQL, errors are not exceptional—they're a normal part of the response model. Design your schema and resolvers with this in mind.

---

**Next:** [Chapter 14: Batching and Performance Optimization →](14-batching-performance.md)
