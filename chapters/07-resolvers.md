# Chapter 7: Resolvers - The Bridge Between Queries and Data

**Status:** Detailed Outline (To Be Expanded)

**What we'll discover:** How resolvers are the fundamental building blocks that connect your GraphQL schema to your actual data sources.

---

## Outline

### Introduction: The Missing Link
- We have a schema (the contract)
- We have queries (the requests)
- But how does data actually get fetched?
- **Resolvers are the answer**

### Part 1: What is a Resolver?

#### The Core Concept
A resolver is a function with this signature:
```
(parent, arguments, context, info) í field value
```

**Parameters:**
- **parent**: The parent object (result from parent resolver)
- **arguments**: Arguments passed to this field
- **context**: Shared data (auth, database connections, etc.)
- **info**: Query AST and schema info

**Example:**
```java
@FunctionalInterface
public interface FieldResolver<T, R> {
    R resolve(
        T parent,
        Map<String, Object> arguments,
        DataFetchingContext context,
        FieldInfo info
    );
}
```

#### Trivial Resolver Example
```graphql
type User {
    id: ID!
    name: String!
}
```

```java
public class UserResolvers {

    // Resolver for User.id
    public String getId(User user, Map<String, Object> args,
                        DataFetchingContext ctx, FieldInfo info) {
        return user.getId();  // Just return the field
    }

    // Resolver for User.name
    public String getName(User user, Map<String, Object> args,
                          DataFetchingContext ctx, FieldInfo info) {
        return user.getName();  // Just return the field
    }
}
```

**Default resolvers:** Most GraphQL implementations provide default resolvers for simple field access.

### Part 2: Resolver Chains

#### Parent-Child Relationship
```graphql
query {
    user(id: 123) {    # Resolver 1: Query.user
        name           # Resolver 2: User.name
        posts {        # Resolver 3: User.posts
            title      # Resolver 4: Post.title
        }
    }
}
```

**Execution flow:**
```
1. Query.user(parent=null, args={id: 123})
   í Returns User object

2. User.name(parent=User, args={})
   í Returns "Alice"

3. User.posts(parent=User, args={})
   í Returns [Post1, Post2, Post3]

4. For each Post:
   Post.title(parent=Post, args={})
   í Returns "Post Title"
```

#### Java Implementation
```java
public class ResolverChain {

    @Resolver(type = "Query", field = "user")
    public User getUser(
            Object parent,  // null for root queries
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        Long id = (Long) args.get("id");
        return userRepository.findById(id).orElse(null);
    }

    @Resolver(type = "User", field = "name")
    public String getUserName(
            User parent,  // The User object from previous resolver
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getName();
    }

    @Resolver(type = "User", field = "posts")
    public List<Post> getUserPosts(
            User parent,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return postRepository.findByAuthorId(parent.getId());
    }

    @Resolver(type = "Post", field = "title")
    public String getPostTitle(
            Post parent,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        return parent.getTitle();
    }
}
```

### Part 3: Resolver Registration

#### Annotation-Based Registration
```java
@Component
public class UserResolvers {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PostRepository postRepository;

    @GraphQLQueryResolver
    public User user(@Argument("id") Long id) {
        return userRepository.findById(id).orElse(null);
    }

    @GraphQLResolver(type = User.class)
    public List<Post> posts(User user,
                           @Argument("limit") Integer limit) {
        return postRepository.findByAuthorId(
            user.getId(),
            limit != null ? limit : 10
        );
    }
}
```

#### Programmatic Registration
```java
public class ResolverRegistry {

    private Map<String, Map<String, FieldResolver<?, ?>>> resolvers;

    public <T, R> void register(
            Class<T> type,
            String fieldName,
            FieldResolver<T, R> resolver) {

        String typeName = type.getSimpleName();
        resolvers
            .computeIfAbsent(typeName, k -> new HashMap<>())
            .put(fieldName, resolver);
    }

    public FieldResolver<?, ?> get(String typeName, String fieldName) {
        return resolvers
            .getOrDefault(typeName, Collections.emptyMap())
            .get(fieldName);
    }
}

// Usage
registry.register(User.class, "posts",
    (user, args, ctx, info) -> postRepository.findByAuthorId(user.getId())
);
```

#### Schema-First Registration
```java
// Define schema in SDL
String schema = """
    type Query {
        user(id: ID!): User
    }
    type User {
        id: ID!
        name: String!
        posts: [Post!]!
    }
    """;

// Wire resolvers to schema
RuntimeWiring wiring = RuntimeWiring.newRuntimeWiring()
    .type("Query", builder -> builder
        .dataFetcher("user", env -> {
            Long id = env.getArgument("id");
            return userRepository.findById(id).orElse(null);
        }))
    .type("User", builder -> builder
        .dataFetcher("posts", env -> {
            User user = env.getSource();
            return postRepository.findByAuthorId(user.getId());
        }))
    .build();

GraphQLSchema graphQLSchema = SchemaGenerator.buildSchema(schema, wiring);
```

### Part 4: Context and Dependency Injection

#### The Context Object
```java
public class DataFetchingContext {
    private final HttpServletRequest request;
    private final Authentication authentication;
    private final DataLoaderRegistry dataLoaders;
    private final DatabaseConnection database;
    private final CacheManager cache;

    // Getters...

    public User getCurrentUser() {
        return authentication.getUser();
    }

    public <K, V> DataLoader<K, V> getDataLoader(String name) {
        return dataLoaders.getDataLoader(name);
    }
}
```

#### Building Context Per Request
```java
@Component
public class GraphQLContextBuilder {

    @Autowired
    private AuthenticationService authService;

    @Autowired
    private DataLoaderRegistry dataLoaderRegistry;

    public DataFetchingContext build(HttpServletRequest request) {
        // Extract auth token
        String token = request.getHeader("Authorization");
        Authentication auth = authService.authenticate(token);

        // Create fresh DataLoaders for this request
        DataLoaderRegistry loaders = new DataLoaderRegistry();
        loaders.register("users", createUserLoader());
        loaders.register("posts", createPostLoader());

        return new DataFetchingContext(
            request,
            auth,
            loaders,
            database,
            cache
        );
    }

    private DataLoader<Long, User> createUserLoader() {
        return DataLoader.newDataLoader(keys ->
            CompletableFuture.supplyAsync(() ->
                userRepository.findAllById(keys)
            )
        );
    }
}
```

#### Using Context in Resolvers
```java
@GraphQLResolver(type = Post.class)
public class PostResolvers {

    @GraphQLField
    public User author(Post post, DataFetchingContext ctx) {
        // Use DataLoader from context
        return ctx.getDataLoader("users")
            .load(post.getAuthorId())
            .join();
    }

    @GraphQLField
    public boolean canEdit(Post post, DataFetchingContext ctx) {
        // Check authorization
        User currentUser = ctx.getCurrentUser();
        return post.getAuthorId().equals(currentUser.getId());
    }
}
```

### Part 5: Async Resolvers

#### Why Async?
- Database queries are I/O bound
- Network calls to microservices
- External API calls
- Don't block threads waiting

#### CompletableFuture Resolvers
```java
public class AsyncResolvers {

    @Autowired
    private AsyncUserRepository userRepository;

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

        return CompletableFuture.supplyAsync(() ->
            postRepository.findByAuthorId(parent.getId())
        );
    }
}
```

#### Parallel Execution
```java
public class ParallelResolver {

    @Resolver(type = "User", field = "stats")
    public CompletableFuture<UserStats> getStats(
            User user,
            Map<String, Object> args,
            DataFetchingContext ctx,
            FieldInfo info) {

        // Execute multiple queries in parallel
        CompletableFuture<Integer> postCount =
            CompletableFuture.supplyAsync(() ->
                postRepository.countByAuthor(user.getId())
            );

        CompletableFuture<Integer> commentCount =
            CompletableFuture.supplyAsync(() ->
                commentRepository.countByAuthor(user.getId())
            );

        CompletableFuture<Integer> followerCount =
            CompletableFuture.supplyAsync(() ->
                followerRepository.countFollowers(user.getId())
            );

        // Combine results
        return CompletableFuture.allOf(postCount, commentCount, followerCount)
            .thenApply(v -> new UserStats(
                postCount.join(),
                commentCount.join(),
                followerCount.join()
            ));
    }
}
```

### Part 6: Resolver Patterns

#### Pattern 1: Computed Fields
```graphql
type User {
    firstName: String!
    lastName: String!
    fullName: String!  # Computed from firstName + lastName
}
```

```java
@Resolver(type = "User", field = "fullName")
public String getFullName(User user) {
    return user.getFirstName() + " " + user.getLastName();
}
```

#### Pattern 2: Aggregation Fields
```graphql
type Post {
    likeCount: Int!     # Aggregate from likes table
    commentCount: Int!  # Aggregate from comments table
}
```

```java
@Resolver(type = "Post", field = "likeCount")
public Integer getLikeCount(Post post, DataFetchingContext ctx) {
    return ctx.getDataLoader("likeCounts")
        .load(post.getId())
        .join();
}
```

#### Pattern 3: Delegating to Other Services
```graphql
type User {
    recommendations: [Post!]!  # From recommendation service
}
```

```java
@Resolver(type = "User", field = "recommendations")
public CompletableFuture<List<Post>> getRecommendations(
        User user,
        DataFetchingContext ctx) {

    // Call external recommendation service
    return recommendationServiceClient
        .getRecommendations(user.getId())
        .thenCompose(postIds ->
            ctx.getDataLoader("posts").loadMany(postIds)
        );
}
```

#### Pattern 4: Authorization in Resolvers
```java
@Resolver(type = "User", field = "email")
public String getEmail(User user, DataFetchingContext ctx) {
    User currentUser = ctx.getCurrentUser();

    // Only show email to the user themselves or admins
    if (currentUser.getId().equals(user.getId()) ||
        currentUser.isAdmin()) {
        return user.getEmail();
    }

    throw new UnauthorizedException("Cannot access email");
}
```

#### Pattern 5: Pagination
```graphql
type Query {
    posts(first: Int, after: String): PostConnection!
}

type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
}

type PostEdge {
    node: Post!
    cursor: String!
}
```

```java
@Resolver(type = "Query", field = "posts")
public PostConnection getPosts(
        Object parent,
        Map<String, Object> args,
        DataFetchingContext ctx) {

    Integer first = (Integer) args.get("first");
    String after = (String) args.get("after");

    // Decode cursor
    Long afterId = after != null
        ? decodeCursor(after)
        : 0L;

    // Fetch posts
    List<Post> posts = postRepository
        .findAfter(afterId, first + 1);  // Fetch one extra to check hasNext

    boolean hasNext = posts.size() > first;
    if (hasNext) {
        posts = posts.subList(0, first);
    }

    // Build edges
    List<PostEdge> edges = posts.stream()
        .map(post -> new PostEdge(
            post,
            encodeCursor(post.getId())
        ))
        .collect(Collectors.toList());

    // Build page info
    PageInfo pageInfo = new PageInfo(
        hasNext,
        !edges.isEmpty() ? edges.get(0).getCursor() : null,
        !edges.isEmpty() ? edges.get(edges.size() - 1).getCursor() : null
    );

    return new PostConnection(edges, pageInfo);
}
```

### Part 7: Error Handling in Resolvers

#### Throwing Exceptions
```java
@Resolver(type = "Query", field = "user")
public User getUser(Object parent, Map<String, Object> args) {
    Long id = (Long) args.get("id");

    return userRepository.findById(id)
        .orElseThrow(() ->
            new ResourceNotFoundException("User not found: " + id)
        );
}
```

#### Custom Error Types
```java
public class GraphQLError {
    private String message;
    private List<Location> locations;
    private List<String> path;
    private Map<String, Object> extensions;
}

public class ResourceNotFoundException extends RuntimeException
        implements GraphQLErrorProvider {

    @Override
    public GraphQLError toGraphQLError() {
        return GraphQLError.builder()
            .message(getMessage())
            .extensions(Map.of(
                "code", "RESOURCE_NOT_FOUND",
                "timestamp", Instant.now()
            ))
            .build();
    }
}
```

#### Partial Failures
```java
@Resolver(type = "User", field = "posts")
public List<Post> getPosts(User user, DataFetchingContext ctx) {
    try {
        return postRepository.findByAuthorId(user.getId());
    } catch (DatabaseException e) {
        // Log error but return partial result
        logger.error("Failed to fetch posts for user: " + user.getId(), e);

        // Return empty list and error will be in "errors" array
        ctx.addError(new GraphQLError(
            "Failed to fetch posts: " + e.getMessage(),
            ctx.getPath()
        ));

        return Collections.emptyList();
    }
}
```

### Part 8: Testing Resolvers

#### Unit Testing
```java
@Test
public void testUserPostsResolver() {
    // Setup
    User user = new User(1L, "Alice");
    Post post1 = new Post(1L, "Title 1");
    Post post2 = new Post(2L, "Title 2");

    when(postRepository.findByAuthorId(1L))
        .thenReturn(List.of(post1, post2));

    // Execute resolver
    UserResolvers resolver = new UserResolvers(postRepository);
    List<Post> result = resolver.getPosts(user, Map.of(), mockContext, mockInfo);

    // Verify
    assertEquals(2, result.size());
    assertEquals("Title 1", result.get(0).getTitle());
}
```

#### Integration Testing
```java
@SpringBootTest
@AutoConfigureGraphQLTester
public class UserResolverIntegrationTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @Test
    public void testUserQuery() {
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
            .path("user.posts").entityList(Post.class).hasSize(2);
    }
}
```

### Key Insights

1. **Resolvers are functions** - simple, testable, composable
2. **One resolver per field** - fine-grained control
3. **Async by default** - non-blocking I/O
4. **Context is powerful** - share data across resolvers
5. **Errors are first-class** - partial failures are okay

### Exercises

1. **Write a resolver chain** for a 3-level nested query
2. **Implement async resolver** using CompletableFuture
3. **Build context object** with authentication and caching
4. **Create computed field resolver** that aggregates data
5. **Handle errors gracefully** - implement partial failure handling

---

**Next:** [Chapter 8: Schema Design - SDL from Scratch í](08-schema-design.md)

**Previous:** [ê Chapter 6: The Graph Mental Model - Thinking in Relationships](06-graph-mental-model.md)
