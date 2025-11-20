# Chapter 9: Query Parsing and AST - Building the Compiler

> "Between a string of characters and executable code lies the Abstract Syntax Tree—the bridge that transforms intention into action."

---

## From Text to Execution: The Parsing Pipeline

We've designed a beautiful query language. We've defined a robust type system and schema. But there's a critical gap: **how do we go from a string of text to something our server can actually execute?**

When a client sends this query:

```graphql
{
  user(id: "123") {
    name
    posts(limit: 5) {
      title
    }
  }
}
```

Our server receives it as a plain string. We need to:
1. **Tokenize** it (break it into meaningful pieces)
2. **Parse** it (understand its structure)
3. **Validate** it (ensure it's legal according to our schema)
4. **Transform** it (prepare it for execution)

This is the job of a compiler's front-end. Let's build one for GraphQL.

---

## Part 1: Lexical Analysis (Tokenization)

### The Problem: Characters vs. Meaning

A query is just characters: `{`, `user`, `(`, `id`, `:`, `"123"`, `)`, ...

But we don't think in characters—we think in **tokens**: "left brace", "identifier", "left paren", "argument name", "colon", "string literal", "right paren"...

**Lexical analysis** (lexing) converts a stream of characters into a stream of tokens.

### Token Types in GraphQL

What kinds of tokens do we need?

```java
public enum TokenType {
    // Structural
    BRACE_OPEN,           // {
    BRACE_CLOSE,          // }
    BRACKET_OPEN,         // [
    BRACKET_CLOSE,        // ]
    PAREN_OPEN,           // (
    PAREN_CLOSE,          // )

    // Punctuation
    COLON,                // :
    COMMA,                // ,
    EQUALS,               // =
    AT,                   // @
    PIPE,                 // |
    BANG,                 // !
    DOLLAR,               // $
    SPREAD,               // ...

    // Literals
    NAME,                 // identifier, field names, type names
    INT,                  // 123, -456
    FLOAT,                // 3.14, -2.5e10
    STRING,               // "hello", """multiline"""
    BOOLEAN,              // true, false
    NULL,                 // null

    // Keywords
    QUERY,                // query
    MUTATION,             // mutation
    SUBSCRIPTION,         // subscription
    FRAGMENT,             // fragment
    ON,                   // on

    // Special
    EOF,                  // End of file
    COMMENT               // # comments
}
```

### Building a Lexer

A lexer is a state machine that reads characters and produces tokens:

```java
public class Lexer {
    private final String source;
    private int position = 0;
    private int line = 1;
    private int column = 1;

    public Token nextToken() {
        skipWhitespaceAndComments();

        if (position >= source.length()) {
            return new Token(TokenType.EOF, "", line, column);
        }

        char current = source.charAt(position);

        // Single-character tokens
        switch (current) {
            case '{': return singleChar(TokenType.BRACE_OPEN);
            case '}': return singleChar(TokenType.BRACE_CLOSE);
            case '[': return singleChar(TokenType.BRACKET_OPEN);
            case ']': return singleChar(TokenType.BRACKET_CLOSE);
            case '(': return singleChar(TokenType.PAREN_OPEN);
            case ')': return singleChar(TokenType.PAREN_CLOSE);
            case ':': return singleChar(TokenType.COLON);
            case ',': return singleChar(TokenType.COMMA);
            case '=': return singleChar(TokenType.EQUALS);
            case '@': return singleChar(TokenType.AT);
            case '|': return singleChar(TokenType.PIPE);
            case '!': return singleChar(TokenType.BANG);
            case '$': return singleChar(TokenType.DOLLAR);
        }

        // Multi-character tokens
        if (current == '.') return readSpread();
        if (current == '"') return readString();
        if (current == '-' || Character.isDigit(current)) return readNumber();
        if (Character.isLetter(current) || current == '_') return readName();

        throw new LexerException("Unexpected character: " + current, line, column);
    }

    private Token readName() {
        int start = position;
        while (position < source.length()) {
            char c = source.charAt(position);
            if (!Character.isLetterOrDigit(c) && c != '_') break;
            position++;
            column++;
        }

        String value = source.substring(start, position);
        TokenType type = getKeywordType(value);
        return new Token(type, value, line, start);
    }

    private Token readString() {
        int start = position;
        position++; // skip opening quote
        column++;

        // Check for block string (""")
        if (position + 1 < source.length()
            && source.charAt(position) == '"'
            && source.charAt(position + 1) == '"') {
            return readBlockString();
        }

        StringBuilder value = new StringBuilder();
        while (position < source.length()) {
            char c = source.charAt(position);

            if (c == '"') {
                position++;
                column++;
                return new Token(TokenType.STRING, value.toString(), line, start);
            }

            if (c == '\\') {
                position++;
                column++;
                if (position >= source.length()) {
                    throw new LexerException("Unterminated string", line, column);
                }
                char escaped = source.charAt(position);
                value.append(handleEscapeSequence(escaped));
            } else if (c == '\n') {
                throw new LexerException("Unterminated string (newline)", line, column);
            } else {
                value.append(c);
            }

            position++;
            column++;
        }

        throw new LexerException("Unterminated string (EOF)", line, column);
    }

    private Token readNumber() {
        int start = position;
        boolean isFloat = false;

        // Optional negative sign
        if (source.charAt(position) == '-') {
            position++;
            column++;
        }

        // Integer part
        while (position < source.length() && Character.isDigit(source.charAt(position))) {
            position++;
            column++;
        }

        // Decimal part
        if (position < source.length() && source.charAt(position) == '.') {
            isFloat = true;
            position++;
            column++;
            while (position < source.length() && Character.isDigit(source.charAt(position))) {
                position++;
                column++;
            }
        }

        // Exponent part
        if (position < source.length() &&
            (source.charAt(position) == 'e' || source.charAt(position) == 'E')) {
            isFloat = true;
            position++;
            column++;
            if (position < source.length() &&
                (source.charAt(position) == '+' || source.charAt(position) == '-')) {
                position++;
                column++;
            }
            while (position < source.length() && Character.isDigit(source.charAt(position))) {
                position++;
                column++;
            }
        }

        String value = source.substring(start, position);
        TokenType type = isFloat ? TokenType.FLOAT : TokenType.INT;
        return new Token(type, value, line, start);
    }

    private void skipWhitespaceAndComments() {
        while (position < source.length()) {
            char c = source.charAt(position);

            // Whitespace
            if (c == ' ' || c == '\t' || c == '\r') {
                position++;
                column++;
                continue;
            }

            // Newline
            if (c == '\n') {
                position++;
                line++;
                column = 1;
                continue;
            }

            // Comments
            if (c == '#') {
                while (position < source.length() && source.charAt(position) != '\n') {
                    position++;
                }
                continue;
            }

            break;
        }
    }
}
```

### Error Recovery in Lexing

Good error messages are critical. Instead of crashing on invalid input, we should:

1. **Report precise locations**: "Syntax error at line 5, column 23"
2. **Suggest fixes**: "Expected closing quote for string"
3. **Continue parsing**: Find multiple errors in one pass

```java
public class LexerException extends RuntimeException {
    private final int line;
    private final int column;
    private final String suggestion;

    public LexerException(String message, int line, int column) {
        super(String.format("Lexer error at %d:%d - %s", line, column, message));
        this.line = line;
        this.column = column;
        this.suggestion = suggestFix(message);
    }

    private String suggestFix(String message) {
        if (message.contains("Unterminated string")) {
            return "Add closing quote (\")";
        }
        if (message.contains("Unexpected character")) {
            return "Check for typos or missing punctuation";
        }
        return null;
    }
}
```

---

## Part 2: Abstract Syntax Tree (AST) Structure

### Why AST Instead of Direct Execution?

We could execute queries directly while parsing. Why build an intermediate AST?

**Reasons for AST:**

1. **Validation**: Check the query structure before execution
2. **Optimization**: Transform the query for better performance
3. **Analysis**: Calculate query complexity, depth, cost
4. **Debugging**: Visualize and inspect the query structure
5. **Tooling**: IDEs, linters, formatters all work on AST
6. **Caching**: Reuse parsed queries

### AST Node Types

Every part of a GraphQL query becomes a node in the tree:

```java
// Base interface for all AST nodes
public interface ASTNode {
    NodeType getNodeType();
    SourceLocation getLocation();
    List<ASTNode> getChildren();
}

public enum NodeType {
    // Document structure
    DOCUMENT,
    OPERATION_DEFINITION,
    FRAGMENT_DEFINITION,

    // Selection
    FIELD,
    FRAGMENT_SPREAD,
    INLINE_FRAGMENT,

    // Values
    VARIABLE,
    INT_VALUE,
    FLOAT_VALUE,
    STRING_VALUE,
    BOOLEAN_VALUE,
    NULL_VALUE,
    ENUM_VALUE,
    LIST_VALUE,
    OBJECT_VALUE,

    // Type references
    NAMED_TYPE,
    LIST_TYPE,
    NON_NULL_TYPE,

    // Other
    ARGUMENT,
    DIRECTIVE,
    VARIABLE_DEFINITION
}
```

### Document: The Root Node

Every GraphQL query parses into a Document:

```java
public class Document implements ASTNode {
    private final List<OperationDefinition> operations;
    private final List<FragmentDefinition> fragments;
    private final SourceLocation location;

    @Override
    public NodeType getNodeType() {
        return NodeType.DOCUMENT;
    }

    @Override
    public List<ASTNode> getChildren() {
        List<ASTNode> children = new ArrayList<>();
        children.addAll(operations);
        children.addAll(fragments);
        return children;
    }
}
```

### Operation Definition

Queries, mutations, and subscriptions:

```java
public class OperationDefinition implements ASTNode {
    private final OperationType operationType; // QUERY, MUTATION, SUBSCRIPTION
    private final String name;                 // Optional operation name
    private final List<VariableDefinition> variableDefinitions;
    private final List<Directive> directives;
    private final SelectionSet selectionSet;   // The actual fields requested
    private final SourceLocation location;

    // Example query:
    // query GetUser($id: ID!) {
    //   user(id: $id) { name }
    // }
    //
    // operationType = QUERY
    // name = "GetUser"
    // variableDefinitions = [VariableDefinition("id", "ID!")]
    // selectionSet = [Field("user", args=[Argument("id", Variable("id"))],
    //                       selectionSet=[Field("name")])]
}
```

### Field: The Core Selection

Fields represent data requests:

```java
public class Field implements ASTNode {
    private final String alias;                // Optional field alias
    private final String name;                 // Field name
    private final List<Argument> arguments;    // Field arguments
    private final List<Directive> directives;  // Field directives
    private final SelectionSet selectionSet;   // Nested fields (for objects)
    private final SourceLocation location;

    // Example:
    // userName: user(id: "123") @include(if: $showUser) {
    //   name
    // }
    //
    // alias = "userName"
    // name = "user"
    // arguments = [Argument("id", StringValue("123"))]
    // directives = [Directive("include", args=[Argument("if", Variable("showUser"))])]
    // selectionSet = [Field("name")]
}
```

### Fragment Definitions

Reusable selection sets:

```java
public class FragmentDefinition implements ASTNode {
    private final String name;
    private final TypeCondition typeCondition;
    private final List<Directive> directives;
    private final SelectionSet selectionSet;
    private final SourceLocation location;

    // Example:
    // fragment UserInfo on User {
    //   id
    //   name
    //   email
    // }
    //
    // name = "UserInfo"
    // typeCondition = TypeCondition("User")
    // selectionSet = [Field("id"), Field("name"), Field("email")]
}
```

### Value Nodes

Arguments and variables use value nodes:

```java
public interface ValueNode extends ASTNode {}

public class IntValue implements ValueNode {
    private final int value;
}

public class StringValue implements ValueNode {
    private final String value;
}

public class Variable implements ValueNode {
    private final String name;  // Variable name without $
}

public class ListValue implements ValueNode {
    private final List<ValueNode> values;
}

public class ObjectValue implements ValueNode {
    private final Map<String, ValueNode> fields;
}
```

### AST Visualization

For this query:

```graphql
query GetUserPosts($userId: ID!) {
  user(id: $userId) {
    name
    posts(limit: 5) {
      title
    }
  }
}
```

The AST looks like:

```
Document
└── OperationDefinition (QUERY, "GetUserPosts")
    ├── VariableDefinition
    │   ├── name: "userId"
    │   └── type: NamedType("ID", NON_NULL)
    └── SelectionSet
        └── Field("user")
            ├── Argument("id", Variable("userId"))
            └── SelectionSet
                ├── Field("name")
                └── Field("posts")
                    ├── Argument("limit", IntValue(5))
                    └── SelectionSet
                        └── Field("title")
```

---

## Part 3: Parser Implementation

### Recursive Descent Parsing

GraphQL's grammar is naturally recursive. Queries can nest fields, which can have selections, which can have more fields...

**Recursive descent** parsing matches this structure perfectly:

```java
public class Parser {
    private final Lexer lexer;
    private Token currentToken;

    public Parser(String source) {
        this.lexer = new Lexer(source);
        this.currentToken = lexer.nextToken();
    }

    public Document parseDocument() {
        List<OperationDefinition> operations = new ArrayList<>();
        List<FragmentDefinition> fragments = new ArrayList<>();

        while (currentToken.getType() != TokenType.EOF) {
            if (isOperationStart()) {
                operations.add(parseOperation());
            } else if (currentToken.getType() == TokenType.FRAGMENT) {
                fragments.add(parseFragmentDefinition());
            } else {
                throw new ParseException("Expected operation or fragment",
                                        currentToken.getLocation());
            }
        }

        return new Document(operations, fragments);
    }

    private OperationDefinition parseOperation() {
        SourceLocation location = currentToken.getLocation();

        // Shorthand query: { ... }
        if (currentToken.getType() == TokenType.BRACE_OPEN) {
            SelectionSet selectionSet = parseSelectionSet();
            return new OperationDefinition(
                OperationType.QUERY,
                null,  // no name
                List.of(),  // no variables
                List.of(),  // no directives
                selectionSet,
                location
            );
        }

        // Full operation: query Name($var: Type) { ... }
        OperationType operationType = parseOperationType();
        String name = null;
        if (currentToken.getType() == TokenType.NAME) {
            name = currentToken.getValue();
            advance();
        }

        List<VariableDefinition> variables = List.of();
        if (currentToken.getType() == TokenType.PAREN_OPEN) {
            variables = parseVariableDefinitions();
        }

        List<Directive> directives = parseDirectives();
        SelectionSet selectionSet = parseSelectionSet();

        return new OperationDefinition(operationType, name, variables,
                                      directives, selectionSet, location);
    }

    private SelectionSet parseSelectionSet() {
        expect(TokenType.BRACE_OPEN);

        List<Selection> selections = new ArrayList<>();
        while (currentToken.getType() != TokenType.BRACE_CLOSE) {
            selections.add(parseSelection());
        }

        expect(TokenType.BRACE_CLOSE);
        return new SelectionSet(selections);
    }

    private Selection parseSelection() {
        if (currentToken.getType() == TokenType.SPREAD) {
            return parseFragment();
        }
        return parseField();
    }

    private Field parseField() {
        SourceLocation location = currentToken.getLocation();

        // Parse field name (with optional alias)
        String nameOrAlias = expectName();
        String alias = null;
        String name = nameOrAlias;

        if (currentToken.getType() == TokenType.COLON) {
            advance();
            alias = nameOrAlias;
            name = expectName();
        }

        // Parse arguments
        List<Argument> arguments = List.of();
        if (currentToken.getType() == TokenType.PAREN_OPEN) {
            arguments = parseArguments();
        }

        // Parse directives
        List<Directive> directives = parseDirectives();

        // Parse selection set (if field is an object)
        SelectionSet selectionSet = null;
        if (currentToken.getType() == TokenType.BRACE_OPEN) {
            selectionSet = parseSelectionSet();
        }

        return new Field(alias, name, arguments, directives, selectionSet, location);
    }

    private List<Argument> parseArguments() {
        expect(TokenType.PAREN_OPEN);

        List<Argument> arguments = new ArrayList<>();
        while (currentToken.getType() != TokenType.PAREN_CLOSE) {
            String argName = expectName();
            expect(TokenType.COLON);
            ValueNode value = parseValue();
            arguments.add(new Argument(argName, value));

            if (currentToken.getType() == TokenType.COMMA) {
                advance();
            }
        }

        expect(TokenType.PAREN_CLOSE);
        return arguments;
    }

    private ValueNode parseValue() {
        switch (currentToken.getType()) {
            case INT:
                int intVal = Integer.parseInt(currentToken.getValue());
                advance();
                return new IntValue(intVal);

            case FLOAT:
                double floatVal = Double.parseDouble(currentToken.getValue());
                advance();
                return new FloatValue(floatVal);

            case STRING:
                String strVal = currentToken.getValue();
                advance();
                return new StringValue(strVal);

            case BOOLEAN:
                boolean boolVal = Boolean.parseBoolean(currentToken.getValue());
                advance();
                return new BooleanValue(boolVal);

            case NULL:
                advance();
                return new NullValue();

            case DOLLAR:
                advance();
                String varName = expectName();
                return new Variable(varName);

            case BRACKET_OPEN:
                return parseListValue();

            case BRACE_OPEN:
                return parseObjectValue();

            case NAME:
                // Enum value
                String enumVal = currentToken.getValue();
                advance();
                return new EnumValue(enumVal);

            default:
                throw new ParseException("Expected value, got " + currentToken.getType(),
                                        currentToken.getLocation());
        }
    }

    private void expect(TokenType expectedType) {
        if (currentToken.getType() != expectedType) {
            throw new ParseException(
                String.format("Expected %s, got %s", expectedType, currentToken.getType()),
                currentToken.getLocation()
            );
        }
        advance();
    }

    private String expectName() {
        if (currentToken.getType() != TokenType.NAME) {
            throw new ParseException("Expected name", currentToken.getLocation());
        }
        String name = currentToken.getValue();
        advance();
        return name;
    }

    private void advance() {
        currentToken = lexer.nextToken();
    }
}
```

### Handling Fragments

Fragments add complexity to parsing:

```java
private Selection parseFragment() {
    SourceLocation location = currentToken.getLocation();
    expect(TokenType.SPREAD);  // ...

    // Inline fragment: ... on Type { }
    if (currentToken.getType() == TokenType.ON) {
        advance();
        String typeName = expectName();
        List<Directive> directives = parseDirectives();
        SelectionSet selectionSet = parseSelectionSet();
        return new InlineFragment(typeName, directives, selectionSet, location);
    }

    // Fragment spread: ...FragmentName
    String fragmentName = expectName();
    List<Directive> directives = parseDirectives();
    return new FragmentSpread(fragmentName, directives, location);
}
```

### Parser Error Messages

Good error messages save debugging time:

```java
public class ParseException extends RuntimeException {
    private final SourceLocation location;

    public ParseException(String message, SourceLocation location) {
        super(formatMessage(message, location));
        this.location = location;
    }

    private static String formatMessage(String message, SourceLocation location) {
        return String.format("Parse error at %d:%d - %s\n%s",
            location.getLine(),
            location.getColumn(),
            message,
            getContextSnippet(location)
        );
    }

    private static String getContextSnippet(SourceLocation location) {
        // Show the line where the error occurred with a pointer
        String line = location.getLineContent();
        String pointer = " ".repeat(location.getColumn() - 1) + "^";
        return line + "\n" + pointer;
    }
}
```

Example error output:

```
Parse error at 5:23 - Expected }, got EOF
    posts(limit: 5) {
                      ^
```

---

## Part 4: Validation Rules

Parsing creates an AST, but doesn't guarantee the query is **valid**. We need validation:

### Document Validation

**Rule: Operation names must be unique**

```java
public class UniqueOperationNamesValidator implements Validator {
    @Override
    public List<ValidationError> validate(Document document) {
        Map<String, OperationDefinition> seenNames = new HashMap<>();
        List<ValidationError> errors = new ArrayList<>();

        for (OperationDefinition op : document.getOperations()) {
            String name = op.getName();
            if (name != null) {
                if (seenNames.containsKey(name)) {
                    errors.add(new ValidationError(
                        "Duplicate operation name: " + name,
                        op.getLocation()
                    ));
                }
                seenNames.put(name, op);
            }
        }

        return errors;
    }
}
```

**Rule: Fragment names must be unique**

```java
public class UniqueFragmentNamesValidator implements Validator {
    @Override
    public List<ValidationError> validate(Document document) {
        Map<String, FragmentDefinition> seenNames = new HashMap<>();
        List<ValidationError> errors = new ArrayList<>();

        for (FragmentDefinition fragment : document.getFragments()) {
            String name = fragment.getName();
            if (seenNames.containsKey(name)) {
                errors.add(new ValidationError(
                    "Duplicate fragment name: " + name,
                    fragment.getLocation()
                ));
            }
            seenNames.put(name, fragment);
        }

        return errors;
    }
}
```

### Field Selection Validation

**Rule: Fields must exist on the type**

```java
public class FieldsOnCorrectTypeValidator implements Validator {
    private final Schema schema;

    @Override
    public List<ValidationError> validate(Document document) {
        List<ValidationError> errors = new ArrayList<>();

        for (OperationDefinition operation : document.getOperations()) {
            TypeDefinition parentType = getRootType(operation.getOperationType());
            validateSelectionSet(operation.getSelectionSet(), parentType, errors);
        }

        return errors;
    }

    private void validateSelectionSet(SelectionSet selectionSet,
                                      TypeDefinition parentType,
                                      List<ValidationError> errors) {
        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;
                FieldDefinition fieldDef = parentType.getField(field.getName());

                if (fieldDef == null) {
                    errors.add(new ValidationError(
                        String.format("Field '%s' does not exist on type '%s'",
                            field.getName(), parentType.getName()),
                        field.getLocation()
                    ));
                    continue;
                }

                // Validate nested selections
                if (field.getSelectionSet() != null) {
                    TypeDefinition fieldType = schema.getType(fieldDef.getType());
                    validateSelectionSet(field.getSelectionSet(), fieldType, errors);
                }
            }
        }
    }
}
```

### Type Compatibility Checking

**Rule: Arguments must match expected types**

```java
public class ArgumentsOfCorrectTypeValidator implements Validator {
    private final Schema schema;

    @Override
    public List<ValidationError> validate(Document document) {
        List<ValidationError> errors = new ArrayList<>();

        // Walk the AST and check every argument
        new ASTVisitor() {
            @Override
            public void visitField(Field field) {
                FieldDefinition fieldDef = getFieldDefinition(field);
                if (fieldDef == null) return;

                for (Argument arg : field.getArguments()) {
                    ArgumentDefinition argDef = fieldDef.getArgument(arg.getName());
                    if (argDef == null) continue;

                    if (!isValueCompatibleWithType(arg.getValue(), argDef.getType())) {
                        errors.add(new ValidationError(
                            String.format("Argument '%s' has invalid type", arg.getName()),
                            arg.getLocation()
                        ));
                    }
                }
            }
        }.visit(document);

        return errors;
    }

    private boolean isValueCompatibleWithType(ValueNode value, TypeReference type) {
        // Check if value's type matches expected type
        if (value instanceof IntValue) {
            return type.getName().equals("Int") || type.getName().equals("Float");
        }
        if (value instanceof StringValue) {
            return type.getName().equals("String") || type.getName().equals("ID");
        }
        if (value instanceof Variable) {
            // Variables validated separately
            return true;
        }
        // ... more type checks
        return false;
    }
}
```

### Variable Usage Validation

**Rule: Variables must be defined and used correctly**

```java
public class VariableUsageValidator implements Validator {
    @Override
    public List<ValidationError> validate(Document document) {
        List<ValidationError> errors = new ArrayList<>();

        for (OperationDefinition operation : document.getOperations()) {
            // Collect all variable definitions
            Set<String> definedVars = operation.getVariableDefinitions().stream()
                .map(VariableDefinition::getName)
                .collect(Collectors.toSet());

            // Find all variable usages
            Set<String> usedVars = new HashSet<>();
            new ASTVisitor() {
                @Override
                public void visitVariable(Variable variable) {
                    usedVars.add(variable.getName());

                    if (!definedVars.contains(variable.getName())) {
                        errors.add(new ValidationError(
                            "Variable $" + variable.getName() + " is not defined",
                            variable.getLocation()
                        ));
                    }
                }
            }.visit(operation);

            // Check for unused variables
            for (String definedVar : definedVars) {
                if (!usedVars.contains(definedVar)) {
                    errors.add(new ValidationError(
                        "Variable $" + definedVar + " is defined but not used",
                        operation.getLocation()
                    ));
                }
            }
        }

        return errors;
    }
}
```

### The Visitor Pattern for AST Traversal

Validation requires walking the entire AST. The **Visitor pattern** makes this clean:

```java
public abstract class ASTVisitor {
    public void visit(ASTNode node) {
        switch (node.getNodeType()) {
            case DOCUMENT:
                visitDocument((Document) node);
                break;
            case OPERATION_DEFINITION:
                visitOperationDefinition((OperationDefinition) node);
                break;
            case FIELD:
                visitField((Field) node);
                break;
            case FRAGMENT_SPREAD:
                visitFragmentSpread((FragmentSpread) node);
                break;
            // ... more cases
        }

        // Visit children
        for (ASTNode child : node.getChildren()) {
            visit(child);
        }
    }

    protected void visitDocument(Document document) {}
    protected void visitOperationDefinition(OperationDefinition operation) {}
    protected void visitField(Field field) {}
    protected void visitVariable(Variable variable) {}
    // ... more visit methods
}
```

---

## Part 5: AST Transformations

Once we have a valid AST, we can optimize it before execution:

### Fragment Inlining

Replace fragment spreads with their actual content:

**Before:**

```graphql
query {
  user {
    ...UserFields
  }
}

fragment UserFields on User {
  id
  name
}
```

**After transformation:**

```graphql
query {
  user {
    id
    name
  }
}
```

**Implementation:**

```java
public class FragmentInliner implements ASTTransformer {
    @Override
    public Document transform(Document document) {
        Map<String, FragmentDefinition> fragments = buildFragmentMap(document);

        for (OperationDefinition operation : document.getOperations()) {
            inlineFragments(operation.getSelectionSet(), fragments);
        }

        return document;
    }

    private void inlineFragments(SelectionSet selectionSet,
                                Map<String, FragmentDefinition> fragments) {
        List<Selection> newSelections = new ArrayList<>();

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof FragmentSpread) {
                FragmentSpread spread = (FragmentSpread) selection;
                FragmentDefinition fragment = fragments.get(spread.getName());

                if (fragment != null) {
                    // Replace spread with fragment's selections
                    newSelections.addAll(fragment.getSelectionSet().getSelections());
                }
            } else {
                newSelections.add(selection);

                if (selection instanceof Field) {
                    Field field = (Field) selection;
                    if (field.getSelectionSet() != null) {
                        inlineFragments(field.getSelectionSet(), fragments);
                    }
                }
            }
        }

        selectionSet.setSelections(newSelections);
    }
}
```

### Field Deduplication

Merge duplicate field selections:

**Before:**

```graphql
{
  user {
    name
    email
    name  # duplicate!
  }
}
```

**After:**

```graphql
{
  user {
    name
    email
  }
}
```

### Constant Folding

Evaluate constant expressions at parse time:

```graphql
# Before
{
  posts(limit: $defaultLimit) {  # where $defaultLimit = 10
    title
  }
}

# After (if variable is known at parse time)
{
  posts(limit: 10) {
    title
  }
}
```

---

## Part 6: Query Complexity Analysis

Before executing a query, we should check if it's too expensive:

### Calculating Query Depth

How deeply nested is this query?

```java
public class DepthCalculator {
    public int calculateDepth(SelectionSet selectionSet) {
        int maxDepth = 0;

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;
                int fieldDepth = 1;

                if (field.getSelectionSet() != null) {
                    fieldDepth += calculateDepth(field.getSelectionSet());
                }

                maxDepth = Math.max(maxDepth, fieldDepth);
            }
        }

        return maxDepth;
    }
}
```

**Example:**

```graphql
{
  user {           # depth 1
    posts {        # depth 2
      comments {   # depth 3
        author {   # depth 4
          name     # depth 5
        }
      }
    }
  }
}
```

Depth = 5

### Calculating Query Breadth

How many fields at each level?

```java
public class BreadthCalculator {
    public int calculateBreadth(SelectionSet selectionSet) {
        int totalFields = selectionSet.getSelections().size();

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;
                if (field.getSelectionSet() != null) {
                    totalFields += calculateBreadth(field.getSelectionSet());
                }
            }
        }

        return totalFields;
    }
}
```

### Cost-Based Analysis

Assign costs to fields and calculate total query cost:

```java
public class CostCalculator {
    private final Map<String, Integer> fieldCosts;

    public CostCalculator(Schema schema) {
        this.fieldCosts = calculateFieldCosts(schema);
    }

    public int calculateCost(OperationDefinition operation) {
        return calculateSelectionSetCost(operation.getSelectionSet(), 1);
    }

    private int calculateSelectionSetCost(SelectionSet selectionSet, int multiplier) {
        int totalCost = 0;

        for (Selection selection : selectionSet.getSelections()) {
            if (selection instanceof Field) {
                Field field = (Field) selection;
                int baseCost = fieldCosts.getOrDefault(field.getName(), 1);
                int fieldCost = baseCost * multiplier;

                // List fields multiply cost
                if (isListField(field)) {
                    int listSize = getListSizeEstimate(field);
                    if (field.getSelectionSet() != null) {
                        fieldCost += calculateSelectionSetCost(
                            field.getSelectionSet(),
                            multiplier * listSize
                        );
                    }
                } else if (field.getSelectionSet() != null) {
                    fieldCost += calculateSelectionSetCost(
                        field.getSelectionSet(),
                        multiplier
                    );
                }

                totalCost += fieldCost;
            }
        }

        return totalCost;
    }

    private int getListSizeEstimate(Field field) {
        // Check limit argument
        for (Argument arg : field.getArguments()) {
            if (arg.getName().equals("limit") && arg.getValue() instanceof IntValue) {
                return ((IntValue) arg.getValue()).getValue();
            }
        }
        return 10; // Default estimate
    }
}
```

### Rejecting Expensive Queries

```java
public class QueryComplexityValidator {
    private static final int MAX_DEPTH = 10;
    private static final int MAX_BREADTH = 100;
    private static final int MAX_COST = 1000;

    public void validate(Document document) throws ComplexityException {
        for (OperationDefinition operation : document.getOperations()) {
            int depth = new DepthCalculator().calculateDepth(
                operation.getSelectionSet()
            );
            if (depth > MAX_DEPTH) {
                throw new ComplexityException(
                    "Query depth " + depth + " exceeds maximum " + MAX_DEPTH
                );
            }

            int breadth = new BreadthCalculator().calculateBreadth(
                operation.getSelectionSet()
            );
            if (breadth > MAX_BREADTH) {
                throw new ComplexityException(
                    "Query breadth " + breadth + " exceeds maximum " + MAX_BREADTH
                );
            }

            int cost = new CostCalculator(schema).calculateCost(operation);
            if (cost > MAX_COST) {
                throw new ComplexityException(
                    "Query cost " + cost + " exceeds maximum " + MAX_COST
                );
            }
        }
    }
}
```

---

## Putting It All Together: The Complete Pipeline

Here's how all the pieces fit:

```java
public class GraphQLQueryProcessor {
    private final Schema schema;
    private final List<Validator> validators;
    private final List<ASTTransformer> transformers;

    public ExecutableQuery process(String queryString) {
        // Step 1: Lex and Parse
        Parser parser = new Parser(queryString);
        Document document;
        try {
            document = parser.parseDocument();
        } catch (ParseException e) {
            throw new GraphQLException("Syntax error: " + e.getMessage(), e);
        }

        // Step 2: Validate
        for (Validator validator : validators) {
            List<ValidationError> errors = validator.validate(document);
            if (!errors.isEmpty()) {
                throw new ValidationException(errors);
            }
        }

        // Step 3: Check complexity
        new QueryComplexityValidator().validate(document);

        // Step 4: Transform/Optimize
        for (ASTTransformer transformer : transformers) {
            document = transformer.transform(document);
        }

        // Step 5: Create executable representation
        return new ExecutableQuery(document, schema);
    }
}
```

---

## Key Takeaways

1. **Lexing** breaks text into tokens, handling all the edge cases (strings, numbers, comments)

2. **Parsing** builds an AST from tokens using recursive descent, matching GraphQL's recursive structure

3. **AST nodes** represent every part of a query: operations, fields, fragments, values, types

4. **Validation** ensures queries are legal: fields exist, types match, variables are defined

5. **Transformations** optimize queries: inline fragments, deduplicate fields, fold constants

6. **Complexity analysis** prevents expensive queries: depth limits, breadth limits, cost analysis

7. **Error messages** must be precise and helpful: line/column numbers, context, suggestions

The AST is the bridge between what clients want (query text) and what servers execute (data fetching). It's where we ensure correctness, optimize performance, and provide great developer experience.

---

**Next:** [Chapter 10: Execution Engine - How Queries Actually Run →](10-execution-engine.md)
