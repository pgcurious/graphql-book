# Chapter 9: Query Parsing and AST - Building the Compiler

**Status:** Detailed Outline (To Be Expanded)

---

## Outline

### Part 1: Lexical Analysis (Tokenization)
- Token types in GraphQL
- Building a lexer from scratch
- Handling whitespace, comments, strings
- Error recovery in lexing

### Part 2: Abstract Syntax Tree (AST) Structure
- AST node types: Document, Operation, Field, Fragment
- Why AST instead of direct execution?
- AST visualization
- AST traversal patterns

### Part 3: Parser Implementation
- Recursive descent parsing
- Operator precedence
- Handling fragments and inline fragments
- Parser error messages

### Part 4: Validation Rules
- Document validation
- Field selection validation
- Type compatibility checking
- Fragment spread validation
- Variable usage validation

### Part 5: AST Transformations
- Query optimization at AST level
- Fragment inlining
- Constant folding
- Dead code elimination

### Part 6: Query Complexity Analysis
- Calculating query depth
- Calculating query breadth
- Cost-based analysis
- Rejecting expensive queries before execution

### Java Implementation Topics:
- ANTLR vs hand-written parser
- Visitor pattern for AST traversal
- Immutable AST nodes
- AST caching strategies

---

**Next:** [Chapter 10: Execution Engine ’](10-execution-engine.md)
