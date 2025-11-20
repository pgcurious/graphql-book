# Chapter 5: Query Language Design - Syntax That Matches Intent

> "The best syntax disappears. It should feel like you're describing your intent, not instructing a machine."

**What we'll discover:** How GraphQL's syntax was carefully designed to be intuitive, expressive, and efficient—and why every design decision matters.

---

## Introduction: Syntax Matters

We've built a type system. We know what data exists and what shape it takes. Now comes the crucial question: **How do clients express what they want?**

This isn't a trivial design problem. The syntax you choose affects:
- **Ease of use**: Can developers write queries quickly?
- **Readability**: Can you understand a query at a glance?
- **Tooling**: Can IDEs autocomplete and validate?
- **Performance**: Can the server optimize execution?
- **Safety**: Can you prevent injection attacks?

Let's design the query language from first principles.

## Part 1: The Grammar Foundation

### Starting with the Basics

What's the simplest possible query? Asking for a single field:

```graphql
{
  hello
}
```

Result:
```json
{
  "data": {
    "hello": "world"
  }
}
```

**Key insight:** The query shape matches the result shape. When you write `{ hello }`, you get `{ "hello": "..." }`. This is a fundamental design principle: **queries look like the data they return**.

### Adding Complexity: Nested Objects

What if `hello` isn't a string, but an object with fields?

```graphql
{
  user {
    name
    email
  }
}
```

Result:
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com"
    }
  }
}
```

The nesting in the query mirrors the nesting in the response. No surprises.

### Fields with Arguments

Fields can take arguments to parameterize the request:

```graphql
{
  post(id: 123) {
    title
    content
  }
}
```

**Syntax choices:**
- Arguments in parentheses: `post(id: 123)`
- Key-value pairs: `id: 123`
- No quotes needed for numbers: `123` not `"123"`

**Why parentheses?** Familiar from function calls in most programming languages. It signals "this field is parameterized."

### The Complete Query Structure

A full query has several parts:

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

**Anatomy:**
1. **Operation type**: `query` (also `mutation`, `subscription`)
2. **Field selection**: `post`
3. **Arguments**: `(id: 123)`
4. **Selection set**: `{ title, author { name } }`

### Operation Names and Variables

In production, you don't hardcode values. You use variables:

```graphql
query GetPost($postId: ID!) {
  post(id: $postId) {
    title
    content
  }
}
```

**Variables:**
```json
{
  "postId": "123"
}
```

**Why this matters:**
1. **Query reuse**: Same query, different variables
2. **Security**: No string concatenation, no injection attacks
3. **Type safety**: Variables are typed (`$postId: ID!`)
4. **Logging**: Named operations appear in logs

**Default values:**

```graphql
query GetPost($postId: ID!, $includeComments: Boolean = false) {
  post(id: $postId) {
    title
    comments @include(if: $includeComments) {
      text
    }
  }
}
```

If `includeComments` isn't provided, it defaults to `false`.

### Implementing the AST in Java

Every query is parsed into an Abstract Syntax Tree (AST):

```java
public abstract class ASTNode {
    private Location location;  // Line and column for error messages

    public Location getLocation() {
        return location;
    }
}

public class Document extends ASTNode {
    private List<Definition> definitions;

    public List<Definition> getDefinitions() {
        return definitions;
    }
}

public class OperationDefinition extends Definition {
    private OperationType type;  // QUERY, MUTATION, SUBSCRIPTION
    private String name;
    private List<VariableDefinition> variableDefinitions;
    private SelectionSet selectionSet;

    public OperationType getType() { return type; }
    public String getName() { return name; }
    public List<VariableDefinition> getVariableDefinitions() {
        return variableDefinitions;
    }
    public SelectionSet getSelectionSet() { return selectionSet; }
}

public enum OperationType {
    QUERY, MUTATION, SUBSCRIPTION
}

public class FieldNode extends Selection {
    private String alias;
    private String name;
    private List<Argument> arguments;
    private List<Directive> directives;
    private SelectionSet selectionSet;

    public String getAlias() { return alias; }
    public String getName() { return name; }
    public List<Argument> getArguments() { return arguments; }
    public SelectionSet getSelectionSet() { return selectionSet; }
}

public class SelectionSet extends ASTNode {
    private List<Selection> selections;

    public List<Selection> getSelections() {
        return selections;
    }
}

public abstract class Selection extends ASTNode {
    // Base class for Field, FragmentSpread, InlineFragment
}
```

This AST represents the query structure in memory, ready for validation and execution.

## Part 2: Advanced Query Features

### Aliases: Requesting the Same Field Multiple Times

What if you want to fetch multiple posts in one query?

**Problem:**

```graphql
{
  post(id: 1) {
    title
  }
  post(id: 2) {
    title
  }
}
```

This doesn't work. You can't have duplicate keys in the result.

**Solution: Aliases**

```graphql
{
  firstPost: post(id: 1) {
    title
  }
  secondPost: post(id: 2) {
    title
  }
  thirdPost: post(id: 3) {
    title
  }
}
```

Result:

```json
{
  "data": {
    "firstPost": { "title": "Hello World" },
    "secondPost": { "title": "GraphQL Intro" },
    "thirdPost": { "title": "Advanced Patterns" }
  }
}
```

**Syntax:** `alias: field(arguments)`

The alias becomes the key in the response, but the field name is still used for resolution.

**Java implementation:**

```java
public class AliasResolver {

    public Map<String, Object> resolveField(FieldNode field, Object parent) {
        // Resolve the actual field
        Object value = fetchFieldValue(field.getName(), field.getArguments(), parent);

        // Use alias if present, otherwise use field name
        String key = field.getAlias() != null
            ? field.getAlias()
            : field.getName();

        return Map.of(key, value);
    }
}
```

### Fragments: Reusable Field Sets

Imagine you're building a React app with a post component that always needs the same fields:

```graphql
{
  post1: post(id: 1) {
    id
    title
    content
    author {
      name
      avatarUrl
    }
    createdAt
  }
  post2: post(id: 2) {
    id
    title
    content
    author {
      name
      avatarUrl
    }
    createdAt
  }
}
```

This is repetitive. Enter **fragments**:

```graphql
fragment PostDetails on Post {
  id
  title
  content
  author {
    name
    avatarUrl
  }
  createdAt
}

query {
  post1: post(id: 1) {
    ...PostDetails
  }
  post2: post(id: 2) {
    ...PostDetails
  }
}
```

**Benefits:**
1. **DRY principle**: Define once, use everywhere
2. **Component-driven**: Each React component has its own fragment
3. **Maintainability**: Change the fragment, all uses update
4. **Composition**: Fragments can include other fragments

**Syntax:**
- **Definition**: `fragment FragmentName on TypeName { fields }`
- **Use**: `...FragmentName`

**Java implementation:**

```java
public class FragmentDefinition extends Definition {
    private String name;
    private String typeCondition;
    private SelectionSet selectionSet;

    public FragmentDefinition(String name, String typeCondition, SelectionSet selectionSet) {
        this.name = name;
        this.typeCondition = typeCondition;
        this.selectionSet = selectionSet;
    }

    public String getName() { return name; }
    public String getTypeCondition() { return typeCondition; }
    public SelectionSet getSelectionSet() { return selectionSet; }
}

public class FragmentSpread extends Selection {
    private String fragmentName;

    public FragmentSpread(String fragmentName) {
        this.fragmentName = fragmentName;
    }

    public SelectionSet expand(Map<String, FragmentDefinition> fragments) {
        FragmentDefinition fragment = fragments.get(fragmentName);
        if (fragment == null) {
            throw new ValidationException("Fragment '" + fragmentName + "' not found");
        }
        return fragment.getSelectionSet();
    }
}

public class FragmentExpander {

    public SelectionSet expandFragments(
            SelectionSet selectionSet,
            Map<String, FragmentDefinition> fragments) {

        List<Selection> expandedSelections = new ArrayList<>();

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof FragmentSpread) {
                FragmentSpread spread = (FragmentSpread) selection;
                SelectionSet fragmentSelections = spread.expand(fragments);
                expandedSelections.addAll(fragmentSelections.getSelections());
            } else {
                expandedSelections.add(selection);
            }
        }

        return new SelectionSet(expandedSelections);
    }
}
```

### Inline Fragments: Type-Specific Fields

Fragments have another use: handling union and interface types.

Recall from Chapter 4:

```graphql
union SearchResult = Post | User | Comment
```

When querying a union, you need to specify which fields to fetch for each type:

```graphql
{
  search(query: "hello") {
    ... on Post {
      title
      content
    }
    ... on User {
      name
      email
    }
    ... on Comment {
      text
      author {
        name
      }
    }
  }
}
```

**Inline fragment syntax:** `... on TypeName { fields }`

The fragment is only applied if the result is of that type.

**Result:**

```json
{
  "data": {
    "search": [
      {
        "title": "Hello World",
        "content": "..."
      },
      {
        "name": "Alice",
        "email": "alice@example.com"
      },
      {
        "text": "Great post!",
        "author": { "name": "Bob" }
      }
    ]
  }
}
```

**Java implementation:**

```java
public class InlineFragment extends Selection {
    private String typeCondition;
    private SelectionSet selectionSet;

    public InlineFragment(String typeCondition, SelectionSet selectionSet) {
        this.typeCondition = typeCondition;
        this.selectionSet = selectionSet;
    }

    public boolean appliesTo(Object object) {
        return object.getClass().getSimpleName().equals(typeCondition);
    }

    public String getTypeCondition() { return typeCondition; }
    public SelectionSet getSelectionSet() { return selectionSet; }
}

public class UnionResolver {

    public Map<String, Object> resolveUnionField(
            FieldNode field,
            Object parent,
            List<InlineFragment> inlineFragments) {

        Map<String, Object> result = new HashMap<>();

        // Determine the actual type of the result
        Object value = fetchFieldValue(field.getName(), parent);

        // Apply the appropriate inline fragment
        for (InlineFragment fragment : inlineFragments) {
            if (fragment.appliesTo(value)) {
                Map<String, Object> fragmentResult =
                    resolveSelectionSet(fragment.getSelectionSet(), value);
                result.putAll(fragmentResult);
            }
        }

        return result;
    }
}
```

## Part 3: Arguments and Input

### Argument Types

Arguments can be any scalar or input type:

```graphql
{
  posts(
    limit: 10                         # Int
    offset: 20                        # Int
    status: PUBLISHED                 # Enum
    tags: ["tech", "graphql"]         # List of Strings
    filter: {                         # Input Object
      authorId: 123
      minLikes: 5
    }
  ) {
    title
  }
}
```

### Input Objects: Complex Arguments

For complex filters, use input objects:

```graphql
input PostFilter {
  authorId: ID
  minLikes: Int
  maxLikes: Int
  status: PostStatus
  tags: [String!]
  dateRange: DateRangeInput
}

input DateRangeInput {
  start: DateTime!
  end: DateTime!
}

type Query {
  posts(filter: PostFilter): [Post!]!
}
```

**Usage:**

```graphql
query {
  posts(filter: {
    authorId: "123",
    minLikes: 10,
    status: PUBLISHED,
    dateRange: {
      start: "2024-01-01T00:00:00Z",
      end: "2024-12-31T23:59:59Z"
    }
  }) {
    title
  }
}
```

**Java implementation:**

```java
public class PostFilter {
    private String authorId;
    private Integer minLikes;
    private Integer maxLikes;
    private PostStatus status;
    private List<String> tags;
    private DateRangeInput dateRange;

    // Getters and setters
    public String getAuthorId() { return authorId; }
    public void setAuthorId(String authorId) { this.authorId = authorId; }
    // ... etc
}

public class DateRangeInput {
    private Instant start;
    private Instant end;

    public Instant getStart() { return start; }
    public void setStart(Instant start) { this.start = start; }
    public Instant getEnd() { return end; }
    public void setEnd(Instant end) { this.end = end; }
}

public class PostQueryResolver {

    @Autowired
    private EntityManager entityManager;

    public List<Post> posts(PostFilter filter) {
        if (filter == null) {
            return entityManager
                .createQuery("SELECT p FROM Post p", Post.class)
                .getResultList();
        }

        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> query = cb.createQuery(Post.class);
        Root<Post> root = query.from(Post.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getAuthorId() != null) {
            predicates.add(cb.equal(
                root.get("authorId"),
                filter.getAuthorId()
            ));
        }

        if (filter.getMinLikes() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                root.get("likeCount"),
                filter.getMinLikes()
            ));
        }

        if (filter.getMaxLikes() != null) {
            predicates.add(cb.lessThanOrEqualTo(
                root.get("likeCount"),
                filter.getMaxLikes()
            ));
        }

        if (filter.getStatus() != null) {
            predicates.add(cb.equal(
                root.get("status"),
                filter.getStatus()
            ));
        }

        if (filter.getDateRange() != null) {
            predicates.add(cb.between(
                root.get("createdAt"),
                filter.getDateRange().getStart(),
                filter.getDateRange().getEnd()
            ));
        }

        query.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(query).getResultList();
    }
}
```

Input objects make complex queries clean and type-safe.

## Part 4: Directives - Meta-Programming

Directives are annotations that modify execution behavior.

### Built-in Directives

**@include(if: Boolean)**: Include field if condition is true

```graphql
query GetUser($includeEmail: Boolean!) {
  user(id: 123) {
    name
    email @include(if: $includeEmail)
  }
}
```

If `includeEmail` is `false`, the `email` field is not fetched or returned.

**@skip(if: Boolean)**: Skip field if condition is true

```graphql
query GetUser($skipPhone: Boolean!) {
  user(id: 123) {
    name
    phone @skip(if: $skipPhone)
  }
}
```

If `skipPhone` is `true`, the `phone` field is omitted.

**Why use directives instead of separate queries?**

Directives let you parameterize queries without creating multiple versions. One query handles all cases.

### Custom Directives

You can define your own directives for cross-cutting concerns:

**Authorization:**

```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION

enum Role {
  USER
  ADMIN
  MODERATOR
}

type Query {
  adminData: [Secret!]! @auth(requires: ADMIN)
  userData: User! @auth(requires: USER)
}
```

**Java implementation:**

```java
public interface SchemaDirective {
    Object onField(
        FieldNode field,
        Object source,
        DataFetchingEnvironment env
    ) throws Exception;
}

public class AuthDirective implements SchemaDirective {

    @Override
    public Object onField(
            FieldNode field,
            Object source,
            DataFetchingEnvironment env) throws Exception {

        // Get the required role from the directive
        Directive directive = field.getDirective("auth");
        Role requiredRole = (Role) directive.getArgument("requires").getValue();

        // Get current user from context
        GraphQLContext context = env.getContext();
        User currentUser = context.get("user");

        if (currentUser == null || !currentUser.hasRole(requiredRole)) {
            throw new UnauthorizedException(
                "This field requires role: " + requiredRole
            );
        }

        // User is authorized, proceed with field resolution
        return env.getDataFetcher().get(env);
    }
}
```

### Common Directive Use Cases

1. **Authorization**: `@auth`, `@hasPermission`
2. **Caching**: `@cacheControl(maxAge: 3600)`
3. **Formatting**: `@dateFormat(format: "YYYY-MM-DD")`, `@upperCase`
4. **Deprecation**: `@deprecated(reason: "Use newField instead")`
5. **Rate limiting**: `@rateLimit(limit: 100, window: 3600)`
6. **Logging**: `@log(level: INFO)`

Directives let you add behavior without cluttering resolvers.

## Part 5: Query Complexity and Cost Analysis

### The Problem: Expensive Queries

GraphQL's flexibility creates a problem: clients can craft expensive queries that overwhelm your server.

**Example:**

```graphql
{
  posts {                    # 1000 posts
    comments {               # 100 comments per post
      author {               # Each comment has an author
        posts {              # Each author has posts
          comments {         # Each post has comments
            author {         # Each comment has an author
              # This explodes exponentially!
            }
          }
        }
      }
    }
  }
}
```

This query could fetch millions of records and crash your server.

### Solution 1: Query Depth Limiting

Limit how deeply queries can nest:

```java
public class QueryDepthAnalyzer {

    private static final int MAX_DEPTH = 10;

    public int calculateDepth(SelectionSet selectionSet) {
        return calculateDepthRecursive(selectionSet, 0);
    }

    private int calculateDepthRecursive(SelectionSet selectionSet, int currentDepth) {
        int maxDepth = currentDepth;

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof FieldNode) {
                FieldNode field = (FieldNode) selection;
                if (field.getSelectionSet() != null) {
                    int depth = calculateDepthRecursive(
                        field.getSelectionSet(),
                        currentDepth + 1
                    );
                    maxDepth = Math.max(maxDepth, depth);
                }
            }
        }

        return maxDepth;
    }

    public void validateDepth(Document document) {
        for (Definition definition : document.getDefinitions()) {
            if (definition instanceof OperationDefinition) {
                OperationDefinition operation = (OperationDefinition) definition;
                int depth = calculateDepth(operation.getSelectionSet());

                if (depth > MAX_DEPTH) {
                    throw new QueryTooDeepException(
                        "Query depth " + depth + " exceeds maximum " + MAX_DEPTH
                    );
                }
            }
        }
    }
}
```

### Solution 2: Query Cost Calculation

Assign a cost to each field and limit total cost:

```java
public class QueryCostAnalyzer {

    private static final int MAX_QUERY_COST = 1000;
    private final GraphQLSchema schema;

    public int calculateCost(SelectionSet selectionSet, GraphQLType parentType) {
        int totalCost = 0;

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof FieldNode) {
                FieldNode field = (FieldNode) selection;
                totalCost += calculateFieldCost(field, parentType);
            }
        }

        return totalCost;
    }

    private int calculateFieldCost(FieldNode field, GraphQLType parentType) {
        GraphQLFieldDefinition fieldDef = getFieldDefinition(parentType, field.getName());

        // Base cost (from schema annotation or default to 1)
        int baseCost = fieldDef.getDirective("cost") != null
            ? (int) fieldDef.getDirective("cost").getArgument("value").getValue()
            : 1;

        int cost = baseCost;

        // If field returns a list, multiply by list size
        if (GraphQLTypeUtil.isList(fieldDef.getType())) {
            Integer limit = getArgumentValue(field, "limit");
            int multiplier = limit != null ? limit : 10; // Default estimate
            cost *= multiplier;
        }

        // Add cost of nested selections
        if (field.getSelectionSet() != null) {
            GraphQLType fieldType = GraphQLTypeUtil.unwrapAll(fieldDef.getType());
            int nestedCost = calculateCost(field.getSelectionSet(), fieldType);
            cost += nestedCost;
        }

        return cost;
    }

    public void validateCost(OperationDefinition operation) {
        int cost = calculateCost(
            operation.getSelectionSet(),
            schema.getQueryType()
        );

        if (cost > MAX_QUERY_COST) {
            throw new QueryTooComplexException(
                "Query cost " + cost + " exceeds maximum " + MAX_QUERY_COST
            );
        }
    }

    private Integer getArgumentValue(FieldNode field, String argName) {
        for (Argument arg : field.getArguments()) {
            if (arg.getName().equals(argName)) {
                return (Integer) arg.getValue();
            }
        }
        return null;
    }
}
```

**Usage in schema:**

```graphql
type Query {
  posts: [Post!]! @cost(value: 10)
  user(id: ID!): User @cost(value: 1)
}
```

### Solution 3: Timeout

Set a maximum execution time:

```java
public class TimeoutExecutor {

    private static final long MAX_EXECUTION_TIME_MS = 5000;

    public ExecutionResult executeWithTimeout(
            GraphQL graphql,
            ExecutionInput executionInput) {

        ExecutorService executor = Executors.newSingleThreadExecutor();

        Future<ExecutionResult> future = executor.submit(() ->
            graphql.execute(executionInput)
        );

        try {
            return future.get(MAX_EXECUTION_TIME_MS, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            throw new QueryTimeoutException(
                "Query execution exceeded " + MAX_EXECUTION_TIME_MS + "ms"
            );
        } catch (Exception e) {
            throw new RuntimeException("Query execution failed", e);
        } finally {
            executor.shutdown();
        }
    }
}
```

## Part 6: Parsing Implementation

Let's see how to turn query strings into AST nodes.

### Lexer: Tokenization

The lexer breaks the query string into tokens:

```java
public enum TokenType {
    // Structural
    BRACE_OPEN,      // {
    BRACE_CLOSE,     // }
    PAREN_OPEN,      // (
    PAREN_CLOSE,     // )
    BRACKET_OPEN,    // [
    BRACKET_CLOSE,   // ]

    // Syntax
    COLON,           // :
    COMMA,           // ,
    EQUALS,          // =
    PIPE,            // |
    BANG,            // !
    SPREAD,          // ...
    AT,              // @

    // Literals
    NAME,            // field names, type names
    STRING,          // "value"
    INT,             // 123
    FLOAT,           // 3.14
    BOOLEAN,         // true, false
    NULL,            // null

    EOF
}

public class Token {
    private final TokenType type;
    private final String value;
    private final int line;
    private final int column;

    public Token(TokenType type, String value, int line, int column) {
        this.type = type;
        this.value = value;
        this.line = line;
        this.column = column;
    }

    public TokenType getType() { return type; }
    public String getValue() { return value; }
    public int getLine() { return line; }
    public int getColumn() { return column; }
}

public class Lexer {
    private final String source;
    private int position = 0;
    private int line = 1;
    private int column = 1;

    public Lexer(String source) {
        this.source = source;
    }

    public List<Token> tokenize() {
        List<Token> tokens = new ArrayList<>();

        while (position < source.length()) {
            char c = current();

            if (Character.isWhitespace(c)) {
                if (c == '\n') {
                    line++;
                    column = 1;
                } else {
                    column++;
                }
                position++;
            } else if (c == '#') {
                skipComment();
            } else if (c == '{') {
                tokens.add(new Token(TokenType.BRACE_OPEN, "{", line, column));
                advance();
            } else if (c == '}') {
                tokens.add(new Token(TokenType.BRACE_CLOSE, "}", line, column));
                advance();
            } else if (c == '(') {
                tokens.add(new Token(TokenType.PAREN_OPEN, "(", line, column));
                advance();
            } else if (c == ')') {
                tokens.add(new Token(TokenType.PAREN_CLOSE, ")", line, column));
                advance();
            } else if (c == ':') {
                tokens.add(new Token(TokenType.COLON, ":", line, column));
                advance();
            } else if (c == '"') {
                tokens.add(readString());
            } else if (Character.isDigit(c) || c == '-') {
                tokens.add(readNumber());
            } else if (Character.isLetter(c) || c == '_') {
                tokens.add(readName());
            } else if (c == '.') {
                if (peek(1) == '.' && peek(2) == '.') {
                    tokens.add(new Token(TokenType.SPREAD, "...", line, column));
                    advance();
                    advance();
                    advance();
                }
            } else {
                throw new LexerException(
                    "Unexpected character: " + c + " at line " + line + ", column " + column
                );
            }
        }

        tokens.add(new Token(TokenType.EOF, "", line, column));
        return tokens;
    }

    private char current() {
        return source.charAt(position);
    }

    private char peek(int offset) {
        int pos = position + offset;
        return pos < source.length() ? source.charAt(pos) : '\0';
    }

    private void advance() {
        position++;
        column++;
    }

    private void skipComment() {
        while (position < source.length() && current() != '\n') {
            advance();
        }
    }

    private Token readString() {
        int startColumn = column;
        advance(); // Skip opening quote

        StringBuilder value = new StringBuilder();
        while (position < source.length() && current() != '"') {
            if (current() == '\\') {
                advance();
                if (position < source.length()) {
                    value.append(current());
                }
            } else {
                value.append(current());
            }
            advance();
        }

        advance(); // Skip closing quote
        return new Token(TokenType.STRING, value.toString(), line, startColumn);
    }

    private Token readNumber() {
        int startColumn = column;
        StringBuilder value = new StringBuilder();

        if (current() == '-') {
            value.append(current());
            advance();
        }

        while (position < source.length() && Character.isDigit(current())) {
            value.append(current());
            advance();
        }

        TokenType type = TokenType.INT;

        if (position < source.length() && current() == '.') {
            type = TokenType.FLOAT;
            value.append(current());
            advance();

            while (position < source.length() && Character.isDigit(current())) {
                value.append(current());
                advance();
            }
        }

        return new Token(type, value.toString(), line, startColumn);
    }

    private Token readName() {
        int startColumn = column;
        StringBuilder value = new StringBuilder();

        while (position < source.length() &&
               (Character.isLetterOrDigit(current()) || current() == '_')) {
            value.append(current());
            advance();
        }

        String name = value.toString();

        // Check for keywords
        if (name.equals("true") || name.equals("false")) {
            return new Token(TokenType.BOOLEAN, name, line, startColumn);
        } else if (name.equals("null")) {
            return new Token(TokenType.NULL, name, line, startColumn);
        }

        return new Token(TokenType.NAME, name, line, startColumn);
    }
}
```

### Parser: AST Construction

The parser consumes tokens and builds the AST:

```java
public class Parser {
    private final List<Token> tokens;
    private int current = 0;

    public Parser(List<Token> tokens) {
        this.tokens = tokens;
    }

    public Document parse() {
        List<Definition> definitions = new ArrayList<>();

        while (!isAtEnd()) {
            definitions.add(parseDefinition());
        }

        return new Document(definitions);
    }

    private Definition parseDefinition() {
        Token token = peek();

        String value = token.getValue();
        if (value.equals("query") || value.equals("mutation") || value.equals("subscription")) {
            return parseOperationDefinition();
        } else if (value.equals("fragment")) {
            return parseFragmentDefinition();
        }

        throw new ParseException("Unexpected token: " + token.getValue());
    }

    private OperationDefinition parseOperationDefinition() {
        Token typeToken = consume();
        OperationType type = OperationType.valueOf(typeToken.getValue().toUpperCase());

        String name = null;
        if (peek().getType() == TokenType.NAME) {
            name = consume().getValue();
        }

        List<VariableDefinition> variables = new ArrayList<>();
        if (peek().getType() == TokenType.PAREN_OPEN) {
            variables = parseVariableDefinitions();
        }

        SelectionSet selectionSet = parseSelectionSet();

        return new OperationDefinition(type, name, variables, selectionSet);
    }

    private SelectionSet parseSelectionSet() {
        consume(TokenType.BRACE_OPEN);

        List<Selection> selections = new ArrayList<>();

        while (peek().getType() != TokenType.BRACE_CLOSE) {
            selections.add(parseSelection());
        }

        consume(TokenType.BRACE_CLOSE);
        return new SelectionSet(selections);
    }

    private Selection parseSelection() {
        if (peek().getType() == TokenType.SPREAD) {
            consume(); // ...

            if (peek().getValue().equals("on")) {
                // Inline fragment
                consume(); // on
                String typeCondition = consume(TokenType.NAME).getValue();
                SelectionSet selectionSet = parseSelectionSet();
                return new InlineFragment(typeCondition, selectionSet);
            } else {
                // Fragment spread
                String fragmentName = consume(TokenType.NAME).getValue();
                return new FragmentSpread(fragmentName);
            }
        } else {
            return parseField();
        }
    }

    private FieldNode parseField() {
        String aliasOrName = consume(TokenType.NAME).getValue();
        String name = aliasOrName;
        String alias = null;

        if (peek().getType() == TokenType.COLON) {
            consume(); // :
            alias = aliasOrName;
            name = consume(TokenType.NAME).getValue();
        }

        List<Argument> arguments = new ArrayList<>();
        if (peek().getType() == TokenType.PAREN_OPEN) {
            arguments = parseArguments();
        }

        SelectionSet selectionSet = null;
        if (peek().getType() == TokenType.BRACE_OPEN) {
            selectionSet = parseSelectionSet();
        }

        return new FieldNode(alias, name, arguments, selectionSet);
    }

    private List<Argument> parseArguments() {
        consume(TokenType.PAREN_OPEN);

        List<Argument> arguments = new ArrayList<>();

        while (peek().getType() != TokenType.PAREN_CLOSE) {
            String name = consume(TokenType.NAME).getValue();
            consume(TokenType.COLON);
            Object value = parseValue();
            arguments.add(new Argument(name, value));

            if (peek().getType() == TokenType.COMMA) {
                consume();
            }
        }

        consume(TokenType.PAREN_CLOSE);
        return arguments;
    }

    private Object parseValue() {
        Token token = peek();

        switch (token.getType()) {
            case INT:
                return Integer.parseInt(consume().getValue());
            case FLOAT:
                return Double.parseDouble(consume().getValue());
            case STRING:
                return consume().getValue();
            case BOOLEAN:
                return Boolean.parseBoolean(consume().getValue());
            case NULL:
                consume();
                return null;
            case NAME:
                // Variable or enum value
                if (token.getValue().startsWith("$")) {
                    return new VariableReference(consume().getValue());
                } else {
                    return consume().getValue(); // Enum value
                }
            case BRACKET_OPEN:
                return parseListValue();
            case BRACE_OPEN:
                return parseObjectValue();
            default:
                throw new ParseException("Unexpected value token: " + token);
        }
    }

    private Token peek() {
        return tokens.get(current);
    }

    private Token consume() {
        return tokens.get(current++);
    }

    private Token consume(TokenType expected) {
        Token token = peek();
        if (token.getType() != expected) {
            throw new ParseException(
                "Expected " + expected + " but got " + token.getType()
            );
        }
        return consume();
    }

    private boolean isAtEnd() {
        return peek().getType() == TokenType.EOF;
    }
}
```

## Key Insights

Let's distill what we've learned:

1. **Syntax mirrors structure**: The query shape matches the result shape. This makes GraphQL intuitive and predictable.

2. **Fragments enable reuse**: Critical for component-based UIs where each component declares its data needs.

3. **Variables prevent injection**: Never concatenate query strings. Always use variables.

4. **Directives are powerful**: Meta-programming for queries without bloating resolvers.

5. **Cost analysis is essential**: Prevent DoS attacks by analyzing query complexity before execution.

6. **Aliases enable batching**: Fetch multiple variations of the same field in one query.

7. **Type conditions handle polymorphism**: Inline fragments query unions and interfaces cleanly.

The query language is carefully designed for developer ergonomics, safety, and performance. Every syntax choice serves a purpose.

## What's Next

We've designed the type system and query language. But we're still thinking in terms of fields and objects.

In the next chapter, we'll shift our mental model: instead of thinking about endpoints and resources, we'll think about **graphs**.

---

**Next:** [Chapter 6: The Graph Mental Model - Thinking in Relationships →](06-graph-mental-model.md)

**Previous:** [← Chapter 4: Type Systems - Why We Need Strongly Typed Schemas](04-type-systems.md)
