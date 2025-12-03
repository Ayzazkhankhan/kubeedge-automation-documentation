# 🚀 EdgeCore Setup – Single-File Edition  
*(copy-paste friendly)*

---

## 1. Hardware & OS (quick pick)
- **Pi 3B+ or x86_64**, 4 GB RAM, 32 GB SD/SSD  
- **Ubuntu 22.04** (fresh image)  
- **Camera**: USB or CSI (optional)

---

## 2. Connect & Prepare (copy whole block)
```bash
ssh ubuntu@edge-device-ip
sudo hostnamectl set-hostname edge-node-001
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim net-tools
```

---

## 3. containerd + CNI (one-liner stack)
```bash
# install
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# CNI plugins
sudo mkdir -p /opt/cni/bin
wget -q https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar -xzf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin

# network
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# start
sudo systemctl restart containerd && sudo systemctl enable containerd
```

---

## 4. Install keadm (30 s)
```bash
cd ~
wget https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
```

---

## 5. Join Edge to Cloud (one line – **replace token**)
```bash
sudo keadm join \
  --cloudcore-ipport=3.70.53.21:30374 \
  --edgenode-name=edge-node-001 \
  --token=dab11b0b6882dd33cd715a3bc9fe0bcbefd29daf56904d42d2ccbcc6da1aa361.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NjQ0ODA4NzZ9.jgFzkOUfFqF73rRptL7SHGFHcUxV0vy5TF4KFSzNjzI \
  --kubeedge-version=1.18.0 \
  --remote-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroupdriver=systemd
```

---

## 6. Verify (copy-paste)
```bash
sudo systemctl status edgecore
kubectl get nodes   # run on cloud-core
```

---

## 7. Camera Test (optional)
```bash
ls /dev/video*
python3 -c "import cv2; cap=cv2.VideoCapture(0); print('OK' if cap.read()[0] else 'FAIL'); cap.release()"
```

---

## 8. Edge App Directory (create once)
```bash
sudo mkdir -p /opt/edge-app
sudo chown $USER:$USER /opt/edge-app
cd /opt/edge-app
```

---

## 9. Health Check Script (save as `health-edge.sh`)
```bash
#!/bin/bash
echo "=== Edge Health ==="
systemctl is-active edgecore && echo "✅ EdgeCore running" || echo "❌ EdgeCore stopped"
curl -m2 http://localhost:5000/health 2&gt;/dev/null && echo "✅ App OK" || echo "⚠️  App not ready"
```

---

## 10. Quick Clean (if needed)
```bash
sudo systemctl stop edgecore
sudo rm -rf /etc/kubeedge /var/lib/kubeedge
sudo systemctl disable edgecore
```

---

## 11. One-Command Full Setup (entire edge)
```bash
# run on fresh Ubuntu 22.04 – **replace TOKEN**
wget -qO- https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz | tar -xz &&
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/ &&
sudo apt update && sudo apt install -y containerd &&
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml &&
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml &&
sudo systemctl restart containerd &&
sudo keadm join --cloudcore-ipport=3.70.53.21:30374 --edgenode-name=$(hostname) --token=PUT_YOUR_TOKEN_HERE --kubeedge-version=1.18.0 --remote-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroupdriver=systemd
```

---

## 12. Next Step
Your edge node appears in `kubectl get nodes` – proceed to deploy **xis.rt** or any container.

---

## 📋 TL;DR for README
```markdown
## Edge Setup – 3 Copy-Paste Blocks
1. Flash Ubuntu 22.04 on Pi / PC
2. Paste blocks 2 → 5 above
3. Run block 11 (entire one-liner) with token from cloud
4. Done – node shows up in cloud dashboard
```

✅ **Done** – entire EdgeCore guide in one file, ready for GitHub.
