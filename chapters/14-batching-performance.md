# Chapter 14: Batching and Performance Optimization

> "GraphQL gives clients incredible power. With great power comes great responsibility—to keep your server from melting."

---

## The Performance Challenge

You've built a GraphQL server. It works. But then you deploy to production with real traffic, and you discover GraphQL's dirty secret:

**A single innocent-looking query can bring your server to its knees.**

```graphql
query {
  posts(limit: 100) {
    title
    author {
      name
      posts(limit: 100) {
        title
        comments(limit: 100) {
          text
          author {
            name
          }
        }
      }
    }
  }
}
```

**What just happened?**
- 100 posts
- Each post has an author with 100 posts
- Each of those posts has 100 comments
- Each comment has an author

**Total database queries:** 100 + (100 × 100) + (100 × 100 × 100) + (100 × 100 × 100) = **1,010,100 queries**

**Response time:** Your database explodes. 🔥

This chapter is about preventing that.

---

## Part 1: Request Batching

### The Concept

Instead of clients making separate HTTP requests for multiple queries:

```javascript
// Without batching: 3 HTTP requests
fetch('/graphql', { body: JSON.stringify({ query: query1 }) })
fetch('/graphql', { body: JSON.stringify({ query: query2 }) })
fetch('/graphql', { body: JSON.stringify({ query: query3 }) })
```

**Batch them into one:**

```javascript
// With batching: 1 HTTP request
fetch('/graphql', {
  body: JSON.stringify([
    { query: query1 },
    { query: query2 },
    { query: query3 }
  ])
})
```

### Server-Side Implementation

```java
@RestController
public class GraphQLController {
    @Autowired
    private GraphQL graphQL;

    @PostMapping("/graphql")
    public ResponseEntity<?> execute(@RequestBody String body) throws IOException {
        ObjectMapper mapper = new ObjectMapper();
        JsonNode jsonNode = mapper.readTree(body);

        if (jsonNode.isArray()) {
            // Batch request
            List<ExecutionResult> results = new ArrayList<>();

            for (JsonNode node : jsonNode) {
                GraphQLRequest request = mapper.treeToValue(node, GraphQLRequest.class);
                ExecutionResult result = executeQuery(request);
                results.add(result);
            }

            return ResponseEntity.ok(results);

        } else {
            // Single request
            GraphQLRequest request = mapper.treeToValue(jsonNode, GraphQLRequest.class);
            ExecutionResult result = executeQuery(request);
            return ResponseEntity.ok(result);
        }
    }

    private ExecutionResult executeQuery(GraphQLRequest request) {
        return graphQL.execute(
            ExecutionInput.newExecutionInput()
                .query(request.getQuery())
                .variables(request.getVariables())
                .build()
        );
    }
}
```

**Client request:**

```http
POST /graphql
Content-Type: application/json

[
  {
    "query": "{ user(id: \"1\") { name } }"
  },
  {
    "query": "{ post(id: \"10\") { title } }"
  },
  {
    "query": "{ settings { theme } }"
  }
]
```

**Server response:**

```json
[
  {
    "data": { "user": { "name": "Alice" } }
  },
  {
    "data": { "post": { "title": "GraphQL Batching" } }
  },
  {
    "data": { "settings": { "theme": "dark" } }
  }
]
```

### Benefits and Trade-offs

**Benefits:**
- **Fewer HTTP requests** - reduces network overhead
- **Reduced latency** - one round trip instead of multiple
- **Connection pooling** - fewer concurrent connections

**Trade-offs:**
- **All-or-nothing** - one slow query delays the entire batch
- **Increased payload size** - larger requests and responses
- **Harder to cache** - batched requests vary more than individual ones

**When to use:**
- Mobile apps making multiple queries on screen load
- Server-to-server GraphQL calls
- Initial page loads requiring multiple data sources

---

## Part 2: Database Query Optimization

### The N+1 Problem (Revisited)

We covered DataLoader in Chapter 11, but let's look at database-specific optimizations.

### JPA/Hibernate Entity Graphs

**Problem:** Lazy loading causes N+1:

```java
@Entity
public class Post {
    @Id
    private Long id;

    private String title;

    @ManyToOne(fetch = FetchType.LAZY)  // Lazy = N+1 problem
    private User author;

    @OneToMany(fetch = FetchType.LAZY)
    private List<Comment> comments;
}
```

**Solution 1: Entity Graph**

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    @EntityGraph(attributePaths = {"author", "comments"})
    List<Post> findAll();

    @EntityGraph(attributePaths = {"author"})
    Optional<Post> findWithAuthorById(Long id);
}
```

**Generated SQL:**

```sql
-- Without Entity Graph (N+1 problem)
SELECT * FROM posts;                    -- 1 query
SELECT * FROM users WHERE id = ?;       -- N queries (one per post)

-- With Entity Graph (optimized)
SELECT p.*, u.*
FROM posts p
LEFT JOIN users u ON p.author_id = u.id;  -- 1 query with JOIN
```

**Solution 2: Dynamic Entity Graph**

```java
@Service
public class PostService {
    @Autowired
    private EntityManager entityManager;

    public List<Post> getPosts(boolean includeAuthor, boolean includeComments) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> query = cb.createQuery(Post.class);
        Root<Post> root = query.from(Post.class);

        // Dynamically include joins based on query requirements
        if (includeAuthor) {
            root.fetch("author", JoinType.LEFT);
        }
        if (includeComments) {
            root.fetch("comments", JoinType.LEFT);
        }

        return entityManager.createQuery(query).getResultList();
    }
}
```

**GraphQL Integration:**

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {
    @Autowired
    private PostService postService;

    public List<Post> posts(DataFetchingEnvironment env) {
        // Inspect the query to see what fields are requested
        DataFetchingFieldSelectionSet selectionSet = env.getSelectionSet();

        boolean includeAuthor = selectionSet.contains("author");
        boolean includeComments = selectionSet.contains("comments");

        // Fetch only what's needed
        return postService.getPosts(includeAuthor, includeComments);
    }
}
```

**Result:** If query asks for `{ posts { title } }`, we don't join authors or comments. If it asks for `{ posts { title author { name } } }`, we join authors in a single query.

### Query Complexity Analysis

**Detect expensive queries before executing them:**

```java
@Component
public class QueryComplexityInstrumentation extends SimpleInstrumentation {
    private static final int MAX_COMPLEXITY = 1000;

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        Document document = parameters.getDocument();
        int complexity = calculateComplexity(document);

        if (complexity > MAX_COMPLEXITY) {
            throw new QueryTooComplexException(
                String.format("Query complexity %d exceeds maximum %d",
                             complexity, MAX_COMPLEXITY)
            );
        }

        return super.beginExecution(parameters);
    }

    private int calculateComplexity(Document document) {
        ComplexityCalculator calculator = new ComplexityCalculator();

        // Walk the query AST
        document.getDefinitions().forEach(def -> {
            if (def instanceof OperationDefinition) {
                OperationDefinition op = (OperationDefinition) def;
                calculator.visit(op.getSelectionSet());
            }
        });

        return calculator.getComplexity();
    }
}

class ComplexityCalculator {
    private int complexity = 0;

    public void visit(SelectionSet selectionSet) {
        for (Selection<?> selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;

                // Each field adds 1 to complexity
                complexity += 1;

                // Lists multiply complexity by estimated size
                if (isListField(field)) {
                    int limit = getListLimit(field);
                    complexity += limit;  // Conservative estimate
                }

                // Recurse into nested fields
                if (field.getSelectionSet() != null) {
                    visit(field.getSelectionSet());
                }
            }
        }
    }

    private boolean isListField(Field field) {
        // Check if field returns a list type
        // Implementation depends on your schema metadata
        return field.getName().equals("posts") ||
               field.getName().equals("comments");
    }

    private int getListLimit(Field field) {
        // Extract 'limit' argument if present
        Argument limitArg = field.getArguments().stream()
            .filter(arg -> arg.getName().equals("limit"))
            .findFirst()
            .orElse(null);

        if (limitArg != null && limitArg.getValue() instanceof IntValue) {
            return ((IntValue) limitArg.getValue()).getValue().intValue();
        }

        return 10;  // Default assumed limit
    }

    public int getComplexity() {
        return complexity;
    }
}
```

**Example:**

```graphql
query {
  posts(limit: 100) {        # complexity += 100
    title                    # complexity += 1
    author {                 # complexity += 1
      name                   # complexity += 1
      posts(limit: 50) {     # complexity += 50
        title                # complexity += 1
      }
    }
  }
}

# Total complexity: 100 + 1 + 1 + 1 + 50 + 1 = 154
```

If `MAX_COMPLEXITY = 1000`, this query is allowed. But our earlier 1 million query example would be rejected.

---

## Part 3: Async and Parallel Execution

### The Problem: Sequential Execution

**Naive implementation:**

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {
    @Autowired
    private UserService userService;

    @Autowired
    private PostService postService;

    @Autowired
    private SettingsService settingsService;

    public Dashboard dashboard(Long userId) {
        // Sequential execution - slow!
        User user = userService.getUser(userId);           // 100ms
        List<Post> posts = postService.getRecentPosts();   // 150ms
        Settings settings = settingsService.getSettings(); // 50ms

        return new Dashboard(user, posts, settings);
        // Total time: 100 + 150 + 50 = 300ms
    }
}
```

**These queries are independent—they could run in parallel!**

### Solution: CompletableFuture

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {
    @Autowired
    private UserService userService;

    @Autowired
    private PostService postService;

    @Autowired
    private SettingsService settingsService;

    public CompletableFuture<Dashboard> dashboard(Long userId) {
        // Start all queries in parallel
        CompletableFuture<User> userFuture =
            CompletableFuture.supplyAsync(() -> userService.getUser(userId));

        CompletableFuture<List<Post>> postsFuture =
            CompletableFuture.supplyAsync(() -> postService.getRecentPosts());

        CompletableFuture<Settings> settingsFuture =
            CompletableFuture.supplyAsync(() -> settingsService.getSettings());

        // Wait for all to complete
        return CompletableFuture.allOf(userFuture, postsFuture, settingsFuture)
            .thenApply(v -> new Dashboard(
                userFuture.join(),
                postsFuture.join(),
                settingsFuture.join()
            ));
        // Total time: max(100, 150, 50) = 150ms ✅
    }
}
```

**Performance gain:** 300ms → 150ms (2x faster!)

### GraphQL Java's Built-in Async Support

GraphQL Java executes resolvers asynchronously by default:

```java
@Component
public class UserResolver implements GraphQLResolver<User> {
    @Autowired
    private PostService postService;

    @Autowired
    private SettingsService settingsService;

    // Return CompletableFuture - GraphQL Java handles the rest
    public CompletableFuture<List<Post>> posts(User user) {
        return CompletableFuture.supplyAsync(() ->
            postService.getPostsByAuthorId(user.getId())
        );
    }

    public CompletableFuture<Settings> settings(User user) {
        return CompletableFuture.supplyAsync(() ->
            settingsService.getUserSettings(user.getId())
        );
    }
}
```

**Query:**

```graphql
query {
  user(id: "123") {
    name              # Sync
    posts { title }   # Async (CompletableFuture)
    settings { theme } # Async (CompletableFuture)
  }
}
```

**Execution:** `posts` and `settings` resolvers run in parallel automatically.

### Custom Executor for CPU-Bound Tasks

```java
@Configuration
public class AsyncConfig {

    @Bean(name = "graphQLExecutor")
    public Executor graphQLExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("graphql-");
        executor.initialize();
        return executor;
    }
}

@Component
public class QueryResolver implements GraphQLQueryResolver {
    @Autowired
    @Qualifier("graphQLExecutor")
    private Executor executor;

    public CompletableFuture<User> user(Long id) {
        return CompletableFuture.supplyAsync(
            () -> performExpensiveOperation(id),
            executor  // Use custom thread pool
        );
    }
}
```

---

## Part 4: Database Connection Pooling

### HikariCP Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: password
    hikari:
      maximum-pool-size: 20        # Max connections (tune based on load)
      minimum-idle: 5               # Min idle connections
      connection-timeout: 30000     # 30 seconds
      idle-timeout: 600000          # 10 minutes
      max-lifetime: 1800000         # 30 minutes
      leak-detection-threshold: 60000  # Warn if connection held > 60s
```

**Sizing the pool:**

Formula: `connections = ((core_count * 2) + effective_spindle_count)`

For a 4-core server with SSD: `(4 * 2) + 1 = 9 connections`

**Why not more?**
- Each connection consumes memory on the database server
- More connections ≠ better performance (context switching overhead)
- Better to queue requests than overload the database

### Monitoring Connection Pool

```java
@Component
public class ConnectionPoolMetrics {
    @Autowired
    private HikariDataSource dataSource;

    @Scheduled(fixedRate = 60000)  // Every minute
    public void logPoolStats() {
        HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();

        logger.info("Connection Pool Stats - " +
                   "Active: {}, Idle: {}, Waiting: {}, Total: {}",
                   pool.getActiveConnections(),
                   pool.getIdleConnections(),
                   pool.getThreadsAwaitingConnection(),
                   pool.getTotalConnections());

        if (pool.getThreadsAwaitingConnection() > 5) {
            logger.warn("High connection wait queue! Consider increasing pool size.");
        }
    }
}
```

---

## Part 5: Field-Level Performance Tracking

### Instrumentation for Timing

```java
@Component
public class PerformanceInstrumentation extends SimpleInstrumentation {
    private static final Logger logger = LoggerFactory.getLogger(PerformanceInstrumentation.class);

    @Override
    public InstrumentationContext<Object> beginFieldFetch(
            InstrumentationFieldFetchParameters parameters) {

        long startTime = System.currentTimeMillis();

        return new InstrumentationContext<Object>() {
            @Override
            public void onCompleted(Object result, Throwable t) {
                long duration = System.currentTimeMillis() - startTime;

                String path = parameters.getExecutionStepInfo().getPath().toString();
                String fieldName = parameters.getField().getName();

                if (duration > 100) {  // Log slow fields (>100ms)
                    logger.warn("Slow field: {} took {}ms", path, duration);
                }

                // Send to metrics system
                recordMetric("graphql.field.duration", duration,
                           "field", fieldName,
                           "path", path);
            }
        };
    }

    private void recordMetric(String name, long value, String... tags) {
        // Integration with Micrometer, Prometheus, etc.
        // meterRegistry.timer(name, tags).record(value, TimeUnit.MILLISECONDS);
    }
}
```

**Example log output:**

```
WARN - Slow field: /user/posts took 450ms
WARN - Slow field: /user/posts/2/comments took 320ms
```

This helps identify which resolvers need optimization.

### Query Tracing

Enable detailed tracing:

```java
@Bean
public GraphQL graphQL(GraphQLSchema schema) {
    return GraphQL.newGraphQL(schema)
        .instrumentation(new TracingInstrumentation())  // Built-in tracing
        .build();
}
```

**Response includes tracing data:**

```json
{
  "data": { ... },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2024-01-15T10:00:00.000Z",
      "endTime": "2024-01-15T10:00:01.234Z",
      "duration": 1234000000,
      "execution": {
        "resolvers": [
          {
            "path": ["user"],
            "parentType": "Query",
            "fieldName": "user",
            "returnType": "User",
            "startOffset": 50000,
            "duration": 100000000
          },
          {
            "path": ["user", "posts"],
            "parentType": "User",
            "fieldName": "posts",
            "returnType": "[Post]",
            "startOffset": 150000,
            "duration": 450000000
          }
        ]
      }
    }
  }
}
```

Clients can visualize this in dev tools to see which resolvers are slow.

---

## Part 6: Query Depth and Breadth Limiting

### Depth Limiting

Prevent deeply nested queries that explode complexity:

```java
@Bean
public GraphQL graphQL(GraphQLSchema schema) {
    return GraphQL.newGraphQL(schema)
        .instrumentation(new MaxQueryDepthInstrumentation(10))  // Max depth: 10
        .build();
}
```

**Rejected query:**

```graphql
query {
  user {           # depth 1
    posts {        # depth 2
      author {     # depth 3
        posts {    # depth 4
          author { # depth 5
            posts {  # depth 6
              # ... continues to depth 15 (rejected!)
            }
          }
        }
      }
    }
  }
}
```

**Error:**

```json
{
  "errors": [{
    "message": "Query depth exceeds maximum allowed depth of 10",
    "extensions": {
      "code": "QUERY_TOO_DEEP",
      "actualDepth": 15,
      "maxDepth": 10
    }
  }]
}
```

### Breadth Limiting (List Size Limits)

Enforce maximum list sizes:

```graphql
type Query {
  posts(limit: Int = 10): [Post!]!  # Default limit: 10
}
```

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {
    private static final int MAX_LIMIT = 100;

    public List<Post> posts(Integer limit) {
        // Enforce maximum
        int actualLimit = Math.min(limit != null ? limit : 10, MAX_LIMIT);

        return postRepository.findAll(PageRequest.of(0, actualLimit))
                            .getContent();
    }
}
```

**Rejected:**

```graphql
query {
  posts(limit: 10000) {  # Too large, capped at 100
    title
  }
}
```

---

## Part 7: Caching at Multiple Layers

### 1. DataLoader (Request-Scoped Cache)

```java
@Component
@Scope("request")
public class DataLoaders {
    @Autowired
    private UserRepository userRepository;

    private DataLoader<Long, User> userLoader;

    @PostConstruct
    public void init() {
        this.userLoader = DataLoader.newDataLoader(ids -> {
            List<User> users = userRepository.findAllById(ids);
            return CompletableFuture.completedFuture(users);
        });
    }

    public DataLoader<Long, User> getUserLoader() {
        return userLoader;
    }
}
```

### 2. Spring Cache (Persistent Cache)

```java
@Service
public class UserService {
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

### 3. Query Result Cache

Cache entire GraphQL responses:

```java
@Component
public class QueryCacheInstrumentation extends SimpleInstrumentation {
    @Autowired
    private CacheManager cacheManager;

    @Override
    public CompletableFuture<ExecutionResult> instrumentExecutionResult(
            ExecutionResult executionResult,
            InstrumentationExecutionParameters parameters) {

        String query = parameters.getQuery();
        Map<String, Object> variables = parameters.getVariables();

        String cacheKey = generateCacheKey(query, variables);

        // Store in cache
        if (executionResult.getErrors().isEmpty()) {
            cacheManager.getCache("query-results").put(cacheKey, executionResult);
        }

        return CompletableFuture.completedFuture(executionResult);
    }
}
```

---

## Part 8: Pagination Best Practices

### Cursor-Based Pagination

**Better than offset-based for large datasets:**

```graphql
type Query {
  posts(first: Int!, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

**Implementation:**

```java
@Component
public class QueryResolver implements GraphQLQueryResolver {

    public PostConnection posts(int first, String after) {
        // Decode cursor (base64 encoded ID)
        Long afterId = after != null ? decodeCursor(after) : 0L;

        // Fetch one extra to check if there's a next page
        List<Post> posts = postRepository.findByIdGreaterThan(
            afterId,
            PageRequest.of(0, first + 1)
        ).getContent();

        boolean hasNextPage = posts.size() > first;
        if (hasNextPage) {
            posts = posts.subList(0, first);  // Remove the extra
        }

        List<PostEdge> edges = posts.stream()
            .map(post -> new PostEdge(encodeCursor(post.getId()), post))
            .collect(Collectors.toList());

        String endCursor = edges.isEmpty() ? null :
                          edges.get(edges.size() - 1).getCursor();

        return new PostConnection(
            edges,
            new PageInfo(hasNextPage, endCursor)
        );
    }

    private String encodeCursor(Long id) {
        return Base64.getEncoder().encodeToString(id.toString().getBytes());
    }

    private Long decodeCursor(String cursor) {
        return Long.parseLong(new String(Base64.getDecoder().decode(cursor)));
    }
}
```

**Benefits:**
- **Stable** - adding/removing items doesn't break pagination
- **Efficient** - database can use indexed seeks
- **No duplicates** - offset-based pagination can show duplicates if data changes

---

## Part 9: Rate Limiting

### Per-User Rate Limiting

```java
@Component
public class RateLimitingInstrumentation extends SimpleInstrumentation {
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        String userId = extractUserId(parameters.getContext());

        RateLimiter limiter = limiters.computeIfAbsent(userId,
            k -> RateLimiter.create(100.0));  // 100 queries per second

        if (!limiter.tryAcquire()) {
            throw new RateLimitExceededException(
                "Rate limit exceeded: 100 queries per second"
            );
        }

        return super.beginExecution(parameters);
    }
}
```

### Cost-Based Rate Limiting

```java
@Component
public class CostBasedRateLimiter {
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    public void checkLimit(String userId, int queryCost) {
        TokenBucket bucket = buckets.computeIfAbsent(userId,
            k -> new TokenBucket(1000, 100));  // 1000 points, refill 100/sec

        if (!bucket.consume(queryCost)) {
            throw new RateLimitExceededException(
                String.format("Insufficient query budget. Cost: %d, Available: %d",
                             queryCost, bucket.getAvailable())
            );
        }
    }
}

class TokenBucket {
    private final int capacity;
    private final int refillRate;
    private int tokens;
    private long lastRefill;

    public TokenBucket(int capacity, int refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefill = System.currentTimeMillis();
    }

    public synchronized boolean consume(int cost) {
        refill();

        if (tokens >= cost) {
            tokens -= cost;
            return true;
        }
        return false;
    }

    public int getAvailable() {
        refill();
        return tokens;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsedSeconds = (now - lastRefill) / 1000;

        if (elapsedSeconds > 0) {
            tokens = Math.min(capacity, tokens + (int)(elapsedSeconds * refillRate));
            lastRefill = now;
        }
    }
}
```

---

## Part 10: Monitoring and Observability

### Metrics Collection with Micrometer

```java
@Configuration
public class MetricsConfig {
    @Bean
    public GraphQLInstrumentation metricsInstrumentation(MeterRegistry registry) {
        return new MetricsInstrumentation(registry);
    }
}

@Component
public class MetricsInstrumentation extends SimpleInstrumentation {
    private final MeterRegistry registry;

    public MetricsInstrumentation(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public InstrumentationContext<ExecutionResult> beginExecution(
            InstrumentationExecutionParameters parameters) {

        Timer.Sample sample = Timer.start(registry);

        return new InstrumentationContext<ExecutionResult>() {
            @Override
            public void onCompleted(ExecutionResult result, Throwable t) {
                sample.stop(Timer.builder("graphql.query.duration")
                    .tag("operation", getOperationName(parameters))
                    .tag("success", String.valueOf(t == null && result.getErrors().isEmpty()))
                    .register(registry));

                // Count errors
                if (!result.getErrors().isEmpty()) {
                    registry.counter("graphql.errors",
                        "operation", getOperationName(parameters)
                    ).increment(result.getErrors().size());
                }
            }
        };
    }

    private String getOperationName(InstrumentationExecutionParameters parameters) {
        return parameters.getOperation() != null ?
               parameters.getOperation() : "anonymous";
    }
}
```

**Exposed metrics (Prometheus format):**

```
graphql_query_duration_seconds{operation="getUser",success="true"} 0.150
graphql_query_duration_seconds{operation="getPosts",success="true"} 0.320
graphql_errors{operation="getUser"} 5
```

### Distributed Tracing (OpenTelemetry)

```java
@Component
public class TracingInstrumentation extends SimpleInstrumentation {
    @Autowired
    private Tracer tracer;

    @Override
    public InstrumentationContext<Object> beginFieldFetch(
            InstrumentationFieldFetchParameters parameters) {

        Span span = tracer.spanBuilder(parameters.getField().getName())
            .setAttribute("graphql.field", parameters.getField().getName())
            .setAttribute("graphql.path", parameters.getPath().toString())
            .startSpan();

        return new InstrumentationContext<Object>() {
            @Override
            public void onCompleted(Object result, Throwable t) {
                if (t != null) {
                    span.recordException(t);
                    span.setStatus(StatusCode.ERROR);
                }
                span.end();
            }
        };
    }
}
```

Traces show up in Jaeger/Zipkin with full resolver execution breakdown.

---

## Part 11: Load Testing

### JMeter GraphQL Test Plan

```xml
<HTTPSamplerProxy>
  <stringProp name="HTTPSampler.domain">localhost</stringProp>
  <stringProp name="HTTPSampler.port">8080</stringProp>
  <stringProp name="HTTPSampler.path">/graphql</stringProp>
  <stringProp name="HTTPSampler.method">POST</stringProp>
  <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
  <elementProp name="HTTPsampler.Arguments">
    <collectionProp name="Arguments.arguments">
      <elementProp name="" elementType="HTTPArgument">
        <stringProp name="Argument.value">
          {
            "query": "{ posts(limit: 10) { title author { name } } }"
          }
        </stringProp>
        <stringProp name="Argument.metadata">=</stringProp>
      </elementProp>
    </collectionProp>
  </elementProp>
  <stringProp name="HTTPSampler.contentEncoding">UTF-8</stringProp>
  <HeaderManager>
    <collectionProp>
      <elementProp>
        <stringProp name="Header.name">Content-Type</stringProp>
        <stringProp name="Header.value">application/json</stringProp>
      </elementProp>
    </collectionProp>
  </HeaderManager>
</HTTPSamplerProxy>
```

**Test scenarios:**
1. **Baseline:** 10 users, simple queries
2. **Stress test:** 100 users, complex queries
3. **Spike test:** Sudden jump from 10 to 500 users
4. **Endurance test:** 50 users for 1 hour

**Metrics to monitor:**
- Response time (p50, p95, p99)
- Throughput (requests/second)
- Error rate
- Database connection pool utilization
- CPU and memory usage

---

## Part 12: Summary: Performance Optimization Checklist

### Database Layer
- ✅ Use DataLoader for batching
- ✅ Enable JPA Entity Graphs
- ✅ Configure connection pool (HikariCP)
- ✅ Add database indexes on frequently queried fields
- ✅ Use read replicas for read-heavy workloads

### Application Layer
- ✅ Return CompletableFuture for async execution
- ✅ Use custom thread pools for CPU-bound tasks
- ✅ Implement field-level caching
- ✅ Cache query results where appropriate
- ✅ Use pagination (cursor-based preferred)

### Query Protection
- ✅ Limit query depth (max 10-15 levels)
- ✅ Limit list sizes (max 100-1000 items)
- ✅ Calculate query complexity
- ✅ Implement rate limiting (per-user, cost-based)
- ✅ Set timeouts on resolvers

### Monitoring
- ✅ Collect metrics (Micrometer + Prometheus)
- ✅ Enable distributed tracing (OpenTelemetry)
- ✅ Log slow queries (>100ms)
- ✅ Monitor connection pool stats
- ✅ Set up alerting for anomalies

### Testing
- ✅ Load test with realistic queries
- ✅ Stress test with complex queries
- ✅ Profile with production-like data volumes
- ✅ Test with slow/failing external services

---

## Real-World Performance Example

**Before optimization:**

```
Query: { posts(limit: 100) { title author { name } } }

Database queries: 101 (1 for posts + 100 for authors)
Response time: 1,250ms
Throughput: 8 req/sec
Database connections: 50/50 (exhausted)
```

**After optimization:**

```
Changes applied:
1. Added DataLoader for authors (batching)
2. Enabled Redis caching (5min TTL)
3. Configured HikariCP (20 connections)
4. Returned CompletableFuture from resolvers
5. Added query complexity limits

Results:
Database queries: 2 (1 for posts + 1 batched for authors)
Response time (first): 180ms
Response time (cached): 12ms
Throughput: 450 req/sec
Database connections: 8/20 (healthy)

Performance improvement: 50x throughput, 100x faster (cached)
```

---

**Key Takeaway:** GraphQL's flexibility is powerful but dangerous. Implement these optimizations proactively—before your production database melts. 🔥→❄️

---

**Next:** [Chapter 15: Mutations and Side Effects →](15-mutations.md)
