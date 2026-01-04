# Hướng dẫn Triển khai Project: Service Mesh & DevSecOps

## Tổng quan

Hướng dẫn này chia thành 2 phần chính:
- **Phần 1: Service Mesh** - Cấu hình Istio với mTLS, Authorization Policy, và Retry Policy
- **Phần 2: DevSecOps** - Tích hợp SonarQube, Snyk, OWASP ZAP, và GitLeaks vào CI/CD

---

# PHẦN 1: SERVICE MESH

## Mục tiêu
- Deploy spring-petclinic-microservices lên Kubernetes
- Cấu hình Istio Service Mesh với mTLS
- Thiết lập Authorization Policy
- Cấu hình Retry Policy
- Visualize topology với Kiali

---

## Bước 1: Chuẩn bị Kubernetes Cluster và Istio

### 1.1. Kiểm tra cluster đang chạy
```bash
kubectl cluster-info
kubectl get nodes
```

### 1.2. Cài đặt Istio (nếu chưa có)
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio với profile default
istioctl install --set values.defaultRevision=default -y

# Verify installation
kubectl get pods -n istio-system
```

### 1.3. Enable Istio injection cho namespace
```bash
# Tạo namespace cho petclinic
kubectl create namespace petclinic

# Enable automatic sidecar injection
kubectl label namespace petclinic istio-injection=enabled

# Verify
kubectl get namespace -L istio-injection
```

### 1.4. Cài đặt Kiali và các addons

**Cách 1: Sử dụng istioctl (Khuyến nghị)**
```bash
# Cài đặt addons bằng istioctl
istioctl install --set values.defaultRevision=default --set addonComponents.kiali.enabled=true --set addonComponents.prometheus.enabled=true --set addonComponents.grafana.enabled=true -y
```

**Cách 2: Sử dụng file YAML từ thư mục Istio**
```bash
# Vào thư mục Istio
cd ~/istio-1.28.2

# Kiểm tra xem có thư mục samples/addons không
ls -la samples/addons/

# Nếu có, bạn có thể cài đặt addons bằng:
kubectl apply -f samples/addons/kiali.yaml
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml

# Nếu thư mục không tồn tại, có thể download lại Istio:
# curl -L https://istio.io/downloadIstio | sh -
# cd istio-*
# kubectl apply -f samples/addons/kiali.yaml
# kubectl apply -f samples/addons/prometheus.yaml
# kubectl apply -f samples/addons/grafana.yaml
```

**Cách 3: Kiểm tra xem đã cài chưa (nếu pods đã chạy)**
```bash
# Kiểm tra pods addons
kubectl get pods -n istio-system | grep -E "kiali|prometheus|grafana"

# Nếu đã có pods đang chạy, có thể bỏ qua bước này
```

**Sau khi cài đặt (hoặc nếu đã có sẵn):**
```bash
# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=kiali -n istio-system --timeout=300s

# Get Kiali access credentials
# Cách 1: Thử lấy từ secret (nếu có)
kubectl get secret -n istio-system kiali -o jsonpath='{.data.username}' 2>/dev/null | base64 -d && echo
kubectl get secret -n istio-system kiali -o jsonpath='{.data.passphrase}' 2>/dev/null | base64 -d && echo

# Cách 2: Kiểm tra ConfigMap của Kiali để xem authentication strategy
kubectl get configmap -n istio-system kiali -o yaml | grep -A 5 "auth:"

# Nếu thấy "strategy: anonymous" → KHÔNG CẦN username/password, truy cập trực tiếp
# Nếu thấy "strategy: login" → Cần username/password (thử admin/admin)
# Nếu thấy "strategy: token" → Cần token để đăng nhập

# Kiểm tra tất cả secrets trong istio-system
kubectl get secrets -n istio-system | grep kiali
```

### 1.5. Expose Kiali (Port Forward)
```bash
# Port forward Kiali service để truy cập từ localhost hoặc máy khác trong mạng
# --address 0.0.0.0 cho phép truy cập từ bất kỳ địa chỉ IP nào (không chỉ localhost)
kubectl port-forward --address 0.0.0.0 -n istio-system svc/kiali 20001:20001

# Sau khi chạy lệnh trên, truy cập Kiali tại: http://localhost:20001
# Hoặc từ máy khác trong cùng mạng: http://<IP-của-máy-này>:20001

# Lưu ý về Authentication:
# - Kiểm tra authentication strategy bằng lệnh ở Bước 1.4
# - Nếu strategy là "anonymous": Không cần đăng nhập, truy cập trực tiếp
# - Nếu strategy là "login": Đăng nhập với username/password (mặc định: admin/admin)
# - Nếu strategy là "token": Cần token để đăng nhập (lấy từ secret hoặc config)
```

---

## Bước 2: Build và Deploy Spring PetClinic Microservices

### 2.1. Clone và build project
```bash
cd ~
git clone https://github.com/spring-petclinic/spring-petclinic-microservices.git
cd spring-petclinic-microservices

# Build project
./mvnw clean install -DskipTests
```

### 2.2. Build Docker images
```bash
# Build all images
./mvnw clean install -P buildDocker

# Tag images (nếu cần push lên registry)
# docker tag springcommunity/spring-petclinic-config-server:latest <your-registry>/spring-petclinic-config-server:latest
```

### 2.3. Tạo Kubernetes manifests

Tạo thư mục cho manifests:
```bash
mkdir -p k8s-manifests
cd k8s-manifests
```

#### 2.3.1. Config Server Deployment
Tạo file `config-server-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-server
  template:
    metadata:
      labels:
        app: config-server
    spec:
      containers:
      - name: config-server
        image: springcommunity/spring-petclinic-config-server:latest
        ports:
        - containerPort: 8888
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: config-server
  namespace: petclinic
spec:
  selector:
    app: config-server
  ports:
  - port: 8888
    targetPort: 8888
```

#### 2.3.2. Discovery Server (Eureka) Deployment
Tạo file `discovery-server-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discovery-server
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: discovery-server
  template:
    metadata:
      labels:
        app: discovery-server
    spec:
      containers:
      - name: discovery-server
        image: springcommunity/spring-petclinic-discovery-server:latest
        ports:
        - containerPort: 8761
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: discovery-server
  namespace: petclinic
spec:
  selector:
    app: discovery-server
  ports:
  - port: 8761
    targetPort: 8761
```

#### 2.3.3. Customers Service Deployment
Tạo file `customers-service-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customers-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: customers-service
  template:
    metadata:
      labels:
        app: customers-service
    spec:
      containers:
      - name: customers-service
        image: springcommunity/spring-petclinic-customers-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: customers-service
  namespace: petclinic
spec:
  selector:
    app: customers-service
  ports:
  - port: 8081
    targetPort: 8081
```

#### 2.3.4. Vets Service Deployment
Tạo file `vets-service-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vets-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vets-service
  template:
    metadata:
      labels:
        app: vets-service
    spec:
      containers:
      - name: vets-service
        image: springcommunity/spring-petclinic-vets-service:latest
        ports:
        - containerPort: 8083
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: vets-service
  namespace: petclinic
spec:
  selector:
    app: vets-service
  ports:
  - port: 8083
    targetPort: 8083
```

#### 2.3.5. Visits Service Deployment
Tạo file `visits-service-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: visits-service
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: visits-service
  template:
    metadata:
      labels:
        app: visits-service
    spec:
      containers:
      - name: visits-service
        image: springcommunity/spring-petclinic-visits-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: visits-service
  namespace: petclinic
spec:
  selector:
    app: visits-service
  ports:
  - port: 8082
    targetPort: 8082
```

#### 2.3.6. API Gateway Deployment
Tạo file `api-gateway-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: springcommunity/spring-petclinic-api-gateway:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "native"
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: petclinic
  type: LoadBalancer  # Hoặc NodePort nếu không có LoadBalancer
spec:
  selector:
    app: api-gateway
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080  # Nếu dùng NodePort
```

### 2.4. Deploy tất cả services
```bash
# Deploy theo thứ tự
kubectl apply -f config-server-deployment.yaml
sleep 10

kubectl apply -f discovery-server-deployment.yaml
sleep 10

kubectl apply -f customers-service-deployment.yaml
kubectl apply -f vets-service-deployment.yaml
kubectl apply -f visits-service-deployment.yaml
sleep 10

kubectl apply -f api-gateway-deployment.yaml

# Kiểm tra pods
kubectl get pods -n petclinic

# Kiểm tra services
kubectl get svc -n petclinic
```

### 2.5. Verify deployment
```bash
# Check pods có sidecar (istio-proxy)
kubectl get pods -n petclinic
# Mỗi pod phải có 2 containers: app container và istio-proxy

# Check logs
kubectl logs -n petclinic -l app=api-gateway -c istio-proxy
```

---

## Bước 3: Cấu hình mTLS (Mutual TLS)

### 3.1. Tạo PeerAuthentication Policy (Enable mTLS cho toàn namespace)
Tạo file `mtls-peer-authentication.yaml`:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: petclinic
spec:
  mtls:
    mode: STRICT  # Bắt buộc mTLS cho tất cả traffic
```

### 3.2. Tạo DestinationRule (Định nghĩa mTLS cho destinations)
Tạo file `mtls-destination-rule.yaml`:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: default
  namespace: petclinic
spec:
  host: "*.petclinic.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### 3.3. Apply mTLS configuration
```bash
kubectl apply -f mtls-peer-authentication.yaml
kubectl apply -f mtls-destination-rule.yaml

# Verify
kubectl get peerauthentication -n petclinic
kubectl get destinationrule -n petclinic
```

### 3.4. Test mTLS
```bash
# Get a pod name
POD_NAME=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')

# Test connection với mTLS (sẽ thành công)
kubectl exec -n petclinic $POD_NAME -c istio-proxy -- curl -v http://vets-service.petclinic.svc.cluster.local:8083/vets

# Kiểm tra TLS trong logs
kubectl logs -n petclinic $POD_NAME -c istio-proxy | grep -i tls
```

---

## Bước 4: Cấu hình Authorization Policy

### 4.1. Tạo Authorization Policy (Chỉ cho phép một số service giao tiếp)
Tạo file `authorization-policy.yaml`:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: customers-service-policy
  namespace: petclinic
spec:
  selector:
    matchLabels:
      app: customers-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/petclinic/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET", "POST"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: vets-service-policy
  namespace: petclinic
spec:
  selector:
    matchLabels:
      app: vets-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/petclinic/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: visits-service-policy
  namespace: petclinic
spec:
  selector:
    matchLabels:
      app: visits-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/petclinic/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET", "POST"]
---
# Deny all other traffic by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: petclinic
spec:
  action: DENY
  rules:
  - {}
```

### 4.2. Apply Authorization Policies
```bash
kubectl apply -f authorization-policy.yaml

# Verify
kubectl get authorizationpolicy -n petclinic
```

### 4.3. Test Authorization Policy
```bash
# Test 1: API Gateway -> Customers Service (should succeed)
GATEWAY_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $GATEWAY_POD -c istio-proxy -- curl -v http://customers-service.petclinic.svc.cluster.local:8081/customers

# Test 2: Customers Service -> Vets Service (should fail - not allowed)
CUSTOMERS_POD=$(kubectl get pod -n petclinic -l app=customers-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $CUSTOMERS_POD -c istio-proxy -- curl -v http://vets-service.petclinic.svc.cluster.local:8083/vets
# Expected: 403 Forbidden hoặc connection refused

# Test 3: Direct access từ pod khác (should fail)
kubectl run test-pod --image=curlimages/curl -n petclinic --rm -it --restart=Never -- curl -v http://customers-service.petclinic.svc.cluster.local:8081/customers
# Expected: 403 Forbidden
```

---

## Bước 5: Cấu hình Retry Policy

### 5.1. Tạo VirtualService với Retry Policy
Tạo file `retry-policy.yaml`:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: customers-service-retry
  namespace: petclinic
spec:
  hosts:
  - customers-service
  http:
  - match:
    - headers:
        x-retry:
          exact: "true"
    route:
    - destination:
        host: customers-service
      weight: 100
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure,refused-stream
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vets-service-retry
  namespace: petclinic
spec:
  hosts:
  - vets-service
  http:
  - route:
    - destination:
        host: vets-service
      weight: 100
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: visits-service-retry
  namespace: petclinic
spec:
  hosts:
  - visits-service
  http:
  - route:
    - destination:
        host: visits-service
      weight: 100
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
```

### 5.2. Apply Retry Policies
```bash
kubectl apply -f retry-policy.yaml

# Verify
kubectl get virtualservice -n petclinic
```

### 5.3. Test Retry Policy (Tạo service trả về 500 để test)
Tạo file `test-error-service.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-error-service
  namespace: petclinic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-error-service
  template:
    metadata:
      labels:
        app: test-error-service
    spec:
      containers:
      - name: test-error
        image: nginx:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo 'server {
            listen 8080;
            location / {
              return 500;
            }
          }' > /etc/nginx/conf.d/default.conf
          nginx -g "daemon off;"
---
apiVersion: v1
kind: Service
metadata:
  name: test-error-service
  namespace: petclinic
spec:
  selector:
    app: test-error-service
  ports:
  - port: 8080
    targetPort: 8080
```

Tạo VirtualService cho test service:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: test-error-service-retry
  namespace: petclinic
spec:
  hosts:
  - test-error-service
  http:
  - route:
    - destination:
        host: test-error-service
      weight: 100
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
```

```bash
kubectl apply -f test-error-service.yaml
kubectl apply -f test-error-service-retry.yaml

# Test retry
TEST_POD=$(kubectl get pod -n petclinic -l app=api-gateway -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n petclinic $TEST_POD -c istio-proxy -- curl -v http://test-error-service.petclinic.svc.cluster.local:8080/

# Check logs để thấy retry attempts
kubectl logs -n petclinic $TEST_POD -c istio-proxy | grep -i retry
```

---

## Bước 6: Visualize với Kiali

### 6.1. Access Kiali
```bash
# Port forward Kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001

# Truy cập: http://localhost:20001
# Login với username/password từ Bước 1.4
```

### 6.2. Xem Service Graph
1. Vào Kiali UI
2. Chọn namespace `petclinic`
3. Vào tab **Graph**
4. Chọn **Service Graph** view
5. Screenshot topology và ghi chú flow:
   - API Gateway → Customers Service
   - API Gateway → Vets Service
   - API Gateway → Visits Service
   - Config Server và Discovery Server

### 6.3. Xem mTLS status
1. Trong Kiali Graph, chọn **Security** badge
2. Kiểm tra các service có mTLS enabled (màu xanh)
3. Screenshot

### 6.4. Xem Metrics và Traces
1. Click vào từng service trong graph
2. Xem metrics: Request rate, Error rate, Duration
3. Xem traces nếu có Jaeger/Zipkin

---

## Bước 7: Tạo Test Plan và Documentation

### 7.1. Test Plan Template
Tạo file `TEST-PLAN-SERVICE-MESH.md`:

```markdown
# Test Plan - Service Mesh

## Test Case 1: mTLS Verification
- **Mục tiêu**: Verify mTLS được enable và hoạt động
- **Steps**:
  1. Check PeerAuthentication policy
  2. Check DestinationRule
  3. Test connection và verify TLS trong logs
- **Expected**: Tất cả traffic giữa services đều dùng mTLS
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]

## Test Case 2: Authorization Policy
- **Mục tiêu**: Verify chỉ API Gateway có thể gọi các services
- **Steps**:
  1. API Gateway -> Customers Service (should succeed)
  2. Customers Service -> Vets Service (should fail)
  3. External pod -> Customers Service (should fail)
- **Expected**: 
  - API Gateway calls: 200 OK
  - Other calls: 403 Forbidden
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]

## Test Case 3: Retry Policy
- **Mục tiêu**: Verify retry khi service trả 500
- **Steps**:
  1. Deploy test service trả 500
  2. Call service từ API Gateway
  3. Check logs để thấy retry attempts
- **Expected**: 3 retry attempts được thực hiện
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]
```

### 7.2. Tạo README cho Service Mesh
Tạo file `README-SERVICE-MESH.md` với:
- Tổng quan về cấu hình
- Các bước triển khai
- Cách test
- Troubleshooting

---

# PHẦN 2: DEVSECOPS

## Mục tiêu
- Tích hợp SonarQube vào Jenkins pipeline (SAST)
- Tích hợp Snyk vào pipeline (Dependency scanning)
- Tích hợp OWASP ZAP vào pipeline (DAST)
- Setup GitLeaks pre-commit hook (Secret scanning)

---

## Bước 1: Chuẩn bị SonarQube

### 1.1. Kiểm tra SonarQube đang chạy
```bash
# Check SonarQube container
docker ps | grep sonarqube

# Nếu chưa có, start SonarQube (đã cài ở SETUP-VM-GUIDE.md)
docker start sonarqube

# Wait for SonarQube to be ready
docker logs -f sonarqube
# Wait until you see: "SonarQube is operational"
```

### 1.2. Tạo SonarQube Project và Token
1. Truy cập: http://localhost:9000
2. Login: admin/admin (đổi password nếu lần đầu)
3. Tạo project:
   - **Administration** → **Projects** → **Create Project**
   - Project key: `spring-petclinic-microservices`
   - Display name: `Spring PetClinic Microservices`
4. Tạo token:
   - **My Account** → **Security** → **Generate Token**
   - Token name: `jenkins-token`
   - Copy token (chỉ hiển thị 1 lần)

### 1.3. Cài đặt SonarQube Scanner
```bash
# Download SonarQube Scanner
cd ~
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip sonar-scanner-cli-5.0.1.3006-linux.zip
sudo mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner
sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner

# Verify
sonar-scanner --version
```

---

## Bước 2: Cấu hình Jenkins Pipeline với SonarQube

### 2.1. Cài đặt SonarQube Plugin trong Jenkins
1. Truy cập Jenkins: http://localhost:8080
2. **Manage Jenkins** → **Manage Plugins**
3. Tab **Available** → Tìm "SonarQube Scanner"
4. Install và restart Jenkins

### 2.2. Cấu hình SonarQube Server trong Jenkins
1. **Manage Jenkins** → **Configure System**
2. Tìm section **SonarQube servers**
3. Add SonarQube:
   - **Name**: `sonarqube-local`
   - **Server URL**: `http://localhost:9000`
   - **Server authentication token**: Paste token từ Bước 1.2
4. **Save**

### 2.3. Tạo SonarQube Properties File
Trong project root, tạo file `sonar-project.properties`:
```properties
sonar.projectKey=spring-petclinic-microservices
sonar.projectName=Spring PetClinic Microservices
sonar.projectVersion=4.0.1
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.java.binaries=**/target/classes
sonar.exclusions=**/target/**,**/node_modules/**,**/*.jar
sonar.coverage.exclusions=**/src/test/**,**/src/main/java/**/config/**
```

### 2.4. Tạo Jenkinsfile với SonarQube
Tạo file `Jenkinsfile` trong project root:
```groovy
pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://localhost:9000'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/spring-petclinic/spring-petclinic-microservices.git'
            }
        }
        
        stage('Build') {
            steps {
                sh './mvnw clean compile'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-local') {
                    sh './mvnw sonar:sonar \
                        -Dsonar.projectKey=spring-petclinic-microservices \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=${SONAR_TOKEN}'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }
        
        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
    }
}
```

### 2.5. Tạo Jenkins Job
1. **New Item** → **Pipeline**
2. Name: `petclinic-pipeline`
3. **Pipeline** → **Definition**: Pipeline script from SCM
4. **SCM**: Git
5. **Repository URL**: `https://github.com/spring-petclinic/spring-petclinic-microservices.git`
6. **Script Path**: `Jenkinsfile`
7. **Save** và **Build Now**

---

## Bước 3: Tích hợp Snyk vào Pipeline

### 3.1. Tạo Snyk Account và Get Token
1. Đăng ký tại: https://snyk.io
2. **Settings** → **Auth Token** → **Generate Token**
3. Copy token

### 3.2. Cài đặt Snyk CLI (nếu chưa có)
```bash
# Đã cài ở SETUP-VM-GUIDE.md, verify:
snyk --version

# Authenticate
snyk auth <your-token>
```

### 3.3. Test Snyk Scan
```bash
cd ~/spring-petclinic-microservices
snyk test --severity-threshold=high
snyk monitor  # Monitor project trên Snyk dashboard
```

### 3.4. Cài đặt Snyk Plugin trong Jenkins
1. **Manage Jenkins** → **Manage Plugins**
2. Tìm "Snyk Security"
3. Install và restart

### 3.5. Cấu hình Snyk trong Jenkins
1. **Manage Jenkins** → **Configure System**
2. Tìm **Snyk**
3. Add Snyk installation:
   - **Name**: `snyk-cli`
   - **Installation path**: `/usr/local/bin/snyk`
4. **Save**

### 3.6. Cập nhật Jenkinsfile với Snyk
Thêm vào Jenkinsfile:
```groovy
stage('Snyk Dependency Scan') {
    steps {
        script {
            try {
                sh 'snyk test --severity-threshold=high --json > snyk-report.json || true'
                sh 'snyk test --severity-threshold=high'
            } catch (Exception e) {
                echo "Snyk scan found vulnerabilities"
                archiveArtifacts artifacts: 'snyk-report.json'
            }
        }
    }
}
```

### 3.7. Fix một vulnerability (Demo)
```bash
# Scan và xem vulnerabilities
snyk test

# Fix suggestions
snyk test --json | jq '.vulnerabilities[0]'

# Update dependency nếu có fix
# Ví dụ: update trong pom.xml
```

---

## Bước 4: Tích hợp OWASP ZAP vào Pipeline (DAST)

### 4.1. Tạo ZAP Baseline Script
Tạo file `zap-baseline-scan.sh`:
```bash
#!/bin/bash

# Start ZAP in daemon mode
docker run -d --name zap -p 8080:8080 owasp/zap2docker-stable zap.sh -daemon \
    -host 0.0.0.0 -port 8080 -config api.disablekey=true

# Wait for ZAP to start
sleep 30

# Get API Gateway URL (adjust based on your setup)
TARGET_URL="http://localhost:30080"  # NodePort hoặc LoadBalancer IP

# Run baseline scan
docker exec zap zap-baseline.py -t $TARGET_URL -J zap-report.json

# Generate HTML report
docker exec zap zap-cli report -o /zap/report.html -f html

# Copy report out
docker cp zap:/zap/report.html ./zap-report.html
docker cp zap:/zap/zap-report.json ./zap-report.json

# Stop ZAP
docker stop zap
docker rm zap
```

```bash
chmod +x zap-baseline-scan.sh
```

### 4.2. Tạo ZAP Active Scan Script (nếu cần)
Tạo file `zap-active-scan.sh`:
```bash
#!/bin/bash

TARGET_URL="http://localhost:30080"

# Start ZAP
docker run -d --name zap -p 8080:8080 owasp/zap2docker-stable zap.sh -daemon \
    -host 0.0.0.0 -port 8080 -config api.disablekey=true

sleep 30

# Spider scan
docker exec zap zap-cli spider $TARGET_URL

# Active scan
docker exec zap zap-cli active-scan $TARGET_URL

# Generate report
docker exec zap zap-cli report -o /zap/report.html -f html
docker cp zap:/zap/report.html ./zap-report.html

docker stop zap
docker rm zap
```

### 4.3. Cập nhật Jenkinsfile với ZAP
Thêm stage:
```groovy
stage('ZAP DAST Scan') {
    steps {
        script {
            // Đảm bảo application đang chạy
            sh './zap-baseline-scan.sh'
            archiveArtifacts artifacts: 'zap-report.html,zap-report.json'
            
            // Check for high severity issues
            sh '''
                if grep -q "High" zap-report.json; then
                    echo "High severity vulnerabilities found!"
                    exit 1
                fi
            '''
        }
    }
}
```

---

## Bước 5: Setup GitLeaks Pre-commit Hook

### 5.1. Cài đặt GitLeaks (nếu chưa có)
```bash
# Đã cài ở SETUP-VM-GUIDE.md, verify:
gitleaks --version
```

### 5.2. Tạo Pre-commit Hook Script
Tạo file `.gitleaks.toml` trong project root:
```toml
title = "Gitleaks config for Spring PetClinic"

[extend]
# Use default regexes
useDefault = true

[allowlist]
description = "Allowlist for known safe patterns"
paths = [
    '''\.md$''',
    '''\.txt$''',
]
```

Tạo file `.git/hooks/pre-commit`:
```bash
#!/bin/bash

# Run gitleaks
gitleaks detect --source . --verbose --no-banner

# Exit with gitleaks exit code
exit $?
```

```bash
chmod +x .git/hooks/pre-commit
```

### 5.3. Test GitLeaks Hook
```bash
# Tạo file test với secret
echo "password=secret123" > test-secret.txt

# Try to commit
git add test-secret.txt
git commit -m "Test secret"
# Expected: Gitleaks should block commit

# Remove test file
rm test-secret.txt
```

### 5.4. Tạo Server-side Hook (Optional)
Nếu có Git server, tạo `pre-receive` hook:
```bash
#!/bin/bash

while read oldrev newrev refname; do
    git diff --name-only $oldrev $newrev | while read file; do
        gitleaks detect --source . --verbose --no-banner --log-opts="$oldrev..$newrev"
        if [ $? -ne 0 ]; then
            echo "ERROR: Secret detected in $file"
            exit 1
        fi
    done
done
```

---

## Bước 6: Hoàn thiện Jenkins Pipeline

### 6.1. Jenkinsfile hoàn chỉnh
Tạo file `Jenkinsfile-complete.groovy`:
```groovy
pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://localhost:9000'
        SNYK_TOKEN = credentials('snyk-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/spring-petclinic/spring-petclinic-microservices.git'
            }
        }
        
        stage('GitLeaks Scan') {
            steps {
                sh 'gitleaks detect --source . --verbose --no-banner'
            }
        }
        
        stage('Build') {
            steps {
                sh './mvnw clean compile'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-local') {
                    sh './mvnw sonar:sonar \
                        -Dsonar.projectKey=spring-petclinic-microservices \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=${SONAR_TOKEN}'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Snyk Dependency Scan') {
            steps {
                script {
                    sh 'snyk auth ${SNYK_TOKEN}'
                    sh 'snyk test --severity-threshold=high --json > snyk-report.json || true'
                    sh 'snyk test --severity-threshold=high'
                }
            }
        }
        
        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }
        
        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }
        
        stage('Deploy to K8s') {
            steps {
                sh 'kubectl apply -f k8s-manifests/ -n petclinic'
            }
        }
        
        stage('ZAP DAST Scan') {
            steps {
                script {
                    sleep 30  // Wait for app to be ready
                    sh './zap-baseline-scan.sh'
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '**/target/*.jar,snyk-report.json,zap-report.html,zap-report.json', fingerprint: true
            publishHTML([
                reportDir: '.',
                reportFiles: 'zap-report.html',
                reportName: 'ZAP Security Report'
            ])
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

### 6.2. Tạo Credentials trong Jenkins
1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials**
2. Add credentials:
   - **Kind**: Secret text
   - **Secret**: SonarQube token
   - **ID**: `sonar-token`
3. Add Snyk token tương tự với ID: `snyk-token`

---

## Bước 7: Tạo Test Plan và Documentation cho DevSecOps

### 7.1. Test Plan Template
Tạo file `TEST-PLAN-DEVSECOPS.md`:

```markdown
# Test Plan - DevSecOps

## Test Case 1: SonarQube SAST
- **Mục tiêu**: Verify SonarQube scan trong pipeline
- **Steps**:
  1. Trigger Jenkins build
  2. Check SonarQube stage
  3. Verify quality gate
- **Expected**: Quality gate PASS hoặc build fail
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]

## Test Case 2: Snyk Dependency Scan
- **Mục tiêu**: Verify Snyk phát hiện vulnerabilities
- **Steps**:
  1. Check Snyk stage trong pipeline
  2. View Snyk report
  3. Fix một vulnerability
- **Expected**: Snyk report vulnerabilities
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]

## Test Case 3: OWASP ZAP DAST
- **Mục tiêu**: Verify ZAP scan ứng dụng đang chạy
- **Steps**:
  1. Deploy application
  2. Run ZAP scan
  3. Check report
- **Expected**: No high-severity vulnerabilities
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]

## Test Case 4: GitLeaks Hook
- **Mục tiêu**: Verify GitLeaks chặn commit có secret
- **Steps**:
  1. Tạo file với secret
  2. Try to commit
  3. Verify commit bị chặn
- **Expected**: Commit bị reject
- **Result**: [PASS/FAIL]
- **Screenshot/Logs**: [Attach]
```

### 7.2. Tạo README cho DevSecOps
Tạo file `README-DEVSECOPS.md` với:
- Cấu hình các tools
- Cách chạy pipeline
- Cách xem reports
- Troubleshooting

---

## Deliverables Checklist

### Service Mesh:
- [ ] YAML manifests (mTLS, Authorization, Retry)
- [ ] Screenshot Kiali topology
- [ ] Test plan với results
- [ ] README hướng dẫn

### DevSecOps:
- [ ] Jenkinsfile với đầy đủ stages
- [ ] SonarQube properties và reports
- [ ] Snyk reports
- [ ] ZAP reports (HTML/JSON)
- [ ] GitLeaks hook script
- [ ] Test plan với results

---

## Troubleshooting

### Service Mesh Issues:
- **Pods không có sidecar**: Check namespace label `istio-injection=enabled`
- **mTLS không hoạt động**: Verify PeerAuthentication và DestinationRule
- **Authorization bị chặn**: Check service account và principal names

### DevSecOps Issues:
- **SonarQube connection failed**: Check token và URL
- **Snyk auth failed**: Verify token và network
- **ZAP scan timeout**: Increase timeout hoặc check app accessibility
- **GitLeaks không chạy**: Check hook permissions và gitleaks installation

---

## Tài liệu tham khảo

- [Istio Documentation](https://istio.io/latest/docs/)
- [Kiali Documentation](https://kiali.io/documentation/)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Snyk Documentation](https://docs.snyk.io/)
- [OWASP ZAP Documentation](https://www.zaproxy.org/docs/)
- [GitLeaks Documentation](https://github.com/gitleaks/gitleaks)

