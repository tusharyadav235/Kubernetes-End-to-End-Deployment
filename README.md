# 🚀 EmpTrack — Employee CRUD App on Amazon EKS

A production-ready full-stack CRUD application built with:
- **Frontend**: HTML + CSS + Vanilla JS (served by Nginx)
- **Backend**: Java 17 + Spring Boot 3.2
- **Database**: MySQL 8.0
- **Container**: Docker (multi-stage builds)
- **Orchestration**: Kubernetes on Amazon EKS

---

## 📁 Project Structure

```
k8s-crud-project/
├── frontend/
│   ├── index.html          # Single-page UI
│   ├── nginx.conf          # Nginx reverse proxy config
│   └── Dockerfile          # Nginx-based image
│
├── backend/
│   ├── src/
│   │   └── main/java/com/crud/app/
│   │       ├── EmployeeApplication.java
│   │       ├── controller/EmployeeController.java
│   │       ├── model/Employee.java
│   │       ├── repository/EmployeeRepository.java
│   │       └── service/EmployeeService.java
│   ├── pom.xml
│   └── Dockerfile          # Multi-stage Maven build
│
├── k8s/
│   ├── 01-mysql-secret.yaml
│   ├── 02-mysql-deployment.yaml
│   ├── 03-backend-deployment.yaml
│   ├── 04-frontend-deployment.yaml
│   ├── 05-ingress.yaml     # AWS ALB Ingress
│   └── 06-hpa.yaml         # HorizontalPodAutoscaler
│
├── docker-compose.yml      # Local development
└── README.md
```

---

## 🧪 Run Locally (Docker Compose)

```bash
# Clone / unzip the project
cd k8s-crud-project

# Start all services
docker-compose up --build

# Access app at:  http://localhost:3000
# API health:     http://localhost:8080/api/employees/health
```

---

## ☁️ Deploy to Amazon EKS — Step by Step

### Prerequisites
| Tool | Install |
|------|---------|
| AWS CLI | `brew install awscli` or https://aws.amazon.com/cli |
| eksctl | `brew tap weaveworks/tap && brew install eksctl` |
| kubectl | `brew install kubectl` |
| Docker | https://docker.com |

---

### Step 1: Create EKS Cluster

```bash
eksctl create cluster \
  --name emptrack-cluster \
  --region ap-south-1 \
  --nodegroup-name ng-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# Verify
kubectl get nodes
```

---

### Step 2: Create ECR Repositories

```bash
# Set your AWS account ID
AWS_ACCOUNT=<your-aws-account-id>
REGION=ap-south-1

# Create repos
aws ecr create-repository --repository-name employee-backend --region $REGION
aws ecr create-repository --repository-name employee-frontend --region $REGION

# Login to ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com
```

---

### Step 3: Build & Push Docker Images

```bash
ECR_URI=$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com

# Backend
docker build -t employee-backend ./backend
docker tag employee-backend:latest $ECR_URI/employee-backend:latest
docker push $ECR_URI/employee-backend:latest

# Frontend
docker build -t employee-frontend ./frontend
docker tag employee-frontend:latest $ECR_URI/employee-frontend:latest
docker push $ECR_URI/employee-frontend:latest
```

---

### Step 4: Update K8s Manifests with ECR URIs

In `k8s/03-backend-deployment.yaml` and `k8s/04-frontend-deployment.yaml`,
replace `YOUR_ECR_URI` with your actual ECR URI:

```yaml
image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/employee-backend:latest
```

---

### Step 5: Install AWS Load Balancer Controller

```bash
# Create IAM policy for ALB controller
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create service account
eksctl create iamserviceaccount \
  --cluster=emptrack-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=emptrack-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### Step 6: Deploy Everything

```bash
# Apply manifests in order
kubectl apply -f k8s/01-mysql-secret.yaml
kubectl apply -f k8s/02-mysql-deployment.yaml
kubectl apply -f k8s/03-backend-deployment.yaml
kubectl apply -f k8s/04-frontend-deployment.yaml
kubectl apply -f k8s/05-ingress.yaml
kubectl apply -f k8s/06-hpa.yaml
```

---

### Step 7: Get Your App URL

```bash
# Wait for ALB to provision (1-3 minutes)
kubectl get ingress app-ingress

# Output will show the ALB DNS name:
# NAME          CLASS   HOSTS   ADDRESS                                    PORTS
# app-ingress   alb     *       k8s-default-xxx.ap-south-1.elb.amazonaws.com   80
```

Open the **ADDRESS** in your browser — your app is live! 🎉

---

## 🔍 Useful kubectl Commands

```bash
# Check all pods
kubectl get pods

# Check logs
kubectl logs -l app=backend --tail=50
kubectl logs -l app=frontend --tail=50

# Scale manually
kubectl scale deployment backend --replicas=3

# Watch HPA
kubectl get hpa --watch

# Describe ingress
kubectl describe ingress app-ingress
```

---

## 📡 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/employees` | Get all employees |
| GET | `/api/employees/{id}` | Get employee by ID |
| POST | `/api/employees` | Create employee |
| PUT | `/api/employees/{id}` | Update employee |
| DELETE | `/api/employees/{id}` | Delete employee |
| GET | `/api/employees/department/{dept}` | Filter by department |
| GET | `/api/employees/health` | Health check |

---

## 🧹 Cleanup

```bash
# Delete K8s resources
kubectl delete -f k8s/

# Delete EKS cluster (saves cost!)
eksctl delete cluster --name emptrack-cluster --region ap-south-1
```
