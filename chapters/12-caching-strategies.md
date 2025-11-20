# Chapter 12: Caching Strategies - Field-Level vs Query-Level

**Status:** Detailed Outline (To Be Expanded)

---

## Outline

### Part 1: Why Caching is Different in GraphQL
- Dynamic queries make HTTP caching hard
- Field-level cache granularity
- Cache invalidation challenges
- POST requests and browser caching

### Part 2: Client-Side Caching
- Normalized cache (Apollo, Relay)
- Query result caching
- Cache updates after mutations
- Optimistic updates

### Part 3: Server-Side Caching Strategies
- Full query response caching
- Field-level caching
- DataLoader per-request cache
- Redis integration

### Part 4: Cache Control Directives
- @cacheControl directive design
- maxAge and scope
- Public vs private caching
- Cache hints aggregation

### Part 5: Automatic Persisted Queries (APQ)
- What is APQ?
- Query ID generation (SHA256 hash)
- APQ protocol flow
- Benefits and tradeoffs

### Part 6: CDN and Edge Caching
- Making POST requests cacheable
- GET queries with query hashes
- Edge workers for GraphQL
- Regional cache strategies

### Part 7: Cache Invalidation
- Tag-based invalidation
- Type-based invalidation
- Mutation response invalidation
- Subscription-based invalidation

### Java Implementation Topics:
- Spring Cache integration
- Redis caching layer
- Caffeine for in-memory caching
- Cache key generation
- Cache serialization

### Performance Metrics:
- Cache hit rates
- Response time improvements
- Memory usage
- Stale data handling

---

**Next:** [Chapter 13: Error Handling ’](13-error-handling.md)
