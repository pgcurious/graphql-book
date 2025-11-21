# Chapter 17: Security - Query Depth, Complexity, and Cost Analysis

> "In REST, malicious queries are expensive. In GraphQL, they can be devastating. But with the right defenses, GraphQL can be more secure than REST ever was."

---

## The GraphQL Security Problem

REST APIs have natural security boundaries:

```
GET /api/users          # Fixed cost
GET /api/posts?limit=10 # Limited by parameters
```

GraphQL removes these boundaries:

```graphql
query MaliciousQuery {
  users {
    posts {
      comments {
        author {
          posts {
            comments {
              author {
                posts {
                  comments {
                    # ... infinite nesting
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

This single query could:
- Trigger thousands of database queries
- Consume gigabytes of memory
- Take minutes to execute
- Crash your server

**The problem:** GraphQL's flexibility is also its vulnerability.

### Why Traditional Security Doesn't Work

**URL-based rate limiting fails:**
```
POST /graphql  # All queries hit the same endpoint
```

Every request looks the same to traditional WAFs and rate limiters.

**Request size limiting fails:**
```graphql
# Tiny query, massive cost
query { users { posts { comments { author { posts } } } } }
```

**Traditional authentication/authorization isn't enough:**
```graphql
# Authenticated user can still craft expensive queries
query {
  me {
    friends {
      friends {
        friends {
          # ...
        }
      }
    }
  }
}
```

We need **GraphQL-specific security mechanisms**.

---

## Query Depth Limiting

The first line of defense: limit how deep queries can nest.

### Understanding Depth

```graphql
query {
  user(id: "1") {           # depth: 1
    name                     # depth: 2
    posts {                  # depth: 2
      title                  # depth: 3
      comments {             # depth: 3
        text                 # depth: 4
        author {             # depth: 4
          name               # depth: 5
        }
      }
    }
  }
}
```

**Depth = maximum nesting level in the query.**

### Implementing Depth Analysis

```java
public class QueryDepthAnalyzer {

    private final int maxDepth;

    public QueryDepthAnalyzer(int maxDepth) {
        this.maxDepth = maxDepth;
    }

    public void validateDepth(Document document) {
        int depth = calculateDepth(document);

        if (depth > maxDepth) {
            throw new QueryTooDeepException(
                String.format("Query depth %d exceeds maximum allowed depth %d",
                    depth, maxDepth)
            );
        }
    }

    private int calculateDepth(Document document) {
        // Get all operation definitions (queries, mutations)
        List<OperationDefinition> operations = document.getDefinitions().stream()
            .filter(def -> def instanceof OperationDefinition)
            .map(def -> (OperationDefinition) def)
            .collect(Collectors.toList());

        // Calculate max depth across all operations
        return operations.stream()
            .mapToInt(this::calculateOperationDepth)
            .max()
            .orElse(0);
    }

    private int calculateOperationDepth(OperationDefinition operation) {
        return calculateSelectionSetDepth(operation.getSelectionSet(), 0);
    }

    private int calculateSelectionSetDepth(SelectionSet selectionSet, int currentDepth) {
        if (selectionSet == null || selectionSet.getSelections().isEmpty()) {
            return currentDepth;
        }

        int maxDepth = currentDepth;

        for (Selection<?> selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;

                // Recurse into nested selections
                int fieldDepth = calculateSelectionSetDepth(
                    field.getSelectionSet(),
                    currentDepth + 1
                );

                maxDepth = Math.max(maxDepth, fieldDepth);

            } else if (selection instanceof InlineFragment) {
                InlineFragment fragment = (InlineFragment) selection;

                int fragmentDepth = calculateSelectionSetDepth(
                    fragment.getSelectionSet(),
                    currentDepth
                );

                maxDepth = Math.max(maxDepth, fragmentDepth);

            } else if (selection instanceof FragmentSpread) {
                // Need to look up fragment definition and recurse
                // (simplified for example)
                maxDepth = Math.max(maxDepth, currentDepth + 1);
            }
        }

        return maxDepth;
    }
}
```

### Integrating with GraphQL Java

```java
@Configuration
public class GraphQLConfig {

    @Bean
    public GraphQL graphQL(GraphQLSchema schema) {
        return GraphQL.newGraphQL(schema)
            .instrumentation(new DepthLimitingInstrumentation(10))
            .build();
    }
}

public class DepthLimitingInstrumentation extends SimpleInstrumentation {

    private final int maxDepth;

    public DepthLimitingInstrumentation(int maxDepth) {
        this.maxDepth = maxDepth;
    }

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        // Validate depth before execution
        QueryDepthAnalyzer analyzer = new QueryDepthAnalyzer(maxDepth);
        analyzer.validateDepth(parameters.getDocument());

        return super.beginExecution(parameters);
    }
}

public class QueryTooDeepException extends GraphQLException {
    public QueryTooDeepException(String message) {
        super(message);
    }
}
```

### Choosing the Right Depth Limit

```java
// Too restrictive
maxDepth = 3;  // Many legitimate queries will fail

// Common choice
maxDepth = 7-10;  // Handles most use cases

// Too permissive
maxDepth = 20;  // Still vulnerable to attacks
```

**Trade-off:** Higher depth = more flexibility, but more risk.

---

## Query Complexity Analysis

Depth limiting isn't enough:

```graphql
query {
  users(limit: 10000) {    # depth: 1, but returns 10,000 items!
    posts(limit: 100) {    # depth: 2, but 1,000,000 total items!
      title
    }
  }
}
```

This query has depth 2, but could return 1 million posts!

### Complexity Scoring

Assign a "cost" to each field:

```graphql
type Query {
  user(id: ID!): User         # cost: 1
  users(limit: Int!): [User!] # cost: depends on limit
}

type User {
  id: ID!                     # cost: 0 (scalar)
  name: String!               # cost: 0 (scalar)
  posts: [Post!]!             # cost: depends on result size
}
```

### Static Complexity Calculation

```java
public class QueryComplexityAnalyzer {

    private final int maxComplexity;
    private final GraphQLSchema schema;

    public QueryComplexityAnalyzer(int maxComplexity, GraphQLSchema schema) {
        this.maxComplexity = maxComplexity;
        this.schema = schema;
    }

    public void validateComplexity(Document document, Map<String, Object> variables) {
        int complexity = calculateComplexity(document, variables);

        if (complexity > maxComplexity) {
            throw new QueryTooComplexException(
                String.format("Query complexity %d exceeds maximum %d",
                    complexity, maxComplexity)
            );
        }
    }

    private int calculateComplexity(Document document, Map<String, Object> variables) {
        List<OperationDefinition> operations = document.getDefinitions().stream()
            .filter(def -> def instanceof OperationDefinition)
            .map(def -> (OperationDefinition) def)
            .collect(Collectors.toList());

        return operations.stream()
            .mapToInt(op -> calculateOperationComplexity(op, variables))
            .max()
            .orElse(0);
    }

    private int calculateOperationComplexity(
            OperationDefinition operation,
            Map<String, Object> variables) {

        GraphQLObjectType rootType = getRootType(operation);
        return calculateSelectionSetComplexity(
            operation.getSelectionSet(),
            rootType,
            variables,
            1  // multiplier
        );
    }

    private int calculateSelectionSetComplexity(
            SelectionSet selectionSet,
            GraphQLObjectType parentType,
            Map<String, Object> variables,
            int multiplier) {

        if (selectionSet == null) {
            return 0;
        }

        int totalComplexity = 0;

        for (Selection<?> selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;

                // Get field definition from schema
                GraphQLFieldDefinition fieldDef = parentType.getFieldDefinition(field.getName());

                if (fieldDef == null) {
                    continue;
                }

                // Base cost of this field
                int fieldCost = getFieldCost(fieldDef);

                // Calculate multiplier based on arguments (e.g., limit parameter)
                int fieldMultiplier = calculateFieldMultiplier(field, variables);

                // Cost of this field
                int cost = fieldCost * multiplier * fieldMultiplier;
                totalComplexity += cost;

                // Recurse into nested fields
                GraphQLType fieldType = unwrapNonNull(fieldDef.getType());

                if (fieldType instanceof GraphQLObjectType) {
                    int nestedComplexity = calculateSelectionSetComplexity(
                        field.getSelectionSet(),
                        (GraphQLObjectType) fieldType,
                        variables,
                        multiplier * fieldMultiplier
                    );

                    totalComplexity += nestedComplexity;
                }
            }
        }

        return totalComplexity;
    }

    private int getFieldCost(GraphQLFieldDefinition fieldDef) {
        // Check for custom cost directive
        GraphQLDirective costDirective = fieldDef.getDirective("cost");

        if (costDirective != null) {
            GraphQLArgument complexityArg = costDirective.getArgument("complexity");
            if (complexityArg != null && complexityArg.getValue() != null) {
                return (Integer) complexityArg.getValue();
            }
        }

        // Default costs
        GraphQLType type = unwrapNonNull(fieldDef.getType());

        if (type instanceof GraphQLList) {
            return 10;  // Lists are more expensive
        } else if (type instanceof GraphQLObjectType) {
            return 1;   // Objects have moderate cost
        } else {
            return 0;   // Scalars are cheap
        }
    }

    private int calculateFieldMultiplier(Field field, Map<String, Object> variables) {
        // Check for 'limit' or 'first' arguments
        Argument limitArg = field.getArgument("limit");
        if (limitArg == null) {
            limitArg = field.getArgument("first");
        }

        if (limitArg != null) {
            Value<?> value = limitArg.getValue();

            if (value instanceof IntValue) {
                return ((IntValue) value).getValue().intValue();
            } else if (value instanceof VariableReference) {
                String varName = ((VariableReference) value).getName();
                Object varValue = variables.get(varName);

                if (varValue instanceof Integer) {
                    return (Integer) varValue;
                }
            }
        }

        // Default multiplier
        return 1;
    }

    private GraphQLType unwrapNonNull(GraphQLType type) {
        if (type instanceof GraphQLNonNull) {
            return ((GraphQLNonNull) type).getWrappedType();
        }
        return type;
    }

    private GraphQLObjectType getRootType(OperationDefinition operation) {
        switch (operation.getOperation()) {
            case QUERY:
                return schema.getQueryType();
            case MUTATION:
                return schema.getMutationType();
            case SUBSCRIPTION:
                return schema.getSubscriptionType();
            default:
                throw new IllegalArgumentException("Unknown operation type");
        }
    }
}
```

### Using Custom Directives for Costs

Define costs in your schema:

```graphql
directive @cost(complexity: Int!) on FIELD_DEFINITION

type Query {
  user(id: ID!): User @cost(complexity: 1)
  users(limit: Int!): [User!]! @cost(complexity: 10)
  search(query: String!): [SearchResult!]! @cost(complexity: 50)
}

type User {
  id: ID!
  name: String!
  posts(limit: Int = 10): [Post!]! @cost(complexity: 5)
  followers(limit: Int = 20): [User!]! @cost(complexity: 20)
}
```

### Instrumentation Integration

```java
public class ComplexityLimitingInstrumentation extends SimpleInstrumentation {

    private final int maxComplexity;
    private final GraphQLSchema schema;

    public ComplexityLimitingInstrumentation(int maxComplexity, GraphQLSchema schema) {
        this.maxComplexity = maxComplexity;
        this.schema = schema;
    }

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        QueryComplexityAnalyzer analyzer = new QueryComplexityAnalyzer(
            maxComplexity,
            schema
        );

        analyzer.validateComplexity(
            parameters.getDocument(),
            parameters.getVariables()
        );

        return super.beginExecution(parameters);
    }
}
```

---

## Rate Limiting

Even with depth and complexity limits, malicious users can spam requests.

### Per-User Rate Limiting

```java
@Component
public class GraphQLRateLimiter {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    // Allow 100 requests per minute per user
    private static final int REQUESTS_PER_MINUTE = 100;

    public void checkRateLimit(String userId) {
        RateLimiter limiter = limiters.computeIfAbsent(
            userId,
            k -> RateLimiter.create(REQUESTS_PER_MINUTE / 60.0)  // per second
        );

        if (!limiter.tryAcquire()) {
            throw new RateLimitExceededException(
                "Rate limit exceeded. Try again in a few seconds."
            );
        }
    }

    // Clean up old limiters periodically
    @Scheduled(fixedRate = 3600000)  // Every hour
    public void cleanup() {
        limiters.entrySet().removeIf(entry ->
            entry.getValue().getRate() == 0
        );
    }
}
```

### Cost-Based Rate Limiting

Instead of counting requests, count complexity points:

```java
@Component
public class CostBasedRateLimiter {

    // Map: userId -> remaining cost budget
    private final Map<String, TokenBucket> budgets = new ConcurrentHashMap<>();

    // Each user gets 10,000 complexity points per minute
    private static final int COST_PER_MINUTE = 10_000;
    private static final int REFILL_INTERVAL_SECONDS = 60;

    public void consumeCost(String userId, int cost) {
        TokenBucket bucket = budgets.computeIfAbsent(
            userId,
            k -> new TokenBucket(COST_PER_MINUTE, REFILL_INTERVAL_SECONDS)
        );

        if (!bucket.tryConsume(cost)) {
            throw new CostLimitExceededException(
                String.format("Cost limit exceeded. Query cost: %d", cost)
            );
        }
    }
}

public class TokenBucket {
    private final int capacity;
    private final int refillAmount;
    private final long refillIntervalSeconds;

    private int tokens;
    private long lastRefillTime;

    public TokenBucket(int capacity, long refillIntervalSeconds) {
        this.capacity = capacity;
        this.refillAmount = capacity;
        this.refillIntervalSeconds = refillIntervalSeconds;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
    }

    public synchronized boolean tryConsume(int cost) {
        refill();

        if (tokens >= cost) {
            tokens -= cost;
            return true;
        }

        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsedSeconds = (now - lastRefillTime) / 1000;

        if (elapsedSeconds >= refillIntervalSeconds) {
            tokens = Math.min(capacity, tokens + refillAmount);
            lastRefillTime = now;
        }
    }
}
```

### Integration with Instrumentation

```java
public class RateLimitingInstrumentation extends SimpleInstrumentation {

    private final GraphQLRateLimiter rateLimiter;
    private final CostBasedRateLimiter costLimiter;
    private final QueryComplexityAnalyzer complexityAnalyzer;

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        // Get current user
        GraphQLContext context = parameters.getContext();
        String userId = context.get("userId");

        if (userId == null) {
            userId = "anonymous";
        }

        // Check request rate limit
        rateLimiter.checkRateLimit(userId);

        // Calculate query cost
        int cost = complexityAnalyzer.calculateComplexity(
            parameters.getDocument(),
            parameters.getVariables()
        );

        // Check cost-based rate limit
        costLimiter.consumeCost(userId, cost);

        return super.beginExecution(parameters);
    }
}
```

---

## Authentication and Authorization

### JWT Authentication

```java
@Component
public class GraphQLAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenProvider jwtProvider;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        try {
            String token = extractJwtFromRequest(request);

            if (token != null && jwtProvider.validateToken(token)) {
                String userId = jwtProvider.getUserIdFromToken(token);

                // Store user in security context
                Authentication auth = new UsernamePasswordAuthenticationToken(
                    userId,
                    null,
                    Collections.emptyList()
                );

                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        } catch (Exception e) {
            logger.error("Could not set user authentication", e);
        }

        filterChain.doFilter(request, response);
    }

    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");

        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }

        return null;
    }
}
```

### GraphQL Context with User Info

```java
@Configuration
public class GraphQLContextBuilder {

    @Bean
    public GraphQLServletContextBuilder contextBuilder() {
        return new GraphQLServletContextBuilder() {
            @Override
            public GraphQLContext build(
                    HttpServletRequest request,
                    HttpServletResponse response) {

                // Get authenticated user from Spring Security
                Authentication auth = SecurityContextHolder.getContext()
                    .getAuthentication();

                GraphQLContext context = new GraphQLContext();

                if (auth != null && auth.isAuthenticated()) {
                    String userId = (String) auth.getPrincipal();
                    context.put("userId", userId);
                    context.put("roles", auth.getAuthorities());
                }

                return context;
            }
        };
    }
}
```

### Field-Level Authorization

```java
public class UserResolver implements GraphQLResolver<User> {

    @Autowired
    private AuthorizationService authService;

    public String email(User user, DataFetchingEnvironment env) {
        GraphQLContext context = env.getContext();
        String currentUserId = context.get("userId");

        // Only the user themselves or admins can see email
        if (!user.getId().equals(currentUserId) && !authService.isAdmin(currentUserId)) {
            throw new UnauthorizedException("Not authorized to view this email");
        }

        return user.getEmail();
    }

    public String ssn(User user, DataFetchingEnvironment env) {
        GraphQLContext context = env.getContext();
        String currentUserId = context.get("userId");

        // Only admins can see SSN
        if (!authService.isAdmin(currentUserId)) {
            throw new UnauthorizedException("Admin access required");
        }

        return user.getSsn();
    }
}
```

### Directive-Based Authorization

Define authorization directive:

```graphql
directive @auth(requires: Role = USER) on FIELD_DEFINITION

enum Role {
  USER
  ADMIN
  OWNER
}

type User {
  id: ID!
  name: String!
  email: String! @auth(requires: OWNER)
  ssn: String! @auth(requires: ADMIN)
  posts: [Post!]!
}
```

Implement directive:

```java
public class AuthDirectiveWiring implements SchemaDirectiveWiring {

    @Override
    public GraphQLFieldDefinition onField(
            SchemaDirectiveWiringEnvironment<GraphQLFieldDefinition> environment) {

        GraphQLFieldDefinition field = environment.getElement();
        GraphQLDirective directive = environment.getDirective();

        // Get required role from directive
        String requiredRole = (String) directive.getArgument("requires")
            .getValue();

        // Wrap original data fetcher with authorization check
        DataFetcher<?> originalFetcher = environment.getCodeRegistry()
            .getDataFetcher(environment.getFieldsContainer(), field);

        DataFetcher<?> authFetcher = (DataFetchingEnvironment env) -> {
            GraphQLContext context = env.getContext();
            String userId = context.get("userId");

            // Check authorization
            if (!hasRequiredRole(userId, requiredRole, env)) {
                throw new UnauthorizedException(
                    "Requires role: " + requiredRole
                );
            }

            // Call original fetcher if authorized
            return originalFetcher.get(env);
        };

        // Update code registry with new fetcher
        environment.getCodeRegistry().dataFetcher(
            FieldCoordinates.coordinates(
                environment.getFieldsContainer(),
                field
            ),
            authFetcher
        );

        return field;
    }

    private boolean hasRequiredRole(
            String userId,
            String requiredRole,
            DataFetchingEnvironment env) {

        GraphQLContext context = env.getContext();

        switch (requiredRole) {
            case "ADMIN":
                return context.get("roles").contains("ADMIN");

            case "OWNER":
                // Check if user owns the resource
                Object source = env.getSource();
                if (source instanceof User) {
                    return ((User) source).getId().equals(userId);
                }
                return false;

            case "USER":
                return userId != null;

            default:
                return false;
        }
    }
}
```

---

## Disabling Introspection in Production

Introspection reveals your entire schema:

```graphql
query {
  __schema {
    types {
      name
      fields {
        name
        type {
          name
        }
      }
    }
  }
}
```

This helps attackers understand your API structure.

### Disabling Introspection

```java
@Configuration
public class GraphQLConfig {

    @Value("${graphql.introspection.enabled:false}")
    private boolean introspectionEnabled;

    @Bean
    public GraphQL graphQL(GraphQLSchema schema) {
        GraphQL.Builder builder = GraphQL.newGraphQL(schema);

        if (!introspectionEnabled) {
            // Disable introspection queries
            builder.preparsedDocumentProvider((executionInput, computeFunction) -> {
                Document document = (Document) computeFunction.apply(executionInput);

                if (containsIntrospection(document)) {
                    throw new GraphQLException("Introspection is disabled");
                }

                return document;
            });
        }

        return builder.build();
    }

    private boolean containsIntrospection(Document document) {
        for (Definition<?> definition : document.getDefinitions()) {
            if (definition instanceof OperationDefinition) {
                OperationDefinition operation = (OperationDefinition) definition;

                if (hasIntrospectionFields(operation.getSelectionSet())) {
                    return true;
                }
            }
        }

        return false;
    }

    private boolean hasIntrospectionFields(SelectionSet selectionSet) {
        if (selectionSet == null) {
            return false;
        }

        for (Selection<?> selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;
                String fieldName = field.getName();

                // Check for introspection fields
                if (fieldName.equals("__schema") ||
                    fieldName.equals("__type")) {
                    return true;
                }

                // Recurse
                if (hasIntrospectionFields(field.getSelectionSet())) {
                    return true;
                }
            }
        }

        return false;
    }
}
```

### Alternative: Schema Publishing

Instead of runtime introspection, publish your schema separately:

```yaml
# application-prod.yml
graphql:
  introspection:
    enabled: false
  schema:
    publicUrl: https://schema-registry.example.com/api/v1/schema
```

---

## Query Whitelisting (Persisted Queries)

The ultimate defense: only allow pre-approved queries.

### How It Works

1. **Build time:** Extract all queries from client code
2. **Deploy time:** Upload query hashes to server
3. **Runtime:** Clients send query hash instead of full query

### Client Side

```graphql
# queries/GetUser.graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

Build tool generates hash:
```javascript
// Generated code
const GET_USER_HASH = "a4b3c2d1e5f6...";

// Client sends
fetch('/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    queryId: GET_USER_HASH,
    variables: { id: "123" }
  })
});
```

### Server Side

```java
@Component
public class PersistedQueryRegistry {

    private final Map<String, String> queryMap = new ConcurrentHashMap<>();

    @PostConstruct
    public void loadQueries() {
        // Load from file, database, or config service
        try {
            ObjectMapper mapper = new ObjectMapper();
            Map<String, String> queries = mapper.readValue(
                new File("persisted-queries.json"),
                new TypeReference<Map<String, String>>() {}
            );

            queryMap.putAll(queries);

            logger.info("Loaded {} persisted queries", queryMap.size());
        } catch (IOException e) {
            logger.error("Failed to load persisted queries", e);
        }
    }

    public String getQuery(String queryId) {
        String query = queryMap.get(queryId);

        if (query == null) {
            throw new UnknownQueryException(
                "Unknown query ID: " + queryId
            );
        }

        return query;
    }
}

@RestController
public class GraphQLController {

    @Autowired
    private GraphQL graphQL;

    @Autowired
    private PersistedQueryRegistry queryRegistry;

    @Value("${graphql.persisted-queries.enforced:false}")
    private boolean enforcePersistedQueries;

    @PostMapping("/graphql")
    public ResponseEntity<ExecutionResult> execute(
            @RequestBody GraphQLRequest request) {

        String query;

        if (request.getQueryId() != null) {
            // Persisted query
            query = queryRegistry.getQuery(request.getQueryId());

        } else if (request.getQuery() != null) {
            // Ad-hoc query

            if (enforcePersistedQueries) {
                throw new GraphQLException(
                    "Ad-hoc queries are not allowed. Use persisted queries."
                );
            }

            query = request.getQuery();

        } else {
            throw new GraphQLException("No query or queryId provided");
        }

        ExecutionInput input = ExecutionInput.newExecutionInput()
            .query(query)
            .variables(request.getVariables())
            .build();

        ExecutionResult result = graphQL.execute(input);

        return ResponseEntity.ok(result);
    }
}
```

### Benefits

- **100% protection** against malicious queries
- **Smaller request size** (hash vs full query)
- **Better caching** (can cache by hash)
- **Query analytics** (know exactly which queries are used)
- **Backwards compatibility** (can update queries server-side)

---

## Input Validation and Sanitization

### SQL Injection Prevention

GraphQL doesn't prevent SQL injection by itself:

```java
// VULNERABLE CODE
public List<User> searchUsers(String query, DataFetchingEnvironment env) {
    // DON'T DO THIS
    String sql = "SELECT * FROM users WHERE name LIKE '%" + query + "%'";
    return jdbcTemplate.query(sql, userRowMapper);
}
```

**Correct approach:**

```java
public List<User> searchUsers(String query, DataFetchingEnvironment env) {
    // Use parameterized queries
    String sql = "SELECT * FROM users WHERE name LIKE ?";
    return jdbcTemplate.query(sql, userRowMapper, "%" + query + "%");
}

// Or use JPA
public List<User> searchUsers(String query, DataFetchingEnvironment env) {
    return userRepository.findByNameContaining(query);
}
```

### Input Size Limiting

```java
@Component
public class InputValidationInstrumentation extends SimpleInstrumentation {

    private static final int MAX_INPUT_LENGTH = 10_000;

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        // Check query length
        String query = parameters.getQuery();
        if (query.length() > MAX_INPUT_LENGTH) {
            throw new GraphQLException("Query too large");
        }

        // Check variable values
        Map<String, Object> variables = parameters.getVariables();
        validateVariables(variables);

        return super.beginExecution(parameters);
    }

    private void validateVariables(Map<String, Object> variables) {
        for (Map.Entry<String, Object> entry : variables.entrySet()) {
            Object value = entry.getValue();

            if (value instanceof String) {
                String str = (String) value;
                if (str.length() > MAX_INPUT_LENGTH) {
                    throw new GraphQLException(
                        "Variable value too large: " + entry.getKey()
                    );
                }
            } else if (value instanceof Collection) {
                Collection<?> collection = (Collection<?>) value;
                if (collection.size() > 1000) {
                    throw new GraphQLException(
                        "Variable array too large: " + entry.getKey()
                    );
                }
            }
        }
    }
}
```

---

## Timeout Protection

Prevent queries from running forever:

```java
@Configuration
public class GraphQLConfig {

    @Bean
    public GraphQL graphQL(GraphQLSchema schema) {
        return GraphQL.newGraphQL(schema)
            .instrumentation(new TimeoutInstrumentation(5000))  // 5 seconds
            .build();
    }
}

public class TimeoutInstrumentation extends SimpleInstrumentation {

    private final long timeoutMillis;

    public TimeoutInstrumentation(long timeoutMillis) {
        this.timeoutMillis = timeoutMillis;
    }

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        // Create timeout future
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        ScheduledFuture<?> timeoutFuture = scheduler.schedule(() -> {
            throw new QueryTimeoutException("Query execution timeout");
        }, timeoutMillis, TimeUnit.MILLISECONDS);

        return new SimpleInstrumentationContext<ExecutionResult>() {
            @Override
            public void onCompleted(ExecutionResult result, Throwable t) {
                // Cancel timeout if query completes
                timeoutFuture.cancel(false);
                scheduler.shutdown();
            }
        };
    }
}
```

---

## Security Configuration Example

Putting it all together:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration {

    @Bean
    public GraphQL graphQL(
            GraphQLSchema schema,
            SecurityConfig securityConfig) {

        return GraphQL.newGraphQL(schema)
            .instrumentation(new ChainedInstrumentation(Arrays.asList(
                // Depth limiting
                new DepthLimitingInstrumentation(
                    securityConfig.getMaxDepth()
                ),

                // Complexity analysis
                new ComplexityLimitingInstrumentation(
                    securityConfig.getMaxComplexity(),
                    schema
                ),

                // Rate limiting
                new RateLimitingInstrumentation(
                    rateLimiter(),
                    costLimiter(),
                    complexityAnalyzer(schema)
                ),

                // Timeout protection
                new TimeoutInstrumentation(
                    securityConfig.getQueryTimeout()
                ),

                // Input validation
                new InputValidationInstrumentation()
            )))
            .build();
    }

    @Bean
    public SecurityConfig securityConfig() {
        return SecurityConfig.builder()
            .maxDepth(10)
            .maxComplexity(10_000)
            .queryTimeout(5000)
            .introspectionEnabled(false)
            .enforcePersistedQueries(true)
            .build();
    }
}
```

---

## Key Takeaways

1. **Depth limiting** - Prevent infinite nesting attacks
2. **Complexity analysis** - Calculate and limit query cost
3. **Rate limiting** - Protect against spam (request-based and cost-based)
4. **Authentication** - Verify user identity (JWT, sessions)
5. **Authorization** - Field-level and resource-level permissions
6. **Introspection control** - Disable in production
7. **Persisted queries** - Ultimate protection against malicious queries
8. **Input validation** - Prevent injection attacks
9. **Timeouts** - Prevent runaway queries
10. **Defense in depth** - Layer multiple protections

GraphQL's flexibility makes it powerful but also vulnerable. With these security mechanisms in place, you can safely expose GraphQL APIs to the world.

---

**Next:** [Chapter 18: Federation - Distributing the Graph →](18-federation.md)
**Previous:** [← Chapter 16: Subscriptions - Real-time Graphs](16-subscriptions.md)
