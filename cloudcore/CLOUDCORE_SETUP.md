
## 1. AWS EC2 (30 s click-work)
- AMI: Ubuntu 22.04  
- Type: t3.medium  
- Storage: 30 GB GP2  
- **Security-Group Inbound**  
  - 22, 6443, 30374, 31305, 32203, 30500 ← all TCP / 0.0.0.0

---

## 2. Connect & Prepare (copy whole block)
```bash
ssh -i YOUR_KEY.pem ubuntu@3.70.53.21
sudo hostnamectl set-hostname cloud-core
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim net-tools
```

---

## 3. Docker 29.x (one-liner)
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
sudo usermod -aG docker $USER && newgrp docker
docker --version   # 29.x
```

---

## 4. K3s (single-command k8s)
```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc && source ~/.bashrc
kubectl get nodes   # Ready master
```

---

## 5. KubeEdge keadm (30 s)
```bash
cd ~
wget https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
```

---

## 6. Start CloudCore (one line)
```bash
export PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
sudo keadm init --advertise-address=$PUBLIC_IP --kubeedge-version=1.18.0 --kube-config=/etc/rancher/k3s/k3s.yaml
```

---

## 7. Expose NodePorts (so edges can reach)
```bash
kubectl patch svc cloudcore -n kubeedge -p '{"spec":{"type":"NodePort"}}'
kubectl get svc -n kubeedge cloudcore   # note 30374 / 31305 / 32203
```

---

## 8. Edge Join Token (save this!)
```bash
sudo keadm gettoken --kube-config=/etc/rancher/k3s/k3s.yaml
# long string starting with dab11b0b... copy it
```

---

## 9. Sample Master Dashboard (optional)
```bash
mkdir -p ~/masterapp && cd ~/masterapp
cat > master-deployment.yaml <<'EOF'
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

## 10. Health Check (copy-paste)
```bash
kubectl get nodes
kubectl get pods -A
curl http://localhost:30500/health
```

---

## 11. Quick Clean-Slate (if broken)
```bash
sudo /usr/local/bin/k3s-uninstall.sh
sudo rm -rf /etc/rancher /var/lib/rancher ~/.kube
sudo pkill -9 k3s
```

---

## 12. Backup Configs (one-liner tar)
```bash
mkdir -p ~/backup && cp /etc/kubeedge/config/cloudcore.yaml ~/backup/
cp /etc/rancher/k3s/k3s.yaml ~/backup/
cp ~/.kube/config ~/backup/
tar -czf cloudcore-backup-$(date +%F).tar.gz -C ~/backup .
```

---

## 13. Monitoring (optional, 2 commands)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --set server.service.type=NodePort --set server.service.nodePort=30900
```
Visit: `http://3.70.53.21:30900`
