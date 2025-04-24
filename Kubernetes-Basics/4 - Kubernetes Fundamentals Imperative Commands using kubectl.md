

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



Great! Here's a detailed study material based on the provided transcript about **Kubernetes Architecture**. This breakdown explains each line thoroughly with examples, definitions, and visual guidance suggestions where relevant.

---

# 🧠 Kubernetes Architecture - Study Material

---

## 🎓 Introduction

**Transcript:**
> "Instructor: Welcome back. In this lecture we are going to understand a little bit about Kubernetes architecture."

**Explanation:**
Kubernetes is an open-source container orchestration platform. This lecture introduces the architectural overview of Kubernetes — a system that manages containerized applications across a cluster of machines.

---

## ❓ Why Kubernetes?

**Transcript:**
> "So why Kubernetes? Kubernetes is a portable, extensible, open source platform for managing container workloads."

**Definition:**
- **Portable:** Can run on any environment (on-premise, cloud, hybrid).
- **Extensible:** Can be customized and integrated with other tools.
- **Open Source:** Free to use and supported by a vast community.

**Use Case:**
Kubernetes is ideal for deploying scalable web applications, such as e-commerce sites or SaaS platforms.

---

## ✨ Features of Kubernetes

**Transcript:**
> "Out of the box the following features are available in Kubernetes..."

| Feature                    | Description                                                                 |
|---------------------------|-----------------------------------------------------------------------------|
| Service Discovery         | Identifies services using DNS names or IPs.                                 |
| Load Balancing            | Distributes network traffic evenly.                                         |
| Storage Orchestration     | Automatically mounts the storage system of your choice.                     |
| Automated Rollouts/Rollbacks | Ensures continuous deployment with zero downtime.                     |
| Automatic Bin Packing     | Efficiently schedules containers based on resource needs and availability. |
| Self-healing              | Automatically replaces and reschedules failed containers.                  |
| Secrets & Config Management | Stores and manages sensitive information securely.                      |

**Example:**
If a web app fails, Kubernetes restarts the container automatically to maintain uptime (self-healing).

---

## 🧱 Kubernetes Architecture Overview

**Transcript:**
> "We’ll have both master and worker nodes... all these nodes will have container runtime installed..."

### 🔹 Components of Kubernetes Architecture

#### 🧠 Master Node

| Component                | Role                                                                 |
|--------------------------|----------------------------------------------------------------------|
| **Kube API Server**      | Entry point for all API requests, validates and configures resources.|
| **etcd**                 | Key-value store for all cluster data.                               |
| **Kube Scheduler**       | Assigns pods to available worker nodes.                              |
| **Kube Controller Manager** | Manages controllers that handle node status, replication, etc.     |
| **Cloud Controller Manager** | Interacts with cloud services like load balancers, routes, etc. |

> All master node components interact with the **Kube API Server**.

#### 🧑‍🏭 Worker Node

| Component                | Role                                                                 |
|--------------------------|----------------------------------------------------------------------|
| **Kubelet**              | Ensures containers are running as expected.                         |
| **Kube Proxy**           | Maintains network rules to allow communication.                     |
| **Container Runtime**    | Software like Docker or containerd that runs containers.            |

---

## 🔁 Master Node Components in Detail

### 1. Kube API Server

**Transcript:**
> "Acts as the front end for the control plane, exposes the Kubernetes API."

**Function:**
All tools (kubectl, UI dashboards) and components (scheduler, etc.) interact through this server.

📌 **Example:** When you run `kubectl get pods`, the request is handled by the Kube API Server.

---

### 2. etcd

**Transcript:**
> "A consistent and highly available key-value store..."

**Function:**
Stores cluster state, node information, secrets, config maps.

📌 **Example:** etcd stores pod definitions and node IPs.

---

### 3. Kube Scheduler

**Transcript:**
> "Responsible for distributing containers across nodes..."

**Function:**
Assigns unassigned pods to suitable worker nodes based on resource availability, affinity rules, etc.

📌 **Example:** A pod needing 1 CPU and 1GB RAM is scheduled to a node with enough free resources.

---

### 4. Kube Controller Manager

**Transcript:**
> "Combination of multiple controllers..."

| Controller              | Responsibility                                               |
|-------------------------|--------------------------------------------------------------|
| Node Controller         | Detects node failures.                                       |
| Replication Controller  | Maintains correct number of pod replicas.                   |
| Endpoints Controller    | Connects services and pods.                                  |
| Service Account & Token Controller | Creates default accounts and tokens.          |

---

### 5. Cloud Controller Manager

**Transcript:**
> "Embeds cloud-specific control logic..."

| Controller        | Cloud-Specific Role                                                  |
|-------------------|----------------------------------------------------------------------|
| Node Controller   | Checks if cloud-deleted nodes need removal from cluster.            |
| Route Controller  | Sets up cloud routing between pods/services.                        |
| Service Controller| Manages cloud load balancers for Kubernetes Services.               |

📌 **Note:** Only present in cloud-based Kubernetes setups.

---

## 🧑‍💻 Worker Node Components in Detail

### 1. Container Runtime

**Transcript:**
> "Docker or any other run time is mandatory on every node."

**Examples of Runtimes:**
- Docker
- containerd
- CRI-O

---

### 2. Kubelet

**Transcript:**
> "Agent that runs on every node... ensures containers are running."

**Function:**
Fetches pod specs from the API server and ensures the containers run as defined.

📌 **Example:** If a pod crashes, kubelet restarts it.

---

### 3. Kube Proxy

**Transcript:**
> "Maintains network rules... allows communication inside and outside cluster."

**Function:**
Enables service networking and external exposure of services.

📌 **Example:** If a user accesses `myapp.example.com`, kube-proxy routes that to the correct pod IP.

---

## 📊 Visual Architecture Diagram (Suggested)

Here’s a recommended structure you can visualize:

```
+-----------------------------+
|         Master Node        |
|-----------------------------|
| Kube API Server             |
| etcd                        |
| Kube Scheduler              |
| Kube Controller Manager     |
| Cloud Controller Manager    |
+-----------------------------+

+-----------------------------+
|        Worker Node          |
|-----------------------------|
| Container Runtime (Docker)  |
| Kubelet                     |
| Kube Proxy                  |
+-----------------------------+
```

---

Great! Here's a detailed breakdown of the EKS architecture and how Kubernetes core components fit into it, including imperative and declarative management approaches using `kubectl` and YAML:

---

## 🚀 **Amazon EKS Architecture Overview**

Amazon EKS (Elastic Kubernetes Service) is a managed Kubernetes service provided by AWS. It simplifies running Kubernetes by offloading the control plane management to AWS.

### 🔧 **EKS Architecture Components**

| Component             | Description                                                                 |
|-----------------------|-----------------------------------------------------------------------------|
| **EKS Control Plane** | Managed by AWS. Runs `kube-apiserver`, `etcd`, `controller-manager`, `scheduler`. |
| **Worker Nodes**      | EC2 instances (or Fargate) that run your application workloads (Pods).      |
| **Node Groups**       | Collections of EC2 instances (auto-scaling, managed/unmanaged).             |
| **Networking**        | Uses VPC, subnets, security groups, ENIs for pod & service communication.   |
| **IAM Roles**         | Used for authentication/authorization in EKS (e.g., for worker nodes).      |
| **Add-ons**           | AWS supports native add-ons like CoreDNS, kube-proxy, VPC CNI, etc.         |

---

## 🧱 **Core Kubernetes Components (EKS Compatible)**

| Component     | Purpose                                                                                 |
|---------------|-----------------------------------------------------------------------------------------|
| **Pod**       | Smallest unit in Kubernetes; runs one or more containers.                              |
| **ReplicaSet**| Ensures a specified number of pod replicas are running at all times.                   |
| **Deployment**| Manages ReplicaSets; allows rolling updates, rollback, and versioning.                 |
| **Service**   | Exposes Pods; handles internal/external access and load balancing.                     |

---

## 🛠️ **Management Approaches**

### 1. **Imperative Approach** (`kubectl` CLI commands)

Manage resources directly via CLI. Good for quick changes.

#### 📌 Examples:
```bash
# Create a pod
kubectl run mypod --image=nginx

# Scale deployment
kubectl scale deployment my-deploy --replicas=3

# Expose a deployment as a service
kubectl expose deployment my-deploy --type=NodePort --port=80
```

---

### 2. **Declarative Approach** (YAML manifests)

Define resources in YAML files, and apply them using `kubectl apply`.

#### 📄 Example: Deployment YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

#### Apply YAML:
```bash
kubectl apply -f nginx-deployment.yaml
```

---

### 💡 Tip:
- Use **imperative** for ad-hoc tasks.
- Use **declarative** for production and CI/CD (version-controllable).

---

## 📈 Visual EKS Architecture

```plaintext
+-----------------------------------------------------------+
|                    AWS Managed Control Plane              |
|  +----------------------+  +-------------------------+    |
|  | kube-apiserver       |  | controller-manager      |    |
|  | scheduler            |  | etcd                    |    |
|  +----------------------+  +-------------------------+    |
+-----------------------------------------------------------+
                  |
       +----------|-----------+
       |                      |
+--------------+     +--------------------+
|  Worker Node |     |    Worker Node     |
| (EC2/Fargate)|     |    (EC2/Fargate)   |
|   kubelet    |     |     kubelet        |
|   pods       |     |     pods           |
+--------------+     +--------------------+

     <- IAM, CNI, kube-proxy, CoreDNS installed here ->
```
Thanks! Here's a **study guide** based on the transcript you shared from the "Stack Simplify Kubernetes Fundamentals" course. This guide includes **key concepts, explanations, commands, examples**, and **diagrams** where relevant.

---

## 🧠 **Kubernetes Core Concept: Pods (Hands-On)**

---

### 🔹 What is a Pod?

> A **Pod** is the smallest and simplest unit in the Kubernetes object model that you can create or deploy. It represents a **single instance of a running process** in your cluster.

- A Pod can contain **one or more containers**, but usually just one.
- All containers in a Pod share the **same network namespace** (IP address and port space) and can communicate via `localhost`.

---

### 🔧 Creating a Pod Imperatively

#### ✅ Command:
```bash
kubectl run nginx-pod --image=nginx
```

- `kubectl run`: Used to create a Pod directly from the command line.
- `nginx-pod`: Name of the Pod.
- `--image=nginx`: Specifies the container image to use (from DockerHub).

#### 📦 Result:
This creates a pod named `nginx-pod` running a container from the `nginx` image.

---

### 🔍 Verifying the Pod

#### Command:
```bash
kubectl get pods
```

Shows:
- NAME
- READY (containers ready)
- STATUS
- RESTARTS
- AGE

---

### 📄 Detailed Pod Info

#### Command:
```bash
kubectl describe pod nginx-pod
```

Displays:
- **Pod conditions** (e.g., `Ready`, `Initialized`)
- **Events** (like `Scheduled`, `Pulled`, `Created`, `Started`)
- Node where it is scheduled
- Labels, IP address, container status, etc.

---

### 🔁 Repeating Status Flow in Pod Lifecycle:

```
Pending ➝ ContainerCreating ➝ Running
```

Example from `kubectl describe`:
```bash
Events:
  Type     Reason          Age   From               Message
  ----     ------          ----  ----               -------
  Normal   Scheduled       3m    default-scheduler  Successfully assigned default/nginx-pod
  Normal   Pulled          3m    kubelet            Successfully pulled image "nginx"
  Normal   Created         3m    kubelet            Created container nginx
  Normal   Started         3m    kubelet            Started container nginx
```

---

### 📌 Getting More Details

#### Pod Status Only:
```bash
kubectl get pod nginx-pod -o wide
```

Shows additional fields like:
- Pod IP
- Node
- Container image
- Restarts

#### View Pod's YAML Configuration:
```bash
kubectl get pod nginx-pod -o yaml
```

You can see:
- Metadata (name, namespace, labels)
- Spec (containers, images, ports)
- Status

---

### 🗑️ Deleting a Pod

#### Command:
```bash
kubectl delete pod nginx-pod
```

---

### 🖼️ Diagram: Pod Structure

```
+--------------------------------------------------------+
|                        Pod: nginx-pod                 |
|                                                        |
|  +------------------+                                  |
|  | Container: nginx |  <-- shares same IP/namespace -->|
|  +------------------+                                  |
|                                                        |
+--------------------------------------------------------+
```

---

Here's a detailed and structured summary based on the transcript you provided. It explains the key Kubernetes networking concepts—particularly around **NodePort services**—with added examples, definitions, and visuals to aid understanding.

---

## 🎓 **Lecture Summary: Exposing Pods via NodePort Service in Kubernetes**

---

### 📘 **Key Concepts**

#### ✅ 1. **Ports in Kubernetes Services**
When creating a service in Kubernetes, three important ports need to be understood:

| Term         | Description                                                                                  | Example |
|--------------|----------------------------------------------------------------------------------------------|---------|
| **port**     | The port that the **Service** exposes internally within the **Kubernetes cluster**.          | 80      |
| **targetPort**| The **container's port** on which the application is running. It is **mapped from** `port`. | 80      |
| **nodePort** | A port on the **worker node** that allows external traffic to access the Service.            | 31084   |

---

### 🛠️ **Creating a Pod and Exposing it via NodePort**

1. **Create a Pod**
   ```bash
   kubectl run mypod --image=stacksimplify/kubenginx:1.0 --port=80
   ```

2. **Verify Pod**
   ```bash
   kubectl get pods
   ```

3. **Expose the Pod using a NodePort Service**
   ```bash
   kubectl expose pod mypod \
     --type=NodePort \
     --port=80 \
     --name=my-first-service
   ```

---

### 🔍 **View Services**
```bash
kubectl get services
# or shorthand
kubectl get svc
```

You’ll see output similar to:

```
NAME               TYPE       CLUSTER-IP     PORT(S)          AGE
kubernetes         ClusterIP  10.96.0.1      443/TCP           1h
my-first-service   NodePort   10.96.32.10    80:31084/TCP      2m
```

---

### 🌐 **Accessing Your Application**

To access the app:

```bash
http://<NODE_PUBLIC_IP>:<NODE_PORT>
# Example:
http://173.10.0.45:31084
```

> Even if the Pod is on a different node, the service is **replicated across all nodes**, so it can be accessed via **any node's public IP**.

---

### ⚠️ **What If You Don’t Specify `targetPort`?**

If `targetPort` is not defined explicitly:
- Kubernetes assumes `targetPort = port`.

If your Pod runs on a different port (e.g., 80), but the service is set to 81 and you don’t define the `targetPort`, it won’t work.

```bash
kubectl expose pod mypod \
  --type=NodePort \
  --port=81 \
  --name=my-second-service
# ⛔ Not accessible if app is actually on port 80
```

---

### ✅ **Fix: Define `targetPort` Properly**
To map Service `port` 81 to container’s `targetPort` 80:

```bash
kubectl expose pod mypod \
  --type=NodePort \
  --port=81 \
  --target-port=80 \
  --name=my-third-service
```

Now it will work!

---

### 🧠 **Key Takeaways**

- **port** = service port exposed inside the cluster.
- **targetPort** = container (app) port.
- **nodePort** = external access port exposed on the node.
- NodePort services are accessible on **any node IP**.
- If `targetPort` is not explicitly defined, Kubernetes assumes it equals `port`.

---

### 📊 Diagram: NodePort Service

```
Client (Browser)
      ↓
http://NodeIP:NodePort
      ↓
NodePort Service
      ↓
ClusterIP → Pod on targetPort
```

---
Here's a **detailed and structured study guide** based on the lecture transcript you provided. This focuses on **interacting with Kubernetes Pods**, viewing logs, executing commands inside containers, and getting YAML definitions.

---

## 📘 **Kubernetes - Interacting with Pods: Study Guide**

### 📌 **1. Viewing Pod Logs**

#### ✅ Basic Command
```bash
kubectl logs <pod-name>
```
- Fetches logs from the default container of the specified pod.

#### ✅ Stream Logs in Real Time
```bash
kubectl logs -f <pod-name>
```
- `-f` (or `--follow`) streams the logs live, useful for debugging in real-time.

#### ✅ Logs from Specific Container in Multi-Container Pod
```bash
kubectl logs <pod-name> -c <container-name>
```

> 🧠 **Example**:
```bash
kubectl logs my-first-pod
kubectl logs -f my-first-pod
```

---

### 📌 **2. Executing Commands Inside a Container**

#### ✅ Start Interactive Shell Inside Pod
```bash
kubectl exec -it <pod-name> -- /bin/bash
```
- `-it`: Interactive mode with a terminal.
- `-- /bin/bash`: Starts a bash shell if available.

> 🧠 **Example**:
```bash
kubectl exec -it my-first-pod -- /bin/bash
```

Once inside:
```bash
ls
cd /usr/share/nginx/html
cat index.html
```

#### ✅ Run Commands Directly from Outside (Non-interactive)
```bash
kubectl exec -it <pod-name> -- <command>
```

> 🧠 **Examples**:
```bash
kubectl exec -it my-first-pod -- env
kubectl exec -it my-first-pod -- cat /usr/share/nginx/html/index.html
```

---

### 📌 **3. Retrieving YAML Definition of a Pod or Service**

#### ✅ Get Pod YAML
```bash
kubectl get pod <pod-name> -o yaml
```

#### ✅ Get Service YAML
```bash
kubectl get svc <service-name> -o yaml
```

> 🧠 Useful to:
- Understand how resources are defined.
- Use as a base for writing declarative YAML files.
- Debug configuration issues.

> 🧠 **Example Output:**
- NodePort (e.g., `nodePort: 31084`)
- Target port (e.g., `targetPort: 80`)
- Port (e.g., `port: 80`)

---

### 📌 **4. kubectl Cheat Sheet Recommendation**
- Check official [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) for a wide range of commands.
- Focus on `logs`, `exec`, and `get` with output options like `-o yaml`.

---

## 🧠 Summary Table

| Command | Purpose |
|--------|---------|
| `kubectl logs <pod>` | Get logs |
| `kubectl logs -f <pod>` | Stream logs |
| `kubectl logs <pod> -c <container>` | Logs from specific container |
| `kubectl exec -it <pod> -- /bin/bash` | Open shell in container |
| `kubectl exec -it <pod> -- <command>` | Run single command in container |
| `kubectl get pod <pod> -o yaml` | Get Pod YAML |
| `kubectl get svc <svc> -o yaml` | Get Service YAML |

---

Here’s a detailed breakdown and study notes from the transcript you've shared. It covers two major parts: **cleanup of Kubernetes objects** and **introduction to ReplicaSets**.

---

## 🧹 Part 1: Cleaning Up Kubernetes Objects

### 🎯 Objective:
To delete Kubernetes objects (Pods and Services) created in earlier steps using `kubectl`.

### 🛠️ Key Commands:
- `kubectl get all`  
  Lists **all resources** in the default namespace (Pods, Services, etc.).
  
- `kubectl delete pod <pod-name>`  
  Deletes a specific Pod.
  
- `kubectl delete svc <service-name>`  
  Deletes a specific Service.

### 💡 Example:
```bash
kubectl delete pod my-first-pod
kubectl delete svc my-first-service
kubectl delete svc my-first-service-two
kubectl delete svc my-first-service-three
```

### ✅ Final Verification:
Run:
```bash
kubectl get all
```
You should see only the Kubernetes default service (`kubernetes`), everything else should be deleted.

---

## 📦 Part 2: Introduction to ReplicaSets

### 🎯 Purpose of ReplicaSets:
1. **High Availability**: Ensures your application is always running by maintaining a fixed number of pods.
2. **Reliability**: Automatically replaces failed pods.
3. **Scalability**: Easily scale pods up/down.
4. **Load Balancing**: Distributes traffic via Services.

---

### 🔁 What is a ReplicaSet?

A **ReplicaSet** ensures that a **specified number of replicas (pods)** are running at all times. If a pod crashes or is deleted, the ReplicaSet recreates it.

---

### ⚙️ How It Works:

#### 📍 Example Scenario:
- You have 2 pods running.
- One pod crashes.
- The ReplicaSet detects the failure and immediately creates a new pod to maintain the desired state (e.g., 2 pods total).

---

### 🔗 Ties with Other Kubernetes Components:

| Component   | Role |
|-------------|------|
| **Pods**    | Containers that run the app |
| **ReplicaSet** | Maintains pod replicas |
| **Services** | Load balances traffic to pods |
| **Labels & Selectors** | Connect ReplicaSets, Pods, and Services |

---

### 🌐 Cluster Scenario:
- Two worker nodes.
- ReplicaSet spans across both nodes.
- Service sits in front of the pods to distribute traffic.
  
  This allows:
  - High availability
  - Horizontal scalability
  - Load balancing

---

### 📏 Scaling Options:

1. **Manually**:
   ```bash
   kubectl scale rs <replica-set-name> --replicas=5
   ```

2. **By YAML update**:
   Change the `replicas:` field in the ReplicaSet YAML.

3. **Automatically**:
   Enable **Horizontal Pod Autoscaler (HPA)**.

---

## 📄 YAML Manifest (Basics)

### Important fields:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: mycontainer
          image: nginx
```

- `replicas`: Number of pods to maintain
- `selector`: Matches pods based on labels
- `template`: Pod specification

---

Here’s a structured and detailed explanation of the lecture transcript, breaking down all the core Kubernetes concepts demonstrated: **Exposing a ReplicaSet as a Service, Load Balancing, High Availability, Scaling, and Cleanup**.

---

## 🌐 Exposing ReplicaSet as a Service

### 🎯 Objective:
To make a Kubernetes ReplicaSet accessible from a browser by exposing it through a Service of type `NodePort`.

### 🧩 Key Concepts:
- **ReplicaSet (RS)** ensures a specified number of pod replicas are running.
- **Service (NodePort)** exposes the pods to external traffic via a node IP and port.

### 🛠️ Command:
```bash
kubectl expose rs <replica-set-name> --type=NodePort --name=my-hello-world-rs-service --port=80 --target-port=8080
```

### ⚙️ Parameters Explained:
| Flag             | Description                                          |
|------------------|------------------------------------------------------|
| `--type=NodePort`| Makes the service accessible externally              |
| `--port`         | Service's port (ClusterIP)                           |
| `--target-port`  | Container port where the app is running (e.g., 8080) |
| `--name`         | Name of the service                                  |

---

## 🌍 Access the App via Browser

### 🔍 Steps:
1. Get the NodePort assigned:
   ```bash
   kubectl get svc
   ```
2. Get the public/external IP of the Node:
   ```bash
   kubectl get nodes -o wide
   ```
3. Access the app:  
   ```
   http://<node-external-ip>:<node-port>/hello
   ```

### 🔁 Load Balancing:
Refreshing the browser repeatedly hits different pods. Each pod prints its identifier, demonstrating Kubernetes’ round-robin load balancing across pod replicas.

---

## 🔁 High Availability with ReplicaSets

### 🔥 Simulation:
Manually delete a pod to simulate failure:
```bash
kubectl delete pod <pod-name>
```

### 🤖 What Happens:
Kubernetes automatically **creates a new pod** to maintain the desired number of replicas (e.g., 3 pods always).

### 🧪 Test:
```bash
kubectl get pods
```

### 🧠 Concept Recap:
**ReplicaSet provides self-healing** by ensuring that the actual number of pods matches the desired count defined in the YAML spec.

---

## 📈 Scaling ReplicaSet

### ⚙️ Manual Update:
Edit the YAML file (e.g., `replica-set-demo.yaml`) and change the replica count:
```yaml
spec:
  replicas: 6
```

Then apply the update:
```bash
kubectl replace -f replica-set-demo.yaml
```

### 🔍 Verify:
```bash
kubectl get rs
kubectl get pods
```

### 💡 Insight:
Scaling up/down happens dynamically. New pods are created or terminated automatically based on the desired count.

---

## 🧹 Cleanup Resources

### 🗑️ Delete ReplicaSet:
```bash
kubectl delete rs my-hello-world-rs
```

### 🗑️ Delete Service:
```bash
kubectl delete svc my-hello-world-rs-service
```

---

## 🖼️ Visual Summary

| Concept          | Description                                       |
|------------------|---------------------------------------------------|
| **Expose**       | `kubectl expose` makes pods accessible            |
| **NodePort**     | Opens a static port on each Node                  |
| **Load Balancing**| Kubernetes balances traffic across pod replicas  |
| **High Availability**| Pods auto-recreated if one fails              |
| **Scaling**      | Update replica count in YAML and apply            |
| **Cleanup**      | Use `kubectl delete` to remove RS and Service     |

---

Here’s a detailed and structured explanation of the Kubernetes **Deployment Demo** covered in the transcript you shared. I've broken it into sections with examples, command explanations, and diagrams where needed to make your study easier.

---

## 🚀 Deployment Demo with `kubectl`

### 📁 Directory: `04-deployments-with-kubectl/0401`

### ✅ Goals:
1. Create a Deployment.
2. Scale the Deployment.
3. Expose the Deployment as a Service.
4. Test features: rollout, rollback, pause/resume.

---

## 🔧 Step 1: Create a Deployment

**Command:**
```bash
kubectl create deployment my-first-deployment --image=stacksimplify/kube-nginx:1.0.0
```

**Explanation:**
- `kubectl create deployment`: Creates a Deployment object.
- `my-first-deployment`: Name of the deployment.
- `--image`: Docker image used (here, `kube-nginx` version 1.0.0 from StackSimplify).

**Check Deployment:**
```bash
kubectl get deployments
# or alias
kubectl get deploy
```

---

## 🔍 Step 2: Inspect the Deployment

**Command:**
```bash
kubectl describe deployment my-first-deployment
```

- Displays detailed information including:
  - Strategy
  - Current/desired replicas
  - Pod template
  - Events (e.g., scaled up replicas)

---

## 🔁 Step 3: Understand the Underlying ReplicaSet

**Command:**
```bash
kubectl get rs
# rs = ReplicaSet
```

When a Deployment is created, it creates a **ReplicaSet** behind the scenes.

---

## 🧱 Step 4: View the Pods

**Command:**
```bash
kubectl get pods
```

This shows the Pods created by the ReplicaSet.

---

## 📈 Step 5: Scale the Deployment

**Command:**
```bash
kubectl scale deployment my-first-deployment --replicas=20
```

- Increases the number of Pods to **20**.
- Check with:
```bash
kubectl get pods
```

You’ll see 20 running pods.

**To scale down:**
```bash
kubectl scale deployment my-first-deployment --replicas=10
```

- Terminates extra pods, only 10 remain active.

---

## 🌐 Step 6: Expose the Deployment as a Service

**Command:**
```bash
kubectl expose deployment my-first-deployment \
--type=NodePort \
--port=80 \
--target-port=80 \
--name=my-first-deployment-service
```

- `--type=NodePort`: Makes service accessible from outside the cluster.
- `--port`: Port on the service.
- `--target-port`: Port in the pod/container.

**View Service:**
```bash
kubectl get svc
# or
kubectl get svc my-first-deployment-service
```

You’ll get an external port (e.g., `32169`).

---

## 🌍 Step 7: Access the Application

Get the **Node IP**:

```bash
kubectl get nodes -o wide
```

Open in browser:

```
http://<NODE_IP>:<NodePort>
Example:
http://192.168.0.5:32169
```

You’ll see the **Nginx app** running (`v1` version).

---

## 📊 Diagram: Deployment Architecture

```
             +----------------------------+
             |      Deployment            |
             |  (my-first-deployment)     |
             +-------------+--------------+
                           |
                           v
               +-----------------------+
               |     ReplicaSet        |
               |    (1 per version)    |
               +----------+------------+
                          |
             +------------+------------+
             |            |            |
        +----v----+   +---v-----+  +---v-----+
        |  Pod    |   |  Pod    |  |  Pod    |
        | nginx:v1|   | nginx:v1|  | nginx:v1|
        +---------+   +---------+  +---------+

            <--- Accessible through Service --->

        +-------------------------------------+
        |           NodePort Service          |
        | my-first-deployment-service         |
        | Port 80 -> NodePort (e.g., 32169)   |
        +-------------------------------------+

```

---

## 📚 Summary

| Step | Command | Purpose |
|------|---------|---------|
| Create | `kubectl create deployment` | Define and run your app |
| View | `kubectl get deploy/pods/rs` | Inspect resources |
| Scale | `kubectl scale` | Increase or decrease replicas |
| Expose | `kubectl expose deployment` | Make app available externally |
| Access | `NODE_IP:NodePort` | Open app in browser |

---



1. **Updating the Deployment (V2 to V3)**
   - You demonstrate how to use `kubectl edit deployment` to manually update a deployment's image version, changing from version 2.0.0 to 3.0.0.
   - The process involves using `kubectl edit` followed by specifying the deployment name and directly editing the `image` tag in the deployment spec.
   - After making the changes, you save them, and Kubernetes automatically rolls out the updated version, terminating old pods and creating new ones with the new image.

2. **Rolling Back the Deployment**
   - You then explain how to roll back the deployment to a previous version. This can be done by using `kubectl rollout undo deployment`, which reverts the deployment to its previous state (from version 3.0.0 back to 2.0.0).
   - You show how to check the rollout history using `kubectl rollout history`, where you can observe the specific versions and changes made to the deployment, including the image versions used in each revision.
   - After rolling back, the deployment's revision is updated, and the history reflects this rollback.





1. **Why Pausing and Resuming Deployments**:
   - Pausing deployments helps when you need to make multiple changes (e.g., update the application version or modify resource limits) without immediately applying those changes to the live deployment.
   - If you apply changes without pausing, Kubernetes will immediately start recreating pods, which can be disruptive.

2. **Steps Involved**:
   - **Pausing the Deployment**: Use `kubectl rollout pause deployment <deployment-name>` to pause the deployment.
   - **Making Changes**: After pausing, you can make changes such as updating the application version (e.g., from V3 to V4) and applying CPU/memory limits.
   - **Verifying Current State**: Before resuming, check the rollout history and the number of replica sets to ensure no changes have been applied yet.
   - **Resuming the Deployment**: Use `kubectl rollout resume deployment <deployment-name>` to resume the deployment, which triggers the deployment process, creating new replica sets and rolling out changes.

3. **Resource Management**:
   - You can apply resource limits for containers using `kubectl set resources` command, specifying CPU and memory values to avoid overuse of cluster resources.

4. **Cleaning Up**:
   - After the deployment process, it’s important to clean up any resources created (e.g., delete the deployment and service) using `kubectl delete deployment <deployment-name>` and `kubectl delete service <service-name>`.

Kubernetes Services and demonstrates how to set up services like Cluster IP, NodePort, Load Balancer, Ingress, and External Name. Here's a summary of the key points from the video:

    Cluster IP: This is used for communication between applications within the same Kubernetes cluster. For example, a front-end and back-end application communicating internally would use Cluster IP. The IP range is internal to the Kubernetes cluster.

    NodePort: Used for accessing applications outside of the Kubernetes cluster. A NodePort exposes a service on each worker node's IP at a static port. This allows external access to the service.

    Load Balancer: Typically used in cloud environments, like AWS, to automatically provision a load balancer that routes traffic to the Kubernetes cluster. It integrates with cloud provider load balancing services like AWS's Elastic Load Balancer.

    Ingress: An advanced load balancer that enables HTTP-based routing, SSL/TLS termination, path-based routing, and more. It is useful for managing complex traffic routing to services within the cluster, similar to AWS's Application Load Balancer.

    External Name: This service type allows accessing externally hosted resources like databases by resolving a DNS name within the Kubernetes cluster. This is useful when your application inside Kubernetes needs to connect to an external database or service.

Demo Breakdown:

    Backend Service: A simple Spring Boot REST application (Hello World) is deployed in the Kubernetes cluster. A Cluster IP Service is created to expose this backend.

    Frontend Service: An Nginx pod is set up as the frontend, which will proxy requests to the backend's Cluster IP service. Additionally, a NodePort Service is deployed for external users to access the frontend application.