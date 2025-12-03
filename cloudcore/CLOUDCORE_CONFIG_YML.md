# 🚀 KubeEdge CloudCore – Complete Configuration (All-in-One)
&gt; Copy-paste **entire** file into GitHub README or any page – zero folder structure needed.

---

## 🔧 1. CloudCore Main Config (`cloudcore.yaml`)
```yaml
apiVersion: cloudcore.config.kubeedge.io/v1alpha1
kind: CloudCore
modules:
  cloudHub:
    advertiseAddress: [3.70.53.21]
    enable: true
    keepAliveInterval: 30
    nodeLimit: 100
    port: 10000
    quic: {enable: false, maxIncomingStreams: 10000, port: 10001}
    unixsocket: {enable: true, address: unix:///var/lib/kubeedge/cloudcore.sock}
    websocket: {enable: true, handshakeTimeout: 30, port: 10004, readDeadline: 15, writeDeadline: 15}
  cloudStream:
    enable: true
    streamPort: 10002
    tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt
    tlsStreamCertFile: /etc/kubeedge/certs/stream.crt
    tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key
  edgeController:
    enable: true
    nodeUpdateFrequency: 10
    buffer: {updatePodStatus: 1024}
  deviceController:
    enable: true
    buffer: {updateDeviceStatus: 1024}
  syncController: {enable: true}
kubeAPIConfig:
  burst: 200
  contentType: "application/vnd.kubernetes.protobuf"
  kubeConfig: "/etc/kubeedge/kubeconfig.yaml"
  qps: 100
```

---

## 📄 2. K3s kubeconfig (`k3s.yaml`)
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://127.0.0.1:6443
  name: default
contexts:
- context: {cluster: default, user: default}
  name: default
current-context: default
kind: Config
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJU...
    client-key-data: LS0tLS1CRUdJTiBSU0EtUFJJVkFURS0t...
```

---

## 🔐 3. Certificates (PEM samples)
**Root CA** (`rootCA.crt`)
```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL5M6PjD1QOMA0GCSq...
-----END CERTIFICATE-----
```

**CloudCore cert** (`cloudcore.crt`)
```
-----BEGIN CERTIFICATE-----
MIIDYzCCAkugAwIBAgIIV5JZ5Z5Z5Z4wDQYJ...
-----END CERTIFICATE-----
```

---

## 🚀 4. Systemd Services

**CloudCore** (`cloudcore.service`)
```ini
[Unit]
Description=KubeEdge CloudCore
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/cloudcore
Restart=always

[Install]
WantedBy=multi-user.target
```

**K3s** (`k3s.service`)
```ini
[Unit]
Description=Lightweight Kubernetes
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/k3s server
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 🚀 5. Kubernetes Deployment + Service + ConfigMap + RBAC

Apply with: `kubectl apply -f &lt;file&gt;`

**Deployment** (`deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudcore
  namespace: kubeedge
spec:
  replicas: 1
  selector:
    matchLabels: { app: cloudcore }
  template:
    metadata:
      labels: { app: cloudcore }
    spec:
      serviceAccountName: cloudcore
      containers:
      - name: cloudcore
        image: kubeedge/cloudcore:v1.18.0
        args: ["--config=/etc/kubeedge/config/cloudcore.yaml"]
        ports:
        - containerPort: 10000
        - containerPort: 10002
        - containerPort: 10004
        volumeMounts:
        - name: config-volume
          mountPath: /etc/kubeedge/config
      volumes:
      - name: config-volume
        configMap:
          name: cloudcore-config
```

**Service** (`service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloudcore
  namespace: kubeedge
spec:
  type: NodePort
  selector:
    app: cloudcore
  ports:
  - name: cloudhub
    port: 10000
    nodePort: 30374
  - name: cloudstream
    port: 10002
    nodePort: 31305
  - name: websocket
    port: 10004
    nodePort: 32203
```

**ConfigMap** (`cloudcore-config.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudcore-config
  namespace: kubeedge
data:
  cloudcore.yaml: |
    apiVersion: cloudcore.config.kubeedge.io/v1alpha1
    kind: CloudCore
    modules:
      cloudHub:
        advertiseAddress: [3.70.53.21]
        enable: true
        keepAliveInterval: 30
        nodeLimit: 100
        port: 10000
        websocket:
          enable: true
          port: 10004
      cloudStream:
        enable: true
        streamPort: 10002
      edgeController:
        enable: true
        nodeUpdateFrequency: 10
      deviceController:
        enable: true
```

**RBAC** (`rbac.yaml`)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudcore
  namespace: kubeedge
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudcore
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status", "pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: ["devices.kubeedge.io"]
  resources: ["devices", "devicemodels"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloudcore
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cloudcore
subjects:
- kind: ServiceAccount
  name: cloudcore
  namespace: kubeedge
```

---

## 🟦 6. Master App (dashboard) Deployment + Service

**Deployment** (`master-deployment.yaml`)
```yaml
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
```

**Service** (`master-service.yaml`)
```yaml
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
```

---

## 📊 7. Monitoring (Prometheus + Grafana)

**ServiceMonitor** (for Prometheus operator)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cloudcore-monitor
spec:
  selector:
    matchLabels:
      app: cloudcore
  endpoints:
  - port: cloudhub
    interval: 30s
```

**Grafana dashboard JSON** (paste into UI → Import)
```json
{
  "title": "KubeEdge CloudCore",
  "panels": [
    {
      "title": "Active Edge Nodes",
      "targets": [{"expr": "kubeedge_cloudcore_active_nodes"}]
    }
  ]
}
```

---

## 🛡️ 8. Security – Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloudcore-network-policy
spec:
  podSelector:
    matchLabels:
      app: cloudcore
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: edgecore
  egress:
  - {}
```

---

## 🛠️ 9. Troubleshooting Scripts

**Bash quick-check**
```bash
#!/bin/bash
echo "=== CloudCore Health ==="
kubectl get pods -n kubeedge
kubectl logs -n kubeedge deployment/cloudcore --tail=30
systemctl is-active --quiet cloudcore && echo "CloudCore systemd OK"
```

**Python health probe**
```python
import requests, sys
endpoints = ["http://localhost:10000/readyz"]
for ep in endpoints:
    try:
        r = requests.get(ep, timeout=5)
        print(ep, r.status_code)
    except Exception as e:
        print(ep, "error:", e); sys.exit(1)
```

---

## ✅ 10. One-Line Install (entire stack)
```bash
# run on fresh Ubuntu 22.04
curl -fsSL https://get.docker.com | sh - && \
curl -sfL https://get.k3s.io | sh - && \
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && \
wget -qO- https://github.com/kubeedge/kubeedge/releases/download/v1.18.0/keadm-v1.18.0-linux-amd64.tar.gz | tar -xz && \
sudo cp keadm-v1.18.0-linux-amd64/keadm/keadm /usr/local/bin/ && \
sudo keadm init --advertise-address=$(curl -s http://checkip.amazonaws.com) --kubeedge-version=1.18.0 --kube-config=/etc/rancher/k3s/k3s.yaml && \
kubectl patch svc cloudcore -n kubeedge -p '{"spec":{"type":"NodePort"}}' && \
sudo keadm gettoken --kube-config=/etc/rancher/k3s/k3s.yaml
```

---

## 🎉 Next Step
Copy the token printed above and head to the **EdgeCore join** section (or any Pi) – your cloud is ready!
