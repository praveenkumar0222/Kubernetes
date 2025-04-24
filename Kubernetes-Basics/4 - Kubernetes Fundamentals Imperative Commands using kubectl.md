

# **Kubernetes Fundamentals Study Guide**

## **1. Kubernetes Architecture**
### **1.1 Why Kubernetes?**
Kubernetes is a **portable, extensible, open-source platform** for managing containerized workloads. Key features:
- **Service discovery & load balancing**  
- **Storage orchestration**  
- **Automated rollouts/rollbacks**  
- **Self-healing** (auto-replaces failed containers)  
- **Secrets & configuration management**  

### **1.2 Cluster Components**
#### **Master Node Components**
| Component          | Function                                                                 |
|--------------------|--------------------------------------------------------------------------|
| **kube-apiserver** | Frontend for control plane; exposes Kubernetes API.                      |
| **etcd**           | Key-value store for cluster data (node info, configs).                   |
| **kube-scheduler** | Assigns pods to nodes based on resource availability.                    |
| **kube-controller-manager** | Manages controllers (node, replication, endpoints).               |
| **cloud-controller-manager** | Integrates with cloud providers (AWS/GCP/Azure).                  |

#### **Worker Node Components**
| Component       | Function                                                                 |
|----------------|--------------------------------------------------------------------------|
| **kubelet**    | Agent ensuring containers run in pods.                                   |
| **kube-proxy** | Manages network rules for pod communication.                             |
| **Container Runtime** | Runs containers (Docker, containerd, etc.).                          |

![K8s Architecture](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.png)

---

## **2. Core Concepts**
### **2.1 Pods**
**Definition**: Smallest deployable unit in Kubernetes. A pod encapsulates one or more containers sharing storage/network.  
**Key Points**:
- 1 Pod ≈ 1 Container (99% cases). Multi-container pods are rare (e.g., sidecar for logging).  
- Pods are **ephemeral**—if deleted, they’re recreated by controllers (Deployments/ReplicaSets).  

**Example**:  
```bash
kubectl run my-first-pod --image=stacksimplify/kubenginx:1.0.0 --generator=run-pod/v1
```
**Multi-container Pod Use Case**:  
- **Sidecar container**: Collects logs from the main app container (e.g., Nginx + Fluentd).

---

### **2.2 ReplicaSets**
**Definition**: Ensures a specified number of pod replicas run at all times.  

**Features**:
- **High Availability**: Auto-replaces crashed pods.  
- **Scaling**: Manually or via HPA (Horizontal Pod Autoscaler).  

**Example YAML**:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-helloworld-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: my-helloworld-app
        image: stacksimplify/kube-helloworld:1.0.0
```

**Commands**:
```bash
kubectl scale rs my-helloworld-rs --replicas=5  # Scale up
kubectl delete pod my-helloworld-rs-abc123      # Self-healing test
```

---

### **2.3 Deployments**
**Definition**: Manages ReplicaSets and enables declarative updates.  

**Features**:
- **Rolling Updates**: Zero-downtime deployments (default strategy).  
- **Rollback**: Revert to previous versions.  
- **Pause/Resume**: Batch multiple changes.  

**Example Commands**:
```bash
kubectl create deployment my-first-deploy --image=stacksimplify/kubenginx:1.0.0
kubectl set image deploy/my-first-deploy kubenginx=stacksimplify/kubenginx:2.0.0 --record
kubectl rollout undo deploy/my-first-deploy  # Rollback
```

**Rollout History**:
```bash
kubectl rollout history deploy/my-first-deploy
REVISION  CHANGE-CAUSE
1         kubectl create --image=1.0.0
2         kubectl set image to 2.0.0
```

---

### **2.4 Services**
**Definition**: Exposes pods as network services.  

#### **Service Types**
| Type          | Use Case                                | Example                          |
|---------------|----------------------------------------|----------------------------------|
| **ClusterIP** | Internal communication (default).       | Backend-to-database.             |
| **NodePort**  | External access via node IP + port.     | `http://<NodeIP>:30080`          |
| **LoadBalancer** | Cloud-integrated external LB.        | AWS ALB/NLB.                     |
| **Ingress**   | Advanced HTTP routing (path-based).     | SSL, redirects, path routing.    |

**NodePort Example**:
```bash
kubectl expose deploy/my-frontend --type=NodePort --port=80 --target-port=80 --name=frontend-svc
```
**Access**:
```
http://<WorkerNodeIP>:<NodePort>
```

---

## **3. Hands-On Examples**
### **3.1 Frontend + Backend Setup**
**Scenario**:  
- **Backend**: SpringBoot app (`my-backend`) with ClusterIP.  
- **Frontend**: Nginx proxy to backend, exposed via NodePort.  

**Steps**:
1. Deploy backend:
   ```bash
   kubectl create deploy my-backend --image=stacksimplify/kube-helloworld:1.0.0
   kubectl expose deploy/my-backend --port=8080 --target-port=8080 --name=backend-svc
   ```
2. Deploy frontend:
   ```bash
   kubectl create deploy my-frontend --image=stacksimplify/kube-frontend-nginx:1.0.0
   kubectl expose deploy/my-frontend --type=NodePort --port=80 --name=frontend-svc
   ```
3. Access:
   ```bash
   kubectl get svc frontend-svc -o wide
   # Open http://<NodeIP>:<NodePort>/hello
   ```

---

## **4. Key Takeaways**
- **Pods**: Atomic units; use Deployments/ReplicaSets for management.  
- **Services**: Enable communication (internal/external).  
- **Rolling Updates**: Minimize downtime during deployments.  
- **Self-Healing**: Kubernetes auto-replaces failed pods.  

**Diagram**:  
![K8s Workflow](https://www.weave.works/wp-content/uploads/2022/04/Kubernetes-Architecture-Diagram.png)

---

## **5. Cheat Sheet**
| Command                                      | Purpose                                  |
|----------------------------------------------|------------------------------------------|
| `kubectl get pods -o wide`                   | List pods with node/IP details.          |
| `kubectl describe pod <pod-name>`            | Debug pod issues.                        |
| `kubectl logs <pod-name> -f`                 | Stream logs.                             |
| `kubectl exec -it <pod-name> -- /bin/bash`   | Enter container shell.                   |
| `kubectl rollout status deploy/<name>`       | Check deployment progress.               |

---

