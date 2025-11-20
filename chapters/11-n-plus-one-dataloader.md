# Chapter 11: The N+1 Problem and DataLoader Pattern

> "The N+1 problem is GraphQL's most famous performance trap. It's also one of the most elegantly solvable."

---

## The Problem: A Performance Disaster Hiding in Plain Sight

You've built your GraphQL server. Your schema is beautiful. Your resolvers work perfectly. You deploy to production. And then... your database explodes under load.

What happened? Let's trace through a seemingly innocent query:

```graphql
query {
  posts {
    id
    title
    author {
      id
      name
      email
    }
  }
}
```

**Expected behavior:** Fetch posts and their authors.

**What actually happens:** Let's trace the resolver execution:

### Resolver Execution (Naive Implementation)

```java
public class QueryResolver {
    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    public List<Post> posts() {
        // 1 query: SELECT * FROM posts
        return postRepository.findAll();
    }
}

public class PostResolver {
    @Autowired
    private UserRepository userRepository;

    public User author(Post post) {
        // Called once PER POST!
        // SELECT * FROM users WHERE id = ?
        return userRepository.findById(post.getAuthorId());
    }
}
```

**Database queries executed:**

```sql
-- Query 1: Fetch all posts
SELECT * FROM posts;
-- Returns 100 posts

-- Query 2: Fetch author for post 1
SELECT * FROM users WHERE id = 1;

-- Query 3: Fetch author for post 2
SELECT * FROM users WHERE id = 2;

-- Query 4: Fetch author for post 3
SELECT * FROM users WHERE id = 3;

... (97 more queries)

-- Query 101: Fetch author for post 100
SELECT * FROM users WHERE id = 100;
```

**Total queries: 1 + 100 = 101 queries**

This is the **N+1 problem**:
- **1 query** to fetch N posts
- **N queries** to fetch each post's author
- Total: **N+1 queries**

### Why This Is Catastrophic

For 100 posts:
- **101 database round trips**
- Each query has latency (5-10ms in-datacenter, 50-100ms cross-region)
- 101 queries × 10ms = **1,010ms = 1 second** just in network latency!
- Add query execution time: easily **2-3 seconds** for a simple query

For 1,000 posts:
- **1,001 queries**
- **10+ seconds** response time
- Database connection pool exhaustion
- Server collapse under moderate load

---

## Why GraphQL Makes This Worse (and Better)

### The Bad News: GraphQL Amplifies N+1

In REST APIs, the N+1 problem is hidden in specific endpoints:

```
GET /posts          → 1 query
GET /posts/1/author → 1 query
GET /posts/2/author → 1 query
...
```

You'd never make 101 HTTP requests, so the problem is less visible.

In GraphQL, **the resolver model naturally creates N+1 problems**:

1. Each field is resolved independently
2. Resolvers don't know about each other
3. Nested queries compound the problem

```graphql
query {
  posts {                    # 1 query
    author {                 # N queries
      posts {                # N * M queries
        comments {           # N * M * K queries
          author {           # N * M * K * J queries
            ...
          }
        }
      }
    }
  }
}
```

This can explode into **thousands** of queries!

### The Good News: GraphQL Makes It Solvable

Because all queries go through a **single execution engine**, we can:
1. Detect batching opportunities
2. Defer and batch database calls
3. Deduplicate requests
4. Cache results within a single request

This is where **DataLoader** comes in.

---

## The Solution: DataLoader Pattern

The core insight: **Instead of executing queries immediately, batch them together.**

### Conceptual Model

**Without DataLoader:**
```
Resolver 1: getUserById(1)  → DB Query → Result 1
Resolver 2: getUserById(2)  → DB Query → Result 2
Resolver 3: getUserById(3)  → DB Query → Result 3
```

**With DataLoader:**
```
Resolver 1: getUserById(1)  → Add to batch
Resolver 2: getUserById(2)  → Add to batch
Resolver 3: getUserById(3)  → Add to batch
                             ↓
                    Batch executes once
                             ↓
              DB Query: getUsersByIds([1, 2, 3])
                             ↓
                    Results distributed back
                             ↓
Resolver 1: ← Result 1
Resolver 2: ← Result 2
Resolver 3: ← Result 3
```

**Result:**
- 3 queries → **1 query**
- 101 queries → **2 queries** (1 for posts, 1 batched for authors)

---

## Implementing DataLoader in Java

### Step 1: The Core DataLoader Class

```java
public class DataLoader<K, V> {
    private final BatchLoader<K, V> batchLoader;
    private final Map<K, CompletableFuture<V>> cache;
    private final List<K> queue;
    private boolean dispatched;

    public DataLoader(BatchLoader<K, V> batchLoader) {
        this.batchLoader = batchLoader;
        this.cache = new HashMap<>();
        this.queue = new ArrayList<>();
        this.dispatched = false;
    }

    /**
     * Load a value by key. Returns immediately with a CompletableFuture.
     * The actual loading happens when dispatch() is called.
     */
    public CompletableFuture<V> load(K key) {
        // Check cache first
        if (cache.containsKey(key)) {
            return cache.get(key);
        }

        // Create a future for this key
        CompletableFuture<V> future = new CompletableFuture<>();
        cache.put(key, future);
        queue.add(key);

        return future;
    }

    /**
     * Execute the batch load and fulfill all pending futures.
     */
    public CompletableFuture<Void> dispatch() {
        if (dispatched || queue.isEmpty()) {
            return CompletableFuture.completedFuture(null);
        }

        dispatched = true;
        List<K> keys = new ArrayList<>(queue);
        queue.clear();

        return batchLoader.load(keys)
            .thenAccept(values -> {
                // Fulfill all futures with their corresponding values
                for (int i = 0; i < keys.size(); i++) {
                    K key = keys.get(i);
                    V value = values.get(i);
                    cache.get(key).complete(value);
                }
                dispatched = false;
            })
            .exceptionally(error -> {
                // Fulfill all futures with the error
                for (K key : keys) {
                    cache.get(key).completeExceptionally(error);
                }
                dispatched = false;
                return null;
            });
    }

    /**
     * Clear the cache (typically done after each request).
     */
    public void clear() {
        cache.clear();
        queue.clear();
        dispatched = false;
    }
}
```

### Step 2: The Batch Loader Interface

```java
@FunctionalInterface
public interface BatchLoader<K, V> {
    /**
     * Given a list of keys, return a CompletableFuture that will resolve
     * to a list of values in the same order.
     *
     * IMPORTANT: The returned list must be the same size as the input list,
     * and values must be in the same order as their corresponding keys.
     */
    CompletableFuture<List<V>> load(List<K> keys);
}
```

### Step 3: Creating a User DataLoader

```java
@Component
public class UserDataLoader {
    @Autowired
    private UserRepository userRepository;

    public DataLoader<Long, User> create() {
        return new DataLoader<>(keys -> {
            // This is called once per batch with ALL accumulated keys
            System.out.println("Batch loading users: " + keys);

            // Single query with IN clause
            List<User> users = userRepository.findAllById(keys);

            // CRITICAL: Return users in the same order as keys
            Map<Long, User> userMap = users.stream()
                .collect(Collectors.toMap(User::getId, u -> u));

            List<User> orderedUsers = keys.stream()
                .map(userMap::get)
                .collect(Collectors.toList());

            return CompletableFuture.completedFuture(orderedUsers);
        });
    }
}
```

**The SQL query that gets executed:**

```sql
-- Instead of 100 separate queries:
-- SELECT * FROM users WHERE id = 1;
-- SELECT * FROM users WHERE id = 2;
-- ...

-- We execute ONE query:
SELECT * FROM users WHERE id IN (1, 2, 3, ..., 100);
```

### Step 4: Using DataLoader in Resolvers

```java
@Component
public class PostResolver {
    public CompletableFuture<User> author(
        Post post,
        DataLoader<Long, User> userDataLoader
    ) {
        // Don't query immediately!
        // Just register that we need this user
        return userDataLoader.load(post.getAuthorId());
    }
}
```

### Step 5: Dispatching Batches

The GraphQL execution engine needs to dispatch batches at the right time:

```java
public class GraphQLExecutor {
    public CompletableFuture<ExecutionResult> execute(
        String query,
        Map<String, DataLoader<?, ?>> dataLoaders
    ) {
        // Parse and validate query
        Document document = parser.parse(query);

        // Execute query
        CompletableFuture<Object> result = executeQuery(document, dataLoaders);

        // After each "level" of execution, dispatch all data loaders
        return result.thenCompose(data -> {
            // Dispatch all batches
            List<CompletableFuture<Void>> dispatches = dataLoaders.values()
                .stream()
                .map(DataLoader::dispatch)
                .collect(Collectors.toList());

            return CompletableFuture.allOf(
                dispatches.toArray(new CompletableFuture[0])
            ).thenApply(v -> new ExecutionResult(data));
        });
    }
}
```

---

## Advanced DataLoader Features

### Feature 1: Caching

DataLoader caches results within a single request:

```java
// First call - goes to database
userDataLoader.load(123L);

// Second call - returns cached result
userDataLoader.load(123L);  // No database query!
```

**Why per-request caching?**
- Same user might be referenced multiple times in one query
- Avoid duplicate work
- Consistency within a single request

**Why not cross-request caching?**
- Data staleness
- Memory leaks
- Different users need different data (authorization)

### Feature 2: Error Handling

What if batch loading fails for some keys?

```java
public class UserDataLoader {
    public DataLoader<Long, User> create() {
        return new DataLoader<>(keys -> {
            try {
                List<User> users = userRepository.findAllById(keys);
                Map<Long, User> userMap = users.stream()
                    .collect(Collectors.toMap(User::getId, u -> u));

                // Handle missing users
                List<User> orderedUsers = keys.stream()
                    .map(key -> userMap.getOrDefault(key, null))
                    .collect(Collectors.toList());

                return CompletableFuture.completedFuture(orderedUsers);
            } catch (Exception e) {
                // Fail the entire batch
                return CompletableFuture.failedFuture(e);
            }
        });
    }
}
```

### Feature 3: Deduplication

DataLoader automatically deduplicates keys:

```java
userDataLoader.load(123L);
userDataLoader.load(123L);
userDataLoader.load(123L);
userDataLoader.load(456L);

// Batch call receives: [123, 456] (not [123, 123, 123, 456])
```

### Feature 4: Max Batch Size

For very large batches, split into multiple queries:

```java
public class DataLoader<K, V> {
    private final int maxBatchSize;

    public DataLoader(BatchLoader<K, V> batchLoader, int maxBatchSize) {
        this.batchLoader = batchLoader;
        this.maxBatchSize = maxBatchSize;
        // ...
    }

    public CompletableFuture<Void> dispatch() {
        if (queue.size() <= maxBatchSize) {
            return dispatchBatch(queue);
        }

        // Split into multiple batches
        List<CompletableFuture<Void>> batches = new ArrayList<>();
        for (int i = 0; i < queue.size(); i += maxBatchSize) {
            int end = Math.min(i + maxBatchSize, queue.size());
            List<K> batch = queue.subList(i, end);
            batches.add(dispatchBatch(batch));
        }

        return CompletableFuture.allOf(
            batches.toArray(new CompletableFuture[0])
        );
    }
}
```

**Why limit batch size?**
- Database query limits (e.g., SQL IN clause max 1000 items)
- Network packet size limits
- Fairness (don't let one huge batch block others)

---

## Real-World Example: Complex Query

Let's see DataLoader in action with a more complex query:

```graphql
query {
  posts {
    id
    title
    author {
      id
      name
    }
    comments {
      id
      text
      author {
        id
        name
      }
    }
  }
}
```

### Without DataLoader

```sql
-- 1 query: Fetch posts
SELECT * FROM posts;  -- Returns 100 posts

-- 100 queries: Fetch each post's author
SELECT * FROM users WHERE id = ?;  -- × 100

-- 100 queries: Fetch comments for each post
SELECT * FROM comments WHERE post_id = ?;  -- × 100, returns 10 comments each

-- 1000 queries: Fetch author for each comment
SELECT * FROM users WHERE id = ?;  -- × 1000

-- Total: 1,201 queries!
```

### With DataLoader

```sql
-- 1 query: Fetch posts
SELECT * FROM posts;  -- Returns 100 posts

-- 1 query: Batch fetch authors for all posts
SELECT * FROM users WHERE id IN (1, 2, 3, ..., 100);

-- 1 query: Batch fetch comments for all posts
SELECT * FROM comments WHERE post_id IN (1, 2, 3, ..., 100);  -- Returns 1000 comments

-- 1 query: Batch fetch authors for all comments
SELECT * FROM users WHERE id IN (1, 5, 7, ..., 89);  -- Unique author IDs

-- Total: 4 queries!
```

**Improvement: 1,201 queries → 4 queries (99.7% reduction!)**

---

## Implementation in Spring Boot with GraphQL Java

### Step 1: Configure DataLoader Registry

```java
@Configuration
public class DataLoaderConfiguration {

    @Bean
    public DataLoaderRegistry dataLoaderRegistry(
        UserRepository userRepository,
        PostRepository postRepository
    ) {
        DataLoaderRegistry registry = new DataLoaderRegistry();

        // User DataLoader
        registry.register("user", DataLoader.newDataLoader(
            (List<Long> userIds) -> CompletableFuture.supplyAsync(() -> {
                List<User> users = userRepository.findAllById(userIds);
                Map<Long, User> userMap = users.stream()
                    .collect(Collectors.toMap(User::getId, u -> u));

                return userIds.stream()
                    .map(userMap::get)
                    .collect(Collectors.toList());
            })
        ));

        // Post DataLoader (for comments -> post)
        registry.register("post", DataLoader.newDataLoader(
            (List<Long> postIds) -> CompletableFuture.supplyAsync(() -> {
                List<Post> posts = postRepository.findAllById(postIds);
                Map<Long, Post> postMap = posts.stream()
                    .collect(Collectors.toMap(Post::getId, p -> p));

                return postIds.stream()
                    .map(postMap::get)
                    .collect(Collectors.toList());
            })
        ));

        // Comments by Post ID (one-to-many relationship)
        registry.register("commentsByPostId", DataLoader.newMappedDataLoader(
            (Set<Long> postIds) -> CompletableFuture.supplyAsync(() -> {
                List<Comment> comments = commentRepository.findByPostIdIn(postIds);

                return comments.stream()
                    .collect(Collectors.groupingBy(Comment::getPostId));
            })
        ));

        return registry;
    }
}
```

### Step 2: Use in Resolvers

```java
@Component
public class PostResolver implements GraphQLResolver<Post> {

    public CompletableFuture<User> author(
        Post post,
        DataLoader<Long, User> userDataLoader
    ) {
        return userDataLoader.load(post.getAuthorId());
    }

    public CompletableFuture<List<Comment>> comments(
        Post post,
        DataLoader<Long, List<Comment>> commentsByPostIdDataLoader
    ) {
        return commentsByPostIdDataLoader.load(post.getId());
    }
}

@Component
public class CommentResolver implements GraphQLResolver<Comment> {

    public CompletableFuture<User> author(
        Comment comment,
        DataLoader<Long, User> userDataLoader
    ) {
        return userDataLoader.load(comment.getAuthorId());
    }

    public CompletableFuture<Post> post(
        Comment comment,
        DataLoader<Long, Post> postDataLoader
    ) {
        return postDataLoader.load(comment.getPostId());
    }
}
```

### Step 3: Instrumentation for Auto-Dispatch

```java
@Component
public class DataLoaderDispatchInstrumentation extends SimpleInstrumentation {

    @Override
    public CompletableFuture<ExecutionResult> instrumentExecutionResult(
        ExecutionResult executionResult,
        InstrumentationExecutionParameters parameters
    ) {
        DataLoaderRegistry registry = parameters
            .getGraphQLContext()
            .get(DataLoaderRegistry.class);

        // Dispatch all pending batches
        return registry.dispatchAll()
            .thenApply(v -> executionResult);
    }
}
```

---

## One-to-Many Relationships: Mapped DataLoader

The examples above handle **one-to-one** relationships (post → author). What about **one-to-many** (post → comments)?

### The Challenge

```java
// We need to load comments for multiple posts at once
// Post 1 → [Comment 1, Comment 2, Comment 3]
// Post 2 → [Comment 4, Comment 5]
// Post 3 → []
```

### Solution: Mapped DataLoader

```java
public class MappedDataLoader<K, V> {
    private final MappedBatchLoader<K, V> batchLoader;

    public CompletableFuture<List<V>> load(K key) {
        // Similar to regular DataLoader
        // ...
    }

    public CompletableFuture<Void> dispatch() {
        Set<K> keys = new HashSet<>(queue);

        return batchLoader.load(keys)
            .thenAccept(resultMap -> {
                // resultMap is Map<K, List<V>>
                for (K key : keys) {
                    List<V> values = resultMap.getOrDefault(
                        key,
                        Collections.emptyList()
                    );
                    cache.get(key).complete(values);
                }
            });
    }
}

@FunctionalInterface
public interface MappedBatchLoader<K, V> {
    CompletableFuture<Map<K, List<V>>> load(Set<K> keys);
}
```

### Usage Example

```java
// Create a mapped data loader for comments by post ID
DataLoader<Long, List<Comment>> commentsLoader = new MappedDataLoader<>(
    postIds -> CompletableFuture.supplyAsync(() -> {
        // Fetch all comments for these posts
        List<Comment> comments = commentRepository.findByPostIdIn(postIds);

        // Group by post ID
        return comments.stream()
            .collect(Collectors.groupingBy(Comment::getPostId));
    })
);
```

**SQL executed:**

```sql
-- Instead of N queries:
-- SELECT * FROM comments WHERE post_id = 1;
-- SELECT * FROM comments WHERE post_id = 2;
-- ...

-- Execute 1 query:
SELECT * FROM comments WHERE post_id IN (1, 2, 3, ..., 100);
```

---

## Performance Comparison

Let's benchmark the difference with real numbers:

### Scenario: Fetch 100 posts with authors and comments

**Without DataLoader:**
- Queries: 1 (posts) + 100 (authors) + 100 (comments per post) + 1000 (comment authors) = **1,201 queries**
- Time: ~12 seconds (10ms per query)
- Database connections: 1,201 (connection pool exhaustion!)

**With DataLoader:**
- Queries: 1 (posts) + 1 (batch authors) + 1 (batch comments) + 1 (batch comment authors) = **4 queries**
- Time: ~40ms
- Database connections: 4

**Improvement:**
- **300× fewer queries**
- **300× faster response**
- **Sustainable** under load

---

## Common Pitfalls and Best Practices

### Pitfall 1: Wrong Result Order

```java
// WRONG: Returns users in arbitrary order
public CompletableFuture<List<User>> load(List<Long> ids) {
    List<User> users = userRepository.findAllById(ids);
    return CompletableFuture.completedFuture(users);  // Wrong order!
}

// CORRECT: Returns users in the same order as input IDs
public CompletableFuture<List<User>> load(List<Long> ids) {
    List<User> users = userRepository.findAllById(ids);
    Map<Long, User> userMap = users.stream()
        .collect(Collectors.toMap(User::getId, u -> u));

    List<User> orderedUsers = ids.stream()
        .map(userMap::get)  // Preserves input order
        .collect(Collectors.toList());

    return CompletableFuture.completedFuture(orderedUsers);
}
```

### Pitfall 2: Not Clearing Cache Between Requests

```java
// WRONG: Reusing same DataLoader across requests
@Bean
@Scope("singleton")
public DataLoader<Long, User> userDataLoader() {
    return new DataLoader<>(...);  // Shared across requests!
}

// CORRECT: New DataLoader per request
@Bean
@Scope("request")
public DataLoader<Long, User> userDataLoader() {
    return new DataLoader<>(...);  // Fresh per request
}
```

### Pitfall 3: Forgetting to Handle Nulls

```java
// WRONG: Null pointer exception if user doesn't exist
List<User> orderedUsers = ids.stream()
    .map(userMap::get)  // Returns null for missing users
    .map(User::getName)  // NPE!
    .collect(Collectors.toList());

// CORRECT: Handle missing users gracefully
List<User> orderedUsers = ids.stream()
    .map(id -> userMap.getOrDefault(id, null))  // Explicit null
    .collect(Collectors.toList());
```

### Best Practice 1: Use DataLoader for ALL Database Access

Don't mix direct database calls with DataLoader:

```java
// BAD: Inconsistent data access patterns
public User author(Post post) {
    if (someCondition) {
        return userRepository.findById(post.getAuthorId());  // Direct query
    } else {
        return userDataLoader.load(post.getAuthorId()).join();  // DataLoader
    }
}

// GOOD: Always use DataLoader
public CompletableFuture<User> author(Post post) {
    return userDataLoader.load(post.getAuthorId());
}
```

### Best Practice 2: Monitor Batch Sizes

Log batch sizes to detect issues:

```java
public DataLoader<Long, User> create() {
    return new DataLoader<>(keys -> {
        logger.info("Batch loading {} users", keys.size());

        if (keys.size() > 1000) {
            logger.warn("Very large batch detected: {}", keys.size());
        }

        // ... batch loading logic
    });
}
```

### Best Practice 3: Set Reasonable Max Batch Sizes

```java
DataLoader<Long, User> userLoader = DataLoader.newDataLoader(
    batchLoader,
    DataLoaderOptions.newOptions()
        .setMaxBatchSize(100)  // Limit batch size
        .setBatchingEnabled(true)
        .setCachingEnabled(true)
);
```

---

## Key Takeaways

1. **N+1 is GraphQL's most common performance problem** - One query becomes N+1 queries
2. **DataLoader solves N+1 through batching** - Collect multiple requests, execute as one
3. **DataLoader provides per-request caching** - Same entity requested multiple times = one load
4. **Order matters** - Batch results must match input key order
5. **Scope matters** - DataLoader should be request-scoped, not singleton
6. **Use for all database access** - Consistent pattern prevents N+1
7. **Monitor and tune** - Track batch sizes, set limits

The N+1 problem is the reason many developers worry about GraphQL performance. DataLoader is the reason they shouldn't. It's not just an optimization—it's a fundamental design pattern that makes GraphQL practical at scale.

---

**Next:** [Chapter 12: Caching Strategies →](12-caching-strategies.md)
**Previous:** [← Chapter 10: Execution Engine](10-execution-engine.md)
