# Chapter 15: Mutations and Side Effects

**Status:** Detailed Outline (To Be Expanded)

---

## Outline

### Part 1: Why Mutations are Different from Queries
- Queries are read-only, mutations modify data
- Serial execution vs parallel execution
- Input validation
- Transaction management

### Part 2: Mutation Design Patterns
- Input object pattern
- Payload pattern with clientMutationId
- Optimistic response design
- Idempotent mutations

### Part 3: Implementing Mutations in Java
```java
type Mutation {
    createPost(input: CreatePostInput!): CreatePostPayload!
    updatePost(input: UpdatePostInput!): UpdatePostPayload!
    deletePost(id: ID!): DeletePostPayload!
}

input CreatePostInput {
    title: String!
    content: String!
    authorId: ID!
}

type CreatePostPayload {
    post: Post
    errors: [UserError!]
}
```

### Part 4: Transaction Management
- Database transactions in resolvers
- Rollback on errors
- @Transactional in Spring
- Distributed transactions

### Part 5: Validation and Error Handling
- Input validation
- Business rule validation
- Returning user-friendly errors
- Field-level errors vs mutation-level errors

### Part 6: Authorization in Mutations
- Checking permissions before mutation
- Resource-based authorization
- Audit logging

### Part 7: Optimistic Concurrency Control
- Version fields
- Detecting concurrent modifications
- Conflict resolution strategies

### Part 8: File Uploads
- Multipart request handling
- Stream-based uploads
- Progress tracking
- File validation

### Java Implementation Topics:
- Spring Data JPA for mutations
- Bean validation (@Valid)
- Custom validators
- File upload with graphql-java-servlet

---

**Next:** [Chapter 16: Subscriptions - Real-time Graphs ’](16-subscriptions.md)
