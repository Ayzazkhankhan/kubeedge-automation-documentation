# 📖 CloudCore API Documentation – All-in-One

[![KubeEdge](https://img.shields.io/badge/KubeEdge-v1.18.0-blue)](https://kubeedge.io)
[![API](https://img.shields.io/badge/API-REST-green)](https://swagger.io)

&gt; Complete OpenAPI reference – **copy-paste friendly**.  
&gt; Base URL: `http://3.70.53.21:5000/api/kubeedge`

---

## 🔐 Authentication

### JWT (User)
```bash
# Login → get token
curl -X POST http://3.70.53.21:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# Use token
curl -H "Authorization: Bearer &lt;token&gt;" \
     http://3.70.53.21:5000/api/kubeedge/devices
```

### ServiceAccount (Cluster-internal)
```python
from kubernetes import config
config.load_incluster_config()   # uses pod SA
```

---

## 📊 Endpoints Quick Index

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | API health |
| POST | `/cloudcore/deploy` | Deploy CloudCore |
| GET | `/cloudcore/status` | CloudCore pods & services |
| POST | `/token/generate` | Create edge join token |
| GET | `/tokens` | List tokens |
| DELETE | `/token/{node_name}` | Revoke token |
| POST | `/device/join` | Add single edge device |
| GET | `/devices` | List all devices |
| GET | `/device/{id}` | Single device details |
| DELETE | `/device/{id}` | Remove device |
| POST | `/batch-join` | Add multiple devices |
| GET | `/monitor/stream` | SSE real-time events |
| GET | `/metrics` | Aggregated metrics |
| GET | `/config` | Current config |
| PUT | `/config` | Update config |
| POST | `/apps/deploy` | Deploy app to edges |
| POST | `/device/{id}/command` | Send command |
| GET | `/command/{id}/status` | Command result |
| POST | `/backup` | Create backup |
| POST | `/restore` | Restore from backup |
| GET | `/webhooks` | Webhook mgmt |
| POST | `/test/connection` | Test edge reachability |
| GET | `/openapi.json` | OpenAPI spec |
| GET | `/swagger-ui` | Swagger UI |
| GET | `/postman.json` | Postman collection |

---

## 1. Health Check
```bash
curl http://3.70.53.21:5000/api/health
```
```json
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
```

---

## 2. CloudCore Management

### Deploy / Update
```bash
curl -X POST http://3.70.53.21:5000/api/cloudcore/deploy \
  -H "Authorization: Bearer &lt;token&gt;" \
  -H "Content-Type: application/json" \
  -d '{"version":"1.18.0","replicas":1}'
```
```json
{
  "status": "success",
  "deployment_name": "cloudcore",
  "namespace": "kubeedge",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Status
```bash
curl -H "Authorization: Bearer &lt;token&gt;" \
     http://3.70.53.21:5000/api/cloudcore/status
```
```json
{
  "status": "running",
  "pods": [{"name":"cloudcore-xxx","ready":"1/1","restarts":0}],
  "services": [{"name":"cloudcore","type":"NodePort","ports":[{"nodePort":30374}]}]
}
```

---

## 3. Token Management

### Generate
```bash
curl -X POST http://3.70.53.21:5000/api/token/generate \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"node_name":"edge-001","expiry_hours":72}'
```
```json
{
  "token": "dab11b0b6882...eyJ...",
  "expires_at": "2024-01-18T10:30:00Z",
  "join_command": "keadm join --cloudcore-ipport=3.70.53.21:30374 --edgenode-name=edge-001 --token=..."
}
```

### List
```bash
curl -H "Authorization: Bearer &lt;token&gt;" \
     http://3.70.53.21:5000/api/tokens
```

### Revoke
```bash
curl -X DELETE http://3.70.53.21:5000/api/token/edge-001 \
  -H "Authorization: Bearer &lt;token&gt;"
```

---

## 4. Device Management

### Add Single Device
```bash
curl -X POST http://3.70.53.21:5000/api/device/join \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"node_name":"edge-001","device_id":"camera-001"}'
```

### List Devices
```bash
curl -H "Authorization: Bearer &lt;token&gt;" \
     http://3.70.53.21:5000/api/devices
```

### Remove Device
```bash
curl -X DELETE http://3.70.53.21:5000/api/device/camera-001 \
  -H "Authorization: Bearer &lt;token&gt;"
```

### Batch Join
```bash
curl -X POST http://3.70.53.21:5000/api/batch-join \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"devices":[{"node_name":"e-001","device_id":"c-001"},{"node_name":"e-002","device_id":"c-002"}]}'
```

---

## 5. Real-Time & Metrics

### Server-Sent Events (SSE)
```bash
curl http://3.70.53.21:5000/api/monitor/stream
```
Stream sample:
```
data: {"event":"device_joined","node_name":"e-001","timestamp":"..."}
```

### Aggregated Metrics
```bash
curl -H "Authorization: Bearer &lt;token&gt;" \
     http://3.70.53.21:5000/api/metrics
```
```json
{
  "total_devices": 15,
  "online_devices": 12,
  "total_detections_today": 1250,
  "average_latency": "45ms"
}
```

---

## 6. Command Execution

### Send Command
```bash
curl -X POST http://3.70.53.21:5000/api/device/camera-001/command \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"command":"capture_image","parameters":{"resolution":"1920x1080"}}'
```

### Check Result
```bash
curl http://3.70.53.21:5000/api/command/cmd_12345/status
```
```json
{
  "status": "completed",
  "result": {"image_url":"http://s3.../img.jpg","size":"1.2MB"}
}
```

---

## 7. Backup & Restore

### Create Backup
```bash
curl -X POST http://3.70.53.21:5000/api/backup \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"name":"daily-20240115","include":["config","tokens","devices"]}'
```

### Restore
```bash
curl -X POST http://3.70.53.21:5000/api/restore \
  -H "Authorization: Bearer &lt;token&gt;" \
  -d '{"backup_id":"daily-20240115"}'
```

---

## 8. Python SDK (quick start)

Install:
```bash
pip install kubeedge-cloudcore-sdk
```

Usage:
```python
from kubeedge_cloudcore import CloudCoreClient

client = CloudCoreClient(base_url="http://3.70.53.21:5000", token="&lt;token&gt;")

# List devices
devices = client.list_devices()
print(devices)

# Join device
result = client.join_device(node_name="e-001", device_id="c-001")
print(result["join_command"])

# Real-time stream
for event in client.monitor_stream():
    print(event)
```

---

## 9. WebSocket Events

Connect:
```javascript
const ws = new WebSocket('ws://3.70.53.21:5000/ws');
ws.onmessage = (e) =&gt; console.log(JSON.parse(e.data));
```

Main events:
| Event | Sample Payload |
|-------|----------------|
| `device_joined` | `{"node_name":"e-001","timestamp":"..."}` |
| `detection` | `{"device_id":"c-001","object":"person","confidence":0.95}` |
| `command_result` | `{"command_id":"cmd123","result":"ok"}` |

---

## 10. MQTT Interface

Broker: `tcp://3.70.53.21:1883` (no auth for read)

Subscribe:
```bash
mosquitto_sub -h 3.70.53.21 -t "cloudcore/device/+/status" -v
```

Publish command:
```bash
mosquitto_pub -h 3.70.53.21 -t "cloudcore/device/camera-001/command" -m '{"command":"reboot"}'
```

---

## 11. Rate Limits & Headers

Limits:
- API: 100 req/min per token
- WebSocket: 50 concurrent
- MQTT: 100 concurrent

Response headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1610739180
```

---

## 12. Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| API-001 | 400 | Invalid params |
| API-002 | 401 | Auth failed |
| API-003 | 403 | Forbidden |
| API-004 | 404 | Not found |
| API-005 | 409 | Conflict |
| API-006 | 429 | Rate limit |
| API-007 | 500 | Server error |
| API-008 | 502 | Edge unreachable |

---

## 13. OpenAPI & Collections

| URL | Description |
|-----|-------------|
| `/openapi.json` | Swagger 3.0 JSON |
| `/swagger-ui` | Interactive docs |
| `/postman.json` | Postman collection |

---

## 14. Quick Reference Commands

```bash
# 1. Add device
curl -X POST $BASE/device/join -H "Authorization: Bearer $TOKEN" \
  -d '{"node_name":"e-001","device_id":"c-001"}'

# 2. Send command
curl -X POST $BASE/device/c-001/command -H "Authorization: Bearer $TOKEN" \
  -d '{"command":"capture_image"}'

# 3. Live stream
curl $BASE/monitor/stream
```

---

## 🎯 Next Step
Proceed to **EdgeCore join** – use the token you saved and head to the EdgeCore guide in the same repo.
