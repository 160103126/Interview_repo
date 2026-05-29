# Docker — Theory & Concepts

> Containerization fundamentals: Images, Containers, Dockerfiles, Volumes, and networking.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Docker? Why is it better than a Virtual Machine (VM)?

**Answer:**

Docker is a platform that uses OS-level virtualization to deliver software in packages called **Containers**. Containers bundle the application code, dependencies, and environment variables into a single unit that can run anywhere.

**Docker vs. VM:**
- **Virtual Machines** include a full Guest Operating System (e.g., a full 10GB Windows or Ubuntu OS) running on top of a Hypervisor. They are heavy, slow to boot (minutes), and consume massive resources.
- **Docker Containers** share the Host OS Kernel. They don't need a guest OS. They are lightweight (MBs), boot in milliseconds, and you can run hundreds of them on a single machine.

*The main benefit:* "It works on my machine" is no longer an excuse. If a container runs on a developer's laptop, it will run exactly the same way on a production AWS server.

---

### Q2: What is a Docker Image vs a Docker Container?

**Answer:**

- **Docker Image:** A read-only template containing instructions for creating a container (The blueprint/recipe). It contains the code, libraries, and environment variables. (e.g., `ubuntu:latest` or `my-python-app:v1`).
- **Docker Container:** A runnable, isolated instance of an Image. If an image is a Class in OOP, a container is an Object instantiated from that class. You can run multiple containers from a single image.

---

## 🟡 Medium (Intermediate)

### Q3: Explain the structure of a standard `Dockerfile`.

**Answer:**

A `Dockerfile` is a text document containing the commands to build an image.

```dockerfile
# 1. Base Image: Start with an official lightweight Python image
FROM python:3.11-slim

# 2. Working Directory: Set the internal directory for subsequent commands
WORKDIR /app

# 3. Dependencies: Copy requirements first to leverage Docker layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. Copy Code: Copy the rest of the application code
COPY . .

# 5. Port Expose: Document that this container listens on port 8000
EXPOSE 8000

# 6. Execution: The command to run when the container starts
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### Q4: Explain Docker Layer Caching. Why did we copy `requirements.txt` before the rest of the code?

**Answer:**

Every `RUN`, `COPY`, and `ADD` command in a Dockerfile creates a new "Layer". Docker caches these layers. 
If you rebuild an image and a layer hasn't changed, Docker reuses the cached layer, saving massive amounts of time.

**The Strategy:**
Source code changes constantly. Dependencies (`requirements.txt`) change rarely. 
If we put `COPY . .` *before* `RUN pip install`, every time we change one line of Python code, the `COPY` layer is invalidated. This forces Docker to invalidate all subsequent layers, meaning it will re-download and re-install all pip packages every single time. 
By copying `requirements.txt` first and installing, we cache the heavy pip install layer. Changing the python code now only invalidates the final layer, resulting in a 1-second build instead of a 5-minute build.

---

### Q5: What is Docker Compose?

**Answer:**

While `docker run` is fine for a single container, modern applications require multiple containers (e.g., a FastAPI backend container, a PostgreSQL database container, and a Redis cache container).

**Docker Compose** is a tool that allows you to define and run multi-container applications using a single `docker-compose.yml` file. 

```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secretpassword
```
Running `docker-compose up` automatically spins up both containers and sets up a private internal network so they can talk to each other.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: How does Docker handle Data Persistence? (Volumes vs Bind Mounts)

**Answer:**

Containers are ephemeral. If a container crashes and is deleted, all data inside it is lost forever. If a database is running in a container, this is catastrophic.

**Solutions for Persistence:**
1. **Docker Volumes:** Managed strictly by Docker. They are stored in a hidden directory on the host OS (`/var/lib/docker/volumes/`). They are secure, easy to back up, and the recommended way to persist database data.
2. **Bind Mounts:** You explicitly map a specific directory on your host machine (e.g., `/Users/dev/my_code`) to a directory inside the container (e.g., `/app`). 
   - *Use case:* Local development. When you edit code on your Mac, the changes instantly appear inside the container without rebuilding the image.

---

### Q7: What are Multi-Stage Builds in Docker?

**Answer:**

Used to create extremely small, secure production images.

**The Problem:** Building an app (like compiling a React frontend or a Go backend) requires heavy build tools, compilers, and source code. If you ship this container to production, it will be 2GB in size and full of security vulnerabilities.

**Multi-Stage Build:** You use multiple `FROM` statements in one Dockerfile.
1. **Stage 1 (Builder):** Uses a heavy base image with all the compilers. It compiles the code into a binary (or static files).
2. **Stage 2 (Production):** Uses a minimal base image (like `alpine` - 5MB). It *copies only the compiled binary* from Stage 1, leaving all the heavy build tools behind.
*Result:* Your production image drops from 2GB to 20MB.

---

### Q8: Explain Docker Networking modes.

**Answer:**

Containers need to communicate with each other and the outside world. Docker provides several networking modes:

1. **Bridge (Default):** Containers on the same bridge network can communicate via container names (DNS). Isolated from the host. Most common for single-host deployments.
2. **Host:** The container shares the host's network directly. No port mapping needed — the container's port IS the host's port. Eliminates networking overhead but loses isolation.
   - *Use case:* Performance-critical applications where network latency matters (ML inference servers).
3. **None:** Complete network isolation. The container has no network interface.
4. **Overlay:** Multi-host networking for Docker Swarm or Kubernetes. Containers on different physical machines can communicate as if they're on the same network.

```bash
# Create a custom bridge network
docker network create my-app-network

# Run containers on the same network (they can reach each other by name)
docker run --network my-app-network --name api my-api-image
docker run --network my-app-network --name db postgres:15
# Inside the api container: connect to "db:5432" (DNS resolution works automatically)
```

---

### Q9: What are Docker Health Checks?

**Answer:**

By default, Docker considers a container "healthy" as long as the main process is running. But a FastAPI server might have a running process that is stuck in a deadlock — Docker won't know.

**Health Checks** let you define a command that Docker runs periodically to verify the container is actually working:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

**States:**
- `healthy`: The health check command succeeded.
- `unhealthy`: The command failed 3 consecutive times (retries).
- `starting`: The container just started and is within the start period.

**Why it matters:**
- Docker Compose's `depends_on` with `condition: service_healthy` won't start dependent containers until the dependency is healthy.
- Kubernetes' liveness/readiness probes serve the same purpose.
- Load balancers use health checks to remove unhealthy containers from rotation.

---

### Q10: Explain `.dockerignore` and Image Size Optimization.

**Answer:**

**`.dockerignore`** prevents unnecessary files from being sent to the Docker daemon during `docker build`, reducing build context size and preventing sensitive files from entering the image.

```
# .dockerignore
__pycache__/
*.pyc
.git/
.env
node_modules/
data/
*.md
tests/
.vscode/
```

**Image Size Optimization Techniques:**
1. **Use slim/alpine base images:** `python:3.11-slim` (150MB) vs `python:3.11` (900MB).
2. **Combine RUN commands:** Each `RUN` creates a layer. `RUN pip install && apt-get clean` is one layer instead of two.
3. **Multi-stage builds:** (See Q7). Copy only the final binary/files, not build tools.
4. **`--no-cache-dir` for pip:** `pip install --no-cache-dir -r requirements.txt` prevents pip from storing downloaded packages.
5. **Remove unnecessary system packages:** `apt-get remove --purge build-essential && rm -rf /var/lib/apt/lists/*`

**Target:** A production ML API container should be 200-500MB, not 5GB.

---

### Q11: How do you scan Docker images for Security Vulnerabilities?

**Answer:**

Docker images can contain outdated OS packages with known security vulnerabilities (CVEs). In a MAANG environment, deploying unscanned images is a policy violation.

**Tools:**
1. **Docker Scout (Built-in):** `docker scout cves my-image:latest` — Scans against the Docker vulnerability database.
2. **Trivy (Open-source, by Aqua Security):** `trivy image my-image:latest` — Most popular open-source scanner. Scans OS packages, language dependencies, and IaC configs.
3. **Snyk Container:** Integrates with CI/CD pipelines.
4. **GCP Artifact Registry:** Automatically scans images on push (see GCP section).

**CI/CD Integration:**
```yaml
# GitHub Actions: Block deployment if critical CVEs are found
- name: Scan Docker image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-app:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail the pipeline on critical vulnerabilities
```

---

### Q12: How do you use Docker for ML/GPU workloads?

**Answer:**

Standard Docker containers cannot access GPUs. You need the **NVIDIA Container Toolkit** (nvidia-docker).

**Setup:**
```bash
# Install NVIDIA Container Toolkit
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker

# Run a GPU-enabled container
docker run --gpus all nvidia/cuda:12.0-runtime-ubuntu22.04 nvidia-smi
```

**ML-Specific Dockerfile:**
```dockerfile
FROM nvidia/cuda:12.0-cudnn8-runtime-ubuntu22.04

# Install Python and ML dependencies
RUN apt-get update && apt-get install -y python3 python3-pip
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["python3", "serve_model.py"]
```

**Key Considerations:**
- **Base Image:** Use `nvidia/cuda` for GPU workloads, not `python:3.11-slim`.
- **CUDA Compatibility:** The CUDA version in the image must match (or be older than) the host's NVIDIA driver version.
- **Model Weights:** Don't bake 10GB model weights into the Docker image. Mount them as volumes or download them on startup.
- **Shared Memory:** Set `--shm-size=8g` for PyTorch DataLoader multi-worker setups (default 64MB is too small).

---

*End of Docker Theory — 12 comprehensive questions covering Containers vs VMs, Dockerfile optimization, Layer Caching, Docker Compose, Volumes, Multi-Stage Builds, Networking, Health Checks, Security Scanning, and GPU Workloads.*

