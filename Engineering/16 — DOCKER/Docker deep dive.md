---
tags: [docker, containers, deployment]
status: not-started
---

# Docker deep dive

> **What this is:** the full version of containers — images, layers, networking, volumes, multi-stage builds, and the specific problems that hit data and ML workloads.
> **Why you care:** every deployment in this stack is a container. The basics are in [[Dev environment - Git, Docker, CLI]]; this is everything after that.

---

## The idea in plain English

Your code needs a specific Python version, specific libraries, a specific ODBC driver, and specific environment variables. Getting all of that identical on your laptop, a colleague's Mac, the CI server, and a cloud VM is the oldest problem in software.

A container is **the whole environment, packaged as a file**. Not a copy of a computer — a copy of everything *above* the operating system kernel.

### Container vs. virtual machine

```
VIRTUAL MACHINE                    CONTAINER
┌─────────────────────┐            ┌─────────────────────┐
│  Your app           │            │  Your app           │
│  Libraries          │            │  Libraries          │
│  Guest OS  (~2 GB)  │            │  ─────────────────  │  ← shares the host kernel
│  Hypervisor         │            │  Container runtime  │
│  Host OS            │            │  Host OS            │
└─────────────────────┘            └─────────────────────┘
   Boots in ~30s                      Starts in ~0.1s
   Gigabytes                          Megabytes
```

A VM carries an entire operating system. A container shares the host's kernel and carries only what's above it. That's why containers start instantly and why you can run fifty of them on a laptop.

> **The consequence you must remember: containers share the host kernel.** Linux containers need a Linux kernel. On Windows, Docker Desktop quietly runs a Linux VM to provide one — which is why your first build is slow and why file access to mounted Windows folders is sluggish.

---

## 1. Images and layers

An **image** is a stack of read-only layers. Each instruction in your `Dockerfile` creates one.

```dockerfile
FROM python:3.12-slim          # layer 1  (~120 MB)
WORKDIR /app                   # layer 2  (~0)
COPY pyproject.toml uv.lock ./ # layer 3  (~4 KB)
RUN uv sync --frozen           # layer 4  (~800 MB)  ← the expensive one
COPY api/ ./api/               # layer 5  (~200 KB)
```

A **container** is that stack plus one thin writable layer on top. Delete the container, the writable layer goes; the image is untouched.

### The caching rule — the most valuable thing in this note

Docker reuses a cached layer **only if that instruction and everything before it is unchanged.** One change invalidates that layer and every layer below it.

```dockerfile
# ✗ SLOW — every code edit reinstalls PyTorch (3+ minutes)
COPY . .
RUN uv sync --frozen

# ✓ FAST — code edits only rebuild layer 5 (2 seconds)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen
COPY api/ ./api/
```

> **Order your Dockerfile from least-frequently-changed to most.** Base image → system packages → Python dependencies → your code. Your code changes fifty times a day; your dependencies change monthly. Put them in that order and you stop waiting.

### `.dockerignore`

Everything in the build directory gets sent to the Docker daemon before the build starts. Without a `.dockerignore`, that's your `.venv/`, `.git/`, and your entire `data/` folder.

```
.venv/
.git/
data/
mlruns/
__pycache__/
*.pyc
*.pth
notebooks/
.pytest_cache/
```

> Symptom of a missing `.dockerignore`: "Sending build context to Docker daemon 4.2GB" and a build that takes minutes before it does anything.

---

## 2. Multi-stage builds

Build tools are needed to *build* and not to *run*. A multi-stage build compiles in one image and copies only the result into a clean one.

```dockerfile
# ---------- stage 1: build ----------
FROM python:3.12 AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# ---------- stage 2: runtime ----------
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv     # ← only the result crosses over
ENV PATH="/app/.venv/bin:$PATH"
COPY api/ ./api/
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Compilers, headers and caches stay in stage 1 and never reach the final image. Typical saving: **1.2GB → 350MB**.

This matters even more for [[C++ for this stack]] and [[Rust for this stack]], where the toolchain is enormous and the output is a single binary:

```dockerfile
FROM rust:1.83 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/ingest /usr/local/bin/ingest
CMD ["ingest"]
```

> A Rust build image is ~1.5GB. The final image here is ~80MB, most of which is Debian. With `FROM scratch` and a static build it can be under 10MB.

### Choosing a base image

| Base | Size | Use |
|---|---|---|
| `python:3.12` | ~1 GB | Building only |
| `python:3.12-slim` | ~120 MB | **Default for runtime** |
| `python:3.12-alpine` | ~50 MB | ⚠️ Avoid for data work — see below |
| `nvidia/cuda:12.4-runtime` | ~2 GB | GPU inference ([[CUDA and GPU programming]]) |
| `debian:bookworm-slim` | ~75 MB | Compiled binaries |
| `scratch` | 0 | Fully static binaries |

> **Don't use Alpine for Python data work.** Alpine uses musl instead of glibc, so pip can't use the standard pre-built wheels for numpy, pandas, scipy or PyTorch — it compiles them from source. Builds go from 30 seconds to 30 minutes, and the result is often *slower*. `slim` is the right default.

---

## 3. Running containers

```bash
docker build -t retail-api:v1 .
docker run -p 8000:8000 --env-file .env retail-api:v1

docker run -d --name api retail-api:v1        # detached
docker ps                                      # what's running
docker ps -a                                   # including dead ones
docker logs -f api                             # follow logs
docker exec -it api bash                       # shell inside a RUNNING container
docker stop api && docker rm api
docker stats                                   # live CPU/memory
```

### Debugging a container that dies instantly

```bash
docker logs <container_id>                     # 90% of the time the answer is here
docker run -it --entrypoint bash retail-api:v1 # get a shell instead of running the app
docker history retail-api:v1                   # which layer made it huge
```

> `docker exec` needs a *running* container. If yours exits immediately, use `--entrypoint bash` to get inside a fresh one and look around.

### Resource limits

```bash
docker run --memory=2g --cpus=1.5 retail-api:v1
```

> **Set memory limits locally to match production.** A container with no limit will happily use all your RAM, work perfectly, and then get OOM-killed in Azure where it has 2GB. Reproduce the constraint on your machine and find out early.

---

## 4. Networking

Three facts that solve most problems:

**1. `localhost` inside a container means that container.** Not your machine, not another container. This is the single most common container networking mistake.

**2. Containers on the same user-defined network reach each other by name.**

```bash
docker network create retail-net
docker run -d --name api       --network retail-net retail-api:v1
docker run -d --name dashboard --network retail-net -p 8501:8501 retail-dashboard:v1
# inside dashboard, the API is at http://api:8000
```

**3. Port publishing is `-p HOST:CONTAINER`.**

```bash
-p 8000:8000    # localhost:8000 → container 8000
-p 3000:8000    # localhost:3000 → container 8000
```

> And your app must bind to `0.0.0.0`, not `127.0.0.1`. Binding to localhost inside a container means "reachable only from inside this container" — publishing the port won't help. `--host 0.0.0.0` for uvicorn, `--server.address=0.0.0.0` for Streamlit.

---

## 5. Volumes and data

A container's writable layer dies with the container. To keep data, mount something.

```bash
# named volume — Docker manages it. For databases.
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# bind mount — a host folder. For development.
docker run -v "$(pwd)/api:/app/api" retail-api:v1

# read-only
docker run -v "$(pwd)/models:/app/models:ro" retail-api:v1
```

| | Named volume | Bind mount |
|---|---|---|
| Managed by | Docker | You |
| Use for | Database data, model caches | Live-reloading source in dev |
| Portable | Yes | No — depends on host paths |

> **Bind-mount your source in development** so you don't rebuild on every edit. **Never bind-mount source in production** — the image should be self-contained and immutable. If prod depends on a host folder, it isn't reproducible.

> **Windows note:** bind mounts into WSL2 are slow for many small files. Keep your repo *inside* the WSL filesystem, not on `/mnt/c/`, if builds feel sluggish.

---

## 6. Docker Compose

Several containers, one file, one command.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  dashboard:
    build: ./dashboard
    ports: ["8501:8501"]
    environment:
      API_URL: http://api:8000          # ← service name, not localhost
    depends_on: [api]

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: devonly
      POSTGRES_DB: retail
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```

```bash
docker compose up --build          # start everything
docker compose up -d               # detached
docker compose logs -f api         # one service's logs
docker compose down                # stop
docker compose down -v             # stop AND delete volumes (destroys data)
```

> **`depends_on` alone only waits for the container to *start*, not to be *ready*.** Postgres takes a few seconds to accept connections, so your API starts, fails to connect, and exits. Use `condition: service_healthy` with a healthcheck, as above — this is the fix for the classic "works on the second `docker compose up`" bug.

---

## 7. Data and ML specifics

### The ODBC driver

`pyodbc` needs Microsoft's driver, which is not in any base image. This is the number one "works locally, breaks in Docker" failure for [[Azure SQL Database]]:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl gnupg unixodbc \
 && curl -sSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" > /etc/apt/sources.list.d/mssql.list \
 && apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
 && rm -rf /var/lib/apt/lists/*
```

> Chain it into **one `RUN`** and end with `rm -rf /var/lib/apt/lists/*`. Separate `RUN` commands each create a layer, and deleting a file in a later layer doesn't shrink the earlier one — the data is still in the image.

### Model files

| Approach | Pros | Cons |
|---|---|---|
| **In the image** (`COPY models/`) | Simple; code and model versions can't drift | Big image; rebuild per model |
| Downloaded at startup | Small image; update without rebuild | Startup dependency; must handle failure |
| Mounted volume | Flexible | Not portable to Container Apps |

**For the capstone: in the image.** For large models in production, download from blob storage at startup.

### Don't put PyTorch in a serving image

```dockerfile
# ✗ ~2.5 GB, and you only need inference
RUN pip install torch

# ✓ ~300 MB
RUN pip install onnxruntime numpy
```

Export to ONNX ([[Model export and serving]]) and the serving container never needs PyTorch at all. Smaller image, faster cold starts, lower cost.

### GPU containers

```bash
docker run --gpus all nvidia/cuda:12.4-runtime-ubuntu22.04 nvidia-smi
```

Needs the NVIDIA Container Toolkit on the host. The **host driver** is used; the container carries the CUDA runtime. See [[CUDA and GPU programming]].

---

## 8. Production hygiene

```dockerfile
FROM python:3.12-slim

# don't run as root
RUN useradd --create-home --uid 1000 appuser

WORKDIR /app
COPY --chown=appuser:appuser . .

USER appuser

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

| Practice | Why |
|---|---|
| Non-root user | Container escape shouldn't mean host root |
| `PYTHONUNBUFFERED=1` | **Without it your logs vanish** — Python buffers stdout and the container dies before flushing |
| Pinned base tag | `python:3.12-slim`, not `python:latest` |
| Versioned image tags | `:v1`, not `:latest` — otherwise you can't tell what's running or roll back |
| Healthcheck | The orchestrator needs to know if you're alive |
| No secrets in the image | `docker history` reveals every build arg and layer |
| `--workers N` sized to memory | Each worker loads its own copy of the model |

> **Never `COPY .env` or pass secrets as `ARG`.** Anyone with the image can read them from the layer history. Pass secrets as runtime environment variables or, better, use a managed identity ([[Azure fundamentals]]).

### Scanning

```bash
docker scout cves retail-api:v1
trivy image retail-api:v1
```

Wire one into CI ([[CI-CD pipelines]]) and fail the build on critical CVEs in your own dependencies.

---

## Cheat sheet

```bash
# build & run
docker build -t name:tag .
docker build --no-cache -t name:tag .          # ignore cache
docker run -p 8000:8000 --env-file .env name:tag
docker run -it --entrypoint bash name:tag      # debug shell

# inspect
docker ps -a
docker logs -f <id>
docker exec -it <id> bash
docker inspect <id>
docker history name:tag                        # layer sizes
docker stats

# clean up (images pile up FAST)
docker system df                               # what's using disk
docker system prune -a --volumes               # ⚠️ deletes unused images AND volumes

# compose
docker compose up --build
docker compose logs -f <service>
docker compose exec <service> bash
docker compose down -v

# registry
az acr login --name myacr
docker tag retail-api:v1 myacr.azurecr.io/retail-api:v1
docker push myacr.azurecr.io/retail-api:v1
```

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Connection refused from browser | App bound to `127.0.0.1` | `--host 0.0.0.0` |
| Container exits immediately | App crashed or the process ended | `docker logs <id>` |
| Rebuild takes minutes every time | `COPY . .` before installing deps | Copy dependency files first |
| "Sending build context… 4.2GB" | No `.dockerignore` | Add one |
| No logs at all | Python output buffered | `ENV PYTHONUNBUFFERED=1` |
| Service can't reach another | Used `localhost` | Use the service name |
| API starts before the database is ready | `depends_on` without a healthcheck | `condition: service_healthy` |
| `pyodbc`: "Data source name not found" | ODBC Driver 18 missing | Add the apt block |
| Image is 3GB | PyTorch in a serving image; no multi-stage | ONNX Runtime; multi-stage build |
| Alpine build takes 30 minutes | musl — no pre-built wheels | Use `-slim` |
| OOM-killed in the cloud, fine locally | No local memory limit | `--memory=2g` to reproduce |
| Data gone after restart | Written to the container layer | Use a volume |
| `docker exec` fails | Container isn't running | `--entrypoint bash` on a new one |
| Disk full | Old images and volumes | `docker system prune -a` |
| Secret visible in the image | Passed as `ARG` or copied in | Runtime env vars or managed identity |
| Works on Mac, fails on the server | Architecture mismatch (arm64 vs amd64) | `docker build --platform linux/amd64` |

---

## Practice checklist

- [ ] Containers vs. VMs, and the shared-kernel consequence
- [ ] Images, layers, containers, and the writable layer
- [ ] **Layer caching and Dockerfile ordering** — the biggest time-saver
- [ ] `.dockerignore`
- [ ] Multi-stage builds, and why they matter most for compiled languages
- [ ] Choosing a base image, and **why not Alpine for Python data work**
- [ ] Running, inspecting, and debugging containers
- [ ] Resource limits, and reproducing production constraints locally
- [ ] Networking: `localhost` means the container; service names; `-p HOST:CONTAINER`; bind `0.0.0.0`
- [ ] Volumes vs. bind mounts, dev vs. prod
- [ ] Compose, and **healthchecks vs. bare `depends_on`**
- [ ] The ODBC driver block, and one-`RUN` apt hygiene
- [ ] ONNX instead of PyTorch in serving images
- [ ] GPU containers
- [ ] Production hygiene: non-root, `PYTHONUNBUFFERED`, pinned tags, healthcheck, no secrets

## Hands-on

- [ ] Take the capstone API image and cut its size with a multi-stage build; record before/after
- [ ] Break the layer cache deliberately, then fix the ordering and time both builds
- [ ] Run the API and dashboard on a Docker network; make the dashboard reach the API by name
- [ ] Deliberately use `localhost` between services and observe the failure
- [ ] Add healthchecks to a Compose file and confirm the API waits for Postgres
- [ ] Run with `--memory=512m` and watch it get OOM-killed
- [ ] Run `docker history` and find the layer making your image big

## Resources

- [Docker docs: build best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Compose file reference](https://docs.docker.com/compose/compose-file/)

## Next

[[Running the whole stack locally]]
