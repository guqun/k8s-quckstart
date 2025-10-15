# 完整 CI/CD 流程详解

## 概述

本文档详细说明从代码提交到应用部署的完整 GitLab CI/CD 流程，包括镜像构建、推送、部署的每个步骤。

---

## 一、整体流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     GitLab CI/CD Pipeline                        │
│                                                                  │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐            │
│  │  Stage:  │       │  Stage:  │       │  Stage:  │            │
│  │  build   │  ───→ │  deploy  │  ───→ │   运行   │            │
│  └──────────┘       └──────────┘       └──────────┘            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

详细展开：

1️⃣ 开发者推送代码到 main 分支
        ↓
2️⃣ GitLab 触发 Pipeline
        ↓
3️⃣ Runner Manager 接收 build-job 任务
        ↓
4️⃣ 创建 Job Pod（Build + DinD + Helper）
        ↓
5️⃣ Helper 克隆代码
        ↓
6️⃣ Build Container 执行 docker build
        ↓
7️⃣ Build Container 推送镜像到 ACR
        ↓
8️⃣ build-job 完成，Pod 销毁
        ↓
9️⃣ Runner Manager 接收 deploy-job 任务（手动触发）
        ↓
🔟 创建 Deploy Pod（kubectl 镜像）
        ↓
⓫ 使用 envsubst 替换 YAML 中的镜像变量
        ↓
⓬ 通过 ServiceAccount 在当前集群部署应用
        ↓
⓭ 应用通过 NodePort 对外提供服务
```

---

## 二、Pipeline 配置解析

### 2.1 全局配置

```yaml
stages:
  - build   # 阶段 1：构建镜像
  - deploy  # 阶段 2：部署应用

variables:
  # 动态镜像 tag：使用 commit SHA + pipeline ID
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

  # Kubeconfig 文件路径（deploy-job 使用）
  KUBECONFIG: "/tmp/kubeconfig"
```

**变量说明**:
- `CI_COMMIT_SHORT_SHA`: GitLab 内置变量，当前 commit 的短 SHA（8位）
- `CI_PIPELINE_ID`: GitLab 内置变量，当前 pipeline 的唯一 ID
- `ACR_USERNAME`, `ACR_PASSWORD`: 在 GitLab 项目设置中配置的 Secret 变量
- `ACK_KUBECONFIG`: 目标 K8s 集群的 kubeconfig（本例实际未使用）

---

## 三、Build 阶段详解

### 3.1 Job 配置

```yaml
build-job:
  stage: build
  tags:
    - k8s-runner  # 指定使用 k8s-runner 这个 Runner

  image: registry.example.com/library/docker:latest

  services:
    - name: registry.example.com/library/docker:dind
      alias: docker
      command:
        - "--tls=false"  # 禁用 TLS，简化配置
        - "--insecure-registry=private-registry.example.com"

  variables:
    DOCKER_HOST: tcp://docker:2375      # 连接到 DinD
    DOCKER_TLS_CERTDIR: ""              # 禁用 TLS 证书目录

  rules:
    - if: $CI_COMMIT_BRANCH == "main"   # 仅在 main 分支触发
      when: always
```

### 3.2 before_script：等待 DinD 启动

```yaml
before_script:
  - echo "等待 Docker 守护进程启动..."
  - |
    timeout=60
    until docker info >/dev/null 2>&1; do
      if [ $timeout -le 0 ]; then
        echo "❌ 等待超时：Docker 守护进程未启动"
        echo "尝试查看 dockerd 日志..."
        nc -zv docker 2375 || echo "端口 2375 不可达"
        exit 1
      fi
      echo "等待中... (剩余 ${timeout}s)"
      sleep 2
      timeout=$((timeout - 2))
    done
  - echo "✅ Docker 守护进程已就绪"
  - docker version
  - docker info
```

**作用**:
1. 等待 DinD 容器完全启动（通常需要 5-10 秒）
2. 验证 Docker 客户端可以连接到 DinD 守护进程
3. 显示 Docker 版本信息，便于调试

### 3.3 script：构建与推送镜像

```yaml
script:
  # 1. 验证 Docker 连接
  - docker info

  # 2. 构建镜像
  - docker build -t $DOCKER_IMAGE .

  # 3. 登录私有镜像仓库
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin private-registry.example.com

  # 4. 推送镜像
  - docker push $DOCKER_IMAGE

  # 5. 输出成功信息
  - echo "构建完成，镜像已推送ACR：$DOCKER_IMAGE"
```

### 3.4 执行流程时序图

```
时间线 →

0s   │ Runner Manager 接收任务
     │
1s   │ 创建 Job Pod
     │ ├── Init: Helper Container 启动
     │ │   └── 克隆代码到 /builds/
     │
3s   │ Helper 完成，进入等待
     │
4s   │ 启动 Build Container
     │ 启动 Service Container (DinD)
     │
5s   │ Build Container 运行 before_script
     │ └── 等待 DinD 启动
     │
8s   │ DinD 启动完成，监听 tcp://0.0.0.0:2375
     │
9s   │ docker info 成功
     │ before_script 完成
     │
10s  │ 开始执行 script
     │ └── docker build -t <image> .
     │
11s  │ DinD 开始构建镜像
     │ ├── 读取 Dockerfile
     │ ├── DNS 解析 registry.example.com
     │ │   └── 查询 10.0.0.2 (节点 DNS)
     │ ├── 拉取基础镜像 python:3.9-slim
     │
45s  │ 基础镜像下载完成
     │ ├── 执行 COPY . .
     │ ├── 执行 RUN pip install -r requirements.txt
     │
120s │ 镜像构建完成
     │ 镜像 ID: sha256:abc123...
     │
125s │ docker login 登录 ACR
     │
130s │ docker push 推送镜像
     │ ├── 推送 layer 1 (already exists)
     │ ├── 推送 layer 2 (已存在，跳过)
     │ ├── 推送 layer 3 (新增，上传)
     │
160s │ 推送完成
     │ Tag: 56cc4792-86
     │
165s │ script 完成
     │ build-job 状态：success
     │
170s │ Helper Container 上传 artifacts（如果有）
     │
175s │ Job Pod 销毁
```

---

## 四、Deploy 阶段详解

### 4.1 Job 配置

```yaml
deploy-job:
  stage: deploy
  tags:
    - k8s-runner

  # 使用包含 kubectl 的镜像
  image: registry.example.com/bitnami/kubectl:latest

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # 手动触发，防止误部署
```

### 4.2 before_script：准备部署环境

```yaml
before_script:
  # 1. 创建 kubeconfig 目录
  - mkdir -p $(dirname $KUBECONFIG)

  # 2. 写入 kubeconfig 内容（从 Secret 变量）
  - printf '%s' "$ACK_KUBECONFIG" > $KUBECONFIG

  # 3. 设置权限
  - chmod 600 $KUBECONFIG

  # 4. 验证 kubeconfig（可选）
  - kubectl --kubeconfig=$KUBECONFIG config view

  # 5. 验证集群连接
  - kubectl cluster-info

  # 6. 创建镜像拉取凭证
  - kubectl create secret docker-registry acr-secret
      --docker-server=private-registry.example.com
      --docker-username=$ACR_USERNAME
      --docker-password=$ACR_PASSWORD
      --namespace=default
      --dry-run=client -o yaml | kubectl apply -f -
```

**关键步骤**:

**Step 6**: 创建 Docker Registry Secret
- 使用 `--dry-run=client -o yaml` 生成 YAML
- 通过 `kubectl apply` 应用（幂等操作，已存在则更新）
- Secret 用于 Pod 拉取私有镜像

### 4.3 script：部署应用

```yaml
script:
  # 1. 使用 envsubst 替换环境变量
  - envsubst < calculator.yaml > calculator-deploy.yaml

  # 2. 应用部署配置
  - kubectl --kubeconfig=$KUBECONFIG apply -f calculator-deploy.yaml

  # 3. 输出成功信息
  - echo "ACK集群部署完成"
```

**envsubst 工作原理**:

**输入文件** (`calculator.yaml`):
```yaml
spec:
  containers:
  - name: calculator
    image: ${DOCKER_IMAGE}  # 环境变量占位符
```

**环境变量**:
```bash
DOCKER_IMAGE="private-registry.example.com/my-namespace/my-app:56cc4792-86"
```

**输出文件** (`calculator-deploy.yaml`):
```yaml
spec:
  containers:
  - name: calculator
    image: private-registry.example.com/my-namespace/my-app:56cc4792-86
```

### 4.4 after_script：清理临时文件

```yaml
after_script:
  - rm -f $KUBECONFIG calculator-deploy.yaml
```

**作用**:
- 删除包含敏感信息的 kubeconfig 文件
- 删除临时生成的部署配置
- 无论 job 成功或失败都会执行

---

## 五、Kubernetes 部署配置解析

### 5.1 Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calculator
  namespace: default
spec:
  replicas: 2  # 2 个副本，提供高可用

  selector:
    matchLabels:
      app: calculator  # 选择器，匹配 Pod 标签

  template:
    metadata:
      labels:
        app: calculator  # Pod 标签

    spec:
      # 镜像拉取凭证（引用 deploy-job 创建的 Secret）
      imagePullSecrets:
      - name: acr-secret

      containers:
      - name: calculator
        image: ${DOCKER_IMAGE}  # 由 envsubst 替换
        imagePullPolicy: Always  # 总是拉取最新镜像

        ports:
        - name: http
          containerPort: 30194  # 应用监听端口

        # 时区配置（挂载节点时间）
        volumeMounts:
        - name: host-time
          mountPath: /etc/localtime
          readOnly: true

      volumes:
      - name: host-time
        hostPath:
          path: /etc/localtime  # 节点时区文件
```

**关键配置说明**:

1. **imagePullSecrets**: 引用 `acr-secret` Secret，用于拉取私有镜像
2. **imagePullPolicy: Always**: 确保每次都拉取最新镜像（因为 tag 总是变化）
3. **containerPort: 30194**: 应用内部监听的端口（Flask 应用配置）

### 5.2 Service 配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: calculator
  namespace: default
spec:
  type: NodePort  # 通过节点端口暴露服务

  selector:
    app: calculator  # 选择带有此标签的 Pod

  ports:
  - port: 30194        # Service 内部端口
    nodePort: 30194    # 节点暴露端口（30000-32767）
    targetPort: 30194  # Pod 容器端口
```

**端口说明**:
- **port**: Service 的 ClusterIP 端口（集群内部访问）
- **targetPort**: Pod 容器的监听端口
- **nodePort**: 节点上暴露的端口（外部访问）

**访问方式**:
```bash
# 集群内部访问
curl http://calculator.default.svc.cluster.local:30194/

# 外部访问（任意节点 IP）
curl http://192.168.1.101:30194/
curl http://192.168.1.102:30194/
```

---

## 六、完整流程示例

### 6.1 场景：开发者修改代码并部署

**Step 1**: 开发者提交代码
```bash
git add app.py
git commit -m "fix: 修复计算器除法bug"
git push origin main
```

**Step 2**: GitLab 自动触发 Pipeline
```
Pipeline #86 启动
├── Stage: build
│   └── build-job (自动运行)
└── Stage: deploy
    └── deploy-job (等待手动触发)
```

**Step 3**: build-job 执行（自动）
```
[00:00] Job Pod 创建
[00:03] Helper 克隆代码
[00:05] DinD 启动，Build 容器启动
[00:10] docker build 开始
  ├── 拉取基础镜像 python:3.9-slim (35s)
  ├── COPY 代码 (1s)
  ├── pip install (15s)
  └── 构建完成
[01:00] docker login ACR
[01:05] docker push
  ├── Layer 1: already exists
  ├── Layer 2: already exists
  ├── Layer 3: Pushed (新代码)
  └── Tag: 56cc4792-86
[01:40] build-job 完成 ✅
```

**Step 4**: 开发者手动触发 deploy-job
```
GitLab UI → Pipelines → Pipeline #86 → deploy-job → [运行]
```

**Step 5**: deploy-job 执行（手动）
```
[00:00] Deploy Pod 创建
[00:03] Helper 克隆代码
[00:05] kubectl 容器启动
[00:08] 验证 kubeconfig
[00:10] kubectl cluster-info (连接集群)
[00:12] 创建 acr-secret
[00:15] envsubst 替换镜像变量
  输入: image: ${DOCKER_IMAGE}
  输出: image: private-registry.example.com/.../repository:56cc4792-86
[00:18] kubectl apply -f calculator-deploy.yaml
  ├── Deployment "calculator" configured
  └── Service "calculator" unchanged
[00:20] deploy-job 完成 ✅
```

**Step 6**: Kubernetes 滚动更新应用
```
[00:20] Deployment 检测到新镜像
[00:22] 创建新 ReplicaSet
[00:25] 启动新 Pod (calculator-767f559f9c-xxxxx)
  ├── 拉取镜像 (使用 acr-secret)
  ├── 镜像下载完成 (3s)
  └── 容器启动
[00:30] 新 Pod 就绪
[00:32] 删除旧 Pod
[00:35] 滚动更新完成 ✅
```

**Step 7**: 验证部署
```bash
# 查看 Pod 状态
kubectl get pods -n default -l app=calculator
NAME                          READY   STATUS    RESTARTS   AGE
calculator-767f559f9c-2j7vz   1/1     Running   0          5m
calculator-767f559f9c-5qnx4   1/1     Running   0          5m

# 访问应用
curl http://192.168.1.101:30194/
# 返回：Hello from Calculator v2!
```

---

## 七、关键变量传递链路

### 7.1 镜像 Tag 的传递

```
1. Pipeline 启动时生成
   CI_COMMIT_SHORT_SHA = "56cc4792"
   CI_PIPELINE_ID = "86"
   ↓
2. 合成全局变量
   DOCKER_IMAGE = "cr-ee...repository:56cc4792-86"
   ↓
3. build-job 使用
   docker build -t $DOCKER_IMAGE .
   docker push $DOCKER_IMAGE
   ↓
4. deploy-job 继承（全局变量）
   envsubst 读取 $DOCKER_IMAGE
   ↓
5. 写入 Kubernetes YAML
   image: cr-ee...repository:56cc4792-86
   ↓
6. Kubelet 拉取镜像
   使用 imagePullSecrets: acr-secret
```

### 7.2 认证信息传递

```
1. GitLab 项目 Settings → CI/CD → Variables
   设置:
   - ACR_USERNAME = admin
   - ACR_PASSWORD = ********
   - ACK_KUBECONFIG = <base64_encoded_kubeconfig>
   ↓
2. Pipeline 执行时，作为环境变量注入
   ↓
3. build-job 使用
   echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME ...
   ↓
4. deploy-job 使用
   kubectl create secret docker-registry ...
     --docker-username=$ACR_USERNAME
     --docker-password=$ACR_PASSWORD
   ↓
5. Kubernetes Secret 创建
   name: acr-secret
   ↓
6. Pod 使用 Secret 拉取镜像
   imagePullSecrets:
   - name: acr-secret
```

---

## 八、常见问题与优化

### 8.1 镜像 Tag 冲突

**问题**:
```
unknown: The requested tag already exists and cannot be overwritten.
```

**原因**:
- 重新运行同一个 commit 的 pipeline
- tag 相同，镜像仓库不允许覆盖

**解决方案**:
```yaml
# 方案 1：添加 pipeline ID（已采用）
DOCKER_IMAGE: "...repository:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

# 方案 2：添加时间戳
DOCKER_IMAGE: "...repository:${CI_COMMIT_SHORT_SHA}-$(date +%s)"

# 方案 3：忽略错误
script:
  - docker push $DOCKER_IMAGE || echo "镜像已存在，跳过"
```

### 8.2 部署失败：镜像拉取错误

**问题**:
```
Failed to pull image: pull access denied
```

**排查步骤**:
1. 检查 Secret 是否存在：`kubectl get secret acr-secret -n default`
2. 检查 Secret 内容：`kubectl get secret acr-secret -o yaml`
3. 测试凭证：`echo $ACR_PASSWORD | docker login -u $ACR_USERNAME ...`
4. 检查 Pod 配置：`kubectl get pod <pod> -o yaml | grep imagePullSecrets`

### 8.3 应用无法访问

**问题**:
- 部署成功，但访问 NodePort 无响应

**排查步骤**:
1. 检查 Pod 状态：`kubectl get pods -n default -l app=calculator`
2. 检查 Service：`kubectl get svc calculator -n default`
3. 检查 Endpoints：`kubectl get endpoints calculator -n default`
4. 测试 Pod 内部：`kubectl exec <pod> -- curl localhost:30194`
5. 测试 Service：`kubectl run test --rm -i --image=alpine -- wget -O- http://calculator.default:30194`
6. 检查节点端口：`netstat -tuln | grep 30194`
7. 检查安全组：确认节点安全组开放 30194 端口

### 8.4 性能优化建议

**构建优化**:
```yaml
# 使用 Docker 缓存
script:
  - docker build --cache-from $DOCKER_IMAGE:latest -t $DOCKER_IMAGE .
  - docker tag $DOCKER_IMAGE $DOCKER_IMAGE:latest
  - docker push $DOCKER_IMAGE
  - docker push $DOCKER_IMAGE:latest
```

**部署优化**:
```yaml
# 添加健康检查
livenessProbe:
  httpGet:
    path: /health
    port: 30194
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 30194
  initialDelaySeconds: 5
  periodSeconds: 5
```

**资源限制**:
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

## 九、最佳实践总结

### 9.1 CI/CD Pipeline

✅ **推荐做法**:
- 使用语义化版本或 commit SHA 作为镜像 tag
- build 阶段自动触发，deploy 阶段手动触发
- 在 before_script 中验证依赖服务可用性
- 使用 Secret 变量存储敏感信息
- 添加详细的日志输出，便于调试

❌ **避免**:
- 使用 `latest` tag（无法追溯历史版本）
- 在脚本中硬编码密码
- 省略错误处理
- 缺少资源限制导致集群资源耗尽

### 9.2 Kubernetes 部署

✅ **推荐做法**:
- 使用 Deployment 而不是裸 Pod
- 配置多副本提供高可用
- 添加健康检查（liveness/readiness probe）
- 设置资源 requests 和 limits
- 使用 Secret 管理敏感配置

❌ **避免**:
- 使用 hostPath 挂载（除非必要）
- 缺少健康检查
- 没有资源限制
- 直接在 YAML 中写死敏感信息

### 9.3 镜像管理

✅ **推荐做法**:
- 使用多阶段构建减小镜像体积
- 使用明确的基础镜像版本（避免 latest）
- 定期清理旧镜像
- 配置镜像扫描检测漏洞

❌ **避免**:
- 在镜像中包含源代码（除非必要）
- 使用过大的基础镜像
- 忽略镜像安全扫描

---

## 十、完整配置清单

### 10.1 必需文件

| 文件 | 位置 | 作用 |
|------|------|------|
| `.gitlab-ci.yml` | 项目根目录 | 定义 CI/CD 流程 |
| `Dockerfile` | 项目根目录 | 定义镜像构建步骤 |
| `calculator.yaml` | 项目根目录 | Kubernetes 部署配置 |
| `gitlab-runner-values.yaml` | Runner 配置 | Runner Helm Chart 配置 |

### 10.2 必需的 GitLab Variables

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `ACR_USERNAME` | Secret | 镜像仓库用户名 |
| `ACR_PASSWORD` | Secret | 镜像仓库密码 |
| `ACK_KUBECONFIG` | Secret | K8s 集群配置（如需跨集群部署） |

### 10.3 必需的 Kubernetes 资源

| 资源类型 | 名称 | 命名空间 | 说明 |
|---------|------|---------|------|
| Namespace | `gitlab` | - | Runner 运行空间 |
| ServiceAccount | `gitlab-runner` | gitlab | Runner 使用的账号 |
| Role | `gitlab-runner-deploy` | default | 部署权限 |
| RoleBinding | `gitlab-runner-deploy` | default | 权限绑定 |
| Secret | `acr-secret` | default | 镜像拉取凭证 |

---

## 十一、总结

### 完整流程回顾

```
代码提交 → Pipeline 触发 → build-job 执行 → 镜像构建推送
→ deploy-job 手动触发 → K8s 滚动更新 → 应用对外服务
```

### 关键技术点

1. **GitLab Runner Kubernetes Executor**: 动态创建 Job Pod
2. **Docker-in-Docker**: 在容器中构建容器镜像
3. **DNS 配置**: 解决内网环境域名解析问题
4. **ServiceAccount**: 跨命名空间部署权限
5. **envsubst**: 动态替换部署配置中的变量
6. **NodePort Service**: 对外暴露应用

### 成功指标

- ✅ 代码提交后自动构建镜像
- ✅ 镜像成功推送到私有仓库
- ✅ 一键部署到 Kubernetes 集群
- ✅ 应用自动滚动更新
- ✅ 通过 NodePort 可访问应用
- ✅ 支持多副本高可用部署

**当前配置已实现完整的 CI/CD 自动化流程！** 🎉
