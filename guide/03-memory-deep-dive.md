# 03 Memory Deep Dive: Heap, Stack, and Go's Green Tea GC ☕
### this is the stuff that separates juniors from seniors

---

nobody teaches memory properly. tutorials say "the garbage collector handles memory for you" and move on, like that's sufficient. it's not.

because even with a GC, memory-related bugs and performance problems are *everywhere* in production systems:
- memory leaks in Go (yes, they happen even with a GC)
- GC pauses that cause latency spikes
- excessive heap allocations that destroy throughput
- stack overflows from deep recursion
- goroutine leaks that slowly eat all your RAM

understanding memory at this level isn't about being clever, it's about being able to diagnose real production problems when they happen. and they will happen 😭

---

## the physical reality: RAM

your computer has RAM (Random Access Memory). it's fast storage that loses its contents when power is cut. your CPU has registers (tiny, extremely fast storage directly on the chip) and cache levels (L1, L2, L3, smaller and faster than RAM but still fast). when your program accesses memory that isn't in cache, the CPU stalls waiting for RAM. this is why **memory access patterns matter** for performance.

when your OS loads your program, it gives it a virtual address space. on a 64-bit system this is theoretically 2^64 bytes (the actual usable range is more like 128TB depending on OS). your program thinks it has the whole address space to itself. the OS and hardware (via the Memory Management Unit, MMU) transparently map virtual addresses to actual physical RAM.

within this virtual address space, your program's memory is divided into regions:

```
High addresses
┌──────────────────────┐
│       Stack          │  ← grows downward, per-goroutine in Go
├──────────────────────┤
│          ↕           │  free space
├──────────────────────┤
│        Heap          │  ← grows upward, shared, GC-managed
├──────────────────────┤
│   BSS / Data         │  global variables
├──────────────────────┤
│        Text          │  your compiled code (read-only)
└──────────────────────┘
Low addresses
```

(Note: Go's model is slightly different per-goroutine but this is the conceptual structure)

---

## the stack: simple, fast, automatic

**the stack** is a LIFO (Last In, First Out) data structure for function call frames. when you call a function, a new frame is pushed onto the stack containing:
- the function's local variables
- the return address (where to resume after the function returns)
- function arguments

when the function returns, the frame is popped (the stack pointer just moves). the memory is "freed" instantly. no GC involvement. no bookkeeping. just arithmetic on a pointer.

this is why stack allocation is so fast, roughly 1-2 CPU cycles vs potentially hundreds for heap allocation.

```go
func add(a, b int) int {
    result := a + b    // 'result' is on the stack
    return result      // frame pops, 'result' ceases to exist
}
```

**Go goroutine stacks are dynamic.** this is unusual. in most languages and runtimes, each thread has a fixed-size stack (typically 1-8MB). if you recurse too deeply or have too many local variables, you get a stack overflow. game over.

Go starts each goroutine with just **2KB** of stack and grows it dynamically as needed. if the Go runtime detects you're about to exceed the current stack size, it:

1. allocates a new, larger stack (typically 2x the current size)
2. copies all stack frames to the new stack
3. updates all pointers that pointed to the old stack
4. frees the old stack

this means you can have deep recursion in Go goroutines without manually sizing stacks. you can also have a million goroutines each with a tiny 2KB stack rather than a million OS threads each burning 8MB of stack. that's the reason Go handles massive concurrency so well.

---

## the heap: flexible, powerful, needs management

the **heap** is for data that needs to outlive the function that created it, or whose size isn't known at compile time, or that's too large for the stack.

```go
func newUser(name string) *User {
    u := &User{Name: name}  // 'u' is allocated on the heap
    return u                // the pointer to 'u' is returned
                            // 'u' itself lives on after this function returns
}
```

the `&` operator takes the address of something. `&User{...}` creates a `User` on the heap and returns a pointer to it. the `User` data outlives the `newUser` function.

**escape analysis** is the compiler phase that decides what goes on the stack vs the heap. if a value "escapes" the function (gets returned by pointer, stored in a global, sent on a channel, used in an interface), it goes to the heap. if it doesn't escape, it stays on the stack.

```bash
# see what's escaping to the heap
go build -gcflags="-m" ./...

# example output:
# ./main.go:12:11: &User{...} escapes to heap
# ./main.go:8:13: name does not escape
```

**minimizing heap allocations = less GC pressure = better throughput.** this is a real optimization concern in high-performance Go code.

---

## garbage collection: the basics before we get into Green Tea

**garbage collection** is automatic memory management. the GC periodically finds objects on the heap that are no longer reachable from your code and frees them.

the fundamental algorithm is **mark-and-sweep**:

1. **mark phase:** starting from "roots" (global variables, stack variables, registers), traverse all pointers and mark every reachable object
2. **sweep phase:** scan the heap. any object that wasn't marked is unreachable (garbage). free it.

```
Roots (globals, goroutine stacks)
  │
  ├── [User A] → [Post 1] → [Comment 1]
  │                       → [Comment 2]
  ├── [User B] → [Post 2]
  │
  └── (nothing points to [User C] → [Post 3])
       ↑ this is garbage, will be freed
```

in the above, `User C` and `Post 3` are unreachable, nothing in your program holds a reference to them. the GC will free them.

**this is why you can't have memory leaks in theory, but can have them in practice.** if you accidentally keep a reference to something (a slice that keeps growing, a map that never gets entries removed, a goroutine that's blocked forever but holds references), the GC won't free it. *it's still reachable.*

the classic Go memory leak:

```go
// this goroutine leaks if the channel is never closed and nobody reads from it
func startWorker(ch chan Data) {
    go func() {
        for data := range ch {
            process(data) // goroutine blocked here forever if ch is never closed
        }
    }()
}
// if you call startWorker many times without ever closing the channels,
// you'll accumulate goroutines, each holding references to data
```

goroutine leaks are Go's most common "memory leak." use `context.Context` with cancellation to stop goroutines when they're no longer needed:

```go
func startWorker(ctx context.Context, ch chan Data) {
    go func() {
        for {
            select {
            case data, ok := <-ch:
                if !ok {
                    return // channel closed
                }
                process(data)
            case <-ctx.Done():
                return // context cancelled, goroutine exits cleanly
            }
        }
    }()
}
```

---

## Go's GC before Green Tea: tricolor mark-and-sweep

Go uses a **concurrent, tricolor mark-and-sweep** GC. "concurrent" means the GC runs alongside your application goroutines (mostly) rather than stopping the world completely.

the tricolor algorithm uses three sets:
- **white:** not yet visited (initially everything is white)
- **gray:** reachable but children not yet scanned
- **black:** reachable and children scanned

the algorithm transitions objects from white → gray → black. when no more gray objects remain, anything still white is garbage.

**stop-the-world (STW)** pauses happen briefly (usually sub-millisecond in modern Go) at the beginning (to start the GC cycle) and end (to finalize). between those pauses, the GC marks concurrently alongside your code.

Go's GC has been progressively tuned over years. STW pauses went from milliseconds in early Go to **sub-millisecond** in recent versions. this makes Go viable for latency-sensitive services.

the key GC tuning parameter is `GOGC`:
```bash
GOGC=100  # default: trigger GC when heap doubles
GOGC=200  # trigger GC when heap triples (less frequent GC, uses more memory)
GOGC=50   # trigger GC when heap grows 50% (more frequent GC, less memory used)
GOGC=off  # no GC (dangerous, only for specific use cases)
```

---

## Green Tea GC: Go 1.26's big deal ☕

**Green Tea GC** shipped as an experiment in Go 1.25 (August 2025) and became the **default GC in Go 1.26** (February 2026). it delivers **10-40% lower GC overhead** for most programs. no code changes required.

[(source: go.dev/blog/greenteagc)](https://go.dev/blog/greenteagc)

**the key insight:** instead of tracking individual objects, Green Tea operates at the **memory page level.**

```
Traditional GC approach:
  Heap: [obj1][obj2][obj3][obj4][obj5]...
        track each object individually
        
Green Tea approach:
  Heap: [page 1: obj1, obj2][page 2: obj3, obj4][page 3: obj5...]
        track pages globally
        track individual objects locally WITHIN each page
```

this changes the scanning strategy from depth-first (following pointers wherever they lead across the heap) to a combination of global page-level scanning and local within-page scanning. the result is better cache locality during GC, the GC is scanning memory that's physically close together rather than chasing pointers across the entire heap.

**performance impact:**
- most workloads: **~10% less time in GC**
- high-concurrency services with trees, graphs, or indexes: **up to 40% reduction**
- multi-core machines benefit more (Green Tea scales better with core count)
- microbenchmarks on high-core machines show **10-50% GC CPU reduction**

**when Green Tea excels:**
- workloads spending >10% of CPU in GC
- data structures with high fan-out (trees, graphs, caches)
- services that hold large amounts of data in memory
- multi-core deployments (8+ cores see the biggest gains)

**when Green Tea is neutral or slightly negative:**
- single-core or dual-core deployments
- very low fan-out data structures that mutate frequently
- extremely tight latency requirements (sub-millisecond p99), Green Tea's breadth-first page scanning has different latency characteristics

to check if you benefit: look at your runtime metrics. if `go_gc_duration_seconds` or GC CPU percentage is significant, Green Tea will help.

---

## memory profiling: finding the actual problem

okay theory is great, but how do you find actual memory problems in a real program?

**pprof:** Go's built-in profiling tool is exceptional.

```go
import _ "net/http/pprof" // import for side effect: registers pprof handlers

func main() {
    // in your HTTP server
    go http.ListenAndServe(":6060", nil) // pprof endpoint
    // ... rest of your app
}
```

then while your program runs:

```bash
# memory profile: what's on the heap RIGHT NOW
go tool pprof http://localhost:6060/debug/pprof/heap

# goroutine profile: what are all goroutines doing
go tool pprof http://localhost:6060/debug/pprof/goroutine

# CPU profile: where is time being spent
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

inside pprof:

```
(pprof) top10          # top 10 allocating functions
(pprof) web            # open SVG flame graph in browser (requires graphviz)
(pprof) list funcName  # annotate source with allocation counts
```

**reading a heap profile:**
- `alloc_objects` / `alloc_space`: everything ever allocated (total since program start)
- `inuse_objects` / `inuse_space`: currently in use (what the GC hasn't freed)

if `alloc_space` is huge but `inuse_space` is small: allocating a lot then freeing it, GC pressure issue.
if `inuse_space` grows continuously over time: memory leak.

---

## the sync.Pool: avoiding GC pressure for frequently allocated objects

`sync.Pool` is a cache for temporary objects. instead of allocating a new buffer every time you handle an HTTP request, you get one from the pool, use it, and return it.

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        buf := make([]byte, 0, 4096) // 4KB buffer
        return &buf
    },
}

func handleRequest(data []byte) {
    // get a buffer from the pool instead of allocating
    bufPtr := bufferPool.Get().(*[]byte)
    buf := (*bufPtr)[:0] // reset length but keep capacity
    
    defer func() {
        *bufPtr = buf
        bufferPool.Put(bufPtr) // return to pool for reuse
    }()
    
    // use buf for processing
    buf = append(buf, data...)
    // ... process
}
```

this can dramatically reduce GC pressure in high-throughput systems. the Go runtime itself uses `sync.Pool` heavily internally for things like HTTP header buffers.

one gotcha: `sync.Pool` objects can be collected by the GC at any time. don't use it for objects you need to persist across GC cycles. use it only for temporary scratch space.

---

## common memory mistakes in Go

**1. growing a slice in a loop without pre-allocation**

```go
// bad: each append may allocate a new backing array
var results []int
for i := 0; i < 100000; i++ {
    results = append(results, i)
}

// good: pre-allocate
results := make([]int, 0, 100000)
for i := 0; i < 100000; i++ {
    results = append(results, i)
}
```

slices in Go have a length and a capacity. when you `append` past capacity, Go allocates a new backing array (typically 2x the size) and copies everything. pre-allocating avoids many of these reallocations.

**2. keeping a large slice alive by holding a small slice of it**

```go
func getFirstBytes(data []byte) []byte {
    // BAD: returns a slice backed by the original large data
    // the entire 'data' stays in memory as long as this result lives
    return data[:10]
}

func getFirstBytes(data []byte) []byte {
    // GOOD: copy the data, original can be GC'd
    result := make([]byte, 10)
    copy(result, data[:10])
    return result
}
```

slices share backing arrays. a small slice of a large slice keeps the entire large slice alive.

**3. string concatenation in a loop**

```go
// bad: each += creates a new string (strings are immutable in Go)
var result string
for _, s := range words {
    result += s // O(n²) allocations
}

// good: use strings.Builder
var b strings.Builder
for _, s := range words {
    b.WriteString(s)
}
result := b.String()
```

**4. holding references in goroutines**

```go
// this captures 'largeData' in the closure
// even if main() doesn't need largeData anymore,
// the goroutine holds a reference, preventing GC
largeData := loadHugeThing()
go func() {
    // process largeData
    for _, item := range largeData {
        doWork(item)
    }
    // after this goroutine exits, largeData can be GC'd
}()
// if the goroutine leaks (runs forever), largeData leaks too
```

---

## GOGC and GOMEMLIMIT: tuning for production

go 1.19 added `GOMEMLIMIT`, which lets you set a soft memory limit. the GC will try to keep memory usage below this limit:

```bash
GOMEMLIMIT=1GiB go run main.go
```

or in code:
```go
import "runtime/debug"

func main() {
    debug.SetMemoryLimit(1 * 1024 * 1024 * 1024) // 1 GiB
    // ...
}
```

`GOMEMLIMIT` is crucial in containerized environments (Docker, Kubernetes) where your container has a memory limit. without it, Go doesn't know about the container limit and might trigger OOM kills before the GC kicks in aggressively enough.

the recommended production configuration:
- set `GOMEMLIMIT` to ~90% of your container's memory limit (leave headroom for the stack and non-Go allocations)
- `GOGC=100` (default) is usually fine with `GOMEMLIMIT` set

```yaml
# Kubernetes deployment
env:
  - name: GOMEMLIMIT
    value: "900MiB"  # if container limit is 1Gi
  - name: GOMAXPROCS
    value: "4"  # match CPU limits
```

speaking of `GOMAXPROCS`: this controls how many OS threads run Go code simultaneously. defaults to the number of CPU cores. in a container with a CPU limit (e.g., 2 CPU), the runtime may see 32 physical cores but you only get 2. set `GOMAXPROCS` to match your container's CPU limit. the `go.uber.org/automaxprocs` library does this automatically at startup.

---

## comparing memory models across languages

| Language | Memory Model | GC | Developer Control |
|----------|-------------|-----|-------------------|
| C/C++ | manual | none | full (and terrifying) |
| Rust | ownership + borrowing | none | full, enforced by compiler |
| Go | GC (Green Tea) | concurrent mark-sweep | escape analysis, pools |
| Java | GC (G1, ZGC, Shenandoah) | concurrent | very limited |
| Python | ref counting + GC | yes | very limited |
| JavaScript | GC (V8's Orinoco) | concurrent | very limited |

**Rust's ownership model** is genuinely a different paradigm. the compiler enforces rules about who owns memory and how long references are valid. no GC at all. memory is freed *exactly* when the owner goes out of scope, at compile time. this gives you C-like performance and C-like control with *safety* guarantees. the trade-off: learning the borrow checker is genuinely hard. many developers describe it as a paradigm shift that takes weeks to months to internalize.

Go sits in the sweet spot for most backend work: you get automatic memory management (no manual `free`, no use-after-free bugs) with performance close enough to Rust for the vast majority of applications. where Rust wins is on *extremely* latency-sensitive code (game engines, HFT, real-time systems) and embedded/systems programming where GC pauses are unacceptable.

---

## the bottom line

understanding memory doesn't mean you'll be manually managing it (in Go, you're not). it means:

- you can look at a heap profile and diagnose why your service is using 4GB of RAM when it should be using 400MB
- you can design data structures that minimize GC pressure
- you can tune `GOGC` and `GOMEMLIMIT` appropriately for your container limits
- you can use `sync.Pool` where it matters
- you understand *why* Go's Green Tea GC makes your throughput better and what kind of workloads benefit most
- you can find and fix goroutine leaks before they eat your production server at 3am

the GC handles the scary parts. you handle the design parts.

---

*next: [04 Git Mastery and the Professional Dev Workflow](04-git-and-workflow.md)*

*sources: [Go Green Tea GC](https://go.dev/blog/greenteagc) | [Go 1.26 GC 40% faster](https://byteiota.com/go-1-26-green-tea-gc-40-faster-garbage-collection/) | [Green Tea GC InfoQ](https://www.infoq.com/news/2025/11/go-green-tea-gc/)*
