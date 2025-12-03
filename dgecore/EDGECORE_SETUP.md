# EdgeCore Setup Guide

## 📋 Prerequisites for Edge Devices

### Supported Hardware
- **Minimum**: Raspberry Pi 3B+ or equivalent
- **Recommended**: x86_64 with 4GB RAM, 32GB storage
- **Camera**: USB webcam or CSI camera module

### Operating Systems
- ✅ Ubuntu 22.04 LTS (Recommended)
- ✅ Ubuntu 20.04 LTS
- ✅ Debian 11/12
- ⚠️ Raspberry Pi OS (32-bit may have issues)

### Network Requirements
- **Internet Connection**: Required for initial setup
- **Ports**: Outbound to CloudCore (30374, 31305, 32203)
- **Firewall**: Allow outgoing connections to cloud

## 🚀 Step-by-Step Edge Device Setup

### Step 1: Initial Device Configuration

#### Set Hostname
```bash
sudo hostnamectl set-hostname edge-node-001
echo "127.0.0.1 edge-node-001" | sudo tee -a /etc/hosts
Update System
bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim net-tools
Configure Timezone (Important!)
bash
sudo timedatectl set-timezone UTC
sudo timedatectl set-ntp true
Step 2: Install Container Runtime (containerd)
Install containerd
bash
# Install containerd
sudo apt install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver (REQUIRED)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
Verify containerd Installation
bash
sudo systemctl status containerd
containerd --version
# Expected: containerd v1.7.28
Step 3: Install CNI Plugins
Download and Install CNI
bash
# Create CNI directory
sudo mkdir -p /opt/cni/bin

# Download CNI plugins v1.3.0
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz

# Extract to /opt/cni/bin
sudo tar -xzf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
Create CNI Configuration
bash
# Create CNI config directory
sudo mkdir -p /etc/cni/net.d

# Create network configuration
cat <<EOF | sudo tee /etc/cni/net.d/10-containerd-net.conflist
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
        "ranges": [
          [{
            "subnet": "10.88.0.0/16"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
Step 4: Install KubeEdge EdgeCore (keadm)
Download keadm
bash
cd ~
wget https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
Verify keadm Installation
bash
keadm version
# Expected: version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0"}
Step 5: Join Edge Node to CloudCore
Prepare Join Command
bash
# Cloud details
CLOUD_IP="3.70.53.21"
CLOUD_PORT="30374"
NODE_NAME="edge-node-001"
DEVICE_ID="edge-device-001"

# Your token from CloudCore setup
EDGE_TOKEN="dab11b0b6882dd33cd715a3bc9fe0bcbefd29daf56904d42d2ccbcc6da1aa361.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NjQ0ODA4NzZ9.jgFzkOUfFqF73rRptL7SHGFHcUxV0vy5TF4KFSzNjzI"
Join Command
bash
sudo keadm join \
  --cloudcore-ipport=${CLOUD_IP}:${CLOUD_PORT} \
  --edgenode-name=${NODE_NAME} \
  --token=${EDGE_TOKEN} \
  --kubeedge-version=1.18.0 \
  --remote-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroupdriver=systemd
Join Script (Alternative)
Create a join script for automation:

bash
cat > join-edge.sh << 'EOF'
#!/bin/bash

# Configuration
CLOUD_IP="3.70.53.21"
CLOUD_PORT="30374"
NODE_NAME="edge-node-001"
TOKEN="YOUR_TOKEN_HERE"

echo "Joining edge node to KubeEdge cluster..."

# Clean any previous installation
sudo rm -rf /etc/kubeedge/

# Join command
sudo keadm join \
  --cloudcore-ipport=${CLOUD_IP}:${CLOUD_PORT} \
  --edgenode-name=${NODE_NAME} \
  --token=${TOKEN} \
  --kubeedge-version=1.18.0 \
  --remote-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroupdriver=systemd

if [ $? -eq 0 ]; then
    echo "✅ Edge node joined successfully!"
    echo "Check status with: sudo systemctl status edgecore"
else
    echo "❌ Failed to join edge node"
    exit 1
fi
EOF

chmod +x join-edge.sh
Step 6: Verify EdgeCore Installation
Check EdgeCore Service
bash
sudo systemctl status edgecore
# Expected: Active: running

# View logs
sudo journalctl -u edgecore -f
Check Kubernetes Node Registration
bash
# On CloudCore server, check node registration
kubectl get nodes
# Should see your edge node
Step 7: Configure Edge Device
Set Device ID
bash
echo "export DEVICE_ID=edge-device-001" | sudo tee -a /etc/environment
echo "export NODE_NAME=edge-node-001" | sudo tee -a /etc/environment
source /etc/environment
Create Edge App Directory
bash
sudo mkdir -p /opt/edge-app
sudo chown -R $USER:$USER /opt/edge-app
Step 8: Install Edge Application Dependencies
Install Python and Dependencies
bash
sudo apt install -y python3-pip python3-venv
sudo apt install -y libopencv-dev python3-opencv

# Create virtual environment
cd /opt/edge-app
python3 -m venv venv
source venv/bin/activate

# Install Python packages
pip install --upgrade pip
pip install flask opencv-python numpy paho-mqtt
Install Camera Support
bash
# For USB cameras
sudo apt install -y v4l-utils

# For Raspberry Pi CSI camera
sudo apt install -y python3-picamera2  # Raspberry Pi only

# Test camera
v4l2-ctl --list-devices
Step 9: Network Configuration
Configure Static IP (Optional)
bash
# Edit network configuration
sudo nano /etc/netplan/00-installer-config.yaml

# Add static IP configuration
network:
  ethernets:
    eth0:
      dhcp4: false
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2

# Apply configuration
sudo netplan apply
Configure Firewall (If enabled)
bash
# Allow required ports
sudo ufw allow 5001/tcp  # Edge app port
sudo ufw allow 1883/tcp  # MQTT port
sudo ufw allow out 30374/tcp  # CloudCore connection
Step 10: Testing and Validation
Test Cloud Connectivity
bash
# Test connection to CloudCore
nc -zv 3.70.53.21 30374
# Expected: Connection successful

# Test API connectivity
curl http://3.70.53.21:30500/health
Test Camera
bash
# Test USB camera
ls -la /dev/video*

# Simple Python test script
cat > test_camera.py << 'EOF'
import cv2
cap = cv2.VideoCapture(0)
if cap.isOpened():
    print("✅ Camera detected")
    ret, frame = cap.read()
    if ret:
        print(f"✅ Frame captured: {frame.shape}")
    cap.release()
else:
    print("❌ Camera not detected")
EOF
python3 test_camera.py
Test Container Runtime
bash
# Test containerd
sudo ctr version

# Test CNI
ls -la /opt/cni/bin/
⚙️ EdgeCore Configuration Files
Main Configuration: /etc/kubeedge/config/edgecore.yaml
yaml
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
🔧 Automated Setup Script
Complete Setup Script (setup-edge.sh)
bash
#!/bin/bash
set -e

echo "🚀 Starting Edge Device Setup..."

# Configuration
CLOUD_IP="3.70.53.21"
NODE_NAME="edge-node-$(hostname)"
DEVICE_ID="edge-device-$(hostname)"
TOKEN="$1"

if [ -z "$TOKEN" ]; then
    echo "❌ Error: Token required"
    echo "Usage: $0 <EDGE_TOKEN>"
    exit 1
fi

echo "📝 Configuration:"
echo "  Cloud IP: $CLOUD_IP"
echo "  Node Name: $NODE_NAME"
echo "  Device ID: $DEVICE_ID"

# Step 1: Update system
echo "🔄 Updating system..."
sudo apt update && sudo apt upgrade -y

# Step 2: Install containerd
echo "🐳 Installing containerd..."
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Step 3: Install CNI plugins
echo "🌐 Installing CNI plugins..."
sudo mkdir -p /opt/cni/bin
wget -q https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar -xzf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
rm cni-plugins-linux-amd64-v1.3.0.tgz

# Step 4: Create CNI config
echo "🔧 Creating CNI configuration..."
sudo mkdir -p /etc/cni/net.d
cat <<EOF | sudo tee /etc/cni/net.d/10-containerd-net.conflist
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
        "ranges": [
          [{
            "subnet": "10.88.0.0/16"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF

# Step 5: Install keadm
echo "📦 Installing KubeEdge keadm..."
wget -q https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.18.0-linux-amd64.tar.gz
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/
sudo chmod +x /usr/local/bin/keadm
rm -rf keadm-v1.18.0-linux-amd64*

# Step 6: Join edge node
echo "🔗 Joining edge node to cluster..."
sudo keadm join \
  --cloudcore-ipport=${CLOUD_IP}:30374 \
  --edgenode-name=${NODE_NAME} \
  --token=${TOKEN} \
  --kubeedge-version=1.18.0 \
  --remote-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroupdriver=systemd

# Step 7: Verify installation
echo "✅ Verifying installation..."
sleep 10
sudo systemctl status edgecore --no-pager

# Step 8: Set device ID
echo "🎯 Setting device ID..."
echo "export DEVICE_ID=${DEVICE_ID}" | sudo tee -a /etc/environment
echo "export NODE_NAME=${NODE_NAME}" | sudo tee -a /etc/environment

echo "🎉 Edge device setup completed!"
echo ""
echo "📋 Next steps:"
echo "1. Deploy edge application: kubectl apply -f edge-deployment.yaml"
echo "2. Check node status: kubectl get nodes"
echo "3. Monitor logs: sudo journalctl -u edgecore -f"
📊 Device Verification
Verification Script (verify-edge.sh)
bash
#!/bin/bash
echo "🔍 Verifying Edge Device Setup..."

echo "1. System Information:"
hostnamectl

echo ""
echo "2. Container Runtime:"
sudo systemctl status containerd --no-pager
containerd --version

echo ""
echo "3. CNI Plugins:"
ls -la /opt/cni/bin/ | head -10

echo ""
echo "4. EdgeCore Service:"
sudo systemctl status edgecore --no-pager

echo ""
echo "5. Network Configuration:"
ip addr show

echo ""
echo "6. Cloud Connectivity:"
timeout 5 nc -zv 3.70.53.21 30374 && echo "✅ Connected to CloudCore" || echo "❌ Cannot connect to CloudCore"

echo ""
echo "7. Running Containers:"
sudo ctr --namespace k8s.io containers ls

echo ""
echo "🔍 Verification Complete!"
🔧 Troubleshooting
Common Issues and Solutions
Issue 1: EdgeCore Fails to Start
bash
# Check logs
sudo journalctl -u edgecore -f

# Common fix: Recreate certificates
sudo rm -rf /etc/kubeedge/certs/
sudo systemctl restart edgecore
Issue 2: Cannot Connect to CloudCore
bash
# Test network connectivity
ping 3.70.53.21
nc -zv 3.70.53.21 30374

# Check firewall
sudo ufw status
Issue 3: CNI Not Initialized
bash
# Check CNI configuration
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-containerd-net.conflist

# Reinstall CNI plugins
sudo rm -rf /opt/cni/bin/*
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar -xzf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
Issue 4: Container Runtime Issues
bash
# Check containerd status
sudo systemctl status containerd

# Restart containerd
sudo systemctl restart containerd

# Test container creation
sudo ctr run --rm docker.io/library/hello-world:latest test
📈 Performance Tuning
Optimize for Raspberry Pi
bash
# Reduce swap usage
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo systemctl disable dphys-swapfile

# Enable GPU memory for camera
echo "gpu_mem=128" | sudo tee -a /boot/config.txt

# Overclock (optional)
echo "arm_freq=1400" | sudo tee -a /boot/config.txt
echo "over_voltage=4" | sudo tee -a /boot/config.txt
Optimize for x86_64
bash
# Enable performance governor
sudo apt install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils
🔒 Security Configuration
Hardening Script
bash
#!/bin/bash
echo "🔒 Hardening Edge Device..."

# Disable root login via SSH
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Configure firewall
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp  # SSH
sudo ufw allow 5001/tcp  # Edge app

# Disable unnecessary services
sudo systemctl disable avahi-daemon
sudo systemctl disable cups
sudo systemctl disable bluetooth

echo "✅ Security hardening completed!"
📦 Pre-built Images
Create Custom Edge Image
bash
#!/bin/bash
# create-edge-image.sh

# Install Packer
sudo apt install -y packer

# Create packer configuration
cat > edge-image.pkr.hcl << 'EOF'
source "qemu" "edge-os" {
  iso_url = "https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso"
  iso_checksum = "sha256:5e38b55d57d94ff029719342357325ed3bda38fa80054f9330dc789cd2d43931"
  disk_size = "10000"
  memory = "2048"
  cpus = "2"
  
  ssh_username = "ubuntu"
  ssh_password = "ubuntu"
  ssh_timeout = "30m"
  
  shutdown_command = "sudo shutdown -P now"
}

build {
  sources = ["source.qemu.edge-os"]
  
  provisioner "shell" {
    inline = [
      "sudo apt update",
      "sudo apt upgrade -y",
      "sudo apt install -y containerd curl wget git",
      # Add your edge setup commands here
    ]
  }
}
EOF

# Build image
packer build edge-image.pkr.hcl
🧪 Testing Environment
Docker Compose for Testing
yaml
# docker-compose-test.yml
version: '3.8'
services:
  edge-simulator:
    image: ubuntu:22.04
    container_name: edge-test
    privileged: true
    network_mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./edge-config:/etc/kubeedge
    command: >
      sh -c "
      apt update && apt install -y curl &&
      curl -sfL https://get.k3s.io | sh - &&
      sleep infinity"
📋 Checklist
Pre-installation Checklist
Hardware meets requirements

OS is supported version

Network connectivity to cloud

Sufficient storage (20GB+)

Sufficient RAM (2GB+)

Installation Checklist
System updated

containerd installed and configured

CNI plugins installed

CNI configuration created

keadm installed

Edge token obtained

EdgeCore joined to cluster

EdgeCore service running

Post-installation Checklist
Node appears in cloud cluster

Can deploy edge applications

Camera detected (if applicable)

Network connectivity verified

Security measures applied

📞 Support
Getting Help
Check logs: sudo journalctl -u edgecore -f

Verify connectivity: ping 3.70.53.21

Check system resources: htop, df -h

Review configuration files: Check /etc/kubeedge/config/

Common Resources
KubeEdge Documentation: https://kubeedge.io

GitHub Issues: https://github.com/kubeedge/kubeedge/issues

Community Slack: #kubeedge on Kubernetes Slack

🎯 Next Steps
After EdgeCore setup:

Deploy edge applications

Configure monitoring

Set up backup procedures

Implement security policies

Test failover scenarios
