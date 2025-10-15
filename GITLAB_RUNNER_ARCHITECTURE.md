# GitLab Runner 容器架构详解

## 概述

GitLab Runner 在 Kubernetes 环境中运行时，涉及**四类容器**的协同工作。本文档详细说明各容器的功能、运行时机及交互方式。

---

## 一、四类容器详解

### 1. Runner Manager Pod（管理器容器）

**镜像**: `gitlab/gitlab-runner:v18.3.0`

**功能**:
- 持续运行的常驻进程
- 向 GitLab Server 注册并轮询获取 CI/CD 任务
- 接收到任务后，在 Kubernetes 中创建 Job Pod
- 监控 Job Pod 的执行状态
- 收集任务日志并上报给 GitLab Server
- 管理 Runner 的生命周期

**运行时机**:
- Helm 安装后持续运行
- 配置为 Deployment，副本数可配置（本例为 3 个副本）

**网络配置**:
- 使用 Kubernetes 默认 DNS (kube-dns)
- 需要访问 GitLab Server (`http://gitlab.example.com/`)
- 监听 Prometheus metrics 端口 (9252)

**关键配置** (`gitlab-runner-values.yaml`):
```yaml
image:
  registry: registry.example.com
  image: gitlab/gitlab-runner
  tag: v18.3.0

replicas: 3  # 3 个 Manager 副本

concurrent: 10  # 每个 Manager 最多并发 10 个 job
```

---

### 2. Helper Container（辅助容器）

**镜像**: `gitlab-runner-helper:x86_64-v18.3.0`

**功能**:
- 克隆 Git 仓库代码到共享 Volume
- 从 GitLab Server 下载 artifacts（构建产物）
- 执行 Job 完成后，上传 artifacts、cache、test reports 到 GitLab
- 处理 Git LFS、子模块等复杂 Git 操作
- 在 Job 的 init、pre-script、post-script 阶段运行

**运行时机**:
- Job Pod 启动时作为 **init container** 先运行
- 克隆代码后进入等待状态
- Job 执行完成后，再次激活上传产物

**网络配置**:
- 继承 Job Pod 的 DNS 配置
- 需要访问 GitLab Server (拉取代码、上传产物)
- 需要访问配置的镜像仓库 (拉取 helper 镜像本身)

**关键配置**:
```yaml
helper_image = "registry.example.com/gitlab/gitlab-runner-helper:x86_64-v18.3.0"
helper_cpu_limit = "500m"
helper_memory_limit = "512Mi"
```

**文件共享**:
- 通过 Kubernetes emptyDir Volume 与 Build Container 共享 `/builds` 目录
- 代码克隆到 `/builds/<namespace>/<project>` 路径

---

### 3. Build Container（构建容器）

**镜像**: 在 `.gitlab-ci.yml` 中通过 `image` 字段指定

**示例**:
```yaml
# build-job 使用的镜像
image: registry.example.com/docker:latest

# deploy-job 使用的镜像
image: registry.example.com/bitnami/kubectl:latest
```

**功能**:
- 执行 CI/CD 脚本（before_script、script、after_script）
- 运行构建、测试、部署命令
- 访问 Service Container（如 DinD、数据库等）
- 工作目录默认为 `/builds/<namespace>/<project>`

**运行时机**:
- Helper Container 克隆代码完成后启动
- 执行完所有脚本后退出

**网络配置**:
- 继承 Job Pod 的 DNS 配置（**本例配置为节点内网 DNS**）
- 与 Service Container 共享 Pod 网络命名空间
- 通过 Service alias 访问服务（如 `tcp://docker:2375`）

**关键配置**:
```yaml
# 资源限制
cpu_limit = "2"
memory_limit = "2Gi"

# 镜像拉取策略
pull_policy = ["if-not-present"]
```

---

### 4. Service Container（服务容器）

**镜像**: 在 `.gitlab-ci.yml` 中通过 `services` 字段指定

**本例使用**: Docker-in-Docker (DinD)
```yaml
services:
  - name: registry.example.com/docker:dind
    alias: docker  # 关键：定义服务的网络别名
    command:
      - "--tls=false"
      - "--insecure-registry=private-registry.example.com"
```

#### 4.1 基本功能

**核心职责**:
- 提供后台服务供 Build Container 使用
- **DinD 场景**: 运行完整的 Docker 守护进程 (dockerd)
- 其他常见场景：
  - 数据库服务（MySQL、PostgreSQL）
  - 缓存服务（Redis、Memcached）
  - 消息队列（RabbitMQ、Kafka）
  - UI 测试（Selenium、Chrome Headless）

#### 4.2 生命周期详解

**完整时间线**（以 DinD 为例）:

```
T+0s    │ Job Pod 创建完成
        │ Kubernetes 调度器选择节点
        │
T+1s    │ ========== 阶段 1: 容器启动 ==========
        │
        │ Helper Container (init) 启动
        │ └── 克隆代码 (2s)
        │
T+3s    │ Helper 完成，进入 Waiting 状态
        │
        │ ========== 阶段 2: Service 容器启动 ==========
        │
T+4s    │ Service Container (DinD) 启动
        │ └── 执行 ENTRYPOINT: dockerd-entrypoint.sh
        │     ├── 初始化 Docker 存储驱动 (overlay2)
        │     ├── 设置 iptables 规则
        │     ├── 启动 containerd
        │     └── 启动 dockerd 守护进程
        │
T+5s    │ dockerd 启动中...
        │ 日志输出:
        │   time="..." level=info msg="Starting up"
        │   time="..." level=info msg="containerd not running, starting managed containerd"
        │
T+7s    │ dockerd 监听端口
        │ 日志输出:
        │   time="..." level=info msg="API listen on /var/run/docker.sock"
        │   time="..." level=info msg="API listen on [::]:2375"
        │
T+8s    │ ========== 阶段 3: DinD 就绪 ==========
        │
        │ DinD 完全就绪，等待客户端连接
        │ 状态标记: Ready = true
        │
        │ Build Container 启动
        │ └── 执行 before_script
        │
T+9s    │ ========== 阶段 4: 接收第一个请求 ==========
        │
        │ Build Container 执行: docker info
        │
        │ 网络流向:
        │   Build Container (docker 命令)
        │       ↓ TCP 连接
        │   tcp://docker:2375
        │       ↓ 解析 alias
        │   127.0.0.1:2375
        │       ↓ 本地回环
        │   DinD Container (dockerd 进程)
        │
        │ DinD 日志:
        │   time="..." level=debug msg="Calling GET /v1.xx/info"
        │
        │ 返回结果 → Build Container
        │   Server Version: 27.1.1
        │   Storage Driver: vfs
        │   ✅ 连接成功确认
        │
T+10s   │ ========== 阶段 5: 执行构建任务 ==========
        │
        │ Build Container 执行: docker build -t myapp .
        │
        │ 【请求流程】
        │   ① Build: 发送构建上下文 (tar 包)
        │      ↓
        │   ② DinD: 接收上下文，解压到临时目录
        │      日志: Building context from tarball
        │      ↓
        │   ③ DinD: 读取 Dockerfile
        │      解析: FROM python:3.9-slim
        │      ↓
        │   ④ DinD: 检查本地镜像缓存
        │      结果: 未找到
        │      ↓
        │   ⑤ DinD: DNS 解析镜像仓库
        │      查询: /etc/resolv.conf
        │      nameserver: 10.0.0.2
        │      域名: registry.example.com
        │      结果: 203.0.113.10
        │      ↓
        │   ⑥ DinD: 建立 HTTPS 连接
        │      连接: 203.0.113.10:443
        │      ↓
        │   ⑦ DinD: 拉取镜像层
        │      日志:
        │        Pulling from library/python
        │        Layer 1: Downloading [=>  ] 5MB/50MB
        │        Layer 2: Downloading [===>] 15MB/30MB
        │      (35s 下载完成)
        │      ↓
        │   ⑧ DinD: 执行 Dockerfile 指令
        │      COPY . . → 复制代码到镜像
        │      RUN pip install → 执行安装命令
        │      ↓
        │   ⑨ DinD: 创建新镜像
        │      Image ID: sha256:abc123...
        │      Tag: myapp:latest
        │      ↓
        │   ⑩ DinD: 返回构建结果
        │      ↓
        │   Build Container: 接收成功消息
        │      Successfully built abc123...
        │      Successfully tagged myapp:latest
        │
T+120s  │ ========== 阶段 6: 推送镜像 ==========
        │
        │ Build Container 执行: docker push myapp:latest
        │
        │ 【推送流程】
        │   ① Build: 发送 push 请求
        │      ↓
        │   ② DinD: 读取镜像层
        │      检查哪些层已存在于远程仓库
        │      ↓
        │   ③ DinD: 连接目标镜像仓库
        │      地址: private-registry.example.com
        │      认证: 使用 docker login 凭证
        │      ↓
        │   ④ DinD: 上传镜像层
        │      日志:
        │        Layer 1: Pushed (already exists)
        │        Layer 2: Pushed (already exists)
        │        Layer 3: Pushing [=>  ] 10MB/50MB
        │      ↓
        │   ⑤ DinD: 推送 manifest
        │      Tag: 56cc4792-86
        │      ↓
        │   ⑥ DinD: 返回成功
        │      ↓
        │   Build Container: 接收成功消息
        │      56cc4792-86: digest: sha256:def456... size: 1234
        │
T+160s  │ ========== 阶段 7: Build 脚本完成 ==========
        │
        │ Build Container 执行完 script
        │ 容器进入 Completed 状态
        │
        │ DinD Container 仍在运行
        │ └── 等待 Pod 终止信号
        │
T+165s  │ ========== 阶段 8: 清理阶段 ==========
        │
        │ Kubernetes 发送 SIGTERM 到所有容器
        │
        │ DinD 接收信号
        │ 日志:
        │   time="..." level=info msg="Processing signal 'terminated'"
        │   time="..." level=info msg="Stopping daemon"
        │   time="..." level=info msg="Daemon shutdown complete"
        │
        │ DinD 优雅退出 (5s)
        │ └── 停止 containerd
        │ └── 清理临时文件
        │ └── 释放端口
        │
T+170s  │ Pod 删除完成
```

#### 4.3 重要澄清：谁在哪里执行？

很多人容易误解：**`docker build` 命令在 Build Container 执行，但实际构建工作在 DinD Container 进行**。

这是因为 Docker 采用的是**客户端-服务器架构**。

##### 📌 Docker 客户端-服务器模型

```
┌─────────────────────────────────────────────────────────┐
│                     Job Pod                              │
│                                                          │
│  ┌──────────────────┐         ┌──────────────────┐     │
│  │ Build Container  │         │  DinD Container  │     │
│  │                  │         │                  │     │
│  │ ✅ 执行命令的地方  │         │ ✅ 干活的地方      │     │
│  │ (客户端)         │         │ (服务器)         │     │
│  │                  │         │                  │     │
│  │ $ docker build   │───────→ │  dockerd 进程    │     │
│  │   ↑             │  API    │   ↓             │     │
│  │   只是客户端工具  │  请求   │   真正的构建引擎  │     │
│  │                  │         │                  │     │
│  │ 职责:            │         │ 职责:            │     │
│  │ - 读取 Dockerfile│         │ - 拉取基础镜像    │     │
│  │ - 打包代码目录    │         │ - 解压构建上下文  │     │
│  │ - 发送 API 请求  │         │ - 执行 RUN 指令   │     │
│  │ - 显示构建日志    │         │ - 创建镜像层      │     │
│  │                  │         │ - 生成镜像 ID     │     │
│  │ 不做:            │         │ 不做:            │     │
│  │ - 不拉取镜像      │         │ - 不读用户输入    │     │
│  │ - 不执行 RUN 指令│         │ - 不显示终端输出  │     │
│  │ - 不创建镜像层    │         │                  │     │
│  └──────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

##### 📊 详细职责划分表

| 步骤 | Build Container 做什么 | DinD Container 做什么 | 发生位置 |
|------|----------------------|---------------------|---------|
| **1. 启动命令** | 执行 `docker build -t myapp .` | dockerd 守护进程监听 2375 端口 | Build |
| **2. 准备阶段** | 读取当前目录的 Dockerfile | 等待接收请求 | Build |
| **3. 打包上下文** | 将当前目录打包成 tar 文件 (5MB) | - | Build |
| **4. 发送请求** | 通过 HTTP POST 发送 tar 到<br>`tcp://docker:2375/build` | 接收 tar 流，解压到<br>`/tmp/docker-build-xxx` | Build → DinD |
| **5. 解析 Dockerfile** | - | 读取 Dockerfile，解析指令 | DinD |
| **6. 拉取镜像** | - | **执行 `FROM python:3.9-slim`**<br>DNS 解析、拉取镜像 (35s) | DinD |
| **7. 构建层** | 接收并显示日志流 | **执行 `RUN pip install`**<br>在临时容器中运行命令 | DinD |
| **8. 完成** | 显示 "Successfully built abc123" | 提交镜像层，生成镜像 ID | DinD → Build |

##### 🔍 执行流程放大镜

```
用户在 Build Container 的 shell 中执行:
$ docker build -t myapp .

    ↓ 【Build Container 内部】
┌────────────────────────────────────────────────┐
│ Step 1: docker CLI 扫描当前目录                 │
│ - 读取 Dockerfile                              │
│ - 扫描所有文件                                  │
│ - 创建 .tar.gz (构建上下文)                     │
│ - 大小: 5MB                                    │
└─────────────────┬──────────────────────────────┘
                  │
                  │ 网络传输 (localhost)
                  │ HTTP POST /v1.41/build?t=myapp
                  │ Content-Type: application/x-tar
                  │ Body: <5MB tar 流>
                  ↓
    ↓ 【DinD Container 内部】
┌────────────────────────────────────────────────┐
│ Step 2: dockerd 接收请求                        │
│ - 监听端口 2375                                │
│ - 接收 tar 流                                  │
│ - 解压到 /tmp/docker-build-abc123/             │
└─────────────────┬──────────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────────┐
│ Step 3: dockerd 读取 Dockerfile                 │
│ FROM python:3.9-slim                           │
│ COPY . .                                       │
│ RUN pip install -r requirements.txt            │
└─────────────────┬──────────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────────┐
│ Step 4: dockerd 拉取基础镜像                    │
│ - DNS 解析 registry.example.com (DinD 的 DNS) │
│ - HTTPS 连接 203.0.113.10:443                  │
│ - 下载 python:3.9-slim (50MB)                 │
│ - 保存到 /var/lib/docker/image/               │
└─────────────────┬──────────────────────────────┘
                  │
                  │ 流式返回日志 (JSON)
                  │ {"stream": "Pulling from library/python\n"}
                  │ {"status": "Downloading", "progress": "[=>]"}
                  ↓
    ↑ 【Build Container 接收日志】
    ┌────────────────────────────────────┐
    │ 终端显示:                           │
    │ Pulling from library/python        │
    │ Layer 1: Downloading [=>  ] 5MB    │
    └────────────────────────────────────┘
                  │
                  ↓ 【继续在 DinD Container】
┌────────────────────────────────────────────────┐
│ Step 5: dockerd 执行 COPY 指令                  │
│ - 从 /tmp/docker-build-abc123/ 复制文件        │
│ - 创建新的镜像层                               │
└─────────────────┬──────────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────────┐
│ Step 6: dockerd 执行 RUN 指令                   │
│ - 基于 python:3.9-slim 创建临时容器            │
│ - 在临时容器中执行: pip install                │
│ - 提交容器为新的镜像层                          │
│ - 删除临时容器                                 │
└─────────────────┬──────────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────────┐
│ Step 7: dockerd 完成构建                        │
│ - 生成镜像 ID: sha256:abc123...               │
│ - 打标签: myapp:latest                         │
│ - 存储到 /var/lib/docker/                     │
└─────────────────┬──────────────────────────────┘
                  │
                  │ 返回成功 (JSON)
                  │ {"stream": "Successfully built abc123...\n"}
                  │ {"stream": "Successfully tagged myapp:latest\n"}
                  ↓
    ↑ 【Build Container 接收结果】
    ┌────────────────────────────────────┐
    │ 终端显示:                           │
    │ Successfully built abc123...       │
    │ Successfully tagged myapp:latest   │
    │                                    │
    │ $ (命令执行完成，返回 shell 提示符) │
    └────────────────────────────────────┘
```

##### 🎯 类比理解

**类比 1：餐厅点餐**
```
Build Container = 顾客
  └── 说："我要一份炒饭"（docker build）
  └── 等待上菜（接收日志）
  └── 吃饭（使用构建好的镜像）

DinD Container = 厨房
  └── 接单（接收 API 请求）
  └── 准备食材（拉取基础镜像）
  └── 炒菜（执行 Dockerfile 指令）
  └── 出餐（返回镜像 ID）

顾客不会自己炒菜，厨房不会直接面对顾客
```

**类比 2：浏览器访问网站**
```
Build Container = Chrome 浏览器
  └── 输入网址，点击"搜索"
  └── 发送 HTTP 请求
  └── 显示返回的 HTML

DinD Container = Web 服务器
  └── 接收请求
  └── 查询数据库
  └── 生成 HTML
  └── 返回给浏览器

浏览器不处理业务逻辑，服务器不显示页面
```

##### ❓ 为什么这样设计？

**1. 分离关注点**
- Build Container：用户交互、脚本执行
- DinD Container：Docker 引擎、镜像管理

**2. 资源隔离**
- 构建失败不会导致 Build Container 崩溃
- 可以独立限制 DinD 的资源使用（CPU/内存）

**3. 安全性**
- Build Container 不需要特权模式
- 只有 DinD 需要 `privileged: true`
- 降低安全风险

**4. 灵活性**
- 可以连接远程 Docker 守护进程
- 同一个 DinD 可以服务多个构建任务（虽然 GitLab Runner 不这么用）

##### 🔍 验证方式

**在 Build Container 中查看：**
```bash
# 只有 docker 命令行工具
$ which docker
/usr/local/bin/docker

# 是一个客户端程序
$ file /usr/local/bin/docker
/usr/local/bin/docker: ELF 64-bit LSB executable

# 没有 dockerd 守护进程
$ which dockerd
(not found)

# 连接的是远程 dockerd
$ echo $DOCKER_HOST
tcp://docker:2375

# 进程列表中没有 dockerd
$ ps aux | grep dockerd
(no results)
```

**在 DinD Container 中查看：**
```bash
# 有 dockerd 守护进程
$ ps aux | grep dockerd
root  21  dockerd --host=tcp://0.0.0.0:2375 --tls=false

# 守护进程监听端口
$ netstat -tuln | grep 2375
tcp6  0  0  :::2375  :::*  LISTEN

# 也有 docker CLI（但很少用）
$ which docker
/usr/local/bin/docker
```

##### 📌 总结

| 问题 | 答案 |
|------|------|
| `docker build` 命令在哪里执行？ | ✅ Build Container（命令行输入） |
| Dockerfile 在哪里读取？ | ✅ Build Container（打包上下文时）<br>✅ DinD Container（解析指令时） |
| 基础镜像在哪里拉取？ | ✅ DinD Container |
| `RUN pip install` 在哪里执行？ | ✅ DinD Container（临时容器中） |
| 镜像层在哪里创建？ | ✅ DinD Container |
| 镜像存储在哪里？ | ✅ DinD Container（`/var/lib/docker/`） |
| 构建日志在哪里显示？ | ✅ Build Container（终端输出） |
| DNS 解析在哪里发生？ | ✅ DinD Container（拉取镜像时） |

**核心结论**：
- ✅ **命令在 Build Container 执行**（用户界面）
- ✅ **工作在 DinD Container 进行**（实际引擎）
- 🔑 **Build Container 只是客户端**，通过网络 API 调用 DinD

这就像你在手机上点"外卖"，但做饭的是餐厅，不是你的手机！

---

#### 4.4 与 Build Container 的交互机制

**通信协议**: Docker Remote API (REST over TCP)

**交互模式**:

```
┌─────────────────────────────────────────────────────┐
│                    Job Pod                          │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐   │
│  │ Build Container  │      │  DinD Container  │   │
│  │                  │      │                  │   │
│  │  docker CLI      │      │    dockerd       │   │
│  │  ├── version     │      │    └── API       │   │
│  │  ├── build       │      │        Server    │   │
│  │  ├── push        │      │        :2375     │   │
│  │  └── ...         │      │                  │   │
│  │                  │      │                  │   │
│  │  环境变量:        │      │  监听端口:        │   │
│  │  DOCKER_HOST=    │      │  - 2375/tcp      │   │
│  │  tcp://docker:2375     │  - /var/run/     │   │
│  │                  │      │    docker.sock   │   │
│  └────────┬─────────┘      └────────┬─────────┘   │
│           │                         │             │
│           │    HTTP/REST API        │             │
│           │◄──────────────────────►│             │
│           │                         │             │
│  共享网络命名空间 (localhost)                       │
│  /etc/hosts: 127.0.0.1 docker                     │
└─────────────────────────────────────────────────────┘
```

**典型 API 调用示例**:

1. **docker info**
   ```
   Build → DinD: GET /v1.41/info
   DinD → Build: 200 OK
   {
     "ServerVersion": "27.1.1",
     "StorageDriver": "vfs",
     "Containers": 0,
     "Images": 0
   }
   ```

2. **docker build**
   ```
   Build → DinD: POST /v1.41/build?t=myapp:latest
   Content-Type: application/x-tar
   Body: <tar 流>
   
   DinD → Build: 200 OK (流式响应)
   {"stream": "Step 1/5 : FROM python:3.9-slim\n"}
   {"stream": "Pulling from library/python\n"}
   {"status": "Downloading", "progress": "[===>  ] 15MB/50MB"}
   ...
   {"stream": "Successfully built abc123def456\n"}
   {"stream": "Successfully tagged myapp:latest\n"}
   ```

3. **docker push**
   ```
   Build → DinD: POST /v1.41/images/myapp:latest/push
   X-Registry-Auth: <base64_encoded_credentials>
   
   DinD → Build: 200 OK (流式响应)
   {"status": "The push refers to repository [...]"}
   {"status": "Preparing"}
   {"status": "Pushing", "progress": "[=>  ] 10MB/50MB"}
   ...
   {"status": "56cc4792-86: digest: sha256:... size: 1234"}
   ```

#### 4.4 结果反馈机制

**1. 实时日志流**

DinD 将操作日志通过 API 响应流实时返回给 Build Container：

```
Build Container stdout:
  Step 1/5 : FROM python:3.9-slim    ← DinD 返回
  ---> Pulling from library/python   ← DinD 返回
  Layer 1: Downloading [=>  ] 5MB    ← DinD 返回（实时更新）
  Layer 1: Download complete         ← DinD 返回
  Successfully built abc123def456    ← DinD 返回
```

**2. 退出码传递**

```
DinD 执行结果 → HTTP 状态码 → Build Container 解析 → Shell 退出码

成功: 200 OK → exit 0
失败: 500 Internal Server Error → exit 1
```

**3. 镜像 ID 传递**

```
docker build 成功后:
  DinD 生成: Image ID = sha256:abc123...
  ↓
  返回 JSON: {"stream": "Successfully built abc123...\n"}
  ↓
  docker CLI 解析并存储
  ↓
  后续命令可引用: docker push abc123...
```

**4. 数据持久化**（镜像存储）

```
DinD 内部存储:
  /var/lib/docker/
  ├── image/
  │   └── vfs/
  │       └── imagedb/
  │           └── content/
  │               └── sha256/
  │                   └── abc123... (镜像元数据)
  │
  └── vfs/
      └── dir/
          └── abc123... (镜像层数据)

这些数据在 DinD 容器内，Pod 销毁后丢失
已推送到远程仓库的镜像不受影响
```

#### 4.5 网络与 DNS 配置

##### 📡 为什么 DNS 配置是关键？

在 DinD 场景中，DNS 解析是**最常见的故障点**，因为：
1. **dockerd 需要拉取镜像** → 必须能解析 `registry.example.com`
2. **默认 kube-dns 只解析集群内服务** → 无法解析外部域名
3. **--dns 参数容易误解** → 很多人以为它影响 dockerd 自己的 DNS

##### 🔄 DNS 配置的传递机制

```
┌─────────────────────────────────────────────────────────────────┐
│                      配置来源与传递路径                           │
└─────────────────────────────────────────────────────────────────┘

【第 1 层：Runner 配置】gitlab-runner-values.yaml
    │
    │  dns_policy = "none"
    │  [runners.kubernetes.dns_config]
    │    nameservers = ["10.0.0.2", "10.1.0.251", "10.1.0.252"]
    │
    ↓ Runner Manager 创建 Job Pod 时应用此配置

【第 2 层：Job Pod 配置】Kubernetes Pod Spec
    │
    │  spec:
    │    dnsPolicy: None
    │    dnsConfig:
    │      nameservers:
    │        - 10.0.0.2
    │        - 10.1.0.251
    │        - 10.1.0.252
    │      searches: ["gitlab.svc.cluster.local", "svc.cluster.local"]
    │
    ↓ Kubernetes 将此配置写入容器的 /etc/resolv.conf

【第 3 层：容器内部文件】/etc/resolv.conf（所有容器共享）
    │
    │  nameserver 10.0.0.2
    │  nameserver 10.1.0.251
    │  nameserver 10.1.0.252
    │  search gitlab.svc.cluster.local svc.cluster.local cluster.local
    │  options ndots:2
    │
    ↓ Build Container、Helper、DinD 都继承这个配置

【第 4 层：应用使用】
    ├─→ Build Container 的 DNS 查询 → 读取 /etc/resolv.conf
    ├─→ Helper Container 的 DNS 查询 → 读取 /etc/resolv.conf
    └─→ DinD Container 的 dockerd 进程 → 读取 /etc/resolv.conf ✅ 关键！
```

##### ⚠️ 最容易混淆的概念：`--dns` 参数的两层含义

**错误理解**：
```yaml
services:
  - name: docker:dind
    command:
      - "--dns=8.8.8.8"  # ❌ 很多人以为这配置的是 dockerd 自己的 DNS
```

**正确理解**：

```
┌──────────────────────────────────────────────────────────────────┐
│           dockerd 的 DNS 配置 - 两个完全不同的层面                 │
└──────────────────────────────────────────────────────────────────┘

【场景 1：dockerd 进程本身拉取镜像】
  ┌─────────────────────────────────────────────────────────┐
  │ 时机：执行 docker build 时，FROM python:3.9-slim         │
  │                                                         │
  │ dockerd 进程发起 DNS 查询：                               │
  │   域名：registry.example.com                            │
  │   ↓                                                     │
  │   读取：/etc/resolv.conf (容器自己的)                     │
  │   ↓                                                     │
  │   使用：10.0.0.2 (来自 Job Pod 的 dnsConfig)            │
  │   ↓                                                     │
  │   结果：解析到 203.0.113.10                              │
  │                                                         │
  │ 💡 --dns 参数对此无效！                                   │
  └─────────────────────────────────────────────────────────┘

【场景 2：dockerd 创建的容器内部】
  ┌─────────────────────────────────────────────────────────┐
  │ 时机：执行 RUN pip install 时，在临时容器中               │
  │                                                         │
  │ dockerd 创建临时容器，并设置其 /etc/resolv.conf：         │
  │   nameserver 8.8.8.8  ← 来自 --dns=8.8.8.8 参数        │
  │   ↓                                                     │
  │   临时容器内的 pip 命令发起 DNS 查询：                    │
  │   域名：pypi.org                                        │
  │   ↓                                                     │
  │   使用：8.8.8.8                                         │
  │   ↓                                                     │
  │   结果：解析到 PyPI 的 IP                                │
  │                                                         │
  │ 💡 --dns 参数在此生效！                                   │
  └─────────────────────────────────────────────────────────┘
```

##### 📊 对比表：两种 DNS 配置

| 对比项 | Pod dnsConfig | DinD --dns 参数 |
|--------|--------------|----------------|
| **配置位置** | gitlab-runner-values.yaml | .gitlab-ci.yml services.command |
| **影响对象** | DinD 容器的 /etc/resolv.conf | dockerd 创建的临时容器的 /etc/resolv.conf |
| **生效时机** | 容器启动时 | dockerd 执行 RUN 指令时 |
| **影响范围** | **dockerd 进程自己**的 DNS 查询 | dockerd **创建的容器内部**的 DNS 查询 |
| **典型场景** | 拉取镜像（FROM python:3.9） | 容器内安装软件（RUN pip install） |
| **配置示例** | `nameservers: ["10.0.0.2"]` | `command: ["--dns=8.8.8.8"]` |
| **解决的问题** | ✅ 解析镜像仓库域名 | ✅ 容器内部访问外网 |
| **不解决的问题** | ❌ 容器内部的 DNS | ❌ dockerd 拉取镜像的 DNS |

##### 🎯 实际场景演示

**场景 1：docker build 拉取基础镜像**
```bash
# .gitlab-ci.yml 中执行
docker build -t myapp .

# Dockerfile 内容
FROM registry.example.com/python:3.9-slim
RUN pip install flask

# DNS 解析流程
┌─────────────────────────────────────────────────────────┐
│ Step 1: 拉取 FROM 指定的基础镜像                          │
│                                                         │
│ dockerd 进程需要解析：                                    │
│   registry.example.com                                 │
│   ↓                                                     │
│ dockerd 读取自己的 /etc/resolv.conf：                    │
│   nameserver 10.0.0.2  ← 来自 Pod dnsConfig            │
│   ↓                                                     │
│ 查询结果：203.0.113.10                                   │
│ ✅ 成功拉取镜像                                           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Step 2: 在临时容器中执行 RUN 指令                         │
│                                                         │
│ dockerd 创建临时容器执行：pip install flask              │
│                                                         │
│ 临时容器需要解析：pypi.org                               │
│   ↓                                                     │
│ dockerd 为临时容器设置 /etc/resolv.conf：                │
│   如果有 --dns=8.8.8.8 → nameserver 8.8.8.8           │
│   如果没有 --dns → 继承 DinD 容器的配置                   │
│   ↓                                                     │
│ 查询 pypi.org 并下载 flask                              │
│ ✅ 安装成功                                              │
└─────────────────────────────────────────────────────────┘
```

##### 🛠️ 正确配置示例

**本项目的完整配置**：

```yaml
# ============================================
# 文件 1: gitlab-runner-values.yaml
# ============================================
runners:
  config: |
    [[runners]]
      executor = "kubernetes"
      [runners.kubernetes]
        # 🔑 关键配置：让 DinD 容器能拉取镜像
        dns_policy = "none"
        [runners.kubernetes.dns_config]
          nameservers = ["10.0.0.2", "10.1.0.251", "10.1.0.252"]
          searches = ["gitlab.svc.cluster.local", "svc.cluster.local", "cluster.local"]
          [[runners.kubernetes.dns_config.options]]
            name = "ndots"
            value = "2"

# ============================================
# 文件 2: .gitlab-ci.yml
# ============================================
services:
  - name: registry.example.com/docker:dind
    alias: docker
    command:
      - "--tls=false"
      - "--insecure-registry=private-registry.example.com"
      # 注意：没有 --dns 参数，因为：
      # 1. dockerd 拉取镜像已通过 Pod dnsConfig 解决
      # 2. RUN 指令内的网络访问会继承 DinD 容器的 DNS (10.0.0.2)
      # 3. 如果 RUN 指令需要特殊 DNS，可以添加 --dns=8.8.8.8
```

##### 🔍 验证 DNS 配置的方法

**1. 检查 DinD 容器的 DNS 配置**
```bash
# 查看 DinD 容器的 /etc/resolv.conf
kubectl exec -n gitlab <job-pod-name> -c docker -- cat /etc/resolv.conf

# 预期输出：
nameserver 10.0.0.2
nameserver 10.1.0.251
nameserver 10.1.0.252
search gitlab.svc.cluster.local svc.cluster.local cluster.local
options ndots:2
```

**2. 测试 DinD 容器能否解析镜像仓库**
```bash
# 在 DinD 容器内测试 DNS 解析
kubectl exec -n gitlab <job-pod-name> -c docker -- \
  nslookup registry.example.com

# 预期输出：
Server:    10.0.0.2
Address 1: 10.0.0.2

Name:      registry.example.com
Address 1: 203.0.113.10
```

**3. 测试 dockerd 能否拉取镜像**
```bash
# 在 Build Container 中执行
kubectl exec -n gitlab <job-pod-name> -c build -- \
  docker -H tcp://docker:2375 pull alpine:latest

# 如果成功，说明 DNS 配置正确
```

##### ❌ 常见错误配置

**错误 1：只配置 --dns 参数，不配置 Pod dnsConfig**
```yaml
# ❌ 错误配置
services:
  - name: docker:dind
    command:
      - "--dns=8.8.8.8"  # 这个不影响 dockerd 拉取镜像

# 结果：
# FROM python:3.9 → DNS 解析失败
# 因为 dockerd 读取的是 /etc/resolv.conf（默认指向 kube-dns）
```

**错误 2：在 .gitlab-ci.yml 中尝试配置 Pod DNS**
```yaml
# ❌ 错误配置
variables:
  KUBERNETES_DNS_CONFIG: "10.0.0.2"  # 无效，CI 变量不控制 Pod DNS

# 正确做法：
# 必须在 gitlab-runner-values.yaml 的 runners.kubernetes.dns_config 中配置
```

**错误 3：使用 kube-dns 却期望解析外部域名**
```yaml
# ❌ 错误配置
# gitlab-runner-values.yaml 中没有配置 dns_policy 和 dns_config
# 默认使用 kube-dns (10.96.0.10)

# 结果：
# kube-dns 只能解析集群内服务（如 gitlab.svc.cluster.local）
# 无法解析 registry.example.com
```

##### 🚨 故障排查步骤

当遇到 DNS 相关错误时（如 `dial tcp: lookup xxx: i/o timeout`），按以下顺序检查：

**1. 确认 Pod 的 DNS 配置**
```bash
kubectl get pod <job-pod-name> -n gitlab -o yaml | grep -A 10 dnsPolicy

# 预期：
# dnsPolicy: None
# dnsConfig:
#   nameservers:
#   - 10.0.0.2
```

**2. 确认 DinD 容器的 resolv.conf**
```bash
kubectl exec -n gitlab <job-pod-name> -c docker -- cat /etc/resolv.conf

# 检查 nameserver 是否是 10.0.0.2 而不是 10.96.0.10
```

**3. 测试 DNS 服务器可达性**
```bash
# 从 DinD 容器内 ping DNS 服务器
kubectl exec -n gitlab <job-pod-name> -c docker -- ping -c 3 10.0.0.2

# 如果不通，检查网络策略或防火墙
```

**4. 测试域名解析**
```bash
# 使用指定 DNS 服务器解析
kubectl exec -n gitlab <job-pod-name> -c docker -- \
  nslookup registry.example.com 10.0.0.2

# 如果失败，可能是 DNS 服务器配置问题
```

**5. 检查 Runner 配置是否生效**
```bash
# 查看 Runner Manager 的配置
kubectl exec -n gitlab <runner-manager-pod> -- cat /home/gitlab-runner/.gitlab-runner/config.toml

# 确认 dns_policy 和 dns_config 存在
```

##### 📝 配置总结

| 需求场景 | 配置方法 | 配置位置 |
|---------|---------|---------|
| **dockerd 拉取镜像** | 配置 Pod `dnsConfig` | `gitlab-runner-values.yaml`<br>`runners.kubernetes.dns_config` |
| **RUN 指令内访问外网** | 方案1: 继承 Pod DNS（推荐）<br>方案2: 添加 `--dns` 参数 | 不配置（默认继承）<br>或 `.gitlab-ci.yml` services.command |
| **Build Container 访问外网** | 配置 Pod `dnsConfig`（自动生效） | `gitlab-runner-values.yaml` |
| **Helper Container 克隆代码** | 配置 Pod `dnsConfig`（自动生效） | `gitlab-runner-values.yaml` |

**核心原则**：
- 🎯 **Pod 级别的 DNS 配置影响所有容器**（包括 dockerd 进程）
- 🎯 **--dns 参数只影响 dockerd 创建的容器内部**
- 🎯 **优先配置 Pod dnsConfig，--dns 按需配置**

---

#### 4.6 资源消耗分析

**启动阶段** (T+4s → T+8s):
- CPU: ~500m（启动 dockerd + containerd）
- 内存: ~200Mi（加载服务）

**空闲阶段** (T+8s → T+10s):
- CPU: ~50m（守护进程等待）
- 内存: ~250Mi（基础占用）

**构建阶段** (T+10s → T+120s):
- CPU: 1.5-2 核（拉取镜像 + 构建）
- 内存: 1-2Gi（镜像层缓存 + 构建缓存）
- 磁盘 I/O: 中等（下载镜像层）

**推送阶段** (T+120s → T+160s):
- CPU: ~1 核（压缩上传）
- 内存: ~500Mi（读取镜像层）
- 网络 I/O: 高（上传数据）

**关键配置**:
```yaml
# Runner 配置
service_cpu_limit = "2"          # 防止 CPU 过载
service_cpu_request = "500m"     # 保证启动资源
service_memory_limit = "2Gi"     # 防止 OOM
service_memory_request = "512Mi" # 保证基础内存

# 特权模式（必需）
privileged = true  # DinD 需要内核功能访问
```

#### 4.7 常见场景与其他 Service

**数据库服务示例** (MySQL):
```yaml
services:
  - name: mysql:8.0
    alias: mysql
    variables:
      MYSQL_ROOT_PASSWORD: "test123"
      MYSQL_DATABASE: "testdb"

# Build Container 中:
# - 等待 MySQL 启动（约 15s）
# - 连接: mysql -h mysql -u root -ptest123
# - 执行测试: pytest tests/integration/
# - 结果反馈: 测试通过/失败 → 退出码
```

**Redis 缓存服务**:
```yaml
services:
  - name: redis:7-alpine
    alias: redis

# Build Container 中:
# - 等待 Redis 启动（约 2s）
# - 连接: redis-cli -h redis ping → PONG
# - 运行测试: npm test
# - 结果反馈: 测试日志 → CI 报告
```

**Selenium UI 测试**:
```yaml
services:
  - name: selenium/standalone-chrome:latest
    alias: selenium

# Build Container 中:
# - 等待 Selenium 启动（约 10s）
# - 连接: WebDriver(command_executor='http://selenium:4444')
# - 执行 UI 测试
# - 结果反馈: 截图 + 测试报告 → Artifacts
```

---

## 二、容器间交互流程

### 阶段 1：任务接收与 Pod 创建

```
┌─────────────────────┐
│  GitLab Server      │
│  (gitlab.example.com)│
└──────────┬──────────┘
           │ ① 推送代码触发 Pipeline
           ↓
┌─────────────────────┐
│ Runner Manager Pod  │
│ (轮询任务)           │
└──────────┬──────────┘
           │ ② 获取到 build-job 任务
           ↓
    创建 Job Pod
```

### 阶段 2：Job Pod 内部容器启动顺序

```
Job Pod 创建
    │
    ├─→ ③ Helper Container (init)
    │      - 克隆代码到 /builds/
    │      - 下载 artifacts
    │      - 完成后进入等待
    │
    ├─→ ④ Service Container (DinD)
    │      - 启动 dockerd 守护进程
    │      - 监听 tcp://0.0.0.0:2375
    │
    └─→ ⑤ Build Container
           - 等待 DinD 启动 (before_script)
           - 执行 docker build
           - 执行 docker push
```

### 阶段 3：Build 阶段的网络交互

```
┌──────────────────────────────────────────┐
│             Job Pod                       │
│  ┌────────────────┐   ┌────────────────┐ │
│  │ Build Container│   │ DinD Container │ │
│  │  (docker CLI)  │   │   (dockerd)    │ │
│  └────────┬───────┘   └────────┬───────┘ │
│           │                    │         │
│           │ ⑥ DOCKER_HOST=     │         │
│           │   tcp://docker:2375│         │
│           └────────────────────┘         │
│                    │                     │
│                    │ ⑦ dockerd 拉取镜像   │
│                    │   需要 DNS 解析      │
│                    ↓                     │
│           DNS: 10.255.9.2                │
│                (节点内网 DNS)              │
└──────────────────────────────────────────┘
                     │
                     ↓
        ⑧ 拉取 Python 基础镜像
   swr.cn-north-4.myhuaweicloud.com
```

### 阶段 4：构建产物上传

```
Build Container
    │ ⑨ docker push 完成
    ↓
Helper Container (重新激活)
    │ ⑩ 上传 artifacts (如果有)
    │ ⑪ 上传 cache (如果有)
    ↓
GitLab Server
    │ ⑫ 存储构建产物
    │ ⑬ 更新 Pipeline 状态
```

---

## 三、关键配置说明

### 3.1 DNS 配置（解决 DinD 无法解析域名的问题）

**问题**:
- 默认 Job Pod 使用 kube-dns (10.96.0.10)
- kube-dns 无法解析外部域名 `registry.example.com`
- 导致 `docker build` 时拉取基础镜像失败

**解决方案**:
在 `gitlab-runner-values.yaml` 中配置使用自定义 DNS：

```toml
dns_policy = "none"
[runners.kubernetes.dns_config]
  nameservers = ["10.0.0.2", "10.1.0.251", "10.1.0.252"]
  searches = ["gitlab.svc.cluster.local", "svc.cluster.local", "cluster.local"]
  [[runners.kubernetes.dns_config.options]]
    name = "ndots"
    value = "2"
```

**效果**:
- Job Pod 内所有容器（Build、Helper、Service）都使用自定义 DNS
- DinD 内部的 dockerd 可以正确解析镜像仓库域名

### 3.2 Service Alias（容器间通信）

**配置**:
```yaml
services:
  - name: registry.example.com/docker:dind
    alias: docker  # 定义别名
```

**作用**:
- Kubernetes 在 Pod 的 `/etc/hosts` 中添加：`127.0.0.1 docker`
- Build Container 可以通过 `docker` 主机名访问 DinD
- 配合 `DOCKER_HOST=tcp://docker:2375` 使用

**如果没有 alias**:
- 主机名会是完整镜像路径：`registry.example.com-docker`
- 导致 DNS 解析失败

### 3.3 DinD 特权模式

**配置**:
```yaml
privileged = true  # Runner 配置中启用
```

**原因**:
- DinD 需要访问内核功能（如 cgroups、namespaces）
- 需要挂载 `/sys/fs/cgroup` 等系统目录
- 需要创建和管理容器

**安全注意**:
- 特权容器可以访问宿主机内核
- 仅在可信环境中使用
- 生产环境建议使用 Kaniko 或 Buildah 替代 DinD

---

## 四、资源配额与并发控制

### 4.1 Runner Manager 级别

```yaml
# 3 个 Manager Pod，每个最多 10 并发
replicas: 3
concurrent: 10

# 总并发能力 = 3 × 10 = 30 个 job
```

### 4.2 Job Pod 资源配额

```toml
# Build Container
cpu_limit = "2"
memory_limit = "2Gi"

# Service Container (DinD)
service_cpu_limit = "2"
service_memory_limit = "2Gi"

# Helper Container
helper_cpu_limit = "500m"
helper_memory_limit = "512Mi"
```

**单个 Job Pod 总资源需求**:
- CPU: 2 (build) + 2 (dind) + 0.5 (helper) = **4.5 核**
- 内存: 2Gi + 2Gi + 0.5Gi = **4.5Gi**

---

## 五、故障排查命令

### 查看 Runner Manager 日志
```bash
kubectl logs -n gitlab -l app=gitlab-runner -f
```

### 查看 Job Pod 状态
```bash
kubectl get pods -n gitlab | grep runner-.*concurrent
```

### 查看 Job Pod 各容器日志
```bash
# Build Container
kubectl logs -n gitlab <pod-name> -c build

# Helper Container
kubectl logs -n gitlab <pod-name> -c helper

# Service Container (DinD)
kubectl logs -n gitlab <pod-name> -c docker
```

### 检查 DNS 配置
```bash
kubectl exec -n gitlab <pod-name> -c build -- cat /etc/resolv.conf
```

### 测试 DinD 连接
```bash
kubectl exec -n gitlab <pod-name> -c build -- docker -H tcp://docker:2375 info
```

---

## 六、总结

| 容器类型 | 镜像 | 运行时长 | 主要职责 |
|---------|------|---------|---------|
| **Runner Manager** | gitlab-runner:v18.3.0 | 持续运行 | 任务调度、Pod 创建 |
| **Helper** | runner-helper:v18.3.0 | Job 期间 | 代码克隆、产物上传 |
| **Build** | 用户指定 (如 docker:latest) | Job 期间 | 执行 CI/CD 脚本 |
| **Service** | 用户指定 (如 dind) | Job 期间 | 提供辅助服务 |

**关键交互**:
1. Manager 轮询 GitLab → 创建 Job Pod
2. Helper 克隆代码 → Build 开始执行
3. Build 通过 alias 访问 Service (DinD)
4. Build 完成 → Helper 上传产物 → 向 GitLab 汇报
5. Job Pod 销毁 → Manager 继续轮询下一个任务

**网络要点**:
- Job Pod 内所有容器共享网络命名空间
- 通过 DNS 配置确保 DinD 能访问镜像仓库
- Service alias 简化容器间通信
