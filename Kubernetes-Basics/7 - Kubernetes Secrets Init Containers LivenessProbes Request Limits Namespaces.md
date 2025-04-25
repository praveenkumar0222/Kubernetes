
# **Kubernetes Advanced Concepts Study Guide**
## **1. Kubernetes Secrets**
### **1.1 Definition**
Secrets store sensitive data (passwords, tokens, SSH keys) in Base64-encoded format, separate from application manifests.

### **1.2 Key Features**
- **Encryption**: Base64-encoded (not encrypted by default; consider enabling encryption at rest).
- **Types**: `Opaque` (default for arbitrary data), `docker-registry`, `tls`.

### **1.3 Example: MySQL Password Secret**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-db-password
type: Opaque
data:
  db-password: ZGJwYXNzd29yZDEx  # Base64 for "dbpassword11"
```

**Usage in Deployment**:
```yaml
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-db-password
      key: db-password
```

### **1.4 Visual: Secret Flow**
```
+---------------------+
|   Secret Created    |
| (mysql-db-password) |
+----------+----------+
           |
           v
+----------+----------+
| Deployment References|
| via secretKeyRef     |
+---------------------+
```

---

## **2. Init Containers**
### **2.1 Definition**
Init containers run **before** the main app container to perform setup tasks (e.g., DB readiness checks).

### **2.2 Key Features**
- **Order**: Runs sequentially; all must succeed before the main container starts.
- **Use Cases**: 
  - Wait for dependencies (e.g., MySQL).
  - Pre-load data or configurations.

### **2.3 Example: MySQL Readiness Check**
```yaml
initContainers:
- name: init-db
  image: busybox
  command: ['sh', '-c', 'until nc -z mysql 3306; do echo "Waiting for MySQL"; sleep 2; done']
```

### **2.4 Visual: Init Container Workflow**
```
+---------------------+
|   Init Container    |
| (Checks MySQL)      |
+----------+----------+
           |
           v
+----------+----------+
|   Main Container    |
| (User Management)   |
+---------------------+
```

---

## **3. Liveness & Readiness Probes**
### **3.1 Definitions**
| Probe Type | Purpose | Example |
|------------|---------|---------|
| **Liveness** | Restarts container if app is unresponsive | HTTP GET `/health` |
| **Readiness** | Allows traffic only when app is ready | TCP Socket `:8095` |
| **Startup** | Disables other probes during slow starts | Delay checks for legacy apps |

### **3.2 Example: Probes in Deployment**
```yaml
livenessProbe:
  exec:
    command: ["nc", "-z", "localhost", "8095"]
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8095
  initialDelaySeconds: 60
```

### **3.3 Visual: Probe Timing**
```
Container Starts
│
├─ 60s Delay (initialDelaySeconds)
│  ├─ Liveness Probe Executes (every 10s)
│  └─ Readiness Probe Checks /health
│
└─ Traffic Allowed (Readiness Success)
```

---

## **4. Resource Requests & Limits**
### **4.1 Definitions**
- **Requests**: Guaranteed resources (scheduler uses this for placement).
- **Limits**: Maximum allowed resources (prevents overconsumption).

### **4.2 Example: CPU/Memory Constraints**
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "500m"  # 0.5 CPU core
  limits:
    memory: "500Mi"
    cpu: "1000m" # 1 CPU core
```

### **4.3 Conversion Table**
| Unit | Description |
|------|-------------|
| `1 CPU` | 1 vCPU core (e.g., AWS) |
| `1000m` | Same as `1 CPU` |
| `128Mi` | 128 Mebibytes (134MB) |

### **4.4 Visual: Resource Allocation**
```
Node Resources (2 CPU, 4GB RAM)
│
├─ App Requests: 0.5 CPU, 128MiB
│  └─ Scheduler reserves this
└─ App Limits: 1 CPU, 500MiB
   └─ Container throttled if exceeded
```

---

## **5. Namespaces**
### **5.1 Definition**
Logical partitions in a cluster to isolate resources (e.g., `dev`, `prod`).

### **5.2 Example: Create a Namespace**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### **5.3 Visual: Namespace Isolation**
```
Cluster
├── Namespace: dev
│   ├── Pod: user-mgmt-dev
│   └── Service: mysql-dev
└── Namespace: prod
    ├── Pod: user-mgmt-prod
    └── Service: mysql-prod
```

---

## **Troubleshooting Cheatsheet**
| Issue | Solution |
|-------|----------|
| **PVC Pending** | Check StorageClass and IAM permissions |
| **Init Container Fails** | Verify dependency (e.g., MySQL service name) |
| **Readiness Probe Fails** | Increase `initialDelaySeconds` |
| **CPU Throttling** | Adjust `limits.cpu` |

---

## **Key Takeaways**
1. **Secrets**: Securely manage credentials.
2. **Init Containers**: Ensure dependencies are ready.
3. **Probes**: Control traffic flow and auto-healing.
4. **Resources**: Optimize cluster utilization.
5. **Namespaces**: Isolate environments.

---

# Kubernetes Namespaces, Limit Ranges, and Resource Quotas - Comprehensive Study Material

## Table of Contents
1. [Introduction to Kubernetes Namespaces](#introduction-to-kubernetes-namespaces)
2. [Working with Namespaces](#working-with-namespaces)
3. [Limit Ranges](#limit-ranges)
4. [Resource Quotas](#resource-quotas)
5. [Practical Implementation](#practical-implementation)
6. [Best Practices](#best-practices)

## Introduction to Kubernetes Namespaces

### Definition
Kubernetes namespaces are virtual clusters within a physical Kubernetes cluster that provide isolation boundaries for objects. They are often called "virtual clusters" because they create logical partitions within a single physical cluster.

### Key Characteristics
- **Isolation**: Objects in one namespace are isolated from objects in other namespaces
- **Resource Organization**: Helps organize resources for different teams, projects, or environments
- **Access Control**: Enables RBAC (Role-Based Access Control) at namespace level
- **Resource Management**: Allows setting resource quotas per namespace

### Default Namespaces
Every Kubernetes cluster comes with these built-in namespaces:

| Namespace | Purpose |
|-----------|---------|
| `default` | Where objects are created if no namespace is specified |
| `kube-system` | Contains system components like kube-dns, kube-proxy, etc. |
| `kube-public` | Readable by all users (even unauthenticated) for cluster-wide resources |
| `kube-node-lease` | Holds lease objects for node heartbeats (improves performance) |

### Why Use Namespaces?
1. **Multi-tenancy**: Support multiple teams/projects on same cluster
2. **Environment Separation**: Dev/QA/Prod environments isolation
3. **Resource Allocation**: Different resource limits per environment
4. **Access Control**: Different permissions per namespace
5. **Name Collision Prevention**: Same resource names can exist in different namespaces

## Working with Namespaces

### Creating Namespaces

#### Imperative Method
```bash
kubectl create namespace dev1
kubectl create namespace dev2
```

#### Declarative Method (YAML)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev3
```

### Viewing Namespaces
```bash
kubectl get namespaces  # or kubectl get ns
```

### Deploying to Specific Namespaces

#### Using -n flag (imperative)
```bash
kubectl apply -f manifests/ -n dev1
```

#### Specifying in YAML (declarative)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-management
  namespace: dev3  # Specified here
spec:
  # ... rest of deployment spec
```

### Namespace-scoped vs Cluster-scoped Objects

| Namespace-scoped | Cluster-scoped |
|------------------|----------------|
| Pods             | Nodes          |
| Services         | PersistentVolumes |
| Deployments      | StorageClasses |
| ConfigMaps       | ClusterRoles   |
| Secrets          | Namespaces themselves |

### Practical Example: Multi-environment Deployment

```bash
# Create namespaces for different environments
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Deploy same application to different namespaces
kubectl apply -f app-manifests/ -n dev
kubectl apply -f app-manifests/ -n staging
kubectl apply -f app-manifests/ -n production
```

## Limit Ranges

### Definition
Limit ranges allow you to:
- Set default resource requests/limits for containers in a namespace
- Define min/max constraints for resources
- Apply automatically to containers that don't specify their own limits

### Why Use Limit Ranges?
1. **Prevent resource starvation**: Ensure containers get minimum required resources
2. **Prevent overconsumption**: Limit maximum resource usage
3. **Set defaults**: Apply standard resource limits across namespace
4. **Enforce policies**: Ensure all containers have resource constraints

### Limit Range Manifest Example
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-cpu-mem-limit-range
  namespace: dev3
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 300m
      memory: 256Mi
    type: Container
```

### Key Fields Explained

| Field | Description | Example Value |
|-------|-------------|---------------|
| `default` | Default limit values if not specified | `cpu: 500m, memory: 512Mi` |
| `defaultRequest` | Default request values if not specified | `cpu: 300m, memory: 256Mi` |
| `type` | What the limits apply to (Container, Pod, PersistentVolumeClaim) | `Container` |

### Behavior When Limits Not Specified

1. If `default` is not specified:
   - CPU: Defaults to 1 vCPU
   - Memory: Defaults to 512Mi

2. If `defaultRequest` is not specified:
   - Uses the `default` values (which could lead to over-provisioning)

### Viewing Limit Ranges
```bash
kubectl describe limitrange -n dev3
```

### Practical Example: Applying Limit Range

1. Create limit range:
```bash
kubectl apply -f limit-range.yaml
```

2. Deploy application without resource specs:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    # No resources specified!
```

3. Verify limits were applied:
```bash
kubectl get pod nginx-pod -o yaml
# Will show the default limits from LimitRange
```

## Resource Quotas

### Definition
Resource quotas constrain aggregate resource consumption per namespace. They can limit:
- Compute resources (CPU, memory)
- Storage resources
- Object counts (number of pods, services, etc.)

### Why Use Resource Quotas?
1. **Prevent resource exhaustion**: Protect cluster from one namespace consuming all resources
2. **Cost control**: Enforce budget constraints per team/project
3. **Fair sharing**: Ensure equitable resource distribution
4. **Governance**: Enforce organizational policies

### Resource Quota Manifest Example
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-resource-quota
  namespace: dev3
spec:
  hard:
    pods: "5"
    configmaps: "5"
    persistentvolumeclaims: "5"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

### Supported Quota Types

| Category | Examples |
|----------|----------|
| Compute Resources | `requests.cpu`, `limits.memory`, `requests.ephemeral-storage` |
| Storage Resources | `persistentvolumeclaims`, `requests.storage` |
| Object Counts | `pods`, `services`, `configmaps`, `secrets` |

### Viewing Resource Quotas
```bash
kubectl get resourcequotas -n dev3
kubectl describe resourcequota ns-resource-quota -n dev3
```

Sample output:
```
Name:            ns-resource-quota
Namespace:       dev3
Resource         Used  Hard
--------         ----  ----
configmaps       1     5
limits.cpu       1     2
limits.memory    1Gi   2Gi
pods             2     5
requests.cpu     1     2
requests.memory  1Gi   2Gi
```

### Practical Example: Quota Enforcement

1. Create quota allowing only 2 pods:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
  namespace: test
spec:
  hard:
    pods: "2"
```

2. Try creating 3 pods:
```bash
kubectl run pod1 --image=nginx -n test
kubectl run pod2 --image=nginx -n test
kubectl run pod3 --image=nginx -n test  # This will fail!
```

Error message:
```
Error from server (Forbidden): pods "pod3" is forbidden: 
exceeded quota: pod-quota, requested: pods=1, 
used: pods=2, limited: pods=2
```

## Practical Implementation

### Multi-environment Setup with Namespaces, Limits and Quotas

1. **Create namespaces**:
```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production
```

2. **Apply different limit ranges**:
```bash
# Dev namespace - relaxed limits
kubectl apply -f dev-limits.yaml -n dev

# Production namespace - strict limits
kubectl apply -f prod-limits.yaml -n production
```

3. **Apply resource quotas**:
```bash
# Dev team gets limited resources
kubectl apply -f dev-quota.yaml -n dev

# Production gets more resources
kubectl apply -f prod-quota.yaml -n production
```

4. **Deploy applications**:
```bash
kubectl apply -f app/ -n dev
kubectl apply -f app/ -n staging
kubectl apply -f app/ -n production
```

### Sample dev-limits.yaml
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
spec:
  limits:
  - default:
      cpu: 1
      memory: 1Gi
    defaultRequest:
      cpu: 500m
      memory: 512Mi
    type: Container
```

### Sample prod-quota.yaml
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
spec:
  hard:
    pods: "20"
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "10"
```

## Best Practices

1. **Always use namespaces**:
   - Even for small clusters to maintain organization
   - Prevents "default" namespace clutter

2. **Namespace naming conventions**:
   - `team-project` (e.g., `backend-payments`)
   - `environment-purpose` (e.g., `prod-databases`)
   - Avoid special characters

3. **Limit Range recommendations**:
   - Set reasonable defaults for your workload types
   - Consider setting min/max constraints in production
   - Document your limit policies

4. **Resource Quota strategies**:
   - Start conservative and increase as needed
   - Monitor usage with `kubectl describe quota`
   - Consider separate quotas for different resource types

5. **Combining with RBAC**:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: dev-team-binding
     namespace: dev
   subjects:
   - kind: Group
     name: dev-team
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: edit
     apiGroup: rbac.authorization.k8s.io
   ```

6. **Monitoring and alerts**:
   - Set up alerts when quota usage exceeds 80%
   - Regularly review actual usage vs quotas

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Forgetting to specify namespace | Use `kubectl config set-context --current --namespace=dev` |
| Hitting quota limits unexpectedly | Monitor usage regularly, set up alerts |
| Containers failing due to limits | Set proper `defaultRequest` values in LimitRange |
| Cluster-scoped objects in namespaces | Understand which objects are namespace-scoped vs cluster-scoped |
| Too many small namespaces | Consolidate when possible to reduce management overhead |

## Conclusion

Kubernetes namespaces with limit ranges and resource quotas provide powerful tools for:
- Organizing cluster resources
- Isolating workloads
- Preventing resource starvation
- Enforcing policies
- Managing multi-tenant environments

By implementing these concepts thoughtfully, you can create more secure, stable, and manageable Kubernetes clusters that efficiently utilize resources while preventing any single application or team from monopolizing cluster capacity.