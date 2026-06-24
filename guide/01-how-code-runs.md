# 01 How Code Actually Runs 🔧
### compilers, interpreters, bytecode, and why this matters more than you think!

---

okay before we talk about Go or databases or anything cool, we need to talk about something that most tutorials completely skip over.

*what actually happens when you run code.*

like not the abstract hand-wavy "the computer executes your instructions" explanation. the actual thing. because once you understand this, a *lot* of other stuff starts making way more sense. why is Python slower than Go? why does Rust make you manage memory manually? why does a compiled Go binary just... work on any Linux machine without installing anything? why do interpreted languages need a "runtime"?

it all comes from this.

---

## the CPU doesn't speak Python

here's the first thing to internalize: **your CPU only understands one language, and it isn't Python, Go, Java, or anything you've ever written.**

it speaks machine code. binary instructions. things like `0x48 0x8B 0x45 0xF8` which means "move the value at memory address [rbp-8] into register rax." your entire program, however you wrote it, eventually has to become this.

the question is: *when* does that translation happen?

---

## compilers: translate first, run later

a **compiler** translates your entire program from the language you wrote it in into machine code *before* you run it. the result is an executable binary, a file that the OS can load directly into memory and hand off to the CPU.

```
your code (.go, .c, .rs)
       |
       v
   [ COMPILER ]
       |
       v
  machine code binary
       |
       v
   CPU executes it
```

languages that compile: **Go, C, C++, Rust, Zig, Swift**

the classic compiler structure has multiple stages:

**1. Lexing (tokenization)**
the compiler reads your source code character by character and breaks it into "tokens." `func`, `main`, `(`, `)`, `{`, `return`, `42`, etc. it doesn't know what they mean yet, it just identifies that these are the words.

**2. Parsing**
the tokens get organized into an **Abstract Syntax Tree (AST).** this is a tree structure that represents the *meaning* of your code. a function call becomes a node with children for the function name and arguments. an if-statement becomes a node with children for the condition and the body.

**3. Semantic analysis**
the compiler checks that the meaning is valid. are you calling a function that actually exists? are you passing the right types? this is where type errors get caught in statically typed languages. in Go, passing a `string` where an `int` is expected fails *here*, at compile time, not while users are using your app.

**4. Optimization**
this is where it gets interesting. the compiler looks for ways to make your code faster *without changing what it does.* inlining small functions (copying the function body to wherever it's called, removing function call overhead), constant folding (computing `3 * 7` at compile time so the binary just has `21`), dead code elimination (removing code that can never run), loop unrolling.

modern compilers (Go's gc compiler, LLVM which powers Clang/Rust) do *aggressive* optimization. the code your CPU runs often looks very different from the code you wrote.

**5. Code generation**
the optimized intermediate representation gets turned into actual machine instructions for the target architecture. `GOARCH=amd64` generates x86-64 instructions. `GOARCH=arm64` generates ARM instructions. this is how cross-compilation works, you're just changing the target of step 5.

---

## interpreters: translate and run simultaneously

an **interpreter** doesn't produce a binary. instead, it reads your code at runtime, translates it to machine instructions on the fly, and executes those instructions immediately.

```
your code (.py, .js, .rb)
       |
       v
  [ INTERPRETER ] ← reads line/statement by line
       |
       v
   executes immediately
```

languages that interpret: **Python (CPython), Ruby, older PHP, Bash**

the obvious downside: **it's slower.** you're doing translation work every time you run the program, and you can't do as aggressive optimization because you don't have the whole program visible at once.

the upside: **portability and flexibility.** you don't need to compile for specific platforms. you can do things like `eval()` which executes strings as code at runtime. you can inspect and modify code as it runs. these properties make interpreted languages great for scripting, automation, and data science.

this is why Python is *so* dominant in ML/AI, the flexibility of an interpreted language makes experimenting fast. you don't recompile after every change. you just re-run.

---

## the messy middle: bytecode + VMs

most modern "interpreted" languages don't actually interpret source code directly, that's super slow. instead they do a hybrid:

1. compile source to **bytecode** (an intermediate representation, simpler than machine code but not specific to any CPU)
2. run the bytecode in a **virtual machine** that interprets it

this is what Python (CPython) actually does. when you run a `.py` file, Python compiles it to bytecode (`.pyc` files in `__pycache__`). the CPython VM then interprets that bytecode. you barely notice the compile step because it's fast.

Java does this too but took it further: the **JVM** (Java Virtual Machine) takes Java bytecode (`.class` files) and runs them. the JVM has a **JIT compiler** (Just-In-Time) that detects "hot" code (code running frequently) and compiles *those specific parts* to native machine code at runtime. so Java starts interpreted and progressively compiles the parts that matter.

```
Java:
  .java → [ javac ] → .class bytecode → [ JVM + JIT ] → native machine code (for hot paths)
```

modern JavaScript engines (V8 in Chrome/Node.js) do the same thing. V8 has a JIT compiler called Turbofan that compiles hot JS to machine code. this is why Node.js is fast enough to run real servers despite JavaScript being "interpreted."

---

## Go specifically: the compiler that doesn't mess around

Go uses a ahead-of-time compiler called `gc` (Go compiler, not garbage collector, confusing naming, I know). it's designed to be **fast to compile** while producing **fast code.**

```bash
go build -o myapp main.go
# produces a self-contained binary
```

the result is a **statically linked** binary by default. that means all the dependencies are bundled into the executable. no "do you have the right version of X runtime installed?" it just runs. copy the binary to any Linux machine with the right CPU architecture and it executes.

cross-compilation is trivially easy:

```bash
# build a Linux binary from macOS or Windows
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go

# build a Windows exe from Linux
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go
```

`GOOS` = target operating system. `GOARCH` = target CPU architecture. the compiler handles all the platform-specific code generation.

Go's compilation is genuinely fast. a medium-sized codebase that would take minutes in Rust compiles in seconds in Go. this matters enormously for developer experience and CI/CD pipelines.

---

## static typing vs dynamic typing (and why it's not just preference)

**statically typed** languages (Go, Rust, Java, TypeScript) know the type of every variable at compile time. the compiler can catch type errors before the program runs.

**dynamically typed** languages (Python, JavaScript, Ruby) determine types at runtime. type errors only appear when you actually execute the problematic code.

this has real consequences:

```python
# Python - this is valid code that will crash at runtime
def add(a, b):
    return a + b

result = add("5", 10)  # crashes with TypeError when this line executes
```

```go
// Go - this fails at compile time, never reaches users
func add(a int, b int) int {
    return a + b
}

result := add("5", 10) // compiler error: cannot use "5" (type string) as type int
```

in Go, the compiler rejects your code. in Python, users see the crash.

this isn't about Go being "better" or Python being "bad." it's a genuine trade-off:
- static typing catches bugs earlier, enables better IDE tooling, makes refactoring safer, and lets the compiler optimize more aggressively
- dynamic typing enables faster prototyping, more flexible code patterns, and easier metaprogramming

for backend systems that run 24/7 and handle real data, catching bugs at compile time instead of at 3am production incident is extremely valuable. this is why Go, Rust, Java, and TypeScript dominate backend/systems work.

---

## what "stack" and "heap" mean at this level

while we're here, let's establish these terms because they'll come up constantly.

your program has two main memory regions:

**the stack**
- a region of memory that grows and shrinks as functions are called and return
- each function call gets a "stack frame" that holds its local variables
- when the function returns, its stack frame is *automatically* freed
- fast: allocation is just moving a pointer
- limited in size (typically 8MB on Linux, though Go's goroutine stacks start at just 2KB and grow dynamically)
- you can't return pointers to stack-allocated data because the memory disappears when the function returns

```go
func add(a, b int) int {
    result := a + b  // 'result' is on the stack
    return result    // stack frame freed after this
}
```

**the heap**
- a region of memory for data that needs to outlive the function that created it
- in C/C++, you allocate heap memory manually (`malloc`) and must free it manually (`free`). forgetting to free = memory leak. freeing too early = use-after-free bug.
- in Go, Java, Python: a **garbage collector** handles freeing heap memory automatically
- slower than stack: allocation requires finding free space, GC needs to track everything

```go
func newUser(name string) *User {
    u := &User{Name: name}  // allocated on the heap (escapes the function)
    return u                // heap memory persists after return
}
```

the Go compiler does **escape analysis** to figure out what goes on the stack vs the heap. if a variable's lifetime can be determined at compile time (it doesn't escape the function), it goes on the stack. if it "escapes" (gets returned, stored in a global, sent to a goroutine), it goes to the heap. this optimization is automatic and massively reduces GC pressure.

---

## why "interpreted" Python is still fast enough for many things

you might be wondering: if Python is so slow (and it *is* slower, like 10-100x slower than Go for CPU-intensive work), why is it used everywhere?

because most of the *actual computation* in Python programs isn't done by Python.

NumPy, pandas, PyTorch, these libraries are written in C and C++. when you do `numpy.dot(a, b)`, Python is just calling into highly optimized C code. the Python overhead is just the function call. the actual matrix multiplication runs at C speed.

so "Python ML" is really "Python as a scripting layer over C++ math libraries." Python's flexibility and expressiveness are what developers interact with. the performance-critical parts are pre-compiled C extensions.

this is a totally legitimate architecture! it's just important to *understand* what's actually happening. if you try to write a tight numerical loop in pure Python, it'll be 100x slower than Go. if you vectorize it into NumPy operations, it'll be fast because you're calling C.

---

## the stuff no one tells you about runtime environments

here's something that trips up a lot of developers: the "runtime" of a language isn't just about interpretation. even compiled languages have a runtime.

Go programs include the **Go runtime** in the binary. it's just compiled in and invisible. the Go runtime handles:
- goroutine scheduling (the M:N threading model, more on this in file 02)
- garbage collection
- the channel implementation
- stack growth

when people say "Go produces a single binary," they mean the application code *plus* the runtime are all compiled together. there's no separate Go runtime you need installed on the server (unlike, say, a Node.js app that needs Node installed, or a Java app that needs the JVM).

this is why Go binaries are larger than they "should" be given the source code size, they include the runtime. a hello world in Go produces a ~2MB binary. a hello world in C is like 12KB. but that 2MB includes everything you need to run concurrent programs with a garbage collector.

---

## compilers, interpreters, and your CI pipeline

this all connects directly to your development and deployment workflow:

**compiled languages:** your CI pipeline has a compile step. if it fails, nothing deploys. type errors and some logic errors are caught here. build artifacts (binaries, JARs, Docker images containing binaries) are what get deployed.

**interpreted languages:** no compile step, so your CI pipeline relies more heavily on tests to catch errors. the "artifact" is usually the source code itself, plus installed dependencies. the deployment target needs the interpreter installed.

**hybrid (JVM/V8):** compile step produces bytecode, but you still need the runtime on the server. JVM apps in Docker bundle the JVM in the container image.

```yaml
# GitHub Actions for a Go service
jobs:
  build:
    steps:
      - run: go build ./...        # compile everything
      - run: go test ./...         # run tests
      - run: go vet ./...          # static analysis
      - run: docker build .        # package into container
```

```yaml
# GitHub Actions for a Python service  
jobs:
  build:
    steps:
      # no compilation step
      - run: pip install -r requirements.txt
      - run: pytest                 # tests are even more critical here
      - run: ruff check .          # linting
      - run: mypy .                # type checking (optional, but smart)
      - run: docker build .
```

the absence of a compile step in Python isn't necessarily a *problem*, but it means your test suite is the primary safety net. a bug that a Go compiler would catch at build time will only be caught by Python tests... if you wrote the test.

---

## summary: the mental model

```
Source Code
    │
    │  Compilation (Go, C, Rust)     OR    Interpretation (Python, Ruby)
    │  [happens before running]             [happens while running]
    │
    ▼
Machine Code / Bytecode
    │
    │  CPU executes directly        OR    VM interprets bytecode
    │                                     + JIT compiles hot paths (Java, V8)
    ▼
Your program runs
```

key takeaways:
- **compiled = faster execution, compile-time error detection, larger deployment artifacts**
- **interpreted = slower execution, runtime errors, more flexible**
- **bytecode + VM = middle ground, JIT can get close to compiled speeds**
- **Go sits firmly in "compiled, fast, statically typed"** which is why it's great for backend systems
- **the stack is fast and automatic, the heap is flexible and managed by GC**
- **escape analysis in Go minimizes unnecessary heap allocations**

in the next file we actually start *writing* Go. but this context matters. when you see `go build` produce a binary in 3 seconds, you'll know what just happened. when you see Python import a C extension, you'll know why it's fast despite being "interpreted." when a type error in Go saves you from a production crash, you'll know why the compiler complained.

---

*next: [02 Go: The Language That Actually Makes Sense](02-go-the-language.md)*

*sources: [Go release history](https://go.dev/doc/devel/release) | [Go compiler internals](https://go.dev/doc/compile) | [CPython bytecode](https://docs.python.org/3/library/dis.html)*
