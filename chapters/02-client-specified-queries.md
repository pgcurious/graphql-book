# Chapter 2: What If Clients Could Specify Exactly What They Need?

**What we'll discover:** By thinking from first principles about what we actually want, we'll stumble onto ideas that feel radical but are actually inevitable.

---

## The Core Insight

In Chapter 1, we saw that REST's problems stem from a fundamental mismatch: **the server decides the shape of responses, but only the client knows what it needs.**

Let's flip this around. What if we started with a simple question:

**What if the client could ask for exactly what it wants?**

Not through URL parameters. Not through custom endpoints. But through a real, expressive way to describe data requirements.

## Thought Experiment 1: The Wish List

Imagine you're the mobile developer. You're building the post feed screen. You sit down with a piece of paper and write:

```
For each post, I need:
- title
- content
- author's name
- author's avatar URL
- number of likes
- first 3 comments, each with:
  - comment text
  - commenter's name
  - commenter's avatar
```

This is your **specification**. You know exactly what you need because you're building the UI. The question is: **how do we turn this specification into a request the server can understand?**

## Approach 1: Structured JSON Request

Let's try expressing this as JSON:

```json
{
  "resource": "post",
  "id": 123,
  "fields": [
    "title",
    "content",
    {
      "field": "author",
      "subfields": ["name", "avatarUrl"]
    },
    {
      "field": "likes",
      "subfields": ["count"]
    },
    {
      "field": "comments",
      "limit": 3,
      "subfields": [
        "text",
        {
          "field": "author",
          "subfields": ["name", "avatarUrl"]
        }
      ]
    }
  ]
}
```

This is **declarative**. We're saying what we want, not how to get it.

Let's think about what a server would need to do with this:

```java
@PostMapping("/api/query")
public ResponseEntity<Map<String, Object>> query(@RequestBody QueryRequest request) {
    // 1. Parse the request structure
    // 2. Validate that requested fields exist
    // 3. Check permissions for each field
    // 4. Build database queries
    // 5. Fetch data
    // 6. Shape response to match request structure
    // 7. Return only what was asked for

    String resource = request.getResource();
    Long id = request.getId();
    List<FieldRequest> fields = request.getFields();

    // This is going to get complex...
    Map<String, Object> response = new HashMap<>();

    for (FieldRequest field : fields) {
        if (field.isSimple()) {
            // Direct field access
            Object value = fetchField(resource, id, field.getName());
            response.put(field.getName(), value);
        } else if (field.hasSubfields()) {
            // Nested resource
            Object relatedResource = fetchRelatedResource(
                resource, id, field.getName()
            );
            Map<String, Object> nested = new HashMap<>();
            for (String subfield : field.getSubfields()) {
                nested.put(subfield, fetchField(
                    field.getName(),
                    getIdOf(relatedResource),
                    subfield
                ));
            }
            response.put(field.getName(), nested);
        }
    }

    return ResponseEntity.ok(response);
}
```

This works! We've built a single endpoint that can handle any query. But look at what we're doing: **we're building a query language**.

## Approach 2: What About SQL-Style?

Wait, why are we inventing new syntax? We already have query languages! What if we borrowed from SQL?

```sql
SELECT
  post.title,
  post.content,
  author.name AS authorName,
  author.avatarUrl AS authorAvatar,
  COUNT(likes.*) AS likeCount,
  comments[0:3].text AS commentText,
  comments[0:3].author.name AS commenterName
FROM posts
WHERE post.id = 123
```

Hmm, this feels familiar but also weird:
- SQL is designed for relational databases, not object graphs
- The nested relationships (`comments[0:3].author.name`) feel awkward
- Do we really want to expose our database schema to clients?
- What about security? Can clients SELECT from any table?

## Approach 3: Back to Hierarchical Structure

The problem with SQL is that our data isn't flatit's hierarchical. Posts contain authors, which contain fields. Posts contain comments, which contain authors.

What if we used a format that naturally represents hierarchy?

```
post(id: 123) {
  title
  content
  author {
    name
    avatarUrl
  }
  likes {
    count
  }
  comments(limit: 3) {
    text
    author {
      name
      avatarUrl
    }
  }
}
```

Now we're getting somewhere. This feels more natural. The nesting matches the data structure. The syntax is clean.

Let's think about what we've created:
- **Hierarchical**: Matches the shape of our data
- **Selective**: Only asks for what's needed
- **Declarative**: Describes what, not how
- **Arguments**: We can pass parameters like `limit: 3`

## Implementing Client-Specified Queries in Java

Let's sketch out what the server-side implementation might look like:

```java
/**
 * A query request that specifies exactly what fields to fetch
 */
public class FieldQuery {
    private String fieldName;
    private Map<String, Object> arguments;
    private List<FieldQuery> subfields;

    // Getters and constructors...
}

@RestController
@RequestMapping("/api")
public class FlexibleQueryController {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private CommentRepository commentRepository;

    /**
     * Single endpoint that handles client-specified queries
     */
    @PostMapping("/query")
    public ResponseEntity<Map<String, Object>> executeQuery(
            @RequestBody QueryRequest queryRequest) {

        String rootType = queryRequest.getType(); // "post"
        Long id = queryRequest.getId();           // 123
        List<FieldQuery> fields = queryRequest.getFields();

        Map<String, Object> result = new HashMap<>();

        // Fetch root object
        Object rootObject = fetchObject(rootType, id);

        // Resolve each requested field
        for (FieldQuery field : fields) {
            Object value = resolveField(rootObject, field);
            result.put(field.getFieldName(), value);
        }

        return ResponseEntity.ok(result);
    }

    /**
     * Core resolution logic: given an object and a field query,
     * return the value for that field
     */
    private Object resolveField(Object object, FieldQuery fieldQuery) {
        String fieldName = fieldQuery.getFieldName();

        // Get the value for this field
        Object fieldValue = getFieldValue(object, fieldName);

        // If there are subfields, we need to resolve them too
        if (fieldQuery.hasSubfields()) {
            if (fieldValue instanceof Collection) {
                // Handle lists (e.g., comments)
                return ((Collection<?>) fieldValue).stream()
                    .map(item -> resolveSubfields(item, fieldQuery.getSubfields()))
                    .collect(Collectors.toList());
            } else {
                // Handle single objects (e.g., author)
                return resolveSubfields(fieldValue, fieldQuery.getSubfields());
            }
        }

        return fieldValue;
    }

    private Map<String, Object> resolveSubfields(
            Object object,
            List<FieldQuery> subfields) {

        Map<String, Object> result = new HashMap<>();

        for (FieldQuery subfield : subfields) {
            Object value = resolveField(object, subfield);
            result.put(subfield.getFieldName(), value);
        }

        return result;
    }

    /**
     * Fetch a field value from an object using reflection or explicit getters
     */
    private Object getFieldValue(Object object, String fieldName) {
        if (object instanceof Post) {
            Post post = (Post) object;
            switch (fieldName) {
                case "title": return post.getTitle();
                case "content": return post.getContent();
                case "author":
                    return userRepository.findById(post.getAuthorId()).orElse(null);
                case "comments":
                    return commentRepository.findByPostId(post.getId());
                case "likes":
                    return likeRepository.countByPostId(post.getId());
                default:
                    throw new IllegalArgumentException("Unknown field: " + fieldName);
            }
        }
        // Similar for User, Comment, etc.
        throw new IllegalArgumentException("Unknown object type");
    }

    private Object fetchObject(String type, Long id) {
        switch (type) {
            case "post":
                return postRepository.findById(id).orElseThrow();
            case "user":
                return userRepository.findById(id).orElseThrow();
            default:
                throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}
```

## What Have We Gained?

Let's compare this to REST:

### Before (REST):
```bash
# 7 separate requests
GET /api/posts/123
GET /api/users/456
GET /api/posts/123/comments?limit=3
GET /api/users/789
GET /api/users/790
GET /api/users/791
GET /api/posts/123/likes
```

### After (Client-Specified Query):
```bash
# 1 request
POST /api/query
{
  "type": "post",
  "id": 123,
  "fields": [...]  # Our hierarchical structure
}
```

**Benefits:**
1.  **One round trip** instead of 7
2.  **No over-fetching**: Only returns what's asked for
3.  **No under-fetching**: Gets everything in one go
4.  **No endpoint proliferation**: One flexible endpoint
5.  **Client autonomy**: Frontend can change without backend changes

## But Wait... New Problems

This approach solves our original problems, but introduces new ones:

### Problem 1: Security

```json
{
  "type": "user",
  "id": 123,
  "fields": [
    "name",
    "email",         // Maybe okay
    "passwordHash",  // =® DEFINITELY NOT OKAY
    "ssn",           // =® YIKES
    "creditCard"     // =® ABORT
  ]
}
```

How do we control what fields clients can access?

### Problem 2: Relationships

```json
{
  "type": "post",
  "id": 123,
  "fields": [
    "author",  // What type is this? How do we know to fetch from users table?
    "likes"    // Is this a number or a list? What does "likes" mean?
  ]
}
```

We need to know the **types** of fields and how they relate to each other.

### Problem 3: N+1 Queries

Remember this code?

```java
case "comments":
    return commentRepository.findByPostId(post.getId());
```

And then for each comment:

```java
case "author":
    return userRepository.findById(comment.getAuthorId()).orElse(null);
```

If we have 100 comments, we just made 101 database queries:
- 1 query for comments
- 100 queries for authors (one per comment)

This is the **N+1 problem**, and it's even worse in our flexible system.

### Problem 4: Performance Unpredictability

```json
{
  "type": "post",
  "id": 123,
  "fields": [
    "comments",  // Could be 10,000 comments
    {
      "field": "author",
      "subfields": [
        "followers",  // Could be 1,000,000 followers
        {
          "field": "posts",  // Each follower's posts
          "subfields": [
            "comments"  // And all their comments
          ]
        }
      ]
    }
  ]
}
```

This single query could take down your database. We need **query complexity limits**.

### Problem 5: Discoverability

How does a client know what fields are available? What's the type of each field? What arguments can you pass?

We need some form of **schema** or **contract**.

## The Pattern Emerges

Look at what we've discovered we need:

1. **A query language**: To express hierarchical data requirements
2. **A type system**: To define what fields exist and their types
3. **A security layer**: To control field access
4. **An execution engine**: To efficiently fetch data
5. **Query optimization**: To avoid N+1 and other performance pitfalls
6. **Introspection**: So clients can discover the API's capabilities

We're not just building a flexible endpoint. We're building a complete **query runtime**.

## The Aha Moment

Here's the crucial insight: **we're not building a REST API with a twist. We're building something fundamentally different.**

We're building:
- A type system (like a programming language)
- A query language (like SQL)
- A runtime (like a database query executor)
- An API contract (like OpenAPI, but more powerful)

All of this is necessary. We can't cherry-pick which parts to implement. If we want client-specified queries that are safe, performant, and maintainable, we need all of these pieces.

## What This Means

When you see GraphQL for the first time, it might seem over-engineered:
- "Why do I need a schema definition language?"
- "Why is there a query parser?"
- "Why do I need to think about resolvers?"

The answer is: because we're not just building an API. We're building a query runtime that sits between clients and data. Each piece solves a real problem that emerges from the core idea of client-specified queries.

## Java Implementation Preview

Here's a glimpse of what a more complete implementation might look like:

```java
/**
 * The schema defines what types and fields exist
 */
public class Schema {
    private Map<String, TypeDefinition> types = new HashMap<>();

    public void defineType(String typeName, TypeDefinition definition) {
        types.put(typeName, definition);
    }

    public TypeDefinition getType(String typeName) {
        return types.get(typeName);
    }
}

/**
 * A type definition includes its fields and how to fetch them
 */
public class TypeDefinition {
    private String name;
    private Map<String, FieldDefinition> fields = new HashMap<>();

    public void addField(String fieldName, FieldDefinition definition) {
        fields.put(fieldName, definition);
    }

    public FieldDefinition getField(String fieldName) {
        return fields.get(fieldName);
    }
}

/**
 * Each field knows its type and how to resolve its value
 */
public class FieldDefinition {
    private String name;
    private String type;  // "String", "Int", "User", "List<Comment>", etc.
    private FieldResolver resolver;
    private boolean isNullable;
    private List<ArgumentDefinition> arguments;

    // The resolver is a function that fetches this field's value
    public Object resolve(Object parent, Map<String, Object> args) {
        return resolver.resolve(parent, args);
    }
}

/**
 * A resolver is a function: (parent object, arguments) -> field value
 */
@FunctionalInterface
public interface FieldResolver {
    Object resolve(Object parent, Map<String, Object> arguments);
}

/**
 * Example schema definition
 */
public class SocialNetworkSchema {

    public static Schema build(
            UserRepository userRepo,
            PostRepository postRepo,
            CommentRepository commentRepo) {

        Schema schema = new Schema();

        // Define Post type
        TypeDefinition postType = new TypeDefinition("Post");
        postType.addField("id", new FieldDefinition("id", "ID",
            (parent, args) -> ((Post) parent).getId()));
        postType.addField("title", new FieldDefinition("title", "String",
            (parent, args) -> ((Post) parent).getTitle()));
        postType.addField("author", new FieldDefinition("author", "User",
            (parent, args) -> {
                Post post = (Post) parent;
                return userRepo.findById(post.getAuthorId()).orElse(null);
            }));
        postType.addField("comments", new FieldDefinition("comments", "List<Comment>",
            (parent, args) -> {
                Post post = (Post) parent;
                int limit = (int) args.getOrDefault("limit", 10);
                return commentRepo.findByPostIdLimit(post.getId(), limit);
            }));

        schema.defineType("Post", postType);

        // Define User type
        TypeDefinition userType = new TypeDefinition("User");
        userType.addField("id", new FieldDefinition("id", "ID",
            (parent, args) -> ((User) parent).getId()));
        userType.addField("name", new FieldDefinition("name", "String",
            (parent, args) -> ((User) parent).getName()));
        userType.addField("avatarUrl", new FieldDefinition("avatarUrl", "String",
            (parent, args) -> ((User) parent).getAvatarUrl()));

        schema.defineType("User", userType);

        return schema;
    }
}
```

This is starting to look like a real system. We have:
- Types defined in a schema
- Resolvers that know how to fetch data
- Type safety (we know "author" returns a "User")
- Extensibility (easy to add new fields)

## The Road Ahead

We've discovered that client-specified queries are possible, but they require:

1. **A formal query syntax** - How do clients express what they want?
2. **A type system** - What's the contract between client and server?
3. **A resolver system** - How does the server fetch data?
4. **An execution engine** - How do we run queries efficiently?
5. **Performance optimizations** - How do we avoid N+1 and other pitfalls?

In the next chapter, we'll design these pieces from scratch. We'll think through what an ideal query language would look like, building it piece by piece from first principles.

## Key Insights

1. **Client-specified queries are desirable**: They solve REST's fundamental problems
2. **They require a complete system**: Query language + type system + runtime
3. **Each piece is necessary**: We can't skip any part
4. **It's like building a database query engine**: For APIs instead of databases
5. **The complexity is inherent**: It comes from the problem, not the solution

## Exercises

1. **Design your own syntax**: How would you express this query in a way that feels natural?
   - "Get user 123's last 5 posts, each with its title, like count, and top 2 comments"

2. **Security challenge**: How would you prevent clients from querying sensitive fields like `passwordHash`? Design a permission system.

3. **Performance analysis**: Given the Java code above, trace through exactly what database queries would execute for:
   ```
   post(id: 123) {
     title
     comments(limit: 3) {
       author { name }
     }
   }
   ```
   How many database hits? Where's the N+1 problem?

4. **Type system design**: If you were designing the type system, what primitive types would you need? (String, Int, Boolean... what else?) How would you represent lists? Nullable fields?

5. **Implementation challenge**: Take the Java code snippets above and try to implement a working prototype. What additional pieces do you need?

---

**Next:** [Chapter 3: Thought Experiment - Designing the Perfect Query Language í](03-designing-query-language.md)

**Previous:** [ê Chapter 1: The REST API Problem](01-rest-api-problem.md)
