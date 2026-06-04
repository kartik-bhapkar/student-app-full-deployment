# Student Management Application Deployment on AWS EKS

## Project Overview

This project demonstrates the deployment of a full-stack Student Management Application on Amazon EKS (Elastic Kubernetes Service).

The application consists of:

* React Frontend
* Spring Boot Backend
* MariaDB Database
* AWS EKS Cluster
* AWS EBS Persistent Storage
* NGINX Ingress Controller
* Horizontal Pod Autoscaler (HPA)

---

# Architecture

```text
Internet
    |
AWS Load Balancer
    |
NGINX Ingress Controller
    |
------------------------------------------------
|                       |                      |
Frontend Service    Backend Service      DB Service (MariaDB)
                                               |
                                          Amazon EBS
```

---

# Prerequisites

* AWS Account
* AWS CLI
* kubectl
* Docker
* Git
* Internet Access

---

# Phase 1 - EKS Cluster Creation

## Step 1 - Create EKS Cluster using AWS Console

1. Open AWS Console
2. Navigate to EKS
3. Click Create Cluster

### Cluster Configuration

| Parameter          | Value            |
| ------------------ | ---------------- |
| Cluster Name       | k-cluster        |
| Region             | ap-south-1       |
| Endpoint Access    | Public + Private |
| Kubernetes Version | Latest Stable    |

### Cluster IAM Role

Attach:

* AmazonEKSClusterPolicy

Create the cluster and wait until status becomes:

```text
ACTIVE
```

---

## Step 2 - Create Node Group

After cluster creation:

1. Open Cluster
2. Compute Tab
3. Add Node Group

### Node Group Configuration

| Parameter     | Value          |
| ------------- | -------------- |
| Name          | n-group        |
| Instance Type | c7i-flex.large |
| Desired Size  | 2              |
| Min Size      | 2              |
| Max Size      | 2              |

### Node IAM Role Policies

Attach:

* AmazonEKSWorkerNodeMinimalPolicy
* AmazonEKSComputePolicy
* AmazonEC2ContainerRegistryPullOnly
* AmazonElasticContainerRegistryPublicReadOnly

Wait until Node Group status becomes:

```text
ACTIVE
```

---

# Phase 2 - Configure kubectl

Update kubeconfig:

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name k-cluster
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME           STATUS
node-1         Ready
node-2         Ready
```

---

# Phase 3 - Install eksctl

Install:

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

Verify:

```bash
eksctl version
```

---

# Phase 4 - Install Helm

Install:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# Phase 5 - Install EBS CSI Driver

Add Helm Repository:

```bash
helm repo add aws-ebs-csi-driver \
https://kubernetes-sigs.github.io/aws-ebs-csi-driver
```

Update Repository:

```bash
helm repo update
```

Install Driver:

```bash
helm upgrade --install aws-ebs-csi-driver \
aws-ebs-csi-driver/aws-ebs-csi-driver \
--namespace kube-system
```

Verify:

```bash
kubectl get pods -n kube-system
```

Expected:

```text
ebs-csi-controller Running
ebs-csi-node Running
```

---

# Phase 6 - Install NGINX Ingress Controller

Install:

```bash
kubectl apply -f \
https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

Verify:

```bash
kubectl get svc -n ingress-nginx
```

Expected:

```text
ingress-nginx-controller   LoadBalancer
```

Wait for ELB creation.

---

# Phase 7 - Database Deployment

## Database Components

```text
secret.yaml
ebs-storageclass.yaml
student-db-pvc.yaml
statefulset.yaml
service.yaml
```

Apply:

```bash
kubectl apply -f secret.yaml
kubectl apply -f ebs-storageclass.yaml
kubectl apply -f student-db-pvc.yaml
kubectl apply -f statefulset.yaml
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get pvc
kubectl get pods
kubectl get svc
```

Expected:

```text
PVC Bound
student-db-sts Running
student-db-svc Created
```

---

# Phase 8 - Backend Deployment

## application.properties Changes

Before:

```properties
spring.datasource.url=jdbc:mariadb://localhost:3306/student_db
spring.datasource.password=redhat
```

After:

```properties
spring.datasource.url=jdbc:mariadb://student-db-svc:3306/${DB_NAME}
spring.datasource.password=${DB_PASSWORD}
```

Using Kubernetes Secrets for database configuration.

---

## Backend Dockerfile

```dockerfile
FROM maven:3.8.3-openjdk-17 AS builder

WORKDIR /app

COPY . .

RUN mvn clean package

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=builder \
/app/target/student-registration-backend-0.0.1-SNAPSHOT.jar \
app.jar

EXPOSE 8080

ENTRYPOINT ["java","-jar","app.jar"]
```

---

## Build Backend Image

```bash
docker build -t <dockerhub-username>/backend:latest .
```

---

## Push Backend Image

```bash
docker push <dockerhub-username>/backend:latest
```

---

## Backend Kubernetes Resources

```text
deployment.yaml
service.yaml
hpa.yaml
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
kubectl get hpa
```

---

# Phase 9 - Frontend Deployment

## Update .env

Before:

```env
VITE_API_URL=http://BACKEND_IP:8080/api
```

After:

```env
VITE_API_URL=/api
```

Reason:

Ingress will handle routing to backend services.

---

## Frontend Dockerfile

```dockerfile
FROM node:20 AS builder

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist/. /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

---

## Build Frontend Image

```bash
docker build -t <dockerhub-username>/frontend:latest .
```

---

## Push Frontend Image

```bash
docker push <dockerhub-username>/frontend:latest
```

---

## Frontend Kubernetes Resources

```text
deployment.yaml
service.yaml
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
```

---

# Phase 10 - Ingress Configuration

Create:

```text
ingress.yaml
```

Ingress Routes:

```text
/      -> student-frontend-svc:80
/api   -> student-app-svc:8080
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl describe ingress student-app-ingress
```

Expected:

```text
/      student-frontend-svc:80
/api   student-app-svc:8080
```

---

# Phase 11 - Application Validation

Check all resources:

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl get hpa
kubectl get ingress
```

Retrieve Load Balancer DNS:

```bash
kubectl get svc -n ingress-nginx
```

Open:

```text
http://<LOAD_BALANCER_DNS>
```

Expected:

* Frontend loads successfully
* User registration works
* Backend APIs respond successfully
* Data is stored in MariaDB
* Persistent storage attached through EBS

---

# Troubleshooting

## Node Group Stuck in Creating

Cause:

```text
UnauthorizedOperation
ec2:DescribeInstances
```

Fix:

Attach:

```text
AmazonEKSComputePolicy
```

to Node IAM Role.

---

## ImagePullBackOff

Cause:

Docker image not pushed.

Fix:

```bash
docker push <image>
```

---

## CrashLoopBackOff

Cause:

Database configuration or Secret issues.

Fix:

Verify Secret references and environment variables.

---

## Frontend White Screen

Cause:

Incorrect Docker COPY command.

Incorrect:

```dockerfile
COPY --from=builder /app/dist/* /usr/share/nginx/html
```

Correct:

```dockerfile
COPY --from=builder /app/dist/. /usr/share/nginx/html/
```

---

## API Connection Refused

Cause:

Frontend using backend node IP.

Incorrect:

```env
VITE_API_URL=http://BACKEND_IP:8080/api
```

Correct:

```env
VITE_API_URL=/api
```

with Ingress routing.

---

# Final Outcome

Successfully deployed a production-style application on AWS EKS using:

* Amazon EKS
* Managed Node Groups
* Kubernetes Deployments
* StatefulSets
* Persistent Volumes
* EBS CSI Driver
* Kubernetes Secrets
* Horizontal Pod Autoscaler
* NGINX Ingress Controller
* Docker
* React
* Spring Boot
* MariaDB

This project demonstrates end-to-end Kubernetes deployment, storage management, networking, autoscaling, and troubleshooting on AWS.
