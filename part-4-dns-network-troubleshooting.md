# GitLab Runner on Kubernetes: A Complete Guide
## Part 4 — Solving DNS and Network Issues

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

> *Catch up: [Part 1](link) (setup), [Part 2](link) (architecture), [Part 3](link) (CI/CD pipeline).*

---

In Part 3, we built a complete CI/CD pipeline. Everything worked... until it didn't. When I tried to build a Docker image in the pipeline, I encountered this error:

```
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

This single error led me down a deep rabbit hole of Kubernetes networking, DNS resolution, and container networking. What started as a simple "why can't DinD pull images?" turned into a comprehensive understanding of how network packets flow through five different layers in a Kubernetes environment.

In this part, I'll share:
- The five-layer Kubernetes network model
- Why DNS configuration is the most common pitfall
- My three failed attempts and what I learned from each
- The final solution and how to verify it works

Let's start by understanding the network architecture.

---

## The Five-Layer Network Model

When DinD tries to pull an image, the network request passes through **five distinct network layers**. Understanding this is crucial for debugging.

```
┌────────────────────────────────────────────────────────┐
│ Layer 5: DinD Internal Network (Docker-in-Docker)     │
│          172.17.0.0/16 (virtual, inside DinD)         │
└──────────────────────┬─────────────────────────────────┘
                       │
┌──────────────────────┴─────────────────────────────────┐
│ Layer 4: Container Network (shared by all containers) │
│          127.0.0.1 (localhost within Pod)             │
└──────────────────────┬─────────────────────────────────┘
                       │
┌──────────────────────┴─────────────────────────────────┐
│ Layer 3: Pod Network (CNI overlay network)            │
│          172.24.0.0/16 (Pod CIDR)                     │
└──────────────────────┬─────────────────────────────────┘
                       │
┌──────────────────────┴─────────────────────────────────┐
│ Layer 2: Service Network (virtual IPs)                │
│          172.21.0.0/16 (ClusterIP range)              │
└──────────────────────┬─────────────────────────────────┘
                       │
┌──────────────────────┴─────────────────────────────────┐
│ Layer 1: Node Network (physical/VM network)           │
│          192.168.1.0/24 (Node IPs)                    │
└──────────────────────┬─────────────────────────────────┘
                       │
┌──────────────────────┴─────────────────────────────────┐
│ Layer 0: VPC Network (datacenter/cloud network)       │
│          10.0.0.0/16 (VPC CIDR)                       │
└────────────────────────────────────────────────────────┘
```

Let's examine each layer.

### Layer 0: VPC Network

This is your datacenter or cloud provider network:

```
VPC: 10.0.0.0/16

Components:
- Kubernetes nodes: 192.168.1.100-103
- VPC DNS: 10.0.0.2
- NAT Gateway (for internet access)
- Internal DNS servers: 10.1.0.251, 10.1.0.252
```

**Key point**: Not all IPs in the VPC are reachable from Pods. This becomes important later.

### Layer 1: Node Network

Each Kubernetes node has a physical (or VM) network interface:

```
Master:  192.168.1.100
Worker1: 192.168.1.101
Worker2: 192.168.1.102
Worker3: 192.168.1.103
```

Nodes communicate directly at this layer. When Pods on different nodes communicate, traffic flows: Pod → Node → Network → Node → Pod.

### Layer 2: Service Network

Kubernetes Services get virtual IPs (ClusterIP):

```
kube-dns Service:    10.96.0.10
gitlab Service:      172.21.5.10
calculator Service:  172.21.8.20
```

**Important**: These IPs don't exist on any network interface. They're implemented by kube-proxy using iptables rules.

### Layer 3: Pod Network

Each Pod gets an IP from the CNI plugin (e.g., Flannel, Calico):

```
Node1 Pods: 172.24.1.0/24
Node2 Pods: 172.24.2.0/24
Node3 Pods: 172.24.3.0/24

Example Pod: 172.24.1.100
```

All containers in a Pod share this IP address.

### Layer 4: Container Network

All containers within a Pod share the **same network namespace**:

```
Job Pod Network Namespace:
- IP: 172.24.1.100
- Interfaces: lo (127.0.0.1), eth0 (172.24.1.100)
- Containers: helper, build, dind (all see same network)
```

This is why `docker:2375` resolves to `127.0.0.1:2375` — same namespace!

### Layer 5: DinD Internal Network

Inside the DinD container, dockerd creates **another virtual network**:

```
DinD docker0 bridge: 172.17.0.1
Temporary containers: 172.17.0.2, 172.17.0.3, ...
```

When DinD executes `RUN` commands in Dockerfile, it creates temporary containers in this network.

---

## The DNS Resolution Problem

Now that we understand the layers, let's see where DNS fits in.

### Default Kubernetes DNS Configuration

By default, Pods use kube-dns for DNS resolution:

```yaml
# Automatic Pod configuration
dnsPolicy: ClusterFirst

# Results in:
# /etc/resolv.conf
nameserver 10.96.0.10  # kube-dns
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**This works fine for**:
- Kubernetes services: `gitlab.gitlab.svc.cluster.local` ✅
- Internal services: `database.default.svc.cluster.local` ✅

**This fails for**:
- External domains: `registry.example.com` ❌
- Private registry domains: `private-registry.company.com` ❌

### Why Does It Fail?

kube-dns (CoreDNS) is designed to resolve Kubernetes services. To resolve external domains, it needs to forward requests to an upstream DNS server.

**Check your CoreDNS configuration**:
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

If you don't see a `forward` directive like this:
```
forward . 8.8.8.8 1.1.1.1
```

Then kube-dns **cannot resolve external domains**.

---

## My Debugging Journey: Three Failed Attempts

Let me walk you through my actual debugging process, including the failures.

### Attempt 1: Using Default DNS ❌

**Configuration**: No custom DNS settings

**Result**:
```
$ docker build -t myapp .
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

**Diagnosis**:
```bash
# Check DinD container DNS
kubectl exec -n gitlab <job-pod> -c docker -- cat /etc/resolv.conf

# Output:
nameserver 10.96.0.10  # kube-dns
search gitlab.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

# Test resolution
kubectl exec -n gitlab <job-pod> -c docker -- nslookup registry.example.com

# Output:
;; connection timed out; no servers could be reached
```

**Root cause**: kube-dns doesn't have upstream DNS configured.

**What I learned**: Default DNS won't work in internal networks without proper CoreDNS configuration.

### Attempt 2: Using --dns Parameter ❌

I thought: "Just tell DinD to use different DNS servers!"

**Configuration**:
```yaml
services:
  - name: docker:dind
    command:
      - "--dns=10.1.0.251"  # Node DNS server
      - "--dns=10.1.0.252"
```

**Expected**: DinD would use these DNS servers to resolve domains.

**Actual result**: Still failed!

```
$ docker build -t myapp .
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

**Diagnosis**:
```bash
# Check what --dns actually affects
kubectl exec -n gitlab <job-pod> -c docker -- cat /etc/resolv.conf

# Output:
nameserver 10.96.0.10  # Still kube-dns!
```

**Root cause**: The `--dns` parameter **doesn't affect dockerd itself**. It only affects the `/etc/resolv.conf` of containers that dockerd **creates** (like temporary containers during `RUN` commands).

```
┌─────────────────────────────────────────────────────┐
│ What --dns actually controls:                      │
│                                                     │
│ dockerd itself:        Uses DinD container's DNS   │
│                       (from Pod DNS config)         │
│                       ❌ NOT affected by --dns      │
│                                                     │
│ Temporary containers: Get custom /etc/resolv.conf  │
│ (RUN commands)       with --dns servers            │
│                       ✅ Affected by --dns          │
└─────────────────────────────────────────────────────┘
```

**What I learned**: The `--dns` parameter is for **temporary build containers**, not for dockerd's own DNS resolution.

### Attempt 3: Using Node DNS Directly ❌

Next attempt: "Configure Pod DNS to use the node's DNS servers."

**Configuration**:
```yaml
# gitlab-runner-values.yaml
[runners.kubernetes]
  dns_policy = "none"
  [runners.kubernetes.dns_config]
    nameservers = ["10.1.0.251", "10.1.0.252"]
```

**Expected**: Pods would use internal DNS servers.

**Actual result**: DinD container failed to start!

```
$ kubectl logs <job-pod> -c docker

time="..." level=error msg="failed to start daemon"
error="failed to initialize DNS resolver: dial tcp 10.1.0.251:53: i/o timeout"
```

**Diagnosis**:
```bash
# Test connectivity from a Pod to node DNS
kubectl run nettest --rm -i --restart=Never \
  --overrides='{
    "spec": {
      "dnsPolicy": "None",
      "dnsConfig": {"nameservers": ["10.1.0.251"]}
    }
  }' \
  --image=alpine -- nc -zv 10.1.0.251 53

# Output:
nc: 10.1.0.251 (10.1.0.251:53): Connection timed out
```

**Root cause**: The DNS servers at `10.1.0.251/252` are in a **different subnet** that Pods cannot reach.

```
Network topology:
┌─────────────────────────────────────────────┐
│ Node network: 10.202.3.0/24                 │
│ DNS servers:  10.1.0.251/252 (different!)  │
│ Routing: Nodes CAN reach (via router)      │
└─────────────────────────────────────────────┘
                   ↑ Reachable
                   │
┌─────────────────────────────────────────────┐
│ Pod network: 172.24.0.0/16 (overlay)        │
│ Routing: Pods CANNOT reach 10.1.0.x        │
└─────────────────────────────────────────────┘
                   ↓ Not reachable
```

**What I learned**: Pod network (overlay) != Node network (underlay). Not all node-reachable IPs are Pod-reachable.

---

## The Solution: VPC DNS

After three failures, I examined the node's DNS configuration more carefully:

```bash
# On a Kubernetes node
cat /etc/resolv.conf

nameserver 10.1.0.251  # Internal DNS 1
nameserver 10.1.0.252  # Internal DNS 2
nameserver 10.0.0.2    # VPC DNS ← What's this?
```

The third DNS server caught my attention: `10.0.0.2`. This is the **VPC-level DNS service**.

### Testing VPC DNS

```bash
# Create test Pod with VPC DNS
kubectl run dns-test --restart=Never \
  --image=alpine \
  --overrides='{
    "spec": {
      "dnsPolicy": "None",
      "dnsConfig": {"nameservers": ["10.0.0.2"]}
    }
  }' \
  -- nslookup registry.example.com

# Wait for completion
kubectl wait --for=condition=completed pod/dns-test --timeout=10s

# Check logs
kubectl logs dns-test

# Output:
Server:    10.0.0.2
Address:   10.0.0.2:53

Name:      registry.example.com
Address:   203.0.113.10

✅ DNS resolution successful!
```

### Why VPC DNS Works

```
VPC DNS (10.0.0.2) characteristics:
┌─────────────────────────────────────────────────┐
│ - Located in VPC network (10.0.0.0/16)         │
│ - Reachable from Pod network (route exists)    │
│ - Can resolve internal domains                  │
│ - Can resolve external domains                  │
│ - Similar to AWS's 169.254.169.253             │
└─────────────────────────────────────────────────┘
```

### Final Configuration

```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      executor = "kubernetes"
      [runners.kubernetes]
        dns_policy = "none"

        [runners.kubernetes.dns_config]
          nameservers = [
            "10.0.0.2",      # VPC DNS (primary)
            "10.1.0.251",    # Backup (may not be reachable)
            "10.1.0.252"     # Backup
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

### Deployment and Verification

```bash
# Update Runner
helm upgrade gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab \
  -f gitlab-runner-values.yaml

# Wait for rollout
kubectl rollout status deployment gitlab-runner -n gitlab

# Trigger a new pipeline
# Watch job pod creation
kubectl get pods -n gitlab -w

# Once job pod is running, verify DNS
kubectl exec -n gitlab <job-pod> -c build -- cat /etc/resolv.conf

# Expected:
nameserver 10.0.0.2
nameserver 10.1.0.251
nameserver 10.1.0.252
search gitlab.svc.cluster.local svc.cluster.local cluster.local
options ndots:2

# Test resolution
kubectl exec -n gitlab <job-pod> -c docker -- nslookup registry.example.com

# Expected:
Server:    10.0.0.2
Address:   10.0.0.2:53
Name:      registry.example.com
Address:   203.0.113.10
```

### Final Pipeline Test

```yaml
# .gitlab-ci.yml
test-dns:
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker info
    - docker build -t myapp:test .

# Output in GitLab UI:
Step 1/5 : FROM python:3.9-slim
latest: Pulling from library/python
a803e7c4b030: Pull complete
✅ Successfully built abc123def456
```

**Success!** DinD can now pull images.

---

## Understanding the Two DNS Configurations

This is the most confusing part, so let me clarify:

### Pod DNS vs. --dns Parameter

```
┌──────────────────────────────────────────────────────┐
│ Configuration 1: Pod DNS (gitlab-runner-values.yaml) │
└──────────────────────────────────────────────────────┘

[runners.kubernetes.dns_config]
  nameservers = ["10.0.0.2"]

Effect:
- Sets /etc/resolv.conf for ALL containers in Job Pod
- Affects:
  ✅ Helper container
  ✅ Build container
  ✅ DinD container (dockerd process itself)

When used:
- dockerd pulls images: FROM python:3.9-slim
  → Reads /etc/resolv.conf
  → Uses 10.0.0.2
  → ✅ Can resolve registry.example.com

┌──────────────────────────────────────────────────────┐
│ Configuration 2: --dns Parameter (.gitlab-ci.yml)    │
└──────────────────────────────────────────────────────┘

services:
  - name: docker:dind
    command: ["--dns=8.8.8.8"]

Effect:
- Sets /etc/resolv.conf for containers that dockerd CREATES
- Affects:
  ✅ Temporary containers during RUN commands
  ❌ Does NOT affect dockerd itself

When used:
- RUN pip install flask
  → dockerd creates temporary container
  → Container's /etc/resolv.conf: nameserver 8.8.8.8
  → pip needs to resolve pypi.org
  → Uses 8.8.8.8
  → ✅ Can resolve pypi.org

dockerd pulling base image:
  → Reads dockerd's own /etc/resolv.conf
  → ❌ NOT affected by --dns parameter
```

### Comparison Table

| Aspect | Pod DNS Config | --dns Parameter |
|--------|---------------|-----------------|
| **Config location** | `gitlab-runner-values.yaml` | `.gitlab-ci.yml` services |
| **Affects** | DinD container's `/etc/resolv.conf` | Temp container's `/etc/resolv.conf` |
| **When applied** | Container startup | dockerd creates container |
| **Affects dockerd** | ✅ Yes (dockerd reads its own resolv.conf) | ❌ No |
| **Use case** | Pull base images (FROM) | Install packages (RUN) |
| **Example** | `FROM python:3.9` DNS | `RUN pip install` DNS |

### Recommended Configuration

**For most cases** (including our scenario):

```yaml
# gitlab-runner-values.yaml - Set Pod DNS
[runners.kubernetes.dns_config]
  nameservers = ["10.0.0.2"]

# .gitlab-ci.yml - Don't set --dns
services:
  - name: docker:dind
    command:
      - "--tls=false"
      # No --dns parameter needed
```

**Why?**
- Pod DNS handles both dockerd pulling images AND temporary container DNS
- Temporary containers inherit DinD's DNS by default
- Simpler configuration

**When to use --dns**:

Only if temporary containers need **different** DNS than dockerd:

```yaml
# Example: dockerd uses internal DNS, but RUN commands need public DNS
services:
  - name: docker:dind
    command:
      - "--dns=8.8.8.8"  # RUN commands use public DNS

# Pod DNS still set to internal:
nameservers = ["10.0.0.2"]  # dockerd uses internal DNS
```

---

## Packet Flow Analysis

Let's trace what happens when DinD pulls an image:

### Step-by-Step Flow

```
Step 1: dockerd initiates image pull
┌────────────────────────────────────────────────┐
│ DinD Container                                 │
│ $ dockerd pull registry.example.com/app:latest│
│                                                │
│ dockerd needs to resolve domain                │
└───────────────────┬────────────────────────────┘
                    │
Step 2: Read /etc/resolv.conf
┌───────────────────┴────────────────────────────┐
│ /etc/resolv.conf (DinD container)              │
│ nameserver 10.0.0.2                            │
└───────────────────┬────────────────────────────┘
                    │
Step 3: Send DNS query
┌───────────────────┴────────────────────────────┐
│ DNS Query Packet                               │
│ Source: 172.24.1.100:34567 (Pod IP)           │
│ Dest:   10.0.0.2:53 (VPC DNS)                 │
│ Query:  registry.example.com A record         │
└───────────────────┬────────────────────────────┘
                    │
Step 4: Exit Pod network
┌───────────────────┴────────────────────────────┐
│ Pod eth0 → veth pair → cni0 bridge            │
└───────────────────┬────────────────────────────┘
                    │
Step 5: Node routing
┌───────────────────┴────────────────────────────┐
│ Node routing table:                            │
│ 10.0.0.2 via 10.202.3.1 dev eth0             │
└───────────────────┬────────────────────────────┘
                    │
Step 6: SNAT (Source NAT)
┌───────────────────┴────────────────────────────┐
│ iptables MASQUERADE                            │
│ Source: 192.168.1.101:34567 (Node IP)         │
│ Dest:   10.0.0.2:53                           │
└───────────────────┬────────────────────────────┘
                    │
Step 7: VPC network
┌───────────────────┴────────────────────────────┐
│ Packet reaches VPC DNS                         │
│ DNS resolves: registry.example.com            │
│ → 203.0.113.10                                │
└───────────────────┬────────────────────────────┘
                    │
Step 8: DNS response (reverse path)
┌───────────────────┴────────────────────────────┐
│ Response: 203.0.113.10                        │
│ → Node (reverse NAT)                           │
│ → cni0 → veth → Pod eth0                      │
│ → dockerd receives IP                          │
└────────────────────────────────────────────────┘

Step 9: HTTP connection to registry
┌────────────────────────────────────────────────┐
│ dockerd connects to 203.0.113.10:443         │
│ Same packet path as DNS query                 │
│ Downloads image layers                         │
│ ✅ Success                                     │
└────────────────────────────────────────────────┘
```

---

## Troubleshooting Checklist

When you encounter DNS issues, follow this checklist:

### 1. Check Pod DNS Configuration

```bash
# Get job pod name
kubectl get pods -n gitlab | grep runner-

# Check DNS policy
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 10 dnsPolicy

# Expected:
dnsPolicy: None
dnsConfig:
  nameservers:
  - 10.0.0.2
```

### 2. Check Container resolv.conf

```bash
# Check DinD container
kubectl exec -n gitlab <job-pod> -c docker -- cat /etc/resolv.conf

# Should show:
nameserver 10.0.0.2
```

### 3. Test DNS Server Reachability

```bash
# Test from DinD container
kubectl exec -n gitlab <job-pod> -c docker -- nc -zv 10.0.0.2 53

# Expected:
10.0.0.2 (10.0.0.2:53) open
```

### 4. Test DNS Resolution

```bash
# Test resolving the problematic domain
kubectl exec -n gitlab <job-pod> -c docker -- nslookup registry.example.com

# Expected:
Server:    10.0.0.2
Name:      registry.example.com
Address:   <some-ip>
```

### 5. Test HTTP Connectivity

```bash
# Test HTTPS connection
kubectl exec -n gitlab <job-pod> -c docker -- wget -O- https://registry.example.com

# Should return content (or authentication error, which means connection works)
```

### 6. Verify Runner Configuration

```bash
# Check if DNS config is in Runner's config.toml
kubectl exec -n gitlab <runner-manager-pod> -- \
  cat /home/gitlab-runner/.gitlab-runner/config.toml | grep -A 10 dns_config

# Should show:
dns_policy = "none"
[runners.kubernetes.dns_config]
  nameservers = ["10.0.0.2", ...]
```

---

## Key Takeaways

**Network Layers**:
- Kubernetes has 5 network layers: VPC → Node → Service → Pod → Container → DinD
- Each layer has its own IP addressing scheme
- DNS resolution happens at the Pod layer

**DNS Configuration**:
- Default kube-dns may not resolve external domains
- Pod DNS config affects all containers in the Pod
- `--dns` parameter only affects temporary build containers
- VPC DNS is often the best choice for private cloud environments

**Common Pitfalls**:
- Using `--dns` expecting it to fix dockerd DNS (it doesn't)
- Assuming all node-reachable IPs are Pod-reachable (they're not)
- Not testing DNS resolution before deploying

**Debugging Strategy**:
1. Check Pod DNS configuration
2. Verify container resolv.conf
3. Test DNS server reachability
4. Test DNS resolution
5. Test actual connectivity

---

## What's Next?

In Part 4, we solved the most challenging technical issue: DNS and networking. Now we're ready for the final part.

**Coming up in Part 5**:
- 🔒 **Security best practices** — RBAC, NetworkPolicy, secrets management
- ⚡ **Performance optimization** — Resource tuning, caching strategies
- 📊 **Monitoring and logging** — Prometheus metrics, log aggregation
- 🛠️ **Production readiness** — High availability, disaster recovery

**Preview**: We'll take everything we've learned and add the production-grade features needed for a reliable, secure, and scalable GitLab Runner deployment.

---

## Practice Exercises

**Exercise 1**: Diagnose DNS issues in your cluster

```bash
# Create a test Pod with default DNS
kubectl run dnstest-default --image=alpine --restart=Never -- \
  nslookup google.com

# Check if it works
kubectl logs dnstest-default

# Create a test Pod with custom DNS
kubectl run dnstest-custom --image=alpine --restart=Never \
  --overrides='{"spec":{"dnsPolicy":"None","dnsConfig":{"nameservers":["8.8.8.8"]}}}' -- \
  nslookup google.com

# Compare results
kubectl logs dnstest-custom
```

**Exercise 2**: Trace packet flow

```bash
# Install tcpdump in a running job pod
kubectl exec -n gitlab <job-pod> -c docker -- \
  apk add tcpdump

# Capture DNS queries
kubectl exec -n gitlab <job-pod> -c docker -- \
  tcpdump -i any -nn 'port 53' -c 10

# In another terminal, trigger DNS query
kubectl exec -n gitlab <job-pod> -c docker -- \
  nslookup registry.example.com
```

**Exercise 3**: Test different DNS configurations

```bash
# Test with kube-dns
# Test with public DNS (8.8.8.8)
# Test with VPC DNS
# Compare resolution times and success rates
```

---

## Resources

- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CoreDNS Configuration](https://coredns.io/manual/toc/)
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

---

**Part 5 coming soon** — the final piece: production best practices and security hardening.

If you found this helpful, feel free to share it with others debugging Kubernetes networking issues.

---

*This is Part 4 of a 5-part series on GitLab Runner on Kubernetes, documenting the learning and implementation process.*

**Series index**:
- [Part 1: Architecture & Quick Setup](link)
- [Part 2: Deep Dive into Container Architecture](link)
- [Part 3: Building a Real-World CI/CD Pipeline](link)
- **Part 4: Solving DNS and Network Issues** ← You are here
- Part 5: Production Best Practices (coming soon)

---

*Tags: #GitLab #Kubernetes #Networking #DNS #Troubleshooting #DevOps*
