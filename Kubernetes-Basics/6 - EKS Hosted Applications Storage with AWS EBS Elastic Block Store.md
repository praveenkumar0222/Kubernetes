Here's a detailed study material prepared from the provided transcript, covering AWS EKS storage concepts with explanations, examples, definitions, and visual aids:

---

# **AWS EKS Storage: Persistent Volumes with EBS CSI Driver**
## **Module 1: Introduction to EKS Storage**
### **1.1 Storage Drivers in EKS**
AWS EKS supports multiple storage drivers for persistent volumes:
- **EBS CSI Driver** (Primary focus)
- **EFS CSI Driver** (Elastic File System)
- **FSx for Lustre CSI Driver** (Windows file systems)

**Legacy Provisioner**: `in-tree EBS provisioner` (deprecated, being replaced by CSI drivers)

**Why CSI?**  
Container Storage Interface (CSI) is a standard for exposing storage systems to containerized workloads.  
- Decouples Kubernetes from storage backend implementations  
- Enables dynamic volume provisioning  

### **1.2 Key Concepts**
| Term | Definition | Example |
|------|-----------|---------|
| **StorageClass** | Defines a "class" of storage (e.g., SSD/HDD, provisioner) | `gp2`, `io1` |
| **PersistentVolume (PV)** | A piece of storage in the cluster | Dynamically created EBS volume |
| **PersistentVolumeClaim (PVC)** | User request for storage | Claims 4GB from a StorageClass |
| **Volume Binding** | How PVC binds to PV | `WaitForFirstConsumer` |

---

## **Module 2: EBS CSI Driver Setup**
### **2.1 IAM Policy for EBS CSI Driver**
**Purpose**: Grants permissions to manage EBS volumes from EKS worker nodes.

**Policy JSON** (Attached to worker node IAM role):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:DescribeVolumes"
      ],
      "Resource": "*"
    }
  ]
}
```

**Steps**:
1. Create IAM policy named `AmazonEBSCSIDriver`.
2. Attach to worker node role (found via `kubectl -n kube-system describe configmap aws-auth`).

### **2.2 Installing EBS CSI Driver**
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

**Verify Installation**:
```bash
kubectl get pods -n kube-system | grep ebs-csi
```
Expected output:
```
ebs-csi-controller-xxx    Running
ebs-csi-node-xxx          Running
```

---

## **Module 3: Dynamic Volume Provisioning**
### **3.1 StorageClass Manifest**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

**Key Parameters**:
- `provisioner`: `ebs.csi.aws.com` (EBS CSI Driver)
- `volumeBindingMode`:  
  - `Immediate`: Binds PVC immediately (may cause AZ mismatches).  
  - `WaitForFirstConsumer`: Binds when Pod is scheduled (ensures AZ affinity).

### **3.2 PersistentVolumeClaim (PVC)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

**Access Modes**:
- `ReadWriteOnce` (RWO): Single node read/write.  
- `ReadOnlyMany` (ROX): Multiple nodes read-only.  
- `ReadWriteMany` (RWX): Multi-node read/write (not supported by EBS).

---

## **Module 4: MySQL Deployment with Persistent Storage**
### **4.1 ConfigMap for DB Initialization**
Creates a `user_mgmt` schema automatically:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-management-db-creation-script
data:
  mysql_usermgmt.sql: |
    DROP DATABASE IF EXISTS user_mgmt;
    CREATE DATABASE user_mgmt;
```

### **4.2 MySQL Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  strategy:
    type: Recreate  # Required for stateful apps
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "dbpassword11"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        - name: db-creation-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: ebs-mysql-pv-claim
      - name: db-creation-script
        configMap:
          name: user-management-db-creation-script
```

**Key Points**:
- **Recreate Strategy**: Ensures clean restarts for databases.  
- **Volume Mounts**:  
  - `/var/lib/mysql` → EBS volume (persistent data).  
  - `/docker-entrypoint-initdb.d` → ConfigMap (initialization script).  

### **4.3 ClusterIP Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless service (direct pod IP)
  selector:
    app: mysql
  ports:
    - port: 3306
```

---

## **Module 5: Verification**
### **5.1 Check Resources**
```bash
kubectl get sc,pvc,pv,pods,svc
```
Expected output:
```
NAME                               PROVISIONER       AGE
storageclass.storage.k8s.io/ebs-sc ebs.csi.aws.com   5m

NAME                                  STATUS   VOLUME   CAPACITY   ACCESS MODES
persistentvolumeclaim/ebs-mysql-pvc   Bound    pvc-xxx  4Gi        RWO

NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
persistentvolume/pvc-xxx    4Gi        RWO            Delete           Bound    default/ebs-mysql-pvc

NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-5f6b8c6d4d-abcde   1/1     Running   0          2m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/mysql        ClusterIP   None         <none>        3306/TCP   2m
```

### **5.2 Connect to MySQL**
```bash
kubectl run -it --rm --restart=Never mysql-client \
  --image=mysql:5.6 \
  -- mysql -h mysql -pdbpassword11
```

**Verify Schema**:
```sql
SHOW DATABASES;  # Should list `user_mgmt`
```

---

## **Visual Summary**
### **Architecture Diagram**
```
+-----------------------+
|       EKS Cluster     |
| +-------------------+ |
| | MySQL Pod         | |
| | - Uses EBS Volume | |
| +-------------------+ |
|                       |
|  EBS CSI Driver       |
|  (Manages EBS Volumes)|
+-----------------------+
          |
          v
+-----------------------+
|   Amazon EBS Volume   |
| - 4GB GP2 SSD        |
| - Persists Data      |
+-----------------------+
```

### **Binding Flow**
1. PVC created → Waits (`WaitForFirstConsumer`).  
2. Pod scheduled → EBS volume provisioned in the same AZ.  
3. Pod binds to PVC → Volume attached to the node.  

---

## **Troubleshooting**
| Issue | Solution |
|-------|----------|
| PVC stuck in "Pending" | Check IAM permissions or StorageClass. |
| Pod fails to start | Verify `docker-entrypoint-initdb.d` script permissions. |
| AZ mismatch errors | Use `WaitForFirstConsumer` in StorageClass. |

---



Here's a comprehensive study material prepared from the provided transcripts, covering Kubernetes EBS CSI driver, microservice deployment, and related concepts:

---

## **Section 1: EBS CSI Driver References**
### **1.1 EBS CSI Driver Overview**
- **Definition**: The EBS CSI (Container Storage Interface) driver allows Kubernetes to manage Amazon EBS volumes for persistent storage.
- **Key Features**:
  - Static & Dynamic Provisioning
  - Volume Snapshots (Alpha)
  - Volume Resizing (Alpha)
  - Block Volume Support (Beta)
  - NVMe Support

### **1.2 Feature Stability**
| Feature           | Stability  | Production Ready? |
|-------------------|------------|-------------------|
| Dynamic Provisioning | Beta      | ✅ Yes           |
| Volume Snapshots  | Alpha      | ❌ No            |
| Volume Resizing   | Alpha      | ❌ No            |
| Block Volumes     | Beta       | ✅ Yes           |

📌 **Note**: Alpha features may break or change; avoid in production.

### **1.3 Example: Dynamic Provisioning**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
  encrypted: "true"
```

🔹 **Explanation**:
- `provisioner: ebs.csi.aws.com` → Uses AWS EBS CSI driver.
- `volumeBindingMode: WaitForFirstConsumer` → Volume created only when a Pod uses it.

---

## **Section 2: User Management Microservice Deployment**
### **2.1 Deployment YAML Breakdown**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-mgmt-microservice
  labels:
    app: user-mgmt-rest-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-mgmt-rest-app
  template:
    metadata:
      labels:
        app: user-mgmt-rest-app
    spec:
      containers:
      - name: user-mgmt-rest-app
        image: kubesimplify/user-mgmt-microservice:1.0.0
        ports:
        - containerPort: 8095
        env:
        - name: DB_HOST
          value: "mysql"  # Points to MySQL Service
        - name: DB_PORT
          value: "3306"
```

🔹 **Key Components**:
1. **Replicas**: Defines how many Pods run (here: `1`).
2. **Selector**: Ensures Deployment manages Pods with label `app: user-mgmt-rest-app`.
3. **Environment Variables**:
   - `DB_HOST`: Points to MySQL service (`mysql` resolves via K8s DNS).
   - `DB_PORT`: Default MySQL port.

---

### **2.2 NodePort Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-mgmt-service
spec:
  type: NodePort
  selector:
    app: user-mgmt-rest-app  # Matches Pod label
  ports:
    - port: 8095      # Service port
      targetPort: 8095 # Pod port
      nodePort: 31231  # External access port
```

🔹 **Explanation**:
- **NodePort**: Exposes the service on a static port (e.g., `31231`) on each worker node.
- **Access URL**: `http://<WorkerNode-IP>:31231/user-mgmt/health-status`

---

## **Section 3: Testing with Postman**
### **3.1 API Endpoints**
| Endpoint                     | Method | Purpose                     |
|------------------------------|--------|-----------------------------|
| `/user-mgmt/health-status`   | GET    | Check service health        |
| `/user-mgmt/user`            | POST   | Create user (JSON body)     |
| `/user-mgmt/user`            | GET    | List all users              |
| `/user-mgmt/user/{username}` | DELETE | Delete user by username     |

### **3.2 Example: Create User**
**Request**:
```json
POST http://<IP>:31231/user-mgmt/user
Body:
{
  "username": "admin1",
  "email": "admin1@test.com",
  "role": "admin",
  "enabled": true,
  "firstName": "Fname1",
  "lastName": "Lname1",
  "password": "pass123"
}
```

**Response**:
```json
{
  "status": "User created but notification failed",
  "error": "Notification service unreachable"
}
```
📌 **Note**: Notification service is optional (not deployed here).

---

## **Section 4: Database Validation**
### **4.1 Connecting to MySQL**
```bash
kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h mysql -p
```
**Commands**:
```sql
SHOW DATABASES;
USE user_mgmt_schema;
SELECT * FROM users;
```

### **4.2 Expected Output**
| username | email            | role  | firstName | lastName |
|----------|------------------|-------|-----------|----------|
| admin1   | admin1@test.com  | admin | Fname1    | Lname1   |

---

## **Section 5: Problem & Solution**
### **5.1 Issue Observed**
- **Problem**: User microservice starts before MySQL is ready → crashes and restarts.
- **Cause**: No dependency management between containers.

### **5.2 Solution: Init Containers**
```yaml
spec:
  initContainers:
  - name: wait-for-mysql
    image: busybox
    command: ['sh', '-c', 'until nc -z mysql 3306; do echo "Waiting for MySQL"; sleep 2; done']
  containers:
  - name: user-mgmt-rest-app
    image: kubesimplify/user-mgmt-microservice:1.0.0
```

🔹 **How It Works**:
1. `initContainer` runs first and checks if MySQL is reachable (`nc -z mysql 3306`).
2. Only after MySQL is ready, the main app container starts.

---

## **Section 6: Cleanup**
```bash
kubectl delete -f kube-manifest/
```
**Verification**:
```bash
kubectl get all          # No resources left
kubectl get pv,pvc,sc    # No persistent volumes
```

---

## **Visual Summary**
### **Architecture Diagram**
```
[User Microservice Pod]
  ├── Init Container (waits for MySQL)
  └── Main Container (user-mgmt-app)
       └── Connects to MySQL Service
             └── Uses EBS Volume (PVC)
```

### **Flow of Requests**
1. User → NodePort (31231) → Service → Pod (8095)
2. Pod → MySQL Service → Persistent Data (EBS)

---

