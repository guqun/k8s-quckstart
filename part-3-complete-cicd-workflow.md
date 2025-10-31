# GitLab Runner on Kubernetes: A Complete Guide
## Part 3 — Building a Real-World CI/CD Pipeline

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

> *Catch up: [Part 1](link) covers setup, [Part 2](link) explains the container architecture.*

---

In Parts 1 and 2, we deployed the runner and understood how the four containers work together. Now let's build something real: **a complete CI/CD pipeline that builds a Docker image, pushes it to a registry, and deploys it to Kubernetes**.

By the end of this part, you'll have:
- A working `.gitlab-ci.yml` with build and deploy stages
- Docker image builds using DinD (Docker-in-Docker)
- Automated deployments to your Kubernetes cluster
- Understanding of how variables flow through the pipeline

Let's start with the complete flow overview.

---

## The Complete Pipeline Flow

Here's what we're building:

```
┌─────────────────────────────────────────────────────────────┐
│                    GitLab CI/CD Pipeline                     │
│                                                              │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│  │  Stage:  │       │  Stage:  │       │   App    │        │
│  │  build   │  ───→ │  deploy  │  ───→ │  Running │        │
│  └──────────┘       └──────────┘       └──────────┘        │
└─────────────────────────────────────────────────────────────┘

Detailed flow:

1️⃣  Developer pushes code to main branch
        ↓
2️⃣  GitLab triggers Pipeline
        ↓
3️⃣  Runner Manager picks up build-job
        ↓
4️⃣  Job Pod created (Build + DinD + Helper)
        ↓
5️⃣  Helper clones code
        ↓
6️⃣  Build Container: docker build
        ↓
7️⃣  Build Container: docker push to registry
        ↓
8️⃣  build-job completes, Pod deleted
        ↓
9️⃣  Runner Manager picks up deploy-job (manual trigger)
        ↓
🔟 Deploy Pod created (kubectl image)
        ↓
⓫  envsubst replaces image variable in YAML
        ↓
⓬  kubectl applies deployment to cluster
        ↓
⓭  App deployed, accessible via NodePort
```

Let's implement this step by step.

---

## Project Structure

We'll use a simple Python Flask application:

```
my-app/
├── .gitlab-ci.yml          # CI/CD configuration
├── Dockerfile              # Image build instructions
├── calculator.yaml         # Kubernetes deployment config
├── app.py                  # Application code
└── requirements.txt        # Python dependencies
```

### Sample Application

**app.py**:
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        "message": "Calculator API",
        "version": "1.0"
    })

@app.route('/add', methods=['POST'])
def add():
    data = request.json
    result = data['a'] + data['b']
    return jsonify({"result": result})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=30194)
```

**requirements.txt**:
```
Flask==2.3.0
```

**Dockerfile**:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 30194

CMD ["python", "app.py"]
```

---

## Pipeline Configuration

Now for the CI/CD configuration. This is where everything comes together.

### Complete .gitlab-ci.yml

```yaml
stages:
  - build   # Stage 1: Build and push Docker image
  - deploy  # Stage 2: Deploy to Kubernetes

variables:
  # Dynamic image tag: commit SHA + pipeline ID
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

  # Kubeconfig path for deploy job
  KUBECONFIG: "/tmp/kubeconfig"

#################################################################
# BUILD JOB: Build Docker image and push to registry
#################################################################

build-job:
  stage: build
  tags:
    - kubernetes  # Route to K8s runner

  image: registry.example.com/library/docker:latest

  services:
    - name: registry.example.com/library/docker:dind
      alias: docker
      command:
        - "--tls=false"
        - "--insecure-registry=private-registry.example.com"

  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always

  before_script:
    - echo "Waiting for Docker daemon to start..."
    - |
      timeout=60
      until docker info >/dev/null 2>&1; do
        if [ $timeout -le 0 ]; then
          echo "❌ Timeout: Docker daemon not starting"
          echo "Attempting to check dockerd logs..."
          nc -zv docker 2375 || echo "Port 2375 not reachable"
          exit 1
        fi
        echo "Waiting... (${timeout}s remaining)"
        sleep 2
        timeout=$((timeout - 2))
      done
    - echo "✅ Docker daemon ready"
    - docker version
    - docker info

  script:
    # 1. Verify Docker connection
    - docker info

    # 2. Build image
    - docker build -t $DOCKER_IMAGE .

    # 3. Login to private registry
    - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin private-registry.example.com

    # 4. Push image
    - docker push $DOCKER_IMAGE

    # 5. Output success message
    - echo "Build complete, image pushed to registry: $DOCKER_IMAGE"

#################################################################
# DEPLOY JOB: Deploy to Kubernetes cluster
#################################################################

deploy-job:
  stage: deploy
  tags:
    - kubernetes

  # Use image with kubectl
  image: registry.example.com/bitnami/kubectl:latest

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Manual trigger to prevent accidental deploys

  before_script:
    # 1. Create kubeconfig directory
    - mkdir -p $(dirname $KUBECONFIG)

    # 2. Write kubeconfig from secret variable
    - printf '%s' "$ACK_KUBECONFIG" > $KUBECONFIG

    # 3. Set permissions
    - chmod 600 $KUBECONFIG

    # 4. Verify kubeconfig
    - kubectl --kubeconfig=$KUBECONFIG config view

    # 5. Verify cluster connection
    - kubectl cluster-info

    # 6. Create image pull secret
    - kubectl create secret docker-registry acr-secret
        --docker-server=private-registry.example.com
        --docker-username=$ACR_USERNAME
        --docker-password=$ACR_PASSWORD
        --namespace=default
        --dry-run=client -o yaml | kubectl apply -f -

  script:
    # 1. Replace environment variables in YAML
    - envsubst < calculator.yaml > calculator-deploy.yaml

    # 2. Apply deployment
    - kubectl --kubeconfig=$KUBECONFIG apply -f calculator-deploy.yaml

    # 3. Output success message
    - echo "Deployment complete to cluster"

  after_script:
    # Clean up sensitive files
    - rm -f $KUBECONFIG calculator-deploy.yaml
```

Let's break down each section.

---

## Build Job Deep Dive

### Global Variables

```yaml
variables:
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
```

**Why this format?**
- `CI_COMMIT_SHORT_SHA`: Git commit hash (8 characters) - ensures traceability
- `CI_PIPELINE_ID`: Unique pipeline ID - prevents tag collisions on re-runs
- Example result: `my-app:56cc4792-86`

**Alternative strategies**:
```yaml
# Semantic versioning
DOCKER_IMAGE: "registry.com/app:v1.2.3"

# Date-based
DOCKER_IMAGE: "registry.com/app:$(date +%Y%m%d-%H%M%S)"

# Branch + SHA
DOCKER_IMAGE: "registry.com/app:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHORT_SHA}"
```

### Docker-in-Docker Configuration

```yaml
services:
  - name: registry.example.com/library/docker:dind
    alias: docker
    command:
      - "--tls=false"
      - "--insecure-registry=private-registry.example.com"
```

**Key parameters**:

| Parameter | Purpose | Security Note |
|-----------|---------|---------------|
| `alias: docker` | Network hostname for DinD | Without this, hostname would be full image path |
| `--tls=false` | Disable TLS between client/server | ⚠️ Use only in trusted networks |
| `--insecure-registry` | Allow non-HTTPS registries | Needed for internal registries without certs |

### Waiting for Docker Daemon

This is critical — DinD takes 5-10 seconds to start:

```yaml
before_script:
  - |
    timeout=60
    until docker info >/dev/null 2>&1; do
      if [ $timeout -le 0 ]; then
        exit 1
      fi
      sleep 2
      timeout=$((timeout - 2))
    done
```

**What happens without this?**
```
$ docker build -t myapp .
Cannot connect to the Docker daemon at tcp://docker:2375
Is the docker daemon running?
❌ Job fails immediately
```

**With proper waiting**:
```
Waiting... (60s remaining)
Waiting... (58s remaining)
✅ Docker daemon ready
Successfully connected to server
```

### Build and Push Process

```yaml
script:
  - docker build -t $DOCKER_IMAGE .
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin private-registry.example.com
  - docker push $DOCKER_IMAGE
```

**Execution timeline** (from Part 2's timing analysis):

```
T+10s   docker build starts
        ├─ DinD pulls base image: python:3.9-slim (35s)
        ├─ Copies application code (1s)
        ├─ Runs pip install (15s)
        └─ Creates image layers

T+61s   Build complete
        Image ID: sha256:abc123...

T+62s   docker login

T+65s   docker push starts
        ├─ Layer 1: already exists (skip)
        ├─ Layer 2: already exists (skip)
        ├─ Layer 3: Pushed (new code)
        └─ Manifest pushed

T+95s   Push complete
        Tag: 56cc4792-86
```

**Total build-job time**: ~95 seconds (varies by image size)

---

## Deploy Job Deep Dive

The deploy job runs in a separate Pod with a `kubectl` image.

### Authentication Setup

```yaml
before_script:
  - printf '%s' "$ACK_KUBECONFIG" > $KUBECONFIG
  - chmod 600 $KUBECONFIG
  - kubectl cluster-info
```

**Important**: The kubeconfig is stored as a GitLab CI/CD variable:

1. Go to **Settings** → **CI/CD** → **Variables**
2. Add variable:
   - Key: `ACK_KUBECONFIG`
   - Value: Contents of your `~/.kube/config` (base64 encoded or raw)
   - Type: File or Variable
   - Protected: ✅
   - Masked: ❌ (too long to mask)

### Image Pull Secret Creation

```yaml
kubectl create secret docker-registry acr-secret \
  --docker-server=private-registry.example.com \
  --docker-username=$ACR_USERNAME \
  --docker-password=$ACR_PASSWORD \
  --namespace=default \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Why `--dry-run=client -o yaml | kubectl apply`?**
- `--dry-run=client`: Generate YAML without creating resource
- `| kubectl apply`: Apply the YAML (idempotent - safe to run multiple times)

**Without this**, you'd get:
```
Error from server (AlreadyExists): secrets "acr-secret" already exists
❌ Job fails on second run
```

**With this approach**:
```
secret/acr-secret configured
✅ Works every time (updates if exists)
```

### Environment Variable Substitution

This is the magic that injects the dynamic image tag:

```yaml
script:
  - envsubst < calculator.yaml > calculator-deploy.yaml
  - kubectl apply -f calculator-deploy.yaml
```

**Input** (`calculator.yaml`):
```yaml
spec:
  containers:
  - name: calculator
    image: ${DOCKER_IMAGE}  # ← Variable placeholder
```

**After `envsubst`** (`calculator-deploy.yaml`):
```yaml
spec:
  containers:
  - name: calculator
    image: private-registry.example.com/my-namespace/my-app:56cc4792-86  # ← Replaced!
```

**How it works**:
1. `envsubst` reads the file
2. Finds variables matching `${VAR_NAME}`
3. Replaces with environment variable values
4. Outputs to new file

---

## Kubernetes Deployment Configuration

### calculator.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calculator
  namespace: default
spec:
  replicas: 2  # Two pods for high availability

  selector:
    matchLabels:
      app: calculator

  template:
    metadata:
      labels:
        app: calculator

    spec:
      # Use the secret created by deploy job
      imagePullSecrets:
      - name: acr-secret

      containers:
      - name: calculator
        image: ${DOCKER_IMAGE}  # Will be replaced by envsubst
        imagePullPolicy: Always  # Always pull (tag changes every pipeline)

        ports:
        - name: http
          containerPort: 30194

        # Mount host timezone (optional)
        volumeMounts:
        - name: host-time
          mountPath: /etc/localtime
          readOnly: true

      volumes:
      - name: host-time
        hostPath:
          path: /etc/localtime

---
apiVersion: v1
kind: Service
metadata:
  name: calculator
  namespace: default
spec:
  type: NodePort

  selector:
    app: calculator

  ports:
  - port: 30194        # Service port
    targetPort: 30194  # Container port
    nodePort: 30194    # Node port (30000-32767 range)
```

### Key Configuration Points

**1. imagePullPolicy: Always**

```yaml
imagePullPolicy: Always
```

**Why?** Because our tag changes every time:
- Pipeline 85: `my-app:56cc4792-85`
- Pipeline 86: `my-app:56cc4792-86`

Without `Always`, Kubernetes might use a cached image.

**2. imagePullSecrets**

```yaml
imagePullSecrets:
- name: acr-secret
```

This references the secret created in `deploy-job` before_script. Without it:
```
Failed to pull image: pull access denied
ErrImagePull
```

**3. NodePort Service**

```yaml
type: NodePort
nodePort: 30194
```

**Allows external access**:
```bash
# Access from any node IP
curl http://192.168.1.101:30194/
curl http://192.168.1.102:30194/
```

**Alternative**: Use `LoadBalancer` type if you have a cloud load balancer.

---

## Variable Flow Through Pipeline

Let's trace how the image tag flows from pipeline start to deployment:

```
┌─────────────────────────────────────────────────────────┐
│ 1. Pipeline starts                                      │
│    CI_COMMIT_SHORT_SHA = "56cc4792"                     │
│    CI_PIPELINE_ID = "86"                                │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Global variable synthesized                          │
│    DOCKER_IMAGE = "registry.../app:56cc4792-86"         │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 3. build-job uses variable                              │
│    $ docker build -t $DOCKER_IMAGE .                    │
│    $ docker push $DOCKER_IMAGE                          │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 4. deploy-job inherits variable (same pipeline)         │
│    $ envsubst < calculator.yaml                         │
│    Reads: $DOCKER_IMAGE                                 │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Written to Kubernetes YAML                           │
│    image: registry.../app:56cc4792-86                   │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 6. Kubelet pulls image                                  │
│    Using imagePullSecrets: acr-secret                   │
│    Result: Pod running with new image                   │
└─────────────────────────────────────────────────────────┘
```

---

## Complete Execution Example

Let's walk through a real deployment from start to finish.

### Step 1: Developer Makes Changes

```bash
# Fix a bug in app.py
vim app.py

# Commit changes
git add app.py
git commit -m "fix: correct division by zero error"
git push origin main
```

### Step 2: Pipeline Triggered

GitLab UI shows:
```
Pipeline #86 started
├── Stage: build
│   └── build-job (running)
└── Stage: deploy
    └── deploy-job (waiting for manual trigger)
```

### Step 3: build-job Execution

```
[00:00] Job Pod created
[00:03] Helper cloned code
[00:05] DinD started, Build container started
[00:10] docker build begins
  ├── Pulling base image python:3.9-slim (35s)
  ├── COPY app.py . (1s)
  ├── pip install flask (15s)
  └── Build complete
[01:00] docker login to registry
[01:05] docker push
  ├── Layer 1: already exists
  ├── Layer 2: already exists
  ├── Layer 3: Pushed (new code)
  └── Tag: 56cc4792-86
[01:40] build-job completed ✅
```

### Step 4: Manual Deploy Trigger

Developer clicks **Run** on deploy-job in GitLab UI.

### Step 5: deploy-job Execution

```
[00:00] Deploy Pod created
[00:03] Helper cloned code
[00:05] kubectl container started
[00:08] Verified kubeconfig
[00:10] kubectl cluster-info (connected)
[00:12] Created acr-secret
[00:15] envsubst replaced image variable
  Input:  image: ${DOCKER_IMAGE}
  Output: image: registry.../app:56cc4792-86
[00:18] kubectl apply -f calculator-deploy.yaml
  ├── Deployment "calculator" configured
  └── Service "calculator" unchanged
[00:20] deploy-job completed ✅
```

### Step 6: Kubernetes Rolling Update

```
[00:20] Deployment detects image change
[00:22] Creates new ReplicaSet
[00:25] Starts new Pod (calculator-767f559f9c-xxxxx)
  ├── Pulls image (using acr-secret)
  ├── Image downloaded (3s)
  └── Container starts
[00:30] New Pod ready
[00:32] Deletes old Pod
[00:35] Rolling update complete ✅
```

### Step 7: Verification

```bash
# Check Pod status
kubectl get pods -n default -l app=calculator

NAME                          READY   STATUS    RESTARTS   AGE
calculator-767f559f9c-2j7vz   1/1     Running   0          5m
calculator-767f559f9c-5qnx4   1/1     Running   0          5m

# Test application
curl http://192.168.1.101:30194/
{"message": "Calculator API", "version": "1.0"}

# Test the fix
curl -X POST http://192.168.1.101:30194/add \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 5}'

{"result": 15}
```

---

## Common Issues and Solutions

### Issue 1: Build fails - "Cannot connect to Docker daemon"

**Symptom**:
```
Cannot connect to the Docker daemon at tcp://docker:2375
```

**Cause**: DinD not ready yet

**Solution**: Add proper wait logic in `before_script` (as shown above)

### Issue 2: Push fails - "unauthorized: authentication required"

**Symptom**:
```
Error response from daemon: Get "https://registry.../": unauthorized
```

**Cause**: Not logged in to registry

**Solution**:
```yaml
script:
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
  - docker push $DOCKER_IMAGE
```

Ensure `ACR_USERNAME` and `ACR_PASSWORD` are set in GitLab CI/CD variables.

### Issue 3: Deployment fails - "ErrImagePull"

**Symptom**:
```
Failed to pull image: pull access denied
```

**Causes and solutions**:

**1. Missing imagePullSecrets**:
```yaml
# Add to Pod spec:
imagePullSecrets:
- name: acr-secret
```

**2. Secret not created**:
```bash
# Verify secret exists
kubectl get secret acr-secret -n default

# If not, check deploy-job logs
```

**3. Wrong credentials**:
```bash
# Test manually
echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
```

### Issue 4: envsubst doesn't replace variables

**Symptom**:
```yaml
# After envsubst, still shows:
image: ${DOCKER_IMAGE}
```

**Cause**: Variable not exported or wrong syntax

**Solution**:
```yaml
# Ensure variable is defined in global or job level:
variables:
  DOCKER_IMAGE: "registry.../app:${CI_COMMIT_SHORT_SHA}"

# Use ${VAR} not $VAR in YAML:
image: ${DOCKER_IMAGE}  # ✅ Correct
image: $DOCKER_IMAGE    # ❌ Won't work with envsubst
```

---

## Best Practices

### 1. Image Tagging Strategy

**✅ Good - Traceability**:
```yaml
DOCKER_IMAGE: "registry.../app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
```

**❌ Avoid - No history**:
```yaml
DOCKER_IMAGE: "registry.../app:latest"
```

### 2. Manual Deploy Trigger

**✅ Good - Prevent accidents**:
```yaml
deploy-job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

**❌ Risky - Auto-deploys**:
```yaml
deploy-job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
```

### 3. Resource Limits

**✅ Good - Prevent OOM**:
```yaml
# In gitlab-runner-values.yaml
cpu_limit = "2"
memory_limit = "2Gi"
```

**❌ Avoid - Unlimited resources**:
```yaml
# No limits set = can exhaust cluster resources
```

### 4. Cleanup Sensitive Data

**✅ Good - Security**:
```yaml
after_script:
  - rm -f $KUBECONFIG
  - rm -f calculator-deploy.yaml
```

**❌ Risky - Leaves credentials**:
```yaml
# No cleanup = kubeconfig persists in Pod (until deletion)
```

---

## Key Takeaways

**Pipeline Structure**:
- Two stages: `build` (automatic) and `deploy` (manual)
- Build stage creates and pushes Docker image
- Deploy stage applies changes to Kubernetes

**Docker-in-Docker**:
- Always wait for daemon to be ready
- Use proper authentication for private registries
- Tag images with traceable identifiers

**Kubernetes Deployment**:
- Use `envsubst` for dynamic variable injection
- Set `imagePullPolicy: Always` for frequently changing tags
- Create imagePullSecrets before deployment

**Variable Flow**:
- GitLab → Pipeline variables → Build → Registry
- Pipeline variables → Deploy → envsubst → Kubernetes

---

## What's Next?

In Part 3, we built a complete CI/CD workflow. But what happens when things go wrong? In particular, **network and DNS issues** are the most common problems in Kubernetes deployments.

**Coming up in Part 4**:
- 🌐 **Kubernetes networking deep dive** — 5 network layers explained
- 🔍 **DNS resolution troubleshooting** — Why DinD can't pull images
- 📡 **Packet flow analysis** — Following traffic from Pod to internet
- 🛠️ **Real debugging session** — Solving actual DNS failure

**Preview**: We'll walk through a real troubleshooting session where DinD couldn't pull images due to DNS misconfiguration, showing exactly how we diagnosed and fixed it.

---

## Practice Exercises

**Exercise 1**: Implement multi-stage deployment

Add a `test` stage between `build` and `deploy`:
```yaml
stages:
  - build
  - test
  - deploy

test-job:
  stage: test
  image: python:3.9-slim
  script:
    - pip install pytest
    - pytest tests/
```

**Exercise 2**: Add health checks to deployment

Modify `calculator.yaml`:
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 30194
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 30194
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Exercise 3**: Monitor deployment progress

```bash
# Watch rolling update in real-time
kubectl rollout status deployment/calculator -n default

# View rollout history
kubectl rollout history deployment/calculator -n default

# Rollback if needed
kubectl rollout undo deployment/calculator -n default
```

---

## Resources

- [GitLab CI/CD YAML Reference](https://docs.gitlab.com/ee/ci/yaml/)
- [Docker Build Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**Part 4 coming soon** — we'll tackle the most challenging aspect: network and DNS configuration.

If you found this helpful, feel free to share it with others learning GitLab CI/CD.

---

*This is Part 3 of a 5-part series on GitLab Runner on Kubernetes, documenting the learning and implementation process.*

**Series index**:
- [Part 1: Architecture & Quick Setup](link)
- [Part 2: Deep Dive into Container Architecture](link)
- **Part 3: Building a Real-World CI/CD Pipeline** ← You are here
- Part 4: Solving DNS and Network Issues (coming soon)
- Part 5: Production Best Practices (coming soon)

---

*Tags: #GitLab #Kubernetes #CICD #DevOps #Docker #ContainerRegistry*
