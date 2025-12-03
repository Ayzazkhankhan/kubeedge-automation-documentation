# 🚀 EdgeCore Setup – **Complete** Single-File Edition  
*(includes scripts, configs, security, tuning, troubleshooting – zero omissions)*

---

## 1. Hardware & OS
- Pi 3B+ or x86_64, 4 GB RAM, 32 GB storage  
- Ubuntu 22.04 fresh image  
- Camera optional (USB / CSI)

---

## 2. Connect & Prepare
```bash
ssh ubuntu@edge-device-ip
sudo hostnamectl set-hostname edge-node-001
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim net-tools
```

---

## 3. containerd + CNI (full config)
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

# CNI network config
sudo mkdir -p /etc/cni/net.d
cat &lt;&lt;EOF | sudo tee /etc/cni/net.d/10-containerd-net.conflist
{
  "cniVersion": "0.4.0",
  "name": "containerd-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "promiscMode": true,
      "ipam": {
        "type": "host-local",
        "ranges": [[{"subnet": "10.88.0.0/16"}]],
        "routes": [{"dst": "0.0.0.0/0"}]
      }
    },
    {"type": "portmap", "capabilities": {"portMappings": true}}
  ]
}
EOF

# kernel
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# start
sudo systemctl restart containerd && sudo systemctl enable containerd
```

---

## 4. Install keadm
```bash
cd ~
wget https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
```

---

## 5. Join Edge to Cloud (replace token)
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

## 6. Verify
```bash
sudo systemctl status edgecore
kubectl get nodes   # on cloud-core
```

---

## 7. EdgeCore Config File (`/etc/kubeedge/config/edgecore.yaml`)
```yaml
apiVersion: edgecore.config.kubeedge.io/v1alpha1
kind: EdgeCore
modules:
  edged:
    enable: true
    cgroupDriver: systemd
    networkPluginName: cni
    containerRuntime: containerd
    remoteRuntimeEndpoint: unix:///run/containerd/containerd.sock
    remoteImageEndpoint: unix:///run/containerd/containerd.sock
  edgehub:
    enable: true
    heartbeat: 15
    websocket:
      enable: true
      handshakeTimeout: 30
      readDeadline: 15
      server: 3.70.53.21:30374
      writeDeadline: 15
  eventbus:
    enable: true
    mqttMode: 2
    mqttServer: tcp://127.0.0.1:1883
  metaManager:
    enable: true
```

---

## 8. Camera Test (optional)
```bash
ls /dev/video*
python3 -c "import cv2; cap=cv2.VideoCapture(0); print('OK' if cap.read()[0] else 'FAIL'); cap.release()"
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

## 10. Full Automation Script (save as `setup-edge.sh`)
```bash
#!/bin/bash
set -e
TOKEN=${1:? "Usage: $0 &lt;EDGE_TOKEN&gt;"}

echo "🚀 Installing dependencies..."
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

echo "📦 Installing keadm..."
wget -qO- https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz | tar -xz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm

echo "🔗 Joining cluster..."
sudo keadm join --cloudcore-ipport=3.70.53.21:30374 --edgenode-name=$(hostname) --token="$TOKEN" --kubeedge-version=1.18.0 --remote-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroupdriver=systemd

echo "✅ Done – check with: sudo systemctl status edgecore"
```

Run: `chmod +x setup-edge.sh && ./setup-edge.sh dab11b0b...`

---

## 11. Performance Tuning (Pi)
```bash
# reduce swap
sudo dphys-swapfile swapoff && sudo dphys-swapfile uninstall && sudo systemctl disable dphys-swapfile
# GPU mem for camera
echo "gpu_mem=128" | sudo tee -a /boot/config.txt
# performance governor
sudo apt install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
```

---

## 12. Security Hardening (optional)
```bash
#!/bin/bash
# disable root ssh
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
sudo systemctl restart sshd
# basic firewall
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw allow 22/tcp && sudo ufw allow 5001/tcp
```

---

## 13. Troubleshooting One-Liners
```bash
sudo journalctl -u edgecore -f                            # live logs
sudo systemctl restart edgecore                           # restart service
sudo rm -rf /etc/kubeedge /var/lib/kubeedge               # full reset
netcat -zv 3.70.53.21 30374                               # network test
```

---

## 14. Next Step
Your edge node appears in `kubectl get nodes` – deploy **xis.rt** or any container.

---

## 📋 TL;DR for README
```markdown
## Edge Setup – 2 Blocks
1. Flash Ubuntu 22.04
2. Run full script (block 10) with token from cloud
3. Done – node shows in dashboard
```

✅ **Complete** – nothing omitted; ready for GitHub.
