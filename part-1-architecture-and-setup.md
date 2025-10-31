# GitLab Runner on Kubernetes: A Complete Guide
## Part 1 — Architecture & Quick Setup

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

Let's dive in.

---

## The Challenge

When working with GitLab CI/CD, you might encounter challenges with traditional runner setups:

- **Resource contention**: Multiple jobs competing for the same host resources
- **Disk space management**: Build artifacts accumulating over time
- **Scaling limitations**: Adding capacity means provisioning new VMs
- **Dependency conflicts**: Jobs interfering with each other's environments

**Kubernetes Executor offers a different approach**: treating each CI/CD job as an isolated, disposable Pod that's created on-demand and automatically cleaned up.

In this 5-part series, we'll explore how this architecture works and build a production-ready setup. By the end of Part 1, you'll have a working runner executing your first pipeline.

---

## Why Kubernetes Executor?

Let's compare the three main GitLab Runner executors:

### Traditional Approaches

**Shell Executor**
```yaml
# Runs directly on the host machine
runners:
  executor: shell
```
- ✅ Simple setup
- ❌ No isolation between jobs
- ❌ Dependency conflicts ("works on my machine")
- ❌ Difficult to scale

**Docker Executor**
```yaml
# Runs jobs in Docker containers
runners:
  executor: docker
```
- ✅ Job isolation
- ✅ Clean environment per job
- ❌ Requires Docker daemon on each runner host
- ❌ Manual scaling (need to provision more runner VMs)

### The Kubernetes Way

**Kubernetes Executor**
```yaml
# Dynamically creates Pods for each job
runners:
  executor: kubernetes
```
- ✅ **Elastic scaling**: Pods created on-demand
- ✅ **Resource isolation**: CPU/memory limits per job
- ✅ **Cost efficiency**: No idle runners consuming resources
- ✅ **High availability**: Multiple runner managers for redundancy
- ✅ **Self-healing**: Failed jobs automatically retry on healthy nodes

---

## Architecture Overview

Before diving into setup, let's understand what we're building.

### The Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                    Your Infrastructure                   │
│                                                          │
│  ┌──────────────────┐         ┌────────────────────┐   │
│  │  GitLab Server   │         │  Kubernetes Cluster │   │
│  │  (VM/Container)  │◄───────►│                     │   │
│  │                  │  API    │  ┌──────────────┐   │   │
│  │  - Web UI        │  Calls  │  │ Runner       │   │   │
│  │  - Git repos     │         │  │ Manager Pods │   │   │
│  │  - CI/CD config  │         │  └──────┬───────┘   │   │
│  └──────────────────┘         │         │           │   │
│                                │         │ Creates   │   │
│                                │         ▼           │   │
│                                │  ┌──────────────┐   │   │
│                                │  │  Job Pods    │   │   │
│                                │  │  (Dynamic)   │   │   │
│                                │  └──────────────┘   │   │
│                                └────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Flow**:
1. Developer pushes code to GitLab
2. GitLab triggers a pipeline
3. Runner Manager (running in K8s) picks up the job
4. Manager creates a dedicated Job Pod
5. Job Pod executes the CI/CD script
6. Pod reports results back to GitLab
7. Pod is deleted (no residue!)

---

## The Four-Container Architecture

Here's where Kubernetes Executor shines: **each CI/CD job runs in a Pod with up to 4 specialized containers**.

### Quick Overview

| Container | Image | Purpose | Lifespan |
|-----------|-------|---------|----------|
| **Runner Manager** | `gitlab-runner:v18.3.0` | Polls GitLab for jobs, creates Pods | Persistent (Deployment) |
| **Helper** | `gitlab-runner-helper:v18.3.0` | Clones code, uploads artifacts | Per-job (Init Container) |
| **Build** | *User-defined* (e.g., `docker:latest`) | Executes CI/CD scripts | Per-job |
| **Service** | *User-defined* (e.g., `docker:dind`) | Provides background services (databases, DinD) | Per-job |

### Example: A Docker Build Job

When you run a job that builds a Docker image, here's what happens:

```yaml
# .gitlab-ci.yml
build-job:
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t myapp:latest .
```

**Behind the scenes**:

```
Job Pod Created
│
├─ [Init] Helper Container
│   └─ Clones your Git repository
│   └─ Exits (job continues)
│
├─ [Main] Build Container (docker:latest)
│   └─ Executes: docker build -t myapp:latest .
│   └─ Sends commands to → Service Container
│
└─ [Service] DinD Container (docker:dind)
    └─ Runs dockerd (Docker daemon)
    └─ Actually performs the build
    └─ Returns results to Build Container
```

**Key insight**: The `docker` command in your script runs in the **Build Container**, but the actual build work happens in the **DinD (Docker-in-Docker) Service Container**. They communicate via TCP on `localhost:2375`.

> 💡 **Note**: This separation provides fault isolation — if the Build Container crashes, the DinD daemon continues running. You can also swap build tools without affecting the service layer.

We'll dive deep into each container in **Part 2**. For now, just understand that **isolation + specialization = reliability**.

---

## Quick Setup (15 Minutes)

Let's get hands-on. We'll deploy a GitLab Runner on Kubernetes using the official Helm chart.

### Prerequisites

```bash
# Verify you have:
kubectl cluster-info  # Kubernetes cluster access
helm version          # Helm 3.x installed
```

You'll also need:
- A running GitLab instance (self-hosted or GitLab.com)
- Cluster admin access (to create namespaces and RBAC)

### Step 1: Get Your Runner Token

1. Log in to your GitLab instance
2. Navigate to **Admin Area** → **CI/CD** → **Runners**
3. Click **New instance runner**
4. Add tags: `kubernetes`, `docker`
5. Check **Run untagged jobs**
6. Click **Create runner**
7. **Copy the registration token** (format: `glrt-xxxxxxxxxxxx`)

⚠️ **Save this token** — you'll need it in Step 3.

### Step 2: Install the Helm Chart

```bash
# Add GitLab Helm repository
helm repo add gitlab https://charts.gitlab.io
helm repo update

# Create namespace
kubectl create namespace gitlab

# Verify
kubectl get namespace gitlab
```

### Step 3: Configure the Runner

Create a file `gitlab-runner-values.yaml`:

```yaml
# GitLab connection
gitlabUrl: https://gitlab.com/  # or your self-hosted URL
runnerToken: "glrt-YOUR_TOKEN_HERE"  # ⚠️ Replace this!

# Runner image
image:
  registry: docker.io
  image: gitlab/gitlab-runner
  tag: v18.3.0

# Scaling
replicas: 2              # 2 manager pods for HA
concurrent: 10           # Each manager handles 10 jobs max

# Kubernetes executor config
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "alpine:latest"

        # Resource limits (per job pod)
        cpu_limit = "1"
        memory_limit = "1Gi"
        cpu_request = "500m"
        memory_request = "512Mi"

        # Service containers (e.g., DinD)
        service_cpu_limit = "1"
        service_memory_limit = "1Gi"

        # Enable privileged mode (needed for Docker-in-Docker)
        privileged = true
```

**Configuration explained**:
- `replicas: 2` — Two manager pods for high availability
- `concurrent: 10` — Total capacity: 2 × 10 = **20 concurrent jobs**
- `privileged = true` — Required for DinD to access kernel features

> 🔒 **Security note**: Privileged containers have elevated permissions. In Part 5, we'll discuss safer alternatives like Kaniko.

### Step 4: Deploy

```bash
helm install gitlab-runner \
  --namespace gitlab \
  --version 0.80.0 \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# Watch deployment
kubectl rollout status deployment/gitlab-runner -n gitlab

# Verify pods
kubectl get pods -n gitlab -l app=gitlab-runner
```

**Expected output**:
```
NAME                             READY   STATUS    RESTARTS   AGE
gitlab-runner-5d8f7c9b4d-abc12   1/1     Running   0          30s
gitlab-runner-5d8f7c9b4d-def34   1/1     Running   0          30s
```

### Step 5: Verify Registration

Check if runners are online:

```bash
# Check logs
kubectl logs -n gitlab -l app=gitlab-runner --tail=20

# Look for:
# ✓ Configuration loaded
# ✓ Runner registered successfully
```

Or check in GitLab UI:
- **Admin Area** → **CI/CD** → **Runners**
- You should see 2 runners with a green **online** indicator

---

## Your First Pipeline

Let's run a simple test to confirm everything works.

### Create a Test Project

1. Create a new GitLab project: `runner-test`
2. Add file `.gitlab-ci.yml`:

```yaml
stages:
  - test

hello-kubernetes:
  stage: test
  tags:
    - kubernetes  # Routes to your K8s runner
  image: alpine:latest
  script:
    - echo "Hello from Kubernetes Runner!"
    - echo "Running on pod: $(hostname)"
    - echo "Date: $(date)"
    - echo "Testing network:"
    - ping -c 3 google.com || echo "No internet (might be expected)"
```

3. Commit and push

### Watch the Magic

**In GitLab UI** (CI/CD → Pipelines):
- Pipeline status: Running → Passed ✅
- Job logs show output from the Pod

**In Kubernetes**:
```bash
# Watch pods being created
kubectl get pods -n gitlab -w

# You'll see:
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Pending
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Init:0/1
# runner-xxxxx-project-1-concurrent-0-xxxxx   2/2   Running
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Completed
```

**Lifecycle breakdown**:
- `Pending` (2s): Kubernetes schedules the Pod
- `Init:0/1` (3s): Helper container clones your code
- `Running` (10s): Build container executes your script
- `Completed`: Pod exits, results sent to GitLab
- Auto-deleted after a few minutes

> ✅ **Verification complete**: You've successfully run a CI/CD job in a dynamically created Kubernetes Pod.

---

## What Just Happened?

Let's trace the complete flow:

```
1. You pushed code
2. GitLab created a pipeline
3. Runner Manager (in K8s) polled GitLab API
4. Manager saw the job and created a Pod
5. Helper container cloned your repo
6. Build container (alpine) ran your script
7. Script output was streamed to GitLab
8. Pod reported "success" and exited
9. GitLab marked the job as passed ✅
10. Kubernetes deleted the Pod
```

**The key benefit**: No manual cleanup required. Kubernetes handles the entire lifecycle automatically.

---

## Key Takeaways

Let's summarize the main differences between traditional setups and Kubernetes Executor:

| Traditional Setup | Kubernetes Executor |
|-------------------|---------------------|
| 💾 Disk space accumulation | 🗑️ Ephemeral Pods (auto-cleanup) |
| 📉 Manual scaling | 📈 Auto-scales with cluster |
| 🔥 Resource conflicts | 🔒 Isolated per job |
| 💰 Always-on runners | 💸 Pay-per-use (only when jobs run) |

---

## What's Next?

In this first part, we covered the "what" and "why" of Kubernetes Executor and got a working setup. But we only scratched the surface.

**Coming up in Part 2**:
- 🔍 **Deep dive into the 4 containers** — What's really happening inside?
- 🐳 **Docker-in-Docker demystified** — Client vs. Server architecture
- 🌐 **Network namespaces** — How containers communicate
- 💾 **Resource management** — Preventing OOM kills

**Preview**: We'll answer questions like "Why does `docker build` run in the Build Container but the actual image layers are created in the DinD Container?" with detailed timing diagrams and execution flows.

---

## Practice Exercises

Before moving to Part 2, try these exercises to deepen your understanding:

**Exercise 1**: Modify the test job to use a different image:
```yaml
image: node:18-alpine
script:
  - node --version
  - npm --version
```

**Exercise 2**: Add a second stage:
```yaml
stages:
  - test
  - build

test-job:
  stage: test
  script:
    - echo "Running tests..."

build-job:
  stage: build
  script:
    - echo "Building app..."
```

**Exercise 3**: Watch the Pod lifecycle in real-time:
```bash
kubectl get pods -n gitlab -w --show-all
```

---

## Resources

- [Official GitLab Runner Docs](https://docs.gitlab.com/runner/)
- [Kubernetes Executor Config Reference](https://docs.gitlab.com/runner/executors/kubernetes.html)
- [Helm Chart Repository](https://gitlab.com/gitlab-org/charts/gitlab-runner)

---

## Questions?

Drop a comment below! Common questions I'll address in upcoming parts:
- "Can I use this with GitLab.com (SaaS)?" — Yes!
- "How do I handle private Docker registries?" — Part 3
- "What about network policies and security?" — Part 5

---

**Part 2 coming soon** — we'll explore the internal architecture of each container in detail.

If you found this helpful, feel free to share it with others learning GitLab CI/CD.

---

*This is Part 1 of a 5-part series on GitLab Runner on Kubernetes, documenting the learning and implementation process.*

**Series index**:
- **Part 1: Architecture & Quick Setup** ← You are here
- Part 2: Deep Dive into Container Architecture (coming soon)
- Part 3: Building a Real-World CI/CD Pipeline (coming soon)
- Part 4: Solving DNS and Network Issues (coming soon)
- Part 5: Production Best Practices (coming soon)

---

*Tags: #GitLab #Kubernetes #CICD #DevOps #Docker #CloudNative*
