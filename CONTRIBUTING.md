# Contributing to "Inventing GraphQL from First Principles"

Thank you for your interest in improving this book! This is an open-source educational project, and contributions are welcome and appreciated.

## Ways to Contribute

### 1. Report Issues
Found a typo, technical error, or confusing explanation?
- Open an issue on GitHub
- Provide the chapter and section
- Explain what's unclear or incorrect
- Suggest how it could be improved

### 2. Suggest Improvements
Have ideas for:
- Better examples
- Additional exercises
- Clearer explanations
- New diagrams

Open an issue with the `enhancement` label and describe your suggestion.

### 3. Fix Typos and Errors
Small fixes are always welcome:
- Spelling and grammar
- Code formatting
- Broken links
- Missing imports in code examples

Submit a PR directly!

### 4. Improve Code Examples
- Add better comments
- Show alternative implementations
- Fix bugs in example code
- Add more comprehensive tests

### 5. Expand Chapter Outlines
Chapters 4-21 are currently outlines. You can help expand them:
- Write full content following the outline
- Add Java code examples
- Create diagrams
- Write exercises

### 6. Translate
Interested in translating to another language? Please open an issue first to coordinate.

## Contribution Guidelines

### Writing Style
- **Conversational but technical**: Explain like you're teaching a colleague
- **First principles approach**: Build intuition before introducing terminology
- **Show, don't just tell**: Use concrete examples and code
- **Avoid jargon**: Define terms when first introduced
- **Progressive complexity**: Start simple, build up

### Code Standards
- **Java 17+** for all examples
- **Spring Boot 3.x** where applicable
- **Follow Java naming conventions**
- **Include comments** explaining non-obvious code
- **Working examples**: Code should compile and run
- **Tests**: Add tests for complex examples

### Example Code Format
```java
/**
 * Brief description of what this does
 */
public class ExampleClass {

    // Clear, descriptive variable names
    private final UserRepository userRepository;

    /**
     * Explain the purpose and key concepts
     */
    public User findUser(Long id) {
        // Comments for non-obvious logic
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }
}
```

### Markdown Guidelines
- Use code fences with language tags: ` ```java `
- Use **bold** for emphasis, *italic* sparingly
- Break up long paragraphs
- Use headers to structure content
- Add links to related chapters

### Diagrams
- ASCII art for simple diagrams (embedded in markdown)
- Mermaid for complex visualizations
- Keep diagrams simple and focused
- Include text description for accessibility

## Pull Request Process

### 1. Fork and Clone
```bash
git clone https://github.com/YOUR_USERNAME/graphql-book.git
cd graphql-book
git checkout -b feature/your-improvement
```

### 2. Make Changes
- Edit the relevant files
- Test any code examples
- Run spell check

### 3. Commit Guidelines
Write clear commit messages:
```
type: brief description

Longer explanation if needed

Fixes #issue-number
```

Types:
- `content`: Book content changes
- `code`: Example code changes
- `fix`: Bug fixes
- `docs`: Documentation updates
- `chore`: Repository maintenance

Examples:
```
content: expand Chapter 4 with type system examples

- Add Java implementation of scalar types
- Include validation examples
- Add exercises

fix: correct N+1 query example in Chapter 11

The previous example had incorrect SQL. Updated to show
actual N+1 behavior with proper queries.

Fixes #42
```

### 4. Submit PR
- Push your branch
- Open PR on GitHub
- Fill out the PR template
- Link related issues

### 5. Code Review
- Address feedback
- Make requested changes
- Be responsive to questions

## Review Criteria

PRs will be reviewed for:
- **Accuracy**: Is the information technically correct?
- **Clarity**: Is it easy to understand?
- **Consistency**: Does it match the book's style?
- **Completeness**: Are code examples complete and tested?
- **Value**: Does it improve the learning experience?

## What Not to Contribute

Please avoid:
- Promotional content
- Off-topic information
- Duplicate content
- Untested code examples
- Large scope changes without discussion first

## Getting Help

Not sure about something?
- Open an issue with questions
- Check existing issues and PRs
- Reference similar chapters as examples

## Recognition

Contributors will be acknowledged in:
- GitHub contributors list
- Acknowledgments section of the book (for significant contributions)

## License

By contributing, you agree that your contributions will be licensed under the same Creative Commons Attribution-ShareAlike 4.0 International License as the rest of the book.

---

**Thank you for helping make this book better!** =O
