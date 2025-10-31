# Kubernetes 上的 GitLab Runner：完整指南
## 第一部分 — 架构与快速搭建

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

让我们开始吧。

---

## 面临的挑战

在使用 GitLab CI/CD 时，你可能会遇到传统 runner 设置的挑战：

- **资源竞争**：多个任务争夺相同的主机资源
- **磁盘空间管理**：构建产物随时间累积
- **扩展限制**：增加容量意味着要配置新的虚拟机
- **依赖冲突**：任务之间相互干扰彼此的环境

**Kubernetes Executor 提供了不同的方法**：将每个 CI/CD 任务视为按需创建并自动清理的隔离、可丢弃的 Pod。

在这个5部分系列中，我们将探索这种架构如何工作，并构建一个生产就绪的设置。在第一部分结束时，你将拥有一个正在执行你第一个流水线的 runner。

---

## 为什么选择 Kubernetes Executor？

让我们比较三种主要的 GitLab Runner executor：

### 传统方法

**Shell Executor**
```yaml
# 直接在主机上运行
runners:
  executor: shell
```
- ✅ 设置简单
- ❌ 任务之间没有隔离
- ❌ 依赖冲突（"在我的机器上可以运行"）
- ❌ 难以扩展

**Docker Executor**
```yaml
# 在 Docker 容器中运行任务
runners:
  executor: docker
```
- ✅ 任务隔离
- ✅ 每个任务都有干净的环境
- ❌ 每个 runner 主机都需要 Docker 守护进程
- ❌ 手动扩展（需要配置更多 runner 虚拟机）

### Kubernetes 方式

**Kubernetes Executor**
```yaml
# 为每个任务动态创建 Pod
runners:
  executor: kubernetes
```
- ✅ **弹性扩展**：按需创建 Pod
- ✅ **资源隔离**：每个任务的 CPU/内存限制
- ✅ **成本效率**：没有空闲的 runner 消耗资源
- ✅ **高可用性**：多个 runner 管理器实现冗余
- ✅ **自我修复**：失败的任务在健康节点上自动重试

---

## 架构概览

在深入设置之前，让我们了解我们正在构建什么。

### 全局视图

```
┌─────────────────────────────────────────────────────────┐
│                    你的基础设施                          │
│                                                          │
│  ┌──────────────────┐         ┌────────────────────┐   │
│  │  GitLab 服务器   │         │  Kubernetes 集群    │   │
│  │  (VM/容器)       │◄───────►│                     │   │
│  │                  │  API    │  ┌──────────────┐   │   │
│  │  - Web UI        │  调用   │  │ Runner       │   │   │
│  │  - Git 仓库      │         │  │ Manager Pods │   │   │
│  │  - CI/CD 配置    │         │  └──────┬───────┘   │   │
│  └──────────────────┘         │         │           │   │
│                                │         │ 创建      │   │
│                                │         ▼           │   │
│                                │  ┌──────────────┐   │   │
│                                │  │  任务 Pods   │   │   │
│                                │  │  (动态创建)  │   │   │
│                                │  └──────────────┘   │   │
│                                └────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**流程**：
1. 开发者推送代码到 GitLab
2. GitLab 触发流水线
3. Runner Manager（运行在 K8s 中）接收任务
4. Manager 创建一个专用的任务 Pod
5. 任务 Pod 执行 CI/CD 脚本
6. Pod 将结果报告回 GitLab
7. Pod 被删除（没有残留！）

---

## 四容器架构

这是 Kubernetes Executor 的亮点：**每个 CI/CD 任务在一个包含最多4个专用容器的 Pod 中运行**。

### 快速概览

| 容器 | 镜像 | 目的 | 生命周期 |
|------|------|------|----------|
| **Runner Manager** | `gitlab-runner:v18.3.0` | 轮询 GitLab 获取任务，创建 Pod | 持久化（Deployment） |
| **Helper** | `gitlab-runner-helper:v18.3.0` | 克隆代码，上传产物 | 按任务（Init Container） |
| **Build** | *用户定义*（如 `docker:latest`） | 执行 CI/CD 脚本 | 按任务 |
| **Service** | *用户定义*（如 `docker:dind`） | 提供后台服务（数据库、DinD） | 按任务 |

### 示例：Docker 构建任务

当你运行一个构建 Docker 镜像的任务时，会发生以下情况：

```yaml
# .gitlab-ci.yml
build-job:
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t myapp:latest .
```

**幕后发生的事**：

```
创建任务 Pod
│
├─ [Init] Helper 容器
│   └─ 克隆你的 Git 仓库
│   └─ 退出（任务继续）
│
├─ [Main] Build 容器 (docker:latest)
│   └─ 执行: docker build -t myapp:latest .
│   └─ 将命令发送到 → Service 容器
│
└─ [Service] DinD 容器 (docker:dind)
    └─ 运行 dockerd (Docker 守护进程)
    └─ 实际执行构建
    └─ 将结果返回给 Build 容器
```

**关键洞察**：脚本中的 `docker` 命令在 **Build 容器** 中运行，但实际的构建工作发生在 **DinD (Docker-in-Docker) Service 容器** 中。它们通过 TCP 在 `localhost:2375` 上通信。

> 💡 **注意**：这种分离提供了故障隔离——如果 Build 容器崩溃，DinD 守护进程会继续运行。你也可以在不影响服务层的情况下替换构建工具。

我们将在**第二部分**深入探讨每个容器。现在，只需理解**隔离 + 专业化 = 可靠性**。

---

## 快速搭建（15分钟）

让我们动手实践。我们将使用官方 Helm chart 在 Kubernetes 上部署 GitLab Runner。

### 前置条件

```bash
# 验证你有：
kubectl cluster-info  # Kubernetes 集群访问权限
helm version          # Helm 3.x 已安装
```

你还需要：
- 一个运行中的 GitLab 实例（自托管或 GitLab.com）
- 集群管理员访问权限（创建命名空间和 RBAC）

### 步骤 1：获取你的 Runner Token

1. 登录到你的 GitLab 实例
2. 导航到 **管理中心** → **CI/CD** → **Runners**
3. 点击 **新建实例 runner**
4. 添加标签：`kubernetes`、`docker`
5. 勾选 **运行未标记的任务**
6. 点击 **创建 runner**
7. **复制注册令牌**（格式：`glrt-xxxxxxxxxxxx`）

⚠️ **保存此令牌** —— 你将在步骤3中需要它。

### 步骤 2：安装 Helm Chart

```bash
# 添加 GitLab Helm 仓库
helm repo add gitlab https://charts.gitlab.io
helm repo update

# 创建命名空间
kubectl create namespace gitlab

# 验证
kubectl get namespace gitlab
```

### 步骤 3：配置 Runner

创建文件 `gitlab-runner-values.yaml`：

```yaml
# GitLab 连接
gitlabUrl: https://gitlab.com/  # 或你的自托管 URL
runnerToken: "glrt-YOUR_TOKEN_HERE"  # ⚠️ 替换这个！

# Runner 镜像
image:
  registry: docker.io
  image: gitlab/gitlab-runner
  tag: v18.3.0

# 扩展
replicas: 2              # 2个管理器 pod 实现高可用
concurrent: 10           # 每个管理器最多处理10个任务

# Kubernetes executor 配置
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "alpine:latest"

        # 资源限制（每个任务 pod）
        cpu_limit = "1"
        memory_limit = "1Gi"
        cpu_request = "500m"
        memory_request = "512Mi"

        # Service 容器（如 DinD）
        service_cpu_limit = "1"
        service_memory_limit = "1Gi"

        # 启用特权模式（Docker-in-Docker 需要）
        privileged = true
```

**配置说明**：
- `replicas: 2` — 两个管理器 pod 实现高可用性
- `concurrent: 10` — 总容量：2 × 10 = **20个并发任务**
- `privileged = true` — DinD 访问内核功能所需

> 🔒 **安全提示**：特权容器具有提升的权限。在第5部分中，我们将讨论更安全的替代方案，如 Kaniko。

### 步骤 4：部署

```bash
helm install gitlab-runner \
  --namespace gitlab \
  --version 0.80.0 \
  -f gitlab-runner-values.yaml \
  gitlab/gitlab-runner

# 监视部署
kubectl rollout status deployment/gitlab-runner -n gitlab

# 验证 pod
kubectl get pods -n gitlab -l app=gitlab-runner
```

**预期输出**：
```
NAME                             READY   STATUS    RESTARTS   AGE
gitlab-runner-5d8f7c9b4d-abc12   1/1     Running   0          30s
gitlab-runner-5d8f7c9b4d-def34   1/1     Running   0          30s
```

### 步骤 5：验证注册

检查 runner 是否在线：

```bash
# 检查日志
kubectl logs -n gitlab -l app=gitlab-runner --tail=20

# 查找：
# ✓ Configuration loaded
# ✓ Runner registered successfully
```

或在 GitLab UI 中检查：
- **管理中心** → **CI/CD** → **Runners**
- 你应该看到2个带有绿色 **在线** 指示器的 runner

---

## 你的第一个流水线

让我们运行一个简单的测试来确认一切正常工作。

### 创建测试项目

1. 创建一个新的 GitLab 项目：`runner-test`
2. 添加文件 `.gitlab-ci.yml`：

```yaml
stages:
  - test

hello-kubernetes:
  stage: test
  tags:
    - kubernetes  # 路由到你的 K8s runner
  image: alpine:latest
  script:
    - echo "来自 Kubernetes Runner 的问候！"
    - echo "运行在 pod: $(hostname)"
    - echo "日期: $(date)"
    - echo "测试网络:"
    - ping -c 3 google.com || echo "无网络连接（可能符合预期）"
```

3. 提交并推送

### 观察魔法

**在 GitLab UI 中**（CI/CD → 流水线）：
- 流水线状态：运行中 → 通过 ✅
- 任务日志显示来自 Pod 的输出

**在 Kubernetes 中**：
```bash
# 监视正在创建的 pod
kubectl get pods -n gitlab -w

# 你会看到：
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Pending
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Init:0/1
# runner-xxxxx-project-1-concurrent-0-xxxxx   2/2   Running
# runner-xxxxx-project-1-concurrent-0-xxxxx   0/2   Completed
```

**生命周期分解**：
- `Pending`（2秒）：Kubernetes 调度 Pod
- `Init:0/1`（3秒）：Helper 容器克隆你的代码
- `Running`（10秒）：Build 容器执行你的脚本
- `Completed`：Pod 退出，结果发送到 GitLab
- 几分钟后自动删除

> ✅ **验证完成**：你已成功在动态创建的 Kubernetes Pod 中运行了一个 CI/CD 任务。

---

## 刚刚发生了什么？

让我们追踪完整的流程：

```
1. 你推送了代码
2. GitLab 创建了流水线
3. Runner Manager（在 K8s 中）轮询 GitLab API
4. Manager 看到任务并创建了一个 Pod
5. Helper 容器克隆了你的仓库
6. Build 容器（alpine）运行了你的脚本
7. 脚本输出被流式传输到 GitLab
8. Pod 报告"成功"并退出
9. GitLab 将任务标记为通过 ✅
10. Kubernetes 删除了 Pod
```

**关键好处**：无需手动清理。Kubernetes 自动处理整个生命周期。

---

## 关键要点

让我们总结传统设置和 Kubernetes Executor 之间的主要区别：

| 传统设置 | Kubernetes Executor |
|---------|---------------------|
| 💾 磁盘空间累积 | 🗑️ 临时 Pod（自动清理） |
| 📉 手动扩展 | 📈 随集群自动扩展 |
| 🔥 资源冲突 | 🔒 每个任务隔离 |
| 💰 始终在线的 runner | 💸 按使用付费（仅在任务运行时） |

---

## 接下来是什么？

在第一部分中，我们介绍了 Kubernetes Executor 的"是什么"和"为什么"，并得到了一个可工作的设置。但我们只是触及了表面。

**第二部分即将推出**：
- 🔍 **深入探讨4个容器** — 内部究竟发生了什么？
- 🐳 **揭开 Docker-in-Docker 的神秘面纱** — 客户端与服务器架构
- 🌐 **网络命名空间** — 容器如何通信
- 💾 **资源管理** — 防止 OOM 终止

**预览**：我们将回答诸如"为什么 `docker build` 在 Build 容器中运行，但实际的镜像层在 DinD 容器中创建？"这样的问题，并提供详细的时序图和执行流程。

---

## 实践练习

在进入第二部分之前，尝试这些练习来加深你的理解：

**练习 1**：修改测试任务以使用不同的镜像：
```yaml
image: node:18-alpine
script:
  - node --version
  - npm --version
```

**练习 2**：添加第二个阶段：
```yaml
stages:
  - test
  - build

test-job:
  stage: test
  script:
    - echo "运行测试..."

build-job:
  stage: build
  script:
    - echo "构建应用..."
```

**练习 3**：实时监视 Pod 生命周期：
```bash
kubectl get pods -n gitlab -w --show-all
```

---

## 资源

- [官方 GitLab Runner 文档](https://docs.gitlab.com/runner/)
- [Kubernetes Executor 配置参考](https://docs.gitlab.com/runner/executors/kubernetes.html)
- [Helm Chart 仓库](https://gitlab.com/gitlab-org/charts/gitlab-runner)

---

## 有问题？

在下方留言！我将在接下来的部分中解答常见问题：
- "我可以在 GitLab.com（SaaS）上使用这个吗？" — 可以！
- "如何处理私有 Docker 镜像仓库？" — 第3部分
- "网络策略和安全性呢？" — 第5部分

---

**第二部分即将推出** — 我们将详细探索每个容器的内部架构。

如果你觉得这有帮助，欢迎与其他学习 GitLab CI/CD 的人分享。

---

*这是关于 Kubernetes 上的 GitLab Runner 的5部分系列的第1部分，记录了学习和实施过程。*

**系列索引**：
- **第1部分：架构与快速搭建** ← 你在这里
- 第2部分：深入探讨容器架构（即将推出）
- 第3部分：构建真实世界的 CI/CD 流水线（即将推出）
- 第4部分：解决 DNS 和网络问题（即将推出）
- 第5部分：生产最佳实践（即将推出）

---

*标签：#GitLab #Kubernetes #CICD #DevOps #Docker #CloudNative*
