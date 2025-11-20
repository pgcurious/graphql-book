# Chapter 3: Thought Experiment - Designing the Perfect Query Language

**What we'll discover:** By designing our ideal query language from scratch, we'll rediscover the core principles that make GraphQL work.

---

## The Design Challenge

Let's start with a clean slate. Forget everything you know about GraphQL. We're going to design a query language from first principles, making decisions based purely on what would make the best system.

Our requirements, discovered in previous chapters:

1. **Hierarchical**: Must express nested data naturally
2. **Selective**: Only request the fields you need
3. **Typed**: Strong types for safety and tooling
4. **Efficient**: One request, minimal data transfer
5. **Discoverable**: Clients can learn what's available
6. **Safe**: Can't query unauthorized data
7. **Performant**: Server can optimize execution

Let's design each piece.

## Part 1: Syntax Design

### Question 1: What Should a Query Look Like?

Let's start with our social network example. We want a post with some fields. What feels most natural?

**Option A: JSON-style**
```json
{
  "query": {
    "post": {
      "id": 123,
      "fields": ["title", "content"]
    }
  }
}
```

**Option B: SQL-style**
```sql
SELECT title, content FROM post WHERE id = 123
```

**Option C: Function call style**
```javascript
post(id: 123).fields(title, content)
```

**Option D: Hierarchical block style**
```
post(id: 123) {
  title
  content
}
```

Let's analyze each:

#### JSON-style
 Familiar to web developers
L Verbose, lots of brackets and quotes
L Nested queries get unwieldy

#### SQL-style
 Familiar to database developers
L Gets awkward with deep nesting
L `SELECT comments.author.name FROM post WHERE...` - weird

#### Function call style
 Concise
L Ambiguous chaining behavior
L How do you represent nested selections?

#### Hierarchical block style
 Clean and readable
 Nesting is natural
 Looks like the data structure you'll get back
 Minimal punctuation

**Winner: Hierarchical block style**

The shape of the query mirrors the shape of the response. This is powerful for developer ergonomics.

### Question 2: How Do We Handle Arguments?

We need to pass parameters like `id: 123` or `limit: 10`. Where do they go?

**Option A: Separate parameters object**
```
query {
  parameters: { id: 123, limit: 10 }
  post {
    title
    comments { text }
  }
}
```

**Option B: Inline with field**
```
post(id: 123) {
  title
  comments(limit: 10) {
    text
  }
}
```

**Option B is clearly better**: arguments are attached to the field they affect. If `comments` needs a limit, put it right there.

### Question 3: What About Multiple Root Queries?

What if we want to fetch two different posts?

```
post(id: 123) {
  title
}
post(id: 456) {
  title
}
```

This is ambiguous - two fields with the same name. We need a way to distinguish them.

**Aliases to the rescue:**
```
firstPost: post(id: 123) {
  title
}
secondPost: post(id: 456) {
  title
}
```

Now the response can be:
```json
{
  "firstPost": { "title": "..." },
  "secondPost": { "title": "..." }
}
```

### Question 4: How Do We Avoid Repetition?

Imagine we're fetching multiple users and want the same fields for each:

```
user(id: 1) {
  name
  email
  avatarUrl
  bio
}
user(id: 2) {
  name
  email
  avatarUrl
  bio
}
user(id: 3) {
  name
  email
  avatarUrl
  bio
}
```

This violates DRY (Don't Repeat Yourself). We need fragments:

```
fragment UserInfo on User {
  name
  email
  avatarUrl
  bio
}

user1: user(id: 1) {
  ...UserInfo
}
user2: user(id: 2) {
  ...UserInfo
}
user3: user(id: 3) {
  ...UserInfo
}
```

Much better! And look what we've invented: **reusable query fragments**.

## Part 2: Type System Design

### Question 5: What Types Do We Need?

A query language without types is like programming without types - you'll make mistakes. What types should we support?

**Scalar types** (atomic values):
```
Int       # 42
Float     # 3.14
String    # "hello"
Boolean   # true
ID        # "unique-identifier" (like String but semantic meaning)
```

**Object types** (composite structures):
```
type User {
  id: ID
  name: String
  email: String
}
```

**List types** (collections):
```
type Post {
  comments: [Comment]  # List of comments
}
```

**Non-null types** (required fields):
```
type User {
  id: ID!         # Required, cannot be null
  name: String!   # Required
  nickname: String  # Optional, can be null
}
```

**Enum types** (fixed set of values):
```
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

### Question 6: How Do We Define Relationships?

In our social network, a Post has an Author (User), and Comments, each with an Author. How do we express this?

```
type Post {
  id: ID!
  title: String!
  content: String!
  author: User!           # Relationship to User
  comments: [Comment!]!   # List of Comments
  likeCount: Int!
}

type User {
  id: ID!
  name: String!
  avatarUrl: String
  posts: [Post!]!         # User's posts
}

type Comment {
  id: ID!
  text: String!
  author: User!           # Who wrote this comment
  post: Post!             # Which post it's on
}
```

Notice what we've created: **a graph of types**. Posts point to Users, Users point to Posts, Comments point to both. This is a graph of relationships.

### Question 7: How Do We Handle Arguments in the Schema?

Some fields need arguments. Like `comments(limit: 10)`. How do we define that in the type?

```
type Post {
  # Simple field, no arguments
  title: String!

  # Field with arguments
  comments(
    limit: Int = 10,        # Default value
    offset: Int = 0,
    orderBy: CommentOrder
  ): [Comment!]!
}

enum CommentOrder {
  NEWEST_FIRST
  OLDEST_FIRST
  MOST_LIKED
}
```

This is declarative - the schema tells you exactly what arguments are available and their types.

## Part 3: The Query Execution Model

Now we need to think about **how** queries are executed.

### Question 8: How Does the Server Process a Query?

Given this query:
```
post(id: 123) {
  title
  author {
    name
  }
  comments(limit: 3) {
    text
  }
}
```

What happens on the server?

**Step 1: Parse the query**
Convert text into a data structure (AST - Abstract Syntax Tree):
```
QueryNode {
  field: "post",
  arguments: { id: 123 },
  selections: [
    FieldNode { field: "title" },
    FieldNode {
      field: "author",
      selections: [
        FieldNode { field: "name" }
      ]
    },
    FieldNode {
      field: "comments",
      arguments: { limit: 3 },
      selections: [
        FieldNode { field: "text" }
      ]
    }
  ]
}
```

**Step 2: Validate against schema**
- Does the `Post` type have a `title` field? 
- Does `Post.author` return a `User`? 
- Does `User` have a `name` field? 
- Is `comments` a valid field with a `limit` argument? 

**Step 3: Execute**
This is where it gets interesting. How do we fetch the data?

### Question 9: What's a Resolver?

A **resolver** is a function that knows how to fetch a field's value. Every field has a resolver.

```java
public class PostResolver {

    @Autowired
    private PostRepository postRepository;

    /**
     * Resolver for the root "post" field
     */
    public Post getPost(Long id) {
        return postRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));
    }

    /**
     * Resolver for Post.title
     * Input: a Post object
     * Output: the title string
     */
    public String getTitle(Post post) {
        return post.getTitle();
    }

    /**
     * Resolver for Post.author
     * Input: a Post object
     * Output: a User object
     */
    public User getAuthor(Post post) {
        return userRepository.findById(post.getAuthorId())
            .orElse(null);
    }

    /**
     * Resolver for Post.comments
     * Input: a Post object, arguments { limit }
     * Output: List of Comments
     */
    public List<Comment> getComments(Post post, int limit) {
        return commentRepository
            .findByPostId(post.getId(), PageRequest.of(0, limit))
            .getContent();
    }
}
```

The execution engine calls resolvers in the right order:

```
1. Resolve post(id: 123)           í Post object
2. Resolve post.title              í "My Post Title"
3. Resolve post.author             í User object
4. Resolve post.author.name        í "Alice"
5. Resolve post.comments(limit: 3) í [Comment, Comment, Comment]
6. For each comment:
   - Resolve comment.text          í "Great post!", "Thanks", "Amazing"
```

### Question 10: What Order Should We Execute Resolvers?

Look at this query:
```
post(id: 123) {
  title
  author { name }
}
```

We have two execution strategies:

**Strategy A: Depth-first**
```
1. post(id: 123)      í Post object
2. post.title         í "Hello"
3. post.author        í User object
4. post.author.name   í "Alice"
```

**Strategy B: Breadth-first**
```
1. post(id: 123)      í Post object
2. post.title         í "Hello"
3. post.author        í User object
4. post.author.name   í "Alice"
```

Wait, they're the same in this case. Let's try a different example:

```
post(id: 123) {
  comments {
    author { name }
  }
}
```

**Depth-first:**
```
1. post(id: 123)           í Post
2. post.comments           í [Comment1, Comment2, Comment3]
3. comment1.author         í User1
4. comment1.author.name    í "Alice"
5. comment2.author         í User2
6. comment2.author.name    í "Bob"
7. comment3.author         í User3
8. comment3.author.name    í "Charlie"
```

**Breadth-first:**
```
1. post(id: 123)           í Post
2. post.comments           í [Comment1, Comment2, Comment3]
3. comment1.author         í User1
4. comment2.author         í User2
5. comment3.author         í User3
6. comment1.author.name    í "Alice"
7. comment2.author.name    í "Bob"
8. comment3.author.name    í "Charlie"
```

**Breadth-first is better** because we can batch operations at each level. Instead of fetching users one at a time, we can fetch all three users in one database query:

```sql
SELECT * FROM users WHERE id IN (user1_id, user2_id, user3_id)
```

This prevents N+1 queries!

## Part 4: Practical Implementation Patterns

### The Resolver Pattern in Java

Let's implement a complete resolver system:

```java
/**
 * Base interface for all resolvers
 */
@FunctionalInterface
public interface FieldResolver<T, R> {
    R resolve(T source, Map<String, Object> arguments, Context context);
}

/**
 * Context passed to every resolver
 */
public class ResolverContext {
    private final DataLoader dataLoader;
    private final Authentication auth;
    private final Map<String, Object> variables;

    // Getters...
}

/**
 * Registry of all resolvers in the system
 */
public class ResolverRegistry {
    private Map<String, Map<String, FieldResolver<?, ?>>> resolvers = new HashMap<>();

    public <T, R> void registerResolver(
            String typeName,
            String fieldName,
            FieldResolver<T, R> resolver) {

        resolvers
            .computeIfAbsent(typeName, k -> new HashMap<>())
            .put(fieldName, resolver);
    }

    public FieldResolver<?, ?> getResolver(String typeName, String fieldName) {
        return resolvers
            .getOrDefault(typeName, Collections.emptyMap())
            .get(fieldName);
    }
}

/**
 * Post type resolvers
 */
@Component
public class PostResolvers {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private CommentRepository commentRepository;

    @PostConstruct
    public void registerResolvers(ResolverRegistry registry) {

        // Root resolver for "post" query
        registry.registerResolver("Query", "post",
            (source, args, ctx) -> {
                Long id = (Long) args.get("id");
                return postRepository.findById(id).orElse(null);
            });

        // Post.title resolver (trivial)
        registry.registerResolver("Post", "title",
            (Post post, args, ctx) -> post.getTitle());

        // Post.author resolver (fetches related User)
        registry.registerResolver("Post", "author",
            (Post post, args, ctx) -> {
                // Use DataLoader to batch user fetches
                return ctx.getDataLoader()
                    .load(User.class, post.getAuthorId());
            });

        // Post.comments resolver with arguments
        registry.registerResolver("Post", "comments",
            (Post post, args, ctx) -> {
                int limit = (int) args.getOrDefault("limit", 10);
                int offset = (int) args.getOrDefault("offset", 0);

                return commentRepository.findByPostId(
                    post.getId(),
                    PageRequest.of(offset / limit, limit)
                ).getContent();
            });
    }
}

/**
 * User type resolvers
 */
@Component
public class UserResolvers {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PostRepository postRepository;

    @PostConstruct
    public void registerResolvers(ResolverRegistry registry) {

        registry.registerResolver("User", "name",
            (User user, args, ctx) -> user.getName());

        registry.registerResolver("User", "avatarUrl",
            (User user, args, ctx) -> user.getAvatarUrl());

        registry.registerResolver("User", "posts",
            (User user, args, ctx) -> {
                return postRepository.findByAuthorId(user.getId());
            });
    }
}
```

### The Execution Engine

Now we need an engine that walks the query and calls resolvers:

```java
/**
 * Executes queries by walking the AST and calling resolvers
 */
public class ExecutionEngine {

    private final ResolverRegistry resolverRegistry;
    private final Schema schema;

    public CompletableFuture<Map<String, Object>> execute(
            Query query,
            ResolverContext context) {

        Map<String, Object> result = new HashMap<>();

        // Execute each root field in the query
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (FieldNode field : query.getFields()) {
            CompletableFuture<Void> future = executeField(
                "Query",
                null,  // No parent for root fields
                field,
                context
            ).thenAccept(value -> {
                String alias = field.getAlias() != null
                    ? field.getAlias()
                    : field.getName();
                result.put(alias, value);
            });

            futures.add(future);
        }

        // Wait for all root fields to complete
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> result);
    }

    private CompletableFuture<Object> executeField(
            String parentType,
            Object parent,
            FieldNode field,
            ResolverContext context) {

        // Get the resolver for this field
        FieldResolver resolver = resolverRegistry.getResolver(
            parentType,
            field.getName()
        );

        if (resolver == null) {
            throw new RuntimeException(
                "No resolver found for " + parentType + "." + field.getName()
            );
        }

        // Call the resolver
        return CompletableFuture.supplyAsync(() -> {
            try {
                Object value = resolver.resolve(
                    parent,
                    field.getArguments(),
                    context
                );

                // If there are sub-selections, resolve them too
                if (field.hasSelections() && value != null) {
                    return resolveSelections(value, field, context).join();
                }

                return value;
            } catch (Exception e) {
                // Error handling (we'll explore this more later)
                return null;
            }
        });
    }

    private CompletableFuture<Object> resolveSelections(
            Object parent,
            FieldNode field,
            ResolverContext context) {

        String parentType = getTypeOf(parent);

        if (parent instanceof List) {
            // Resolve selections for each item in the list
            List<?> list = (List<?>) parent;
            List<CompletableFuture<Object>> futures = list.stream()
                .map(item -> resolveSelections(item, field, context))
                .collect(Collectors.toList());

            return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .thenApply(v -> futures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList()));
        } else {
            // Resolve selections for a single object
            Map<String, Object> result = new HashMap<>();
            List<CompletableFuture<Void>> futures = new ArrayList<>();

            for (FieldNode subField : field.getSelections()) {
                CompletableFuture<Void> future = executeField(
                    parentType,
                    parent,
                    subField,
                    context
                ).thenAccept(value -> {
                    String alias = subField.getAlias() != null
                        ? subField.getAlias()
                        : subField.getName();
                    result.put(alias, value);
                });

                futures.add(future);
            }

            return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .thenApply(v -> result);
        }
    }

    private String getTypeOf(Object object) {
        // Map Java objects to GraphQL types
        if (object instanceof Post) return "Post";
        if (object instanceof User) return "User";
        if (object instanceof Comment) return "Comment";
        throw new RuntimeException("Unknown type: " + object.getClass());
    }
}
```

## Part 5: Advanced Features We Need

### Introspection: Querying the Schema

Clients need to discover what's available. Let's make the schema itself queryable:

```
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

This is **introspection** - querying the API structure itself. This enables:
- Auto-generated documentation
- IDE autocomplete
- GraphQL explorers (like GraphiQL)
- Type-safe client code generation

### Variables: Reusable Queries

Hard-coding values in queries is bad:

```
post(id: 123) {
  title
}
```

What if we want to reuse this query with different IDs? **Variables**:

```
query GetPost($postId: ID!) {
  post(id: $postId) {
    title
  }
}

# Variables (separate JSON):
{
  "postId": 123
}
```

Now clients can prepare queries once and execute them many times with different variables.

### Directives: Conditional Fields

Sometimes you want fields only conditionally:

```
query GetPost($includeComments: Boolean!) {
  post(id: 123) {
    title
    comments @include(if: $includeComments) {
      text
    }
  }
}
```

The `@include` directive controls whether a field is fetched.

## Part 6: What We've Invented

Let's step back and look at what we've designed:

1. **Query Language**
   - Hierarchical syntax that mirrors data structure
   - Arguments on fields
   - Aliases for multiple fetches
   - Fragments for reuse
   - Variables for parameterization

2. **Type System**
   - Scalar types (Int, String, Boolean, ID, Float)
   - Object types with fields
   - Lists and non-null modifiers
   - Enums
   - A schema that defines everything

3. **Execution Model**
   - Resolvers for each field
   - Breadth-first execution for batching
   - Async/parallel execution with CompletableFuture
   - Context passed to every resolver

4. **Developer Experience**
   - Introspection for tooling
   - Strong types for safety
   - Single endpoint
   - Declarative queries

## The Big Reveal

What we've designed, piece by piece, from first principles, is essentially **GraphQL**.

We didn't start by learning GraphQL's syntax and trying to understand it. We started with problems and built solutions. And we arrived at the same place GraphQL did, because these solutions are the natural consequences of the requirements.

**GraphQL isn't arbitrary.** Every feature exists for a reason:
- The hierarchical syntax? Because data is hierarchical
- The type system? For safety and tooling
- Resolvers? To separate fetching logic from schema definition
- Introspection? So clients can be smart
- Variables? For query reuse
- Fragments? To avoid repetition

## Comparing Our Design to Real GraphQL

Let's compare what we designed to actual GraphQL:

**Our query syntax:**
```
post(id: 123) {
  title
  author {
    name
  }
}
```

**Real GraphQL:**
```graphql
query {
  post(id: 123) {
    title
    author {
      name
    }
  }
}
```

Basically identical! We just need to wrap it in `query { }`.

**Our schema:**
```
type Post {
  id: ID!
  title: String!
  author: User!
}
```

**Real GraphQL SDL:**
```graphql
type Post {
  id: ID!
  title: String!
  author: User!
}
```

Exactly the same!

**Our resolver:**
```java
registry.registerResolver("Post", "author",
  (Post post, args, ctx) -> {
    return userRepository.findById(post.getAuthorId());
  });
```

**Real GraphQL Java (graphql-java library):**
```java
.dataFetcher("author", env -> {
  Post post = env.getSource();
  return userRepository.findById(post.getAuthorId());
})
```

Same concept, slightly different API!

## Key Insights

1. **GraphQL is the natural solution** to client-specified queries
2. **Every feature has a purpose** - nothing is arbitrary
3. **The syntax reflects the data structure** - hierarchical queries for hierarchical data
4. **Strong typing is essential** - for safety, tooling, and performance
5. **Resolvers decouple schema from implementation** - the schema is your API contract
6. **It's a runtime, not just an API format** - parser, validator, executor

## What's Next

We've designed the core concepts. But we haven't addressed:

- How do we handle mutations (writes)?
- How do we prevent N+1 queries in practice?
- How do we handle errors gracefully?
- How do we secure field access?
- How do we handle real-time updates?
- How do we scale across multiple services?

In the next chapters, we'll dive deep into each of these, always from first principles. We'll implement real solutions in Java and understand the engineering trade-offs.

## Exercises

1. **Extend the schema**: Add a `Like` type that tracks who liked what. What relationships does it need?

2. **Query complexity**: Design a system to calculate query "cost" to prevent expensive queries. What factors should contribute to cost?

3. **Implement a mini executor**: Write a Java program that takes a simple query like:
   ```
   user(id: 1) {
     name
     email
   }
   ```
   Parse it (you can use simple string parsing), and execute it against a hardcoded User object.

4. **Design mutations**: How would you express "create a new post" in our query language? What would the syntax look like?

5. **Authorization**: How would you prevent clients from querying `User.ssn` or other sensitive fields? Where does the auth check happen - in the resolver, the executor, or somewhere else?

---

**Next:** [Chapter 4: Type Systems - Why We Need Strongly Typed Schemas í](04-type-systems.md)

**Previous:** [ê Chapter 2: What If Clients Could Specify Exactly What They Need?](02-client-specified-queries.md)
