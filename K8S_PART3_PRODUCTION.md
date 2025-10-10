# Kubernetes快速上手教程 - 第三篇：生产环境与最佳实践

## 序言

欢迎来到Kubernetes快速上手教程的第三篇！这是我们教程的最后一篇，将带您从实验室环境走向真正的生产部署。

在前面两篇中，您已经：
- 掌握了Kubernetes的核心概念和架构
- 学会了配置文件编写和应用部署

现在，我们将专注于：
- **生产环境的考虑因素** - 安全性、可靠性、可扩展性
- **运维最佳实践** - 监控、备份、故障处理
- **性能优化策略** - 资源管理、调度优化

这篇内容将帮助您建立生产级Kubernetes集群的运维能力，确保您的应用在真实环境中稳定可靠地运行。

**学习前提**：
- 完成前两篇的学习
- 有一定的系统运维经验
- 了解基本的网络和安全概念

**适合场景**：
- 准备在生产环境部署Kubernetes的团队
- 负责Kubernetes集群运维的工程师
- 需要制定最佳实践规范的技术管理者

**预计阅读时间**：2-3小时

---

## 目录

6. [生产环境最佳实践](#6-生产环境最佳实践)
7. [常用Kubernetes命令](#7-常用kubernetes命令)

---

## 6. 生产环境最佳实践

### 6.1 安全性最佳实践

生产环境的Kubernetes安全需要充分利用其**原生安全机制**，包括RBAC授权、网络策略、Pod安全标准、准入控制器等核心特性。

#### **Kubernetes安全控制平面**

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes安全控制流程                      │
│                                                         │
│ kubectl → API Server → 认证 → 授权 → 准入控制 → etcd    │
│    │          │        │      │       │         │      │
│  客户端     入口网关    认证    RBAC   AdmissionC  存储   │
│  证书      TLS终结    Token   权限     OPA策略   加密    │
│  kubeconfig 负载均衡  JWT/OIDC ClusterRole PodSecurity etcd │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes RBAC权限体系**

**RBAC四元素关系**：
- **User/ServiceAccount**：身份主体
- **Role/ClusterRole**：权限定义
- **RoleBinding/ClusterRoleBinding**：绑定关系
- **Resources + Verbs**：资源和操作

**生产级RBAC设计模式**：

| 角色类型 | 权限范围 | 典型资源 | 关键verbs | 使用场景 |
|----------|----------|----------|-----------|----------|
| **cluster-admin** | 集群级 | 所有资源 | * | 平台管理员 |
| **namespace-admin** | 命名空间级 | 命名空间内所有 | create,delete,get,list,update | 项目负责人 |
| **developer** | 命名空间级 | pods,services,deployments | get,list,watch,create,update | 开发人员 |
| **readonly** | 命名空间级 | 所有资源 | get,list,watch | 只读访问 |
| **service-account** | 应用级 | 特定资源 | 最小权限 | 应用Pod |

**ServiceAccount最佳实践配置**：
```yaml
# 应用专用ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
automountServiceAccountToken: false  # 禁用自动挂载
---
# 最小权限Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

#### **2. Kubernetes网络策略实现**

**NetworkPolicy工作原理**：
- 基于Pod标签选择器实现微分段
- 支持Ingress(入站)和Egress(出站)流量控制
- 通过CNI插件(如Calico、Cilium)执行策略

**分层网络安全策略**：
```yaml
# 1. 全局默认拒绝策略
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# 2. 允许前端访问后端
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# 3. 允许访问DNS服务
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

#### **3. Pod安全标准(PSS)实施**

**Kubernetes Pod安全三级标准**：
- **Privileged**：不限制，允许已知的特权升级
- **Baseline**：防止已知的特权升级
- **Restricted**：严格限制，遵循Pod加固最佳实践

**Pod安全策略配置**：
```yaml
# 命名空间级Pod安全标准
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Restricted级别Pod配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "200m"
```

#### **4. Kubernetes准入控制器**

**关键准入控制器配置**：
```yaml
# OPA Gatekeeper策略示例
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }
---
# 应用策略约束
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-env-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["environment", "team"]
```

#### **5. Secret加密和管理**

**etcd静态加密配置**：
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <32字节base64编码密钥>
  - identity: {}
```

**Secret使用最佳实践**：
```yaml
# 1. 限制Secret访问权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["mysecret"]  # 限制特定Secret
  verbs: ["get"]
---
# 2. 使用projected volumes
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: password
            path: db-password
            mode: 0400  # 只读权限
```

**生产环境安全检查清单**：
- ☑️ 启用API Server审计日志
- ☑️ 配置etcd静态加密
- ☑️ 实施Pod Security Standards
- ☑️ 部署OPA Gatekeeper策略
- ☑️ 配置全覆盖网络策略
- ☑️ 使用最小权限RBAC
- ☑️ 定期轮转ServiceAccount token
- ☑️ 集成镜像安全扫描

### 6.2 高可用性配置

生产环境需要充分利用Kubernetes的**原生高可用机制**，包括副本控制、调度策略、健康检查、自动扩缩容等特性来构建弹性应用。

#### **Kubernetes高可用控制器**

```
┌─────────────────────────────────────────────────────────┐
│               Kubernetes高可用机制                       │
│                                                         │
│ Deployment → ReplicaSet → Pods → kubelet → 容器运行时    │
│      │           │         │        │         │        │
│   副本管理      Pod管理    调度器    节点代理    健康检查   │
│   滚动更新      标签选择   亲和性    探针检查    自动重启   │
│   HPA扩缩容     故障转移   拓扑分散   资源监控    日志收集   │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes调度与亲和性**

**Pod调度器工作原理**：
- **调度队列**：待调度Pod排队等待
- **过滤阶段**：筛选可调度节点
- **打分阶段**：为候选节点评分
- **绑定阶段**：将Pod分配到最优节点

**高可用调度策略配置**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 3
  template:
    spec:
      # 1. Pod反亲和性 - 分散到不同节点
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["critical-app"]
            topologyKey: kubernetes.io/hostname
        # 2. 节点亲和性 - 优选高性能节点
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values: ["high-performance"]
      # 3. 拓扑分布约束 - 跨可用区均匀分布
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: critical-app
```

#### **2. Kubernetes健康检查机制**

**三种探针类型对比**：

| 探针类型 | 触发时机 | 失败行为 | 适用场景 |
|----------|----------|----------|----------|
| **startupProbe** | 容器启动期 | 杀死容器重启 | 慢启动应用 |
| **livenessProbe** | 容器运行期 | 杀死容器重启 | 检测死锁/内存泄漏 |
| **readinessProbe** | 容器运行期 | 移出Service端点 | 检测应用就绪状态 |

**生产级健康检查配置**：
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        # 启动探针 - 给慢启动应用更多时间
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30  # 允许150秒启动时间
        # 存活探针 - 检测应用是否健康运行
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        # 就绪探针 - 检测应用是否准备好接收流量
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
```

#### **3. Kubernetes资源管理**

**QoS等级与调度优先级**：
```yaml
# Guaranteed QoS - 最高优先级
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"  # 与requests相等
        cpu: "500m"      # 与requests相等
---
# Burstable QoS - 中等优先级
apiVersion: v1
kind: Pod  
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "256Mi"
        cpu: "200m"
      limits:
        memory: "512Mi"  # 大于requests
        cpu: "800m"      # 大于requests
---
# 资源配额控制
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200" 
    limits.memory: 400Gi
    pods: "50"
    persistentvolumeclaims: "20"
```

#### **4. Kubernetes自动扩缩容**

**HPA控制器工作原理**：
- **指标收集**：metrics-server收集Pod资源使用情况
- **计算算法**：期望副本数 = ceil[当前副本数 * (当前指标值 / 期望指标值)]
- **平滑机制**：避免频繁扩缩容的震荡

**生产级HPA配置**：
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 100
  metrics:
  # CPU指标
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # 内存指标
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # 自定义指标 - QPS
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  # 扩缩容行为控制
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      selectPolicy: Min
```

#### **5. Kubernetes存储高可用**

**StatefulSet有状态应用部署**：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-cluster
spec:
  serviceName: mysql
  replicas: 3
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  # 每个Pod独立的PVC
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
---
# Headless Service用于稳定网络标识
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless Service
  selector:
    app: mysql-cluster
  ports:
  - port: 3306
```

**Kubernetes高可用检查清单**：
- ☑️ 配置Pod反亲和性和拓扑分布
- ☑️ 实施多层健康检查(startup/liveness/readiness)
- ☑️ 设置合理的资源requests和limits
- ☑️ 启用HPA自动水平扩缩容
- ☑️ 对有状态应用使用StatefulSet
- ☑️ 配置PodDisruptionBudget防止过度驱逐
- ☑️ 使用多副本和跨AZ部署
- ☑️ 监控Pod重启和调度失败事件

### 6.3 备份和恢复策略

生产环境需要重点关注**Kubernetes集群级别的备份恢复**，包括etcd集群状态、Kubernetes资源对象、持久化存储等核心组件的数据保护。

#### **Kubernetes备份恢复架构**

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes集群备份体系                      │
│                                                         │
│ etcd集群 → 资源对象 → 持久化卷 → 配置数据 → 应用状态     │
│     │         │         │         │         │          │
│  集群状态   API对象   存储数据   配置管理   业务数据      │
│  快照备份   YAML导出  卷快照     Secret     数据库备份    │
│  多副本     版本控制  跨AZ复制   加密存储   流式复制      │
└─────────────────────────────────────────────────────────┘
```

#### **1. etcd集群备份策略**

**etcd高可用架构备份**：
```bash
# 1. 创建etcd快照
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S).db

# 2. 验证快照状态
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /backup/etcd-snapshot-*.db

# 3. 恢复etcd集群
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name m1 \
  --data-dir=/var/lib/etcd-from-backup \
  --initial-cluster=m1=https://host1:2380,m2=https://host2:2380,m3=https://host3:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://host1:2380
```

**自动化etcd备份CronJob**：
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # 每6小时备份一次
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: etcd-backup
          hostNetwork: true
          containers:
          - name: etcd-backup
            image: quay.io/coreos/etcd:v3.5.0
            command:
            - /bin/sh
            - -c
            - |
              ETCDCTL_API=3 etcdctl \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
                --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
                snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
              # 上传到对象存储
              aws s3 cp /backup/etcd-$(date +%Y%m%d-%H%M%S).db s3://k8s-backup-bucket/etcd/
            volumeMounts:
            - mountPath: /etc/kubernetes/pki/etcd
              name: etcd-certs
              readOnly: true
            - mountPath: /backup
              name: backup-storage
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

#### **2. Kubernetes资源对象备份**

**使用Velero进行集群级备份**：
```yaml
# 1. 安装Velero备份工具
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
# 2. 创建备份存储位置
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: aws-backup-location
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: kubernetes-backups
    region: us-west-2
  config:
    region: us-west-2
---
# 3. 创建备份计划
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 每天凌晨2点
  template:
    includedNamespaces:
    - production
    - staging
    excludedResources:
    - events
    - events.events.k8s.io
    storageLocation: aws-backup-location
    volumeSnapshotLocations:
    - aws-volume-snapshot-location
    ttl: 720h  # 保留30天
```

**手动备份Kubernetes资源**：
```bash
# 1. 备份所有命名空间的资源
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# 2. 备份特定资源类型
kubectl get deployments,services,configmaps,secrets -n production -o yaml > production-backup.yaml

# 3. 备份自定义资源(CRD)
kubectl get crd -o yaml > crd-backup.yaml
kubectl get <custom-resource> --all-namespaces -o yaml > custom-resources-backup.yaml

# 4. 使用kubectl备份脚本
#!/bin/bash
BACKUP_DIR="/backup/k8s-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# 备份核心资源
for resource in deployments services configmaps secrets persistentvolumeclaims; do
  kubectl get $resource --all-namespaces -o yaml > $BACKUP_DIR/$resource.yaml
done

# 备份集群级资源
for resource in nodes persistentvolumes storageclasses; do
  kubectl get $resource -o yaml > $BACKUP_DIR/$resource.yaml
done
```

#### **3. 持久化存储备份**

**使用CSI快照进行卷备份**：
```yaml
# 1. 定义卷快照类
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
# 2. 创建卷快照
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-data-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: mysql-data-pvc
---
# 3. 从快照恢复PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
spec:
  storageClassName: gp2
  dataSource:
    name: mysql-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

**自动化卷快照备份**：
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: volume-snapshot-backup
spec:
  schedule: "0 1 * * *"  # 每天凌晨1点
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-controller
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # 获取所有需要备份的PVC
              for pvc in $(kubectl get pvc -l backup=enabled -o name); do
                pvc_name=$(echo $pvc | cut -d'/' -f2)
                snapshot_name="$pvc_name-$(date +%Y%m%d)"
                
                # 创建快照
                cat <<EOF | kubectl apply -f -
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: $snapshot_name
              spec:
                volumeSnapshotClassName: csi-aws-vsc
                source:
                  persistentVolumeClaimName: $pvc_name
              EOF
              done
          restartPolicy: OnFailure
```

#### **4. 跨集群灾难恢复**

**多集群备份策略**：
```yaml
# 1. 跨集群资源同步
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: disaster-recovery-sync
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-configs
    targetRevision: HEAD
    path: disaster-recovery
  destination:
    server: https://backup-cluster-api-server
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# 2. 跨区域PV复制
apiVersion: v1
kind: ConfigMap
metadata:
  name: cross-region-backup-config
data:
  script: |
    #!/bin/bash
    # 跨区域卷数据同步
    SOURCE_CLUSTER="arn:aws:eks:us-west-2:account:cluster/prod"
    TARGET_CLUSTER="arn:aws:eks:us-east-1:account:cluster/dr"
    
    # 同步PV快照到灾备区域
    aws ec2 copy-snapshot \
      --source-region us-west-2 \
      --source-snapshot-id $SNAPSHOT_ID \
      --destination-region us-east-1 \
      --description "DR backup from prod cluster"
```

#### **5. 备份验证与恢复测试**

**自动化恢复测试**：
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-validation
spec:
  schedule: "0 4 * * 0"  # 每周日凌晨4点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup-validator
            image: backup-validator:v1.0
            command:
            - /bin/sh
            - -c
            - |
              # 1. 验证etcd快照完整性
              etcdctl snapshot status /backup/latest-etcd-snapshot.db
              
              # 2. 验证Velero备份状态
              velero backup get --output json | jq '.items[] | select(.status.phase != "Completed")'
              
              # 3. 验证卷快照状态
              kubectl get volumesnapshots -o json | jq '.items[] | select(.status.readyToUse != true)'
              
              # 4. 发送验证结果通知
              curl -X POST $SLACK_WEBHOOK_URL \
                -H 'Content-type: application/json' \
                --data '{"text":"Backup validation completed at $(date)"}'
          restartPolicy: OnFailure
```

**Kubernetes备份检查清单**：
- ☑️ 配置etcd自动备份和加密存储
- ☑️ 部署Velero进行应用级备份
- ☑️ 启用CSI卷快照功能
- ☑️ 实施跨AZ/跨区域备份复制
- ☑️ 定期执行恢复测试验证
- ☑️ 监控备份任务执行状态
- ☑️ 建立备份数据保留策略
- ☑️ 配置备份失败告警机制

### 6.4 性能优化

生产环境性能优化需要深入理解**Kubernetes调度器**、**容器运行时**、**网络模型**等核心组件的工作原理，通过精细化配置实现最优性能。

#### **Kubernetes性能调优体系**

```
┌─────────────────────────────────────────────────────────┐
│               Kubernetes性能调优链路                     │
│                                                         │
│ kube-scheduler → kubelet → 容器运行时 → CNI网络 → CSI存储 │
│       │            │         │           │        │     │
│   调度算法       节点代理    容器隔离     网络插件   存储驱动 │
│   亲和性规则     资源管理    CGroup     iptables   块设备   │
│   优先级队列     监控指标    命名空间   负载均衡   文件系统  │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes调度器性能优化**

**调度器配置优化**：
```yaml
# kube-scheduler配置文件
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: NodeAffinity  
        weight: 2
      - name: PodTopologySpread
        weight: 2
    filter:
      enabled:
      - name: NodeResourcesFit
      - name: NodeAffinity
      - name: PodTopologySpread
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated  # 优化资源利用率
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

**高性能调度策略**：
```yaml
# 1. CPU拓扑感知调度
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # CPU Manager策略 - 为关键应用预留专用CPU
      containers:
      - name: cpu-intensive-app
        resources:
          requests:
            cpu: "4"          # 整数CPU确保独占
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
      nodeSelector:
        node.kubernetes.io/instance-type: "c5.2xlarge"
---
# 2. NUMA节点亲和性
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: numa-aware-app
    resources:
      requests:
        cpu: "8"
        memory: "16Gi"
      limits:
        cpu: "8" 
        memory: "16Gi"
  # 使用拓扑管理器策略
  runtimeClassName: numa-aware
```

#### **2. 容器运行时性能调优**

**containerd运行时优化**：
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  # 启用CPU Manager
  [plugins."io.containerd.grpc.v1.cri".containerd]
    default_runtime_name = "runc"
    
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
      
  # 性能优化配置
  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://mirror.aliyuncs.com"]
```

**kubelet性能参数优化**：
```yaml
# kubelet配置文件
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# CPU管理策略
cpuManagerPolicy: "static"
cpuManagerReconcilePeriod: "10s"
# 内存管理策略  
memoryManagerPolicy: "Static"
reservedMemory:
- numaNode: 0
  limits:
    memory: "1Gi"
# 拓扑管理器策略
topologyManagerPolicy: "single-numa-node"
# 性能相关参数
maxPods: 110
podPidsLimit: 4096
systemReserved:
  cpu: "500m"
  memory: "1Gi"
kubeReserved:
  cpu: "500m" 
  memory: "1Gi"
```

#### **3. Kubernetes网络性能优化**

**CNI网络插件性能对比**：

| CNI插件 | 延迟 | 带宽 | CPU开销 | 特性支持 |
|---------|------|------|---------|----------|
| **Flannel** | 低 | 高 | 低 | 简单易用 |
| **Calico** | 中 | 高 | 中 | 网络策略/BGP |
| **Cilium** | 低 | 极高 | 低 | eBPF/L7策略 |
| **Weave** | 高 | 中 | 高 | 加密通信 |

**Cilium eBPF网络加速配置**：
```yaml
# Cilium ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  # eBPF数据路径加速
  datapath-mode: "veth"
  enable-bpf-masquerade: "true"
  enable-host-routing: "true"
  
  # 性能优化选项
  kube-proxy-replacement: "partial"
  enable-bandwidth-manager: "true"
  enable-local-redirect-policy: "true"
  
  # XDP负载均衡
  enable-node-port: "true"
  node-port-mode: "hybrid"
  loadbalancer-mode: "dsr"
```

**Service性能优化配置**：
```yaml
# 高性能Service配置
apiVersion: v1
kind: Service
metadata:
  name: high-perf-service
  annotations:
    # Cilium特有的负载均衡优化
    service.cilium.io/lb-mode: "dsr"
    service.cilium.io/global: "true"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # 避免二次转发
  internalTrafficPolicy: Local    # 本地流量优化
  sessionAffinity: ClientIP       # 会话保持
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

#### **4. Kubernetes存储性能优化**

**CSI存储驱动性能配置**：
```yaml
# 高性能StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-nvme-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "16000"        # 最大IOPS
  throughput: "1000"   # 最大吞吐量MiB/s
  fsType: ext4
mountOptions:
- noatime              # 减少访问时间更新
- nodiratime          # 减少目录访问时间更新
- barrier=0           # 禁用写屏障(谨慎使用)
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# 本地SSD存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner
parameters:
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
```

**高IOPS应用Pod配置**：
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: database
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
    volumeMounts:
    - name: data-volume
      mountPath: /var/lib/mysql
      # 挂载选项优化
      mountPropagation: None
    - name: logs-volume  
      mountPath: /var/log/mysql
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: mysql-data-pvc
  - name: logs-volume
    emptyDir:
      medium: Memory     # 使用内存作为日志存储
      sizeLimit: 1Gi
```

#### **5. 应用级性能监控与调优**

**Kubernetes性能指标收集**：
```yaml
# VPA垂直扩缩容推荐
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: webapp
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      # 资源推荐策略
      mode: Auto
---
# 性能监控ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubernetes-performance
spec:
  selector:
    matchLabels:
      app: kubernetes-state-metrics
  endpoints:
  - port: http-metrics
    interval: 15s
    path: /metrics
    # 关键性能指标
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'kube_pod_container_resource_(requests|limits)_(cpu_cores|memory_bytes)'
      action: keep
```

**Kubernetes性能调优检查清单**：
- ☑️ 启用CPU Manager和Memory Manager
- ☑️ 配置NUMA拓扑感知调度
- ☑️ 优化容器运行时参数
- ☑️ 选择高性能CNI插件(Cilium)
- ☑️ 配置本地流量策略
- ☑️ 使用高性能存储类
- ☑️ 启用VPA资源右调
- ☑️ 监控关键性能指标

---

## 生产环境最佳实践框架总结

### 🎯 生产级Kubernetes部署决策体系

通过第6部分四个核心领域的学习，我们构建了完整的生产环境运维体系。以下是系统化的决策框架：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          生产级Kubernetes运维框架                                 │
│                                                                                 │
│  安全防护体系 → 高可用设计 → 备份恢复策略 → 性能优化调优 → 运维监控体系            │
│       │            │            │            │            │                    │
│   多层防护       容灾设计       数据保护       全链路优化       全栈监控           │
│   权限最小化     故障自愈       灾难恢复       资源调优       智能告警             │
│   网络隔离       弹性扩缩       业务连续性     架构优化       运维自动化           │
│   合规审计       服务治理       RTO/RPO目标    持续调优       问题诊断             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### **生产环境成熟度评估模型**

**安全成熟度等级**：

| 等级 | 特征描述 | 核心能力 | 适用规模 |
|------|----------|----------|----------|
| **L1-基础** | 基本RBAC + 网络策略 | 访问控制、网络隔离 | 开发测试环境 |
| **L2-标准** | Pod安全标准 + Secret管理 | 容器安全、密钥保护 | 预生产环境 |
| **L3-增强** | 多层防护 + 审计日志 | 深度防御、合规管理 | 生产环境 |
| **L4-专家** | 零信任架构 + 威胁检测 | 自适应安全、智能防护 | 关键业务系统 |

**可用性成熟度等级**：

| 等级 | SLA目标 | 架构特征 | 容灾能力 |
|------|---------|----------|----------|
| **L1-基础** | 99.5% | 单副本部署 | 基本重启 |
| **L2-标准** | 99.9% | 多副本 + 反亲和 | 节点级容灾 |
| **L3-增强** | 99.95% | 多AZ部署 + 自动扩缩容 | 区域级容灾 |
| **L4-专家** | 99.99% | 多区域 + 服务网格 | 地理级容灾 |


### 🚀 云原生运维最佳实践

#### **DevOps集成最佳实践**

**CI/CD安全集成**：
- **镜像安全**：集成漏洞扫描到构建流水线
- **配置验证**：使用OPA Gatekeeper策略验证
- **渐进发布**：金丝雀发布 + 自动回滚机制
- **环境一致性**：基础设施即代码(IaC)

**GitOps运维模式**：
- **配置版本化**：所有配置存储在Git仓库
- **声明式管理**：使用ArgoCD等工具自动同步
- **变更审计**：所有变更都有完整的审计链路
- **环境隔离**：开发、测试、生产环境独立管理

#### **SRE文化与实践**

**可靠性工程原则**：
- **错误预算**：建立可量化的可靠性目标
- **监控驱动**：基于SLI/SLO/SLA的监控体系
- **故障学习**：无责文化的事后复盘机制
- **自动化优先**：减少手工操作，提高效率

**运维效率提升策略**：
- **平台工程**：构建内部开发者平台
- **自助服务**：开发团队自主运维能力
- **知识沉淀**：建立运维知识库和标准操作程序
- **持续改进**：基于度量数据的持续优化

#### **成本优化与资源管理**

**资源成本优化框架**：

| 优化维度 | 策略方法 | 预期收益 | 实施难度 |
|----------|----------|----------|----------|
| **资源配置优化** | 基于监控数据调整requests/limits | 20-30% | 低 |
| **调度策略优化** | 节点亲和性 + 污点容忍 | 10-15% | 中 |
| **存储成本优化** | 分层存储 + 生命周期管理 | 30-40% | 中 |
| **网络成本优化** | 流量路由优化 + CDN集成 | 15-25% | 高 |

**FinOps实践**：
- **成本可视化**：实时成本监控和分配
- **预算控制**：基于ResourceQuota的预算管理
- **使用优化**：右规模调整和弹性调度
- **采购优化**：竞价实例和长期合约优化

---

## 7. 常用Kubernetes命令

在日常的Kubernetes运维工作中，熟练掌握kubectl命令是必不可少的技能。本章将介绍最常用且实用的kubectl命令，帮助您提高运维效率。

### 7.1 资源查看与信息获取

#### **集群信息查看**

```bash
# 查看集群信息
kubectl cluster-info
# 作用：显示Kubernetes集群的基本信息，包括API服务器地址和DNS服务地址
# 输出示例：
# Kubernetes control plane is running at https://kubernetes.docker.internal:6443
# CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# 查看集群节点信息
kubectl get nodes
# 作用：列出集群中所有节点及其状态
# 参数说明：
# -o wide：显示更详细信息（IP地址、操作系统等）
# --show-labels：显示节点标签

kubectl get nodes -o wide
# 输出示例：
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
# minikube   Ready    control-plane   5d    v1.24.3   192.168.49.2   <none>        Ubuntu 20.04.4 LTS   5.4.0-122-generic   docker://20.10.17

# 查看节点详细信息
kubectl describe node <节点名称>
# 作用：显示指定节点的详细信息，包括资源使用情况、系统信息、Pod分布等
# 示例：kubectl describe node minikube
```

#### **命名空间管理**

```bash
# 查看所有命名空间
kubectl get namespaces
# 简写：kubectl get ns
# 作用：列出集群中所有命名空间

# 查看当前使用的命名空间
kubectl config view --minify --output 'jsonpath={..namespace}'
# 作用：显示当前kubectl上下文使用的默认命名空间

# 切换命名空间
kubectl config set-context --current --namespace=<命名空间名称>
# 作用：设置当前上下文的默认命名空间
# 示例：kubectl config set-context --current --namespace=production

# 创建命名空间
kubectl create namespace <命名空间名称>
# 作用：创建新的命名空间
# 示例：kubectl create namespace development
```

#### **Pod资源查看**

```bash
# 查看Pod列表
kubectl get pods
# 作用：列出当前命名空间中的所有Pod
# 常用参数：
# -n <namespace>：指定命名空间
# --all-namespaces：查看所有命名空间（简写：-A）
# -o wide：显示详细信息
# --watch：实时监控变化（简写：-w）

kubectl get pods -n kube-system
# 示例：查看系统命名空间的Pod

kubectl get pods --all-namespaces -o wide
# 示例：查看所有命名空间Pod的详细信息

# 查看Pod详细信息
kubectl describe pod <Pod名称>
# 作用：显示Pod的详细信息，包括事件、容器状态、资源使用等
# 示例：kubectl describe pod nginx-deployment-66b6c48dd5-5x8ql

# 查看Pod状态变化（实时监控）
kubectl get pods --watch
# 作用：实时监控Pod状态变化，常用于调试和部署监控
```

### 7.2 应用部署与管理

#### **配置文件管理最佳实践**

**配置文件存放位置**：
```bash
# 典型的项目结构
my-app/
├── k8s/                     # Kubernetes配置目录
│   ├── base/               # 基础配置
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── overlays/           # 环境特定配置
│   │   ├── dev/           # 开发环境
│   │   │   └── kustomization.yaml
│   │   ├── staging/       # 预发布环境
│   │   │   └── kustomization.yaml
│   │   └── prod/          # 生产环境
│   │       └── kustomization.yaml
│   └── README.md
├── src/                    # 应用源代码
└── Dockerfile             # 容器镜像定义
```

**配置文件来源和创建时机**：

1. **手动编写**（开发阶段）
```bash
# 创建配置文件模板
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
# 然后手动编辑deployment.yaml添加更多配置
```

2. **从现有资源导出**（迁移/备份）
```bash
# 导出运行中的资源配置
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
# 清理状态字段后保存为模板
```

3. **从官方/社区模板**（使用第三方应用）
```bash
# 下载官方配置文件
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
# 根据需要修改后保存到项目中
```

**版本控制**：
```bash
# 配置文件应该与代码一起进行版本控制
git add k8s/
git commit -m "Add Kubernetes deployment configurations"
git push
```

**典型工作流程**：
```bash
# 1. 开发阶段 - 创建初始配置文件
kubectl create deployment myapp --image=myapp:v1 --dry-run=client -o yaml > k8s/deployment.yaml
kubectl create service clusterip myapp --tcp=80:8080 --dry-run=client -o yaml > k8s/service.yaml

# 2. 编辑配置文件 - 添加资源限制、健康检查等
vim k8s/deployment.yaml

# 3. 本地测试 - 在开发集群应用
kubectl apply -f k8s/

# 4. 提交到代码仓库
git add k8s/
git commit -m "feat: add k8s deployment configs"
git push

# 5. CI/CD - 自动部署到不同环境
# 在CI/CD管道中：
kubectl apply -f k8s/ -n development  # 开发环境
kubectl apply -f k8s/ -n production   # 生产环境
```

**配置文件存储位置总结**：
- **本地开发**：项目根目录的 k8s/ 或 deploy/ 文件夹
- **版本控制**：Git仓库，与应用代码一起管理
- **CI/CD**：从Git仓库检出，在流水线中使用
- **配置中心**：大型组织可能使用Helm Chart仓库或GitOps工具（如ArgoCD）

#### **通过配置文件部署（推荐）**

```bash
# 应用配置文件
kubectl apply -f <配置文件>
# 作用：声明式管理资源，创建或更新资源
# 优势：可重复执行、版本控制、易于管理
# 示例：kubectl apply -f deployment.yaml

# 应用多个配置文件
kubectl apply -f <文件1> -f <文件2>
# 示例：kubectl apply -f deployment.yaml -f service.yaml

# 应用目录下所有配置文件
kubectl apply -f <目录路径>/
# 作用：递归应用目录下所有YAML文件
# 示例：kubectl apply -f ./k8s-configs/

# 应用远程配置文件
kubectl apply -f <URL>
# 作用：直接从URL应用配置
# 示例：kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/cloud/deploy.yaml

# 查看将要应用的变更（dry-run）
kubectl apply -f deployment.yaml --dry-run=client
# 作用：预览变更但不实际应用
# 参数说明：
# --dry-run=client：客户端模拟
# --dry-run=server：服务端模拟

# 查看配置文件与集群当前状态的差异
kubectl diff -f deployment.yaml
# 作用：显示配置文件与集群中资源的差异
# 示例：修改deployment.yaml后，查看具体改动

# 记录部署命令（用于回滚）
kubectl apply -f deployment.yaml --record
# 作用：记录命令到资源注释中，便于追踪变更历史
```

#### **Deployment管理**

```bash
# 命令行创建Deployment（快速测试用）
kubectl create deployment <部署名称> --image=<镜像名称>
# 作用：快速创建一个Deployment
# 参数说明：
# --image：指定容器镜像
# --replicas：指定副本数量
# --port：指定容器端口

kubectl create deployment nginx-app --image=nginx:latest --replicas=3 --port=80
# 示例：创建一个名为nginx-app的Deployment，3个副本

# 查看Deployment状态
kubectl get deployments
# 简写：kubectl get deploy
# 作用：列出所有Deployment及其状态

kubectl describe deployment <部署名称>
# 作用：查看Deployment详细信息，包括副本状态、更新历史等
# 示例：kubectl describe deployment nginx-app

# 扩缩容Deployment
kubectl scale deployment <部署名称> --replicas=<副本数>
# 作用：调整Deployment的副本数量
# 示例：kubectl scale deployment nginx-app --replicas=5

# 设置自动扩缩容
kubectl autoscale deployment <部署名称> --min=<最小副本> --max=<最大副本> --cpu-percent=<CPU阈值>
# 作用：为Deployment设置HPA自动扩缩容
# 示例：kubectl autoscale deployment nginx-app --min=2 --max=10 --cpu-percent=70
```

#### **Service管理**

```bash
# 创建Service（暴露服务）
kubectl expose deployment <部署名称> --type=<服务类型> --port=<端口>
# 作用：为Deployment创建Service
# 参数说明：
# --type：服务类型（ClusterIP、NodePort、LoadBalancer）
# --port：Service端口
# --target-port：Pod端口

kubectl expose deployment nginx-app --type=NodePort --port=80 --target-port=80
# 示例：为nginx-app创建NodePort类型的Service

# 查看Service列表
kubectl get services
# 简写：kubectl get svc
# 作用：列出所有Service及其端点信息

kubectl get svc -o wide
# 显示Service的详细信息，包括端点和选择器

# 查看Service详细信息
kubectl describe service <服务名称>
# 作用：显示Service的详细配置和端点信息
# 示例：kubectl describe service nginx-app
```

### 7.3 配置管理

#### **ConfigMap管理**

```bash
# 从文件创建ConfigMap
kubectl create configmap <配置名称> --from-file=<文件路径>
# 作用：从文件创建ConfigMap
# 示例：kubectl create configmap app-config --from-file=./config.properties

# 从字面值创建ConfigMap
kubectl create configmap <配置名称> --from-literal=<key>=<value>
# 作用：从命令行键值对创建ConfigMap
# 示例：kubectl create configmap db-config --from-literal=host=localhost --from-literal=port=3306

# 查看ConfigMap
kubectl get configmaps
# 简写：kubectl get cm
# 作用：列出所有ConfigMap

kubectl describe configmap <配置名称>
# 作用：查看ConfigMap的详细内容
# 示例：kubectl describe configmap app-config

# 编辑ConfigMap
kubectl edit configmap <配置名称>
# 作用：在线编辑ConfigMap内容
# 示例：kubectl edit configmap app-config
```

#### **Secret管理**

```bash
# 创建Secret
kubectl create secret generic <密钥名称> --from-literal=<key>=<value>
# 作用：创建通用Secret
# 示例：kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret123

# 从文件创建Secret
kubectl create secret generic <密钥名称> --from-file=<文件路径>
# 示例：kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key

# 查看Secret（不显示内容）
kubectl get secrets
# 作用：列出所有Secret，但不显示具体值

kubectl describe secret <密钥名称>
# 作用：查看Secret的元数据信息，不显示具体值
# 示例：kubectl describe secret db-secret

# 查看Secret内容（base64解码）
kubectl get secret <密钥名称> -o jsonpath='{.data.<key>}' | base64 --decode
# 作用：解码并显示Secret中特定键的值
# 示例：kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode
```

### 7.4 应用调试与故障排查

#### **日志查看**

```bash
# 查看Pod日志
kubectl logs <Pod名称>
# 作用：查看Pod中第一个容器的日志
# 常用参数：
# -f：实时跟踪日志
# --tail=<行数>：显示最后N行
# --since=<时间>：显示指定时间后的日志
# -c <容器名称>：多容器Pod中指定容器

kubectl logs nginx-app-66b6c48dd5-5x8ql -f --tail=100
# 示例：实时查看Pod最后100行日志

# 查看多容器Pod的特定容器日志
kubectl logs <Pod名称> -c <容器名称>
# 示例：kubectl logs multi-container-pod -c nginx-container

# 查看上一次重启前的日志
kubectl logs <Pod名称> --previous
# 作用：查看Pod重启前的日志，常用于调试容器崩溃问题
# 示例：kubectl logs crashed-pod --previous
```

#### **Pod执行命令**

```bash
# 在Pod中执行命令
kubectl exec -it <Pod名称> -- <命令>
# 作用：在运行的Pod中执行命令
# 参数说明：
# -i：保持标准输入开启
# -t：分配TTY终端
# -c <容器名称>：多容器Pod中指定容器

kubectl exec -it nginx-app-66b6c48dd5-5x8ql -- /bin/bash
# 示例：进入Pod的bash终端

kubectl exec -it nginx-app-66b6c48dd5-5x8ql -- ls -la /etc/nginx/
# 示例：在Pod中执行ls命令

# 多容器Pod中执行命令
kubectl exec -it <Pod名称> -c <容器名称> -- <命令>
# 示例：kubectl exec -it multi-container-pod -c app-container -- /bin/sh
```

#### **文件传输**

```bash
# 从Pod复制文件到本地
kubectl cp <Pod名称>:<Pod中的文件路径> <本地路径>
# 作用：从Pod中复制文件到本地
# 示例：kubectl cp nginx-app-66b6c48dd5-5x8ql:/etc/nginx/nginx.conf ./nginx.conf

# 从本地复制文件到Pod
kubectl cp <本地文件路径> <Pod名称>:<Pod中的路径>
# 作用：从本地复制文件到Pod中
# 示例：kubectl cp ./config.json nginx-app-66b6c48dd5-5x8ql:/app/config.json

# 多容器Pod文件传输
kubectl cp <本地路径> <Pod名称>:<Pod路径> -c <容器名称>
# 示例：kubectl cp ./app.jar multi-container-pod:/app/ -c app-container
```

### 7.5 资源管理与维护

#### **配置文件资源管理**

```bash
# 通过配置文件删除资源
kubectl delete -f <配置文件>
# 作用：删除配置文件中定义的所有资源
# 示例：kubectl delete -f deployment.yaml

# 删除目录下所有配置定义的资源
kubectl delete -f <目录路径>/
# 示例：kubectl delete -f ./k8s-configs/

# 创建资源（不存在时才创建）
kubectl create -f <配置文件>
# 作用：创建资源，如果已存在会报错
# 与apply的区别：apply可更新已存在的资源

# 替换资源（完全替换）
kubectl replace -f <配置文件>
# 作用：完全替换现有资源
# 注意：资源必须已存在，否则报错

# 导出资源配置到文件
kubectl get deployment nginx-app -o yaml > nginx-deployment.yaml
# 作用：将现有资源导出为YAML文件
# 用途：备份配置、迁移资源、作为模板

# 导出资源配置（清理状态字段）
kubectl get deployment nginx-app -o yaml --export > nginx-template.yaml
# 作用：导出不含状态信息的纯配置
# 注意：--export在新版本已废弃，建议手动清理状态字段
```

#### **资源删除**

```bash
# 删除Pod
kubectl delete pod <Pod名称>
# 作用：删除指定Pod
# 注意：由Deployment管理的Pod删除后会自动重建

# 删除Deployment
kubectl delete deployment <部署名称>
# 作用：删除Deployment及其管理的所有Pod
# 示例：kubectl delete deployment nginx-app

# 删除Service
kubectl delete service <服务名称>
# 作用：删除指定Service
# 示例：kubectl delete service nginx-app

# 强制删除卡住的Pod
kubectl delete pod <Pod名称> --force --grace-period=0
# 作用：强制删除无法正常终止的Pod
# 参数说明：
# --force：强制删除
# --grace-period=0：立即删除，不等待优雅关闭
```

#### **资源编辑**

```bash
# 在线编辑资源
kubectl edit <资源类型> <资源名称>
# 作用：在线编辑Kubernetes资源的YAML配置
# 示例：kubectl edit deployment nginx-app

# 补丁更新资源
kubectl patch <资源类型> <资源名称> -p '<JSON补丁>'
# 作用：使用JSON补丁局部更新资源
# 示例：kubectl patch deployment nginx-app -p '{"spec":{"replicas":5}}'

# 更新镜像版本
kubectl set image deployment/<部署名称> <容器名称>=<新镜像>
# 作用：更新Deployment中容器的镜像版本
# 示例：kubectl set image deployment/nginx-app nginx=nginx:1.21

# 查看部署历史
kubectl rollout history deployment/<部署名称>
# 作用：查看Deployment的版本历史
# 示例：kubectl rollout history deployment/nginx-app

# 查看特定版本的详情
kubectl rollout history deployment/<部署名称> --revision=<版本号>
# 示例：kubectl rollout history deployment/nginx-app --revision=2

# 回滚到上一个版本
kubectl rollout undo deployment/<部署名称>
# 作用：回滚Deployment到上一个版本
# 示例：kubectl rollout undo deployment/nginx-app

# 回滚到指定版本
kubectl rollout undo deployment/<部署名称> --to-revision=<版本号>
# 作用：回滚到特定版本
# 示例：kubectl rollout undo deployment/nginx-app --to-revision=3

# 查看滚动更新状态
kubectl rollout status deployment/<部署名称>
# 作用：实时查看Deployment滚动更新的进度
# 示例：kubectl rollout status deployment/nginx-app

# 暂停滚动更新
kubectl rollout pause deployment/<部署名称>
# 作用：暂停正在进行的滚动更新
# 用途：发现问题时紧急暂停

# 恢复滚动更新
kubectl rollout resume deployment/<部署名称>
# 作用：恢复已暂停的滚动更新

# 重启Deployment（触发滚动更新）
kubectl rollout restart deployment/<部署名称>
# 作用：触发Deployment的滚动重启
# 用途：配置更新后强制重启所有Pod
```

### 7.6 高级运维命令

#### **资源监控**

```bash
# 查看节点资源使用情况
kubectl top nodes
# 作用：显示集群节点的CPU和内存使用情况
# 注意：需要安装metrics-server

# 查看Pod资源使用情况
kubectl top pods
# 作用：显示Pod的CPU和内存使用情况
# 常用参数：
# -n <namespace>：指定命名空间
# --all-namespaces：所有命名空间
# --sort-by=<字段>：按指定字段排序

kubectl top pods --all-namespaces --sort-by=memory
# 示例：按内存使用量排序显示所有Pod

# 查看容器资源使用
kubectl top pods <Pod名称> --containers
# 作用：显示Pod中每个容器的资源使用情况
```

#### **事件查看**

```bash
# 查看集群事件
kubectl get events
# 作用：显示集群中的事件信息，用于故障诊断
# 常用参数：
# --sort-by='.lastTimestamp'：按时间排序
# --field-selector：字段过滤

kubectl get events --sort-by='.lastTimestamp'
# 示例：按时间顺序查看事件

# 监控实时事件
kubectl get events --watch
# 作用：实时监控集群事件

# 查看特定对象的事件
kubectl describe <资源类型> <资源名称>
# 事件信息包含在describe输出的Events部分
```

#### **标签与选择器**

```bash
# 给资源添加标签
kubectl label <资源类型> <资源名称> <key>=<value>
# 作用：为Kubernetes资源添加标签
# 示例：kubectl label pod nginx-pod environment=production

# 修改标签
kubectl label <资源类型> <资源名称> <key>=<新value> --overwrite
# 作用：修改已存在的标签值
# 示例：kubectl label pod nginx-pod environment=staging --overwrite

# 删除标签
kubectl label <资源类型> <资源名称> <key>-
# 作用：删除指定标签
# 示例：kubectl label pod nginx-pod environment-

# 基于标签选择资源
kubectl get pods -l <标签选择器>
# 作用：根据标签选择器查找资源
# 示例：
kubectl get pods -l environment=production
kubectl get pods -l 'environment in (production,staging)'
kubectl get pods -l app=nginx,version!=v1.0
```

### 7.7 实用技巧与快捷操作

#### **命令简写**

```bash
# 常用资源类型简写
kubectl get po        # pods
kubectl get svc       # services  
kubectl get deploy    # deployments
kubectl get rs        # replicasets
kubectl get ns        # namespaces
kubectl get cm        # configmaps
kubectl get pv        # persistentvolumes
kubectl get pvc       # persistentvolumeclaims
kubectl get ing       # ingresses

# 查看资源的简写形式
kubectl api-resources
# 作用：显示所有资源类型及其简写形式
```

#### **输出格式化**

```bash
# YAML格式输出
kubectl get pod <Pod名称> -o yaml
# 作用：以YAML格式显示资源配置

# JSON格式输出
kubectl get pod <Pod名称> -o json
# 作用：以JSON格式显示资源配置

# 自定义列显示
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
# 作用：自定义显示列

# JSONPath提取特定字段
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
# 作用：使用JSONPath提取特定字段值
```

#### **批量操作**

```bash
# 批量删除资源
kubectl delete pods --all
# 作用：删除当前命名空间所有Pod

kubectl delete deployment,service -l app=nginx
# 作用：删除标签为app=nginx的所有deployment和service

# 从文件批量操作
kubectl apply -f <目录>/
# 作用：应用目录下所有YAML文件

kubectl delete -f <目录>/
# 作用：删除目录下所有YAML文件定义的资源
```

### 7.8 命令使用最佳实践

#### **生产环境安全建议**

```bash
# 1. 始终指定命名空间，避免误操作
kubectl get pods -n production

# 2. 删除操作前确认资源
kubectl get deployment nginx-app -o yaml > backup.yaml
kubectl delete deployment nginx-app

# 3. 使用dry-run验证操作
kubectl apply -f deployment.yaml --dry-run=client
kubectl delete pod nginx-pod --dry-run=client

# 4. 重要操作使用确认提示
kubectl delete deployment nginx-app --wait=true
```

#### **效率提升技巧**

```bash
# 1. 设置bash别名
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'

# 2. 使用kubectl插件
kubectl krew install tree
kubectl tree deployment nginx-app

# 3. 配置多集群上下文
kubectl config get-contexts
kubectl config use-context production-cluster

# 4. 使用--help获取命令帮助
kubectl logs --help
kubectl exec --help
```

这些命令涵盖了Kubernetes日常运维的核心场景，熟练掌握后将大大提高您的工作效率。建议在实际环境中多加练习，逐步形成肌肉记忆。

---

## 第三篇总结

通过本篇的学习，您已经掌握了Kubernetes生产环境的核心运维技能：

### 🎯 主要收获

**安全防护**：
- 掌握了RBAC权限控制的配置和最佳实践
- 学会了网络策略的设计和实施
- 理解了Pod安全标准和容器安全配置
- 了解了Secret管理和加密机制

**高可用设计**：
- 学会了Pod反亲和性和节点调度策略
- 掌握了资源配额和限制的配置方法
- 理解了自动扩缩容的配置和调优
- 了解了多副本和故障转移机制

**运维保障**：
- 掌握了etcd和应用数据的备份策略
- 学会了监控系统的搭建和配置
- 理解了日志收集和分析的最佳实践
- 了解了故障恢复和应急处理流程

**性能优化**：
- 学会了资源配置的优化策略
- 掌握了网络性能调优方法
- 理解了存储性能优化技巧
- 了解了集群性能监控和分析

### 🚀 完整学习回顾

恭喜您完成了Kubernetes快速上手教程的全部学习！通过三篇文章的系统学习：

**第一篇**：建立了坚实的理论基础
**第二篇**：获得了实战部署能力  
**第三篇**：具备了生产运维技能


现在，您已经具备了完整的Kubernetes技能体系，从基础概念到生产运维，是时候在真实项目中发挥这些能力了！

祝您在云原生的道路上越走越远！🚀

---

## 系列文章索引

- [第一篇：基础概念与架构](K8S_PART1_FUNDAMENTALS.md)
- [第二篇：配置管理与实战应用](K8S_PART2_PRACTICE.md)  
- [第三篇：生产环境与最佳实践](K8S_PART3_PRODUCTION.md)

*感谢您完成本教程的学习。如果您觉得有帮助，欢迎分享给更多需要的人。*
