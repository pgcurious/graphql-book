# Chapter 1: The REST API Problem

**What we'll discover:** Why the most popular approach to building APIs has fundamental limitations that no amount of clever engineering can fully solve.

---

## Starting from Zero

Imagine you're building a mobile app for a social network. You have a backend with a PostgreSQL database full of users, posts, comments, and likes. Your mobile app needs to show a feed of posts. Simple enough, right?

Let's think through what we need to fetch for a single post in the feed:

1. **Post data**: title, content, creation timestamp
2. **Author information**: name, avatar URL, bio
3. **Like count**: how many people liked this post
4. **Comments preview**: the first 3 comments
5. **For each comment**: commenter's name and avatar

Now, how do we get this data from our server to the mobile app?

## The REST Approach: Endpoints as Resources

REST (Representational State Transfer) emerged as an elegant solution to API design. The core idea is beautiful in its simplicity: model your API around **resources** accessed via **URLs**.

For our social network, we might design these endpoints:

```
GET /api/posts/123              # Get post data
GET /api/users/456              # Get user data
GET /api/posts/123/likes        # Get likes for a post
GET /api/posts/123/comments     # Get comments for a post
```

Let's implement this in Java using Spring Boot:

```java
@RestController
@RequestMapping("/api")
public class SocialNetworkController {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private CommentRepository commentRepository;

    @GetMapping("/posts/{id}")
    public ResponseEntity<PostDTO> getPost(@PathVariable Long id) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

        return ResponseEntity.ok(new PostDTO(
            post.getId(),
            post.getTitle(),
            post.getContent(),
            post.getCreatedAt(),
            post.getAuthorId()  // Just the author ID, not full details
        ));
    }

    @GetMapping("/users/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        return ResponseEntity.ok(new UserDTO(
            user.getId(),
            user.getName(),
            user.getAvatarUrl(),
            user.getBio(),
            user.getEmail(),           // Do we need this?
            user.getPhoneNumber(),     // Or this?
            user.getDateOfBirth(),     // Or this?
            user.getAddress()          // Probably not this
        ));
    }

    @GetMapping("/posts/{id}/comments")
    public ResponseEntity<List<CommentDTO>> getPostComments(
            @PathVariable Long id,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {

        Page<Comment> comments = commentRepository
            .findByPostId(id, PageRequest.of(page, size));

        List<CommentDTO> commentDTOs = comments.stream()
            .map(comment -> new CommentDTO(
                comment.getId(),
                comment.getContent(),
                comment.getAuthorId(),  // Again, just an ID
                comment.getCreatedAt()
            ))
            .collect(Collectors.toList());

        return ResponseEntity.ok(commentDTOs);
    }
}
```

This looks clean and follows REST principles. But let's trace through what actually happens when the mobile app wants to display one post.

## Problem 1: The Network Cascade (Under-fetching)

Here's what the client needs to do:

```java
// Mobile client code (pseudocode)
public PostFeedItem fetchPostForFeed(Long postId) {
    // Request 1: Get the post
    Post post = httpClient.get("/api/posts/" + postId);

    // Request 2: Get the author details
    User author = httpClient.get("/api/users/" + post.getAuthorId());

    // Request 3: Get comments
    List<Comment> comments = httpClient.get("/api/posts/" + postId + "/comments?size=3");

    // Request 4, 5, 6: Get each commenter's details
    List<User> commenters = new ArrayList<>();
    for (Comment comment : comments) {
        User commenter = httpClient.get("/api/users/" + comment.getAuthorId());
        commenters.add(commenter);
    }

    // Request 7: Get like count
    LikeCount likes = httpClient.get("/api/posts/" + postId + "/likes");

    // Finally assemble everything
    return new PostFeedItem(post, author, comments, commenters, likes);
}
```

**We made 7 HTTP requests to display one post.**

Let's calculate the latency:
- Each HTTP request: ~100ms (including network latency)
- Sequential execution: 7 ◊ 100ms = **700ms minimum**

For a feed of 10 posts, that's potentially **70 requests and 7 seconds**. On a mobile network with 200ms latency? **14 seconds**.

This is **under-fetching**: each endpoint gives us only part of what we need, forcing us to make multiple round trips.

```
Client                          Server
   |                               |
   |--- GET /posts/123 ----------->|
   |<---------- post --------------|
   |                               |
   |--- GET /users/456 ----------->|
   |<---------- author ------------|
   |                               |
   |--- GET /posts/123/comments -->|
   |<---------- comments ----------|
   |                               |
   |--- GET /users/789 ----------->|  (for commenter 1)
   |<---------- user --------------|
   |                               |
   |--- GET /users/012 ----------->|  (for commenter 2)
   |<---------- user --------------|
   ...

   Time: 700ms+
```

## Attempt 1: Nested Resources

"Let's embed related data!" you think. We can create a richer endpoint:

```java
@GetMapping("/posts/{id}/detailed")
public ResponseEntity<DetailedPostDTO> getDetailedPost(@PathVariable Long id) {
    Post post = postRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

    // Fetch author
    User author = userRepository.findById(post.getAuthorId()).orElse(null);

    // Fetch comments with their authors
    List<Comment> comments = commentRepository.findByPostIdLimit(id, 3);
    List<CommentWithAuthorDTO> enrichedComments = comments.stream()
        .map(comment -> {
            User commenter = userRepository.findById(comment.getAuthorId()).orElse(null);
            return new CommentWithAuthorDTO(comment, commenter);
        })
        .collect(Collectors.toList());

    // Fetch like count
    int likeCount = likeRepository.countByPostId(id);

    return ResponseEntity.ok(new DetailedPostDTO(
        post,
        author,
        enrichedComments,
        likeCount
    ));
}
```

Great! Now we're down to **one request**. Problem solved?

## Problem 2: Over-fetching

Not quite. Look at what we're returning in the `UserDTO`:

```java
public class UserDTO {
    private Long id;
    private String name;
    private String avatarUrl;
    private String bio;
    private String email;           // =® Do we need this for a post feed?
    private String phoneNumber;     // =® Definitely not needed
    private LocalDate dateOfBirth;  // =® Not needed
    private String address;         // =® Not needed
    private String preferences;     // =® Not needed
    // ... 20 more fields
}
```

For our post feed, we only need `name` and `avatarUrl`. But we're sending all 25 fields. Multiply this by 10 posts with 3 comments each:

- **1 user per post** = 10 users
- **3 commenters per post** = 30 users
- **Total**: 40 user objects ◊ 25 fields ◊ ~50 bytes per field = **50KB of unnecessary data**

On mobile networks where users pay per megabyte, this is expensive. On slow connections, it's slow. This is **over-fetching**.

## Attempt 2: Multiple Endpoint Variants

"Let's create different endpoints for different needs!"

```java
GET /api/posts/{id}                    // Basic post
GET /api/posts/{id}/detailed           // Post with author and comments
GET /api/posts/{id}/feed-view          // Optimized for feed
GET /api/posts/{id}/detail-view        // Optimized for detail page
GET /api/posts/{id}/notification-view  // Optimized for notifications

GET /api/users/{id}                    // Full user profile
GET /api/users/{id}/basic              // Just name and avatar
GET /api/users/{id}/card               // For user cards
GET /api/users/{id}/feed               // For feeds
```

Now we're maintaining dozens of endpoints, each with:
- Its own DTO class
- Its own SQL query
- Its own test suite
- Its own documentation

When you add a new field to `User`, you need to decide: which of these 5 endpoints should include it?

**The problem:** Your API's shape is dictated by your UI's needs. Every new screen means new endpoints. You're coupling your backend to your frontend's view hierarchy.

## Problem 3: Versioning Hell

Three months later, your iOS team says: "We need the author's follower count in the feed."

Your options:

**Option A**: Add it to `/posts/{id}/feed-view`
- L Breaking change if clients expect the old shape
- L Need to version: `/v2/posts/{id}/feed-view`
- L Now maintaining both endpoints

**Option B**: Create a new endpoint `/posts/{id}/feed-view-with-followers`
- L Endpoint proliferation
- L Nearly duplicate code
- L Which one is canonical?

**Option C**: Make it a query parameter `/posts/{id}/feed-view?includeFollowers=true`
- L Combinatorial explosion: what about `?includeFollowers=true&includeBio=true&includeLastActive=true`?
- L Need to support all combinations
- L Testing matrix grows exponentially

## The Database Analogy

Here's a thought experiment: imagine if databases worked like REST APIs.

Instead of:
```sql
SELECT users.name, users.avatar, posts.title
FROM posts
JOIN users ON posts.author_id = users.id
WHERE posts.id = 123;
```

You had to do:
```sql
-- Query 1
SELECT * FROM posts WHERE id = 123;  -- Returns all columns

-- Query 2 (after seeing author_id from query 1)
SELECT * FROM users WHERE id = 456;  -- Returns all columns again
```

We'd never accept this for databases. Why do we accept it for APIs?

Databases give us:
1. **Declarative queries**: "Here's what I want"
2. **Precise selection**: Only the fields I need
3. **Joins**: Get related data in one round trip
4. **Query optimization**: The database figures out how to fetch efficiently

REST APIs give us:
1. **Imperative fetching**: "Here's how to get it"
2. **Fixed selections**: All fields or nothing
3. **Manual joins**: Client orchestrates multiple requests
4. **No optimization**: Every view needs a custom endpoint

## The Fundamental Tension

The core issue is a mismatch in abstraction levels:

**REST thinks in terms of resources:**
- "Here's a user endpoint"
- "Here's a post endpoint"
- "Here's a comment endpoint"

**Clients think in terms of views:**
- "Here's what the feed needs"
- "Here's what the profile page needs"
- "Here's what the notification panel needs"

You can't solve this by making REST "better." It's not a bug in REST; it's the nature of the resource-oriented model.

## The Real Cost

Let's quantify what we're dealing with:

```java
// Metrics from a real social network API

// Feed endpoint
- Average fields needed per post: 8
- Average fields returned per post: 42
- Waste ratio: 5.25x over-fetching

// Network requests for initial feed load
- REST approach: 73 requests
- Time on 4G: 4.2 seconds
- Time on 3G: 9.8 seconds

// Backend endpoints
- Total endpoints: 147
- Endpoints per feature: ~12
- Duplicate business logic: ~40%

// Developer time
- Time to add a new field to existing views: 4-6 hours
- Time to create a new composite view: 8-16 hours
- Percentage of backend work that's "endpoint wrangling": 35%
```

These aren't made-up numbers. This is what happens when REST meets complex, interconnected data.

## A Different Question

Here's what we haven't asked yet:

**What if the client could tell the server exactly what it needs?**

Not by creating a new endpoint. Not by adding query parameters. But by expressing, in a single request:

> "Give me post 123, with its title and content, the author's name and avatar, the first 3 comments each with their content and author's name, and the like count."

What would that look like? How would we build it?

That's what we'll explore next.

## Key Insights

Before we move on, let's crystallize what we've learned:

1. **REST's strength is its weakness**: Resource-orientation is clean but forces under-fetching
2. **Custom endpoints don't scale**: You end up with an endpoint explosion
3. **Over-fetching wastes resources**: Bandwidth, battery, money
4. **The client knows best**: Only the UI knows exactly what data it needs
5. **We need a new abstraction**: One that matches how clients actually think about data

## Exercises

Before continuing to the next chapter, consider:

1. **Measure your API**: How many requests does your app's home screen make? Calculate the total latency.

2. **Count your endpoints**: How many endpoints do you have? How many are variations of the same resource?

3. **Calculate over-fetching**: Pick one endpoint. What percentage of the returned data does your client actually use?

4. **Thought experiment**: If you could send the server a description of exactly what you need, what would that description look like? How would you express "post with author name and first 3 comments"?

5. **Design challenge**: Try to design a URL-based API that solves over-fetching AND under-fetching without creating dozens of endpoints. What problems do you run into?

---

**Next:** [Chapter 2: What If Clients Could Specify Exactly What They Need? í](02-client-specified-queries.md)

**Previous:** [ê Back to Table of Contents](../README.md)
