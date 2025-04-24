# **Comprehensive Study Material: AWS EKS & CLI Tools**  
**Based on Transcript (Step-by-Step Guide with Examples, Diagrams & Tables)**  

---

## **1. Introduction to AWS EKS**  
### **What is AWS EKS?**  
**Amazon Elastic Kubernetes Service (EKS)** is a fully managed Kubernetes service that automates cluster deployment, scaling, and maintenance.  

### **Core Components of EKS**  
| Component | Role | Managed by AWS? |  
|-----------|------|----------------|  
| **Control Plane** | Manages Kubernetes master nodes (API server, scheduler, etc.) | ✅ |  
| **Worker Nodes** | EC2 instances running application workloads (pods) | ❌ (User-managed) |  
| **Node Groups** | Logical group of worker nodes with identical configurations | ❌ (User-managed) |  
| **Fargate Profiles** | Serverless compute for pods (no EC2 management) | ✅ |  
| **VPC** | Isolated network for EKS resources | ❌ (User-configured) |  

![EKS Architecture](https://d1.awsstatic.com/product-marketing/EKS/eks-diagram.6c0c32550d0a5bf0a9a7d8a6a8d4a9a1e5e4f2a9.png)  

---

## **2. Three Essential CLI Tools for EKS**  
### **(A) AWS CLI**  
**Purpose:** Interact with AWS services (EKS, EC2, IAM).  

#### **Installation (Mac)**  
```bash
# Download & Install AWS CLI v2
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version  # Verify
```

#### **Configuration**  
```bash
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default Region (e.g., `us-east-1`)
# - Output Format (`json`)
```

**Example:** List EKS clusters  
```bash
aws eks list-clusters --region us-east-1
```

---

### **(B) kubectl**  
**Purpose:** Manage Kubernetes clusters (deploy apps, check nodes).  

#### **Installation (EKS-Optimized Version)**  
```bash
# Download kubectl for EKS
curl -O https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short
```

**Example Commands:**  
| Command | Purpose |  
|---------|---------|  
| `kubectl get nodes` | List worker nodes |  
| `kubectl get pods -A` | List all pods |  

---

### **(C) eksctl**  
**Purpose:** Simplify EKS cluster creation/deletion.  

#### **Installation (Mac/Homebrew)**  
```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version
```

#### **Key Features**  
- Auto-generates VPC, node groups, and IAM roles.  
- Supports Fargate and managed node groups.  

---

## **3. Creating an EKS Cluster**  
### **Step 1: Create Cluster (Control Plane)**  
```bash
eksctl create cluster \
  --name eks-demo-1 \
  --region us-east-1 \
  --zones us-east-1a,us-east-1b \
  --without-nodegroup  # Skip worker nodes initially
```
- Takes **15-20 mins** to provision the control plane.  

### **Step 2: Add Node Group (Worker Nodes)**  
```bash
eksctl create nodegroup \
  --cluster eks-demo-1 \
  --name ng-public-1 \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --ssh-access \
  --ssh-public-key kube-demo
```
**Flags Explained:**  
| Flag | Purpose |  
|------|---------|  
| `--node-type` | EC2 instance type (e.g., `t3.medium`) |  
| `--ssh-access` | Enable SSH for troubleshooting |  
| `--ssh-public-key` | Assign an EC2 key pair |  

### **Step 3: Verify Cluster**  
```bash
eksctl get cluster           # List clusters
kubectl get nodes -o wide   # Show worker node details
```

---

## **4. IAM & Security Setup**  
### **(A) IAM OIDC Provider**  
**Purpose:** Link Kubernetes service accounts to AWS IAM roles.  

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster eks-demo-1 \
  --region us-east-1 \
  --approve
```

### **(B) Worker Node IAM Policies**  
Attach policies for auto-scaling, ECR access, etc.:  
```bash
eksctl create nodegroup ... \
  --asg-access \
  --external-dns-access \
  --full-ecr-access
```
**Policies Added Automatically:**  
- **Auto Scaling** → Scale worker nodes.  
- **ECR Access** → Pull Docker images.  

---

## **5. Networking & VPC**  
### **Key Concepts**  
- **Public Subnets:** Worker nodes with public IPs.  
- **Private Subnets:** Isolated nodes (required for Fargate).  

**Example:** Modify security groups to allow traffic:  
```bash
# Allow all traffic (for testing)
aws ec2 authorize-security-group-ingress \
  --group-id sg-123456 \
  --protocol all \
  --cidr 0.0.0.0/0
```

---

## **6. Deleting the Cluster**  
### **Step 1: Delete Node Group**  
```bash
eksctl delete nodegroup \
  --cluster eks-demo-1 \
  --name ng-public-1
```

### **Step 2: Delete Cluster**  
```bash
eksctl delete cluster \
  --name eks-demo-1 \
  --region us-east-1
```
**Note:** NAT gateways and load balancers may incur costs if not deleted.  

---

## **7. Pricing & Cost Optimization**  
| Resource | Cost (us-east-1) |  
|----------|------------------|  
| EKS Control Plane | $0.10/hour (~$72/month) |  
| t3.medium Worker Node | ~$30/month |  
| Fargate | $0.0404/vCPU/hour + $0.0044/GB/hour |  

**Cost-Saving Tip:** Always delete clusters after labs!  

---

## **8. Troubleshooting**  
### **Common Issues**  
1. **Cluster Creation Fails**  
   - Check CloudFormation logs in AWS Console.  
2. **kubectl Connection Errors**  
   - Update `kubeconfig`:  
     ```bash
     aws eks update-kubeconfig --name eks-demo-1
     ```  

---

## **9. Summary Cheat Sheet**  
| Task | Command |  
|------|---------|  
| Install AWS CLI | `curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"` |  
| Create Cluster | `eksctl create cluster --name=my-cluster` |  
| Add Node Group | `eksctl create nodegroup --cluster=my-cluster` |  
| Delete Cluster | `eksctl delete cluster --name=my-cluster` |  

---
