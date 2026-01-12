# 🚀 Quick Start - Deploy LiveKit trên K3s Self-Host

Hướng dẫn deploy LiveKit Server nhanh nhất trên server với K3s đã cài sẵn.

---

## ⚡ Bước 1: Chuẩn bị môi trường (1 phút)

```bash
# SSH vào server
ssh user@YOUR_SERVER_IP

# Clone repo về
git clone <repo-url> /opt/services/livekit-helm
cd /opt/services/livekit-helm

# Tạo namespace
kubectl create namespace livekit
```

## ⚡ Bước 2: Tạo file .env (1 phút)

```bash
# Generate API keys
API_KEY=$(openssl rand -hex 16)
API_SECRET=$(openssl rand -base64 32)

# Tạo file .env
cat > .env <<EOF
# LiveKit Configuration
LIVEKIT_NAMESPACE=livekit
SERVER_IP=$(curl -s ifconfig.me)

# API Keys
API_KEY=$API_KEY
API_SECRET=$API_SECRET

# Redis
REDIS_PASSWORD=Jipom321@
REDIS_ADDRESS=redis-master.livekit.svc.cluster.local:6379

# Ports
HTTP_NODE_PORT=30080
RTC_TCP_PORT=7881
RTC_UDP_START=50000
RTC_UDP_END=50100

# Resources
SERVER_CPU_REQUEST=500m
SERVER_MEM_REQUEST=512Mi
SERVER_CPU_LIMIT=2000m
SERVER_MEM_LIMIT=2Gi

# Replicas
SERVER_REPLICAS=1

# Log level
LOG_LEVEL=info
EOF

# Xem thông tin
echo "========================================="
echo "API Key: $API_KEY"
echo "API Secret: $API_SECRET"
echo "Server IP: $(curl -s ifconfig.me)"
echo "========================================="
```

## ⚡ Bước 3: Tạo scripts deploy (1 phút)

```bash
# Script deploy server
cat > deploy-selfhost.sh <<'SCRIPT'
#!/bin/bash
set -e

if [ -f .env ]; then
    export $(cat .env | grep -v '^#' | xargs)
else
    echo "Error: .env file not found!"
    exit 1
fi

echo "🚀 Deploying LiveKit Server..."
echo "Server IP: $SERVER_IP"

kubectl create namespace $LIVEKIT_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Install Redis
if ! helm list -n $LIVEKIT_NAMESPACE | grep -q redis; then
    echo "📦 Installing Redis..."
    helm repo add bitnami https://charts.bitnami.com/bitnami 2>/dev/null || true
    helm repo update
    
    helm install redis bitnami/redis \
        --namespace $LIVEKIT_NAMESPACE \
        --set auth.password="$REDIS_PASSWORD" \
        --set master.persistence.enabled=false
    
    echo "⏳ Waiting for Redis..."
    kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=redis -n $LIVEKIT_NAMESPACE --timeout=180s
fi

# Generate values
mkdir -p build
cat > build/server-values.yaml <<YAML
replicaCount: $SERVER_REPLICAS

livekit:
  keys:
    $API_KEY: "$API_SECRET"
  
  redis:
    address: $REDIS_ADDRESS
    password: "$REDIS_PASSWORD"
  
  rtc:
    use_external_ip: true
    port_range_start: $RTC_UDP_START
    port_range_end: $RTC_UDP_END
    tcp_port: $RTC_TCP_PORT
  
  log_level: $LOG_LEVEL

loadBalancer:
  type: disable

podHostNetwork: true
terminationGracePeriodSeconds: 18000

resources:
  requests:
    cpu: $SERVER_CPU_REQUEST
    memory: $SERVER_MEM_REQUEST
  limits:
    cpu: $SERVER_CPU_LIMIT
    memory: $SERVER_MEM_LIMIT
YAML

# Deploy LiveKit
echo "🎯 Deploying LiveKit Server..."
helm upgrade --install livekit ./livekit-server \
    --namespace $LIVEKIT_NAMESPACE \
    --values build/server-values.yaml

echo "⏳ Waiting for LiveKit..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=livekit-server -n $LIVEKIT_NAMESPACE --timeout=300s

echo ""
echo "✅ LiveKit Server deployed!"
echo "📊 Connection Info:"
echo "   URL: ws://$SERVER_IP:7880"
echo "   API Key: $API_KEY"
echo "   API Secret: $API_SECRET"
echo ""
echo "🔥 Open firewall:"
echo "   sudo ufw allow $RTC_TCP_PORT/tcp"
echo "   sudo ufw allow $RTC_UDP_START:$RTC_UDP_END/udp"
SCRIPT

chmod +x deploy-selfhost.sh

# Script check status
cat > check-status.sh <<'SCRIPT'
#!/bin/bash

if [ -f .env ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

echo "🔍 LiveKit Status"
echo "======================="
echo ""
echo "📦 Pods:"
kubectl get pods -n $LIVEKIT_NAMESPACE
echo ""
echo "🌐 Services:"
kubectl get svc -n $LIVEKIT_NAMESPACE
echo ""
echo "📊 Connection Info:"
echo "   URL: ws://$SERVER_IP:7880"
echo "   API Key: $API_KEY"
echo ""
SCRIPT

chmod +x check-status.sh

# Script cleanup
cat > cleanup.sh <<'SCRIPT'
#!/bin/bash

if [ -f .env ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

echo "🗑️  Cleaning up LiveKit..."

helm uninstall livekit -n $LIVEKIT_NAMESPACE 2>/dev/null || true
helm uninstall redis -n $LIVEKIT_NAMESPACE 2>/dev/null || true

kubectl delete namespace $LIVEKIT_NAMESPACE

echo "✅ Cleanup done!"
SCRIPT

chmod +x cleanup.sh

echo "✅ Scripts created!"
```

## ⚡ Bước 4: Deploy (2 phút)

```bash
# Deploy server
./deploy-selfhost.sh

# Kiểm tra status
./check-status.sh
```

## ⚡ Bước 5: Mở firewall (30 giây)

```bash
# Mở ports
source .env
sudo ufw allow $RTC_TCP_PORT/tcp
sudo ufw allow $RTC_UDP_START:$RTC_UDP_END/udp
sudo ufw reload

# Nếu dùng AWS EC2 - mở Security Group:
# - TCP 7881
# - UDP 50000-50100
```

## ⚡ Bước 6: Test (30 giây)

```bash
source .env

# Test từ server
kubectl get pods -n livekit

# Test connection
kubectl run test --rm -it --image=curlimages/curl --restart=Never -n livekit -- \
  curl -v http://livekit-livekit-server.livekit.svc.cluster.local

# Lấy thông tin kết nối
echo "========================================="
echo "LiveKit URL: ws://$SERVER_IP:7880"
echo "API Key: $API_KEY"
echo "API Secret: $API_SECRET"
echo "========================================="
```

---

## 📝 Tóm tắt lệnh

```bash
# Setup
cd /opt/services/livekit-helm
# Tạo .env (copy từ Bước 2)
# Tạo scripts (copy từ Bước 3)

# Deploy
./deploy-selfhost.sh

# Check
./check-status.sh

# Test
kubectl get pods -n livekit
source .env && echo "URL: ws://$SERVER_IP:7880"

# Cleanup (nếu cần)
./cleanup.sh
```

---

## 🎯 Kết quả mong đợi

Sau khi hoàn thành, bạn sẽ có:

- ✅ LiveKit Server đang chạy trên K3s
- ✅ Redis cluster cho backend
- ✅ Pods status: Running
- ✅ Connection URL: `ws://YOUR_IP:7880`
- ✅ API Key/Secret để authenticate

---

## 🔧 Troubleshooting

### Pod không start

```bash
kubectl describe pod -n livekit -l app.kubernetes.io/name=livekit-server
kubectl logs -n livekit -l app.kubernetes.io/name=livekit-server
```

### Redis lỗi

```bash
kubectl logs -n livekit -l app.kubernetes.io/name=redis
helm uninstall redis -n livekit
./deploy-selfhost.sh
```

### Không kết nối được

```bash
# Check firewall
sudo ufw status

# Check service
kubectl get svc -n livekit

# Check pods
kubectl get pods -n livekit
```

---

## 📚 Next Steps

Sau khi deploy xong LiveKit Server, có thể:

1. **Deploy Egress** (recording): Xem file `egress-sample.yaml`
2. **Deploy Ingress** (RTMP streaming): Xem file `ingress-sample.yaml`
3. **Setup SSL/TLS**: Cài cert-manager và configure domain
4. **Enable monitoring**: Setup Prometheus metrics
5. **Scale up**: Enable autoscaling với HPA

Chi tiết xem file [README.md](README.md)

---

## ⏱️ Tổng thời gian: ~5 phút

- Bước 1-3: Setup (3 phút)
- Bước 4: Deploy (2 phút)
- Bước 5-6: Test (1 phút)

**Xong! LiveKit đã sẵn sàng sử dụng. 🎉**
