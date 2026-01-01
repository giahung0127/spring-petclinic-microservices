# Hướng dẫn Truy cập Jenkins từ Windows

## Vấn đề
Jenkins mặc định chạy trên port **8080**, nhưng API Gateway cũng dùng port này. Cần giải quyết xung đột port.

## Giải pháp 1: Thay đổi Port của Jenkins (Khuyến nghị)

### Bước 1: Thay đổi port Jenkins trong Ubuntu VM

```bash
# Chỉnh sửa file cấu hình Jenkins
sudo nano /etc/default/jenkins

# Tìm dòng HTTP_PORT=8080 và đổi thành:
HTTP_PORT=8081

# Hoặc nếu không có, thêm dòng:
HTTP_PORT=8081

# Lưu file (Ctrl+O, Enter, Ctrl+X)

# Khởi động lại Jenkins
sudo systemctl restart jenkins

# Kiểm tra Jenkins đang chạy
sudo systemctl status jenkins

# Kiểm tra port
sudo netstat -tlnp | grep jenkins
# Hoặc
sudo ss -tlnp | grep jenkins
```

### Bước 2: Cấu hình Port Forwarding trong VMware

1. **Tắt máy ảo** (nếu đang chạy)

2. **Mở cài đặt VM:**
   - Right-click vào VM → **Settings** (hoặc **Edit virtual machine settings**)
   - Chọn **Network Adapter**

3. **Cấu hình NAT:**
   - Chọn **NAT** (Network Address Translation)
   - Click **NAT Settings...** (hoặc **Advanced** → **Port Forwarding**)

4. **Thêm Port Forwarding:**
   - Click **Add** hoặc **+**
   - **Host Port**: `8081` (port trên Windows)
   - **Type**: TCP
   - **Virtual Machine IP Address**: Để trống hoặc IP của VM (sẽ tự động)
   - **Guest Port**: `8081` (port Jenkins trong VM)
   - **Description**: Jenkins
   - Click **OK**

5. **Lưu và khởi động lại VM**

### Bước 3: Truy cập Jenkins từ Windows

1. **Mở trình duyệt trên Windows**
2. **Truy cập:** `http://localhost:8081`
3. **Hoặc:** `http://127.0.0.1:8081`

### Bước 4: Lấy mật khẩu ban đầu

Trong Ubuntu VM, chạy lệnh:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy mật khẩu và paste vào Jenkins setup wizard.

---

## Giải pháp 2: Sử dụng Bridged Network (Không cần Port Forwarding)

### Bước 1: Cấu hình Bridged Network trong VMware

1. **Tắt máy ảo**

2. **Mở cài đặt VM:**
   - Right-click VM → **Settings**
   - Chọn **Network Adapter**
   - Chọn **Bridged: Connected directly to the physical network**
   - Click **OK**

3. **Khởi động VM**

### Bước 2: Lấy IP của VM

Trong Ubuntu VM, chạy:
```bash
# Lấy IP address
ip addr show
# Hoặc
hostname -I
# Hoặc
ifconfig
```

Bạn sẽ thấy IP như: `192.168.1.xxx` hoặc `192.168.0.xxx`

### Bước 3: Truy cập Jenkins từ Windows

1. **Mở trình duyệt trên Windows**
2. **Truy cập:** `http://<IP-VM>:8080`
   - Ví dụ: `http://192.168.1.100:8080`

### Bước 4: Kiểm tra Firewall (nếu không truy cập được)

Trong Ubuntu VM:
```bash
# Kiểm tra firewall
sudo ufw status

# Nếu firewall đang bật, mở port Jenkins
sudo ufw allow 8080/tcp
# Hoặc nếu đã đổi port
sudo ufw allow 8081/tcp

# Reload firewall
sudo ufw reload
```

---

## Giải pháp 3: Sử dụng Port Forwarding với Port khác (Giữ Jenkins port 8080)

Nếu muốn giữ Jenkins ở port 8080 trong VM nhưng forward sang port khác trên Windows:

### Cấu hình Port Forwarding:
- **Host Port**: `9090` (hoặc port khác bạn muốn)
- **Guest Port**: `8080`
- **Type**: TCP

Truy cập từ Windows: `http://localhost:9090`

---

## Kiểm tra và Troubleshooting

### 1. Kiểm tra Jenkins đang chạy
```bash
# Trong Ubuntu VM
sudo systemctl status jenkins
```

### 2. Kiểm tra port đang lắng nghe
```bash
# Trong Ubuntu VM
sudo netstat -tlnp | grep jenkins
# Hoặc
sudo ss -tlnp | grep jenkins
```

### 3. Kiểm tra từ Windows
```powershell
# Mở PowerShell trên Windows
Test-NetConnection -ComputerName localhost -Port 8081
# Hoặc nếu dùng Bridged
Test-NetConnection -ComputerName <IP-VM> -Port 8080
```

### 4. Xem log Jenkins nếu có lỗi
```bash
# Trong Ubuntu VM
sudo journalctl -u jenkins -f
# Hoặc
sudo tail -f /var/log/jenkins/jenkins.log
```

### 5. Kiểm tra firewall Windows
- Đảm bảo Windows Firewall không chặn port
- Hoặc tạm thời tắt để test

---

## Tóm tắt các Port cần Forward (nếu dùng NAT)

| Service | Guest Port | Host Port (Windows) |
|---------|-----------|-------------------|
| Jenkins | 8081 | 8081 |
| API Gateway | 8080 | 8080 |
| Eureka | 8761 | 8761 |
| Config Server | 8888 | 8888 |
| SonarQube | 9000 | 9000 |
| Kiali | 20001 | 20001 |

---

## Lưu ý

1. **NAT Mode**: Cần cấu hình port forwarding, nhưng VM không có IP riêng trên mạng
2. **Bridged Mode**: VM có IP riêng, không cần port forwarding, nhưng cần cùng mạng với Windows
3. **Jenkins Port**: Mặc định 8080, nên đổi sang 8081 để tránh xung đột với API Gateway
4. **Firewall**: Đảm bảo cả Windows và Ubuntu đều cho phép kết nối

---

## Cách Reset Jenkins về Setup Wizard

Nếu bạn muốn reset Jenkins về trang setup wizard ban đầu (để cấu hình lại từ đầu):

### Phương pháp 1: Xóa file cấu hình (Khuyến nghị - Giữ lại dữ liệu)

```bash
# Trong Ubuntu VM, dừng Jenkins
sudo systemctl stop jenkins

# Xóa file đánh dấu đã cài đặt
sudo rm /var/lib/jenkins/jenkins.install.InstallUtil.lastExecVersion

# Hoặc nếu file không tồn tại, tìm và xóa file tương tự:
sudo find /var/lib/jenkins -name "*InstallUtil*" -type f -delete

# Khởi động lại Jenkins
sudo systemctl start jenkins
```

Sau khi khởi động lại, truy cập Jenkins sẽ hiển thị setup wizard.

### Phương pháp 2: Xóa toàn bộ thư mục Jenkins (Reset hoàn toàn)

**⚠️ CẢNH BÁO**: Phương pháp này sẽ xóa TẤT CẢ dữ liệu, jobs, plugins, và cấu hình của Jenkins!

```bash
# Trong Ubuntu VM, dừng Jenkins
sudo systemctl stop jenkins

# Backup thư mục Jenkins (tùy chọn, nếu muốn giữ lại)
sudo cp -r /var/lib/jenkins /var/lib/jenkins.backup

# Xóa toàn bộ thư mục Jenkins
sudo rm -rf /var/lib/jenkins

# Tạo lại thư mục với quyền phù hợp
sudo mkdir /var/lib/jenkins
sudo chown jenkins:jenkins /var/lib/jenkins

# Khởi động lại Jenkins (sẽ tự động tạo lại cấu hình mặc định)
sudo systemctl start jenkins

# Lấy mật khẩu ban đầu mới
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Phương pháp 3: Sử dụng System Property (Tạm thời)

Nếu muốn chỉ hiển thị setup wizard một lần mà không xóa dữ liệu:

```bash
# Dừng Jenkins
sudo systemctl stop jenkins

# Chỉnh sửa file cấu hình Jenkins
sudo nano /etc/default/jenkins

# Thêm dòng sau vào cuối file:
JENKINS_JAVA_OPTIONS="-Djenkins.install.runSetupWizard=true"

# Lưu file (Ctrl+O, Enter, Ctrl+X)

# Khởi động lại Jenkins
sudo systemctl start jenkins
```

Sau khi hoàn thành setup wizard, nhớ xóa dòng này để tránh hiển thị lại mỗi lần khởi động.

### Phương pháp 4: Reset chỉ cấu hình người dùng (Giữ lại Jobs)

Nếu chỉ muốn reset tài khoản admin và cấu hình người dùng:

```bash
# Dừng Jenkins
sudo systemctl stop jenkins

# Xóa cấu hình người dùng
sudo rm -rf /var/lib/jenkins/users
sudo rm -rf /var/lib/jenkins/secrets/initialAdminPassword
sudo rm /var/lib/jenkins/jenkins.install.InstallUtil.lastExecVersion

# Khởi động lại Jenkins
sudo systemctl start jenkins

# Lấy mật khẩu ban đầu mới
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Kiểm tra sau khi reset

1. Truy cập Jenkins từ trình duyệt: `http://localhost:8081` (hoặc port bạn đã cấu hình)
2. Bạn sẽ thấy trang setup wizard
3. Lấy mật khẩu ban đầu:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
4. Paste mật khẩu vào setup wizard và tiếp tục cấu hình

### Lưu ý

- **Phương pháp 1** là an toàn nhất, chỉ reset setup wizard nhưng giữ lại jobs và plugins
- **Phương pháp 2** sẽ xóa hoàn toàn, phù hợp khi muốn bắt đầu lại từ đầu
- Luôn backup dữ liệu quan trọng trước khi reset: `sudo cp -r /var/lib/jenkins /var/lib/jenkins.backup`
- Sau khi reset, bạn sẽ cần cài đặt lại plugins và cấu hình lại jobs

