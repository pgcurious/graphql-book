# Chapter 19: Building a Simple GraphQL Server in Java

> "The best way to understand a system is to build one yourself. Let's put everything we've learned together."

---

## From Theory to Practice

We've spent 18 chapters discussing GraphQL concepts, trade-offs, and design decisions. Now it's time to build a real, working GraphQL server from scratch.

We'll build a **blog API** with:
- Users and authentication
- Posts with authors
- Comments with nested replies
- Tags and categories
- Full CRUD operations
- DataLoader for N+1 prevention
- Error handling
- Security measures

**By the end, you'll have a production-ready foundation.**

---

## Project Setup

### Step 1: Initialize Spring Boot Project

```bash
# Using Spring Initializr
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,h2,graphql \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.0 \
  -d groupId=com.example \
  -d artifactId=graphql-blog \
  -d name=GraphQLBlog \
  -d packageName=com.example.graphqlblog \
  -d javaVersion=17 \
  -o graphql-blog.zip

unzip graphql-blog.zip
cd graphql-blog
```

### Step 2: Add Dependencies

**pom.xml:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>graphql-blog</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>GraphQL Blog</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-graphql</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.graphql</groupId>
            <artifactId>spring-graphql-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Lombok (optional, but helpful) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Step 3: Project Structure

```
graphql-blog/
├── src/
│   ├── main/
│   │   ├── java/com/example/graphqlblog/
│   │   │   ├── GraphQLBlogApplication.java
│   │   │   ├── config/
│   │   │   │   ├── DataLoaderConfig.java
│   │   │   │   └── GraphQLConfig.java
│   │   │   ├── model/
│   │   │   │   ├── User.java
│   │   │   │   ├── Post.java
│   │   │   │   ├── Comment.java
│   │   │   │   └── Tag.java
│   │   │   ├── repository/
│   │   │   │   ├── UserRepository.java
│   │   │   │   ├── PostRepository.java
│   │   │   │   ├── CommentRepository.java
│   │   │   │   └── TagRepository.java
│   │   │   ├── resolver/
│   │   │   │   ├── QueryResolver.java
│   │   │   │   ├── MutationResolver.java
│   │   │   │   ├── UserResolver.java
│   │   │   │   ├── PostResolver.java
│   │   │   │   └── CommentResolver.java
│   │   │   ├── service/
│   │   │   │   ├── UserService.java
│   │   │   │   ├── PostService.java
│   │   │   │   └── CommentService.java
│   │   │   ├── dataloader/
│   │   │   │   ├── UserDataLoader.java
│   │   │   │   └── TagDataLoader.java
│   │   │   ├── input/
│   │   │   │   ├── CreatePostInput.java
│   │   │   │   ├── UpdatePostInput.java
│   │   │   │   └── CreateCommentInput.java
│   │   │   └── exception/
│   │   │       ├── ResourceNotFoundException.java
│   │   │       └── GraphQLExceptionHandler.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── schema.graphqls
│   │       └── data.sql
│   └── test/
│       └── java/com/example/graphqlblog/
└── pom.xml
```

---

## Defining the GraphQL Schema

**src/main/resources/graphql/schema.graphqls:**

```graphql
scalar DateTime

type Query {
  # Users
  user(id: ID!): User
  users: [User!]!

  # Posts
  post(id: ID!): Post
  posts(limit: Int, offset: Int): [Post!]!
  postsByAuthor(authorId: ID!): [Post!]!

  # Comments
  comment(id: ID!): Comment
  commentsByPost(postId: ID!): [Comment!]!

  # Tags
  tag(id: ID!): Tag
  tags: [Tag!]!
}

type Mutation {
  # User operations
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!

  # Post operations
  createPost(input: CreatePostInput!): CreatePostPayload!
  updatePost(input: UpdatePostInput!): UpdatePostPayload!
  deletePost(id: ID!): DeletePostPayload!
  publishPost(id: ID!): PublishPostPayload!

  # Comment operations
  createComment(input: CreateCommentInput!): CreateCommentPayload!
  deleteComment(id: ID!): DeleteCommentPayload!
}

# ========================================
# Types
# ========================================

type User {
  id: ID!
  username: String!
  email: String!
  name: String!
  bio: String
  avatarUrl: String
  createdAt: DateTime!

  # Relationships
  posts: [Post!]!
  comments: [Comment!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  excerpt: String
  status: PostStatus!
  publishedAt: DateTime
  createdAt: DateTime!
  updatedAt: DateTime!

  # Relationships
  author: User!
  comments: [Comment!]!
  tags: [Tag!]!

  # Computed fields
  commentCount: Int!
  isPublished: Boolean!
}

type Comment {
  id: ID!
  text: String!
  createdAt: DateTime!
  updatedAt: DateTime!

  # Relationships
  author: User!
  post: Post!
  parent: Comment
  replies: [Comment!]!

  # Computed fields
  replyCount: Int!
}

type Tag {
  id: ID!
  name: String!
  slug: String!

  # Relationships
  posts: [Post!]!

  # Computed fields
  postCount: Int!
}

# ========================================
# Enums
# ========================================

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

# ========================================
# Input Types
# ========================================

input CreateUserInput {
  username: String!
  email: String!
  name: String!
  bio: String
  password: String!
}

input UpdateUserInput {
  id: ID!
  username: String
  email: String
  name: String
  bio: String
  avatarUrl: String
}

input CreatePostInput {
  title: String!
  content: String!
  excerpt: String
  authorId: ID!
  tagIds: [ID!]
  status: PostStatus = DRAFT
}

input UpdatePostInput {
  id: ID!
  title: String
  content: String
  excerpt: String
  tagIds: [ID!]
  status: PostStatus
}

input CreateCommentInput {
  text: String!
  postId: ID!
  authorId: ID!
  parentId: ID
}

# ========================================
# Payload Types
# ========================================

type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UpdateUserPayload {
  user: User
  errors: [UserError!]!
}

type DeleteUserPayload {
  deletedUserId: ID
  success: Boolean!
  errors: [UserError!]!
}

type CreatePostPayload {
  post: Post
  errors: [UserError!]!
}

type UpdatePostPayload {
  post: Post
  errors: [UserError!]!
}

type DeletePostPayload {
  deletedPostId: ID
  success: Boolean!
  errors: [UserError!]!
}

type PublishPostPayload {
  post: Post
  errors: [UserError!]!
}

type CreateCommentPayload {
  comment: Comment
  errors: [UserError!]!
}

type DeleteCommentPayload {
  deletedCommentId: ID
  success: Boolean!
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String
}
```

---

## Data Models (JPA Entities)

### User Entity

**src/main/java/com/example/graphqlblog/model/User.java:**

```java
package com.example.graphqlblog.model;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String name;

    @Column(length = 500)
    private String bio;

    private String avatarUrl;

    @Column(nullable = false)
    private String passwordHash;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Post> posts = new ArrayList<>();

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    public User(String username, String email, String name, String passwordHash) {
        this.username = username;
        this.email = email;
        this.name = name;
        this.passwordHash = passwordHash;
    }
}
```

### Post Entity

**src/main/java/com/example/graphqlblog/model/Post.java:**

```java
package com.example.graphqlblog.model;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "posts")
@Data
@NoArgsConstructor
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, length = 10000)
    private String content;

    @Column(length = 500)
    private String excerpt;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PostStatus status = PostStatus.DRAFT;

    private LocalDateTime publishedAt;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    @ManyToMany
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private List<Tag> tags = new ArrayList<>();

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    public boolean isPublished() {
        return status == PostStatus.PUBLISHED;
    }

    public enum PostStatus {
        DRAFT, PUBLISHED, ARCHIVED
    }
}
```

### Comment Entity

**src/main/java/com/example/graphqlblog/model/Comment.java:**

```java
package com.example.graphqlblog.model;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "comments")
@Data
@NoArgsConstructor
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 2000)
    private String text;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Comment parent;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> replies = new ArrayList<>();

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### Tag Entity

**src/main/java/com/example/graphqlblog/model/Tag.java:**

```java
package com.example.graphqlblog.model;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "tags")
@Data
@NoArgsConstructor
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    @Column(nullable = false, unique = true)
    private String slug;

    @ManyToMany(mappedBy = "tags")
    private List<Post> posts = new ArrayList<>();

    public Tag(String name, String slug) {
        this.name = name;
        this.slug = slug;
    }
}
```

---

## Repositories

**src/main/java/com/example/graphqlblog/repository/UserRepository.java:**

```java
package com.example.graphqlblog.repository;

import com.example.graphqlblog.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.id IN :ids")
    List<User> findAllByIdIn(List<Long> ids);
}
```

**src/main/java/com/example/graphqlblog/repository/PostRepository.java:**

```java
package com.example.graphqlblog.repository;

import com.example.graphqlblog.model.Post;
import com.example.graphqlblog.model.Post.PostStatus;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByAuthorId(Long authorId);
    List<Post> findByStatus(PostStatus status);

    @Query("SELECT p FROM Post p ORDER BY p.createdAt DESC")
    List<Post> findAllOrdered(Pageable pageable);

    @Query("SELECT COUNT(c) FROM Comment c WHERE c.post.id = :postId")
    int countCommentsByPostId(Long postId);
}
```

**src/main/java/com/example/graphqlblog/repository/CommentRepository.java:**

```java
package com.example.graphqlblog.repository;

import com.example.graphqlblog.model.Comment;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface CommentRepository extends JpaRepository<Comment, Long> {
    List<Comment> findByPostId(Long postId);
    List<Comment> findByAuthorId(Long authorId);
    List<Comment> findByParentId(Long parentId);

    @Query("SELECT c FROM Comment c WHERE c.post.id = :postId AND c.parent IS NULL ORDER BY c.createdAt DESC")
    List<Comment> findTopLevelCommentsByPostId(Long postId);

    @Query("SELECT COUNT(c) FROM Comment c WHERE c.parent.id = :parentId")
    int countRepliesByParentId(Long parentId);
}
```

**src/main/java/com/example/graphqlblog/repository/TagRepository.java:**

```java
package com.example.graphqlblog.repository;

import com.example.graphqlblog.model.Tag;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;
import java.util.Optional;

public interface TagRepository extends JpaRepository<Tag, Long> {
    Optional<Tag> findBySlug(String slug);

    @Query("SELECT t FROM Tag t WHERE t.id IN :ids")
    List<Tag> findAllByIdIn(List<Long> ids);
}
```

---

## Query Resolvers

**src/main/java/com/example/graphqlblog/resolver/QueryResolver.java:**

```java
package com.example.graphqlblog.resolver;

import com.example.graphqlblog.model.*;
import com.example.graphqlblog.repository.*;
import com.example.graphqlblog.exception.ResourceNotFoundException;
import org.springframework.data.domain.PageRequest;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.QueryMapping;
import org.springframework.stereotype.Controller;
import java.util.List;

@Controller
public class QueryResolver {

    private final UserRepository userRepository;
    private final PostRepository postRepository;
    private final CommentRepository commentRepository;
    private final TagRepository tagRepository;

    public QueryResolver(
            UserRepository userRepository,
            PostRepository postRepository,
            CommentRepository commentRepository,
            TagRepository tagRepository
    ) {
        this.userRepository = userRepository;
        this.postRepository = postRepository;
        this.commentRepository = commentRepository;
        this.tagRepository = tagRepository;
    }

    // ========================================
    // User Queries
    // ========================================

    @QueryMapping
    public User user(@Argument Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    @QueryMapping
    public List<User> users() {
        return userRepository.findAll();
    }

    // ========================================
    // Post Queries
    // ========================================

    @QueryMapping
    public Post post(@Argument Long id) {
        return postRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Post", id));
    }

    @QueryMapping
    public List<Post> posts(
            @Argument Integer limit,
            @Argument Integer offset
    ) {
        int pageSize = limit != null ? limit : 20;
        int page = offset != null ? offset / pageSize : 0;

        return postRepository.findAllOrdered(PageRequest.of(page, pageSize));
    }

    @QueryMapping
    public List<Post> postsByAuthor(@Argument Long authorId) {
        return postRepository.findByAuthorId(authorId);
    }

    // ========================================
    // Comment Queries
    // ========================================

    @QueryMapping
    public Comment comment(@Argument Long id) {
        return commentRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Comment", id));
    }

    @QueryMapping
    public List<Comment> commentsByPost(@Argument Long postId) {
        return commentRepository.findTopLevelCommentsByPostId(postId);
    }

    // ========================================
    // Tag Queries
    // ========================================

    @QueryMapping
    public Tag tag(@Argument Long id) {
        return tagRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Tag", id));
    }

    @QueryMapping
    public List<Tag> tags() {
        return tagRepository.findAll();
    }
}
```

---

## Field Resolvers

These resolvers handle relationships and computed fields.

**src/main/java/com/example/graphqlblog/resolver/PostResolver.java:**

```java
package com.example.graphqlblog.resolver;

import com.example.graphqlblog.model.Comment;
import com.example.graphqlblog.model.Post;
import com.example.graphqlblog.model.Tag;
import com.example.graphqlblog.model.User;
import com.example.graphqlblog.repository.CommentRepository;
import graphql.schema.DataFetchingEnvironment;
import org.dataloader.DataLoader;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;
import java.util.List;
import java.util.concurrent.CompletableFuture;

@Controller
public class PostResolver {

    private final CommentRepository commentRepository;

    public PostResolver(CommentRepository commentRepository) {
        this.commentRepository = commentRepository;
    }

    @SchemaMapping(typeName = "Post", field = "author")
    public CompletableFuture<User> author(Post post, DataFetchingEnvironment env) {
        DataLoader<Long, User> loader = env.getDataLoader("userLoader");
        return loader.load(post.getAuthor().getId());
    }

    @SchemaMapping(typeName = "Post", field = "comments")
    public List<Comment> comments(Post post) {
        return commentRepository.findTopLevelCommentsByPostId(post.getId());
    }

    @SchemaMapping(typeName = "Post", field = "tags")
    public List<Tag> tags(Post post) {
        return post.getTags();
    }

    @SchemaMapping(typeName = "Post", field = "commentCount")
    public int commentCount(Post post) {
        return commentRepository.findByPostId(post.getId()).size();
    }

    @SchemaMapping(typeName = "Post", field = "isPublished")
    public boolean isPublished(Post post) {
        return post.isPublished();
    }
}
```

**src/main/java/com/example/graphqlblog/resolver/CommentResolver.java:**

```java
package com.example.graphqlblog.resolver;

import com.example.graphqlblog.model.Comment;
import com.example.graphqlblog.model.Post;
import com.example.graphqlblog.model.User;
import com.example.graphqlblog.repository.CommentRepository;
import graphql.schema.DataFetchingEnvironment;
import org.dataloader.DataLoader;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;
import java.util.List;
import java.util.concurrent.CompletableFuture;

@Controller
public class CommentResolver {

    private final CommentRepository commentRepository;

    public CommentResolver(CommentRepository commentRepository) {
        this.commentRepository = commentRepository;
    }

    @SchemaMapping(typeName = "Comment", field = "author")
    public CompletableFuture<User> author(Comment comment, DataFetchingEnvironment env) {
        DataLoader<Long, User> loader = env.getDataLoader("userLoader");
        return loader.load(comment.getAuthor().getId());
    }

    @SchemaMapping(typeName = "Comment", field = "post")
    public Post post(Comment comment) {
        return comment.getPost();
    }

    @SchemaMapping(typeName = "Comment", field = "parent")
    public Comment parent(Comment comment) {
        return comment.getParent();
    }

    @SchemaMapping(typeName = "Comment", field = "replies")
    public List<Comment> replies(Comment comment) {
        return commentRepository.findByParentId(comment.getId());
    }

    @SchemaMapping(typeName = "Comment", field = "replyCount")
    public int replyCount(Comment comment) {
        return commentRepository.countRepliesByParentId(comment.getId());
    }
}
```

**src/main/java/com/example/graphqlblog/resolver/UserResolver.java:**

```java
package com.example.graphqlblog.resolver;

import com.example.graphqlblog.model.Comment;
import com.example.graphqlblog.model.Post;
import com.example.graphqlblog.model.User;
import com.example.graphqlblog.repository.CommentRepository;
import com.example.graphqlblog.repository.PostRepository;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;
import java.util.List;

@Controller
public class UserResolver {

    private final PostRepository postRepository;
    private final CommentRepository commentRepository;

    public UserResolver(
            PostRepository postRepository,
            CommentRepository commentRepository
    ) {
        this.postRepository = postRepository;
        this.commentRepository = commentRepository;
    }

    @SchemaMapping(typeName = "User", field = "posts")
    public List<Post> posts(User user) {
        return postRepository.findByAuthorId(user.getId());
    }

    @SchemaMapping(typeName = "User", field = "comments")
    public List<Comment> comments(User user) {
        return commentRepository.findByAuthorId(user.getId());
    }
}
```

---

## Mutation Resolvers

**src/main/java/com/example/graphqlblog/resolver/MutationResolver.java:**

```java
package com.example.graphqlblog.resolver;

import com.example.graphqlblog.input.*;
import com.example.graphqlblog.payload.*;
import com.example.graphqlblog.service.*;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.MutationMapping;
import org.springframework.stereotype.Controller;
import org.springframework.transaction.annotation.Transactional;

@Controller
public class MutationResolver {

    private final UserService userService;
    private final PostService postService;
    private final CommentService commentService;

    public MutationResolver(
            UserService userService,
            PostService postService,
            CommentService commentService
    ) {
        this.userService = userService;
        this.postService = postService;
        this.commentService = commentService;
    }

    // ========================================
    // User Mutations
    // ========================================

    @MutationMapping
    @Transactional
    public CreateUserPayload createUser(@Argument CreateUserInput input) {
        return userService.createUser(input);
    }

    @MutationMapping
    @Transactional
    public UpdateUserPayload updateUser(@Argument UpdateUserInput input) {
        return userService.updateUser(input);
    }

    @MutationMapping
    @Transactional
    public DeleteUserPayload deleteUser(@Argument Long id) {
        return userService.deleteUser(id);
    }

    // ========================================
    // Post Mutations
    // ========================================

    @MutationMapping
    @Transactional
    public CreatePostPayload createPost(@Argument CreatePostInput input) {
        return postService.createPost(input);
    }

    @MutationMapping
    @Transactional
    public UpdatePostPayload updatePost(@Argument UpdatePostInput input) {
        return postService.updatePost(input);
    }

    @MutationMapping
    @Transactional
    public DeletePostPayload deletePost(@Argument Long id) {
        return postService.deletePost(id);
    }

    @MutationMapping
    @Transactional
    public PublishPostPayload publishPost(@Argument Long id) {
        return postService.publishPost(id);
    }

    // ========================================
    // Comment Mutations
    // ========================================

    @MutationMapping
    @Transactional
    public CreateCommentPayload createComment(@Argument CreateCommentInput input) {
        return commentService.createComment(input);
    }

    @MutationMapping
    @Transactional
    public DeleteCommentPayload deleteComment(@Argument Long id) {
        return commentService.deleteComment(id);
    }
}
```

---

## Service Layer

I'll show one complete service as an example:

**src/main/java/com/example/graphqlblog/service/PostService.java:**

```java
package com.example.graphqlblog.service;

import com.example.graphqlblog.input.CreatePostInput;
import com.example.graphqlblog.input.UpdatePostInput;
import com.example.graphqlblog.model.Post;
import com.example.graphqlblog.model.Post.PostStatus;
import com.example.graphqlblog.model.Tag;
import com.example.graphqlblog.model.User;
import com.example.graphqlblog.payload.*;
import com.example.graphqlblog.repository.PostRepository;
import com.example.graphqlblog.repository.TagRepository;
import com.example.graphqlblog.repository.UserRepository;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Service
public class PostService {

    private final PostRepository postRepository;
    private final UserRepository userRepository;
    private final TagRepository tagRepository;

    public PostService(
            PostRepository postRepository,
            UserRepository userRepository,
            TagRepository tagRepository
    ) {
        this.postRepository = postRepository;
        this.userRepository = userRepository;
        this.tagRepository = tagRepository;
    }

    public CreatePostPayload createPost(CreatePostInput input) {
        // Validate author exists
        User author = userRepository.findById(input.getAuthorId())
            .orElse(null);

        if (author == null) {
            return new CreatePostPayload(
                null,
                List.of(new UserError("authorId", "Author not found", "NOT_FOUND"))
            );
        }

        // Validate input
        List<UserError> errors = validatePostInput(input.getTitle(), input.getContent());
        if (!errors.isEmpty()) {
            return new CreatePostPayload(null, errors);
        }

        // Create post
        Post post = new Post();
        post.setTitle(input.getTitle());
        post.setContent(input.getContent());
        post.setExcerpt(input.getExcerpt());
        post.setAuthor(author);
        post.setStatus(input.getStatus() != null ? input.getStatus() : PostStatus.DRAFT);

        // Add tags
        if (input.getTagIds() != null && !input.getTagIds().isEmpty()) {
            List<Tag> tags = tagRepository.findAllByIdIn(input.getTagIds());
            post.setTags(tags);
        }

        post = postRepository.save(post);

        return new CreatePostPayload(post, Collections.emptyList());
    }

    public UpdatePostPayload updatePost(UpdatePostInput input) {
        Post post = postRepository.findById(input.getId())
            .orElse(null);

        if (post == null) {
            return new UpdatePostPayload(
                null,
                List.of(new UserError("id", "Post not found", "NOT_FOUND"))
            );
        }

        // Apply updates
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
            post.setStatus(input.getStatus());
        }

        if (input.getTagIds() != null) {
            List<Tag> tags = tagRepository.findAllByIdIn(input.getTagIds());
            post.setTags(tags);
        }

        post = postRepository.save(post);

        return new UpdatePostPayload(post, Collections.emptyList());
    }

    public DeletePostPayload deletePost(Long id) {
        if (!postRepository.existsById(id)) {
            return new DeletePostPayload(
                null,
                false,
                List.of(new UserError("id", "Post not found", "NOT_FOUND"))
            );
        }

        postRepository.deleteById(id);

        return new DeletePostPayload(id, true, Collections.emptyList());
    }

    public PublishPostPayload publishPost(Long id) {
        Post post = postRepository.findById(id)
            .orElse(null);

        if (post == null) {
            return new PublishPostPayload(
                null,
                List.of(new UserError("id", "Post not found", "NOT_FOUND"))
            );
        }

        if (post.getStatus() == PostStatus.PUBLISHED) {
            return new PublishPostPayload(
                null,
                List.of(new UserError("id", "Post is already published", "INVALID_STATE"))
            );
        }

        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(LocalDateTime.now());
        post = postRepository.save(post);

        return new PublishPostPayload(post, Collections.emptyList());
    }

    private List<UserError> validatePostInput(String title, String content) {
        List<UserError> errors = new ArrayList<>();

        if (title == null || title.isBlank()) {
            errors.add(new UserError("title", "Title is required", "VALIDATION_ERROR"));
        } else if (title.length() > 200) {
            errors.add(new UserError("title", "Title must be less than 200 characters", "VALIDATION_ERROR"));
        }

        if (content == null || content.isBlank()) {
            errors.add(new UserError("content", "Content is required", "VALIDATION_ERROR"));
        } else if (content.length() > 50000) {
            errors.add(new UserError("content", "Content is too long", "VALIDATION_ERROR"));
        }

        return errors;
    }
}
```

---

## DataLoader Configuration

Prevent N+1 queries with DataLoader:

**src/main/java/com/example/graphqlblog/config/DataLoaderConfig.java:**

```java
package com.example.graphqlblog.config;

import com.example.graphqlblog.model.User;
import com.example.graphqlblog.repository.UserRepository;
import graphql.GraphQLContext;
import org.dataloader.DataLoader;
import org.dataloader.DataLoaderRegistry;
import org.springframework.context.annotation.Configuration;
import org.springframework.graphql.execution.DataLoaderRegistrar;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

@Configuration
public class DataLoaderConfig implements DataLoaderRegistrar {

    private final UserRepository userRepository;

    public DataLoaderConfig(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public void registerDataLoaders(
            DataLoaderRegistry registry,
            GraphQLContext context
    ) {
        // User DataLoader
        DataLoader<Long, User> userLoader = DataLoader.newDataLoader(
            (List<Long> userIds) -> CompletableFuture.supplyAsync(() -> {
                List<User> users = userRepository.findAllByIdIn(userIds);

                // Create map for efficient lookup
                Map<Long, User> userMap = users.stream()
                    .collect(Collectors.toMap(User::getId, user -> user));

                // Return users in the same order as requested IDs
                return userIds.stream()
                    .map(userMap::get)
                    .collect(Collectors.toList());
            })
        );

        registry.register("userLoader", userLoader);
    }
}
```

---

## Configuration

**src/main/resources/application.yml:**

```yaml
spring:
  application:
    name: GraphQL Blog

  datasource:
    url: jdbc:h2:mem:graphql_blog
    driver-class-name: org.h2.Driver
    username: sa
    password:

  h2:
    console:
      enabled: true
      path: /h2-console

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  graphql:
    path: /graphql
    graphiql:
      enabled: true
      path: /graphiql
    schema:
      printer:
        enabled: true

server:
  port: 8080

logging:
  level:
    com.example.graphqlblog: DEBUG
    org.springframework.graphql: DEBUG
```

---

## Sample Data

**src/main/resources/data.sql:**

```sql
-- Insert users
INSERT INTO users (id, username, email, name, bio, password_hash, created_at) VALUES
(1, 'alice', 'alice@example.com', 'Alice Johnson', 'Backend developer passionate about GraphQL', '$2a$10$...', CURRENT_TIMESTAMP),
(2, 'bob', 'bob@example.com', 'Bob Smith', 'Frontend enthusiast and UX designer', '$2a$10$...', CURRENT_TIMESTAMP),
(3, 'charlie', 'charlie@example.com', 'Charlie Brown', 'DevOps engineer', '$2a$10$...', CURRENT_TIMESTAMP);

-- Insert tags
INSERT INTO tags (id, name, slug) VALUES
(1, 'GraphQL', 'graphql'),
(2, 'Java', 'java'),
(3, 'Spring Boot', 'spring-boot'),
(4, 'API Design', 'api-design');

-- Insert posts
INSERT INTO posts (id, title, content, excerpt, status, published_at, created_at, updated_at, author_id) VALUES
(1, 'Introduction to GraphQL', 'GraphQL is a query language for APIs...', 'Learn the basics of GraphQL', 'PUBLISHED', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 1),
(2, 'Building GraphQL Servers in Java', 'In this post we explore...', NULL, 'PUBLISHED', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 1),
(3, 'Draft Post', 'Work in progress...', NULL, 'DRAFT', NULL, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 2);

-- Insert post_tags
INSERT INTO post_tags (post_id, tag_id) VALUES
(1, 1), (1, 4),
(2, 1), (2, 2), (2, 3);

-- Insert comments
INSERT INTO comments (id, text, created_at, updated_at, author_id, post_id, parent_id) VALUES
(1, 'Great introduction! Very helpful.', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 2, 1, NULL),
(2, 'Thanks! Glad it was useful.', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 1, 1, 1),
(3, 'Can you elaborate on resolvers?', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 3, 1, NULL),
(4, 'Sure, I''ll write a follow-up post on that.', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, 1, 1, 3);
```

---

## Running the Server

```bash
# Build
mvn clean package

# Run
mvn spring-boot:run

# Or run the JAR
java -jar target/graphql-blog-0.0.1-SNAPSHOT.jar
```

**Access:**
- GraphQL endpoint: `http://localhost:8080/graphql`
- GraphiQL UI: `http://localhost:8080/graphiql`
- H2 Console: `http://localhost:8080/h2-console`

---

## Example Queries

### Query 1: Fetch User with Posts

```graphql
query {
  user(id: 1) {
    name
    email
    bio
    posts {
      title
      status
      commentCount
    }
  }
}
```

### Query 2: Fetch Post with Author and Comments

```graphql
query {
  post(id: 1) {
    title
    content
    author {
      name
    }
    comments {
      text
      author {
        name
      }
      replies {
        text
        author {
          name
        }
      }
    }
    tags {
      name
    }
  }
}
```

### Mutation 1: Create Post

```graphql
mutation {
  createPost(input: {
    title: "My New Post"
    content: "This is the content of my post..."
    excerpt: "A brief excerpt"
    authorId: 1
    tagIds: [1, 2]
    status: DRAFT
  }) {
    post {
      id
      title
      status
    }
    errors {
      field
      message
    }
  }
}
```

### Mutation 2: Publish Post

```graphql
mutation {
  publishPost(id: 3) {
    post {
      id
      status
      publishedAt
    }
    errors {
      message
    }
  }
}
```

---

## Testing

**src/test/java/com/example/graphqlblog/GraphQLBlogApplicationTests.java:**

```java
package com.example.graphqlblog;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.graphql.GraphQlTest;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.graphql.test.tester.GraphQlTester;

@SpringBootTest
@GraphQlTest
class GraphQLBlogApplicationTests {

    @Autowired
    private GraphQlTester graphQlTester;

    @Test
    void testUserQuery() {
        graphQlTester
            .document("""
                query {
                    user(id: 1) {
                        name
                        email
                    }
                }
                """)
            .execute()
            .path("user.name")
            .entity(String.class)
            .isEqualTo("Alice Johnson");
    }

    @Test
    void testCreatePost() {
        graphQlTester
            .document("""
                mutation {
                    createPost(input: {
                        title: "Test Post"
                        content: "Test content"
                        authorId: 1
                    }) {
                        post {
                            title
                        }
                        errors {
                            message
                        }
                    }
                }
                """)
            .execute()
            .path("createPost.post.title")
            .entity(String.class)
            .isEqualTo("Test Post");
    }
}
```

---

## Key Takeaways

1. **Spring Boot GraphQL** makes setup straightforward
2. **Schema-first approach** - Define schema in .graphqls files
3. **@Controller + @QueryMapping/@MutationMapping** - Clean resolver definitions
4. **@SchemaMapping** - Handle field resolvers and relationships
5. **DataLoader** - Prevent N+1 queries with batching
6. **Service layer** - Business logic separate from resolvers
7. **Input/Payload pattern** - Clean mutation interfaces
8. **JPA entities** - Standard persistence layer
9. **Validation in services** - Return user-friendly errors
10. **Testing with GraphQlTester** - Easy integration tests

You now have a complete, working GraphQL server that demonstrates all the concepts from previous chapters!

---

**Next:** [Chapter 20: Client Implementation Patterns →](20-client-patterns.md)
**Previous:** [← Chapter 18: Federation - Distributing the Graph](18-federation.md)
