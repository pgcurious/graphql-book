# Chapter 4: Type Systems - Why We Need Strongly Typed Schemas

**Status:** Detailed Outline (To Be Expanded)

**What we'll discover:** Why a strong type system isn't just nice-to-haveit's essential for making client-specified queries safe, fast, and maintainable.

---

## Outline

### Introduction: The Problem with Untyped APIs
- Quick recap: we've designed a query language, but there's no contract
- What happens when a client asks for a field that doesn't exist?
- What if they pass the wrong type of argument?
- Without types, we find errors at runtime, not development time

### Part 1: Why Types Matter

#### The Runtime vs Compile-Time Tradeoff
- Dynamic systems (no types): flexible but error-prone
- Static systems (strong types): rigid but safe
- Our query system needs both: flexible queries, guaranteed correctness

#### Real-World Example: The Cascading Failure
**Java example:**
```java
// Without type system
Query: post(id: "abc") { title }  // "abc" is a string, should be number
// Server crashes at runtime

// With type system
Query: post(id: "abc") { title }  // Validation error before execution
Error: Expected type 'Int', got 'String'
```

#### What Types Give Us
1. **Validation before execution** - catch errors early
2. **Self-documentation** - schema is the API docs
3. **Tooling** - IDEs can autocomplete, type-check queries
4. **Performance** - know types í optimize execution
5. **Versioning** - schema changes are explicit and trackable

### Part 2: Designing the Type System

#### Scalar Types (Primitives)
```graphql
scalar Int      # 32-bit signed integer
scalar Float    # Double-precision floating point
scalar String   # UTF-8 character sequence
scalar Boolean  # true or false
scalar ID       # Unique identifier (serialized as String)
```

**Discussion:** Why ID vs String? Semantic meaning matters.

**Java mapping:**
```java
public class ScalarTypes {
    Int     í java.lang.Integer
    Float   í java.lang.Double
    String  í java.lang.String
    Boolean í java.lang.Boolean
    ID      í java.lang.String (but with special semantics)
}
```

#### Custom Scalars
What if we need Date, DateTime, URL, Email, etc?

```graphql
scalar DateTime  # ISO 8601 formatted date-time
scalar URL       # Valid URL string
scalar Email     # Valid email address
```

**Java implementation:**
```java
public class DateTimeScalar implements Coercing<DateTime, String> {
    @Override
    public String serialize(DateTime value) {
        return value.toString();
    }

    @Override
    public DateTime parseValue(String value) {
        return DateTime.parse(value);
    }

    @Override
    public DateTime parseLiteral(Object value) {
        if (value instanceof StringValue) {
            return DateTime.parse(((StringValue) value).getValue());
        }
        throw new CoercingParseLiteralException("Expected string");
    }
}
```

#### Object Types
```graphql
type Post {
    id: ID!
    title: String!
    content: String
    published: Boolean!
}
```

**Key concepts:**
- Fields have types
- `!` means non-nullable (required)
- Without `!`, field can be null

**Java mapping:**
```java
@GraphQLType
public class Post {
    @GraphQLField(type = "ID!")
    private String id;

    @GraphQLField(type = "String!")
    private String title;

    @GraphQLField(type = "String")  // Nullable
    private String content;

    @GraphQLField(type = "Boolean!")
    private Boolean published;
}
```

#### Enum Types
```graphql
enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
    DELETED
}

type Post {
    status: PostStatus!
}
```

**Java implementation:**
```java
public enum PostStatus {
    DRAFT,
    PUBLISHED,
    ARCHIVED,
    DELETED
}
```

#### List Types
```graphql
type User {
    friends: [User!]!     # Non-null list of non-null users
    posts: [Post!]        # Nullable list of non-null posts
    tags: [String]        # Nullable list of nullable strings
}
```

**The four combinations:**
1. `[User]` - nullable list, nullable items
2. `[User!]` - nullable list, non-null items
3. `[User]!` - non-null list, nullable items
4. `[User!]!` - non-null list, non-null items

**Java representation:**
```java
public class User {
    private List<User> friends;       // [User]
    private List<User> posts;         // [User!]
    private List<String> tags;        // [String]
}
```

#### Interface Types
```graphql
interface Node {
    id: ID!
}

type Post implements Node {
    id: ID!
    title: String!
}

type User implements Node {
    id: ID!
    name: String!
}
```

**Use case:** Querying heterogeneous collections

**Java with fragments:**
```graphql
query {
    search(query: "hello") {  # Returns [Node]
        ... on Post {
            title
        }
        ... on User {
            name
        }
    }
}
```

#### Union Types
```graphql
union SearchResult = Post | User | Comment

type Query {
    search(query: String!): [SearchResult]
}
```

**When to use:** When a field can return one of several types

**Java implementation:**
```java
public class SearchResult {
    private Object value;  // Post, User, or Comment

    public boolean isPost() { return value instanceof Post; }
    public boolean isUser() { return value instanceof User; }
    public boolean isComment() { return value instanceof Comment; }
}
```

### Part 3: Input Types

#### Why Separate Input Types?
- Output types (queries) have different requirements than input types (mutations)
- Input types can't have circular references
- Input types don't have resolvers

```graphql
type User {              # Output type
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
}

input CreateUserInput {  # Input type
    name: String!
    email: String!
    # No id (generated by server)
    # No createdAt (set by server)
}
```

**Java example:**
```java
// Output type
public class User {
    private String id;
    private String name;
    private String email;
    private DateTime createdAt;
}

// Input type
public class CreateUserInput {
    private String name;
    private String email;
}
```

### Part 4: Schema Definition Language (SDL)

#### The Complete Schema
```graphql
schema {
    query: Query
    mutation: Mutation
    subscription: Subscription
}

type Query {
    post(id: ID!): Post
    posts(limit: Int = 10, offset: Int = 0): [Post!]!
    user(id: ID!): User
}

type Mutation {
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post
    deletePost(id: ID!): Boolean!
}

type Post {
    id: ID!
    title: String!
    content: String
    author: User!
    comments(limit: Int = 5): [Comment!]!
}

# ... other types
```

#### Schema in Java (Programmatic API)
```java
public class SchemaBuilder {

    public GraphQLSchema buildSchema() {
        GraphQLObjectType queryType = GraphQLObjectType.newObject()
            .name("Query")
            .field(GraphQLFieldDefinition.newFieldDefinition()
                .name("post")
                .type(postType)
                .argument(GraphQLArgument.newArgument()
                    .name("id")
                    .type(GraphQLNonNull.nonNull(Scalars.GraphQLID))
                    .build())
                .dataFetcher(postDataFetcher)
                .build())
            .build();

        return GraphQLSchema.newSchema()
            .query(queryType)
            .build();
    }
}
```

### Part 5: Type Validation

#### Query Validation Rules
1. **Field selection**: Only select fields that exist on the type
2. **Type matching**: Arguments must match declared types
3. **Required arguments**: Non-null arguments must be provided
4. **Fragment compatibility**: Fragments must match the type
5. **Circular references**: Detect infinite query structures

**Example validation errors:**

```graphql
# Error: Field doesn't exist
post(id: 123) {
    title
    invalidField  # L Post type has no 'invalidField'
}

# Error: Wrong argument type
post(id: "not-a-number") {  # L Expected Int, got String
    title
}

# Error: Missing required argument
post {  # L Required argument 'id' not provided
    title
}
```

#### Java Validation Implementation
```java
public class QueryValidator {

    private Schema schema;

    public List<ValidationError> validate(Query query) {
        List<ValidationError> errors = new ArrayList<>();

        for (FieldNode field : query.getFields()) {
            errors.addAll(validateField(field, schema.getQueryType()));
        }

        return errors;
    }

    private List<ValidationError> validateField(
            FieldNode field,
            TypeDefinition parentType) {

        List<ValidationError> errors = new ArrayList<>();

        // Check if field exists on type
        FieldDefinition fieldDef = parentType.getField(field.getName());
        if (fieldDef == null) {
            errors.add(new ValidationError(
                "Field '" + field.getName() + "' doesn't exist on type '"
                + parentType.getName() + "'"
            ));
            return errors;
        }

        // Validate arguments
        errors.addAll(validateArguments(field, fieldDef));

        // Validate sub-selections
        if (field.hasSelections()) {
            TypeDefinition fieldType = schema.getType(fieldDef.getType());
            for (FieldNode subField : field.getSelections()) {
                errors.addAll(validateField(subField, fieldType));
            }
        }

        return errors;
    }

    private List<ValidationError> validateArguments(
            FieldNode field,
            FieldDefinition fieldDef) {

        // Check all required arguments are provided
        // Check argument types match
        // Check no unknown arguments
        // ...
    }
}
```

### Part 6: Introspection - The Schema Queries Itself

#### The Meta Schema
```graphql
query IntrospectSchema {
    __schema {
        types {
            name
            kind
            fields {
                name
                type {
                    name
                    kind
                }
            }
        }
    }
}
```

**Result:**
```json
{
    "data": {
        "__schema": {
            "types": [
                {
                    "name": "Post",
                    "kind": "OBJECT",
                    "fields": [
                        {"name": "id", "type": {"name": "ID", "kind": "SCALAR"}},
                        {"name": "title", "type": {"name": "String", "kind": "SCALAR"}}
                    ]
                }
            ]
        }
    }
}
```

#### Why Introspection Matters
1. **Auto-generated documentation** - GraphiQL, GraphQL Playground
2. **IDE tooling** - Autocomplete, type checking
3. **Code generation** - Generate client types from schema
4. **Schema diffing** - Detect breaking changes

#### Java Implementation
```java
public class IntrospectionResolver {

    @Resolver("__schema")
    public SchemaIntrospection getSchema() {
        return new SchemaIntrospection(schema);
    }

    @Resolver("__type")
    public TypeIntrospection getType(String name) {
        TypeDefinition type = schema.getType(name);
        return new TypeIntrospection(type);
    }
}
```

### Part 7: Type System Benefits in Practice

#### Type-Safe Client Code
Using the schema, generate type-safe client code:

```java
// Generated from schema
public class PostQuery {
    private String id;
    private String title;
    private UserQuery author;  // Nested type

    public static class UserQuery {
        private String name;
        private String avatarUrl;
    }
}

// Usage
PostQuery result = graphql.query(
    "query { post(id: 123) { id, title, author { name } } }",
    PostQuery.class
);

String title = result.getTitle();  // Type-safe!
```

#### Schema Evolution
- Add fields: always safe (clients don't have to use them)
- Remove fields: breaking change (deprecate first)
- Change field type: breaking change
- Make nullable field non-null: breaking change
- Make non-null field nullable: safe

**Deprecation:**
```graphql
type User {
    email: String! @deprecated(reason: "Use 'emails' for multiple addresses")
    emails: [String!]!
}
```

### Key Insights

1. **Types are the contract** between client and server
2. **Validation happens before execution** - fail fast
3. **Introspection enables tooling** - the schema documents itself
4. **Type safety improves** developer experience dramatically
5. **Schema evolution** is manageable with proper design

### Exercises

1. **Design a schema** for a blog with Posts, Authors, Comments, and Tags
2. **Implement validation** for a simple query in Java
3. **Build introspection** - return schema information for a type
4. **Create a custom scalar** for JSON or Money type
5. **Schema versioning** - design a migration path from v1 to v2

---

**Next:** [Chapter 5: Query Language Design - Syntax That Matches Intent í](05-query-language-design.md)

**Previous:** [ê Chapter 3: Thought Experiment - Designing the Perfect Query Language](03-designing-query-language.md)
