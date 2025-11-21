# Chapter 20: Client Implementation Patterns

> "A well-designed GraphQL server is only half the story. Effective client implementation unlocks the true power of GraphQL."

---

## Why Client Patterns Matter

Server-side GraphQL is impressive, but the real benefits emerge when clients leverage GraphQL's capabilities:

- **Request exactly what you need** (no over/under-fetching)
- **Batch multiple resources** in a single request
- **Type safety** with code generation
- **Normalized caching** for optimal performance
- **Optimistic updates** for responsive UIs

This chapter explores how to build robust GraphQL clients in Java and Android.

---

## GraphQL Client Options

### Option 1: Plain HTTP Client

The simplest approach—use any HTTP client:

```java
public class SimpleGraphQLClient {

    private final HttpClient httpClient;
    private final String endpoint;
    private final ObjectMapper objectMapper;

    public SimpleGraphQLClient(String endpoint) {
        this.endpoint = endpoint;
        this.httpClient = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(Duration.ofSeconds(10))
            .build();
        this.objectMapper = new ObjectMapper();
    }

    public <T> CompletableFuture<T> query(
            String query,
            Map<String, Object> variables,
            Class<T> responseType) {

        try {
            // Build request body
            Map<String, Object> requestBody = new HashMap<>();
            requestBody.put("query", query);
            requestBody.put("variables", variables);

            String json = objectMapper.writeValueAsString(requestBody);

            // Create HTTP request
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(endpoint))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(json))
                .build();

            // Send request
            return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(response -> parseResponse(response, responseType));

        } catch (JsonProcessingException e) {
            return CompletableFuture.failedFuture(e);
        }
    }

    private <T> T parseResponse(HttpResponse<String> response, Class<T> responseType) {
        try {
            if (response.statusCode() != 200) {
                throw new GraphQLException(
                    "HTTP error: " + response.statusCode()
                );
            }

            // Parse GraphQL response
            JsonNode root = objectMapper.readTree(response.body());

            if (root.has("errors")) {
                JsonNode errors = root.get("errors");
                throw new GraphQLException(
                    "GraphQL errors: " + errors.toString()
                );
            }

            JsonNode data = root.get("data");

            return objectMapper.treeToValue(data, responseType);

        } catch (JsonProcessingException e) {
            throw new GraphQLException("Failed to parse response", e);
        }
    }
}
```

### Using the Simple Client

```java
public class Example {

    public static void main(String[] args) {
        SimpleGraphQLClient client = new SimpleGraphQLClient("https://api.example.com/graphql");

        String query = """
            query GetUser($id: ID!) {
                user(id: $id) {
                    id
                    name
                    email
                }
            }
            """;

        Map<String, Object> variables = Map.of("id", "123");

        client.query(query, variables, UserResponse.class)
            .thenAccept(response -> {
                System.out.println("User: " + response.getUser().getName());
            })
            .exceptionally(e -> {
                System.err.println("Error: " + e.getMessage());
                return null;
            });
    }
}

// Response DTOs
public class UserResponse {
    private User user;

    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}

public class User {
    private String id;
    private String name;
    private String email;

    // Getters and setters
}
```

**Pros:**
- Simple, no dependencies
- Full control over requests
- Works anywhere

**Cons:**
- Manual JSON mapping
- No type safety
- No caching
- Verbose

---

## Option 2: Apollo Android (Type-Safe Client)

For Android apps, Apollo Client is the gold standard.

### Setup

```gradle
// build.gradle (app)
plugins {
    id 'com.apollographql.apollo3' version '3.8.2'
}

dependencies {
    implementation 'com.apollographql.apollo3:apollo-runtime:3.8.2'
}

apollo {
    service("api") {
        packageName.set("com.example.graphql")
        // Schema location
        schemaFile.set(file("src/main/graphql/schema.graphqls"))
    }
}
```

### Define Queries

```graphql
# src/main/graphql/GetUser.graphql
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

### Generated Code

Apollo generates type-safe classes:

```java
// Generated by Apollo
public class GetUserQuery {
    private final String id;

    public GetUserQuery(String id) {
        this.id = id;
    }

    // Generated data classes
    public static class Data {
        public User user;
    }

    public static class User {
        public String id;
        public String name;
        public String email;
        public List<Post> posts;
    }

    public static class Post {
        public String id;
        public String title;
    }
}
```

### Using Apollo Client

```java
public class GraphQLRepository {

    private final ApolloClient apolloClient;

    public GraphQLRepository() {
        apolloClient = new ApolloClient.Builder()
            .serverUrl("https://api.example.com/graphql")
            .build();
    }

    public CompletableFuture<GetUserQuery.User> getUser(String userId) {
        GetUserQuery query = new GetUserQuery(userId);

        return apolloClient.query(query)
            .toFlow()
            .single()
            .toCompletableFuture()
            .thenApply(response -> {
                if (response.hasErrors()) {
                    throw new GraphQLException(
                        response.errors.get(0).message
                    );
                }

                return response.data.user;
            });
    }
}
```

**Pros:**
- **Type safety** - Compile-time checking
- **Code generation** - Less boilerplate
- **Built-in caching** - Normalized cache
- **Error handling** - Structured errors
- **Optimistic updates** - UI responsiveness

**Cons:**
- Build-time code generation required
- Additional dependency
- Learning curve

---

## Authentication

### Adding Headers

```java
public class AuthenticatedGraphQLClient {

    private final HttpClient httpClient;
    private final String endpoint;
    private final String authToken;

    public AuthenticatedGraphQLClient(String endpoint, String authToken) {
        this.endpoint = endpoint;
        this.authToken = authToken;
        this.httpClient = HttpClient.newHttpClient();
    }

    public <T> CompletableFuture<T> query(
            String query,
            Map<String, Object> variables,
            Class<T> responseType) {

        // Build request with Authorization header
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(endpoint))
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer " + authToken)
            .POST(buildBody(query, variables))
            .build();

        return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(response -> parseResponse(response, responseType));
    }
}
```

### With Apollo Client

```java
apolloClient = new ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .addHttpInterceptor(new AuthorizationInterceptor(authToken))
    .build();

class AuthorizationInterceptor implements ApolloInterceptor {
    private final String token;

    AuthorizationInterceptor(String token) {
        this.token = token;
    }

    @Override
    public <D extends Operation.Data> Flow<ApolloResponse<D>> intercept(
            ApolloRequest<D> request,
            ApolloInterceptorChain chain) {

        return chain.proceed(
            request.newBuilder()
                .addHttpHeader("Authorization", "Bearer " + token)
                .build()
        );
    }
}
```

---

## Error Handling

GraphQL has two types of errors:

1. **Network errors** - Failed HTTP request
2. **GraphQL errors** - Query succeeded, but returned errors

### Handling Both

```java
public class RobustGraphQLClient {

    public <T> CompletableFuture<T> query(
            String query,
            Map<String, Object> variables,
            Class<T> responseType) {

        return executeRequest(query, variables)
            .thenApply(response -> {
                // Check for GraphQL errors
                if (response.has("errors")) {
                    List<GraphQLError> errors = parseErrors(response.get("errors"));
                    throw new GraphQLException(errors);
                }

                // Parse data
                JsonNode data = response.get("data");

                if (data == null || data.isNull()) {
                    throw new GraphQLException("No data in response");
                }

                return parseData(data, responseType);
            })
            .exceptionally(e -> {
                // Handle network errors
                if (e.getCause() instanceof IOException) {
                    throw new NetworkException("Network error", e);
                }

                // Handle parsing errors
                if (e.getCause() instanceof JsonProcessingException) {
                    throw new ParseException("Failed to parse response", e);
                }

                // Rethrow other errors
                throw new RuntimeException(e);
            });
    }

    private List<GraphQLError> parseErrors(JsonNode errorsNode) {
        List<GraphQLError> errors = new ArrayList<>();

        for (JsonNode errorNode : errorsNode) {
            String message = errorNode.get("message").asText();
            String path = errorNode.has("path")
                ? errorNode.get("path").toString()
                : null;

            errors.add(new GraphQLError(message, path));
        }

        return errors;
    }
}

public class GraphQLError {
    private final String message;
    private final String path;

    public GraphQLError(String message, String path) {
        this.message = message;
        this.path = path;
    }

    public String getMessage() { return message; }
    public String getPath() { return path; }
}
```

### Retry Logic

```java
public class RetryableGraphQLClient {

    private static final int MAX_RETRIES = 3;
    private static final long INITIAL_BACKOFF_MS = 1000;

    public <T> CompletableFuture<T> queryWithRetry(
            String query,
            Map<String, Object> variables,
            Class<T> responseType) {

        return queryWithRetry(query, variables, responseType, 0);
    }

    private <T> CompletableFuture<T> queryWithRetry(
            String query,
            Map<String, Object> variables,
            Class<T> responseType,
            int attemptNumber) {

        return query(query, variables, responseType)
            .exceptionally(e -> {
                // Don't retry on validation errors
                if (e instanceof GraphQLException) {
                    throw (GraphQLException) e;
                }

                // Retry on network errors
                if (attemptNumber < MAX_RETRIES) {
                    long backoffMs = INITIAL_BACKOFF_MS * (1L << attemptNumber);

                    try {
                        Thread.sleep(backoffMs);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException(ie);
                    }

                    // Retry
                    return queryWithRetry(
                        query,
                        variables,
                        responseType,
                        attemptNumber + 1
                    ).join();
                }

                // Max retries exceeded
                throw new RuntimeException("Max retries exceeded", e);
            });
    }
}
```

---

## Caching Strategies

### Simple In-Memory Cache

```java
public class CachingGraphQLClient {

    private final SimpleGraphQLClient client;
    private final Map<String, CacheEntry<?>> cache = new ConcurrentHashMap<>();
    private final long cacheExpirationMs = 60_000; // 1 minute

    public <T> CompletableFuture<T> query(
            String query,
            Map<String, Object> variables,
            Class<T> responseType,
            boolean useCache) {

        if (!useCache) {
            return client.query(query, variables, responseType);
        }

        // Generate cache key
        String cacheKey = generateCacheKey(query, variables);

        // Check cache
        CacheEntry<?> entry = cache.get(cacheKey);

        if (entry != null && !entry.isExpired()) {
            // Cache hit
            return CompletableFuture.completedFuture((T) entry.getData());
        }

        // Cache miss - fetch from server
        return client.query(query, variables, responseType)
            .thenApply(data -> {
                // Store in cache
                cache.put(cacheKey, new CacheEntry<>(data, cacheExpirationMs));
                return data;
            });
    }

    private String generateCacheKey(String query, Map<String, Object> variables) {
        try {
            String json = new ObjectMapper().writeValueAsString(
                Map.of("query", query, "variables", variables)
            );

            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(json.getBytes(StandardCharsets.UTF_8));

            return Base64.getEncoder().encodeToString(hash);

        } catch (Exception e) {
            throw new RuntimeException("Failed to generate cache key", e);
        }
    }

    public void invalidateCache(String query, Map<String, Object> variables) {
        String cacheKey = generateCacheKey(query, variables);
        cache.remove(cacheKey);
    }

    public void clearCache() {
        cache.clear();
    }
}

class CacheEntry<T> {
    private final T data;
    private final long expirationTime;

    CacheEntry(T data, long ttlMs) {
        this.data = data;
        this.expirationTime = System.currentTimeMillis() + ttlMs;
    }

    T getData() { return data; }

    boolean isExpired() {
        return System.currentTimeMillis() > expirationTime;
    }
}
```

### Apollo Normalized Cache

Apollo maintains a normalized cache:

```java
// Configure normalized cache
apolloClient = new ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .normalizedCache(
        new MemoryCacheFactory(maxSizeBytes = 10 * 1024 * 1024)  // 10 MB
    )
    .build();

// Fetch with cache
apolloClient.query(new GetUserQuery("123"))
    .fetchPolicy(FetchPolicy.CacheFirst)  // Try cache first
    .toFlow()
    .collect(response -> {
        // Will use cached data if available
        User user = response.data.user;
    });

// Invalidate cache after mutation
apolloClient.mutation(new UpdateUserMutation(...))
    .execute()
    .thenAccept(response -> {
        // Apollo automatically updates cache
        // based on mutation response
    });
```

**Cache policies:**
- `CacheFirst` - Use cache if available, fallback to network
- `CacheOnly` - Only use cache, never network
- `NetworkFirst` - Fetch from network, fallback to cache
- `NetworkOnly` - Always fetch from network
- `CacheAndNetwork` - Return cached data immediately, then fetch fresh

---

## Pagination

### Offset-Based Pagination

```java
public class PaginatedQuery {

    public CompletableFuture<PostsPage> getPosts(int page, int pageSize) {
        String query = """
            query GetPosts($offset: Int!, $limit: Int!) {
                posts(offset: $offset, limit: $limit) {
                    id
                    title
                    content
                }
                postsCount
            }
            """;

        Map<String, Object> variables = Map.of(
            "offset", page * pageSize,
            "limit", pageSize
        );

        return client.query(query, variables, PostsPage.class);
    }
}

public class PostsPage {
    private List<Post> posts;
    private int postsCount;

    public boolean hasNextPage(int currentPage, int pageSize) {
        return (currentPage + 1) * pageSize < postsCount;
    }

    // Getters
}
```

### Cursor-Based Pagination (Relay-style)

```java
public class CursorPaginatedQuery {

    public CompletableFuture<PostsConnection> getPosts(
            String afterCursor,
            int first) {

        String query = """
            query GetPosts($after: String, $first: Int!) {
                posts(after: $after, first: $first) {
                    edges {
                        node {
                            id
                            title
                            content
                        }
                        cursor
                    }
                    pageInfo {
                        hasNextPage
                        endCursor
                    }
                }
            }
            """;

        Map<String, Object> variables = new HashMap<>();
        variables.put("first", first);
        if (afterCursor != null) {
            variables.put("after", afterCursor);
        }

        return client.query(query, variables, PostsResponse.class)
            .thenApply(response -> response.getPosts());
    }

    // Load all pages
    public CompletableFuture<List<Post>> getAllPosts() {
        List<Post> allPosts = new ArrayList<>();

        return loadPage(null, allPosts)
            .thenApply(ignored -> allPosts);
    }

    private CompletableFuture<Void> loadPage(
            String cursor,
            List<Post> accumulator) {

        return getPosts(cursor, 50)
            .thenCompose(connection -> {
                // Add posts to accumulator
                connection.getEdges().forEach(edge ->
                    accumulator.add(edge.getNode())
                );

                // Load next page if available
                if (connection.getPageInfo().isHasNextPage()) {
                    String nextCursor = connection.getPageInfo().getEndCursor();
                    return loadPage(nextCursor, accumulator);
                }

                return CompletableFuture.completedFuture(null);
            });
    }
}

// Response structures
public class PostsConnection {
    private List<PostEdge> edges;
    private PageInfo pageInfo;

    // Getters
}

public class PostEdge {
    private Post node;
    private String cursor;

    // Getters
}

public class PageInfo {
    private boolean hasNextPage;
    private String endCursor;

    // Getters
}
```

---

## Optimistic Updates

Update UI immediately, rollback if mutation fails:

```java
public class OptimisticUpdateExample {

    private final ApolloClient apolloClient;
    private final ObservableField<User> userField = new ObservableField<>();

    public void updateUserName(String userId, String newName) {
        // 1. Update UI optimistically
        User currentUser = userField.get();
        User optimisticUser = currentUser.copy(name = newName);
        userField.set(optimisticUser);

        // 2. Send mutation to server
        UpdateUserMutation mutation = new UpdateUserMutation(userId, newName);

        apolloClient.mutation(mutation)
            .execute()
            .thenAccept(response -> {
                if (response.hasErrors()) {
                    // 3. Rollback on error
                    userField.set(currentUser);
                    showError("Failed to update user");
                } else {
                    // 4. Update with server response
                    userField.set(response.data.updateUser);
                }
            })
            .exceptionally(e -> {
                // 3. Rollback on network error
                userField.set(currentUser);
                showError("Network error");
                return null;
            });
    }
}
```

### With Apollo's Built-in Support

```java
apolloClient.mutation(new UpdateUserMutation(userId, newName))
    .optimisticUpdates(Optional.of(new UpdateUserMutation.Data(
        new UpdateUserMutation.UpdateUser(
            userId,
            newName,
            currentUser.email
        )
    )))
    .execute()
    .thenAccept(response -> {
        // Apollo automatically:
        // 1. Applies optimistic update to cache
        // 2. Updates UI via cache observers
        // 3. Replaces with real data when response arrives
        // 4. Rolls back if mutation fails
    });
```

---

## File Uploads

GraphQL doesn't natively support file uploads, but we can use multipart requests:

```java
public class FileUploadClient {

    public CompletableFuture<String> uploadFile(File file) {
        String mutation = """
            mutation UploadFile($file: Upload!) {
                uploadFile(file: $file) {
                    url
                }
            }
            """;

        // Create multipart request
        MultipartEntityBuilder builder = MultipartEntityBuilder.create();

        // Add operations part
        Map<String, Object> operations = Map.of(
            "query", mutation,
            "variables", Map.of("file", null)
        );

        builder.addTextBody(
            "operations",
            new ObjectMapper().writeValueAsString(operations),
            ContentType.APPLICATION_JSON
        );

        // Add map part
        Map<String, List<String>> map = Map.of(
            "0", List.of("variables.file")
        );

        builder.addTextBody(
            "map",
            new ObjectMapper().writeValueAsString(map),
            ContentType.APPLICATION_JSON
        );

        // Add file part
        builder.addBinaryBody(
            "0",
            file,
            ContentType.APPLICATION_OCTET_STREAM,
            file.getName()
        );

        // Send request
        HttpPost httpPost = new HttpPost("https://api.example.com/graphql");
        httpPost.setEntity(builder.build());

        return HttpAsyncClients.createDefault()
            .execute(httpPost, new FutureCallback<HttpResponse>() {
                @Override
                public void completed(HttpResponse response) {
                    // Parse response
                }

                @Override
                public void failed(Exception e) {
                    // Handle error
                }

                @Override
                public void cancelled() {
                    // Handle cancellation
                }
            });
    }
}
```

---

## Subscriptions (WebSocket)

For real-time updates:

```java
public class SubscriptionClient {

    private final ApolloClient apolloClient;

    public SubscriptionClient() {
        // Configure WebSocket
        apolloClient = new ApolloClient.Builder()
            .serverUrl("https://api.example.com/graphql")
            .subscriptionNetworkTransport(
                new WebSocketNetworkTransport.Builder()
                    .serverUrl("wss://api.example.com/graphql")
                    .build()
            )
            .build();
    }

    public Subscription subscribeToMessages(String chatId, MessageCallback callback) {
        MessageSubscription subscription = new MessageSubscription(chatId);

        return apolloClient.subscription(subscription)
            .toFlow()
            .collect(response -> {
                if (response.data != null) {
                    Message message = response.data.messageReceived;
                    callback.onMessage(message);
                }

                if (response.errors != null) {
                    callback.onError(response.errors);
                }
            });
    }
}

interface MessageCallback {
    void onMessage(Message message);
    void onError(List<Error> errors);
}
```

---

## Android Integration Patterns

### ViewModel with GraphQL

```java
public class UserViewModel extends ViewModel {

    private final GraphQLRepository repository;
    private final MutableLiveData<User> userLiveData = new MutableLiveData<>();
    private final MutableLiveData<String> errorLiveData = new MutableLiveData<>();

    public UserViewModel(GraphQLRepository repository) {
        this.repository = repository;
    }

    public LiveData<User> getUser() {
        return userLiveData;
    }

    public LiveData<String> getError() {
        return errorLiveData;
    }

    public void loadUser(String userId) {
        repository.getUser(userId)
            .thenAccept(user -> {
                userLiveData.postValue(user);
            })
            .exceptionally(e -> {
                errorLiveData.postValue(e.getMessage());
                return null;
            });
    }

    public void updateUser(String userId, String newName) {
        repository.updateUser(userId, newName)
            .thenAccept(user -> {
                userLiveData.postValue(user);
            })
            .exceptionally(e -> {
                errorLiveData.postValue(e.getMessage());
                return null;
            });
    }
}
```

### With Kotlin Coroutines

```kotlin
class UserViewModel(
    private val repository: GraphQLRepository
) : ViewModel() {

    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()

    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(userId).await()
                _user.value = user
            } catch (e: Exception) {
                _error.value = e.message
            }
        }
    }
}
```

---

## Key Takeaways

1. **Simple HTTP client** - Easy to implement, no dependencies
2. **Apollo Client** - Type-safe, feature-rich, recommended for Android
3. **Authentication** - Add headers via interceptors
4. **Error handling** - Handle both network and GraphQL errors
5. **Retry logic** - Implement exponential backoff for network errors
6. **Caching** - Use normalized cache for optimal performance
7. **Pagination** - Cursor-based preferred over offset-based
8. **Optimistic updates** - Update UI immediately for better UX
9. **File uploads** - Use multipart requests
10. **Subscriptions** - WebSocket for real-time updates

The client is where GraphQL's benefits truly shine. Invest in proper client implementation to unlock GraphQL's full potential.

---

**Next:** [Chapter 21: Production Considerations →](21-production-considerations.md)
**Previous:** [← Chapter 19: Building a Simple GraphQL Server in Java](19-building-server.md)
