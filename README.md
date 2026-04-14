# social_network_system_design

## Functional Requirements (FR)

### Users & Profiles

- User registration and authentication (email / phone / OAuth)
- View and edit user profile (avatar, bio, travel preferences)
- View other users’ profiles
- Follow / unfollow users
- View followers and following lists

### Posts

- Create a travel post:
  - photos / videos
  - short description
  - location (place link)
  - categories (e.g., mountains, beach)
- Edit / delete posts
- View posts
- Save posts to favorites

### Places

- Attach post to a place
- View place page:
  - posts from that place
  - popularity
- Search places
- Autocomplete places during post creation

### Feed

- Following feed (reverse chronological order)
- Recommendation feed (personalized)
- Infinite scroll (cursor-based pagination)
- Refresh feed

### Recommendations

- Personalized recommendations based on:
  - liked posts
  - categories
  - subscriptions
- Boost new posts (<24h)
- Trending places and posts
- Cold start recommendations (popular content)

### Comments & Chat

- Add comment (≤ 2KB)
- Reply to comments (threads)
- Like comments
- Voice messages in discussion

### Reactions

- Like posts
- Remove like
- View like count

### Search

- Search users, posts, places
- Autocomplete (places, categories)
- Sort:
  - by popularity (default)
  - by recency
- Filters:
  - categories
  - region

### Content Control

- Hide post / user / place (30 days)
- Block users

### Notifications

- New likes
- Comments
- New followers

## Non-Functional Requirements (NFR)

- DAU = 10M
- MAU = 300M
- Availability (SLO): 99.99%
- Latency:
  - Feed load: < 200 ms (cached)
  - Post load: < 300 ms
- Media:
  - Stored compressed (avg 4 MB per post)
  - Delivered via CDN
- Regions:
  - Primary: CIS
- Scalability:
  - Horizontal scaling (stateless services)
- Consistency:
  - Eventual consistency for feeds and counters
- Durability:
  - Media stored in object storage (3x replication)
- Seasonality:
  - Peaks during holidays (×2–3 traffic)
- Max comment size: 2 KB
- Feed pagination: cursor-based
- Caching:
  - CDN for media
  - Redis for feeds, recommendations
- Fault tolerance:
  - Multi-region failover

## Traffic Estimation

### Assumptions

- DAU = 10,000,000
- Seconds per day = 86,400

### Actions Load Table

| Action                   | User RPD | Total RPD   | RPS  | Avg Size | MBPS  |
| ------------------------ | -------- | ----------- | ---- | -------- | ----- |
| Publish post             | 1/30     | 333,333     | 3.86 | 4 MB     | 15.43 |
| Read post                | 10       | 100,000,000 | 1157 | 4 MB     | 4629  |
| Read feed                | 5        | 50,000,000  | 578  | 50 KB    | 28.9  |
| Refresh feed             | 2        | 20,000,000  | 231  | 10 KB    | 2.31  |
| Add comment              | 3        | 30,000,000  | 347  | 2 KB     | 0.68  |
| Reply to comment         | 2        | 20,000,000  | 231  | 2 KB     | 0.46  |
| Read comments            | 6        | 60,000,000  | 694  | 10 KB    | 6.9   |
| Like post                | 5        | 50,000,000  | 578  | 100 B    | 0.055 |
| Like comment             | 2        | 20,000,000  | 231  | 100 B    | 0.023 |
| Save post (favorite)     | 1        | 10,000,000  | 115  | 200 B    | 0.023 |
| Search                   | 3        | 30,000,000  | 347  | 5 KB     | 1.73  |
| Autocomplete             | 5        | 50,000,000  | 578  | 2 KB     | 1.15  |
| View profile             | 2        | 20,000,000  | 231  | 10 KB    | 2.31  |
| Edit profile             | 0.1      | 1,000,000   | 11.5 | 5 KB     | 0.057 |
| Follow / unfollow        | 1        | 10,000,000  | 115  | 500 B    | 0.057 |
| View followers/following | 1        | 10,000,000  | 115  | 20 KB    | 2.3   |
| Load recommendations     | 4        | 40,000,000  | 463  | 50 KB    | 23.15 |
| Trending                 | 1        | 10,000,000  | 115  | 20 KB    | 2.3   |
| Hide content             | 0.5      | 5,000,000   | 57   | 200 B    | 0.011 |
| Block user               | 0.05     | 500,000     | 5.7  | 200 B    | 0.001 |
| Notifications read       | 3        | 30,000,000  | 347  | 5 KB     | 1.73  |
| Voice message upload     | 0.2      | 2,000,000   | 23   | 200 KB   | 4.6   |
| Voice message playback   | 0.5      | 5,000,000   | 57   | 200 KB   | 11.4  |

### Key Observations

- Read-heavy system:
  - Read post → 4.6 GB/s (CDN required)
- Feed is cheap in terms of traffic, but expensive in terms of CPU
- Comments have high RPS but small payload
- Likes are lightweight but require hot counter scaling
- Voice messages produce noticeable traffic

### Connections

```
Simultaneous connections = DAU * 0.1 = 1,000,000
```

It means:

- WebSocket / HTTP keep-alive
- Need to keep alive ~1M connections (load balancers and kernel tuning)

## API Service Implementation Details

This document describes key implementation decisions and architectural patterns used in the API layer of the system.

### 1. Media Upload via Presigned URLs

### Problem

Uploading large media files (images/videos) through the backend causes:

- High network load on API servers
- Increased latency
- Backend becoming a bottleneck under scale

### Solution: Presigned URLs (Direct-to-Storage Upload)

### Flow

1. Client requests upload URL:

```

POST /media/upload-url

```

2. API Service:

- Generates a presigned URL (S3 / MinIO)
- Applies constraints:
  - TTL (e.g., 300 seconds)
  - max file size
  - allowed content-type (image/jpeg, video/mp4)

3. Client uploads directly to object storage:

```

PUT https://storage-service/bucket/object?signature=...

```

4. After upload, client submits metadata:

```

POST /posts
{
"mediaUrls": [...]
}

```

### Benefits

- Backend is removed from data plane
- Upload scales with object storage, not API servers
- Lower latency and cost
- Easier CDN integration for delivery

### 2. Feed Generation (Hybrid Fan-out Model)

### Problem

Pure fan-out on write does not scale for users with large follower graphs.

Example:

- Influencer with millions of followers
- One post triggers millions of feed updates

### Solution: Hybrid Fan-out Strategy

### Strategy by user type

| User type    | Strategy         |
| ------------ | ---------------- |
| Normal users | Fan-out on write |
| Influencers  | Fan-out on read  |

### Write Path

```

Post Service
↓
Kafka (event stream)
↓
Feed Workers
↓
Redis (precomputed feeds)

```

### Read Path

```

GET /feed
↓
Redis (precomputed feed items)
↓
Optional merge with:

* influencer posts (on-the-fly)
* ranking layer

```

### Key Idea

Feed is partially precomputed and partially assembled at read time depending on cost profile.

### 3. Feed Storage Design (Redis)

### Data structure

Sorted Set (ZSET):

```

feed:user:{userId}

```

- score = timestamp or ranking score
- value = post_id

### Benefits

- Efficient range queries
- Natural ordering by time
- Supports pagination via cursor

### 4. Event-Driven Architecture (Kafka)

### Usage

All write-heavy actions emit events:

- PostCreated
- CommentAdded
- LikeAdded
- UserFollowed

### Why Kafka is used

- Decouples services
- Enables async processing
- Allows rebuilding projections (feeds, counters)
- Provides buffering during traffic spikes

### 5. Counters Scaling (Likes, Reactions)

### Problem

Direct DB updates on hot counters cause:

- row locking
- contention
- performance degradation

### Solution: Sharded Counters

Instead of:

```

likes_count = 1,000,000

```

Use:

```

likes_post_1_shard_1 = 120k
likes_post_1_shard_2 = 110k
likes_post_1_shard_3 = 130k

```

### Aggregation

Final value:

```

total_likes = SUM(all_shards)

```

### Optimization

- Cache aggregated value in Redis
- Periodic async compaction job

### 6. Caching Strategy

### Multi-layer caching

### CDN

- Media files (images/videos)
- Static assets

### Redis

- Feeds
- Recommendations
- Trending data

### Application cache

- Hot user profiles
- Frequently accessed places

### Cache invalidation strategy

- TTL-based expiration
- Event-driven invalidation (Kafka events)

### 7. Search Architecture

### Problem

Database search does not scale for:

- autocomplete
- fuzzy search
- ranking by popularity

### Solution

Dedicated search engine:

- Elasticsearch / OpenSearch

### Indexes:

- users index
- posts index
- places index

### Features

- full-text search
- autocomplete (prefix matching)
- ranking by popularity + recency

### 8. Multi-Region Architecture

### Setup

- Primary region: CIS (Moscow)
- Secondary region: Almaty

### Strategy

- Active-passive or active-active depending on service
- Async replication of:
  - posts
  - feeds
  - user actions

### Failover

- DNS or global load balancer
- Health-check based routing

### 9. Consistency Model

### Chosen model

Eventual consistency for:

- feeds
- likes counters
- recommendations

Strong consistency for:

- authentication
- user profile updates

### 10. Rate Limiting & Protection

### Mechanisms

- Token bucket per user/IP
- API gateway enforcement
- Per-endpoint throttling

### Protection goals

- prevent spam likes/comments
- protect feed service from abuse
- stabilize system during traffic spikes

### Summary

The system is designed around:

- Offloading heavy operations to async pipelines
- Using presigned URLs for media scalability
- Hybrid feed generation model
- Event-driven architecture (Kafka)
- Strong caching strategy (Redis + CDN)
- Eventual consistency for scalability
