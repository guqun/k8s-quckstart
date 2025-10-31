# Kubernetes 上的 GitLab Runner：完整指南
## 第二部分 — 深入探讨容器架构

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

> *如果你还没有阅读[第一部分](link-to-part-1)，请先阅读——我们在其中介绍了基础知识并部署了一个可工作的 runner。*

---

在第一部分中，我们在 Kubernetes 上部署了 GitLab Runner 并运行了第一个流水线。我们简要提到了"四个容器"，但没有探索每个容器内部实际发生的事情。

在本部分中，我们将剖析完整的架构。最后，你将理解：
- 为什么有四个不同的容器以及每个容器做什么
- Docker-in-Docker 如何真正工作（剧透：它是客户端-服务器，不是魔法）
- 从"git push"到"流水线成功"的完整生命周期
- 资源管理以及并发运行任务时会发生什么

让我们从架构概览开始。

---

## 四容器架构

当你在 Kubernetes executor 上触发 CI/CD 任务时，GitLab Runner 创建一个包含**最多四个专用容器**的 Pod。每个容器都有不同的角色：

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes 集群                           │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Runner Manager Pod（持久化）                      │     │
│  │  ┌──────────────────────────────────────────────┐  │     │
│  │  │  gitlab-runner:v18.3.0                       │  │     │
│  │  │  - 轮询 GitLab 获取任务                      │  │     │
│  │  │  - 当有工作到来时创建任务 Pod                │  │     │
│  │  │  - 监控任务执行                              │  │     │
│  │  └──────────────────────────────────────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
│                          │                                   │
│                          │ 按需创建                          │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────┐     │
│  │  任务 Pod（临时）                                  │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  1. Helper（Init Container）               │   │     │
│  │  │     gitlab-runner-helper:v18.3.0           │   │     │
│  │  │     - 克隆 Git 仓库                         │   │     │
│  │  │     - 下载产物                              │   │     │
│  │  │     - 任务后上传结果                        │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  2. Build（主容器）                        │   │     │
│  │  │     用户定义镜像（如 docker:latest）        │   │     │
│  │  │     - 执行 CI/CD 脚本                       │   │     │
│  │  │     - 向服务发送命令                        │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  │                                                    │     │
│  │  ┌────────────────────────────────────────────┐   │     │
│  │  │  3. Service（Sidecar 容器）                │   │     │
│  │  │     用户定义（如 docker:dind）              │   │     │
│  │  │     - 提供后台服务                          │   │     │
│  │  │     - 在整个任务持续时间运行                │   │     │
│  │  └────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

让我们详细检查每个容器。

---

## 容器 1：Runner Manager Pod

**镜像**：`gitlab/gitlab-runner:v18.3.0`
**类型**：Deployment（持久化）
**生命周期**：持续运行

### 职责

Runner Manager 是永不停止运行的"协调器"：

1. **任务轮询**：持续查询 GitLab API 获取待处理任务
2. **Pod 创建**：当工作到来时在 Kubernetes 中创建任务 Pod
3. **状态监控**：跟踪任务执行并报告回 GitLab
4. **日志收集**：实时将任务日志流式传输到 GitLab UI
5. **清理**：确保已完成的任务 Pod 被移除

### 配置示例

```yaml
# 来自 gitlab-runner-values.yaml
replicas: 3              # 3个管理器 pod 实现高可用
concurrent: 10           # 每个管理器最多处理10个任务
```

**容量计算**：
- 3个管理器 × 10并发 = **30个任务可以同时运行**

### 工作原理

```
Manager Pod 主循环：
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

管理器本身不执行任何 CI/CD 脚本——它纯粹是一个控制平面组件。

---

## 容器 2：Helper 容器

**镜像**：`gitlab-runner-helper:x86_64-v18.3.0`
**类型**：Init Container
**生命周期**：在任务开始和结束时运行

### 职责

Helper 处理所有 Git 和产物操作：

**任务开始时**：
- 将 Git 仓库克隆到共享卷
- 下载前一阶段的产物
- 设置工作目录

**任务结束时**：
- 上传产物到 GitLab
- 上传缓存（如果配置）
- 上传测试报告

### 执行流程

```
T+0s   创建任务 Pod
       │
T+1s   Helper 容器启动（作为 init 容器）
       │
       ├─ 步骤 1：克隆仓库
       │  $ git clone --depth 50 https://gitlab.com/user/repo.git
       │
       ├─ 步骤 2：检出特定提交
       │  $ git checkout $CI_COMMIT_SHA
       │
       ├─ 步骤 3：下载产物（如果有）
       │  $ wget $GITLAB_API/jobs/$JOB_ID/artifacts
       │
       └─ 步骤 4：完成（退出）

T+3s   Helper 退出，Build 容器可以启动

       [Build 容器运行...]

T+160s Build 完成

       Helper 重新激活（作为任务后容器）
       │
       ├─ 上传产物
       │  $ curl -F "file=@output.zip" $GITLAB_API/artifacts
       │
       └─ 上传缓存
          $ curl -F "file=@.cache.tar.gz" $GITLAB_API/cache
```

### 共享卷

Helper 和 Build 容器共享一个卷：

```
共享 emptyDir 卷：/builds
├── namespace/
│   └── project-name/
│       ├── .git/
│       ├── src/
│       ├── Dockerfile
│       └── .gitlab-ci.yml
```

**关键点**：Build 容器本身不克隆代码——它只是使用 Helper 准备的内容。

---

## 容器 3：Build 容器

**镜像**：由 `.gitlab-ci.yml` 中的 `image` 定义
**类型**：主容器
**生命周期**：在任务持续时间内运行

### 职责

这是你的 CI/CD 脚本实际执行的地方：

- 运行 `before_script`、`script`、`after_script`
- 可以访问克隆的仓库
- 通过网络与 Service 容器通信
- 将退出代码报告回 GitLab

### 配置

在你的 `.gitlab-ci.yml` 中：

```yaml
build-job:
  image: docker:latest      # ← 定义 Build 容器镜像
  script:
    - docker build -t myapp .
    - docker push myapp
```

Build 容器使用这些默认值运行：
- 工作目录：`/builds/<namespace>/<project>`
- 用户：`root`（可配置）
- 网络：与 Service 容器共享

### 资源限制

来自 `gitlab-runner-values.yaml`：

```yaml
[runners.kubernetes]
  cpu_limit = "2"              # 最多 2 CPU 核心
  memory_limit = "2Gi"         # 最多 2GB RAM
  cpu_request = "500m"         # 保证 0.5 核心
  memory_request = "512Mi"     # 保证 512MB
```

如果超过会发生什么？
- **CPU 限制**：被限流（任务变慢）
- **内存限制**：OOM 终止（任务失败）

---

## 容器 4：Service 容器（Docker-in-Docker）

**镜像**：由 `.gitlab-ci.yml` 中的 `services` 定义
**类型**：Sidecar 容器
**生命周期**：在整个任务持续时间内运行

这是最复杂的容器，所以让我们彻底分解它。

### 什么是 Docker-in-Docker？

当你的 CI/CD 任务需要构建 Docker 镜像时，你面临一个挑战：**Build 容器已经在 Docker 内部**（运行在 Kubernetes 上）。你如何在 Docker 内部运行 Docker？

**解决方案**：在一个单独的容器中运行一个完整的 Docker 守护进程（`dockerd`）。

### 配置示例

```yaml
# .gitlab-ci.yml
services:
  - name: docker:dind
    alias: docker          # ← 网络别名（重要！）
    command:
      - "--tls=false"      # 为简单起见禁用 TLS
      - "--insecure-registry=my-registry.com"

variables:
  DOCKER_HOST: tcp://docker:2375  # ← 连接到 DinD
```

### 完整生命周期（0秒到170秒）

让我通过实际时间带你走过整个生命周期：

```
T+0s    Runner Manager 创建任务 Pod
        Kubernetes 调度器选择一个节点

T+1s    ========== 阶段 1：容器启动 ==========

        Helper 容器（init）启动
        └─ 克隆代码（2秒）

T+3s    Helper 完成，进入等待状态

        ========== 阶段 2：Service 容器启动 ==========

T+4s    DinD 容器启动
        └─ 执行 ENTRYPOINT：dockerd-entrypoint.sh
            ├─ 初始化存储驱动（overlay2）
            ├─ 设置 iptables 规则
            ├─ 启动 containerd
            └─ 启动 dockerd 守护进程

T+5s    dockerd 启动中...
        日志：
          level=info msg="Starting up"
          level=info msg="containerd not running, starting managed containerd"

T+7s    dockerd 监听端口
        日志：
          level=info msg="API listen on /var/run/docker.sock"
          level=info msg="API listen on [::]:2375"

T+8s    ========== 阶段 3：DinD 就绪 ==========

        DinD 完全就绪，等待客户端连接
        状态：Ready = true

        Build 容器启动
        └─ 执行 before_script

T+9s    ========== 阶段 4：第一个请求 ==========

        Build 容器：docker info

        网络流：
          Build 容器（docker CLI）
              ↓ TCP 连接
          tcp://docker:2375
              ↓ 通过别名解析
          127.0.0.1:2375（localhost）
              ↓ 本地回环
          DinD 容器（dockerd 进程）

        DinD 日志：
          level=debug msg="Calling GET /v1.40/info"

        返回：Server Version: 27.1.1
        ✅ 连接确认

T+10s   ========== 阶段 5：执行构建 ==========

        Build 容器：docker build -t myapp .

        请求流：
          ① Build 发送构建上下文（tar 文件）
             ↓
          ② DinD 接收上下文，提取到临时目录
             日志："Building context from tarball"
             ↓
          ③ DinD 读取 Dockerfile
             解析：FROM python:3.9-slim
             ↓
          ④ DinD 检查本地镜像缓存
             结果：未找到
             ↓
          ⑤ DinD 执行 DNS 解析
             查询：/etc/resolv.conf
             名称服务器：10.0.0.2
             域：registry.example.com
             结果：203.0.113.10
             ↓
          ⑥ DinD 建立 HTTPS 连接
             连接：203.0.113.10:443
             ↓
          ⑦ DinD 拉取镜像层
             日志：
               "Pulling from library/python"
               "Layer 1: Downloading [=>  ] 5MB/50MB"
               "Layer 2: Downloading [===>] 15MB/30MB"
             （35秒下载时间）
             ↓
          ⑧ DinD 执行 Dockerfile 指令
             COPY . . → 将代码复制到镜像中
             RUN pip install → 执行安装命令
             ↓
          ⑨ DinD 创建新镜像
             镜像 ID：sha256:abc123...
             标签：myapp:latest
             ↓
          ⑩ DinD 返回构建结果
             ↓
          Build 容器接收成功消息
             "Successfully built abc123..."
             "Successfully tagged myapp:latest"

T+120s  ========== 阶段 6：推送镜像 ==========

        Build 容器：docker push myapp:latest

        推送流：
          ① Build 发送推送请求
             ↓
          ② DinD 读取镜像层
             检查远程存在哪些层
             ↓
          ③ DinD 连接到目标仓库
             地址：private-registry.example.com
             认证：使用 docker login 凭据
             ↓
          ④ DinD 上传镜像层
             日志：
               "Layer 1: Pushed (already exists)"
               "Layer 2: Pushed (already exists)"
               "Layer 3: Pushing [=>  ] 10MB/50MB"
             ↓
          ⑤ DinD 推送清单
             标签：56cc4792-86
             ↓
          ⑥ DinD 返回成功
             ↓
          Build 容器接收成功
             "56cc4792-86: digest: sha256:def456... size: 1234"

T+160s  ========== 阶段 7：Build 脚本完成 ==========

        Build 容器完成脚本执行
        容器进入 Completed 状态

        DinD 容器仍在运行
        └─ 等待 Pod 终止信号

T+165s  ========== 阶段 8：清理 ==========

        Kubernetes 向所有容器发送 SIGTERM

        DinD 接收信号
        日志：
          level=info msg="Processing signal 'terminated'"
          level=info msg="Stopping daemon"
          level=info msg="Daemon shutdown complete"

        DinD 优雅关闭（5秒）
        └─ 停止 containerd
        └─ 清理临时文件
        └─ 释放端口

T+170s  Pod 删除完成
```

---

## 关键概念：Docker 客户端-服务器架构

这是最常被误解的方面。让我澄清一下：

### 实际发生的事

```
┌─────────────────────────────────────────────────────────┐
│                     任务 Pod                            │
│                                                         │
│  ┌──────────────────┐         ┌──────────────────┐    │
│  │ Build 容器       │         │  DinD 容器       │    │
│  │                  │         │                  │    │
│  │ ✅ 你在哪里      │         │ ✅ 工作实际      │    │
│  │ 输入命令         │         │ 发生的地方       │    │
│  │ （客户端）       │         │ （服务器）       │    │
│  │                  │         │                  │    │
│  │ $ docker build   │────────►│  dockerd         │    │
│  │   ↑              │  API    │   ↓              │    │
│  │   只是CLI工具    │  调用   │   构建引擎       │    │
│  │                  │         │                  │    │
│  │ 做的事：         │         │ 做的事：         │    │
│  │ - 读取Dockerfile │         │ - 拉取镜像       │    │
│  │ - 打包上下文     │         │ - 提取上下文     │    │
│  │ - 发送API请求    │         │ - 运行命令       │    │
│  │ - 显示日志       │         │ - 创建层         │    │
│  │                  │         │ - 生成ID         │    │
│  │ 不做的事：       │         │ 不做的事：       │    │
│  │ - 拉取镜像       │         │ - 读取用户输入   │    │
│  │ - 运行命令       │         │ - 显示输出       │    │
│  │ - 创建层         │         │                  │    │
│  └──────────────────┘         └──────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 详细职责

| 步骤 | Build 容器做的 | DinD 容器做的 | 在哪里？ |
|------|---------------|--------------|---------|
| **1. 启动命令** | 执行 `docker build -t myapp .` | dockerd 监听 2375 | Build |
| **2. 准备** | 读取当前目录中的 Dockerfile | 等待请求 | Build |
| **3. 打包上下文** | 创建当前目录的 tar 文件（5MB） | - | Build |
| **4. 发送请求** | HTTP POST tar 到 `tcp://docker:2375/build` | 接收 tar，提取到 `/tmp/docker-build-xxx` | Build → DinD |
| **5. 解析 Dockerfile** | - | 读取 Dockerfile，解析指令 | DinD |
| **6. 拉取镜像** | - | 执行 `FROM python:3.9-slim`<br>DNS 解析，镜像拉取（35秒） | DinD |
| **7. 构建层** | 接收并显示日志流 | 执行 `RUN pip install`<br>在临时容器中运行命令 | DinD |
| **8. 完成** | 显示 "Successfully built abc123" | 提交镜像层，生成镜像 ID | DinD → Build |

### 为什么这样设计？

**1. 关注点分离**
- Build 容器：用户交互，脚本执行
- DinD 容器：Docker 引擎，镜像管理

**2. 资源隔离**
- Build 失败不会导致 Docker 守护进程崩溃
- 你可以独立限制 DinD 资源

**3. 安全性**
- Build 容器不需要特权模式
- 只有 DinD 需要 `privileged: true`
- 减少安全风险面

**4. 灵活性**
- 可以连接到远程 Docker 守护进程
- 可以在不触及服务的情况下替换构建工具

### 验证

你可以验证这个架构：

**在 Build 容器中**：
```bash
# 只有 docker CLI 工具
$ which docker
/usr/local/bin/docker

# 它只是一个客户端二进制文件
$ file /usr/local/bin/docker
/usr/local/bin/docker: ELF 64-bit LSB executable

# 没有 dockerd 守护进程
$ which dockerd
(not found)

# 连接到远程 dockerd
$ echo $DOCKER_HOST
tcp://docker:2375

# 没有 dockerd 进程
$ ps aux | grep dockerd
(no results)
```

**在 DinD 容器中**：
```bash
# 有正在运行的 dockerd 守护进程
$ ps aux | grep dockerd
root  21  dockerd --host=tcp://0.0.0.0:2375 --tls=false

# 守护进程监听端口
$ netstat -tuln | grep 2375
tcp6  0  0  :::2375  :::*  LISTEN
```

---

## 容器通信

任务 Pod 中的所有容器共享相同的网络命名空间：

```
┌──────────────────────────────────────────────────────┐
│           任务 Pod 网络命名空间                       │
│           （所有容器共享这个）                        │
│                                                      │
│  网络接口：                                          │
│  ┌────────────────────────────────────────────┐     │
│  │  lo（回环）                                 │     │
│  │  127.0.0.1/8                               │     │
│  │  用于：容器到容器（docker）                 │     │
│  └────────────────────────────────────────────┘     │
│                                                      │
│  ┌────────────────────────────────────────────┐     │
│  │  eth0（veth 对）                            │     │
│  │  172.24.1.100/24（示例）                    │     │
│  │  用于：外部通信                             │     │
│  └────────────────────────────────────────────┘     │
│                                                      │
│  /etc/hosts:                                         │
│  ┌────────────────────────────────────────────┐     │
│  │  127.0.0.1 localhost                       │     │
│  │  127.0.0.1 docker    # Service 别名        │     │
│  │  172.24.1.100 runner-xxx-xxx               │     │
│  └────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

**关键点**：
- Service 别名（`docker`）解析为 `127.0.0.1`
- 通信通过 localhost 发生（没有网络开销）
- 所有容器看到相同的 IP 地址和网络接口

---

## 资源管理

理解资源分配对于稳定运行至关重要。

### 每个任务的资源

每个任务 Pod 可以消耗：

```yaml
# Build 容器
cpu_limit = "2"              # 最多 2 核心
memory_limit = "2Gi"         # 最多 2GB

# Service 容器（DinD）
service_cpu_limit = "2"      # 最多 2 核心
service_memory_limit = "2Gi" # 最多 2GB

# Helper 容器
helper_cpu_limit = "500m"    # 最多 0.5 核心
helper_memory_limit = "512Mi" # 最多 512MB
```

**每个任务总计**：最多 4.5 CPU 核心和 4.5GB 内存

### 集群容量规划

示例计算：

```
集群：3个工作节点，每个有 8 核心和 32GB RAM

可用于任务：
- CPU：3 × 8 = 24 核心（假设 100% 可分配）
- 内存：3 × 32GB = 96GB

最大并发任务数（如果所有任务都使用最大资源）：
- 按 CPU：24 ÷ 4.5 = 5 个任务
- 按内存：96 ÷ 4.5 = 21 个任务
- 瓶颈：CPU → 最多 5 个并发任务

实际容量（典型 50% CPU 使用率）：
- 约 10 个并发任务舒适运行
```

### 超过限制时会发生什么？

**CPU 限制**：
```
任务尝试使用 3 核心（限制是 2）
   ↓
Kubernetes 限流容器
   ↓
任务运行较慢但不会失败
   ↓
在指标中可见：CPU 限流
```

**内存限制**：
```
任务尝试分配 3GB（限制是 2GB）
   ↓
容器立即被 OOM 终止
   ↓
Pod 重启（如果重启策略允许）
   ↓
任务失败，错误："OOMKilled: Container exceeded memory limit"
```

---

## 综合起来

让我们追踪从开始到结束的完整任务：

```
开发者：git push

GitLab 服务器：创建流水线
   ↓
Runner Manager：轮询 API，看到新任务
   ↓
Runner Manager：创建任务 Pod
   ↓
Kubernetes：将 Pod 调度到 worker-node-2
   ↓
Helper 容器：启动（init）
   - 克隆仓库
   - 下载产物
   - 退出
   ↓
Service 容器（DinD）：启动
   - 启动 dockerd
   - 监听 tcp://0.0.0.0:2375
   - 就绪
   ↓
Build 容器：启动
   - 读取 /builds/namespace/project/
   - 执行 before_script
   - 执行 script:
     * docker build
        → 连接到 tcp://docker:2375
        → DinD 执行构建
        → 返回成功
     * docker push
        → 连接到 tcp://docker:2375
        → DinD 推送镜像
        → 返回成功
   - 以代码 0 退出
   ↓
Helper 容器：重新激活
   - 上传产物
   - 上传缓存
   - 退出
   ↓
Runner Manager：收集日志
   - 流式传输到 GitLab
   - 将任务标记为成功
   ↓
Kubernetes：删除任务 Pod
   ↓
GitLab：显示 ✅ 流水线通过
```

**总时间**：约 160 秒（根据镜像大小和网络而异）

---

## 关键要点

**容器角色**：
1. **Runner Manager**：协调器（从不执行任务）
2. **Helper**：Git 和产物处理程序
3. **Build**：脚本执行器（你的代码在这里运行）
4. **Service（DinD）**：后台服务（实际工作在这里发生）

**Docker-in-Docker 架构**：
- Build 容器 = Docker **客户端**（发送命令）
- DinD 容器 = Docker **服务器**（做工作）
- 通过 localhost 上的 TCP 通信

**资源管理**：
- 每个任务最多可以使用 4.5 CPU 核心和 4.5GB RAM（使用默认限制）
- CPU 限制导致限流，内存限制导致 OOM 终止
- 根据并发任务需求规划集群容量

**网络**：
- 所有容器共享 Pod 网络命名空间
- Service 别名解析为 127.0.0.1
- 容器间通信没有网络开销

---

## 接下来是什么？

在第二部分中，我们探索了每个容器的内部架构和时序。我们现在理解了各个部分如何组合在一起。

**第三部分即将推出**：
- 🎬 **完整的 CI/CD 工作流** — 从代码推送到生产部署
- 📝 **.gitlab-ci.yml 最佳实践** — 构建、测试和部署阶段
- 🐳 **Docker 镜像优化** — 更快的构建，更小的镜像
- ☸️ **Kubernetes 部署** — 从 CI/CD 使用 kubectl

**预览**：我们将构建一个真实世界的流水线，构建 Docker 镜像，推送到仓库，并部署到 Kubernetes——全部自动化。

---

## 实践练习

**练习 1**：计算你的集群任务容量

给定：
- 4 个工作节点
- 每个节点 16 核心和 64GB RAM
- 任务限制：2 CPU，2GB 内存

你可以运行多少个并发任务？

**练习 2**：监控容器资源使用

```bash
# 实时监视任务 Pod 资源
kubectl top pod -n gitlab --containers

# 与限制比较
kubectl get pod <job-pod> -n gitlab -o yaml | grep -A 5 resources
```

**练习 3**：检查 DinD 容器

```bash
# 当任务运行时，exec 进入 DinD
kubectl exec -n gitlab <job-pod> -c docker -- ps aux

# 检查 dockerd 监听端口
kubectl exec -n gitlab <job-pod> -c docker -- netstat -tuln | grep 2375
```

---

## 资源

- [GitLab Runner Executors 文档](https://docs.gitlab.com/runner/executors/)
- [Kubernetes 资源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Docker Engine API 参考](https://docs.docker.com/engine/api/)

---

**第三部分即将推出** — 我们将构建一个包含 Docker 构建和 Kubernetes 部署的完整 CI/CD 流水线。

如果你觉得这有帮助，欢迎与其他学习 GitLab CI/CD 的人分享。

---

*这是关于 Kubernetes 上的 GitLab Runner 的5部分系列的第2部分，记录了学习和实施过程。*

**系列索引**：
- [第1部分：架构与快速搭建](link-to-part-1)
- **第2部分：深入探讨容器架构** ← 你在这里
- 第3部分：构建真实世界的 CI/CD 流水线（即将推出）
- 第4部分：解决 DNS 和网络问题（即将推出）
- 第5部分：生产最佳实践（即将推出）

---

*标签：#GitLab #Kubernetes #CICD #DevOps #Docker #容器架构*
