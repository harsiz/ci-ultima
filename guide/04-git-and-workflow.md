# 04 Git Mastery and the Professional Dev Workflow 🌿
### this is the continuation you didn't know you needed

---

*if you read the first two parts of this series (cicd-explained.md and cicd-explained-part2.md), a lot of this will feel familiar. this chapter goes further, with more focus on backend-specific workflows, Go projects, and the mental models that actually make you good at this.*

---

real talk: git is the one tool you'll use every single day as a developer, often dozens of times, and yet most tutorials teach you like five commands and call it done. `git add .`, `git commit -m "stuff"`, `git push`, repeat. and then three months into your first job you run `git rebase` without understanding it and destroy two days of work and now you're googling "git undo everything please" at 1am.

not on our watch 😭

---

## git internals: what's actually inside `.git/`

every git repository has a `.git/` directory. let's look inside:

```
.git/
├── objects/          ← the content-addressable store (THE database)
│   ├── 00/
│   ├── 1a/
│   ├── ...
│   ├── info/
│   └── pack/
├── refs/
│   ├── heads/        ← local branches
│   ├── remotes/      ← remote-tracking branches
│   └── tags/         ← tags
├── HEAD              ← what branch/commit you're currently on
├── COMMIT_EDITMSG    ← last commit message
├── config            ← repo-specific config
├── index             ← the staging area (index)
└── logs/
    └── refs/         ← reflog entries
```

**the `objects/` directory is everything.** git stores four types of objects, each identified by the SHA-1 hash of its content:

- **blob:** raw file contents (just the bytes, no filename)
- **tree:** a directory listing (maps filenames to blob/tree hashes)
- **commit:** a snapshot (points to a tree, has parent commits, has author/message)
- **tag:** an annotated tag (points to any object)

```bash
# look at any object
git cat-file -t 1a2b3c4d   # print type: blob/tree/commit/tag
git cat-file -p 1a2b3c4d   # print pretty content
```

when you `git commit`, git:
1. creates blobs for any changed files
2. creates trees representing the directory structure
3. creates a commit object pointing to the top-level tree and parent commit(s)
4. moves the branch pointer to the new commit

your branches (`refs/heads/main`) are just files containing a 40-character commit hash. that's it. a branch is just a named pointer to a commit.

```bash
cat .git/refs/heads/main
# 8f21da1c4a3b7d19e2f0985b4a23c1d5e6f7a890
```

---

## the index (staging area): the step everyone misunderstands

`git add` doesn't add to the commit. it adds to the **index** (also called the staging area). the index is a binary file (`.git/index`) that represents what the next commit will look like.

```
working directory → [ git add ] → index → [ git commit ] → repository
```

this three-stage model is actually powerful:
- you can modify 5 files but only stage 3 of them for a commit
- you can stage part of a file (`git add -p` for interactive partial staging)
- you can look at what's staged vs unstaged: `git diff` (unstaged) vs `git diff --cached` (staged)

```bash
# see exactly what will be in the next commit
git diff --cached

# interactively choose which hunks (sections) to stage
git add -p

# stage everything in a subdirectory
git add src/auth/

# unstage something (remove from index without touching working directory)
git restore --staged auth.go
```

---

## commits: writing history people can actually read

commits are the permanent record. once pushed to a shared branch, they're part of the project's history forever. write them like they matter.

### conventional commits

```
feat(auth): add JWT refresh token rotation

Refresh tokens now rotate on each use. A new token is issued
with every refresh request, and the previous token is
immediately invalidated. This reduces the window of exposure
if a token is compromised.

The old behavior (static refresh tokens) was controlled by
REFRESH_TOKEN_ROTATION=false, which is now removed.

Closes #312
BREAKING CHANGE: clients must store and use the new refresh token returned in each response
```

**type:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `ci`, `revert`
**scope:** optional, what part of the codebase. `(auth)`, `(api)`, `(db)`, etc.
**description:** imperative mood, present tense. "add" not "added" or "adds"
**body:** explain *why*, not *what*. the diff shows what.
**footer:** breaking changes, closes, references

why conventional commits? because **semantic-release** can parse them automatically. it reads your git history and bumps the version number (major for BREAKING CHANGE, minor for feat, patch for fix) without human input. your changelog writes itself.

```bash
# install semantic-release
npm install --save-dev semantic-release

# in CI, after tests pass:
npx semantic-release
# reads commits since last tag
# determines next version
# creates git tag
# publishes release notes
# pushes to registry if configured
```

---

## branches: a decision tree

**GitHub Flow** (recommended for most web services):

```
main ──●────────────────────────────────●───────
        \                               ↑
feature  ●──●──●──●──●──●  (PR review)  ↑
                               └─────────┘
                               merge after CI + approval
```

rules:
1. `main` is always deployable
2. feature branches from `main`
3. PR required to merge
4. CI must pass before merge
5. merge → auto-deploy (or button-press deploy)

**naming conventions that actually help:**

```
feature/USER-123-add-oauth-login
fix/null-pointer-charge-processor
chore/upgrade-go-1.26
hotfix/memory-leak-session-store
refactor/extract-user-service
```

including ticket numbers makes your git history searchable and connects commits to the issue tracker automatically (GitHub links them).

---

## the git commands you need to actually know

### rebase: reapply commits on a new base

```bash
# you're on feature/my-feature, main has moved forward
git fetch origin
git rebase origin/main
```

rebase takes your commits and *replays* them on top of the new base. each commit gets a new hash (because the parent changed). the history is linear and clean, no merge commits.

**interactive rebase: cleaning up before a PR**

```bash
git rebase -i HEAD~4   # rewrite the last 4 commits
```

```
pick a1b2c3d feat: add user model
pick d4e5f6g wip halfway there
pick g7h8i9j fix typo lol
pick j0k1l2m add actual tests
```

change to:
```
pick a1b2c3d feat: add user model
squash d4e5f6g wip halfway there   # squash into previous
squash g7h8i9j fix typo lol        # squash into previous
squash j0k1l2m add actual tests    # squash into previous
```

result: one clean commit "feat: add user model" with all the content. this is the history your teammates see in the PR. your messy drafts are invisible.

**RULE:** never interactive-rebase commits that are already on a shared branch (anything other people have pulled). rewriting those commits and force-pushing will create chaos.

```bash
# force push your feature branch after rebase (it's YOUR branch)
git push --force-with-lease

# --force-with-lease is safer than --force:
# it checks if the remote has been updated since you last fetched
# and refuses to push if it has, preventing accidental overwrite
```

### cherry-pick: surgical commit transfer

```bash
# apply one specific commit to current branch
git cherry-pick abc1234

# apply a range
git cherry-pick abc1234..def5678

# cherry-pick without committing (leaves changes staged)
git cherry-pick --no-commit abc1234
```

use case: a fix was committed to the wrong branch. or a feature branch has one commit you want in main without the whole branch. cherry-pick copies the *diff* introduced by that commit and applies it as a new commit on your current branch.

### reset vs revert: the important distinction

```bash
# RESET: moves the branch pointer backward. rewrites history.
git reset --soft HEAD~1    # undo commit, keep changes staged
git reset --mixed HEAD~1   # undo commit, keep changes unstaged (DEFAULT)
git reset --hard HEAD~1    # undo commit, DISCARD changes (dangerous)

# REVERT: creates a new commit that undoes the changes. safe for shared branches.
git revert abc1234         # new commit that undoes abc1234
git revert HEAD~3..HEAD    # new commits undoing the last 3 commits
```

**use reset for:** your own local branch cleanup before pushing
**use revert for:** undoing commits that are already on shared branches or main

### bisect: binary search for bugs

the most underused powerful git command. you have a bug. you know it wasn't there last week. you have 200 commits since last week.

```bash
git bisect start
git bisect bad                           # current code has the bug
git bisect good v1.2.3                   # this tagged version was fine

# git checks out the midpoint commit
# test if the bug exists
git bisect good   # or
git bisect bad

# repeat ~8 times (log2(200) ≈ 7.6)
# git tells you: abc1234 is the first bad commit
```

automate it:

```bash
# test script returns 0 = good, nonzero = bad
git bisect run ./scripts/test_bug.sh
# git will binary search automatically
git bisect reset    # when done
```

---

## CI/CD for Go projects: the actual pipeline

picking up from where Part 1 left off, here's a complete GitHub Actions workflow for a Go backend service:

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.26'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─────────────────────────────────────────────
  # JOB 1: code quality (fast, runs first)
  # ─────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true  # automatically caches Go module cache
      
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m

  # ─────────────────────────────────────────────
  # JOB 2: tests (parallel with lint)
  # ─────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      
      - name: Run migrations
        run: go run ./cmd/migrate/main.go up
        env:
          DATABASE_URL: postgres://test:testpass@localhost:5432/testdb
      
      - name: Run tests with race detector
        run: go test -race -coverprofile=coverage.out ./...
        env:
          DATABASE_URL: postgres://test:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage below 70% threshold"
            exit 1
          fi
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out

  # ─────────────────────────────────────────────
  # JOB 3: security scanning
  # ─────────────────────────────────────────────
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Gosec security scanner
        uses: securego/gosec@master
        with:
          args: ./...
      
      - name: Check for known vulnerabilities
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  # ─────────────────────────────────────────────
  # JOB 4: build + push Docker image (main branch only)
  # ─────────────────────────────────────────────
  build:
    needs: [lint, test, security]   # only runs if all above pass
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build-push.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,format=long
            type=raw,value=latest
      
      - uses: docker/build-push-action@v5
        id: build-push
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─────────────────────────────────────────────
  # JOB 5: deploy to staging
  # ─────────────────────────────────────────────
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          # update kubernetes deployment with new image
          kubectl set image deployment/myapp \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build.outputs.image-digest }}
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
      
      - name: Wait for rollout
        run: kubectl rollout status deployment/myapp --timeout=300s
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
```

**things in this pipeline worth calling out:**

**`-race` flag in tests:** Go's race detector finds data races at runtime. a data race is when two goroutines access the same memory concurrently and at least one is writing. race conditions are one of the hardest bug classes to find manually. run with `-race` in CI always.

**`govulncheck`:** scans your code and dependencies for known CVEs. it's smarter than dependency-level scanners, it only reports vulnerabilities in code that's actually called by your program, not every transitive dependency.

**image digest vs tag:** deploying by digest (`sha256:abc123...`) rather than tag (`latest`) ensures you deploy *exactly* the image you tested. tags are mutable (someone can push a new image with the same tag). digests are immutable.

---

## gitops: when your deployment config is also in git

**GitOps** means your infrastructure and deployment configuration is stored in a git repository, and changes to that repo automatically trigger deployment. it's the natural evolution of CI/CD for Kubernetes.

```
Application repo (code changes)
    │
    └── CI builds image → pushes to registry
                         → updates image tag in config repo

Config repo (Kubernetes manifests)
    │
    └── ArgoCD watches this repo
        └── detects change
            └── applies to cluster
```

tools: **ArgoCD** (most popular, declarative GitOps for Kubernetes) and **Flux** (alternative, CNCF project).

```yaml
# in your config repo: apps/myapp/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: ghcr.io/myorg/myapp@sha256:abc123...   # CI updates this line
```

your CI pipeline does a `git commit` to the config repo updating the image digest. ArgoCD sees the commit, syncs the cluster. the cluster state is always what's in git. to roll back: `git revert` the commit. to see deployment history: `git log`.

---

## pre-commit hooks: catch it before it's committed

commit hooks run locally before a commit is created. if the hook fails, the commit is rejected.

```bash
# .git/hooks/pre-commit  (must be executable: chmod +x)
#!/bin/sh
set -e

echo "Running pre-commit checks..."

# format check
if ! gofmt -l . | grep -q .; then
    echo "✓ gofmt"
else
    echo "✗ gofmt: please run 'gofmt -w .'"
    exit 1
fi

# vet
go vet ./...
echo "✓ go vet"

# run quick tests (no integration tests, too slow)
go test -short ./...
echo "✓ tests"
```

or use **pre-commit** (the tool, `pre-commit.com`) for managing hooks with a config file:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/dnephin/pre-commit-golang
    rev: v0.5.1
    hooks:
      - id: go-fmt
      - id: go-vet
      - id: go-unit-tests
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: check-yaml
      - id: detect-private-key
      - id: check-added-large-files
```

the `detect-private-key` hook is important, it prevents accidentally committing API keys or private keys. at every company I've seen, this has caught at least one person about to push credentials.

---

## the daily workflow: what a day actually looks like

**morning:**

```bash
git fetch origin                    # update remote-tracking branches
git log --oneline origin/main -10   # see what happened overnight
git checkout -b feature/PROJ-156-rate-limiting origin/main  # new branch
```

**during development:**

```bash
# commit often, clean up later
git add -p                          # stage specific hunks interactively
git commit -m "wip: rate limiter middleware compiles"

# ...more work...
git add src/middleware/ratelimit.go src/middleware/ratelimit_test.go
git commit -m "feat(api): add token bucket rate limiter"
```

**before opening a PR:**

```bash
git fetch origin
git rebase origin/main              # get latest main changes
git rebase -i HEAD~5               # squash/clean up your 5 commits

# push to remote
git push origin feature/PROJ-156-rate-limiting
# or if you've rebased (history changed):
git push --force-with-lease origin feature/PROJ-156-rate-limiting
```

**after CI passes and PR is approved:**

```bash
# the PR is merged on GitHub (squash merge or regular merge)
# delete local branch
git checkout main
git pull origin main
git branch -d feature/PROJ-156-rate-limiting
```

**oh no, production is broken:**

```bash
git checkout main
git pull

# option 1: revert the merge commit
git log --oneline -5    # find the merge commit hash
git revert -m 1 abc1234   # create a revert commit
git push origin main       # triggers CI + auto-deploy

# option 2: hotfix branch
git checkout -b hotfix/rate-limiter-panic
# fix the bug
git commit -m "fix(ratelimit): handle nil context in middleware"
git push origin hotfix/rate-limiter-panic
# open PR, expedited review, merge, deploy
```

---

## the underrated git commands: your secret weapons

```bash
# find which commit introduced a string
git log -S "functionName" --oneline

# show all commits that touched a file
git log --follow -p -- path/to/file.go

# what changed between two branches
git diff main...feature/my-branch

# who wrote each line of a file
git blame -L 10,20 main.go   # blame lines 10-20

# find commits by message
git log --grep="fix" --oneline

# create an alias
git config --global alias.lg "log --oneline --graph --decorate --all"
git lg    # now works everywhere
```

`git log --oneline --graph --decorate --all` is genuinely one of the most useful things to alias. it gives you a visual ASCII representation of your branch history.

---

## the stuff that will save you eventually

**`git reflog` saves lives.** every state HEAD has been in is recorded here. `git reset --hard` and lost commits? check the reflog.

```bash
git reflog
# a3f7bc2 HEAD@{0}: reset: moving to HEAD~3
# 8f21da1 HEAD@{1}: commit: the commit you need

git checkout 8f21da1         # get your commit back
# or
git reset --hard 8f21da1    # move branch back to there
```

**`git stash` is a clipboard for code.** half-done change and you need to switch context?

```bash
git stash push -m "WIP: auth middleware, need to fix token expiry"
# your working directory is clean

git stash list   # see all stashes
git stash pop    # apply most recent stash
git stash apply stash@{2}  # apply a specific one
```

**`git worktree` is the hidden gem.** multiple branches checked out simultaneously in different directories:

```bash
git worktree add ../myapp-hotfix hotfix/crash-fix
# ../myapp-hotfix now has the hotfix branch checked out
# your main directory keeps its current branch

# work in both simultaneously
# when done:
git worktree remove ../myapp-hotfix
```

---

*next: [05 API Design: REST vs GraphQL and Everything in Between](05-api-design.md)*

*sources: [git internals book](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) | [semantic-release](https://semantic-release.gitbook.io/) | [GitHub Actions docs](https://docs.github.com/actions)*
