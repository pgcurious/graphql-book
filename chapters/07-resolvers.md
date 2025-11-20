# Chapter 7: Resolvers - The Bridge Between Queries and Data

> "Resolvers are where the magic happens. They're the translators between your elegant queries and your messy reality."

**What we'll discover:** How resolvers are the fundamental building blocks that connect your GraphQL schema to your actual data sources.

---

## Introduction: The Missing Link

We've built an impressive system:
- A **schema** defines what data exists and its shape
- A **type system** provides validation and safety
- A **query language** lets clients specify exactly what they need
- A **graph model** enables flexible traversal

But there's a crucial piece missing: **How does the data actually get fetched?**

When a client queries for `user(id: 123) { name, posts { title } }`, something needs to:
1. Fetch the user from the database
2. Extract the name field
3. Fetch the user's posts
4. Extract each post's title

**Resolvers are the answer.** They're the glue between your schema and your data.

## Part 1: What is a Resolver?

### The Core Concept

A resolver is a function that fetches the value for a single field. Every field in your schema can have a resolver.

**Function signature:**
```
(parent, arguments, context, info) → field value
```

**Four parameters:**

1. **parent**: The result from the parent field's resolver
2. **arguments**: The arguments passed to this field in the query
3. **context**: Shared data accessible to all resolvers (auth, database, cache, etc.)
4. **info**: Metadata about the query (AST, field info, return type)

**Java interface:**

```java
@FunctionalInterface
public interface FieldResolver<TParent, TResult> {
    TResult resolve(
        TParent parent,
        Map<String, Object> arguments,
        DataFetchingContext context,
        FieldInfo info
    );
}
```

### Trivial Example

For this schema:

```graphql
type User {
  id: ID!
  name: String!
}
```

The resolvers:

```java
public class UserResolvers {

    // Resolver for User.id
    public String getId(
            User parent,  // The User object
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getId();  // Just return the field
    }

    // Resolver for User.name
    public String getName(
            User parent,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getName();  // Just return the field
    }
}
```

These resolvers are trivial—they just access fields on the parent object.

**Default resolvers:** Most GraphQL implementations provide default resolvers for simple field access. If `User` has a `getName()` method, you don't need to write a resolver. The framework generates one automatically.

## Part 2: Resolver Chains

Resolvers form chains. The output of one resolver becomes the input (parent) of the next.

### Execution Flow

Query:
```graphql
{
  user(id: 123) {    # Step 1
    name             # Step 2
    posts {          # Step 3
      title          # Step 4
    }
  }
}
```

**Execution:**

**Step 1:** Resolve `Query.user`
- Parent: `null` (root query has no parent)
- Arguments: `{id: 123}`
- Returns: `User` object

**Step 2:** Resolve `User.name`
- Parent: `User` object (from step 1)
- Arguments: `{}`
- Returns: `"Alice"`

**Step 3:** Resolve `User.posts`
- Parent: `User` object (from step 1)
- Arguments: `{}`
- Returns: `[Post1, Post2, Post3]`

**Step 4:** Resolve `Post.title` for each post
- Parent: `Post1` (first iteration)
- Arguments: `{}`
- Returns: `"Post 1 Title"`

(Repeat step 4 for Post2 and Post3)

### Java Implementation

```java
public class ResolverChain {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PostRepository postRepository;

    // Step 1: Root query resolver
    @Resolver(type = "Query", field = "user")
    public User getUser(
            Object parent,  // null for root queries
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        Long id = (Long) args.get("id");
        return userRepository.findById(id).orElse(null);
    }

    // Step 2: Scalar field resolver (usually auto-generated)
    @Resolver(type = "User", field = "name")
    public String getUserName(
            User parent,  // The User from getUser()
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getName();
    }

    // Step 3: Relationship resolver
    @Resolver(type = "User", field = "posts")
    public List<Post> getUserPosts(
            User parent,  // The User from getUser()
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return postRepository.findByAuthorId(parent.getId());
    }

    // Step 4: Scalar field resolver for each Post
    @Resolver(type = "Post", field = "title")
    public String getPostTitle(
            Post parent,  // Each Post from getUserPosts()
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getTitle();
    }
}
```

The chain flows naturally: root query → user → posts → titles.

## Part 3: Context and Shared Data

### The Context Object

Context is shared across all resolvers in a single query execution. It's created once per request and contains:

- **Authentication**: Current user, roles, permissions
- **Data loaders**: For batching and caching
- **Database connections**: Transaction-scoped connections
- **Caching**: Request-level cache
- **Logging**: Request ID for tracing

```java
public class DataFetchingContext {
    private final HttpServletRequest request;
    private final User currentUser;
    private final DataLoaderRegistry dataLoaders;
    private final EntityManager entityManager;
    private final CacheManager cache;
    private final String requestId;

    // Getters
    public User getCurrentUser() {
        return currentUser;
    }

    public <K, V> DataLoader<K, V> getDataLoader(String name) {
        return dataLoaders.getDataLoader(name);
    }

    public String getRequestId() {
        return requestId;
    }
}
```

### Building Context

Context is built once per request:

```java
@Component
public class GraphQLContextBuilder {

    @Autowired
    private AuthenticationService authService;

    @Autowired
    private DataLoaderFactory dataLoaderFactory;

    public DataFetchingContext build(HttpServletRequest request) {
        // Extract and validate auth token
        String token = request.getHeader("Authorization");
        User currentUser = authService.authenticate(token);

        // Create request-scoped data loaders
        DataLoaderRegistry dataLoaders = new DataLoaderRegistry();
        dataLoaders.register("users", dataLoaderFactory.createUserLoader());
        dataLoaders.register("posts", dataLoaderFactory.createPostLoader());
        dataLoaders.register("comments", dataLoaderFactory.createCommentLoader());

        // Generate request ID for tracing
        String requestId = UUID.randomUUID().toString();

        return new DataFetchingContext(
            request,
            currentUser,
            dataLoaders,
            entityManager,
            cache,
            requestId
        );
    }
}
```

### Using Context in Resolvers

```java
@Resolver(type = "Post", field = "author")
public CompletableFuture<User> getAuthor(
        Post parent,
        Map<String, Object> args,
        DataFetchingContext ctx,
        FieldInfo info) {

    // Use DataLoader from context to batch requests
    return ctx.getDataLoader("users")
        .load(parent.getAuthorId());
}

@Resolver(type = "Post", field = "canEdit")
public boolean canEdit(
        Post parent,
        Map<String, Object> args,
        DataFetchingContext ctx,
        FieldInfo info) {

    // Use current user from context for authorization
    User currentUser = ctx.getCurrentUser();
    return parent.getAuthorId().equals(currentUser.getId())
        || currentUser.hasRole("ADMIN");
}
```

Context enables clean separation of concerns. Resolvers focus on fetching data, while cross-cutting concerns (auth, caching, tracing) live in context.

## Part 4: Async Resolvers

### Why Async?

Most data fetching is I/O bound:
- Database queries
- HTTP calls to microservices
- External API calls
- File system access

Blocking threads while waiting for I/O is wasteful. Async resolvers let you handle thousands of concurrent requests with a small thread pool.

### CompletableFuture Resolvers

```java
public class AsyncResolvers {

    @Autowired
    private AsyncUserRepository userRepository;

    @Autowired
    private AsyncPostRepository postRepository;

    @Resolver(type = "Query", field = "user")
    public CompletableFuture<User> getUser(
            Object parent,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        Long id = (Long) args.get("id");
        return userRepository.findByIdAsync(id);
    }

    @Resolver(type = "User", field = "posts")
    public CompletableFuture<List<Post>> getPosts(
            User parent,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return postRepository.findByAuthorIdAsync(parent.getId());
    }
}
```

The execution engine waits for the `CompletableFuture` to complete before proceeding to child fields.

### Parallel Execution

Fetch multiple pieces of data in parallel:

```java
@Resolver(type = "User", field = "stats")
public CompletableFuture<UserStats> getStats(
        User user,
        Map<String, Object> args,
        DataFetchingContext ctx,
        FieldInfo info) {

    // Launch 3 queries in parallel
    CompletableFuture<Long> postCount =
        CompletableFuture.supplyAsync(() ->
            postRepository.countByAuthor(user.getId())
        );

    CompletableFuture<Long> commentCount =
        CompletableFuture.supplyAsync(() ->
            commentRepository.countByAuthor(user.getId())
        );

    CompletableFuture<Long> followerCount =
        CompletableFuture.supplyAsync(() ->
            followerRepository.countFollowers(user.getId())
        );

    // Wait for all to complete, then combine results
    return CompletableFuture.allOf(postCount, commentCount, followerCount)
        .thenApply(v -> new UserStats(
            postCount.join(),
            commentCount.join(),
            followerCount.join()
        ));
}
```

All three queries run concurrently. Total time is the max of the three, not the sum.

## Part 5: Common Resolver Patterns

### Pattern 1: Computed Fields

```graphql
type User {
  firstName: String!
  lastName: String!
  fullName: String!  # Computed
}
```

```java
@Resolver(type = "User", field = "fullName")
public String getFullName(User parent) {
    return parent.getFirstName() + " " + parent.getLastName();
}
```

The database stores `firstName` and `lastName`. The resolver computes `fullName` on demand.

### Pattern 2: Aggregations

```graphql
type Post {
  id: ID!
  title: String!
  likeCount: Int!     # Aggregated from likes table
  commentCount: Int!  # Aggregated from comments table
}
```

```java
@Resolver(type = "Post", field = "likeCount")
public CompletableFuture<Integer> getLikeCount(
        Post parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    // Use DataLoader to batch count queries
    return ctx.getDataLoader("likeCounts")
        .load(parent.getId());
}
```

Aggregations are expensive. DataLoaders batch them efficiently.

### Pattern 3: Delegating to Other Services

```graphql
type User {
  id: ID!
  name: String!
  recommendations: [Post!]!  # From recommendation service
}
```

```java
@Resolver(type = "User", field = "recommendations")
public CompletableFuture<List<Post>> getRecommendations(
        User parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    // Call external recommendation service
    return recommendationClient
        .getRecommendations(parent.getId())
        .thenCompose(postIds ->
            // Then fetch posts using DataLoader
            ctx.getDataLoader("posts").loadMany(postIds)
        );
}
```

GraphQL unifies data from multiple sources behind a single schema.

### Pattern 4: Authorization

```java
@Resolver(type = "User", field = "email")
public String getEmail(
        User parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    User currentUser = ctx.getCurrentUser();

    // Only show email to the user themselves or admins
    if (parent.getId().equals(currentUser.getId()) ||
        currentUser.hasRole("ADMIN")) {
        return parent.getEmail();
    }

    throw new UnauthorizedException("Cannot access email");
}
```

Field-level authorization. Different fields can have different permissions.

### Pattern 5: Pagination (Relay-style)

```graphql
type Query {
  posts(first: Int, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

```java
@Resolver(type = "Query", field = "posts")
public PostConnection getPosts(
        Object parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    Integer first = (Integer) args.getOrDefault("first", 10);
    String after = (String) args.get("after");

    // Decode cursor (base64 encoded ID)
    Long afterId = after != null ? decodeCursor(after) : 0L;

    // Fetch first+1 to determine hasNextPage
    List<Post> posts = postRepository.findAfter(afterId, first + 1);

    boolean hasNextPage = posts.size() > first;
    if (hasNextPage) {
        posts = posts.subList(0, first);
    }

    // Build edges with cursors
    List<PostEdge> edges = posts.stream()
        .map(post -> new PostEdge(
            post,
            encodeCursor(post.getId())
        ))
        .collect(Collectors.toList());

    // Build page info
    String endCursor = !edges.isEmpty()
        ? edges.get(edges.size() - 1).getCursor()
        : null;

    PageInfo pageInfo = new PageInfo(hasNextPage, endCursor);

    return new PostConnection(edges, pageInfo, posts.size());
}

private String encodeCursor(Long id) {
    return Base64.getEncoder().encodeToString(id.toString().getBytes());
}

private Long decodeCursor(String cursor) {
    String decoded = new String(Base64.getDecoder().decode(cursor));
    return Long.parseLong(decoded);
}
```

Cursor-based pagination is stable even when data changes between requests.

## Part 6: Error Handling

### Throwing Exceptions

Resolvers can throw exceptions:

```java
@Resolver(type = "Query", field = "user")
public User getUser(
        Object parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    Long id = (Long) args.get("id");

    return userRepository.findById(id)
        .orElseThrow(() ->
            new ResourceNotFoundException("User not found: " + id)
        );
}
```

The exception is caught by the GraphQL engine and converted to an error in the response.

### Custom Error Extensions

Add metadata to errors:

```java
public class ResourceNotFoundException extends RuntimeException {
    private final String resourceType;
    private final String resourceId;

    public ResourceNotFoundException(String resourceType, String resourceId) {
        super(resourceType + " not found: " + resourceId);
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }

    public Map<String, Object> getExtensions() {
        return Map.of(
            "code", "RESOURCE_NOT_FOUND",
            "resourceType", resourceType,
            "resourceId", resourceId,
            "timestamp", Instant.now().toString()
        );
    }
}
```

Response:

```json
{
  "errors": [
    {
      "message": "User not found: 123",
      "path": ["user"],
      "extensions": {
        "code": "RESOURCE_NOT_FOUND",
        "resourceType": "User",
        "resourceId": "123",
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ],
  "data": {
    "user": null
  }
}
```

### Partial Failures

GraphQL supports partial failures. If one field fails, other fields can still succeed:

```java
@Resolver(type = "User", field = "posts")
public List<Post> getPosts(
        User parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    try {
        return postRepository.findByAuthorId(parent.getId());
    } catch (DatabaseException e) {
        logger.error("Failed to fetch posts for user: " + parent.getId(), e);

        // Return empty list; error goes in errors array
        return Collections.emptyList();
    }
}
```

Result:

```json
{
  "errors": [
    {
      "message": "Database error while fetching posts",
      "path": ["user", "posts"]
    }
  ],
  "data": {
    "user": {
      "name": "Alice",
      "posts": []
    }
  }
}
```

The client gets the user's name even though fetching posts failed.

## Part 7: Testing Resolvers

### Unit Tests

Resolvers are just functions. Test them like any other function:

```java
@Test
public void testGetUserPosts() {
    // Setup
    User user = new User(1L, "Alice");
    Post post1 = new Post(1L, "Title 1");
    Post post2 = new Post(2L, "Title 2");

    when(postRepository.findByAuthorId(1L))
        .thenReturn(List.of(post1, post2));

    // Execute
    UserResolvers resolver = new UserResolvers(postRepository);
    List<Post> result = resolver.getPosts(
        user,
        Map.of(),
        mockContext,
        mockInfo
    );

    // Verify
    assertEquals(2, result.size());
    assertEquals("Title 1", result.get(0).getTitle());
    assertEquals("Title 2", result.get(1).getTitle());

    verify(postRepository).findByAuthorId(1L);
}
```

### Integration Tests

Test the entire GraphQL execution:

```java
@SpringBootTest
@AutoConfigureGraphQLTester
public class UserQueryIntegrationTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @Test
    public void testUserQueryWithPosts() {
        graphQlTester
            .document("""
                query {
                    user(id: 1) {
                        name
                        posts {
                            title
                        }
                    }
                }
                """)
            .execute()
            .path("user.name").entity(String.class).isEqualTo("Alice")
            .path("user.posts").entityList(Post.class).hasSize(2)
            .path("user.posts[0].title").entity(String.class).isEqualTo("First Post");
    }
}
```

Integration tests verify the entire stack: parsing, validation, execution, and serialization.

## Key Insights

Let's crystallize what we've learned:

1. **Resolvers are functions** that fetch field values. One resolver per field.

2. **Resolvers form chains**. The output of one becomes the parent input of the next.

3. **Context is shared** across all resolvers in a request. Use it for auth, caching, and data loaders.

4. **Async resolvers are crucial** for performance. Don't block threads waiting for I/O.

5. **Errors are first-class**. Partial failures are acceptable; clients get what they can.

6. **Resolvers encapsulate data fetching**. The schema defines the what; resolvers define the how.

7. **Resolvers are testable**. They're just functions with inputs and outputs.

Resolvers are where GraphQL's elegance meets your messy reality. They translate between your clean schema and your complex data sources. Master resolvers, and you master GraphQL.

## What's Next

We've covered the fundamentals: schemas, types, queries, graphs, and resolvers. But there's more depth to explore.

In the next chapters, we'll dive into implementation details:
- How to parse queries into an Abstract Syntax Tree
- How the execution engine actually runs queries
- How to solve the N+1 problem with DataLoader
- How to handle mutations, subscriptions, and security

The journey continues.

---

**Next:** [Chapter 8: Schema Design - SDL from Scratch →](08-schema-design.md)

**Previous:** [← Chapter 6: The Graph Mental Model - Thinking in Relationships](06-graph-mental-model.md)
