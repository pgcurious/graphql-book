# Chapter 21: Production Considerations

> "Getting GraphQL to work in development is the easy part. Running it reliably at scale is where the real engineering begins."

---

## From Development to Production

You've built a GraphQL server. It works beautifully on your laptop with a handful of test queries. But production is different:

- **Scale**: Thousands of concurrent users, not just you
- **Unknowns**: Clients will send queries you never imagined
- **Reliability**: 99.9% uptime is the baseline expectation
- **Performance**: Every millisecond matters
- **Security**: Attackers actively probe for weaknesses
- **Evolution**: Schema must change without breaking clients

This chapter covers what it takes to run GraphQL in production successfully.

---

## Monitoring and Observability

### What to Monitor

**Golden Signals:**

1. **Latency**: How long do queries take?
2. **Traffic**: How many requests per second?
3. **Errors**: What percentage of requests fail?
4. **Saturation**: How close to capacity are you?

### GraphQL-Specific Metrics

```java
@Component
public class GraphQLMetrics {

    private final MeterRegistry registry;

    public GraphQLMetrics(MeterRegistry registry) {
        this.registry = registry;
    }

    public void recordQuery(String operationName, long durationMs, boolean success) {
        Timer.builder("graphql.query.duration")
            .tag("operation", operationName)
            .tag("success", String.valueOf(success))
            .register(registry)
            .record(durationMs, TimeUnit.MILLISECONDS);

        Counter.builder("graphql.query.count")
            .tag("operation", operationName)
            .tag("success", String.valueOf(success))
            .register(registry)
            .increment();
    }

    public void recordResolverExecution(String fieldName, String parentType, long durationMs) {
        Timer.builder("graphql.resolver.duration")
            .tag("field", fieldName)
            .tag("parent", parentType)
            .register(registry)
            .record(durationMs, TimeUnit.MILLISECONDS);
    }

    public void recordComplexity(int complexity, int depth) {
        DistributionSummary.builder("graphql.query.complexity")
            .register(registry)
            .record(complexity);

        DistributionSummary.builder("graphql.query.depth")
            .register(registry)
            .record(depth);
    }
}
```

### Instrumenting Your GraphQL Server

```java
@Component
public class InstrumentedDataFetcherExceptionHandler implements DataFetcherExceptionHandler {

    @Autowired
    private GraphQLMetrics metrics;

    @Autowired
    private Logger logger;

    @Override
    public DataFetcherExceptionHandlerResult onException(DataFetcherExceptionHandlerParameters params) {
        Throwable exception = params.getException();
        SourceLocation location = params.getSourceLocation();
        ExecutionPath path = params.getPath();

        // Log with context
        logger.error("GraphQL resolver error at path {}: {}",
            path, exception.getMessage(), exception);

        // Record metric
        metrics.recordError(path.toString(), exception.getClass().getSimpleName());

        // Build error response
        GraphqlError error = GraphqlErrorBuilder.newError()
            .message(exception.getMessage())
            .location(location)
            .path(path)
            .errorType(ErrorType.DataFetchingException)
            .build();

        return DataFetcherExceptionHandlerResult.newResult()
            .error(error)
            .build();
    }
}
```

### Key Performance Indicators (KPIs)

Track these over time:

```java
// Average query execution time
SELECT percentile(duration, 50) as p50,
       percentile(duration, 95) as p95,
       percentile(duration, 99) as p99
FROM graphql_queries
WHERE timestamp > now() - interval '1 hour'
GROUP BY time(5m)
```

**Warning Thresholds:**
- P95 > 1 second → Investigate slow queries
- P99 > 5 seconds → Critical performance issue
- Error rate > 1% → Service degradation
- Query complexity trending up → Potential abuse

---

## Logging Best Practices

### What to Log

**DO log:**
- Query operation names (not full queries by default)
- Variables (sanitized for PII)
- Execution time
- Errors with full context
- User ID / request ID for tracing

**DON'T log:**
- Passwords or tokens in variables
- Full query text in high-traffic production (unless debugging)
- Personal Identifiable Information (PII) without consent

### Structured Logging

```java
@Component
public class GraphQLRequestLogger {

    private final Logger logger = LoggerFactory.getLogger(GraphQLRequestLogger.class);

    public void logRequest(ExecutionInput input, ExecutionResult result) {
        Map<String, Object> logData = new HashMap<>();

        logData.put("operationName", input.getOperationName());
        logData.put("executionId", input.getExecutionId());
        logData.put("duration", calculateDuration(input));

        // Sanitize variables
        Map<String, Object> sanitizedVars = sanitizeVariables(input.getVariables());
        logData.put("variables", sanitizedVars);

        // Error information
        if (result.getErrors() != null && !result.getErrors().isEmpty()) {
            logData.put("errorCount", result.getErrors().size());
            logData.put("errors", result.getErrors().stream()
                .map(GraphQLError::getMessage)
                .collect(Collectors.toList()));
        }

        // Context
        GraphQLContext context = input.getGraphQLContext();
        logData.put("userId", context.get("userId"));
        logData.put("correlationId", context.get("correlationId"));

        logger.info("GraphQL request executed: {}", logData);
    }

    private Map<String, Object> sanitizeVariables(Map<String, Object> variables) {
        Map<String, Object> sanitized = new HashMap<>(variables);

        // Redact sensitive fields
        List<String> sensitiveKeys = Arrays.asList("password", "token", "secret", "apiKey");

        sanitized.replaceAll((key, value) -> {
            if (sensitiveKeys.stream().anyMatch(k -> key.toLowerCase().contains(k))) {
                return "[REDACTED]";
            }
            return value;
        });

        return sanitized;
    }
}
```

### Correlation IDs for Distributed Tracing

```java
@Component
public class CorrelationIdInterceptor implements WebGraphQlInterceptor {

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        String correlationId = request.getHeaders().getFirst("X-Correlation-ID");

        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }

        request.configureExecutionInput((input, builder) ->
            builder.graphQLContext(ctx -> ctx.put("correlationId", correlationId))
                .build()
        );

        // Set in MDC for logging
        MDC.put("correlationId", correlationId);

        return chain.next(request)
            .doFinally(signal -> MDC.remove("correlationId"));
    }
}
```

**Usage across services:**

```
Client Request
  |
  +-- X-Correlation-ID: abc-123
      |
      +-- GraphQL Server (logs with abc-123)
          |
          +-- Database query (tagged with abc-123)
          +-- External API call (includes abc-123)
          +-- Cache lookup (tagged with abc-123)
```

Now you can trace a single request across your entire system.

---

## Performance Tuning

### JVM Tuning for GraphQL

**Heap Size:**

```bash
# Allocate sufficient heap
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -jar graphql-server.jar
```

**Why G1GC?**
- Better for large heaps (> 4GB)
- Predictable pause times
- Good for request/response workloads

**GC Logging:**

```bash
java -Xlog:gc*:file=gc.log:time,uptime:filecount=10,filesize=10M \
     -jar graphql-server.jar
```

Monitor for:
- Long GC pauses (> 200ms)
- Frequent full GCs
- Heap approaching max size

### Connection Pool Sizing

```yaml
# application.yml
spring:
  datasource:
    hikari:
      # Formula: connections = (core_count * 2) + effective_spindle_count
      # For 8-core machine with SSD: (8 * 2) + 1 = 17
      maximum-pool-size: 17
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

      # Prevent connection leaks
      leak-detection-threshold: 60000
```

**Why not just use 100 connections?**
- Database has finite resources
- More connections ≠ better performance
- Context switching overhead
- Memory per connection

### Thread Pool Configuration

```java
@Configuration
public class ExecutorConfig {

    @Bean
    public Executor dataFetcherExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // Core threads equal to CPU cores
        int processors = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(processors);

        // Max for burst traffic
        executor.setMaxPoolSize(processors * 2);

        // Queue size (too large = memory issues, too small = rejections)
        executor.setQueueCapacity(500);

        executor.setThreadNamePrefix("graphql-resolver-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();

        return executor;
    }
}
```

### Caching for Performance

**Field-level caching:**

```java
@Component
public class UserResolver {

    @Autowired
    private CacheManager cacheManager;

    @SchemaMapping
    @Cacheable(value = "users", key = "#user.id")
    public User user(@Argument String id) {
        return userService.getUser(id);
    }

    @SchemaMapping(typeName = "User", field = "posts")
    @Cacheable(value = "userPosts", key = "#user.id")
    public List<Post> posts(User user) {
        return postService.getPostsByUserId(user.getId());
    }
}
```

**Cache configuration:**

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager("users", "userPosts", "posts");
    }

    @Bean
    public Caffeine<Object, Object> caffeineConfig() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats();
    }
}
```

**When to cache:**
- Data that changes infrequently
- Expensive computations
- External API calls

**When NOT to cache:**
- User-specific data (unless keyed by user)
- Real-time data (stock prices, sports scores)
- Large datasets (memory pressure)

---

## Schema Management

### Versioning Strategy

**Option 1: Additive Changes Only (Recommended)**

```graphql
# v1
type User {
  id: ID!
  name: String!
}

# v2 (add new field, mark old as deprecated)
type User {
  id: ID!
  name: String! @deprecated(reason: "Use firstName and lastName instead")
  firstName: String!
  lastName: String!
}
```

**Benefits:**
- No breaking changes
- Gradual migration
- Backwards compatible

**Option 2: Schema Versioning**

```
/graphql/v1
/graphql/v2
```

**Problems:**
- Maintain multiple schemas
- Harder to evolve shared types
- Client confusion

**Best practice:** Use deprecation, not versioning.

### Breaking Change Detection

```java
@Component
public class SchemaChangeValidator {

    public List<BreakingChange> detectBreakingChanges(GraphQLSchema oldSchema,
                                                       GraphQLSchema newSchema) {
        List<BreakingChange> changes = new ArrayList<>();

        // Check for removed types
        oldSchema.getTypeMap().forEach((typeName, oldType) -> {
            GraphQLType newType = newSchema.getType(typeName);
            if (newType == null) {
                changes.add(new BreakingChange(
                    "Type removed: " + typeName,
                    Severity.CRITICAL
                ));
            }
        });

        // Check for removed fields
        oldSchema.getTypeMap().values().stream()
            .filter(type -> type instanceof GraphQLObjectType)
            .map(type -> (GraphQLObjectType) type)
            .forEach(oldType -> {
                GraphQLObjectType newType = (GraphQLObjectType) newSchema.getType(oldType.getName());
                if (newType != null) {
                    oldType.getFieldDefinitions().forEach(oldField -> {
                        GraphQLFieldDefinition newField = newType.getFieldDefinition(oldField.getName());
                        if (newField == null && !oldField.isDeprecated()) {
                            changes.add(new BreakingChange(
                                "Field removed: " + oldType.getName() + "." + oldField.getName(),
                                Severity.CRITICAL
                            ));
                        }
                    });
                }
            });

        // Check for argument changes
        // Check for type changes
        // etc.

        return changes;
    }
}
```

### Schema Registry

For federated or multi-service setups:

```java
@Service
public class SchemaRegistryClient {

    @Value("${schema.registry.url}")
    private String registryUrl;

    /**
     * Publish schema to registry
     */
    public void publishSchema(String serviceName, String schema, String version) {
        SchemaPublication publication = SchemaPublication.builder()
            .serviceName(serviceName)
            .schema(schema)
            .version(version)
            .timestamp(Instant.now())
            .build();

        restTemplate.postForObject(
            registryUrl + "/schemas",
            publication,
            SchemaRegistrationResponse.class
        );
    }

    /**
     * Check compatibility before deployment
     */
    public boolean isCompatible(String serviceName, String newSchema) {
        CompatibilityCheck check = CompatibilityCheck.builder()
            .serviceName(serviceName)
            .proposedSchema(newSchema)
            .build();

        CompatibilityResult result = restTemplate.postForObject(
            registryUrl + "/compatibility",
            check,
            CompatibilityResult.class
        );

        return result.isCompatible();
    }
}
```

### Deprecation Workflow

```graphql
type User {
  # Step 1: Add new field
  email: String!

  # Step 2: Deprecate old field with migration timeline
  emailAddress: String! @deprecated(
    reason: "Use 'email' instead. This field will be removed on 2024-06-01."
  )
}
```

**Communication plan:**
1. Announce deprecation 90 days in advance
2. Update documentation with migration guide
3. Monitor usage of deprecated fields
4. Send reminders to teams still using deprecated fields
5. Remove field after deadline

**Track deprecated field usage:**

```java
@Component
public class DeprecationTracker {

    private final MeterRegistry registry;

    public void trackDeprecatedFieldUsage(String typeName, String fieldName) {
        Counter.builder("graphql.deprecated.usage")
            .tag("type", typeName)
            .tag("field", fieldName)
            .register(registry)
            .increment();
    }
}
```

---

## Deployment Strategies

### Blue-Green Deployment

```
              Load Balancer
                   |
         +---------+---------+
         |                   |
    Blue (v1.0)         Green (v1.1)
    [Active]            [Idle]

# Deploy new version to Green
# Test Green
# Switch traffic to Green
# Keep Blue for quick rollback
```

**For GraphQL:**
- Both versions must support same schema (or compatible)
- Test with introspection queries
- Gradual traffic shift (10% → 50% → 100%)

### Canary Releases

```java
@Component
public class CanaryRouter {

    @Value("${canary.percentage}")
    private int canaryPercentage; // 5% of traffic

    public String selectBackend(HttpServletRequest request) {
        String userId = request.getHeader("X-User-ID");

        // Consistent hashing for same user
        int hash = Math.abs(userId.hashCode() % 100);

        if (hash < canaryPercentage) {
            return "canary-backend";
        } else {
            return "stable-backend";
        }
    }
}
```

**Canary checklist:**
1. Deploy to 5% of traffic
2. Monitor error rates, latency, CPU
3. If metrics look good, increase to 25%
4. Then 50%, then 100%
5. Rollback immediately if any issues

### Feature Flags

```java
@Component
public class FeatureFlags {

    @Autowired
    private FeatureFlagService flagService;

    public boolean isEnabled(String feature, String userId) {
        return flagService.check(feature, userId);
    }
}

@Controller
public class UserResolver {

    @Autowired
    private FeatureFlags featureFlags;

    @SchemaMapping
    public User user(@Argument String id, @ContextValue String userId) {
        User user = userService.getUser(id);

        // New field gated by feature flag
        if (featureFlags.isEnabled("user-preferences", userId)) {
            user.setPreferences(preferenceService.getPreferences(id));
        }

        return user;
    }
}
```

**Benefits:**
- Test in production with subset of users
- Instant rollback (flip flag)
- A/B testing
- Gradual rollout

---

## Security Hardening

### HTTPS Enforcement

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .requiresChannel()
            .anyRequest()
            .requiresSecure() // Force HTTPS
            .and()
            .headers()
            .httpStrictTransportSecurity()
            .maxAgeInSeconds(31536000) // 1 year
            .includeSubDomains(true);
    }
}
```

### Security Headers

```java
@Component
public class SecurityHeadersFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // Prevent clickjacking
        httpResponse.setHeader("X-Frame-Options", "DENY");

        // Prevent MIME sniffing
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");

        // XSS protection
        httpResponse.setHeader("X-XSS-Protection", "1; mode=block");

        // Content Security Policy
        httpResponse.setHeader("Content-Security-Policy",
            "default-src 'self'; script-src 'self' 'unsafe-inline'");

        chain.doFilter(request, response);
    }
}
```

### Rate Limiting in Production

```java
@Component
public class ProductionRateLimiter {

    private final LoadingCache<String, RateLimiter> limiters;

    public ProductionRateLimiter() {
        this.limiters = CacheBuilder.newBuilder()
            .maximumSize(10000)
            .expireAfterAccess(1, TimeUnit.HOURS)
            .build(new CacheLoader<String, RateLimiter>() {
                @Override
                public RateLimiter load(String key) {
                    // Tier-based rate limiting
                    if (isPremiumUser(key)) {
                        return RateLimiter.create(1000.0); // 1000 req/sec
                    } else {
                        return RateLimiter.create(100.0); // 100 req/sec
                    }
                }
            });
    }

    public boolean allowRequest(String userId, String ipAddress) {
        // Rate limit by user
        if (!limiters.getUnchecked(userId).tryAcquire()) {
            return false;
        }

        // Also rate limit by IP (prevent abuse)
        if (!limiters.getUnchecked("ip:" + ipAddress).tryAcquire()) {
            return false;
        }

        return true;
    }
}
```

### DDoS Protection

**Application-level:**

```java
@Component
public class DDoSProtection {

    private final ConcurrentHashMap<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();

    @Scheduled(fixedRate = 1000)
    public void resetCounters() {
        requestCounts.clear();
    }

    public boolean shouldBlock(String ipAddress) {
        AtomicInteger count = requestCounts.computeIfAbsent(ipAddress, k -> new AtomicInteger(0));

        // Block if > 100 requests per second from single IP
        return count.incrementAndGet() > 100;
    }
}
```

**Infrastructure-level:**
- CloudFlare for DDoS protection
- AWS Shield
- Rate limiting at load balancer
- Geographic blocking if needed

---

## Disaster Recovery

### Health Checks

```java
@Component
public class GraphQLHealthIndicator implements HealthIndicator {

    @Autowired
    private GraphQL graphQL;

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try {
            // Test database connection
            Connection conn = dataSource.getConnection();
            conn.close();

            // Test GraphQL schema
            ExecutionResult result = graphQL.execute("{__schema{queryType{name}}}");
            if (!result.getErrors().isEmpty()) {
                return Health.down()
                    .withDetail("error", "Schema introspection failed")
                    .build();
            }

            return Health.up()
                .withDetail("schema", "OK")
                .withDetail("database", "OK")
                .build();

        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

**Expose health endpoint:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

### Circuit Breakers

```java
@Service
public class ResilientExternalService {

    @Autowired
    private RestTemplate restTemplate;

    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    @Retry(name = "externalService")
    public ExternalData fetchData(String id) {
        return restTemplate.getForObject("/api/data/" + id, ExternalData.class);
    }

    public ExternalData fallback(String id, Exception e) {
        log.warn("External service call failed for id {}, using fallback", id, e);

        // Return cached data or default value
        return ExternalData.builder()
            .id(id)
            .data("Service temporarily unavailable")
            .cached(true)
            .build();
    }
}
```

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3

  retry:
    instances:
      externalService:
        maxAttempts: 3
        waitDuration: 1s
        exponentialBackoffMultiplier: 2
```

### Graceful Degradation

```java
@Component
public class DegradedModeService {

    private volatile boolean degradedMode = false;

    @Autowired
    private MetricRegistry metrics;

    /**
     * Enable degraded mode when system is under stress
     */
    public void enableDegradedMode() {
        degradedMode = true;
        log.warn("Entering degraded mode");
        metrics.counter("degraded.mode.enabled").inc();
    }

    public boolean isDegraded() {
        return degradedMode;
    }
}

@Controller
public class ResilientResolver {

    @Autowired
    private DegradedModeService degradedMode;

    @SchemaMapping
    public List<Post> posts(User user) {
        if (degradedMode.isDegraded()) {
            // Return cached or reduced data set
            return cacheService.getCachedPosts(user.getId(), 10); // Only 10 instead of all
        }

        return postService.getAllPosts(user.getId());
    }
}
```

### Backup Strategies

**Database backups:**

```bash
# Daily full backup
0 2 * * * pg_dump -U postgres -d graphql_db | gzip > /backups/full_$(date +\%Y\%m\%d).sql.gz

# Hourly incremental backup (WAL archiving)
archive_command = 'cp %p /backups/wal/%f'
```

**Schema backups:**

```java
@Scheduled(cron = "0 0 * * * *") // Every hour
public void backupSchema() {
    SchemaPrinter printer = new SchemaPrinter();
    String schema = printer.print(graphQLSchema);

    String filename = "schema_" + Instant.now().toString() + ".graphql";
    Files.write(Paths.get("/backups/schemas/", filename), schema.getBytes());
}
```

---

## Documentation

### Schema Documentation

```graphql
"""
Represents a user in the system.

Users can create posts, comment on posts, and follow other users.
"""
type User {
  """
  Unique identifier for the user.
  This ID is stable and will never change.
  """
  id: ID!

  """
  Display name of the user.
  Must be between 1 and 50 characters.
  """
  name: String!

  """
  User's email address.
  Used for notifications and account recovery.
  """
  email: String!

  """
  All posts created by this user.

  Results are paginated and sorted by creation date (newest first).

  Example:
  ```
  user(id: "123") {
    posts(limit: 10) {
      id
      title
    }
  }
  ```
  """
  posts(
    """Maximum number of posts to return (default: 20, max: 100)"""
    limit: Int = 20

    """Skip the first N posts (for pagination)"""
    offset: Int = 0
  ): [Post!]!
}
```

### Auto-Generated Documentation

```java
@Configuration
public class GraphQLDocumentation {

    @Bean
    public GraphQLSchema schemaWithDocs(GraphQLSchema schema) {
        // Generate markdown documentation from schema
        SchemaDocGenerator generator = new SchemaDocGenerator();
        String markdown = generator.generate(schema);

        // Save to docs directory
        try {
            Files.write(
                Paths.get("docs/schema.md"),
                markdown.getBytes()
            );
        } catch (IOException e) {
            log.error("Failed to write schema documentation", e);
        }

        return schema;
    }
}
```

### Example Queries in Documentation

```markdown
# User Queries

## Get user by ID

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    posts {
      id
      title
    }
  }
}
```

**Variables:**
```json
{
  "id": "user-123"
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "id": "user-123",
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        {
          "id": "post-456",
          "title": "My First Post"
        }
      ]
    }
  }
}
```
```

### GraphQL Playground in Production

**Enable only for authorized users:**

```java
@Configuration
public class PlaygroundConfig {

    @Value("${graphql.playground.enabled:false}")
    private boolean playgroundEnabled;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        if (playgroundEnabled) {
            http.authorizeRequests()
                .antMatchers("/graphiql").hasRole("ADMIN")
                .anyRequest().authenticated();
        } else {
            http.authorizeRequests()
                .antMatchers("/graphiql").denyAll();
        }

        return http.build();
    }
}
```

**Better: Separate environments**
- Development: GraphiQL/Playground enabled
- Staging: Enabled for internal users
- Production: Disabled (use separate admin portal if needed)

---

## Cost Analysis

### Infrastructure Costs

**Baseline costs for 100k daily active users:**

```
Application Servers:
  - 4x c5.2xlarge EC2 instances
  - $0.34/hour × 4 × 730 hours/month = $993/month

Database:
  - RDS PostgreSQL db.r5.xlarge
  - $0.48/hour × 730 hours = $350/month

Cache:
  - ElastiCache Redis cache.r5.large
  - $0.208/hour × 730 hours = $152/month

Load Balancer:
  - Application Load Balancer
  - $0.025/hour × 730 hours = $18/month
  - Data processing: ~$50/month

Monitoring:
  - CloudWatch/DataDog
  - $100/month

Total: ~$1,663/month
```

### Database Load Patterns

Monitor query costs:

```java
@Component
public class QueryCostAnalyzer {

    public void analyzeQueryCost(ExecutionResult result, long executionTimeMs) {
        int databaseQueries = getDatabaseQueryCount(result);
        int cacheHits = getCacheHitCount(result);

        double cost = calculateCost(databaseQueries, executionTimeMs);

        if (cost > EXPENSIVE_QUERY_THRESHOLD) {
            log.warn("Expensive query detected: {} db queries, {}ms execution, cost: ${}",
                databaseQueries, executionTimeMs, cost);
        }
    }

    private double calculateCost(int dbQueries, long timeMs) {
        // Rough cost estimate
        double dbCostPerQuery = 0.001; // $0.001 per query
        double cpuCostPerMs = 0.00001; // $0.00001 per ms

        return (dbQueries * dbCostPerQuery) + (timeMs * cpuCostPerMs);
    }
}
```

### Optimization Opportunities

**Before optimization:**
```
Query: user → posts → comments (N+1 problem)
- 1 query for user
- 1 query per post (×100 posts)
- 1 query per comment (×1000 comments)
Total: 1,101 database queries
Cost: $1.10 per request
```

**After DataLoader:**
```
Query: user → posts → comments (batched)
- 1 query for user
- 1 batched query for all posts
- 1 batched query for all comments
Total: 3 database queries
Cost: $0.003 per request
```

**Savings: 99.7% reduction in database queries**

---

## Team Organization

### Schema Ownership

```
                 GraphQL Gateway
                       |
        +--------------+--------------+
        |              |              |
    User Service   Post Service  Comment Service
        |              |              |
    User Team      Post Team     Comment Team
```

**Ownership model:**
- Each team owns their schema portion
- Teams can evolve their schema independently
- Central team reviews for consistency
- Automated compatibility checks in CI/CD

### Review Process

```yaml
# .github/CODEOWNERS
schema/user.graphql @user-team
schema/post.graphql @post-team
schema/comment.graphql @comment-team

# All schema changes require GraphQL guild review
schema/ @graphql-guild
```

**Schema change checklist:**
- [ ] Schema compiles without errors
- [ ] No breaking changes (or approved migration plan)
- [ ] Documentation added for new fields
- [ ] Examples provided
- [ ] Tests updated
- [ ] Performance impact assessed
- [ ] Backwards compatibility verified
- [ ] Reviewed by GraphQL guild

### Breaking Change Policy

**Severity levels:**

1. **Critical Breaking Change** (Forbidden)
   - Removing a type
   - Removing a field (non-deprecated)
   - Changing field type incompatibly
   - Making nullable field non-null

2. **Managed Breaking Change** (Requires approval)
   - Removing deprecated field (after notice period)
   - Changing enum values
   - Renaming (must provide alias)

3. **Non-Breaking Change** (Allowed)
   - Adding new type
   - Adding new field
   - Adding new optional argument
   - Deprecating field

### On-Call Procedures

**Runbook for common issues:**

```markdown
# GraphQL Service Runbook

## High Error Rate

1. Check Grafana dashboard: http://grafana.company.com/graphql
2. Look for error spike in specific operation
3. Query logs: `kubectl logs -l app=graphql --tail=1000 | grep ERROR`
4. If specific query: Block query pattern via rate limiter
5. If database issue: Check connection pool, slow query log
6. Escalate to DBA if database problem

## High Latency

1. Check p95/p99 latency in dashboard
2. Identify slow operations: `SELECT operation, AVG(duration) FROM metrics WHERE timestamp > now() - 5m GROUP BY operation`
3. Check database slow query log
4. Check external API latency
5. Enable query complexity logging
6. Consider enabling degraded mode if critical

## Memory Leak

1. Heap dump: `jcmd <pid> GC.heap_dump /tmp/heap.hprof`
2. Analyze with VisualVM or Eclipse MAT
3. Check for connection leaks: `SELECT count(*) FROM pg_stat_activity`
4. Check cache size: `CACHE_SIZE` metric in Grafana
5. Restart pod if necessary: `kubectl rollout restart deployment/graphql`

## Database Connection Exhaustion

1. Check active connections: `SELECT count(*) FROM pg_stat_activity WHERE datname='graphql_db'`
2. Kill long-running queries: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle in transaction' AND query_start < now() - interval '5 minutes'`
3. Increase connection pool temporarily if needed
4. Check for N+1 queries
5. Review recent deployments
```

---

## Key Takeaways

1. **Monitor everything** - Latency, errors, saturation, and GraphQL-specific metrics

2. **Log with context** - Correlation IDs, structured logging, PII sanitization

3. **Tune for production** - JVM settings, connection pools, thread pools optimized for load

4. **Manage schema evolution** - Deprecation over versioning, breaking change detection

5. **Deploy safely** - Blue-green, canary releases, feature flags for gradual rollout

6. **Harden security** - HTTPS, security headers, rate limiting, DDoS protection

7. **Plan for disasters** - Health checks, circuit breakers, graceful degradation, backups

8. **Document thoroughly** - Schema docs, example queries, runbooks

9. **Analyze costs** - Infrastructure, database queries, optimization opportunities

10. **Organize teams** - Clear ownership, review processes, on-call procedures

---

## Production Readiness Checklist

Before going to production:

**Observability:**
- [ ] Metrics collection configured (Prometheus/DataDog/CloudWatch)
- [ ] Structured logging in place
- [ ] Distributed tracing configured (Jaeger/Zipkin)
- [ ] Dashboards created for key metrics
- [ ] Alerts configured for critical thresholds

**Performance:**
- [ ] Load testing completed (target: 2x expected peak traffic)
- [ ] Database connection pool sized appropriately
- [ ] JVM tuning applied
- [ ] Caching strategy implemented
- [ ] Query complexity limits enforced

**Security:**
- [ ] HTTPS enforced
- [ ] Authentication/authorization in place
- [ ] Rate limiting configured
- [ ] Query depth/complexity limits set
- [ ] Security headers configured
- [ ] PII handling reviewed

**Reliability:**
- [ ] Health check endpoint implemented
- [ ] Circuit breakers for external dependencies
- [ ] Graceful degradation strategy
- [ ] Backup and restore procedures tested
- [ ] Disaster recovery plan documented

**Operations:**
- [ ] Schema versioning strategy defined
- [ ] Breaking change policy established
- [ ] Deployment process documented
- [ ] Runbooks created
- [ ] On-call rotation established
- [ ] Schema documentation complete

**Compliance:**
- [ ] Data retention policies implemented
- [ ] GDPR/privacy requirements met
- [ ] Audit logging for sensitive operations
- [ ] Access controls reviewed

---

**You've learned GraphQL from first principles. Now you're ready to build it, run it, and scale it in production.**

**Previous:** [← Chapter 20: Client Implementation Patterns](20-client-patterns.md)

**[Back to Table of Contents](../README.md)**
