
# 🗳️ Voting App on Amazon EKS

Deploying a multi-service voting application on a production-grade Amazon EKS (Kubernetes) cluster with automated CI/CD, HTTPS, and Kubernetes Secrets management.

---

## 🏗️ AWS Architecture

```
🌍 INTERNET
      |
      | HTTPS (443)
      ▼
🌐 Route 53 DNS
      | vote.nata.ironlabs.online
      | result.nata.ironlabs.online
      ▼
⚖️  AWS Load Balancer (ELB)
      |
      ▼
┌─────────────────────────────────────────┐
│           🐳 EKS Cluster                │
│         (spot-eks-lab-nata)             │
│                                         │
│  🔀 NGINX Ingress Controller            │
│      | /vote   → vote-service           │
│      | /result → result-service         │
│      ▼                                  │
│  ┌──────────────────────────────────┐   │
│  │      📦 voting-app namespace     │   │
│  │                                  │   │
│  │  🐍 Vote App (Python/Flask)      │   │
│  │          |                       │   │
│  │          | push vote             │   │
│  │          ▼                       │   │
│  │  🔴 Redis  :6379                 │   │
│  │          |                       │   │
│  │          | read queue            │   │
│  │          ▼                       │   │
│  │  ⚙️  Worker (.NET)               │   │
│  │          |                       │   │
│  │          | write result          │   │
│  │          ▼                       │   │
│  │  🐘 PostgreSQL :5432             │   │
│  │          |                       │   │
│  │          | read results          │   │
│  │          ▼                       │   │
│  │  🟢 Result App (Node.js)         │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 🧩 Microservices

| Service | Role | Technology | Port |
|---------|------|------------|------|
| 🐍 vote | Voting UI — cast your vote | Python / Flask | 80 |
| 🟢 result | Live results dashboard | Node.js | 80 |
| ⚙️ worker | Processes votes from Redis → Postgres | .NET | - |
| 🔴 redis | Message queue | Redis | 6379 |
| 🐘 postgres | Persistent vote storage | PostgreSQL 15 | 5432 |

---

## 📁 Project Structure

```
voting-app-eks/
│
├── 📂 .github/
│   └── 📂 workflows/
│       └── 📄 ci-cd-pipeline.yaml      # GitHub Actions — auto deploy on push
│
├── 📂 k8s/
│   ├── 📂 redis/
│   │   └── 📄 redis-deployment.yaml    # Redis Deployment + Service
│   ├── 📂 postgres/
│   │   └── 📄 postgres-deployment.yaml # Postgres Deployment + Service
│   ├── 📂 vote/
│   │   └── 📄 vote-deployment.yaml     # Vote App Deployment + Service
│   ├── 📂 result/
│   │   └── 📄 result-deployment.yaml   # Result App Deployment + Service
│   ├── 📂 worker/
│   │   └── 📄 worker-deployment.yaml   # Worker Deployment (no service needed)
│   └── 📂 ingress/
│       ├── 📄 ingress.yaml             # NGINX Ingress + TLS config
│       └── 📄 cluster-issuer.yaml      # Let's Encrypt ClusterIssuer
│
└── 📄 README.md
```

---

## 🔄 Vote Flow

```
👤 User visits https://vote.nata.ironlabs.online
        |
        | clicks Cats or Dogs
        ▼
🐍 Vote App (Python)
        |
        | pushes vote to queue
        ▼
🔴 Redis (message queue)
        |
        | worker reads queue
        ▼
⚙️  Worker (.NET)
        |
        | writes result to DB
        ▼
🐘 PostgreSQL
        |
        | result app reads DB
        ▼
🟢 Result App (Node.js)
        |
        ▼
👤 User sees live % at https://result.nata.ironlabs.online
```

---

## 🚀 Deployment Guide

### ✅ Prerequisites
- AWS CLI configured with credentials
- `kubectl` installed
- `eksctl` installed
- EKS cluster running

### 1️⃣ Connect to EKS cluster
```bash
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
kubectl get nodes   # verify nodes are Ready
```

### 2️⃣ Create namespace
```bash
kubectl create namespace voting-app
```

### 3️⃣ Create Kubernetes Secrets
> ⚠️ Never commit passwords to GitHub — create secrets directly on the cluster!
```bash
kubectl create secret generic app-secrets \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=<your-password> \
  --from-literal=DATABASE_URL=postgres://postgres:<your-password>@db:5432/postgres \
  --from-literal=REDIS_HOST=redis \
  -n voting-app
```

### 4️⃣ Deploy all services
```bash
kubectl apply -f k8s/redis/redis-deployment.yaml
kubectl apply -f k8s/postgres/postgres-deployment.yaml
kubectl apply -f k8s/vote/vote-deployment.yaml
kubectl apply -f k8s/result/result-deployment.yaml
kubectl apply -f k8s/worker/worker-deployment.yaml
kubectl apply -f k8s/ingress/cluster-issuer.yaml
kubectl apply -f k8s/ingress/ingress.yaml
```

### 5️⃣ Verify deployment
```bash
kubectl get pods -n voting-app        # all pods Running
kubectl get ingress -n voting-app     # ingress has ADDRESS
kubectl get certificate -n voting-app # certificate READY=True
```

---

## ⚙️ CI/CD Pipeline

```
📝 Push code to main branch
        |
        ▼
🤖 GitHub Actions triggered
        |
        ▼
🔑 Configure AWS credentials
        |
        ▼
🔗 Update kubeconfig → connect to EKS
        |
        ▼
📦 kubectl apply all manifests
        |
        ▼
✅ App updated on EKS automatically!
```

### 🔑 Required GitHub Secrets
| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Your AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Your AWS secret key |

---

## 🔐 Security

| Feature | Implementation |
|---------|---------------|
| 🔒 No hardcoded secrets | All passwords in Kubernetes Secrets |
| 🔒 HTTPS everywhere | cert-manager + Let's Encrypt SSL |
| 🔒 Git protection | .gitignore prevents secret file commits |
| 🔒 Runtime injection | Secrets injected via `secretKeyRef` |

---

## 🌐 Live URLs

| App | URL |
|-----|-----|
| 🗳️ Vote App | https://vote.nata.ironlabs.online |
| 📊 Result App | https://result.nata.ironlabs.online |

---

## 🛠️ Useful Commands

```bash
# 📋 Check all pods
kubectl get pods -n voting-app

# 📋 Check all services
kubectl get svc -n voting-app

# 📋 Check secrets (values hidden)
kubectl get secrets -n voting-app

# 📋 Check certificate status
kubectl get certificate -n voting-app

# 📜 View logs
kubectl logs -l app=worker -n voting-app
kubectl logs -l app=vote -n voting-app
kubectl logs -l app=result -n voting-app

# 🖥️ Local testing via port-forward
kubectl port-forward svc/vote-service 8080:80 -n voting-app
kubectl port-forward svc/result-service 8081:80 -n voting-app

# 🔄 Restart a deployment
kubectl rollout restart deployment/vote -n voting-app
```