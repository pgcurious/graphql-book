# Chapter 12: Caching Strategies - Field-Level vs Query-Level

> "In GraphQL, every query is unique. In caching, every unique thing is a cache miss. This is the paradox we must solve."

---

## The Caching Challenge in GraphQL

You've solved the N+1 problem with DataLoader. Your GraphQL server is humming along nicely. But then you notice something: you're hitting the database on every single request, even for data that rarely changes.

In REST, caching is straightforward:
```
GET /api/users/123 → Cache for 1 hour
GET /api/posts → Cache for 5 minutes
```

Each URL is a cache key. HTTP caching works out of the box. CDNs understand it. Browsers cache it automatically.

But in GraphQL, **every query goes to the same endpoint**:

```
POST /graphql
```

And every query can be different:

```graphql
# Query 1
query {
  user(id: "123") {
    name
  }
}

# Query 2 - same user, different fields
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}

# Query 3 - same fields, different user
query {
  user(id: "456") {
    name
  }
}
```

All three queries hit `POST /graphql`. Traditional HTTP caching fails completely.

**The question:** How do we cache dynamic, client-specified queries?

---

## Part 1: Understanding GraphQL's Caching Problem

### Why Traditional HTTP Caching Fails

**Problem 1: POST requests aren't cached by default**

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{"query": "{ user(id: \"123\") { name } }"}
```

- Browsers don't cache POST requests
- CDNs typically don't cache POST requests
- Proxies skip POST requests
- All HTTP caching infrastructure is built for GET

**Problem 2: Same endpoint, infinite variations**

Even if we cached POST requests, the cache key would be:
- URL: `/graphql` (always the same)
- Body: The entire query string (always different)

Two queries asking for the same user with slightly different fields = two different cache keys = cache miss.

**Problem 3: Field-level granularity**

```graphql
query {
  user(id: "123") {
    name          # Changes rarely (cache for 1 hour)
    email         # Changes occasionally (cache for 5 minutes)
    onlineStatus  # Changes constantly (cache for 10 seconds)
  }
}
```

How do you cache a response that contains data with different cache lifetimes?

### The Trade-offs We'll Navigate

| Strategy | Granularity | Hit Rate | Complexity | Where Used |
|----------|-------------|----------|------------|------------|
| Full Query Caching | Entire response | Low | Low | Simple apps |
| Field-Level Caching | Per field | High | Medium | Server-side |
| Normalized Caching | Per entity | Very High | High | Clients (Apollo, Relay) |
| Persisted Queries | Static queries | High | Medium | Production apps |
| CDN Caching | GET queries | Medium | Low | Public data |

Each strategy has its place. Let's build understanding from first principles.

---

## Part 2: Server-Side Caching Strategies

### Strategy 1: Full Query Response Caching

**Idea:** Cache the entire GraphQL response using the query string as the key.

```java
@Service
public class GraphQLExecutor {
    @Autowired
    private GraphQL graphQL;

    @Autowired
    private CacheManager cacheManager;

    public ExecutionResult execute(String query, Map<String, Object> variables) {
        String cacheKey = generateCacheKey(query, variables);

        // Check cache first
        ExecutionResult cached = cacheManager.get("graphql-responses", cacheKey);
        if (cached != null) {
            return cached;
        }

        // Execute query
        ExecutionResult result = graphQL.execute(
            ExecutionInput.newExecutionInput()
                .query(query)
                .variables(variables)
                .build()
        );

        // Cache the result
        cacheManager.put("graphql-responses", cacheKey, result);

        return result;
    }

    private String generateCacheKey(String query, Map<String, Object> variables) {
        // Normalize the query (remove whitespace, comments)
        String normalizedQuery = normalizeQuery(query);

        // Include variables in the key
        String variablesJson = new ObjectMapper().writeValueAsString(variables);

        // Generate hash
        return DigestUtils.sha256Hex(normalizedQuery + variablesJson);
    }

    private String normalizeQuery(String query) {
        // Remove extra whitespace and comments
        return query.replaceAll("\\s+", " ")
                   .replaceAll("#[^\\n]*", "")
                   .trim();
    }
}
```

**Pros:**
- Simple to implement
- Works with any caching backend (Redis, Caffeine, Memcached)
- Low overhead per request

**Cons:**
- **Low cache hit rate** - even tiny query differences cause cache misses
- **No field-level granularity** - can't cache different fields with different TTLs
- **Wasted cache memory** - storing duplicate data

**When to use:** Simple applications with limited query variations (mobile apps with fixed queries).

### Strategy 2: Field-Level Caching

**Idea:** Cache individual field values, not entire responses.

This requires hooking into the GraphQL execution engine at the resolver level.

```java
@Component
public class CachingInstrumentation extends SimpleInstrumentation {
    @Autowired
    private CacheManager cacheManager;

    @Override
    public InstrumentationContext<Object> beginFieldFetch(
            InstrumentationFieldFetchParameters parameters) {

        GraphQLFieldDefinition fieldDef = parameters.getField();
        Object source = parameters.getSource();

        // Check for @cacheControl directive
        CacheDirective cacheDirective = getCacheDirective(fieldDef);
        if (cacheDirective == null) {
            return super.beginFieldFetch(parameters);
        }

        // Generate cache key based on field path
        String cacheKey = generateFieldCacheKey(parameters);

        // Try to get from cache
        Object cachedValue = cacheManager.get("field-cache", cacheKey);
        if (cachedValue != null) {
            // Return cached value, skip resolver execution
            return new CachedFieldContext(cachedValue);
        }

        return new InstrumentationContext<Object>() {
            @Override
            public void onCompleted(Object result, Throwable t) {
                if (t == null && result != null) {
                    // Cache the result with TTL from directive
                    cacheManager.put("field-cache", cacheKey, result,
                                   cacheDirective.maxAge());
                }
            }
        };
    }

    private String generateFieldCacheKey(InstrumentationFieldFetchParameters params) {
        // Key format: TypeName.fieldName:parentId:args
        StringBuilder key = new StringBuilder();

        // Type and field
        key.append(params.getExecutionStepInfo().getParent().getType().getName());
        key.append(".");
        key.append(params.getField().getName());

        // Parent object ID (if available)
        Object source = params.getSource();
        if (source != null) {
            String id = extractId(source);
            if (id != null) {
                key.append(":").append(id);
            }
        }

        // Arguments
        Map<String, Object> args = params.getArguments();
        if (!args.isEmpty()) {
            String argsJson = new ObjectMapper().writeValueAsString(args);
            key.append(":").append(DigestUtils.md5Hex(argsJson));
        }

        return key.toString();
    }
}
```

**Schema with cache directives:**

```graphql
directive @cacheControl(
  maxAge: Int!
  scope: CacheScope = PUBLIC
) on FIELD_DEFINITION | OBJECT

enum CacheScope {
  PUBLIC
  PRIVATE
}

type User @cacheControl(maxAge: 3600) {
  id: ID!
  name: String!
  email: String! @cacheControl(maxAge: 300)
  onlineStatus: Boolean! @cacheControl(maxAge: 10)
  posts: [Post!]! @cacheControl(maxAge: 60)
}
```

**Cache key examples:**

```
User.name:123 → "Alice"
User.email:123 → "alice@example.com"
User.onlineStatus:123 → true
User.posts:123 → [Post@456, Post@789]
Post.title:456 → "GraphQL Caching"
```

**Pros:**
- **High cache hit rate** - individual fields cached independently
- **Fine-grained TTLs** - different lifetimes per field
- **Efficient memory** - shared data across queries

**Cons:**
- **Complex implementation** - requires custom instrumentation
- **Cache key management** - tricky to get right
- **Invalidation complexity** - which keys to clear on updates?

**When to use:** Production servers with high read:write ratios and diverse queries.

### Strategy 3: DataLoader as a Per-Request Cache

DataLoader isn't just for batching—it's also a **per-request cache**.

```java
@Component
@Scope("request")
public class UserDataLoader {
    private final DataLoader<Long, User> loader;
    private final UserRepository userRepository;

    public UserDataLoader(UserRepository userRepository) {
        this.userRepository = userRepository;

        this.loader = DataLoader.newDataLoader(ids -> {
            List<User> users = userRepository.findAllById(ids);
            // This is called ONCE per request, even if user is requested 100 times
            return CompletableFuture.completedFuture(users);
        });
    }

    public CompletableFuture<User> load(Long id) {
        // First call: loads from database
        // Subsequent calls in same request: returns cached value
        return loader.load(id);
    }
}
```

**Within a single request:**

```graphql
query {
  post(id: "1") {
    author { name }  # Loads user 123
  }

  comment(id: "5") {
    author { name }  # User 123 already in DataLoader cache!
  }

  recentPosts {
    author { name }  # If any post by user 123, cached!
  }
}
```

**All three `author` resolvers for user 123 hit the cache within the same request.**

**Pros:**
- **Automatic** - no explicit caching code in resolvers
- **Request-scoped** - no stale data across requests
- **Combines with batching** - solve N+1 AND caching together

**Cons:**
- **Request-scoped only** - doesn't help across requests
- **Cleared after each request** - no persistent caching

**When to use:** Always. This should be your baseline.

### Strategy 4: Redis Integration

For persistent caching across requests:

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withCacheConfiguration("users",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("posts",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(10)))
            .build();
    }
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearUserCache() {
        // Called when bulk operations happen
    }
}
```

**Integration with resolvers:**

```java
@Component
public class UserResolver implements GraphQLResolver<User> {
    @Autowired
    private PostService postService;  // Uses @Cacheable

    public List<Post> posts(User user) {
        // postService methods hit Redis cache
        return postService.getPostsByAuthorId(user.getId());
    }
}
```

**Pros:**
- **Persistent cache** - survives restarts
- **Shared across servers** - multiple instances use same cache
- **Spring Cache annotations** - minimal code changes

**Cons:**
- **Network latency** - Redis call overhead (usually 1-3ms)
- **Serialization cost** - converting objects to/from JSON
- **Operational complexity** - Redis cluster management

**When to use:** Production deployments with multiple server instances.

---

## Part 3: Client-Side Caching

### The Normalized Cache (Apollo Client, Relay)

Clients like Apollo Client implement a **normalized cache** - the gold standard for GraphQL caching.

**Concept:** Store data by entity type and ID, not by query.

**Instead of caching by query:**

```javascript
// Query 1
cache["{ user(id: 123) { name } }"] = { user: { name: "Alice" } }

// Query 2 (cache miss!)
cache["{ user(id: 123) { email } }"] = { user: { email: "alice@example.com" } }
```

**Normalize by entity:**

```javascript
cache["User:123"] = {
  __typename: "User",
  id: "123",
  name: "Alice",
  email: "alice@example.com",
  posts: [{ __ref: "Post:456" }, { __ref: "Post:789" }]
}

cache["Post:456"] = {
  __typename: "Post",
  id: "456",
  title: "GraphQL Caching"
}
```

**Now both queries hit the cache:**

```graphql
# Query 1 - loads from server, populates cache["User:123"].name
query {
  user(id: "123") {
    name
  }
}

# Query 2 - cache hit! Reads cache["User:123"].email
# If email is missing, fetches only the email field
query {
  user(id: "123") {
    name  # cache hit
    email # cache hit (if previously fetched)
  }
}
```

### How It Works

**1. Response normalization:**

When a GraphQL response arrives, the client extracts all objects:

```javascript
// Server response
{
  "data": {
    "user": {
      "__typename": "User",
      "id": "123",
      "name": "Alice",
      "posts": [
        { "__typename": "Post", "id": "456", "title": "Caching" }
      ]
    }
  }
}

// Normalized cache storage
{
  "User:123": {
    "__typename": "User",
    "id": "123",
    "name": "Alice",
    "posts": [{ "__ref": "Post:456" }]
  },
  "Post:456": {
    "__typename": "Post",
    "id": "456",
    "title": "Caching"
  }
}
```

**2. Query reading:**

When executing a query, the client reads from the cache:

```javascript
// Query
query {
  user(id: "123") {
    name
    posts {
      title
    }
  }
}

// Cache read process
1. Read User:123 → { name: "Alice", posts: [{ __ref: "Post:456" }] }
2. Follow reference to Post:456 → { title: "Caching" }
3. Assemble result: { user: { name: "Alice", posts: [{ title: "Caching" }] } }
```

**3. Cache updates after mutations:**

```graphql
mutation {
  updateUser(id: "123", name: "Alice Smith") {
    id
    name
  }
}

# Response automatically updates cache["User:123"].name
# All queries reading User:123 now see the new name!
```

**Server-side hint for clients:**

Ensure your schema always returns `id` and `__typename`:

```graphql
type User {
  id: ID!  # Required for normalization
  name: String!
}

type Post {
  id: ID!  # Required for normalization
  title: String!
}
```

Clients use `__typename:id` as the cache key.

---

## Part 4: Automatic Persisted Queries (APQ)

**Problem:** GraphQL queries can be large (kilobytes). Sending them on every request wastes bandwidth.

**Solution:** Send a hash instead of the full query.

### The APQ Protocol

**First request (query not seen before):**

```http
POST /graphql
{
  "query": null,
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "ecf4edb46db40b5132295c0291d62fb65d6759a9eedfa4d5d612dd5ec54a6b38"
    }
  }
}
```

**Server response (hash not found):**

```json
{
  "errors": [{
    "message": "PersistedQueryNotFound",
    "extensions": { "code": "PERSISTED_QUERY_NOT_FOUND" }
  }]
}
```

**Client retry (with full query):**

```http
POST /graphql
{
  "query": "{ user(id: \"123\") { name email } }",
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "ecf4edb46db40b5132295c0291d62fb65d6759a9eedfa4d5d612dd5ec54a6b38"
    }
  }
}
```

**Server:**
1. Executes the query
2. **Stores** `hash → query` mapping in cache
3. Returns result

**Subsequent requests:**

```http
POST /graphql
{
  "query": null,
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "ecf4edb46db40b5132295c0291d62fb65d6759a9eedfa4d5d612dd5ec54a6b38"
    }
  }
}
```

**Server:**
1. Looks up hash in cache
2. Finds full query
3. Executes and returns result

**Bandwidth savings:**
- Full query: ~500 bytes - 2KB
- Hash: 64 bytes
- **Savings: 90-95% on repeated queries**

### Java Implementation

```java
@Component
public class PersistedQuerySupport {
    @Autowired
    private CacheManager cacheManager;

    public String resolveQuery(String query, PersistedQueryExtension extension) {
        if (extension == null) {
            return query;  // Normal query
        }

        String hash = extension.getSha256Hash();

        // Try to get query from cache
        String cachedQuery = cacheManager.get("persisted-queries", hash);
        if (cachedQuery != null) {
            return cachedQuery;
        }

        // Not in cache
        if (query == null) {
            throw new PersistedQueryNotFoundException(hash);
        }

        // Verify hash matches query
        String computedHash = DigestUtils.sha256Hex(query);
        if (!computedHash.equals(hash)) {
            throw new PersistedQueryHashMismatchException();
        }

        // Store for future requests
        cacheManager.put("persisted-queries", hash, query);

        return query;
    }
}

@RestController
public class GraphQLController {
    @Autowired
    private PersistedQuerySupport persistedQuerySupport;

    @Autowired
    private GraphQL graphQL;

    @PostMapping("/graphql")
    public ResponseEntity<ExecutionResult> execute(@RequestBody GraphQLRequest request) {
        try {
            // Resolve query (from hash or full query)
            String query = persistedQuerySupport.resolveQuery(
                request.getQuery(),
                request.getExtensions().getPersistedQuery()
            );

            ExecutionResult result = graphQL.execute(
                ExecutionInput.newExecutionInput()
                    .query(query)
                    .variables(request.getVariables())
                    .build()
            );

            return ResponseEntity.ok(result);

        } catch (PersistedQueryNotFoundException e) {
            return ResponseEntity.ok(
                ExecutionResult.newExecutionResult()
                    .addError(GraphQLError.newError()
                        .message("PersistedQueryNotFound")
                        .extensions(Map.of("code", "PERSISTED_QUERY_NOT_FOUND"))
                        .build())
                    .build()
            );
        }
    }
}
```

**Benefits:**
- **Reduced bandwidth** - especially important for mobile
- **Reduced parsing time** - server can cache parsed AST by hash
- **Security** - can whitelist known hashes (see Chapter 17)

---

## Part 5: Making GraphQL Cacheable by CDNs

### Problem: POST Requests and CDNs

CDNs (CloudFront, Fastly, Cloudflare) don't cache POST requests by default.

**Solutions:**

### Solution 1: GET Requests with Query in URL

```http
GET /graphql?query={user(id:"123"){name}}&variables={}
```

**Pros:**
- Standard HTTP caching works
- CDN caching works out of the box
- Browser caching automatic

**Cons:**
- **URL length limits** - queries can exceed 2KB, hitting limits
- **No request body** - limits query size
- **Logged in URLs** - queries appear in access logs (potential PII leak)
- **Cache pollution** - every unique query is a cache key

**When to use:** Small, public queries (homepage data, public profiles).

### Solution 2: GET with Persisted Query Hashes

Combine APQ with GET requests:

```http
GET /graphql?extensions={"persistedQuery":{"version":1,"sha256Hash":"ecf4e..."}}
```

**Pros:**
- **Short URLs** - hash is always 64 bytes
- **CDN cacheable** - stable cache keys
- **High hit rate** - mobile apps use same queries

**Cons:**
- Requires APQ support
- Clients must pre-register queries

**When to use:** Production mobile apps, public data.

### Solution 3: Custom Cache-Control Headers

Even with POST, you can hint to CDNs:

```java
@RestController
public class GraphQLController {

    @PostMapping("/graphql")
    public ResponseEntity<ExecutionResult> execute(@RequestBody GraphQLRequest request) {
        ExecutionResult result = graphQL.execute(request.getQuery());

        // Calculate cache hints from query
        CacheHints hints = calculateCacheHints(result);

        HttpHeaders headers = new HttpHeaders();
        if (hints.isCacheable()) {
            headers.setCacheControl(
                CacheControl.maxAge(hints.getMaxAge(), TimeUnit.SECONDS)
                    .cachePublic()
            );
        } else {
            headers.setCacheControl(CacheControl.noStore());
        }

        return ResponseEntity.ok()
            .headers(headers)
            .body(result);
    }

    private CacheHints calculateCacheHints(ExecutionResult result) {
        // Extract cache hints from @cacheControl directives
        // Find the minimum maxAge across all fields
        // If any field is PRIVATE, response is private
        return CacheHints.fromExecutionResult(result);
    }
}
```

**Configure CDN to respect Cache-Control even for POST:**

```javascript
// Cloudflare Workers example
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  if (request.method === 'POST' && new URL(request.url).pathname === '/graphql') {
    // Generate cache key from request body
    const body = await request.clone().text()
    const cacheKey = new Request(request.url, {
      method: 'GET',
      headers: request.headers,
      body: body
    })

    let response = await caches.default.match(cacheKey)
    if (!response) {
      response = await fetch(request)
      // Cache based on response headers
      if (response.headers.get('Cache-Control')) {
        event.waitUntil(caches.default.put(cacheKey, response.clone()))
      }
    }
    return response
  }
  return fetch(request)
}
```

---

## Part 6: Cache Invalidation Strategies

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

### Strategy 1: Time-Based Invalidation (TTL)

**Simplest approach:** Let cache entries expire.

```java
@Cacheable(value = "users", key = "#id", ttl = 3600)
public User getUserById(Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

**Pros:**
- Simple to implement
- Predictable memory usage
- Works for eventually consistent data

**Cons:**
- **Stale data** for entire TTL period
- **Cache stampede** - all entries expire at once, thundering herd

**When to use:** Data that can tolerate staleness (user profiles, static content).

### Strategy 2: Explicit Invalidation

**On mutations, clear affected cache entries:**

```java
@Service
public class UserService {
    @Autowired
    private CacheManager cacheManager;

    @Transactional
    public User updateUser(Long id, UserUpdateInput input) {
        User user = userRepository.findById(id).orElseThrow();
        user.setName(input.getName());
        user.setEmail(input.getEmail());
        User updated = userRepository.save(user);

        // Clear cache for this user
        cacheManager.evict("users", id);

        // Clear related caches
        cacheManager.evict("user-posts", id);
        cacheManager.evict("user-comments", id);

        return updated;
    }
}
```

**Pros:**
- **Immediate consistency** - cache never stale
- **Targeted invalidation** - only affected entries cleared

**Cons:**
- **Manual tracking** - must remember all affected caches
- **Easy to miss** - forgotten eviction = stale data
- **Complex relationships** - hard to find all dependencies

### Strategy 3: Tag-Based Invalidation

**Tag cache entries by entity type:**

```java
@Service
public class CacheService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void cacheWithTags(String key, Object value, String... tags) {
        // Store the value
        redisTemplate.opsForValue().set(key, value);

        // Associate with tags
        for (String tag : tags) {
            redisTemplate.opsForSet().add("tag:" + tag, key);
        }
    }

    public void invalidateTag(String tag) {
        // Get all keys with this tag
        Set<String> keys = redisTemplate.opsForSet().members("tag:" + tag);

        // Delete all tagged keys
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }

        // Delete the tag set itself
        redisTemplate.delete("tag:" + tag);
    }
}

// Usage
@Service
public class UserService {
    @Autowired
    private CacheService cacheService;

    public User getUserById(Long id) {
        String cacheKey = "user:" + id;
        User cached = (User) cacheService.get(cacheKey);
        if (cached != null) return cached;

        User user = userRepository.findById(id).orElseThrow();

        // Cache with tags
        cacheService.cacheWithTags(cacheKey, user, "User", "User:" + id);

        return user;
    }

    public void updateUser(Long id, UserUpdateInput input) {
        // Update database
        User user = userRepository.findById(id).orElseThrow();
        user.setName(input.getName());
        userRepository.save(user);

        // Invalidate all caches tagged with this user
        cacheService.invalidateTag("User:" + id);
    }
}
```

**Pros:**
- **Flexible invalidation** - clear all related entries at once
- **Type-based clearing** - `invalidateTag("User")` clears all users
- **Easier to maintain** - less manual tracking

**Cons:**
- **Redis overhead** - extra SET operations
- **Memory usage** - storing tag sets
- **Complex implementation**

### Strategy 4: Subscription-Based Invalidation

**Use GraphQL subscriptions to invalidate client caches:**

```graphql
type Subscription {
  userUpdated(id: ID!): User!
}
```

```java
@Component
public class UserMutationResolver {
    @Autowired
    private ReactivePubSub pubSub;

    @MutationResolver
    public User updateUser(@Argument Long id, @Argument UserUpdateInput input) {
        User updated = userService.updateUser(id, input);

        // Notify subscribers
        pubSub.publish("userUpdated:" + id, updated);

        return updated;
    }
}
```

**Client-side (Apollo):**

```javascript
client.subscribe({
  query: gql`
    subscription {
      userUpdated(id: "123") {
        id
        name
        email
      }
    }
  `
}).subscribe({
  next({ data }) {
    // Update automatically writes to cache
    // All components re-render with new data
  }
})
```

**Pros:**
- **Real-time updates** - clients get changes instantly
- **No polling** - efficient
- **Automatic cache updates** - Apollo Client handles it

**Cons:**
- **WebSocket complexity** - infrastructure overhead
- **Scalability** - managing many subscriptions
- **Client support required** - not all clients support subscriptions

---

## Part 7: Practical Recommendations

### For Small Applications

**Stack:**
- DataLoader for per-request caching (always)
- Spring @Cacheable with Caffeine (in-memory)
- TTL-based expiration

**Code:**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager("users", "posts");
    }
}

@Service
public class UserService {
    @Cacheable("users")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#result.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

### For Medium Applications

**Stack:**
- DataLoader (always)
- Redis for shared cache across servers
- @cacheControl directives in schema
- Explicit invalidation on mutations

**Code:**

```java
@Configuration
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(10))
            )
            .build();
    }
}
```

### For Large Applications

**Stack:**
- DataLoader (always)
- Redis cluster
- Automatic Persisted Queries
- CDN caching for public data (GET + APQ)
- Tag-based invalidation
- Client-side normalized cache (Apollo/Relay)

**Architecture:**

```
[Client with Apollo Cache]
    ↓ (APQ hash)
[CDN] → cache hit? → return cached response
    ↓ cache miss
[Load Balancer]
    ↓
[GraphQL Server 1] ←→ [Redis Cluster] ←→ [GraphQL Server 2]
    ↓                                          ↓
[Database]                                  [Database]
```

---

## Performance Impact: Real Numbers

### Without Caching

```
Query: { users { name posts { title } } }  # 100 users, 10 posts each

Database queries:
- 1 query for users: 50ms
- 100 queries for posts (N+1): 100 × 10ms = 1000ms

Total: 1050ms per request
```

### With DataLoader Only

```
Database queries:
- 1 query for users: 50ms
- 1 batched query for posts: 50ms

Total: 100ms per request
10x improvement! ✅
```

### With DataLoader + Redis Caching

```
First request: 100ms (cache miss)
Subsequent requests (within TTL): 2-5ms (Redis hit)

20-50x improvement! ✅✅
```

### With Full Stack (DataLoader + Redis + CDN + Client Cache)

```
First request: 100ms (full miss)
Same query, different user: 2ms (Redis hit)
Same query, same user: 0ms (client cache hit)

Infinite improvement! ✅✅✅
```

---

## Summary: The Caching Hierarchy

```
Request arrives
    ↓
┌─────────────────────────────────┐
│ 1. Client Cache (Apollo/Relay)  │ ← Fastest (0ms)
│    Normalized entity cache       │
└─────────────────────────────────┘
    ↓ (cache miss)
┌─────────────────────────────────┐
│ 2. CDN Cache                    │ ← Very Fast (10-50ms)
│    GET queries, APQ hashes       │
└─────────────────────────────────┘
    ↓ (cache miss)
┌─────────────────────────────────┐
│ 3. Server Memory (DataLoader)   │ ← Fast (0ms in-request)
│    Per-request deduplication     │
└─────────────────────────────────┘
    ↓ (first access in request)
┌─────────────────────────────────┐
│ 4. Redis Cache                  │ ← Fast (1-3ms)
│    Shared across servers         │
└─────────────────────────────────┘
    ↓ (cache miss)
┌─────────────────────────────────┐
│ 5. Database                     │ ← Slow (10-100ms)
│    Source of truth               │
└─────────────────────────────────┘
```

**Key Takeaway:** Implement caching at multiple layers. Each layer catches what the previous layer missed, creating a robust caching strategy that scales from 1 user to 1 million users.

---

**Next:** [Chapter 13: Error Handling - Partial Failures in Graphs →](13-error-handling.md)
