# Chapter 10: Execution Engine - How Queries Actually Run

> "Parsing tells us what the client wants. Execution delivers it. The gap between these two is where all the interesting work happens."

---

## From AST to Response: The Execution Journey

We have a validated AST. We know the query is legal and safe. Now what?

Consider this query:

```graphql
{
  user(id: "123") {
    name
    posts(limit: 5) {
      title
      author {
        name
      }
    }
  }
}
```

Our execution engine must:
1. **Resolve** the `user` field (database lookup)
2. **Resolve** the `name` field (extract from user object)
3. **Resolve** the `posts` field (another database query)
4. For each post, **resolve** `title` and `author`
5. For each author, **resolve** `name`
6. **Assemble** all this into a nested JSON response

The order matters. The concurrency matters. Error handling matters. Let's build an execution engine that handles all of this correctly and efficiently.

---

## Part 1: Execution Model Overview

### The Three Phases

GraphQL execution has three distinct phases:

```java
public class GraphQLEngine {
    public ExecutionResult execute(String queryString, Object rootValue,
                                   Map<String, Object> variables) {
        // Phase 1: Parse
        Document document = new Parser(queryString).parseDocument();

        // Phase 2: Validate
        List<ValidationError> errors = new Validator(schema).validate(document);
        if (!errors.isEmpty()) {
            return ExecutionResult.errors(errors);
        }

        // Phase 3: Execute
        return new Executor(schema, rootValue, variables).execute(document);
    }
}
```

We've covered phases 1-2. Let's dive into phase 3.

### Breadth-First vs Depth-First Execution

How should we traverse the query tree?

**Option 1: Depth-First**

```
Execute user
  ├─ Execute name (wait for user)
  └─ Execute posts (wait for user)
      ├─ Execute posts[0].title (wait for posts)
      ├─ Execute posts[0].author (wait for posts)
      │   └─ Execute posts[0].author.name (wait for author)
      ├─ Execute posts[1].title (wait for posts)
      ...
```

**Problem**: We can't parallelize sibling fields. We wait for `name` to finish before starting `posts`.

**Option 2: Breadth-First**

```
Level 0: Execute user
Level 1: Execute name, posts (in parallel!)
Level 2: Execute posts[0].title, posts[0].author, posts[1].title, ... (in parallel!)
Level 3: Execute posts[0].author.name, posts[1].author.name, ... (in parallel!)
```

**Advantage**: Maximum parallelism. All fields at the same level can run concurrently.

**GraphQL uses breadth-first execution** to maximize concurrency.

### Why Breadth-First Wins

Consider this query:

```graphql
{
  fastField    # takes 10ms
  slowField    # takes 1000ms
  anotherFast  # takes 10ms
}
```

**Depth-first total time**: 10ms + 1000ms + 10ms = **1020ms**

**Breadth-first total time**: max(10ms, 1000ms, 10ms) = **1000ms**

All three fields start in parallel. We wait only for the slowest one.

---

## Part 2: Field Resolution

### The Core Resolution Loop

Execution is fundamentally about resolving fields:

```java
public class Executor {
    private final Schema schema;
    private final Object rootValue;
    private final Map<String, Object> variables;

    public ExecutionResult execute(Document document) {
        OperationDefinition operation = document.getOperations().get(0);
        TypeDefinition rootType = getRootType(operation.getOperationType());

        // Start execution at the root
        Map<String, Object> result = executeSelectionSet(
            operation.getSelectionSet(),
            rootType,
            rootValue,
            new ExecutionContext(variables)
        );

        return ExecutionResult.success(result);
    }

    private Map<String, Object> executeSelectionSet(
            SelectionSet selectionSet,
            TypeDefinition parentType,
            Object parentValue,
            ExecutionContext context) {

        Map<String, Object> result = new LinkedHashMap<>();

        // Group fields by response key (handling aliases)
        Map<String, List<Field>> fieldsByKey = groupFieldsByResponseKey(selectionSet);

        for (Map.Entry<String, List<Field>> entry : fieldsByKey.entrySet()) {
            String responseKey = entry.getKey();
            List<Field> fields = entry.getValue(); // Multiple if same field requested twice

            Field field = fields.get(0); // All fields with same key are equivalent
            Object fieldValue = executeField(field, parentType, parentValue, context);

            result.put(responseKey, fieldValue);
        }

        return result;
    }
}
```

### Resolver Lookup and Invocation

Each field has a resolver function:

```java
private Object executeField(Field field,
                           TypeDefinition parentType,
                           Object parentValue,
                           ExecutionContext context) {

    // 1. Look up the field definition
    FieldDefinition fieldDef = parentType.getField(field.getName());
    if (fieldDef == null) {
        throw new ExecutionException("Field not found: " + field.getName());
    }

    // 2. Look up the resolver function
    FieldResolver resolver = getResolver(parentType, field.getName());

    // 3. Prepare arguments
    Map<String, Object> args = resolveArguments(field, fieldDef, context);

    // 4. Invoke the resolver
    Object resolvedValue;
    try {
        resolvedValue = resolver.resolve(parentValue, args, context);
    } catch (Exception e) {
        return handleFieldError(field, e);
    }

    // 5. Complete the value (handle nested selections)
    return completeValue(fieldDef.getType(), resolvedValue, field, context);
}
```

### Resolver Functions

A resolver is just a function that fetches data:

```java
@FunctionalInterface
public interface FieldResolver {
    Object resolve(Object parent, Map<String, Object> args, ExecutionContext context);
}
```

**Example resolvers:**

```java
// Simple property access
FieldResolver nameResolver = (parent, args, context) -> {
    User user = (User) parent;
    return user.getName();
};

// Database query
FieldResolver userResolver = (parent, args, context) -> {
    String userId = (String) args.get("id");
    return context.getDatabase().getUserById(userId);
};

// Computed field
FieldResolver fullNameResolver = (parent, args, context) -> {
    User user = (User) parent;
    return user.getFirstName() + " " + user.getLastName();
};

// Async resolver
FieldResolver postsResolver = (parent, args, context) -> {
    User user = (User) parent;
    int limit = (int) args.getOrDefault("limit", 10);

    return context.getDatabase()
        .getPostsByAuthorId(user.getId(), limit)
        .thenApply(posts -> posts);  // Returns CompletableFuture<List<Post>>
};
```

### Default Field Resolver

If no resolver is specified, use a default one:

```java
public class DefaultFieldResolver implements FieldResolver {
    @Override
    public Object resolve(Object parent, Map<String, Object> args,
                         ExecutionContext context) {
        if (parent == null) {
            return null;
        }

        String fieldName = context.getCurrentField().getName();

        // Try Map access
        if (parent instanceof Map) {
            return ((Map<?, ?>) parent).get(fieldName);
        }

        // Try getter method
        try {
            Method getter = parent.getClass().getMethod("get" + capitalize(fieldName));
            return getter.invoke(parent);
        } catch (NoSuchMethodException e) {
            // Try direct field access
            try {
                Field field = parent.getClass().getDeclaredField(fieldName);
                field.setAccessible(true);
                return field.get(parent);
            } catch (Exception ex) {
                return null;
            }
        } catch (Exception e) {
            throw new ExecutionException("Failed to resolve field: " + fieldName, e);
        }
    }
}
```

### Completing Values: Scalar vs Object

After resolving, we need to **complete** the value based on its type:

```java
private Object completeValue(TypeReference type,
                             Object resolvedValue,
                             Field field,
                             ExecutionContext context) {

    // Handle null
    if (resolvedValue == null) {
        if (type.isNonNull()) {
            throw new NonNullViolationException(field.getName());
        }
        return null;
    }

    // Unwrap NonNull
    if (type.isNonNull()) {
        return completeValue(type.getWrappedType(), resolvedValue, field, context);
    }

    // Handle List
    if (type.isList()) {
        return completeListValue(type, resolvedValue, field, context);
    }

    // Handle Object/Interface/Union (needs nested selection)
    TypeDefinition typeDef = schema.getType(type.getName());
    if (typeDef.isObjectType() || typeDef.isInterfaceType() || typeDef.isUnionType()) {
        return completeObjectValue(typeDef, resolvedValue, field, context);
    }

    // Handle Scalar/Enum (leaf value)
    return completeLeafValue(type, resolvedValue);
}
```

### Handling Null Values

Null propagation is tricky in GraphQL:

```java
// Non-null field returns null
{
  user {
    name  // String! (non-null)
  }
}

// If name resolver returns null:
// 1. Throw NonNullViolationException
// 2. Set user = null (propagate up)
// 3. If user is also non-null, propagate further
```

**Implementation:**

```java
private Object completeValue(TypeReference type, Object value,
                             Field field, ExecutionContext context) {
    if (value == null) {
        if (type.isNonNull()) {
            // Non-null field returned null - this is an error
            throw new NonNullViolationException(
                "Non-null field '" + field.getName() + "' returned null",
                field.getLocation()
            );
        }
        return null;
    }

    // ... rest of completion logic
}

// Caller catches exception and propagates null upward:
private Object executeField(Field field, TypeDefinition parentType,
                            Object parentValue, ExecutionContext context) {
    try {
        Object resolved = resolver.resolve(parentValue, args, context);
        return completeValue(fieldDef.getType(), resolved, field, context);
    } catch (NonNullViolationException e) {
        // Null violation propagates up
        if (parentType.isNonNull()) {
            throw e; // Keep propagating
        }
        return null; // Stop here, return null for this field
    }
}
```

### List Resolution

Lists need special handling:

```java
private List<Object> completeListValue(TypeReference listType,
                                       Object resolvedValue,
                                       Field field,
                                       ExecutionContext context) {

    if (!(resolvedValue instanceof Iterable)) {
        throw new ExecutionException("Expected iterable for list field");
    }

    Iterable<?> items = (Iterable<?>) resolvedValue;
    List<Object> result = new ArrayList<>();

    TypeReference itemType = listType.getWrappedType();

    for (Object item : items) {
        Object completedItem = completeValue(itemType, item, field, context);
        result.add(completedItem);
    }

    return result;
}
```

**Example:**

```graphql
{
  posts {  # [Post!]!
    title
  }
}
```

Execution:
1. Resolve `posts` → `List<Post>`
2. For each post, complete as object type → execute selection set `{ title }`
3. Return list of completed objects

---

## Part 3: Concurrent Execution

### The Challenge: Parallelizing Field Resolution

Fields at the same level can execute in parallel:

```graphql
{
  user {        # Level 1
    name        # Level 2 - can run in parallel with email and posts
    email       # Level 2
    posts {     # Level 2
      title     # Level 3
    }
  }
}
```

But we must **wait** for level N to complete before starting level N+1.

### Using CompletableFuture

Java's `CompletableFuture` is perfect for this:

```java
private CompletableFuture<Map<String, Object>> executeSelectionSetAsync(
        SelectionSet selectionSet,
        TypeDefinition parentType,
        Object parentValue,
        ExecutionContext context) {

    Map<String, List<Field>> fieldsByKey = groupFieldsByResponseKey(selectionSet);

    // Create futures for all fields at this level
    List<CompletableFuture<FieldResult>> fieldFutures = new ArrayList<>();

    for (Map.Entry<String, List<Field>> entry : fieldsByKey.entrySet()) {
        String responseKey = entry.getKey();
        Field field = entry.getValue().get(0);

        CompletableFuture<FieldResult> fieldFuture = executeFieldAsync(
            field, parentType, parentValue, context
        ).thenApply(value -> new FieldResult(responseKey, value));

        fieldFutures.add(fieldFuture);
    }

    // Wait for all fields to complete
    return CompletableFuture.allOf(
        fieldFutures.toArray(new CompletableFuture[0])
    ).thenApply(v -> {
        Map<String, Object> result = new LinkedHashMap<>();
        for (CompletableFuture<FieldResult> future : fieldFutures) {
            FieldResult fr = future.join();
            result.put(fr.key, fr.value);
        }
        return result;
    });
}
```

### Async Field Resolution

Resolvers can return `CompletableFuture`:

```java
private CompletableFuture<Object> executeFieldAsync(
        Field field,
        TypeDefinition parentType,
        Object parentValue,
        ExecutionContext context) {

    FieldDefinition fieldDef = parentType.getField(field.getName());
    FieldResolver resolver = getResolver(parentType, field.getName());
    Map<String, Object> args = resolveArguments(field, fieldDef, context);

    // Invoke resolver
    Object resolverResult = resolver.resolve(parentValue, args, context);

    // Handle both sync and async resolvers
    CompletableFuture<Object> resolvedFuture;
    if (resolverResult instanceof CompletableFuture) {
        resolvedFuture = (CompletableFuture<Object>) resolverResult;
    } else {
        resolvedFuture = CompletableFuture.completedFuture(resolverResult);
    }

    // Complete the value
    return resolvedFuture.thenCompose(resolved ->
        completeValueAsync(fieldDef.getType(), resolved, field, context)
    );
}
```

### Thread Pool Configuration

Control concurrency with an executor:

```java
public class ExecutionContext {
    private final ExecutorService executorService;

    public ExecutionContext() {
        // Thread pool sized for I/O-bound work
        this.executorService = new ThreadPoolExecutor(
            10,  // core threads
            100, // max threads
            60L, TimeUnit.SECONDS,  // keepalive
            new LinkedBlockingQueue<>(1000),  // queue size
            new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
        );
    }

    public <T> CompletableFuture<T> supplyAsync(Supplier<T> supplier) {
        return CompletableFuture.supplyAsync(supplier, executorService);
    }

    public void shutdown() {
        executorService.shutdown();
        try {
            executorService.awaitTermination(30, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            executorService.shutdownNow();
        }
    }
}
```

### Orchestrating Async Resolvers

Complex example with multiple async dependencies:

```java
// Resolver that depends on multiple async calls
FieldResolver recommendedPostsResolver = (parent, args, context) -> {
    User user = (User) parent;

    // Three parallel async calls
    CompletableFuture<List<Post>> recentPosts =
        context.getDatabase().getRecentPostsByUser(user.getId());

    CompletableFuture<List<User>> following =
        context.getDatabase().getFollowingUsers(user.getId());

    CompletableFuture<List<String>> interests =
        context.getDatabase().getUserInterests(user.getId());

    // Combine results
    return CompletableFuture.allOf(recentPosts, following, interests)
        .thenCompose(v -> {
            List<Post> recent = recentPosts.join();
            List<User> followingList = following.join();
            List<String> interestList = interests.join();

            // Use all three to compute recommendations
            return context.getRecommendationEngine()
                .getRecommendations(recent, followingList, interestList);
        });
};
```

---

## Part 4: Execution Context

### Building the Context

The execution context carries request-scoped data:

```java
public class ExecutionContext {
    private final Map<String, Object> variables;
    private final Map<String, Object> contextValues;
    private final ExecutorService executorService;
    private final DataLoaderRegistry dataLoaders;
    private final Field currentField;
    private final List<Object> path;

    // Services
    private final Database database;
    private final AuthService authService;
    private final Cache cache;

    public ExecutionContext(Map<String, Object> variables,
                           Map<String, Object> contextValues) {
        this.variables = variables;
        this.contextValues = contextValues;
        this.executorService = createExecutorService();
        this.dataLoaders = new DataLoaderRegistry();
        this.path = new ArrayList<>();

        // Initialize services from context
        this.database = (Database) contextValues.get("database");
        this.authService = (AuthService) contextValues.get("authService");
        this.cache = (Cache) contextValues.get("cache");
    }

    public Object getVariable(String name) {
        return variables.get(name);
    }

    public <T> T getContextValue(String key) {
        return (T) contextValues.get(key);
    }

    public Database getDatabase() {
        return database;
    }

    public User getCurrentUser() {
        return authService.getCurrentUser();
    }

    public ExecutionContext withField(Field field) {
        ExecutionContext child = new ExecutionContext(variables, contextValues);
        child.currentField = field;
        child.path.addAll(this.path);
        child.path.add(field.getName());
        return child;
    }
}
```

### Passing Data Between Resolvers

Use context to share data:

```java
// Resolver that sets context data
FieldResolver userResolver = (parent, args, context) -> {
    String userId = (String) args.get("id");
    User user = context.getDatabase().getUserById(userId);

    // Store in context for later resolvers
    context.set("currentViewedUser", user);

    return user;
};

// Later resolver that uses it
FieldResolver canEditProfileResolver = (parent, args, context) -> {
    User viewedUser = context.get("currentViewedUser");
    User currentUser = context.getCurrentUser();

    return currentUser.isAdmin() || currentUser.getId().equals(viewedUser.getId());
};
```

### Request-Scoped Dependencies

Inject per-request dependencies:

```java
public ExecutionResult execute(String query, HttpServletRequest request) {
    Map<String, Object> contextValues = new HashMap<>();

    // Request-scoped services
    contextValues.put("database", databaseConnectionPool.getConnection());
    contextValues.put("authService", new AuthService(request));
    contextValues.put("request", request);
    contextValues.put("currentUser", extractUser(request));

    ExecutionContext context = new ExecutionContext(
        extractVariables(request),
        contextValues
    );

    try {
        return executor.execute(query, context);
    } finally {
        context.cleanup();
    }
}
```

### Context Cleanup

Always clean up resources:

```java
public class ExecutionContext implements AutoCloseable {
    @Override
    public void close() {
        // Shutdown executor
        executorService.shutdown();

        // Return database connection to pool
        Database db = (Database) contextValues.get("database");
        if (db != null) {
            db.close();
        }

        // Dispatch all pending DataLoader batches
        dataLoaders.dispatchAll();
    }
}

// Usage with try-with-resources
try (ExecutionContext context = new ExecutionContext(vars, contextVals)) {
    return executor.execute(query, context);
}
```

---

## Part 5: Directive Execution

### The @skip and @include Directives

Built-in directives control field execution:

```graphql
query GetUser($includeEmail: Boolean!) {
  user {
    name
    email @include(if: $includeEmail)
    posts @skip(if: false) {
      title
    }
  }
}
```

### Implementation

Check directives before executing a field:

```java
private CompletableFuture<Object> executeFieldAsync(
        Field field,
        TypeDefinition parentType,
        Object parentValue,
        ExecutionContext context) {

    // Check @skip directive
    for (Directive directive : field.getDirectives()) {
        if (directive.getName().equals("skip")) {
            boolean shouldSkip = evaluateDirectiveCondition(directive, context);
            if (shouldSkip) {
                return CompletableFuture.completedFuture(null);
            }
        }
    }

    // Check @include directive
    for (Directive directive : field.getDirectives()) {
        if (directive.getName().equals("include")) {
            boolean shouldInclude = evaluateDirectiveCondition(directive, context);
            if (!shouldInclude) {
                return CompletableFuture.completedFuture(null);
            }
        }
    }

    // Execute normally
    // ... rest of field execution
}

private boolean evaluateDirectiveCondition(Directive directive,
                                          ExecutionContext context) {
    Argument ifArg = directive.getArgument("if");
    if (ifArg == null) return false;

    ValueNode value = ifArg.getValue();
    if (value instanceof BooleanValue) {
        return ((BooleanValue) value).getValue();
    } else if (value instanceof Variable) {
        String varName = ((Variable) value).getName();
        return (boolean) context.getVariable(varName);
    }

    return false;
}
```

### Custom Directives

Users can define custom directives:

```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION

type Query {
  adminData: [Data!]! @auth(requires: ADMIN)
}
```

**Implementation:**

```java
public interface DirectiveHandler {
    Object handle(Object resolvedValue, Directive directive, ExecutionContext context);
}

public class AuthDirectiveHandler implements DirectiveHandler {
    @Override
    public Object handle(Object resolvedValue, Directive directive,
                        ExecutionContext context) {
        Argument requiresArg = directive.getArgument("requires");
        String requiredRole = ((EnumValue) requiresArg.getValue()).getValue();

        User currentUser = context.getCurrentUser();
        if (!currentUser.hasRole(requiredRole)) {
            throw new UnauthorizedException(
                "Required role: " + requiredRole
            );
        }

        return resolvedValue;
    }
}

// Register handler
DirectiveRegistry registry = new DirectiveRegistry();
registry.register("auth", new AuthDirectiveHandler());
```

---

## Part 6: Error Handling

### Partial Success Model

GraphQL allows **partial failures**:

```graphql
{
  user {
    name      # succeeds
    email     # succeeds
    posts {   # FAILS - database error
      title
    }
  }
}
```

**Response:**

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": null
    }
  },
  "errors": [
    {
      "message": "Database connection failed",
      "path": ["user", "posts"],
      "locations": [{"line": 4, "column": 5}]
    }
  ]
}
```

### Error Collection

Collect errors during execution:

```java
public class ExecutionResult {
    private final Map<String, Object> data;
    private final List<GraphQLError> errors;

    public boolean hasErrors() {
        return !errors.isEmpty();
    }

    public Map<String, Object> toMap() {
        Map<String, Object> result = new LinkedHashMap<>();
        result.put("data", data);
        if (hasErrors()) {
            result.put("errors", errors.stream()
                .map(GraphQLError::toMap)
                .collect(Collectors.toList()));
        }
        return result;
    }
}

public class GraphQLError {
    private final String message;
    private final List<Object> path;
    private final SourceLocation location;
    private final Map<String, Object> extensions;

    public Map<String, Object> toMap() {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("message", message);
        if (path != null) map.put("path", path);
        if (location != null) {
            map.put("locations", List.of(
                Map.of("line", location.getLine(),
                       "column", location.getColumn())
            ));
        }
        if (extensions != null) map.put("extensions", extensions);
        return map;
    }
}
```

### Capturing Errors During Execution

```java
private CompletableFuture<Object> executeFieldAsync(
        Field field,
        TypeDefinition parentType,
        Object parentValue,
        ExecutionContext context) {

    return CompletableFuture.supplyAsync(() -> {
        try {
            FieldResolver resolver = getResolver(parentType, field.getName());
            Object resolved = resolver.resolve(parentValue, args, context);
            return completeValue(fieldDef.getType(), resolved, field, context);

        } catch (Exception e) {
            // Log error
            GraphQLError error = new GraphQLError(
                e.getMessage(),
                context.getPath(),
                field.getLocation(),
                Map.of("exception", e.getClass().getSimpleName())
            );
            context.addError(error);

            // Return null for failed field
            return null;
        }
    }, context.getExecutorService());
}
```

---

## Part 7: Performance Optimizations

### Field Deduplication

Avoid executing the same field multiple times:

```graphql
{
  user {
    name
    name  # duplicate!
  }
}
```

**Solution: Group fields by response key**

```java
private Map<String, List<Field>> groupFieldsByResponseKey(SelectionSet selectionSet) {
    Map<String, List<Field>> grouped = new LinkedHashMap<>();

    for (Selection selection : selectionSet.getSelections()) {
        if (selection instanceof Field) {
            Field field = (Field) selection;
            String responseKey = field.getAlias() != null
                ? field.getAlias()
                : field.getName();

            grouped.computeIfAbsent(responseKey, k -> new ArrayList<>()).add(field);
        }
    }

    return grouped;
}
```

### Resolver Result Caching

Cache resolver results per request:

```java
public class CachingResolver implements FieldResolver {
    private final FieldResolver delegate;
    private final Map<CacheKey, Object> cache = new ConcurrentHashMap<>();

    @Override
    public Object resolve(Object parent, Map<String, Object> args,
                         ExecutionContext context) {
        CacheKey key = new CacheKey(parent, args);

        return cache.computeIfAbsent(key, k ->
            delegate.resolve(parent, args, context)
        );
    }

    private static class CacheKey {
        private final Object parent;
        private final Map<String, Object> args;

        @Override
        public boolean equals(Object o) {
            if (!(o instanceof CacheKey)) return false;
            CacheKey that = (CacheKey) o;
            return Objects.equals(parent, that.parent) &&
                   Objects.equals(args, that.args);
        }

        @Override
        public int hashCode() {
            return Objects.hash(parent, args);
        }
    }
}
```

### Memoization for Expensive Computations

```java
public class MemoizedResolver implements FieldResolver {
    private final FieldResolver delegate;
    private final Cache<String, Object> cache;

    public MemoizedResolver(FieldResolver delegate) {
        this.delegate = delegate;
        this.cache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build();
    }

    @Override
    public Object resolve(Object parent, Map<String, Object> args,
                         ExecutionContext context) {
        String cacheKey = buildCacheKey(parent, args);

        try {
            return cache.get(cacheKey, () ->
                delegate.resolve(parent, args, context)
            );
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## Putting It All Together: Complete Execution Engine

```java
public class GraphQLExecutor {
    private final Schema schema;
    private final Map<TypeFieldPair, FieldResolver> resolvers;
    private final ExecutorService executorService;

    public CompletableFuture<ExecutionResult> executeAsync(
            Document document,
            Object rootValue,
            Map<String, Object> variables,
            Map<String, Object> context) {

        ExecutionContext executionContext = new ExecutionContext(
            variables,
            context,
            executorService
        );

        try {
            OperationDefinition operation = document.getOperations().get(0);
            TypeDefinition rootType = getRootType(operation.getOperationType());

            return executeSelectionSetAsync(
                operation.getSelectionSet(),
                rootType,
                rootValue,
                executionContext
            ).thenApply(data -> {
                return new ExecutionResult(data, executionContext.getErrors());
            }).exceptionally(throwable -> {
                GraphQLError error = new GraphQLError(
                    throwable.getMessage(),
                    null,
                    null,
                    null
                );
                return new ExecutionResult(null, List.of(error));
            });

        } finally {
            executionContext.close();
        }
    }

    private CompletableFuture<Map<String, Object>> executeSelectionSetAsync(
            SelectionSet selectionSet,
            TypeDefinition parentType,
            Object parentValue,
            ExecutionContext context) {

        Map<String, List<Field>> fieldsByKey = groupFieldsByResponseKey(selectionSet);

        List<CompletableFuture<FieldResult>> fieldFutures = fieldsByKey.entrySet()
            .stream()
            .map(entry -> {
                String responseKey = entry.getKey();
                Field field = entry.getValue().get(0);

                return executeFieldAsync(field, parentType, parentValue, context)
                    .thenApply(value -> new FieldResult(responseKey, value));
            })
            .collect(Collectors.toList());

        return CompletableFuture.allOf(fieldFutures.toArray(new CompletableFuture[0]))
            .thenApply(v -> {
                Map<String, Object> result = new LinkedHashMap<>();
                for (CompletableFuture<FieldResult> future : fieldFutures) {
                    FieldResult fr = future.join();
                    result.put(fr.key, fr.value);
                }
                return result;
            });
    }

    // ... other methods from earlier sections
}
```

---

## Key Takeaways

1. **Breadth-first execution** maximizes parallelism by executing all fields at the same level concurrently

2. **Field resolution** is the core operation: lookup resolver, invoke with arguments, complete value based on type

3. **CompletableFuture** enables async resolvers and concurrent execution while maintaining proper sequencing

4. **ExecutionContext** carries request-scoped data, services, and state through the execution tree

5. **Directives** like `@skip` and `@include` control field execution; custom directives extend functionality

6. **Partial failures** are supported: errors collected per-field, rest of query continues

7. **Performance optimizations**: field deduplication, resolver caching, memoization, efficient thread pools

The execution engine is where GraphQL's power comes to life—transforming declarative queries into efficient, parallelized data fetching.

---

**Next:** [Chapter 11: The N+1 Problem and DataLoader →](11-n-plus-one-dataloader.md)
