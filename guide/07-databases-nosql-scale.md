# 07 NoSQL, Redis, and Scaling Your Database 📈
### because eventually PostgreSQL on one server isn't enough

---

"just use PostgreSQL" is good advice for 95% of projects. then the other 5% hits you. your query that used to take 2ms now takes 2 seconds because the table has 500 million rows. your write throughput peaks at 5,000 writes/sec and you need 50,000. a single region isn't enough latency-wise for your global user base.

this is the scaling problem. and the solutions are genuinely interesting once you understand *why* they exist.

---

## the CAP theorem: a framework (not a rule)

before NoSQL, you need to understand the CAP theorem because it explains why different databases make different tradeoffs.

**CAP stands for:**
- **C**onsistency: every read receives the most recent write or an error
- **A**vailability: every request receives a non-error response (though it might be stale)
- **P**artition tolerance: the system continues operating when network partitions occur (some nodes can't talk to others)

the theorem says: **in the presence of a network partition, you must choose between Consistency and Availability.** you can't have both.

```
PostgreSQL/MySQL:    CP (choose consistency, may be unavailable during partition)
MongoDB:            usually AP (stays available, might serve stale data)
Cassandra:          AP (eventual consistency, always available)
DynamoDB:           AP by default, strongly consistent reads available
Redis:              varies by configuration
```

network partitions happen. cables get cut. switches fail. AWS zones have issues. building a distributed system means accepting you'll face partitions and deciding which side of C/A you fall on.

in practice: **most backend applications should default to CP databases (PostgreSQL) because consistency is easier to reason about.** choose AP when your use case genuinely requires always-on availability (social media feeds, recommendation systems, analytics) over perfect consistency.

---

## Redis: the data structure server

Redis (Remote Dictionary Server) is one of the most useful tools in a backend engineer's arsenal. it's often called an "in-memory database" but that undersells it, Redis is better described as a **data structure server** that happens to live in memory.

it supports: strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, streams, geospatial indexes.

in-memory means sub-millisecond operations. a Redis lookup is typically **< 1ms**. a PostgreSQL query (even with an index, on a local machine) is typically 1-10ms. Redis is 10-100x faster for the operations it supports.

### the canonical use cases

**1. Caching**

```go
import "github.com/redis/go-redis/v9"

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    
    // try cache first
    cached, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var user User
        if err := json.Unmarshal(cached, &user); err == nil {
            return &user, nil  // cache hit
        }
    }
    
    // cache miss: fetch from database
    user, err := s.db.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // store in cache with 5-minute TTL
    data, _ := json.Marshal(user)
    s.redis.Set(ctx, key, data, 5*time.Minute)
    
    return user, nil
}

// when user is updated, invalidate the cache
func (s *UserService) UpdateUser(ctx context.Context, user *User) error {
    if err := s.db.UpdateUser(ctx, user); err != nil {
        return err
    }
    
    key := fmt.Sprintf("user:%d", user.ID)
    s.redis.Del(ctx, key)  // invalidate cache
    return nil
}
```

**2. Session storage**

```go
// store session data in Redis with expiry
sessionID := generateSessionID()
sessionData, _ := json.Marshal(session)
s.redis.Set(ctx, "session:"+sessionID, sessionData, 24*time.Hour)

// retrieve
data, err := s.redis.Get(ctx, "session:"+sessionID).Bytes()
```

**3. Rate limiting with sliding window**

```go
func (rl *RateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().UnixMilli()
    window := int64(60000) // 60 seconds in milliseconds
    
    pipe := rl.redis.Pipeline()
    
    // remove old entries outside the window
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(now-window, 10))
    
    // count current requests in window
    count := pipe.ZCard(ctx, key)
    
    // add current request
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})
    
    // set expiry on the key
    pipe.Expire(ctx, key, time.Minute)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    return count.Val() < int64(rl.limit), nil
}
```

**4. Distributed locks**

```go
// ensure only one instance does a job at a time
func (s *Service) runWithLock(ctx context.Context, lockKey string, fn func() error) error {
    // SET lock_key 1 NX EX 30  (set if not exists, expire in 30s)
    set, err := s.redis.SetNX(ctx, "lock:"+lockKey, "1", 30*time.Second).Result()
    if err != nil {
        return err
    }
    if !set {
        return ErrLockNotAcquired // another instance holds the lock
    }
    defer s.redis.Del(ctx, "lock:"+lockKey)
    
    return fn()
}
```

**5. Pub/Sub for real-time features**

```go
// publisher
s.redis.Publish(ctx, "notifications:user:123", message)

// subscriber (in a goroutine)
sub := s.redis.Subscribe(ctx, "notifications:user:123")
for msg := range sub.Channel() {
    // handle real-time notification
    sendToWebSocket(msg.Payload)
}
```

**6. Sorted sets for leaderboards**

```go
// add/update score
s.redis.ZAdd(ctx, "leaderboard:daily", redis.Z{
    Score:  float64(score),
    Member: userID,
})

// get top 10
top10, _ := s.redis.ZRevRangeWithScores(ctx, "leaderboard:daily", 0, 9).Result()

// get user's rank
rank, _ := s.redis.ZRevRank(ctx, "leaderboard:daily", userID).Result()
```

sorted sets are O(log N) for add/update and O(log N + K) for range queries. leaderboard for 10 million users, instant lookup. this would be painful in SQL.

---

## Redis data structures in depth

understanding which data structure to use is where Redis expertise comes from:

| Structure | Commands | Use case |
|-----------|----------|---------|
| String | GET, SET, INCR | simple caching, counters, feature flags |
| Hash | HGET, HSET, HMGET | user profiles, object caching (avoid JSON serialization) |
| List | LPUSH, RPOP, LRANGE | queues, activity feeds, chat history |
| Set | SADD, SMEMBERS, SINTER | unique user tracking, tagging, friend lists |
| Sorted Set | ZADD, ZRANGE, ZRANK | leaderboards, rate limiting, delayed job queues |
| Stream | XADD, XREAD | event sourcing, message queuing (replaces Pub/Sub for durability) |
| HyperLogLog | PFADD, PFCOUNT | approximate unique count (memory-efficient: 12KB for any cardinality) |

**HyperLogLog for counting unique visitors:**

```go
// each page view
s.redis.PFAdd(ctx, "page_views:homepage:"+today, userID)

// get approximate unique count (error ~0.81%)
count, _ := s.redis.PFCount(ctx, "page_views:homepage:"+today).Result()
```

exact counting of unique visitors with a traditional SET would use O(n) memory. 1 million unique visitors = 1 million entries in a Redis set = ~50MB. HyperLogLog gives you the same information (with ~0.81% error) using 12KB regardless of cardinality. that's a 4000x memory reduction.

---

## Redis at scale: clustering and replication

a single Redis instance handles up to ~1 million operations per second. that's enough for most applications. when you need more:

**Redis Replication (for high availability):**
- one primary, multiple replicas
- replicas handle reads (read scaling)
- if primary fails, promote a replica (automatic with Redis Sentinel)
- replicas are eventually consistent (async replication)

**Redis Cluster (for horizontal scaling):**
- data is sharded across multiple nodes using **consistent hashing**
- Redis Cluster uses 16,384 **hash slots**
- each key maps to a slot: `CRC16(key) % 16384`
- slots are distributed across nodes

```
Node 1: slots 0-5460
Node 2: slots 5461-10922
Node 3: slots 10923-16383
```

each node has replicas. if a node fails, its replica is promoted. reads/writes are routed to the correct node automatically.

scaling Redis Cluster to **1 million operations/second**: use ~10 shards (primary + 2 replicas each = 30 nodes total). each node handles ~100k ops/sec. [real architecture from Slack](https://slack.engineering/scaling-slack-the-good-the-unexpected-and-the-road-ahead/) handles billions of Redis commands per day this way.

---

## MongoDB: when to actually use it

MongoDB is the most popular NoSQL database. it stores **documents** (JSON-like BSON objects) in **collections** (like tables but schema-less).

```javascript
// MongoDB document
{
  "_id": ObjectId("..."),
  "username": "moe",
  "profile": {
    "bio": "just a gopher",
    "location": "somewhere",
    "avatar_url": "..."
  },
  "tags": ["developer", "go-fan"],
  "posts": [
    {
      "title": "my first post",
      "content": "...",
      "created_at": ISODate("2026-01-15")
    }
  ]
}
```

**MongoDB is good when:**
- your data is genuinely document-shaped with variable structure per record
- you're prototyping and the schema isn't settled yet
- you need horizontal write scaling from day one (sharding is a first-class feature)
- you're storing event data, log data, or catalog data with highly variable fields

**MongoDB is wrong when:**
- you need ACID transactions across multiple documents (MongoDB has multi-document transactions since 4.0, but they're slower and less mature than PostgreSQL's)
- your data is relational (many-to-many relationships, complex joins)
- you're thinking "I'll use MongoDB because no schema = faster development", this is almost always wrong. you still have a de-facto schema, you've just moved the enforcement to application code, which is worse.

the "flexible schema = faster development" myth has caused a lot of MongoDB regret. you end up with a collection where 60% of documents have `user_id`, 30% have `userId`, and 10% have `uid`, all meaning the same thing. relational databases enforce consistency. MongoDB trusts you to enforce it yourself. teams often don't.

---

## other NoSQL: a quick tour

**Cassandra / ScyllaDB:**
- column-family store, designed for massive write throughput and horizontal scaling
- no joins, no transactions (AP system)
- data must be modeled around your query patterns (not normalized)
- great for: IoT time-series, activity feeds, messaging at billions-of-messages scale
- ScyllaDB is Cassandra reimplemented in C++ and is significantly faster

**DynamoDB:**
- AWS's managed NoSQL, serverless, infinitely scalable
- key-value + document store
- pricing is tricky (read/write capacity units)
- great for: serverless architectures, AWS-native apps, gaming (leaderboards, sessions)
- bad for: complex queries, joins, anything relational

**ClickHouse:**
- columnar OLAP database for analytics
- optimized for aggregate queries on huge datasets (billions of rows, millisecond response)
- NOT for transactional workloads (high insert throughput but poor point updates)
- great for: real-time analytics, dashboards, log analysis, time-series

**TimescaleDB:**
- PostgreSQL extension for time-series data
- you get all of PostgreSQL's features plus time-series optimizations
- automatic partitioning by time, time-based aggregation functions
- great for: metrics, monitoring, IoT, any timestamped data

---

## scaling PostgreSQL: the real techniques

you've decided to stick with PostgreSQL (good choice for most things). here's how you scale it:

### read replicas

```
                ┌─────────────────┐
                │  PostgreSQL     │
writes ─────────│  PRIMARY        │─────── to replicas
                │  (read/write)   │
                └─────────────────┘
                         │
           streaming replication
           (async, sub-second lag)
                         │
           ┌─────────────┴──────────┐
           │                         │
   ┌───────┴──────┐         ┌───────┴──────┐
   │  Replica 1   │         │  Replica 2   │
   │  (read only) │         │  (read only) │
   └──────────────┘         └──────────────┘
           ↑                         ↑
     reporting queries         API read queries
```

in Go with pgx, you can route reads to replicas:

```go
type DB struct {
    primary  *pgxpool.Pool
    replicas []*pgxpool.Pool
    rng      *rand.Rand
}

func (db *DB) ReadPool() *pgxpool.Pool {
    // random load balancing across replicas
    return db.replicas[db.rng.Intn(len(db.replicas))]
}
```

**replication lag** is real: replicas are eventually consistent with the primary. if you write data and immediately read it from a replica, you might get stale data. for operations where freshness is critical (verify payment before showing success page), always read from primary.

### table partitioning

```sql
-- partition a large events table by month
CREATE TABLE events (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- create partitions (can be automated with pg_partman)
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

queries with a `WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31'` only scan `events_2026_01` instead of the full events table. query pruning is automatic.

old partitions (over 1 year old) can be archived to cheaper storage or dropped, without touching current data.

### sharding: when one PostgreSQL server isn't enough

sharding means splitting the database across multiple independent servers (shards). a shard key determines which shard a row lives in.

```
user_id % 4 = 0  → shard 0 (users 0, 4, 8, ...)
user_id % 4 = 1  → shard 1 (users 1, 5, 9, ...)
user_id % 4 = 2  → shard 2 (users 2, 6, 10, ...)
user_id % 4 = 3  → shard 3 (users 3, 7, 11, ...)
```

sharding is complex and should be a last resort. the problems:
- cross-shard queries (joining data from multiple shards) are slow or impossible
- transactions across shards require distributed transaction protocols (very hard)
- resharding (changing the number of shards) is painful

tools that help: **Citus** (PostgreSQL extension that shards transparently), **Vitess** (MySQL sharding, used by YouTube), **Amazon Aurora** (managed with automatic storage scaling).

the order of scaling operations for PostgreSQL:
1. **optimize queries** (indexes, EXPLAIN ANALYZE)
2. **add read replicas** (for read-heavy workloads)
3. **add connection pooling** (PgBouncer)
4. **upgrade hardware** (vertical scaling, larger instance, faster disk)
5. **table partitioning** (for time-series or very large tables)
6. **caching** (Redis in front of expensive queries)
7. **CQRS + separate read model** (denormalized for reads)
8. **sharding** (last resort, only when you've exhausted everything else)

---

## the N+1 query problem (again, but for SQL)

this isn't just a GraphQL problem. it happens everywhere:

```go
// N+1 in Go/SQL
users, _ := db.QueryAllUsers(ctx)  // 1 query: SELECT * FROM users

for _, user := range users {
    // N queries: one per user
    posts, _ := db.QueryPostsByUser(ctx, user.ID)  // SELECT * FROM posts WHERE author_id = ?
}
// total: 1 + N queries (where N = number of users)
```

fix: **eager loading with JOIN or IN clause**

```go
// approach 1: JOIN (good when you need full post data alongside user data)
rows, _ := db.QueryContext(ctx, `
    SELECT u.id, u.username, p.id as post_id, p.title
    FROM users u
    LEFT JOIN posts p ON p.author_id = u.id
`)

// approach 2: IN clause (good when N is reasonable)
userIDs := extractIDs(users)
posts, _ := db.QueryContext(ctx, `
    SELECT * FROM posts WHERE author_id = ANY($1)
`, pq.Array(userIDs))  // single query for all posts
```

the second approach is what DataLoader does for GraphQL, batch all IDs and fire one query.

---

*next: [08 Cryptography: The Rabbit Hole](08-cryptography.md)*

*sources: [Redis docs](https://redis.io/docs/) | [PostgreSQL scaling](https://www.velodb.io/glossary/ways-to-scale-postgresql) | [Redis 1M ops/sec](https://medium.com/insiderengineering/scaling-redis-to-1m-ops-sec-architecture-sharding-techniques-and-best-practices-0fb1d4e2946e)*
