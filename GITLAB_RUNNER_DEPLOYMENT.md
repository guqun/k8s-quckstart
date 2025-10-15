# GitLab Runner on Kubernetes 部署指南

## 一、环境概述

### 1.1 架构说明

本环境采用**混合部署架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                    VPC: 10.0.0.0/16                        │
│                                                              │
│  ┌──────────────────────┐        ┌────────────────────────┐ │
│  │   ECS 虚拟机         │        │  Kubernetes 集群        │ │
│  │                      │        │                        │ │
│  │  极狐 GitLab         │◄──────►│  GitLab Runner Manager │ │
│  │  v18.3.2-jh          │  内网  │  (Helm Chart)          │ │
│  │                      │  通信  │                        │ │
│  │  IP: 192.168.1.50     │        │  - Runner Manager Pods │ │
│  │  Port: 80            │        │  - Job Pods (动态创建)  │ │
│  └──────────────────────┘        └────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**关键特点**：
- ✅ GitLab Server 运行在 **ECS 虚拟机**上（传统部署，非容器化）
- ✅ GitLab Runner 运行在 **独立的 K8s 集群**中（容器化部署）
- ✅ 两者通过 **内网 VPC** 通信（无需公网）
- ✅ Job Pod 在 K8s 中**动态创建和销毁**

### 1.2 网络信息

| 组件 | 位置 | IP/CIDR | 说明 |
|------|------|---------|------|
| **GitLab Server** | ECS | 192.168.1.50 | 极狐 GitLab v18.3.2-jh |
| **K8s Master** | 物理/虚拟机 | 192.168.1.100 | Control Plane |
| **K8s Worker 1** | 物理/虚拟机 | 192.168.1.101 | Worker Node |
| **K8s Worker 2** | 物理/虚拟机 | 192.168.1.102 | Worker Node |
| **K8s Worker 3** | 物理/虚拟机 | 192.168.1.103 | Worker Node |
| **Pod Network** | K8s | 172.24.0.0/16 | Pod CIDR |
| **Service Network** | K8s | 172.21.0.0/16 | Service CIDR |
| **VPC Network** | 阿里云 | 10.0.0.0/16 | 整体 VPC |
| **内网 DNS** | 阿里云 | 10.0.0.2 | 主 DNS 服务器 |
| **备用 DNS 1** | - | 10.1.0.251 | 备用 DNS |
| **备用 DNS 2** | - | 10.1.0.252 | 备用 DNS |

### 1.3 镜像仓库

| 类型 | 地址 | 用途 |
|------|------|------|
| **公共加速镜像** | registry.example.com | 华为云镜像加速<br>（拉取 GitLab Runner、Docker 等） |
| **私有镜像仓库** | private-registry.example.com | 存储项目构建的镜像 |

---

## 二、前置条件检查

### 2.1 GitLab Server 检查

```bash
# 1. 检查 GitLab 可访问性
curl http://192.168.1.50/

# 预期输出: 返回 GitLab 登录页面 HTML

# 2. 登录 GitLab Web 界面
# 浏览器访问: http://192.168.1.50/
```

### 2.2 创建 Runner 并获取 Token

**操作步骤**：

1. 登录 GitLab: `http://192.168.1.50/`
2. 点击右上角头像 → **Admin Area**
3. 左侧菜单 → **CI/CD** → **Runners**
4. 点击 **New instance runner** 按钮
5. 填写配置：
   - **Tags**: `kubernetes,docker,x86_64`
   - **Run untagged jobs**: ✅ 勾选
6. 点击 **Create runner**
7. **复制显示的 Registration token**（格式：`glrt-xxxxxxxxxxxx`）

⚠️ **重要**：保存好这个 token，后续配置需要使用！

### 2.3 Kubernetes 集群检查

```bash
# 1. 检查集群连接
kubectl cluster-info

# 预期输出:
# Kubernetes control plane is running at https://192.168.1.100:6443

# 2. 检查节点状态
kubectl get nodes

# 预期输出:
# NAME            STATUS   ROLES           AGE   VERSION
# k8s-master      Ready    control-plane   ...   v1.28.x
# k8s-worker-1    Ready    <none>          ...   v1.28.x
# k8s-worker-2    Ready    <none>          ...   v1.28.x
# k8s-worker-3    Ready    <none>          ...   v1.28.x

# 3. 检查 kube-dns 服务
kubectl get svc -n kube-system kube-dns

# 4. 检查节点资源
kubectl top nodes
```

### 2.4 网络连通性检查

```bash
# 1. 从 K8s 节点测试访问 GitLab
# SSH 登录任意 Worker 节点
ssh user@192.168.1.101

# 测试连通性
curl -v http://192.168.1.50/

# 预期输出: 能够访问 GitLab 页面

# 2. 测试 DNS 解析（使用内网 DNS）
nslookup registry.example.com 10.0.0.2

# 预期输出: 能够解析到 IP 地址

# 3. 测试镜像拉取
docker pull registry.example.com/library/alpine:latest

# 预期输出: 镜像拉取成功
```

---

## 三、安装 GitLab Runner

### 3.1 添加 Helm 仓库

```bash
# 1. 添加 GitLab 官方 Helm 仓库
helm repo add gitlab https://charts.gitlab.io

# 2. 更新仓库索引
helm repo update

# 3. 查看可用的 Chart 版本
helm search repo gitlab/gitlab-runner --versions | head -10

# 4. 查看 Chart 详细信息
helm show chart gitlab/gitlab-runner --version 0.80.0
helm show values gitlab/gitlab-runner --version 0.80.0 | less
```

### 3.2 创建命名空间

```bash
# 创建 gitlab 命名空间
kubectl create namespace gitlab

# 验证命名空间
kubectl get namespace gitlab

# 设置默认命名空间（可选）
kubectl config set-context --current --namespace=gitlab
```

### 3.3 准备配置文件

配置文件已在当前目录：**`gitlab-runner-values.yaml`**

**需要修改的关键配置**：

```yaml
# 找到这一行，替换为你在 2.2 步骤中获取的 token
runnerToken: "YOUR_RUNNER_TOKEN_HERE"  # ⚠️ 必须替换！
```

**编辑配置文件**：

```bash
# 使用你喜欢的编辑器
vim gitlab-runner-values.yaml

# 或
nano gitlab-runner-values.yaml

# 将第 15 行的 YOUR_RUNNER_TOKEN_HERE 替换为实际的 token
# 例如: runnerToken: "glrt-zaXRuZy_O8ffdzpqT1B5"
```

**配置文件关键部分说明**：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `gitlabUrl` | `http://192.168.1.50/` | ECS 上的 GitLab 地址 |
| `runnerToken` | `glrt-xxx...` | 从 GitLab 获取的注册 token |
| `image.registry` | `swr.cn-north-4...` | 华为云镜像加速 |
| `replicas` | `3` | Runner Manager 副本数 |
| `concurrent` | `10` | 每个 Manager 最大并发数 |
| `dns_policy` | `"none"` | **关键！** 禁用默认 DNS |
| `nameservers` | `["10.0.0.2", ...]` | **关键！** 使用内网 DNS |

### 3.4 执行安装

```bash
# 执行 Helm 安装
helm install gitlab-runner \
  --namespace gitlab \
  --version 0.80.0 \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# 预期输出:
# NAME: gitlab-runner
# LAST DEPLOYED: ...
# NAMESPACE: gitlab
# STATUS: deployed
# REVISION: 1
```

### 3.5 等待部署完成

```bash
# 监控部署状态
kubectl rollout status deployment/gitlab-runner -n gitlab

# 预期输出:
# Waiting for deployment "gitlab-runner" rollout to finish: 0 of 3 updated replicas are available...
# Waiting for deployment "gitlab-runner" rollout to finish: 1 of 3 updated replicas are available...
# Waiting for deployment "gitlab-runner" rollout to finish: 2 of 3 updated replicas are available...
# deployment "gitlab-runner" successfully rolled out

# 查看 Pod 状态
kubectl get pods -n gitlab -l app=gitlab-runner -w
```

---

## 四、验证部署

### 4.1 检查 Runner Manager Pod

```bash
# 1. 查看 Pod 列表
kubectl get pods -n gitlab -l app=gitlab-runner

# 预期输出（3 个副本）:
# NAME                             READY   STATUS    RESTARTS   AGE
# gitlab-runner-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# gitlab-runner-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# gitlab-runner-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

# 2. 查看 Pod 详细信息
kubectl describe pod -n gitlab -l app=gitlab-runner | less

# 3. 查看 Runner 日志
kubectl logs -n gitlab -l app=gitlab-runner --tail=50 -f

# 预期关键日志:
# Configuration loaded                                builds=0
# Checking for jobs... received                       job=xxxxx
# Runner registered successfully. Feel free to start it
```

### 4.2 在 GitLab 中验证

**操作步骤**：

1. 浏览器访问: `http://192.168.1.50/`
2. 登录后点击右上角头像 → **Admin Area**
3. 左侧菜单 → **CI/CD** → **Runners**

**预期结果**：

应该看到 **3 个在线的 Runner**：
- 状态：🟢 **online** (绿色圆点)
- Tags: `kubernetes, docker, x86_64`
- 描述：包含版本号 `v18.3.0`
- IP 地址：来自 K8s Worker 节点

### 4.3 测试 CI/CD Pipeline

**创建测试项目**：

1. GitLab 首页 → **New project** → **Create blank project**
2. 项目名称：`runner-test`
3. 创建项目

**添加 `.gitlab-ci.yml`**：

在项目根目录创建文件 `.gitlab-ci.yml`，内容如下：

```yaml
stages:
  - test

test-runner:
  stage: test
  tags:
    - kubernetes
  image: alpine:latest
  script:
    - echo "=== GitLab Runner 测试 ==="
    - echo "主机名: $(hostname)"
    - echo "运行用户: $(whoami)"
    - echo "当前目录: $(pwd)"
    - echo ""
    - echo "=== DNS 配置 ==="
    - cat /etc/resolv.conf
    - echo ""
    - echo "=== 网络测试 ==="
    - ping -c 3 192.168.1.50 || echo "无法 ping GitLab Server"
    - echo ""
    - echo "=== 测试成功 ==="
```

**运行 Pipeline**：

1. 提交代码后自动触发 Pipeline
2. 或手动触发：**CI/CD** → **Pipelines** → **Run pipeline**

**观察 Job Pod 创建**：

```bash
# 新开终端，实时查看 Pod 创建
kubectl get pods -n gitlab -w

# 你会看到类似以下的 Pod 被创建:
# runner-xxxxx-project-x-concurrent-0-xxxxx   0/3     Pending   0          0s
# runner-xxxxx-project-x-concurrent-0-xxxxx   0/3     Init:0/1  0          2s
# runner-xxxxx-project-x-concurrent-0-xxxxx   0/3     Init:0/1  0          5s
# runner-xxxxx-project-x-concurrent-0-xxxxx   0/3     PodInitializing   0    8s
# runner-xxxxx-project-x-concurrent-0-xxxxx   3/3     Running           0    10s
# runner-xxxxx-project-x-concurrent-0-xxxxx   2/3     NotReady          0    35s
# runner-xxxxx-project-x-concurrent-0-xxxxx   0/3     Completed         0    40s
```

**查看 Job 日志**：

在 GitLab UI 中查看，或使用 kubectl：

```bash
# 找到 Job Pod（在 Running 状态时）
kubectl get pods -n gitlab | grep runner-

# 查看 build 容器日志
kubectl logs -n gitlab <job-pod-name> -c build

# 预期输出:
# === GitLab Runner 测试 ===
# 主机名: runner-xxxxx-project-x-concurrent-0-xxxxx
# 运行用户: root
# 当前目录: /builds/root/runner-test
#
# === DNS 配置 ===
# nameserver 10.0.0.2
# nameserver 10.1.0.251
# ...
# === 测试成功 ===
```

---

## 五、Docker-in-Docker (DinD) 测试

### 5.1 创建 DinD 测试 Pipeline

在项目中创建新文件 `.gitlab-ci.yml`（或替换原有内容）：

```yaml
stages:
  - build

docker-build-test:
  stage: build
  tags:
    - kubernetes
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

  before_script:
    - echo "等待 Docker 守护进程启动..."
    - |
      timeout=60
      until docker info >/dev/null 2>&1; do
        if [ $timeout -le 0 ]; then
          echo "❌ Docker 守护进程启动超时"
          exit 1
        fi
        echo "等待中... (剩余 ${timeout}s)"
        sleep 2
        timeout=$((timeout - 2))
      done
    - echo "✅ Docker 守护进程已就绪"
    - docker version

  script:
    - echo "=== 测试 Docker 功能 ==="
    - docker info
    - echo ""
    - echo "=== 拉取测试镜像 ==="
    - docker pull alpine:latest
    - echo ""
    - echo "=== 运行测试容器 ==="
    - docker run --rm alpine:latest echo "Hello from Docker in Kubernetes!"
    - echo ""
    - echo "=== DinD 测试成功 ==="
```

### 5.2 观察 DinD Job Pod

```bash
# 查看 Job Pod（应该有 3 个容器）
kubectl get pod <job-pod-name> -n gitlab

# 预期输出:
# NAME                                      READY   STATUS    RESTARTS   AGE
# runner-xxxxx-project-x-concurrent-0-xxx   3/3     Running   0          30s

# 查看 Pod 内的容器
kubectl describe pod <job-pod-name> -n gitlab | grep "Container ID"

# 应该看到:
# - helper (init container)
# - build (主容器)
# - docker (DinD service 容器)
```

### 5.3 验证 DNS 配置

```bash
# 查看 DinD 容器的 DNS 配置
kubectl exec -n gitlab <job-pod-name> -c docker -- cat /etc/resolv.conf

# 预期输出:
# nameserver 10.0.0.2
# nameserver 10.1.0.251
# nameserver 10.1.0.252
# search gitlab.svc.cluster.local svc.cluster.local cluster.local
# options ndots:2

# 测试 DNS 解析
kubectl exec -n gitlab <job-pod-name> -c docker -- \
  nslookup registry.example.com

# 预期输出: 能够成功解析
```

---

## 六、常见问题排查

### 6.1 Runner 未注册成功

**现象**：
```
ERROR: Registering runner... failed
error=couldn't execute POST against http://192.168.1.50/api/v4/runners:
dial tcp 192.168.1.50:80: i/o timeout
```

**原因**：网络连通性问题

**排查步骤**：

```bash
# 1. 从 Runner Pod 内测试网络
kubectl exec -n gitlab $(kubectl get pod -n gitlab -l app=gitlab-runner -o jsonpath='{.items[0].metadata.name}') \
  -- curl -v http://192.168.1.50/

# 2. 检查 GitLab URL 配置
helm get values gitlab-runner -n gitlab | grep gitlabUrl

# 3. 检查安全组/防火墙
# 确保 K8s 节点（192.168.1.101/110/111）可以访问 ECS（192.168.1.50）的 80 端口

# 4. 在 K8s 节点上测试
ssh user@192.168.1.101
curl -v http://192.168.1.50/
```

**解决方案**：
- 检查 ECS 安全组规则，允许来自 K8s 节点的访问
- 检查 VPC 路由表配置
- 检查 K8s 节点防火墙规则

### 6.2 Job Pod 拉取镜像失败

**现象**：
```
ERROR: Job failed: image pull failed:
dial tcp: lookup registry.example.com: i/o timeout
```

**原因**：DNS 配置未生效

**排查步骤**：

```bash
# 1. 检查 Job Pod 的 DNS 配置
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 10 dnsPolicy

# 预期看到:
# dnsPolicy: None
# dnsConfig:
#   nameservers:
#   - 10.0.0.2

# 2. 检查 Runner 配置
helm get values gitlab-runner -n gitlab | grep -A 5 dns_policy

# 3. 查看 Runner Manager 配置文件
kubectl exec -n gitlab <runner-manager-pod> -- \
  cat /home/gitlab-runner/.gitlab-runner/config.toml | grep -A 10 dns_config
```

**解决方案**：

```bash
# 如果 DNS 配置未生效，重新部署
helm upgrade gitlab-runner \
  --namespace gitlab \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# 强制重启 Runner Manager
kubectl rollout restart deployment/gitlab-runner -n gitlab
```

### 6.3 DinD 容器无法启动

**现象**：
```
Cannot connect to the Docker daemon at tcp://docker:2375
```

**排查步骤**：

```bash
# 1. 检查 Job Pod 容器状态
kubectl get pod <job-pod> -n gitlab

# 应该有 3 个容器都 Ready

# 2. 查看 DinD 容器日志
kubectl logs <job-pod> -n gitlab -c docker

# 3. 检查特权模式
kubectl get pod <job-pod> -n gitlab -o yaml | grep privileged

# 应该看到: privileged: true

# 4. 检查 Runner 配置
helm get values gitlab-runner -n gitlab | grep privileged
```

**解决方案**：

确保 `gitlab-runner-values.yaml` 中包含：

```yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        privileged = true
```

### 6.4 私有镜像仓库认证失败

**现象**：
```
Error response from daemon: Get "https://cr-ee.registry.cn-hangzhou-zjy-d01.../":
unauthorized: authentication required
```

**解决方案**：

在 `.gitlab-ci.yml` 中添加 `before_script`：

```yaml
before_script:
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME \
      --password-stdin private-registry.example.com
```

在 GitLab 项目中添加 CI/CD 变量：

1. **Settings** → **CI/CD** → **Variables** → **Add variable**
2. 添加以下变量：
   - Key: `ACR_USERNAME`, Value: `你的用户名`, Protected: ✅, Masked: ❌
   - Key: `ACR_PASSWORD`, Value: `你的密码`, Protected: ✅, Masked: ✅

### 6.5 Job Pod 资源不足

**现象**：
```
0/3 nodes are available: 3 Insufficient memory.
```

**排查步骤**：

```bash
# 1. 查看节点资源使用
kubectl top nodes

# 2. 查看 Job Pod 资源请求
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 10 resources

# 3. 查看集群可用资源
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**解决方案**：

调整资源配额（编辑 `gitlab-runner-values.yaml`）：

```yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        # 降低资源请求
        cpu_request = "250m"
        memory_request = "256Mi"

        service_cpu_request = "250m"
        service_memory_request = "256Mi"
```

然后升级：

```bash
helm upgrade gitlab-runner \
  --namespace gitlab \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner
```

---

## 七、性能调优

### 7.1 调整并发数

**当前配置**：
- 3 个 Runner Manager 副本
- 每个副本最大 10 并发
- **总并发能力 = 3 × 10 = 30 个 job**

**优化方案**：

**方案 1：增加 Manager 副本数**

```yaml
# gitlab-runner-values.yaml
replicas: 5  # 5 × 10 = 50 并发
```

**方案 2：增加单个 Manager 并发数**

```yaml
# gitlab-runner-values.yaml
concurrent: 15  # 3 × 15 = 45 并发
```

**方案 3：组合调整**

```yaml
replicas: 4
concurrent: 12
# 总并发 = 4 × 12 = 48
```

### 7.2 调整资源配额

**场景 1：大型项目构建**

```yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        # Build Container
        cpu_limit = "4"
        cpu_request = "1"
        memory_limit = "4Gi"
        memory_request = "1Gi"

        # DinD Service
        service_cpu_limit = "4"
        service_cpu_request = "1"
        service_memory_limit = "4Gi"
        service_memory_request = "1Gi"
```

**场景 2：小型项目/测试**

```yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        cpu_limit = "1"
        cpu_request = "250m"
        memory_limit = "1Gi"
        memory_request = "256Mi"

        service_cpu_limit = "1"
        service_cpu_request = "250m"
        service_memory_limit = "1Gi"
        service_memory_request = "256Mi"
```

### 7.3 使用节点亲和性

**场景**：将 CI/CD 任务调度到特定节点

```yaml
# gitlab-runner-values.yaml

# 1. 给节点打标签
# kubectl label nodes k8s-worker-2 workload=ci
# kubectl label nodes k8s-worker-3 workload=ci

nodeSelector:
  workload: ci

# 或使用亲和性
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: workload
          operator: In
          values:
          - ci
```

### 7.4 启用镜像缓存（可选）

使用 S3 兼容存储缓存 Docker 层：

```yaml
runners:
  cache:
    type: s3
    s3:
      ServerAddress: s3.amazonaws.com
      BucketName: gitlab-runner-cache
      BucketLocation: us-east-1
      Insecure: false
      AccessKey: "YOUR_ACCESS_KEY"
      SecretKey: "YOUR_SECRET_KEY"
```

---

## 八、监控与日志

### 8.1 查看 Runner 状态

```bash
# 1. 查看 Runner Manager Pod
kubectl get pods -n gitlab -l app=gitlab-runner

# 2. 查看 Deployment 状态
kubectl get deployment gitlab-runner -n gitlab

# 3. 查看 Runner 事件
kubectl get events -n gitlab --sort-by='.lastTimestamp'

# 4. 查看资源使用
kubectl top pod -n gitlab -l app=gitlab-runner
```

### 8.2 查看 Job Pod 历史

```bash
# 查看最近的 Job Pod
kubectl get pods -n gitlab --sort-by='.metadata.creationTimestamp' | grep runner-

# 查看所有 Pod（包括已完成的）
kubectl get pods -n gitlab --show-all
```

### 8.3 导出 Runner 日志

```bash
# 导出所有 Runner Manager 日志
kubectl logs -n gitlab -l app=gitlab-runner --all-containers=true > runner-logs.txt

# 实时查看日志
kubectl logs -n gitlab -l app=gitlab-runner -f --tail=100
```

### 8.4 Prometheus 监控（可选）

Runner 默认暴露 metrics 端口（9252）：

```yaml
# gitlab-runner-values.yaml
metrics:
  enabled: true
  port: 9252
  portName: metrics
```

查看 metrics：

```bash
kubectl port-forward -n gitlab deployment/gitlab-runner 9252:9252
curl http://localhost:9252/metrics
```

---

## 九、升级与维护

### 9.1 升级 Runner 版本

```bash
# 1. 查看当前版本
helm list -n gitlab

# 2. 查看可用版本
helm search repo gitlab/gitlab-runner --versions

# 3. 更新配置文件中的镜像版本
vim gitlab-runner-values.yaml
# 修改 image.tag: v18.3.0 → v18.4.0

# 4. 执行升级
helm upgrade gitlab-runner \
  --namespace gitlab \
  --version 0.81.0 \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# 5. 验证升级
kubectl rollout status deployment/gitlab-runner -n gitlab
```

### 9.2 修改配置

```bash
# 1. 编辑配置文件
vim gitlab-runner-values.yaml

# 2. 应用新配置
helm upgrade gitlab-runner \
  --namespace gitlab \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# 3. 重启 Runner（可选）
kubectl rollout restart deployment/gitlab-runner -n gitlab
```

### 9.3 备份配置

```bash
# 备份当前配置
helm get values gitlab-runner -n gitlab > gitlab-runner-backup-$(date +%Y%m%d).yaml

# 备份整个 Release
helm get all gitlab-runner -n gitlab > gitlab-runner-full-backup-$(date +%Y%m%d).yaml
```

---

## 十、卸载

### 10.1 完全卸载

```bash
# 1. 卸载 Helm Release
helm uninstall gitlab-runner -n gitlab

# 2. 删除命名空间（会删除所有资源）
kubectl delete namespace gitlab

# 3. 在 GitLab 中删除 Runner
# 浏览器访问: http://192.168.1.50/admin/runners
# 找到对应的 Runner → 点击删除
```

### 10.2 保留配置的卸载

```bash
# 仅卸载 Helm Release，保留命名空间和 PVC
helm uninstall gitlab-runner -n gitlab

# 命名空间和其他资源保留
kubectl get all -n gitlab
```

---

## 十一、完整示例：构建并推送镜像

### 11.1 项目结构

```
my-app/
├── .gitlab-ci.yml
├── Dockerfile
├── app.py
└── requirements.txt
```

### 11.2 Dockerfile

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

### 11.3 .gitlab-ci.yml

```yaml
stages:
  - build

variables:
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

build-job:
  stage: build
  tags:
    - kubernetes

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

  before_script:
    - echo "等待 Docker 守护进程启动..."
    - |
      timeout=60
      until docker info >/dev/null 2>&1; do
        if [ $timeout -le 0 ]; then
          echo "❌ 等待超时"
          exit 1
        fi
        sleep 2
        timeout=$((timeout - 2))
      done
    - echo "✅ Docker 守护进程已就绪"
    - docker version

  script:
    - echo "=== 构建镜像 ==="
    - docker build -t $DOCKER_IMAGE .

    - echo "=== 登录私有仓库 ==="
    - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME \
        --password-stdin private-registry.example.com

    - echo "=== 推送镜像 ==="
    - docker push $DOCKER_IMAGE

    - echo "=== 构建完成 ==="
    - echo "镜像地址: $DOCKER_IMAGE"

  only:
    - main
```

### 11.4 配置 CI/CD 变量

在 GitLab 项目中：**Settings** → **CI/CD** → **Variables**

添加：
- `ACR_USERNAME`: 镜像仓库用户名
- `ACR_PASSWORD`: 镜像仓库密码（Masked）

### 11.5 运行 Pipeline

提交代码后自动触发，或手动触发：

```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

---

## 十二、架构优势总结

### ✅ 资源隔离
- GitLab Server 和 Runner 完全分离
- Server 故障不影响 Runner，反之亦然

### ✅ 弹性扩展
- Job Pod 动态创建和销毁
- 自动调度到可用节点
- 支持横向扩展（增加 Worker 节点）

### ✅ 成本优化
- Job Pod 按需创建，用完即销毁
- 不占用常驻资源
- 资源利用率高

### ✅ 高可用
- 3 个 Runner Manager 副本
- 单点故障自动恢复
- 支持滚动更新

### ✅ 安全性
- 内网 VPC 通信
- 无需暴露公网端口
- RBAC 权限控制

---

## 十三、参考资料

- [极狐 GitLab Runner 官方文档](https://docs.gitlab.cn/runner/)
- [GitLab Runner Helm Chart](https://docs.gitlab.com/runner/install/kubernetes.html)
- [Kubernetes Executor 配置](https://docs.gitlab.com/runner/executors/kubernetes.html)
- [Docker-in-Docker 最佳实践](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker)

---

## 十四、常用命令速查

```bash
# ===== 部署 =====
helm install gitlab-runner --namespace gitlab -f gitlab-runner-values.yaml gitlab/gitlab-runner

# ===== 查看状态 =====
kubectl get pods -n gitlab -l app=gitlab-runner
kubectl logs -n gitlab -l app=gitlab-runner -f

# ===== 升级 =====
helm upgrade gitlab-runner --namespace gitlab -f gitlab-runner-values.yaml gitlab/gitlab-runner

# ===== 调试 =====
kubectl describe pod <pod-name> -n gitlab
kubectl exec -n gitlab <pod-name> -- cat /etc/resolv.conf

# ===== 卸载 =====
helm uninstall gitlab-runner -n gitlab
kubectl delete namespace gitlab
```

---

**部署完成！** 🎉

现在您的 GitLab Runner 已成功部署在 Kubernetes 集群中，可以处理来自 ECS 上极狐 GitLab 的所有 CI/CD 任务。
