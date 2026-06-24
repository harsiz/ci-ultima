# 06 SQL and Relational Databases: The Stuff That Actually Stores Your Data 🗄️
### because "just use a database" isn't advice

---

databases are where most application bugs hide in production. your code might be perfect. your API design might be beautiful. but if your queries are wrong, your schema is unindexed, or your transactions are incorrect, users lose data and you lose sleep.

this chapter is about SQL databases properly, not just "write a SELECT statement" but understanding *why* things work the way they do, so you can design schemas that scale and write queries that don't make your DBA want to quit.

---

## why relational databases won (and still win)

the relational model was invented by Edgar Codd at IBM in 1970. the core insight: organize data into **relations** (tables) with strict schemas, and use **set operations** (queries) to retrieve data. relate tables using **foreign keys** rather than embedding data.

this model has won in most production environments for 50+ years because:

**ACID guarantees:**
- **A**tomicity: transactions are all-or-nothing. if any part fails, the whole transaction rolls back. no partial updates.
- **C**onsistency: the database always moves from one valid state to another. foreign key constraints, unique constraints, check constraints are enforced.
- **I**solation: concurrent transactions don't see each other's incomplete changes.
- **D**urability: once committed, data survives crashes. it's on disk (and usually replicated).

these guarantees are *deeply* valuable for anything involving money, user data, or any state that matters. when you ask "did that bank transfer actually complete?", ACID is why you can trust the answer.

NoSQL databases (MongoDB, DynamoDB, Cassandra) often trade some ACID guarantees for scalability or flexibility. we'll cover those tradeoffs in file 07.

---

## PostgreSQL: the one you should actually use

**PostgreSQL** is the open-source relational database that's been quietly becoming the dominant choice for new projects over the past decade. some real stats:

- ranked #4 in DB-Engines ranking as of 2026, growing consistently
- Stack Overflow developer survey: most-loved database for multiple years running
- used at: Apple, Netflix, Reddit, Spotify, Instagram (at massive scale), GitHub, Airbnb

why PostgreSQL over MySQL:

- **better SQL standards compliance:** PostgreSQL implements more of the SQL standard properly. MySQL has historical quirks (like `GROUP BY` not requiring all non-aggregate columns, which technically should be an error).
- **richer data types:** native JSON/JSONB, arrays, hstore (key-value), UUID, ranges, custom types. MySQL has added some of these but PostgreSQL did them better, earlier.
- **better concurrency:** MVCC (Multi-Version Concurrency Control) implementation is more complete. readers never block writers, writers never block readers.
- **extensions:** PostGIS (geospatial), pgvector (vector embeddings for AI), pg_trgm (fuzzy search), timescaledb (time-series). the extension ecosystem is extraordinary.
- **JSON operators:** PostgreSQL's JSONB with GIN indexes enables surprisingly efficient semi-structured data queries without going full NoSQL.

```sql
-- PostgreSQL JSONB example
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- GIN index on all JSONB keys
CREATE INDEX idx_events_data ON events USING GIN (data);

-- query into JSONB
SELECT * FROM events 
WHERE data @> '{"user_id": 123}'  -- contains this JSON
  AND data ->> 'action' = 'purchase';
```

---

## SQLite: when simple is smart

SQLite is fundamentally different from PostgreSQL. it's not a server, it's a library that's part of your application. the entire database is a single file.

```
PostgreSQL:  your app → network → PostgreSQL server process → files on disk
SQLite:      your app → SQLite library (same process) → single .db file
```

SQLite is the most deployed database in the world. it's in every Android and iOS device, every browser, almost every desktop application. it's in Apple's Messages app, Firefox's browser storage, Adobe products, Skype.

**when SQLite is the right choice:**

- **local applications:** desktop apps, mobile apps, CLI tools that need to store structured data
- **test databases:** in your CI pipeline, SQLite is perfect for integration tests, no server to spin up, no cleanup needed, runs in-memory if you want (`database/url=:memory:`)
- **embedded analytics:** storing telemetry or logs locally before sending them to a central store
- **low-concurrency web services:** SQLite in WAL (Write-Ahead Logging) mode handles moderate traffic. [Litestream](https://litestream.io/) can replicate it to S3 for disaster recovery.

**when SQLite is the wrong choice:**

- multiple servers writing to the same database (SQLite has no server, you can't share a file across machines cleanly)
- high write concurrency (SQLite supports only one writer at a time)
- complex queries on large datasets (PostgreSQL's query planner is significantly more sophisticated)

**Go + SQLite:**

```go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"  // SQLite driver (requires CGO)
    // OR
    _ "modernc.org/sqlite"           // pure Go SQLite (no CGO needed)
)

db, err := sql.Open("sqlite", "./myapp.db?_journal_mode=WAL&_foreign_keys=on")
```

`_journal_mode=WAL` is essentially always what you want for SQLite in applications, enables concurrent reads while writes happen.
`_foreign_keys=on` enables foreign key enforcement (it's off by default in SQLite for backward compatibility reasons, which is a footgun).

---

## schema design: the decisions you'll live with forever

schema design is the most consequential part of backend development. it's hard to change later because:
- migrations on large tables are slow and risky
- you've already shipped an API based on the schema
- clients have hardcoded assumptions about field names and types

### normalization (and when to break it)

**1NF (First Normal Form):** each column contains atomic values. no arrays in cells, no repeating groups.

```sql
-- BAD (violates 1NF)
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    product_ids TEXT  -- "1,2,3,4" stored as a string
);

-- GOOD
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, ...);
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT NOT NULL
);
```

**3NF (Third Normal Form):** every non-key column depends only on the primary key, not on other non-key columns. this eliminates update anomalies.

in practice: normalize until it hurts, then denormalize where performance requires it. normalized schemas are correct and maintainable. denormalization is a *deliberate* performance optimization, not a design style.

### primary keys: the UUID debate

```sql
-- option 1: BIGSERIAL (auto-increment integer)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    ...
);
-- fast inserts (sequential writes to btree index)
-- reveals information (IDs are guessable: user 1, 2, 3...)
-- can't generate client-side

-- option 2: UUID (universally unique)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
);
-- no information leakage
-- can generate client-side (offline-first apps)
-- random UUIDs cause index fragmentation (inserts spread across btree)

-- option 3: UUIDv7 (time-ordered UUID) ← 2025 recommendation
-- sequential enough for index performance + globally unique
-- UUID v7 includes a timestamp prefix, so inserts are roughly sequential
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- use UUIDv7 function
    ...
);
```

UUIDv7 is the best of both worlds. PostgreSQL 17+ has `uuidv7()` built in. before that, you use an extension or generate it in application code.

```go
// go-uuid library supports v7
import "github.com/google/uuid"

id, _ := uuid.NewV7()
```

### foreign keys: enforce your data integrity

```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    author_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    -- ON DELETE CASCADE: when user is deleted, their posts are deleted too
    -- ON DELETE RESTRICT: prevents deleting user if they have posts (default)
    -- ON DELETE SET NULL: sets author_id to NULL when user deleted
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`ON DELETE CASCADE` sounds convenient but use it carefully. cascading deletes can remove more data than you intended if your schema has deep relationships.

`ON DELETE RESTRICT` (or `ON DELETE NO ACTION`) is safer, it forces you to explicitly clean up child records before deleting the parent. more manual work, but you won't accidentally nuke data.

### timestamps: always use TIMESTAMPTZ

```sql
-- BAD
created_at TIMESTAMP   -- stores timestamp without timezone info

-- GOOD
created_at TIMESTAMPTZ  -- stores timestamp WITH timezone (UTC internally)
-- full name: TIMESTAMP WITH TIME ZONE
```

`TIMESTAMPTZ` stores dates as UTC internally and converts to the client's timezone when displaying. `TIMESTAMP` stores raw values with no timezone info, if your server changes timezone settings, all your timestamps are wrong. always use `TIMESTAMPTZ`.

---

## indexes: the most impactful performance optimization

an index is a data structure that lets the database find rows without scanning the whole table.

```sql
-- without index: O(n) scan through 10 million rows
SELECT * FROM users WHERE email = 'moe@example.com';

-- with index: O(log n) lookup in btree
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'moe@example.com';  -- now fast
```

**B-Tree indexes** (default in PostgreSQL): great for equality (`=`), range queries (`<`, `>`, `BETWEEN`), sorting (`ORDER BY`). most indexes you'll create are B-Trees.

**GIN indexes:** for full-text search, JSONB containment queries, array operations. slower to update than B-Tree but faster for contains/overlap queries.

**GiST indexes:** for geometric data, IP ranges, full-text search (alternative to GIN for some cases).

**BRIN indexes:** for naturally ordered data (like timestamps in a time-series table). tiny and fast for sequential scans.

### what to index:

```sql
-- definitely index these:
CREATE UNIQUE INDEX idx_users_email ON users(email);      -- lookup + uniqueness
CREATE INDEX idx_posts_author_id ON posts(author_id);     -- foreign key (always!)
CREATE INDEX idx_orders_created_at ON orders(created_at); -- time-range queries

-- compound index: useful when you frequently filter on both columns together
CREATE INDEX idx_posts_author_status ON posts(author_id, status);
-- this index supports: WHERE author_id = X, WHERE author_id = X AND status = Y
-- does NOT efficiently support: WHERE status = Y alone (leading column missing)
```

**index foreign keys.** PostgreSQL doesn't automatically create indexes on foreign key columns (unlike MySQL). if `posts.author_id` references `users.id` and you do `WHERE author_id = X`, it will full-scan `posts` without an index. always add an index to foreign key columns.

### the EXPLAIN ANALYZE command: your query debugger

```sql
EXPLAIN ANALYZE
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.username
ORDER BY post_count DESC
LIMIT 10;
```

output tells you:
- which indexes were used (or not used)
- actual vs estimated row counts (if wildly different, your statistics are stale, run `ANALYZE`)
- time spent in each node
- sequential scans (red flag on large tables)

look for `Seq Scan` on large tables, that's a missing index. look for the actual rows vs estimated rows, big discrepancies mean `VACUUM ANALYZE` is needed.

---

## transactions: making multiple operations atomic

```sql
BEGIN;

UPDATE accounts 
SET balance = balance - 100 
WHERE id = 1 AND balance >= 100;

-- check if the first update actually worked
-- (in application code, check rows affected)

UPDATE accounts 
SET balance = balance + 100 
WHERE id = 2;

COMMIT;
-- or ROLLBACK; if something went wrong
```

without a transaction, if the server crashes between these two updates, user 1 lost money and user 2 never received it. with a transaction, either both happen or neither does.

in Go with `database/sql`:

```go
func transfer(ctx context.Context, db *sql.DB, fromID, toID int64, amount decimal.Decimal) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelReadCommitted, // or LevelSerializable for max safety
    })
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()  // no-op if already committed
    
    // lock both rows in a consistent order (always by ID ascending to prevent deadlocks)
    var fromBalance decimal.Decimal
    err = tx.QueryRowContext(ctx,
        `SELECT balance FROM accounts WHERE id = $1 FOR UPDATE`,  // lock the row
        fromID,
    ).Scan(&fromBalance)
    if err != nil {
        return fmt.Errorf("lock from account: %w", err)
    }
    
    if fromBalance.LessThan(amount) {
        return domain.ErrInsufficientFunds
    }
    
    _, err = tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance - $1 WHERE id = $2`,
        amount, fromID,
    )
    if err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    
    _, err = tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance + $1 WHERE id = $2`,
        amount, toID,
    )
    if err != nil {
        return fmt.Errorf("credit: %w", err)
    }
    
    return tx.Commit()
}
```

`FOR UPDATE` locks the selected rows for the duration of the transaction. other transactions trying to `FOR UPDATE` the same rows will block until this transaction commits or rolls back. this prevents race conditions.

**transaction isolation levels:**
- `READ UNCOMMITTED`: can see uncommitted changes from other transactions. dangerous. PostgreSQL doesn't actually implement this (treats as READ COMMITTED).
- `READ COMMITTED` (default): sees only committed changes. phantom reads can occur.
- `REPEATABLE READ`: same query returns same results within a transaction. still some anomalies possible.
- `SERIALIZABLE`: fully isolated. transactions appear to run one at a time. slowest but safest.

for financial operations, `SERIALIZABLE` or explicit row-level locking (`FOR UPDATE`) is appropriate.

---

## connection pooling: you need this in production

your application opens database connections. connections are expensive (each one is a process in PostgreSQL, using ~5-10MB of memory). if each request opens a new connection, you'll exhaust connections under load.

**pgxpool** (for PostgreSQL in Go) is the standard:

```go
import "github.com/jackc/pgx/v5/pgxpool"

func main() {
    config, _ := pgxpool.ParseConfig(os.Getenv("DATABASE_URL"))
    config.MaxConns = 10           // max connections in pool
    config.MinConns = 2            // minimum idle connections
    config.MaxConnLifetime = time.Hour
    config.MaxConnIdleTime = 30 * time.Minute
    
    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        log.Fatal("cannot connect to database:", err)
    }
    defer pool.Close()
    
    // use pool throughout your application
}
```

**PgBouncer** is a dedicated connection pooler that sits between your apps and PostgreSQL. it's useful when you have many application instances (multiple pods in Kubernetes) because each instance has its own pool, and without PgBouncer you might have 50 pods × 10 connections = 500 connections overwhelming PostgreSQL.

with PgBouncer in `transaction mode`, a connection is only held from the pool during an active transaction. 500 app connections might actually use only 20 PostgreSQL connections. this scales much better.

---

## migrations: changing the schema without destroying everything

as your application evolves, your schema needs to change. migrations are versioned SQL scripts that move the schema from one state to another.

tools for Go:
- **golang-migrate:** simple, widely used, supports many databases
- **goose:** supports Go migrations (not just SQL), good for complex migrations
- **Atlas:** newer, schema-aware, detects drift between schema and database

```sql
-- migrations/000001_initial_schema.up.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/000001_initial_schema.down.sql
DROP TABLE users;
```

```bash
# apply migrations
migrate -path ./migrations -database "$DATABASE_URL" up

# rollback one migration
migrate -path ./migrations -database "$DATABASE_URL" down 1
```

in CI/CD: always run `migrate up` before your integration tests and as part of your deployment. the migration history is in a `schema_migrations` table in the database.

**the expand-contract pattern for safe migrations:**

1. `migrate up` adds new_column as nullable
2. deploy new code that writes to both old_column and new_column
3. backfill old data into new_column
4. `migrate up` makes new_column NOT NULL, drops old_column
5. deploy code that only uses new_column

this way, there's never a moment when deployed code and database schema are incompatible.

---

*next: [07 NoSQL, Redis, and Scaling Databases](07-databases-nosql-scale.md)*

*sources: [PostgreSQL docs](https://www.postgresql.org/docs/) | [SQLite docs](https://sqlite.org/docs.html) | [pgx docs](https://github.com/jackc/pgx)*
