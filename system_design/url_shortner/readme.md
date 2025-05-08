# Chhola-Muri URL Shortener

A high-performance URL shortening service designed to handle high-traffic loads with minimal latency.

## Team Members

This system is being collaboratively designed and implemented by:

- Andalib
- Selim
- Habib
- Lenin
- Opu
- Shemul

## Project Overview

Chhola-Muri is a URL shortening service that converts long URLs into shorter, more manageable links. The service is designed to be highly available, scalable, and capable of handling burst traffic patterns.

## Functional Requirements

- **URL Shortening**: Convert long URLs to short, easy-to-share links
- **Custom Aliases**: Allow users to specify custom aliases for their shortened URLs
- **Time-To-Live (TTL)**: Support for links that expire after a specified period
- **Metrics and Analytics**: Track usage statistics for shortened URLs

## Non-Functional Requirements

- **Availability**: Prioritize availability over consistency (AP system)
- **Traffic Pattern**: Read-heavy system with 80:20 read-to-write ratio
- **Access Pattern**: Support for burst query access patterns
- **Scale**: Support for up to 1 billion short URLs
- **Latency**: Maximum 50ms latency for redirect operations
- **User Base**: Support for 100,000 daily active users

## System Architecture

[View our complete system architecture diagram](https://excalidraw.com/#room=a1b2bd4d684df3a5c8f2,GQSe2tHBOz5WF1Wh_gnj6g)

Our architecture follows modern distributed systems design principles to meet the non-functional requirements:

- Load balancers for traffic distribution
- Caching layer for frequently accessed URLs
- Distributed database for storage
- Metrics collection and monitoring

## Database Design

The system uses a combination of storage solutions:
- Fast key-value store for URL mappings
- Relational database for user data and analytics
- Cache for frequently accessed URLs

## ID Generation Strategy

We evaluated several approaches for generating short URL identifiers:

### Considered Options:

1. **Distributed ID with Base62 encoding**
   - Pros: Simple implementation, sequential IDs
   - Cons: May leak information about usage patterns

2. **Hash (SHA-256 + Base62)**
   - Pros: Uniform distribution, fixed length
   - Cons: Not guaranteed to be collision-free at scale

3. **UUID**
   - Pros: Universally unique, well-established
   - Cons: Results in longer short URLs (not ideal)

4. **Snowflake**
   - Pros: Time-sorted, distributed, uniqueness guaranteed
   - Cons: Requires synchronized time across servers

5. **Custom hash(userID, timestamp, longURL)**
   - Pros: Can include user-specific information
   - Cons: More complex, potential privacy concerns

### Our Choice: Snowflake ID with Base62 Encoding

We selected **Snowflake** with Base62 encoding for our short URL ID generation because:

1. It provides **guaranteed uniqueness** in a distributed environment
2. The time-sorted nature allows for **efficient database operations**
3. It can generate **8.7+ billion IDs** per second per machine
4. The resulting IDs can be **encoded to short strings** using Base62
5. It works well with our **sharded architecture**

The Snowflake ID structure:
```
  41 bits - Timestamp (milliseconds since epoch)
  10 bits - Machine/worker ID
  12 bits - Sequence number (per millisecond)
```

### Collision Handling with Redis Bloom Filters

Despite Snowflake's uniqueness guarantees, we implement an additional collision prevention mechanism using **Redis Bloom filters**:

- We utilize Redis's native Bloom filter data structure (RedisBloom module)
- All service nodes share a centralized Redis Bloom filter for consistent collision detection
- Before confirming a new short URL, we check the Redis Bloom filter
- If the filter indicates a possible collision, we perform a database lookup
- False positives from the Bloom filter only result in unnecessary lookups, not collisions
- The Redis Bloom filter is tuned for a false positive rate of 0.01% with 1 billion items

This approach:
- Minimizes database queries for collision detection
- Provides a memory-efficient way to detect potential collisions
- Leverages Redis's high-performance, distributed architecture
- Ensures consistency across all application instances

## API Endpoints

### Create Short URL
```
POST /api/v1/urls
```

**Request:**
```json
{
    "long_url": "https://example.com/very/long/url",
    "custom_alias": "myCustomUrl" (optional)
}
```

**Response:**
```json
{
    "short_url": "http://short.ly/abc123",
    "long_url": "https://example.com/very/long/url",
    "created_at": "2025-04-24T03:05:00Z"
}
```

### Redirect to Original URL
```
GET /{short_url}
```

**Response:** 302 Redirect to original URL

**Note on 302 Redirect:** We use HTTP 302 (Found/Temporary Redirect) status code for redirects to ensure accurate metrics tracking. Unlike 301 redirects which are permanently cached by browsers, 302 redirects ensure that each click generates a request to our servers. This is critical for our URL shortener service as it allows us to:
1. Accurately count clicks and track usage metrics
2. Record user agent and referrer information for analytics
3. Apply time-based expiration and access controls on each request
4. Gather real-time data on link performance

**Why 302 Instead of 301:** While HTTP 301 (Moved Permanently) would provide some performance benefits through caching, it would prevent accurate metrics collection since browsers would bypass our servers on subsequent visits. Since metrics tracking is a core functional requirement of our URL shortener service, we prioritize complete tracking capability over the marginal performance improvement that 301 redirects would provide.

### URL Data Model
```json
{
    "id": "unique_identifier",
    "short_url": "string(4-32 chars)",
    "long_url": "string",
    "user_id": "foreign_key",
    "created_at": "timestamp",
    "updated_at": "timestamp",
    "expiry_date": "timestamp",
    "is_custom": "boolean",
    "is_active": "boolean",
    "click_count": "integer"
}
```

### Update Short URL
```
PUT /api/v1/urls/{short_url}
```

### Delete Short URL
```
DELETE /api/v1/urls/{short_url}
```

### Get URL Metrics
```
GET /api/v1/metrics/{short_url}
```