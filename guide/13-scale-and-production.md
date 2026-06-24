# 13 Scale and Production: Handling a Million Users 📊
### the things that only break at scale

---

"premature optimization is the root of all evil", Donald Knuth. this is absolutely true and absolutely misused as an excuse to ship systems with fundamental architectural problems.

the correct interpretation: don't optimize before you have data showing it's needed. the incorrect interpretation: don't think about scalability until your system is on fire at 3am.

this chapter is about architectural decisions that you make *early* which either enable scaling later or make it impossible without a full rewrite. it's also about the actual mechanics of what breaks at scale and how you fix it.

---

## the anatomy of a scalable system

before diving into techniques, understand what "scale" actually means. there are multiple dimensions:

- **throughput:** requests per second
- **latency:** time per request (p50, p95, p99, not just average)
- **data volume:** gigabytes/terabytes in your database
- **user count:** concurrent active users
- **geographic distribution:** users across multiple regions

these don't scale the same way. you might handle high throughput easily (your API is stateless, add more pods) but struggle with data volume (your table has 1 billion rows and queries are slow). different problems, different solutions.

---

## latency: what you're actually measuring

"average latency" is a lie. if 99% of requests take 10ms and 1% take 5000ms, your average might be 60ms, which sounds fine but 1% of your users are waiting 5 seconds.

**percentiles are the truth:**

```
p50 (median): 10ms , half of requests are faster than this
p75: 18ms
p90: 45ms
p95: 120ms , 95th percentile: 5% of requests are slower
p99: 800ms , "the tail": 1% of requests, often the worst UX
p99.9: 3000ms , one in a thousand is this slow
```

in a system with 1,000 RPS, your p99 affects 10 users per second. that's not a rare event, that's constant.

**tail latency amplification:** if your API calls three services, the probability of at least one being slow is much higher than any individual service being slow.

```
P(all fast) = P(A fast) × P(B fast) × P(C fast)
If each is fast 99% of the time:
P(all fast) = 0.99 × 0.99 × 0.99 = 0.970 = 97%
P(at least one slow) = 3%
```

a 1% tail per service becomes a 3% tail on the composed system. with ten services: 10% of requests hit at least one slow service. this is why microservices require careful latency budget management.

---

## caching: the highest leverage optimization

caching is storing the result of an expensive computation so it doesn't need to be computed again. it's the highest-leverage optimization in most systems.

**cache taxonomy:**

**Application cache (in-process):** data stored in memory within your running process.

```go
// simple in-memory LRU cache
import "github.com/hashicorp/golang-lru/v2"

type UserCache struct {
    cache *lru.Cache[int64, *User]
}

func NewUserCache(size int) *UserCache {
    c, _ := lru.New[int64, *User](size)
    return &UserCache{cache: c}
}

func (c *UserCache) Get(id int64) (*User, bool) {
    return c.cache.Get(id)
}

func (c *UserCache) Set(id int64, user *User) {
    c.cache.Add(id, user)
}
```

pros: zero network latency (memory access). cons: not shared between instances (each pod has its own cache), lost on restart, limited by process memory.

**Distributed cache (Redis):** covered in file 07. shared across all instances, survives restarts (with persistence), network latency (~1ms). right choice for most caching.

**CDN (Content Delivery Network):** caches static assets and API responses at edge nodes geographically close to users. sub-10ms latency for cached content. Cloudflare, Fastly, CloudFront.

for API responses that are the same for all users (public data, configuration, product catalogs), CDN caching is incredibly effective:

```go
func handleProductList(w http.ResponseWriter, r *http.Request) {
    // tell CDN to cache this response for 5 minutes
    w.Header().Set("Cache-Control", "public, max-age=300, stale-while-revalidate=60")
    // stale-while-revalidate: serve stale cache while fetching fresh in background
    
    // ... return product list
}
```

### cache invalidation: the hard problem

"there are only two hard things in computer science: cache invalidation and naming things.", Phil Karlton

when you update data, cached copies become stale. strategies:

**TTL (Time To Live):** cache expires after N seconds. simple, always eventually consistent, but you might serve stale data for up to TTL.

```go
// cache with 5-minute TTL
redis.Set(ctx, key, value, 5*time.Minute)
```

**write-through:** update cache and database simultaneously when data changes.

```go
func (s *UserService) UpdateUser(ctx context.Context, user *User) error {
    // update database first
    if err := s.db.UpdateUser(ctx, user); err != nil {
        return err
    }
    // then update cache
    s.cache.Set(user.ID, user, 5*time.Minute)
    return nil
}
```

**write-behind (write-back):** update cache immediately, write to database asynchronously. faster writes, risk of data loss if cache fails before the write.

**cache-aside (lazy loading):** on cache miss, load from database, populate cache. most common pattern, shown in file 07.

**cache invalidation by key pattern:** when updating a user, delete all related cache keys.

```go
func (s *UserService) UpdateUser(ctx context.Context, user *User) error {
    if err := s.db.UpdateUser(ctx, user); err != nil {
        return err
    }
    
    // invalidate all related cache keys
    keys := []string{
        fmt.Sprintf("user:%d", user.ID),
        fmt.Sprintf("user:email:%s", user.Email),
        fmt.Sprintf("user:%d:posts", user.ID),  // cached post list might reference user
    }
    s.redis.Del(ctx, keys...)
    return nil
}
```

---

## queues and async processing: decoupling work

not everything needs to happen in the HTTP request handler. sending emails, generating reports, processing images, charging subscriptions, these can happen asynchronously.

**the naive approach (don't do this):**

```go
func handleUserRegistration(w http.ResponseWriter, r *http.Request) {
    user := createUser(...)
    sendWelcomeEmail(user)  // takes 500ms, blocks the response
    // user is waiting for a response while email sends
}
```

**the right approach:**

```go
func handleUserRegistration(w http.ResponseWriter, r *http.Request) {
    user := createUser(...)
    
    // enqueue email job, return immediately
    queue.Enqueue(ctx, EmailJob{
        Type:   "welcome",
        UserID: user.ID,
    })
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
    // response sent in <10ms, email sent by a separate worker
}

// separate worker process
func emailWorker(ctx context.Context, queue JobQueue) {
    for {
        job, err := queue.Dequeue(ctx)
        if err != nil {
            // handle error, maybe retry
            continue
        }
        
        user, _ := db.GetUser(ctx, job.UserID)
        sendEmail(user.Email, "Welcome to our app!")
        queue.Complete(ctx, job.ID)
    }
}
```

### message queues and job queues

**Redis-based job queues:**

- **Asynq** (Go): Redis-backed job queue with scheduling, retries, priorities, and a web UI. commonly used in Go applications.
- **River** (Go): PostgreSQL-backed job queue. uses the database you already have, ACID guarantees for job state.

```go
// using Asynq
import "github.com/hibiken/asynq"

// enqueue
client := asynq.NewClient(asynq.RedisClientOpt{Addr: redisAddr})
task := asynq.NewTask("email:welcome", payload)
info, err := client.Enqueue(task,
    asynq.MaxRetry(3),
    asynq.Timeout(30 * time.Second),
    asynq.Delay(5 * time.Second),     // wait 5s before processing
)

// worker
srv := asynq.NewServer(asynq.RedisClientOpt{Addr: redisAddr}, asynq.Config{
    Concurrency: 10,  // 10 concurrent workers
})
mux := asynq.NewServeMux()
mux.HandleFunc("email:welcome", handleWelcomeEmail)
srv.Run(mux)
```

**message queues (distributed, durable):**

- **RabbitMQ:** traditional AMQP message broker. routing, fanout, dead letter queues. mature ecosystem.
- **Apache Kafka:** distributed log for high-throughput event streaming. not a traditional queue, consumers read from offsets, messages are retained. use for event sourcing, data pipelines, audit logs, microservice event bus.
- **AWS SQS / Google Cloud Pub/Sub:** managed queues, no ops burden.
- **NATS:** lightweight, fast messaging. `nats.go` is the Go client.

**when to use what:**
- simple job queue (email, report generation): Asynq or River (Redis or PostgreSQL-backed)
- event streaming / microservices communication: NATS or Kafka
- complex routing rules, fanout: RabbitMQ
- managed, zero ops: SQS

---

## rate limiting at scale

rate limiting protects your service from abuse and ensures fair resource usage. we covered the basic implementation in file 09. at scale, it's more nuanced.

**rate limiting algorithms:**

**Token bucket:** a bucket starts full of N tokens. each request consumes one token. tokens refill at R per second. allows bursting up to N.

**Sliding window:** count requests in the last N seconds using a sorted set in Redis. most accurate but more Redis operations.

**Fixed window:** count requests in the current 1-minute window. simple but susceptible to boundary attacks (2x the limit at a window boundary).

**the Golang rate package:**

```go
import "golang.org/x/time/rate"

// global rate limiter: 100 requests/second, burst of 200
limiter := rate.NewLimiter(rate.Limit(100), 200)

func handler(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
        w.Header().Set("Retry-After", "1")
        writeError(w, http.StatusTooManyRequests, "rate limit exceeded")
        return
    }
    // ...
}
```

for distributed rate limiting (across multiple pods), use Redis:

```go
// distributed rate limiting with Redis sliding window
// return true if request is allowed
func (rl *RedisRateLimiter) Allow(ctx context.Context, key string, limit int, window time.Duration) (bool, error) {
    now := time.Now().UnixNano()
    windowStart := now - window.Nanoseconds()
    
    pipe := rl.redis.TxPipeline()
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))
    count := pipe.ZCard(ctx, key)
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})
    pipe.Expire(ctx, key, window)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    return count.Val() < int64(limit), nil
}
```

**rate limit by:**
- IP address (basic DDoS protection)
- user ID (fair use per authenticated user)
- API key (business tiers: free=100/min, pro=1000/min)
- endpoint (expensive endpoints get stricter limits)

---

## load balancing: distributing traffic

a **load balancer** distributes incoming requests across multiple server instances. it's what makes horizontal scaling possible.

```
Clients
   │
   ▼
[Load Balancer]      ← nginx, HAProxy, AWS ALB
   │ │ │
   ▼ ▼ ▼
[Server 1] [Server 2] [Server 3]   ← your Go services
```

**load balancing algorithms:**

**Round robin:** requests distributed in order. server 1, server 2, server 3, server 1... simple and works when all requests are similar.

**Least connections:** route to the server with fewest active connections. better for variable-duration requests (some requests take 100ms, some take 5s).

**IP hash:** same client IP always routes to the same server. useful for stateful sessions (but proper session design shouldn't need this).

**Weighted:** distribute based on server capacity. a 4-core server gets 2x the traffic of a 2-core server.

Kubernetes services do round-robin load balancing between pods by default. for more sophisticated balancing, service meshes like Istio or Linkerd provide per-request load balancing with circuit breaking.

---

## circuit breakers: failing gracefully

when service B is down or slow, requests to service A that depend on B pile up, filling goroutine pools and exhausting resources. **cascade failure**: one service's failure brings down everything that depends on it.

a **circuit breaker** monitors failure rate. when failures exceed a threshold, it "opens" the circuit and fails fast (returns an error immediately without making the downstream call). after a timeout, it tries again ("half-open"). if successful, it closes.

```go
import "github.com/sony/gobreaker"

var cb = gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 3,         // allow 3 requests when half-open
    Interval:    30 * time.Second,
    Timeout:     10 * time.Second, // open for 10s before half-open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 3 && failureRatio >= 0.6
    },
})

func callPaymentService(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) {
    result, err := cb.Execute(func() (interface{}, error) {
        return paymentClient.Charge(ctx, req)
    })
    if err != nil {
        if err == gobreaker.ErrOpenState {
            return nil, ErrPaymentServiceUnavailable  // fail fast, don't call downstream
        }
        return nil, err
    }
    return result.(*PaymentResponse), nil
}
```

---

## database connection pooling at scale

each PostgreSQL connection is a process (~5-10MB RAM). with 50 pods × 10 connections per pod = 500 connections. PostgreSQL starts degrading above a few hundred connections.

**PgBouncer** (covered in file 06) is the standard solution. but at real scale, you need to think about this more carefully.

```
50 pods × 10 connections = 500 to PgBouncer
PgBouncer × 20 server connections = 20 to PostgreSQL
```

in transaction mode, PgBouncer multiplexes efficiently. 500 application connections share 20 database connections. works well for typical transactional workloads.

**pgbouncer.ini:**
```ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
server_reset_query = DISCARD ALL
```

---

## observability: knowing what's happening

you can't optimize what you can't measure. **observability** is the ability to understand what your system is doing from its outputs.

the three pillars:

**Metrics:** numerical measurements over time. CPU usage, request rate, error rate, latency percentiles, queue depth. tools: Prometheus (collect) + Grafana (visualize).

```go
import "github.com/prometheus/client_golang/prometheus"
import "github.com/prometheus/client_golang/prometheus/promhttp"

var (
    httpDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "Duration of HTTP requests",
        Buckets: []float64{0.001, 0.01, 0.05, 0.1, 0.5, 1, 2, 5},
    }, []string{"method", "path", "status"})
)

func init() {
    prometheus.MustRegister(httpDuration)
}

// metrics middleware
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, status: 200}
        
        next.ServeHTTP(rw, r)
        
        httpDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
            strconv.Itoa(rw.status),
        ).Observe(time.Since(start).Seconds())
    })
}

// expose metrics endpoint
http.Handle("/metrics", promhttp.Handler())
```

**Logs:** timestamped records of events. structured logs (JSON) are searchable. tools: `log/slog` (Go standard library since 1.21), Loki for aggregation.

```go
// structured logging with slog
slog.Info("user logged in",
    "user_id", userID,
    "ip", r.RemoteAddr,
    "method", r.Method,
    "path", r.URL.Path,
    "duration_ms", time.Since(start).Milliseconds(),
)

// output as JSON (in production):
// {"time":"2026-06-24T10:00:00Z","level":"INFO","msg":"user logged in","user_id":123,"ip":"..."}
```

**Traces:** distributed request tracing across services. tracks a request as it flows through multiple services, shows where time is spent. tools: OpenTelemetry (instrumentation) + Jaeger/Tempo (storage + visualization).

```go
import "go.opentelemetry.io/otel"

func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("myapp").Start(r.Context(), "handleRequest")
    defer span.End()
    
 // pass ctx down, spans are nested
    user, err := s.userService.GetUser(ctx, id)
    // the GetUser function creates a child span, showing time in that operation
}
```

a distributed trace shows: 
- HTTP handler: 150ms total
  - database query: 120ms (!!!)
    - index scan: 5ms
    - data fetch: 115ms (missing index on this column)
  - cache lookup: 2ms
  - response encoding: 8ms

you can see immediately that the database query is the bottleneck and drill into which specific query.

---

## the production readiness checklist

before "ready for production" means what it should:

```
Correctness:
☐ unit tests with >80% coverage
☐ integration tests against real dependencies (real PostgreSQL, real Redis)
☐ race condition testing (go test -race)
☐ load testing (k6, hey, wrk)

Observability:
☐ structured logging with request IDs
☐ Prometheus metrics for latency, error rate, throughput
☐ distributed tracing for cross-service requests
☐ alerts on error rate, latency p99, queue depth

Reliability:
☐ health + readiness endpoints
☐ graceful shutdown (handle SIGTERM, drain in-flight requests)
☐ circuit breakers for downstream services
☐ timeouts on all outbound calls (HTTP, database, cache)
☐ retries with exponential backoff on transient failures

Security:
☐ no secrets in environment variables without external secrets management
☐ TLS everywhere (database connections too)
☐ security headers
☐ input validation at boundaries
☐ dependency vulnerability scanning (govulncheck)
☐ RBAC and least privilege in Kubernetes

Operations:
☐ runbook for common issues
☐ on-call rotation documented
☐ rollback procedure tested
☐ database backup and restore tested (actually restore it, don't just backup)
```

**graceful shutdown in Go:**

```go
func main() {
    server := &http.Server{Addr: ":8080", Handler: handler}
    
    // listen for shutdown signal in a goroutine
    go func() {
        quit := make(chan os.Signal, 1)
        signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
        <-quit
        
        // give in-flight requests up to 30s to complete
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        
        if err := server.Shutdown(ctx); err != nil {
            log.Printf("server shutdown error: %v", err)
        }
    }()
    
    log.Println("server starting")
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("server error: %v", err)
    }
    log.Println("server stopped gracefully")
}
```

Kubernetes sends SIGTERM when it wants to terminate a pod. without graceful shutdown, in-flight requests are dropped. with it, new requests stop and current ones complete.

---

*next: [14 Python, Comparisons, and Final Thoughts](14-python-and-comparisons.md)*

*sources: [Redis scaling 1M ops/sec](https://medium.com/insiderengineering/scaling-redis-to-1m-ops-sec-architecture-sharding-techniques-and-best-practices-0fb1d4e2946e) | [PostgreSQL scaling strategies](https://www.velodb.io/glossary/ways-to-scale-postgresql) | [OpenTelemetry Go](https://opentelemetry.io/docs/instrumentation/go/)*
