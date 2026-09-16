---
name: docker-clean-architecture
description: Universal Docker containerization, multi-stage build optimization, security hardening, and Docker Compose orchestration skill. Directs the creation of production-grade, minimal-footprint, non-root, and high-performance container images and multi-container environments. Covers layer caching efficiency, multi-stage compile vs runtime separation, bridge/overlay network isolation, volume persistency, secrets handling, healthcheck definitions, and .dockerignore hygiene across any modern development stack. Triggers on "docker", "dockerfile", "docker compose", "containerize", "container security", "multi-stage build".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# CORE CONTAINERIZATION & ARCHITECTURAL PRINCIPLES

## Immutability and Ephemeral Lifecycle
Containers must be treated as completely disposable, immutable infrastructure. Any state or dynamic configuration generated at runtime must not reside within the container's writable layer, as this layer is destroyed upon container termination.
1. Ephemeral Container Execution: Design containers to start, stop, crash, or scale horizontally without data loss or manual host configuration, adhering to Twelve-Factor app principles.
2. Decoupled Service Architecture: Restrict each container to a single core responsibility or service process (e.g., decoupling application servers from database and cache daemons). Decoupling simplifies horizontal scaling, process isolation, and container reuse across environments.

## Minimal Attack Surface Strategy
Choosing minimal, secure base images reduces vulnerability density, decreases network transfer latency, and speeds up deployment pipelines.
1. Trusted Base Distributions: Base all container definitions on official, verified, or hardened base images. Prefer minimal distributions such as `alpine` (sub-6 MB runtime) or distroless images over full Linux distributions like Ubuntu or Debian.
2. Zero Unnecessary Toolchains: Exclude build tools, shell utilities, compilers, and text editors from final runtime images. Dropping unnecessary runtime binaries directly shrinks image footprint and mitigates privilege escalation vectors.

## Principle of Least Privilege (Non-Root Execution)
By default, Docker containers run processes with `root` privileges (UID 0), creating severe security risks if a container escape vulnerability is exploited.
1. Mandatory `USER` Instruction: Explicitly create a non-system dedicated group and user (e.g., `appuser` with an explicit non-root UID/GID), and set the active user context via the `USER` instruction prior to specifying the execution command.
2. Avoid Sudo and Privileged Execution: Prohibit installing `sudo` or executing application runtimes as root. Ensure directory permissions inside `WORKDIR` are granted to the non-root user during image build time.

## Layer Caching Efficiency
Docker builds images sequentially layer by layer, caching each instruction. An instruction change invalidates the build cache for all subsequent instructions.
1. Strategic Instruction Ordering: Order `Dockerfile` instructions strictly from least-frequently changed to most-frequently changed. Place system package installation and dependency manifests (e.g., `package.json`, `go.mod`, `Cargo.toml`) before copying application source code.
2. Atomic Command Chains: Combine related package manager operations into single `RUN` layers (e.g., `apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*`) to prevent stale cache bugs and eliminate temporary package index files from stored image layers.

---

# MULTI-STAGE BUILDS & IMAGE SIZE OPTIMIZATION

## Compile vs. Runtime Separation
Multi-stage builds utilize multiple `FROM` instructions within a single `Dockerfile` to separate the build-time environment from the production runtime.
1. Isolated Build Stages: Name distinct build stages using `FROM base AS <stage_name>` syntax. Install compilers, SDKs, native header packages, and build tools exclusively inside build stages.
2. Selective Artifact Extraction: Use `COPY --from=<stage_name>` to copy compiled binaries or pruned production dependencies into a clean, minimal runtime stage (such as `alpine` or `scratch`). All intermediate build toolchains and source files are discarded, leaving a lightweight runtime container.

## Eliminating Build Toolchains and Dependencies
Production images must contain only the explicit artifacts required for runtime execution.
1. Dependency Pruning: Execute production dependency pruning (e.g., `npm ci --only=production` or `go build -ldflags="-w -s"`) inside intermediate build stages.
2. Static Binary Target: When building statically compiled binaries (Go, Rust), compile with static linking and copy the output binary directly into an empty `scratch` base image for a zero-OS footprint.

## BuildKit and Advanced Cache Mounts
Docker BuildKit introduces parallel execution and specialized cache mount directives that drastically accelerate build pipelines without persisting build cache in output image layers.
1. Package Manager Mounts: Utilize `--mount=type=cache` in `RUN` instructions to persist package manager download caches (e.g., `/root/.npm`, `/root/.cache/go-build`, `/var/cache/apt`) across build runs.
2. Build Secrets Isolation: Use `--mount=type=secret,id=MY_SECRET` to safely expose API keys or access tokens to build commands without baking credentials into image layers or build arguments.

---

# WORKFLOW: CONTAINERIZING AN APPLICATION

Engineers and AI assistants must follow this 5-stage sequential workflow when containerizing any new or existing application service:

## Stage 1: Build Context Hygiene (.dockerignore)
Create a `.dockerignore` file at the root of the build context before writing the `Dockerfile`. Exclude local dependency directories (`node_modules`, `vendor`), version control metadata (`.git`), local environment files (`.env*`), build outputs (`dist`, `target`), logs, and documentation.

## Stage 2: Base Image Selection and Pinning
Select an official, minimal base image tailored to the application runtime. Pin base images using specific version tags paired with immutable SHA-256 digests (`image:tag@sha256:digest`) to prevent unexpected upstream changes and supply chain tampering.

## Stage 3: Multi-Stage Layer Ordering
Structure the `Dockerfile` into named stages (e.g., `deps`, `builder`, `runner`). Order instructions to copy dependency manifests first, run package installation with BuildKit cache mounts, copy application code, build production artifacts, and copy final assets into a non-root runtime container.

## Stage 4: Process and Signal Handling Configuration
Configure a lightweight init system (such as `tini` or `dumb-init`) as `ENTRYPOINT` to properly reap zombie processes and forward OS process signals (`SIGTERM`, `SIGINT`) to application workers. Define execution commands using JSON array syntax (`CMD ["node", "server.js"]` or `CMD ["./main"]`).

## Stage 5: Compose Orchestration and Verification
Define multi-container runtime topologies inside `docker-compose.yml`. Enforce private bridge networks, healthcheck conditions, volume persistency rules, non-root execution checks, and container resource limits. Validate using `docker compose config` and test container builds using BuildKit.

---

# DOCKER COMPOSE & MULTI-CONTAINER ORCHESTRATION

## Declarative Service Topology
Docker Compose manages multi-container applications using the modern Compose Specification format, replacing obsolete `version` top-level headers. Services represent application components that are deployed, scaled, and isolated declaratively.

## Private Bridge Networks and Isolation
User-defined bridge networks provide automatic internal DNS resolution and network isolation between containers.
1. Internal DNS Discovery: Containers attached to the same user-defined bridge network communicate securely using service names as network hostnames.
2. Restricting Public Port Exposure: Only publish ports (`ports:`) on frontend or edge proxy containers that require external host access. Keep backend databases, caches, and internal workers unexposed to the host network.

## Dependency Chains and Healthchecks
Relying solely on container start order causes race conditions because a started container may not yet be ready to accept connections.
1. Healthcheck Definitions: Define explicit `healthcheck` routines for database and cache services using native CLI checks (e.g., `pg_isready` or `redis-cli ping`).
2. Conditional Startup Sequences: Use `depends_on` with `condition: service_healthy` to delay dependent service startup until upstream infrastructure is fully operational.

## Persistent Volumes vs. Development Bind Mounts
Docker provides distinct storage mount mechanisms tailored for production stability and development workflows.
1. Named Volumes for Production Data: Use managed named volumes (`volumes:`) for database engines (PostgreSQL, Redis, MySQL) to achieve native host file performance and state persistence independent of container lifecycles.
2. Bind Mounts for Hot-Reloading: Use read-only or targeted bind mounts (`./src:/app/src:ro`) strictly in development environment overrides to enable live code reloading without rebuilding containers.

---

# ADVANCED PATTERNS AND CODE EXAMPLES

## a) Multi-Stage Production Dockerfile (Node.js/TypeScript)

### INCORRECT / ANTIPATTERN
A single-stage Dockerfile running as root, copying all files without layer caching, containing dev dependencies and compilers in production, and using shell-form CMD without signal handling.
```dockerfile
# INCORRECT: Single stage, running as root, bloated build context
FROM node:latest

WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

EXPOSE 3000
CMD node dist/index.js
```

### CORRECT / CLEAN
A hardened multi-stage Dockerfile using Alpine, non-root user, BuildKit cache mounts, tini init process, and strict layer ordering.
```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Dependency installation
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --include=dev

# Stage 2: Application compilation
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && \
    npm prune --omit=dev

# Stage 3: Minimal production runtime
FROM node:22-alpine AS runner
WORKDIR /app

# Install tini for proper signal handling and zombie process reaping
RUN apk add --no-cache tini

# Create dedicated non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only pruned production dependencies and compiled output
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./package.json

USER appuser

EXPOSE 3000
ENV NODE_ENV=production

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

## b) Production-Ready docker-compose.yml

### INCORRECT / ANTIPATTERN
Obsolete format, exposed database ports to the host, missing healthchecks, plain-text environment credentials, and no network isolation.
```yaml
# INCORRECT: Insecure Compose configuration
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
      - DB_PASS=secret123
  db:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=secret123
```

### CORRECT / CLEAN
Modern Compose Specification setup featuring private bridge networks, conditional healthchecks, named persistent volumes, and Docker secrets.
```yaml
name: production-stack

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runner
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL_FILE: /run/secrets/db_url
      REDIS_URL: redis://cache:6379
    secrets:
      - db_url
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - frontend_net
      - backend_net
    read_only: false
    user: "10001:10001"

  db:
    image: postgres:17-alpine@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c
    restart: unless-stopped
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d app_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  cache:
    image: redis:7-alpine@sha256:54320b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend_net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

networks:
  frontend_net:
    driver: bridge
  backend_net:
    driver: bridge
    internal: true

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_url:
    file: ./secrets/db_url.txt
```

## c) Restrictive .dockerignore File
A comprehensive `.dockerignore` file protecting build context hygiene, accelerating build transfer speed, and preventing credential leaks.
```ignore
# Version control
.git
.gitignore
.gitattributes

# Package manager and local dependencies
node_modules
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Environment and secret files
.env
.env.*
!.env.example
secrets/
*.pem
*.key

# Build outputs and temporary caches
dist
build
out
.next
.cache
coverage
.tmp

# IDE, OS, and documentation files
.idea
.vscode
*.swp
.DS_Store
Thumbs.db
README.md
*.pdf
Dockerfile*
docker-compose*.yml
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

## 1. Running Containers as Root User
* Anti-pattern: Omitting the `USER` instruction, allowing the application to execute as UID 0 inside the container.
* Consequence: Container breakout vulnerabilities allow attackers to gain root access on the host system.
* Prevention: Create a non-root service user and set `USER appuser` prior to the runtime entrypoint.

## 2. Using Mutable 'latest' Image Tags
* Anti-pattern: Specifying `FROM node:latest` or `FROM postgres:latest` in Dockerfiles or Compose files.
* Consequence: Builds become non-deterministic, causing unexpected production outages when upstream base images are updated.
* Prevention: Pin explicit image version tags and SHA-256 immutable digests (e.g., `node:22-alpine@sha256:...`).

## 3. Omitting .dockerignore File
* Anti-pattern: Building images without a `.dockerignore` file at the root of the project.
* Consequence: Local `node_modules`, `.git` history, and sensitive `.env` credential files are sent to the Docker daemon build context, inflating image size and leaking secrets.
* Prevention: Maintain a strict `.dockerignore` file excluding VCS files, dependencies, build outputs, and credentials.

## 4. Baking Secrets into Image Layers
* Anti-pattern: Using `ENV` or `ARG` to pass database passwords, API keys, or private tokens during image build time.
* Consequence: Environment variables persist in image layer metadata and can be extracted via `docker history` or `docker inspect`.
* Prevention: Use BuildKit secret mounts (`RUN --mount=type=secret`) for build-time secrets and Docker secrets or file-based runtime mounts for production deployments.

## 5. Multi-Process Containers Without Init Systems
* Anti-pattern: Running PID 1 application processes directly without an init daemon, or running multiple processes via shell scripts.
* Consequence: PID 1 does not handle default Linux signal forwarding or harvest orphaned child processes, resulting in resource leaks and ungraceful container shutdowns.
* Prevention: Enforce one service process per container, use `tini` or `dumb-init` as `ENTRYPOINT`, and execute applications using JSON array syntax.
