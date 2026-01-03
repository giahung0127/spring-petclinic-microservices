# Hướng dẫn Setup Môi trường trên Máy ảo Linux

## Yêu cầu phần cứng
- **RAM**: Tối thiểu 8GB (khuyến nghị 16GB)
- **CPU**: 4 cores trở lên
- **Disk**: 50GB trống
- **OS**: Ubuntu 22.04 LTS

## Bước 1: Tạo máy ảo

### Với VirtualBox (Miễn phí)
1. Download VirtualBox: https://www.virtualbox.org/
2. Download Ubuntu 22.04 ISO: https://ubuntu.com/download/desktop
3. Tạo VM mới:
   - Name: `petclinic-devsecops`
   - Type: Linux
   - Version: Ubuntu (64-bit)
   - RAM: 8192 MB (8GB)
   - Hard disk: 50GB (VDI, Dynamically allocated)

### Với VMware Workstation Player (Miễn phí)
1. Download: https://www.vmware.com/products/workstation-player.html
2. Tạo VM tương tự VirtualBox

## Bước 2: Cài đặt Ubuntu
1. Boot từ ISO
2. Chọn "Install Ubuntu"
3. Cấu hình:
   - Language: English hoặc Vietnamese
   - Keyboard: theo ý bạn
   - Installation type: Erase disk and install Ubuntu
   - User: tạo user và password

## Bước 3: Cập nhật hệ thống
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

## Bước 4: Cài đặt Docker
```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
```

## Bước 5: Cài đặt Kubernetes (Kind - Khuyến nghị cho dev)
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name petclinic-cluster

# Verify
kubectl cluster-info --context kind-petclinic-cluster
kubectl get nodes
```

## Bước 6: Cài đặt Istio
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set values.defaultRevision=default -y

# Install addons (Kiali, Prometheus, Grafana)
kubectl apply -f samples/addons/kiali.yaml
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml

# Verify
kubectl get pods -n istio-system
```

## Bước 7: Cài đặt Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## Bước 8: Cài đặt Jenkins
```bash
# Add Jenkins repo
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt-get update
sudo apt-get install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Bước 9: Cài đặt SonarQube (Docker)
```bash
# Create network
docker network create sonarqube

# Run SonarQube
docker run -d --name sonarqube \
  --network sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community

# Wait for SonarQube to start (check logs)
docker logs -f sonarqube
# Access at http://localhost:9000
# Default: admin/admin (change on first login)
```

## Bước 10: Cài đặt các tools khác
```bash
# Snyk CLI - Cách 1: Download binary từ CDN chính thức (Khuyến nghị)
curl --compressed https://downloads.snyk.io/cli/stable/snyk-linux -o snyk
chmod +x ./snyk
sudo mv ./snyk /usr/local/bin/
snyk --version

# Snyk CLI - Cách 2: Download từ GitHub Releases
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
    ARCH="amd64"
fi
curl -Lo snyk https://github.com/snyk/cli/releases/latest/download/snyk-linux-${ARCH}
chmod +x snyk
sudo mv snyk /usr/local/bin/
snyk --version

# Snyk CLI - Cách 3: Cài đặt bằng npm (cần fix permissions trước)
# Xem hướng dẫn fix npm permissions ở cuối file
# npm install -g snyk

# Thêm vào PATH nếu cần
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Kiểm tra
snyk --version

# Hoặc nếu muốn dùng npm (cần fix permissions trước):
# Xem hướng dẫn fix npm permissions ở cuối file
# npm install -g snyk

# OWASP ZAP (Docker)
docker pull owasp/zap2docker:stable

# GitLeaks
wget https://github.com/gitleaks/gitleaks/releases/download/v8.18.0/gitleaks_8.18.0_linux_x64.tar.gz
tar -xzf gitleaks_8.18.0_linux_x64.tar.gz
sudo mv gitleaks /usr/local/bin/
```

## Bước 11: Clone project
```bash
# Install Git
sudo apt install -y git

# Clone project
cd ~
git clone https://github.com/spring-petclinic/spring-petclinic-microservices.git
cd spring-petclinic-microservices

# Install Java 17 (nếu chưa có)
sudo apt install -y openjdk-17-jdk
java -version
```

## Bước 12: Cấu hình mạng máy ảo

### Lựa chọn 1: Bridge Mode (Khuyến nghị - Không cần port forwarding)

Khi dùng **Bridge mode**, VM sẽ có IP trực tiếp trên mạng LAN như một máy thật. **KHÔNG CẦN** cấu hình port forwarding.

#### VirtualBox:
- Settings → Network → Adapter 1 → **Bridged Adapter**
- Chọn tên card mạng của host (thường là Ethernet hoặc Wi-Fi)
- VM sẽ tự động nhận IP từ router DHCP

#### VMware:
- Network Adapter → **Bridged**
- Chọn card mạng tương ứng
- VM sẽ tự động nhận IP từ router DHCP

**Lưu ý khi dùng Bridge:**
- VM có thể truy cập trực tiếp từ host hoặc các máy khác trên mạng
- Đảm bảo firewall trên VM cho phép các port cần thiết:
  ```bash
  sudo ufw allow 8080/tcp  # API Gateway, Jenkins
  sudo ufw allow 8761/tcp  # Eureka
  sudo ufw allow 8888/tcp  # Config Server
  sudo ufw allow 9000/tcp  # SonarQube
  sudo ufw allow 20001/tcp # Kiali
  ```

### Lựa chọn 2: NAT Mode (Cần port forwarding)

Nếu dùng **NAT mode**, bạn cần cấu hình port forwarding để truy cập các service từ host.

#### VirtualBox:
- Settings → Network → Adapter 1 → **NAT**
- Advanced → Port Forwarding:
  - 8080 → 8080 (API Gateway)
  - 8761 → 8761 (Eureka)
  - 8888 → 8888 (Config Server)
  - 9000 → 9000 (SonarQube)
  - 8080 → 8080 (Jenkins)
  - 20001 → 20001 (Kiali)

#### VMware:
- Network Adapter → **NAT**
- Settings → Network Adapter → NAT Settings → Port Forwarding
- Thêm các port tương tự như trên

## Kiểm tra
```bash
# Check Docker
docker ps

# Check Kubernetes
kubectl get nodes

# Check Istio
kubectl get pods -n istio-system

# Check Jenkins
sudo systemctl status jenkins

# Check SonarQube
docker ps | grep sonarqube
```

## Lưu ý
- Tạo snapshot sau mỗi bước quan trọng
- Backup cấu hình quan trọng
- Đảm bảo máy ảo có đủ tài nguyên

## Mở rộng Disk Space cho VM (Extend Filesystem)

Nếu bạn đã tăng kích thước virtual disk trong VMware/VirtualBox (ví dụ: từ 20GB lên 40GB) nhưng `df -h` vẫn chỉ hiển thị dung lượng cũ, bạn cần mở rộng filesystem bên trong VM.

### Bước 1: Kiểm tra kích thước thực tế của disk

```bash
# Kiểm tra kích thước thực tế của disk (sẽ hiển thị 40GB nếu đã tăng)
lsblk

# Hoặc
sudo fdisk -l /dev/sda
```

Bạn sẽ thấy `/dev/sda` có kích thước lớn hơn (40GB), nhưng partition `/dev/sda2` vẫn chỉ 20GB.

### Bước 2: Mở rộng partition (nếu cần)

**Lưu ý**: Nếu partition đã chiếm hết không gian, bỏ qua bước này và chuyển sang Bước 3.

```bash
# Cài đặt growpart nếu chưa có
sudo apt-get update
sudo apt-get install -y cloud-guest-utils

# Mở rộng partition 2 (thường là root partition)
sudo growpart /dev/sda 2
```

### Bước 3: Mở rộng filesystem

```bash
# Kiểm tra loại filesystem
df -T /

# Nếu là ext4 (thường gặp), dùng:
sudo resize2fs /dev/sda2

# Nếu là xfs, dùng:
# sudo xfs_growfs /
```

### Bước 4: Kiểm tra lại

```bash
# Kiểm tra dung lượng đã tăng chưa
df -h /

# Kiểm tra partition
lsblk
```

Bây giờ bạn sẽ thấy filesystem đã sử dụng đầy đủ 40GB.

### Troubleshooting

#### Nếu `growpart` không hoạt động:

```bash
# Sử dụng fdisk để mở rộng partition thủ công
sudo fdisk /dev/sda

# Trong fdisk:
# 1. Nhấn 'p' để xem partition table
# 2. Nhấn 'd' để xóa partition 2 (chỉ xóa partition entry, không xóa data)
# 3. Nhấn 'n' để tạo partition mới
# 4. Chọn 'p' (primary)
# 5. Chọn '2' (partition number)
# 6. Nhấn Enter để dùng first sector mặc định
# 7. Nhấn Enter để dùng last sector mặc định (sẽ dùng hết không gian)
# 8. Nhấn 'N' khi hỏi remove signature
# 9. Nhấn 'w' để write và exit

# Sau đó mở rộng filesystem
sudo resize2fs /dev/sda2
```

#### Nếu disk vẫn 100% full sau khi mở rộng:

```bash
# Tìm các file/thư mục chiếm nhiều dung lượng
sudo du -h --max-depth=1 / | sort -hr | head -20

# Dọn dẹp Docker (nếu không cần)
docker system prune -a --volumes

# Dọn dẹp apt cache
sudo apt-get clean
sudo apt-get autoremove

# Dọn dẹp log files cũ
sudo journalctl --vacuum-time=7d
```

### Lưu ý quan trọng

1. **Backup trước khi thực hiện**: Tạo snapshot của VM trước khi mở rộng filesystem
2. **Không thực hiện khi đang sử dụng**: Tốt nhất là thực hiện khi VM đang chạy nhưng không có tác vụ quan trọng
3. **Kiểm tra filesystem**: Đảm bảo filesystem không bị lỗi trước khi mở rộng:
   ```bash
   sudo fsck -f /dev/sda2
   ```
4. **VirtualBox**: Nếu dùng VirtualBox, cần dùng `VBoxManage` để resize disk trước:
   ```bash
   # Trên Windows host
   VBoxManage modifyhd "path/to/disk.vdi" --resize 40960
   ```
   Sau đó mới thực hiện các bước trên trong VM.

