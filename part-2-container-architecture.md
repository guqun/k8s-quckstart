# GitLab Runner on Kubernetes: A Complete Guide
## Part 2 — Deep Dive into Container Architecture

---

## About This Series

This series documents my journey of learning and deploying GitLab Runner on Kubernetes, from complete beginner to production deployment. What makes this unique is that **I learned and implemented this alongside Claude Code** — an AI coding assistant that helped me:

- Navigate complex Kubernetes networking concepts
- Debug DNS resolution issues in containerized environments
- Understand the four-container architecture of GitLab Runner
- Build production-ready configurations

**Why share this?** Because AI-assisted learning accelerated my understanding dramatically. Concepts that would typically take weeks to grasp through trial-and-error became clear in days through:
- Real-time explanations of error messages
- Step-by-step debugging of network issues
- Detailed breakdowns of container interactions
- Architecture visualizations and timing diagrams

If you're exploring unfamiliar technologies (like I was with GitLab on K8s), this series demonstrates how AI can be a powerful learning companion — not by giving you copy-paste solutions, but by helping you **understand deeply** as you build.

**Learning Tips**: As you read through this series, you'll encounter many technical concepts — Kubernetes networking, Docker-in-Docker, DNS resolution, container orchestration, and more. Don't hesitate to use AI assistants (like Claude, ChatGPT, or others) alongside this guide to:
- Explain unfamiliar terms and acronyms in context
- Break down complex concepts into digestible explanations
- Generate examples that match your specific use case
- Ask "why" questions about architectural decisions
- Debug errors you encounter in your own setup

The best learning happens through active engagement. Use AI tools to explore concepts at your own pace and depth.

> *Read [Part 1](link-to-part-1) first if you haven't already — we cover the basics and get a working runner deployed.*

---

In Part 1, we deployed a GitLab Runner on Kubernetes and ran our first pipeline. We briefly mentioned the "four containers" but didn't explore what's actually happening inside each one.

In this part, we'll dissect the complete architecture. By the end, you'll understand:
- Why there are four different containers and what each one does
- How Docker-in-Docker really works (spoiler: it's client-server, not magic)
- The complete lifecycle from "git push" to "pipeline success"
- Resource management and what happens when jobs run concurrently

Let's start with the architecture overview.

---

## The Four-Container Architecture

When you trigger a CI/CD job on Kubernetes executor, GitLab Runner creates a Pod with **up to four specialized containers**. Each has a distinct role:

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Runner Manager Pod (Persistent)                   │     │
│  │  ┌──────────────────────────────────────────────┐  │     │
│  │  │  gitlab-runner:v18.3.0                       │  │     │
│  │  │  - Polls GitLab for jobs                     │  │     │
│  │  │  - Creates Job Pods when work arrives        │  │     │
│  │  │  - Monitors job execution                    │  │     │
│  │  └──────────────────────────────────────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
│                          │                                   │
│                          │ Creates on-demand                 │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Job Pod (Ephemeral)                               │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  1. Helper (Init Container)                │   │     │
│  │  │     gitlab-runner-helper:v18.3.0           │   │     │
│  │  │     - Clones Git repository                │   │     │
│  │  │     - Downloads artifacts                  │   │     │
│  │  │     - Uploads results after job            │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  2. Build (Main Container)                 │   │     │
│  │  │     User-defined image (e.g., docker:latest)│   │     │
│  │  │     - Executes CI/CD script                │   │     │
│  │  │     - Sends commands to services           │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  3. Service (Sidecar Container)            │   │     │
│  │  │     User-defined (e.g., docker:dind)       │   │     │
│  │  │     - Provides background services         │   │     │
│  │  │     - Runs entire job duration             │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

Let's examine each container in detail.

---

## Container 1: Runner Manager Pod

**Image**: `gitlab/gitlab-runner:v18.3.0`
**Type**: Deployment (persistent)
**Lifespan**: Runs continuously

### Responsibilities

The Runner Manager is the "orchestrator" that never stops running:

1. **Job Polling**: Continuously queries GitLab API for pending jobs
2. **Pod Creation**: Creates Job Pods in Kubernetes when work arrives
3. **Status Monitoring**: Tracks job execution and reports back to GitLab
4. **Log Collection**: Streams job logs to GitLab UI in real-time
5. **Cleanup**: Ensures completed Job Pods are removed

### Configuration Example

```yaml
# From gitlab-runner-values.yaml
replicas: 3              # 3 manager pods for HA
concurrent: 10           # Each manager handles 10 jobs max
```

**Capacity calculation**:
- 3 managers × 10 concurrent = **30 jobs can run simultaneously**

### How It Works

```
Manager Pod Main Loop:
┌─────────────────────────────────────────┐
│ while true:                             │
│   jobs = poll_gitlab_api()              │
│   for job in jobs:                      │
│     if capacity_available():            │
│       pod = create_job_pod(job)         │
│       monitor_pod_status(pod)           │
│       stream_logs_to_gitlab(pod)        │
│   sleep(3 seconds)                      │
└─────────────────────────────────────────┘
```

The manager itself doesn't execute any CI/CD scripts — it's purely a control plane component.

---

## Container 2: Helper Container

**Image**: `gitlab-runner-helper:x86_64-v18.3.0`
**Type**: Init Container
**Lifespan**: Runs at job start and end

### Responsibilities

The Helper handles all Git and artifact operations:

**At Job Start**:
- Clones the Git repository to a shared volume
- Downloads artifacts from previous stages
- Sets up the working directory

**At Job End**:
- Uploads artifacts to GitLab
- Uploads cache (if configured)
- Uploads test reports

### Execution Flow

```
T+0s   Job Pod created
       │
T+1s   Helper Container starts (as init container)
       │
       ├─ Step 1: Clone repository
       │  $ git clone --depth 50 https://gitlab.com/user/repo.git
       │
       ├─ Step 2: Checkout specific commit
       │  $ git checkout $CI_COMMIT_SHA
       │
       ├─ Step 3: Download artifacts (if any)
       │  $ wget $GITLAB_API/jobs/$JOB_ID/artifacts
       │
       └─ Step 4: Complete (exit)

T+3s   Helper exits, Build Container can start

       [Build Container runs...]

T+160s Build completes

       Helper reactivates (as post-job container)
       │
       ├─ Upload artifacts
       │  $ curl -F "file=@output.zip" $GITLAB_API/artifacts
       │
       └─ Upload cache
          $ curl -F "file=@.cache.tar.gz" $GITLAB_API/cache
```

### Shared Volume

The Helper and Build containers share a volume:

```
Shared emptyDir Volume: /builds
├── namespace/
│   └── project-name/
│       ├── .git/
│       ├── src/
│       ├── Dockerfile
│       └── .gitlab-ci.yml
```

**Key point**: The Build Container doesn't clone code itself — it just uses what Helper prepared.

---

## Container 3: Build Container

**Image**: Defined by `image` in `.gitlab-ci.yml`
**Type**: Main container
**Lifespan**: Runs for the duration of the job

### Responsibilities

This is where your CI/CD script actually executes:

- Runs `before_script`, `script`, `after_script`
- Has access to the cloned repository
- Communicates with Service Containers via network
- Reports exit code back to GitLab

### Configuration

In your `.gitlab-ci.yml`:

```yaml
build-job:
  image: docker:latest      # ← Defines Build Container image
  script:
    - docker build -t myapp .
    - docker push myapp
```

The Build Container runs with these defaults:
- Working directory: `/builds/<namespace>/<project>`
- User: `root` (configurable)
- Network: Shared with Service Containers

### Resource Limits

From `gitlab-runner-values.yaml`:

```yaml
[runners.kubernetes]
  cpu_limit = "2"              # Max 2 CPU cores
  memory_limit = "2Gi"         # Max 2GB RAM
  cpu_request = "500m"         # Guaranteed 0.5 cores
  memory_request = "512Mi"     # Guaranteed 512MB
```

What happens if exceeded?
- **CPU limit**: Throttled (job slows down)
- **Memory limit**: OOM killed (job fails)

---

## Container 4: Service Container (Docker-in-Docker)

**Image**: Defined by `services` in `.gitlab-ci.yml`
**Type**: Sidecar container
**Lifespan**: Runs entire job duration

This is the most complex container, so let's break it down thoroughly.

### What is Docker-in-Docker?

When your CI/CD job needs to build Docker images, you face a challenge: **the Build Container is already inside Docker** (running on Kubernetes). How do you run Docker inside Docker?

**Solution**: Run a complete Docker daemon (`dockerd`) in a separate container.

### Configuration Example

```yaml
# .gitlab-ci.yml
services:
  - name: docker:dind
    alias: docker          # ← Network alias (important!)
    command:
      - "--tls=false"      # Disable TLS for simplicity
      - "--insecure-registry=my-registry.com"

variables:
  DOCKER_HOST: tcp://docker:2375  # ← Connect to DinD
```

### Complete Lifecycle (0s to 170s)

Let me walk you through the entire lifecycle with actual timing:

```
T+0s    Job Pod created by Runner Manager
        Kubernetes scheduler selects a node

T+1s    ========== Phase 1: Container Startup ==========

        Helper Container (init) starts
        └─ Clones code (2s)

T+3s    Helper completes, enters Waiting state

        ========== Phase 2: Service Container Startup ==========

T+4s    DinD Container starts
        └─ Executes ENTRYPOINT: dockerd-entrypoint.sh
            ├─ Initialize storage driver (overlay2)
            ├─ Set up iptables rules
            ├─ Start containerd
            └─ Start dockerd daemon

T+5s    dockerd starting...
        Logs:
          level=info msg="Starting up"
          level=info msg="containerd not running, starting managed containerd"

T+7s    dockerd listening on ports
        Logs:
          level=info msg="API listen on /var/run/docker.sock"
          level=info msg="API listen on [::]:2375"

T+8s    ========== Phase 3: DinD Ready ==========

        DinD fully ready, waiting for client connections
        Status: Ready = true

        Build Container starts
        └─ Executes before_script

T+9s    ========== Phase 4: First Request ==========

        Build Container: docker info

        Network flow:
          Build Container (docker CLI)
              ↓ TCP connection
          tcp://docker:2375
              ↓ Resolves via alias
          127.0.0.1:2375 (localhost)
              ↓ Local loopback
          DinD Container (dockerd process)

        DinD logs:
          level=debug msg="Calling GET /v1.40/info"

        Returns: Server Version: 27.1.1
        ✅ Connection confirmed

T+10s   ========== Phase 5: Execute Build ==========

        Build Container: docker build -t myapp .

        Request flow:
          ① Build sends build context (tar file)
             ↓
          ② DinD receives context, extracts to temp dir
             Logs: "Building context from tarball"
             ↓
          ③ DinD reads Dockerfile
             Parsing: FROM python:3.9-slim
             ↓
          ④ DinD checks local image cache
             Result: Not found
             ↓
          ⑤ DinD performs DNS resolution
             Query: /etc/resolv.conf
             Nameserver: 10.0.0.2
             Domain: registry.example.com
             Result: 203.0.113.10
             ↓
          ⑥ DinD establishes HTTPS connection
             Connect: 203.0.113.10:443
             ↓
          ⑦ DinD pulls image layers
             Logs:
               "Pulling from library/python"
               "Layer 1: Downloading [=>  ] 5MB/50MB"
               "Layer 2: Downloading [===>] 15MB/30MB"
             (35s download time)
             ↓
          ⑧ DinD executes Dockerfile instructions
             COPY . . → Copy code into image
             RUN pip install → Execute install command
             ↓
          ⑨ DinD creates new image
             Image ID: sha256:abc123...
             Tag: myapp:latest
             ↓
          ⑩ DinD returns build result
             ↓
          Build Container receives success message
             "Successfully built abc123..."
             "Successfully tagged myapp:latest"

T+120s  ========== Phase 6: Push Image ==========

        Build Container: docker push myapp:latest

        Push flow:
          ① Build sends push request
             ↓
          ② DinD reads image layers
             Check which layers exist on remote
             ↓
          ③ DinD connects to target registry
             Address: private-registry.example.com
             Auth: Uses docker login credentials
             ↓
          ④ DinD uploads image layers
             Logs:
               "Layer 1: Pushed (already exists)"
               "Layer 2: Pushed (already exists)"
               "Layer 3: Pushing [=>  ] 10MB/50MB"
             ↓
          ⑤ DinD pushes manifest
             Tag: 56cc4792-86
             ↓
          ⑥ DinD returns success
             ↓
          Build Container receives success
             "56cc4792-86: digest: sha256:def456... size: 1234"

T+160s  ========== Phase 7: Build Script Completes ==========

        Build Container finishes script execution
        Container enters Completed state

        DinD Container still running
        └─ Waiting for Pod termination signal

T+165s  ========== Phase 8: Cleanup ==========

        Kubernetes sends SIGTERM to all containers

        DinD receives signal
        Logs:
          level=info msg="Processing signal 'terminated'"
          level=info msg="Stopping daemon"
          level=info msg="Daemon shutdown complete"

        DinD graceful shutdown (5s)
        └─ Stop containerd
        └─ Clean temporary files
        └─ Release ports

T+170s  Pod deletion complete
```

---

## The Critical Concept: Docker Client-Server Architecture

This is the most commonly misunderstood aspect. Let me clarify:

### What Actually Happens

```
┌─────────────────────────────────────────────────────────┐
│                     Job Pod                             │
│                                                         │
│  ┌──────────────────┐         ┌──────────────────┐    │
│  │ Build Container  │         │  DinD Container  │    │
│  │                  │         │                  │    │
│  │ ✅ Where you     │         │ ✅ Where work    │    │
│  │ type commands    │         │ actually happens │    │
│  │ (Client)         │         │ (Server)         │    │
│  │                  │         │                  │    │
│  │ $ docker build   │────────►│  dockerd         │    │
│  │   ↑              │  API    │   ↓              │    │
│  │   Just CLI tool  │  Call   │   Build engine   │    │
│  │                  │         │                  │    │
│  │ Does:            │         │ Does:            │    │
│  │ - Read Dockerfile│         │ - Pull images    │    │
│  │ - Package context│         │ - Extract context│    │
│  │ - Send API req   │         │ - Run commands   │    │
│  │ - Display logs   │         │ - Create layers  │    │
│  │                  │         │ - Generate IDs   │    │
│  │ Doesn't:         │         │ Doesn't:         │    │
│  │ - Pull images    │         │ - Read user input│    │
│  │ - Run commands   │         │ - Display output │    │
│  │ - Create layers  │         │                  │    │
│  └──────────────────┘         └──────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Detailed Responsibilities

| Step | Build Container Does | DinD Container Does | Where? |
|------|---------------------|---------------------|--------|
| **1. Start command** | Execute `docker build -t myapp .` | dockerd listens on 2375 | Build |
| **2. Preparation** | Read Dockerfile in current dir | Wait for requests | Build |
| **3. Package context** | Create tar file of current dir (5MB) | - | Build |
| **4. Send request** | HTTP POST tar to `tcp://docker:2375/build` | Receive tar, extract to `/tmp/docker-build-xxx` | Build → DinD |
| **5. Parse Dockerfile** | - | Read Dockerfile, parse instructions | DinD |
| **6. Pull images** | - | Execute `FROM python:3.9-slim`<br>DNS resolution, image pull (35s) | DinD |
| **7. Build layers** | Receive and display log stream | Execute `RUN pip install`<br>Run command in temporary container | DinD |
| **8. Complete** | Display "Successfully built abc123" | Commit image layers, generate image ID | DinD → Build |

### Why This Design?

**1. Separation of Concerns**
- Build Container: User interaction, script execution
- DinD Container: Docker engine, image management

**2. Resource Isolation**
- Build failure doesn't crash the Docker daemon
- You can limit DinD resources independently

**3. Security**
- Build Container doesn't need privileged mode
- Only DinD needs `privileged: true`
- Reduces security risk surface

**4. Flexibility**
- Can connect to remote Docker daemons
- Can swap build tools without touching services

### Verification

You can verify this architecture:

**In Build Container**:
```bash
# Only has docker CLI tool
$ which docker
/usr/local/bin/docker

# It's just a client binary
$ file /usr/local/bin/docker
/usr/local/bin/docker: ELF 64-bit LSB executable

# No dockerd daemon
$ which dockerd
(not found)

# Connects to remote dockerd
$ echo $DOCKER_HOST
tcp://docker:2375

# No dockerd process
$ ps aux | grep dockerd
(no results)
```

**In DinD Container**:
```bash
# Has dockerd daemon running
$ ps aux | grep dockerd
root  21  dockerd --host=tcp://0.0.0.0:2375 --tls=false

# Daemon listening on port
$ netstat -tuln | grep 2375
tcp6  0  0  :::2375  :::*  LISTEN
```

---

## Container Communication

All containers in the Job Pod share the same network namespace:

```
┌──────────────────────────────────────────────────────┐
│           Job Pod Network Namespace                  │
│           (All containers share this)                │
│                                                      │
│  Network Interfaces:                                 │
│  ┌────────────────────────────────────────────┐     │
│  │  lo (loopback)                             │     │
│  │  127.0.0.1/8                               │     │
│  │  Used for: Container-to-container (docker) │     │
│  └────────────────────────────────────────────┘     │
│                                                      │
│  ┌────────────────────────────────────────────┐     │
│  │  eth0 (veth pair)                          │     │
│  │  172.24.1.100/24 (example)                 │     │
│  │  Used for: External communication          │     │
│  └────────────────────────────────────────────┘     │
│                                                      │
│  /etc/hosts:                                         │
│  ┌────────────────────────────────────────────┐     │
│  │  127.0.0.1 localhost                       │     │
│  │  127.0.0.1 docker    # Service alias       │     │
│  │  172.24.1.100 runner-xxx-xxx               │     │
│  └────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

**Key points**:
- Service alias (`docker`) resolves to `127.0.0.1`
- Communication happens via localhost (no network overhead)
- All containers see the same IP address and network interfaces

---

## Resource Management

Understanding resource allocation is crucial for stable operations.

### Per-Job Resources

Each Job Pod can consume:

```yaml
# Build Container
cpu_limit = "2"              # Max 2 cores
memory_limit = "2Gi"         # Max 2GB

# Service Container (DinD)
service_cpu_limit = "2"      # Max 2 cores
service_memory_limit = "2Gi" # Max 2GB

# Helper Container
helper_cpu_limit = "500m"    # Max 0.5 cores
helper_memory_limit = "512Mi" # Max 512MB
```

**Total per job**: Up to 4.5 CPU cores and 4.5GB memory

### Cluster Capacity Planning

Example calculation:

```
Cluster: 3 worker nodes, each with 8 cores and 32GB RAM

Available for jobs:
- CPU: 3 × 8 = 24 cores (assuming 100% allocatable)
- Memory: 3 × 32GB = 96GB

Maximum concurrent jobs (if all use max resources):
- By CPU: 24 ÷ 4.5 = 5 jobs
- By memory: 96 ÷ 4.5 = 21 jobs
- Bottleneck: CPU → 5 concurrent jobs max

Actual capacity (with typical 50% CPU usage):
- ~10 concurrent jobs comfortably
```

### What Happens When Limits Are Exceeded?

**CPU Limit**:
```
Job tries to use 3 cores (limit is 2)
   ↓
Kubernetes throttles the container
   ↓
Job runs slower but doesn't fail
   ↓
Visible in metrics: CPU throttling
```

**Memory Limit**:
```
Job tries to allocate 3GB (limit is 2GB)
   ↓
Container OOM killed immediately
   ↓
Pod restarts (if restart policy allows)
   ↓
Job fails with: "OOMKilled: Container exceeded memory limit"
```

---

## Putting It All Together

Let's trace a complete job from start to finish:

```
Developer: git push

GitLab Server: Pipeline created
   ↓
Runner Manager: Polls API, sees new job
   ↓
Runner Manager: Creates Job Pod
   ↓
Kubernetes: Schedules Pod to worker-node-2
   ↓
Helper Container: Starts (init)
   - Clones repository
   - Downloads artifacts
   - Exits
   ↓
Service Container (DinD): Starts
   - Starts dockerd
   - Listens on tcp://0.0.0.0:2375
   - Ready
   ↓
Build Container: Starts
   - Reads /builds/namespace/project/
   - Executes before_script
   - Executes script:
     * docker build
        → Connects to tcp://docker:2375
        → DinD performs build
        → Returns success
     * docker push
        → Connects to tcp://docker:2375
        → DinD pushes image
        → Returns success
   - Exits with code 0
   ↓
Helper Container: Reactivates
   - Uploads artifacts
   - Uploads cache
   - Exits
   ↓
Runner Manager: Collects logs
   - Streams to GitLab
   - Marks job as success
   ↓
Kubernetes: Deletes Job Pod
   ↓
GitLab: Shows ✅ Pipeline passed
```

**Total time**: ~160 seconds (varies by image size and network)

---

## Key Takeaways

**Container Roles**:
1. **Runner Manager**: Orchestrator (never executes jobs)
2. **Helper**: Git and artifact handler
3. **Build**: Script executor (your code runs here)
4. **Service (DinD)**: Background services (actual work happens here)

**Docker-in-Docker Architecture**:
- Build Container = Docker **client** (sends commands)
- DinD Container = Docker **server** (does the work)
- Communication via TCP on localhost

**Resource Management**:
- Each job can use up to 4.5 CPU cores and 4.5GB RAM (with default limits)
- CPU limits cause throttling, memory limits cause OOM kills
- Plan cluster capacity based on concurrent job requirements

**Network**:
- All containers share a Pod network namespace
- Service aliases resolve to 127.0.0.1
- No network overhead for inter-container communication

---

## What's Next?

In Part 2, we explored the internal architecture and timing of each container. We now understand how the pieces fit together.

**Coming up in Part 3**:
- 🎬 **Complete CI/CD workflow** — From code push to production deployment
- 📝 **.gitlab-ci.yml best practices** — Build, test, and deploy stages
- 🐳 **Docker image optimization** — Faster builds, smaller images
- ☸️ **Kubernetes deployment** — Using kubectl from CI/CD

**Preview**: We'll build a real-world pipeline that builds a Docker image, pushes it to a registry, and deploys it to Kubernetes — all automated.

---

## Practice Exercises

**Exercise 1**: Calculate your cluster's job capacity

Given:
- 4 worker nodes
- 16 cores and 64GB RAM per node
- Job limits: 2 CPU, 2GB memory

How many concurrent jobs can you run?

**Exercise 2**: Monitor container resource usage

```bash
# Watch Job Pod resources in real-time
kubectl top pod -n gitlab --containers

# Compare to limits
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 5 resources
```

**Exercise 3**: Inspect DinD container

```bash
# While a job is running, exec into DinD
kubectl exec -n gitlab <job-pod> -c docker -- ps aux

# Check dockerd listening ports
kubectl exec -n gitlab <job-pod> -c docker -- netstat -tuln | grep 2375
```

---

## Resources

- [GitLab Runner Executors Documentation](https://docs.gitlab.com/runner/executors/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Docker Engine API Reference](https://docs.docker.com/engine/api/)

---

**Part 3 coming soon** — we'll build a complete CI/CD pipeline with Docker builds and Kubernetes deployments.

If you found this helpful, feel free to share it with others learning GitLab CI/CD.

---

*This is Part 2 of a 5-part series on GitLab Runner on Kubernetes, documenting the learning and implementation process.*

**Series index**:
- [Part 1: Architecture & Quick Setup](link-to-part-1)
- **Part 2: Deep Dive into Container Architecture** ← You are here
- Part 3: Building a Real-World CI/CD Pipeline (coming soon)
- Part 4: Solving DNS and Network Issues (coming soon)
- Part 5: Production Best Practices (coming soon)

---

*Tags: #GitLab #Kubernetes #CICD #DevOps #Docker #ContainerArchitecture*
