# GitLab 在 Kubernetes 上的完整部署指南

## 目录
1. [部署架构概述](#1-部署架构概述)
2. [GitLab镜像选择](#2-gitlab镜像选择)
3. [All-in-One部署方案](#3-all-in-one部署方案)
4. [持久化存储配置](#4-持久化存储配置)
5. [GitLab Runner部署](#5-gitlab-runner部署)
6. [Runner工作原理详解](#6-runner工作原理详解)
7. [CI/CD全流程运行机制](#7-cicd全流程运行机制)
8. [Python代码评测系统](#8-python代码评测系统)
9. [GitLab CI/CD配置详解](#9-gitlab-cicd配置详解)
10. [安全和最佳实践](#10-安全和最佳实践)
11. [故障排查指南](#11-故障排查指南)

---

## 1. 部署架构概述

### 1.1 整体架构设计

```
┌─────────────────────────────────────────────────────────┐
│                 阿里云 Kubernetes 集群                    │
│                                                         │
│  ┌──────────────── Control Plane ──────────────────┐  │
│  │      3 × Master Nodes (管理节点)                 │  │
│  └──────────────────────────────────────────────────┘  │
│                           ↓                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │                 Worker Nodes                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │  │
│  │  │   Node 1    │  │   Node 2    │  │  Node 3  │ │  │
│  │  │             │  │             │  │          │ │  │
│  │  │ GitLab Pod  │  │Runner Pods  │  │ 存储节点  │ │  │
│  │  │ (All-in-One)│  │ (CI/CD)     │  │ (NAS)    │ │  │
│  │  └─────────────┘  └─────────────┘  └──────────┘ │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 组件分布

| 组件 | 部署方式 | 节点分布 | 资源需求 |
|-----|---------|---------|---------|
| **GitLab CE** | StatefulSet | 单节点部署 | 4核/8GB |
| **GitLab Runner** | Deployment | 多节点分布 | 1核/2GB×3 |
| **存储系统** | NAS/云盘 | 共享存储 | 100GB+ |
| **负载均衡** | Ingress/SLB | 边缘节点 | - |

---

## 2. GitLab镜像选择

### 2.1 推荐版本配置

```yaml
# 核心组件版本选择
GitLab CE版本: gitlab/gitlab-ce:16.6.2-ce.0
PostgreSQL版本: postgres:13.12-alpine
Redis版本: redis:7.0-alpine
Runner版本: gitlab/gitlab-runner:alpine-v16.6.1
```

### 2.2 镜像源选择（针对阿里云）

```yaml
# 方式1：官方镜像
image: gitlab/gitlab-ce:16.6.2-ce.0

# 方式2：阿里云镜像加速（推荐）
image: registry.cn-hangzhou.aliyuncs.com/gitlab-cn/gitlab-ce:16.6.2-ce.0

# 方式3：极狐GitLab（国内版本）
image: registry.gitlab.cn/omnibus/gitlab-jh:16.6.2-jh.0
```

### 2.3 版本选择建议

| 环境类型 | 推荐版本 | 特点 | 适用场景 |
|---------|---------|------|---------|
| **生产环境** | 16.6.x (N-1版本) | 稳定性高 | 企业级应用 |
| **测试环境** | 16.7.x (最新版) | 新功能体验 | 功能验证 |
| **开发环境** | 16.6.x | 与生产一致 | 开发调试 |

---

## 3. All-in-One部署方案

### 3.1 方案说明

**All-in-One优势：**
- 部署简单，配置统一
- 管理方便，单点维护
- 资源占用相对较少

**适用场景：**
- 中小型团队（<50用户）
- 开发/测试环境
- 快速部署需求

### 3.2 包含的组件

GitLab CE的Omnibus镜像包含：

✅ **已包含组件：**
- GitLab核心应用（Rails）
- 内置PostgreSQL数据库
- 内置Redis缓存
- Nginx Web服务器
- Sidekiq后台任务处理
- GitLab Shell（SSH访问）
- Prometheus监控
- Grafana仪表盘
- GitLab Pages
- Container Registry

❌ **需要单独部署：**
- GitLab Runner（CI/CD执行器）

### 3.3 基础配置

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab
  labels:
    name: gitlab
```

---

## 4. 持久化存储配置

### 4.1 存储需求分析

| 目录路径 | 用途 | 重要性 | 推荐大小 |
|---------|------|--------|---------|
| `/etc/gitlab` | 配置文件、密钥 | ⭐⭐⭐ | 10GB |
| `/var/opt/gitlab` | 应用数据、数据库、Git仓库 | ⭐⭐⭐ | 100GB+ |
| `/var/log/gitlab` | 日志文件 | ⭐⭐ | 20GB |

### 4.2 阿里云存储选择

**推荐：阿里云NAS**
```yaml
优势：
- 支持多节点同时读写 (ReadWriteMany)
- 性能稳定，自动扩容
- 适合GitLab的文件存储需求
- 支持快照备份

配置示例：
storageClassName: alicloud-nas
accessModes: [ReadWriteMany]
```

**备选：阿里云云盘**
```yaml
限制：
- 只支持单节点挂载 (ReadWriteOnce)
- Pod迁移需要手动处理
- 不适合有状态应用

适用场景：
- 测试环境
- 单节点部署
```

### 4.3 存储配置文件

```yaml
# gitlab-storage.yaml
---
# NAS存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gitlab-nas-storage
provisioner: nasplugin.csi.alibabacloud.com
parameters:
  server: "your-nas-id.cn-hangzhou.nas.aliyuncs.com"
  path: "/gitlab"
  vers: "3"
  options: "noresvport,nolock"
reclaimPolicy: Retain
allowVolumeExpansion: true

---
# GitLab数据存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-data-pvc
  namespace: gitlab
spec:
  accessModes: [ReadWriteMany]
  storageClassName: gitlab-nas-storage
  resources:
    requests:
      storage: 100Gi

---
# GitLab配置存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-config-pvc
  namespace: gitlab
spec:
  accessModes: [ReadWriteMany]
  storageClassName: gitlab-nas-storage
  resources:
    requests:
      storage: 10Gi

---
# GitLab日志存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-logs-pvc
  namespace: gitlab
spec:
  accessModes: [ReadWriteMany]
  storageClassName: gitlab-nas-storage
  resources:
    requests:
      storage: 20Gi
```

---

## 5. GitLab Runner部署

### 5.1 Runner类型选择

| 执行器类型 | 适用场景 | 优缺点 | 推荐度 |
|-----------|---------|--------|--------|
| **Kubernetes执行器** | 云原生应用 | ✅自动扩缩容 ✅资源管理 ❌配置复杂 | ⭐⭐⭐⭐⭐ |
| **Docker执行器** | 通用CI/CD | ✅简单 ✅隔离性好 ❌需要DinD | ⭐⭐⭐⭐ |
| **Shell执行器** | 简单脚本 | ✅最简单 ❌无隔离 ❌安全性差 | ⭐⭐ |

### 5.2 RBAC权限配置

```yaml
# gitlab-runner-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-runner
  namespace: gitlab

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: gitlab
rules:
  # Pod管理权限
  - apiGroups: [""]
    resources: ["pods", "pods/exec", "pods/log"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # 配置管理权限
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "create", "delete"]
  # 存储权限
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner
  namespace: gitlab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gitlab-runner
subjects:
  - kind: ServiceAccount
    name: gitlab-runner
    namespace: gitlab
```

### 5.3 Runner配置部署

```yaml
# gitlab-runner-config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-runner-config
  namespace: gitlab
data:
  config.toml: |
    concurrent = 10
    check_interval = 3
    
    [[runners]]
      name = "kubernetes-runner"
      url = "http://gitlab-service.gitlab.svc.cluster.local"
      token = "YOUR_REGISTRATION_TOKEN"
      executor = "kubernetes"
      
      [runners.kubernetes]
        namespace = "gitlab"
        image = "alpine:latest"
        privileged = true
        
        # 资源限制
        cpu_request = "100m"
        cpu_limit = "1"
        memory_request = "128Mi"
        memory_limit = "1Gi"
        
        # 服务账号
        service_account = "gitlab-runner"
        
        # 缓存配置
        [[runners.kubernetes.volumes.pvc]]
          name = "cache"
          mount_path = "/cache"
          claim_name = "runner-cache-pvc"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-runner
  namespace: gitlab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gitlab-runner
  template:
    metadata:
      labels:
        app: gitlab-runner
    spec:
      serviceAccountName: gitlab-runner
      containers:
      - name: runner
        image: gitlab/gitlab-runner:alpine-v16.6.1
        command: ["gitlab-runner", "run"]
        volumeMounts:
        - name: config
          mountPath: /etc/gitlab-runner
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 1Gi
      volumes:
      - name: config
        configMap:
          name: gitlab-runner-config
```

---

## 6. Runner工作原理详解

### 6.1 Runner与GitLab通信流程

```
┌─────────────────────────────────────────────┐
│                GitLab Server                 │
│  ┌─────────────────────────────────────┐    │
│  │     GitLab API (Jobs Queue)         │    │
│  │  /api/v4/jobs/request (轮询)        │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
                      ▲ ▼
              1.轮询  │ │  2.获取Job
              (POST)  │ │  (Response)
                      │ │
┌─────────────────────────────────────────────┐
│               GitLab Runner                  │
│  ┌─────────────────────────────────────┐    │
│  │  Runner进程 (长轮询 GitLab API)     │    │
│  └─────────────────────────────────────┘    │
│                     ▼                        │
│  ┌─────────────────────────────────────┐    │
│  │     Kubernetes执行器                │    │
│  │  创建Pod → 执行脚本 → 清理Pod        │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### 6.2 Pod创建和执行流程

```yaml
# 步骤1: Runner接收Job
Job信息:
  id: 1234
  image: "python:3.9"
  script: ["pytest tests/"]
  variables: {...}

# 步骤2: 创建Pod定义
apiVersion: v1
kind: Pod
metadata:
  name: runner-{token}-project-{id}-concurrent-{n}
  namespace: gitlab
spec:
  restartPolicy: Never
  containers:
  - name: build
    image: python:3.9
    command: ["/bin/bash", "-c"]
    args: ["pytest tests/"]
    env:
    - name: CI_JOB_ID
      value: "1234"

# 步骤3: 监控Pod状态
Pending → ContainerCreating → Running → Succeeded/Failed

# 步骤4: 收集日志并上报
kubectl logs {pod-name} → GitLab API

# 步骤5: 清理Pod
kubectl delete pod {pod-name}
```

### 6.3 轮询机制详解

```go
// Runner核心轮询逻辑（伪代码）
for {
    // 每3秒请求一次GitLab
    response = HTTP.POST("/api/v4/jobs/request", {
        token: RUNNER_TOKEN,
        executor: "kubernetes",
        features: {"variable": true, "image": true}
    })
    
    if response.hasJob() {
        job = response.getJob()
        
        // 在K8s中创建Pod执行Job
        pod := createPod(job)
        executeJob(pod, job)
        uploadLogs(job.id)
        updateJobStatus(job.id, result)
        cleanupPod(pod)
    }
    
    sleep(3) // 等待3秒继续轮询
}
```

---

## 7. CI/CD全流程运行机制

### 7.1 整体架构运行流程图

```
开发者推送代码
      ↓
┌─────────────────────────────────────────────────────────────────┐
│                         GitLab Server                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Git Repo      │  │  Pipeline       │  │  Artifacts      │ │
│  │   - main.py     │  │  Engine         │  │  Storage        │ │
│  │   - .gitlab-    │  │  - Job Queue    │  │  - build/       │ │
│  │     ci.yml      │  │  - Status       │  │  - reports/     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
      ↓ 检测.gitlab-ci.yml        ↓ 轮询获取Job        ↑ 上传结果
      ↓ 创建Pipeline            ↓                    ↑
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 GitLab Runner Pod                       │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │  gitlab-runner:alpine-v16.6.1                  │   │   │
│  │  │  - API轮询进程                                  │   │   │
│  │  │  - Pod管理器                                    │   │   │
│  │  │  - 日志收集器                                   │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  创建临时Job Pods ↓                                            │
│                                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐ │
│  │ validate   │  │   build    │  │    test    │  │  deploy  │ │
│  │ Job Pod    │  │  Job Pod   │  │  Job Pod   │  │ Job Pod  │ │
│  │python:3.9  │  │ node:16    │  │ python:3.9 │  │kubectl   │ │
│  │            │  │            │  │            │  │          │ │
│  │运行状态:    │  │运行状态:    │  │运行状态:    │  │运行状态: │ │
│  │✅ Success  │  │✅ Success  │  │❌ Failed   │  │⏸ Skipped │ │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Pipeline状态变化时序图

```
时间线 →
GitLab   │ Runner  │ Pod1      │ Pod2      │ Pod3      │ 状态
         │         │(validate) │(build)    │(test)     │
─────────┼─────────┼───────────┼───────────┼───────────┼──────────
创建     │         │           │           │           │ created
Pipeline │         │           │           │           │
         │         │           │           │           │
启动     │ 轮询    │           │           │           │ pending
Pipeline │ API     │           │           │           │
         │         │           │           │           │
分配Job1 │ ←───────│ 创建Pod   │           │           │ running
         │ 获取Job │           │           │           │
         │         │           │           │           │
Job1执行 │         │ 运行中... │           │           │ running
         │         │ ✅完成    │           │           │
         │         │ 上传结果  │           │           │
         │         │           │           │           │
分配Job2 │ ←───────│ 销毁Pod   │ 创建Pod   │           │ running  
         │ 获取Job │           │           │           │
         │         │           │           │           │
Job2执行 │         │           │ 运行中... │           │ running
         │         │           │ ✅完成    │           │
         │         │           │ 上传结果  │           │
         │         │           │           │           │
分配Job3 │ ←───────│           │ 销毁Pod   │ 创建Pod   │ running
         │ 获取Job │           │           │           │
         │         │           │           │           │
Job3执行 │         │           │           │ 运行中... │ running
         │         │           │           │ ❌失败    │
         │         │           │           │ 上传日志  │
         │         │           │           │           │
Pipeline │         │           │           │ 销毁Pod   │ failed
完成     │         │           │           │           │
```

### 7.3 镜像使用与数据流动机制

#### 7.3.1 双层镜像架构

```yaml
# 两个独立的镜像概念
Runner镜像 (管理层)
└── gitlab/gitlab-runner:alpine-v16.6.1
    └── 职责：API轮询、Pod管理、日志上传

Job镜像 (执行层)  
└── python:3.9 / node:16 / 自定义镜像
    └── 职责：实际执行CI/CD任务
```

#### 7.3.2 数据流转详细机制

```
┌──────────────────────────────────────────────────────────────────┐
│                     数据流转全流程                                │
└──────────────────────────────────────────────────────────────────┘

Stage 1: validate
┌─────────────────┐
│ validate Job    │  输入: Git仓库代码
│ python:3.9      │  ├── main.py
│                 │  ├── requirements.txt  
│ 执行:           │  └── .gitlab-ci.yml
│ - 语法检查      │
│ - 代码风格检查  │  输出: 无artifacts
│                 │  状态: ✅ Success
└─────────────────┘

           ↓ (无数据传递，仅状态)

Stage 2: build  
┌─────────────────┐
│ build Job       │  输入: Git仓库代码 (自动下载)
│ node:16         │  ├── package.json
│                 │  ├── src/
│ 执行:           │  └── webpack.config.js
│ - npm install   │
│ - npm run build │  输出: artifacts
│                 │  ├── dist/bundle.js
│                 │  ├── dist/index.html
│                 │  └── package-lock.json
└─────────────────┘
           ↓
    ┌─────────────┐
    │ GitLab      │ ← 上传artifacts
    │ Artifacts   │
    │ Storage     │
    └─────────────┘
           ↓
Stage 3: test (并行执行多个Job)
┌─────────────────┐         ┌─────────────────┐
│ unit-test Job   │         │ e2e-test Job    │
│ python:3.9      │         │ cypress:12.0.0  │
│                 │         │                 │
│ 输入:           │         │ 输入:           │
│ - Git代码       │         │ - Git代码       │
│ - build产物     │ ←───────┤ - build产物     │
│   (自动下载)    │         │   (自动下载)    │
│                 │         │                 │
│ 执行:           │         │ 执行:           │
│ - pytest tests/│         │ - cypress run   │
│                 │         │                 │
│ 输出:           │         │ 输出:           │
│ - test报告      │         │ - 截图/视频     │
│ - 覆盖率报告    │         │ - 测试报告      │
│                 │         │                 │
│ 状态: ✅Success │         │ 状态: ❌Failed  │
└─────────────────┘         └─────────────────┘
           ↓                         ↓
    ┌─────────────┐         ┌─────────────┐
    │ 上传测试    │         │ 上传失败    │
    │ 报告        │         │ 信息        │
    └─────────────┘         └─────────────┘

Stage 4: deploy (因为test失败而跳过)
┌─────────────────┐
│ deploy Job      │  状态: ⏸ Skipped
│ kubectl:latest  │  原因: 前置Job失败
│                 │
│ 本应执行:       │
│ - 构建Docker镜像│
│ - 推送到仓库    │
│ - 部署到K8s     │
└─────────────────┘
```

### 7.4 Pod生命周期状态图

```
每个Job Pod的完整生命周期:

Runner轮询到Job
       ↓
┌─────────────────┐
│ Pod创建阶段     │ 状态: Pending
│ ┌─────────────┐ │ ├── 调度到Node
│ │ API调用     │ │ ├── 拉取镜像  
│ │ K8s创建Pod  │ │ └── 创建容器
│ └─────────────┘ │
└─────────────────┘
       ↓
┌─────────────────┐
│ Pod运行阶段     │ 状态: Running
│ ┌─────────────┐ │ ├── 下载artifacts
│ │ 初始化容器  │ │ ├── 设置环境变量
│ │ 下载依赖    │ │ ├── 执行script
│ │ 执行脚本    │ │ ├── 实时上传日志
│ │ 收集结果    │ │ └── 上传artifacts
│ └─────────────┘ │
└─────────────────┘
       ↓
┌─────────────────┐
│ Pod完成阶段     │ 状态: Succeeded/Failed
│ ┌─────────────┐ │ ├── 容器退出
│ │ 上传日志    │ │ ├── 收集退出码
│ │ 清理资源    │ │ ├── 更新Job状态
│ │ 销毁Pod     │ │ └── 释放资源
│ └─────────────┘ │
└─────────────────┘
       ↓
回到Runner继续轮询下一个Job
```

### 7.5 存储和缓存机制

```
┌──────────────────────────────────────────────────────────────────┐
│                    存储和缓存机制                                │
└──────────────────────────────────────────────────────────────────┘

Git仓库 ────────────────┐
                        ↓
        ┌─────────────────────────────────┐
        │         每个Job Pod              │
        │  ┌─────────────────────────────┐ │
        │  │ 1. 下载源代码 (git clone)   │ │
        │  │ 2. 恢复cache (如果有)       │ │  ← Cache存储
        │  │ 3. 下载artifacts (如果需要) │ │  ← Artifacts存储
        │  │ 4. 执行Job脚本              │ │
        │  │ 5. 上传cache (如果配置)     │ │  → Cache存储
        │  │ 6. 上传artifacts (如果配置) │ │  → Artifacts存储
        │  └─────────────────────────────┘ │
        └─────────────────────────────────┘

存储层级:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Git仓库     │  │ Cache存储   │  │ Artifacts   │
│ (源代码)    │  │ (加速构建)  │  │ (Job产物)   │
│             │  │             │  │             │
│ - 持久存储  │  │ - 临时存储  │  │ - 临时存储  │
│ - 版本控制  │  │ - Runner级别│  │ - Pipeline级别│
│ - 所有Job   │  │ - 相同key   │  │ - Job间传递 │
│   都能访问  │  │   共享缓存  │  │             │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 7.6 并行执行流程图

```
Pipeline并行执行示例:

时间 →
     │ Stage 1 │    Stage 2     │       Stage 3       │ Stage 4 │
     │         │                │                     │         │
Job1 │validate │                │                     │         │
     │ ✅      │                │                     │         │
     └─────────┼────────────────┼─────────────────────┼─────────┘
               │                │                     │
Job2           │ build          │                     │
               │ ✅             │                     │
               └────────────────┼─────────────────────┼─────────┐
                                │                     │         │
Job3                            │ unit-test           │         │
                                │ ✅                  │         │
                                ├─────────────────────┼─────────┤
                                │                     │         │
Job4                            │ lint-test           │         │
                                │ ✅                  │         │
                                ├─────────────────────┼─────────┤
                                │                     │         │
Job5                            │ e2e-test            │         │
                                │ ❌                  │         │
                                └─────────────────────┼─────────┤
                                                      │         │
Job6                                                  │ deploy  │
                                                      │ ⏸Skip   │
                                                      └─────────┘

数据依赖关系:
validate → build → [unit-test, lint-test, e2e-test] → deploy
   ↓        ↓              ↓         ↓         ↓         ↓
  无产物   产生dist/     需要dist/  需要dist/  需要dist/  需要全部成功
```

### 7.7 错误处理和重试流程

```
Job执行失败处理流程:

Job开始执行
     ↓
执行过程中出错
     ↓
┌─────────────────┐
│ 错误类型判断    │
├─────────────────┤
│ • 脚本执行失败  │ → 标记Job Failed
│ • 超时          │ → 标记Job Failed  
│ • 镜像拉取失败  │ → 重试/标记Failed
│ • 资源不足      │ → 等待/重试
│ • 网络问题      │ → 重试
└─────────────────┘
     ↓
┌─────────────────┐
│ 重试策略        │
├─────────────────┤
│ retry:          │
│   max: 2        │ → 最多重试2次
│   when:         │
│   - runner_     │ → 只在系统错误时重试
│     system_     │
│     failure     │
└─────────────────┘
     ↓
重试次数用完或手动标记失败
     ↓
Pipeline状态更新为Failed
```

### 7.8 关键运行原理总结

**执行模型：**
- **1个Runner Pod（常驻）+ N个Job Pod（按需创建销毁）**
- Runner Pod负责管理，Job Pod负责执行
- 每个Job都在独立的Pod中运行，确保隔离性

**状态流转：**
- **created → pending → running → success/failed**
- Pipeline状态由所有Job状态决定
- 失败的Job会影响后续依赖的Job

**数据流动：**
- **通过GitLab服务器中转artifacts，不是Pod直接通信**
- Cache用于加速构建，Artifacts用于Job间数据传递
- Git仓库代码每个Job都会重新下载

**并行机制：**
- **相同stage内的Job可并行，不同stage按依赖顺序执行**
- 通过needs关键字控制精确的依赖关系
- 资源允许的情况下最大化并行执行

**资源管理：**
- **每个Job独立Pod，完成后立即清理**
- 临时存储随Pod销毁，持久数据通过artifacts保存
- Runner Pod复用，减少资源消耗

---

## 8. Python代码评测系统

### 8.1 评测系统架构

```
提交代码 → GitLab → Runner创建Pod → 编译执行 → 评测程序 → 结果输出
    ↓           ↓         ↓           ↓         ↓         ↓
  python     触发CI    隔离环境     运行代码   判分系统   分数/报告
   文件      pipeline    容器       获取输出   比对结果    存储
```

### 7.2 CI/CD Pipeline配置

```yaml
# .gitlab-ci.yml
stages:
  - validate    # 代码验证
  - test        # 运行测试
  - judge       # 评测判分

variables:
  PYTHON_VERSION: "3.9"
  TIMEOUT: "30"
  MEMORY_LIMIT: "512m"

# 代码验证阶段
validate-code:
  stage: validate
  image: python:3.9-alpine
  tags: [kubernetes]
  script:
    - python -m py_compile *.py
    - pip install flake8
    - flake8 --max-line-length=120 *.py

# 运行测试阶段
run-tests:
  stage: test
  image: python:3.9-alpine
  tags: [kubernetes]
  script:
    - ./run_testcases.sh
  artifacts:
    paths: [outputs/, logs/]
    expire_in: 1 day
  parallel:
    matrix:
      - TESTCASE: [case1, case2, case3, case4, case5]

# 评测判分阶段
judge-results:
  stage: judge
  image: python:3.9-alpine
  tags: [kubernetes]
  script:
    - python judge.py
    - python generate_report.py
  artifacts:
    paths: [judge_result.json, report.html]
    reports:
      junit: judge_result.xml
```

### 7.3 安全执行脚本

```bash
#!/bin/bash
# run_testcases.sh - 安全运行Python代码

set -e

PYTHON_FILE=${1:-"solution.py"}
TESTCASE=${TESTCASE:-"case1"}
TIMEOUT=${TIMEOUT:-30}

echo "Running testcase: $TESTCASE"
mkdir -p outputs logs

run_with_limits() {
    local input_file="testcases/${TESTCASE}/input.txt"
    local output_file="outputs/${TESTCASE}_output.txt"
    local log_file="logs/${TESTCASE}_log.txt"
    
    # 资源限制
    (
        ulimit -v 524288    # 512MB内存限制
        ulimit -t $TIMEOUT  # CPU时间限制
        ulimit -f 10240     # 10MB文件大小限制
        
        timeout ${TIMEOUT}s python3 "$PYTHON_FILE" \
            < "$input_file" > "$output_file" 2> "$log_file"
    )
    
    return $?
}

if run_with_limits; then
    echo "✅ Testcase $TESTCASE: SUCCESS"
else
    echo "❌ Testcase $TESTCASE: FAILED"
fi
```

### 7.4 评测程序实现

```python
#!/usr/bin/env python3
# judge.py - 评测判分程序

import os
import json
import difflib
from pathlib import Path

class Judge:
    def __init__(self):
        self.testcases_dir = Path("testcases")
        self.outputs_dir = Path("outputs")
        self.results = []
        
    def compare_output(self, expected_file, actual_file, case_name):
        """比较期望输出和实际输出"""
        try:
            with open(expected_file, 'r') as f:
                expected = f.read().strip()
            with open(actual_file, 'r') as f:
                actual = f.read().strip()
                
            # 精确匹配
            if expected == actual:
                return {"status": "AC", "score": 100, "message": "Accepted"}
            
            # 浮点数容错
            if self.compare_floating_point(expected, actual):
                return {"status": "AC", "score": 100, "message": "Accepted (float tolerance)"}
            
            # 部分匹配
            similarity = difflib.SequenceMatcher(None, expected, actual).ratio()
            if similarity > 0.8:
                return {"status": "PA", "score": int(similarity * 100), 
                       "message": f"Partially Accepted ({similarity:.2%})"}
            else:
                return {"status": "WA", "score": 0, "message": "Wrong Answer"}
                
        except FileNotFoundError:
            return {"status": "RE", "score": 0, "message": "Runtime Error"}
    
    def judge_all_cases(self):
        """评测所有测试用例"""
        total_score = 0
        total_cases = 0
        
        for case_dir in self.testcases_dir.iterdir():
            if case_dir.is_dir():
                case_name = case_dir.name
                expected_file = case_dir / "output.txt"
                actual_file = self.outputs_dir / f"{case_name}_output.txt"
                
                result = self.compare_output(expected_file, actual_file, case_name)
                result["case"] = case_name
                
                self.results.append(result)
                total_score += result["score"]
                total_cases += 1
                
        final_score = total_score / total_cases if total_cases > 0 else 0
        
        summary = {
            "final_score": final_score,
            "total_cases": total_cases,
            "results": self.results
        }
        
        with open("judge_result.json", "w") as f:
            json.dump(summary, f, indent=2)
            
        print(f"🎯 Final Score: {final_score:.1f}%")
        return summary

if __name__ == "__main__":
    judge = Judge()
    result = judge.judge_all_cases()
    
    # 设置CI/CD退出码
    exit(0 if result["final_score"] >= 60 else 1)
```

---

## 9. GitLab CI/CD配置详解

### 9.1 关键字分类

| 类型 | 示例 | 可自定义程度 | 说明 |
|------|------|-------------|------|
| **固定关键字** | `script`, `image`, `stage` | ❌ 不可更改 | GitLab预定义 |
| **Job名称** | `test-python`, `我的任务` | ✅ 完全自定义 | 任意命名 |
| **Stage名称** | `build`, `test`, `部署阶段` | ✅ 完全自定义 | 任意命名 |
| **变量名** | `MY_VAR`, `PYTHON_VERSION` | ✅ 完全自定义 | 用户定义 |
| **某些值** | `when: on_success` | ❌ 预定义选项 | 固定可选值 |

### 9.2 完整配置示例

```yaml
# 混合使用固定关键字和自定义值
stages:                        # 固定关键字
  - 安全检查                    # 自定义stage名称
  - 单元测试                    # 自定义stage名称

variables:                     # 固定关键字
  数据库类型: "postgresql"       # 自定义变量
  测试超时: "300"               # 自定义变量

我公司的Python质量检查:         # 自定义Job名称
  stage: 安全检查              # 固定关键字，引用自定义stage
  image: python:3.9            # 固定关键字，自定义镜像
  tags: [kubernetes, 高性能节点] # 固定关键字，自定义tag
  script:                      # 固定关键字
    - pip install bandit       # 自定义脚本
    - bandit -r . -f json      # 自定义脚本
  artifacts:                   # 固定关键字
    when: always               # 固定值选项
    paths: [安全报告.json]       # 自定义路径
  rules:                       # 固定关键字
    - if: $CI_COMMIT_BRANCH == "main"
      when: always             # 固定值选项
```

### 9.3 预定义变量

```yaml
# GitLab提供的内置变量（只读）
script:
  - echo $CI_PIPELINE_ID       # Pipeline ID
  - echo $CI_JOB_NAME          # Job名称
  - echo $CI_PROJECT_PATH      # 项目路径
  - echo $CI_COMMIT_SHA        # 提交哈希
  - echo $GITLAB_USER_EMAIL    # 用户邮箱
  - echo $CI_NODE_INDEX        # 并行任务索引
```

---

## 10. 安全和最佳实践

### 10.1 Pod安全配置

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: runner-build
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    resources:
      limits:
        cpu: "1"
        memory: "1Gi"
        ephemeral-storage: "1Gi"
```

### 10.2 网络安全

```yaml
# NetworkPolicy示例
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gitlab-network-policy
  namespace: gitlab
spec:
  podSelector:
    matchLabels:
      app: gitlab
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### 10.3 密钥管理

```yaml
# 敏感信息使用Secret
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-secrets
  namespace: gitlab
type: Opaque
data:
  root-password: <base64-encoded>
  db-password: <base64-encoded>
  registry-password: <base64-encoded>
```

---

## 11. 故障排查指南

### 11.1 常见问题及解决方案

| 问题现象 | 可能原因 | 解决方案 |
|---------|---------|---------|
| Pod创建失败 | RBAC权限不足 | 检查ServiceAccount权限 |
| 镜像拉取失败 | 网络或权限问题 | 配置镜像仓库凭证 |
| Runner不执行Job | 标签不匹配 | 检查tags配置 |
| Job超时 | 资源限制过低 | 增加CPU/内存限制 |
| 存储挂载失败 | PVC配置错误 | 检查StorageClass |

### 11.2 调试命令

```bash
# 查看GitLab状态
kubectl get pods -n gitlab
kubectl describe pod gitlab-xxx -n gitlab

# 查看Runner日志
kubectl logs -f gitlab-runner-xxx -n gitlab

# 查看存储状态
kubectl get pvc -n gitlab
kubectl describe pvc gitlab-data-pvc -n gitlab

# 进入容器调试
kubectl exec -it gitlab-xxx -n gitlab -- /bin/bash

# 查看网络连接
kubectl get svc -n gitlab
kubectl get ingress -n gitlab
```

### 11.3 性能监控

```yaml
# 资源使用监控
kubectl top pods -n gitlab
kubectl top nodes

# 查看事件
kubectl get events -n gitlab --sort-by='.lastTimestamp'

# 查看资源配额
kubectl describe quota -n gitlab
```

---

## 总结

本指南涵盖了GitLab在Kubernetes上的完整部署流程，从基础架构设计到高级功能实现。主要特点：

1. **完整性**：覆盖部署、配置、运维全流程
2. **实用性**：提供生产环境可用的配置示例
3. **安全性**：包含安全最佳实践
4. **可扩展性**：支持从小型团队到企业级应用

建议根据实际需求选择合适的部署方案，并根据团队规模和使用场景进行相应调整。