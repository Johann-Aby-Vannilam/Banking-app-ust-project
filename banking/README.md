# Banking Application – Kubernetes Deployment Guide

> **Stack implemented:** Namespace · Kyverno · StatefulSets (MySQL 3-replica + RabbitMQ + Redis) · Microservices · kGateway · HAProxy · NFS Dynamic Provisioning

---

## Project Structure

```
banking/
├── admission-controller/
│   ├── bankingImagePolicy.yaml       # Kyverno: allowed image registries
│   └── resourcePolicy.yaml           # Kyverno: default resource limits
├── deployments/                      # All microservice Deployments + Services
│   ├── account-service.yml
│   ├── config-service.yml
│   ├── fraud-detection.yml
│   ├── frontend.yml
│   ├── loan-service.yml
│   ├── notification-service.yml
│   ├── payment-service.yml
│   ├── reporting-service.yml
│   ├── service-discovery.yml
│   ├── transaction-service.yml
│   └── user-service.yml
├── kgateway/                         # kGateway routes
│   ├── gateway.yaml
│   ├── account_route.yaml
│   ├── front_route.yaml
│   ├── loan_route.yaml
│   ├── report_route.yaml
│   ├── tx_route.yaml
│   └── user_route.yaml
└── volumes/
    ├── mysql.yml                     # MySQL StatefulSet (3 replicas + GTID replication)
    ├── rabbitmq.yml                  # RabbitMQ StatefulSet
    ├── redis.yml                     # Redis Deployment
    └── nfs/
        ├── nfs-storageclass.yaml     # StorageClass: nfs-client
        └── nfs-provisioner.yaml      # NFS Subdir External Provisioner (RBAC + Deployment)
```

---

## Architecture

```
Internet
   │
   ▼
HAProxy (EC2, port 80/443)
   │
   ▼
kGateway (banking-gateway) ──► HTTPRoutes
   │
   ├──► Frontend (Next.js)
   ├──► User Service     (DB: mysql-0 primary)
   ├──► Account Service  (DB: mysql-0 primary)
   ├──► Transaction Svc  (DB: mysql-0 primary)
   ├──► Loan Service     (DB: mysql-0 primary)
   └──► Reporting Svc    (DB: mysql-0 primary)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
               mysql-0             mysql-read (ClusterIP)
              (Primary)            routes to all 3 pods
                 │
        ┌────────┴────────┐
        ▼                 ▼
     mysql-1           mysql-2
    (Replica)         (Replica)
        │                 │
        └────────┬────────┘
                 ▼
         NFS Server (EC2)
         /srv/nfs/k8s
         ├── mysql-storage-mysql-0/
         ├── mysql-storage-mysql-1/
         ├── mysql-storage-mysql-2/
         └── rabbitmq-storage-rabbitmq-0/
```

---

## MySQL Replication Explained

| Pod | Hostname | Role | server-id |
|-----|----------|------|-----------|
| mysql-0 | mysql-0.mysql.banking-app.svc.cluster.local | **Primary** (read+write) | 100 |
| mysql-1 | mysql-1.mysql.banking-app.svc.cluster.local | **Replica** (read-only) | 101 |
| mysql-2 | mysql-2.mysql.banking-app.svc.cluster.local | **Replica** (read-only) | 102 |

**How it works:**
1. `init-mysql` init container runs on every pod, writes `server-id` and copies `primary.cnf` or `replica.cnf` based on pod ordinal
2. `init.sql` in `/docker-entrypoint-initdb.d/` creates the DB schema **and** the `replicator` user (runs on first start when data dir is empty)
3. The `postStart` lifecycle hook on pods 1 & 2 waits for mysql-0 to be ready, then runs `CHANGE REPLICATION SOURCE TO` and `START REPLICA` with `SOURCE_AUTO_POSITION=1` (GTID mode)
4. MySQL GTID replication streams all writes from primary to replicas automatically

---

## Prerequisites

- Kubernetes cluster (master + worker nodes, EC2 or equivalent)
- `kubectl` configured
- Kyverno installed
- kGateway (kgateway) installed
- HAProxy configured (already done)
- A **separate EC2 instance** for the NFS server

---

## Step-by-Step Implementation

### STEP 1 – Namespace

```bash
kubectl create namespace banking-app
```

---

### STEP 2 – Kyverno Admission Controllers

```bash
kubectl apply -f admission-controller/bankingImagePolicy.yaml
kubectl apply -f admission-controller/resourcePolicy.yaml
```

---

### STEP 3 – NFS Server Setup (on the NFS EC2 Instance)

SSH into your **NFS EC2 instance** and run:

```bash
# Install NFS server
sudo apt update && sudo apt install -y nfs-kernel-server

# Create the export directory
sudo mkdir -p /srv/nfs/k8s
sudo chmod 777 /srv/nfs/k8s
sudo chown nobody:nogroup /srv/nfs/k8s

# Configure NFS exports
# Replace 10.0.0.0/16 with your VPC CIDR (the subnet your K8s nodes are in)
echo "/srv/nfs/k8s  10.0.0.0/16(rw,sync,no_subtree_check,no_root_squash)" \
  | sudo tee -a /etc/exports

# Apply exports and start NFS
sudo exportfs -rav
sudo systemctl enable --now nfs-kernel-server

# Verify
sudo exportfs -v
showmount -e localhost
```

> **Note your NFS EC2 private IP** – you'll need it in Steps 5 and 6.

---

### STEP 4 – NFS Client Setup (on ALL Kubernetes Nodes: Master + Workers)

SSH into **each** K8s node and run:

```bash
# Install NFS client utilities
sudo apt update && sudo apt install -y nfs-common

# Test NFS mount from master node (replace <NFS_SERVER_IP>)
showmount -e <NFS_SERVER_IP>
# You should see: /srv/nfs/k8s  10.0.0.0/16
```

---

### STEP 5 – Edit nfs-provisioner.yaml with Your NFS Server IP

Open `volumes/nfs/nfs-provisioner.yaml` and replace both occurrences of `<NFS_SERVER_IP>` with your NFS EC2 **private** IP:

```yaml
# In env section:
- name: NFS_SERVER
  value: "10.0.1.50"        # ← your actual NFS EC2 private IP

# In volumes section:
volumes:
- name: nfs-client-root
  nfs:
    server: "10.0.1.50"     # ← same IP here
    path: "/srv/nfs/k8s"
```

---

### STEP 6 – Deploy NFS StorageClass and Provisioner

```bash
# Deploy StorageClass first
kubectl apply -f volumes/nfs/nfs-storageclass.yaml

# Deploy the provisioner (RBAC + Deployment in kube-system)
kubectl apply -f volumes/nfs/nfs-provisioner.yaml

# Wait for provisioner to be Running
kubectl get pods -n kube-system -l app=nfs-subdir-external-provisioner -w

# Verify StorageClass
kubectl get sc nfs-client
```

---

### STEP 7 – Deploy StatefulSets (MySQL, RabbitMQ, Redis)

```bash
# MySQL – 3 replicas with GTID replication
kubectl apply -f volumes/mysql.yml

# Watch pods come up (mysql-0 first, then 1 and 2)
kubectl get pods -n banking-app -l app=mysql -w

# RabbitMQ
kubectl apply -f volumes/rabbitmq.yml

# Redis
kubectl apply -f volumes/redis.yml
```

> **Wait for all 3 MySQL pods to be Running and Ready before proceeding.**

---

### STEP 8 – Deploy Microservices

```bash
kubectl apply -f deployments/user-service.yml
kubectl apply -f deployments/account-service.yml
kubectl apply -f deployments/transaction-service.yml
kubectl apply -f deployments/loan-service.yml
kubectl apply -f deployments/reporting-service.yml
kubectl apply -f deployments/payment-service.yml
kubectl apply -f deployments/notification-service.yml
kubectl apply -f deployments/fraud-detection.yml
kubectl apply -f deployments/config-service.yml
kubectl apply -f deployments/service-discovery.yml
kubectl apply -f deployments/frontend.yml
```

---

### STEP 9 – Deploy kGateway

```bash
kubectl apply -f kgateway/gateway.yaml
kubectl apply -f kgateway/user_route.yaml
kubectl apply -f kgateway/account_route.yaml
kubectl apply -f kgateway/tx_route.yaml
kubectl apply -f kgateway/loan_route.yaml
kubectl apply -f kgateway/report_route.yaml
kubectl apply -f kgateway/front_route.yaml
```

---

### STEP 10 – Full Verification

#### MySQL Replication Check

```bash
# All 3 MySQL pods should be Running
kubectl get pods -n banking-app -l app=mysql

# Check PRIMARY status on mysql-0
kubectl exec -n banking-app mysql-0 -- \
  mysql -u root -prootpassword -e "SHOW MASTER STATUS\G"

# Check REPLICA status on mysql-1 (look for Replica_IO_Running: Yes)
kubectl exec -n banking-app mysql-1 -- \
  mysql -u root -prootpassword -e "SHOW REPLICA STATUS\G"

# Check REPLICA status on mysql-2
kubectl exec -n banking-app mysql-2 -- \
  mysql -u root -prootpassword -e "SHOW REPLICA STATUS\G"

# Test replication: insert on primary, verify on replica
kubectl exec -n banking-app mysql-0 -- \
  mysql -u root -prootpassword banking_db \
  -e "INSERT INTO users (username, password, email) VALUES ('testuser', 'pass', 'test@test.com');"

kubectl exec -n banking-app mysql-1 -- \
  mysql -u root -prootpassword banking_db \
  -e "SELECT * FROM users WHERE username='testuser';"
# Should show the row - confirming replication works!
```

#### NFS / PVC Check

```bash
# Check PVCs are Bound (dynamic provisioning worked)
kubectl get pvc -n banking-app

# Check PVs were auto-created
kubectl get pv

# Check NFS directories were created on the server
# (Run on NFS EC2)
ls -la /srv/nfs/k8s/
```

#### Overall Cluster Health

```bash
kubectl get pods -n banking-app
kubectl get svc -n banking-app
kubectl get pvc -n banking-app
kubectl get pv
```

---

## Git – Push to New Repository

### Initialize and Push

```bash
# Navigate to the project root (one level above banking/)
cd "d:/UST Training/k8s/k8s-banking-application"

# Initialize git
git init
git add .
git commit -m "feat: initial commit - K8s banking app with MySQL replication and NFS dynamic provisioning"

# Create a new repo on GitHub (via GitHub CLI or manually on github.com)
# Then add remote and push:
git remote add origin https://github.com/<YOUR_USERNAME>/k8s-banking-app.git
git branch -M main
git push -u origin main
```

### GitHub CLI (faster)

```bash
# Install gh CLI if not already: https://cli.github.com/
gh auth login
gh repo create k8s-banking-app --public --source=. --remote=origin --push
```

### Recommended .gitignore

```bash
cat > .gitignore << 'EOF'
*.bak
*.tmp
.DS_Store
Thumbs.db
EOF
git add .gitignore
git commit -m "chore: add .gitignore"
git push
```

---

## Troubleshooting

### MySQL pod stuck in Init state

```bash
# Check init container logs
kubectl logs -n banking-app mysql-1 -c init-mysql
```

### Replication not starting (Replica_IO_Running: No)

```bash
# Check postStart logs (they go to container log)
kubectl logs -n banking-app mysql-1

# Manually trigger replication setup
kubectl exec -n banking-app mysql-1 -- \
  mysql -u root -prootpassword -e "
    STOP REPLICA;
    RESET REPLICA ALL;
    CHANGE REPLICATION SOURCE TO
      SOURCE_HOST='mysql-0.mysql.banking-app.svc.cluster.local',
      SOURCE_USER='replicator',
      SOURCE_PASSWORD='replicationpassword',
      SOURCE_PORT=3306,
      SOURCE_AUTO_POSITION=1;
    START REPLICA;"
```

### NFS PVC stuck in Pending

```bash
# Check provisioner logs
kubectl logs -n kube-system -l app=nfs-subdir-external-provisioner

# Test NFS connectivity from a worker node
showmount -e <NFS_SERVER_IP>

# Ensure nfs-common is installed on all nodes
sudo apt install -y nfs-common
```

### PVC Pending - StorageClass not found

```bash
kubectl get sc
# If nfs-client is missing, re-apply:
kubectl apply -f volumes/nfs/nfs-storageclass.yaml
```
