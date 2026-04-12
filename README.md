# social_network_system_design

## Functional Requirements (FR)

### Users & Profiles

- User registration and authentication (email / phone / OAuth)
- View and edit user profile (avatar, bio, travel preferences)
- View other users’ profiles
- Follow / unfollow users
- View followers and following lists

---

### Posts

- Create a travel post:
  - photos / videos
  - short description
  - location (place link)
  - categories (e.g., mountains, beach)
- Edit / delete posts
- View posts
- Save posts to favorites

---

### Places

- Attach post to a place
- View place page:
  - posts from that place
  - popularity
- Search places
- Autocomplete places during post creation

---

### Feed

- Following feed (reverse chronological order)
- Recommendation feed (personalized)
- Infinite scroll (cursor-based pagination)
- Refresh feed

---

### Recommendations

- Personalized recommendations based on:
  - liked posts
  - categories
  - subscriptions
- Boost new posts (<24h)
- Trending places and posts
- Cold start recommendations (popular content)

---

### Comments & Chat

- Add comment (≤ 2KB)
- Reply to comments (threads)
- Like comments
- Voice messages in discussion

---

### Reactions

- Like posts
- Remove like
- View like count

---

### Search

- Search users, posts, places
- Autocomplete (places, categories)
- Sort:
  - by popularity (default)
  - by recency
- Filters:
  - categories
  - region

---

### Content Control

- Hide post / user / place (30 days)
- Block users

---

### Notifications

- New likes
- Comments
- New followers

---

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

---

## Traffic Estimation

### Assumptions

- DAU = 10,000,000
- Seconds per day = 86,400

---

## Actions Load Table

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

---

## Key Observations

- Read-heavy system:
  - Read post → 4.6 GB/s (CDN required)
- Feed is cheap in terms of traffic, but expensive in terms of CPU
- Comments have high RPS but small payload
- Likes are lightweight but require hot counter scaling
- Voice messages produce noticeable traffic

---

## Connections

```
Simultaneous connections = DAU * 0.1 = 1,000,000
```

It means:

- WebSocket / HTTP keep-alive
- Need to keep alive ~1M connections (load balancers and kernel tuning)

---
