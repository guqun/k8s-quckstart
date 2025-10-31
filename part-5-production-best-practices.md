# GitLab Runner on Kubernetes: A Complete Guide
## Part 5 — Production Best Practices & Troubleshooting

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

**About Part 5**: Unlike the previous four parts which document my actual learning and deployment experience, this final part was entirely written by AI. It combines general industry best practices with the knowledge from Parts 1-4, serving as a comprehensive reference guide for production deployments. My hope is that it will be a valuable resource for my future work and for others deploying GitLab Runner at scale.

> *This is the final part! Catch up: [Part 1](link) (setup), [Part 2](link) (architecture), [Part 3](link) (pipeline), [Part 4](link) (networking).*

---

In the previous four parts, we built a complete GitLab Runner on Kubernetes. It works, but is it ready for production? In this final part, we'll cover:

- Configuration strategies for different environments
- Common production issues and solutions
- Performance optimization techniques
- Monitoring and operational considerations
- Security recommendations (general best practices)

This part combines **lessons from my actual deployment** (Parts 1-4) with **general industry best practices** for running CI/CD infrastructure at scale.

---

## DNS Configuration Strategies

DNS configuration is the most critical aspect we've covered (Part 4). Let's review the strategies for different environments.

### Strategy 1: Internal Network (Our Implementation)

**Environment**: Private datacenter or VPC without direct internet access

**Configuration**:
```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        dns_policy = "none"

        [runners.kubernetes.dns_config]
          nameservers = [
            "10.0.0.2",      # VPC DNS (primary)
            "10.1.0.251",    # Internal DNS (backup)
            "10.1.0.252"     # Internal DNS (backup)
          ]
          searches = [
            "gitlab.svc.cluster.local",
            "svc.cluster.local",
            "cluster.local"
          ]
          [[runners.kubernetes.dns_config.options]]
            name = "ndots"
            value = "2"
```

**Advantages**:
- ✅ Works with internal registries
- ✅ Resolves private domain names
- ✅ Maintains Kubernetes service discovery (search domains)

**Limitations**:
- ⚠️ Kubernetes service DNS may be slower (search domain traversal)
- ⚠️ Requires VPC DNS to be reachable from Pods

### Strategy 2: Public Cloud with Internet Access

**Environment**: Public cloud (AWS, GCP, Azure) with direct internet

**Configuration**:
```yaml
# Use default kube-dns with upstream DNS
# Ensure CoreDNS forwards to public DNS

# Edit CoreDNS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 1.1.1.1  # ← Add this line
        cache 30
        loop
        reload
        loadbalance
    }
```

**Advantages**:
- ✅ Fast Kubernetes service resolution
- ✅ Works with public registries (Docker Hub, etc.)
- ✅ Simple configuration

**Limitations**:
- ⚠️ May not work with internal services
- ⚠️ Depends on public DNS availability

### Strategy 3: Hybrid Environment

**Environment**: Mix of internal and external services

**Configuration**:
```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        dns_policy = "none"

        [runners.kubernetes.dns_config]
          nameservers = [
            "10.96.0.10",   # kube-dns (K8s services)
            "10.0.0.2",     # VPC DNS (internal)
            "8.8.8.8"       # Public DNS (fallback)
          ]
          searches = [
            "svc.cluster.local",
            "cluster.local"
          ]
          [[runners.kubernetes.dns_config.options]]
            name = "ndots"
            value = "2"
          [[runners.kubernetes.dns_config.options]]
            name = "timeout"
            value = "2"
```

**Advantages**:
- ✅ Covers all scenarios
- ✅ Multiple fallback options

**Limitations**:
- ⚠️ DNS queries may be slower (tries multiple servers)
- ⚠️ Must ensure all DNS servers are reachable

---

## Resource Management and Performance

### Capacity Planning

Based on the four-container architecture (Part 2), calculate your cluster capacity:

**Resource consumption per job**:
```
Build Container:  2 CPU, 2Gi memory
DinD Service:     2 CPU, 2Gi memory
Helper Container: 0.5 CPU, 512Mi memory
-------------------------------------------
Total per job:    4.5 CPU, 4.5Gi memory
```

**Cluster capacity example**:
```
3 worker nodes × 8 cores × 32GB RAM each

Available for jobs (assuming 80% allocatable):
- CPU: 3 × 8 × 0.8 = 19.2 cores
- Memory: 3 × 32GB × 0.8 = 76.8GB

Theoretical max concurrent jobs:
- By CPU: 19.2 ÷ 4.5 = 4 jobs
- By memory: 76.8 ÷ 4.5 = 17 jobs
- Bottleneck: CPU → ~4 concurrent jobs

Practical capacity (50% average utilization):
- ~8-10 concurrent jobs comfortably
```

### Tuning Resource Limits

Adjust based on your workload:

**For small projects (fast builds)**:
```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        cpu_limit = "1"
        memory_limit = "1Gi"
        cpu_request = "250m"
        memory_request = "256Mi"

        service_cpu_limit = "1"
        service_memory_limit = "1Gi"
        service_cpu_request = "250m"
        service_memory_request = "256Mi"
```

**For large projects (complex builds)**:
```yaml
        cpu_limit = "4"
        memory_limit = "4Gi"
        cpu_request = "1"
        memory_request = "1Gi"

        service_cpu_limit = "4"
        service_memory_limit = "4Gi"
        service_cpu_request = "1"
        service_memory_request = "1Gi"
```

### Adjusting Concurrency

**Current configuration** (from Part 1):
```yaml
replicas: 3      # 3 Runner Manager pods
concurrent: 10   # 10 jobs per manager
# Total capacity: 3 × 10 = 30 jobs
```

**Scaling strategies**:

**Horizontal scaling** (more managers):
```yaml
replicas: 5
concurrent: 10
# Total: 5 × 10 = 50 jobs
```

**Vertical scaling** (more jobs per manager):
```yaml
replicas: 3
concurrent: 15
# Total: 3 × 15 = 45 jobs
```

**Combined**:
```yaml
replicas: 4
concurrent: 12
# Total: 4 × 12 = 48 jobs
```

### Node Affinity for CI/CD Workloads

Dedicate specific nodes to CI/CD:

```bash
# Label nodes for CI/CD
kubectl label nodes worker-2 workload=ci
kubectl label nodes worker-3 workload=ci

# Taint to prevent other workloads
kubectl taint nodes worker-2 workload=ci:NoSchedule
kubectl taint nodes worker-3 workload=ci:NoSchedule
```

```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        node_selector = { "workload" = "ci" }

        [[runners.kubernetes.node_tolerations]]
          key = "workload"
          operator = "Equal"
          value = "ci"
          effect = "NoSchedule"
```

---

## Common Production Issues

Let me share the issues I encountered and how to resolve them.

### Issue 1: Runner Registration Fails

**Symptom**:
```
ERROR: Registering runner... failed
error=couldn't execute POST against http://gitlab.example.com/api/v4/runners:
dial tcp: i/o timeout
```

**Diagnosis steps**:

1. **Check network connectivity from Pod**:
```bash
kubectl run nettest --rm -i --image=alpine -n gitlab -- \
  wget -O- http://gitlab.example.com
```

2. **Check Runner Manager logs**:
```bash
kubectl logs -n gitlab -l app=gitlab-runner --tail=50
```

3. **Verify GitLab URL in configuration**:
```bash
helm get values gitlab-runner -n gitlab | grep gitlabUrl
```

**Solutions**:

**A. GitLab URL is incorrect**:
```yaml
# Fix URL in values file
gitlabUrl: "http://gitlab.example.com/"  # Add trailing slash
```

**B. Network policy blocking**:
```bash
# Check if NetworkPolicy exists
kubectl get networkpolicy -n gitlab

# Temporarily allow all egress for testing
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: gitlab
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - {}
EOF
```

**C. Security group/firewall rules**:
- Ensure Kubernetes nodes can reach GitLab server
- Check cloud provider security groups
- Verify on-premise firewall rules

### Issue 2: Job Pod Image Pull Failures

**Symptom**:
```
Failed to pull image "registry.example.com/docker:latest":
rpc error: code = Unknown desc = failed to pull and unpack image:
failed to resolve reference: failed to do request:
Head "https://registry.example.com/...": dial tcp: lookup registry.example.com: i/o timeout
```

**Diagnosis** (from Part 4):

1. **Check Pod DNS**:
```bash
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 10 dnsPolicy
```

2. **Test DNS resolution**:
```bash
kubectl exec -n gitlab <job-pod> -c build -- nslookup registry.example.com
```

3. **Check DNS server reachability**:
```bash
kubectl exec -n gitlab <job-pod> -c build -- nc -zv 10.0.0.2 53
```

**Solution**: Follow Part 4's DNS configuration strategy.

### Issue 3: DinD Container OOM Killed

**Symptom**:
```
Error from server: container "docker" in pod "runner-xxx" is terminated (OOMKilled)
```

**Diagnosis**:

1. **Check Pod events**:
```bash
kubectl describe pod <job-pod> -n gitlab | grep -A 10 Events
```

2. **Check actual memory usage** (while job running):
```bash
kubectl top pod <job-pod> -n gitlab --containers
```

**Solutions**:

**A. Increase memory limits**:
```yaml
# gitlab-runner-values.yaml
service_memory_limit = "4Gi"    # Increase from 2Gi
service_memory_request = "2Gi"  # Increase from 1Gi
```

**B. Optimize Docker builds**:
```dockerfile
# Use smaller base images
FROM python:3.9-slim  # Good
# FROM python:3.9     # Too large

# Combine RUN commands
RUN apt-get update && \
    apt-get install -y pkg1 pkg2 && \
    rm -rf /var/lib/apt/lists/*  # Clean up

# Use multi-stage builds
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install

FROM node:18-slim
COPY --from=builder /app/node_modules ./node_modules
```

### Issue 4: Job Pod Fails to Schedule

**Symptom**:
```
0/3 nodes are available: 3 Insufficient memory.
```

**Diagnosis**:

1. **Check node resources**:
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

2. **Check pending pods**:
```bash
kubectl get pods -n gitlab | grep Pending
kubectl describe pod <pending-pod> -n gitlab
```

**Solutions**:

**A. Reduce resource requests** (allows more jobs):
```yaml
cpu_request = "250m"      # Down from 500m
memory_request = "256Mi"  # Down from 512Mi
```

**B. Add more nodes** or **scale up existing nodes**

**C. Use pod priority** (important jobs first):
```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        priority_class_name = "high-priority-ci"
```

Create PriorityClass:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-ci
value: 1000
globalDefault: false
description: "High priority for CI/CD jobs"
```

### Issue 5: Slow Image Pulls

**Symptom**: Jobs take 2-5 minutes just pulling images

**Solutions**:

**A. Use image pull policy wisely**:
```yaml
# .gitlab-ci.yml
build-job:
  image: docker:latest

variables:
  # Cache images on nodes
  DOCKER_PULL_POLICY: if-not-present
```

**B. Pre-pull common images** on all nodes:
```yaml
# DaemonSet to pre-pull images
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
  namespace: gitlab
spec:
  selector:
    matchLabels:
      app: prepuller
  template:
    metadata:
      labels:
        app: prepuller
    spec:
      containers:
      - name: docker
        image: docker:latest
        command: ["sh", "-c", "sleep infinity"]
      - name: kubectl
        image: bitnami/kubectl:latest
        command: ["sh", "-c", "sleep infinity"]
```

**C. Use a registry mirror/cache**:
```yaml
# In DinD configuration
services:
  - name: docker:dind
    command:
      - "--registry-mirror=http://registry-cache.local:5000"
```

### Issue 6: Jobs Stuck in Pending

**Symptom**: Job Pod stays in Pending state, never starts

**Common causes**:

1. **No nodes match nodeSelector**:
```bash
# Check node labels
kubectl get nodes --show-labels

# Remove restrictive nodeSelector if needed
```

2. **Persistent Volume claims not bound** (if using):
```bash
kubectl get pvc -n gitlab
```

3. **Image pull backoff**:
```bash
kubectl describe pod <pending-pod> -n gitlab | grep -i image
```

4. **Admission webhook blocking**:
```bash
# Check admission controller logs
kubectl logs -n kube-system -l component=kube-apiserver
```

---

## Performance Optimization

### Reduce DNS Query Latency

From our DNS configuration (Part 4), optimize query speed:

```yaml
[runners.kubernetes.dns_config]
  nameservers = [
    "10.0.0.2"  # Only fast, reachable DNS servers
  ]

  [[runners.kubernetes.dns_config.options]]
    name = "timeout"
    value = "1"      # Reduce from default 5s

  [[runners.kubernetes.dns_config.options]]
    name = "attempts"
    value = "2"      # Reduce from default 5
```

### Optimize Job Pod Startup

**Current timeline** (from Part 2):
```
T+0s   Pod created
T+3s   Helper clones code
T+8s   DinD ready
T+10s  Job starts
```

**Optimizations**:

**A. Use smaller helper image**:
```yaml
helper_image = "gitlab/gitlab-runner-helper:alpine-x86_64-v18.3.0"
# Instead of the full version
```

**B. Reduce clone depth**:
```yaml
variables:
  GIT_DEPTH: 1  # Shallow clone
  GIT_STRATEGY: fetch  # Reuse if possible
```

**C. Cache dependencies**:
```yaml
# .gitlab-ci.yml
build-job:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .pip-cache/
```

### Concurrent Build Optimization

For multiple jobs on the same project:

```yaml
# .gitlab-ci.yml
# Run tests in parallel
test-unit:
  stage: test
  script: pytest tests/unit

test-integration:
  stage: test
  script: pytest tests/integration

test-e2e:
  stage: test
  script: pytest tests/e2e

# All three run simultaneously
```

---

## Monitoring and Operations

### Basic Health Checks

**Check Runner Manager health**:
```bash
# View Runner Manager pods
kubectl get pods -n gitlab -l app=gitlab-runner

# Check logs for errors
kubectl logs -n gitlab -l app=gitlab-runner --tail=100

# Watch for job pod creation
kubectl get pods -n gitlab -w
```

**Monitor resource usage**:
```bash
# Runner Manager resources
kubectl top pod -n gitlab -l app=gitlab-runner

# Job Pod resources (while running)
kubectl top pod -n gitlab | grep runner-.*concurrent
```

### Prometheus Metrics

Runner exposes metrics on port 9252:

```yaml
# gitlab-runner-values.yaml
metrics:
  enabled: true
  port: 9252
  portName: metrics
```

**Access metrics**:
```bash
# Port-forward to view metrics
kubectl port-forward -n gitlab deployment/gitlab-runner 9252:9252

# View metrics
curl http://localhost:9252/metrics
```

**Key metrics**:
```
# Number of concurrent jobs
gitlab_runner_jobs{state="running"}

# Job duration
gitlab_runner_job_duration_seconds

# Runner errors
gitlab_runner_errors_total
```

### Log Management

**View logs by component**:

```bash
# Runner Manager logs
kubectl logs -n gitlab -l app=gitlab-runner -f

# Specific job logs
kubectl logs -n gitlab <job-pod> -c build -f
kubectl logs -n gitlab <job-pod> -c docker -f

# All containers in job pod
kubectl logs -n gitlab <job-pod> --all-containers=true
```

**Export logs** for analysis:
```bash
# Export Runner logs
kubectl logs -n gitlab -l app=gitlab-runner \
  --since=24h > runner-logs-$(date +%Y%m%d).txt

# Export all job pod logs
for pod in $(kubectl get pods -n gitlab -o name | grep runner-); do
  kubectl logs -n gitlab $pod --all-containers=true > ${pod}-logs.txt
done
```

---

## Security Recommendations

> **Note**: The following are general security best practices for Kubernetes CI/CD environments. Adapt these to your organization's security policies.

### 1. Network Policies

Restrict network access for Job Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gitlab-runner-policy
  namespace: gitlab
spec:
  podSelector:
    matchLabels:
      app: gitlab-runner
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow GitLab server
  - to:
    - podSelector:
        matchLabels:
          app: gitlab
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
  # Allow image registry
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

### 2. RBAC (Role-Based Access Control)

Minimize permissions for Runner ServiceAccount:

```yaml
# Service account for Runner
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-runner
  namespace: gitlab

---
# Role for managing job pods
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: gitlab
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]

---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner
  namespace: gitlab
subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: gitlab
roleRef:
  kind: Role
  name: gitlab-runner
  apiGroup: rbac.authorization.k8s.io
```

**For deploy jobs** (needs cluster access):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner-deploy
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "secrets"]
  verbs: ["get", "list", "create", "update"]
```

### 3. Secrets Management

**Use GitLab CI/CD variables** for sensitive data:

```yaml
# In GitLab project: Settings → CI/CD → Variables
# Add:
- ACR_USERNAME (Protected: ✅, Masked: ❌)
- ACR_PASSWORD (Protected: ✅, Masked: ✅)
- KUBECONFIG (Protected: ✅, Masked: ❌, Type: File)
```

**Never hardcode secrets**:
```yaml
# ❌ Bad
script:
  - docker login -u admin -p mypassword registry.com

# ✅ Good
script:
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
```

**Rotate secrets regularly**:
- Update GitLab CI/CD variables
- Update Kubernetes secrets
- Update imagePullSecrets

### 4. Image Security

**General recommendations**:

- Use specific image versions, not `latest`
- Scan images for vulnerabilities (tools: Trivy, Clair)
- Use minimal base images (alpine, distroless)
- Keep base images updated

**Example secure Dockerfile**:
```dockerfile
# Use specific version
FROM python:3.9.18-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

# Install dependencies as root
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Switch to non-root user
USER appuser
WORKDIR /home/appuser

# Copy application
COPY --chown=appuser:appuser app.py .

# Run as non-root
CMD ["python", "app.py"]
```

---

## Upgrade and Maintenance

### Upgrading GitLab Runner

```bash
# Check current version
helm list -n gitlab

# View available versions
helm search repo gitlab/gitlab-runner --versions | head -10

# Backup current configuration
helm get values gitlab-runner -n gitlab > backup-$(date +%Y%m%d).yaml

# Update chart repository
helm repo update

# Upgrade (use same values file)
helm upgrade gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab \
  --version 0.81.0 \
  -f gitlab-runner-values.yaml

# Verify upgrade
kubectl rollout status deployment/gitlab-runner -n gitlab
```

### Backup Strategy

**What to backup**:

1. **Helm values file**: `gitlab-runner-values.yaml`
2. **Runner registration token**: Stored in GitLab
3. **CI/CD variable definitions**: Document in version control
4. **Custom scripts and configurations**: Store in Git repository

**Restore process**:

1. Redeploy Runner using backed-up values file
2. Runner auto-registers using token from values
3. Job Pods will automatically have correct configuration

**No state to backup**: Runner is stateless; all state is in GitLab server.

---

## Production Readiness Checklist

Before going to production, verify:

**Infrastructure**:
- [ ] Cluster has sufficient resources for expected concurrent jobs
- [ ] Nodes are properly sized (CPU, memory, disk)
- [ ] Network policies are configured (if required)
- [ ] DNS resolution works for all required domains

**Runner Configuration**:
- [ ] DNS configuration tested and working (Part 4)
- [ ] Resource limits configured appropriately
- [ ] Multiple Runner Manager replicas for HA
- [ ] Concurrent job limit set correctly
- [ ] Node affinity/tolerations configured (if needed)

**Security**:
- [ ] RBAC configured with minimal permissions
- [ ] Secrets stored in GitLab variables, not code
- [ ] Network policies restrict unnecessary access
- [ ] Image pull secrets configured correctly
- [ ] DinD runs with `--tls=false` only in trusted networks

**Monitoring**:
- [ ] Prometheus metrics enabled and scraped
- [ ] Alerts configured for Runner failures
- [ ] Log aggregation in place
- [ ] Resource usage dashboards created

**Operations**:
- [ ] Upgrade procedure documented
- [ ] Backup strategy in place
- [ ] Troubleshooting runbook created
- [ ] On-call team trained

---

## Key Takeaways

**DNS Configuration**:
- Choose strategy based on environment (internal, public, hybrid)
- Test DNS resolution before deploying
- VPC DNS often works best for private clouds

**Resource Management**:
- Plan capacity based on 4.5 CPU / 4.5GB per job
- Monitor actual usage and adjust limits
- Use node affinity for dedicated CI/CD nodes

**Common Issues**:
- Most problems are DNS or network-related (Part 4)
- OOM kills indicate insufficient memory limits
- Pending pods indicate resource or scheduling constraints

**Production Operations**:
- Monitor Runner Manager and Job Pods
- Export metrics to Prometheus
- Implement log aggregation
- Maintain backup of configuration

**Security** (general recommendations):
- Minimize RBAC permissions
- Use NetworkPolicies to restrict access
- Manage secrets via GitLab variables
- Scan images for vulnerabilities

---

## Conclusion

Over this five-part series, we've covered:

1. **Part 1**: Architecture overview and initial setup
2. **Part 2**: Deep dive into the four-container model
3. **Part 3**: Building complete CI/CD pipelines
4. **Part 4**: Solving complex DNS and network issues
5. **Part 5**: Production best practices and troubleshooting

**The journey from learning to production**:
- Started with basic concepts and quick deployment
- Understood the internal architecture deeply
- Built real-world CI/CD workflows
- Debugged complex networking issues
- Applied production-grade practices

**Key lessons learned**:
- Kubernetes networking has five distinct layers
- DNS configuration is the most common pitfall
- Docker-in-Docker is client-server, not magic
- Resource planning prevents scheduling issues
- AI assistance can accelerate learning dramatically

This deployment now runs production workloads reliably. The knowledge gained through debugging (especially Part 4's DNS issues) was invaluable.

**Final thoughts**: Building GitLab Runner on Kubernetes is complex, but understanding each layer makes it manageable. The AI-assisted learning approach let me tackle problems systematically rather than through blind trial-and-error.

---

## Resources

**Official Documentation**:
- [GitLab Runner Documentation](https://docs.gitlab.com/runner/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)

**Related Topics**:
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC in Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Container Security Best Practices](https://kubernetes.io/docs/concepts/security/)

---

## Thank You

Thank you for following this series! If you found it helpful:
- Share it with others learning GitLab on Kubernetes
- Leave feedback on what worked or what could be improved
- Connect with me if you have questions

This series was possible because of AI-assisted learning with Claude Code. If you're exploring complex technologies, consider using AI as a learning companion — it can help you understand deeply rather than just copy solutions.

---

*This is Part 5 (final) of a 5-part series on GitLab Runner on Kubernetes, documenting the learning and implementation process.*

**Series index**:
- [Part 1: Architecture & Quick Setup](link)
- [Part 2: Deep Dive into Container Architecture](link)
- [Part 3: Building a Real-World CI/CD Pipeline](link)
- [Part 4: Solving DNS and Network Issues](link)
- **Part 5: Production Best Practices** ← You are here

---

*Tags: #GitLab #Kubernetes #CICD #DevOps #ProductionReady #BestPractices*
