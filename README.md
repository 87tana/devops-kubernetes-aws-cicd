# DevOps Kubernetes AWS CI/CD

A production-like deployment of a microservices voting application on **Amazon EKS** with a fully automated **CI/CD pipeline using GitHub Actions**.

## Architecture

```
User → NGINX Ingress → /vote   → Vote App (Python/Flask) → Redis
                     → /result → Result App (Node.js)    ← PostgreSQL
                                                               ↑
                                                        Worker (.NET)
```

## Tech Stack

| Category | Technology |
|----------|-----------|
| Cloud Provider | AWS (EKS) |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |
| Container Registry | Docker Hub |
| Ingress | NGINX via Helm |
| Secret Management | Kubernetes Secrets |
| Infrastructure CLI | eksctl, kubectl, AWS CLI v2, Helm |

## Microservices

| Service | Language | Role |
|---------|----------|------|
| vote | Python (Flask) | Voting frontend |
| result | Node.js | Results frontend |
| worker | .NET | Processes votes from Redis to PostgreSQL |
| redis | Redis | Message queue |
| postgres | PostgreSQL | Persistent storage |

## Project Structure

```
├── vote/                        # Python voting app + Dockerfile
├── result/                      # Node.js result app + Dockerfile
├── worker/                      # .NET worker + Dockerfile
├── K8s/
│   ├── redis-deployment.yaml
│   ├── postgres-deployment.yaml
│   ├── vote-deployment.yaml
│   ├── result-deployment.yaml
│   ├── worker-deployment.yaml
│   └── ingress.yaml
└── .github/workflows/
    └── ci-cd-pipeline.yml
```

## CI/CD Pipeline

Triggers automatically on every push to `main`:

```
Push to main
  → Build Docker images (vote, result, worker)
  → Push to Docker Hub
  → Configure AWS credentials
  → Update kubeconfig
  → kubectl apply -f K8s/
```

## Quick Start

### Prerequisites
- AWS Account with IAM user (AdministratorAccess)
- Docker Hub account
- AWS CLI v2, eksctl, kubectl, Docker, Helm installed

### 1. Create EKS Cluster
```bash
eksctl create cluster \
  --name voting-app-cluster \
  --region eu-central-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2
```

### 2. Connect kubectl to cluster
```bash
aws eks update-kubeconfig --region eu-central-1 --name voting-app-cluster
```

### 3. Install NGINX Ingress Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

### 4. Build and Push Images
```bash
docker build -t <dockerhub-username>/vote:latest ./vote && docker push <dockerhub-username>/vote:latest
docker build -t <dockerhub-username>/result:latest ./result && docker push <dockerhub-username>/result:latest
docker build -t <dockerhub-username>/worker:latest ./worker && docker push <dockerhub-username>/worker:latest
```

### 5. Create Kubernetes Secret
```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=postgres
```

### 6. Deploy
```bash
kubectl apply -f K8s/
kubectl get pods
kubectl get ingress
```

Access the app:
- `http://<INGRESS-ADDRESS>/vote`
- `http://<INGRESS-ADDRESS>/result`

### 7. GitHub Actions Setup
Add these secrets to your GitHub repo (Settings → Secrets → Actions):

| Secret | Description |
|--------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub password |
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |

## Security
- Database credentials managed via **Kubernetes Secrets**
- AWS and Docker Hub credentials stored as **GitHub Secrets**
- No sensitive data in version control

## Cleanup
```bash
eksctl delete cluster --name voting-app-cluster --region eu-central-1
```
