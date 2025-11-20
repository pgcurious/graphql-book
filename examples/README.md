# GraphQL Book - Code Examples

This directory contains all the working code examples from the book "Inventing GraphQL from First Principles."

## Prerequisites

- Java 17 or higher
- Maven 3.6+
- Your favorite IDE (IntelliJ IDEA, Eclipse, or VS Code with Java extensions)

## Structure

Each chapter with code examples has its own directory:

### Part 1: The Problem Space
- `chapter-01-rest-api/` - REST API examples showing the problems we're solving

### Part 5: Implementation
- **`chapter-19-complete-server/`** - Complete GraphQL server built from scratch
  - Full Spring Boot application
  - JPA integration
  - DataLoader implementation
  - Comprehensive tests
  - Ready to run!

## Running Examples

Each example directory contains its own README with specific instructions. Generally:

```bash
cd chapter-XX-name
mvn clean install
mvn spring-boot:run
```

Then visit http://localhost:8080/graphiql to explore the GraphQL API.

## Example Projects Overview

### Chapter 19: Complete GraphQL Server

A fully functional social network API with:
- Users, Posts, and Comments
- Query operations (reading data)
- Mutation operations (creating/updating data)
- Relationships and nested queries
- DataLoader for N+1 prevention
- Error handling
- Input validation

**Technologies:**
- Spring Boot 3.x
- GraphQL Java Spring Boot Starter
- Spring Data JPA
- H2 Database (for easy setup)
- JUnit 5 for testing

## Development Workflow

### 1. Clone and Setup
```bash
git clone <repository-url>
cd graphql-book/examples
```

### 2. Build All Examples
```bash
# From examples directory
for dir in chapter-*/; do
    if [ -f "$dir/pom.xml" ]; then
        echo "Building $dir..."
        (cd "$dir" && mvn clean install)
    fi
done
```

### 3. Run Individual Example
```bash
cd chapter-19-complete-server
mvn spring-boot:run
```

### 4. Run Tests
```bash
mvn test
```

## IDE Setup

### IntelliJ IDEA
1. Open the specific chapter folder as a project
2. IDEA will automatically detect the Maven project
3. Wait for dependencies to download
4. Run the `Application.java` main class

### VS Code
1. Install Java Extension Pack
2. Open the chapter folder
3. VS Code will detect Maven project
4. Use "Run" button or terminal: `mvn spring-boot:run`

### Eclipse
1. Import as "Existing Maven Project"
2. Select the chapter directory
3. Right-click on project ’ Run As ’ Spring Boot App

## Learning Path

We recommend working through the examples in this order:

1. **Chapter 1** - Understand REST API limitations
2. **Chapter 19** - See the complete solution
3. Go back through earlier chapters to understand how each piece works

## Troubleshooting

### Port Already in Use
If port 8080 is already in use, modify `application.yml`:
```yaml
server:
  port: 8081
```

### Build Failures
```bash
# Clean Maven cache
mvn clean

# Update dependencies
mvn dependency:resolve

# Skip tests if needed
mvn clean install -DskipTests
```

### Database Issues
The examples use H2 in-memory database. If you see database errors:
- The database resets on each restart (expected behavior)
- Check `application.yml` for database configuration
- H2 console: http://localhost:8080/h2-console

## Contributing Examples

Have an improvement or found a bug? See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## Questions?

- Open an issue on GitHub
- Refer to the book chapters for detailed explanations
- Check the chapter-specific README for example-specific help

---

**Happy coding!** =€
