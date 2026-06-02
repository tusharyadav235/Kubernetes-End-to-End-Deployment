# EmpTrack — Full Stack DevOps Project on Amazon EKS

A production-ready Employee Management CRUD application built with 
industry-standard DevOps practices covering containerization, 
Kubernetes orchestration, CI/CD automation, and real-time monitoring.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML, CSS, JavaScript, Nginx |
| Backend | Java 17, Spring Boot 3.2, REST API |
| Database | AWS RDS MySQL 8.0 |
| Containerization | Docker (multi-stage builds) |
| Orchestration | Kubernetes, Amazon EKS (EC2) |
| CI/CD | Jenkins (declarative pipeline) |
| Ingress | AWS ALB Ingress Controller |
| Auto Scaling | Horizontal Pod Autoscaler (HPA) |
| Monitoring | Prometheus + Grafana (Helm) |
| Registry | Docker Hub |

---

## Architecture
GitHub Push → Jenkins Pipeline
↓
Build Docker Images
↓
Push to Docker Hub
↓
Deploy to EKS (kubectl)
↓
AWS ALB routes traffic:
/api/* → Spring Boot backend
/      → Nginx frontend
↓
Backend → AWS RDS MySQL
↓
Prometheus scrapes metrics
↓
Grafana displays dashboards

---

## Project Structure
emptrack/
├── frontend/
│   ├── index.html
│   ├── nginx.conf
│   └── Dockerfile
├── backend/
│   ├── src/main/java/com/crud/app/
│   │   ├── controller/
│   │   ├── model/
│   │   ├── repository/
│   │   └── service/
│   ├── pom.xml
│   └── Dockerfile
├── k8s/
│   ├── crud_mysecret.yml 
│   ├── crudebackend.yml
│   ├── crudfrontendservice.yml 
│   ├── crudingress.yml 
│  
├── Jenkinsfile
├── docker-compose.yml
└── README.md

---

## Run Locally

```bash
git clone https://github.com/tusharyadav235/Kubernetes-End-to-End-Deployment.git
cd Kubernetes-End-to-End-Deployment

docker-compose up --build
# App: http://localhost:3000
# API: http://localhost:8080/api/employees/health
```

---

## Deploy to EKS

```bash
# 1. Create cluster
eksctl create cluster \
  --name emptrack-cluster \
  --region us-east-1 \
  --node-type t3.medium \
  --nodes 2 --managed

# 2. Connect kubectl
aws eks update-kubeconfig --region us-east-1 --name emptrack-cluster

# 3. Create namespace and secret
kubectl create namespace emptrack
kubectl create secret generic mysql-secret -n emptrack \
  --from-literal=DB_HOST=your-rds-endpoint.rds.amazonaws.com \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_NAME=employeedb1 \
  --from-literal=DB_USER=empuser \
  --from-literal=DB_PASSWORD=yourpassword
# 4. Create serviceaccount
eksctl create iamserviceaccount \
  --cluster emptrack-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<Account id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve 
# 5. Install ALB Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=emptrack-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \

# 6. Deploy app
kubectl apply -f k8s/ -n emptrack

# 7. Get ALB URL
kubectl get ingress -n emptrack
```

---

## CI/CD Pipeline

Jenkins runs on Docker on EC2. Every git push triggers:

Pull code from GitHub
Build backend + frontend Docker images
Tag with build number for versioning
Push to Docker Hub
kubectl set image → rolling deploy to EKS
Verify rollout status


---

## Monitoring

```bash
# Install Prometheus + Grafana
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# http://localhost:3000 (admin/admin123)
```

Import dashboard IDs: `15760` (cluster), `1860` (nodes), `6417` (pods)

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/employees` | Get all |
| POST | `/api/employees` | Create |
| PUT | `/api/employees/{id}` | Update |
| DELETE | `/api/employees/{id}` | Delete |
| GET | `/api/employees/health` | Health check |

---

## Cleanup

```bash
kubectl delete -f k8s/ -n emptrack
eksctl delete cluster --name emptrack-cluster --region us-east-1
# Delete RDS from AWS Console
```

---

**Author: Tushar Yadav**
github.com/tusharyadav235
