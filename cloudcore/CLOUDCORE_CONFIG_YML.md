# 🚀 KubeEdge CloudCore – Complete Configuration (All-in-One)

This document contains **all configurations** required for CloudCore, K3s, Services, Certificates, Deployments, RBAC, Monitoring, Security, and Troubleshooting — all combined in one file for easy GitHub publishing.

---

# 🔧 CloudCore Main Configuration (`cloudcore.yaml`)

```yaml
apiVersion: cloudcore.config.kubeedge.io/v1alpha1
kind: CloudCore
modules:
  cloudHub:
    advertiseAddress:
    - 3.70.53.21
    enable: true
    keepAliveInterval: 30
    nodeLimit: 100
    port: 10000
    quic:
      enable: false
      maxIncomingStreams: 10000
      port: 10001
    unixsocket:
      address: unix:///var/lib/kubeedge/cloudcore.sock
      enable: true
    websocket:
      enable: true
      handshakeTimeout: 30
      port: 10004
      readDeadline: 15
      writeDeadline: 15
  
  cloudStream:
    enable: true
    streamPort: 10002
    tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt
    tlsStreamCertFile: /etc/kubeedge/certs/stream.crt
    tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key
  
  edgeController:
    enable: true
    nodeUpdateFrequency: 10
    buffer:
      updatePodStatus: 1024
  
  deviceController:
    enable: true
    buffer:
      updateDeviceStatus: 1024

  syncController:
    enable: true

kubeAPIConfig:
  burst: 200
  contentType: "application/vnd.kubernetes.protobuf"
  kubeConfig: "/etc/kubeedge/kubeconfig.yaml"
  qps: 100
📄 K3s Kubernetes Config (k3s.yaml)
yaml
Copy code
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://127.0.0.1:6443  
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJU...
    client-key-data: LS0tLS1CRUdJTiBSU0EtUFJJVkFURS0t...
🔐 Certificates
Root CA (rootCA.crt)
pem
Copy code
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL5M6PjD1QOMA0GCSq...
-----END CERTIFICATE-----
CloudCore Certificate (cloudcore.crt)
pem
Copy code
-----BEGIN CERTIFICATE-----
MIIDYzCCAkugAwIBAgIIV5JZ5Z5Z5Z4wDQYJ...
-----END CERTIFICATE-----
🔌 CloudCore Systemd Service (cloudcore.service)
ini
Copy code
[Unit]
Description=KubeEdge CloudCore
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/cloudcore
Restart=always

[Install]
WantedBy=multi-user.target
🐳 K3s Service (k3s.service)
ini
Copy code
[Unit]
Description=Lightweight Kubernetes
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/k3s server
Restart=always

[Install]
WantedBy=multi-user.target
🚀 CloudCore Deployment (deployment.yaml)
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudcore
  namespace: kubeedge
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cloudcore
    spec:
      serviceAccountName: cloudcore
      containers:
      - name: cloudcore
        image: kubeedge/cloudcore:v1.18.0
        args:
        - --config=/etc/kubeedge/config/cloudcore.yaml
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
🌐 CloudCore Service (service.yaml)
yaml
Copy code
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
🧩 ConfigMap (cloudcore-config.yaml)
yaml
Copy code
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
        advertiseAddress:
        - 3.70.53.21
        enable: true
🔑 RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding)
ServiceAccount
yaml
Copy code
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudcore
  namespace: kubeedge
ClusterRole
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudcore
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status"]
  verbs: ["get", "list", "watch", "update"]
ClusterRoleBinding
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloudcore
roleRef:
  kind: ClusterRole
  name: cloudcore
subjects:
- kind: ServiceAccount
  name: cloudcore
  namespace: kubeedge
🟦 Master App Deployment
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: master-app
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: master-app
    spec:
      containers:
      - name: master-app
        image: ahmedxeno/master-app:latest
        ports:
        - containerPort: 5000
🟩 Master App Service
yaml
Copy code
apiVersion: v1
kind: Service
metadata:
  name: master-app
spec:
  type: NodePort
  ports:
  - port: 5000
    nodePort: 30500
  selector:
    app: master-app
📊 Monitoring (Prometheus + Grafana)
ServiceMonitor
yaml
Copy code
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
Grafana Dashboard
yaml
Copy code
{
  "title": "KubeEdge CloudCore Dashboard",
  "panels": [
    {
      "title": "Active Edge Nodes",
      "targets": [{"expr": "kubeedge_cloudcore_active_devices"}]
    }
  ]
}
🛡️ Security – Network Policy
yaml
Copy code
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
🛠️ Troubleshooting Script
bash
Copy code
#!/bin/bash
echo "Checking CloudCore..."
kubectl get pods -n kubeedge
kubectl logs -n kubeedge deployment/cloudcore --tail=50
🧪 Health Check Script
python
Copy code
import requests

def check_cloudcore():
    for ep in ["http://localhost:10000/readyz"]:
        try:
            r = requests.get(ep, timeout=5)
            print(ep, r.status_code)
        except Exception as e:
            print(ep, "error:", e)
✔️ Configuration Validation Script
bash
Copy code
#!/bin/bash

echo "Validating CloudCore..."

if [ -f"/etc/kubeedge/config/cloudcore.yaml" ]; then
  echo "OK"
fi

systemctl is-active --quiet cloudcore && echo "CloudCore running"
🎯 Tips
Enable TLS everywhere

Adjust buffer sizes for large deployments

Monitor certificate expiration

Tune WebSocket & keepalive for network conditions
``
