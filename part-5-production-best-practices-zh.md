# Kubernetes 上的 GitLab Runner：完整指南
## 第五部分 — 生产最佳实践与故障排除

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

**关于第五部分**：与前四部分记录我的实际学习和部署经验不同，这最后一部分完全由 AI 编写。它结合了通用的行业最佳实践和第1-4部分的知识，作为生产环境部署的综合参考指南。我希望它能成为我后续实际工作以及其他大规模部署 GitLab Runner 的人的有价值的参考资源。

> *这是最后一部分！快速回顾：[第1部分](link)（搭建），[第2部分](link)（架构），[第3部分](link)（流水线），[第4部分](link)（网络）。*

---

在前面四个部分中，我们构建了一个完整的 Kubernetes 上的 GitLab Runner。它能工作，但它准备好投入生产了吗？在这最后一部分中，我们将涵盖：

- 不同环境的配置策略
- 常见生产问题及解决方案
- 性能优化技术
- 监控和运维考虑
- 安全建议（一般最佳实践）

这一部分结合了**我实际部署的经验教训**（第1-4部分）和**大规模运行 CI/CD 基础设施的行业最佳实践**。

---

## DNS 配置策略

DNS 配置是我们涵盖的最关键方面（第4部分）。让我们回顾不同环境的策略。

### 策略 1：内部网络（我们的实现）

**环境**：私有数据中心或没有直接互联网访问的 VPC

**配置**：
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

**优势**：
- ✅ 适用于内部镜像仓库
- ✅ 解析私有域名
- ✅ 保持 Kubernetes 服务发现（搜索域）

**限制**：
- ⚠️ Kubernetes 服务 DNS 可能较慢（搜索域遍历）
- ⚠️ 需要 VPC DNS 能从 Pod 访问

### 策略 2：具有互联网访问的公有云

**环境**：公有云（AWS、GCP、Azure）具有直接互联网访问

**配置**：
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

**优势**：
- ✅ 快速的 Kubernetes 服务解析
- ✅ 适用于公共镜像仓库（Docker Hub 等）
- ✅ 配置简单

**限制**：
- ⚠️ 可能不适用于内部服务
- ⚠️ 依赖公共 DNS 可用性

### 策略 3：混合环境

**环境**：内部和外部服务的混合

**配置**：
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

**优势**：
- ✅ 涵盖所有场景
- ✅ 多个备用选项

**限制**：
- ⚠️ DNS 查询可能较慢（尝试多个服务器）
- ⚠️ 必须确保所有 DNS 服务器都可访问

---

## 资源管理和性能

### 容量规划

基于四容器架构（第2部分），计算您的集群容量：

**每个任务的资源消耗**：
```
Build 容器:       2 CPU, 2Gi 内存
DinD 服务:        2 CPU, 2Gi 内存
Helper 容器:      0.5 CPU, 512Mi 内存
-------------------------------------------
每个任务总计:     4.5 CPU, 4.5Gi 内存
```

**集群容量示例**：
```
3个工作节点 × 8核 × 32GB RAM 每个

可用于任务（假设80%可分配）：
- CPU: 3 × 8 × 0.8 = 19.2 核
- 内存: 3 × 32GB × 0.8 = 76.8GB

理论最大并发任务：
- 按 CPU: 19.2 ÷ 4.5 = 4 个任务
- 按内存: 76.8 ÷ 4.5 = 17 个任务
- 瓶颈: CPU → 约4个并发任务

实际容量（50%平均利用率）：
- 约8-10个并发任务舒适运行
```

### 调整资源限制

根据您的工作负载进行调整：

**对于小型项目（快速构建）**：
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

**对于大型项目（复杂构建）**：
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

### 调整并发性

**当前配置**（来自第1部分）：
```yaml
replicas: 3      # 3 个 Runner Manager pod
concurrent: 10   # 每个管理器10个任务
# 总容量: 3 × 10 = 30 个任务
```

**扩展策略**：

**水平扩展**（更多管理器）：
```yaml
replicas: 5
concurrent: 10
# 总计: 5 × 10 = 50 个任务
```

**垂直扩展**（每个管理器更多任务）：
```yaml
replicas: 3
concurrent: 15
# 总计: 3 × 15 = 45 个任务
```

**组合**：
```yaml
replicas: 4
concurrent: 12
# 总计: 4 × 12 = 48 个任务
```

### CI/CD 工作负载的节点亲和性

将特定节点专用于 CI/CD：

```bash
# 为 CI/CD 标记节点
kubectl label nodes worker-2 workload=ci
kubectl label nodes worker-3 workload=ci

# 污点以防止其他工作负载
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

## 常见生产问题

让我分享我遇到的问题以及如何解决它们。

### 问题 1：Runner 注册失败

**症状**：
```
ERROR: Registering runner... failed
error=couldn't execute POST against http://gitlab.example.com/api/v4/runners:
dial tcp: i/o timeout
```

**诊断步骤**：

1. **从 Pod 检查网络连接**：
```bash
kubectl run nettest --rm -i --image=alpine -n gitlab -- \
  wget -O- http://gitlab.example.com
```

2. **检查 Runner Manager 日志**：
```bash
kubectl logs -n gitlab -l app=gitlab-runner --tail=50
```

3. **验证配置中的 GitLab URL**：
```bash
helm get values gitlab-runner -n gitlab | grep gitlabUrl
```

**解决方案**：

**A. GitLab URL 不正确**：
```yaml
# 修复 values 文件中的 URL
gitlabUrl: "http://gitlab.example.com/"  # 添加尾部斜杠
```

**B. 网络策略阻塞**：
```bash
# 检查 NetworkPolicy 是否存在
kubectl get networkpolicy -n gitlab

# 临时允许所有出站流量进行测试
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

**C. 安全组/防火墙规则**：
- 确保 Kubernetes 节点可以访问 GitLab 服务器
- 检查云提供商安全组
- 验证本地防火墙规则

### 问题 2：任务 Pod 镜像拉取失败

**症状**：
```
Failed to pull image "registry.example.com/docker:latest":
rpc error: code = Unknown desc = failed to pull and unpack image:
failed to resolve reference: failed to do request:
Head "https://registry.example.com/...": dial tcp: lookup registry.example.com: i/o timeout
```

**诊断**（来自第4部分）：

1. **检查 Pod DNS**：
```bash
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 10 dnsPolicy
```

2. **测试 DNS 解析**：
```bash
kubectl exec -n gitlab <job-pod> -c build -- nslookup registry.example.com
```

3. **检查 DNS 服务器可达性**：
```bash
kubectl exec -n gitlab <job-pod> -c build -- nc -zv 10.0.0.2 53
```

**解决方案**：遵循第4部分的 DNS 配置策略。

### 问题 3：DinD 容器 OOM Killed

**症状**：
```
Error from server: container "docker" in pod "runner-xxx" is terminated (OOMKilled)
```

**诊断**：

1. **检查 Pod 事件**：
```bash
kubectl describe pod <job-pod> -n gitlab | grep -A 10 Events
```

2. **检查实际内存使用**（任务运行时）：
```bash
kubectl top pod <job-pod> -n gitlab --containers
```

**解决方案**：

**A. 增加内存限制**：
```yaml
# gitlab-runner-values.yaml
service_memory_limit = "4Gi"    # 从 2Gi 增加
service_memory_request = "2Gi"  # 从 1Gi 增加
```

**B. 优化 Docker 构建**：
```dockerfile
# 使用更小的基础镜像
FROM python:3.9-slim  # Good
# FROM python:3.9     # Too large

# 合并 RUN 命令
RUN apt-get update && \
    apt-get install -y pkg1 pkg2 && \
    rm -rf /var/lib/apt/lists/*  # Clean up

# 使用多阶段构建
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install

FROM node:18-slim
COPY --from=builder /app/node_modules ./node_modules
```

### 问题 4：任务 Pod 调度失败

**症状**：
```
0/3 nodes are available: 3 Insufficient memory.
```

**诊断**：

1. **检查节点资源**：
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

2. **检查待定的 Pod**：
```bash
kubectl get pods -n gitlab | grep Pending
kubectl describe pod <pending-pod> -n gitlab
```

**解决方案**：

**A. 减少资源请求**（允许更多任务）：
```yaml
cpu_request = "250m"      # 从 500m 降低
memory_request = "256Mi"  # 从 512Mi 降低
```

**B. 添加更多节点**或**扩展现有节点**

**C. 使用 Pod 优先级**（重要任务优先）：
```yaml
# gitlab-runner-values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        priority_class_name = "high-priority-ci"
```

创建 PriorityClass：
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-ci
value: 1000
globalDefault: false
description: "High priority for CI/CD jobs"
```

### 问题 5：镜像拉取缓慢

**症状**：任务仅拉取镜像就需要2-5分钟

**解决方案**：

**A. 明智地使用镜像拉取策略**：
```yaml
# .gitlab-ci.yml
build-job:
  image: docker:latest

variables:
  # Cache images on nodes
  DOCKER_PULL_POLICY: if-not-present
```

**B. 在所有节点上预拉取常用镜像**：
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

**C. 使用镜像仓库镜像/缓存**：
```yaml
# In DinD configuration
services:
  - name: docker:dind
    command:
      - "--registry-mirror=http://registry-cache.local:5000"
```

### 问题 6：任务卡在 Pending 状态

**症状**：任务 Pod 停留在 Pending 状态，永远不启动

**常见原因**：

1. **没有节点匹配 nodeSelector**：
```bash
# 检查节点标签
kubectl get nodes --show-labels

# 如果需要，移除限制性的 nodeSelector
```

2. **持久卷声明未绑定**（如果使用）：
```bash
kubectl get pvc -n gitlab
```

3. **镜像拉取回退**：
```bash
kubectl describe pod <pending-pod> -n gitlab | grep -i image
```

4. **准入 Webhook 阻塞**：
```bash
# 检查准入控制器日志
kubectl logs -n kube-system -l component=kube-apiserver
```

---

## 性能优化

### 减少 DNS 查询延迟

从我们的 DNS 配置（第4部分），优化查询速度：

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

### 优化任务 Pod 启动

**当前时间线**（来自第2部分）：
```
T+0s   Pod 创建
T+3s   Helper 克隆代码
T+8s   DinD 就绪
T+10s  任务开始
```

**优化**：

**A. 使用更小的 helper 镜像**：
```yaml
helper_image = "gitlab/gitlab-runner-helper:alpine-x86_64-v18.3.0"
# Instead of the full version
```

**B. 减少克隆深度**：
```yaml
variables:
  GIT_DEPTH: 1  # Shallow clone
  GIT_STRATEGY: fetch  # Reuse if possible
```

**C. 缓存依赖**：
```yaml
# .gitlab-ci.yml
build-job:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .pip-cache/
```

### 并发构建优化

对于同一项目上的多个任务：

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

## 监控和运维

### 基本健康检查

**检查 Runner Manager 健康**：
```bash
# 查看 Runner Manager pod
kubectl get pods -n gitlab -l app=gitlab-runner

# 检查错误日志
kubectl logs -n gitlab -l app=gitlab-runner --tail=100

# 监视任务 Pod 创建
kubectl get pods -n gitlab -w
```

**监控资源使用**：
```bash
# Runner Manager 资源
kubectl top pod -n gitlab -l app=gitlab-runner

# 任务 Pod 资源（运行时）
kubectl top pod -n gitlab | grep runner-.*concurrent
```

### Prometheus 指标

Runner 在9252端口暴露指标：

```yaml
# gitlab-runner-values.yaml
metrics:
  enabled: true
  port: 9252
  portName: metrics
```

**访问指标**：
```bash
# 端口转发以查看指标
kubectl port-forward -n gitlab deployment/gitlab-runner 9252:9252

# 查看指标
curl http://localhost:9252/metrics
```

**关键指标**：
```
# Number of concurrent jobs
gitlab_runner_jobs{state="running"}

# Job duration
gitlab_runner_job_duration_seconds

# Runner errors
gitlab_runner_errors_total
```

### 日志管理

**按组件查看日志**：

```bash
# Runner Manager 日志
kubectl logs -n gitlab -l app=gitlab-runner -f

# 特定任务日志
kubectl logs -n gitlab <job-pod> -c build -f
kubectl logs -n gitlab <job-pod> -c docker -f

# 任务 Pod 中的所有容器
kubectl logs -n gitlab <job-pod> --all-containers=true
```

**导出日志**以进行分析：
```bash
# 导出 Runner 日志
kubectl logs -n gitlab -l app=gitlab-runner \
  --since=24h > runner-logs-$(date +%Y%m%d).txt

# 导出所有任务 Pod 日志
for pod in $(kubectl get pods -n gitlab -o name | grep runner-); do
  kubectl logs -n gitlab $pod --all-containers=true > ${pod}-logs.txt
done
```

---

## 安全建议

> **注意**：以下是 Kubernetes CI/CD 环境的一般安全最佳实践。根据您组织的安全策略调整这些建议。

### 1. 网络策略

限制任务 Pod 的网络访问：

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

### 2. RBAC（基于角色的访问控制）

最小化 Runner ServiceAccount 的权限：

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

**对于部署任务**（需要集群访问）：

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

### 3. 密钥管理

**使用 GitLab CI/CD 变量**存储敏感数据：

```yaml
# In GitLab project: Settings → CI/CD → Variables
# Add:
- ACR_USERNAME (Protected: ✅, Masked: ❌)
- ACR_PASSWORD (Protected: ✅, Masked: ✅)
- KUBECONFIG (Protected: ✅, Masked: ❌, Type: File)
```

**永远不要硬编码密钥**：
```yaml
# ❌ Bad
script:
  - docker login -u admin -p mypassword registry.com

# ✅ Good
script:
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
```

**定期轮换密钥**：
- 更新 GitLab CI/CD 变量
- 更新 Kubernetes secrets
- 更新 imagePullSecrets

### 4. 镜像安全

**一般建议**：

- 使用特定的镜像版本，而不是 `latest`
- 扫描镜像漏洞（工具：Trivy、Clair）
- 使用最小基础镜像（alpine、distroless）
- 保持基础镜像更新

**安全 Dockerfile 示例**：
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

## 升级和维护

### 升级 GitLab Runner

```bash
# 检查当前版本
helm list -n gitlab

# 查看可用版本
helm search repo gitlab/gitlab-runner --versions | head -10

# 备份当前配置
helm get values gitlab-runner -n gitlab > backup-$(date +%Y%m%d).yaml

# 更新 chart 仓库
helm repo update

# 升级（使用相同的 values 文件）
helm upgrade gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab \
  --version 0.81.0 \
  -f gitlab-runner-values.yaml

# 验证升级
kubectl rollout status deployment/gitlab-runner -n gitlab
```

### 备份策略

**需要备份的内容**：

1. **Helm values 文件**：`gitlab-runner-values.yaml`
2. **Runner 注册令牌**：存储在 GitLab 中
3. **CI/CD 变量定义**：在版本控制中记录
4. **自定义脚本和配置**：存储在 Git 仓库中

**恢复流程**：

1. 使用备份的 values 文件重新部署 Runner
2. Runner 使用 values 中的令牌自动注册
3. 任务 Pod 将自动具有正确的配置

**无需备份状态**：Runner 是无状态的；所有状态都在 GitLab 服务器中。

---

## 生产就绪检查清单

在投入生产之前，验证：

**基础设施**：
- [ ] 集群有足够的资源用于预期的并发任务
- [ ] 节点规模合适（CPU、内存、磁盘）
- [ ] 网络策略已配置（如果需要）
- [ ] DNS 解析适用于所有必需的域

**Runner 配置**：
- [ ] DNS 配置已测试并正常工作（第4部分）
- [ ] 资源限制配置适当
- [ ] 多个 Runner Manager 副本实现高可用
- [ ] 并发任务限制设置正确
- [ ] 节点亲和性/容忍度已配置（如果需要）

**安全性**：
- [ ] RBAC 配置为最小权限
- [ ] 密钥存储在 GitLab 变量中，而不是代码中
- [ ] 网络策略限制不必要的访问
- [ ] 镜像拉取密钥配置正确
- [ ] DinD 仅在受信任的网络中使用 `--tls=false`

**监控**：
- [ ] Prometheus 指标已启用并被抓取
- [ ] 为 Runner 故障配置警报
- [ ] 日志聚合已就位
- [ ] 创建资源使用仪表板

**运维**：
- [ ] 升级程序已记录
- [ ] 备份策略已就位
- [ ] 创建故障排除手册
- [ ] 值班团队已培训

---

## 关键要点

**DNS 配置**：
- 根据环境选择策略（内部、公共、混合）
- 在部署前测试 DNS 解析
- VPC DNS 通常最适合私有云

**资源管理**：
- 基于每个任务4.5 CPU / 4.5GB 规划容量
- 监控实际使用情况并调整限制
- 为专用 CI/CD 节点使用节点亲和性

**常见问题**：
- 大多数问题与 DNS 或网络相关（第4部分）
- OOM kills 表示内存限制不足
- Pending pod 表示资源或调度约束

**生产运维**：
- 监控 Runner Manager 和任务 Pod
- 将指标导出到 Prometheus
- 实施日志聚合
- 维护配置备份

**安全性**（一般建议）：
- 最小化 RBAC 权限
- 使用 NetworkPolicies 限制访问
- 通过 GitLab 变量管理密钥
- 扫描镜像漏洞

---

## 结论

在这个五部分系列中，我们涵盖了：

1. **第1部分**：架构概述和初始设置
2. **第2部分**：深入探讨四容器模型
3. **第3部分**：构建完整的 CI/CD 流水线
4. **第4部分**：解决复杂的 DNS 和网络问题
5. **第5部分**：生产最佳实践和故障排除

**从学习到生产的旅程**：
- 从基本概念和快速部署开始
- 深入理解内部架构
- 构建真实世界的 CI/CD 工作流
- 调试复杂的网络问题
- 应用生产级实践

**学到的关键经验**：
- Kubernetes 网络有五个不同的层
- DNS 配置是最常见的陷阱
- Docker-in-Docker 是客户端-服务器，而不是魔法
- 资源规划可防止调度问题
- AI 辅助可以极大地加速学习

这个部署现在可靠地运行生产工作负载。通过调试获得的知识（尤其是第4部分的 DNS 问题）是非常宝贵的。

**最后的想法**：在 Kubernetes 上构建 GitLab Runner 是复杂的，但理解每一层使其变得可管理。AI 辅助学习方法让我能够系统地解决问题，而不是通过盲目的试错。

---

## 资源

**官方文档**：
- [GitLab Runner 文档](https://docs.gitlab.com/runner/)
- [Kubernetes 最佳实践](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Helm Chart 最佳实践](https://helm.sh/docs/chart_best_practices/)

**相关主题**：
- [Kubernetes 网络策略](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes 中的 RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [容器安全最佳实践](https://kubernetes.io/docs/concepts/security/)

---

## 感谢您

感谢您关注本系列！如果您觉得有帮助：
- 与其他学习 Kubernetes 上的 GitLab 的人分享
- 留下反馈，说明哪些有效或可以改进
- 如果有问题，请与我联系

本系列得益于 Claude Code 的 AI 辅助学习。如果您正在探索复杂技术，请考虑使用 AI 作为学习伙伴——它可以帮助您深入理解，而不仅仅是复制解决方案。

---

*这是关于 Kubernetes 上的 GitLab Runner 的5部分系列的第5部分（最终部分），记录了学习和实施过程。*

**系列索引**：
- [第1部分：架构与快速搭建](link)
- [第2部分：深入探讨容器架构](link)
- [第3部分：构建真实世界的 CI/CD 流水线](link)
- [第4部分：解决 DNS 和网络问题](link)
- **第5部分：生产最佳实践** ← 你在这里

---

*标签：#GitLab #Kubernetes #CICD #DevOps #ProductionReady #BestPractices*
