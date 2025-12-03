# CloudCore Configuration Files

## 📁 Configuration File Structure
/etc/kubeedge/config/
├── cloudcore.yaml # Main CloudCore configuration
├── ca/ # Certificate authority
│ ├── rootCA.crt
│ └── rootCA.key
└── certs/ # Server certificates
├── cloudcore.crt
└── cloudcore.key

text

## 🔧 Main Configuration: `cloudcore.yaml`

### Complete Configuration File
```yaml
apiVersion: cloudcore.config.kubeedge.io/v1alpha1
kind: CloudCore
modules:
  cloudHub:
    advertiseAddress:
    - 3.70.53.21           # AWS public IP
    enable: true
    keepAliveInterval: 30   # Seconds
    nodeLimit: 100          # Maximum edge nodes
    port: 10000            # CloudHub port
    quic:
      enable: false         # QUIC protocol disabled
      maxIncomingStreams: 10000
      port: 10001
    unixsocket:
      address: unix:///var/lib/kubeedge/cloudcore.sock
      enable: true
    websocket:
      enable: true
      handshakeTimeout: 30  # Seconds
      port: 10004          # WebSocket port
      readDeadline: 15     # Seconds
      writeDeadline: 15    # Seconds
  
  cloudStream:
    enable: true
    streamPort: 10002      # CloudStream port
    tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt
    tlsStreamCertFile: /etc/kubeedge/certs/stream.crt
    tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key
  
  edgeController:
    buffer:
      configMap: 1024
      endpoints: 1024
      pod: 1024
      queryConfigMap: 1024
      querySecret: 1024
      replicaSet: 1024
      secret: 1024
      service: 1024
      serviceAccount: 1024
      updateNodeStatus: 1024
      updatePodStatus: 1024
    enable: true
    loadBalance: false
    nodeUpdateFrequency: 10  # Seconds
  
  deviceController:
    buffer:
      deviceEvent: 1024
      deviceModelEvent: 1024
      updateDeviceStatus: 1024
    enable: true
  
  router:
    enable: false
  
  dynamicController:
    enable: false
  
  syncController:
    enable: true
  
  iptablesManager:
    enable: false
  
  cloudHubAgent:
    enable: false

kubeAPIConfig:
  burst: 200
  contentType: "application/vnd.kubernetes.protobuf"
  kubeConfig: "/etc/kubeedge/kubeconfig.yaml"
  master: ""
  qps: 100

mesh:
  enable: false

metaserver:
  enable: false
Configuration Sections Explained
1. CloudHub Module
yaml
cloudHub:
  advertiseAddress:
  - 3.70.53.21      # Public IP for edge nodes to connect
  port: 10000       # Base port
  websocket:
    enable: true    # WebSocket protocol for communication
    port: 10004     # WebSocket specific port
Purpose: Manages connections from edge nodes. Acts as gateway.

2. CloudStream Module
yaml
cloudStream:
  enable: true
  streamPort: 10002  # For streaming data (logs, exec, port-forward)
Purpose: Handles data streaming between cloud and edge.

3. EdgeController Module
yaml
edgeController:
  nodeUpdateFrequency: 10  # How often to sync node status
  buffer:
    updatePodStatus: 1024  # Buffer size for pod status updates
Purpose: Syncs Kubernetes resources to edge nodes.

4. DeviceController Module
yaml
deviceController:
  enable: true
  buffer:
    updateDeviceStatus: 1024  # Buffer for device status
Purpose: Manages IoT devices at the edge.

📄 K3s Configuration: /etc/rancher/k3s/k3s.yaml
Sample Configuration
yaml
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
preferences: {}
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQo=
Location: /etc/rancher/k3s/k3s.yaml
Purpose: Kubernetes cluster configuration

📄 Service Configuration Files
1. CloudCore Service (/etc/systemd/system/cloudcore.service)
ini
[Unit]
Description=KubeEdge CloudCore
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudcore
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
2. K3s Service (/etc/systemd/system/k3s.service)
ini
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
After=network.target

[Service]
Type=notify
EnvironmentFile=/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s server

[Install]
WantedBy=multi-user.target
📁 Certificate Files
1. Root CA Certificate (/etc/kubeedge/ca/rootCA.crt)
pem
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL5M6PjD1QOMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
BAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRIwEAYDVQQKDAlLdWJlRWRnZTES
MBAGA1UEAwwJUm9vdCBDQSAxMB4XDTI0MDEwMTAwMDAwMFoXDTM0MDEwMTAwMDAw
MFowRTELMAkGA1UEBhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExEjAQBgNVBAoM
CUt1YmVFZGdlMRIwEAYDVQQDDAlSb290IENBIDEwggEiMA0GCSqGSIb3DQEBAQUA
A4IBDwAwggEKAoIBAQC5x5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6
PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5
M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAK
L5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOMAKL5M6PjD1QOM
-----END CERTIFICATE-----
2. CloudCore Certificate (/etc/kubeedge/certs/cloudcore.crt)
pem
-----BEGIN CERTIFICATE-----
MIIDYzCCAkugAwIBAgIIV5JZ5Z5Z5Z4wDQYJKoZIhvcNAQELBQAwRTELMAkGA1UE
BhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExEjAQBgNVBAoMCUt1YmVFZGdlMRIw
EAYDVQQDDAlSb290IENBIDEwHhcNMjQwMTAxMDAwMDAwWhcNMjUwMTAxMDAwMDAw
WjBPMQswCQYDVQQGEwJVUzETMBEGA1UECAwKQ2FsaWZvcm5pYTESMBAGA1UECgwJ
S3ViZUVkZ2UxGTAXBgNVBAMMEGNsb3VkY29yZS5sb2NhbDCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBALnHkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4w
Aovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4
wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA
4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA
4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA
4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA4wAovkzo+MPVA
-----END CERTIFICATE-----
📄 Deployment YAML Files
1. CloudCore Deployment (deployment.yaml)
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudcore
  namespace: kubeedge
  labels:
    app: cloudcore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudcore
  template:
    metadata:
      labels:
        app: cloudcore
    spec:
      serviceAccountName: cloudcore
      containers:
      - name: cloudcore
        image: kubeedge/cloudcore:v1.18.0
        imagePullPolicy: IfNotPresent
        command: ["./cloudcore"]
        args:
        - --config=/etc/kubeedge/config/cloudcore.yaml
        ports:
        - containerPort: 10000
          name: cloudhub
        - containerPort: 10002
          name: cloudstream
        - containerPort: 10004
          name: websocket
        volumeMounts:
        - name: config-volume
          mountPath: /etc/kubeedge/config
        - name: cert-volume
          mountPath: /etc/kubeedge/certs
        - name: ca-volume
          mountPath: /etc/kubeedge/ca
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: config-volume
        configMap:
          name: cloudcore-config
      - name: cert-volume
        secret:
          secretName: cloudcore-cert
      - name: ca-volume
        secret:
          secretName: cloudcore-ca
2. CloudCore Service (service.yaml)
yaml
apiVersion: v1
kind: Service
metadata:
  name: cloudcore
  namespace: kubeedge
spec:
  selector:
    app: cloudcore
  ports:
  - name: cloudhub
    port: 10000
    targetPort: 10000
    nodePort: 30374
  - name: cloudstream
    port: 10002
    targetPort: 10002
    nodePort: 31305
  - name: websocket
    port: 10004
    targetPort: 10004
    nodePort: 32203
  type: NodePort
3. CloudCore ConfigMap (configmap.yaml)
yaml
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
📄 Kubernetes RBAC Configuration
1. ServiceAccount (serviceaccount.yaml)
yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudcore
  namespace: kubeedge
2. ClusterRole (clusterrole.yaml)
yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudcore
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/status"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["devices.kubeedge.io"]
  resources: ["devices", "devicemodels"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
3. ClusterRoleBinding (clusterrolebinding.yaml)
yaml
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
📄 Master Application Configuration
1. Master App Deployment (master-deployment.yaml)
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: master-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: master-app
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
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
2. Master App Service (master-service.yaml)
yaml
apiVersion: v1
kind: Service
metadata:
  name: master-app
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
  selector:
    app: master-app
🔧 Environment Configuration
1. Environment Variables (env-configmap.yaml)
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: master-app-config
  namespace: default
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  MAX_DEVICES: "100"
  POLL_INTERVAL: "5"
  CLOUD_IP: "3.70.53.21"
  CLOUDHUB_PORT: "30374"
2. Secrets Configuration (secrets.yaml)
yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=          # admin
  password: cGFzc3dvcmQxMjM=  # password123
  host: MTI3LjAuMC4x          # 127.0.0.1
  port: NTQzMg==              # 5432
📊 Monitoring Configuration
1. Prometheus ServiceMonitor (servicemonitor.yaml)
yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cloudcore-monitor
  namespace: kubeedge
spec:
  selector:
    matchLabels:
      app: cloudcore
  endpoints:
  - port: cloudhub
    interval: 30s
    path: /metrics
  namespaceSelector:
    matchNames:
    - kubeedge
2. Grafana Dashboard Config (grafana-dashboard.yaml)
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-cloudcore
  namespace: monitoring
data:
  cloudcore-dashboard.json: |
    {
      "title": "KubeEdge CloudCore Dashboard",
      "panels": [
        {
          "title": "Active Edge Nodes",
          "targets": [{
            "expr": "kubeedge_cloudcore_active_devices",
            "legendFormat": "Devices"
          }]
        }
      ]
    }
🛠️ Troubleshooting Configuration
1. Debug ConfigMap (debug-config.yaml)
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: debug-config
  namespace: kubeedge
data:
  debug-cloudcore.sh: |
    #!/bin/bash
    echo "=== CloudCore Debug Information ==="
    echo "Date: $(date)"
    echo "Hostname: $(hostname)"
    echo "IP Address: $(hostname -I)"
    
    # Check CloudCore status
    kubectl get pods -n kubeedge
    
    # Check logs
    kubectl logs -n kubeedge deployment/cloudcore --tail=50
    
    # Check services
    kubectl get svc -n kubeedge cloudcore
    
    # Check node connections
    netstat -tulpn | grep -E '10000|10002|10004'
2. Health Check Endpoint (health-check.yaml)
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudcore-health-check
  namespace: kubeedge
data:
  health-check.py: |
    #!/usr/bin/env python3
    import requests
    import json
    
    def check_cloudcore_health():
        endpoints = [
            "http://localhost:10000/readyz",
            "http://localhost:10002/healthz",
            "http://localhost:10004/health"
        ]
        
        results = {}
        for endpoint in endpoints:
            try:
                response = requests.get(endpoint, timeout=5)
                results[endpoint] = {
                    "status": response.status_code,
                    "healthy": response.status_code == 200
                }
            except Exception as e:
                results[endpoint] = {
                    "status": "error",
                    "error": str(e)
                }
        
        return results
    
    if __name__ == "__main__":
        health = check_cloudcore_health()
        print(json.dumps(health, indent=2))
🔐 Security Configuration
1. Network Policies (network-policy.yaml)
yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloudcore-network-policy
  namespace: kubeedge
spec:
  podSelector:
    matchLabels:
      app: cloudcore
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: edgecore
    ports:
    - protocol: TCP
      port: 10000
    - protocol: TCP
      port: 10002
    - protocol: TCP
      port: 10004
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
2. Pod Security Standards (pod-security.yaml)
yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: cloudcore-psp
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - configMap
  - secret
  - emptyDir
📝 Configuration Validation
Validation Script (validate-config.sh)
bash
#!/bin/bash
# Validate CloudCore configuration

echo "Validating CloudCore configuration..."

# Check required files
required_files=(
    "/etc/kubeedge/config/cloudcore.yaml"
    "/etc/kubeedge/ca/rootCA.crt"
    "/etc/kubeedge/certs/cloudcore.crt"
    "/etc/kubeedge/certs/cloudcore.key"
)

for file in "${required_files[@]}"; do
    if [ -f "$file" ]; then
        echo "✓ $file exists"
    else
        echo "✗ $file missing"
        exit 1
    fi
done

# Validate YAML syntax
if command -v yamllint &> /dev/null; then
    yamllint /etc/kubeedge/config/cloudcore.yaml
fi

# Check service status
systemctl is-active --quiet cloudcore
if [ $? -eq 0 ]; then
    echo "✓ CloudCore service is running"
else
    echo "✗ CloudCore service is not running"
    exit 1
fi

echo "Configuration validation passed!"
📁 File Locations Summary
File	Location	Purpose
Main Config	/etc/kubeedge/config/cloudcore.yaml	Core configuration
Certificates	/etc/kubeedge/certs/	SSL/TLS certificates
CA Certificates	/etc/kubeedge/ca/	Certificate authority
K3s Config	/etc/rancher/k3s/k3s.yaml	Kubernetes config
Kubectl Config	~/.kube/config	User kubeconfig
Service File	/etc/systemd/system/cloudcore.service	Systemd service
Logs	/var/log/kubeedge/cloudcore.log	Application logs
🎯 Configuration Tips
Production Settings:

Set nodeLimit based on expected edge devices

Enable TLS for all communications

Configure proper resource limits

Performance Tuning:

Adjust buffer sizes based on device count

Tune keepAliveInterval for network conditions

Set appropriate timeouts

Security Best Practices:

Rotate certificates regularly

Use network policies to restrict access

Enable audit logging

Monitoring:

Configure Prometheus scraping

Set up alert rules

Monitor certificate expiry
