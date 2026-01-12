# LiveKit Helm Charts

Helm charts cho việc deploy **LiveKit Server**, **Egress**, và **Ingress** trên Kubernetes. LiveKit là nền tảng real-time mã nguồn mở để streaming audio, video và data.

## 📋 Tổng quan

Repository này bao gồm 3 Helm charts chính:

- **livekit-server**: Core LiveKit server cho real-time communication
- **egress**: Export/recording LiveKit rooms sang file hoặc streaming
- **ingress**: Nhận RTMP/WHIP streams vào LiveKit rooms

## 🚀 Cài đặt nhanh

### Bước 1: Thêm Helm repository

```bash
helm repo add livekit https://helm.livekit.io
helm repo update
```

### Bước 2: Tạo file values tùy chỉnh

```bash
# Chọn một trong các file sample phù hợp:
cp server-sample.yaml my-values.yaml
# Hoặc
cp examples/server-gke.yaml my-values.yaml  # Cho GKE
cp examples/server-eks.yaml my-values.yaml  # Cho EKS
cp examples/server-do.yaml my-values.yaml   # Cho DigitalOcean
```

### Bước 3: Cấu hình các giá trị bắt buộc

Chỉnh sửa `my-values.yaml` và cập nhật:

```yaml
livekit:
  redis:
    address: "your-redis-host:6379"  # Redis bắt buộc cho production
  keys:
    your-api-key: "your-api-secret"  # Generate từ https://livekit.io/cloud/projects
  turn:
    enabled: true
    domain: "turn.yourdomain.com"
    secretName: "turn-tls-secret"    # Kubernetes secret chứa TLS cert

loadBalancer:
  type: gke  # hoặc alb (AWS), do (DigitalOcean)
  tls:
    - hosts:
        - "livekit.yourdomain.com"
      secretName: "livekit-tls-secret"
```

### Bước 4: Deploy LiveKit Server

```bash
# Tạo namespace
kubectl create namespace livekit

# Deploy
helm install livekit-server livekit/livekit-server \
  --namespace livekit \
  --values my-values.yaml

# Kiểm tra status
kubectl get pods -n livekit
kubectl get svc -n livekit
```

## 📦 Chi tiết các Charts

### LiveKit Server

Core server cho real-time communication với WebRTC.

**Tính năng:**
- WebRTC signaling và media routing
- Horizontal autoscaling với HPA
- TURN server tích hợp
- Redis cho multi-node clustering
- Support nhiều cloud providers (GKE, EKS, DigitalOcean)

**Cài đặt:**

```bash
helm install livekit-server livekit/livekit-server \
  --namespace livekit \
  --values server-values.yaml
```

**File cấu hình mẫu:** `server-sample.yaml`, `examples/server-*.yaml`

### Egress

Export và recording LiveKit rooms.

**Tính năng:**
- Recording room sang file (MP4, WebM)
- Export sang S3, GCS, Azure Storage
- Stream sang RTMP endpoints
- Track compositing và layout templates

**Cài đặt:**

```bash
helm install livekit-egress livekit/egress \
  --namespace livekit \
  --values egress-values.yaml
```

**File cấu hình mẫu:** `egress-sample.yaml`

**Cấu hình tối thiểu:**

```yaml
egress:
  api_key: "server-api-key"
  api_secret: "server-api-secret"
  ws_url: "ws://livekit-server:7880"
  redis:
    address: "redis-host:6379"
  s3:
    access_key: "access_key"
    secret: "secret"
    region: "us-west-2"
    bucket: "my-egress-bucket"
```

### Ingress

Nhận external streams vào LiveKit.

**Tính năng:**
- RTMP ingress (OBS, streaming software)
- WHIP ingress (WebRTC-based)
- HTTP relay cho low-latency streaming
- Auto-scaling dựa trên số streams

**Cài đặt:**

```bash
helm install livekit-ingress livekit/ingress \
  --namespace livekit \
  --values ingress-values.yaml
```

**File cấu hình mẫu:** `ingress-sample.yaml`

**Cấu hình tối thiểu:**

```yaml
ingress:
  api_key: "server-api-key"
  api_secret: "server-api-secret"
  ws_url: "ws://livekit-server:7880"
  redis:
    address: "redis-host:6379"
  rtmp_port: 1935
  whip_port: 8080
```

## 🔧 Cấu hình Production

### 1. Redis Cluster (Bắt buộc)

Redis cần thiết cho multi-node deployment và clustering.

```yaml
livekit:
  redis:
    address: "redis-cluster.default.svc.cluster.local:6379"
    db: 0
    username: "redis-user"
    password: "redis-password"
    use_tls: true
```

### 2. API Keys và Secrets

Generate API key/secret từ [LiveKit Cloud](https://livekit.io/cloud/projects) hoặc tự tạo:

```bash
# Tạo API key/secret
openssl rand -hex 32  # API key
openssl rand -base64 32  # API secret
```

Có 2 cách lưu trữ API keys:

**Option 1: Trong ConfigMap (mặc định)**
```yaml
livekit:
  keys:
    APIabcdefg: "secretXYZ123"
```

**Option 2: Trong Secret (khuyến nghị cho production)**
```yaml
storeKeysInSecret:
  enabled: true
  keys:
    APIabcdefg: "secretXYZ123"
```

### 3. TLS/SSL Certificates

**Import TLS certificate vào Kubernetes:**

```bash
# Từ cert files
kubectl create secret tls livekit-tls-secret \
  --cert=livekit.crt \
  --key=livekit.key \
  --namespace livekit

# Từ Let's Encrypt (với cert-manager)
kubectl create secret tls turn-tls-secret \
  --cert=turn.crt \
  --key=turn.key \
  --namespace livekit
```

### 4. Load Balancer Configuration

**GKE (Google Kubernetes Engine):**
```yaml
loadBalancer:
  type: gke
  tls:
    - hosts:
        - livekit.yourdomain.com
      secretName: livekit-tls-secret
```

**EKS (AWS Elastic Kubernetes Service):**
```yaml
loadBalancer:
  type: alb
  tls:
    - hosts:
        - livekit.yourdomain.com
  # Cert phải tồn tại trong AWS Certificate Manager (ACM)
```

**DigitalOcean:**
```yaml
loadBalancer:
  type: do
  clusterIssuer: letsencrypt-prod
  tls:
    - hosts:
        - livekit.yourdomain.com
      secretName: livekit-tls-secret
```

### 5. Resources và Autoscaling

**Recommended resources cho production:**

```yaml
resources:
  requests:
    cpu: 4000m
    memory: 1024Mi
  limits:
    cpu: 7500m
    memory: 2048Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
```

**Lưu ý:** Chỉ chạy 1 LiveKit pod trên mỗi node vật lý do port restrictions.

### 6. Node Selection và Affinity

Isolate LiveKit pods trên các nodes riêng:

```yaml
nodeSelector:
  node.kubernetes.io/instance-type: c5.4xlarge  # AWS
  # cloud.google.com/machine-family: c2  # GKE

# Anti-affinity để spread pods
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - livekit-server
        topologyKey: kubernetes.io/hostname
```

### 7. Monitoring và Metrics

Enable Prometheus metrics:

```yaml
livekit:
  prometheus_port: 6789

serviceMonitor:
  create: true
  annotations:
    prometheus.io/scrape: "true"
```

### 8. Graceful Shutdown

Set termination grace period phù hợp:

```yaml
# LiveKit Server - 5 hours
terminationGracePeriodSeconds: 18000

# Egress - 1 hour
terminationGracePeriodSeconds: 3600

# Ingress - 3 hours
terminationGracePeriodSeconds: 10800
```

## 🌐 Network và Firewall

### Ports cần mở

**LiveKit Server:**
- `7880/TCP` - HTTP API và WebSocket
- `7881/TCP` - RTC over TCP
- `50000-60000/UDP` - RTC media (WebRTC)
- `3478/TCP` - TURN/TLS
- `3478/UDP` - TURN/UDP (recommended: port 443)

**Egress:**
- `8080/TCP` - Health check

**Ingress:**
- `7888/TCP` - Health check
- `1935/TCP` - RTMP
- `8080/TCP` - WHIP
- `7885/UDP` - RTC media
- `9090/TCP` - HTTP relay

### Firewall Rules (GCP example)

```bash
# RTC UDP ports
gcloud compute firewall-rules create livekit-rtc-udp \
  --allow=udp:50000-60000 \
  --target-tags=livekit-node

# TURN
gcloud compute firewall-rules create livekit-turn \
  --allow=tcp:3478,udp:443 \
  --target-tags=livekit-node
```

## 🔍 Troubleshooting

### Kiểm tra pods status

```bash
kubectl get pods -n livekit
kubectl describe pod <pod-name> -n livekit
kubectl logs <pod-name> -n livekit
```

### Kiểm tra services

```bash
kubectl get svc -n livekit
kubectl describe svc livekit-server -n livekit
```

### Kiểm tra endpoints

```bash
kubectl get endpoints -n livekit
```

### Test kết nối

```bash
# Port forward để test local
kubectl port-forward svc/livekit-server 7880:80 -n livekit

# Test API
curl http://localhost:7880/
```

### Common Issues

**1. Pods không start được:**
- Kiểm tra image pull: `kubectl describe pod <pod-name> -n livekit`
- Check resources: Đảm bảo cluster có đủ resources

**2. External IP pending:**
- Đợi cloud provider provision load balancer (có thể mất 5-10 phút)
- Check cloud provider quotas

**3. Không kết nối được WebRTC:**
- Verify firewall rules cho UDP ports 50000-60000
- Check `use_external_ip: true` trong config
- Verify TURN server configuration

**4. TURN không hoạt động:**
- Check TLS certificate: `kubectl get secret turn-tls-secret -n livekit`
- Verify domain matches certificate
- Test TURN connectivity

## 📝 Ví dụ Complete Deployment

### GKE Production Setup

```yaml
# production-gke.yaml
replicaCount: 3

terminationGracePeriodSeconds: 18000

livekit:
  log_level: info
  prometheus_port: 6789
  rtc:
    use_external_ip: true
    port_range_start: 50000
    port_range_end: 60000
    tcp_port: 7881
  redis:
    address: "redis-ha-master.default.svc.cluster.local:6379"
    password: "your-redis-password"
    use_tls: true
  keys:
    APIabcdefghijk: "your-secret-key-here"
  turn:
    enabled: true
    domain: "turn.yourdomain.com"
    tls_port: 3478
    udp_port: 443
    secretName: "turn-tls-secret"

loadBalancer:
  type: gke
  tls:
    - hosts:
        - "livekit.yourdomain.com"
      secretName: "livekit-tls-secret"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60

resources:
  requests:
    cpu: 4000m
    memory: 1024Mi
  limits:
    cpu: 7500m
    memory: 2048Mi

nodeSelector:
  cloud.google.com/machine-family: c2

serviceMonitor:
  create: true
```

**Deploy:**

```bash
# Tạo secrets
kubectl create secret tls livekit-tls-secret \
  --cert=livekit.crt --key=livekit.key -n livekit

kubectl create secret tls turn-tls-secret \
  --cert=turn.crt --key=turn.key -n livekit

# Deploy
helm install livekit-prod livekit/livekit-server \
  --namespace livekit \
  --values production-gke.yaml

# Deploy egress
helm install livekit-egress livekit/egress \
  --namespace livekit \
  --values egress-values.yaml

# Deploy ingress
helm install livekit-ingress livekit/ingress \
  --namespace livekit \
  --values ingress-values.yaml
```

## 🔄 Update và Upgrade

```bash
# Update helm repo
helm repo update

# Upgrade installation
helm upgrade livekit-server livekit/livekit-server \
  --namespace livekit \
  --values my-values.yaml

# Rollback nếu cần
helm rollback livekit-server -n livekit
```

## 🗑️ Uninstall

```bash
helm uninstall livekit-server -n livekit
helm uninstall livekit-egress -n livekit
helm uninstall livekit-ingress -n livekit

# Xóa namespace (cẩn thận!)
kubectl delete namespace livekit
```

## 📚 Resources

- **Documentation:** https://docs.livekit.io/
- **LiveKit Cloud:** https://livekit.io/cloud
- **GitHub:** https://github.com/livekit/livekit
- **Discord Community:** https://livekit.io/discord

## 🛠️ Development (For Chart Maintainers)

### Publishing charts

Requires helm-s3 plugin:

```bash
helm plugin install https://github.com/hypnoglow/helm-s3.git
AWS_REGION=us-east-1 helm repo add livekit s3://livekit-helm

# Deploy server chart
./deploy.sh

# Deploy egress chart
./deploy-egress.sh

# Deploy ingress chart
./deploy-ingress.sh
```

## 🤝 Contributing

Contributions welcome!
