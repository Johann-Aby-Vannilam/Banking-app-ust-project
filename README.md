# 📦 NFS Server Setup for Kubernetes (Dynamic Provisioning)

Below is a **clean, end-to-end runbook** to set up NFS for your Kubernetes cluster with **dynamic provisioning** (what you’re already using with `nfs-client`).
 
---
 
# 🧭 Architecture (what you’re building)
 
```text
Worker-1 (NFS Server)
   └── /mnt/nfs-share  (exported)
 
Worker-2, Worker-3
   └── NFS client packages
 
Master (Control Plane)
   └── NFS provisioner (creates PVs dynamically)
 
Kubernetes
   PVC → StorageClass(nfs-client) → Provisioner → NFS folders
```
 
---
 
# 🧱 1) Worker Node (NFS Server)
 
> Do this on **one worker** (your NFS host, e.g., `172.31.64.100`)
 
## Install NFS server
 
```bash
sudo apt update
sudo apt install nfs-kernel-server -y
```
 
## Create export directory
 
```bash
sudo mkdir -p /mnt/nfs-share
sudo chown nobody:nogroup /mnt/nfs-share
sudo chmod 777 /mnt/nfs-share
```
 
## Configure exports
 
```bash
sudo nano /etc/exports
```
 
Add (use your VPC CIDR, not `*`):
 
```bash
/mnt/nfs-share 172.31.0.0/16(rw,sync,no_subtree_check,no_root_squash)
```
 
## Apply and start
 
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```
 
## Verify
 
```bash
showmount -e
```
 
Expected:
 
```text
/mnt/nfs-share 172.31.0.0/16
```
 
---
 
# 🌐 2) AWS / Network (critical)
 
Allow NFS traffic **into the NFS server node**:
 
| Type       | Protocol | Port | Source        |
| ---------- | -------- | ---- | ------------- |
| Custom TCP | TCP      | 2049 | 172.31.0.0/16 |
 
---
 
# 🧰 3) All Worker Nodes (including NFS server)
 
Install NFS client:
 
```bash
sudo apt install nfs-common -y
```
 
## Quick connectivity test (from another worker)
 
```bash
sudo mkdir -p /mnt/test-nfs
sudo mount 172.31.64.100:/mnt/nfs-share /mnt/test-nfs
touch /mnt/test-nfs/ok.txt
```
 
If this works → network + exports are correct.
 
---
 
# 🎛️ 4) Master Node — Install Dynamic Provisioner
 
> Run **only on master** (control plane)
 
## (If needed) Install Helm
 
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
 
## Add repo
 
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```
 
## Install provisioner
 
```bash
kubectl create namespace nfs-provisioner
 
helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --set nfs.server=172.31.64.100 \
  --set nfs.path=/mnt/nfs-share \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false
```
 
---
 
# 🔍 5) Verify Provisioner
 
```bash
kubectl get pods -n nfs-provisioner
```
 
Expected:
 
```text
nfs-client-nfs-subdir-external-provisioner-xxxxx   Running
```
 
```bash
kubectl get storageclass
```
 
Expected:
 
```text
nfs-client
```
 
---
# k8s-banking-application
