# Master Kubernetes in Hours with AI: Part 3 - Production-Ready in Record Time

## How This Tutorial Was Created with AI

*This comprehensive Kubernetes tutorial series was created through an innovative collaboration with AI, demonstrating how modern developers can rapidly master complex technologies. In just a few hours of interaction with Claude AI, I transformed from a Kubernetes beginner to having production-ready knowledge - and you can too.*

**The AI-Accelerated Learning Approach:**
- 🤖 **Structured Knowledge Extraction**: AI helped organize scattered Kubernetes concepts into a logical learning path
- 💡 **Interactive Q&A Learning**: Complex topics were clarified through continuous dialogue, just like having a senior engineer beside you
- 🚀 **Practice-Driven Content**: Every concept includes hands-on examples generated and tested with AI assistance
- 📚 **Comprehensive Yet Concise**: AI helped distill years of community knowledge into hours of focused learning

**What Makes This Different:**
Traditional documentation can be overwhelming and fragmented. By leveraging AI, this tutorial provides a curated, progressive learning experience that adapts to real-world needs. Each section answers the "why" before the "how," ensuring you understand not just commands, but the reasoning behind Kubernetes design choices.

**Your Learning Journey:**
Part 1 establishes the foundation - core concepts and architecture. Part 2 delivers hands-on deployment skills. Part 3 prepares you for production. All created through iterative AI collaboration, saving you hundreds of hours of research.

*Ready to experience the future of technical learning? Let's dive in.*

---

## Part 3 Overview

Welcome to Part 3 of mastering Kubernetes with AI! This is the final part of our tutorial series, which will take you from laboratory environments to real production deployments.

In the previous two parts, you have:
- Mastered Kubernetes core concepts and architecture
- Learned configuration file writing and application deployment

Now, we will focus on:
- **Production Environment Considerations** - Security, reliability, scalability
- **Operations Best Practices** - Monitoring, backup, failure handling
- **Performance Optimization Strategies** - Resource management, scheduling optimization

This content will help you build production-grade Kubernetes cluster operations capabilities, ensuring your applications run stably and reliably in real environments.

**Prerequisites**:
- Completed the first two parts of the tutorial
- Some system operations experience
- Understanding of basic network and security concepts

**Applicable Scenarios**:
- Teams preparing to deploy Kubernetes in production environments
- Engineers responsible for Kubernetes cluster operations
- Technical managers needing to establish best practice standards

**Estimated Reading Time**: 2-3 hours

---

## Table of Contents

6. [Production Environment Best Practices](#6-production-environment-best-practices)
7. [Common Kubernetes Commands](#7-common-kubernetes-commands)

---

## 6. Production Environment Best Practices

### 6.1 Security Best Practices

Production Kubernetes security requires full utilization of **native security mechanisms**, including RBAC authorization, network policies, Pod security standards, admission controllers, and other core features.

#### **Kubernetes Security Control Plane**

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes Security Control Flow            │
│                                                         │
│ kubectl → API Server → Authentication → Authorization → Admission Control → etcd │
│    │          │        │      │       │         │      │
│  Client     Gateway   Auth    RBAC   AdmissionC  Storage │
│  Cert      TLS Term   Token   Perms   OPA Policy  Encrypt │
│  kubeconfig LoadBal   JWT/OIDC ClusterRole PodSecurity etcd │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes RBAC Permission System**

**RBAC Four Elements Relationship**:
- **User/ServiceAccount**: Identity subject
- **Role/ClusterRole**: Permission definition
- **RoleBinding/ClusterRoleBinding**: Binding relationship
- **Resources + Verbs**: Resources and operations

**Production-Grade RBAC Design Patterns**:

| Role Type | Permission Scope | Typical Resources | Key verbs | Use Cases |
|-----------|------------------|------------------|-----------|-----------|
| **cluster-admin** | Cluster-level | All resources | * | Platform administrators |
| **namespace-admin** | Namespace-level | All within namespace | create,delete,get,list,update | Project owners |
| **developer** | Namespace-level | pods,services,deployments | get,list,watch,create,update | Developers |
| **readonly** | Namespace-level | All resources | get,list,watch | Read-only access |
| **service-account** | Application-level | Specific resources | Minimal privileges | Application Pods |

**ServiceAccount Best Practice Configuration**:
```yaml
# Application-specific ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
automountServiceAccountToken: false  # Disable auto-mounting
---
# Minimal privilege Role
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

#### **2. Kubernetes Network Policy Implementation**

**NetworkPolicy Working Principle**:
- Implements micro-segmentation based on Pod label selectors
- Supports Ingress (inbound) and Egress (outbound) traffic control
- Enforced through CNI plugins (such as Calico, Cilium)

**Layered Network Security Policies**:
```yaml
# 1. Global default deny policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# 2. Allow frontend to access backend
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
# 3. Allow DNS service access
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

#### **3. Pod Security Standards (PSS) Implementation**

**Kubernetes Pod Security Three-Level Standards**:
- **Privileged**: No restrictions, allows known privilege escalations
- **Baseline**: Prevents known privilege escalations
- **Restricted**: Strict restrictions, follows Pod hardening best practices

**Pod Security Policy Configuration**:
```yaml
# Namespace-level Pod security standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Restricted-level Pod configuration
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

#### **4. Kubernetes Admission Controllers**

**Key Admission Controller Configuration**:
```yaml
# OPA Gatekeeper policy example
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
# Apply policy constraint
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

#### **5. Secret Encryption and Management**

**etcd Static Encryption Configuration**:
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
        secret: <32-byte base64 encoded key>
  - identity: {}
```

**Secret Usage Best Practices**:
```yaml
# 1. Restrict Secret access permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["mysecret"]  # Restrict to specific Secret
  verbs: ["get"]
---
# 2. Use projected volumes
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
            mode: 0400  # Read-only permission
```

**Production Environment Security Checklist**:
- ☑️ Enable API Server audit logging
- ☑️ Configure etcd static encryption
- ☑️ Implement Pod Security Standards
- ☑️ Deploy OPA Gatekeeper policies
- ☑️ Configure comprehensive network policies
- ☑️ Use minimal privilege RBAC
- ☑️ Regularly rotate ServiceAccount tokens
- ☑️ Integrate image security scanning

### 6.2 High Availability Configuration

Production environments need to fully utilize Kubernetes **native high availability mechanisms**, including replica control, scheduling policies, health checks, auto-scaling, and other features to build resilient applications.

#### **Kubernetes High Availability Controllers**

```
┌─────────────────────────────────────────────────────────┐
│               Kubernetes High Availability Mechanisms    │
│                                                         │
│ Deployment → ReplicaSet → Pods → kubelet → Container Runtime │
│      │           │         │        │         │        │
│   Replica Mgmt   Pod Mgmt   Scheduler Node Agent Health Check │
│   Rolling Update Label Select Affinity Probe Check Auto Restart │
│   HPA Scaling   Failover    Topology Spread Resource Monitor Log Collection │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes Scheduling and Affinity**

**Pod Scheduler Working Principle**:
- **Scheduling Queue**: Pending Pods waiting for scheduling
- **Filtering Phase**: Screen schedulable nodes
- **Scoring Phase**: Score candidate nodes
- **Binding Phase**: Assign Pod to optimal node

**High Availability Scheduling Strategy Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 3
  template:
    spec:
      # 1. Pod anti-affinity - distribute to different nodes
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["critical-app"]
            topologyKey: kubernetes.io/hostname
        # 2. Node affinity - prefer high-performance nodes
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values: ["high-performance"]
      # 3. Topology spread constraints - distribute evenly across availability zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: critical-app
```

#### **2. Kubernetes Health Check Mechanisms**

**Three Probe Types Comparison**:

| Probe Type | Trigger Timing | Failure Behavior | Use Cases |
|------------|---------------|------------------|-----------|
| **startupProbe** | Container startup | Kill container and restart | Slow-starting applications |
| **livenessProbe** | Container runtime | Kill container and restart | Detect deadlocks/memory leaks |
| **readinessProbe** | Container runtime | Remove from Service endpoints | Detect application readiness |

**Production-Grade Health Check Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        # Startup probe - give slow-starting applications more time
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30  # Allow 150 seconds startup time
        # Liveness probe - detect if application is running healthily
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        # Readiness probe - detect if application is ready to receive traffic
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

#### **3. Kubernetes Resource Management**

**QoS Classes and Scheduling Priority**:
```yaml
# Guaranteed QoS - highest priority
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
        memory: "512Mi"  # Equal to requests
        cpu: "500m"      # Equal to requests
---
# Burstable QoS - medium priority
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
        memory: "512Mi"  # Greater than requests
        cpu: "800m"      # Greater than requests
---
# Resource quota control
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

#### **4. Kubernetes Auto-scaling**

**HPA Controller Working Principle**:
- **Metrics Collection**: metrics-server collects Pod resource usage
- **Calculation Algorithm**: Desired replicas = ceil[Current replicas * (Current metric value / Target metric value)]
- **Smoothing Mechanism**: Avoid frequent scaling oscillations

**Production-Grade HPA Configuration**:
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
  # CPU metrics
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Memory metrics
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metrics - QPS
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  # Scaling behavior control
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

#### **5. Kubernetes Storage High Availability**

**StatefulSet Stateful Application Deployment**:
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
  # Independent PVC for each Pod
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
# Headless Service for stable network identity
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

**Kubernetes High Availability Checklist**:
- ☑️ Configure Pod anti-affinity and topology distribution
- ☑️ Implement multi-layer health checks (startup/liveness/readiness)
- ☑️ Set reasonable resource requests and limits
- ☑️ Enable HPA horizontal auto-scaling
- ☑️ Use StatefulSet for stateful applications
- ☑️ Configure PodDisruptionBudget to prevent excessive eviction
- ☑️ Use multi-replica and cross-AZ deployment
- ☑️ Monitor Pod restart and scheduling failure events

### 6.3 Backup and Recovery Strategies

Production environments need to focus on **Kubernetes cluster-level backup and recovery**, including etcd cluster state, Kubernetes resource objects, persistent storage, and other core component data protection.

#### **Kubernetes Backup and Recovery Architecture**

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes Cluster Backup System           │
│                                                         │
│ etcd Cluster → Resource Objects → Persistent Volumes → Config Data → Application State │
│     │         │         │         │         │          │
│  Cluster State API Objects Storage Data Config Mgmt Business Data │
│  Snapshot Backup YAML Export Volume Snapshot Secret Storage Database Backup │
│  Multi-replica Version Control Cross-AZ Replication Encrypted Storage Stream Replication │
│└─────────────────────────────────────────────────────────┘
```

#### **1. etcd Cluster Backup Strategy**

**etcd High Availability Architecture Backup**:
```bash
# 1. Create etcd snapshot
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S).db

# 2. Verify snapshot status
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /backup/etcd-snapshot-*.db

# 3. Restore etcd cluster
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name m1 \
  --data-dir=/var/lib/etcd-from-backup \
  --initial-cluster=m1=https://host1:2380,m2=https://host2:2380,m3=https://host3:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://host1:2380
```

**Automated etcd Backup CronJob**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Backup every 6 hours
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
              # Upload to object storage
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

#### **2. Kubernetes Resource Object Backup**

**Using Velero for Cluster-Level Backup**:
```yaml
# 1. Install Velero backup tool
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
# 2. Create backup storage location
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
# 3. Create backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
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
    ttl: 720h  # Retain for 30 days
```

**Manual Kubernetes Resource Backup**:
```bash
# 1. Backup all namespace resources
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# 2. Backup specific resource types
kubectl get deployments,services,configmaps,secrets -n production -o yaml > production-backup.yaml

# 3. Backup custom resources (CRD)
kubectl get crd -o yaml > crd-backup.yaml
kubectl get <custom-resource> --all-namespaces -o yaml > custom-resources-backup.yaml

# 4. kubectl backup script
#!/bin/bash
BACKUP_DIR="/backup/k8s-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup core resources
for resource in deployments services configmaps secrets persistentvolumeclaims; do
  kubectl get $resource --all-namespaces -o yaml > $BACKUP_DIR/$resource.yaml
done

# Backup cluster-level resources
for resource in nodes persistentvolumes storageclasses; do
  kubectl get $resource -o yaml > $BACKUP_DIR/$resource.yaml
done
```

#### **3. Persistent Storage Backup**

**Using CSI Snapshots for Volume Backup**:
```yaml
# 1. Define volume snapshot class
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
# 2. Create volume snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-data-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: mysql-data-pvc
---
# 3. Restore PVC from snapshot
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

**Automated Volume Snapshot Backup**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: volume-snapshot-backup
spec:
  schedule: "0 1 * * *"  # Daily at 1 AM
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
              # Get all PVCs to backup
              for pvc in $(kubectl get pvc -l backup=enabled -o name); do
                pvc_name=$(echo $pvc | cut -d'/' -f2)
                snapshot_name="$pvc_name-$(date +%Y%m%d)"
                
                # Create snapshot
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

#### **4. Cross-Cluster Disaster Recovery**

**Multi-Cluster Backup Strategy**:
```yaml
# 1. Cross-cluster resource synchronization
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
# 2. Cross-region PV replication
apiVersion: v1
kind: ConfigMap
metadata:
  name: cross-region-backup-config
data:
  script: |
    #!/bin/bash
    # Cross-region volume data synchronization
    SOURCE_CLUSTER="arn:aws:eks:us-west-2:account:cluster/prod"
    TARGET_CLUSTER="arn:aws:eks:us-east-1:account:cluster/dr"
    
    # Sync PV snapshots to disaster recovery region
    aws ec2 copy-snapshot \
      --source-region us-west-2 \
      --source-snapshot-id $SNAPSHOT_ID \
      --destination-region us-east-1 \
      --description "DR backup from prod cluster"
```

#### **5. Backup Validation and Recovery Testing**

**Automated Recovery Testing**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-validation
spec:
  schedule: "0 4 * * 0"  # Every Sunday at 4 AM
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
              # 1. Verify etcd snapshot integrity
              etcdctl snapshot status /backup/latest-etcd-snapshot.db
              
              # 2. Verify Velero backup status
              velero backup get --output json | jq '.items[] | select(.status.phase != "Completed")'
              
              # 3. Verify volume snapshot status
              kubectl get volumesnapshots -o json | jq '.items[] | select(.status.readyToUse != true)'
              
              # 4. Send validation result notification
              curl -X POST $SLACK_WEBHOOK_URL \
                -H 'Content-type: application/json' \
                --data '{"text":"Backup validation completed at $(date)"}'
          restartPolicy: OnFailure
```

**Kubernetes Backup Checklist**:
- ☑️ Configure etcd automatic backup and encrypted storage
- ☑️ Deploy Velero for application-level backup
- ☑️ Enable CSI volume snapshot functionality
- ☑️ Implement cross-AZ/cross-region backup replication
- ☑️ Regularly execute recovery testing validation
- ☑️ Monitor backup task execution status
- ☑️ Establish backup data retention policies
- ☑️ Configure backup failure alerting mechanisms

### 6.4 Performance Optimization

Production environment performance optimization requires deep understanding of **Kubernetes scheduler**, **container runtime**, **network model** and other core components' working principles, achieving optimal performance through fine-grained configuration.

#### **Kubernetes Performance Tuning System**

```
┌─────────────────────────────────────────────────────────┐
│               Kubernetes Performance Tuning Chain       │
│                                                         │
│ kube-scheduler → kubelet → Container Runtime → CNI Network → CSI Storage │
│       │            │         │           │        │     │
│   Scheduling Algo  Node Agent Container Isolation Network Plugin Storage Driver │
│   Affinity Rules   Resource Mgmt CGroup     iptables   Block Device │
│   Priority Queue   Monitor Metrics Namespace Load Balance File System │
└─────────────────────────────────────────────────────────┘
```

#### **1. Kubernetes Scheduler Performance Optimization**

**Scheduler Configuration Optimization**:
```yaml
# kube-scheduler configuration file
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
        type: LeastAllocated  # Optimize resource utilization
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

**High-Performance Scheduling Strategies**:
```yaml
# 1. CPU topology-aware scheduling
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # CPU Manager policy - reserve dedicated CPUs for critical applications
      containers:
      - name: cpu-intensive-app
        resources:
          requests:
            cpu: "4"          # Integer CPU ensures exclusivity
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
      nodeSelector:
        node.kubernetes.io/instance-type: "c5.2xlarge"
---
# 2. NUMA node affinity
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
  # Use topology manager policy
  runtimeClassName: numa-aware
```

#### **2. Container Runtime Performance Tuning**

**containerd Runtime Optimization**:
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  # Enable CPU Manager
  [plugins."io.containerd.grpc.v1.cri".containerd]
    default_runtime_name = "runc"
    
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
      
  # Performance optimization configuration
  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://mirror.aliyuncs.com"]
```

**kubelet Performance Parameter Optimization**:
```yaml
# kubelet configuration file
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# CPU management policy
cpuManagerPolicy: "static"
cpuManagerReconcilePeriod: "10s"
# Memory management policy  
memoryManagerPolicy: "Static"
reservedMemory:
- numaNode: 0
  limits:
    memory: "1Gi"
# Topology manager policy
topologyManagerPolicy: "single-numa-node"
# Performance-related parameters
maxPods: 110
podPidsLimit: 4096
systemReserved:
  cpu: "500m"
  memory: "1Gi"
kubeReserved:
  cpu: "500m" 
  memory: "1Gi"
```

#### **3. Kubernetes Network Performance Optimization**

**CNI Network Plugin Performance Comparison**:

| CNI Plugin | Latency | Bandwidth | CPU Overhead | Feature Support |
|------------|---------|-----------|--------------|----------------|
| **Flannel** | Low | High | Low | Simple and easy |
| **Calico** | Medium | High | Medium | Network Policy/BGP |
| **Cilium** | Low | Very High | Low | eBPF/L7 Policy |
| **Weave** | High | Medium | High | Encrypted communication |

**Cilium eBPF Network Acceleration Configuration**:
```yaml
# Cilium ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  # eBPF datapath acceleration
  datapath-mode: "veth"
  enable-bpf-masquerade: "true"
  enable-host-routing: "true"
  
  # Performance optimization options
  kube-proxy-replacement: "partial"
  enable-bandwidth-manager: "true"
  enable-local-redirect-policy: "true"
  
  # XDP load balancing
  enable-node-port: "true"
  node-port-mode: "hybrid"
  loadbalancer-mode: "dsr"
```

**Service Performance Optimization Configuration**:
```yaml
# High-performance Service configuration
apiVersion: v1
kind: Service
metadata:
  name: high-perf-service
  annotations:
    # Cilium-specific load balancing optimization
    service.cilium.io/lb-mode: "dsr"
    service.cilium.io/global: "true"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # Avoid double forwarding
  internalTrafficPolicy: Local    # Local traffic optimization
  sessionAffinity: ClientIP       # Session persistence
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

#### **4. Kubernetes Storage Performance Optimization**

**CSI Storage Driver Performance Configuration**:
```yaml
# High-performance StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-nvme-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "16000"        # Maximum IOPS
  throughput: "1000"   # Maximum throughput MiB/s
  fsType: ext4
mountOptions:
- noatime              # Reduce access time updates
- nodiratime          # Reduce directory access time updates
- barrier=0           # Disable write barriers (use cautiously)
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Local SSD storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner
parameters:
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
```

**High IOPS Application Pod Configuration**:
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
      # Mount option optimization
      mountPropagation: None
    - name: logs-volume  
      mountPath: /var/log/mysql
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: mysql-data-pvc
  - name: logs-volume
    emptyDir:
      medium: Memory     # Use memory for log storage
      sizeLimit: 1Gi
```

#### **5. Application-Level Performance Monitoring and Tuning**

**Kubernetes Performance Metrics Collection**:
```yaml
# VPA vertical auto-scaling recommendations
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
      # Resource recommendation strategy
      mode: Auto
---
# Performance monitoring ServiceMonitor
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
    # Key performance metrics
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'kube_pod_container_resource_(requests|limits)_(cpu_cores|memory_bytes)'
      action: keep
```

**Kubernetes Performance Tuning Checklist**:
- ☑️ Enable CPU Manager and Memory Manager
- ☑️ Configure NUMA topology-aware scheduling
- ☑️ Optimize container runtime parameters
- ☑️ Choose high-performance CNI plugin (Cilium)
- ☑️ Configure local traffic policies
- ☑️ Use high-performance storage classes
- ☑️ Enable VPA resource right-sizing
- ☑️ Monitor key performance metrics

---

## 7. Common Kubernetes Commands

In daily Kubernetes operations work, proficient mastery of kubectl commands is essential. This chapter will introduce the most commonly used and practical kubectl commands to help you improve operational efficiency.

### 7.1 Resource Viewing and Information Retrieval

#### **Cluster Information Viewing**

```bash
# View cluster information
kubectl cluster-info
# Purpose: Display basic Kubernetes cluster information, including API server address and DNS service address
# Example output:
# Kubernetes control plane is running at https://kubernetes.docker.internal:6443
# CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# View cluster node information
kubectl get nodes
# Purpose: List all nodes in the cluster and their status
# Parameters:
# -o wide: Show more detailed information (IP addresses, OS, etc.)
# --show-labels: Show node labels

kubectl get nodes -o wide
# Example output:
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
# minikube   Ready    control-plane   5d    v1.24.3   192.168.49.2   <none>        Ubuntu 20.04.4 LTS   5.4.0-122-generic   docker://20.10.17

# View detailed node information
kubectl describe node <node-name>
# Purpose: Show detailed information about specified node, including resource usage, system info, Pod distribution
# Example: kubectl describe node minikube
```

#### **Namespace Management**

```bash
# View all namespaces
kubectl get namespaces
# Shorthand: kubectl get ns
# Purpose: List all namespaces in the cluster

# View current namespace in use
kubectl config view --minify --output 'jsonpath={..namespace}'
# Purpose: Show the default namespace used by current kubectl context

# Switch namespace
kubectl config set-context --current --namespace=<namespace-name>
# Purpose: Set default namespace for current context
# Example: kubectl config set-context --current --namespace=production

# Create namespace
kubectl create namespace <namespace-name>
# Purpose: Create new namespace
# Example: kubectl create namespace development
```

#### **Pod Resource Viewing**

```bash
# View Pod list
kubectl get pods
# Purpose: List all Pods in current namespace
# Common parameters:
# -n <namespace>: Specify namespace
# --all-namespaces: View all namespaces (shorthand: -A)
# -o wide: Show detailed information
# --watch: Real-time monitoring of changes (shorthand: -w)

kubectl get pods -n kube-system
# Example: View Pods in system namespace

kubectl get pods --all-namespaces -o wide
# Example: View detailed information of Pods in all namespaces

# View Pod detailed information
kubectl describe pod <pod-name>
# Purpose: Show detailed Pod information including events, container status, resource usage
# Example: kubectl describe pod nginx-deployment-66b6c48dd5-5x8ql

# View Pod status changes (real-time monitoring)
kubectl get pods --watch
# Purpose: Monitor Pod status changes in real-time, commonly used for debugging and deployment monitoring
```

### 7.2 Application Deployment and Management

#### **Configuration File Management Best Practices**

**Configuration File Storage Locations**:
```bash
# Typical project structure
my-app/
├── k8s/                     # Kubernetes configuration directory
│   ├── base/               # Base configurations
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── overlays/           # Environment-specific configurations
│   │   ├── dev/           # Development environment
│   │   │   └── kustomization.yaml
│   │   ├── staging/       # Staging environment
│   │   │   └── kustomization.yaml
│   │   └── prod/          # Production environment
│   │       └── kustomization.yaml
│   └── README.md
├── src/                    # Application source code
└── Dockerfile             # Container image definition
```

**Configuration File Sources and Creation Timing**:

1. **Manual Creation** (Development Phase)
```bash
# Create configuration file templates
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
# Then manually edit deployment.yaml to add more configurations
```

2. **Export from Existing Resources** (Migration/Backup)
```bash
# Export running resource configurations
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
# Clean status fields and save as template
```

3. **From Official/Community Templates** (Third-party Applications)
```bash
# Download official configuration files
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
# Modify as needed and save to project
```

**Version Control**:
```bash
# Configuration files should be version controlled with code
git add k8s/
git commit -m "Add Kubernetes deployment configurations"
git push
```

**Typical Workflow**:
```bash
# 1. Development Phase - Create initial configuration files
kubectl create deployment myapp --image=myapp:v1 --dry-run=client -o yaml > k8s/deployment.yaml
kubectl create service clusterip myapp --tcp=80:8080 --dry-run=client -o yaml > k8s/service.yaml

# 2. Edit configuration files - Add resource limits, health checks, etc.
vim k8s/deployment.yaml

# 3. Local Testing - Apply in development cluster
kubectl apply -f k8s/

# 4. Commit to code repository
git add k8s/
git commit -m "feat: add k8s deployment configs"
git push

# 5. CI/CD - Automatically deploy to different environments
# In CI/CD pipeline:
kubectl apply -f k8s/ -n development  # Development environment
kubectl apply -f k8s/ -n production   # Production environment
```

**Configuration File Storage Location Summary**:
- **Local Development**: k8s/ or deploy/ folder in project root
- **Version Control**: Git repository, managed with application code
- **CI/CD**: Checked out from Git repository, used in pipelines
- **Configuration Center**: Large organizations may use Helm Chart repositories or GitOps tools (like ArgoCD)

#### **Configuration File Deployment (Recommended)**

```bash
# Apply configuration file
kubectl apply -f <configuration-file>
# Purpose: Declarative resource management, create or update resources
# Advantages: Repeatable execution, version control, easy management
# Example: kubectl apply -f deployment.yaml

# Apply multiple configuration files
kubectl apply -f <file1> -f <file2>
# Example: kubectl apply -f deployment.yaml -f service.yaml

# Apply all configuration files in directory
kubectl apply -f <directory-path>/
# Purpose: Recursively apply all YAML files in directory
# Example: kubectl apply -f ./k8s-configs/

# Apply remote configuration file
kubectl apply -f <URL>
# Purpose: Apply configuration directly from URL
# Example: kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/cloud/deploy.yaml

# Preview changes to be applied (dry-run)
kubectl apply -f deployment.yaml --dry-run=client
# Purpose: Preview changes without actually applying
# Parameters:
# --dry-run=client: Client-side simulation
# --dry-run=server: Server-side simulation

# View differences between configuration file and current cluster state
kubectl diff -f deployment.yaml
# Purpose: Show differences between configuration file and cluster resources
# Example: After modifying deployment.yaml, view specific changes

# Record deployment command (for rollback)
kubectl apply -f deployment.yaml --record
# Purpose: Record command in resource annotations for tracking change history
```

#### **Deployment Management**

```bash
# Create Deployment
kubectl create deployment <deployment-name> --image=<image-name>
# Purpose: Quickly create a Deployment
# Parameters:
# --image: Specify container image
# --replicas: Specify replica count
# --port: Specify container port

kubectl create deployment nginx-app --image=nginx:latest --replicas=3 --port=80
# Example: Create a Deployment named nginx-app with 3 replicas

# View Deployment status
kubectl get deployments
# Shorthand: kubectl get deploy
# Purpose: List all Deployments and their status

kubectl describe deployment <deployment-name>
# Purpose: View detailed Deployment information including replica status, update history
# Example: kubectl describe deployment nginx-app

# Scale Deployment
kubectl scale deployment <deployment-name> --replicas=<replica-count>
# Purpose: Adjust Deployment replica count
# Example: kubectl scale deployment nginx-app --replicas=5

# Set auto-scaling
kubectl autoscale deployment <deployment-name> --min=<min-replicas> --max=<max-replicas> --cpu-percent=<cpu-threshold>
# Purpose: Set HPA auto-scaling for Deployment
# Example: kubectl autoscale deployment nginx-app --min=2 --max=10 --cpu-percent=70

# Update image version
kubectl set image deployment/<deployment-name> <container-name>=<new-image>
# Purpose: Update container image version in Deployment
# Example: kubectl set image deployment/nginx-app nginx=nginx:1.21

# View deployment history
kubectl rollout history deployment/<deployment-name>
# Purpose: View Deployment version history
# Example: kubectl rollout history deployment/nginx-app

# View specific version details
kubectl rollout history deployment/<deployment-name> --revision=<revision-number>
# Example: kubectl rollout history deployment/nginx-app --revision=2

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name>
# Purpose: Rollback Deployment to previous version
# Example: kubectl rollout undo deployment/nginx-app

# Rollback to specific version
kubectl rollout undo deployment/<deployment-name> --to-revision=<revision-number>
# Purpose: Rollback to specific version
# Example: kubectl rollout undo deployment/nginx-app --to-revision=3

# View rollout status
kubectl rollout status deployment/<deployment-name>
# Purpose: View real-time progress of Deployment rollout
# Example: kubectl rollout status deployment/nginx-app

# Pause rollout
kubectl rollout pause deployment/<deployment-name>
# Purpose: Pause ongoing rollout
# Use case: Emergency pause when issues are detected

# Resume rollout
kubectl rollout resume deployment/<deployment-name>
# Purpose: Resume paused rollout

# Restart Deployment (trigger rolling update)
kubectl rollout restart deployment/<deployment-name>
# Purpose: Trigger rolling restart of Deployment
# Use case: Force restart all Pods after configuration updates
```

#### **Service Management**

```bash
# Create Service (expose service)
kubectl expose deployment <deployment-name> --type=<service-type> --port=<port>
# Purpose: Create Service for Deployment
# Parameters:
# --type: Service type (ClusterIP, NodePort, LoadBalancer)
# --port: Service port
# --target-port: Pod port

kubectl expose deployment nginx-app --type=NodePort --port=80 --target-port=80
# Example: Create NodePort type Service for nginx-app

# View Service list
kubectl get services
# Shorthand: kubectl get svc
# Purpose: List all Services and their endpoint information

kubectl get svc -o wide
# Show detailed Service information including endpoints and selectors

# View Service detailed information
kubectl describe service <service-name>
# Purpose: Show detailed Service configuration and endpoint information
# Example: kubectl describe service nginx-app
```

### 7.3 Configuration Management

#### **ConfigMap Management**

```bash
# Create ConfigMap from file
kubectl create configmap <config-name> --from-file=<file-path>
# Purpose: Create ConfigMap from file
# Example: kubectl create configmap app-config --from-file=./config.properties

# Create ConfigMap from literal values
kubectl create configmap <config-name> --from-literal=<key>=<value>
# Purpose: Create ConfigMap from command line key-value pairs
# Example: kubectl create configmap db-config --from-literal=host=localhost --from-literal=port=3306

# View ConfigMap
kubectl get configmaps
# Shorthand: kubectl get cm
# Purpose: List all ConfigMaps

kubectl describe configmap <config-name>
# Purpose: View detailed ConfigMap content
# Example: kubectl describe configmap app-config

# Edit ConfigMap
kubectl edit configmap <config-name>
# Purpose: Edit ConfigMap content online
# Example: kubectl edit configmap app-config
```

#### **Secret Management**

```bash
# Create Secret
kubectl create secret generic <secret-name> --from-literal=<key>=<value>
# Purpose: Create generic Secret
# Example: kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret123

# Create Secret from file
kubectl create secret generic <secret-name> --from-file=<file-path>
# Example: kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key

# View Secret (without showing content)
kubectl get secrets
# Purpose: List all Secrets but don't show actual values

kubectl describe secret <secret-name>
# Purpose: View Secret metadata information without showing actual values
# Example: kubectl describe secret db-secret

# View Secret content (base64 decode)
kubectl get secret <secret-name> -o jsonpath='{.data.<key>}' | base64 --decode
# Purpose: Decode and show specific key value in Secret
# Example: kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode
```

### 7.4 Application Debugging and Troubleshooting

#### **Log Viewing**

```bash
# View Pod logs
kubectl logs <pod-name>
# Purpose: View logs from first container in Pod
# Common parameters:
# -f: Follow logs in real-time
# --tail=<lines>: Show last N lines
# --since=<time>: Show logs after specified time
# -c <container-name>: Specify container in multi-container Pod

kubectl logs nginx-app-66b6c48dd5-5x8ql -f --tail=100
# Example: Follow last 100 lines of Pod logs in real-time

# View specific container logs in multi-container Pod
kubectl logs <pod-name> -c <container-name>
# Example: kubectl logs multi-container-pod -c nginx-container

# View logs from previous restart
kubectl logs <pod-name> --previous
# Purpose: View logs before Pod restart, commonly used for debugging container crashes
# Example: kubectl logs crashed-pod --previous
```

#### **Pod Command Execution**

```bash
# Execute command in Pod
kubectl exec -it <pod-name> -- <command>
# Purpose: Execute command in running Pod
# Parameters:
# -i: Keep standard input open
# -t: Allocate TTY terminal
# -c <container-name>: Specify container in multi-container Pod

kubectl exec -it nginx-app-66b6c48dd5-5x8ql -- /bin/bash
# Example: Enter Pod's bash terminal

kubectl exec -it nginx-app-66b6c48dd5-5x8ql -- ls -la /etc/nginx/
# Example: Execute ls command in Pod

# Execute command in multi-container Pod
kubectl exec -it <pod-name> -c <container-name> -- <command>
# Example: kubectl exec -it multi-container-pod -c app-container -- /bin/sh
```

#### **File Transfer**

```bash
# Copy file from Pod to local
kubectl cp <pod-name>:<pod-file-path> <local-path>
# Purpose: Copy file from Pod to local
# Example: kubectl cp nginx-app-66b6c48dd5-5x8ql:/etc/nginx/nginx.conf ./nginx.conf

# Copy file from local to Pod
kubectl cp <local-file-path> <pod-name>:<pod-path>
# Purpose: Copy file from local to Pod
# Example: kubectl cp ./config.json nginx-app-66b6c48dd5-5x8ql:/app/config.json

# File transfer in multi-container Pod
kubectl cp <local-path> <pod-name>:<pod-path> -c <container-name>
# Example: kubectl cp ./app.jar multi-container-pod:/app/ -c app-container
```

### 7.5 Configuration File Resource Management

#### **Configuration File Resource Management**

```bash
# Delete resources via configuration file
kubectl delete -f <configuration-file>
# Purpose: Delete all resources defined in the configuration file
# Example: kubectl delete -f deployment.yaml

# Delete all resources in directory
kubectl delete -f <directory-path>/
# Example: kubectl delete -f ./k8s-configs/

# Create resources (only if they don't exist)
kubectl create -f <configuration-file>
# Purpose: Create resources, will error if they already exist
# Difference from apply: apply can update existing resources

# Replace resources (complete replacement)
kubectl replace -f <configuration-file>
# Purpose: Completely replace existing resources
# Note: Resource must already exist, otherwise will error

# Export resource configuration to file
kubectl get deployment nginx-app -o yaml > nginx-deployment.yaml
# Purpose: Export existing resource as YAML file
# Use cases: Backup configuration, resource migration, template creation

# Export resource configuration (clean status fields)
kubectl get deployment nginx-app -o yaml --export > nginx-template.yaml
# Purpose: Export pure configuration without status information
# Note: --export is deprecated in newer versions, recommend manual cleanup of status fields
```

#### **Resource Deletion**

```bash
# Delete Pod
kubectl delete pod <pod-name>
# Purpose: Delete specified Pod
# Note: Pods managed by Deployment will be automatically recreated after deletion

# Delete Deployment
kubectl delete deployment <deployment-name>
# Purpose: Delete Deployment and all Pods it manages
# Example: kubectl delete deployment nginx-app

# Delete Service
kubectl delete service <service-name>
# Purpose: Delete specified Service
# Example: kubectl delete service nginx-app

# Force delete stuck Pod
kubectl delete pod <pod-name> --force --grace-period=0
# Purpose: Force delete Pod that cannot terminate normally
# Parameters:
# --force: Force deletion
# --grace-period=0: Delete immediately without waiting for graceful shutdown
```

#### **Resource Editing**

```bash
# Edit resource online
kubectl edit <resource-type> <resource-name>
# Purpose: Edit Kubernetes resource YAML configuration online
# Example: kubectl edit deployment nginx-app

# Patch update resource
kubectl patch <resource-type> <resource-name> -p '<json-patch>'
# Purpose: Partially update resource using JSON patch
# Example: kubectl patch deployment nginx-app -p '{"spec":{"replicas":5}}'

# Update image version
kubectl set image deployment/<deployment-name> <container-name>=<new-image>
# Purpose: Update container image version in Deployment
# Example: kubectl set image deployment/nginx-app nginx=nginx:1.21
```

### 7.6 Advanced Operations Commands

#### **Resource Monitoring**

```bash
# View node resource usage
kubectl top nodes
# Purpose: Show CPU and memory usage of cluster nodes
# Note: Requires metrics-server installation

# View Pod resource usage
kubectl top pods
# Purpose: Show CPU and memory usage of Pods
# Common parameters:
# -n <namespace>: Specify namespace
# --all-namespaces: All namespaces
# --sort-by=<field>: Sort by specified field

kubectl top pods --all-namespaces --sort-by=memory
# Example: Show all Pods sorted by memory usage

# View container resource usage
kubectl top pods <pod-name> --containers
# Purpose: Show resource usage of each container in Pod
```

#### **Event Viewing**

```bash
# View cluster events
kubectl get events
# Purpose: Show event information in cluster for troubleshooting
# Common parameters:
# --sort-by='.lastTimestamp': Sort by time
# --field-selector: Field filtering

kubectl get events --sort-by='.lastTimestamp'
# Example: View events in chronological order

# Monitor real-time events
kubectl get events --watch
# Purpose: Monitor cluster events in real-time

# View events for specific object
kubectl describe <resource-type> <resource-name>
# Event information is included in the Events section of describe output
```

#### **Labels and Selectors**

```bash
# Add label to resource
kubectl label <resource-type> <resource-name> <key>=<value>
# Purpose: Add label to Kubernetes resource
# Example: kubectl label pod nginx-pod environment=production

# Modify label
kubectl label <resource-type> <resource-name> <key>=<new-value> --overwrite
# Purpose: Modify existing label value
# Example: kubectl label pod nginx-pod environment=staging --overwrite

# Delete label
kubectl label <resource-type> <resource-name> <key>-
# Purpose: Delete specified label
# Example: kubectl label pod nginx-pod environment-

# Select resources based on labels
kubectl get pods -l <label-selector>
# Purpose: Find resources based on label selector
# Examples:
kubectl get pods -l environment=production
kubectl get pods -l 'environment in (production,staging)'
kubectl get pods -l app=nginx,version!=v1.0
```

### 7.7 Practical Tips and Shortcuts

#### **Command Shortcuts**

```bash
# Common resource type shortcuts
kubectl get po        # pods
kubectl get svc       # services  
kubectl get deploy    # deployments
kubectl get rs        # replicasets
kubectl get ns        # namespaces
kubectl get cm        # configmaps
kubectl get pv        # persistentvolumes
kubectl get pvc       # persistentvolumeclaims
kubectl get ing       # ingresses

# View resource shortcuts
kubectl api-resources
# Purpose: Show all resource types and their shortcuts
```

#### **Output Formatting**

```bash
# YAML format output
kubectl get pod <pod-name> -o yaml
# Purpose: Show resource configuration in YAML format

# JSON format output
kubectl get pod <pod-name> -o json
# Purpose: Show resource configuration in JSON format

# Custom column display
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
# Purpose: Custom display columns

# JSONPath extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
# Purpose: Extract specific field values using JSONPath
```

#### **Batch Operations**

```bash
# Batch delete resources
kubectl delete pods --all
# Purpose: Delete all Pods in current namespace

kubectl delete deployment,service -l app=nginx
# Purpose: Delete all deployments and services with app=nginx label

# Batch operations from files
kubectl apply -f <directory>/
# Purpose: Apply all YAML files in directory

kubectl delete -f <directory>/
# Purpose: Delete all resources defined in YAML files in directory
```

### 7.8 Command Usage Best Practices

#### **Production Environment Safety Recommendations**

```bash
# 1. Always specify namespace to avoid misoperations
kubectl get pods -n production

# 2. Confirm resources before deletion operations
kubectl get deployment nginx-app -o yaml > backup.yaml
kubectl delete deployment nginx-app

# 3. Use dry-run to validate operations
kubectl apply -f deployment.yaml --dry-run=client
kubectl delete pod nginx-pod --dry-run=client

# 4. Use confirmation prompts for important operations
kubectl delete deployment nginx-app --wait=true
```

#### **Efficiency Enhancement Tips**

```bash
# 1. Set bash aliases
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'

# 2. Use kubectl plugins
kubectl krew install tree
kubectl tree deployment nginx-app

# 3. Configure multi-cluster contexts
kubectl config get-contexts
kubectl config use-context production-cluster

# 4. Use --help to get command help
kubectl logs --help
kubectl exec --help
```

These commands cover the core scenarios of daily Kubernetes operations. Proficient mastery will greatly improve your work efficiency. We recommend practicing in actual environments to gradually develop muscle memory.

---

## Production Environment Best Practices Framework Summary

### 🎯 Production-Grade Kubernetes Deployment Decision System

Through learning the four core areas in Part 6, we have built a complete production environment operations system. Here is a systematic decision framework:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Production-Grade Kubernetes Operations Framework        │
│                                                                                 │
│  Security Protection → High Availability Design → Backup Recovery Strategy → Performance Optimization → Operations Monitoring │
│       │            │            │            │            │                    │
│   Multi-layer Protection Disaster Recovery Data Protection  Full-chain Optimization Full-stack Monitoring │
│   Minimal Privileges Fault Self-healing Disaster Recovery  Resource Tuning    Intelligent Alerting │
│   Network Isolation  Elastic Scaling Business Continuity   Architecture Optimization Operations Automation │
│   Compliance Auditing Service Governance RTO/RPO Targets   Continuous Tuning   Problem Diagnosis │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### **Production Environment Maturity Assessment Model**

**Security Maturity Levels**:

| Level | Feature Description | Core Capabilities | Applicable Scale |
|-------|-------------------|------------------|------------------|
| **L1-Basic** | Basic RBAC + Network Policies | Access control, network isolation | Development/test environments |
| **L2-Standard** | Pod Security Standards + Secret Management | Container security, key protection | Pre-production environments |
| **L3-Enhanced** | Multi-layer protection + audit logs | Defense in depth, compliance management | Production environments |
| **L4-Expert** | Zero-trust architecture + threat detection | Adaptive security, intelligent protection | Critical business systems |

**Availability Maturity Levels**:

| Level | SLA Target | Architecture Features | Disaster Recovery Capability |
|-------|------------|---------------------|----------------------------|
| **L1-Basic** | 99.5% | Single replica deployment | Basic restart |
| **L2-Standard** | 99.9% | Multi-replica + anti-affinity | Node-level disaster recovery |
| **L3-Enhanced** | 99.95% | Multi-AZ deployment + auto-scaling | Region-level disaster recovery |
| **L4-Expert** | 99.99% | Multi-region + service mesh | Geographic-level disaster recovery |


### 🚀 Cloud-Native Operations Best Practices

#### **DevOps Integration Best Practices**

**CI/CD Security Integration**:
- **Image Security**: Integrate vulnerability scanning into build pipelines
- **Configuration Validation**: Use OPA Gatekeeper policy validation
- **Progressive Delivery**: Canary deployment + automatic rollback mechanisms
- **Environment Consistency**: Infrastructure as Code (IaC)

**GitOps Operations Mode**:
- **Configuration Versioning**: Store all configurations in Git repositories
- **Declarative Management**: Use ArgoCD and other tools for automatic synchronization
- **Change Auditing**: Complete audit trail for all changes
- **Environment Isolation**: Independent management for development, testing, production environments

#### **SRE Culture and Practices**

**Reliability Engineering Principles**:
- **Error Budget**: Establish quantifiable reliability targets
- **Monitoring-Driven**: Monitoring system based on SLI/SLO/SLA
- **Failure Learning**: Blameless post-incident review mechanisms
- **Automation First**: Reduce manual operations, improve efficiency

**Operations Efficiency Improvement Strategies**:
- **Platform Engineering**: Build internal developer platforms
- **Self-Service**: Developer team autonomous operations capabilities
- **Knowledge Consolidation**: Establish operations knowledge base and standard operating procedures
- **Continuous Improvement**: Continuous optimization based on measurement data

#### **Cost Optimization and Resource Management**

**Resource Cost Optimization Framework**:

| Optimization Dimension | Strategy Methods | Expected Benefits | Implementation Difficulty |
|----------------------|------------------|------------------|-------------------------|
| **Resource Configuration Optimization** | Adjust requests/limits based on monitoring data | 20-30% | Low |
| **Scheduling Strategy Optimization** | Node affinity + taint tolerance | 10-15% | Medium |
| **Storage Cost Optimization** | Tiered storage + lifecycle management | 30-40% | Medium |
| **Network Cost Optimization** | Traffic routing optimization + CDN integration | 15-25% | High |

**FinOps Practices**:
- **Cost Visibility**: Real-time cost monitoring and allocation
- **Budget Control**: Budget management based on ResourceQuota
- **Usage Optimization**: Right-sizing and elastic scheduling
- **Procurement Optimization**: Spot instances and long-term contract optimization

---

## Part 3 Summary

Through this part's learning, you have mastered the core operational skills for Kubernetes production environments:

### 🎯 Main Achievements

**Security Protection**:
- Mastered RBAC access control configuration and best practices
- Learned network policy design and implementation
- Understood Pod security standards and container security configuration
- Learned about Secret management and encryption mechanisms

**High Availability Design**:
- Learned Pod anti-affinity and node scheduling strategies
- Mastered resource quota and limit configuration methods
- Understood auto-scaling configuration and tuning
- Learned about multi-replica and failover mechanisms

**Operations Assurance**:
- Mastered etcd and application data backup strategies
- Learned monitoring system setup and configuration
- Understood log collection and analysis best practices
- Learned about failure recovery and emergency handling procedures

**Performance Optimization**:
- Learned resource configuration optimization strategies
- Mastered network performance tuning methods
- Understood storage performance optimization techniques
- Learned about cluster performance monitoring and analysis

### 🚀 Complete Learning Review

Congratulations on completing the entire Kubernetes Quick Start Tutorial! Through systematic learning across three parts:

**Part 1**: Established a solid theoretical foundation
**Part 2**: Gained practical deployment capabilities  
**Part 3**: Acquired production operations skills


Now you have a complete Kubernetes skill set, from basic concepts to production operations. It's time to apply these capabilities in real projects!

We wish you great success on your cloud-native journey! 🚀

---

## Series Index

- [Part 1: Fundamentals and Architecture](K8S_PART1_FUNDAMENTALS_EN.md)
- [Part 2: Configuration Management and Practical Applications](K8S_PART2_PRACTICE_EN.md)  
- [Part 3: Production Environment and Best Practices](K8S_PART3_PRODUCTION_EN.md)

*Thank you for completing this tutorial. If you found it helpful, please share it with others who might benefit.*