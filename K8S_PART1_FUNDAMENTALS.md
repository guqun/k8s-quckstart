# Kubernetes快速上手教程 - 第一篇：基础概念与架构

## 序言

欢迎来到Kubernetes快速上手教程的第一篇！这一篇将为您建立Kubernetes的核心知识基础。

在这一篇中，您将学习到：
- **什么是Kubernetes** - 理解容器编排的核心概念和价值
- **Kubernetes的架构设计** - 掌握集群组件和工作原理
- **核心资源对象** - 深入了解Pod、Service、Deployment等基本概念

这些基础知识是后续实战操作的基石。建议您按顺序阅读，并在条件允许的情况下动手实践每个概念。

**适合读者**：
- 有基本Docker容器经验的开发者
- 希望了解容器编排技术的运维人员
- 准备在生产环境使用Kubernetes的技术团队

**预计阅读时间**：2-3小时

---

## 目录

1. [Kubernetes是什么](#1-kubernetes是什么)
2. [核心架构](#2-核心架构)
3. [核心资源对象详解](#3-核心资源对象详解)

---
## 1. Kubernetes是什么

### 1.1 一句话理解K8s
**Kubernetes是容器编排平台**，自动化部署、扩展和管理容器化应用程序。

### 1.2 解决什么问题

| 传统部署的痛点 | K8s的解决方案 |
|--------------|-------------|
| 手动部署应用到服务器 | 声明式配置，自动部署 |
| 服务器故障，手动迁移 | 自动故障转移，自愈能力 |
| 流量增加，手动扩容 | 自动扩缩容（HPA/VPA） |
| 更新应用需要停机 | 滚动更新，零停机部署 |
| 多环境配置管理混乱 | ConfigMap/Secret统一管理 |
| 服务发现依赖硬编码 | DNS服务发现，负载均衡 |

### 1.3 K8s vs Docker

```
Docker：打包和运行单个容器
K8s：编排和管理成千上万个容器

类比：
- Docker = 集装箱（标准化包装）
- K8s = 港口管理系统（调度、存储、运输）
```

---

## 2. 核心架构

### 2.1 集群架构图

```
┌────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                   │
│                                                        │
│  ┌──────────────── Control Plane ──────────────────┐  │
│  │                                                  │  │
│  │  API Server ←→ etcd                             │  │
│  │      ↕                                          │  │
│  │  Scheduler  Controller Manager  Cloud Provider  │  │
│  └──────────────────────────────────────────────────┘  │
│                           ↓                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │                   Worker Nodes                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │  │
│  │  │   Node 1    │  │   Node 2    │  │  Node N  │ │  │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │          │ │  │
│  │  │ │ kubelet │ │  │ │ kubelet │ │  │ kubelet  │ │  │
│  │  │ │ kube-   │ │  │ │ kube-   │ │  │ kube-    │ │  │
│  │  │ │ proxy   │ │  │ │ proxy   │ │  │ proxy    │ │  │
│  │  │ │ docker  │ │  │ │ docker  │ │  │ docker   │ │  │
│  │  │ └─────────┘ │  │ └─────────┘ │  │          │ │  │
│  │  │   [Pods]    │  │   [Pods]    │  │  [Pods]  │ │  │
│  │  └─────────────┘  └─────────────┘  └──────────┘ │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### 2.2 控制平面组件

| 组件 | 功能 | 类比 |
|-----|------|-----|
| **API Server** | 所有操作的入口，REST API | 前台接待 |
| **etcd** | 存储集群所有数据 | 数据库 |
| **Scheduler** | 决定Pod在哪个节点运行 | 调度员 |
| **Controller Manager** | 确保实际状态符合期望状态 | 监工 |

### 2.3 节点组件

| 组件 | 功能 | 运行位置 |
|-----|------|---------|
| **kubelet** | 管理Pod生命周期 | 每个节点 |
| **kube-proxy** | 网络代理，负载均衡 | 每个节点 |
| **Container Runtime** | 运行容器（Docker/containerd） | 每个节点 |

---

## 3. 核心资源对象详解

### 3.1 工作负载资源

#### Pod - 最小部署单元
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```
**特点**：
- 包含1个或多个容器
- 共享网络和存储
- 生命周期短暂
- 通常不直接创建Pod

#### Deployment - 无状态应用部署
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3              # 副本数
  strategy:
    type: RollingUpdate    # 滚动更新策略
    rollingUpdate:
      maxSurge: 1          # 更新时最多多出1个Pod
      maxUnavailable: 1    # 更新时最多1个Pod不可用
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: myapp:v2
        resources:         # 资源限制
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

#### StatefulSet - 有状态应用
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
  volumeClaimTemplates:    # 持久化存储模板
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```
**特点**：
- 稳定的网络标识（mysql-0, mysql-1）
- 有序部署、有序删除
- 持久化存储

#### DaemonSet - 守护进程
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter
```
**用途**：每个节点运行一个Pod（日志收集、监控代理）

#### Job/CronJob - 任务调度
```yaml
# 一次性任务
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool
        command: ["./backup.sh"]
      restartPolicy: OnFailure

---
# 定时任务
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 每天2点执行
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
          restartPolicy: OnFailure
```

### 3.2 服务发现与负载均衡

#### 为什么需要Service？

**问题场景**：
```
应用A需要调用应用B的API：
- Pod IP是动态的（重启会变）
- 有多个B的Pod副本（该调用哪个？）
- Pod可能随时增减（扩缩容）
```

**Service解决方案**：
```
应用A → Service B（固定入口） → 负载均衡 → Pod B1, B2, B3
```

#### Service工作原理

**服务发现机制**：
```
创建Service时发生了什么：

1. 分配ClusterIP（虚拟IP）
   Service: my-service → 10.96.10.15

2. 创建DNS记录
   my-service.default.svc.cluster.local → 10.96.10.15

3. 创建Endpoints（端点列表）
   监控符合selector的Pod，维护IP列表
   Endpoints: [10.1.0.5:8080, 10.1.0.6:8080, 10.1.0.7:8080]

4. 配置iptables/IPVS规则
   实现负载均衡到后端Pod
```

**负载均衡实现**：
```
客户端请求流程：

1. Pod A: curl http://my-service:8080
                ↓
2. DNS解析: my-service → 10.96.10.15
                ↓
3. 请求发送到: 10.96.10.15:8080
                ↓
4. kube-proxy拦截（iptables规则）
                ↓
5. 负载均衡选择: RoundRobin
                ↓
6. 转发到实际Pod: 10.1.0.6:8080
```

#### Service类型详解

**1. ClusterIP（默认类型）**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP  # 可省略
  selector:
    app: backend
  ports:
  - port: 80        # Service端口
    targetPort: 8080 # Pod端口
```

特点与用途：
- **访问范围**：仅集群内部
- **获得什么**：虚拟IP + DNS名称
- **访问方式**：
  - 短名称：`backend`（同namespace）
  - 完整域名：`backend.default.svc.cluster.local`
  - ClusterIP：`10.96.10.15`
- **适用场景**：内部微服务、数据库、缓存

实际使用：
```python
# 在Pod内的应用代码
import requests
# 推荐：使用Service名
response = requests.get('http://backend/api/data')
```

**2. NodePort**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80         # Service端口
    targetPort: 8080  # Pod端口
    nodePort: 30080   # 节点端口（30000-32767）
```

特点与工作流程：
- **访问范围**：集群外部 + 内部
- **暴露方式**：每个节点的指定端口
- **访问地址**：`<任意节点IP>:30080`
- **适用场景**：开发测试、私有云、简单对外服务

```
外部客户端
    ↓
节点IP:30080（任意节点都可以）
    ↓
kube-proxy转发
    ↓
Service（内部负载均衡）
    ↓
Pod（最终处理请求）
```

**3. LoadBalancer**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

特点与实际效果：
- **前提条件**：需要云厂商支持（AWS ELB、阿里云SLB等）
- **获得什么**：外部负载均衡器 + 公网IP
- **访问方式**：`<LoadBalancer-IP>:80`
- **适用场景**：生产环境、公网服务

```bash
kubectl get svc frontend
NAME       TYPE           EXTERNAL-IP                 PORT(S)
frontend   LoadBalancer   a1b2c3d4.elb.amazonaws.com  80:31234/TCP
```

**4. ExternalName**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ExternalName
  externalName: mysql.example.com
```

特点与用途：
- **功能**：为外部服务创建内部别名
- **实现**：CNAME记录
- **无选择器**：不关联Pod
- **适用场景**：外部数据库、跨集群服务、迁移过渡期

使用示例：
```python
# 应用内使用，不管数据库在哪，统一用Service名访问
conn = mysql.connect(host='database', port=3306)
# 实际解析到: mysql.example.com
```

#### Ingress - 七层路由

**为什么需要Ingress？**

Service的局限：
- LoadBalancer每个服务需要一个公网IP（成本高）
- NodePort端口管理混乱
- 无法基于域名/路径路由
- 不支持SSL/TLS终止

Ingress解决方案：
```
                 Ingress Controller
                         ↓
    app.com/web → web-service → web-pods
    app.com/api → api-service → api-pods
    admin.com   → admin-service → admin-pods
```

**Ingress配置示例**：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**关键概念**：

1. **Ingress Controller**：实际处理流量的组件（nginx、traefik等），需单独安装
2. **Rules**：路由规则
   - `host`：基于域名路由
   - `path`：基于路径路由
   - `pathType`：Exact（精确）、Prefix（前缀）
3. **TLS配置**：自动HTTPS，证书存储在Secret中
4. **Annotations**：控制器特定配置（重写、超时、限流等）

#### Ingress Controller深度解析

**Ingress三要素**：

1. **Ingress资源**：规则定义（只是声明，不处理流量）
2. **Ingress Controller**：规则执行者（实际处理流量的组件）
3. **Ingress Class**：控制器选择（K8s 1.18+支持多Controller）

**工作原理**：
```
┌──────────────────────────────────────────────┐
│           Ingress Controller Pod              │
│                                               │
│  1. 监听器（Watcher）                          │
│     ↓ 监听Ingress资源变化                      │
│                                               │
│  2. 配置生成器                                 │
│     ↓ 将Ingress规则转换为nginx.conf           │
│                                               │
│  3. Nginx进程                                 │
│     ↓ 重载配置，处理请求                       │
│                                               │
│  4. 后端连接                                  │
│     → Service → Pods                         │
└──────────────────────────────────────────────┘
```

**实际部署与配置**：

1. **安装Controller**：
```bash
# 方式1：官方YAML
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# 方式2：Helm（推荐）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

2. **验证安装**：
```bash
# 查看Controller Pod
kubectl get pods -n ingress-nginx

# 查看Service
kubectl get svc -n ingress-nginx
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   34.102.136.180
```

3. **高级配置示例**：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    # 基础配置
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # 性能配置
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    
    # 限流配置
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    
    # 认证配置
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # CORS配置
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
    
    # 金丝雀发布
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**SSL/TLS配置**：
```bash
# 手动证书
kubectl create secret tls tls-secret --key tls.key --cert tls.crt

# 自动证书（cert-manager）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-auto  # cert-manager自动创建
```

**监控与调试**：
```bash
# 查看nginx配置
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf

# 查看访问日志
kubectl logs -n ingress-nginx <controller-pod> -f

# 常见问题排查
kubectl describe ingress my-ingress
kubectl get endpoints my-service
```

#### Ingress Controller vs NodePort 流量对比

**NodePort流量路径**：
```
外部用户
    ↓
任意节点IP:30080 (NodePort)
    ↓
kube-proxy (iptables/IPVS规则)
    ↓
Service (ClusterIP)
    ↓
负载均衡到Pod
    ↓
Pod处理请求
```

**Ingress Controller流量路径**：
```
外部用户
    ↓
域名解析 (app.example.com → LoadBalancer IP)
    ↓
LoadBalancer (云厂商提供)
    ↓
Ingress Controller Pod
    ↓
根据Host/Path规则路由
    ↓
对应的Service (ClusterIP)
    ↓
负载均衡到Pod
    ↓
Pod处理请求
```

**详细对比分析**：

| 维度 | NodePort | Ingress Controller |
|------|---------|-------------------|
| **入口数量** | 每个Service一个端口 | 统一入口(80/443) |
| **端口管理** | 30000-32767范围限制 | 标准HTTP(S)端口 |
| **域名支持** | 不支持 | 支持基于域名路由 |
| **路径路由** | 不支持 | 支持基于路径路由 |
| **SSL终止** | 不支持 | 支持SSL/TLS终止 |
| **成本** | 无额外成本 | 需要LoadBalancer |
| **生产适用性** | 开发测试 | 生产环境首选 |

**实际部署示例对比**：

1. **NodePort方式**：
```yaml
# 需要为每个服务创建NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30001  # web服务

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30002  # API服务

# 访问方式（端口难记）：
# http://node-ip:30001  - Web服务
# http://node-ip:30002  - API服务
```

2. **Ingress方式**：
```yaml
# Service保持ClusterIP类型
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP  # 内部访问
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP  # 内部访问
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080

# 统一通过Ingress路由
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: unified-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

# 访问方式（标准化）：
# https://app.example.com/     - Web服务
# https://app.example.com/api  - API服务
```

#### Service与Ingress Controller的关系

**核心关系**：Service是Ingress Controller的后端目标

```
┌─────────────────────────────────────────────────────┐
│                 集群外部流量                         │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│              Ingress Controller                     │
│  ┌─────────────────────────────────────────────┐   │
│  │         路由规则处理                         │   │
│  │  app.com/web  → web-service                │   │
│  │  app.com/api  → api-service                │   │
│  │  admin.com    → admin-service              │   │
│  └─────────────────────────────────────────────┘   │
└──────────┬──────────────┬──────────────┬───────────┘
           ↓              ↓              ↓
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │web-service  │ │api-service  │ │admin-service│
    │(ClusterIP)  │ │(ClusterIP)  │ │(ClusterIP)  │
    └─────┬───────┘ └─────┬───────┘ └─────┬───────┘
          ↓               ↓               ↓
     ┌─────────┐     ┌─────────┐     ┌─────────┐
     │Web Pods │     │API Pods │     │Admin Pod│
     └─────────┘     └─────────┘     └─────────┘
```

**关系详解**：

1. **Ingress Controller是Service的消费者**
   - Ingress Controller通过Service名称发现后端Pod
   - 不直接与Pod通信，而是与Service通信
   - Service提供负载均衡和服务发现

2. **Service为Ingress提供抽象层**
   - Ingress规则中配置Service名称，而非Pod IP
   - Pod变化时，Service自动更新，Ingress无需修改
   - Service健康检查失效的Pod，Ingress不会路由到故障Pod

3. **两者职责分工**
   ```
   Ingress Controller职责：
   - 七层路由（HTTP/HTTPS）
   - 基于域名、路径的流量分发
   - SSL/TLS终止
   - 认证、限流等高级功能
   
   Service职责：
   - 四层负载均衡（TCP/UDP）
   - Pod服务发现
   - 健康检查
   - 会话亲和性
   ```

**实际工作流程**：

```bash
# 1. 创建Pod和Service
kubectl apply -f deployment.yaml  # 创建Pod
kubectl apply -f service.yaml     # 创建Service

# 2. Service自动发现Pod
kubectl get endpoints my-service  # 查看Service发现的Pod

# 3. 创建Ingress规则
kubectl apply -f ingress.yaml    # 引用Service名称

# 4. Ingress Controller处理流程
外部请求 → Ingress Controller检查Host/Path
        ↓
Ingress Controller查找对应Service
        ↓
通过Service名称解析到ClusterIP
        ↓
Service负载均衡到健康的Pod
        ↓
Pod处理请求并返回
```

**Service类型选择**：

当使用Ingress时，后端Service应该选择什么类型？

```yaml
# ✅ 推荐：ClusterIP（默认）
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # 或者省略（默认）
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080

# ❌ 不推荐：NodePort（冗余暴露）
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: NodePort   # 没必要，Ingress已经处理外部访问
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080

# ❌ 错误：LoadBalancer（双重负载均衡）
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: LoadBalancer  # 与Ingress冲突
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

**Headless Service特殊情况**：

某些场景下，Ingress Controller需要直接访问Pod：

```yaml
# Headless Service - 直接获取Pod IP列表
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  clusterIP: None  # Headless Service
  selector:
    app: stateful-app
  ports:
  - port: 8080

# Ingress Controller可以：
# 1. 获取所有Pod IP列表
# 2. 实现自定义负载均衡算法
# 3. 支持会话粘性
```

**调试Service与Ingress关系**：

```bash
# 1. 检查Service是否正常
kubectl get svc my-service
kubectl get endpoints my-service

# 2. 检查Ingress是否正确引用Service
kubectl describe ingress my-ingress

# 3. 测试Service内部访问
kubectl run test --image=busybox -it --rm -- wget my-service:8080

# 4. 测试Ingress外部访问
curl -H "Host: app.example.com" http://<ingress-ip>/

# 5. 查看流量流向
kubectl logs -n ingress-nginx <controller-pod> | grep my-service
```

#### Service网络原理

**Service IP的本质**：
```bash
# Service IP是虚拟的，不存在于任何网卡上
kubectl get svc my-service
NAME         CLUSTER-IP    
my-service   10.96.10.15

# ping不通（没有ICMP实现）
ping 10.96.10.15  # 超时

# 但可以访问端口（iptables/IPVS转发）
curl 10.96.10.15:80  # 正常
```

**kube-proxy的三种模式**：

1. **iptables模式**（默认）
   - 内核空间转发，性能好
   - 规则多时性能下降

2. **IPVS模式**（推荐）
   - 更好的性能（哈希表 vs 链式规则）
   - 更多负载均衡算法

3. **userspace模式**（已弃用）

**会话亲和性**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP  # 相同客户端IP路由到相同Pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3小时
```

#### Service调试

**排查步骤**：
```bash
# 1. 检查Pod是否运行
kubectl get pods -l app=web

# 2. 检查Service配置
kubectl describe svc web-service

# 3. 检查Endpoints（如果为空，selector不匹配）
kubectl get endpoints web-service

# 4. 测试Pod直接访问
kubectl get pods -o wide  # 获取Pod IP
kubectl exec test-pod -- curl 10.1.0.5:8080

# 5. 测试Service访问
kubectl exec test-pod -- curl web-service:80
```

### 3.3 配置与存储

#### ConfigMap - 配置管理详解

**什么是ConfigMap？**
ConfigMap是K8s中用于存储非敏感配置数据的资源对象，实现了配置与代码的分离。

**为什么需要ConfigMap？**

传统方式的问题：
```dockerfile
# ❌ 传统方式：配置写死在镜像中
FROM node:14
COPY app.js /app/
COPY config.json /app/  # 配置文件打包到镜像
ENV DATABASE_URL=mysql://localhost:3306/prod  # 环境变量写死
CMD ["node", "/app/app.js"]
```

问题：
- 不同环境需要不同镜像
- 配置变更需要重新构建镜像
- 敏感信息暴露在镜像中

ConfigMap解决方案：
```
一个镜像 + 不同ConfigMap = 不同环境部署
```

**ConfigMap创建方式对比**：

1. **命令行创建**（适合快速测试）：
```bash
# 从文件创建
kubectl create configmap app-config --from-file=config.properties

# 从目录创建
kubectl create configmap app-config --from-file=./config/

# 从键值对创建
kubectl create configmap app-config \
  --from-literal=database.host=mysql.example.com \
  --from-literal=database.port=3306

# 查看创建结果
kubectl get configmap app-config -o yaml
```

2. **YAML文件创建**（适合生产环境）：
```yaml
# 方式1：键值对配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # 简单键值对
  database.host: "mysql.production.local"
  database.port: "3306"
  database.name: "myapp"
  cache.enabled: "true"
  log.level: "INFO"

---
# 方式2：文件式配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-files-config
data:
  # 完整配置文件
  database.conf: |
    [database]
    host=mysql.production.local
    port=3306
    database=myapp
    pool_size=20
    timeout=30
    
  nginx.conf: |
    server {
        listen 80;
        server_name app.example.com;
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
        }
    }
    
  app.properties: |
    # Application Configuration
    logging.level=INFO
    cache.enabled=true
    cache.ttl=3600
    feature.new_ui=true
```

**ConfigMap使用方式详解**：

1. **环境变量注入**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        env:
        # 方式1：单个环境变量
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host
        
        # 方式2：批量导入所有key
        envFrom:
        - configMapRef:
            name: app-config
            
        # 应用内使用
        command: ["sh", "-c"]
        args: ["echo Database: $DATABASE_HOST && ./start.sh"]
```

2. **文件挂载**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        volumeMounts:
        # 挂载整个ConfigMap
        - name: config-volume
          mountPath: /etc/nginx/conf.d
        # 挂载单个文件
        - name: app-config
          mountPath: /etc/app/database.conf
          subPath: database.conf
          
      volumes:
      # 挂载所有配置文件
      - name: config-volume
        configMap:
          name: app-files-config
      # 挂载特定文件
      - name: app-config
        configMap:
          name: app-files-config
          items:
          - key: database.conf
            path: database.conf
```

**实际应用场景**：

1. **多环境配置管理**：
```yaml
# 开发环境
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  database.host: "mysql.dev.local"
  database.name: "myapp_dev"
  log.level: "DEBUG"
  cache.enabled: "false"

# 生产环境
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config  # 同名但不同namespace
  namespace: production
data:
  database.host: "mysql.prod.local"
  database.name: "myapp_prod"
  log.level: "WARN"
  cache.enabled: "true"
```

2. **应用配置热更新**：
```bash
# 更新ConfigMap
kubectl patch configmap app-config -p '{"data":{"log.level":"DEBUG"}}'

# 重启Pod使配置生效（环境变量方式）
kubectl rollout restart deployment/myapp

# 文件挂载方式会自动更新（有延迟，默认60秒）
kubectl exec -it <pod-name> -- cat /etc/config/app.properties
```

3. **Nginx配置管理**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        upstream backend {
            server backend-service:8080;
        }
        server {
            listen 80;
            location / {
                proxy_pass http://backend;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

**ConfigMap最佳实践**：

1. **命名规范**：
```bash
# 好的命名
app-config-v1
mysql-config-production
nginx-config-frontend

# 不好的命名
config
data
settings
```

2. **版本管理**：
```yaml
# 使用版本号管理ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2  # 版本化命名
  labels:
    app: myapp
    version: v2
    environment: production
data:
  config.yaml: |
    version: "2.0"
    features:
      new_ui: true
```

3. **配置验证**：
```bash
# 创建前验证语法
kubectl create configmap test-config --dry-run=client -o yaml \
  --from-file=config.yaml

# 检查配置内容
kubectl describe configmap app-config
```

#### ConfigMap与SpringBoot应用集成详解

**实际问题**：开发者经常问"ConfigMap中的`database.host: "mysql.dev.local"`这样的配置如何进入SpringBoot应用？比如在`application-prod.yaml`文件中使用？"

**传统SpringBoot配置方式**：

```yaml
# application-prod.yaml（传统方式）
spring:
  datasource:
    url: jdbc:mysql://mysql.prod.local:3306/myapp_prod
    username: ${DB_USERNAME:admin}
    password: ${DB_PASSWORD:password}
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  redis:
    host: redis.prod.local
    port: 6379
    password: ${REDIS_PASSWORD}
```

**K8s + ConfigMap的集成方式**：

**方式1：环境变量替换（推荐入门）**

1. **创建ConfigMap**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # 数据库配置
  spring.datasource.url: "jdbc:mysql://mysql-service:3306/myapp_prod"
  spring.datasource.username: "appuser"
  spring.redis.host: "redis-service"
  spring.redis.port: "6379"
  
  # 应用配置
  app.name: "MyApp Production"
  app.version: "1.0.0"
  logging.level.com.myapp: "INFO"
```

2. **SpringBoot应用配置**：
```yaml
# application.yaml（容器内）
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    
  redis:
    host: ${SPRING_REDIS_HOST}
    port: ${SPRING_REDIS_PORT}
    password: ${SPRING_REDIS_PASSWORD}

app:
  name: ${APP_NAME}
  version: ${APP_VERSION}

logging:
  level:
    com.myapp: ${LOGGING_LEVEL_COM_MYAPP}
```

3. **Deployment注入环境变量**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        env:
        # 从ConfigMap注入环境变量
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: spring.datasource.url
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: spring.datasource.username
        - name: SPRING_REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: spring.redis.host
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.name
              
        # 从Secret注入敏感信息
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: spring.datasource.password
```

**方式2：配置文件挂载（生产推荐）**

1. **完整配置文件作为ConfigMap**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-config
data:
  application-k8s.yaml: |
    spring:
      datasource:
        url: jdbc:mysql://mysql-service:3306/myapp_prod
        username: appuser
        password: ${DB_PASSWORD}  # 仍然从Secret获取
        driver-class-name: com.mysql.cj.jdbc.Driver
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
      
      redis:
        host: redis-service
        port: 6379
        password: ${REDIS_PASSWORD}
        timeout: 2000ms
        
      jpa:
        hibernate:
          ddl-auto: validate
        show-sql: false
          
    server:
      port: 8080
      servlet:
        context-path: /api
        
    logging:
      level:
        com.myapp: INFO
        org.springframework.security: DEBUG
        
    app:
      name: "MyApp Production"
      version: "1.0.0"
      cors:
        allowed-origins: "https://app.example.com"
```

2. **挂载配置文件到容器**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        
        # 指定SpringBoot使用k8s配置文件
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "k8s"  # 激活k8s profile
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: database.password
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: redis.password
              
        volumeMounts:
        # 挂载配置文件
        - name: config-volume
          mountPath: /app/config
          
        # SpringBoot启动参数
        args:
        - "--spring.config.location=classpath:/application.yaml,/app/config/application-k8s.yaml"
        
      volumes:
      - name: config-volume
        configMap:
          name: springboot-config
```

**方式3：Spring Cloud Kubernetes（最优雅）**

1. **添加依赖**：
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
```

2. **SpringBoot配置**：
```yaml
# application.yaml
spring:
  application:
    name: myapp
  cloud:
    kubernetes:
      config:
        enabled: true
        name: app-config  # ConfigMap名称
        namespace: production
      secrets:
        enabled: true
        name: app-secret  # Secret名称
        namespace: production
      reload:
        enabled: true
        mode: event  # 配置变更时自动重载
```

3. **ConfigMap结构化配置**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp  # 必须与spring.application.name匹配
  namespace: production
data:
  application.yaml: |
    spring:
      datasource:
        url: jdbc:mysql://mysql-service:3306/myapp_prod
        username: appuser
        hikari:
          maximum-pool-size: 20
      redis:
        host: redis-service
        port: 6379
    app:
      name: "MyApp Production"
      cors:
        allowed-origins: "https://app.example.com"
```

#### ConfigMap到SpringBoot的内部流程原理

**完整数据流转过程**：

```
┌─────────────────────────────────────────────────────────┐
│                   K8s集群层面                             │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │ ConfigMap   │    │   Secret    │    │    Pod      │ │
│  │key: value   │    │key: xxx     │    │             │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                   │                   │       │
│         └───────────────────┼───────────────────┘       │
│                            │                           │
└────────────────────────────┼───────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────┐
│                  容器运行时层面                           │
│                                                         │
│  kubelet通过Volume挂载 + 环境变量注入                    │
│         │                                               │
│         ↓                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              容器内部                             │   │
│  │  环境变量: SPRING_DATASOURCE_URL=jdbc:mysql...   │   │
│  │  文件挂载: /app/config/application-k8s.yaml     │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│                SpringBoot应用层面                        │
│                                                         │
│  1. JVM启动 → 读取环境变量                               │
│  2. SpringBoot配置加载机制启动                          │
│  3. PropertySource优先级解析                           │
│  4. 占位符解析和值替换                                   │
│  5. 最终配置对象创建                                     │
└─────────────────────────────────────────────────────────┘
```

**详细内部流程解析**：

**阶段1：K8s资源到容器的映射**
```bash
# 1. ConfigMap存储在etcd中
kubectl get configmap app-config -o yaml
# 输出：key-value数据

# 2. Pod创建时，kubelet读取ConfigMap
# kubelet进程执行以下操作：

# 环境变量注入方式：
kubelet → 读取ConfigMap → 设置容器环境变量
export SPRING_DATASOURCE_URL="jdbc:mysql://mysql-service:3306/myapp"

# 文件挂载方式：
kubelet → 读取ConfigMap → 创建临时文件 → 挂载到容器路径
/var/lib/kubelet/pods/.../volumes/kubernetes.io~configmap/config-volume/application-k8s.yaml
```

**阶段2：容器内部的文件系统状态**
```bash
# 进入容器查看实际状态
kubectl exec -it springboot-pod -- bash

# 环境变量已注入
env | grep SPRING
SPRING_DATASOURCE_URL=jdbc:mysql://mysql-service:3306/myapp_prod
SPRING_DATASOURCE_USERNAME=appuser

# 配置文件已挂载
ls -la /app/config/
-rw-r--r-- 1 root root 1234 Dec  1 10:00 application-k8s.yaml

# 查看挂载文件内容
cat /app/config/application-k8s.yaml
spring:
  datasource:
    url: jdbc:mysql://mysql-service:3306/myapp_prod
    username: appuser
```

**阶段3：SpringBoot配置加载机制**

SpringBoot内部使用`Environment`抽象来管理配置，加载顺序：

```java
// SpringBoot配置加载的内部原理（简化版）
public class ConfigurationPropertySources {
    
    // 1. PropertySource优先级（从高到低）
    List<PropertySource> propertySources = Arrays.asList(
        new SystemPropertySource(),           // -Dprop=value
        new EnvironmentVariablePropertySource(), // 环境变量（K8s注入的）
        new ApplicationPropertySource(),      // application.yaml
        new DefaultPropertySource()           // 默认值
    );
    
    // 2. 占位符解析过程
    @Value("${spring.datasource.url}")
    private String url;
    
    // 解析过程：
    // ${spring.datasource.url} 
    // → 查找环境变量 SPRING_DATASOURCE_URL
    // → 找到值：jdbc:mysql://mysql-service:3306/myapp_prod
    // → 注入到字段
}
```

**实际SpringBoot启动日志分析**：

```bash
# SpringBoot启动时的配置加载日志
kubectl logs springboot-pod

# 你会看到类似日志：
2024-01-01 10:00:01 INFO  o.s.b.c.e.AnnotationConfigEmbeddedWebApplicationContext
- Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext

2024-01-01 10:00:02 INFO  o.s.c.e.PropertySourcesPropertyResolver  
- Found property 'spring.datasource.url' with value 'jdbc:mysql://mysql-service:3306/myapp_prod'

2024-01-01 10:00:03 INFO  o.s.b.c.e.tomcat.TomcatEmbeddedServletContainer
- Tomcat initialized with port(s): 8080 (http)
```

**方式1：环境变量注入的内部原理**

```java
// SpringBoot如何读取环境变量
@Configuration
public class DataSourceConfig {
    
    // 1. SpringBoot扫描到@Value注解
    @Value("${spring.datasource.url}")  
    private String url;
    
    // 2. PropertyResolver解析过程
    String resolveProperty(String key) {
        // a. 查找系统属性 System.getProperty("spring.datasource.url")
        // b. 查找环境变量 System.getenv("SPRING_DATASOURCE_URL")  ← K8s注入的
        // c. 查找application.properties
        // d. 返回找到的第一个值
    }
    
    // 3. 最终创建DataSource时url已经是解析后的值
    @Bean
    public DataSource dataSource() {
        // url = "jdbc:mysql://mysql-service:3306/myapp_prod"
        return DataSourceBuilder.create().url(url).build();
    }
}
```

**方式2：配置文件挂载的内部原理**

```java
// SpringBoot配置文件加载机制
public class ConfigFileApplicationListener implements ApplicationListener {
    
    // 1. 启动时扫描配置文件位置
    List<String> configLocations = Arrays.asList(
        "classpath:/application.yaml",           // 默认
        "/app/config/application-k8s.yaml"      // K8s挂载的
    );
    
    // 2. 按顺序加载配置文件
    for (String location : configLocations) {
        Properties props = loadProperties(location);
        environment.getPropertySources().addLast(
            new PropertiesPropertySource(location, props)
        );
    }
    
    // 3. 后加载的配置会覆盖先加载的（如果key相同）
    // 所以K8s挂载的配置会覆盖默认配置
}
```

**配置优先级实际验证**：

```yaml
# 1. 默认application.yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/default_db  # 默认值

# 2. K8s挂载的application-k8s.yaml  
spring:
  datasource:
    url: jdbc:mysql://mysql-service:3306/myapp_prod  # K8s值

# 3. 环境变量
SPRING_DATASOURCE_URL=jdbc:mysql://mysql-service:3306/myapp_prod

# 最终生效优先级：
# 环境变量 > K8s挂载文件 > 默认application.yaml
```

**Spring Cloud Kubernetes的内部原理**

```java
// Spring Cloud K8s自动配置原理
@Configuration
@EnableConfigurationProperties(KubernetesConfigProperties.class)
public class KubernetesConfigBootstrapConfiguration {
    
    @Bean
    public KubernetesConfigMapPropertySource configMapPropertySource() {
        // 1. 通过K8s API读取ConfigMap
        ConfigMap configMap = kubernetesClient
            .configMaps()
            .inNamespace("production")
            .withName("app-config")
            .get();
            
        // 2. 转换为SpringBoot PropertySource
        Map<String, Object> properties = new HashMap<>();
        configMap.getData().forEach((key, value) -> {
            properties.put(key, value);
        });
        
        return new MapPropertySource("k8s-configmap", properties);
    }
    
    // 3. 监听ConfigMap变化（如果启用reload）
    @EventListener
    public void onConfigMapChange(ConfigMapChangeEvent event) {
        // 重新加载配置
        refreshScope.refreshAll();
    }
}
```

**实际调试和验证步骤**：

```bash
# 1. 查看容器内环境变量
kubectl exec -it springboot-pod -- env | grep -i spring

# 2. 查看SpringBoot实际加载的配置
kubectl port-forward springboot-pod 8080:8080
curl http://localhost:8080/actuator/env

# 输出显示配置来源：
{
  "propertySources": [
    {
      "name": "systemEnvironment",
      "properties": {
        "SPRING_DATASOURCE_URL": {
          "value": "jdbc:mysql://mysql-service:3306/myapp_prod",
          "origin": "System Environment Property \"SPRING_DATASOURCE_URL\""
        }
      }
    }
  ]
}

# 3. 查看配置文件加载情况
curl http://localhost:8080/actuator/configprops

# 4. 实时监控配置变化
kubectl patch configmap app-config -p '{"data":{"new.key":"new.value"}}'
# 观察应用日志中的配置刷新信息
```

**总结：数据流转的完整路径**

```
1. 开发者创建ConfigMap → kubectl apply
2. K8s API Server存储到etcd
3. kubelet读取ConfigMap数据
4. kubelet启动容器时注入环境变量/挂载文件
5. JVM启动，读取环境变量和文件
6. SpringBoot ConfigurationPropertySource扫描
7. PropertyResolver解析占位符
8. 最终配置注入到Bean中
9. 应用使用配置连接数据库/Redis等
```

这样就解释了从ConfigMap创建到SpringBoot应用最终使用配置的完整内部流程和原理。

**多环境配置管理**：

开发环境：
```yaml
# dev namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  spring.datasource.url: "jdbc:mysql://mysql-dev:3306/myapp_dev"
  spring.redis.host: "redis-dev"
  logging.level.com.myapp: "DEBUG"
  app.cors.allowed-origins: "http://localhost:3000"
```

生产环境：
```yaml
# prod namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config  # 同名ConfigMap
  namespace: production
data:
  spring.datasource.url: "jdbc:mysql://mysql-prod:3306/myapp_prod"
  spring.redis.host: "redis-prod"
  logging.level.com.myapp: "WARN"
  app.cors.allowed-origins: "https://app.example.com"
```

**配置验证和调试**：
```bash
# 进入Pod查看实际配置
kubectl exec -it myapp-pod -- env | grep SPRING

# 查看配置文件内容
kubectl exec -it myapp-pod -- cat /app/config/application-k8s.yaml

# 查看SpringBoot actuator endpoints
kubectl port-forward myapp-pod 8080:8080
curl http://localhost:8080/actuator/env

# 配置热更新
kubectl patch configmap app-config -p '{"data":{"logging.level.com.myapp":"ERROR"}}'
curl -X POST http://localhost:8080/actuator/refresh
```

**集成方式对比**：

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **环境变量** | 简单直接，无需额外依赖 | 配置项多时管理复杂 | 少量配置、快速开发 |
| **文件挂载** | 保持SpringBoot原生结构 | 需要重启应用更新配置 | 生产环境、复杂配置 |
| **Spring Cloud K8s** | 自动注入、支持热更新 | 增加依赖、学习成本 | 微服务架构、高级特性 |

#### Secret - 敏感数据管理详解

**什么是Secret？**
Secret是K8s中用于存储敏感信息的资源对象，提供比ConfigMap更安全的方式来管理密码、证书、API密钥等敏感数据。

**为什么需要Secret？**

传统方式的问题：
```dockerfile
# ❌ 传统方式：敏感信息硬编码
FROM java:8
ENV DB_PASSWORD=secretpassword123  # 密码暴露在镜像中
ENV API_KEY=abc123def456           # API密钥可见
COPY app.jar /app.jar
```

问题：
- 敏感信息暴露在镜像层中
- 版本控制系统会记录敏感数据
- 无法对不同环境使用不同密钥
- 缺乏访问控制和审计

Secret解决方案：
```
镜像（不含敏感信息） + Secret（运行时注入） = 安全的容器化部署
```

**Secret vs ConfigMap对比**：

| 特性 | ConfigMap | Secret |
|------|----------|--------|
| **存储内容** | 非敏感配置 | 敏感信息 |
| **编码方式** | 明文 | Base64编码 |
| **内存存储** | 可能写入磁盘 | 存储在tmpfs中 |
| **RBAC控制** | 较宽松 | 更严格的权限控制 |
| **审计日志** | 可记录内容 | 隐藏敏感内容 |
| **大小限制** | 1MB | 1MB |
| **etcd加密** | 可选 | 推荐启用 |

**Secret创建方式对比**：

1. **命令行创建**（适合快速测试）：
```bash
# 从字面值创建
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secretpassword123

# 从文件创建
echo -n 'admin' > username.txt
echo -n 'secretpassword123' > password.txt
kubectl create secret generic db-secret \
  --from-file=username.txt \
  --from-file=password.txt

# 查看Secret（密码被自动隐藏）
kubectl get secret db-secret -o yaml
kubectl describe secret db-secret  # 不显示敏感值
```

2. **YAML文件创建**（适合生产环境）：
```yaml
# 方式1：使用data字段（需要手动Base64编码）
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: production
type: Opaque
data:
  # 手动Base64编码：echo -n 'admin' | base64
  username: YWRtaW4=                    # admin
  password: c2VjcmV0cGFzc3dvcmQxMjM=      # secretpassword123
  api-key: YWJjMTIzZGVmNDU2           # abc123def456

---
# 方式2：使用stringData字段（明文，K8s自动编码）
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-auto
  namespace: production
type: Opaque
stringData:
  # 明文，K8s自动进行Base64编码
  username: admin
  password: secretpassword123
  api-key: abc123def456
  database-url: "mysql://admin:secretpassword123@mysql-service:3306/myapp"
  jwt-secret: "my-super-secret-jwt-key-256-bits-long"
```

**Secret类型详解**：

1. **Opaque（通用类型）**：
```yaml
# 存储各种敏感信息
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  # 数据库凭证
  db-username: appuser
  db-password: prod-password-123
  
  # 第三方API密钥
  aws-access-key: AKIAIOSFODNN7EXAMPLE
  aws-secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  stripe-api-key: sk_test_EXAMPLE_KEY_DO_NOT_USE
  
  # 应用密钥
  jwt-secret: my-256-bit-secret-key-for-jwt-tokens
  encryption-key: my-encryption-key-for-sensitive-data
```

2. **Docker Registry Secret**：
```bash
# 创建私有镜像仓库凭证
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# 查看创建的Secret
kubectl get secret regcred -o yaml
```

```yaml
# 在Pod中使用私有镜像
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
  - name: regcred  # 引用凭证
  containers:
  - name: app
    image: myregistry.com/private/myapp:v1.0
```

3. **TLS Secret**：
```bash
# 创建TLS证书Secret
kubectl create secret tls app-tls-secret \
  --cert=app.crt \
  --key=app.key

# 或从Let's Encrypt证书创建
kubectl create secret tls letsencrypt-tls \
  --cert=fullchain.pem \
  --key=privkey.pem
```

```yaml
# 在Ingress中使用TLS证书
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: app-tls-secret  # 引用TLS Secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

4. **Service Account Token**：
```yaml
# K8s自动创建的Service Account Secret
apiVersion: v1
kind: Secret
metadata:
  name: default-token-abcd1
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
data:
  ca.crt: <base64-encoded-ca-certificate>
  namespace: ZGVmYXVsdA==  # base64编码的"default"
  token: <base64-encoded-jwt-token>
```

**Secret使用方式详解**：

1. **环境变量注入**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        env:
        # 方式1：单个Secret值注入
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: stripe-api-key
        
        # 方式2：批量导入（需谨慎，会暴露在环境变量中）
        envFrom:
        - secretRef:
            name: app-secret
            
        # 应用内使用示例
        command: ["sh", "-c"]
        args: [
          "echo Connecting to DB with user: $DB_USERNAME && 
           java -jar app.jar"
        ]
```

2. **文件挂载**（推荐安全方式）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app-files
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        volumeMounts:
        # 挂载所有Secret为文件
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
        # 挂载特定Secret文件
        - name: db-creds
          mountPath: /etc/db
          readOnly: true
        # 挂载TLS证书
        - name: tls-certs
          mountPath: /etc/ssl/private
          readOnly: true
          
        # 应用内读取文件
        command: ["sh", "-c"]
        args: [
          "DB_USER=$(cat /etc/secrets/db-username) &&
           DB_PASS=$(cat /etc/secrets/db-password) &&
           export DB_URL=\"mysql://$DB_USER:$DB_PASS@mysql-service:3306/myapp\" &&
           ./start-app.sh"
        ]
        
      volumes:
      # 挂载完整Secret
      - name: secret-volume
        secret:
          secretName: app-secret
          defaultMode: 0400  # 只读权限
      # 挂载特定key
      - name: db-creds
        secret:
          secretName: app-secret
          items:
          - key: db-username
            path: username
          - key: db-password
            path: password
          defaultMode: 0400
      # 挂载TLS证书
      - name: tls-certs
        secret:
          secretName: app-tls-secret
          defaultMode: 0400
```

#### Secret的加密机制与生命周期

**Secret数据是如何加密的？何时加密？何时解密？**

Secret在Kubernetes中的完整加密流程包含多个层次的保护机制：

**1. 创建时的编码（不是加密）**：
```bash
# 用户创建Secret时，数据会被Base64编码
echo -n "mypassword" | base64
# 输出：bXlwYXNzd29yZA==

# 这只是编码，不是加密！Base64可以轻易解码
echo "bXlwYXNzd29yZA==" | base64 -d
# 输出：mypassword
```

**Base64编码的作用**：
- 处理二进制数据（如TLS证书）
- YAML格式兼容性
- 统一存储格式
- **不是安全加密机制**

**2. 传输层加密（TLS）**：
```
用户 → kubectl → API Server
     ↓
   HTTPS/TLS加密传输
     ↓
API Server接收Secret
```

**3. etcd存储层加密**：

```yaml
# API Server加密配置（EncryptionConfiguration）
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # 加密提供者（按优先级排序）
    - aescbc:
        keys:
        - name: key1
          secret: <32-byte-base64-key>  # AES-256密钥
    - identity: {}  # 不加密（用于迁移）
```

**完整的加密/解密时序图**：

```
┌─────────────────────────────────────────────────────────────┐
│                    Secret生命周期                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 创建阶段                                                  │
│     kubectl create secret                                    │
│           ↓                                                  │
│     Base64编码（客户端）                                       │
│           ↓                                                  │
│     TLS加密传输 → API Server                                  │
│           ↓                                                  │
│     API Server验证和准入控制                                   │
│           ↓                                                  │
│     AES加密（如果启用etcd加密）                                 │
│           ↓                                                  │
│     存储到etcd                                                │
│                                                               │
│  2. 使用阶段                                                  │
│     Pod创建请求                                               │
│           ↓                                                  │
│     Scheduler分配节点                                         │
│           ↓                                                  │
│     Kubelet请求Secret（TLS加密）                              │
│           ↓                                                  │
│     API Server从etcd读取                                      │
│           ↓                                                  │
│     AES解密（如果启用加密）                                     │
│           ↓                                                  │
│     TLS加密传输给Kubelet                                      │
│           ↓                                                  │
│     Kubelet解码Base64                                        │
│           ↓                                                  │
│     挂载到Pod（tmpfs内存文件系统）                              │
│           ↓                                                  │
│     应用读取明文Secret                                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**4. 实际加密状态验证**：

```bash
# 检查etcd中是否启用加密
# 直接查看etcd中的数据（需要etcd访问权限）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  get /registry/secrets/default/my-secret | hexdump -C

# 如果未加密，可以看到Base64编码的数据
# 如果已加密，看到的是加密后的二进制数据
```

**5. 加密密钥轮换**：

```bash
# 1. 更新加密配置，添加新密钥
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key2  # 新密钥放在最前面
          secret: <new-32-byte-base64-key>
        - name: key1  # 旧密钥用于解密
          secret: <old-32-byte-base64-key>

# 2. 重启API Server使配置生效

# 3. 重新加密所有Secret
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -

# 4. 移除旧密钥
```

**6. 完整的加密/解密流程**：

```
┌─────────────────────────────────────────────────────────┐
│                Secret完整加密/解密流程                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. YAML文件中：                                          │
│     data.password: bXlzZWNyZXRwYXNzd29yZA==             │
│     (base64编码，不是加密)                                │
│                    ↓                                     │
│  2. etcd存储中：                                          │
│     未启用加密：base64数据                                 │
│     启用加密：AES加密的二进制                              │
│                    ↓                                     │
│  3. Pod请求Secret时：                                     │
│     kubelet → API Server (TLS加密请求)                   │
│                    ↓                                     │
│  4. API Server处理：                                      │
│     从etcd读取 → AES解密(如果启用) → 获得base64数据         │
│                    ↓                                     │
│  5. 网络传输：                                            │
│     API Server → kubelet (TLS加密传输base64数据)          │
│                    ↓                                     │
│  6. kubelet处理：                                        │
│     接收TLS解密 → base64解码 → 获得明文                    │
│                    ↓                                     │
│  7. 注入容器：                                            │
│     环境变量: mysecretpassword (明文)                     │
│     或文件挂载: /etc/secrets/password (明文内容)           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**详细解密时机和位置**：

**A. etcd数据解密（API Server中）**：
```go
// API Server读取Secret时的处理逻辑
func (s *Store) Get(key string) (*Secret, error) {
    encryptedData := etcd.Get(key)
    
    // 如果启用了加密，这里会自动解密
    if s.encryptionEnabled {
        decryptedData = aes.Decrypt(encryptedData, s.encryptionKey)
    }
    
    return parseSecret(decryptedData) // 返回包含base64数据的Secret
}
```

**B. TLS传输解密（网络层自动处理）**：
```bash
# TLS握手和数据传输
kubelet → API Server: TLS加密请求 "给我Secret数据"
API Server → kubelet: TLS加密响应 "这是base64编码的Secret"
# TLS层自动解密，kubelet收到明文的base64数据
```

**C. base64解码（kubelet中）**：
```go
// kubelet处理Secret挂载时的解码逻辑
func decodeSecretData(encodedData map[string][]byte) map[string][]byte {
    decodedData := make(map[string][]byte)
    
    for key, base64Value := range encodedData {
        // 这里进行base64解码
        plaintext, _ := base64.StdEncoding.DecodeString(string(base64Value))
        decodedData[key] = []byte(plaintext)
    }
    
    return decodedData // 返回明文数据注入容器
}
```

**实际演示流程**：
```bash
# 1. 创建Secret（base64编码）
kubectl create secret generic demo-secret --from-literal=password=mysecretpassword

# 2. 查看存储的base64数据
kubectl get secret demo-secret -o jsonpath='{.data.password}'
# 输出: bXlzZWNyZXRwYXNzd29yZA==

# 3. 手动解码验证
echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 -d
# 输出: mysecretpassword

# 4. 容器中查看环境变量（kubelet已自动解码）
kubectl exec -it pod-name -- printenv SECRET_PASSWORD
# 输出: mysecretpassword (明文，无需应用再解码)

# 5. 容器中查看文件内容（kubelet已自动解码）
kubectl exec -it pod-name -- cat /etc/secrets/password
# 输出: mysecretpassword (明文文件内容)
```

**关键理解点**：
1. **base64不是加密**：只是编码格式，容器内是明文
2. **真正的加密**：etcd存储层（可选）+ TLS传输层
3. **自动解密**：kubelet自动处理所有解密和解码
4. **应用无需解密**：容器内直接使用明文数据

#### Secret凭证的创建流程与原理

**1. Docker Registry凭证创建原理**：

```bash
# kubectl create secret docker-registry 实际做了什么？
```

**内部流程分解**：

```json
// 1. kubectl收集凭证信息
{
  "auths": {
    "myregistry.com": {
      "username": "myuser",
      "password": "mypassword",
      "email": "myemail@example.com",
      "auth": "bXl1c2VyOm15cGFzc3dvcmQ="  // base64(username:password)
    }
  }
}

// 2. 转换为dockerconfigjson格式
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "regcred"
  },
  "type": "kubernetes.io/dockerconfigjson",
  "data": {
    ".dockerconfigjson": "eyJhdXRocyI6ey..."  // 整个JSON的base64编码
  }
}
```

**实际使用流程**：

```
┌─────────────────────────────────────────────────────┐
│              拉取私有镜像流程                          │
├─────────────────────────────────────────────────────┤
│                                                      │
│  1. Pod定义包含imagePullSecrets                      │
│     ↓                                               │
│  2. Kubelet准备拉取镜像                              │
│     ↓                                               │
│  3. Kubelet从Secret获取凭证                          │
│     ↓                                               │
│  4. 解析.dockerconfigjson                           │
│     ↓                                               │
│  5. 提取对应registry的auth信息                        │
│     ↓                                               │
│  6. Docker/Containerd使用凭证                        │
│     ↓                                               │
│  7. 向Registry发送认证请求                           │
│     Authorization: Basic <base64>                   │
│     ↓                                               │
│  8. Registry验证并返回镜像                           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**2. TLS证书Secret的创建与使用**：

```bash
# 证书创建流程
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com/O=example"

# Secret创建时的处理
kubectl create secret tls my-tls --key=tls.key --cert=tls.crt
```

**内部数据结构**：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls
type: kubernetes.io/tls  # 特殊类型
data:
  tls.crt: <base64-encoded-certificate>  # 证书
  tls.key: <base64-encoded-private-key>   # 私钥
```

**Ingress Controller使用TLS Secret的流程**：

```
┌──────────────────────────────────────────────────┐
│           TLS终止流程                              │
├──────────────────────────────────────────────────┤
│                                                   │
│  1. Ingress Controller启动                        │
│     ↓                                            │
│  2. 监听Ingress资源变化                           │
│     ↓                                            │
│  3. 发现TLS配置引用Secret                         │
│     ↓                                            │
│  4. 从API Server获取Secret                        │
│     ↓                                            │
│  5. 解码tls.crt和tls.key                         │
│     ↓                                            │
│  6. 加载到Nginx/HAProxy配置                       │
│     ↓                                            │
│  7. 客户端HTTPS请求到达                           │
│     ↓                                            │
│  8. 使用私钥解密客户端数据                         │
│     ↓                                            │
│  9. 转发明文请求到后端Pod                          │
│                                                   │
└──────────────────────────────────────────────────┘
```

**3. Service Account Token的自动创建机制**：

```yaml
# ServiceAccount创建时自动生成Token
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
```

**自动创建流程**：

```
┌────────────────────────────────────────────────────┐
│         Service Account Token生成流程                │
├────────────────────────────────────────────────────┤
│                                                     │
│  1. 创建ServiceAccount                              │
│     ↓                                              │
│  2. Token Controller监听到SA创建                     │
│     ↓                                              │
│  3. 生成RSA密钥对（如果不存在）                        │
│     ↓                                              │
│  4. 创建JWT Token                                  │
│     Header: {"alg": "RS256", "typ": "JWT"}        │
│     Payload: {                                    │
│       "iss": "kubernetes/serviceaccount",         │
│       "sub": "system:serviceaccount:default:my-sa",│
│       "aud": ["https://kubernetes.default.svc"], │
│       "exp": <expiry-time>,                       │
│       "iat": <issued-at>,                         │
│       "nbf": <not-before>                         │
│     }                                             │
│     ↓                                              │
│  5. 使用私钥签名Token                               │
│     ↓                                              │
│  6. 创建Secret存储Token                             │
│     ↓                                              │
│  7. 关联到ServiceAccount                           │
│                                                     │
└────────────────────────────────────────────────────┘
```

#### 凭证管理的原理与最佳实践

**Kubernetes Secret管理的核心原理**：

**1. 凭证的生命周期管理**：

```
┌─────────────────────────────────────────────────────────────┐
│                 凭证生命周期全流程                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  创建阶段：                                                   │
│    └── 凭证生成 → 加密存储 → 版本控制 → 分发授权                │
│                                                              │
│  使用阶段：                                                   │
│    └── 身份验证 → 权限检查 → 安全注入 → 监控记录                │
│                                                              │
│  维护阶段：                                                   │
│    └── 定期轮换 → 过期检测 → 安全扫描 → 备份恢复                │
│                                                              │
│  销毁阶段：                                                   │
│    └── 安全删除 → 访问撤销 → 审计记录 → 清理验证                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**2. 凭证层级管理策略**：

```yaml
# 按环境分层管理
# 开发环境 - 宽松策略
apiVersion: v1
kind: Secret
metadata:
  name: dev-db-secret
  namespace: development
  labels:
    environment: dev
    security-level: low
type: Opaque
stringData:
  username: "dev_user"
  password: "simple_password"
  database: "dev_db"

---
# 生产环境 - 严格策略
apiVersion: v1
kind: Secret
metadata:
  name: prod-db-secret
  namespace: production
  labels:
    environment: prod
    security-level: high
  annotations:
    rotation-policy: "monthly"
    last-rotated: "2024-01-15"
    created-by: "vault-operator"
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-complex-password>
  database: <base64-encoded-database>
```

**3. 多种凭证类型的统一管理**：

```yaml
# 1. 数据库凭证
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  annotations:
    secret-type: "database"
    source: "vault"
type: Opaque
stringData:
  # 主库
  master-host: "prod-db-master.internal"
  master-username: "app_user"
  master-password: "complex-generated-password-123"
  
  # 只读库
  slave-host: "prod-db-slave.internal"
  slave-username: "readonly_user"
  slave-password: "readonly-password-456"
  
  # 连接参数
  max-connections: "100"
  ssl-mode: "require"

---
# 2. API密钥集合
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  annotations:
    secret-type: "api-keys"
type: Opaque
stringData:
  # 支付网关
  stripe-publishable: "pk_live_xxxxxxxxxx"
  stripe-secret: "sk_live_xxxxxxxxxx"
  
  # 邮件服务
  sendgrid-api-key: "SG.xxxxxxxxxx"
  
  # 云服务
  aws-access-key: "AKIA xxxxxxxxxx"
  aws-secret-key: "xxxxxxxxxx"
  
  # 监控服务
  datadog-api-key: "xxxxxxxxxx"

---
# 3. OAuth2/JWT配置
apiVersion: v1
kind: Secret
metadata:
  name: oauth-config
  annotations:
    secret-type: "oauth"
type: Opaque
stringData:
  # JWT签名密钥
  jwt-secret: "super-secure-256-bit-secret-key"
  
  # OAuth2 客户端
  oauth-client-id: "client-id-12345"
  oauth-client-secret: "client-secret-abcdef"
  
  # Google OAuth
  google-client-id: "google-oauth-client-id"
  google-client-secret: "google-oauth-secret"
```

**4. 凭证安全传递机制**：

```yaml
# 安全的凭证注入模式
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      # 1. 使用专用ServiceAccount
      serviceAccountName: secure-app-sa
      
      # 2. 安全上下文
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        
      containers:
      - name: app
        image: myapp:latest
        
        # 3. 环境变量方式（简单配置）
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: master-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: master-password
              
        # 4. 文件挂载方式（复杂配置）
        volumeMounts:
        - name: api-keys-volume
          mountPath: "/etc/secrets/api"
          readOnly: true
        - name: oauth-volume
          mountPath: "/etc/secrets/oauth"
          readOnly: true
          
        # 5. 安全配置
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            
      volumes:
      # 使用tmpfs确保Secret只在内存中
      - name: api-keys-volume
        secret:
          secretName: api-keys
          defaultMode: 0400  # 只读权限
      - name: oauth-volume
        secret:
          secretName: oauth-config
          defaultMode: 0400
```

**5. 外部密钥管理系统集成**：

```yaml
# 使用External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-role"
          serviceAccountRef:
            name: external-secrets-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-secret
  namespace: production
spec:
  refreshInterval: 15m  # 定期同步
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: database/production
      property: username
  - secretKey: password
    remoteRef:
      key: database/production
      property: password
```

**6. 凭证轮换自动化策略**：

```yaml
# 自动轮换CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-rotation
  namespace: production
spec:
  schedule: "0 2 * * 0"  # 每周日凌晨2点
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator
          containers:
          - name: rotator
            image: secret-rotator:latest
            env:
            - name: VAULT_ADDR
              value: "https://vault.company.com"
            - name: ROTATION_POLICY
              value: "database,api-keys"
            command:
            - /bin/sh
            - -c
            - |
              # 1. 从Vault生成新凭证
              NEW_DB_PASSWORD=$(vault kv put -format=json secret/database/production password=$(openssl rand -base64 32) | jq -r '.data.password')
              
              # 2. 更新数据库用户密码
              mysql -h ${DB_HOST} -u admin -p${ADMIN_PASSWORD} -e "ALTER USER 'app_user'@'%' IDENTIFIED BY '${NEW_DB_PASSWORD}';"
              
              # 3. 更新Kubernetes Secret
              kubectl patch secret database-credentials -p "{\"data\":{\"password\":\"$(echo -n ${NEW_DB_PASSWORD} | base64)\"}}"
              
              # 4. 触发应用重启
              kubectl rollout restart deployment/myapp
              
              # 5. 验证新凭证
              sleep 30
              kubectl logs deployment/myapp --tail=50 | grep -q "Database connection successful" && echo "Rotation successful"
          restartPolicy: OnFailure
```

**7. 凭证监控与审计**：

```yaml
# 审计策略
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 记录Secret的所有操作
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  omitStages:
  - RequestReceived
  
# 详细记录生产环境Secret访问
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  namespaces: ["production"]
  
# 记录ServiceAccount token的使用
- level: Request
  users: ["system:serviceaccount:*"]
  verbs: ["get", "list"]
  resources:
  - group: ""
    resources: ["secrets"]
```

**8. 零信任安全模型下的Secret管理**：

```yaml
# Pod Security Policy for Secrets
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-secret-access
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'secret'
    - 'emptyDir'
    - 'projected'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

#### Secret安全最佳实践

**1. 启用etcd加密**：

```bash
# 1. 创建加密配置
cat > /etc/kubernetes/encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    - configmaps
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: $(head -c 32 /dev/urandom | base64)
    - identity: {}
EOF

# 2. 配置API Server
# 在API Server启动参数中添加：
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# 3. 验证加密
kubectl create secret generic test-secret --from-literal=key=value
# 直接查看etcd中的数据应该是加密的
```

**2. 使用外部密钥管理系统**：

```yaml
# 使用HashiCorp Vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-secrets
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "my-role"
    objects: |
      - objectName: "database-password"
        secretPath: "secret/data/database"
        secretKey: "password"
```

**3. Secret扫描和轮换**：

```bash
# 扫描过期的证书
kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | select(.type=="kubernetes.io/tls") | 
  "\(.metadata.namespace)/\(.metadata.name)"' | \
  while read secret; do
    kubectl get secret $secret -o jsonpath='{.data.tls\.crt}' | \
    base64 -d | openssl x509 -noout -enddate
  done

# 自动轮换密码
cronjob:
  schedule: "0 2 * * 0"  # 每周日凌晨2点
  script: |
    NEW_PASSWORD=$(openssl rand -base64 32)
    kubectl patch secret db-secret -p \
      "{\"data\":{\"password\":\"$(echo -n $NEW_PASSWORD | base64)\"}}"
    # 触发应用重启以使用新密码
    kubectl rollout restart deployment/myapp
```

**4. RBAC限制Secret访问**：

```yaml
# 最小权限原则
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]  # 只能访问特定Secret
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-app-secret
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**5. 审计Secret访问**：

```yaml
# 审计策略
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 记录所有Secret的访问
  - level: RequestResponse
    omitStages:
    - RequestReceived
    resources:
    - group: ""
      resources: ["secrets"]
    namespaces: ["production"]
```

#### Secret与SpringBoot应用集成详解

**实际问题**：数据库密码、API密钥等敏感信息如何安全地注入到SpringBoot应用中？

**集成方式**：

**方式1：环境变量注入（简单但需注意安全性）**

1. **创建Secret**：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: springboot-secret
  namespace: production
type: Opaque
stringData:
  # 数据库凭证
  spring.datasource.password: "prod-db-password-123"
  spring.redis.password: "redis-password-456"
  
  # JWT密钥
  app.jwt.secret: "my-super-secret-256-bit-jwt-key-for-production"
  
  # 第三方API密钥
  app.stripe.api-key: "sk_live_xxxxxxxxxxxxxxxxxxxx"
  app.sendgrid.api-key: "SG.xxxxxxxxxxxxxxxxxxxx"
```

2. **SpringBoot配置**：
```yaml
# application.yaml（容器内）
spring:
  datasource:
    url: jdbc:mysql://mysql-service:3306/myapp_prod
    username: appuser
    password: ${SPRING_DATASOURCE_PASSWORD}  # 从Secret注入
    
  redis:
    host: redis-service
    port: 6379
    password: ${SPRING_REDIS_PASSWORD}  # 从Secret注入

app:
  jwt:
    secret: ${APP_JWT_SECRET}  # 从Secret注入
    expiration: 3600000
  stripe:
    api-key: ${APP_STRIPE_API_KEY}  # 从Secret注入
```

3. **Deployment配置**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-secure-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        env:
        # 从Secret注入敏感环境变量
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: springboot-secret
              key: spring.datasource.password
        - name: SPRING_REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: springboot-secret
              key: spring.redis.password
        - name: APP_JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: springboot-secret
              key: app.jwt.secret
        - name: APP_STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: springboot-secret
              key: app.stripe.api-key
```

**方式2：文件挂载（生产环境推荐）**

1. **创建Secret配置文件**：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: springboot-config-secret
type: Opaque
stringData:
  application-secrets.yaml: |
    spring:
      datasource:
        password: prod-db-password-123
      redis:
        password: redis-password-456
    app:
      jwt:
        secret: my-super-secret-256-bit-jwt-key-for-production
      encryption:
        key: my-aes-256-encryption-key-for-sensitive-data
      stripe:
        api-key: sk_live_xxxxxxxxxxxxxxxxxxxx
      aws:
        access-key: AKIAIOSFODNN7EXAMPLE
        secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

2. **Deployment挂载Secret文件**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-file-secrets
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        
        volumeMounts:
        # 挂载Secret配置文件
        - name: secret-config
          mountPath: /app/secrets
          readOnly: true
          
        # SpringBoot启动参数
        args:
        - "--spring.config.location=classpath:/application.yaml,/app/secrets/application-secrets.yaml"
        - "--spring.profiles.active=production"
        
      volumes:
      - name: secret-config
        secret:
          secretName: springboot-config-secret
          defaultMode: 0400  # 只读权限
```

#### Secret到SpringBoot的内部流程原理

**安全数据流转的完整过程**：

```
┌─────────────────────────────────────────────────────────┐
│                   K8s集群层面                             │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐                    │
│  │   Secret    │    │    Pod      │                    │
│  │(Base64编码)  │    │             │                    │
│  └─────────────┘    └─────────────┘                    │
│         │                   │                          │
│         └───────────────────┘                          │
│                                                         │
└────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────┐
│                  容器运行时层面                           │
│                                                         │
│  kubelet解码Secret + 设置严格权限                        │
│         │                                               │
│         ↓                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              容器内部                             │   │
│  │  环境变量: SPRING_DATASOURCE_PASSWORD=xxx        │   │
│  │  文件挂载: /app/secrets/application-secrets.yaml │   │
│  │  权限: 400 (只读，仅容器用户可访问)                │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│                SpringBoot应用层面                        │
│                                                         │
│  1. JVM启动 → 读取敏感环境变量（内存中）                  │
│  2. SpringBoot配置加载 → 解析Secret文件                 │
│  3. 敏感配置注入Bean → 建立安全连接                      │
│  4. 运行时保护 → 内存中处理，不写日志                    │
└─────────────────────────────────────────────────────────┘
```

**Secret安全处理的详细步骤**：

**1. etcd存储层的安全**：
```bash
# Secret在etcd中的存储路径
/registry/secrets/<namespace>/<secret-name>

# 未加密时的存储（不推荐）
{
  "apiVersion": "v1",
  "data": {
    "password": "bXlwYXNzd29yZA=="  # Base64编码，可解码
  },
  "kind": "Secret",
  "metadata": {...}
}

# 启用加密后的存储
k8s:enc:aescbc:v1:key1:<encrypted-data>  # AES-256加密
```

**2. 传输过程的安全**：
```
客户端创建Secret → API Server
        ↓
  TLS 1.2/1.3 加密
  双向证书验证
        ↓
API Server → etcd
        ↓
  内部TLS通信
  mTLS双向认证
        ↓
API Server → Kubelet
        ↓
  TLS加密通道
  Bearer Token认证
```

**3. 容器内的安全存储**：
```bash
# Secret挂载到容器时的处理
kubectl exec -it app-pod -- mount | grep secrets
tmpfs on /etc/secrets type tmpfs (ro,relatime,size=65536k)

# tmpfs特点：
# 1. 存储在RAM中，不写入磁盘
# 2. Pod删除时自动清除
# 3. 无法被其他容器访问
# 4. 支持大小限制

# 文件权限
kubectl exec -it app-pod -- ls -la /etc/secrets/
-r-------- 1 1000 1000 23 Dec  1 10:00 db-password  # 400权限
```

**SpringBoot内部处理Secret的原理**：

```java
// SpringBoot如何安全处理Secret
@Configuration
@EnableConfigurationProperties
public class SecureConfigProcessor {
    
    // 1. 环境变量加载器
    @Component
    public class EnvironmentSecretLoader implements EnvironmentPostProcessor {
        @Override
        public void postProcessEnvironment(ConfigurableEnvironment environment,
                                          SpringApplication application) {
            // 从系统环境变量读取Secret
            Map<String, Object> secrets = new HashMap<>();
            
            // 读取K8s注入的环境变量
            String dbPassword = System.getenv("SPRING_DATASOURCE_PASSWORD");
            if (dbPassword != null) {
                // 存储在内存中，不记录日志
                secrets.put("spring.datasource.password", dbPassword);
                
                // 清除环境变量（可选的额外安全措施）
                // 注意：这在Java中无法真正清除系统环境变量
                // 但可以防止通过反射访问
                clearSensitiveData(dbPassword);
            }
            
            // 添加到Spring环境
            environment.getPropertySources().addFirst(
                new MapPropertySource("k8s-secrets", secrets)
            );
        }
        
        private void clearSensitiveData(String data) {
            // 使用char[]而不是String存储密码
            char[] password = data.toCharArray();
            // 使用后清零
            Arrays.fill(password, '\0');
        }
    }
    
    // 2. 文件Secret加载器
    @Component
    public class FileSecretLoader {
        @PostConstruct
        public void loadSecrets() {
            Path secretPath = Paths.get("/app/secrets/application-secrets.yaml");
            if (Files.exists(secretPath)) {
                try {
                    // 读取文件内容
                    byte[] content = Files.readAllBytes(secretPath);
                    
                    // 解析YAML
                    Yaml yaml = new Yaml();
                    Map<String, Object> secrets = yaml.load(
                        new String(content, StandardCharsets.UTF_8)
                    );
                    
                    // 清除原始字节数组
                    Arrays.fill(content, (byte) 0);
                    
                    // 处理Secret数据
                    processSecrets(secrets);
                } catch (IOException e) {
                    // 安全日志，不输出敏感信息
                    logger.error("Failed to load secrets file", e);
                }
            }
        }
    }
    
    // 3. DataSource安全配置
    @Bean
    public DataSource dataSource(
            @Value("${spring.datasource.url}") String url,
            @Value("${spring.datasource.username}") String username,
            @Value("${spring.datasource.password}") String password) {
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        
        // 使用密码
        config.setPassword(password);
        
        // 配置连接池安全选项
        config.setConnectionTestQuery("SELECT 1");
        config.setLeakDetectionThreshold(60000);
        
        // 防止密码泄露到日志
        config.addDataSourceProperty("logger.level", "OFF");
        
        return new HikariDataSource(config);
    }
    
    // 4. 防止密码泄露到日志
    @Bean
    public static BeanPostProcessor secretMaskingProcessor() {
        return new BeanPostProcessor() {
            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) {
                if (bean instanceof DataSource) {
                    // 包装DataSource，隐藏toString中的密码
                    return new SecureDataSourceWrapper((DataSource) bean);
                }
                return bean;
            }
        };
    }
}

// 安全包装器，防止密码泄露
class SecureDataSourceWrapper implements DataSource {
    private final DataSource delegate;
    
    @Override
    public String toString() {
        return "SecureDataSource[url=" + getUrl() + ", user=***]";
    }
    
    // 其他方法委托给原始DataSource
}
```

**实际验证Secret安全性**：

```bash
# 1. 验证Secret在容器内的存储方式
kubectl exec -it springboot-pod -- df -h | grep secrets
tmpfs           64M     0   64M   0% /etc/secrets  # 内存存储

# 2. 检查进程环境变量（确认敏感信息处理）
kubectl exec -it springboot-pod -- cat /proc/1/environ | tr '\0' '\n' | grep -i password
# 应该看到环境变量，但应用启动后可能已清理

# 3. 验证JVM内存中的密码处理
kubectl exec -it springboot-pod -- jcmd 1 VM.system_properties | grep -i password
# 不应该看到明文密码

# 4. 检查应用日志
kubectl logs springboot-pod | grep -i password
# 不应该有密码信息

# 5. 内存转储分析（生产环境慎用）
kubectl exec -it springboot-pod -- jmap -dump:format=b,file=/tmp/heap.dump 1
kubectl cp springboot-pod:/tmp/heap.dump ./heap.dump
# 使用MAT或其他工具分析，查找敏感信息
```

**Secret生命周期管理**：

```yaml
# 1. Secret版本控制
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-v2
  labels:
    version: "v2"
    app: myapp
  annotations:
    kubernetes.io/change-cause: "Update database password"
type: Opaque
data:
  password: <new-base64-password>

---
# 2. 使用Sealed Secrets（加密的Secret）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secret
spec:
  encryptedData:
    password: AgA...  # 公钥加密的数据，可安全存储在Git

---
# 3. External Secrets Operator集成
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/database
      property: password
```

**故障排查和调试**：

```bash
# 1. Secret无法创建
kubectl describe secret my-secret
# 检查Events部分

# 2. Pod无法访问Secret
kubectl describe pod my-pod
# 检查Volume挂载和权限

# 3. Secret内容验证
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# 4. RBAC权限检查
kubectl auth can-i get secrets --as=system:serviceaccount:default:myapp-sa

# 5. 审计日志检查
kubectl logs -n kube-system kube-apiserver-master | grep secret
```

#### 持久化存储详解

**为什么需要持久化存储？**

容器和Pod的特点：
- Pod是临时的，删除后数据丢失
- 容器重启后，本地数据消失
- 多个Pod需要共享数据

持久化存储解决方案：
```
应用数据持久化需求：
- 数据库文件
- 用户上传的文件
- 日志文件
- 配置文件
- 缓存数据
```

**K8s存储架构**：

```
┌─────────────────────────────────────────────┐
│                 Pod                         │
│  ┌─────────────────────────────────────┐   │
│  │          Container                   │   │
│  │  /var/lib/mysql ← 挂载点             │   │
│  └─────────────────────────────────────┘   │
└──────────────┬──────────────────────────────┘
               ↓ volumeMounts
┌──────────────────────────────────────────────┐
│              Volume                          │
│  ┌──────────────────────────────────────┐   │
│  │        PersistentVolume (PV)          │   │
│  │     ┌─────────────────────────┐      │   │
│  │     │    底层存储               │      │   │
│  │     │  (NFS/AWS EBS/...)      │      │   │
│  │     └─────────────────────────┘      │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

**存储相关概念对比**：

| 概念 | 说明 | 生命周期 | 使用场景 |
|------|------|---------|----------|
| **Volume** | Pod级别存储卷 | 与Pod相同 | 临时数据、Pod内容器共享 |
| **PersistentVolume (PV)** | 集群级别存储资源 | 独立于Pod | 持久化数据 |
| **PersistentVolumeClaim (PVC)** | 存储申请/消费 | 独立于Pod | 应用申请存储 |
| **StorageClass** | 动态存储配置 | 集群级别 | 自动创建PV |

**Volume类型详解**：

1. **emptyDir（临时存储）**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: busybox
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: sidecar
    image: busybox
    volumeMounts:
    - name: cache-volume  # 两个容器共享
      mountPath: /shared
  volumes:
  - name: cache-volume
    emptyDir: {}  # Pod删除时数据消失
```

2. **hostPath（节点路径挂载）**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-path-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-logs
      mountPath: /var/log/nginx
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log/pods/nginx  # 节点上的路径
      type: DirectoryOrCreate    # 目录不存在则创建
```

**注意**：hostPath有安全风险，生产环境慎用。

3. **ConfigMap/Secret作为Volume**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
    - name: secret-volume
      mountPath: /etc/ssl/certs
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
  - name: secret-volume
    secret:
      secretName: tls-secret
```

**PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)**：

PV和PVC的关系：
```
开发者（应用）          集群管理员（运维）
      ↓                      ↓
   创建PVC ←───────绑定────→ 创建PV
  (申请存储)              (提供存储)
      ↓                      ↓
  Pod使用PVC              PV对接存储后端
```

1. **静态PV创建**：
```yaml
# 管理员创建PV
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce    # 单节点读写
  persistentVolumeReclaimPolicy: Retain  # 回收策略
  storageClassName: manual
  nfs:  # 后端存储（NFS示例）
    server: nfs.example.com
    path: /data/mysql

---
# 开发者创建PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi  # 申请5GB，小于等于PV容量

---
# Pod使用PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

2. **动态PV创建（StorageClass）**：
```yaml
# 管理员配置StorageClass
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # AWS EBS存储
parameters:
  type: gp3           # SSD类型
  iopsPerGB: "10"     # IOPS配置
  encrypted: "true"   # 加密
allowVolumeExpansion: true  # 允许扩容
reclaimPolicy: Delete       # 删除策略

---
# 开发者创建PVC，自动创建PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd  # 指定StorageClass
  resources:
    requests:
      storage: 20Gi  # K8s自动创建20GB的EBS卷

---
# Pod使用动态PVC
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # StatefulSet专用
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

**访问模式对比**：

| 访问模式 | 缩写 | 描述 | 使用场景 |
|---------|------|------|----------|
| **ReadWriteOnce** | RWO | 单节点读写 | 数据库、单实例应用 |
| **ReadOnlyMany** | ROX | 多节点只读 | 静态资源、共享配置 |
| **ReadWriteMany** | RWX | 多节点读写 | 共享文件系统、集群应用 |

**存储最佳实践**：

1. **存储规划**：
```yaml
# 不同工作负载的存储需求
# 数据库：高IOPS，低延迟
storageClassName: fast-ssd
# 日志：大容量，成本敏感
storageClassName: standard
# 备份：大容量，低成本
storageClassName: cold-storage
```

2. **存储监控**：
```bash
# 查看存储使用情况
kubectl get pv
kubectl get pvc
kubectl describe pvc my-pvc

# 监控存储空间
kubectl top pod --use-protocol-buffers
df -h  # 在Pod内查看磁盘使用
```

3. **存储扩容**：
```bash
# 扩容PVC（需要StorageClass支持）
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# 查看扩容进度
kubectl describe pvc my-pvc
```

---



---

## 第一篇总结

通过本篇的学习，您已经掌握了Kubernetes的核心基础知识：

### 🎯 主要收获

**概念理解**：
- 理解了Kubernetes作为容器编排平台的核心价值
- 掌握了集群架构中Master和Worker节点的分工
- 了解了声明式管理和期望状态的设计理念

**架构认知**：
- 熟悉了API Server、etcd、Scheduler等核心组件的作用
- 理解了kubelet、kube-proxy在Worker节点的工作机制
- 掌握了Kubernetes网络模型的基本原理

**资源对象**：
- 掌握了Pod作为最小调度单元的概念和配置
- 理解了Service的服务发现和负载均衡机制
- 学会了ConfigMap和Secret的配置管理方式
- 了解了Deployment、StatefulSet等工作负载管理

### 🚀 下一步学习建议

有了这些基础知识，您现在可以：

1. **动手实践** - 搭建本地Kubernetes环境进行实验
2. **深入配置** - 学习YAML配置文件的编写技巧
3. **实战项目** - 尝试部署真实的应用程序
4. **继续学习** - 阅读后续文章深入了解实战应用

### 📚 相关资源

- **继续阅读**：[第二篇：配置管理与实战应用](K8S_PART2_PRACTICE.md)
- **官方文档**：[Kubernetes官方文档](https://kubernetes.io/docs/)
- **实验环境**：推荐使用minikube或Docker Desktop进行本地实验

**记住**：Kubernetes的学习是一个渐进的过程，理论与实践相结合才能真正掌握这项技术。现在您已经具备了扎实的理论基础，是时候开始实战了！
