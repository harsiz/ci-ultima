# 15 Testing: The Thing You Keep Skipping 🧪

### yes this chapter had to happen

---

nobody wants to write tests. I know. you know. the codebase knows.

but here's the thing: tests aren't for proving your code works *now*. they're for proving it still works in three months when someone (future you, probably) touches something adjacent and breaks it. the code you're scared to delete, the function you're scared to refactor, the module nobody dares touch? that's what happens when there are no tests. **untested code is code you're afraid of.**

this chapter isn't going to moralize at you. it's going to show you the practical stuff: what to test, what NOT to test, and how to do it in Go without losing your mind.

---

## the testing pyramid (and why most people do it wrong)

the classic model:

```
        /\
       /  \   E2E (few, slow, expensive)
      /----\
     /      \  Integration (medium)
    /--------\
   /          \ Unit (many, fast, cheap)
  /____________\
```

**unit tests** test one function in isolation. fast. no database, no network, no filesystem.

**integration tests** test multiple components together. your handler + your database layer + an actual postgres instance.

**E2E tests** drive the whole system like a real user. browser automation, full API flows, everything wired up.

most tutorials overindex on unit tests. most *real* backend applications actually benefit more from integration tests, because the interesting failures happen at the boundaries, not inside individual functions.

a pure business logic function that takes two numbers and returns a sum? unit test that. an HTTP handler that validates input, writes to postgres, publishes to Redis, and returns a response? test that with a real database and real Redis. mocking all of that out just means you're testing whether your mock works, not whether your code works.

---

## testing in Go: the basics

Go's standard library has a `testing` package. no install needed, no framework to learn.

```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}
```

```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d, want 5", result)
    }
}
```

run with `go test ./...` (the `...` means "all packages recursively"). that's it.

### table-driven tests: the Go way

instead of writing 10 separate test functions for 10 cases, Go devs use table-driven tests:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"zero", 0, 5, 5},
        {"both zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

`t.Run` creates subtests. `go test -run TestAdd/zero` runs just one case. this pattern scales really well when you have 20+ cases.

### what about assertions?

Go's standard library doesn't have `assert.Equal(t, expected, got)`. some people use [testify](https://github.com/stretchr/testify) for this:

```go
import "github.com/stretchr/testify/assert"

assert.Equal(t, 5, Add(2, 3))
assert.NoError(t, err)
assert.Contains(t, body, "expected text")
```

testify is fine. the stdlib-only crowd considers it unnecessary. honestly pick whichever you prefer and stay consistent.

---

## testing HTTP handlers

this is where backend testing actually lives. `net/http/httptest` is the tool:

```go
func TestCreateUser(t *testing.T) {
    // set up your router / handler
    mux := http.NewServeMux()
    mux.HandleFunc("POST /users", CreateUserHandler)

    body := `{"username": "harsiz", "email": "harsiz@example.com"}`
    req := httptest.NewRequest("POST", "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()
    mux.ServeHTTP(w, req)

    resp := w.Result()

    if resp.StatusCode != http.StatusCreated {
        t.Errorf("expected 201, got %d", resp.StatusCode)
    }

    var user UserResponse
    json.NewDecoder(resp.Body).Decode(&user)
    assert.Equal(t, "harsiz", user.Username)
}
```

`httptest.NewRecorder()` captures the response. `httptest.NewRequest()` builds the request. no actual TCP connection, super fast.

the question is: what do you do about the database? this is where opinions diverge.

---

## the mocking debate

**option 1: mock the database layer**

define an interface, implement a fake:

```go
type UserStore interface {
    CreateUser(ctx context.Context, u CreateUserParams) (User, error)
    GetUserByEmail(ctx context.Context, email string) (User, error)
}

// in tests
type MockUserStore struct {
    users map[string]User
}

func (m *MockUserStore) CreateUser(ctx context.Context, u CreateUserParams) (User, error) {
    user := User{ID: 1, Username: u.Username, Email: u.Email}
    m.users[u.Email] = user
    return user, nil
}
```

**pros:** fast. no setup. runs in CI with zero infrastructure.

**cons:** you're testing whether your mock behaves correctly, not whether your actual database queries work. if you write a wrong SQL query that fails on real postgres, the mock test passes happily.

**option 2: integration tests with a real database**

use a real postgres instance. GitHub Actions supports this natively with service containers (covered in chapter 04):

```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
```

your test spins up, runs migrations, inserts test data, runs the actual query, checks the result.

```go
func TestCreateUser_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    db := setupTestDB(t)  // connects to the test postgres
    store := NewUserStore(db)

    user, err := store.CreateUser(context.Background(), CreateUserParams{
        Username: "harsiz",
        Email:    "harsiz@example.com",
    })

    assert.NoError(t, err)
    assert.Equal(t, "harsiz", user.Username)
    assert.NotZero(t, user.ID)
}
```

`testing.Short()` + `t.Skip` lets you run `go test -short ./...` locally (fast, no database) and `go test ./...` in CI (full suite).

**the actual take:** use mocks for testing business logic that doesn't touch external systems. use real databases for testing anything involving SQL. the goal is catching real bugs, not achieving coverage numbers.

---

## test helpers and cleanup

a common pattern: `TestMain` for database setup, `t.Cleanup` for teardown:

```go
var testDB *pgxpool.Pool

func TestMain(m *testing.M) {
    // runs before any tests in the package
    pool, err := pgxpool.New(context.Background(), os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    testDB = pool
    runMigrations(pool)

    code := m.Run()  // run all tests

    pool.Close()
    os.Exit(code)
}

func TestCreateUser(t *testing.T) {
    t.Cleanup(func() {
        // clean up test data after this test
        testDB.Exec(context.Background(), "DELETE FROM users WHERE email LIKE 'test-%'")
    })

    // ... test code
}
```

or use `t.Parallel()` with isolated test schemas per test (more complex but faster in CI).

---

## benchmarks: measuring performance

Go's testing package also does benchmarks. same file, different function name:

```go
func BenchmarkHashPassword(b *testing.B) {
    password := "super-secret-password"

    b.ResetTimer()  // don't count setup time
    for i := 0; i < b.N; i++ {
        bcrypt.GenerateFromPassword([]byte(password), 12)
    }
}
```

run with `go test -bench=. -benchmem ./...`:

```
BenchmarkHashPassword-8     30    39,847,234 ns/op    8192 B/op    3 allocs/op
```

`ns/op` is nanoseconds per operation. `-benchmem` shows allocations. this is how you find slow code without guessing.

useful when: you're optimizing a hot path, comparing two implementations, or verifying a performance regression didn't creep in.

---

## what NOT to test (seriously)

this matters as much as what to test:

**don't test the framework.** if you're using Go's `http.ServeMux`, don't write a test that verifies routing works. the Go team already did that.

**don't test auto-generated code.** `sqlc` generates your database queries from SQL. the SQL is what matters, not the generated Go structs.

**don't test getters and setters.** a function that just returns a field value isn't worth a test.

**don't obsess over 100% coverage.** coverage is a measure of what was *executed*, not what was *correctly tested*. you can hit 100% coverage with completely useless tests. aim for coverage of the risky paths, not the obvious ones.

**don't mock things just to mock them.** if your test is 80% mock setup and 20% actual assertion, that's a sign you're testing implementation details, not behavior.

---

## the race detector

Go has a built-in race detector. run any test with `-race`:

```
go test -race ./...
```

this instruments your code to detect concurrent access to shared memory. a race condition that only shows up once every 10,000 runs will show up reliably with `-race`. it's slower (5-10x), but worth it for any code that uses goroutines.

always run `-race` in CI. it's caught embarrassing bugs in production-bound code more times than I can count.

---

## practical checklist

before you ship anything:

- **happy path tested:** the normal input → expected output
- **error paths tested:** what happens on invalid input, missing fields, DB errors
- **edge cases tested:** empty strings, zero values, nil pointers, duplicate entries
- **race detector clean:** `go test -race ./...` passes
- **integration test for DB queries:** at minimum, test that your SQL doesn't blow up on real data

and in your CI config (chapter 04 covered this), your pipeline should run all of this automatically. the whole point is you shouldn't have to remember to run tests manually.

---

*prev: [14 Python, Comparisons, and What It All Means](14-python-and-comparisons.md)*

*June 2026 | go.dev/doc/code#Testing | github.com/stretchr/testify*
