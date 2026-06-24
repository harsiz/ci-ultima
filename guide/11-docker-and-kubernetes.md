# 11 Docker and Kubernetes: Containers and the World They Built 🐳!
### because "it works on my machine" died in 2013..

---

containers are one of those technologies where once you understand them, you can't imagine going back. the problems they solve are real: environment inconsistency, deployment complexity, dependency conflicts, and the terrifying experience of manually configuring production servers.

Docker standardized how applications get packaged. Kubernetes standardized how they get run at scale. together they define how the overwhelming majority of backend services are deployed in 2026.

---

## what a container actually is

a container is **not a virtual machine.** this is the most important thing to understand.

a VM runs a full operating system with its own kernel, CPU virtualization, and memory allocation. it's heavy (gigabytes of disk, minutes to start, significant overhead).

a container shares the host's kernel. it's a set of processes with isolated:
- **filesystem** (via union filesystems)
- **network** (virtual network interface)
- **process namespace** (container can't see host's processes)
- **resource limits** (CPU and memory limits via cgroups)

```
Virtual Machines:                 Containers:
┌─────────────────────────┐      ┌─────────────────────────┐
│  App A  │  App B        │      │  App A  │  App B        │
├─────────┼───────────────┤      ├─────────┼───────────────┤
│  OS A   │  OS B         │      │ Container│ Container    │
│ (full   │ (full kernel) │      │ Runtime  │ Runtime      │
│ kernel) │               │      ├──────────┴──────────────┤
├─────────┴───────────────┤      │    Host OS Kernel        │
│    Hypervisor            │      ├──────────────────────────┤
├──────────────────────────┤      │    Physical Hardware      │
│    Physical Hardware      │      └──────────────────────────┘
└──────────────────────────┘
```

containers start in milliseconds. VMs start in minutes. containers are megabytes overhead. VMs are gigabytes. containers run thousands per host. VMs run dozens.

the trade-off: containers share the kernel, so a kernel vulnerability potentially affects all containers on the host. VMs provide stronger isolation (different kernels). for most applications, container isolation is sufficient.

---

## Docker: packaging applications

Docker popularized containers (the underlying technology, Linux namespaces + cgroups, existed before Docker). Docker added:
- **Dockerfile:** a standardized way to define container images
- **Docker Hub:** a registry for sharing images
- **Docker CLI:** a developer-friendly tool for building and running containers

### writing a good Dockerfile for Go

```dockerfile
# ─────────────────────────────────────────────
# Stage 1: build
# ─────────────────────────────────────────────
FROM golang:1.26-alpine AS builder

WORKDIR /app

# copy dependency files first (layer caching: only re-download if go.mod/go.sum change)
COPY go.mod go.sum ./
RUN go mod download

# copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \  # strip debug info, reduce binary size
    -o /app/server ./cmd/server

# ─────────────────────────────────────────────
# Stage 2: runtime image (tiny)
# ─────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12 AS runtime
# distroless: no shell, no package manager, minimal attack surface

# create non-root user
COPY --from=builder /etc/passwd /etc/passwd
USER nobody

COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**multi-stage builds** are essential. the first stage has all Go tooling (large). the second stage is just the compiled binary and a minimal base image.

- `golang:1.26-alpine` is ~300MB
- `gcr.io/distroless/static-debian12` is ~2MB
- your final image: ~2MB + binary size (~15-20MB for a typical Go service)

vs naive approach (copy everything into golang:1.26-alpine): ~330MB image.

**`CGO_ENABLED=0`** disables CGO (C bindings). pure Go binary. important for distroless images which have no C standard library.

**`-ldflags="-w -s"`** strips debug information and symbol table. reduces binary size ~30%.

### layer caching: the thing that makes builds fast

Docker builds images layer by layer. each instruction (`COPY`, `RUN`, etc.) creates a layer. layers are cached. if nothing changed, the layer is reused.

```dockerfile
# BAD: one big COPY copies everything including source
# any change to any file invalidates this layer and everything after
COPY . .
RUN go mod download && go build .

# GOOD: separate dependency download from source copy
COPY go.mod go.sum ./    # layer 1: only changes when dependencies change
RUN go mod download       # layer 2: only re-runs when layer 1 changes

COPY . .                  # layer 3: changes when any source file changes
RUN go build .           # layer 4: re-runs when layer 3 changes
```

on a typical development machine, the `go mod download` step saves 30-60 seconds per build because dependencies rarely change.

### what not to do in Docker

```dockerfile
# NEVER: run as root
USER root  # implicit if you don't specify USER
# an attacker who escapes the container process has root on the host

# NEVER: embed secrets in the image
ENV DATABASE_PASSWORD=mysecretpassword  # visible in docker inspect, docker history

# NEVER: use :latest tag for base images
FROM golang:latest  # what version is this? changes without warning
# use specific versions:
FROM golang:1.26.4-alpine3.20

# NEVER: put everything in one huge layer
RUN apt-get install -y package1 package2 package3 && \
    pip install thing1 thing2 && \
    npm install && \
    ...
# clean up in the SAME RUN layer to not bloat the image:
RUN apt-get install -y package1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### docker-compose for local development

```yaml
# docker-compose.yml
version: '3.9'

services:
  api:
    build:
      context: .
      target: builder  # use build stage for development (with Go tools)
    ports:
      - "8080:8080"
    volumes:
      - .:/app  # mount source for hot-reload with air
    environment:
      - DATABASE_URL=postgres://dev:dev@postgres:5432/devdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: air  # hot-reload tool for Go development

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d devdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  postgres_data:
```

`docker compose up` gives you the full local environment in one command. no "do you have PostgreSQL installed? which version? did you create the database?" just works.

**air** is a live-reload tool for Go. it watches for file changes and restarts your Go server automatically. the development loop: save a file → binary recompiles → server restarts. typically 1-2 seconds.

---

## Kubernetes: orchestrating containers at scale

Kubernetes (K8s) is what you use when you need to:
- run multiple instances of your service (for throughput or availability)
- automatically restart containers that crash
- deploy new versions without downtime
- scale up and down based on load
- manage secrets and configuration

Kubernetes is complex. genuinely, deeply complex. it has 50+ resource types, its own RBAC model, a networking model that's different from everything you've seen, and enough concepts to fill a book. this chapter covers the concepts you need, not a comprehensive guide.

### the core objects

**Pod:** the smallest deployable unit. one or more containers that share network and storage. usually: one container per pod.

**Deployment:** manages a set of identical pods. ensures N replicas are running. handles rolling updates and rollbacks.

**Service:** a stable network endpoint for a set of pods. pods are ephemeral (they get new IPs when restarted). Services provide a stable IP and DNS name.

**Ingress:** routes external HTTP/HTTPS traffic to Services. typically backed by Nginx or Traefik.

**ConfigMap:** non-secret configuration. mounted as env vars or files.

**Secret:** sensitive configuration (passwords, API keys). base64-encoded (not encrypted by default, but can be with sealed-secrets or external-secrets).

```yaml
# deployment.yaml, runs your Go service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3  # run 3 instances
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: ghcr.io/myorg/myapp@sha256:abc123...  # specific digest, not :latest
        ports:
        - containerPort: 8080
        
        # REQUIRED: resource limits
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"      # 100 millicores = 0.1 CPU
          limits:
            memory: "128Mi"
            cpu: "500m"
        
        # REQUIRED: health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3  # restart after 3 failures
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
          failureThreshold: 3  # stop sending traffic after 3 failures
        
        # environment from secrets
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
        - name: GOMEMLIMIT
          value: "115MiB"  # ~90% of memory limit (128Mi)
        - name: GOMAXPROCS
          value: "1"        # match CPU limit (0.5 rounds to 1)
      
      # security context: don't run as root
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534    # nobody
        readOnlyRootFilesystem: true
```

**liveness vs readiness probes:**
- **liveness:** is the container healthy? if this fails, Kubernetes restarts the container. use for detecting deadlocks, infinite loops.
- **readiness:** is the container ready to receive traffic? if this fails, Kubernetes removes it from the Service's endpoints but doesn't restart it. use for "still starting up" or "temporarily overloaded."

```go
// your Go health endpoints
mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    // liveness: is the process alive?
    // just return 200 if the process is running
    w.WriteHeader(http.StatusOK)
})

mux.HandleFunc("GET /ready", func(w http.ResponseWriter, r *http.Request) {
    // readiness: can we serve traffic?
    // check database connection, cache connection, etc.
    if err := db.PingContext(r.Context()); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{"error": "database unavailable"})
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

### resource limits: non-negotiable

**always set resource limits.** this is one of the most common mistakes in Kubernetes.

without limits:
- one container with a memory leak can OOM-kill other containers on the same node
- one CPU-hungry job starves other services
- Kubernetes can't efficiently schedule pods across nodes

`resources.requests` = what the pod needs to be scheduled (Kubernetes finds a node with this much available)
`resources.limits` = the maximum it can use

for memory: set `limits = requests × 2` as a starting point (your service can burst but not infinitely). adjust based on profiling.

for CPU: requests and limits can differ more. CPU throttling (vs OOM kill for memory) is less catastrophic.

### horizontal pod autoscaler (HPA)

automatically scale pods based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # scale up when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 100Mi
```

the HPA watches metrics and adjusts replica count. if CPU goes above 70%, more pods are created. if CPU drops, pods are removed. scale-down happens gradually (to avoid thrashing).

### rolling deployments: zero downtime updates

Kubernetes deployments do rolling updates by default:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # create 1 extra pod during update
      maxUnavailable: 0   # never have fewer than desired replicas
```

update sequence for `maxSurge: 1, maxUnavailable: 0` with 3 replicas:
1. create 1 new pod (4 total, 1 not ready)
2. new pod becomes ready (4 ready)
3. terminate 1 old pod (3 ready)
4. repeat until all pods are updated

`maxUnavailable: 0` means traffic never drops below full capacity. `maxSurge: 1` means you briefly have extra capacity during updates. the right settings depend on your cost sensitivity and traffic patterns.

### secrets management: doing it right

Kubernetes Secrets are base64-encoded, not encrypted. anyone with `kubectl get secret` access can decode them.

**better options:**
- **Sealed Secrets** (Bitnami): encrypt secrets with a cluster key, safe to commit encrypted secrets to git
- **External Secrets Operator**: sync secrets from AWS Secrets Manager / GCP Secret Manager / HashiCorp Vault
- **Vault Agent Injector**: HashiCorp Vault injects secrets into pods as files

```yaml
# External Secrets Operator example
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: myapp-secrets  # creates a regular Kubernetes Secret
  data:
  - secretKey: database-url
    remoteRef:
      key: myapp/production/database-url  # AWS Secrets Manager key
```

---

## supply chain security: the 2025-2026 focus

the SolarWinds attack and XZ Utils backdoor demonstrated that the software supply chain is an attack surface. in 2025-2026, Docker and Kubernetes ecosystems now support:

**Sigstore/cosign for image signing:**

```bash
# sign your image
cosign sign --key cosign.key ghcr.io/myorg/myapp@sha256:abc123

# verify before deploying
cosign verify --key cosign.pub ghcr.io/myorg/myapp@sha256:abc123
```

**SBOMs (Software Bill of Materials):**

```bash
# generate SBOM for your image (lists every package inside)
syft ghcr.io/myorg/myapp:latest -o spdx-json=sbom.json

# scan for vulnerabilities
grype sbom:sbom.json
```

**Admission Controllers (policy enforcement):**

```yaml
# Kyverno policy: require signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  rules:
  - name: check-image-signature
    match:
      resources:
        kinds: [Pod]
    verifyImages:
    - image: "ghcr.io/myorg/*"
      key: |-
        -----BEGIN PUBLIC KEY-----
        ...your cosign public key...
        -----END PUBLIC KEY-----
```

---

## the "do you actually need Kubernetes" question

Kubernetes is complex and operational overhead is real. honestly assess whether you need it:

**you probably need Kubernetes if:**
- you have multiple services (microservices architecture)
- you need auto-scaling based on load
- you have strict availability requirements (zero-downtime deployments)
- your team has the operational expertise (or you use a managed service like GKE, EKS, AKS)

**you probably don't need Kubernetes if:**
- you have one or two services
- your traffic is steady and predictable
- a single server handles your load comfortably
- your team is small and DevOps expertise is limited

**alternatives that might be better:**
- **Railway / Render / Fly.io:** PaaS that handles deployment without Kubernetes complexity. deploy from git, automatic HTTPS, reasonable pricing.
- **AWS ECS / Google Cloud Run:** container orchestration without full Kubernetes. simpler but less flexible.
- **single server + systemd:** for services that don't need horizontal scaling, a single well-configured VPS is often the right answer.

Kubernetes is the right tool for the right scale. using it for a personal project with 100 users is carrying a sledgehammer to a thumbtack.

---

*next: [12 Frontend for Backend Devs: React, Angular, and What You Actually Need to Know](12-frontend-for-backend-devs.md)*

*sources: [Kubernetes best practices 2026](https://www.cloudzero.com/blog/kubernetes-best-practices/) | [Docker best practices 2026](https://thinksys.com/devops/docker-best-practices/) | [Supply chain security](https://www.cncf.io/blog/2026/01/19/top-28-kubernetes-resources-for-2026-learn-and-stay-up-to-date/)*
