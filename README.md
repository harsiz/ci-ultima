# ci-ultima 🖤

### a dev education series. backend. devops. the whole thing.

---

okay so here's the deal.

there's a million "learn to code" resources out there and like 95% of them are either written for people who've never touched a computer or they skip straight to framework tutorials without explaining *why* anything works.

this series is neither of those. this is meant for someone who already knows the basics, maybe has written some code, maybe even shipped something small, but wants to understand the actual *depth* of what's happening. like you know `git push` exists, but do you know what a commit object *actually is* inside `.git/objects/`? you know PostgreSQL is a database, but do you know what MVCC is and why it lets multiple people write at the same time without exploding? 

you know passwords get "hashed" but could you explain *why* bcrypt is slower on purpose and why that's the point? 💀

this series is going in. like, actually going in.

---

## what's in here

| # | file | what it's about |
|---|------|-----------------|
| 01 | `01-how-code-runs.md` | compilers, interpreters, how your code becomes something a CPU actually understands |
| 02 | `02-go-the-language.md` | Go deep dive, the language that's been quietly eating the backend world |
| 03 | `03-memory-deep-dive.md` | heap, stack, garbage collection, Go's new Green Tea GC |
| 04 | `04-git-and-workflow.md` | git mastery continued, CI/CD going deeper, the professional dev loop |
| 05 | `05-api-design.md` | REST vs GraphQL, HTTP deep dive, designing APIs that don't suck |
| 06 | `06-databases-sql.md` | SQL, PostgreSQL, SQLite, schema design, indexes |
| 07 | `07-databases-nosql-scale.md` | Redis, NoSQL, when to ditch SQL, connection pooling |
| 08 | `08-cryptography.md` | the actual rabbit hole. SHA-256, MD5 collisions, encryption, the math |
| 09 | `09-auth-and-security.md` | Argon2, bcrypt, OAuth2, OIDC, JWT, session management |
| 10 | `10-payments.md` | Stripe, crypto payments, Bitcoin's limitations, stablecoins |
| 11 | `11-docker-and-kubernetes.md` | containers, Docker, Kubernetes, deploying things that stay alive |
| 12 | `12-frontend-for-backend-devs.md` | React vs Angular, what backend devs actually need to know |
| 13 | `13-scale-and-production.md` | caching, queuing, load balancing, handling a million users |
| 14 | `14-python-and-comparisons.md` | Python's actual role, when to use what, wrapping up |
| 15 | `15-testing.md` | testing philosophy, table-driven tests, integration tests, benchmarks, the race detector |

---

## how to read this

these files aren't meant to be read start to finish in one sitting lol. pick a topic you're curious about. jump around. come back to something when it clicks with something else you're learning.

the files do build on each other *slightly* (like you'll want to read `08` before `09` since auth builds on crypto), but for the most part they're pretty self-contained.

there's real code in here, mostly Go because that's the main language of focus, but also some SQL, some shell commands, some config files. all of it is stuff you'd actually write in a real project, not fake tutorial examples.

---

## what this assumes you know

- you've written *some* code in any language. seriously, any language. even if it was HTML in 2014 for a minecraft fan site
- you know what a terminal is and you're not scared of it
- you've used git at least once, even if you only know `git add .`, `git commit`, `git push`
- you have a vague idea that "the backend" is the server stuff and "the frontend" is what users see

if you *don't* have those things, that's fine, but this series will feel fast. go learn some Python basics first, then come back.

---

## why Go specifically

short answer: because it's genuinely the best language for learning *how backend systems actually work*.

Python abstracts too much. JavaScript has too much framework noise. Java has too much ceremony. Rust is incredible but will break your brain before you even get to HTTP servers.

Go is designed to be simple enough to learn quickly but powerful enough that the things you learn actually transfer to production systems. the concurrency model is real. the performance is real. the deployment story (single binary, cross-compile anywhere) is real. companies like Google, Cloudflare, Uber, Twitch, and Dropbox run serious Go in production.

plus it has a garbage collector so you won't be debugging segfaults at 2am 😭

---

## let's "go"
(pfft. get it? since i said "go" and we're gonna be speaking about "golang".. no?.. okay)

[file 01](/guide/01-how-code-runs.md) starts with something almost nobody actually explains: what happens when you run code. like physically, what is the computer actually doing. from there we build up.

see you in there.

---

*This was initially just stuck on a private (local) git repo for reference before I planned on making it public. It's just a mixture of random thoughts of mine regarding software (in particular, backend) compressed into a lil guide book thingy.*

- [Proof I Had Wrote This](/proofs/p.md)
