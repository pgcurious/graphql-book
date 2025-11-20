# Chapter 18: Federation - Distributing the Graph

**Status:** Detailed Outline (To Be Expanded)

---

## Outline

### Part 1: Why Federation?
- Microservices and GraphQL
- Schema ownership per team
- Independent deployment
- Avoiding the monolithic gateway

### Part 2: Apollo Federation Concepts
- Federated schema architecture
- Entities and keys
- Type extension across services
- The gateway / router

### Part 3: Defining Federated Services
```graphql
# Users service
type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
}

# Posts service
extend type User @key(fields: "id") {
    id: ID! @external
    posts: [Post!]!
}

type Post @key(fields: "id") {
    id: ID!
    title: String!
    authorId: ID!
}
```

### Part 4: Entity Resolution
- Reference resolvers
- __resolveReference implementation
- Batching entity lookups
- Cross-service data fetching

### Part 5: Schema Composition
- Composing multiple schemas
- Handling conflicts
- Type merging
- Directives in federation

### Part 6: The Federation Gateway
- Query planning
- Service orchestration
- Parallel service calls
- Error handling across services

### Part 7: Advanced Federation Patterns
- Computed fields across services
- Value types vs entities
- Shared types
- Custom directives in federation

### Part 8: Federation in Practice
- Service discovery
- Schema registry
- Version management
- Breaking change detection

### Java Implementation Topics:
- Apollo Federation JVM
- Spring Boot federated services
- Service-to-service communication
- Distributed tracing

### Alternatives to Federation:
- Schema stitching
- GraphQL as a BFF layer
- Monolithic GraphQL with modular resolvers

---

**Next:** [Chapter 19: Building a Simple GraphQL Server in Java ’](19-building-server.md)
