# StreamingApp - Container Orchestration, Jenkins CI/CD & GitOps Deployment on AWS EKS

This repository contains the complete container orchestration, continuous integration, packaging, and GitOps continuous delivery setup for **StreamingApp**, a multi-service MERN video streaming platform.

The entire stack is deployed on Amazon EKS with AWS Application Load Balancer (ALB) dynamic path-based routing, parameterized via a modular Helm chart, automated via Jenkins CI pipelines, connected to an external MongoDB database, and continuously synchronized by Argo CD.

**Repository:** [https://github.com/SakarayAnilKumar/StreamingApp](https://github.com/SakarayAnilKumar/StreamingApp)

## Architecture Overview

The system consists of 5 core microservices connected to a managed External MongoDB instance:

| **Service** | **Technology** | **Port** | **Ingress Path** | **Responsibility** | 
| --- | --- | --- | --- | --- |
| **`frontend`** | React SPA / Nginx | 80 | `/` | User UI, video playback interface, admin portal, chat UI | 
| **`authService`** | Node.js / Express | 3001 | `/api/auth` | User registration, authentication, JWT issuing & verification | 
| **`streamingService`** | Node.js / Express | 3002 | `/api/streaming` | Video catalogue management, S3 media stream URLs | 
| **`adminService`** | Node.js / Express | 3003 | `/api/admin` | Asset management, signed AWS S3 video & thumbnail uploads | 
| **`chatService`** | Node.js / Express / Socket.IO | 3004 | `/api/chat` & `/socket.io` | Real-time WebSocket and REST live chat broadcast | 
| **`Database`** | External MongoDB | 27017 / Cloud | External URI | Persistence store for users, video metadata, and chat logs | 

## Prerequisites & Required Tooling

Ensure the following tools are installed and configured before executing deployment steps:

* **Docker Desktop / Engine** (v24.0+)
* **AWS CLI** (v2.x) configured with administrative credentials (`aws configure`)
* **Jenkins Automation Server** with Docker, AWS Credentials Plugin, and Pipeline plugins
* **`kubectl`** (v1.28+)
* **Helm 3** (`helm version`)
* **`eksctl`** (v0.160+)
* **Argo CD** deployed on EKS cluster
* **External MongoDB Instance** (MongoDB Atlas or self-hosted external instance)

## Step-by-Step Implementation Guide

### Phase 1: Continuous Integration with Jenkins CI Pipeline

The application utilizes a declarative `Jenkinsfile` located at the root of the repository to automate image builds and image registry management upon code pushes.

#### Pipeline Workflow & Architecture

1. **Source Code Checkout:** Pulls the latest commit from `https://github.com/SakarayAnilKumar/StreamingApp`.
2. **AWS ECR Authentication:** Uses stored Jenkins AWS Credentials to authenticate Docker against Amazon ECR.
3. **Multi-Service Container Build:** Builds production Docker images for `authService`, `streamingService`, `adminService`, `chatService`, and `frontend`.
4. **Tagging & Image Push:** Tags each image with the build number (`${BUILD_NUMBER}`) and `latest`, then pushes artifacts to Amazon ECR repositories.

```groovy
// Jenkinsfile Architecture Summary
pipeline {
    agent any
    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '972011045520'
        REGISTRY_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/SakarayAnilKumar/StreamingApp.git'
            }
        }
        stage('ECR Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-ecr-credentials', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URI}'
                }
            }
        }
        stage('Build & Push Microservices') {
            steps {
                sh '''
                    # Auth Service
                    docker build -t ${REGISTRY_URI}/streaming-auth:${IMAGE_TAG} backend/authService
                    docker push ${REGISTRY_URI}/streaming-auth:${IMAGE_TAG}

                    # Streaming Service
                    docker build -t ${REGISTRY_URI}/streaming-stream:${IMAGE_TAG} -f backend/streamingService/Dockerfile backend
                    docker push ${REGISTRY_URI}/streaming-stream:${IMAGE_TAG}

                    # Admin Service
                    docker build -t ${REGISTRY_URI}/streaming-admin:${IMAGE_TAG} -f backend/adminService/Dockerfile backend
                    docker push ${REGISTRY_URI}/streaming-admin:${IMAGE_TAG}

                    # Chat Service
                    docker build -t ${REGISTRY_URI}/streaming-chat:${IMAGE_TAG} -f backend/chatService/Dockerfile backend
                    docker push ${REGISTRY_URI}/streaming-chat:${IMAGE_TAG}
                '''
            }
        }
    }
}
```

---

### Phase 2: Helm Chart Packaging & Architecture

The Kubernetes manifests are organized into a modular Helm chart under `charts/streamingapp` (or `./streamingapp`).

```text
charts/streamingapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── configmap.yaml
    ├── secret.yaml
    ├── deployments.yaml
    ├── services.yaml
    └── ingresses.yaml
```

#### Shared AWS ALB Ingress Strategy

All microservices share a single AWS Application Load Balancer by utilizing the `alb.ingress.kubernetes.io/group.name: streamingapp` annotation in `templates/ingresses.yaml`.

---

### Phase 3: AWS EKS Provisioning & ALB Controller Setup

```powershell
# 1. Fetch AWS Account ID
$ACCOUNT_ID = (aws sts get-caller-identity --query "Account" --output text)

# 2. Create EKS Cluster using eksctl
eksctl create cluster `
  --name streamingapp-assignment `
  --region us-east-1 `
  --version 1.34 `
  --nodegroup-name standard-workers `
  --node-type t3.medium `
  --nodes 2 `
  --managed --with-oidc

# 3. Associate IAM OIDC Provider for the cluster
eksctl utils associate-iam-oidc-provider --cluster streamingapp-assignment --region us-east-1 --approve

# 4. Create IAM Policy for AWS Load Balancer Controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# 5. Create IAM ServiceAccount for AWS Load Balancer Controller
eksctl create iamserviceaccount `
  --cluster=streamingapp-assignment `
  --namespace=kube-system `
  --name=aws-load-balancer-controller `
  --role-name AmazonEKSLoadBalancerControllerRole `
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy `
  --approve

# 6. Install AWS Load Balancer Controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=streamingapp-assignment `
  --set serviceAccount.create=false `
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### Phase 4: Cluster Secret Configuration & Initial Helm Installation

To securely connect to the **External MongoDB database** and AWS S3 without exposing credentials in Git, create Kubernetes Secrets directly in the target namespace:

```powershell
# 1. Create target namespace
kubectl create namespace streaming

# 2. Create Kubernetes Secret containing External MongoDB URI & AWS Credentials
kubectl create secret generic streaming-secrets -n streaming `
  --from-literal=mongo-uri="<MONGO_URI>" `
  --from-literal=jwt-secret="lab-jwt-change-me" `
  --from-literal=aws-access-key-id="<ACCESSKEY>" `
  --from-literal=aws-secret-access-key="<SECRET_KEY>" `
  --from-literal=aws-region="us-east-1" `
  --from-literal=aws-s3-bucket="streamingapp-lab-anil"

# 3. Verify Helm template and execute initial release deployment
helm template streamingapp ./streamingapp --namespace streaming
helm upgrade --install streamingapp ./streamingapp --namespace streaming --create-namespace
```

---

### Phase 5: Dynamic Frontend Build & Backend Alignment

Once AWS provisions the Application Load Balancer dynamically, fetch its generated DNS endpoint, inject it into the frontend React build parameters, and deploy:

```powershell
# 1. Retrieve provisioned ALB DNS endpoint
$ALB_DNS = (kubectl get ingress -n streaming -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
echo "Provisioned ALB Endpoint: http://$ALB_DNS"

# 2. Login to Amazon ECR
$region = "us-east-1"
$ecr = "972011045520.dkr.ecr.us-east-1.amazonaws.com"
aws ecr get-login-password --region $region | docker login --username AWS --password-stdin $ecr

# 3. Build & push frontend image with dynamic ALB URL
cd ..\..\StreamingApp\frontend\

docker build --no-cache `
  --build-arg REACT_APP_AUTH_API_URL=http://$ALB_DNS/api `
  --build-arg REACT_APP_STREAMING_API_URL=http://$ALB_DNS/api `
  --build-arg REACT_APP_STREAMING_PUBLIC_URL=http://$ALB_DNS `
  --build-arg REACT_APP_ADMIN_API_URL=http://$ALB_DNS/api/admin `
  --build-arg REACT_APP_CHAT_API_URL=http://$ALB_DNS/api/chat `
  --build-arg REACT_APP_CHAT_SOCKET_URL=http://$ALB_DNS `
  -t 972011045520.dkr.ecr.us-east-1.amazonaws.com/streaming-frontend:v0.1.0 .

docker push 972011045520.dkr.ecr.us-east-1.amazonaws.com/streaming-frontend:v0.1.0

# 4. Trigger rolling restart of frontend deployment
kubectl rollout restart deployment/frontend -n streaming

# 5. Verify dynamic API endpoint injection inside container
kubectl exec -n streaming deploy/frontend -- sh -c "grep -oE 'http://[^""''']+' /usr/share/nginx/html/static/js/*.js | sort -u"
```

---

### Phase 6: GitOps Continuous Delivery Setup with Argo CD

Argo CD handles declarative continuous deployment from the GitHub repository to the EKS cluster.

```powershell
# 1. Install Argo CD on EKS
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Wait for Argo CD pods to enter Running state
kubectl get pods -n argocd -w

# 3. Retrieve initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# 4. Expose Argo CD Server locally
kubectl port-forward svc/argocd-server -n argocd 8084:443
```

#### Argo CD Application Spec (`argocd-application.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: streamingapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://github.com/SakarayAnilKumar/StreamingApp.git'
    targetRevision: HEAD
    path: charts/streamingapp
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: streaming
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply the manifest to initialize automated sync:

```bash
kubectl apply -f argocd-application.yaml
```

---

### Phase 7: Scaling & Zero-Downtime Rolling Update Demonstration

```bash
# 1. Manual replica scaling test
kubectl scale deployment/streaming --replicas=4 -n streaming

# 2. Perform zero-downtime rolling update via Helm
helm upgrade streamingapp ./charts/streamingapp --namespace streaming --set services.auth.tag="v0.1.1"

# 3. Verify zero-downtime rollout status
kubectl rollout status deployment/auth -n streaming
```

---

## Verification & Proof Deliverables

Attach screenshots corresponding to the following placeholders in your submission report:

### Placeholder 1: Jenkins CI Pipeline Execution
> **Description:** Screenshot of the Jenkins Dashboard / Pipeline Stage View showing successful completion of all pipeline stages (Checkout, ECR Authentication, Build & Push of microservice Docker images).
> <img width="1842" height="896" alt="image" src="https://github.com/user-attachments/assets/5cf426da-6995-4d70-8fda-0a37768a5bb9" />


### Placeholder 2: Workloads & Ingress Routing Status
> **Description:** Terminal output showing all Deployments, Pods (`1/1 Running`), Services, and Ingress resources running in the `streaming` namespace.
>
> **Command:** `kubectl get pods,svc,ingress -n streaming -o wide`
> <img width="1891" height="560" alt="image" src="https://github.com/user-attachments/assets/57d6ea33-6e74-418d-a281-2e1693eadd66" />


### Placeholder 3: Shared AWS Application Load Balancer
> **Description:** AWS Management Console displaying the provisioned Application Load Balancer, showing target groups associated with path rules (`/`, `/api/auth`, `/api/streaming`, `/api/admin`, `/api/chat`).
> <img width="1117" height="797" alt="image" src="https://github.com/user-attachments/assets/d12c7b24-07ef-4edb-9ded-ca9246893c94" />
> <img width="1132" height="811" alt="image" src="https://github.com/user-attachments/assets/408b860d-608f-4235-bc97-8c094263fb8b" />

### Placeholder 4: User Authentication & JWT Storage
> **Description:** Web browser showing successful user registration/login along with Browser Developer Tools (`F12 -> Application -> Local Storage`) displaying the valid JWT bearer token.
> <img width="1897" height="926" alt="image" src="https://github.com/user-attachments/assets/760fd9c8-04a2-4dd0-a2f9-26587729e3f0" />

### Placeholder 5: Admin Video & Thumbnail Upload to AWS S3
> **Description:** The `/admin` portal interface showing successful file selection and asset upload to the designated AWS S3 bucket.
<img width="1647" height="542" alt="image" src="https://github.com/user-attachments/assets/83e0833d-499e-4d4a-b1d8-cb673b1eac00" />

### Placeholder 6: Dual-Tab Real-time Live Chat
> **Description:** Side-by-side browser tabs displaying real-time WebSocket communication via `chatService`. Messages typed in Tab 1 immediately broadcast to Tab 2.

### Placeholder 7: Argo CD GitOps Dashboard
> **Description:** The Argo CD graphical user interface displaying the `streamingapp` application in `Synced` and `Healthy` state with a visual DAG graph of all Kubernetes objects.
> <img width="1342" height="587" alt="image" src="https://github.com/user-attachments/assets/001a21cb-76bb-4a6f-ad35-11303df11b7c" />

### Placeholder 8: Rolling Update & Self-Healing Verification
> **Description:** Terminal output showing pod deletion recovery (`kubectl delete pod ...`) and clean rollout status completion (`kubectl rollout status deployment/auth -n streaming`).
> <img width="1852" height="537" alt="image" src="https://github.com/user-attachments/assets/7b40ceb7-ca53-459a-9c59-8b5488bf9f45" />
  <img width="1880" height="497" alt="image" src="https://github.com/user-attachments/assets/13b1ff18-2272-42b3-aa07-839698c1b384" />
---

## Production Readiness & Recommendations Note

For a production-grade Kubernetes deployment, the following enhancements should be implemented:

1. **Custom Domain & TLS Termination:** Integrate **ExternalDNS** with AWS Route 53 and **cert-manager** with Let's Encrypt to expose `https://streamingapp.com` with automated TLS certificates instead of raw ALB DNS endpoints.
2. **Dynamic Ingress & Relative API Paths:** Refactor React frontend source code to use relative paths (`/api`) or runtime environment injection (`window._env_`), eliminating the need to rebuild container images when load balancer endpoints change.
3. **Secret Management:** Replace raw Kubernetes Secrets with **AWS Secrets Manager** integrated via the **External Secrets Operator (ESO)** or **Sealed Secrets**.
