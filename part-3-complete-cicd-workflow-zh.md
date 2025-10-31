# Kubernetes 上的 GitLab Runner：完整指南
## 第三部分 — 构建真实世界的 CI/CD 流水线

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

> *回顾：[第一部分](link)介绍了搭建，[第二部分](link)解释了容器架构。*

---

在第一和第二部分中，我们部署了 runner 并理解了四个容器如何协同工作。现在让我们构建一些真实的东西：**一个完整的 CI/CD 流水线，构建 Docker 镜像，推送到仓库，并部署到 Kubernetes**。

在本部分结束时，你将拥有：
- 一个包含构建和部署阶段的可工作的 `.gitlab-ci.yml`
- 使用 DinD（Docker-in-Docker）的 Docker 镜像构建
- 自动部署到你的 Kubernetes 集群
- 理解变量如何流经流水线

让我们从完整流程概览开始。

---

## 完整的流水线流程

这是我们正在构建的内容：

```
┌─────────────────────────────────────────────────────────────┐
│                    GitLab CI/CD 流水线                       │
│                                                              │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│  │  阶段:   │       │  阶段:   │       │   应用   │        │
│  │  build   │  ───→ │  deploy  │  ───→ │  运行中  │        │
│  └──────────┘       └──────────┘       └──────────┘        │
└─────────────────────────────────────────────────────────────┘

详细流程：

1️⃣  开发者推送代码到 main 分支
        ↓
2️⃣  GitLab 触发流水线
        ↓
3️⃣  Runner Manager 接收 build-job
        ↓
4️⃣  创建任务 Pod（Build + DinD + Helper）
        ↓
5️⃣  Helper 克隆代码
        ↓
6️⃣  Build 容器：docker build
        ↓
7️⃣  Build 容器：docker push 到仓库
        ↓
8️⃣  build-job 完成，Pod 删除
        ↓
9️⃣  Runner Manager 接收 deploy-job（手动触发）
        ↓
🔟 创建 Deploy Pod（kubectl 镜像）
        ↓
⓫  envsubst 替换 YAML 中的镜像变量
        ↓
⓬  kubectl 将部署应用到集群
        ↓
⓭  应用部署，通过 NodePort 可访问
```

让我们一步步实现这个。

---

## 项目结构

我们将使用一个简单的 Python Flask 应用程序：

```
my-app/
├── .gitlab-ci.yml          # CI/CD 配置
├── Dockerfile              # 镜像构建指令
├── calculator.yaml         # Kubernetes 部署配置
├── app.py                  # 应用代码
└── requirements.txt        # Python 依赖
```

### 示例应用程序

**app.py**:
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        "message": "Calculator API",
        "version": "1.0"
    })

@app.route('/add', methods=['POST'])
def add():
    data = request.json
    result = data['a'] + data['b']
    return jsonify({"result": result})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=30194)
```

**requirements.txt**:
```
Flask==2.3.0
```

**Dockerfile**:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 30194

CMD ["python", "app.py"]
```

---

## 流水线配置

现在是 CI/CD 配置。这是一切汇集的地方。

### 完整的 .gitlab-ci.yml

```yaml
stages:
  - build   # 阶段 1：构建并推送 Docker 镜像
  - deploy  # 阶段 2：部署到 Kubernetes

variables:
  # 动态镜像标签：commit SHA + pipeline ID
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

  # deploy job 的 Kubeconfig 路径
  KUBECONFIG: "/tmp/kubeconfig"

#################################################################
# BUILD JOB：构建 Docker 镜像并推送到仓库
#################################################################

build-job:
  stage: build
  tags:
    - kubernetes  # 路由到 K8s runner

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

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always

  before_script:
    - echo "等待 Docker 守护进程启动..."
    - |
      timeout=60
      until docker info >/dev/null 2>&1; do
        if [ $timeout -le 0 ]; then
          echo "❌ 超时：Docker 守护进程未启动"
          echo "尝试检查 dockerd 日志..."
          nc -zv docker 2375 || echo "端口 2375 不可达"
          exit 1
        fi
        echo "等待中... (剩余 ${timeout} 秒)"
        sleep 2
        timeout=$((timeout - 2))
      done
    - echo "✅ Docker 守护进程就绪"
    - docker version
    - docker info

  script:
    # 1. 验证 Docker 连接
    - docker info

    # 2. 构建镜像
    - docker build -t $DOCKER_IMAGE .

    # 3. 登录到私有仓库
    - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin private-registry.example.com

    # 4. 推送镜像
    - docker push $DOCKER_IMAGE

    # 5. 输出成功消息
    - echo "构建完成，镜像已推送到仓库：$DOCKER_IMAGE"

#################################################################
# DEPLOY JOB：部署到 Kubernetes 集群
#################################################################

deploy-job:
  stage: deploy
  tags:
    - kubernetes

  # 使用带 kubectl 的镜像
  image: registry.example.com/bitnami/kubectl:latest

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # 手动触发以防止意外部署

  before_script:
    # 1. 创建 kubeconfig 目录
    - mkdir -p $(dirname $KUBECONFIG)

    # 2. 从 secret 变量写入 kubeconfig
    - printf '%s' "$ACK_KUBECONFIG" > $KUBECONFIG

    # 3. 设置权限
    - chmod 600 $KUBECONFIG

    # 4. 验证 kubeconfig
    - kubectl --kubeconfig=$KUBECONFIG config view

    # 5. 验证集群连接
    - kubectl cluster-info

    # 6. 创建镜像拉取 secret
    - kubectl create secret docker-registry acr-secret
        --docker-server=private-registry.example.com
        --docker-username=$ACR_USERNAME
        --docker-password=$ACR_PASSWORD
        --namespace=default
        --dry-run=client -o yaml | kubectl apply -f -

  script:
    # 1. 替换 YAML 中的环境变量
    - envsubst < calculator.yaml > calculator-deploy.yaml

    # 2. 应用部署
    - kubectl --kubeconfig=$KUBECONFIG apply -f calculator-deploy.yaml

    # 3. 输出成功消息
    - echo "部署完成到集群"

  after_script:
    # 清理敏感文件
    - rm -f $KUBECONFIG calculator-deploy.yaml
```

让我们分解每个部分。

---

## Build Job 深入探讨

### 全局变量

```yaml
variables:
  DOCKER_IMAGE: "private-registry.example.com/my-namespace/my-app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
```

**为什么这种格式？**
- `CI_COMMIT_SHORT_SHA`：Git 提交哈希（8个字符）- 确保可追溯性
- `CI_PIPELINE_ID`：唯一的流水线 ID - 防止重新运行时的标签冲突
- 示例结果：`my-app:56cc4792-86`

**替代策略**：
```yaml
# 语义化版本
DOCKER_IMAGE: "registry.com/app:v1.2.3"

# 基于日期
DOCKER_IMAGE: "registry.com/app:$(date +%Y%m%d-%H%M%S)"

# 分支 + SHA
DOCKER_IMAGE: "registry.com/app:${CI_COMMIT_REF_SLUG}-${CI_COMMIT_SHORT_SHA}"
```

### Docker-in-Docker 配置

```yaml
services:
  - name: registry.example.com/library/docker:dind
    alias: docker
    command:
      - "--tls=false"
      - "--insecure-registry=private-registry.example.com"
```

**关键参数**：

| 参数 | 目的 | 安全提示 |
|------|------|----------|
| `alias: docker` | DinD 的网络主机名 | 没有这个，主机名将是完整的镜像路径 |
| `--tls=false` | 禁用客户端/服务器之间的 TLS | ⚠️ 仅在受信任的网络中使用 |
| `--insecure-registry` | 允许非 HTTPS 仓库 | 内部仓库没有证书时需要 |

### 等待 Docker 守护进程

这很关键——DinD 需要5-10秒来启动：

```yaml
before_script:
  - |
    timeout=60
    until docker info >/dev/null 2>&1; do
      if [ $timeout -le 0 ]; then
        exit 1
      fi
      sleep 2
      timeout=$((timeout - 2))
    done
```

**没有这个会发生什么？**
```
$ docker build -t myapp .
Cannot connect to the Docker daemon at tcp://docker:2375
Is the docker daemon running?
❌ 任务立即失败
```

**有适当的等待**：
```
等待中... (剩余 60 秒)
等待中... (剩余 58 秒)
✅ Docker 守护进程就绪
成功连接到服务器
```

### 构建和推送过程

```yaml
script:
  - docker build -t $DOCKER_IMAGE .
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin private-registry.example.com
  - docker push $DOCKER_IMAGE
```

**执行时间线**（来自第二部分的时序分析）：

```
T+10s   docker build 开始
        ├─ DinD 拉取基础镜像：python:3.9-slim（35秒）
        ├─ 复制应用代码（1秒）
        ├─ 运行 pip install（15秒）
        └─ 创建镜像层

T+61s   构建完成
        镜像 ID：sha256:abc123...

T+62s   docker login

T+65s   docker push 开始
        ├─ Layer 1：already exists（跳过）
        ├─ Layer 2：already exists（跳过）
        ├─ Layer 3：Pushed（新代码）
        └─ Manifest pushed

T+95s   推送完成
        标签：56cc4792-86
```

**总 build-job 时间**：约95秒（根据镜像大小而异）

---

## Deploy Job 深入探讨

deploy job 在一个单独的 Pod 中运行，使用 `kubectl` 镜像。

### 认证设置

```yaml
before_script:
  - printf '%s' "$ACK_KUBECONFIG" > $KUBECONFIG
  - chmod 600 $KUBECONFIG
  - kubectl cluster-info
```

**重要**：kubeconfig 存储为 GitLab CI/CD 变量：

1. 进入 **Settings** → **CI/CD** → **Variables**
2. 添加变量：
   - Key：`ACK_KUBECONFIG`
   - Value：你的 `~/.kube/config` 内容（base64 编码或原始）
   - Type：File 或 Variable
   - Protected：✅
   - Masked：❌（太长无法屏蔽）

### 镜像拉取 Secret 创建

```yaml
kubectl create secret docker-registry acr-secret \
  --docker-server=private-registry.example.com \
  --docker-username=$ACR_USERNAME \
  --docker-password=$ACR_PASSWORD \
  --namespace=default \
  --dry-run=client -o yaml | kubectl apply -f -
```

**为什么 `--dry-run=client -o yaml | kubectl apply`？**
- `--dry-run=client`：生成 YAML 而不创建资源
- `| kubectl apply`：应用 YAML（幂等 - 多次运行安全）

**没有这个**，你会得到：
```
Error from server (AlreadyExists): secrets "acr-secret" already exists
❌ 第二次运行时任务失败
```

**使用这种方法**：
```
secret/acr-secret configured
✅ 每次都有效（如果存在则更新）
```

### 环境变量替换

这是注入动态镜像标签的魔法：

```yaml
script:
  - envsubst < calculator.yaml > calculator-deploy.yaml
  - kubectl apply -f calculator-deploy.yaml
```

**输入**（`calculator.yaml`）：
```yaml
spec:
  containers:
  - name: calculator
    image: ${DOCKER_IMAGE}  # ← 变量占位符
```

**经过 `envsubst` 后**（`calculator-deploy.yaml`）：
```yaml
spec:
  containers:
  - name: calculator
    image: private-registry.example.com/my-namespace/my-app:56cc4792-86  # ← 已替换！
```

**工作原理**：
1. `envsubst` 读取文件
2. 查找匹配 `${VAR_NAME}` 的变量
3. 用环境变量值替换
4. 输出到新文件

---

## Kubernetes 部署配置

### calculator.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calculator
  namespace: default
spec:
  replicas: 2  # 两个 pod 实现高可用

  selector:
    matchLabels:
      app: calculator

  template:
    metadata:
      labels:
        app: calculator

    spec:
      # 使用 deploy job 创建的 secret
      imagePullSecrets:
      - name: acr-secret

      containers:
      - name: calculator
        image: ${DOCKER_IMAGE}  # 将被 envsubst 替换
        imagePullPolicy: Always  # 总是拉取（标签每次流水线都会改变）

        ports:
        - name: http
          containerPort: 30194

        # 挂载主机时区（可选）
        volumeMounts:
        - name: host-time
          mountPath: /etc/localtime
          readOnly: true

      volumes:
      - name: host-time
        hostPath:
          path: /etc/localtime

---
apiVersion: v1
kind: Service
metadata:
  name: calculator
  namespace: default
spec:
  type: NodePort

  selector:
    app: calculator

  ports:
  - port: 30194        # Service 端口
    targetPort: 30194  # Container 端口
    nodePort: 30194    # Node 端口（30000-32767 范围）
```

### 关键配置点

**1. imagePullPolicy: Always**

```yaml
imagePullPolicy: Always
```

**为什么？** 因为我们的标签每次都会改变：
- Pipeline 85：`my-app:56cc4792-85`
- Pipeline 86：`my-app:56cc4792-86`

没有 `Always`，Kubernetes 可能会使用缓存的镜像。

**2. imagePullSecrets**

```yaml
imagePullSecrets:
- name: acr-secret
```

这引用了在 `deploy-job` before_script 中创建的 secret。没有它：
```
Failed to pull image: pull access denied
ErrImagePull
```

**3. NodePort Service**

```yaml
type: NodePort
nodePort: 30194
```

**允许外部访问**：
```bash
# 从任何节点 IP 访问
curl http://192.168.1.101:30194/
curl http://192.168.1.102:30194/
```

**替代方案**：如果你有云负载均衡器，使用 `LoadBalancer` 类型。

---

## 变量流经流水线

让我们追踪镜像标签如何从流水线开始流向部署：

```
┌─────────────────────────────────────────────────────────┐
│ 1. 流水线开始                                           │
│    CI_COMMIT_SHORT_SHA = "56cc4792"                     │
│    CI_PIPELINE_ID = "86"                                │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 2. 合成全局变量                                         │
│    DOCKER_IMAGE = "registry.../app:56cc4792-86"         │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 3. build-job 使用变量                                   │
│    $ docker build -t $DOCKER_IMAGE .                    │
│    $ docker push $DOCKER_IMAGE                          │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 4. deploy-job 继承变量（同一流水线）                    │
│    $ envsubst < calculator.yaml                         │
│    读取：$DOCKER_IMAGE                                  │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 5. 写入 Kubernetes YAML                                 │
│    image: registry.../app:56cc4792-86                   │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────────┐
│ 6. Kubelet 拉取镜像                                     │
│    使用 imagePullSecrets: acr-secret                    │
│    结果：Pod 运行新镜像                                 │
└─────────────────────────────────────────────────────────┘
```

---

## 完整执行示例

让我们从头到尾走过一次真实的部署。

### 步骤 1：开发者进行更改

```bash
# 修复 app.py 中的 bug
vim app.py

# 提交更改
git add app.py
git commit -m "fix: correct division by zero error"
git push origin main
```

### 步骤 2：触发流水线

GitLab UI 显示：
```
Pipeline #86 started
├── Stage: build
│   └── build-job (running)
└── Stage: deploy
    └── deploy-job (waiting for manual trigger)
```

### 步骤 3：build-job 执行

```
[00:00] 创建任务 Pod
[00:03] Helper 克隆代码
[00:05] DinD 启动，Build 容器启动
[00:10] docker build 开始
  ├── 拉取基础镜像 python:3.9-slim（35秒）
  ├── COPY app.py .（1秒）
  ├── pip install flask（15秒）
  └── 构建完成
[01:00] docker login 到仓库
[01:05] docker push
  ├── Layer 1：already exists
  ├── Layer 2：already exists
  ├── Layer 3：Pushed（新代码）
  └── 标签：56cc4792-86
[01:40] build-job 完成 ✅
```

### 步骤 4：手动部署触发

开发者在 GitLab UI 中点击 deploy-job 上的 **Run**。

### 步骤 5：deploy-job 执行

```
[00:00] 创建 Deploy Pod
[00:03] Helper 克隆代码
[00:05] kubectl 容器启动
[00:08] 验证 kubeconfig
[00:10] kubectl cluster-info（已连接）
[00:12] 创建 acr-secret
[00:15] envsubst 替换镜像变量
  输入：image: ${DOCKER_IMAGE}
  输出：image: registry.../app:56cc4792-86
[00:18] kubectl apply -f calculator-deploy.yaml
  ├── Deployment "calculator" configured
  └── Service "calculator" unchanged
[00:20] deploy-job 完成 ✅
```

### 步骤 6：Kubernetes 滚动更新

```
[00:20] Deployment 检测到镜像变化
[00:22] 创建新 ReplicaSet
[00:25] 启动新 Pod（calculator-767f559f9c-xxxxx）
  ├── 拉取镜像（使用 acr-secret）
  ├── 镜像下载完成（3秒）
  └── 容器启动
[00:30] 新 Pod 就绪
[00:32] 删除旧 Pod
[00:35] 滚动更新完成 ✅
```

### 步骤 7：验证

```bash
# 检查 Pod 状态
kubectl get pods -n default -l app=calculator

NAME                          READY   STATUS    RESTARTS   AGE
calculator-767f559f9c-2j7vz   1/1     Running   0          5m
calculator-767f559f9c-5qnx4   1/1     Running   0          5m

# 测试应用程序
curl http://192.168.1.101:30194/
{"message": "Calculator API", "version": "1.0"}

# 测试修复
curl -X POST http://192.168.1.101:30194/add \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 5}'

{"result": 15}
```

---

## 常见问题和解决方案

### 问题 1：构建失败 - "Cannot connect to Docker daemon"

**症状**：
```
Cannot connect to the Docker daemon at tcp://docker:2375
```

**原因**：DinD 尚未就绪

**解决方案**：在 `before_script` 中添加适当的等待逻辑（如上所示）

### 问题 2：推送失败 - "unauthorized: authentication required"

**症状**：
```
Error response from daemon: Get "https://registry.../": unauthorized
```

**原因**：未登录到仓库

**解决方案**：
```yaml
script:
  - echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
  - docker push $DOCKER_IMAGE
```

确保 `ACR_USERNAME` 和 `ACR_PASSWORD` 在 GitLab CI/CD 变量中设置。

### 问题 3：部署失败 - "ErrImagePull"

**症状**：
```
Failed to pull image: pull access denied
```

**原因和解决方案**：

**1. 缺少 imagePullSecrets**：
```yaml
# 添加到 Pod spec：
imagePullSecrets:
- name: acr-secret
```

**2. Secret 未创建**：
```bash
# 验证 secret 存在
kubectl get secret acr-secret -n default

# 如果不存在，检查 deploy-job 日志
```

**3. 错误的凭据**：
```bash
# 手动测试
echo "$ACR_PASSWORD" | docker login -u $ACR_USERNAME --password-stdin registry.com
```

### 问题 4：envsubst 不替换变量

**症状**：
```yaml
# envsubst 后，仍显示：
image: ${DOCKER_IMAGE}
```

**原因**：变量未导出或语法错误

**解决方案**：
```yaml
# 确保变量在全局或 job 级别定义：
variables:
  DOCKER_IMAGE: "registry.../app:${CI_COMMIT_SHORT_SHA}"

# 在 YAML 中使用 ${VAR} 而不是 $VAR：
image: ${DOCKER_IMAGE}  # ✅ 正确
image: $DOCKER_IMAGE    # ❌ envsubst 不起作用
```

---

## 最佳实践

### 1. 镜像标签策略

**✅ 好 - 可追溯性**：
```yaml
DOCKER_IMAGE: "registry.../app:${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"
```

**❌ 避免 - 无历史**：
```yaml
DOCKER_IMAGE: "registry.../app:latest"
```

### 2. 手动部署触发

**✅ 好 - 防止意外**：
```yaml
deploy-job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

**❌ 危险 - 自动部署**：
```yaml
deploy-job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
```

### 3. 资源限制

**✅ 好 - 防止 OOM**：
```yaml
# 在 gitlab-runner-values.yaml 中
cpu_limit = "2"
memory_limit = "2Gi"
```

**❌ 避免 - 无限资源**：
```yaml
# 未设置限制 = 可能耗尽集群资源
```

### 4. 清理敏感数据

**✅ 好 - 安全**：
```yaml
after_script:
  - rm -f $KUBECONFIG
  - rm -f calculator-deploy.yaml
```

**❌ 危险 - 留下凭据**：
```yaml
# 无清理 = kubeconfig 保留在 Pod 中（直到删除）
```

---

## 关键要点

**流水线结构**：
- 两个阶段：`build`（自动）和 `deploy`（手动）
- Build 阶段创建并推送 Docker 镜像
- Deploy 阶段将更改应用到 Kubernetes

**Docker-in-Docker**：
- 始终等待守护进程就绪
- 对私有仓库使用适当的认证
- 使用可追溯的标识符标记镜像

**Kubernetes 部署**：
- 使用 `envsubst` 进行动态变量注入
- 为频繁更改的标签设置 `imagePullPolicy: Always`
- 在部署前创建 imagePullSecrets

**变量流**：
- GitLab → 流水线变量 → Build → Registry
- 流水线变量 → Deploy → envsubst → Kubernetes

---

## 接下来是什么？

在第三部分中，我们构建了一个完整的 CI/CD 工作流。但当事情出错时会发生什么？特别是，**网络和 DNS 问题**是 Kubernetes 部署中最常见的问题。

**第四部分即将推出**：
- 🌐 **Kubernetes 网络深入探讨** — 解释5层网络
- 🔍 **DNS 解析故障排除** — 为什么 DinD 无法拉取镜像
- 📡 **数据包流分析** — 跟踪从 Pod 到互联网的流量
- 🛠️ **真实调试会话** — 解决实际的 DNS 故障

**预览**：我们将走过一次真实的故障排除会话，其中 DinD 由于 DNS 配置错误无法拉取镜像，展示我们如何诊断和修复它。

---

## 实践练习

**练习 1**：实现多阶段部署

在 `build` 和 `deploy` 之间添加 `test` 阶段：
```yaml
stages:
  - build
  - test
  - deploy

test-job:
  stage: test
  image: python:3.9-slim
  script:
    - pip install pytest
    - pytest tests/
```

**练习 2**：向部署添加健康检查

修改 `calculator.yaml`：
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 30194
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 30194
  initialDelaySeconds: 5
  periodSeconds: 5
```

**练习 3**：监控部署进度

```bash
# 实时监视滚动更新
kubectl rollout status deployment/calculator -n default

# 查看rollout历史
kubectl rollout history deployment/calculator -n default

# 如果需要回滚
kubectl rollout undo deployment/calculator -n default
```

---

## 资源

- [GitLab CI/CD YAML 参考](https://docs.gitlab.com/ee/ci/yaml/)
- [Docker 构建最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes 部署策略](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**第四部分即将推出** — 我们将处理最具挑战性的方面：网络和 DNS 配置。

如果你觉得这有帮助，欢迎与其他学习 GitLab CI/CD 的人分享。

---

*这是关于 Kubernetes 上的 GitLab Runner 的5部分系列的第3部分，记录了学习和实施过程。*

**系列索引**：
- [第1部分：架构与快速搭建](link)
- [第2部分：深入探讨容器架构](link)
- **第3部分：构建真实世界的 CI/CD 流水线** ← 你在这里
- 第4部分：解决 DNS 和网络问题（即将推出）
- 第5部分：生产最佳实践（即将推出）

---

*标签：#GitLab #Kubernetes #CICD #DevOps #Docker #容器仓库*
