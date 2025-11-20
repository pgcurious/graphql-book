# Chapter 8: Schema Design - SDL from Scratch

> "A schema is a contract between the client and server. Like any contract, it should be clear, unambiguous, and designed with both parties in mind."

---

## The Case for a Dedicated Schema Language

In Chapter 4, we explored why type systems are essential for GraphQL. We established that a strongly typed schema serves as the contract between clients and servers. But we glossed over an important question: **how do we actually define this schema?**

Let's consider our options:

### Option 1: Code-First Schema Definition

We could define schemas directly in our programming language:

```java
TypeDefinition userType = new TypeDefinition("User")
    .addField("id", IDType.nonNull())
    .addField("name", StringType.nonNull())
    .addField("email", StringType.nonNull())
    .addField("posts", new ListType(new TypeRef("Post")));

TypeDefinition postType = new TypeDefinition("Post")
    .addField("id", IDType.nonNull())
    .addField("title", StringType.nonNull())
    .addField("author", new TypeRef("User"));
```

**Problems with this approach:**
- **Verbose**: Too much ceremony for simple type definitions
- **Language-specific**: Every language needs its own API
- **Not self-documenting**: Hard to see the overall schema structure
- **Tooling-unfriendly**: Difficult to build IDE support, linters, etc.

### Option 2: Schema Definition Language (SDL)

What if we created a dedicated, language-agnostic syntax specifically designed for defining schemas?

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}
```

**Advantages:**
- **Concise**: Minimal syntax, maximum clarity
- **Language-agnostic**: Works with any programming language
- **Self-documenting**: Easy to read and understand
- **Tooling-friendly**: Simple to parse, validate, and analyze
- **Shareable**: Can be version-controlled and shared across teams

The SDL approach wins. Let's build it from first principles.

---

## Designing the SDL Syntax

What should our schema language look like? Let's derive it step by step.

### Core Requirement: Type Definitions

We need to define object types with fields:

```graphql
type User {
  id: ID!
  name: String!
}
```

**Design decisions:**
- `type` keyword makes it clear we're defining a type
- Curly braces group fields (familiar from many languages)
- Each field has a name and type, separated by colon
- Exclamation mark (`!`) indicates non-null (can't be absent)

### Field Arguments

Fields often need parameters:

```graphql
type Query {
  user(id: ID!): User
  posts(limit: Int, offset: Int): [Post!]!
}
```

**Design decisions:**
- Arguments in parentheses (function-like syntax)
- Multiple arguments separated by commas
- Arguments can have default values: `limit: Int = 10`

### Lists

We need to represent collections:

```graphql
type User {
  posts: [Post!]!
}
```

**Interpretation:**
- `[Post!]!` = a non-null list of non-null Posts
- `[Post]` = a nullable list of nullable Posts
- `[Post!]` = a nullable list of non-null Posts
- `[Post]!` = a non-null list of nullable Posts

The distinction matters! `[Post!]!` means:
1. The list itself is always present (might be empty)
2. Every item in the list is a valid Post (no nulls in the list)

### Scalar Types

Built-in primitive types:

```graphql
scalar DateTime
scalar JSON
scalar URL
```

Standard scalars: `Int`, `Float`, `String`, `Boolean`, `ID`

Custom scalars let you define application-specific types with custom parsing/serialization logic.

### Enums

For constrained value sets:

```graphql
enum Role {
  ADMIN
  EDITOR
  VIEWER
}

type User {
  id: ID!
  role: Role!
}
```

**Benefits:**
- Type safety: Only valid values allowed
- Self-documenting: All possible values visible
- Validation: Invalid values rejected at parse time

### Interfaces

For polymorphism and shared fields:

```graphql
interface Node {
  id: ID!
  createdAt: DateTime!
}

type User implements Node {
  id: ID!
  createdAt: DateTime!
  name: String!
  email: String!
}

type Post implements Node {
  id: ID!
  createdAt: DateTime!
  title: String!
  content: String!
}
```

**Use case:** Any type implementing `Node` guarantees `id` and `createdAt` fields.

Queries can request interface types:

```graphql
query {
  node(id: "123") {
    id
    createdAt
    ... on User {
      name
      email
    }
    ... on Post {
      title
    }
  }
}
```

### Unions

For heterogeneous collections:

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}
```

**When to use unions vs interfaces:**
- **Interface**: Types share common fields (structural relationship)
- **Union**: Types are unrelated, just grouped for a specific purpose

Query example:

```graphql
query {
  search(query: "graphql") {
    ... on User {
      name
    }
    ... on Post {
      title
    }
    ... on Comment {
      text
    }
  }
}
```

### Input Types

For complex arguments (mutations, filters):

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: Role = VIEWER
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}
```

**Why separate input types?**
- Output types can have circular references and complex resolvers
- Input types are simpler: just data structures for validation
- Clear separation of concerns

### Directives

Annotations that modify behavior:

```graphql
type User {
  id: ID!
  name: String!
  email: String! @deprecated(reason: "Use emails field instead")
  emails: [String!]!
  internalId: String! @private
}
```

Built-in directives:
- `@deprecated(reason: String)`: Mark fields as deprecated
- `@skip(if: Boolean)`: Conditionally skip field in query
- `@include(if: Boolean)`: Conditionally include field in query

Custom directives:
```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION

type Query {
  users: [User!]! @auth(requires: ADMIN)
}
```

---

## Schema Design Principles

Now that we have the syntax, how do we design *good* schemas?

### Principle 1: Design for Clients, Not Databases

**Bad (database-centric):**
```graphql
type user_table {
  user_id: Int!
  user_name: String
  user_email: String
  created_timestamp: String
  modified_timestamp: String
}
```

**Good (client-centric):**
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

**Why?**
- GraphQL is an API layer, not a database layer
- Clients don't care about your database schema
- Hide implementation details (table names, column names)
- Use meaningful, domain-appropriate names

### Principle 2: Prefer Non-Null Over Nullable

**Why default to non-null?**

```graphql
type User {
  id: ID!           # Always present
  name: String!     # Always present
  bio: String       # Optional (explicitly nullable)
}
```

**Benefits:**
- Fewer null checks in client code
- Clearer intent: nullable means "legitimately might not exist"
- Better error detection: missing required data fails fast

**When to use nullable:**
- Data might legitimately not exist (user bio, middle name)
- Partial failures shouldn't kill the whole query
- Computed fields that might fail to compute

### Principle 3: Use ID for Identifiers

```graphql
type User {
  id: ID!           # Not Int! or String!
  friendIds: [ID!]! # References to other Users
}
```

**Why ID type?**
- Semantic clarity: "this is an identifier"
- Flexibility: Could be Int, UUID, base64, etc.
- Client caching: ID fields are special for cache keys
- Future-proofing: Can change ID format without breaking API

### Principle 4: Naming Conventions

**Use camelCase for fields:**
```graphql
type User {
  firstName: String!    # Not first_name
  lastName: String!
  createdAt: DateTime!  # Not created_at
}
```

**Use PascalCase for types:**
```graphql
type BlogPost {         # Not blog_post or blogPost
  id: ID!
}
```

**Use SCREAMING_SNAKE_CASE for enum values:**
```graphql
enum Status {
  PENDING_REVIEW        # Not pendingReview or PendingReview
  APPROVED
  REJECTED
}
```

**Why consistency matters:**
- Easier to read and understand
- Aligns with community conventions
- Better tooling support

### Principle 5: Design Connections for Pagination

**Bad (naive list):**
```graphql
type User {
  posts: [Post!]!  # What if user has 10,000 posts?
}
```

**Good (connection pattern):**
```graphql
type User {
  posts(first: Int, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

**Why the complexity?**
- **Cursor-based pagination**: More reliable than offset-based
- **Edge pattern**: Attach metadata to each connection (edge weight, timestamp)
- **PageInfo**: Navigate pages without knowing total count
- **Scalability**: Works with millions of items

This is the **Relay Connection Specification** - a battle-tested pattern.

### Principle 6: Mutation Design

**Pattern: Input object + payload response**

```graphql
input CreatePostInput {
  title: String!
  content: String!
  authorId: ID!
}

type CreatePostPayload {
  post: Post!
  author: User!
  errors: [Error!]
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

**Benefits:**
- **Input object**: Easy to extend with new fields
- **Payload object**: Return multiple pieces of related data
- **Errors field**: Handle partial failures gracefully
- **Non-null payload**: Always return something, even on failure

---

## Advanced Schema Patterns

### Pattern 1: The Node Interface

Global object identification:

```graphql
interface Node {
  id: ID!
}

type Query {
  node(id: ID!): Node
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}
```

**Use case:**
- Client needs to refetch any object by ID
- Normalized caching (Apollo, Relay)
- Globally unique IDs across all types

**ID format:** Base64 encode `typename:localId`
```
User:123   -> base64("User:123") -> "VXNlcjoxMjM="
Post:456   -> base64("Post:456") -> "UG9zdDo0NTY="
```

### Pattern 2: Viewer Pattern

Current authenticated user:

```graphql
type Query {
  viewer: User    # Current user (null if not authenticated)
}

type User {
  id: ID!
  name: String!
  email: String!  # Only visible if viewer is this user
}
```

**Benefits:**
- Clear entry point for authenticated queries
- Easy to add user-specific fields
- Supports authorization logic

### Pattern 3: Payload with Multiple Outcomes

```graphql
type LoginPayload {
  success: Boolean!
  user: User
  token: String
  errors: [LoginError!]!
}

type LoginError {
  field: String
  message: String!
}
```

**Handles multiple scenarios:**
- Success: `success: true, user: {...}, token: "..."`
- Invalid credentials: `success: false, errors: [{message: "Invalid password"}]`
- Account locked: `success: false, errors: [{message: "Account locked"}]`

### Pattern 4: Extending Types Across Modules

**Module 1 (Core):**
```graphql
type User {
  id: ID!
  name: String!
}
```

**Module 2 (Posts):**
```graphql
extend type User {
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
}
```

**Benefits:**
- Modular schema design
- Different teams own different parts
- Supports schema federation

---

## Schema Evolution and Versioning

### Non-Breaking Changes (Safe)

1. **Adding new types**
   ```graphql
   type Comment {  # New type, safe to add
     id: ID!
     text: String!
   }
   ```

2. **Adding new fields**
   ```graphql
   type User {
     id: ID!
     name: String!
     bio: String     # New optional field, safe
   }
   ```

3. **Adding new optional arguments**
   ```graphql
   type Query {
     users(limit: Int, role: Role): [User!]!  # role is new
   }
   ```

4. **Deprecating fields**
   ```graphql
   type User {
     fullName: String! @deprecated(reason: "Use firstName and lastName")
     firstName: String!
     lastName: String!
   }
   ```

### Breaking Changes (Dangerous)

1. **Removing fields**
   ```graphql
   type User {
     id: ID!
     # name: String!  # BREAKING: Removed field
   }
   ```

2. **Changing field types**
   ```graphql
   type User {
     id: String!  # BREAKING: Was ID!, now String!
   }
   ```

3. **Making nullable field non-null**
   ```graphql
   type User {
     bio: String!  # BREAKING: Was String, now String!
   }
   ```

4. **Removing arguments**
   ```graphql
   type Query {
     users: [User!]!  # BREAKING: Removed limit argument
   }
   ```

### Safe Evolution Strategy

1. **Deprecate before removing:**
   ```graphql
   type User {
     email: String! @deprecated(reason: "Use emails array")
     emails: [String!]!
   }
   ```

2. **Monitor usage** of deprecated fields

3. **Communicate** with client teams

4. **Wait** for all clients to migrate

5. **Remove** deprecated fields in next major version

### Schema Validation Tools

Build a schema differ:

```java
public class SchemaDiff {
    public List<SchemaChange> diff(Schema oldSchema, Schema newSchema) {
        List<SchemaChange> changes = new ArrayList<>();

        // Check for removed types
        for (Type oldType : oldSchema.getTypes()) {
            if (!newSchema.hasType(oldType.getName())) {
                changes.add(new BreakingChange(
                    "Type removed: " + oldType.getName()
                ));
            }
        }

        // Check for field changes
        for (Type type : newSchema.getTypes()) {
            Type oldType = oldSchema.getType(type.getName());
            if (oldType != null) {
                changes.addAll(diffFields(oldType, type));
            }
        }

        return changes;
    }

    private List<SchemaChange> diffFields(Type oldType, Type newType) {
        List<SchemaChange> changes = new ArrayList<>();

        for (Field oldField : oldType.getFields()) {
            Field newField = newType.getField(oldField.getName());

            if (newField == null) {
                changes.add(new BreakingChange(
                    oldType.getName() + "." + oldField.getName() + " removed"
                ));
            } else if (!oldField.getType().equals(newField.getType())) {
                changes.add(new BreakingChange(
                    oldType.getName() + "." + oldField.getName() +
                    " type changed from " + oldField.getType() +
                    " to " + newField.getType()
                ));
            }
        }

        return changes;
    }
}
```

---

## Building an SDL Parser in Java

Let's implement a simple SDL parser to understand how schemas are processed.

### Step 1: Tokenization

```java
public class SchemaLexer {
    private final String input;
    private int position = 0;

    public List<Token> tokenize() {
        List<Token> tokens = new ArrayList<>();

        while (position < input.length()) {
            char c = peek();

            if (Character.isWhitespace(c)) {
                position++;
            } else if (Character.isLetter(c)) {
                tokens.add(readIdentifier());
            } else if (c == '{' || c == '}' || c == '(' || c == ')'
                    || c == '[' || c == ']' || c == ':' || c == '!') {
                tokens.add(new Token(TokenType.SYMBOL, String.valueOf(c)));
                position++;
            } else {
                throw new ParseException("Unexpected character: " + c);
            }
        }

        return tokens;
    }

    private Token readIdentifier() {
        int start = position;
        while (position < input.length() &&
               (Character.isLetterOrDigit(peek()) || peek() == '_')) {
            position++;
        }
        String value = input.substring(start, position);

        // Check if it's a keyword
        TokenType type = isKeyword(value) ? TokenType.KEYWORD : TokenType.IDENTIFIER;
        return new Token(type, value);
    }

    private boolean isKeyword(String value) {
        return value.equals("type") || value.equals("interface")
            || value.equals("enum") || value.equals("input")
            || value.equals("scalar") || value.equals("implements");
    }

    private char peek() {
        return input.charAt(position);
    }
}
```

### Step 2: Parsing

```java
public class SchemaParser {
    private final List<Token> tokens;
    private int position = 0;

    public Schema parse() {
        Schema schema = new Schema();

        while (!isAtEnd()) {
            TypeDefinition type = parseTypeDefinition();
            schema.addType(type);
        }

        return schema;
    }

    private TypeDefinition parseTypeDefinition() {
        expect("type");
        String typeName = expectIdentifier();

        List<String> interfaces = new ArrayList<>();
        if (match("implements")) {
            interfaces.add(expectIdentifier());
            while (match("&")) {
                interfaces.add(expectIdentifier());
            }
        }

        expect("{");
        List<FieldDefinition> fields = new ArrayList<>();

        while (!check("}")) {
            fields.add(parseFieldDefinition());
        }

        expect("}");

        return new TypeDefinition(typeName, fields, interfaces);
    }

    private FieldDefinition parseFieldDefinition() {
        String fieldName = expectIdentifier();

        List<Argument> arguments = new ArrayList<>();
        if (match("(")) {
            arguments = parseArguments();
            expect(")");
        }

        expect(":");
        TypeReference type = parseTypeReference();

        return new FieldDefinition(fieldName, type, arguments);
    }

    private TypeReference parseTypeReference() {
        TypeReference type;

        if (match("[")) {
            // List type
            TypeReference itemType = parseTypeReference();
            expect("]");
            type = new ListType(itemType);
        } else {
            // Named type
            String typeName = expectIdentifier();
            type = new NamedType(typeName);
        }

        // Non-null modifier
        if (match("!")) {
            type = new NonNullType(type);
        }

        return type;
    }

    // Token navigation helpers
    private boolean match(String expected) {
        if (check(expected)) {
            position++;
            return true;
        }
        return false;
    }

    private void expect(String expected) {
        if (!match(expected)) {
            throw new ParseException("Expected '" + expected +
                "' but found '" + currentToken().getValue() + "'");
        }
    }

    private String expectIdentifier() {
        Token token = currentToken();
        if (token.getType() != TokenType.IDENTIFIER) {
            throw new ParseException("Expected identifier");
        }
        position++;
        return token.getValue();
    }
}
```

### Step 3: Schema Introspection

One of GraphQL's killer features: the schema can describe itself!

```java
public class SchemaIntrospection {
    public static final String INTROSPECTION_QUERY = """
        query IntrospectionQuery {
          __schema {
            types {
              name
              kind
              fields {
                name
                type {
                  name
                  kind
                  ofType {
                    name
                    kind
                  }
                }
              }
            }
          }
        }
        """;

    public Map<String, Object> introspect(Schema schema) {
        Map<String, Object> result = new HashMap<>();
        List<Map<String, Object>> types = new ArrayList<>();

        for (TypeDefinition type : schema.getTypes()) {
            Map<String, Object> typeData = new HashMap<>();
            typeData.put("name", type.getName());
            typeData.put("kind", "OBJECT");

            List<Map<String, Object>> fields = new ArrayList<>();
            for (FieldDefinition field : type.getFields()) {
                Map<String, Object> fieldData = new HashMap<>();
                fieldData.put("name", field.getName());
                fieldData.put("type", serializeType(field.getType()));
                fields.add(fieldData);
            }
            typeData.put("fields", fields);

            types.add(typeData);
        }

        result.put("types", types);
        return result;
    }
}
```

**Why introspection matters:**
- GraphiQL and other tools can auto-discover schema
- Clients can validate queries before sending
- Documentation auto-generation
- IDE autocomplete

---

## Complete Example: Blog Schema

Let's put it all together:

```graphql
# Scalars
scalar DateTime
scalar URL

# Enums
enum Role {
  ADMIN
  EDITOR
  VIEWER
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

# Interfaces
interface Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Types
type User implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!

  name: String!
  email: String!
  role: Role!
  bio: String
  avatarUrl: URL

  posts(
    first: Int
    after: String
    status: PostStatus
  ): PostConnection!

  comments: [Comment!]!
}

type Post implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!

  title: String!
  content: String!
  excerpt: String
  status: PostStatus!

  author: User!
  comments(first: Int, after: String): CommentConnection!
  tags: [Tag!]!
}

type Comment implements Node & Timestamped {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!

  text: String!
  author: User!
  post: Post!
}

type Tag {
  id: ID!
  name: String!
  posts: [Post!]!
}

# Connection types (pagination)
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type CommentConnection {
  edges: [CommentEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type CommentEdge {
  cursor: String!
  node: Comment!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Inputs
input CreatePostInput {
  title: String!
  content: String!
  excerpt: String
  tagIds: [ID!]!
}

input UpdatePostInput {
  id: ID!
  title: String
  content: String
  excerpt: String
  status: PostStatus
  tagIds: [ID!]
}

input CreateCommentInput {
  postId: ID!
  text: String!
}

# Payloads
type CreatePostPayload {
  post: Post
  errors: [Error!]!
}

type UpdatePostPayload {
  post: Post
  errors: [Error!]!
}

type CreateCommentPayload {
  comment: Comment
  post: Post
  errors: [Error!]!
}

type Error {
  field: String
  message: String!
}

# Root types
type Query {
  # Node interface
  node(id: ID!): Node

  # Viewer
  viewer: User

  # Lists
  users(first: Int, after: String): [User!]!
  posts(
    first: Int
    after: String
    status: PostStatus
  ): PostConnection!

  # Single items
  user(id: ID!): User
  post(id: ID!): Post

  # Search
  search(query: String!): [SearchResult!]!
}

type Mutation {
  # Posts
  createPost(input: CreatePostInput!): CreatePostPayload!
  updatePost(input: UpdatePostInput!): UpdatePostPayload!
  deletePost(id: ID!): Boolean!

  # Comments
  createComment(input: CreateCommentInput!): CreateCommentPayload!
  deleteComment(id: ID!): Boolean!
}

union SearchResult = User | Post | Comment

# Directives
directive @auth(requires: Role!) on FIELD_DEFINITION
directive @rateLimit(max: Int!, window: Int!) on FIELD_DEFINITION
```

---

## Key Takeaways

1. **SDL is language-agnostic** - Works with any programming language
2. **Design for clients** - Not your database schema
3. **Use non-null by default** - Make nullable fields explicit
4. **Connection pattern** - For scalable pagination
5. **Input/Payload pattern** - For flexible mutations
6. **Interfaces and Unions** - For polymorphism
7. **Schema evolution** - Deprecate before removing
8. **Introspection** - Schema describes itself

The schema is the heart of GraphQL. Invest time in good schema design - it pays dividends in client developer experience, API longevity, and system maintainability.

---

**Next:** [Chapter 9: Query Parsing and AST →](09-query-parsing-ast.md)
**Previous:** [← Chapter 7: Resolvers](07-resolvers.md)
