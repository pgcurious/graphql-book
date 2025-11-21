# Chapter 16: Subscriptions - Real-time Graphs

> "Queries ask for data now. Mutations change data now. Subscriptions say: tell me when things change."

---

## The Real-Time Data Problem

Imagine you're building a chat application with GraphQL. A user sends a message. How do other users see it?

### The Naive Approach: Polling

```graphql
# Client polls every 2 seconds
query {
  messages(chatId: "123") {
    id
    text
    author { name }
    timestamp
  }
}
```

**Problems with polling:**
- Wasteful: 99% of requests return "no new data"
- Latency: Up to 2 seconds delay before seeing new messages
- Server load: Thousands of unnecessary requests
- Battery drain on mobile devices
- Not truly "real-time"

We could poll faster (every 500ms), but that makes all these problems worse.

### What We Really Want

```graphql
subscription {
  messageReceived(chatId: "123") {
    id
    text
    author { name }
    timestamp
  }
}
```

The server should **push** new messages to the client when they arrive. No polling. No waste. True real-time updates.

This is what GraphQL subscriptions provide.

---

## Subscriptions: The Third Operation Type

GraphQL has three root operation types:

```graphql
type Query {
  # Read operations - execute once, return data
  messages(chatId: ID!): [Message!]!
}

type Mutation {
  # Write operations - execute once, return result
  sendMessage(chatId: ID!, text: String!): Message!
}

type Subscription {
  # Stream operations - stay open, push updates
  messageReceived(chatId: ID!): Message!
}
```

### Key Differences

| Aspect | Query/Mutation | Subscription |
|--------|---------------|--------------|
| **Connection** | Request/response | Long-lived stream |
| **Result** | Single response | Multiple events over time |
| **Protocol** | HTTP | WebSocket (typically) |
| **Lifecycle** | One-shot | Continuous until unsubscribed |

---

## How Subscriptions Work: The Mental Model

Think of subscriptions as a **filtered event stream**:

```
[Server] ---> [Filter: chatId=123] ---> [Client]
   |
   |-- messageReceived(chatId: "123") ✓ Send to client
   |-- messageReceived(chatId: "456") ✗ Filtered out
   |-- messageReceived(chatId: "123") ✓ Send to client
   |-- userTyping(chatId: "123")      ✗ Different event
```

**The flow:**

1. **Client subscribes**: "Tell me about messages in chat 123"
2. **Server registers**: Adds client to the list for that event type
3. **Events happen**: Messages get sent, users type, etc.
4. **Server filters**: Only matching events go to this client
5. **Client unsubscribes**: Remove from the list

---

## The Pub/Sub Pattern

Subscriptions are built on the classic **publish-subscribe** pattern:

```
Publishers                   Subscribers
-----------                 -------------
Mutation resolver     -->   Active subscription clients
REST endpoint        -->   WebSocket connections
Background job       -->   Filtered by topic/arguments
External webhook     -->
```

### Simple In-Memory Pub/Sub

```java
public class EventPublisher {
    // Topic -> List of subscribers
    private Map<String, List<Subscriber>> subscribers = new ConcurrentHashMap<>();

    public void subscribe(String topic, Subscriber subscriber) {
        subscribers.computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>())
                   .add(subscriber);
    }

    public void publish(String topic, Object event) {
        List<Subscriber> topicSubscribers = subscribers.get(topic);
        if (topicSubscribers != null) {
            topicSubscribers.forEach(sub -> sub.onEvent(event));
        }
    }

    public void unsubscribe(String topic, Subscriber subscriber) {
        List<Subscriber> topicSubscribers = subscribers.get(topic);
        if (topicSubscribers != null) {
            topicSubscribers.remove(subscriber);
        }
    }
}
```

**Topics** are typically constructed from:
- Event type: `"messageReceived"`
- Arguments: `"chatId:123"`
- Combined: `"messageReceived:chatId:123"`

---

## Subscription Schema Design

### Basic Subscription

```graphql
type Subscription {
  messageReceived(chatId: ID!): Message!
}

type Message {
  id: ID!
  text: String!
  author: User!
  timestamp: DateTime!
}
```

### With Filters

```graphql
type Subscription {
  # All messages in a chat
  messageReceived(chatId: ID!): Message!

  # Messages from specific user
  messageFromUser(userId: ID!): Message!

  # Typing indicators
  userTyping(chatId: ID!): TypingEvent!

  # Presence updates
  userStatusChanged(userId: ID!): UserStatus!
}

type TypingEvent {
  user: User!
  chatId: ID!
  isTyping: Boolean!
}

enum UserStatus {
  ONLINE
  AWAY
  OFFLINE
}
```

### Subscription Naming Conventions

Good patterns:
- Past tense: `messageReceived`, `postCreated`, `commentAdded`
- Describes the event, not the action
- Clear what data you'll receive

Avoid:
- Imperative: `receiveMessage` (sounds like a command)
- Unclear: `messageUpdate` (too vague - created? edited? deleted?)

---

## WebSocket Protocol

Subscriptions typically use **WebSockets** because:
- **Full-duplex**: Server can push to client
- **Long-lived**: Connection stays open
- **Efficient**: No HTTP overhead per message
- **Standardized**: Works in browsers and mobile apps

### The graphql-ws Protocol

The most common protocol is `graphql-ws` (newer) or `graphql-transport-ws`:

```
Client                          Server
  |                               |
  |-- connection_init ----------->|
  |<---------- connection_ack ----|
  |                               |
  |-- subscribe (id: 1) --------->|
  |   { subscription { ... } }    |
  |                               |
  |<---------- next (id: 1) ------|  Event 1
  |   { data: { ... } }           |
  |                               |
  |<---------- next (id: 1) ------|  Event 2
  |   { data: { ... } }           |
  |                               |
  |-- complete (id: 1) ---------> |  Client unsubscribes
  |                               |
  |-- ping ---------------------->|
  |<----------------------- pong --|
  |                               |
```

### Message Types

```json
// Client -> Server: Initialize connection
{
  "type": "connection_init",
  "payload": {
    "authToken": "..."
  }
}

// Server -> Client: Acknowledge
{
  "type": "connection_ack"
}

// Client -> Server: Start subscription
{
  "id": "1",
  "type": "subscribe",
  "payload": {
    "query": "subscription { messageReceived(chatId: \"123\") { id text } }"
  }
}

// Server -> Client: Push event
{
  "id": "1",
  "type": "next",
  "payload": {
    "data": {
      "messageReceived": {
        "id": "msg-456",
        "text": "Hello!"
      }
    }
  }
}

// Client -> Server: Stop subscription
{
  "id": "1",
  "type": "complete"
}
```

---

## Implementing Subscriptions in Java

### Using Spring Boot with GraphQL Java

**1. Add Dependencies**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-core</artifactId>
</dependency>
```

**2. Define the Schema**

```graphql
type Query {
  messages(chatId: ID!): [Message!]!
}

type Mutation {
  sendMessage(chatId: ID!, text: String!): Message!
}

type Subscription {
  messageReceived(chatId: ID!): Message!
}

type Message {
  id: ID!
  chatId: ID!
  text: String!
  author: User!
  timestamp: DateTime!
}
```

**3. Create a Publisher Service**

```java
@Service
public class MessagePublisher {

    // Topic -> Sinks for that topic
    private final Map<String, Sinks.Many<Message>> sinks = new ConcurrentHashMap<>();

    /**
     * Get or create a publisher for a specific chat
     */
    public Flux<Message> getPublisher(String chatId) {
        String topic = "chat:" + chatId;
        Sinks.Many<Message> sink = sinks.computeIfAbsent(topic,
            k -> Sinks.many().multicast().directBestEffort());

        return sink.asFlux();
    }

    /**
     * Publish a new message
     */
    public void publish(Message message) {
        String topic = "chat:" + message.getChatId();
        Sinks.Many<Message> sink = sinks.get(topic);

        if (sink != null) {
            sink.tryEmitNext(message);
        }
    }

    /**
     * Clean up when no subscribers
     */
    @Scheduled(fixedRate = 60000)
    public void cleanup() {
        sinks.entrySet().removeIf(entry ->
            entry.getValue().currentSubscriberCount() == 0
        );
    }
}
```

**4. Implement the Subscription Resolver**

```java
@Controller
public class MessageSubscriptionResolver {

    @Autowired
    private MessagePublisher publisher;

    @SubscriptionMapping
    public Flux<Message> messageReceived(@Argument String chatId) {
        return publisher.getPublisher(chatId);
    }
}
```

**5. Trigger Events from Mutations**

```java
@Controller
public class MessageMutationResolver {

    @Autowired
    private MessageRepository messageRepository;

    @Autowired
    private MessagePublisher publisher;

    @MutationMapping
    public Message sendMessage(@Argument String chatId, @Argument String text) {
        // Save to database
        Message message = new Message();
        message.setChatId(chatId);
        message.setText(text);
        message.setTimestamp(Instant.now());

        message = messageRepository.save(message);

        // Publish to subscribers
        publisher.publish(message);

        return message;
    }
}
```

**6. Configure WebSocket**

```java
@Configuration
public class WebSocketConfig {

    @Bean
    public WebSocketHandlerDecoratorFactory messageLoggingDecorator() {
        return handler -> new WebSocketHandlerDecorator(handler) {
            @Override
            public void afterConnectionEstablished(WebSocketSession session) throws Exception {
                log.info("WebSocket connection established: {}", session.getId());
                super.afterConnectionEstablished(session);
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) {
                log.info("WebSocket connection closed: {} ({})", session.getId(), closeStatus);
                super.afterConnectionClosed(session, closeStatus);
            }
        };
    }
}
```

---

## Reactive Streams and Backpressure

Subscriptions use **reactive streams** (Project Reactor):

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);
    void onNext(T item);
    void onError(Throwable error);
    void onComplete();
}
```

### Backpressure: What If Events Come Too Fast?

```java
@SubscriptionMapping
public Flux<Message> messageReceived(@Argument String chatId) {
    return publisher.getPublisher(chatId)
        // Buffer up to 100 messages
        .buffer(100)
        .flatMap(Flux::fromIterable)

        // Or sample: only send 1 message per second max
        .sample(Duration.ofSeconds(1))

        // Or conflate: only keep latest
        .onBackpressureLatest();
}
```

**Strategies:**
- **Buffer**: Hold messages until client catches up
- **Drop**: Discard excess messages
- **Latest**: Keep only the most recent
- **Error**: Fail the subscription if overwhelmed

Choice depends on your use case:
- Stock prices: Latest value is what matters
- Chat messages: Can't drop any
- Telemetry: Sampling is acceptable

---

## Filtering and Authorization

### Field-Level Filtering

```java
@SubscriptionMapping
public Flux<Message> messageReceived(@Argument String chatId,
                                     @Argument(required = false) String userId) {
    Flux<Message> stream = publisher.getPublisher(chatId);

    // Filter by user if specified
    if (userId != null) {
        stream = stream.filter(msg -> msg.getAuthor().getId().equals(userId));
    }

    return stream;
}
```

### Authorization

```java
@SubscriptionMapping
@PreAuthorize("@chatAuthService.canAccessChat(#chatId, authentication)")
public Flux<Message> messageReceived(@Argument String chatId) {
    return publisher.getPublisher(chatId);
}
```

Or dynamic authorization per event:

```java
@SubscriptionMapping
public Flux<Message> messageReceived(@Argument String chatId,
                                     @AuthenticationPrincipal User currentUser) {
    return publisher.getPublisher(chatId)
        .filter(message -> {
            // Check permission for each message
            return authService.canViewMessage(currentUser, message);
        });
}
```

**Important:** Authorization happens at:
1. **Connection time**: Valid WebSocket connection
2. **Subscription time**: Can user subscribe to this topic?
3. **Event time**: Can user see this specific event?

---

## Scaling Subscriptions

### The Problem with Multiple Servers

```
Server 1: Client A subscribed to chat:123
Server 2: Client B subscribed to chat:123

Mutation hits Server 2:
- sendMessage(chatId: "123")
- Server 2 publishes locally
- Client B receives update ✓
- Client A receives nothing ✗
```

In-memory pub/sub doesn't work across servers!

### Solution 1: Redis Pub/Sub

```java
@Service
public class RedisMessagePublisher {

    @Autowired
    private RedisTemplate<String, Message> redisTemplate;

    public Flux<Message> getPublisher(String chatId) {
        String channel = "chat:" + chatId;

        return Flux.create(sink -> {
            MessageListener listener = (message, pattern) -> {
                Message msg = deserialize(message.getBody());
                sink.next(msg);
            };

            // Subscribe to Redis channel
            redisTemplate.getConnectionFactory()
                .getConnection()
                .subscribe(listener, channel.getBytes());

            sink.onDispose(() -> {
                // Unsubscribe when client disconnects
                redisTemplate.getConnectionFactory()
                    .getConnection()
                    .getSubscription()
                    .unsubscribe();
            });
        });
    }

    public void publish(Message message) {
        String channel = "chat:" + message.getChatId();
        redisTemplate.convertAndSend(channel, message);
    }
}
```

**Flow with Redis:**
```
Client A --> Server 1 --> Redis --> Server 1 --> Client A
                           |  \
                           |   --> Server 2 --> Client B
                           |
Client C --> Server 2 ----/
```

### Solution 2: Kafka for Subscriptions

```java
@Service
public class KafkaMessagePublisher {

    @Autowired
    private KafkaTemplate<String, Message> kafkaTemplate;

    public Flux<Message> getPublisher(String chatId) {
        return KafkaReceiver.create(receiverOptions)
            .receive()
            .filter(record -> record.key().equals(chatId))
            .map(record -> record.value());
    }

    public void publish(Message message) {
        kafkaTemplate.send("messages", message.getChatId(), message);
    }
}
```

**When to use what:**
- **In-memory**: Single server, development
- **Redis**: Multiple servers, simple pub/sub
- **Kafka**: High throughput, persistence, replay capability

---

## Connection Management

### Handling Disconnects

```java
@Component
public class WebSocketConnectionManager {

    private final Map<String, WebSocketSession> activeSessions = new ConcurrentHashMap<>();

    public void registerSession(String sessionId, WebSocketSession session) {
        activeSessions.put(sessionId, session);
        log.info("Registered session: {} (total: {})", sessionId, activeSessions.size());
    }

    public void removeSession(String sessionId) {
        activeSessions.remove(sessionId);
        log.info("Removed session: {} (remaining: {})", sessionId, activeSessions.size());
    }

    @Scheduled(fixedRate = 30000)
    public void cleanupStaleConnections() {
        activeSessions.entrySet().removeIf(entry -> {
            WebSocketSession session = entry.getValue();
            return !session.isOpen();
        });
    }
}
```

### Reconnection Strategy (Client-Side)

```javascript
class GraphQLSubscriptionClient {
  constructor(url) {
    this.url = url;
    this.reconnectDelay = 1000;
    this.maxReconnectDelay = 30000;
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onclose = () => {
      console.log('Disconnected, reconnecting in', this.reconnectDelay);
      setTimeout(() => this.connect(), this.reconnectDelay);

      // Exponential backoff
      this.reconnectDelay = Math.min(
        this.reconnectDelay * 2,
        this.maxReconnectDelay
      );
    };

    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectDelay = 1000; // Reset on successful connection
      this.resubscribeAll(); // Re-subscribe to previous subscriptions
    };
  }
}
```

### Heartbeats / Keepalive

```java
@Configuration
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(graphQLWebSocketHandler(), "/graphql")
            .setAllowedOrigins("*")
            .withSockJS()
            .setHeartbeatTime(25000); // Send heartbeat every 25 seconds
    }
}
```

Why heartbeats matter:
- Detect dead connections
- Keep NAT/firewall mappings alive
- Mobile network timeouts
- Load balancer connection tracking

---

## Testing Subscriptions

### Unit Test: Publisher Logic

```java
@Test
public void testMessagePublisher() {
    MessagePublisher publisher = new MessagePublisher();

    // Subscribe to chat
    Flux<Message> stream = publisher.getPublisher("chat-123");

    StepVerifier.create(stream)
        .then(() -> {
            // Publish message
            Message msg = new Message("chat-123", "Hello!");
            publisher.publish(msg);
        })
        .expectNextMatches(msg ->
            msg.getChatId().equals("chat-123") &&
            msg.getText().equals("Hello!")
        )
        .thenCancel()
        .verify();
}
```

### Integration Test: WebSocket

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class SubscriptionIntegrationTest {

    @LocalServerPort
    private int port;

    @Test
    public void testMessageSubscription() throws Exception {
        WebSocketStompClient stompClient = new WebSocketStompClient(
            new SockJsClient(List.of(new WebSocketTransport(new StandardWebSocketClient())))
        );

        String url = "ws://localhost:" + port + "/graphql";
        StompSession session = stompClient.connect(url, new StompSessionHandlerAdapter() {})
            .get(5, TimeUnit.SECONDS);

        CountDownLatch latch = new CountDownLatch(1);
        AtomicReference<Message> receivedMessage = new AtomicReference<>();

        session.subscribe("/topic/chat.123", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) {
                return Message.class;
            }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                receivedMessage.set((Message) payload);
                latch.countDown();
            }
        });

        // Send message via mutation
        sendMessageMutation("chat-123", "Test message");

        // Verify subscription received it
        assertTrue(latch.await(5, TimeUnit.SECONDS));
        assertEquals("Test message", receivedMessage.get().getText());
    }
}
```

---

## Common Patterns

### Pattern 1: Mutation + Subscription

```graphql
mutation {
  sendMessage(chatId: "123", text: "Hello") {
    id
    text
    timestamp
  }
}

subscription {
  messageReceived(chatId: "123") {
    id
    text
    timestamp
  }
}
```

The sender receives the message twice:
1. In the mutation response (immediate)
2. Via subscription (pushed)

**Solution:** Client deduplication by ID:

```javascript
const messageCache = new Set();

subscription.subscribe({
  next: (message) => {
    if (!messageCache.has(message.id)) {
      messageCache.add(message.id);
      displayMessage(message);
    }
  }
});
```

### Pattern 2: Optimistic Updates

```javascript
// Immediately show message (optimistic)
const tempId = 'temp-' + Date.now();
displayMessage({ id: tempId, text: 'Hello', sending: true });

// Send mutation
sendMessageMutation({ text: 'Hello' })
  .then(result => {
    // Replace temp message with real one
    replaceMessage(tempId, result.sendMessage);
  })
  .catch(error => {
    // Remove temp message on error
    removeMessage(tempId);
    showError(error);
  });

// Subscription will also receive the message, dedupe it
```

### Pattern 3: Presence Indicators

```graphql
type Subscription {
  userPresence(chatId: ID!): PresenceEvent!
}

type PresenceEvent {
  user: User!
  status: PresenceStatus!
  lastSeen: DateTime
}

enum PresenceStatus {
  ONLINE
  AWAY
  OFFLINE
}
```

```java
@SubscriptionMapping
public Flux<PresenceEvent> userPresence(@Argument String chatId) {
    // Send current presence state immediately
    List<PresenceEvent> currentState = presenceService.getCurrentPresence(chatId);

    // Then stream updates
    Flux<PresenceEvent> updates = presencePublisher.getPublisher(chatId);

    return Flux.concat(
        Flux.fromIterable(currentState),
        updates
    );
}
```

---

## Performance Considerations

### Connection Limits

Each WebSocket connection consumes:
- File descriptor (OS limit: ~65K typically)
- Memory (~10-50 KB per connection)
- CPU for keepalives and events

**Strategies:**
- Connection pooling on client
- Monitor open connections
- Rate limit new connections
- Auto-close idle connections

```java
@Component
public class ConnectionLimiter implements HandshakeInterceptor {

    private final AtomicInteger connectionCount = new AtomicInteger(0);
    private final int maxConnections = 10000;

    @Override
    public boolean beforeHandshake(ServerHttpRequest request,
                                  ServerHttpResponse response,
                                  WebSocketHandler wsHandler,
                                  Map<String, Object> attributes) {
        if (connectionCount.get() >= maxConnections) {
            response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return false;
        }
        connectionCount.incrementAndGet();
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request,
                              ServerHttpResponse response,
                              WebSocketHandler wsHandler,
                              Exception exception) {
        if (exception != null) {
            connectionCount.decrementAndGet();
        }
    }
}
```

### Subscription Fanout

With 10,000 clients subscribed to the same topic:

```java
public void publish(Message message) {
    String topic = "chat:" + message.getChatId();
    Sinks.Many<Message> sink = sinks.get(topic);

    if (sink != null) {
        // This loops through all 10,000 subscribers!
        sink.tryEmitNext(message);
    }
}
```

**Optimizations:**
- Use `Sinks.many().multicast()` for broadcast
- Batch notifications if possible
- Offload to separate thread pool
- Consider message broker for massive fanout

---

## Security Considerations

### Rate Limiting

```java
@Component
public class SubscriptionRateLimiter {

    private final LoadingCache<String, RateLimiter> limiters = CacheBuilder.newBuilder()
        .expireAfterAccess(1, TimeUnit.HOURS)
        .build(new CacheLoader<String, RateLimiter>() {
            @Override
            public RateLimiter load(String key) {
                return RateLimiter.create(10.0); // 10 subscriptions per second
            }
        });

    public boolean allowSubscription(String userId) {
        RateLimiter limiter = limiters.getUnchecked(userId);
        return limiter.tryAcquire();
    }
}
```

### Subscription Complexity

```java
@SubscriptionMapping
public Flux<Message> messageReceived(@Argument String chatId,
                                     @ContextValue GraphQLContext context) {
    // Check subscription depth/complexity
    QueryComplexity complexity = context.get("complexity");
    if (complexity.getDepth() > 5) {
        throw new GraphQLException("Subscription too complex");
    }

    return publisher.getPublisher(chatId);
}
```

### Prevent Subscription Storms

```java
@Component
public class SubscriptionGuard {

    private final Map<String, Set<String>> userSubscriptions = new ConcurrentHashMap<>();
    private final int maxSubscriptionsPerUser = 50;

    public boolean canSubscribe(String userId, String subscriptionId) {
        Set<String> subs = userSubscriptions.computeIfAbsent(userId,
            k -> ConcurrentHashMap.newKeySet());

        if (subs.size() >= maxSubscriptionsPerUser) {
            return false;
        }

        subs.add(subscriptionId);
        return true;
    }

    public void unsubscribe(String userId, String subscriptionId) {
        Set<String> subs = userSubscriptions.get(userId);
        if (subs != null) {
            subs.remove(subscriptionId);
        }
    }
}
```

---

## Subscriptions vs Alternatives

### When to Use Subscriptions

✓ Real-time chat
✓ Live sports scores
✓ Stock price updates
✓ Collaboration (Google Docs-style)
✓ Notifications
✓ Progress indicators

### When NOT to Use Subscriptions

✗ **Rare updates**: Poll if updates are infrequent (e.g., daily reports)
✗ **Historical data**: Use queries for past data
✗ **Mobile apps with background restrictions**: iOS/Android limit background connections
✗ **High-latency networks**: WebSocket overhead might not be worth it
✗ **Simple CRUD**: Standard queries/mutations are simpler

### Alternative: HTTP Streaming

```java
@GetMapping(value = "/messages/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<Message>> streamMessages(@RequestParam String chatId) {
    return publisher.getPublisher(chatId)
        .map(message -> ServerSentEvent.<Message>builder()
            .id(message.getId())
            .event("message")
            .data(message)
            .build());
}
```

**Server-Sent Events (SSE):**
- One-way: Server → Client only
- HTTP-based: Works with standard HTTP/2
- Simpler than WebSocket
- Auto-reconnects built-in

Use SSE if you only need server→client push, no client→server messages.

---

## Real-World Example: Chat Application

**Complete Schema:**

```graphql
type Query {
  chat(id: ID!): Chat
  chats: [Chat!]!
  messages(chatId: ID!, limit: Int = 50): [Message!]!
}

type Mutation {
  createChat(name: String!, userIds: [ID!]!): Chat!
  sendMessage(chatId: ID!, text: String!): Message!
  markAsRead(chatId: ID!): Boolean!
  startTyping(chatId: ID!): Boolean!
  stopTyping(chatId: ID!): Boolean!
}

type Subscription {
  # New messages
  messageReceived(chatId: ID!): Message!

  # Typing indicators
  userTyping(chatId: ID!): TypingEvent!

  # Read receipts
  messagesRead(chatId: ID!): ReadReceipt!

  # User presence
  userPresence(chatId: ID!): PresenceEvent!
}

type Chat {
  id: ID!
  name: String!
  participants: [User!]!
  lastMessage: Message
  unreadCount: Int!
}

type Message {
  id: ID!
  chatId: ID!
  text: String!
  author: User!
  timestamp: DateTime!
  readBy: [User!]!
}

type TypingEvent {
  chatId: ID!
  user: User!
  isTyping: Boolean!
}

type ReadReceipt {
  chatId: ID!
  user: User!
  lastReadMessageId: ID!
  timestamp: DateTime!
}

type PresenceEvent {
  user: User!
  status: PresenceStatus!
  lastSeen: DateTime
}
```

**Complete Resolver:**

```java
@Controller
public class ChatSubscriptionResolver {

    @Autowired
    private MessagePublisher messagePublisher;

    @Autowired
    private TypingPublisher typingPublisher;

    @Autowired
    private ReadReceiptPublisher readReceiptPublisher;

    @Autowired
    private PresencePublisher presencePublisher;

    @SubscriptionMapping
    public Flux<Message> messageReceived(@Argument String chatId) {
        return messagePublisher.getPublisher(chatId);
    }

    @SubscriptionMapping
    public Flux<TypingEvent> userTyping(@Argument String chatId) {
        return typingPublisher.getPublisher(chatId)
            // Debounce: if no updates for 3s, send "stopped typing"
            .timeout(Duration.ofSeconds(3), Flux.empty())
            .onErrorResume(TimeoutException.class, e -> {
                return Flux.just(new TypingEvent(chatId, null, false));
            });
    }

    @SubscriptionMapping
    public Flux<ReadReceipt> messagesRead(@Argument String chatId) {
        return readReceiptPublisher.getPublisher(chatId);
    }

    @SubscriptionMapping
    public Flux<PresenceEvent> userPresence(@Argument String chatId,
                                           @AuthenticationPrincipal User currentUser) {
        // Send current state first
        List<PresenceEvent> currentState = presenceService.getCurrentPresence(chatId);

        // Filter: only show presence for chat participants
        Flux<PresenceEvent> updates = presencePublisher.getPublisher(chatId)
            .filter(event -> chatService.isParticipant(chatId, event.getUser().getId()));

        return Flux.concat(Flux.fromIterable(currentState), updates);
    }
}
```

---

## Key Takeaways

1. **Subscriptions enable real-time GraphQL** - Push updates instead of polling

2. **Built on pub/sub** - Publishers emit events, subscribers receive filtered streams

3. **WebSockets are standard** - `graphql-ws` protocol for bidirectional communication

4. **Reactive streams in Java** - Project Reactor provides Flux for event streams

5. **Scale with message brokers** - Redis/Kafka for multi-server deployments

6. **Authorization at multiple levels** - Connection, subscription, and per-event

7. **Handle backpressure** - Clients can't always keep up with event rate

8. **Monitor connection health** - Heartbeats, reconnection, cleanup

9. **Not always the answer** - Consider polling, SSE, or webhooks for some use cases

10. **Security matters** - Rate limiting, complexity analysis, connection limits

---

**Next Steps:**
- Read [Chapter 17: Security](17-security.md) to learn about protecting subscriptions from abuse
- See [Chapter 19: Building a Server](19-building-server.md) for a complete implementation with subscriptions

---

**Previous:** [← Chapter 15: Mutations and Side Effects](15-mutations.md)
**Next:** [Chapter 17: Security - Query Depth, Complexity, and Cost Analysis →](17-security.md)
