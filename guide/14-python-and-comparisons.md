# 14 Python, Comparisons, and What It All Means 🐍
### wrapping up with the honest takes

---

we've been Go-forward this entire series because Go is the best language for learning backend systems concepts clearly. but your career will involve multiple languages, multiple paradigms, and lots of "which tool for this job" decisions. this chapter is the comparisons, the opinions, and the final send-off.

---

## Python: the language that's everywhere and that's fine

Python is the most commonly known programming language in the world. it's the "first language" for most universities, most data scientists, most ML engineers, most DevOps automation, most scripting. you will use Python in your career regardless of what your primary language is.

### what Python is actually good at

**data science and ML.** this is Python's undisputed home territory. NumPy, pandas, scikit-learn, PyTorch, TensorFlow, these libraries are written in C/C++ with Python bindings. Python is the scripting layer and the development environment. when you see a company say they use "Python for ML," they mean Python as the interface to extremely fast C/C++ code.

```python
import numpy as np
import torch

# this matrix multiplication isn't running in Python
# it's running in highly optimized CUDA (GPU) or C++ code
a = torch.randn(1000, 1000, device='cuda')
b = torch.randn(1000, 1000, device='cuda')
c = torch.matmul(a, b)  # <1ms on a GPU
```

**scripting and automation.** shell scripts become unwieldy fast. Python scripts are more readable, testable, and cross-platform. if you need to process files, call APIs, parse data, automate deployments, Python is the best tool.

**prototyping.** the lack of boilerplate makes Python fast for exploring ideas. no type declarations, no compile step, rich REPL. a prototype that would take an hour in Python might take a day in Go just because of the required structure.

**web scraping and data collection.** `requests`, `BeautifulSoup`, `Scrapy`, `Playwright` (yes, there's a Python version), Python's web scraping ecosystem is the best.

### what Python is not good at

**CPU-intensive work.** pure Python loops are 10-100x slower than Go or Rust. the GIL (Global Interpreter Lock) in CPython prevents true parallel execution of Python code on multiple CPU cores. if you're doing heavy computation in pure Python, you're doing it wrong.

```python
# GIL problem: this doesn't actually parallelize on CPython
import threading

def cpu_intensive():
    # compute something
    
threads = [threading.Thread(target=cpu_intensive) for _ in range(8)]
# these 8 threads don't run truly in parallel on CPython
# only one thread executes Python bytecode at a time
```

solutions: use `multiprocessing` (separate processes, no GIL), use C extensions (numpy, etc.), use `asyncio` for I/O-bound concurrency (doesn't help CPU-bound), or use a different language.

**high-concurrency services.** Python's async story is `asyncio`, which is single-threaded cooperative multitasking. it's fine for I/O-bound servers (most web APIs) but doesn't use multiple CPU cores for your Python code.

**cold start time.** Python takes 200-800ms just to start the interpreter and import modules. for serverless functions or CLI tools that need to start fast, this is painful. Go starts in milliseconds.

**type safety.** `mypy` and `pyright` add static type checking to Python, but it's optional and the type system is structural and incomplete. you can have code that passes mypy but crashes at runtime due to type errors. this is categorically different from Go where the compiler guarantees type safety before execution.

### Python in backend: where it actually fits

Python is a perfectly legitimate backend language for many use cases:

```python
# FastAPI: the current standard for Python APIs
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    existing = await db.execute(
        select(User).where(User.email == user_data.email)
    )
    if existing.scalar_one_or_none():
        raise HTTPException(status_code=409, detail="email already in use")
    
    user = User(
        username=user_data.username,
        email=user_data.email,
        password_hash=hash_password(user_data.password),
    )
    db.add(user)
    await db.commit()
    return UserResponse.model_validate(user)
```

**FastAPI** (2019) is the current standard for Python APIs. it uses Python type hints for automatic request/response validation, OpenAPI documentation generation, and editor autocomplete. it's significantly better than Flask for API development.

**Django** remains the choice for full-stack Python web applications with complex data models. it has the best ORM in any language ecosystem (Django ORM), excellent admin interface generation, and comprehensive batteries-included philosophy.

**where Python backend makes sense:**
- data-heavy applications where you're also doing analysis (share code between API and notebooks)
- internal tooling where startup time and deployment simplicity matter more than performance
- rapid prototyping where you need to ship something now and optimize later
- ML-serving APIs that need to call PyTorch/TensorFlow models

---

## the honest language comparison

| | Go | Python | Rust | TypeScript/Node |
|---|---|---|---|---|
| Learning curve | Low-Medium | Low | Very High | Low |
| Performance | High | Low-Medium* | Highest | Medium |
| Compile time | Fast | N/A | Slow | Fast |
| Deployment | Single binary | Runtime needed | Single binary | Runtime needed |
| Concurrency | Excellent (goroutines) | Limited (GIL) | Excellent | Good (event loop) |
| Ecosystem | Medium | Very Large | Growing | Very Large |
| Type safety | Compile-time | Optional runtime | Compile-time | Compile-time |
| Best for | Backend APIs, CLI, infra | ML, scripting, prototyping | Systems, embedded | Full-stack web |

*Python is fast when using C extensions (numpy, etc.)

### the languages I'd tell you NOT to use for a new backend API

**PHP (in 2026):** modern PHP (8.x, with Fibers for async) is genuinely a different language from the PHP that gave it its reputation. Laravel is a good framework. but the community is shrinking, the ecosystem is losing momentum, and the deployment story (Apache/Nginx + PHP-FPM) adds operational complexity. if you're starting a new API today, choose something else.

**Ruby (new projects):** Rails had its moment and it was glorious (2005-2012). the convention-over-configuration model influenced every framework that came after. but the ecosystem has been declining for a decade. if you're maintaining a Rails app, great, Rails is still capable. but starting a new one in 2026 means hiring from a shrinking pool, using a framework that fewer people are actively maintaining.

**Java for small teams:** Java is fine for large enterprise contexts where operational overhead is acceptable and the JVM ecosystem's tooling is valuable. for a 3-person startup, the verbosity and ceremony of Java (before Kotlin at least) slow you down unnecessarily. Kotlin is Java but actually good.

**"anything because we know it":** the switching cost of learning a new language is much lower than the 5-year maintenance cost of a wrong architectural choice. it's worth the temporary discomfort of learning Go or TypeScript if they're better suited to your use case.

---

## what makes someone a good developer (the actual answer)

this series has covered a lot of technical content. here's what I think actually separates good developers from great ones:

**1. They understand why, not just how.**

knowing that `git rebase` moves commits doesn't make you good at git. knowing *when* to rebase vs merge vs revert, and understanding the implications of each, that's what matters. the same for every topic in this series. "use Argon2id for passwords" is a rule. "passwords need to be slow and memory-hard to resist GPU attacks, and Argon2id is specifically designed for this" is understanding. you can apply understanding to novel situations. you can only apply rules to known situations.

**2. They can estimate and communicate uncertainty.**

"it'll take two days" is a guess. "it'll probably take two days, but if the database schema is different from what I expect it could take four" is an estimate with uncertainty quantified. good engineers communicate what they know, what they don't know, and how confident they are. this is a skill that takes years to develop and is more valuable than almost any technical skill.

**3. They read error messages and documentation.**

sounds obvious. it's shocking how many people don't. a good engineer:
- reads the full error message before searching Google
- reads the relevant documentation section before asking someone
- looks at the code that failed, not just the error

the ability to figure things out independently is multiplicative, you can do much more because you're not blocked waiting for answers.

**4. They write for the reader, not the writer.**

code is read far more than it's written. commit messages, variable names, function names, API designs, every time you write something, it's primarily for the person reading it later (including future you). the question "will this make sense in 6 months" is more valuable than "does this work now."

**5. They know when NOT to build something.**

the best code is the code you don't write. the best feature is the feature that solves the user's problem without adding complexity. the best optimization is the one you don't need because the system is already fast enough. every line of code is a liability, it needs to be maintained, tested, debugged, and understood. strong engineers challenge requirements and look for simpler solutions before building.

---

## the learning path from here

if you've read this series (or even part of it), here's what I'd suggest next:

**build a real project.** pick something you actually want to exist and build it using the concepts here. a Go backend with PostgreSQL, authentication, and a simple frontend. the act of building reveals gaps in understanding that reading never does.

**read source code.** read the source of libraries you use. Go's standard library source is genuinely educational. the `net/http` package shows you how HTTP servers work. the `crypto/tls` package shows you TLS implementation. `database/sql` shows you connection pooling. real code from experienced engineers is the best learning resource.

**go.dev/blog.** the official Go blog has deep dives on GC, concurrency, performance, and language design. it's consistently excellent.

**"Designing Data-Intensive Applications" by Martin Kleppmann.** the best book on databases, distributed systems, and the tradeoffs involved. covers everything from B-trees to consensus algorithms. dense but every page is worthwhile.

**"The Pragmatic Programmer" by David Thomas and Andrew Hunt.** career fundamentals that aren't about any specific language. how to think about problems, communicate with teams, maintain software.

**contribute to open source.** nothing teaches you faster than having your code reviewed by experienced engineers who will tell you, kindly but directly, what's wrong and why.

---

## final take

this is a long series. if you read all of it, genuinely, that's impressive. most people who start a learning resource don't finish it.

the common thread through all of it: **understanding the why behind every tool, technique, and decision.** the tools change. Go 1.26 today, something new in five years. PostgreSQL is dominant now, but databases will evolve. the fundamentals, how memory works, why cryptography needs to be slow for passwords, why you can't just "use a bigger server" forever, why ACID matters, why HTTP is stateless, these stay relevant for the entirety of a software engineering career.

Moe the gopher approves of your journey 🐹

(and if you didn't understand the Moe references: he's the unofficial name for Go's mascot. yes this is a joke. yes the joke was worth it.)

---

*this is the end of the ci-ultima series.*

*June 2026 | verified against current language versions, library releases, and security standards*

*links throughout this series are to primary sources: official documentation, academic papers, and well-sourced technical articles.*
