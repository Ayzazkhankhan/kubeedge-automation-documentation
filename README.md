# KubeEdge Edge Computing Platform

## 🚀 Overview
A complete edge computing platform using KubeEdge for managing distributed edge devices with real-time computer vision capabilities.

## 📋 Features
- ✅ Kubernetes-based edge device management
- ✅ Real-time video processing on edge
- ✅ Web dashboard for monitoring
- ✅ Automated device provisioning
- ✅ Scalable architecture

## 🔧 Prerequisites
- AWS EC2 instance (Ubuntu 22.04)
- Edge devices with Ubuntu 22.04
- Docker & Kubernetes knowledge
- Basic Python/Node.js knowledge

## 📁 Documentation Structure

### Core Components
1. **CloudCore Setup** (`cloudcore/CLOUDCORE_SETUP.md`) - Cloud server setup
2. **EdgeCore Setup** (`edgecore/EDGECORE_SETUP.md`) - Edge device setup
3. **Applications** (`applications/`) - Master & edge applications

### Configuration Files
- `cloudcore/CLOUDCORE_CONFIG_YML.md` - CloudCore YAML files
- `edgecore/EDGECORE_CONFIG_YML.md` - EdgeCore YAML files

### API Documentation
- `api/REST_API.md` - REST API endpoints
- `api/WEBSOCKET_API.md` - Real-time APIs
- `api/KUBERNETES_API.md` - Kubernetes API usage

## 🚦 Quick Start

### 1. Deploy CloudCore
```bash
# Clone repository
git clone https://github.com/yourusername/kubeedge-platform.git
cd kubeedge-platform

# Deploy cloud components
cd scripts
./deploy-cloudcore.sh
