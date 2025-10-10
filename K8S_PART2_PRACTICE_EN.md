# Master Kubernetes in Hours with AI: Part 2 - Real-World Deployments Made Simple

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

## Part 2 Overview

Welcome to Part 2 of mastering Kubernetes with AI! After mastering the basic concepts and architecture, we now move into the practical phase.

This part will guide you through:
- **YAML Configuration File Writing** - Master the core skills of Kubernetes resource configuration
- **Practical Case Studies** - Learn deployment and application management through real-world scenarios

From theory to practice, from concepts to applications, this content will help you develop the ability to use Kubernetes in actual projects.

**Prerequisites**:
- Completed Part 1 and understand basic Kubernetes concepts
- Basic knowledge of YAML syntax
- Docker container usage experience

**Recommended Lab Environment**:
- Local Kubernetes environment (minikube/Docker Desktop)
- Cloud provider Kubernetes clusters
- Online Kubernetes lab environments

**Estimated Reading Time**: 3-4 hours (including hands-on practice)

---

## Table of Contents

4. [YAML Configuration Files Complete Guide](#4-yaml-configuration-files-complete-guide)
5. [Practical Case Studies](#5-practical-case-studies)

---

## 4. YAML Configuration Files Complete Guide

### 4.1 YAML Basic Syntax

**YAML Syntax Rules**:
```yaml
# 1. Indentation represents hierarchy (must use spaces, not tabs)
apiVersion: v1
kind: Pod
metadata:
  name: my-pod        # Level 2 indentation
  labels:
    app: web          # Level 3 indentation
    version: v1
spec:
  containers:
  - name: app         # First array element
    image: nginx
  - name: sidecar     # Second array element
    image: busybox

# 2. Data types
string_value: "Hello World"
number_value: 42
boolean_value: true
null_value: null
multiline_string: |
  This is a multiline string
  Preserving line breaks
folded_string: >
  This is a folded string
  Will be merged into one line

# 3. Array representation
fruits:
  - apple
  - banana
  - orange

# Or inline format
fruits: [apple, banana, orange]
```

### 4.2 K8s Resource Configuration Structure

**Standard K8s Resource Structure**:
```yaml
apiVersion: <API version>    # Required: Resource API version
kind: <Resource type>        # Required: Resource type
metadata:                    # Required: Metadata
  name: <Resource name>      # Required: Resource name
  namespace: <Namespace>     # Optional: Default is 'default'
  labels:                    # Optional: Labels
    key: value
  annotations:               # Optional: Annotations
    key: value
spec:                        # Optional: Desired state specification
  # Specific configuration depends on resource type
status:                      # Read-only: Current state (system maintained)
  # Automatically updated by K8s system
```

**Common API Version Reference**:
```yaml
# Core resources
v1:
  - Pod
  - Service
  - ConfigMap
  - Secret
  - PersistentVolume
  - PersistentVolumeClaim

# Application resources
apps/v1:
  - Deployment
  - StatefulSet
  - DaemonSet
  - ReplicaSet

# Network resources
networking.k8s.io/v1:
  - Ingress
  - NetworkPolicy

# Batch processing
batch/v1:
  - Job
batch/v1beta1:
  - CronJob

# Storage
storage.k8s.io/v1:
  - StorageClass

# Auto-scaling
autoscaling/v2:
  - HorizontalPodAutoscaler
```

### 4.3 Configuration File Best Practices

**1. File Organization Structure**:
```bash
k8s-manifests/
├── namespace.yaml           # Namespace
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

**2. Label and Selector Standards**:
```yaml
# Recommended label standards
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

# Selector in Deployment
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

**3. Resource Limit Configuration**:
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:        # Minimum resource requirements
        memory: "128Mi"
        cpu: "100m"
      limits:          # Maximum resource limits
        memory: "256Mi"
        cpu: "200m"
    livenessProbe:     # Liveness probe
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:    # Readiness probe
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**4. Configuration File Templating**:
```yaml
# Using environment variable substitution
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

### 4.4 Multi-Resource File Management

**1. Single File Multiple Resources**:
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
    # ... deployment configuration

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

**2. Kustomize Configuration Management**:
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

**3. Helm Chart Structure**:
```bash
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── templates/          # Template files
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── charts/            # Dependent Charts
```

---

## 5. Practical Case Studies

### 5.1 Full-Stack Web Application Deployment Case

This section uses a complete three-tier web application architecture to learn the practical application of Kubernetes **resource orchestration, service discovery, data persistence, configuration management** and other core concepts.

#### **Architecture Design and Kubernetes Principles**

**Application Architecture**: React frontend + Spring Boot backend + MySQL database

**Kubernetes Core Concept Mapping**:

| Application Layer | Kubernetes Resources | Core Functions | Design Principles |
|-------------------|---------------------|----------------|-------------------|
| **Frontend Layer** | Deployment + Service + Ingress | Static resource service | Stateless Pods, horizontal scaling |
| **Backend Layer** | Deployment + Service + ConfigMap | Business logic processing | Stateless Pods, load balancing |
| **Data Layer** | StatefulSet + PVC + Service | Data persistence | Stateful Pods, data binding |
| **Configuration Layer** | ConfigMap + Secret | Configuration separation | Decouple configuration from images |

**Traffic and Data Flow**:
```
Internet → Ingress(L7 Load Balancing) → Frontend Service(ClusterIP) → Frontend Pods
    ↓ API calls(Internal DNS)
Backend Service(ClusterIP) → Backend Pods → MySQL Service → MySQL Pod(StatefulSet)
    ↓ Configuration injection                    ↓ Data persistence
ConfigMap/Secret → Environment variables → PVC → Physical storage
```

#### **Kubernetes Deployment Pattern Analysis**

**1. Stateless Application Deployment Pattern** (Frontend/Backend)
- **Deployment Controller**: Ensures Pod replica count, supports rolling updates
- **Service Abstraction**: Provides stable network endpoints, automatic load balancing
- **Horizontal Scaling**: Achieve elastic scaling by adjusting replicas

**2. Stateful Application Deployment Pattern** (Database)
- **StatefulSet Controller**: Provides stable network identity and storage
- **PVC Persistence**: Decouple data from Pod lifecycle
- **Ordered Deployment**: Ensures startup order for stateful applications

**3. Configuration Management Pattern**
- **ConfigMap**: Store non-sensitive configuration, support hot updates
- **Secret**: Store sensitive information, base64 encoding, access control
- **Environment Variable Injection**: Runtime configuration injection, support dynamic configuration

#### **Deployment Framework and Best Practices**

**Namespace Isolation Strategy**:
```yaml
# Environment isolation example
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-prod
  labels:
    environment: production
    project: webapp
```

**Service Discovery Configuration**:
```yaml
# Typical configuration for backend service discovery
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 8080    # Cluster internal access port
    targetPort: 8080  # Pod actual port
```

**Configuration Management Strategy**:
```yaml
# Configuration separation example
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
  database.password: "cGFzc3dvcmQ="  # base64 encoded
```

**Storage Persistence Strategy**:
```yaml
# PVC persistence example
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

#### **Core Learning Points**

**1. Resource Orchestration Principles**:
- Deployment ensures application replica count and update strategy
- Service provides stable network abstraction and service discovery
- Ingress implements external traffic L7 routing

**2. Network Communication Mechanism**:
- Pod-to-pod communication through Service names via DNS resolution
- ClusterIP implements cluster internal load balancing
- NodePort/LoadBalancer expose external access

**3. Data Persistence Mechanism**:
- PVC provides storage abstraction, decoupled from specific storage implementation
- StatefulSet ensures data consistency for stateful applications
- StorageClass dynamically provisions storage resources

**4. Configuration Management Principles**:
- Configuration separation from images improves deployment flexibility
- Environment variables and file mounting for configuration injection
- Secret access control ensures sensitive information security

#### **Deployment Process and Operations Points**

**Deployment Order**:
1. **Base Resources** → Namespace, ConfigMap, Secret
2. **Data Layer** → Database StatefulSet and PVC
3. **Backend Layer** → Backend Deployment and Service  
4. **Frontend Layer** → Frontend Deployment and Service
5. **Gateway Layer** → Ingress configuration and external access

**Operations Monitoring**:
- **Health Checks**: readinessProbe ensures Pod receives traffic only when ready
- **Resource Limits**: requests/limits prevent resource contention
- **Log Collection**: Unified log output for troubleshooting
- **Monitoring Metrics**: Prometheus collects application and infrastructure metrics

This architecture fully demonstrates Kubernetes' **declarative management, automated operations, elastic scaling** and other core advantages, laying the foundation for subsequent microservices and advanced deployment strategies.

#### **Practical Deployment Points**

**Resource Configuration Principles**:
- **Namespace Isolation**: Separate development, staging, production environments
- **Resource Limits**: Set reasonable CPU/memory limits for each container
- **Health Checks**: Configure liveness and readiness probes to ensure service availability
- **Data Persistence**: Use StorageClass to dynamically provision storage resources

**Network Access Strategy**:
- **Internal Communication**: Service + DNS for microservice communication
- **External Access**: Ingress + LoadBalancer provides unified entry point
- **Security Control**: NetworkPolicy restricts Pod-to-Pod communication
- **TLS Termination**: Unified HTTPS certificate handling at Ingress layer

**Configuration Management Strategy**:
- **Environment Configuration**: ConfigMap stores application configuration, supports hot updates
- **Sensitive Information**: Secret stores passwords, API keys, etc.
- **Version Control**: Configuration files managed in Git for traceability
- **Access Control**: RBAC ensures only authorized users can access sensitive configuration

---

#### **Step One: Foundation Infrastructure Setup**

Let's start by setting up the basic infrastructure including namespace, configuration, and storage.

```yaml
---
# Namespace for environment isolation
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    environment: production
    type: web-application
  annotations:
    description: "Full-stack web application namespace"

---
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: webapp
  labels:
    app: webapp
    config-type: application
data:
  # Database configuration
  database.host: "mysql-service"
  database.port: "3306"
  database.name: "webapp_db"
  
  # Application configuration
  app.environment: "production"
  app.log.level: "INFO"
  app.port: "8080"
  
  # Cache configuration
  redis.host: "redis-service"
  redis.port: "6379"

---
# Secret for sensitive information
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: webapp
  labels:
    app: webapp
    secret-type: application
type: Opaque
stringData:
  # Database credentials
  database.username: "webapp_user"
  database.password: "secure_password_123"
  
  # JWT secret
  jwt.secret: "jwt_signing_key_change_in_production"
  
  # API keys
  external.api.key: "external_service_api_key"

---
# StorageClass for dynamic provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs  # Change based on your cloud provider
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
```

#### **Step Two: Database Layer Deployment**

**MySQL Database Deployment Strategy**:
- **StatefulSet**: Provides stable network identity and ordered startup
- **PVC**: Persistent data storage independent of Pod lifecycle
- **Service**: Stable DNS name for database access
- **Configuration**: Secure credential management via Secret

```yaml
---
# MySQL StatefulSet deployment
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: webapp
  labels:
    app: mysql
    component: database
    tier: data
spec:
  serviceName: mysql-service
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
        component: database
        tier: data
        version: "8.0"
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - name: mysql
          containerPort: 3306
          protocol: TCP
        
        # Environment variables from ConfigMap and Secret
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.name
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        
        # Volume mounts
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
        
        # Resource limits
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        
        # Health probes
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 1
      
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  
  # Persistent volume claim template
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi

---
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: webapp
  labels:
    app: mysql
    component: database
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: mysql
    protocol: TCP

---
# MySQL Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: webapp
data:
  my.cnf: |
    [mysqld]
    innodb_buffer_pool_size = 1G
    innodb_log_file_size = 256M
    max_connections = 200
    slow_query_log = 1
    slow_query_log_file = /var/lib/mysql/slow.log
    long_query_time = 2
    
    # Security configurations
    skip-name-resolve
    bind-address = 0.0.0.0
    
    # Character set
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
```

#### **Step Three: Backend Application Deployment**

**Spring Boot Backend Deployment Strategy**:
- **Multi-replica Design**: 3 replicas provide high availability and load distribution
- **Rolling Updates**: Zero-downtime deployment of new versions
- **Configuration Injection**: Database configuration injected via environment variables
- **Health Checks**: Ensure only ready Pods receive traffic

```yaml
---
# Spring Boot backend application deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: webapp
  labels:
    app: backend
    component: api
    tier: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Ensure service continuity
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        component: api
        tier: backend
        version: v1.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: backend
        image: myapp/backend:v1.0  # Replace with actual image
        imagePullPolicy: Always
        
        # Port configuration
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: management
          containerPort: 8081  # Spring Boot Actuator management port
          protocol: TCP
        
        # Environment variable configuration
        env:
        # Database configuration (from ConfigMap)
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host
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
        
        # Database credentials (from Secret)
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        
        # JWT configuration (from Secret)
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt.secret
        
        # Application configuration (from ConfigMap)
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
        
        # Health check configuration
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: management
          initialDelaySeconds: 90
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: management
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        startupProbe:
          httpGet:
            path: /actuator/health/readiness
            port: management
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 18  # Allow up to 3 minutes for startup
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false  # Spring Boot needs write access
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        
        # Volume mounts
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: logs-volume
          mountPath: /app/logs
      
      # Pod-level configuration
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      
      # Volume definitions
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: logs-volume
        emptyDir: {}

---
# Backend service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: webapp
  labels:
    app: backend
    component: api
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  - name: management
    port: 8081
    targetPort: management
    protocol: TCP
  sessionAffinity: None  # RESTful API is stateless
```

#### **Step Four: Frontend Presentation Layer Deployment**

**React Frontend Deployment Strategy**:
- **Static Resource Service**: Use Nginx as web server for static files
- **API Proxy**: Nginx reverse proxy to backend API, solving CORS issues
- **SPA Route Support**: Configure try_files to support React Router
- **Cache Optimization**: Reasonable static resource cache strategy

```yaml
---
# Nginx configuration file
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: webapp
  labels:
    app: frontend
    component: web
data:
  # Main configuration file
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log warn;
    pid /var/run/nginx.pid;
    
    events {
        worker_connections 1024;
        use epoll;
    }
    
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        # Log format
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';
        access_log /var/log/nginx/access.log main;
        
        # Basic settings
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        
        # Gzip compression
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
        
        include /etc/nginx/conf.d/*.conf;
    }
  
  # Virtual host configuration
  default.conf: |
    # Backend service upstream configuration
    upstream backend_servers {
        server backend-service:80;
        keepalive 32;
    }
    
    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        
        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        
        # Static resource cache configuration
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }
        
        # HTML files no cache
        location ~* \.html$ {
            expires -1;
            add_header Cache-Control "no-cache, no-store, must-revalidate";
            add_header Pragma "no-cache";
        }
        
        # API proxy configuration
        location /api/ {
            proxy_pass http://backend_servers/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }
        
        # SPA route support (React Router)
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # Deny access to hidden files
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
    }

---
# React frontend application deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
  labels:
    app: frontend
    component: web
    tier: frontend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        component: web
        tier: frontend
        version: v1.0
    spec:
      containers:
      - name: frontend
        image: myapp/frontend:v1.0  # Image with compiled React app
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        
        # Health check configuration
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Resource limits
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        
        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 101  # Nginx user ID
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        
        # Volume mounts
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
          readOnly: true
        - name: nginx-cache
          mountPath: /var/cache/nginx
        - name: nginx-run
          mountPath: /var/run
        - name: nginx-log
          mountPath: /var/log/nginx
      
      # Volume definitions
      volumes:
      - name: nginx-config
        configMap:
          name: frontend-config
          defaultMode: 0644
      - name: nginx-cache
        emptyDir: {}
      - name: nginx-run
        emptyDir: {}
      - name: nginx-log
        emptyDir: {}

---
# Frontend service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: webapp
  labels:
    app: frontend
    component: web
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  sessionAffinity: None
```

#### **Step Five: External Access Configuration**

**Ingress Configuration Strategy**:
- **Domain Routing**: Route traffic to different services based on domain
- **SSL Termination**: Terminate SSL at Ingress layer, backend uses HTTP
- **Path Rewriting**: Support different URL path rules
- **Load Balancing**: Automatically distribute traffic among multiple Pods

```yaml
---
# Web application external access configuration
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: webapp
  labels:
    app: webapp
  annotations:
    # Nginx Ingress Controller configuration
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    
    # SSL certificate management (using cert-manager)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    cert-manager.io/acme-challenge-type: http01
    
    # Security configuration
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
spec:
  # TLS configuration
  tls:
  - hosts:
    - webapp.example.com
    secretName: webapp-tls
  
  # Routing rules
  rules:
  - host: webapp.example.com
    http:
      paths:
      # Frontend static resource routing
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      
      # API routing (optional if frontend doesn't proxy API)
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

**Frontend Deployment Summary**:
1. **Nginx Optimization**: Reasonable Gzip compression and cache strategy for performance
2. **SPA Support**: Configure try_files to support frontend routing
3. **API Proxy**: Solve CORS issues with unified domain access
4. **Security Configuration**: Add security headers to protect against common web attacks

This comprehensive deployment case demonstrates the practical application of Kubernetes core concepts in a real-world scenario, providing a solid foundation for understanding container orchestration in production environments.

### 5.2 Microservices Architecture Deployment Case

This section focuses on the practical application of Kubernetes in **service governance, configuration management, elastic scaling, and fault isolation** through an e-commerce platform microservices architecture.

#### **Microservices Architecture and Kubernetes Principles**

**Business Architecture**: User Service + Order Service + Payment Service + Gateway Service

**Kubernetes Microservices Governance Model**:

| Governance Dimension | Kubernetes Implementation | Core Value | Technical Principles |
|---------------------|---------------------------|------------|---------------------|
| **Service Discovery** | Service + DNS | Dynamic service location | Built-in DNS resolution, automatic endpoint updates |
| **Load Balancing** | Service ClusterIP | Traffic distribution | kube-proxy implements L4 load balancing |
| **Configuration Management** | ConfigMap + Secret | Configuration and code separation | Environment variable/file mount injection |
| **Elastic Scaling** | HPA + VPA | Auto-scaling | Metrics-based intelligent scheduling |
| **Fault Isolation** | Namespace + NetworkPolicy | Blast radius control | Resource isolation + network policies |
| **Version Management** | Deployment | Rolling updates | Progressive release, zero-downtime deployment |

#### **Step One: Kubernetes Namespace Isolation and Configuration Management**

**K8s Learning Focus**: Namespace resource isolation, ConfigMap configuration separation, Secret sensitive information management

```yaml
---
# Namespace isolation (Kubernetes multi-tenancy foundation)
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    environment: production
    type: microservices
  annotations:
    description: "Microservices architecture demonstration namespace"

---
# ConfigMap configuration externalization (Kubernetes configuration management core concept)
apiVersion: v1
kind: ConfigMap
metadata:
  name: shared-config
  namespace: microservices
  labels:
    config-type: shared
data:
  # Basic service addresses (demonstrate Service discovery)
  database_host: "mysql-service.microservices.svc.cluster.local"
  cache_host: "redis-service.microservices.svc.cluster.local"
  message_queue_host: "rabbitmq-service.microservices.svc.cluster.local"
  
  # Application configuration (demonstrate environment variable injection)
  log_level: "INFO"
  app_env: "production"
  service_port: "8080"
  metrics_port: "9090"

---
# Secret sensitive information management (Kubernetes security best practices)
apiVersion: v1
kind: Secret
metadata:
  name: shared-secrets
  namespace: microservices
  labels:
    secret-type: shared
type: Opaque
stringData:
  # Database credentials (demonstrate Secret mounting)
  db_username: "microservice_user"
  db_password: "secure_password_123"
  
  # API keys (demonstrate sensitive information protection)
  jwt_secret: "jwt_signing_key_256_bits"
  api_key: "external_service_api_key"
```

#### **Step Two: Service Network Communication and Load Balancing**

**K8s Learning Focus**: Service types, service discovery, load balancing algorithms, endpoint management

```yaml
---
# API Gateway service (demonstrate ClusterIP and service discovery)
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: microservices
  labels:
    app: api-gateway
    role: gateway
spec:
  type: ClusterIP
  selector:
    app: api-gateway
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  sessionAffinity: None

---
# User service (demonstrate microservice Service pattern)
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
    tier: backend
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
  - name: http
    port: 8080
    targetPort: http
  - name: metrics
    port: 9090
    targetPort: metrics

---
# Order service (demonstrate load balancing and failover)
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
    status: healthy  # Additional selector: only route to healthy instances
  ports:
  - name: http
    port: 8080
    targetPort: http
  sessionAffinity: ClientIP  # Client IP affinity (session persistence)
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

#### **Step Three: Deployment and Pod Management Practice**

**K8s Learning Focus**: Rolling updates, health checks, resource limits, environment variable injection

```yaml
---
# User service Deployment (demonstrate microservice standard deployment pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
    tier: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        status: healthy
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: user-service
        image: myapp/user-service:v1.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        
        # Environment variable injection (demonstrate ConfigMap and Secret usage)
        env:
        - name: APP_ENV
          value: "production"
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: shared-config
              key: database_host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: shared-secrets
              key: db_password
        
        # Resource limits (demonstrate resource management)
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Health check configuration (Kubernetes health management core)
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 6

---
# Order service (demonstrate different configuration strategies)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
  labels:
    app: order-service
    tier: backend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        status: healthy
    spec:
      # Node scheduling constraints (demonstrate Pod scheduling control)
      nodeSelector:
        workload-type: compute
      
      containers:
      - name: order-service
        image: myapp/order-service:v1.0
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        
        # Template environment variable configuration
        envFrom:
        - configMapRef:
            name: shared-config
        - secretRef:
            name: shared-secrets
        
        env:
        - name: SERVICE_NAME
          value: "order-service"
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # Stricter resource control
        resources:
          requests:
            memory: "512Mi"
            cpu: "300m"
          limits:
            memory: "1Gi"
            cpu: "800m"
        
        # Optimized health checks
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 45
          periodSeconds: 30
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
```

#### **Step Four: HPA Auto-scaling Practice**

**K8s Learning Focus**: Horizontal scaling, metrics monitoring, scaling strategy, resource utilization

```yaml
---
# User service HPA (based on CPU and memory)
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
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  
  # Scaling metrics configuration
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
        
  - type: Resource  
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Scaling behavior control
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60

---
# Order service HPA (based on custom metrics)
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
  minReplicas: 1
  maxReplicas: 8
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
        
  # Custom metrics (demonstrate business metric scaling)
  - type: Pods
    pods:
      metric:
        name: pending_orders
      target:
        type: AverageValue
        averageValue: "100"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods  
        value: 3
        periodSeconds: 30
```

**Microservices Deployment Summary**:
1. **Dependency Management**: Use initContainers to wait for dependent services to start
2. **Configuration Injection**: Environment variable method for configuration injection, supports dynamic updates
3. **Health Checks**: Three types of probes ensure service availability
4. **Resource Management**: Reasonable CPU/memory limit configuration
5. **Security Configuration**: Run as non-root user, minimal privilege principle
6. **Monitoring Integration**: Prometheus metrics, distributed tracing configuration

---

### 5.3 Kubernetes Deployment Strategy Cases

This section focuses on practical applications of Kubernetes core deployment strategies, learning **rolling updates, blue-green deployment, canary releases, version rollback** and other cloud-native deployment patterns.

**Practice Focus**:
- **Rolling Updates**: Standard approach for zero-downtime deployment
- **Blue-Green Deployment**: Fast switching and fast rollback
- **Canary Release**: Progressive risk control
- **Version Management**: Deployment history and rollback mechanisms

#### **Step One: Rolling Update Practice**

**K8s Learning Focus**: `strategy` configuration, `maxUnavailable`/`maxSurge` parameters, health check integration

```yaml
---
# Basic web application - demonstrate rolling update mechanism
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-rolling
  namespace: production
  labels:
    app: web-app-rolling
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%  # Maximum 25% Pods unavailable
      maxSurge: 25%        # Maximum 25% extra Pods
  selector:
    matchLabels:
      app: web-app-rolling
  template:
    metadata:
      labels:
        app: web-app-rolling
        version: v1.0
    spec:
      containers:
      - name: web-app
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # Health checks (key for rolling updates)
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10

---
# Service configuration (traffic routing)
apiVersion: v1
kind: Service
metadata:
  name: web-app-rolling-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web-app-rolling
  ports:
  - port: 80
    targetPort: 80
```

**Rolling Update Operation Demo**:
```bash
# 1. Deploy initial version
kubectl apply -f rolling-update.yaml

# 2. Observe initial state
kubectl get pods -l app=web-app-rolling -w

# 3. Execute rolling update (image version upgrade)
kubectl set image deployment/web-app-rolling web-app=nginx:1.21 -n production

# 4. Monitor update process in real-time
kubectl rollout status deployment/web-app-rolling -n production

# 5. View update history
kubectl rollout history deployment/web-app-rolling -n production
```

#### **Step Two: Blue-Green Deployment Practice**

**K8s Learning Focus**: Label selector switching, Service routing, environment isolation

```yaml
---
# Blue environment (current production version)
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
        image: myapp:v1.0
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
# Green environment (new version)
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
        image: myapp:v2.0
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
# Main service (core of traffic switching)
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
    version: blue  # Initially points to blue environment
  ports:
  - port: 80
    targetPort: 8080
```

**Blue-Green Deployment Switch Demo**:
```bash
# 1. Deploy blue-green environments
kubectl apply -f blue-green.yaml

# 2. Verify blue environment running normally
kubectl get pods -l version=blue -n production

# 3. Deploy green environment (new version)
kubectl get pods -l version=green -n production

# 4. Test green environment (using temporary Service)
kubectl port-forward deployment/web-app-green 8080:8080 -n production

# 5. Switch to green environment (critical step)
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"green"}}}'

# 6. Rollback to blue environment if needed
kubectl patch service web-app-service -n production -p '{"spec":{"selector":{"version":"blue"}}}'
```

#### **Step Three: Canary Release Practice**

**K8s Learning Focus**: Traffic weight control, progressive deployment, replica-based traffic allocation

```yaml
---
# Stable version (main traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-stable
  namespace: production
  labels:
    app: web-app
    track: stable
spec:
  replicas: 9  # 90% traffic (9/10)
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
        image: myapp:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "v1.0-stable"

---
# Canary version (small amount of traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-canary
  namespace: production
  labels:
    app: web-app
    track: canary
spec:
  replicas: 1  # 10% traffic (1/10)
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
        image: myapp:v2.0
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "v2.0-canary"

---
# Unified Service (automatic traffic allocation)
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web-app  # Select both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

**Canary Release Process Demo**:
```bash
# 1. Deploy stable version
kubectl apply -f canary-stable.yaml

# 2. Deploy canary version (10% traffic)
kubectl apply -f canary-deploy.yaml

# 3. Observe traffic allocation
kubectl get pods -l app=web-app -n production

# 4. Gradually increase canary traffic (20%)
kubectl scale deployment web-app-canary --replicas=2 -n production
kubectl scale deployment web-app-stable --replicas=8 -n production

# 5. Continue increasing to 50%
kubectl scale deployment web-app-canary --replicas=5 -n production
kubectl scale deployment web-app-stable --replicas=5 -n production

# 6. Complete switch to new version
kubectl scale deployment web-app-stable --replicas=0 -n production
kubectl scale deployment web-app-canary --replicas=10 -n production
```

#### **Step Four: Version Rollback Mechanism**

**K8s Learning Focus**: Deployment history management, `revisionHistoryLimit`, rollback commands

```yaml
---
# Deployment configuration supporting version rollback
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-versioned
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 5
  revisionHistoryLimit: 10  # Keep 10 historical versions
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
        # Version annotations (for tracking)
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

**Version Rollback Operation Demo**:
```bash
# 1. View current state
kubectl get deployment web-app-versioned -n production

# 2. Execute version update
kubectl set image deployment/web-app-versioned web-app=myapp:v2.0 -n production --record

# 3. Update again to v3.0
kubectl set image deployment/web-app-versioned web-app=myapp:v3.0 -n production --record

# 4. View version history
kubectl rollout history deployment/web-app-versioned -n production

# 5. View specific version details
kubectl rollout history deployment/web-app-versioned --revision=2 -n production

# 6. Rollback to previous version
kubectl rollout undo deployment/web-app-versioned -n production

# 7. Rollback to specific version
kubectl rollout undo deployment/web-app-versioned --to-revision=1 -n production

# 8. Confirm rollback status
kubectl rollout status deployment/web-app-versioned -n production
```

**Deployment Strategy Comparison Summary**:

| Strategy | Advantages | Use Cases | K8s Implementation |
|----------|------------|-----------|-------------------|
| **Rolling Update** | Zero downtime, resource efficient, automated | Daily version updates | Deployment default strategy |
| **Blue-Green Deployment** | Fast switching, complete rollback, environment isolation | Major version releases | Label selector switching |
| **Canary Release** | Risk control, progressive validation, user feedback | Uncertain version impact | Replica count controls traffic ratio |
| **Version Rollback** | Fast recovery, history tracking, safety guarantee | Failure recovery | Deployment history mechanism |

---

### 5.4 Kubernetes Monitoring Integration Cases

This section focuses on practical applications of Kubernetes native monitoring capabilities, learning **Prometheus integration, metrics exposure, ServiceMonitor, alerting rules** and other cloud-native monitoring patterns.

**Practice Focus**:
- **Annotation Monitoring**: Use Kubernetes annotations to automatically configure monitoring
- **ServiceMonitor**: Declarative monitoring with Prometheus Operator
- **Sidecar Pattern**: Containerized deployment of monitoring agents
- **Metrics Standardization**: Cloud-native monitoring metrics standards

#### **Step One: Pod Annotation Monitoring Practice**

**K8s Learning Focus**: `prometheus.io/*` annotations, automatic service discovery, port configuration

```yaml
---
# Web application with built-in monitoring
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
        # Prometheus auto-discovery annotations (K8s standard)
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        prometheus.io/scheme: "http"
    spec:
      containers:
      - name: web-app
        image: myapp/web-with-metrics:v1.0
        ports:
        - name: http
          containerPort: 8080
        - name: metrics  # Monitoring port (critical)
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
        # Health checks include monitoring endpoints
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
        livenessProbe:
          httpGet:
            path: /metrics  # Monitoring endpoint also as health check
            port: 9090
          initialDelaySeconds: 15

---
# Service configuration (monitoring discovery)
apiVersion: v1
kind: Service
metadata:
  name: monitored-web-app-service
  namespace: production
  labels:
    app: monitored-web-app
  annotations:
    # Service-level monitoring annotations
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  type: ClusterIP
  selector:
    app: monitored-web-app
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: metrics  # Monitoring port (important)
    port: 9090
    targetPort: metrics
```

#### **Step Two: ServiceMonitor CRD Practice**

**K8s Learning Focus**: Custom resources, Operator pattern, declarative monitoring configuration

```yaml
---
# ServiceMonitor (Prometheus Operator approach)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-monitor
  namespace: production
  labels:
    app: web-app-monitor
    prometheus: kube-prometheus
spec:
  # Service selector (select Services to monitor)
  selector:
    matchLabels:
      app: monitored-web-app
  # Monitoring endpoint configuration
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
    # Metric relabeling (enhance labels)
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'http_requests_total'
      targetLabel: 'custom_metric'
      replacement: 'web_requests'
  # Namespace selector
  namespaceSelector:
    matchNames:
    - production

---
# PrometheusRule (alerting rules CRD)
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
    # High error rate alert
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
    
    # High latency alert
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
    
    # Pod restart alert
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

#### **Step Three: Sidecar Monitoring Pattern Practice**

**K8s Learning Focus**: Multi-container Pod, inter-container communication, shared storage, monitoring agents

```yaml
---
# Sidecar monitoring pattern Deployment
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
        # Sidecar monitoring annotations
        prometheus.io/scrape: "true"
        prometheus.io/port: "9102"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      # Main application container (no modification needed)
      - name: main-app
        image: myapp/legacy-app:v1.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
        - name: app-metrics
          mountPath: /var/metrics
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
      
      # Monitoring sidecar container
      - name: metrics-exporter
        image: prom/node-exporter:latest
        ports:
        - name: metrics
          containerPort: 9102
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: app-metrics
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
      
      # Log collection sidecar (optional)
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
      - name: app-logs
        emptyDir: {}
      - name: app-metrics
        emptyDir: {}
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config

---
# Fluent Bit configuration
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

#### **Step Four: Monitoring Metrics Standardization**

**K8s Learning Focus**: Standard metrics, label specifications, monitoring best practices

```yaml
---
# Standardized monitoring configuration example
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-standards
  namespace: production
data:
  # Standard metrics specification
  metrics-spec.yaml: |
    # Application-level metrics
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
    
    # Business-level metrics
    business_metrics:
      - name: orders_total
        type: counter
        labels: [status, region]
        description: "Total orders processed"
      
      - name: revenue_total
        type: counter
        labels: [currency, region]
        description: "Total revenue"
    
    # Infrastructure metrics
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
# Monitoring label standardization
apiVersion: apps/v1
kind: Deployment
metadata:
  name: standardized-monitored-app
  namespace: production
  labels:
    # Standardized labels
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
        # Inherit standardized labels
        app.kubernetes.io/name: web-app
        app.kubernetes.io/instance: web-app-prod
        app.kubernetes.io/version: "v1.0"
        app.kubernetes.io/component: frontend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        # Monitoring metadata annotations
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
        # Standardized environment variables
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

**Monitoring Integration Best Practices Summary**:

1. **Annotation-Driven**: Use standard Prometheus annotations for auto-discovery
2. **Label Standards**: Follow Kubernetes recommended labels and monitoring label standards
3. **Port Standardization**: Separate business ports, monitoring ports, and health check ports
4. **Metrics Standardization**: Define unified metric naming and label standards
5. **Alert Hierarchy**: Three-tier alert system: critical, warning, info
6. **Sidecar Pattern**: Non-invasive monitoring transformation for legacy applications

---

## Part 2 Summary

Through this practical learning, you have mastered the core skills of Kubernetes application deployment and management:

### 🎯 Main Achievements

**Configuration Management**:
- Mastered YAML syntax and Kubernetes resource configuration structure
- Learned ConfigMap, Secret and other configuration externalization best practices
- Understood the use cases of namespaces, labels, and annotations

**Practical Deployment**:
- Completed full-stack web application deployment (frontend + backend + database)
- Mastered microservices architecture Service discovery and communication patterns
- Learned deployment strategies and optimization methods for different types of applications

**Deployment Strategies**:
- Understood the applicable scenarios of rolling updates, blue-green deployment, canary releases
- Mastered the operation methods of version management and rollback mechanisms
- Learned deployment strategy implementation based on Kubernetes native capabilities

**Monitoring Integration**:
- Mastered Prometheus annotation-driven monitoring configuration
- Learned ServiceMonitor CRD declarative monitoring
- Understood the Sidecar pattern monitoring agent deployment approach

### 🚀 Next Learning Suggestions

Now you have practical Kubernetes application deployment capabilities. We recommend:

1. **Deepen Practice** - Apply learned skills in real projects
2. **Production Preparation** - Learn Part 3 production environment best practices
3. **Advanced Tools** - Learn Helm, Kustomize and other advanced tools
4. **Complete Monitoring** - Build a complete observability system

### 📚 Related Resources

- **Continue Reading**: [Part 3: Production Environment and Best Practices](K8S_PART3_PRODUCTION_EN.md)
- **Official Documentation**: [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- **Practice Environment**: Recommended use of Kind, Minikube or cloud provider managed Kubernetes services

---

## Series Index

- [Part 1: Fundamentals and Architecture](K8S_PART1_FUNDAMENTALS_EN.md)
- [Part 2: Configuration Management and Practical Applications](K8S_PART2_PRACTICE_EN.md)  
- [Part 3: Production Environment and Best Practices](K8S_PART3_PRODUCTION_EN.md)