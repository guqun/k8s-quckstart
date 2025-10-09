# Kubernetes快速上手教程 - 第二篇：配置管理与实战应用

## 序言

欢迎来到Kubernetes快速上手教程的第二篇！在掌握了基础概念与架构后，现在我们将进入实战阶段。

这一篇将带您深入学习：
- **YAML配置文件编写** - 掌握Kubernetes资源配置的核心技能
- **实战案例集** - 通过真实场景学会部署和管理应用

从理论到实践，从概念到应用，这部分内容将帮助您具备在实际项目中使用Kubernetes的能力。

**学习前提**：
- 已完成第一篇的学习，了解Kubernetes基础概念
- 具备YAML基础语法知识
- 有Docker容器使用经验

**实验环境建议**：
- 本地Kubernetes环境（minikube/Docker Desktop）
- 云服务商提供的Kubernetes集群
- 在线Kubernetes实验环境

**预计阅读时间**：3-4小时（含动手实践）

---

## 目录

4. [YAML配置文件全解析](#4-yaml配置文件全解析)
5. [实战案例集](#5-实战案例集)

---## 4. YAML配置文件全解析

### 4.1 YAML基础语法

**YAML语法规则**：
```yaml
# 1. 缩进表示层级（必须使用空格，不能用Tab）
apiVersion: v1
kind: Pod
metadata:
  name: my-pod        # 2级缩进
  labels:
    app: web          # 3级缩进
    version: v1
spec:
  containers:
  - name: app         # 数组第一个元素
    image: nginx
  - name: sidecar     # 数组第二个元素
    image: busybox

# 2. 数据类型
string_value: "Hello World"
number_value: 42
boolean_value: true
null_value: null
multiline_string: |
  这是多行字符串
  保持换行符
folded_string: >
  这是折叠字符串
  会合并成一行

# 3. 数组表示
fruits:
  - apple
  - banana
  - orange

# 或者内联格式
fruits: [apple, banana, orange]
```

### 4.2 K8s资源配置结构

**标准K8s资源结构**：
```yaml
apiVersion: <API版本>    # 必需：资源API版本
kind: <资源类型>         # 必需：资源类型
metadata:               # 必需：元数据
  name: <资源名称>      # 必需：资源名称
  namespace: <命名空间>  # 可选：默认default
  labels:               # 可选：标签
    key: value
  annotations:          # 可选：注解
    key: value
spec:                   # 可选：期望状态规格
  # 具体配置取决于资源类型
status:                 # 只读：当前状态（系统维护）
  # K8s系统自动更新
```

**常用API版本对照**：
```yaml
# 核心资源
v1:
  - Pod
  - Service
  - ConfigMap
  - Secret
  - PersistentVolume
  - PersistentVolumeClaim

# 应用资源
apps/v1:
  - Deployment
  - StatefulSet
  - DaemonSet
  - ReplicaSet

# 网络资源
networking.k8s.io/v1:
  - Ingress
  - NetworkPolicy

# 批处理
batch/v1:
  - Job
batch/v1beta1:
  - CronJob

# 存储
storage.k8s.io/v1:
  - StorageClass

# 自动扩缩容
autoscaling/v2:
  - HorizontalPodAutoscaler
```

### 4.3 配置文件最佳实践

**1. 文件组织结构**：
```bash
k8s-manifests/
├── namespace.yaml           # 命名空间
├── configmaps/
│   ├── app-config.yaml
│   └── nginx-config.yaml
├── secrets/
│   └── app-secrets.yaml
├── deployments/
│   ├── frontend.yaml
│   ├── backend.yaml
│   └── database.yaml
├── services/
│   ├── frontend-svc.yaml
│   ├── backend-svc.yaml
│   └── database-svc.yaml
├── ingress/
│   └── app-ingress.yaml
└── storage/
    ├── storageclass.yaml
    └── pvcs.yaml
```

**2. 标签和选择器规范**：
```yaml
# 推荐的标签规范
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: myapp-system
    app.kubernetes.io/managed-by: kubectl
    environment: production
    tier: frontend

# Deployment中的选择器
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: myapp-prod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: myapp-prod
        app.kubernetes.io/version: "1.0"
```

**3. 资源限制配置**：
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:        # 最小资源需求
        memory: "128Mi"
        cpu: "100m"
      limits:          # 最大资源限制
        memory: "256Mi"
        cpu: "200m"
    livenessProbe:     # 存活探针
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:    # 就绪探针
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**4. 配置文件模板化**：
```yaml
# 使用环境变量替换
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: ${REPLICA_COUNT}
  template:
    spec:
      containers:
      - name: app
        image: ${IMAGE_NAME}:${IMAGE_TAG}
        env:
        - name: ENV
          value: ${ENVIRONMENT}
```

### 4.4 多资源文件管理

**1. 单文件多资源**：
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    app:
      name: myapp
      port: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    # ... deployment配置

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**2. Kustomize配置管理**：
```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

images:
- name: myapp
  newTag: v1.2.0

replicas:
- name: myapp
  count: 5

configMapGenerator:
- name: app-config
  files:
  - config.properties
```

**3. Helm Chart结构**：
```bash
mychart/
├── Chart.yaml          # Chart元数据
├── values.yaml         # 默认配置值
├── templates/          # 模板文件
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── charts/            # 依赖Chart
```

---

## 5. 实战案例集

### 5.1 Web应用全栈部署案例

本节通过一个完整的三层Web应用架构，学习Kubernetes**资源编排、服务发现、数据持久化、配置管理**等核心概念的实际应用。

#### **架构设计与Kubernetes原理**

**应用架构**：React前端 + Spring Boot后端 + MySQL数据库

**Kubernetes核心概念映射**：

| 应用层 | Kubernetes资源 | 核心功能 | 设计原理 |
|--------|---------------|----------|----------|
| **前端层** | Deployment + Service + Ingress | 静态资源服务 | 无状态Pod，水平扩展 |
| **后端层** | Deployment + Service + ConfigMap | 业务逻辑处理 | 无状态Pod，负载均衡 |
| **数据层** | StatefulSet + PVC + Service | 数据持久化 | 有状态Pod，数据绑定 |
| **配置层** | ConfigMap + Secret | 配置分离 | 配置与镜像解耦 |

**流量与数据流向**：
```
Internet → Ingress(L7负载均衡) → Frontend Service(ClusterIP) → Frontend Pods
    ↓ API调用(内部DNS)
Backend Service(ClusterIP) → Backend Pods → MySQL Service → MySQL Pod(StatefulSet)
    ↓ 配置注入                    ↓ 数据持久化
ConfigMap/Secret → 环境变量 → PVC → 物理存储
```

#### **Kubernetes部署模式分析**

**1. 无状态应用部署模式**（前端/后端）
- **Deployment控制器**：确保Pod副本数量，支持滚动更新
- **Service抽象**：提供稳定的网络端点，自动负载均衡
- **水平扩展**：通过调整replicas实现弹性伸缩

**2. 有状态应用部署模式**（数据库）
- **StatefulSet控制器**：提供稳定的网络标识和存储
- **PVC持久化**：数据与Pod生命周期解耦
- **顺序部署**：保证有状态应用的启动顺序

**3. 配置管理模式**
- **ConfigMap**：存储非敏感配置，支持热更新
- **Secret**：存储敏感信息，base64编码，权限控制
- **环境变量注入**：运行时配置注入，支持动态配置

#### **部署框架与最佳实践**

**命名空间隔离策略**：
```yaml
# 环境隔离示例
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-prod
  labels:
    environment: production
    project: webapp
```

**服务发现配置**：
```yaml
# 后端服务发现前端的典型配置
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 8080    # 集群内访问端口
    targetPort: 8080  # Pod实际端口
```

**配置管理策略**：
```yaml
# 配置分离示例
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "mysql-service"
  database.port: "3306"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  database.password: "cGFzc3dvcmQ="  # base64编码
```

**存储持久化策略**：
```yaml
# PVC持久化示例
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
```

#### **核心学习要点**

**1. 资源编排原理**：
- Deployment确保应用副本数量和更新策略
- Service提供稳定的网络抽象和服务发现
- Ingress实现外部流量的L7路由

**2. 网络通信机制**：
- Pod间通过Service名称进行DNS解析
- ClusterIP实现集群内部负载均衡
- NodePort/LoadBalancer暴露外部访问

**3. 数据持久化机制**：
- PVC提供存储抽象，与具体存储实现解耦
- StatefulSet确保有状态应用的数据一致性
- 存储类(StorageClass)动态供应存储资源

**4. 配置管理原理**：
- 配置与镜像分离，提高部署灵活性
- 环境变量、挂载文件两种配置注入方式
- Secret权限控制，确保敏感信息安全

#### **部署流程与运维要点**

**部署顺序**：
1. **基础资源** → 命名空间、ConfigMap、Secret
2. **数据层** → 数据库StatefulSet和PVC
3. **后端层** → 后端Deployment和Service  
4. **前端层** → 前端Deployment和Service
5. **网关层** → Ingress配置和外部访问

**运维监控**：
- **健康检查**：readinessProbe确保Pod就绪后接收流量
- **资源限制**：requests/limits防止资源争抢
- **日志收集**：统一日志输出，便于故障排查
- **监控指标**：Prometheus采集应用和基础设施指标

这种架构充分体现了Kubernetes的**声明式管理、自动化运维、弹性伸缩**等核心优势，为后续的微服务和高级部署策略奠定了基础。

#### **实际部署要点**

**资源配置原则**：
- **命名空间隔离**：development、staging、production环境分离
- **资源限制**：为每个容器设置合理的CPU/内存限制
- **健康检查**：配置存活探针和就绪探针确保服务可用性
- **数据持久化**：使用StorageClass动态供应存储资源

**网络访问策略**：
- **内部通信**：Service + DNS实现微服务间通信
- **外部访问**：Ingress + LoadBalancer提供统一入口
- **安全控制**：NetworkPolicy限制Pod间通信
- **TLS终止**：在Ingress层统一处理HTTPS证书

**配置管理策略**：
- **环境配置**：ConfigMap存储应用配置，支持热更新
- **敏感信息**：Secret存储密码、API密钥等
- **版本控制**：配置文件纳入Git管理，保证可追溯性
- **权限控制**：RBAC确保只有授权用户能访问敏感配置

---

### 5.2 微服务架构部署案例

本节通过电商平台的微服务架构，深入学习Kubernetes在**服务治理、配置管理、弹性伸缩、故障隔离**等方面的核心能力。

#### **微服务架构与Kubernetes原理**

**业务架构**：用户服务 + 订单服务 + 支付服务 + 网关服务

**Kubernetes微服务治理模式**：

| 治理维度 | Kubernetes实现 | 核心价值 | 技术原理 |
|----------|---------------|----------|----------|
| **服务发现** | Service + DNS | 动态服务定位 | 内置DNS解析，自动端点更新 |
| **负载均衡** | Service ClusterIP | 流量分发 | kube-proxy实现L4负载均衡 |
| **配置管理** | ConfigMap + Secret | 配置与代码分离 | 环境变量/文件挂载注入 |
| **弹性伸缩** | HPA + VPA | 自动扩缩容 | 基于指标的智能调度 |
| **故障隔离** | Namespace + NetworkPolicy | 限爆半径 | 资源隔离+网络策略 |
| **版本管理** | Deployment | 滚动更新 | 渐进式发布，零停机部署 |

#### **微服务网络通信架构**

```
                    ┌─────────────────────────────────────────┐
                    │           Kubernetes Cluster            │
                    │                                         │
┌─────────────┐    │  ┌─────────────┐     ┌─────────────┐   │
│   外部用户   │────┼─→│    Ingress   │────→│ API Gateway │   │
└─────────────┘    │  └─────────────┘     └─────────────┘   │
                    │         │                    │         │
                    │         ▼                    ▼         │
                    │  ┌─────────────┐     ┌─────────────┐   │
                    │  │ User Service│◄────┤Order Service│   │
                    │  └─────────────┘     └─────────────┘   │
                    │         │                    │         │
                    │         ▼                    ▼         │
                    │  ┌─────────────┐     ┌─────────────┐   │
                    │  │   Database  │     │Payment Svc  │   │
                    │  └─────────────┘     └─────────────┘   │
                    └─────────────────────────────────────────┘
```

#### **Kubernetes微服务部署模式**

**1. 服务注册与发现机制**
```yaml
# 自动服务发现示例
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 8080
---
# 其他服务通过DNS访问: http://user-service:8080
```

**核心原理**：
- **DNS解析**：Kubernetes内置DNS自动为Service创建域名记录
- **端点管理**：Endpoints控制器自动维护健康Pod列表
- **负载均衡**：kube-proxy在节点上实现流量分发规则

**2. 配置分层管理策略**
```yaml
# 全局配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-config
data:
  log.level: "INFO"
  tracing.enabled: "true"
---
# 服务专用配置
apiVersion: v1
kind: ConfigMap  
metadata:
  name: user-service-config
data:
  database.pool.size: "20"
  cache.ttl: "3600"
```

**配置管理原理**：
- **配置分层**：全局配置 + 服务配置 + 环境配置
- **热更新**：ConfigMap变更自动通知Pod更新
- **版本控制**：配置变更可追溯，支持回滚

**3. 弹性伸缩策略**
```yaml
# 基于CPU/内存的水平扩展
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**扩缩容原理**：
- **指标收集**：metrics-server收集Pod资源使用情况
- **决策算法**：HPA控制器根据目标利用率计算所需副本数
- **平滑扩展**：避免震荡，支持扩容延迟和缩容冷却期

#### **微服务治理最佳实践**

**服务划分原则**：
- **单一职责**：每个服务负责一个业务领域
- **数据独立**：服务拥有独立的数据存储
- **接口标准**：统一的API设计和文档规范
- **故障隔离**：服务故障不影响整体系统

**部署策略选择**：
- **无状态服务**：使用Deployment + Service
- **有状态服务**：使用StatefulSet + Headless Service
- **任务类服务**：使用Job + CronJob
- **守护进程**：使用DaemonSet

**监控与观测**：
- **健康检查**：liveness/readiness/startup三类探针
- **指标监控**：Prometheus + Grafana监控体系
- **链路追踪**：Jaeger/Zipkin分布式追踪
- **日志聚合**：ELK/Loki统一日志管理

**安全策略**：
- **网络隔离**：NetworkPolicy限制服务间通信
- **身份认证**：ServiceAccount + RBAC权限控制
- **密钥管理**：Secret存储敏感信息，支持加密
- **镜像安全**：定期扫描漏洞，使用最小化基础镜像

---

### 5.3 Kubernetes部署策略案例

本节深入学习Kubernetes的**渐进式部署**能力，掌握滚动更新、蓝绿部署、金丝雀发布等云原生部署模式的原理和实践。

#### **部署策略对比与选择**

| 部署策略 | 资源开销 | 切换速度 | 风险控制 | 适用场景 |
|----------|----------|----------|----------|----------|
| **滚动更新** | 低(1.2x) | 中等 | 中等 | 日常版本迭代 |
| **蓝绿部署** | 高(2x) | 极快 | 高 | 重大版本发布 |
| **金丝雀发布** | 低(1.1x) | 慢 | 极高 | 风险敏感业务 |
| **A/B测试** | 中等(1.5x) | 中等 | 高 | 功能验证 |

#### **滚动更新：Kubernetes默认策略**

**核心原理**：渐进式替换Pod实例，确保服务始终可用

**更新流程**：
```
现有Pod: [v1] [v1] [v1] [v1] [v1] [v1]
Step 1:  [v1] [v1] [v1] [v1] [v2] [v2]  ← 创建新版本Pod
Step 2:  [v1] [v1] [v2] [v2] [v2] [v2]  ← 等待就绪后删除旧Pod
Step 3:  [v2] [v2] [v2] [v2] [v2] [v2]  ← 完成滚动更新
```

**关键配置**：
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%  # 最多不可用Pod比例
    maxSurge: 25%        # 最多超出Pod比例
```

**优势与适用性**：
- ✅ 资源消耗最小，成本效益高
- ✅ 自动化程度高，操作简单
- ✅ 支持自动回滚和暂停机制
- ❌ 更新过程中新旧版本并存

#### **蓝绿部署：极速切换策略**

**核心原理**：维护两套完全相同的环境，通过Service切换实现瞬间发布

**部署架构**：
```
Production Traffic
        │
        ▼
┌─────────────┐       ┌─────────────┐
│   Service   │       │   Service   │
│  (Active)   │       │ (Standby)   │
└─────────────┘       └─────────────┘
        │                     │
        ▼                     ▼
┌─────────────┐       ┌─────────────┐
│   Blue Env  │       │  Green Env  │
│   (v1.0)    │       │   (v2.0)    │
└─────────────┘       └─────────────┘
```

**实现机制**：
```yaml
# 通过修改Service selector实现流量切换
kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'
```

**优势与适用性**：
- ✅ 切换速度极快（秒级）
- ✅ 完整环境验证，风险最低
- ✅ 回滚简单，一键切换
- ❌ 资源消耗高，成本翻倍

#### **金丝雀发布：渐进式风险控制**

**核心原理**：小比例流量验证新版本，根据监控指标决定是否扩大发布范围

**流量分配演进**：
```
Phase 1: Stable(90%) + Canary(10%)  ← 小流量验证
Phase 2: Stable(70%) + Canary(30%)  ← 逐步增加
Phase 3: Stable(50%) + Canary(50%)  ← 对等验证
Phase 4: Stable(0%)  + Canary(100%) ← 完全切换
```

**实现方式**：
```yaml
# 通过调整副本数控制流量比例
kubectl scale deployment app-stable --replicas=9   # 90%流量
kubectl scale deployment app-canary --replicas=1   # 10%流量
```

**监控决策指标**：
- **错误率**：新版本5xx错误率不超过1%
- **响应时间**：P99延迟不超过基线的110%
- **业务指标**：转化率、交易成功率等
- **资源消耗**：CPU/内存使用量在合理范围

#### **部署策略选择指南**

**日常迭代场景**：
- 功能完善、风险较低 → **滚动更新**
- 需要快速验证效果 → **金丝雀发布**

**重大版本发布**：
- 架构重构、数据库变更 → **蓝绿部署**
- 不确定用户接受度 → **A/B测试**

**紧急修复场景**：
- 安全漏洞修复 → **蓝绿部署**（快速切换）
- 性能问题修复 → **金丝雀发布**（效果验证）

---

### 5.4 Kubernetes监控集成案例

本节学习Kubernetes原生的**可观测性**体系，掌握指标监控、日志聚合、链路追踪的云原生实现方式。

#### **云原生监控体系架构**

```
┌─────────────────────────────────────────────────────────┐
│                    监控数据流向                            │
│                                                         │
│  Application  →  Metrics     →  Prometheus  →  Grafana │
│      ↓           Logs        →  Loki        →  Grafana │
│  Kubernetes   →  Traces      →  Jaeger      →  Grafana │
│      ↓           Events      →  EventRouter →  Grafana │
│   Infrastructure                                        │
└─────────────────────────────────────────────────────────┘
```

#### **Prometheus监控集成原理**

**服务发现机制**：
```yaml
# Pod注解驱动的自动发现
annotations:
  prometheus.io/scrape: "true"    # 启用监控
  prometheus.io/port: "9090"      # 指标端口
  prometheus.io/path: "/metrics"  # 指标路径
```

**ServiceMonitor CRD**：
```yaml
# Prometheus Operator声明式监控
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
```

**监控指标分层**：
- **基础设施层**：节点CPU、内存、磁盘、网络
- **Kubernetes层**：Pod状态、Service可用性、资源配额
- **应用层**：业务指标、请求QPS、响应时间、错误率
- **业务层**：订单量、收入、用户活跃度等

#### **云原生监控最佳实践**

**指标标准化**：
- 遵循Prometheus指标命名规范
- 统一标签策略：service、version、environment
- 合理设置指标采集频率和保留期

**告警分级管理**：
- **P0级**：服务完全不可用，需要立即响应
- **P1级**：核心功能受影响，1小时内响应
- **P2级**：部分功能异常，24小时内处理
- **P3级**：性能下降，定期优化处理

**可观测性工具链**：
- **监控**：Prometheus + Grafana
- **日志**：Fluentd + Elasticsearch + Kibana
- **追踪**：Jaeger + OpenTelemetry
- **事件**：Kubernetes Events + AlertManager

这种监控体系确保了Kubernetes集群和应用的全方位可观测性，为稳定运行提供有力保障。

---

## 第五部分总结：Kubernetes部署框架与最佳实践

### 🎯 部署架构决策框架

通过前面四个实战案例的学习，我们可以总结出一套完整的Kubernetes部署决策框架：

```
┌─────────────────────────────────────────────────────────┐
│                 Kubernetes部署决策树                      │
│                                                         │
│  业务需求 → 架构选型 → 资源规划 → 部署策略 → 监控运维    │
│     ↓         ↓         ↓         ↓         ↓         │
│  功能性    技术栈    计算存储    发布方式    可观测性     │
│  非功能性  服务拆分  网络安全    扩缩容      故障处理     │
└─────────────────────────────────────────────────────────┘
```

#### **架构模式选择矩阵**

| 应用特征 | 单体应用 | 微服务架构 | 混合模式 |
|----------|----------|-----------|----------|
| **团队规模** | 1-5人 | 10+人 | 5-10人 |
| **业务复杂度** | 简单-中等 | 复杂 | 中等-复杂 |
| **部署频率** | 周/月级 | 日级 | 周级 |
| **技术栈** | 统一 | 多样化 | 部分统一 |
| **Kubernetes资源** | Deployment | 多个Service | Service+Ingress |

#### **资源配置策略**

**计算资源分配原则**：
- **请求值设置**：基于压测数据的70%作为requests
- **限制值设置**：requests的1.5-2倍作为limits
- **QoS等级选择**：生产应用优先选择Guaranteed类型

**存储策略选择**：
- **临时数据**：emptyDir卷，Pod重启清除
- **配置文件**：ConfigMap/Secret，支持热更新
- **持久数据**：PVC+StorageClass，保证数据安全
- **共享文件**：ReadWriteMany模式，多Pod访问

#### **监控体系建设路径**

**基础监控层**：
1. **资源监控**：CPU、内存、存储、网络
2. **组件监控**：kubelet、kube-proxy、etcd
3. **应用监控**：Pod状态、Service健康度

**业务监控层**：
1. **接口监控**：响应时间、成功率、QPS
2. **业务指标**：用户行为、交易数据、错误率
3. **用户体验**：页面加载时间、操作流畅度

### 🚀 云原生最佳实践总结

#### **开发阶段最佳实践**
- **容器化原则**：一个容器一个进程，无状态设计
- **配置外置**：环境变量、ConfigMap管理配置
- **健康检查**：实现liveness、readiness、startup探针
- **日志规范**：结构化日志，统一输出到stdout

#### **部署阶段最佳实践**
- **渐进发布**：优先选择滚动更新，关键业务用蓝绿
- **资源管理**：设置合理的requests和limits
- **安全配置**：最小权限原则，定期更新镜像
- **监控就绪**：部署前确保监控告警配置完成

#### **运维阶段最佳实践**
- **自动扩缩容**：基于业务指标配置HPA
- **故障恢复**：完善的备份和恢复机制
- **性能调优**：定期分析瓶颈，优化资源配置
- **安全更新**：及时更新Kubernetes版本和组件

          mountPath: /var/lib/mysql # MySQL数据目录
        - name: mysql-config        # 配置文件挂载
          mountPath: /etc/mysql/conf.d
          readOnly: true            # 只读挂载
        
        # 资源限制
        resources:
          requests:                 # 最小资源需求
            memory: "512Mi"         # 最小内存512MB
            cpu: "500m"             # 最小CPU 0.5核
          limits:                   # 最大资源限制
            memory: "1Gi"           # 最大内存1GB
            cpu: "1000m"            # 最大CPU 1核
        
        # 健康检查
        livenessProbe:              # 存活探针：检查容器是否需要重启
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30   # 初始延迟30秒
          periodSeconds: 10         # 每10秒检查一次
          timeoutSeconds: 5         # 超时时间5秒
          failureThreshold: 3       # 连续失败3次才重启
        
        readinessProbe:             # 就绪探针：检查容器是否ready接收流量
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -prootpassword
            - -e
            - "SELECT 1"
          initialDelaySeconds: 10   # 初始延迟10秒
          periodSeconds: 5          # 每5秒检查一次
          timeoutSeconds: 3         # 超时时间3秒
          successThreshold: 1       # 成功1次即认为ready
      
      # 卷定义
      volumes:
      - name: mysql-storage         # 持久化存储卷
        persistentVolumeClaim:
          claimName: mysql-pvc      # 引用PVC
      - name: mysql-config          # 配置文件卷
        configMap:
          name: mysql-config        # 引用ConfigMap（需要单独创建）
          defaultMode: 0644         # 文件权限

---
# MySQL配置文件
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: webapp
data:
  custom.cnf: |                    # MySQL自定义配置
    [mysqld]
    # 性能配置
    innodb_buffer_pool_size = 256M  # InnoDB缓冲池大小
    max_connections = 100           # 最大连接数
    
    # 字符集配置
    character-set-server = utf8mb4  # 服务器字符集
    collation-server = utf8mb4_unicode_ci
    
    # 日志配置
    slow_query_log = 1              # 启用慢查询日志
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 2             # 慢查询阈值（秒）
    
    # 安全配置
    sql_mode = STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO

---
# MySQL服务
apiVersion: v1
kind: Service
metadata:
  name: mysql-service             # 服务名称（其他组件通过此名称访问）
  namespace: webapp               # 指定命名空间
  labels:
    app: mysql                    # 应用标识
    component: database           # 组件类型
spec:
  type: ClusterIP                 # 集群内部服务（不对外暴露）
  selector:
    app: mysql                    # 选择标签为app=mysql的Pod
  ports:
  - name: mysql                   # 端口名称
    port: 3306                    # 服务端口（其他Pod访问的端口）
    targetPort: mysql             # 目标端口（Pod内容器端口名称）
    protocol: TCP                 # 协议类型
  # 会话保持配置（可选）
  sessionAffinity: None           # 不启用会话保持（数据库连接池场景）
```

**数据库部署要点总结**：
1. **存储策略**：使用PVC确保数据持久化，选择适合的存储类
2. **资源配置**：根据业务需求设置合理的CPU/内存限制
3. **健康检查**：配置存活和就绪探针确保服务可用性
4. **配置管理**：MySQL配置文件独立管理，便于调优

---

**第三步：后端服务层部署**

**Spring Boot后端部署策略**：
- **多副本设计**：3个副本提供高可用性和负载分担
- **滚动更新**：零停机时间发布新版本
- **配置注入**：数据库配置通过环境变量注入
- **健康检查**：确保只有ready的Pod接收流量

```yaml
---
# Spring Boot后端应用部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend                   # Deployment名称
  namespace: webapp               # 指定命名空间
  labels:
    app: backend                  # 应用标识
    component: api                # 组件类型
    tier: backend                 # 应用层级
spec:
  replicas: 3                     # 副本数：多副本提供高可用
  strategy:
    type: RollingUpdate           # 滚动更新策略
    rollingUpdate:
      maxSurge: 1                 # 更新时最多新增1个Pod
      maxUnavailable: 0           # 更新时不允许Pod不可用（保证服务连续性）
  selector:
    matchLabels:
      app: backend                # 选择器匹配Pod标签
  template:
    metadata:
      labels:
        app: backend              # Pod标签（必须匹配selector）
        component: api            # 组件标识
        tier: backend             # 层级标识
        version: v1.0             # 版本标识（用于版本控制）
      annotations:
        prometheus.io/scrape: "true"    # Prometheus监控注解
        prometheus.io/port: "8080"      # 监控端口
        prometheus.io/path: "/metrics"  # 监控路径
    spec:
      # 容器配置
      containers:
      - name: backend             # 容器名称
        image: myapp/backend:v1.0 # 应用镜像（应替换为实际镜像地址）
        imagePullPolicy: Always   # 总是拉取最新镜像（开发环境）
        
        # 端口配置
        ports:
        - name: http              # 端口名称（用于Service引用）
          containerPort: 8080     # Spring Boot默认端口
          protocol: TCP           # 协议类型
        - name: management        # 管理端口（Spring Boot Actuator）
          containerPort: 8081     # 管理端口（用于健康检查和监控）
          protocol: TCP
        
        # 环境变量配置
        env:
        # 数据库配置（从ConfigMap获取）
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config    # ConfigMap名称
              key: database.host  # 配置键名
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.port
        - name: DATABASE_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.name
        
        # 数据库凭证（从Secret获取）
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets        # Secret名称
              key: database.username   # Secret键名
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        
        # JWT配置（从Secret获取）
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt.secret
        
        # 应用配置（从ConfigMap获取）
        - name: APP_ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.environment
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.log.level
        
        # 健康检查配置
        livenessProbe:            # 存活探针：检查应用是否运行正常
          httpGet:
            path: /actuator/health/liveness   # Spring Boot Actuator健康检查端点
            port: management        # 使用管理端口
            scheme: HTTP           # 协议类型
          initialDelaySeconds: 90  # 初始延迟90秒（Spring Boot启动较慢）
          periodSeconds: 10        # 每10秒检查一次
          timeoutSeconds: 5        # 超时时间5秒
          failureThreshold: 3      # 连续失败3次才重启
          successThreshold: 1      # 成功1次即恢复
        
        readinessProbe:           # 就绪探针：检查应用是否ready接收请求
          httpGet:
            path: /actuator/health/readiness  # Spring Boot就绪检查端点
            port: management        # 使用管理端口
            scheme: HTTP
          initialDelaySeconds: 30  # 初始延迟30秒
          periodSeconds: 5         # 每5秒检查一次
          timeoutSeconds: 3        # 超时时间3秒
          failureThreshold: 3      # 连续失败3次认为not ready
          successThreshold: 1      # 成功1次即ready
        
        # 启动探针（可选）：检查应用是否启动完成
        startupProbe:
          httpGet:
            path: /actuator/health/readiness
            port: management
          initialDelaySeconds: 10  # 初始延迟10秒
          periodSeconds: 10        # 每10秒检查一次
          timeoutSeconds: 3        # 超时时间3秒
          failureThreshold: 18     # 最多等待3分钟启动（10s * 18 = 180s）
          successThreshold: 1      # 成功1次即启动完成
        
        # 资源限制
        resources:
          requests:               # 最小资源需求
            memory: "256Mi"       # 最小内存256MB
            cpu: "200m"           # 最小CPU 0.2核
          limits:                 # 最大资源限制
            memory: "512Mi"       # 最大内存512MB
            cpu: "500m"           # 最大CPU 0.5核
        
        # 安全上下文
        securityContext:
          allowPrivilegeEscalation: false  # 不允许特权升级
          readOnlyRootFilesystem: false    # 允许写根文件系统（Spring Boot需要）
          runAsNonRoot: true               # 以非root用户运行
          runAsUser: 1000                  # 指定用户ID
          capabilities:
            drop:
            - ALL                          # 移除所有能力
            add:
            - NET_BIND_SERVICE             # 添加绑定端口能力
        
        # 卷挂载
        volumeMounts:
        - name: tmp-volume        # 临时目录
          mountPath: /tmp
        - name: logs-volume       # 日志目录
          mountPath: /app/logs
      
      # Pod级别配置
      restartPolicy: Always       # 重启策略：总是重启
      terminationGracePeriodSeconds: 30  # 优雅停机时间30秒
      
      # 卷定义
      volumes:
      - name: tmp-volume          # 临时卷（用于临时文件）
        emptyDir: {}
      - name: logs-volume         # 日志卷（用于日志文件）
        emptyDir: {}

---
# 后端服务
apiVersion: v1
kind: Service
metadata:
  name: backend-service         # 服务名称（前端通过此名称访问）
  namespace: webapp             # 指定命名空间
  labels:
    app: backend                # 应用标识
    component: api              # 组件类型
spec:
  type: ClusterIP               # 集群内部服务
  selector:
    app: backend                # 选择标签为app=backend的Pod
  ports:
  - name: http                  # 端口名称
    port: 80                    # 服务端口（其他服务访问的端口）
    targetPort: http            # 目标端口（Pod容器端口名称）
    protocol: TCP               # 协议类型
  - name: management            # 管理端口（用于监控和健康检查）
    port: 8081                  # 管理服务端口
    targetPort: management      # 目标管理端口
    protocol: TCP
  sessionAffinity: None         # 不启用会话保持（RESTful API无状态）

---
# 后端服务监控配置（可选）
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: backend-monitor
  namespace: webapp
  labels:
    app: backend
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
  - port: management            # 监控管理端口
    path: /actuator/prometheus  # Prometheus指标路径
    interval: 30s               # 采集间隔
```

**第四步：前端表现层部署**

**React前端部署策略**：
- **静态资源服务**：使用Nginx作为Web服务器提供静态文件
- **API代理**：Nginx反向代理后端API，解决跨域问题
- **SPA路由支持**：配置try_files支持React Router
- **缓存优化**：合理配置静态资源缓存策略

```yaml
---
# Nginx配置文件
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config           # ConfigMap名称
  namespace: webapp               # 指定命名空间
  labels:
    app: frontend                 # 应用标识
    component: web                # 组件类型
data:
  # 主配置文件
  nginx.conf: |
    # Nginx主配置
    user nginx;
    worker_processes auto;        # 自动设置工作进程数
    error_log /var/log/nginx/error.log warn;
    pid /var/run/nginx.pid;
    
    events {
        worker_connections 1024;  # 每个工作进程的连接数
        use epoll;               # 使用epoll事件模型
    }
    
    http {
        # MIME类型配置
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        # 日志格式
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';
        access_log /var/log/nginx/access.log main;
        
        # 基本设置
        sendfile on;              # 启用高效文件传输
        tcp_nopush on;           # 优化网络包
        tcp_nodelay on;          # 禁用Nagle算法
        keepalive_timeout 65;    # Keep-Alive超时时间
        types_hash_max_size 2048;
        
        # Gzip压缩
        gzip on;
        gzip_vary on;
        gzip_min_length 1024;
        gzip_types
            text/plain
            text/css
            text/xml
            text/javascript
            application/javascript
            application/xml+rss
            application/json;
        
        # 包含虚拟主机配置
        include /etc/nginx/conf.d/*.conf;
    }
  
  # 虚拟主机配置
  default.conf: |
    # 后端服务上游配置
    upstream backend_servers {
        server backend-service:80;    # 后端服务地址
        keepalive 32;                # 保持连接池
    }
    
    server {
        listen 80;                   # 监听80端口
        server_name _;               # 默认服务器名
        root /usr/share/nginx/html;  # 静态文件根目录
        index index.html;            # 默认首页
        
        # 安全头设置
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        
        # 静态资源缓存配置
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;              # 静态资源缓存1年
            add_header Cache-Control "public, immutable";
            access_log off;          # 静态资源不记录访问日志
        }
        
        # HTML文件不缓存
        location ~* \.html$ {
            expires -1;              # 不缓存HTML
            add_header Cache-Control "no-cache, no-store, must-revalidate";
            add_header Pragma "no-cache";
        }
        
        # API代理配置
        location /api/ {
            proxy_pass http://backend_servers/;    # 代理到后端服务
            proxy_http_version 1.1;               # 使用HTTP/1.1
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;          # 传递原始Host头
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;     # WebSocket支持
            proxy_connect_timeout 30s;           # 连接超时
            proxy_send_timeout 30s;              # 发送超时
            proxy_read_timeout 30s;              # 读取超时
        }
        
        # SPA路由支持（React Router）
        location / {
            try_files $uri $uri/ /index.html;    # 所有路由都返回index.html
        }
        
        # 健康检查端点
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # 禁止访问隐藏文件
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
    }

---
# React前端应用部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend                  # Deployment名称
  namespace: webapp               # 指定命名空间
  labels:
    app: frontend                 # 应用标识
    component: web                # 组件类型
    tier: frontend                # 应用层级
spec:
  replicas: 2                     # 副本数：2个副本提供高可用
  strategy:
    type: RollingUpdate           # 滚动更新策略
    rollingUpdate:
      maxSurge: 1                 # 更新时最多新增1个Pod
      maxUnavailable: 0           # 更新时不允许Pod不可用
  selector:
    matchLabels:
      app: frontend               # 选择器匹配Pod标签
  template:
    metadata:
      labels:
        app: frontend             # Pod标签（必须匹配selector）
        component: web            # 组件标识
        tier: frontend            # 层级标识
        version: v1.0             # 版本标识
    spec:
      # 容器配置
      containers:
      - name: frontend            # 容器名称
        image: myapp/frontend:v1.0  # 前端镜像（包含编译后的React应用）
        imagePullPolicy: Always   # 总是拉取最新镜像
        
        # 端口配置
        ports:
        - name: http              # 端口名称
          containerPort: 80       # Nginx默认端口
          protocol: TCP           # 协议类型
        
        # 健康检查配置
        livenessProbe:            # 存活探针
          httpGet:
            path: /health         # 健康检查路径
            port: http            # 使用http端口
          initialDelaySeconds: 10 # 初始延迟10秒
          periodSeconds: 10       # 每10秒检查一次
          timeoutSeconds: 5       # 超时时间5秒
          failureThreshold: 3     # 连续失败3次才重启
        
        readinessProbe:           # 就绪探针
          httpGet:
            path: /health         # 就绪检查路径
            port: http            # 使用http端口
          initialDelaySeconds: 5  # 初始延迟5秒
          periodSeconds: 5        # 每5秒检查一次
          timeoutSeconds: 3       # 超时时间3秒
          failureThreshold: 3     # 连续失败3次认为not ready
        
        # 资源限制
        resources:
          requests:               # 最小资源需求
            memory: "64Mi"        # 最小内存64MB（Nginx占用较少）
            cpu: "100m"           # 最小CPU 0.1核
          limits:                 # 最大资源限制
            memory: "128Mi"       # 最大内存128MB
            cpu: "200m"           # 最大CPU 0.2核
        
        # 安全上下文
        securityContext:
          allowPrivilegeEscalation: false  # 不允许特权升级
          readOnlyRootFilesystem: true     # 只读根文件系统
          runAsNonRoot: true               # 以非root用户运行
          runAsUser: 101                   # Nginx用户ID
          capabilities:
            drop:
            - ALL                          # 移除所有能力
            add:
            - NET_BIND_SERVICE             # 添加绑定端口能力
        
        # 卷挂载
        volumeMounts:
        - name: nginx-config      # Nginx配置文件
          mountPath: /etc/nginx
          readOnly: true          # 只读挂载
        - name: nginx-cache       # Nginx缓存目录
          mountPath: /var/cache/nginx
        - name: nginx-run         # Nginx运行目录
          mountPath: /var/run
        - name: nginx-log         # Nginx日志目录
          mountPath: /var/log/nginx
      
      # 卷定义
      volumes:
      - name: nginx-config        # Nginx配置卷
        configMap:
          name: frontend-config   # 引用ConfigMap
          defaultMode: 0644       # 文件权限
      - name: nginx-cache         # Nginx缓存卷
        emptyDir: {}
      - name: nginx-run           # Nginx运行卷
        emptyDir: {}
      - name: nginx-log           # Nginx日志卷
        emptyDir: {}

---
# 前端服务
apiVersion: v1
kind: Service
metadata:
  name: frontend-service        # 服务名称
  namespace: webapp             # 指定命名空间
  labels:
    app: frontend               # 应用标识
    component: web              # 组件类型
spec:
  type: ClusterIP               # 集群内部服务
  selector:
    app: frontend               # 选择标签为app=frontend的Pod
  ports:
  - name: http                  # 端口名称
    port: 80                    # 服务端口
    targetPort: http            # 目标端口（Pod容器端口名称）
    protocol: TCP               # 协议类型
  sessionAffinity: None         # 不启用会话保持（静态资源无状态）
```

**前端部署要点总结**：
1. **Nginx优化**：合理配置Gzip压缩、缓存策略提升性能
2. **SPA支持**：配置try_files支持前端路由
3. **API代理**：解决跨域问题，统一域名访问
4. **安全配置**：添加安全头防护常见Web攻击

---

**第五步：对外访问配置**

**Ingress配置策略**：
- **域名路由**：基于域名将流量路由到不同服务
- **SSL终止**：在Ingress层终止SSL，后端使用HTTP
- **路径重写**：支持不同的URL路径规则
- **负载均衡**：自动在多个Pod间分发流量

```yaml
---
# Web应用对外访问配置
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress            # Ingress名称
  namespace: webapp               # 指定命名空间
  labels:
    app: webapp                   # 应用标识
  annotations:
    # Nginx Ingress Controller配置
    nginx.ingress.kubernetes.io/rewrite-target: /        # 路径重写
    nginx.ingress.kubernetes.io/ssl-redirect: "true"     # 强制HTTPS
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"   # 上传文件大小限制
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"   # 连接超时
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"      # 发送超时
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"      # 读取超时
    
    # SSL证书管理（使用cert-manager）
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # 证书颁发者
    cert-manager.io/acme-challenge-type: http01          # ACME验证方式
    
    # 安全配置
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
spec:
  # TLS配置
  tls:
  - hosts:
    - webapp.example.com          # 域名
    secretName: webapp-tls        # TLS证书Secret名称
  
  # 路由规则
  rules:
  - host: webapp.example.com      # 域名匹配
    http:
      paths:
      # 前端静态资源路由
      - path: /                   # 根路径
        pathType: Prefix          # 前缀匹配
        backend:
          service:
            name: frontend-service    # 前端服务
            port:
              number: 80             # 服务端口
      
      # API路由（可选，如果前端没有代理API）
      - path: /api                # API路径
        pathType: Prefix          # 前缀匹配
        backend:
          service:
            name: backend-service     # 后端服务
            port:
              number: 80             # 服务端口
```

**Ingress部署要点**：
1. **证书管理**：使用cert-manager自动管理SSL证书
2. **安全配置**：添加安全头防护
3. **超时设置**：合理配置各种超时时间
4. **路径路由**：前端和API分别路由到不同服务

---

### 5.1.1 前端项目专项部署方案

本节专门介绍不同类型前端项目的Kubernetes部署策略和最佳实践。

#### **React单页应用（SPA）部署方案**

**部署特点**：
- 编译后的静态文件
- 客户端路由需要特殊配置
- API跨域处理
- 缓存策略优化

```yaml
# React应用专用Dockerfile参考
# FROM node:16-alpine as builder
# WORKDIR /app
# COPY package*.json ./
# RUN npm ci --only=production
# COPY . .
# RUN npm run build
# 
# FROM nginx:alpine
# COPY --from=builder /app/build /usr/share/nginx/html
# COPY nginx.conf /etc/nginx/nginx.conf
# EXPOSE 80
# CMD ["nginx", "-g", "daemon off;"]

---
# React应用配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: react-app-config
  namespace: webapp
data:
  # 环境配置文件
  env.js: |
    window._env_ = {
      REACT_APP_API_URL: "/api",              # API基础URL
      REACT_APP_WS_URL: "wss://webapp.example.com/ws",  # WebSocket URL
      REACT_APP_VERSION: "1.0.0",            # 应用版本
      REACT_APP_ENVIRONMENT: "production"    # 运行环境
    };

---
# Vue.js应用部署配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: vue-app-config
  namespace: webapp
data:
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        
        # Vue Router history模式支持
        location / {
            try_files $uri $uri/ @rewrites;
        }
        
        location @rewrites {
            rewrite ^(.+)$ /index.html last;
        }
        
        # API代理
        location /api/ {
            proxy_pass http://backend-service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        # 静态资源缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
```

#### **Angular应用部署方案**

```yaml
---
# Angular应用配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: angular-app-config
  namespace: webapp
data:
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        
        # Angular路由支持
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        # 支持子路径部署
        location /myapp/ {
            alias /usr/share/nginx/html/;
            try_files $uri $uri/ /myapp/index.html;
        }
        
        # API代理配置
        location /api/ {
            proxy_pass http://backend-service/;
            proxy_buffering off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # PWA支持
        location /sw.js {
            add_header Cache-Control "no-cache";
            proxy_cache_bypass $http_pragma;
            proxy_cache_revalidate on;
            expires off;
            access_log off;
        }
        
        location /manifest.json {
            add_header Cache-Control "no-cache";
        }
        
        # 安全头
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    }

---
# Angular应用部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: angular-app
  template:
    metadata:
      labels:
        app: angular-app
        framework: angular
    spec:
      containers:
      - name: angular-app
        image: myapp/angular-app:latest
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      volumes:
      - name: nginx-config
        configMap:
          name: angular-app-config
```

#### **Next.js服务端渲染（SSR）部署方案**

```yaml
---
# Next.js应用部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
  namespace: webapp
spec:
  replicas: 3                     # SSR应用需要更多副本
  selector:
    matchLabels:
      app: nextjs-app
  template:
    metadata:
      labels:
        app: nextjs-app
        framework: nextjs
    spec:
      containers:
      - name: nextjs-app
        image: myapp/nextjs-app:latest    # 包含Node.js运行时的镜像
        ports:
        - containerPort: 3000             # Next.js默认端口
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: API_URL
          value: "http://backend-service"  # 内部API调用
        
        # 健康检查
        livenessProbe:
          httpGet:
            path: /api/health            # Next.js健康检查端点
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /api/health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
        
        # 资源配置（SSR需要更多资源）
        resources:
          requests:
            memory: "256Mi"              # SSR需要更多内存
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # 卷挂载
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /app/.next/cache    # Next.js缓存目录
      
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: cache-volume
        emptyDir: {}

---
# Next.js服务配置
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
  namespace: webapp
spec:
  selector:
    app: nextjs-app
  ports:
  - port: 80
    targetPort: 3000                    # 映射到Next.js端口
    name: http
  type: ClusterIP
```

#### **静态网站部署方案**

```yaml
---
# 纯静态网站部署（如文档站点、博客等）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-website
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-website
  template:
    metadata:
      labels:
        app: static-website
        type: static
    spec:
      containers:
      - name: static-website
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: website-content          # 静态文件
          mountPath: /usr/share/nginx/html
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        resources:
          requests:
            memory: "32Mi"               # 静态站点资源需求更少
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      volumes:
      - name: website-content
        configMap:
          name: static-website-content   # 或使用PVC挂载
      - name: nginx-config
        configMap:
          name: static-website-nginx-config
```

**前端部署最佳实践总结**：

1. **框架选择**：
   - **SPA应用**：React/Vue/Angular + Nginx
   - **SSR应用**：Next.js/Nuxt.js + Node.js
   - **静态站点**：纯Nginx或CDN

2. **性能优化**：
   - 启用Gzip压缩
   - 配置合理的缓存策略
   - 使用CDN加速静态资源

3. **安全配置**：
   - 添加安全响应头
   - 配置CSP策略
   - 禁止访问敏感文件

4. **路由支持**：
   - SPA路由配置try_files
   - 支持子路径部署
   - API代理配置

### 5.2 微服务架构部署案例

**实战重点**：学习如何在Kubernetes中实现微服务架构的核心概念：**服务发现、配置管理、独立扩展、故障隔离**。

**Kubernetes微服务核心概念**：
- **Service发现**：通过Service实现服务间通信
- **配置分离**：使用ConfigMap/Secret管理不同服务配置
- **独立部署**：每个微服务独立的Deployment和扩展策略
- **网络隔离**：通过NetworkPolicy实现服务间访问控制

**K8s微服务架构图**：
```
┌─────────────────────────────────────────────────┐
│                   Kubernetes Cluster            │
│                                                 │
│  [Ingress] → [API Gateway Service]              │
│                      ↓                          │
│  ┌─────────────────────────────────────────┐    │
│  │            微服务层                      │    │
│  │  [User Service] ←→ [Order Service]      │    │
│  │       ↕              ↕                  │    │
│  │  [Product Service] ←→ [Payment Service] │    │
│  └─────────────────────────────────────────┘    │
│                      ↓                          │
│  ┌─────────────────────────────────────────┐    │
│  │          共享基础服务                    │    │
│  │    [Redis] [MySQL] [RabbitMQ]          │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

**本案例Kubernetes学习要点**：
1. **Service网络**：ClusterIP服务发现机制
2. **配置管理**：多环境ConfigMap/Secret使用
3. **扩展策略**：不同服务的独立HPA配置
4. **健康检查**：微服务专用的探针设计

---

#### **第一步：Kubernetes命名空间隔离和配置管理**

**K8s学习重点**：命名空间资源隔离、ConfigMap配置分离、Secret敏感信息管理

```yaml
---
# 命名空间隔离（Kubernetes多租户基础）
apiVersion: v1
kind: Namespace
metadata:
  name: microservices           # 微服务专用命名空间
  labels:
    environment: production     # 环境标签（用于资源选择器）
    type: microservices        # 类型标签（便于管理）
  annotations:
    description: "微服务架构演示命名空间"

---
# ConfigMap配置外化（Kubernetes配置管理核心概念）
apiVersion: v1
kind: ConfigMap
metadata:
  name: shared-config          # 共享配置，演示配置复用
  namespace: microservices
  labels:
    config-type: shared        # 配置类型标签
data:
  # 基础服务地址（演示Service发现）
  database_host: "mysql-service.microservices.svc.cluster.local"  # 完整DNS名称
  cache_host: "redis-service.microservices.svc.cluster.local"
  message_queue_host: "rabbitmq-service.microservices.svc.cluster.local"
  
  # 应用配置（演示环境变量注入）
  log_level: "INFO"
  app_env: "production"
  service_port: "8080"
  metrics_port: "9090"

---
# Secret敏感信息管理（Kubernetes安全最佳实践）
apiVersion: v1
kind: Secret
metadata:
  name: shared-secrets         # 共享敏感配置
  namespace: microservices
  labels:
    secret-type: shared
type: Opaque
stringData:
  # 数据库凭证（演示Secret挂载）
  db_username: "microservice_user"
  db_password: "secure_password_123"
  
  # API密钥（演示敏感信息保护）
  jwt_secret: "jwt_signing_key_256_bits"
  api_key: "external_service_api_key"
```

---

#### **第二步：Service网络通信和负载均衡**

**K8s学习重点**：Service类型、服务发现、负载均衡算法、端点管理

```yaml
---
# API Gateway服务（演示ClusterIP和服务发现）
apiVersion: v1
kind: Service
metadata:
  name: api-gateway             # 网关服务名（其他服务通过此名称访问）
  namespace: microservices
  labels:
    app: api-gateway
    role: gateway              # 角色标签，便于筛选
spec:
  type: ClusterIP              # 集群内部服务（默认类型）
  selector:
    app: api-gateway           # 选择器：匹配Pod标签
  ports:
  - name: http                 # 端口名称（在Ingress中引用）
    port: 80                   # 服务端口（其他服务访问的端口）
    targetPort: 8080           # 目标端口（Pod容器端口）
    protocol: TCP
  sessionAffinity: None        # 会话亲和性：None=随机负载均衡

---
# 用户服务（演示微服务Service模式）
apiVersion: v1
kind: Service
metadata:
  name: user-service           # 用户服务名称
  namespace: microservices
  labels:
    app: user-service
    tier: backend              # 层级标签
spec:
  type: ClusterIP              # 内部服务类型
  selector:
    app: user-service          # 选择Pod标签
  ports:
  - name: http
    port: 8080                 # 统一端口（服务发现标准）
    targetPort: http           # 引用Pod端口名称
  - name: metrics              # 监控端口（演示多端口服务）
    port: 9090
    targetPort: metrics

---
# 订单服务（演示负载均衡和故障转移）
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: microservices
  labels:
    app: order-service
    tier: backend
spec:
  type: ClusterIP
  selector:
    app: order-service
    status: healthy           # 额外选择器：只路由到健康实例
  ports:
  - name: http
    port: 8080
    targetPort: http
  sessionAffinity: ClientIP   # 客户端IP亲和性（会话保持）
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # 会话超时时间
        keepalive 32;
    }
    
    # 订单服务集群
    upstream order-service {
        least_conn;
        server order-service:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    # 支付服务集群
    upstream payment-service {
        least_conn;
        server payment-service:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    # 库存服务集群
    upstream inventory-service {
        least_conn;
        server inventory-service:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    # 通知服务集群
    upstream notification-service {
        least_conn;
        server notification-service:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
  
  # 虚拟主机配置
  default.conf: |
    server {
        listen 80;
        server_name _;
        
        # 安全头设置
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Request-ID $request_id always;  # 请求追踪ID
        
        # 健康检查端点
        location /health {
            access_log off;
            return 200 "API Gateway is healthy\n";
            add_header Content-Type text/plain;
        }
        
        # 用户服务路由
        location /api/v1/users {
            limit_req zone=api burst=20 nodelay;    # 应用限流
            proxy_pass http://user-service/;
            include /etc/nginx/proxy_params;        # 引用通用代理参数
        }
        
        # 登录接口特殊限流
        location /api/v1/auth/login {
            limit_req zone=login burst=10 nodelay;  # 更严格的登录限流
            proxy_pass http://user-service/auth/login;
            include /etc/nginx/proxy_params;
        }
        
        # 商品服务路由
        location /api/v1/products {
            limit_req zone=api burst=50 nodelay;    # 商品查询允许更多并发
            proxy_pass http://product-service/;
            include /etc/nginx/proxy_params;
            
            # 商品图片缓存
            location ~* /api/v1/products/.*/images/ {
                proxy_pass http://product-service;
                proxy_cache_valid 200 1h;           # 图片缓存1小时
                add_header X-Cache-Status $upstream_cache_status;
            }
        }
        
        # 订单服务路由
        location /api/v1/orders {
            limit_req zone=api burst=15 nodelay;
            proxy_pass http://order-service/;
            include /etc/nginx/proxy_params;
        }
        
        # 支付服务路由（高安全要求）
        location /api/v1/payments {
            limit_req zone=api burst=5 nodelay;     # 支付接口严格限流
            proxy_pass http://payment-service/;
            include /etc/nginx/proxy_params;
            
            # 支付接口额外安全配置
            proxy_ssl_verify on;                    # 启用SSL验证
            proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
        }
        
        # 库存服务路由
        location /api/v1/inventory {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://inventory-service/;
            include /etc/nginx/proxy_params;
        }
        
        # 通知服务路由
        location /api/v1/notifications {
            limit_req zone=api burst=10 nodelay;
            proxy_pass http://notification-service/;
            include /etc/nginx/proxy_params;
        }
        
        # WebSocket支持（实时通知）
        location /ws/ {
            proxy_pass http://notification-service;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 86400;               # WebSocket长连接支持
        }
        
        # 默认路由（404处理）
        location / {
            return 404 '{"error": "API endpoint not found", "request_id": "$request_id"}';
            add_header Content-Type application/json;
        }
    }
  
  # 通用代理参数
  proxy_params: |
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Request-ID $request_id;      # 传递请求ID用于链路追踪
    proxy_connect_timeout 30s;                      # 连接超时
    proxy_send_timeout 60s;                         # 发送超时
    proxy_read_timeout 60s;                         # 读取超时
    proxy_buffering on;                             # 启用缓冲
    proxy_buffer_size 4k;                           # 缓冲区大小
    proxy_buffers 8 4k;                             # 缓冲区数量
    proxy_busy_buffers_size 8k;                     # 繁忙缓冲区大小

---
# API Gateway部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway             # Deployment名称
  namespace: ecommerce          # 指定命名空间
  labels:
    app: api-gateway            # 应用标识
    component: gateway          # 组件类型
    tier: gateway               # 架构层级
spec:
  replicas: 3                   # 3个副本保证高可用
  strategy:
    type: RollingUpdate         # 滚动更新策略
    rollingUpdate:
      maxSurge: 1               # 更新时最多新增1个Pod
      maxUnavailable: 0         # 更新时不允许Pod不可用
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway        # Pod标签
        component: gateway      # 组件标识
        tier: gateway           # 层级标识
        version: v1.0           # 版本标识
      annotations:
        prometheus.io/scrape: "true"        # Prometheus监控
        prometheus.io/port: "9113"          # Nginx exporter端口
        prometheus.io/path: "/metrics"      # 监控指标路径
    spec:
      # 容器配置
      containers:
      # Nginx主容器
      - name: nginx             # 容器名称
        image: nginx:1.21-alpine # Nginx镜像（使用稳定版本）
        imagePullPolicy: IfNotPresent # 镜像拉取策略
        
        # 端口配置
        ports:
        - name: http            # HTTP端口
          containerPort: 80
          protocol: TCP
        
        # 健康检查
        livenessProbe:          # 存活探针
          httpGet:
            path: /health       # 健康检查路径
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:         # 就绪探针
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
        
        # 资源限制
        resources:
          requests:
            memory: "128Mi"     # API Gateway需要更多内存处理路由
            cpu: "200m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        
        # 卷挂载
        volumeMounts:
        - name: nginx-config    # Nginx配置
          mountPath: /etc/nginx
          readOnly: true
        - name: proxy-params    # 代理参数配置
          mountPath: /etc/nginx/proxy_params
          subPath: proxy_params
          readOnly: true
        - name: nginx-cache     # Nginx缓存目录
          mountPath: /var/cache/nginx
        - name: nginx-run       # Nginx运行目录
          mountPath: /var/run
        - name: nginx-log       # 日志目录
          mountPath: /var/log/nginx
      
      # Nginx Prometheus Exporter容器（监控）
      - name: nginx-exporter
        image: nginx/nginx-prometheus-exporter:0.10.0
        args:
        - -nginx.scrape-uri=http://localhost/nginx_status
        ports:
        - name: metrics
          containerPort: 9113
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      
      # 卷定义
      volumes:
      - name: nginx-config      # Nginx配置卷
        configMap:
          name: api-gateway-config
          defaultMode: 0644
      - name: proxy-params      # 代理参数卷
        configMap:
          name: api-gateway-config
          items:
          - key: proxy_params
            path: proxy_params
      - name: nginx-cache       # 缓存卷
        emptyDir: {}
      - name: nginx-run         # 运行卷
        emptyDir: {}
      - name: nginx-log         # 日志卷
        emptyDir: {}

---
# API Gateway服务
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service   # 服务名称
  namespace: ecommerce        # 指定命名空间
  labels:
    app: api-gateway
    component: gateway
spec:
  type: ClusterIP             # 集群内部服务
  selector:
    app: api-gateway          # 选择API Gateway Pod
  ports:
  - name: http                # HTTP端口
    port: 80
    targetPort: http
    protocol: TCP
  - name: metrics             # 监控端口
    port: 9113
    targetPort: metrics
    protocol: TCP
```

---

#### **第三步：Deployment和Pod管理实战**

**K8s学习重点**：滚动更新、健康检查、资源限制、环境变量注入

```yaml
---
# 用户服务Deployment（演示微服务标准部署模式）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service           # 标准应用标签
    tier: backend              # 层级标签
spec:
  replicas: 3                  # 副本数（演示水平扩展）
  strategy:
    type: RollingUpdate        # 滚动更新策略（零停机部署）
    rollingUpdate:
      maxSurge: 1              # 最多增加1个Pod
      maxUnavailable: 1        # 最多不可用1个Pod
  selector:
    matchLabels:
      app: user-service        # 选择器必须匹配Pod标签
  template:
    metadata:
      labels:
        app: user-service      # Pod标签（Service通过此选择）
        status: healthy        # 额外标签（用于Service选择器）
      annotations:
        prometheus.io/scrape: "true"      # 监控注解
        prometheus.io/port: "9090"        # 监控端口
    spec:
      containers:
      - name: user-service
        image: myapp/user-service:v1.0    # 镜像版本管理
        imagePullPolicy: IfNotPresent     # 镜像拉取策略
        
        # 端口配置（多端口演示）
        ports:
        - name: http           # 业务端口（命名便于Service引用）
          containerPort: 8080
        - name: metrics        # 监控端口
          containerPort: 9090
        
        # 环境变量注入（演示ConfigMap和Secret使用）
        env:
        - name: APP_ENV
          value: "production"
        - name: DATABASE_HOST  # 从ConfigMap获取配置
          valueFrom:
            configMapKeyRef:
              name: shared-config
              key: database_host
        - name: DB_PASSWORD    # 从Secret获取敏感信息
          valueFrom:
            secretKeyRef:
              name: shared-secrets
              key: db_password
        
        # 资源限制（演示资源管理）
        resources:
          requests:            # 资源请求（调度保证）
            memory: "256Mi"
            cpu: "200m"
          limits:              # 资源限制（防止资源耗尽）
            memory: "512Mi"
            cpu: "500m"
        
        # 健康检查配置（Kubernetes健康管理核心）
        livenessProbe:         # 存活探针（容器重启条件）
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30    # 初始延迟
          periodSeconds: 10          # 检查间隔
          failureThreshold: 3        # 失败阈值
        
        readinessProbe:        # 就绪探针（流量路由条件）
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        
        startupProbe:          # 启动探针（慢启动应用保护）
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 6

---
# 订单服务（演示不同配置策略）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
  labels:
    app: order-service
    tier: backend
spec:
  replicas: 2                  # 较少副本（按业务需求）
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%            # 百分比配置（灵活扩容）
      maxUnavailable: 0        # 零停机部署
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        status: healthy
    spec:
      # 节点调度约束（演示Pod调度控制）
      nodeSelector:
        workload-type: compute   # 只调度到计算节点
      
      containers:
      - name: order-service
        image: myapp/order-service:v1.0
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        
        # 环境变量模板化配置
        envFrom:
        - configMapRef:
            name: shared-config  # 批量导入ConfigMap
        - secretRef:
            name: shared-secrets # 批量导入Secret
        
        env:
        - name: SERVICE_NAME
          value: "order-service"
        - name: POD_IP          # 获取Pod动态信息
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # 更严格的资源控制
        resources:
          requests:
            memory: "512Mi"      # 订单服务需要更多内存
            cpu: "300m"
          limits:
            memory: "1Gi"
            cpu: "800m"
        
        # 优化的健康检查
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 45
          periodSeconds: 30      # 较长间隔（减少开销）
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
```

#### **第四步：HPA自动扩缩容实战**

**K8s学习重点**：水平扩展、指标监控、扩容策略、资源使用率

```yaml
---
# 用户服务HPA（基于CPU和内存）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: microservices
  labels:
    app: user-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service          # 目标Deployment
  minReplicas: 2               # 最小副本数（保证基础可用性）
  maxReplicas: 10              # 最大副本数（防止资源耗尽）
  
  # 扩缩容指标配置
  metrics:
  - type: Resource
    resource:
      name: cpu                # CPU使用率指标
      target:
        type: Utilization
        averageUtilization: 70 # 平均CPU超过70%扩容
        
  - type: Resource  
    resource:
      name: memory             # 内存使用率指标
      target:
        type: Utilization
        averageUtilization: 80 # 平均内存超过80%扩容
  
  # 扩缩容行为控制
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容稳定窗口（5分钟）
      policies:
      - type: Percent
        value: 50              # 每次最多缩容50%
        periodSeconds: 60      # 缩容间隔
    scaleUp:
      stabilizationWindowSeconds: 60   # 扩容稳定窗口（1分钟）
      policies:
      - type: Pods
        value: 2               # 每次最多扩容2个Pod
        periodSeconds: 60

---
# 订单服务HPA（基于自定义指标）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: microservices
  labels:
    app: order-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 1               # 最小1个实例
  maxReplicas: 8               # 最大8个实例
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60 # 订单服务较敏感，60%即扩容
        
  # 自定义指标（演示业务指标扩容）
  - type: Pods
    pods:
      metric:
        name: pending_orders   # 待处理订单数量指标
      target:
        type: AverageValue
        averageValue: "100"    # 平均每Pod处理100个订单
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # 更长缩容窗口（10分钟）
      policies:
      - type: Pods
        value: 1               # 每次只缩容1个Pod
        periodSeconds: 120     # 2分钟间隔
    scaleUp:
      stabilizationWindowSeconds: 30   # 快速扩容
      policies:
      - type: Pods  
        value: 3               # 每次最多扩容3个Pod
        periodSeconds: 30

---
# VPA垂直扩展（可选，演示资源动态调整）
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: user-service-vpa
  namespace: microservices
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  updatePolicy:
    updateMode: "Auto"         # 自动更新资源请求
  resourcePolicy:
    containerPolicies:
    - containerName: user-service
      maxAllowed:              # 最大资源限制
        cpu: 1000m
        memory: 1Gi
      minAllowed:              # 最小资源保证
        cpu: 100m
        memory: 128Mi
```

**微服务部署要点总结**：
1. **依赖管理**：使用initContainers等待依赖服务启动
2. **配置注入**：环境变量方式注入配置，支持动态更新
3. **健康检查**：三种探针确保服务可用性
4. **资源管理**：合理配置CPU/内存限制
5. **安全配置**：非root用户运行，最小权限原则
6. **监控集成**：Prometheus指标、链路追踪配置

---

### 5.3 Kubernetes部署策略案例

本节专注于Kubernetes核心部署策略的实战应用，重点学习**滚动更新、蓝绿部署、金丝雀发布、版本回滚**等云原生部署模式。

**实战重点**：
- **滚动更新**：零停机部署的标准方式
- **蓝绿部署**：快速切换和快速回滚
- **金丝雀发布**：渐进式风险控制
- **版本管理**：Deployment历史和回滚机制

#### **第一步：滚动更新（RollingUpdate）实战**

**K8s学习重点**：`strategy`配置、`maxUnavailable`/`maxSurge`参数、健康检查集成

```yaml
---
# 基础Web应用 - 演示滚动更新机制
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-rolling
  namespace: production
  labels:
    app: web-app-rolling
spec:
  replicas: 6                    # 足够的副本数演示滚动过程
  strategy:
    type: RollingUpdate          # 滚动更新策略（K8s默认）
    rollingUpdate:
      maxUnavailable: 25%        # 最多25%Pod不可用（向下取整）
      maxSurge: 25%              # 最多超出25%Pod（向上取整）
  selector:
    matchLabels:
      app: web-app-rolling
  template:
    metadata:
      labels:
        app: web-app-rolling
        version: v1.0            # 版本标签（重要）
    spec:
      containers:
      - name: web-app
        image: nginx:1.20        # 初始版本
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # 健康检查（滚动更新关键）
        readinessProbe:          # 就绪探针 - 决定是否接收流量
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:           # 存活探针 - 决定是否重启
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10

---
# Service配置（流量路由）
apiVersion: v1
kind: Service
metadata:
  name: web-app-rolling-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web-app-rolling        # 选择所有版本的Pod
  ports:
  - port: 80
    targetPort: 80
```

**滚动更新操作演示**：
```bash
# 1. 部署初始版本
kubectl apply -f rolling-update.yaml

# 2. 观察初始状态
kubectl get pods -l app=web-app-rolling -w

# 3. 执行滚动更新（镜像版本升级）
kubectl set image deployment/web-app-rolling web-app=nginx:1.21 -n production

# 4. 实时观察更新过程
kubectl rollout status deployment/web-app-rolling -n production

# 5. 查看更新历史
kubectl rollout history deployment/web-app-rolling -n production
```

#### **第二步：蓝绿部署（Blue-Green）实战**

**K8s学习重点**：标签选择器切换、Service路由、环境隔离

```yaml
---
# 蓝色环境（当前生产版本）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
  namespace: production
  labels:
    app: web-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: blue
  template:
    metadata:
      labels:
        app: web-app
        version: blue
        environment: production
    spec:
      containers:
      - name: web-app
        image: myapp:v1.0              # 当前生产版本
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "1.0"
        - name: ENVIRONMENT
          value: "blue"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# 绿色环境（新版本）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
  namespace: production
  labels:
    app: web-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: green
  template:
    metadata:
      labels:
        app: web-app
        version: green
        environment: production
    spec:
      containers:
      - name: web-app
        image: myapp:v2.0              # 新版本
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "2.0"
        - name: ENVIRONMENT
          value: "green"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# 主服务（流量切换的核心）
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
  labels:
    app: web-app
spec:
  type: ClusterIP
  selector:
    app: web-app
    version: blue                      # 初始指向蓝色环境
  ports:
  - port: 80
    targetPort: 8080
```

**蓝绿部署切换演示**：
```bash
# 1. 部署蓝绿环境
kubectl apply -f blue-green.yaml

# 2. 验证蓝色环境运行正常
kubectl get pods -l version=blue -n production

# 3. 部署绿色环境（新版本）
kubectl get pods -l version=green -n production

# 4. 测试绿色环境（使用临时Service）
kubectl port-forward deployment/web-app-green 8080:8080 -n production

# 5. 切换到绿色环境（关键步骤）
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"green"}}}'

# 6. 如需回滚到蓝色环境
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"blue"}}}'
```

#### **第三步：金丝雀发布（Canary）实战**

**K8s学习重点**：流量权重控制、渐进式部署、基于副本数的流量分配

```yaml
---
# 稳定版本（主要流量）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-stable
  namespace: production
  labels:
    app: web-app
    track: stable
spec:
  replicas: 9                          # 90%流量（9/10）
  selector:
    matchLabels:
      app: web-app
      track: stable
  template:
    metadata:
      labels:
        app: web-app
        track: stable
        version: v1.0
    spec:
      containers:
      - name: web-app
        image: myapp:v1.0              # 稳定版本
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "v1.0-stable"

---
# 金丝雀版本（少量流量）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-canary
  namespace: production
  labels:
    app: web-app
    track: canary
spec:
  replicas: 1                          # 10%流量（1/10）
  selector:
    matchLabels:
      app: web-app
      track: canary
  template:
    metadata:
      labels:
        app: web-app
        track: canary
        version: v2.0
    spec:
      containers:
      - name: web-app
        image: myapp:v2.0              # 金丝雀版本
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "v2.0-canary"

---
# 统一Service（自动流量分配）
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web-app                       # 同时选择stable和canary
  ports:
  - port: 80
    targetPort: 8080
```

**金丝雀发布流程演示**：
```bash
# 1. 部署稳定版本
kubectl apply -f canary-stable.yaml

# 2. 部署金丝雀版本（10%流量）
kubectl apply -f canary-deploy.yaml

# 3. 观察流量分配
kubectl get pods -l app=web-app -n production

# 4. 逐步增加金丝雀流量（20%）
kubectl scale deployment web-app-canary --replicas=2 -n production
kubectl scale deployment web-app-stable --replicas=8 -n production

# 5. 继续增加到50%
kubectl scale deployment web-app-canary --replicas=5 -n production
kubectl scale deployment web-app-stable --replicas=5 -n production

# 6. 完全切换到新版本
kubectl scale deployment web-app-stable --replicas=0 -n production
kubectl scale deployment web-app-canary --replicas=10 -n production
```

#### **第四步：版本回滚机制**

**K8s学习重点**：Deployment历史管理、`revisionHistoryLimit`、回滚命令

```yaml
---
# 支持版本回滚的Deployment配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-versioned
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 5
  revisionHistoryLimit: 10           # 保留10个历史版本
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: web-app-versioned
  template:
    metadata:
      labels:
        app: web-app-versioned
      annotations:
        # 版本注解（便于追踪）
        version: "v1.0"
        commit: "abc123"
        buildTime: "2024-01-15T10:00:00Z"
    spec:
      containers:
      - name: web-app
        image: myapp:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: APP_VERSION
          value: "v1.0"
```

**版本回滚操作演示**：
```bash
# 1. 查看当前状态
kubectl get deployment web-app-versioned -n production

# 2. 执行版本更新
kubectl set image deployment/web-app-versioned web-app=myapp:v2.0 -n production --record

# 3. 再次更新到v3.0
kubectl set image deployment/web-app-versioned web-app=myapp:v3.0 -n production --record

# 4. 查看版本历史
kubectl rollout history deployment/web-app-versioned -n production

# 5. 查看特定版本详情
kubectl rollout history deployment/web-app-versioned --revision=2 -n production

# 6. 回滚到上一个版本
kubectl rollout undo deployment/web-app-versioned -n production

# 7. 回滚到指定版本
kubectl rollout undo deployment/web-app-versioned --to-revision=1 -n production

# 8. 确认回滚状态
kubectl rollout status deployment/web-app-versioned -n production
```

**部署策略对比总结**：

| 策略 | 优势 | 适用场景 | K8s实现方式 |
|------|------|----------|-------------|
| **滚动更新** | 零停机、资源节约、自动化 | 日常版本更新 | Deployment默认策略 |
| **蓝绿部署** | 快速切换、完整回滚、环境隔离 | 重大版本发布 | 标签选择器切换 |
| **金丝雀发布** | 风险控制、渐进验证、用户反馈 | 不确定版本影响 | 副本数控制流量比例 |
| **版本回滚** | 快速恢复、历史追踪、安全保障 | 故障恢复 | Deployment历史机制 |

---

### 5.4 Kubernetes监控集成案例

本节专注于Kubernetes原生监控能力的实战应用，重点学习**Prometheus集成、指标暴露、ServiceMonitor、告警规则**等云原生监控模式。

**实战重点**：
- **注解监控**：利用Kubernetes注解自动配置监控
- **ServiceMonitor**：Prometheus Operator的声明式监控
- **Sidecar模式**：监控代理的容器化部署
- **指标标准化**：云原生监控指标规范

#### **第一步：Pod注解监控实战**

**K8s学习重点**：`prometheus.io/*`注解、自动服务发现、端口配置

```yaml
---
# 内置监控的Web应用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitored-web-app
  namespace: production
  labels:
    app: monitored-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: monitored-web-app
  template:
    metadata:
      labels:
        app: monitored-web-app
        tier: frontend
      annotations:
        # Prometheus自动发现注解（K8s标准）
        prometheus.io/scrape: "true"           # 启用监控抓取
        prometheus.io/port: "9090"             # 指标端口
        prometheus.io/path: "/metrics"         # 指标路径
        prometheus.io/scheme: "http"           # 协议类型
    spec:
      containers:
      - name: web-app
        image: myapp/web-with-metrics:v1.0     # 内置Prometheus指标
        ports:
        - name: http                           # 业务端口
          containerPort: 8080
        - name: metrics                        # 监控端口（关键）
          containerPort: 9090
        env:
        - name: METRICS_ENABLED
          value: "true"
        - name: METRICS_PORT
          value: "9090"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # 健康检查包含监控端点
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
        livenessProbe:
          httpGet:
            path: /metrics              # 监控端点同时作为健康检查
            port: 9090
          initialDelaySeconds: 15

---
# Service配置（监控发现）
apiVersion: v1
kind: Service
metadata:
  name: monitored-web-app-service
  namespace: production
  labels:
    app: monitored-web-app
  annotations:
    # Service级别监控注解
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  type: ClusterIP
  selector:
    app: monitored-web-app
  ports:
  - name: http                               # 业务端口
    port: 80
    targetPort: http
  - name: metrics                            # 监控端口（重要）
    port: 9090
    targetPort: metrics
```

#### **第二步：ServiceMonitor CRD实战**

**K8s学习重点**：自定义资源、Operator模式、声明式监控配置

```yaml
---
# ServiceMonitor（Prometheus Operator方式）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-monitor
  namespace: production
  labels:
    app: web-app-monitor
    prometheus: kube-prometheus
spec:
  # 服务选择器（选择要监控的Service）
  selector:
    matchLabels:
      app: monitored-web-app
  # 监控端点配置
  endpoints:
  - port: metrics                            # 监控端口名称
    path: /metrics                           # 指标路径
    interval: 30s                            # 抓取间隔
    scrapeTimeout: 10s                       # 超时时间
    # 指标重新标记（增强标签）
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'http_requests_total'
      targetLabel: 'custom_metric'
      replacement: 'web_requests'
  # 命名空间选择器
  namespaceSelector:
    matchNames:
    - production

---
# PrometheusRule（告警规则CRD）
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: web-app-alerts
  namespace: production
  labels:
    prometheus: kube-prometheus
spec:
  groups:
  - name: web-app-alerts
    rules:
    # 高错误率告警
    - alert: HighErrorRate
      expr: |
        (
          rate(http_requests_total{status=~"5.."}[5m]) /
          rate(http_requests_total[5m])
        ) > 0.1
      for: 5m
      labels:
        severity: critical
        service: web-app
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.instance }}"
    
    # 高延迟告警
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95,
          rate(http_request_duration_seconds_bucket[5m])
        ) > 0.5
      for: 5m
      labels:
        severity: warning
        service: web-app
      annotations:
        summary: "High latency detected"
        description: "95th percentile latency is {{ $value }}s for {{ $labels.instance }}"
    
    # Pod重启告警
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
        service: web-app
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is restarting frequently"
```

#### **第三步：Sidecar监控模式实战**

**K8s学习重点**：多容器Pod、容器间通信、共享存储、监控代理

```yaml
---
# Sidecar监控模式Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-sidecar-monitoring
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sidecar-monitored-app
  template:
    metadata:
      labels:
        app: sidecar-monitored-app
      annotations:
        # sidecar监控注解
        prometheus.io/scrape: "true"
        prometheus.io/port: "9102"             # sidecar暴露的端口
        prometheus.io/path: "/metrics"
    spec:
      containers:
      # 主应用容器（无需修改）
      - name: main-app
        image: myapp/legacy-app:v1.0           # 传统应用，无内置监控
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: app-logs                       # 日志共享
          mountPath: /var/log/app
        - name: app-metrics                    # 指标文件共享
          mountPath: /var/metrics
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
      
      # 监控Sidecar容器
      - name: metrics-exporter
        image: prom/node-exporter:latest       # Prometheus导出器
        ports:
        - name: metrics
          containerPort: 9102
        volumeMounts:
        - name: app-logs                       # 读取应用日志
          mountPath: /var/log/app
          readOnly: true
        - name: app-metrics                    # 读取应用指标
          mountPath: /var/metrics
          readOnly: true
        args:
        - '--path.rootfs=/host'
        - '--collector.filesystem.ignored-mount-points'
        - '^/(dev|proc|sys|var/lib/docker/.+)($|/)'
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      
      # 日志收集Sidecar（可选）
      - name: log-collector
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
      
      volumes:
      - name: app-logs                         # 日志共享卷
        emptyDir: {}
      - name: app-metrics                      # 指标共享卷
        emptyDir: {}
      - name: fluent-bit-config               # 日志配置
        configMap:
          name: fluent-bit-config

---
# Fluent Bit配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: production
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
    
    [INPUT]
        Name              tail
        Path              /var/log/app/*.log
        Parser            json
        Tag               app.logs
        Refresh_Interval  5
    
    [OUTPUT]
        Name  prometheus_exporter
        Match *
        Host  0.0.0.0
        Port  2020
```

#### **第四步：监控指标标准化**

**K8s学习重点**：标准指标、标签规范、监控最佳实践

```yaml
---
# 标准化监控配置示例
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-standards
  namespace: production
data:
  # 标准指标规范
  metrics-spec.yaml: |
    # 应用级别指标
    application_metrics:
      - name: http_requests_total
        type: counter
        labels: [method, status, endpoint]
        description: "Total HTTP requests"
      
      - name: http_request_duration_seconds
        type: histogram
        labels: [method, endpoint]
        description: "HTTP request duration"
      
      - name: application_info
        type: gauge
        labels: [version, environment]
        description: "Application information"
    
    # 业务级别指标
    business_metrics:
      - name: orders_total
        type: counter
        labels: [status, region]
        description: "Total orders processed"
      
      - name: revenue_total
        type: counter
        labels: [currency, region]
        description: "Total revenue"
    
    # 基础设施指标
    infrastructure_metrics:
      - name: pod_cpu_usage_percent
        type: gauge
        labels: [pod, namespace, node]
        description: "Pod CPU usage percentage"
      
      - name: pod_memory_usage_bytes
        type: gauge
        labels: [pod, namespace, node]
        description: "Pod memory usage in bytes"

---
# 监控标签标准化
apiVersion: apps/v1
kind: Deployment
metadata:
  name: standardized-monitored-app
  namespace: production
  labels:
    # 标准化标签
    app.kubernetes.io/name: web-app
    app.kubernetes.io/instance: web-app-prod
    app.kubernetes.io/version: "v1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: e-commerce-platform
    app.kubernetes.io/managed-by: kubernetes
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: web-app
      app.kubernetes.io/instance: web-app-prod
  template:
    metadata:
      labels:
        # 继承标准化标签
        app.kubernetes.io/name: web-app
        app.kubernetes.io/instance: web-app-prod
        app.kubernetes.io/version: "v1.0"
        app.kubernetes.io/component: frontend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        # 监控元数据注解
        monitoring.coreos.com/team: "platform-team"
        monitoring.coreos.com/runbook: "https://runbooks.example.com/web-app"
    spec:
      containers:
      - name: web-app
        image: myapp/web-app:v1.0
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        env:
        # 标准化环境变量
        - name: PROMETHEUS_METRICS_ENABLED
          value: "true"
        - name: PROMETHEUS_METRICS_PORT
          value: "9090"
        - name: PROMETHEUS_METRICS_PATH
          value: "/metrics"
        - name: APP_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['app.kubernetes.io/name']
        - name: APP_VERSION
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['app.kubernetes.io/version']
```

**监控集成最佳实践总结**：

1. **注解驱动**：使用标准Prometheus注解实现自动发现
2. **标签规范**：遵循Kubernetes推荐标签和监控标签标准
3. **端口标准化**：业务端口、监控端口、健康检查端口分离
4. **指标标准化**：定义统一的指标命名和标签规范
5. **告警分级**：critical、warning、info三级告警体系
6. **Sidecar模式**：对传统应用的无侵入监控改造

---

## 第二篇总结

通过本篇的实战学习，您已经掌握了Kubernetes应用部署和管理的核心技能：

### 🎯 主要收获

**配置管理**：
- 掌握了YAML语法和Kubernetes资源配置结构
- 学会了ConfigMap、Secret等配置外化最佳实践
- 理解了命名空间、标签、注解的使用场景

**实战部署**：
- 完成了Web应用全栈部署（前端+后端+数据库）
- 掌握了微服务架构的Service发现和通信模式
- 学会了不同类型应用的部署策略和优化方法

**部署策略**：
- 理解了滚动更新、蓝绿部署、金丝雀发布的适用场景
- 掌握了版本管理和回滚机制的操作方法
- 学会了基于Kubernetes原生能力的部署策略实现

**监控集成**：
- 掌握了Prometheus注解驱动的监控配置
- 学会了ServiceMonitor CRD的声明式监控
- 理解了Sidecar模式的监控代理部署方式

### 🚀 下一步学习建议

现在您已经具备了Kubernetes应用部署的实战能力，建议：

1. **深化实践** - 在真实项目中应用所学技能
2. **生产准备** - 学习第三篇的生产环境最佳实践
3. **工具进阶** - 学习Helm、Kustomize等高级工具
4. **监控完善** - 构建完整的可观测性体系

### 📚 相关资源

- **继续阅读**：[第三篇：生产环境与最佳实践](K8S_PART3_PRODUCTION.md)
- **官方文档**：[Kubernetes官方文档](https://kubernetes.io/docs/)
- **实践环境**：推荐使用Kind、Minikube或云厂商的托管Kubernetes服务

---

## 系列文章索引

- [第一篇：基础概念与架构](K8S_PART1_FUNDAMENTALS.md)
- [第二篇：配置管理与实战应用](K8S_PART2_PRACTICE.md)  
- [第三篇：生产环境与最佳实践](K8S_PART3_PRODUCTION.md)
