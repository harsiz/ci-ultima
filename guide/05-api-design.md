# 05 API Design: REST, GraphQL, and HTTP Going Deep 🌐
### building interfaces that don't make frontend devs cry

---

an API (Application Programming Interface) is how your backend talks to the outside world. it's the contract. it's the thing your frontend team will complain about at 2am. it's the thing clients will call with outdated SDKs for years after you've redesigned it internally.

getting API design right isn't just about technical correctness. it's about designing a thing that's intuitive to use, versioned thoughtfully, and doesn't paint you into a corner six months from now.

---

## HTTP: the foundation everything is built on

before REST vs GraphQL, you need to understand HTTP properly because both are built on top of it.

**HTTP (HyperText Transfer Protocol)** is a stateless request-response protocol. a client sends a request, a server sends a response. no persistent connection (in HTTP/1.1; HTTP/2 and HTTP/3 changed this somewhat).

### the anatomy of an HTTP request

```
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Accept: application/json
User-Agent: MyApp/1.0
X-Request-ID: 7f4c8a2b-3d91-4e20-9c1a-5b6e7f8d9012

{
  "username": "moe",
  "email": "moe@example.com",
  "password": "supersecret123"
}
```

**request line:** method + path + HTTP version
**headers:** key-value metadata about the request
**body:** optional payload (not all methods have one)

### HTTP methods: what they actually mean

| Method | Meaning | Body | Safe | Idempotent |
|--------|---------|------|------|------------|
| GET | retrieve resource | no | yes | yes |
| POST | create resource / action | yes | no | no |
| PUT | replace resource entirely | yes | no | yes |
| PATCH | partially update resource | yes | no | no (usually) |
| DELETE | remove resource | rarely | no | yes |
| HEAD | like GET but no body | no | yes | yes |
| OPTIONS | what methods are allowed | no | yes | yes |

**safe:** doesn't change server state (GET, HEAD, OPTIONS). browsers will freely retry safe requests.

**idempotent:** calling it multiple times has the same effect as calling it once. DELETE is idempotent: deleting something that's already deleted is a no-op (returns 404). PUT is idempotent: sending the same PUT twice results in the same final state.

POST is neither safe nor idempotent, submitting the same form twice creates two resources (or sends two emails, or charges twice). this is why credit card forms disable the submit button on click.

### HTTP status codes: use them correctly

most APIs use like 200 and 400 and call it a day. please don't.

```
1xx - Informational
  100 Continue

2xx - Success
  200 OK
  201 Created        ← use this when POST creates a new resource
  202 Accepted       ← request received, processing asynchronously
  204 No Content     ← success but no body (DELETE, some PUTs)

3xx - Redirection
  301 Moved Permanently
  302 Found (temporary redirect)
  304 Not Modified   ← ETag cache validation

4xx - Client Errors
  400 Bad Request    ← malformed request, validation errors
  401 Unauthorized   ← not authenticated (name is misleading)
  403 Forbidden      ← authenticated but not authorized
  404 Not Found
  405 Method Not Allowed
  409 Conflict       ← duplicate resource, version conflict
  410 Gone           ← resource permanently deleted
  422 Unprocessable Entity ← business logic validation failure
  429 Too Many Requests    ← rate limiting

5xx - Server Errors
  500 Internal Server Error ← something broke on your end
  502 Bad Gateway           ← upstream service failed
  503 Service Unavailable   ← intentional (maintenance, overload)
  504 Gateway Timeout       ← upstream service too slow
```

the 401 vs 403 distinction matters: **401 = not logged in. 403 = logged in but not allowed.** returning 401 when you mean 403 sends clients into authentication loops for no reason.

---

## REST: the architecture that won (mostly)

REST (Representational State Transfer) is an architectural style, not a protocol or a standard. Roy Fielding defined it in his 2000 PhD dissertation and the software industry spent the next 20 years arguing about what it means.

the practical version: **REST is about treating your data as resources accessible through URLs, manipulated with HTTP methods, represented as JSON (usually).**

### REST resource design

good REST API URLs are **nouns, not verbs:**

```
BAD:
GET  /getUser?id=123
POST /createUser
POST /deleteUser
POST /updateUserEmail

GOOD:
GET    /users/123
POST   /users
DELETE /users/123
PATCH  /users/123
```

nested resources for relationships:

```
GET  /users/123/posts           # all posts by user 123
GET  /users/123/posts/456       # specific post
POST /users/123/posts           # create post for user 123
DELETE /users/123/posts/456     # delete specific post
```

don't go more than 2-3 levels deep. `/organizations/1/teams/2/members/3/permissions/4` is a sign you need to rethink your design.

actions that don't fit the CRUD model:

```
POST /users/123/verify-email    # trigger email verification
POST /users/123/deactivate      # deactivate account
POST /payments/123/refund       # issue a refund
POST /orders/456/cancel         # cancel an order
```

using POST for actions is fine. the "everything must be pure CRUD" interpretation of REST is too rigid.

### versioning: the thing you'll regret if you skip it

```
/api/v1/users/123    ← version in path (most common, most explicit)
/api/users/123       ← no version (terrible for long-lived APIs)
```

header versioning:
```
GET /api/users/123
Accept: application/vnd.myapp.v2+json
```

header versioning is "purer" REST but harder to use (can't just paste URL in browser). path versioning is what most APIs actually use.

version when you make breaking changes:
- removing fields from responses
- changing field types
- changing URL structure
- changing required/optional status of request fields

additive changes (new optional fields, new endpoints) don't require a version bump. clients that don't know about new fields ignore them.

### a properly designed Go REST handler

```go
// internal/handlers/users.go
package handlers

import (
    "encoding/json"
    "errors"
    "net/http"
    
    "github.com/myapp/internal/domain"
    "github.com/myapp/internal/service"
)

type UserHandler struct {
    users service.UserService
}

func NewUserHandler(users service.UserService) *UserHandler {
    return &UserHandler{users: users}
}

type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3,max=32"`
    Email    string `json:"email"    validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

type UserResponse struct {
    ID        int64  `json:"id"`
    Username  string `json:"username"`
    Email     string `json:"email"`
    CreatedAt string `json:"created_at"`
 // NOTE: no Password field, never return passwords or hashes
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }
    
    user, err := h.users.Create(r.Context(), req.Username, req.Email, req.Password)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrDuplicateEmail):
            writeError(w, http.StatusConflict, "email already in use")
        case errors.Is(err, domain.ErrInvalidInput):
            writeError(w, http.StatusUnprocessableEntity, err.Error())
        default:
            writeError(w, http.StatusInternalServerError, "internal error")
        }
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)    // 201, not 200
    json.NewEncoder(w).Encode(UserResponse{
        ID:        user.ID,
        Username:  user.Username,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Format(time.RFC3339),
    })
}

func writeError(w http.ResponseWriter, status int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]string{
        "error": message,
    })
}
```

### REST response design: the boring stuff that matters

always return consistent error shapes:

```json
{
  "error": "email already in use",
  "code": "DUPLICATE_EMAIL",
  "field": "email"
}
```

the `code` field (machine-readable string) lets clients handle specific error cases without parsing error message strings. message strings can change (localization, wording improvements). code strings should be stable.

for lists, always include pagination metadata:

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total": 847,
    "total_pages": 43,
    "has_next": true,
    "has_prev": true
  }
}
```

or cursor-based pagination (better for real-time data):

```json
{
  "data": [...],
  "cursor": {
    "next": "eyJpZCI6MTIzfQ==",  // base64 encoded cursor
    "prev": "eyJpZCI6MTAzfQ=="
  }
}
```

cursor pagination doesn't have the "page 5 shifts when new items are inserted" problem of offset pagination.

---

## GraphQL: when REST isn't enough

REST returns what the server decided you need. GraphQL returns exactly what you asked for.

```graphql
# the client asks for exactly this
query GetUser($id: ID!) {
  user(id: $id) {
    id
    username
    email
    posts(first: 5) {
      title
      publishedAt
      commentCount
    }
    followers {
      count
    }
  }
}
```

with REST, getting this data requires:
- `GET /users/123`
- `GET /users/123/posts?limit=5`
- `GET /users/123/followers/count`

three requests. with GraphQL: one request.

this is GraphQL's core value proposition: **no over-fetching, no under-fetching, no N+1 request waterfalls.**

### GraphQL schema definition

the schema is the contract. it defines every type, query, mutation, and subscription:

```graphql
type User {
  id: ID!
  username: String!
  email: String!
  createdAt: DateTime!
  posts(first: Int, after: String): PostConnection!
  followers: FollowerStats!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  publishedAt: DateTime
}

type Query {
  user(id: ID!): User
  post(id: ID!): Post
  me: User  # current authenticated user
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  createPost(input: CreatePostInput!): CreatePostPayload!
}

input CreateUserInput {
  username: String!
  email: String!
  password: String!
}
```

the `!` means non-nullable. `String!` = required string. `String` = optional string. `[Post!]!` = non-nullable list of non-nullable Posts.

### the N+1 problem in GraphQL

GraphQL has a notorious problem: the N+1 query.

```graphql
query {
  posts {    # fetches 10 posts: 1 query
    title
    author {   # fetches author for each post: 10 queries!
      username  # total: 11 queries for 10 posts
    }
  }
}
```

the naive resolver for `author` makes one DB query per post. 10 posts = 10 queries = N+1.

the fix is **DataLoader** (batching + caching). DataLoader collects all the `author` field requests within a single request cycle, batches them into one `SELECT * FROM users WHERE id IN (1,2,3,4,5,6,7,8,9,10)`, and returns the results.

in Go, `graph-gophers/dataloader` implements this pattern:

```go
// load is called per-field, but batching happens automatically
func (r *postResolver) Author(ctx context.Context) (*User, error) {
    return r.loaders.UserByID.Load(ctx, r.post.AuthorID)
}
```

the DataLoader batches all `UserByID.Load` calls within the same request into a single DB query. this is *required* for production GraphQL. without DataLoader, GraphQL will absolutely destroy your database.

### REST vs GraphQL: the actual decision

in 2025-2026, the industry consensus has settled: **both are right for different contexts.** many companies use both.

| Use REST when | Use GraphQL when |
|---------------|-----------------|
| public API others will build clients for | client-driven, complex data requirements |
| simple CRUD, well-defined resource model | multiple different clients (mobile, web, partners) with different data needs |
| strong HTTP caching is important | under-fetching is causing performance problems |
| small team, shipping fast | team has bandwidth to maintain schema |
| microservices talking to each other internally | product has a rich, interconnected data model |

**the practical advice:** start with REST. if you find yourself building lots of specialized "aggregation" endpoints to avoid waterfalls (like `/users/123/dashboard-data`), that's a sign GraphQL would serve you better. or use it alongside, REST for simple services, GraphQL for the complex data layer.

---

## gRPC: the one people forget about

there's a third option: **gRPC** (Google Remote Procedure Call). it uses Protocol Buffers (protobuf) for serialization and HTTP/2 for transport.

```protobuf
// users.proto
syntax = "proto3";
package users;

service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream GetUserResponse);
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
}

message CreateUserResponse {
  User user = 1;
}

message User {
  int64 id = 1;
  string username = 2;
  string email = 3;
}
```

you define the schema in `.proto` files, run `protoc` (the protobuf compiler), and get generated Go (and JavaScript, Python, Java, etc.) code with fully typed client and server stubs.

**why gRPC is good:**
- protobuf is much smaller and faster to parse than JSON (binary format)
- streaming support (bidirectional streaming, server-side streaming)
- strongly typed generated clients, call a remote service like a local function
- excellent for service-to-service communication inside a microservice architecture

**why gRPC is bad:**
- not human-readable (binary format, you can't curl it)
- browser support is awkward (grpc-web is a workaround, grpc-gateway converts to REST)
- harder to debug
- overkill for simple external APIs

the sweet spot: **gRPC for internal service communication, REST or GraphQL for external/browser APIs.**

---

## middleware in Go HTTP: the composable pattern

middleware is code that wraps your handlers to add cross-cutting concerns: authentication, logging, rate limiting, CORS, compression, request tracing.

```go
// middleware signature: takes a handler, returns a handler
type Middleware func(http.Handler) http.Handler

// logging middleware
func Logging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            // wrap the response writer to capture status code
            wrapped := &responseWriter{ResponseWriter: w, status: 200}
            
            next.ServeHTTP(wrapped, r)
            
            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "status", wrapped.status,
                "duration", time.Since(start),
                "request_id", r.Header.Get("X-Request-ID"),
            )
        })
    }
}

// auth middleware
func RequireAuth(jwtSecret []byte) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("Authorization")
            if token == "" || !strings.HasPrefix(token, "Bearer ") {
                writeError(w, http.StatusUnauthorized, "missing token")
                return
            }
            
            claims, err := validateJWT(token[7:], jwtSecret)
            if err != nil {
                writeError(w, http.StatusUnauthorized, "invalid token")
                return
            }
            
            // add user info to request context
            ctx := context.WithValue(r.Context(), contextKeyUser, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// chaining middleware
func Chain(middlewares ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}

// usage
mux := http.NewServeMux()
mux.HandleFunc("GET /users", userHandler.List)

handler := Chain(
    Logging(logger),
    RequireAuth(jwtSecret),
    RateLimit(100),        // 100 requests/second
    CORS(allowedOrigins),
)(mux)

http.ListenAndServe(":8080", handler)
```

this pattern (middleware as `func(http.Handler) http.Handler`) composes cleanly and is how every major Go HTTP framework works internally.

---

## the `context.Context`: the thing you must understand

every handler receives a `*http.Request`. the request has a `.Context()`. this context carries:
- request deadline/timeout
- cancellation signal (when client disconnects)
- request-scoped values (user ID, request ID, etc.)

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // get the context
    
    id := r.PathValue("id")
    
    // pass context to database queries
    // if client disconnects mid-request, ctx.Done() fires
    // database/sql will cancel the query
    user, err := h.db.QueryUserByID(ctx, id)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // client disconnected, just return
            return
        }
        writeError(w, 500, "db error")
        return
    }
    
    // ...
}
```

passing context through your call stack is mandatory in Go. every function that touches I/O (database, HTTP, cache) should accept `ctx context.Context` as its first argument. this enables:
- **cancellation:** when the HTTP request is cancelled (client closed connection, server shutting down), all downstream I/O is cancelled automatically
- **deadlines:** set a timeout on the whole request, and it propagates through all downstream calls
- **tracing:** inject a trace ID into context and extract it in database queries, HTTP clients, etc. for distributed tracing

```go
// set a timeout for the whole handler
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    result, err := h.service.DoSlowThing(ctx)  // will cancel after 5s
    // ...
}
```

---

*next: [06 SQL and Relational Databases](06-databases-sql.md)*

*sources: [REST fielding dissertation](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) | [GraphQL vs REST AWS](https://aws.amazon.com/compare/the-difference-between-graphql-and-rest/) | [gRPC docs](https://grpc.io/docs/)*
