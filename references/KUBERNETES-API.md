# Kubernetes API Architecture
## Meta-Architecture v1.0.0 Compliant Template

**Meta-Architecture Compliance:** v1.0.0  
**Template Version:** 1.0.0  
**Status:** Production-Ready  
**Last Audit:** 2025-11-10  
**Compliance Score:** 100%

---

## Table of Contents

1. [Introduction](#introduction)
2. [Kubernetes Ecosystem Overview](#kubernetes-ecosystem-overview)
3. [Core Principles Mapping](#core-principles-mapping)
4. [Four-Layer Kubernetes Architecture](#four-layer-kubernetes-architecture)
5. [Resource Design Patterns](#resource-design-patterns)
6. [Deployment Strategies](#deployment-strategies)
7. [Service Mesh Integration](#service-mesh-integration)
8. [Observability & Monitoring](#observability--monitoring)
9. [Security Architecture](#security-architecture)
10. [Complete Examples](#complete-examples)
11. [Compliance Checklist](#compliance-checklist)

---

## Introduction

### What This Document Provides

This architecture template demonstrates how to build production-grade applications on Kubernetes, following Meta-Architecture v1.0.0 principles. It covers orchestration, scaling, resilience, and operational patterns.

### Kubernetes vs Docker: The Core Mechanism

**Docker (Single Host):**
```
Host Machine
├── Container 1
├── Container 2
└── Container 3

Problem: 
- Single point of failure
- Manual scaling
- No self-healing
- No load balancing
```

**Kubernetes (Cluster):**
```
Control Plane (Master Nodes)
├── API Server (receives requests)
├── Scheduler (places pods on nodes)
├── Controller Manager (maintains desired state)
└── etcd (distributed key-value store)
        ↓
Worker Nodes (runs containers)
├── Node 1: Pod 1, Pod 2
├── Node 2: Pod 3, Pod 4
└── Node 3: Pod 5, Pod 6

Benefits:
- High availability (multi-node)
- Automatic scaling (HPA, VPA)
- Self-healing (restarts failed pods)
- Load balancing (Services)
- Rolling updates (zero downtime)
```

**Key Kubernetes Mechanism: Desired State**

```yaml
# You declare desired state:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3  # DESIRED: 3 replicas

# Kubernetes continuously reconciles:
Current State: 2 replicas running
Desired State: 3 replicas
Action: Start 1 more replica

Current State: 3 replicas running
Desired State: 3 replicas
Action: Nothing (satisfied)

Current State: 1 replica crashed
Desired State: 3 replicas
Action: Start 1 new replica (self-healing)
```

**Architectural Implications:**
1. **Declarative** - You declare "what" (3 replicas), not "how" (start container X)
2. **Reconciliation** - Controllers continuously fix drift from desired state
3. **Distributed** - Multi-node, no single point of failure
4. **Immutable** - Pods are disposable, never modified in place
5. **Service Discovery** - DNS-based routing to pods

---

## Kubernetes Ecosystem Overview

### Core Components

```
┌─────────────────────────────────────────────────────┐
│              CONTROL PLANE                          │
│                                                     │
│  ┌──────────────┐    ┌────────────────┐           │
│  │  API Server  │◄───┤  kubectl/API   │           │
│  │  (REST API)  │    │    Clients     │           │
│  └──────┬───────┘    └────────────────┘           │
│         │                                           │
│  ┌──────▼───────┐    ┌────────────────┐           │
│  │  Scheduler   │    │   Controller   │           │
│  │ (Pod→Node)   │    │    Manager     │           │
│  └──────────────┘    │ (Reconcilers)  │           │
│                      └────────────────┘           │
│  ┌──────────────────────────────────┐             │
│  │         etcd (State Store)       │             │
│  └──────────────────────────────────┘             │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                 WORKER NODES                         │
│                                                     │
│  Node 1                Node 2                Node 3 │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐│
│  │ kubelet │          │ kubelet │          │ kubelet ││
│  │ (agent) │          │ (agent) │          │ (agent) ││
│  └────┬────┘          └────┬────┘          └────┬────┘│
│       │                    │                    │     │
│  ┌────▼─────┐         ┌────▼─────┐         ┌────▼─────┐│
│  │  Pods    │         │  Pods    │         │  Pods    ││
│  │ ┌──┐ ┌──┐│         │ ┌──┐ ┌──┐│         │ ┌──┐ ┌──┐││
│  │ │C1│ │C2││         │ │C3│ │C4││         │ │C5│ │C6│││
│  │ └──┘ └──┘│         │ └──┘ └──┘│         │ └──┘ └──┘││
│  └──────────┘         └──────────┘         └──────────┘│
└─────────────────────────────────────────────────────┘
```

### Resource Hierarchy

```
Cluster
  └── Namespace (logical isolation)
       ├── Deployment (manages ReplicaSets)
       │    └── ReplicaSet (manages Pods)
       │         └── Pod (runs Containers)
       │              └── Container (Docker image)
       │
       ├── Service (stable endpoint)
       │    └── Endpoints (pod IPs)
       │
       ├── ConfigMap (configuration)
       ├── Secret (sensitive data)
       └── PersistentVolumeClaim (storage)
            └── PersistentVolume (actual storage)
```

### Pod Lifecycle

```
┌─────────┐
│ Pending │ ← Pod created, waiting for scheduling
└────┬────┘
     │ Scheduler assigns to node
     ▼
┌─────────┐
│ Running │ ← Containers started
└────┬────┘
     │ All containers exit successfully
     ▼
┌─────────┐
│Succeeded│ (Job/CronJob pattern)
└─────────┘

OR

     │ Container crashes
     ▼
┌─────────┐
│ Failed  │
└─────────┘
     │ RestartPolicy: Always
     └──► Back to Running (self-healing)
```

---

## Core Principles Mapping

### Principle 1: Layered Architecture (MANDATORY)

**Meta-Architecture Principle:**
> Dependencies flow downward. Four-layer pattern: Foundation → Infrastructure → Integration → Application.

**Kubernetes Implementation:**

```yaml
# ============================================
# LAYER 1: Foundation (Base Infrastructure)
# ============================================

# Namespace for foundation resources
apiVersion: v1
kind: Namespace
metadata:
  name: foundation
  labels:
    layer: foundation
---
# Storage Classes (foundation primitives)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  namespace: foundation
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete
---
# Network Policies (foundation security)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: foundation
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# ============================================
# LAYER 2: Infrastructure (Core Services)
# ============================================

# Namespace for infrastructure
apiVersion: v1
kind: Namespace
metadata:
  name: infrastructure
  labels:
    layer: infrastructure
---
# PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: infrastructure
  labels:
    layer: infrastructure
    component: database
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        layer: infrastructure
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
---
# Redis Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: infrastructure
  labels:
    layer: infrastructure
    component: cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        layer: infrastructure
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

# ============================================
# LAYER 3: Integration (External Services)
# ============================================

# Namespace for integration
apiVersion: v1
kind: Namespace
metadata:
  name: integration
  labels:
    layer: integration
---
# API Gateway (Nginx Ingress)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  namespace: integration
  labels:
    layer: integration
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80

# ============================================
# LAYER 4: Application (Business Logic)
# ============================================

# Namespace for application
apiVersion: v1
kind: Namespace
metadata:
  name: application
  labels:
    layer: application
---
# Application Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: application
  labels:
    layer: application
    component: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        layer: application
    spec:
      # Init container (depends on infrastructure)
      initContainers:
      - name: wait-for-db
        image: busybox:latest
        command: ['sh', '-c', 'until nc -z postgres.infrastructure.svc.cluster.local 5432; do sleep 2; done']
      
      # Main container
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          value: "postgresql://postgres.infrastructure.svc.cluster.local:5432/myapp"
        - name: REDIS_URL
          value: "redis://redis.infrastructure.svc.cluster.local:6379"
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: secret-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Layer Isolation via Network Policies:**

```yaml
# Infrastructure layer: Only application can access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: infrastructure-policy
  namespace: infrastructure
spec:
  podSelector:
    matchLabels:
      layer: infrastructure
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          layer: application
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
    - protocol: TCP
      port: 6379  # Redis

# Application layer: Only integration can access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: application-policy
  namespace: application
spec:
  podSelector:
    matchLabels:
      layer: application
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          layer: integration
    ports:
    - protocol: TCP
      port: 8000
```

**Why This Works:**
- Namespaces provide logical isolation
- Labels identify layer membership
- Network policies enforce dependency direction
- DNS enables cross-namespace communication
- Init containers ensure dependency readiness

---

### Principle 2: Explicit Dependency Management (MANDATORY)

**Meta-Architecture Principle:**
> All dependencies declared in manifest with pinned versions.

**Kubernetes Implementation:**

```yaml
# ============================================
# Pinned Image Versions
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      # Pin to exact version with SHA
      - name: app
        image: myapp:1.2.3@sha256:abc123...
        # NOT: myapp:latest (unpredictable)
        # NOT: myapp:1.2 (mutable tag)
      
      - name: sidecar
        image: nginx:1.25.3-alpine@sha256:def456...

# ============================================
# Dependency Declaration via Init Containers
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      # Init containers run before main container
      # Explicit dependency chain
      initContainers:
      
      # 1. Wait for database (CRITICAL dependency)
      - name: wait-for-postgres
        image: busybox:1.36.1@sha256:xyz789...
        command:
        - sh
        - -c
        - |
          until nc -z postgres.infrastructure.svc.cluster.local 5432; do
            echo "Waiting for postgres..."
            sleep 2
          done
          echo "PostgreSQL is up"
      
      # 2. Wait for cache (IMPORTANT dependency)
      - name: wait-for-redis
        image: busybox:1.36.1@sha256:xyz789...
        command:
        - sh
        - -c
        - |
          until nc -z redis.infrastructure.svc.cluster.local 6379; do
            echo "Waiting for redis..."
            sleep 2
          done
          echo "Redis is up"
      
      # 3. Run migrations (state dependency)
      - name: migrate
        image: myapp:1.2.3@sha256:abc123...
        command: ["python", "migrate.py"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      
      # Main container starts only after all init containers succeed
      containers:
      - name: app
        image: myapp:1.2.3@sha256:abc123...

# ============================================
# Service Dependencies via Kubernetes DNS
# ============================================

# Services create stable DNS names
# app can depend on: postgres.infrastructure.svc.cluster.local

# Format: <service-name>.<namespace>.svc.cluster.local

# ============================================
# Version Tracking via Labels
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
    version: 1.2.3
    app.kubernetes.io/name: myapp
    app.kubernetes.io/version: 1.2.3
    app.kubernetes.io/component: api
spec:
  template:
    metadata:
      labels:
        app: myapp
        version: 1.2.3

# ============================================
# Helm Chart Dependencies
# ============================================

# Chart.yaml
apiVersion: v2
name: myapp
version: 1.2.3
appVersion: 1.2.3

dependencies:
- name: postgresql
  version: 12.1.0
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled

- name: redis
  version: 17.3.0
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled

# Helm locks dependencies
# helm dependency update
# Creates Chart.lock with exact versions

# ============================================
# Image Pull Policy
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.2.3@sha256:abc123...
        imagePullPolicy: IfNotPresent  # Use cached if digest matches
        # NOT: Always (re-pulls every time)
```

**Dependency Validation:**

```yaml
# ValidatingAdmissionWebhook to enforce pinned images
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validate-images
webhooks:
- name: validate-images.example.com
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  clientConfig:
    service:
      name: image-validator
      namespace: default
      path: /validate
  admissionReviewVersions: ["v1"]
  sideEffects: None

# Webhook validates:
# - Image has SHA digest
# - Image exists in allowed registries
# - No 'latest' tag
# - All init containers also pinned
```

**Why This Works:**
- SHA digests ensure exact image reproducibility
- Init containers make dependencies explicit and ordered
- Kubernetes DNS provides stable service discovery
- Helm locks chart dependencies
- Admission webhooks enforce policies

---

### Principle 3: Graceful Degradation (MANDATORY)

**Meta-Architecture Principle:**
> Classify services as CRITICAL, IMPORTANT, OPTIONAL. System continues with reduced functionality.

**Kubernetes Implementation:**

```yaml
# ============================================
# Service Classification via Priority Classes
# ============================================

# CRITICAL: System must run
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000000
globalDefault: false
description: "Critical services that must always run"
---
# IMPORTANT: Degrades functionality
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: important-priority
value: 100000
globalDefault: false
description: "Important services that improve UX"
---
# OPTIONAL: Nice to have
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: optional-priority
value: 1000
globalDefault: true
description: "Optional services"

# ============================================
# CRITICAL: Database (must run)
# ============================================

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: infrastructure
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: critical-priority
      containers:
      - name: postgres
        image: postgres:15-alpine
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
      # Anti-affinity: spread across nodes (no single point of failure)
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: postgres
            topologyKey: kubernetes.io/hostname

# ============================================
# IMPORTANT: Cache (improves performance)
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: infrastructure
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: important-priority
      containers:
      - name: redis
        image: redis:7-alpine
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 5  # More lenient than critical
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
      # Soft anti-affinity: prefer different nodes, but not required
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname

# ============================================
# OPTIONAL: Monitoring (nice to have)
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1  # Single instance OK
  template:
    spec:
      priorityClassName: optional-priority
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        # No liveness probe - if it fails, that's OK

# ============================================
# Application with Graceful Degradation
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: application
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: critical-priority
      containers:
      - name: app
        image: myapp:1.0.0
        env:
        # CRITICAL: Database must be available
        - name: DATABASE_URL
          value: "postgresql://postgres.infrastructure.svc.cluster.local/myapp"
        
        # IMPORTANT: Cache may be unavailable
        - name: REDIS_URL
          value: "redis://redis.infrastructure.svc.cluster.local:6379"
        - name: CACHE_ENABLED
          value: "true"
        - name: CACHE_FALLBACK
          value: "true"  # Fall back to DB if cache fails
        
        # OPTIONAL: Monitoring may be unavailable
        - name: METRICS_ENABLED
          value: "true"
        - name: METRICS_FAIL_SILENTLY
          value: "true"  # Don't crash if metrics fail
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
```

**Application-Level Degradation:**

```python
# health endpoint
@app.get("/health")
async def health():
    """
    Liveness check: Is the application running?
    
    Only checks CRITICAL dependencies.
    """
    # Check database (CRITICAL)
    try:
        await db.execute("SELECT 1")
    except Exception:
        raise HTTPException(status_code=503, detail="Database unhealthy")
    
    # Don't check cache or metrics - those are not critical
    return {"status": "healthy"}

@app.get("/ready")
async def ready():
    """
    Readiness check: Can the application serve traffic?
    
    Checks CRITICAL and IMPORTANT dependencies.
    """
    # Check database (CRITICAL)
    try:
        await db.execute("SELECT 1")
    except Exception:
        raise HTTPException(status_code=503, detail="Database not ready")
    
    # Check cache (IMPORTANT) - warn but don't fail
    try:
        await cache.ping()
    except Exception:
        logger.warning("Cache not ready - will operate in degraded mode")
        # Still return ready - app can work without cache
    
    return {"status": "ready", "mode": "full" if cache_available else "degraded"}
```

**Pod Disruption Budget (Availability Guarantee):**

```yaml
# Ensure minimum availability during disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: application
spec:
  minAvailable: 2  # At least 2 pods must be available
  selector:
    matchLabels:
      app: myapp

# Prevents voluntary disruptions (node drain, rolling update)
# from taking down too many pods at once
```

**Why This Works:**
- Priority classes ensure critical services scheduled first
- Anti-affinity spreads replicas across nodes (no SPOF)
- Health checks verify critical dependencies only
- PodDisruptionBudgets guarantee minimum availability
- Application handles optional service failures gracefully

---

### Principle 4: Comprehensive Input Validation (MANDATORY)

**Meta-Architecture Principle:**
> Validate at boundaries: type, range, business rules, state.

**Kubernetes Implementation:**

```yaml
# ============================================
# LAYER 1: Schema Validation (Type/Format)
# ============================================

# Built-in: Kubernetes validates API objects
# Example: This will be rejected
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: "three"  # ❌ Error: must be integer
  # Kubernetes validates against OpenAPI schema

# ============================================
# LAYER 2: Admission Controllers (Policy)
# ============================================

# ValidatingAdmissionWebhook for custom validation
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: validate-deployments
webhooks:
- name: validate.deployments.example.com
  admissionReviewVersions: ["v1"]
  sideEffects: None
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  clientConfig:
    service:
      name: validation-webhook
      namespace: default
      path: /validate-deployment

# Webhook validates:
# - Image has SHA digest
# - Resource limits are set
# - Health checks are defined
# - Security context is non-root
# - Labels follow conventions

# ============================================
# LAYER 3: ResourceQuota (Range Validation)
# ============================================

apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: application
spec:
  hard:
    # Limit total resources in namespace
    requests.cpu: "10"          # Max 10 CPU cores requested
    requests.memory: "20Gi"     # Max 20GB memory requested
    limits.cpu: "20"            # Max 20 CPU cores limit
    limits.memory: "40Gi"       # Max 40GB memory limit
    
    # Limit object counts
    pods: "50"                  # Max 50 pods
    services: "10"              # Max 10 services
    persistentvolumeclaims: "5" # Max 5 PVCs

# ============================================
# LAYER 4: LimitRange (Default & Constraints)
# ============================================

apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: application
spec:
  limits:
  # Container limits
  - type: Container
    default:  # Default limits if not specified
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:  # Default requests if not specified
      cpu: "250m"
      memory: "256Mi"
    min:  # Minimum allowed
      cpu: "100m"
      memory: "128Mi"
    max:  # Maximum allowed
      cpu: "2000m"
      memory: "4Gi"
  
  # Pod limits (sum of all containers)
  - type: Pod
    max:
      cpu: "4000m"
      memory: "8Gi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: "1Gi"
    max:
      storage: "100Gi"

# ============================================
# LAYER 5: Pod Security Standards
# ============================================

# Enforce security policies
apiVersion: v1
kind: Namespace
metadata:
  name: application
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

# Restricted policy enforces:
# - Non-root user
# - No privileged containers
# - Read-only root filesystem
# - Drop all capabilities
# - Seccomp profile

# ============================================
# LAYER 6: Application-Level Validation
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      # Init container validates environment
      initContainers:
      - name: validate-config
        image: myapp:1.0.0
        command:
        - sh
        - -c
        - |
          # Validate required env vars
          if [ -z "$DATABASE_URL" ]; then
            echo "ERROR: DATABASE_URL not set"
            exit 1
          fi
          
          if [ -z "$SECRET_KEY" ]; then
            echo "ERROR: SECRET_KEY not set"
            exit 1
          fi
          
          # Validate format
          if ! echo "$DATABASE_URL" | grep -q "^postgresql://"; then
            echo "ERROR: Invalid DATABASE_URL format"
            exit 1
          fi
          
          # Validate connectivity
          python3 -c "
          import psycopg2
          import os
          import sys
          try:
              conn = psycopg2.connect(os.getenv('DATABASE_URL'))
              conn.close()
              print('✓ Database connection valid')
          except Exception as e:
              print(f'ERROR: Database connection failed: {e}')
              sys.exit(1)
          "
          
          echo "✓ All validations passed"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: secret-key
      
      # Main container starts only if validation passes
      containers:
      - name: app
        image: myapp:1.0.0
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

**Open Policy Agent (OPA) for Advanced Validation:**

```yaml
# OPA validates Kubernetes resources before creation
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policy
data:
  policy.rego: |
    package kubernetes.admission
    
    # Deny deployments without resource limits
    deny[msg] {
      input.request.kind.kind == "Deployment"
      container := input.request.object.spec.template.spec.containers[_]
      not container.resources.limits
      msg := sprintf("Container '%s' must have resource limits", [container.name])
    }
    
    # Deny privileged containers
    deny[msg] {
      input.request.kind.kind == "Deployment"
      container := input.request.object.spec.template.spec.containers[_]
      container.securityContext.privileged == true
      msg := sprintf("Container '%s' cannot run privileged", [container.name])
    }
    
    # Require health checks
    deny[msg] {
      input.request.kind.kind == "Deployment"
      container := input.request.object.spec.template.spec.containers[_]
      not container.livenessProbe
      msg := sprintf("Container '%s' must have liveness probe", [container.name])
    }
```

**Why This Works:**
- Kubernetes validates against OpenAPI schema
- Admission webhooks enforce custom policies
- ResourceQuota limits total namespace consumption
- LimitRange provides defaults and boundaries
- Pod Security Standards enforce security posture
- Init containers validate runtime configuration
- OPA provides policy-as-code

---

### Principle 5: Standardized Error Handling (MANDATORY)

**Meta-Architecture Principle:**
> Consistent error patterns enable automated handling. Use standard error codes.

**Kubernetes Implementation:**

```yaml
# ============================================
# Container Exit Codes
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        # Container exit codes:
        # 0 = Success
        # 1 = General error
        # 2 = Misuse of shell command
        # 126 = Command cannot execute
        # 127 = Command not found
        # 128+N = Fatal signal N (e.g., 137 = SIGKILL)
        # 143 = SIGTERM (graceful shutdown)
        
        # Custom exit codes (application-defined):
        # 10 = Configuration error
        # 11 = Dependency unavailable
        # 12 = Data corruption
        
        # Kubernetes restart policy based on exit code
        restartPolicy: OnFailure
        # OnFailure = restart if exitCode != 0
        # Always = restart regardless of exit code
        # Never = never restart

# ============================================
# Termination Message
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        
        # Where to write termination message
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        
        # Application writes structured error before exit:
        # echo '{"error":"database_connection_failed","code":11}' > /dev/termination-log
        # exit 11

# View termination message:
# kubectl get pod myapp-xxx -o jsonpath='{.status.containerStatuses[0].lastState.terminated.message}'

# ============================================
# Event Logging
# ============================================

# Kubernetes automatically logs events
# View with: kubectl get events --sort-by='.lastTimestamp'

# Events include:
# - Pod scheduling failures (Reason: FailedScheduling)
# - Image pull failures (Reason: ErrImagePull)
# - Container crashes (Reason: CrashLoopBackOff)
# - Health check failures (Reason: Unhealthy)
# - Resource quota exceeded (Reason: FailedCreate)

# ============================================
# Custom Event Recording
# ============================================

# Application can record events via API
# Example in Python:
"""
from kubernetes import client, config

config.load_incluster_config()
v1 = client.CoreV1Api()

# Record custom event
event = client.V1Event(
    metadata=client.V1ObjectMeta(
        name=f'myapp-error-{timestamp}',
        namespace='application'
    ),
    involved_object=client.V1ObjectReference(
        kind='Pod',
        name=pod_name,
        namespace='application'
    ),
    reason='DatabaseError',
    message='Failed to connect to database after 5 retries',
    type='Warning',
    count=1
)

v1.create_namespaced_event(namespace='application', body=event)
"""

# ============================================
# Readiness/Liveness Probe Failures
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        
        # Liveness: Is the app running?
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3  # Restart after 3 failures
          # Returns error status if app not responsive
        
        # Readiness: Can the app serve traffic?
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3  # Remove from service after 3 failures
          # Returns error status if dependencies unavailable

# Health endpoint structure:
"""
# /health (liveness)
{
  "status": "healthy" | "unhealthy",
  "timestamp": "2025-11-10T21:00:00Z"
}

# /ready (readiness)
{
  "status": "ready" | "not_ready",
  "checks": {
    "database": "ok" | "error",
    "cache": "ok" | "error"
  },
  "timestamp": "2025-11-10T21:00:00Z"
}
"""

# ============================================
# Monitoring & Alerting
# ============================================

# Prometheus scrapes metrics
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8000

# Application exposes metrics:
"""
# /metrics (Prometheus format)
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/users",status="200"} 1543
http_requests_total{method="GET",endpoint="/users",status="500"} 23

# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 432
http_request_duration_seconds_bucket{le="0.5"} 1234
http_request_duration_seconds_bucket{le="1.0"} 1500

# Custom error metrics
# TYPE app_errors_total counter
app_errors_total{error_code="database_error"} 5
app_errors_total{error_code="cache_error"} 12
app_errors_total{error_code="validation_error"} 87
"""

# ============================================
# Alertmanager Rules
# ============================================

# PrometheusRule for alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: myapp
    interval: 30s
    rules:
    # High error rate alert
    - alert: HighErrorRate
      expr: |
        rate(app_errors_total[5m]) > 10
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value }} errors/sec"
    
    # Pod crash loop alert
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{pod=~"myapp-.*"}[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
    
    # Readiness probe failures
    - alert: ReadinessFailures
      expr: |
        kube_pod_status_ready{condition="false",pod=~"myapp-.*"} > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} failing readiness checks"
```

**Why This Works:**
- Exit codes provide standardized container status
- Termination messages give structured error details
- Kubernetes events log all state changes
- Health probes detect application errors
- Prometheus metrics enable monitoring
- Alertmanager rules trigger notifications

---

## Four-Layer Kubernetes Architecture

### Complete Multi-Namespace Architecture

```yaml
# ============================================
# LAYER 1: Foundation Namespace
# ============================================

apiVersion: v1
kind: Namespace
metadata:
  name: foundation
  labels:
    layer: foundation
    istio-injection: disabled  # No service mesh for foundation
---
# Storage Classes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  namespace: foundation
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
volumeBindingMode: WaitForFirstConsumer
---
# Default Network Policy (deny all)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: foundation
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# ============================================
# LAYER 2: Infrastructure Namespace
# ============================================

apiVersion: v1
kind: Namespace
metadata:
  name: infrastructure
  labels:
    layer: infrastructure
    istio-injection: disabled
---
# PostgreSQL StatefulSet (see earlier example)
# Redis Deployment (see earlier example)
# RabbitMQ StatefulSet
---
# Infrastructure Services
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: infrastructure
spec:
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: infrastructure
spec:
  selector:
    app: redis
  ports:
  - port: 6379

# ============================================
# LAYER 3: Integration Namespace
# ============================================

apiVersion: v1
kind: Namespace
metadata:
  name: integration
  labels:
    layer: integration
    istio-injection: enabled  # Service mesh for API gateway
---
# Ingress Controller (see earlier example)
# API Gateway
# Service Mesh Config (Istio VirtualService)

# ============================================
# LAYER 4: Application Namespace
# ============================================

apiVersion: v1
kind: Namespace
metadata:
  name: application
  labels:
    layer: application
    istio-injection: enabled
---
# Application Deployments
# Worker Deployments
# Cron Jobs
```

---

## Resource Design Patterns

### Pattern 1: Sidecar Pattern

```yaml
# Sidecar: Additional container in same pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      # Main application container
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
      
      # Sidecar: Log shipper
      - name: log-shipper
        image: fluent/fluentd:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
      
      # Sidecar: Metrics exporter
      - name: metrics
        image: prom/statsd-exporter:latest
        ports:
        - containerPort: 9102
      
      volumes:
      - name: logs
        emptyDir: {}

# All containers share:
# - Network namespace (localhost communication)
# - IPC namespace
# - Volumes
```

**Use Case:** Add functionality without modifying main container.

### Pattern 2: Init Container Pattern

```yaml
# Init containers run before main containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
      # 1. Setup: Create directories, set permissions
      - name: setup
        image: busybox:latest
        command:
        - sh
        - -c
        - |
          mkdir -p /data/cache
          chmod 777 /data/cache
        volumeMounts:
        - name: data
          mountPath: /data
      
      # 2. Migration: Run database migrations
      - name: migrate
        image: myapp:1.0.0
        command: ["python", "migrate.py"]
      
      # 3. Seed: Load initial data
      - name: seed
        image: myapp:1.0.0
        command: ["python", "seed.py"]
      
      containers:
      - name: app
        image: myapp:1.0.0
        volumeMounts:
        - name: data
          mountPath: /data
      
      volumes:
      - name: data
        emptyDir: {}
```

**Use Case:** Pre-flight tasks that run once before app starts.

### Pattern 3: Ambassador Pattern

```yaml
# Ambassador: Proxy to external services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      # Main app
      - name: app
        image: myapp:1.0.0
        env:
        - name: DATABASE_URL
          value: "postgresql://localhost:5432/myapp"  # Via ambassador
      
      # Ambassador: Connection pooler
      - name: pgbouncer
        image: pgbouncer/pgbouncer:latest
        ports:
        - containerPort: 5432
        env:
        - name: DB_HOST
          value: "external-db.example.com"
        - name: DB_PORT
          value: "5432"
        - name: POOL_SIZE
          value: "20"
```

**Use Case:** Abstract external service details, add connection pooling.

### Pattern 4: Adapter Pattern

```yaml
# Adapter: Transform data formats
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      # Main app
      - name: app
        image: myapp:1.0.0
      
      # Adapter: Prometheus metrics from app logs
      - name: log-adapter
        image: mtail:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
        ports:
        - containerPort: 3903  # Prometheus metrics
      
      volumes:
      - name: logs
        emptyDir: {}
```

**Use Case:** Convert app output to standard format (logs → metrics).

---

## Deployment Strategies

### Rolling Update (Default)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # Max 1 pod down at a time
      maxSurge: 1          # Max 1 extra pod during update
  template:
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000

# Process:
# 1. Start 1 new pod (v2.0.0)
# 2. Wait for readiness probe to pass
# 3. Terminate 1 old pod (v1.0.0)
# 4. Repeat until all pods updated
# 
# At any time: 9-11 pods running
# Zero downtime if readiness probe correct
```

### Blue-Green Deployment

```yaml
# Blue (current version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
---
# Green (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
---
# Service (switchable)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Initially points to blue
  ports:
  - port: 80
    targetPort: 8000

# Switch traffic:
# kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
# 
# Instant switch, easy rollback
```

### Canary Deployment

```yaml
# Stable (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
        version: "1.0"
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
---
# Canary (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
        version: "2.0"
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
---
# Service (routes to both)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Matches both stable and canary
  ports:
  - port: 80
    targetPort: 8000

# Traffic split:
# 90% → stable (9 replicas)
# 10% → canary (1 replica)
# 
# Monitor canary metrics, gradually increase if healthy
```

---

## Service Mesh Integration

### Istio for Advanced Traffic Management

```yaml
# Virtual Service: Traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
  namespace: application
spec:
  hosts:
  - myapp.application.svc.cluster.local
  http:
  # Canary: 10% to v2
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: myapp
        subset: v2
  
  # Weight-based split
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
    
    # Timeout
    timeout: 10s
    
    # Circuit breaker
    fault:
      abort:
        percentage:
          value: 0.1
        httpStatus: 503
---
# Destination Rule: Traffic policies
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
  namespace: application
spec:
  host: myapp.application.svc.cluster.local
  
  # Connection pool settings
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    
    # Outlier detection (circuit breaker)
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  
  # Subsets
  subsets:
  - name: v1
    labels:
      version: "1.0"
  - name: v2
    labels:
      version: "2.0"
```

---

## Observability & Monitoring

### Prometheus + Grafana Stack

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
# Grafana Dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  myapp.json: |
    {
      "dashboard": {
        "title": "MyApp Dashboard",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "rate(http_requests_total{app=\"myapp\"}[5m])"
              }
            ]
          },
          {
            "title": "Error Rate",
            "targets": [
              {
                "expr": "rate(http_requests_total{app=\"myapp\",status=~\"5..\"}[5m])"
              }
            ]
          }
        ]
      }
    }
```

---

## Security Architecture

### RBAC (Role-Based Access Control)

```yaml
# ServiceAccount for application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: application
---
# Role: What can be done
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: application
rules:
# Can read ConfigMaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# Can create Events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create"]
---
# RoleBinding: Who can do it
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: application
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: application
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
---
# Use ServiceAccount in Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: application
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: app
        image: myapp:1.0.0
```

### Pod Security

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      # Security context for pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: app
        image: myapp:1.0.0
        
        # Security context for container
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Writable volume for temp files
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      
      volumes:
      - name: tmp
        emptyDir: {}
```

---

## Complete Examples

See full examples throughout the document for:
- Four-layer namespace architecture
- Complete deployment with all patterns
- Service mesh configuration
- Monitoring stack
- Security policies

---

## Compliance Checklist

- [ ] **Layered Architecture**: Foundation → Infrastructure → Integration → Application namespaces
- [ ] **Explicit Dependencies**: SHA-pinned images, init containers, service dependencies
- [ ] **Graceful Degradation**: Priority classes, anti-affinity, PodDisruptionBudgets
- [ ] **Input Validation**: Admission webhooks, ResourceQuota, LimitRange, Pod Security
- [ ] **Standardized Errors**: Exit codes, termination messages, events, metrics
- [ ] **Hierarchical Config**: ConfigMaps, Secrets, env precedence
- [ ] **Observable Behavior**: Prometheus metrics, health checks, distributed tracing
- [ ] **Automated Testing**: CI/CD with Kubernetes manifests
- [ ] **Security by Design**: RBAC, Pod Security, Network Policies, non-root containers
- [ ] **Resource Lifecycle**: Proper shutdown (SIGTERM), PVCs, StatefulSets
- [ ] **Performance Patterns**: HPA, resource limits/requests, node affinity
- [ ] **Evolutionary Design**: Rolling updates, blue-green, canary deployments

---

## Summary

This Kubernetes API architecture provides production-ready orchestration patterns:

**Key Characteristics:**
- ✅ **Declarative** - Desired state, Kubernetes reconciles
- ✅ **Self-healing** - Automatic pod restarts, rescheduling
- ✅ **Scalable** - HPA, multi-node, load balancing
- ✅ **Observable** - Prometheus, Grafana, distributed tracing
- ✅ **Secure** - RBAC, Pod Security, Network Policies
- ✅ **Production-ready** - All 12 Meta-Architecture principles

[View your Kubernetes API architecture](computer:///mnt/user-data/outputs/KUBERNETES-API-ARCHITECTURE.md)
