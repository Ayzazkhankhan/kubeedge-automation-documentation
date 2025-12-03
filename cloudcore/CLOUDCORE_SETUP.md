# 🚀 CloudCore Setup – Complete Single-File README  
&gt; Copy-paste **entire** file into GitHub – zero folder structure needed.

---

## 📋 Prerequisites

### Hardware Requirements
- AWS EC2 instance: t3.medium or larger  
- Ubuntu 22.04 LTS  
- Minimum 4GB RAM, 20GB SSD  
- Static public IP (Elastic IP recommended)

### Software Requirements
```bash
# Verify versions
ubuntu --version    # 22.04
docker --version    # 29.1.1+
k3s --version       # v1.33.6+
keadm version       # v1.18.0
```

---

## 🚀 Step-by-Step Installation

### Step 1: AWS EC2 Instance Setup
- **AMI**: Ubuntu 22.04 LTS  
- **
- **
- **Security Group**: open these ports  
  - Port 22: SSH (0.0.0.0/0)  
  - Port 6443: Kubernetes API (0.0.0.0/0)  
  - Port 30374: CloudHub NodePort (0.0.0.0/0)  
  - Port 31305: CloudStream NodePort (0.0.0.0/0)  
  - Port 32203: WebSocket NodePort (0.0.0.0/0)  
  - Port 30500: Master App (0.0.0.0/0)

Connect:
```bash
ssh -i your-key.pem ubuntu@3.70.53.21
```

---

### Step 2: System Preparation
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim net-tools
sudo hostnamectl set-hostname cloud-core
echo "3.70.53.21 cloud-core" | sudo tee -a /etc/hosts
```

---

### Step 3: Install Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker
docker --version    # Expected: Docker version 29.1.1
```

---

### Step 4: Install Kubernetes (K3s)
```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' &gt;&gt; ~/.bashrc
kubectl get nodes   # Expected: Ready master
```

---

### Step 5: Install KubeEdge CloudCore
```bash
cd ~
wget https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
```

Initialize:
```bash
export AWS_PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
echo "Public IP: $AWS_PUBLIC_IP"

sudo keadm init \
  --advertise-address=$AWS_PUBLIC_IP \
  --kubeedge-version=1.18.0 \
  --kube-config=/etc/rancher/k3s/k3s.yaml
```

Verify:
```bash
kubectl get pods -n kubeedge
# Expected: cloudcore-xxx Running
```

---

### Step 6: Expose CloudCore Services
```bash
kubectl patch svc cloudcore -n kubeedge -p '{"spec":{"type":"NodePort"}}'
kubectl get svc -n kubeedge cloudcore
# Expected NodePorts: 30374, 31305, 32203
```

---

### Step 7: Generate Edge Join Token
```bash
sudo keadm gettoken --kube-config=/etc/rancher/k3s/k3s.yaml
# Save this token securely!
# Example: dab11b0b6882dd33cd715a3bc9fe0bcbefd29daf56904d42d2ccbcc6da1aa361.eyJ...
```

---

### Step 8: Deploy Master Application
```bash
mkdir -p ~/masterapp && cd ~/masterapp
cat &gt; master-deployment.yaml &lt;&lt;'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: master-app
spec:
  replicas: 1
  selector:
    matchLabels: { app: master-app }
  template:
    metadata:
      labels: { app: master-app }
    spec:
      containers:
      - name: master-app
        image: ahmedxeno/master-app:latest
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: master-app
spec:
  type: NodePort
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
  selector:
    app: master-app
EOF
kubectl apply -f master-deployment.yaml
```

---

### Step 9: Configure Firewall
Add these **Inbound rules** to AWS Security Group:
| Port | Protocol | Source     |
|------|----------|------------|
| 30374 | TCP    | 0.0.0.0/0  |
| 31305 | TCP    | 0.0.0.0/0  |
| 32203 | TCP    | 0.0.0.0/0  |
| 30500 | TCP    | 0.0.0.0/0  |

---

### Step 10: Verification
```bash
kubectl get nodes
kubectl get pods -A
curl http://localhost:30500/health   # master app health
```

---

## ⚙️ Configuration Files
| File | Path |
|------|------|
| CloudCore config | `/etc/kubeedge/config/cloudcore.yaml` |
| K3s kubeconfig | `/etc/rancher/k3s/k3s.yaml` |
| kubectl config | `~/.kube/config` |

---

## 🔧 Troubleshooting

**Port conflicts / clean reinstall**
```bash
sudo /usr/local/bin/k3s-uninstall.sh
sudo rm -rf /var/lib/rancher/k3s /etc/rancher/k3s ~/.kube
sudo pkill -9 k3s
curl -sfL https://get.k3s.io | sh -
```

**CloudCore logs**
```bash
kubectl logs -n kubeedge deployment/cloudcore --tail=50
```

---

## 📊 Monitoring Setup (optional)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --set server.service.type=NodePort --set server.service.nodePort=30900
# Grafana: http://&lt;IP&gt;:30900
```

---

## 🔒 Security (optional SSL)
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
# create Issuer for Let's Encrypt – sample in repo
```

---

## 🧹 Maintenance
**Backup configs**
```bash
mkdir -p ~/backup
cp /etc/kubeedge/config/cloudcore.yaml ~/backup/
cp /etc/rancher/k3s/k3s.yaml ~/backup/
cp ~/.kube/config ~/backup/
tar -czf cloudcore-backup-$(date +%F).tar.gz -C ~/backup .
```

---

## ✅ Verification Checklist
- [ ] AWS instance running  
- [ ] All ports open in security group  
- [ ] Docker installed & running  
- [ ] K3s cluster Ready  
- [ ] CloudCore pod Running  
- [ ] NodePorts assigned  
- [ ] Edge token generated  
- [ ] Master app deployed  
- [ ] Dashboard reachable at `http://&lt;IP&gt;:30500`

---

## 🎉 Next Step
Proceed to **EdgeCore join** – copy the token and head to the EdgeCore guide in the same repo.

---

## 📋 TL;DR for README
```markdown
## Cloud Setup – 5 Commands
1. Launch Ubuntu 22.04 EC2
2. Paste blocks 2 → 10 above
3. Save token printed in step 7
4. Done – CloudCore live at `http://&lt;IP&gt;:30500`
```
