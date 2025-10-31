# Kubernetes 上的 GitLab Runner：完整指南
## 第四部分 — 解决 DNS 和网络问题

---

## 关于本系列

本系列记录了我从零开始学习并部署 GitLab Runner on Kubernetes 的过程，从完全的初学者到生产环境部署。最独特的是，**我是与 Claude Code 一起学习并实现的** —— 一个 AI 编程助手帮助我：

- 理解复杂的 Kubernetes 网络概念
- 调试容器化环境中的 DNS 解析问题
- 理解 GitLab Runner 的四容器架构
- 构建生产就绪的配置

**为什么分享这个？** 因为 AI 辅助学习极大地加速了我的理解。那些通常需要数周通过试错才能掌握的概念，通过以下方式在几天内就变得清晰：
- 对错误信息的实时解释
- 网络问题的分步调试
- 容器交互的详细分解
- 架构可视化和时序图

如果你正在探索不熟悉的技术（就像我探索 GitLab on K8s 一样），本系列展示了 AI 如何成为强大的学习伙伴——不是给你复制粘贴的解决方案，而是帮助你在构建过程中**深入理解**。

**学习建议**：在阅读本系列时，你会遇到许多技术概念——Kubernetes 网络、Docker-in-Docker、DNS 解析、容器编排等。建议你在学习过程中使用 AI 助手（如 Claude、ChatGPT 等）来：
- 在上下文中解释不熟悉的术语和缩写
- 将复杂概念分解为易于理解的解释
- 生成符合你具体场景的示例
- 对架构决策提出"为什么"的问题
- 调试你在自己环境中遇到的错误

最好的学习发生在主动参与时。利用 AI 工具以你自己的节奏和深度探索概念。

> *回顾：[第一部分](link)（搭建），[第二部分](link)（架构），[第三部分](link)（CI/CD 流水线）。*

---

在第三部分中，我们构建了一个完整的 CI/CD 流水线。一切都很顺利……直到出现问题。当我尝试在流水线中构建 Docker 镜像时，遇到了这个错误：

```
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

这个错误让我深入探究了 Kubernetes 网络、DNS 解析和容器网络的深层机制。从一个简单的"为什么 DinD 无法拉取镜像？"开始，最终全面理解了网络数据包如何在 Kubernetes 环境中流经五个不同的层级。

在本部分中，我将分享：
- 五层 Kubernetes 网络模型
- 为什么 DNS 配置是最常见的陷阱
- 我的三次失败尝试及从中学到的经验
- 最终解决方案以及如何验证其工作

让我们从理解网络架构开始。

---

## 五层网络模型

当 DinD 尝试拉取镜像时，网络请求会经过**五个不同的网络层**。理解这一点对于调试至关重要。

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

让我们逐层检查。

### Layer 0: VPC 网络

这是你的数据中心或云提供商网络：

```
VPC: 10.0.0.0/16

Components:
- Kubernetes nodes: 192.168.1.100-103
- VPC DNS: 10.0.0.2
- NAT Gateway (for internet access)
- Internal DNS servers: 10.1.0.251, 10.1.0.252
```

**关键点**：并非 VPC 中的所有 IP 都能从 Pod 访问。这在后面会变得重要。

### Layer 1: 节点网络

每个 Kubernetes 节点都有一个物理（或虚拟机）网络接口：

```
Master:  192.168.1.100
Worker1: 192.168.1.101
Worker2: 192.168.1.102
Worker3: 192.168.1.103
```

节点在这一层直接通信。当不同节点上的 Pod 通信时，流量流向为：Pod → Node → Network → Node → Pod。

### Layer 2: Service 网络

Kubernetes Service 获得虚拟 IP（ClusterIP）：

```
kube-dns Service:    10.96.0.10
gitlab Service:      172.21.5.10
calculator Service:  172.21.8.20
```

**重要**：这些 IP 不存在于任何网络接口上。它们通过 kube-proxy 使用 iptables 规则实现。

### Layer 3: Pod 网络

每个 Pod 从 CNI 插件（例如 Flannel、Calico）获得一个 IP：

```
Node1 Pods: 172.24.1.0/24
Node2 Pods: 172.24.2.0/24
Node3 Pods: 172.24.3.0/24

Example Pod: 172.24.1.100
```

Pod 中的所有容器共享这个 IP 地址。

### Layer 4: 容器网络

Pod 内的所有容器共享**相同的网络命名空间**：

```
Job Pod Network Namespace:
- IP: 172.24.1.100
- Interfaces: lo (127.0.0.1), eth0 (172.24.1.100)
- Containers: helper, build, dind (all see same network)
```

这就是为什么 `docker:2375` 解析为 `127.0.0.1:2375` —— 相同的命名空间！

### Layer 5: DinD 内部网络

在 DinD 容器内部，dockerd 创建**另一个虚拟网络**：

```
DinD docker0 bridge: 172.17.0.1
Temporary containers: 172.17.0.2, 172.17.0.3, ...
```

当 DinD 在 Dockerfile 中执行 `RUN` 命令时，它会在这个网络中创建临时容器。

---

## DNS 解析问题

现在我们理解了这些层级，让我们看看 DNS 在哪里起作用。

### 默认 Kubernetes DNS 配置

默认情况下，Pod 使用 kube-dns 进行 DNS 解析：

```yaml
# Automatic Pod configuration
dnsPolicy: ClusterFirst

# Results in:
# /etc/resolv.conf
nameserver 10.96.0.10  # kube-dns
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**这适用于**：
- Kubernetes services: `gitlab.gitlab.svc.cluster.local` ✅
- Internal services: `database.default.svc.cluster.local` ✅

**这会失败**：
- External domains: `registry.example.com` ❌
- Private registry domains: `private-registry.company.com` ❌

### 为什么会失败？

kube-dns (CoreDNS) 设计用于解析 Kubernetes 服务。要解析外部域名，它需要将请求转发到上游 DNS 服务器。

**检查你的 CoreDNS 配置**：
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

如果你没有看到像这样的 `forward` 指令：
```
forward . 8.8.8.8 1.1.1.1
```

那么 kube-dns **无法解析外部域名**。

---

## 我的调试历程：三次失败的尝试

让我带你了解我实际的调试过程，包括失败经历。

### 尝试 1：使用默认 DNS ❌

**配置**：无自定义 DNS 设置

**结果**：
```
$ docker build -t myapp .
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

**诊断**：
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

**根本原因**：kube-dns 没有配置上游 DNS。

**学到的教训**：在没有正确配置 CoreDNS 的内部网络中，默认 DNS 无法工作。

### 尝试 2：使用 --dns 参数 ❌

我想："直接告诉 DinD 使用不同的 DNS 服务器！"

**配置**：
```yaml
services:
  - name: docker:dind
    command:
      - "--dns=10.1.0.251"  # Node DNS server
      - "--dns=10.1.0.252"
```

**预期**：DinD 将使用这些 DNS 服务器来解析域名。

**实际结果**：仍然失败！

```
$ docker build -t myapp .
#2 ERROR: failed to do request: Head "https://registry.example.com/...":
dial tcp: lookup registry.example.com: i/o timeout
```

**诊断**：
```bash
# Check what --dns actually affects
kubectl exec -n gitlab <job-pod> -c docker -- cat /etc/resolv.conf

# Output:
nameserver 10.96.0.10  # Still kube-dns!
```

**根本原因**：`--dns` 参数**不影响 dockerd 本身**。它只影响 dockerd **创建**的容器（如 `RUN` 命令期间的临时容器）的 `/etc/resolv.conf`。

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

**学到的教训**：`--dns` 参数用于**临时构建容器**，而不是 dockerd 自己的 DNS 解析。

### 尝试 3：直接使用节点 DNS ❌

下一次尝试："配置 Pod DNS 使用节点的 DNS 服务器。"

**配置**：
```yaml
# gitlab-runner-values.yaml
[runners.kubernetes]
  dns_policy = "none"
  [runners.kubernetes.dns_config]
    nameservers = ["10.1.0.251", "10.1.0.252"]
```

**预期**：Pod 将使用内部 DNS 服务器。

**实际结果**：DinD 容器启动失败！

```
$ kubectl logs <job-pod> -c docker

time="..." level=error msg="failed to start daemon"
error="failed to initialize DNS resolver: dial tcp 10.1.0.251:53: i/o timeout"
```

**诊断**：
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

**根本原因**：位于 `10.1.0.251/252` 的 DNS 服务器在**不同的子网**中，Pod 无法访问。

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

**学到的教训**：Pod 网络（overlay）!= 节点网络（underlay）。并非所有节点可访问的 IP 都是 Pod 可访问的。

---

## 解决方案：VPC DNS

经过三次失败后，我更仔细地检查了节点的 DNS 配置：

```bash
# On a Kubernetes node
cat /etc/resolv.conf

nameserver 10.1.0.251  # Internal DNS 1
nameserver 10.1.0.252  # Internal DNS 2
nameserver 10.0.0.2    # VPC DNS ← What's this?
```

第三个 DNS 服务器引起了我的注意：`10.0.0.2`。这是 **VPC 级别的 DNS 服务**。

### 测试 VPC DNS

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

### 为什么 VPC DNS 有效

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

### 最终配置

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

### 部署和验证

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

### 最终流水线测试

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

**成功！** DinD 现在可以拉取镜像了。

---

## 理解两种 DNS 配置

这是最令人困惑的部分，让我澄清一下：

### Pod DNS vs. --dns 参数

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

### 对比表

| 方面 | Pod DNS 配置 | --dns 参数 |
|--------|---------------|-----------------|
| **配置位置** | `gitlab-runner-values.yaml` | `.gitlab-ci.yml` services |
| **影响** | DinD 容器的 `/etc/resolv.conf` | 临时容器的 `/etc/resolv.conf` |
| **应用时机** | 容器启动 | dockerd 创建容器时 |
| **影响 dockerd** | ✅ 是（dockerd 读取自己的 resolv.conf） | ❌ 否 |
| **使用场景** | 拉取基础镜像 (FROM) | 安装包 (RUN) |
| **示例** | `FROM python:3.9` 的 DNS | `RUN pip install` 的 DNS |

### 推荐配置

**对于大多数情况**（包括我们的场景）：

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

**为什么？**
- Pod DNS 同时处理 dockerd 拉取镜像和临时容器 DNS
- 临时容器默认继承 DinD 的 DNS
- 配置更简单

**何时使用 --dns**：

仅当临时容器需要与 dockerd **不同**的 DNS 时：

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

## 数据包流分析

让我们追踪 DinD 拉取镜像时发生了什么：

### 逐步流程

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

## 故障排查清单

当你遇到 DNS 问题时，遵循这个清单：

### 1. 检查 Pod DNS 配置

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

### 2. 检查容器 resolv.conf

```bash
# Check DinD container
kubectl exec -n gitlab <job-pod> -c docker -- cat /etc/resolv.conf

# Should show:
nameserver 10.0.0.2
```

### 3. 测试 DNS 服务器可达性

```bash
# Test from DinD container
kubectl exec -n gitlab <job-pod> -c docker -- nc -zv 10.0.0.2 53

# Expected:
10.0.0.2 (10.0.0.2:53) open
```

### 4. 测试 DNS 解析

```bash
# Test resolving the problematic domain
kubectl exec -n gitlab <job-pod> -c docker -- nslookup registry.example.com

# Expected:
Server:    10.0.0.2
Name:      registry.example.com
Address:   <some-ip>
```

### 5. 测试 HTTP 连接性

```bash
# Test HTTPS connection
kubectl exec -n gitlab <job-pod> -c docker -- wget -O- https://registry.example.com

# Should return content (or authentication error, which means connection works)
```

### 6. 验证 Runner 配置

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

## 关键要点

**网络层级**：
- Kubernetes 有 5 个网络层：VPC → Node → Service → Pod → Container → DinD
- 每一层都有自己的 IP 地址方案
- DNS 解析发生在 Pod 层

**DNS 配置**：
- 默认的 kube-dns 可能无法解析外部域名
- Pod DNS 配置影响 Pod 中的所有容器
- `--dns` 参数仅影响临时构建容器
- VPC DNS 通常是私有云环境的最佳选择

**常见陷阱**：
- 使用 `--dns` 期望它修复 dockerd DNS（并不会）
- 假设所有节点可访问的 IP 都是 Pod 可访问的（并非如此）
- 部署前不测试 DNS 解析

**调试策略**：
1. 检查 Pod DNS 配置
2. 验证容器 resolv.conf
3. 测试 DNS 服务器可达性
4. 测试 DNS 解析
5. 测试实际连接性

---

## 接下来是什么？

在第四部分中，我们解决了最具挑战性的技术问题：DNS 和网络。现在我们准备进入最后一部分。

**第五部分即将推出**：
- 🔒 **安全最佳实践** — RBAC、NetworkPolicy、密钥管理
- ⚡ **性能优化** — 资源调优、缓存策略
- 📊 **监控和日志** — Prometheus 指标、日志聚合
- 🛠️ **生产就绪** — 高可用性、灾难恢复

**预览**：我们将采用我们学到的所有内容，并添加可靠、安全和可扩展的 GitLab Runner 部署所需的生产级功能。

---

## 实践练习

**练习 1**：诊断集群中的 DNS 问题

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

**练习 2**：追踪数据包流

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

**练习 3**：测试不同的 DNS 配置

```bash
# Test with kube-dns
# Test with public DNS (8.8.8.8)
# Test with VPC DNS
# Compare resolution times and success rates
```

---

## 资源

- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CoreDNS Configuration](https://coredns.io/manual/toc/)
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

---

**第五部分即将推出** — 最后一部分：生产最佳实践和安全加固。

如果你觉得这有帮助，欢迎与其他调试 Kubernetes 网络问题的人分享。

---

*这是关于 Kubernetes 上的 GitLab Runner 的5部分系列的第4部分，记录了学习和实施过程。*

**系列索引**：
- [第1部分：架构与快速搭建](link)
- [第2部分：深入探讨容器架构](link)
- [第3部分：构建真实世界的 CI/CD 流水线](link)
- **第4部分：解决 DNS 和网络问题** ← 你在这里
- 第5部分：生产最佳实践（即将推出）

---

*标签：#GitLab #Kubernetes #Networking #DNS #Troubleshooting #DevOps*
