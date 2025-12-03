# CloudCore API Documentation

## 📋 Overview

The CloudCore API provides programmatic access to manage KubeEdge CloudCore components, edge devices, and tokens. This API is built on Flask and integrates directly with Kubernetes API.

## 🚀 Base URL
http://3.70.53.21:5000/api/kubeedge
https://your-cloud-ip:5000/api/kubeedge (if HTTPS configured)

text

## 🔐 Authentication

### JWT Authentication
All endpoints (except public ones) require JWT token authentication.

```bash
# Get token
curl -X POST http://3.70.53.21:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}'

# Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "username": "admin",
    "role": "admin"
  }
}

# Use token
curl -H "Authorization: Bearer <token>" \
  http://3.70.53.21:5000/api/kubeedge/devices
Service Account Authentication
For internal services, use Kubernetes ServiceAccount token:

python
from kubernetes import config
config.load_incluster_config()  # Automatically uses service account
📊 API Endpoints
1. Health Check
GET /api/health
Check if CloudCore API is running.

Request:

bash
curl http://3.70.53.21:5000/api/health
Response:

json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0.0",
  "components": {
    "kubernetes": "connected",
    "cloudcore": "running",
    "database": "connected"
  }
}
2. CloudCore Management
POST /api/kubeedge/cloudcore/deploy
Deploy or update CloudCore.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/cloudcore/deploy \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "version": "1.18.0",
    "replicas": 1,
    "resources": {
      "cpu": "500m",
      "memory": "512Mi"
    }
  }'
Response:

json
{
  "status": "success",
  "message": "CloudCore deployed",
  "deployment_name": "cloudcore",
  "namespace": "kubeedge",
  "created_at": "2024-01-15T10:30:00Z"
}
GET /api/kubeedge/cloudcore/status
Get CloudCore status.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/cloudcore/status \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "status": "running",
  "version": "1.18.0",
  "pods": [
    {
      "name": "cloudcore-6fd9f5779-kcjq8",
      "status": "Running",
      "ready": "1/1",
      "restarts": 0,
      "age": "2d"
    }
  ],
  "services": [
    {
      "name": "cloudcore",
      "type": "NodePort",
      "ports": [
        {"name": "cloudhub", "port": 10000, "nodePort": 30374},
        {"name": "cloudstream", "port": 10002, "nodePort": 31305},
        {"name": "websocket", "port": 10004, "nodePort": 32203}
      ]
    }
  ]
}
3. Token Management
POST /api/kubeedge/token/generate
Generate new edge join token.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/token/generate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "node_name": "edge-node-001",
    "expiry_hours": 72,
    "description": "Factory floor device"
  }'
Response:

json
{
  "status": "success",
  "node_name": "edge-node-001",
  "token": "dab11b0b6882dd33cd715a3bc9fe0bcbefd29daf56904d42d2ccbcc6da1aa361.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NjQ0ODA4NzZ9.jgFzkOUfFqF73rRptL7SHGFHcUxV0vy5TF4KFSzNjzI",
  "expires_at": "2024-01-18T10:30:00Z",
  "join_command": "keadm join --cloudcore-ipport=3.70.53.21:30374 --edgenode-name=edge-node-001 --token=..."
}
GET /api/kubeedge/tokens
List all generated tokens.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/tokens \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "count": 3,
  "tokens": [
    {
      "node_name": "edge-node-001",
      "created_at": "2024-01-15T10:30:00Z",
      "expires_at": "2024-01-18T10:30:00Z",
      "status": "active",
      "device_id": "edge-device-001"
    },
    {
      "node_name": "edge-node-002",
      "created_at": "2024-01-14T15:45:00Z",
      "expires_at": "2024-01-17T15:45:00Z",
      "status": "expired",
      "device_id": "edge-device-002"
    }
  ]
}
DELETE /api/kubeedge/token/{node_name}
Revoke a token.

Request:

bash
curl -X DELETE http://3.70.53.21:5000/api/kubeedge/token/edge-node-001 \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "status": "success",
  "message": "Token revoked for edge-node-001"
}
4. Device Management
POST /api/kubeedge/device/join
Join a new edge device.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/device/join \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "node_name": "edge-node-001",
    "device_id": "edge-device-001",
    "cloud_ip": "3.70.53.21",
    "device_type": "camera",
    "location": "Factory Floor 1",
    "configuration": {
      "cni_type": "containerd",
      "runtime": "containerd",
      "camera_resolution": "1920x1080"
    }
  }'
Response:

json
{
  "status": "success",
  "node_name": "edge-node-001",
  "device_id": "edge-device-001",
  "configmap_created": "edgecore-config-edge-node-001",
  "device_crd_created": "edge-device-001",
  "join_command": "keadm join --cloudcore-ipport=3.70.53.21:30374 --edgenode-name=edge-node-001 --token=...",
  "steps": {
    "token_generated": true,
    "configmap_created": true,
    "device_crd_created": true,
    "rbac_configured": true
  }
}
GET /api/kubeedge/devices
List all edge devices.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/devices \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "count": 2,
  "devices": [
    {
      "node_name": "edge-node-laptop",
      "device_id": "edge-device-laptop-001",
      "status": "Ready",
      "roles": ["agent", "edge"],
      "age": "2d",
      "version": "v1.29.5-kubeedge-v1.18.0",
      "pods_count": 1,
      "addresses": [
        {"type": "InternalIP", "address": "192.168.1.100"}
      ]
    },
    {
      "node_name": "edge-node-raspberry",
      "device_id": "edge-device-raspberry-002",
      "status": "NotReady",
      "roles": ["agent", "edge"],
      "age": "1d",
      "version": "v1.29.5-kubeedge-v1.18.0",
      "pods_count": 0,
      "addresses": []
    }
  ]
}
GET /api/kubeedge/device/{device_id}
Get specific device details.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/device/edge-device-laptop-001 \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "device_id": "edge-device-laptop-001",
  "node_name": "edge-node-laptop",
  "device_status": {
    "online": true,
    "last_heartbeat": "2024-01-15T10:29:45Z",
    "cpu_usage": "45%",
    "memory_usage": "2.3GB/8GB",
    "temperature": "65°C"
  },
  "configuration": {
    "runtime": "containerd",
    "cni_version": "1.3.0",
    "kubeedge_version": "1.18.0",
    "camera_enabled": true
  },
  "applications": [
    {
      "name": "edge-app",
      "status": "Running",
      "image": "ahmedxeno/edge-app:latest",
      "port": 5001
    }
  ],
  "metrics": {
    "uptime": "2 days",
    "total_detections": 1250,
    "average_latency": "45ms"
  }
}
DELETE /api/kubeedge/device/{device_id}
Remove an edge device.

Request:

bash
curl -X DELETE http://3.70.53.21:5000/api/kubeedge/device/edge-device-001 \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "status": "success",
  "message": "Device edge-device-001 removed",
  "actions_taken": [
    "Device CRD deleted",
    "ConfigMap deleted",
    "Node removed from cluster",
    "Token revoked"
  ]
}
5. Batch Operations
POST /api/kubeedge/batch-join
Join multiple devices in batch.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/batch-join \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "devices": [
      {
        "node_name": "edge-node-001",
        "device_id": "camera-001",
        "device_type": "camera",
        "location": "Entrance"
      },
      {
        "node_name": "edge-node-002",
        "device_id": "camera-002",
        "device_type": "camera",
        "location": "Exit"
      },
      {
        "node_name": "edge-node-003",
        "device_id": "sensor-001",
        "device_type": "sensor",
        "location": "Warehouse"
      }
    ]
  }'
Response:

json
{
  "total": 3,
  "success": 2,
  "failed": 1,
  "results": [
    {
      "device": "camera-001",
      "status": "success",
      "details": {
        "node_name": "edge-node-001",
        "device_id": "camera-001"
      }
    },
    {
      "device": "camera-002",
      "status": "success",
      "details": {
        "node_name": "edge-node-002",
        "device_id": "camera-002"
      }
    },
    {
      "device": "sensor-001",
      "status": "failed",
      "error": "Network timeout during configuration"
    }
  ]
}
6. Real-time Monitoring
GET /api/kubeedge/monitor/stream
Server-Sent Events (SSE) stream for real-time monitoring.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/monitor/stream
Response (Stream):

text
data: {"timestamp": "2024-01-15T10:30:00Z", "event": "node_joined", "node_name": "edge-node-001", "status": "Ready"}

data: {"timestamp": "2024-01-15T10:30:05Z", "event": "heartbeat", "node_name": "edge-node-001", "cpu": "45%", "memory": "2.3GB"}

data: {"timestamp": "2024-01-15T10:30:10Z", "event": "detection", "node_name": "edge-node-001", "object": "person", "confidence": "0.95"}
GET /api/kubeedge/metrics
Get aggregated metrics.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/metrics \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "summary": {
    "total_devices": 15,
    "online_devices": 12,
    "offline_devices": 3,
    "total_detections_today": 1250,
    "average_latency": "45ms",
    "storage_used": "45GB"
  },
  "devices_by_type": {
    "camera": 8,
    "sensor": 5,
    "gateway": 2
  },
  "alerts": [
    {
      "device_id": "camera-001",
      "type": "high_cpu",
      "message": "CPU usage > 90% for 5 minutes",
      "timestamp": "2024-01-15T10:25:00Z"
    }
  ]
}
7. Configuration Management
GET /api/kubeedge/config
Get current CloudCore configuration.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/config \
  -H "Authorization: Bearer <token>"
Response:

json
{
  "cloudcore": {
    "version": "1.18.0",
    "advertise_address": "3.70.53.21",
    "ports": {
      "cloudhub": 10000,
      "cloudstream": 10002,
      "websocket": 10004
    },
    "nodeports": {
      "cloudhub": 30374,
      "cloudstream": 31305,
      "websocket": 32203
    }
  },
  "kubernetes": {
    "version": "v1.33.6+k3s1",
    "nodes": 2,
    "pods": 15
  },
  "security": {
    "tls_enabled": true,
    "token_expiry": 72,
    "rate_limiting": true
  }
}
PUT /api/kubeedge/config
Update CloudCore configuration.

Request:

bash
curl -X PUT http://3.70.53.21:5000/api/kubeedge/config \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "token_expiry": 96,
    "max_devices": 100,
    "security": {
      "require_tls": true,
      "rate_limit": 100
    }
  }'
8. Application Deployment
POST /api/kubeedge/apps/deploy
Deploy application to edge devices.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/apps/deploy \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "object-detection",
    "image": "ahmedxeno/edge-app:latest",
    "device_ids": ["edge-device-001", "edge-device-002"],
    "config": {
      "model": "yolov5",
      "confidence": 0.7,
      "interval": 5
    },
    "resources": {
      "cpu": "500m",
      "memory": "512Mi"
    }
  }'
Response:

json
{
  "status": "success",
  "deployment_name": "object-detection",
  "devices_deployed": 2,
  "services_created": [
    {
      "device_id": "edge-device-001",
      "port": 5001,
      "url": "http://edge-device-001:5001"
    }
  ]
}
GET /api/kubeedge/apps
List deployed applications.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/apps \
  -H "Authorization: Bearer <token>"
9. Command Execution
POST /api/kubeedge/device/{device_id}/command
Send command to edge device.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/device/edge-device-001/command \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "command": "capture_image",
    "parameters": {
      "resolution": "1920x1080",
      "format": "jpg",
      "quality": 90
    },
    "timeout": 30
  }'
Response:

json
{
  "status": "success",
  "command_id": "cmd_12345",
  "device_id": "edge-device-001",
  "command": "capture_image",
  "sent_at": "2024-01-15T10:30:00Z",
  "expected_response_timeout": 30
}
GET /api/kubeedge/command/{command_id}/status
Check command execution status.

Request:

bash
curl http://3.70.53.21:5000/api/kubeedge/command/cmd_12345/status
Response:

json
{
  "command_id": "cmd_12345",
  "status": "completed",
  "result": {
    "success": true,
    "image_url": "http://s3.amazonaws.com/bucket/image_12345.jpg",
    "size": "1.2MB",
    "timestamp": "2024-01-15T10:30:15Z"
  },
  "execution_time": "15s"
}
10. Backup & Restore
POST /api/kubeedge/backup
Create backup of CloudCore configuration.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/backup \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "backup-20240115",
    "include": ["config", "tokens", "devices"],
    "description": "Daily backup"
  }'
Response:

json
{
  "status": "success",
  "backup_id": "backup_20240115",
  "size": "45MB",
  "location": "s3://kubeedge-backups/backup-20240115.tar.gz",
  "created_at": "2024-01-15T10:30:00Z"
}
POST /api/kubeedge/restore
Restore from backup.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/restore \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "backup_id": "backup_20240115"
  }'
🔧 Python SDK Example
Installation
bash
pip install kubeedge-cloudcore-sdk
Usage
python
from kubeedge_cloudcore import CloudCoreClient

# Initialize client
client = CloudCoreClient(
    base_url="http://3.70.53.21:5000",
    api_key="your-api-key"
)

# List devices
devices = client.list_devices()
print(f"Total devices: {len(devices)}")

# Join new device
result = client.join_device(
    node_name="edge-node-001",
    device_id="camera-001",
    device_type="camera"
)
print(f"Join command: {result['join_command']}")

# Send command
response = client.send_command(
    device_id="camera-001",
    command="capture_image",
    parameters={"resolution": "1920x1080"}
)
print(f"Command ID: {response['command_id']}")

# Real-time monitoring
for event in client.monitor_stream():
    print(f"Event: {event['event']} from {event['node_name']}")
🌐 WebSocket API
Connect to WebSocket
javascript
const ws = new WebSocket('ws://3.70.53.21:5000/ws');

ws.onopen = () => {
  console.log('Connected to CloudCore WebSocket');
  // Subscribe to events
  ws.send(JSON.stringify({
    action: 'subscribe',
    topics: ['device_status', 'detections', 'commands']
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
  
  switch(data.type) {
    case 'device_joined':
      console.log(`Device ${data.node_name} joined`);
      break;
    case 'detection':
      console.log(`Detection: ${data.object} with ${data.confidence} confidence`);
      break;
    case 'command_result':
      console.log(`Command ${data.command_id} completed: ${data.result}`);
      break;
  }
};
WebSocket Events
Event	Description	Data Format
device_joined	New edge device joined	{"type": "device_joined", "node_name": "...", "device_id": "...", "timestamp": "..."}
device_left	Edge device disconnected	{"type": "device_left", "node_name": "...", "reason": "...", "timestamp": "..."}
heartbeat	Device heartbeat	{"type": "heartbeat", "node_name": "...", "cpu": "...", "memory": "...", "timestamp": "..."}
detection	Object detection event	{"type": "detection", "device_id": "...", "object": "...", "confidence": 0.95, "timestamp": "..."}
command_sent	Command sent to device	{"type": "command_sent", "command_id": "...", "device_id": "...", "command": "...", "timestamp": "..."}
command_result	Command execution result	{"type": "command_result", "command_id": "...", "device_id": "...", "result": "...", "timestamp": "..."}
alert	System alert	{"type": "alert", "level": "warning", "message": "...", "timestamp": "..."}
📡 MQTT API
MQTT Topics
CloudCore also provides MQTT interface for IoT integration:

text
Topic: cloudcore/device/{device_id}/command
Payload: {"command": "capture_image", "parameters": {...}}

Topic: cloudcore/device/{device_id}/status
Payload: {"online": true, "cpu": "45%", "memory": "2.3GB"}

Topic: cloudcore/broadcast/command
Payload: {"command": "reboot", "device_type": "camera"}
MQTT Client Example
python
import paho.mqtt.client as mqtt

def on_connect(client, userdata, flags, rc):
    print(f"Connected with result code {rc}")
    client.subscribe("cloudcore/device/+/status")
    client.subscribe("cloudcore/alerts")

def on_message(client, userdata, msg):
    print(f"Topic: {msg.topic}, Message: {msg.payload.decode()}")

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message
client.connect("3.70.53.21", 1883, 60)
client.loop_forever()
📊 Rate Limiting
Limits
API Rate Limit: 100 requests/minute per API key

WebSocket Connections: 50 concurrent connections

MQTT Connections: 100 concurrent connections

Batch Operations: 10 devices per batch

Headers
Rate limit information in response headers:

text
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1610739180
🔐 Security Endpoints
GET /api/kubeedge/security/certificates
List SSL certificates.

POST /api/kubeedge/security/certificates/renew
Renew SSL certificates.

GET /api/kubeedge/security/logs
Get security logs.

🗃️ Database Schema
Devices Table
sql
CREATE TABLE devices (
    id SERIAL PRIMARY KEY,
    device_id VARCHAR(100) UNIQUE,
    node_name VARCHAR(100),
    device_type VARCHAR(50),
    location VARCHAR(200),
    status VARCHAR(20),
    last_seen TIMESTAMP,
    configuration JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
Tokens Table
sql
CREATE TABLE tokens (
    id SERIAL PRIMARY KEY,
    token_hash VARCHAR(256) UNIQUE,
    node_name VARCHAR(100),
    device_id VARCHAR(100),
    expires_at TIMESTAMP,
    used BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
Commands Table
sql
CREATE TABLE commands (
    id SERIAL PRIMARY KEY,
    command_id VARCHAR(100) UNIQUE,
    device_id VARCHAR(100),
    command_type VARCHAR(50),
    parameters JSONB,
    status VARCHAR(20),
    result JSONB,
    sent_at TIMESTAMP,
    completed_at TIMESTAMP
);
🧪 Testing Endpoints
POST /api/kubeedge/test/connection
Test connection to edge device.

Request:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/test/connection \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "device_id": "edge-device-001",
    "tests": ["ping", "port_check", "camera_test"]
  }'
GET /api/kubeedge/test/results/{test_id}
Get test results.

🚨 Error Codes
Code	HTTP Status	Description
API-001	400	Invalid request parameters
API-002	401	Authentication failed
API-003	403	Insufficient permissions
API-004	404	Resource not found
API-005	409	Resource already exists
API-006	429	Rate limit exceeded
API-007	500	Internal server error
API-008	502	Edge device unreachable
API-009	503	Service unavailable
📝 API Versioning
Version Header
text
Accept: application/vnd.kubeedge.v1+json
Deprecation
Deprecated endpoints include:

text
Deprecation: true
Sunset: Wed, 15 Jan 2025 00:00:00 GMT
Alternate: /api/v2/kubeedge/devices
🔄 Webhooks
Configure Webhook
bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/webhooks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "url": "https://your-server.com/webhook",
    "events": ["device_joined", "device_left", "alert"],
    "secret": "your-webhook-secret"
  }'
Webhook Payload Example
json
{
  "event": "device_joined",
  "data": {
    "device_id": "edge-device-001",
    "node_name": "edge-node-001",
    "timestamp": "2024-01-15T10:30:00Z"
  },
  "signature": "sha256=..."
}
📈 Performance Metrics
GET /api/kubeedge/metrics/prometheus
Prometheus metrics endpoint.

GET /api/kubeedge/metrics/health
Health check with detailed metrics.

🛠️ Maintenance Endpoints
POST /api/kubeedge/maintenance/start
Start maintenance mode.

POST /api/kubeedge/maintenance/stop
Stop maintenance mode.

GET /api/kubeedge/maintenance/status
Get maintenance status.

📚 API Documentation
OpenAPI Specification
text
GET /api/kubeedge/openapi.json
GET /api/kubeedge/swagger-ui
Postman Collection
text
GET /api/kubeedge/postman.json
🎯 Quick Reference
Common Operations
Add new device:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/device/join \
  -H "Authorization: Bearer <token>" \
  -d '{"node_name": "new-node", "device_id": "new-device"}'
Send command:

bash
curl -X POST http://3.70.53.21:5000/api/kubeedge/device/device-001/command \
  -H "Authorization: Bearer <token>" \
  -d '{"command": "get_status"}'
Monitor real-time:

bash
curl http://3.70.53.21:5000/api/kubeedge/monitor/stream
Get device list:

bash
curl http://3.70.53.21:5000/api/kubeedge/devices \
  -H "Authorization: Bearer <token>"
