# Chapter 15: Mutations and Side Effects

> "Queries ask questions. Mutations change the world. The difference matters more than you might think."

---

## Why Mutations Are Different from Queries

So far, we've focused almost exclusively on reading data. But what about writing data? Creating posts, updating users, deleting comments—all the operations that change state?

GraphQL calls these operations **mutations**.

### Query vs Mutation: More Than Just Semantics

At first glance, mutations look like queries:

```graphql
# Query (read)
query {
  user(id: "123") {
    name
    email
  }
}

# Mutation (write)
mutation {
  updateUser(id: "123", name: "New Name") {
    id
    name
    email
  }
}
```

But there's a crucial difference: **execution semantics**.

### Queries Execute in Parallel

Multiple top-level query fields can execute concurrently:

```graphql
query {
  user(id: "1") { name }      # Can execute in parallel
  posts { title }              # Can execute in parallel
  comments { text }            # Can execute in parallel
}
```

GraphQL can optimize this with parallel execution, batching, etc.

### Mutations Execute Serially

Multiple top-level mutation fields execute **one at a time, in order**:

```graphql
mutation {
  createPost(title: "First") { id }   # Executes first
  createPost(title: "Second") { id }  # Executes second
  createPost(title: "Third") { id }   # Executes third
}
```

**Why serial execution?**
- Mutations have side effects
- Order matters (second mutation might depend on first)
- Avoid race conditions
- Predictable behavior

This is a **fundamental design decision** in GraphQL.

---

## The Input Object Pattern

Bad mutation design:

```graphql
type Mutation {
  createPost(
    title: String!
    content: String!
    excerpt: String
    authorId: ID!
    tagIds: [ID!]
    publishedAt: DateTime
    status: PostStatus
  ): Post
}
```

**Problems:**
- Long argument lists
- Hard to extend
- Unclear which fields are required
- No way to represent nested data
- Difficult for client code generation

### Better: Input Object Pattern

```graphql
input CreatePostInput {
  title: String!
  content: String!
  excerpt: String
  authorId: ID!
  tagIds: [ID!]
  publishedAt: DateTime
  status: PostStatus
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

**Benefits:**
- Single argument
- Easy to extend (add fields to input)
- Self-documenting structure
- Reusable input types
- Better client code generation

### Input Types vs Object Types

**Critical distinction:**

```graphql
# Output type (can't be used as input)
type Post {
  id: ID!
  title: String!
  author: User!        # References another type
  comments: [Comment!] # Circular references OK
}

# Input type (can be used as argument)
input CreatePostInput {
  title: String!
  authorId: ID!        # Must use ID, not User type
  # No circular references allowed
}
```

**Why separate types?**
- Input types are simpler (just data)
- Output types can have resolvers, circular refs, etc.
- Clear separation of concerns
- Type safety

---

## The Payload Pattern

What should a mutation return?

### Bad: Return the Entity Directly

```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post
}
```

**Problems:**
- Can't return validation errors
- Can't return related data
- Can't return metadata
- Nullable return type (what if it fails?)

### Better: Return a Payload

```graphql
type CreatePostPayload {
  post: Post
  errors: [UserError!]!
  clientMutationId: String
}

type UserError {
  field: String
  message: String!
  code: String
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
}
```

**Benefits:**
- Non-null payload (always returns something)
- Can return errors without throwing
- Can include related data
- Can include metadata

### Success Case

```graphql
mutation {
  createPost(input: {
    title: "GraphQL Mutations"
    content: "..."
    authorId: "123"
  }) {
    post {
      id
      title
      author {
        name
      }
    }
    errors {
      message
    }
  }
}
```

Response:
```json
{
  "data": {
    "createPost": {
      "post": {
        "id": "456",
        "title": "GraphQL Mutations",
        "author": {
          "name": "Alice"
        }
      },
      "errors": []
    }
  }
}
```

### Error Case

```graphql
mutation {
  createPost(input: {
    title: ""  # Invalid: empty title
    content: "..."
    authorId: "999"  # Invalid: user doesn't exist
  }) {
    post {
      id
    }
    errors {
      field
      message
      code
    }
  }
}
```

Response:
```json
{
  "data": {
    "createPost": {
      "post": null,
      "errors": [
        {
          "field": "title",
          "message": "Title cannot be empty",
          "code": "VALIDATION_ERROR"
        },
        {
          "field": "authorId",
          "message": "User not found",
          "code": "NOT_FOUND"
        }
      ]
    }
  }
}
```

**Notice:** The request succeeds (no GraphQL error), but the payload contains validation errors.

---

## Implementing Mutations in Java

### Step 1: Define Input Types

```java
public class CreatePostInput {
    @NotBlank(message = "Title is required")
    @Size(max = 200, message = "Title must be less than 200 characters")
    private String title;

    @NotBlank(message = "Content is required")
    private String content;

    private String excerpt;

    @NotNull(message = "Author ID is required")
    private Long authorId;

    private List<Long> tagIds = new ArrayList<>();

    private LocalDateTime publishedAt;

    @NotNull
    private PostStatus status = PostStatus.DRAFT;

    // Getters and setters
}
```

### Step 2: Define Payload Types

```java
public class CreatePostPayload {
    private Post post;
    private List<UserError> errors;

    public CreatePostPayload(Post post) {
        this.post = post;
        this.errors = Collections.emptyList();
    }

    public CreatePostPayload(List<UserError> errors) {
        this.post = null;
        this.errors = errors;
    }

    // Getters
}

public class UserError {
    private final String field;
    private final String message;
    private final String code;

    public UserError(String field, String message, String code) {
        this.field = field;
        this.message = message;
        this.code = code;
    }

    // Getters
}
```

### Step 3: Implement Mutation Resolver

```java
@Component
public class MutationResolver implements GraphQLMutationResolver {

    @Autowired
    private PostService postService;

    @Transactional
    public CreatePostPayload createPost(@Valid CreatePostInput input) {
        try {
            // Validate input
            List<UserError> errors = validateCreatePostInput(input);
            if (!errors.isEmpty()) {
                return new CreatePostPayload(errors);
            }

            // Create post
            Post post = postService.createPost(input);

            return new CreatePostPayload(post);

        } catch (UserNotFoundException e) {
            return new CreatePostPayload(List.of(
                new UserError("authorId", "User not found", "NOT_FOUND")
            ));
        } catch (Exception e) {
            return new CreatePostPayload(List.of(
                new UserError(null, "Internal server error", "INTERNAL_ERROR")
            ));
        }
    }

    private List<UserError> validateCreatePostInput(CreatePostInput input) {
        List<UserError> errors = new ArrayList<>();

        if (input.getTitle().isBlank()) {
            errors.add(new UserError(
                "title",
                "Title cannot be empty",
                "VALIDATION_ERROR"
            ));
        }

        if (input.getContent().length() > 10000) {
            errors.add(new UserError(
                "content",
                "Content too long (max 10000 characters)",
                "VALIDATION_ERROR"
            ));
        }

        return errors;
    }
}
```

### Step 4: Service Layer with Business Logic

```java
@Service
public class PostService {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TagRepository tagRepository;

    public Post createPost(CreatePostInput input) {
        // Verify author exists
        User author = userRepository.findById(input.getAuthorId())
            .orElseThrow(() -> new UserNotFoundException(input.getAuthorId()));

        // Create post entity
        Post post = new Post();
        post.setTitle(input.getTitle());
        post.setContent(input.getContent());
        post.setExcerpt(input.getExcerpt());
        post.setAuthor(author);
        post.setStatus(input.getStatus());
        post.setPublishedAt(input.getPublishedAt());
        post.setCreatedAt(LocalDateTime.now());
        post.setUpdatedAt(LocalDateTime.now());

        // Associate tags
        if (input.getTagIds() != null && !input.getTagIds().isEmpty()) {
            List<Tag> tags = tagRepository.findAllById(input.getTagIds());
            post.setTags(tags);
        }

        // Save
        return postRepository.save(post);
    }
}
```

---

## Transaction Management

Mutations often involve multiple database operations that should succeed or fail together.

### Using @Transactional

```java
@Component
public class MutationResolver implements GraphQLMutationResolver {

    @Transactional  // Wrap entire mutation in transaction
    public CreatePostPayload createPost(CreatePostInput input) {
        // All database operations here are in one transaction
        // If any exception is thrown, everything rolls back
    }
}
```

### Manual Transaction Control

```java
@Component
public class MutationResolver {

    @Autowired
    private PlatformTransactionManager transactionManager;

    public CreatePostPayload createPost(CreatePostInput input) {
        TransactionDefinition txDef = new DefaultTransactionDefinition();
        TransactionStatus txStatus = transactionManager.getTransaction(txDef);

        try {
            // Perform mutations
            Post post = postService.createPost(input);

            // Commit if successful
            transactionManager.commit(txStatus);
            return new CreatePostPayload(post);

        } catch (Exception e) {
            // Rollback on error
            transactionManager.rollback(txStatus);
            return new CreatePostPayload(List.of(
                new UserError(null, e.getMessage(), "ERROR")
            ));
        }
    }
}
```

### Complex Multi-Step Mutations

```java
@Transactional
public PublishPostPayload publishPost(PublishPostInput input) {
    // Step 1: Load post
    Post post = postRepository.findById(input.getPostId())
        .orElseThrow(() -> new PostNotFoundException(input.getPostId()));

    // Step 2: Validate post can be published
    if (post.getStatus() == PostStatus.PUBLISHED) {
        return new PublishPostPayload(List.of(
            new UserError("postId", "Post is already published", "INVALID_STATE")
        ));
    }

    // Step 3: Update post status
    post.setStatus(PostStatus.PUBLISHED);
    post.setPublishedAt(LocalDateTime.now());

    // Step 4: Send notifications
    notificationService.notifyFollowers(post.getAuthor(), post);

    // Step 5: Update search index
    searchService.indexPost(post);

    // Step 6: Save
    post = postRepository.save(post);

    // All steps succeed or all rollback
    return new PublishPostPayload(post);
}
```

---

## Update Mutations

Updates are trickier than creates:

```graphql
input UpdatePostInput {
  id: ID!
  title: String           # Optional: only update if provided
  content: String         # Optional
  excerpt: String         # Optional
  status: PostStatus      # Optional
  tagIds: [ID!]           # Optional: replaces existing tags
}

type Mutation {
  updatePost(input: UpdatePostInput!): UpdatePostPayload!
}
```

### Partial Updates

```java
@Transactional
public UpdatePostPayload updatePost(UpdatePostInput input) {
    // Load existing post
    Post post = postRepository.findById(input.getId())
        .orElseThrow(() -> new PostNotFoundException(input.getId()));

    // Check authorization
    if (!authService.canEdit(currentUser, post)) {
        return new UpdatePostPayload(List.of(
            new UserError("id", "Not authorized to edit this post", "FORBIDDEN")
        ));
    }

    // Apply updates (only if provided)
    if (input.getTitle() != null) {
        post.setTitle(input.getTitle());
    }

    if (input.getContent() != null) {
        post.setContent(input.getContent());
    }

    if (input.getExcerpt() != null) {
        post.setExcerpt(input.getExcerpt());
    }

    if (input.getStatus() != null) {
        // Business rule: can't unpublish a post
        if (post.getStatus() == PostStatus.PUBLISHED
            && input.getStatus() != PostStatus.PUBLISHED) {
            return new UpdatePostPayload(List.of(
                new UserError("status", "Cannot unpublish a post", "INVALID_STATE")
            ));
        }
        post.setStatus(input.getStatus());
    }

    if (input.getTagIds() != null) {
        List<Tag> tags = tagRepository.findAllById(input.getTagIds());
        post.setTags(tags);
    }

    post.setUpdatedAt(LocalDateTime.now());

    // Save
    post = postRepository.save(post);

    return new UpdatePostPayload(post);
}
```

---

## Delete Mutations

### Soft Delete vs Hard Delete

```graphql
type Mutation {
  # Soft delete: mark as deleted
  deletePost(id: ID!): DeletePostPayload!

  # Hard delete: permanently remove
  permanentlyDeletePost(id: ID!): DeletePostPayload!
}

type DeletePostPayload {
  deletedPostId: ID
  errors: [UserError!]!
}
```

### Implementation

```java
@Transactional
public DeletePostPayload deletePost(Long id) {
    Post post = postRepository.findById(id)
        .orElseThrow(() -> new PostNotFoundException(id));

    // Check authorization
    if (!authService.canDelete(currentUser, post)) {
        return new DeletePostPayload(List.of(
            new UserError("id", "Not authorized", "FORBIDDEN")
        ));
    }

    // Soft delete
    post.setDeletedAt(LocalDateTime.now());
    postRepository.save(post);

    return new DeletePostPayload(id);
}

@Transactional
public DeletePostPayload permanentlyDeletePost(Long id) {
    Post post = postRepository.findById(id)
        .orElseThrow(() -> new PostNotFoundException(id));

    // Check authorization (requires admin)
    if (!authService.isAdmin(currentUser)) {
        return new DeletePostPayload(List.of(
            new UserError("id", "Admin access required", "FORBIDDEN")
        ));
    }

    // Hard delete
    postRepository.delete(post);

    return new DeletePostPayload(id);
}
```

---

## Optimistic Concurrency Control

Problem: Two users edit the same post simultaneously.

### Version-Based Locking

```graphql
type Post {
  id: ID!
  title: String!
  content: String!
  version: Int!  # Incremented on each update
}

input UpdatePostInput {
  id: ID!
  version: Int!  # Client must provide current version
  title: String
  content: String
}
```

### Implementation

```java
@Entity
@Table(name = "posts")
public class Post {
    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private String content;

    @Version  // JPA optimistic locking
    private Integer version;

    // ...
}

@Transactional
public UpdatePostPayload updatePost(UpdatePostInput input) {
    Post post = postRepository.findById(input.getId())
        .orElseThrow(() -> new PostNotFoundException(input.getId()));

    // Check version
    if (!post.getVersion().equals(input.getVersion())) {
        return new UpdatePostPayload(List.of(
            new UserError(
                "version",
                "Post was modified by someone else. Please reload and try again.",
                "CONFLICT"
            )
        ));
    }

    // Apply updates
    post.setTitle(input.getTitle());
    post.setContent(input.getContent());

    // JPA automatically increments version
    post = postRepository.save(post);

    return new UpdatePostPayload(post);
}
```

**Client workflow:**
1. Fetch post with current version
2. User edits post
3. Submit update with version
4. If version matches: update succeeds
5. If version mismatch: show conflict error, ask user to merge changes

---

## Idempotent Mutations

Problem: Network failure might cause client to retry mutation. How do we avoid duplicate operations?

### Client Mutation ID Pattern

```graphql
input CreatePostInput {
  clientMutationId: String!  # Client-generated unique ID
  title: String!
  content: String!
}

type CreatePostPayload {
  post: Post
  clientMutationId: String!  # Echo back to client
  errors: [UserError!]!
}
```

### Implementation

```java
@Transactional
public CreatePostPayload createPost(CreatePostInput input) {
    // Check if we've already processed this mutation
    Optional<Post> existing = postRepository
        .findByClientMutationId(input.getClientMutationId());

    if (existing.isPresent()) {
        // Already processed - return cached result
        return new CreatePostPayload(
            existing.get(),
            input.getClientMutationId()
        );
    }

    // Process mutation
    Post post = new Post();
    post.setTitle(input.getTitle());
    post.setContent(input.getContent());
    post.setClientMutationId(input.getClientMutationId());

    post = postRepository.save(post);

    return new CreatePostPayload(post, input.getClientMutationId());
}
```

**Benefits:**
- Safe to retry mutations
- No duplicate side effects
- Client can verify which mutation succeeded

---

## Authorization in Mutations

### Field-Level Authorization

```java
@Transactional
public UpdatePostPayload updatePost(UpdatePostInput input) {
    Post post = postRepository.findById(input.getId())
        .orElseThrow(() -> new PostNotFoundException(input.getId()));

    User currentUser = authService.getCurrentUser();

    // Check if user can edit this post
    if (!post.getAuthor().equals(currentUser) && !currentUser.isAdmin()) {
        throw new ForbiddenException("Not authorized to edit this post");
    }

    // Continue with update...
}
```

### Resource-Based Authorization

```java
public interface AuthorizationService {
    boolean canCreate(User user, Class<?> resourceType);
    boolean canRead(User user, Object resource);
    boolean canUpdate(User user, Object resource);
    boolean canDelete(User user, Object resource);
}

@Service
public class PostAuthorizationService implements AuthorizationService {

    @Override
    public boolean canUpdate(User user, Object resource) {
        if (!(resource instanceof Post)) {
            return false;
        }

        Post post = (Post) resource;

        // Author can edit their own posts
        if (post.getAuthor().equals(user)) {
            return true;
        }

        // Admins can edit any post
        if (user.hasRole(Role.ADMIN)) {
            return true;
        }

        // Editors can edit published posts
        if (user.hasRole(Role.EDITOR) && post.isPublished()) {
            return true;
        }

        return false;
    }
}
```

### Using Annotations

```java
@Component
public class MutationResolver {

    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public DeletePostPayload deletePost(Long id) {
        // Only admins can call this
    }

    @PreAuthorize("@postAuthService.canUpdate(authentication.principal, #input.id)")
    @Transactional
    public UpdatePostPayload updatePost(UpdatePostInput input) {
        // Checks custom authorization logic
    }
}
```

---

## File Uploads

GraphQL doesn't natively support file uploads. But we can handle them.

### Schema

```graphql
scalar Upload

type Mutation {
  uploadAvatar(file: Upload!): UploadAvatarPayload!
}

type UploadAvatarPayload {
  url: String
  errors: [UserError!]!
}
```

### Implementation with Multipart Requests

```java
@Component
public class MutationResolver {

    @Autowired
    private FileStorageService fileStorage;

    public UploadAvatarPayload uploadAvatar(Part file) {
        try {
            // Validate file
            if (file.getSize() > 5_000_000) {  // 5MB limit
                return new UploadAvatarPayload(List.of(
                    new UserError("file", "File too large (max 5MB)", "VALIDATION_ERROR")
                ));
            }

            String contentType = file.getContentType();
            if (!contentType.startsWith("image/")) {
                return new UploadAvatarPayload(List.of(
                    new UserError("file", "Must be an image file", "VALIDATION_ERROR")
                ));
            }

            // Store file
            String filename = UUID.randomUUID().toString() + getExtension(file);
            String url = fileStorage.store(file.getInputStream(), filename);

            // Update user's avatar
            User user = authService.getCurrentUser();
            user.setAvatarUrl(url);
            userRepository.save(user);

            return new UploadAvatarPayload(url);

        } catch (IOException e) {
            return new UploadAvatarPayload(List.of(
                new UserError("file", "Failed to upload file", "INTERNAL_ERROR")
            ));
        }
    }
}
```

### Client Request

```http
POST /graphql
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="operations"

{"query": "mutation($file: Upload!) { uploadAvatar(file: $file) { url } }", "variables": {"file": null}}
------WebKitFormBoundary
Content-Disposition: form-data; name="map"

{"0": ["variables.file"]}
------WebKitFormBoundary
Content-Disposition: form-data; name="0"; filename="avatar.jpg"
Content-Type: image/jpeg

[binary data]
------WebKitFormBoundary--
```

---

## Batch Mutations

What if you need to create multiple items at once?

### Option 1: Multiple Mutation Calls

```graphql
mutation {
  post1: createPost(input: {...}) { post { id } }
  post2: createPost(input: {...}) { post { id } }
  post3: createPost(input: {...}) { post { id } }
}
```

Executes serially (slow).

### Option 2: Batch Mutation Field

```graphql
input CreatePostInput {
  title: String!
  content: String!
}

type Mutation {
  createPosts(inputs: [CreatePostInput!]!): CreatePostsPayload!
}

type CreatePostsPayload {
  posts: [Post!]!
  errors: [UserError!]!
}
```

```java
@Transactional
public CreatePostsPayload createPosts(List<CreatePostInput> inputs) {
    List<Post> posts = new ArrayList<>();
    List<UserError> errors = new ArrayList<>();

    for (int i = 0; i < inputs.size(); i++) {
        CreatePostInput input = inputs.get(i);

        try {
            Post post = createSinglePost(input);
            posts.add(post);
        } catch (ValidationException e) {
            errors.add(new UserError(
                "inputs[" + i + "]",
                e.getMessage(),
                "VALIDATION_ERROR"
            ));
        }
    }

    return new CreatePostsPayload(posts, errors);
}
```

---

## Key Takeaways

1. **Mutations execute serially** - Order matters, unlike queries
2. **Input object pattern** - Single input argument, easy to extend
3. **Payload pattern** - Always return a payload with data + errors
4. **Transactions** - Use `@Transactional` to ensure atomicity
5. **Validation** - Return user-friendly errors in payload
6. **Authorization** - Check permissions before mutating
7. **Idempotency** - Use client mutation IDs for safe retries
8. **Versioning** - Use optimistic locking to detect conflicts
9. **Non-null payloads** - Always return something, even on error
10. **Separate input types** - Input types are simpler than output types

Mutations are where the rubber meets the road. Get them right, and your API will be robust, predictable, and pleasant to use. Get them wrong, and you'll spend months debugging race conditions, lost updates, and frustrated users.

---

**Next:** [Chapter 16: Subscriptions - Real-time Graphs →](16-subscriptions.md)
**Previous:** [← Chapter 14: Batching and Performance Optimization](14-batching-performance.md)
