# Chapter 5: Query Language Design - Syntax That Matches Intent

**Status:** Detailed Outline (To Be Expanded)

**What we'll discover:** How GraphQL's syntax was carefully designed to be intuitive, expressive, and efficient.

---

## Outline

### Introduction: Syntax Matters
- "The best syntax disappears" - it should feel natural
- Query language design principles from compiler theory
- Why GraphQL syntax looks the way it does

### Part 1: The Grammar Foundation

#### Basic Query Structure
```graphql
query {           # Operation type
    post(id: 123) {  # Field with arguments
        title        # Scalar field
        author {     # Object field
            name     # Nested scalar
        }
    }
}
```

**Anatomy breakdown:**
- Operation type: `query`, `mutation`, `subscription`
- Field selection: what data to fetch
- Arguments: parameters in parentheses
- Selection set: nested fields in braces

#### Operation Names and Variables
```graphql
query GetPost($postId: ID!, $includeComments: Boolean = false) {
    post(id: $postId) {
        title
        content
        comments @include(if: $includeComments) {
            text
        }
    }
}
```

**Why this matters:**
- Named operations: easier debugging, logging
- Variables: query reuse, security (no string concatenation)
- Default values: sensible defaults

#### Java Implementation - AST Nodes
```java
public abstract class ASTNode {
    private Location location;  // For error messages
}

public class OperationDefinition extends ASTNode {
    private OperationType type;  // QUERY, MUTATION, SUBSCRIPTION
    private String name;
    private List<VariableDefinition> variables;
    private List<Directive> directives;
    private SelectionSet selectionSet;
}

public class FieldNode extends ASTNode {
    private String alias;
    private String name;
    private List<Argument> arguments;
    private List<Directive> directives;
    private SelectionSet selectionSet;
}

public class SelectionSet extends ASTNode {
    private List<Selection> selections;  // Field, FragmentSpread, InlineFragment
}
```

### Part 2: Advanced Query Features

#### Aliases - Multiple Requests to Same Field
```graphql
query {
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

**Result shape:**
```json
{
    "firstPost": { "title": "First" },
    "secondPost": { "title": "Second" },
    "thirdPost": { "title": "Third" }
}
```

**Java processing:**
```java
public class AliasResolver {
    public Map<String, Object> resolveWithAlias(FieldNode field) {
        String key = field.getAlias() != null
            ? field.getAlias()
            : field.getName();

        Object value = resolveField(field);

        return Map.of(key, value);
    }
}
```

#### Fragments - Reusable Field Sets
```graphql
fragment PostDetails on Post {
    id
    title
    content
    author {
        name
        avatarUrl
    }
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

**Why fragments:**
- DRY principle - define once, use many times
- Component-driven data fetching (React components í fragments)
- Maintainability - change in one place

**Java implementation:**
```java
public class FragmentDefinition extends ASTNode {
    private String name;
    private String typeCondition;  // "Post", "User", etc.
    private SelectionSet selectionSet;
}

public class FragmentSpread extends Selection {
    private String fragmentName;

    public SelectionSet expand(Map<String, FragmentDefinition> fragments) {
        FragmentDefinition fragment = fragments.get(fragmentName);
        return fragment.getSelectionSet();
    }
}
```

#### Inline Fragments - Type-Specific Fields
```graphql
query {
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
            post { title }
        }
    }
}
```

**Use case:** Union and interface types

**Java handling:**
```java
public class InlineFragment extends Selection {
    private String typeCondition;
    private SelectionSet selectionSet;

    public boolean appliesTo(Object object) {
        return object.getClass().getSimpleName()
            .equals(typeCondition);
    }
}

public Object resolveWithFragments(
        Object parent,
        List<Selection> selections) {

    Map<String, Object> result = new HashMap<>();

    for (Selection selection : selections) {
        if (selection instanceof FieldNode) {
            // Regular field
        } else if (selection instanceof InlineFragment) {
            InlineFragment fragment = (InlineFragment) selection;
            if (fragment.appliesTo(parent)) {
                // Apply this fragment
                result.putAll(resolve(parent, fragment.getSelectionSet()));
            }
        }
    }

    return result;
}
```

### Part 3: Arguments and Input

#### Argument Types
```graphql
query {
    posts(
        limit: 10                    # Int
        offset: 20                   # Int
        status: PUBLISHED            # Enum
        tags: ["tech", "graphql"]    # List
        filter: {                    # Input object
            authorId: 123
            minLikes: 5
        }
    ) {
        title
    }
}
```

#### Input Objects
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

**Java mapping:**
```java
public class PostFilter {
    private String authorId;
    private Integer minLikes;
    private Integer maxLikes;
    private PostStatus status;
    private List<String> tags;
    private DateRangeInput dateRange;
}

public class PostQueryResolver {
    public List<Post> posts(PostFilter filter) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> query = cb.createQuery(Post.class);
        Root<Post> root = query.from(Post.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getAuthorId() != null) {
            predicates.add(cb.equal(root.get("authorId"), filter.getAuthorId()));
        }

        if (filter.getMinLikes() != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                root.get("likeCount"), filter.getMinLikes()
            ));
        }

        // ... more predicates

        query.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(query).getResultList();
    }
}
```

### Part 4: Directives - Meta-Programming

#### Built-in Directives
```graphql
query GetUser($includeEmail: Boolean!, $skipPhone: Boolean!) {
    user(id: 123) {
        name
        email @include(if: $includeEmail)
        phone @skip(if: $skipPhone)
    }
}
```

**@include(if: Boolean)**: Include field if condition is true
**@skip(if: Boolean)**: Skip field if condition is true

#### Custom Directives
```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION

type Query {
    adminData: [Secret!]! @auth(requires: ADMIN)
    userData: User! @auth(requires: USER)
}
```

**Java implementation:**
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auth {
    Role requires();
}

public enum Role {
    USER, ADMIN, MODERATOR
}

public class AuthDirective implements SchemaDirective {

    @Override
    public Object onField(
            FieldNode field,
            Object source,
            DataFetchingEnvironment env) {

        Auth auth = field.getDirective(Auth.class);
        Role requiredRole = auth.requires();

        User currentUser = env.getContext().getUser();
        if (!currentUser.hasRole(requiredRole)) {
            throw new UnauthorizedException(
                "Requires role: " + requiredRole
            );
        }

        return env.getFieldValue();
    }
}
```

#### Directive Use Cases
1. **Authorization**: `@auth`, `@hasPermission`
2. **Caching**: `@cacheControl(maxAge: 3600)`
3. **Formatting**: `@dateFormat(format: "YYYY-MM-DD")`
4. **Deprecation**: `@deprecated(reason: "Use newField")`
5. **Rate limiting**: `@rateLimit(limit: 100, window: 3600)`

### Part 5: Query Complexity and Cost Analysis

#### The Problem: Expensive Queries
```graphql
query ExpensiveQuery {
    posts {              # 1000 posts
        comments {       # 100 comments each
            author {     # Each author
                posts {  # All their posts
                    comments {  # All comments on those posts
                        # This is millions of records!
                    }
                }
            }
        }
    }
}
```

#### Solution: Query Cost Calculation
```java
public class QueryCostAnalyzer {

    private static final int MAX_QUERY_COST = 1000;

    public int calculateCost(SelectionSet selections, Schema schema) {
        int totalCost = 0;

        for (Selection selection : selections) {
            if (selection instanceof FieldNode) {
                FieldNode field = (FieldNode) selection;
                totalCost += calculateFieldCost(field, schema);
            }
        }

        return totalCost;
    }

    private int calculateFieldCost(FieldNode field, Schema schema) {
        FieldDefinition fieldDef = schema.getField(field.getName());

        // Base cost
        int cost = 1;

        // Multiplier for lists
        if (fieldDef.isList()) {
            Integer limit = field.getArgument("limit");
            int multiplier = limit != null ? limit : 10;  // Default estimate
            cost *= multiplier;
        }

        // Nested selections
        if (field.hasSelections()) {
            int nestedCost = calculateCost(field.getSelectionSet(), schema);
            cost += nestedCost;
        }

        return cost;
    }

    public void validateCost(Query query, Schema schema) {
        int cost = calculateCost(query.getSelectionSet(), schema);

        if (cost > MAX_QUERY_COST) {
            throw new QueryTooComplexException(
                "Query cost " + cost + " exceeds maximum " + MAX_QUERY_COST
            );
        }
    }
}
```

#### Depth Limiting
```java
public class QueryDepthAnalyzer {

    private static final int MAX_DEPTH = 10;

    public int calculateDepth(SelectionSet selections) {
        int maxDepth = 0;

        for (Selection selection : selections) {
            if (selection instanceof FieldNode) {
                FieldNode field = (FieldNode) selection;
                if (field.hasSelections()) {
                    int depth = 1 + calculateDepth(field.getSelectionSet());
                    maxDepth = Math.max(maxDepth, depth);
                }
            }
        }

        return maxDepth;
    }
}
```

### Part 6: Parsing Implementation

#### Lexer - Tokenization
```java
public enum TokenType {
    BRACE_OPEN,      // {
    BRACE_CLOSE,     // }
    PAREN_OPEN,      // (
    PAREN_CLOSE,     // )
    BRACKET_OPEN,    // [
    BRACKET_CLOSE,   // ]
    COLON,           // :
    COMMA,           // ,
    NAME,            // field names, type names
    STRING,          // "value"
    INT,             // 123
    FLOAT,           // 3.14
    DIRECTIVE,       // @include
    SPREAD,          // ...
    EOF
}

public class Token {
    private TokenType type;
    private String value;
    private int line;
    private int column;
}

public class Lexer {
    private String source;
    private int position = 0;

    public List<Token> tokenize() {
        List<Token> tokens = new ArrayList<>();

        while (position < source.length()) {
            char c = source.charAt(position);

            if (Character.isWhitespace(c)) {
                position++;
            } else if (c == '{') {
                tokens.add(new Token(TokenType.BRACE_OPEN, "{"));
                position++;
            } else if (c == '}') {
                tokens.add(new Token(TokenType.BRACE_CLOSE, "}"));
                position++;
            } else if (Character.isLetter(c)) {
                tokens.add(readName());
            } else if (Character.isDigit(c)) {
                tokens.add(readNumber());
            }
            // ... more cases
        }

        return tokens;
    }
}
```

#### Parser - AST Construction
```java
public class Parser {
    private List<Token> tokens;
    private int current = 0;

    public Document parse(List<Token> tokens) {
        this.tokens = tokens;
        List<Definition> definitions = new ArrayList<>();

        while (!isAtEnd()) {
            definitions.add(parseDefinition());
        }

        return new Document(definitions);
    }

    private Definition parseDefinition() {
        Token token = peek();

        if (token.getValue().equals("query") ||
            token.getValue().equals("mutation") ||
            token.getValue().equals("subscription")) {
            return parseOperationDefinition();
        } else if (token.getValue().equals("fragment")) {
            return parseFragmentDefinition();
        }

        throw new ParseException("Unexpected token: " + token);
    }

    private OperationDefinition parseOperationDefinition() {
        Token type = consume();  // query/mutation/subscription

        String name = null;
        if (peek().getType() == TokenType.NAME) {
            name = consume().getValue();
        }

        List<VariableDefinition> variables = new ArrayList<>();
        if (peek().getType() == TokenType.PAREN_OPEN) {
            variables = parseVariableDefinitions();
        }

        SelectionSet selectionSet = parseSelectionSet();

        return new OperationDefinition(
            OperationType.valueOf(type.getValue().toUpperCase()),
            name,
            variables,
            selectionSet
        );
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
}
```

### Key Insights

1. **Syntax mirrors structure** - query shape = result shape
2. **Fragments enable reuse** - critical for component-based UIs
3. **Variables prevent injection** - never concatenate strings
4. **Directives are powerful** - meta-programming for queries
5. **Cost analysis is essential** - prevent DoS via complex queries

### Exercises

1. **Write a lexer** that tokenizes a simple GraphQL query
2. **Implement fragments** - expand fragment spreads in a query
3. **Build cost calculator** - assign costs to fields and calculate total
4. **Design custom directive** - create `@upperCase` that transforms strings
5. **Parse and validate** - build a complete query parser with error messages

---

**Next:** [Chapter 6: The Graph Mental Model - Thinking in Relationships í](06-graph-mental-model.md)

**Previous:** [ê Chapter 4: Type Systems - Why We Need Strongly Typed Schemas](04-type-systems.md)
