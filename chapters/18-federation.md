# Chapter 18: Federation - Distributing the Graph

> "A monolith is easy to reason about but hard to scale. Microservices are easy to scale but hard to reason about. Federation tries to give you both."

---

## The Monolith Problem (Again)

You've built a beautiful GraphQL API. One schema, one server, all your data accessible through a unified graph. Life is good.

Then your company grows:
- **10 teams** working on the same codebase
- **Merge conflicts** every day
- **Deploy coordination** nightmares
- One team's bug **breaks everyone's** features
- **Schema changes** require cross-team approval
- **Domain expertise** scattered across teams

Sound familiar? This is the same problem that drove us from monolithic applications to microservices.

### The Microservices Migration

Your organization decides to split into services:
- **User Service** - Authentication, profiles, teams
- **Post Service** - Blog posts, articles, content
- **Comment Service** - Comments, discussions, reactions
- **Analytics Service** - Metrics, reports, tracking
- **Notification Service** - Emails, push notifications, alerts

Each team owns their domain, deploys independently, scales separately.

**But now you have a new problem:** How do clients query across services?

---

## The Naive Approach: Multiple GraphQL Endpoints

**Option 1:** Each service exposes its own GraphQL endpoint.

```
GET /users-api/graphql
GET /posts-api/graphql
GET /comments-api/graphql
GET /analytics-api/graphql
```

**Client code:**

```javascript
// Fetch user
const userResponse = await fetch('/users-api/graphql', {
  method: 'POST',
  body: JSON.stringify({
    query: '{ user(id: "123") { name email } }'
  })
});

// Fetch their posts
const postsResponse = await fetch('/posts-api/graphql', {
  method: 'POST',
  body: JSON.stringify({
    query: '{ postsByAuthor(authorId: "123") { title } }'
  })
});

// Fetch comments on each post
for (let post of posts) {
  const commentsResponse = await fetch('/comments-api/graphql', {
    method: 'POST',
    body: JSON.stringify({
      query: `{ commentsByPost(postId: "${post.id}") { text } }`
    })
  });
}
```

**We're back to the original REST problem:**
- Multiple round trips
- Client-side joining
- No unified schema
- Over-fetching/under-fetching

This defeats the entire purpose of GraphQL!

---

## The Gateway Pattern

**Option 2:** Put a gateway in front that aggregates the services.

```
Client → Gateway → [User Service, Post Service, Comment Service, ...]
```

The gateway:
1. Receives client queries
2. Breaks them into sub-queries for each service
3. Calls the services in parallel
4. Stitches the results together
5. Returns a unified response

**This sounds great! But how do you implement it?**

### Problem 1: Schema Composition

Each service has its own schema:

```graphql
# User Service
type User {
  id: ID!
  name: String!
  email: String!
}

# Post Service
type Post {
  id: ID!
  title: String!
  authorId: ID!  # Just an ID, not a User object!
}
```

**Client wants:**

```graphql
query {
  post(id: "456") {
    title
    author {  # How does gateway resolve this?
      name
      email
    }
  }
}
```

The Post Service doesn't know about User objects. How does the gateway connect them?

### Problem 2: Query Planning

**Client query:**

```graphql
query {
  user(id: "123") {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

The gateway needs to:
1. Call User Service for user data
2. Call Post Service for posts (needs userId from step 1)
3. Call Comment Service for comments (needs postIds from step 2)
4. Call User Service again for comment authors (needs userIds from step 3)

**This requires intelligent query planning.**

### Problem 3: Type Ownership

Who owns the `User` type?

- User Service defines core fields (id, name, email)
- Post Service wants to add `posts` field
- Comment Service wants to add `comments` field
- Analytics Service wants to add `metrics` field

How do multiple services extend the same type?

---

## Enter Federation

**Apollo Federation** solves these problems with a few key ideas:

1. **Entities** - Types that can be extended by multiple services
2. **Keys** - How to identify an entity across services
3. **Reference Resolvers** - How services resolve entities they don't own
4. **Query Planning** - Automatic, optimized execution across services

Let's build it from first principles.

---

## Concept 1: Entities and Keys

An **entity** is a type that has a primary key and can be extended by multiple services.

### User Service Defines the Entity

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
}

type Query {
  user(id: ID!): User
}
```

The `@key(fields: "id")` directive tells the gateway:
- `User` is an entity
- It can be uniquely identified by its `id` field
- Other services can reference and extend this type

### Post Service Extends the Entity

```graphql
extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!
}

type Post @key(fields: "id") {
  id: ID!
  title: String!
  content: String!
  authorId: ID!
}

type Query {
  post(id: ID!): Post
}
```

**What's happening here:**
- `extend type User` - We're adding fields to an existing type
- `id: ID! @external` - This field is defined elsewhere (User Service)
- `posts: [Post!]!` - We're adding this field to User

**The Post Service doesn't need to know anything about `name` or `email`. It only cares about the user's `id` and their posts.**

### Comment Service Also Extends User

```graphql
extend type User @key(fields: "id") {
  id: ID! @external
  comments: [Comment!]!
}

type Comment @key(fields: "id") {
  id: ID!
  text: String!
  authorId: ID!
}
```

### Composed Schema

The gateway automatically composes these into:

```graphql
type User {
  # From User Service
  id: ID!
  name: String!
  email: String!

  # From Post Service
  posts: [Post!]!

  # From Comment Service
  comments: [Comment!]!
}
```

**Clients see a unified schema, but the data comes from three different services.**

---

## Concept 2: Reference Resolvers

When a client queries:

```graphql
query {
  user(id: "123") {
    name       # User Service
    posts {    # Post Service needs to resolve this
      title
    }
  }
}
```

**The gateway:**
1. Calls User Service: `user(id: "123")` → returns `{ id: "123", name: "Alice", email: "..." }`
2. Needs to resolve `posts` field → calls Post Service
3. **How does Post Service get the user data?**

### The __resolveReference Method

Post Service implements a special resolver:

```java
@Component
public class UserReferenceFetcher implements DataFetcher<User> {

    @Override
    public User get(DataFetchingEnvironment env) {
        Map<String, Object> reference = env.getSource();
        String userId = (String) reference.get("id");

        // We only have the user ID, nothing else
        // Return a representation that can be used to fetch posts
        return new User(userId);
    }
}
```

This is called a **reference resolver**. The gateway uses it to:
1. Take a partial User object (just the id)
2. Create a reference that the Post Service can work with
3. Resolve fields that the Post Service owns

### Implementing posts Field

```java
@Component
public class UserFieldResolver implements DataFetcher<List<Post>> {

    @Autowired
    private PostRepository postRepository;

    @Override
    public List<Post> get(DataFetchingEnvironment env) {
        User user = env.getSource();
        String userId = user.getId();

        // Fetch posts by author
        return postRepository.findByAuthorId(userId);
    }
}
```

**Each service only implements resolvers for fields it owns.**

---

## Concept 3: Query Planning

The gateway is responsible for **query planning** - figuring out which services to call and in what order.

### Example Query

```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
      comments {
        text
      }
    }
  }
}
```

### Query Plan

```
Step 1: Call User Service
  query { user(id: "123") { id name email } }
  → { id: "123", name: "Alice", email: "alice@example.com" }

Step 2: Call Post Service (parallel with User ref)
  query { _entities(representations: [{__typename: "User", id: "123"}]) {
    ... on User { posts { id title } }
  }}
  → { posts: [{id: "1", title: "Post 1"}, {id: "2", title: "Post 2"}] }

Step 3: Call Comment Service (needs post IDs from Step 2)
  query { _entities(representations: [
    {__typename: "Post", id: "1"},
    {__typename: "Post", id: "2"}
  ]) {
    ... on Post { comments { text } }
  }}
  → { posts[0].comments: [...], posts[1].comments: [...] }

Step 4: Merge results
  Return combined response to client
```

### The _entities Query

All federated services implement a special `_entities` query:

```graphql
type Query {
  _entities(representations: [_Any!]!): [_Entity]!
}
```

This allows the gateway to:
- Pass entity references to services
- Fetch fields that the service owns
- Batch multiple entity lookups

**Implementation:**

```java
@Component
public class EntityFetcher implements DataFetcher<List<Object>> {

    @Autowired
    private UserReferenceResolver userResolver;

    @Autowired
    private PostReferenceResolver postResolver;

    @Override
    public List<Object> get(DataFetchingEnvironment env) {
        List<Map<String, Object>> representations = env.getArgument("representations");
        List<Object> entities = new ArrayList<>();

        for (Map<String, Object> ref : representations) {
            String typename = (String) ref.get("__typename");

            if ("User".equals(typename)) {
                entities.add(userResolver.resolve(ref));
            } else if ("Post".equals(typename)) {
                entities.add(postResolver.resolve(ref));
            }
        }

        return entities;
    }
}
```

---

## Setting Up Federation in Java

### Step 1: Add Dependencies

```xml
<dependencies>
    <!-- Apollo Federation JVM -->
    <dependency>
        <groupId>com.apollographql.federation</groupId>
        <artifactId>federation-graphql-java-support</artifactId>
        <version>3.0.0</version>
    </dependency>

    <!-- GraphQL Java -->
    <dependency>
        <groupId>com.graphql-java</groupId>
        <artifactId>graphql-java</artifactId>
        <version>20.0</version>
    </dependency>

    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-graphql</artifactId>
    </dependency>
</dependencies>
```

### Step 2: Define Federated Schema (User Service)

**schema.graphqls:**

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  createdAt: DateTime!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}

scalar DateTime
```

### Step 3: Implement Reference Resolver

```java
@Component
public class UserReferenceResolver {

    @Autowired
    private UserRepository userRepository;

    public User resolveReference(Map<String, Object> reference) {
        String id = (String) reference.get("id");
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

### Step 4: Configure Federation

```java
@Configuration
public class FederationConfig {

    @Bean
    public GraphQL graphQL(
            GraphQLSchema schema,
            UserReferenceResolver userReferenceResolver
    ) {
        // Transform schema to support federation
        GraphQLSchema federatedSchema = Federation.transform(schema)
            .fetchEntities(env -> {
                List<Map<String, Object>> representations =
                    env.getArgument("representations");

                return representations.stream()
                    .map(ref -> {
                        String typename = (String) ref.get("__typename");
                        if ("User".equals(typename)) {
                            return userReferenceResolver.resolveReference(ref);
                        }
                        return null;
                    })
                    .collect(Collectors.toList());
            })
            .resolveEntityType(env -> {
                Object obj = env.getObject();
                if (obj instanceof User) {
                    return env.getSchema().getObjectType("User");
                }
                return null;
            })
            .build();

        return GraphQL.newGraphQL(federatedSchema).build();
    }
}
```

### Step 5: Define Extended Schema (Post Service)

**schema.graphqls:**

```graphql
extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!
}

type Post @key(fields: "id") {
  id: ID!
  title: String!
  content: String!
  authorId: ID!
  createdAt: DateTime!
}

type Query {
  post(id: ID!): Post
  posts: [Post!]!
}

scalar DateTime
```

### Step 6: Implement Extended Field Resolver

```java
@Component
public class UserFieldResolvers {

    @Autowired
    private PostRepository postRepository;

    @SchemaMapping(typeName = "User", field = "posts")
    public List<Post> posts(User user) {
        // User object only has 'id' populated (from reference)
        return postRepository.findByAuthorId(user.getId());
    }
}
```

### Step 7: Implement Post Reference Resolver

```java
@Component
public class PostReferenceResolver {

    @Autowired
    private PostRepository postRepository;

    public Post resolveReference(Map<String, Object> reference) {
        String id = (String) reference.get("id");
        return postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));
    }
}
```

---

## Setting Up the Gateway

The gateway is responsible for:
1. Loading schemas from all services
2. Composing them into a unified schema
3. Planning and executing queries
4. Merging results

### Using Apollo Router (Recommended)

Apollo Router is a high-performance gateway written in Rust.

**router.yaml:**

```yaml
supergraph:
  path: supergraph-schema.graphql

cors:
  origins:
    - "http://localhost:3000"

headers:
  all:
    request:
      - propagate:
          named: "authorization"
```

### Generating Supergraph Schema

```bash
# Install Rover CLI
npm install -g @apollo/rover

# Compose schemas from services
rover supergraph compose \
  --config supergraph-config.yaml \
  --output supergraph-schema.graphql
```

**supergraph-config.yaml:**

```yaml
federation_version: 2

subgraphs:
  users:
    routing_url: http://localhost:8081/graphql
    schema:
      file: ./services/users/schema.graphql

  posts:
    routing_url: http://localhost:8082/graphql
    schema:
      file: ./services/posts/schema.graphql

  comments:
    routing_url: http://localhost:8083/graphql
    schema:
      file: ./services/comments/schema.graphql
```

### Running the Gateway

```bash
# Start Apollo Router
./router --config router.yaml --supergraph supergraph-schema.graphql
```

---

## Advanced Federation Patterns

### Pattern 1: Compound Keys

Sometimes a single field isn't enough to uniquely identify an entity.

```graphql
type Product @key(fields: "sku storeId") {
  sku: String!
  storeId: ID!
  name: String!
  price: Float!
}
```

Reference resolver:

```java
public Product resolveReference(Map<String, Object> reference) {
    String sku = (String) reference.get("sku");
    String storeId = (String) reference.get("storeId");

    return productRepository.findBySkuAndStoreId(sku, storeId)
        .orElseThrow(() -> new ProductNotFoundException(sku, storeId));
}
```

### Pattern 2: Multiple Keys

An entity can have multiple ways to be identified.

```graphql
type User
  @key(fields: "id")
  @key(fields: "email") {
  id: ID!
  email: String!
  name: String!
}
```

Reference resolver needs to handle both:

```java
public User resolveReference(Map<String, Object> reference) {
    if (reference.containsKey("id")) {
        String id = (String) reference.get("id");
        return userRepository.findById(id).orElseThrow();
    } else if (reference.containsKey("email")) {
        String email = (String) reference.get("email");
        return userRepository.findByEmail(email).orElseThrow();
    }
    throw new IllegalArgumentException("Invalid reference");
}
```

### Pattern 3: Value Types

Not everything needs to be an entity. Some types are just data.

```graphql
# This is NOT an entity - no @key directive
type Address {
  street: String!
  city: String!
  country: String!
}

type User @key(fields: "id") {
  id: ID!
  name: String!
  address: Address  # Owned by User Service
}
```

Value types:
- No primary key
- Not extended by other services
- Fully owned by one service
- Simpler, no reference resolution needed

### Pattern 4: Shared Types

Some types are used across multiple services but not extended.

```graphql
# Defined in a shared schema
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

# Used by Order Service
type Order @key(fields: "id") {
  id: ID!
  status: OrderStatus!
}

# Used by Notification Service
extend type Order @key(fields: "id") {
  id: ID! @external
  status: OrderStatus! @external
  notifications: [Notification!]!
}
```

### Pattern 5: Computed Fields Across Services

Sometimes you need data from multiple services to compute a field.

```graphql
# Order Service
type Order @key(fields: "id") {
  id: ID!
  items: [OrderItem!]!
  subtotal: Float!
}

# Shipping Service
extend type Order @key(fields: "id") {
  id: ID! @external
  subtotal: Float! @external
  shippingCost: Float!

  # Computed from subtotal + shippingCost
  total: Float! @requires(fields: "subtotal")
}
```

Implementation:

```java
@SchemaMapping(typeName = "Order", field = "total")
public Double total(Order order) {
    // order.subtotal is populated by gateway from Order Service
    Double shippingCost = shippingService.calculateCost(order.getId());
    return order.getSubtotal() + shippingCost;
}
```

The `@requires` directive tells the gateway to fetch `subtotal` from Order Service before calling Shipping Service.

---

## Performance Optimization in Federation

### Problem: N+1 Across Services

```graphql
query {
  posts {  # Returns 100 posts
    author {  # 100 calls to User Service!
      name
    }
  }
}
```

### Solution: Entity Batching with DataLoader

The gateway automatically batches entity requests:

```
# Instead of 100 individual calls
_entities(representations: [{__typename: "User", id: "1"}])
_entities(representations: [{__typename: "User", id: "2"}])
...

# Gateway sends one batched call
_entities(representations: [
  {__typename: "User", id: "1"},
  {__typename: "User", id: "2"},
  ...
  {__typename: "User", id: "100"}
])
```

Service implementation:

```java
@Component
public class EntityFetcher {

    @Autowired
    private UserRepository userRepository;

    public List<User> fetchUserBatch(List<Map<String, Object>> references) {
        // Extract all user IDs
        List<String> ids = references.stream()
            .map(ref -> (String) ref.get("id"))
            .collect(Collectors.toList());

        // Single database query
        return userRepository.findAllById(ids);
    }
}
```

### Caching Entity Resolutions

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager("users", "posts", "comments");
    }
}

@Component
public class UserReferenceResolver {

    @Cacheable(value = "users", key = "#reference['id']")
    public User resolveReference(Map<String, Object> reference) {
        String id = (String) reference.get("id");
        return userRepository.findById(id).orElseThrow();
    }
}
```

### Query Cost Analysis

Federation amplifies query complexity. A simple client query can trigger many service calls.

```graphql
query {
  users {  # 100 users
    posts {  # 1000 posts total
      comments {  # 10000 comments total
        author {  # Back to users
          name
        }
      }
    }
  }
}
```

**Implement cost limits:**

```java
@Component
public class QueryCostInstrument implements Instrumentation {

    @Override
    public ExecutionStrategyInstrumentationContext beginExecutionStrategy(
            InstrumentationExecutionStrategyParameters params) {

        int cost = calculateQueryCost(params.getExecutionContext());

        if (cost > 10000) {
            throw new QueryTooComplexException(
                "Query cost " + cost + " exceeds limit of 10000"
            );
        }

        return super.beginExecutionStrategy(params);
    }
}
```

---

## Schema Management and Governance

### Problem: Breaking Changes

Post Service wants to rename a field:

```graphql
type Post {
  authorId: ID!  # Rename to 'userId'
}
```

But other services might reference `authorId`!

### Solution: Schema Registry

Use a schema registry to:
1. Track all schema versions
2. Detect breaking changes
3. Require approval for changes
4. Roll back if needed

**Using Apollo Studio:**

```bash
# Publish schema
rover subgraph publish my-graph@main \
  --name posts \
  --schema ./schema.graphql \
  --routing-url http://posts-service/graphql

# Check for breaking changes
rover subgraph check my-graph@main \
  --name posts \
  --schema ./schema.graphql
```

### Field Usage Tracking

The gateway can track which fields are actually used:

```yaml
# router.yaml
telemetry:
  apollo:
    field_level_instrumentation_sampler: 1.0
```

Before removing a field, check if clients are using it.

### Deprecation Workflow

```graphql
type Post @key(fields: "id") {
  id: ID!

  # Old field (deprecated)
  authorId: ID! @deprecated(reason: "Use 'author' field instead")

  # New field
  author: User!
}
```

Steps:
1. Add new field (`author`)
2. Deprecate old field (`authorId`)
3. Wait for clients to migrate
4. Monitor usage in Apollo Studio
5. Remove deprecated field when usage hits zero

---

## Testing Federated Services

### Unit Testing Individual Services

```java
@SpringBootTest
@AutoConfigureGraphQlTester
class UserServiceTest {

    @Autowired
    private GraphQlTester tester;

    @Test
    void testUserQuery() {
        tester.document("""
            query {
                user(id: "123") {
                    name
                    email
                }
            }
            """)
            .execute()
            .path("user.name").entity(String.class).isEqualTo("Alice");
    }

    @Test
    void testEntityResolver() {
        tester.document("""
            query {
                _entities(representations: [{__typename: "User", id: "123"}]) {
                    ... on User {
                        name
                    }
                }
            }
            """)
            .execute()
            .path("_entities[0].name").entity(String.class).isEqualTo("Alice");
    }
}
```

### Integration Testing the Gateway

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class GatewayIntegrationTest {

    @LocalServerPort
    private int port;

    @Test
    void testCrossServiceQuery() {
        // Mock service responses
        mockUserService();
        mockPostService();

        // Execute federated query
        GraphQlTester tester = GraphQlTester.create(
            WebTestClient.bindToServer()
                .baseUrl("http://localhost:" + port)
                .build()
        );

        tester.document("""
            query {
                user(id: "123") {
                    name
                    posts {
                        title
                    }
                }
            }
            """)
            .execute()
            .path("user.name").entity(String.class).isEqualTo("Alice")
            .path("user.posts").entityList(Post.class).hasSize(5);
    }
}
```

### Contract Testing

Ensure services honor their schema contracts:

```java
@Test
void testSchemaCompliance() {
    GraphQLSchema schema = loadSchema("schema.graphqls");
    GraphQLSchema implementedSchema = getGraphQLSchema();

    List<String> differences = SchemaComparator.compare(schema, implementedSchema);

    assertThat(differences).isEmpty();
}
```

---

## Alternatives to Federation

### Option 1: Schema Stitching

Older approach, manually stitch schemas together.

**Pros:**
- More control
- Works with non-GraphQL services

**Cons:**
- Manual configuration
- No automatic query planning
- Harder to maintain

### Option 2: GraphQL as BFF (Backend for Frontend)

Each client gets its own GraphQL layer that aggregates microservices.

```
Mobile App → Mobile BFF (GraphQL) → [Microservices]
Web App    → Web BFF (GraphQL)    → [Microservices]
```

**Pros:**
- Client-specific optimization
- Independent deployment

**Cons:**
- Duplicate schema definitions
- More services to maintain

### Option 3: Monolithic GraphQL with Modular Resolvers

Single GraphQL server, but resolvers are modular and call microservices.

```java
@Component
public class UserResolver {

    @Autowired
    private UserServiceClient userService;  // Calls user microservice

    public User user(String id) {
        return userService.getUser(id);
    }
}
```

**Pros:**
- Simple architecture
- Unified schema
- Easy to reason about

**Cons:**
- Deployment coupling
- Single point of failure
- Doesn't scale team autonomy

---

## When to Use Federation

### Use Federation When:
✅ Multiple teams own different domains
✅ Services need independent deployment
✅ Domain boundaries are clear
✅ You need a unified API for clients
✅ Teams are mature enough to handle distributed systems

### Don't Use Federation When:
❌ Small team, single domain
❌ Unclear service boundaries
❌ Frequent cross-service transactions
❌ Complex distributed query requirements
❌ Team lacks distributed systems experience

**Start with a monolith. Migrate to federation when pain points emerge.**

---

## Key Takeaways

1. **Federation distributes the graph** - Each service owns part of the schema
2. **Entities are the core concept** - Types that can be extended across services
3. **Keys enable references** - Services reference entities by primary key
4. **Query planning is automatic** - Gateway figures out service orchestration
5. **Reference resolvers connect services** - Special `_entities` query resolves references
6. **Batching prevents N+1** - Gateway batches entity resolutions
7. **Schema governance is critical** - Use registry to manage changes
8. **Test services independently** - Unit test service schemas, integration test gateway
9. **Start simple, evolve gradually** - Monolith → modular monolith → federation
10. **Trade-offs matter** - Federation adds complexity, only use when benefits outweigh costs

Federation is powerful but not a silver bullet. It trades simplicity for team autonomy and independent deployment. Choose wisely.

---

**Next:** [Chapter 19: Building a Simple GraphQL Server in Java →](19-building-server.md)
**Previous:** [← Chapter 17: Security - Query Depth, Complexity, and Cost Analysis](17-security.md)
