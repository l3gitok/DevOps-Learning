# Hướng dẫn Cài đặt Docker Swarm cho Ubuntu Server

Tuyệt vời! Việc tự động hóa cài đặt bằng script sẽ giúp bạn tiết kiệm rất nhiều thời gian và đảm bảo tính nhất quán cho các node trong cụm Swarm. Dưới đây là hai script Bash hoàn chỉnh, một cho Manager Node và một cho Worker Node.

Mình sẽ đưa ra hướng dẫn sử dụng chi tiết sau mỗi script.

## 1. Script cài đặt cho Manager Node: `install_docker_manager.sh`

Script này sẽ thực hiện các bước sau trên máy chủ Manager:

- Gỡ bỏ các cài đặt Docker cũ (nếu có) để đảm bảo môi trường sạch.
- Cài đặt Docker Engine CE từ kho APT chính thức của Docker.
- Thêm người dùng hiện tại vào nhóm docker để có thể chạy lệnh docker mà không cần sudo.
- Khởi tạo Docker Swarm trên node này và thiết lập nó làm Manager.
- In ra lệnh docker swarm join để bạn có thể sao chép và sử dụng cho các Worker Node.

### Script install_docker_manager.sh

```bash
#!/bin/bash

# --- Kiểm tra và Chuẩn bị Ban đầu ---
if [[ $EUID -ne 0 ]]; then
   echo "Lỗi: Script này phải chạy với quyền root. Vui lòng sử dụng sudo."
   exit 1
fi

echo "--- Bắt đầu cài đặt Docker và khởi tạo Swarm trên Node Manager ---"

# --- 1. Gỡ bỏ Docker cũ và dọn dẹp (nếu tồn tại) ---
echo "Đang gỡ bỏ Docker Snap (nếu có) và các gói Docker cũ..."
snap remove docker &>/dev/null
apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras -y &>/dev/null
rm -rf /var/lib/docker &>/dev/null
rm -rf /var/lib/containerd &>/dev/null
echo "Đã gỡ bỏ và xóa dữ liệu Docker cũ."

# --- 2. Cài đặt các gói phụ thuộc cần thiết ---
echo "Đang cài đặt các gói phụ thuộc..."
apt-get update -y &>/dev/null
apt-get install -y ca-certificates curl apt-transport-https software-properties-common -y &>/dev/null
echo "Đã cài đặt các gói phụ thuộc."

# --- 3. Thêm khóa GPG chính thức của Docker ---
echo "Đang thêm khóa GPG của Docker..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "Đã thêm khóa GPG."

# --- 4. Thêm repository của Docker vào Apt sources ---
echo "Đang thêm repository Docker vào APT sources..."
echo \
  "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  \"$(. /etc/os-release && echo "$VERSION_CODENAME")\" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
echo "Đã thêm repository Docker."

# --- 5. Cập nhật APT và cài đặt Docker Engine ---
echo "Đang cập nhật APT và cài đặt Docker Engine..."
apt-get update -y &>/dev/null
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y &>/dev/null
echo "Đã cài đặt Docker Engine."

# --- 6. Thêm người dùng hiện tại vào nhóm 'docker' ---
USERNAME=$(logname) # Lấy tên người dùng hiện đang đăng nhập
echo "Đang thêm người dùng '$USERNAME' vào nhóm 'docker'..."
usermod -aG docker "$USERNAME"
echo "Đã thêm người dùng. Bạn cần đăng xuất/đăng nhập lại hoặc khởi động lại máy để thay đổi có hiệu lực."

# --- 7. Khởi tạo Docker Swarm ---
echo "Đang khởi tạo Docker Swarm..."
# Tự động lấy địa chỉ IP của manager node. Nếu có nhiều IP, nó sẽ lấy IP đầu tiên.
MANAGER_IP=$(hostname -I | awk '{print $1}')
if [ -z "$MANAGER_IP" ]; then
    echo "Lỗi: Không tìm thấy địa chỉ IP cho Node Manager. Vui lòng kiểm tra cấu hình mạng."
    exit 1
fi

SWARM_INIT_OUTPUT=$(docker swarm init --advertise-addr "$MANAGER_IP" 2>&1)
echo "$SWARM_INIT_OUTPUT"

# Trích xuất lệnh join token cho Worker Nodes
JOIN_COMMAND=$(echo "$SWARM_INIT_OUTPUT" | grep "docker swarm join --token")

echo "--- Cài đặt Docker trên Node Manager hoàn tất ---"
echo ""
echo "======================================================================================="
echo "  Vui lòng SAO CHÉP và GHI LẠI lệnh sau để các Node Worker tham gia cụm Swarm của bạn:"
echo "---------------------------------------------------------------------------------------"
echo "$JOIN_COMMAND"
echo "======================================================================================="
echo ""
echo "Bạn cần KHỞI ĐỘNG LẠI máy này để các thay đổi về nhóm người dùng có hiệu lực và để Docker Swarm được cấu hình đúng cách."
echo "Vui lòng gõ: sudo reboot"
```

### Hướng dẫn sử dụng install_docker_manager.sh

1. **Lưu script**: Copy nội dung trên và lưu vào một file có tên `install_docker_manager.sh` trên máy tính điều khiển của bạn (ví dụ: máy tính cá nhân hoặc một máy bastion host).

2. **Cấp quyền thực thi**: Mở terminal và chạy lệnh:

   ```bash
   chmod +x install_docker_manager.sh
   ```

3. **Copy script lên Manager Node**:

   ```bash
   scp install_docker_manager.sh your_username@192.168.30.153:/tmp/
   ```

   (Thay `your_username` bằng tên người dùng SSH của bạn trên Manager Node)

4. **Kết nối SSH vào Manager Node**:

   ```bash
   ssh your_username@192.168.30.153
   ```

5. **Chạy script**:

   ```bash
   sudo /tmp/install_docker_manager.sh
   ```

   Script sẽ chạy và hiển thị lệnh `docker swarm join` ở cuối. **Rất quan trọng**: Bạn phải sao chép toàn bộ lệnh này, nó sẽ được dùng cho các Worker Node.

6. **Khởi động lại Manager Node**: Như script đã khuyến nghị, hãy khởi động lại máy để các thay đổi về quyền người dùng và cấu hình Docker có hiệu lực.

   ```bash
   sudo reboot
   ```

Sau khi khởi động lại, đăng nhập lại vào Manager Node và bạn có thể kiểm tra trạng thái Swarm bằng `docker node ls`.

## 2. Script cài đặt cho Worker Node: `install_docker_worker.sh`

Script này sẽ thực hiện các bước sau trên mỗi máy chủ Worker:

- Gỡ bỏ các cài đặt Docker cũ (nếu có).
- Cài đặt Docker Engine CE.
- Thêm người dùng hiện tại vào nhóm docker.
- Buộc node rời khỏi bất kỳ cụm Swarm cũ nào (để đảm bảo khởi đầu sạch sẽ).
- Tham gia vào cụm Docker Swarm sử dụng lệnh `docker swarm join` được truyền vào làm đối số.

### Script install_docker_worker.sh

```bash
#!/bin/bash

# --- Kiểm tra và Chuẩn bị Ban đầu ---
if [[ $EUID -ne 0 ]]; then
   echo "Lỗi: Script này phải chạy với quyền root. Vui lòng sử dụng sudo."
   exit 1
fi

if [ -z "$1" ]; then
    echo "Lỗi: Vui lòng cung cấp LỆNH 'docker swarm join' đầy đủ từ Manager Node làm đối số."
    echo "Ví dụ: sudo $0 \"docker swarm join --token SWMTKN-1-... 192.168.30.153:2377\""
    exit 1
fi

JOIN_COMMAND="$1" # Lấy lệnh join được truyền vào từ đối số

echo "--- Bắt đầu cài đặt Docker và tham gia Swarm trên Node Worker ---"

# --- 1. Gỡ bỏ Docker cũ và dọn dẹp (nếu tồn tại) ---
echo "Đang gỡ bỏ Docker Snap (nếu có) và các gói Docker cũ..."
snap remove docker &>/dev/null
apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras -y &>/dev/null
rm -rf /var/lib/docker &>/dev/null
rm -rf /var/lib/containerd &>/dev/null
echo "Đã gỡ bỏ và xóa dữ liệu Docker cũ."

# --- 2. Cài đặt các gói phụ thuộc cần thiết ---
echo "Đang cài đặt các gói phụ thuộc..."
apt-get update -y &>/dev/null
apt-get install -y ca-certificates curl apt-transport-https software-properties-common -y &>/dev/null
echo "Đã cài đặt các gói phụ thuộc."

# --- 3. Thêm khóa GPG chính thức của Docker ---
echo "Đang thêm khóa GPG của Docker..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "Đã thêm khóa GPG."

# --- 4. Thêm repository của Docker vào Apt sources ---
echo "Đang thêm repository Docker vào APT sources..."
echo \
  "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  \"$(. /etc/os-release && echo "$VERSION_CODENAME")\" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
echo "Đã thêm repository Docker."

# --- 5. Cập nhật APT và cài đặt Docker Engine ---
echo "Đang cập nhật APT và cài đặt Docker Engine..."
apt-get update -y &>/dev/null
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y &>/dev/null
echo "Đã cài đặt Docker Engine."

# --- 6. Thêm người dùng hiện tại vào nhóm 'docker' ---
USERNAME=$(logname) # Lấy tên người dùng hiện đang đăng nhập
echo "Đang thêm người dùng '$USERNAME' vào nhóm 'docker'..."
usermod -aG docker "$USERNAME"
echo "Đã thêm người dùng. Bạn cần đăng xuất/đăng nhập lại hoặc khởi động lại máy để thay đổi có hiệu lực."

# --- 7. Rời khỏi Swarm cũ (đảm bảo) ---
echo "Đang đảm bảo node rời khỏi bất kỳ Swarm cũ nào..."
docker swarm leave --force &>/dev/null
echo "Đã rời khỏi Swarm cũ."

# --- 8. Tham gia Docker Swarm mới ---
echo "Đang tham gia Docker Swarm mới..."
eval "$JOIN_COMMAND" # Dùng eval để thực thi lệnh join đầy đủ
echo "Đã tham gia Swarm. Vui lòng kiểm tra trên Manager Node để xác nhận."

echo "--- Cài đặt Docker trên Node Worker hoàn tất ---"
echo ""
echo "Bạn cần KHỞI ĐỘNG LẠI máy này để các thay đổi về nhóm người dùng có hiệu lực."
echo "Vui lòng gõ: sudo reboot"
```

### Hướng dẫn sử dụng install_docker_worker.sh

1. **Lưu script**: Copy nội dung trên và lưu vào một file có tên `install_docker_worker.sh` trên máy tính điều khiển của bạn.

2. **Cấp quyền thực thi**: Mở terminal và chạy lệnh:

   ```bash
   chmod +x install_docker_worker.sh
   ```

3. **Copy script lên mỗi Worker Node**:

   ```bash
   scp install_docker_worker.sh your_username@192.168.30.154:/tmp/
   # Lặp lại cho các worker node khác nếu có
   ```

   (Thay `your_username` bằng tên người dùng SSH của bạn và `192.168.30.154` bằng IP của Worker Node)

4. **Kết nối SSH vào từng Worker Node**:

   ```bash
   ssh your_username@192.168.30.154
   ```

5. **Chạy script**: Truyền lệnh `docker swarm join` đầy đủ mà bạn đã sao chép từ Manager Node vào script dưới dạng đối số.

   ```bash
   sudo /tmp/install_docker_worker.sh "docker swarm join --token 
   # Đảm bảo toàn bộ lệnh join nằm trong dấu ngoặc kép " "
   # Thay thế token và IP bằng của bạn
   ```

6. **Khởi động lại Worker Node**:

   ```bash
   sudo reboot
   ```

7. **Lặp lại các bước 3-6** cho tất cả các Worker Node khác của bạn.

## 3. Tích Hợp Portainer vào Node Manager

Quá trình này sẽ diễn ra trên Node Manager của bạn.

### 1. Tạo Docker Volume cho Portainer

Portainer cần một Docker Volume để lưu trữ dữ liệu cấu hình, người dùng, và các cài đặt của nó một cách bền vững. Nếu không có volume này, mọi thay đổi bạn thực hiện trong Portainer sẽ bị mất khi container Portainer được khởi động lại hoặc xóa.

Đăng nhập vào Node Manager và chạy lệnh sau:

```bash
docker volume create portainer_data
```

### 2. Tạo File Stack cho Portainer

Chúng ta sẽ sử dụng một file Docker Compose (hay Stack file trong ngữ cảnh Swarm) để định nghĩa Portainer như một service trong cụm Swarm. Điều này đảm bảo Portainer được quản lý bởi Swarm và có thể tự động khởi động lại nếu có sự cố.

Tạo một file mới trên Node Manager của bạn, ví dụ: `portainer-stack.yml`. Bạn có thể dùng nano hoặc vi:

```bash
nano portainer-stack.yml
```

Dán nội dung sau vào file:

```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    command: -H unix:///var/run/docker.sock
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

volumes:
  portainer_data:
    external: true
```

Lưu và đóng file (Ctrl+O, Enter, Ctrl+X nếu dùng nano).

### 3. Triển khai Portainer Stack lên Docker Swarm

Bây giờ, chúng ta sẽ sử dụng lệnh `docker stack deploy` để triển khai Portainer như một service trong cụm Swarm của bạn. Lệnh này phải được chạy trên Node Manager.

Đảm bảo bạn đang ở cùng thư mục với file `portainer-stack.yml` và chạy:

```bash
docker stack deploy -c portainer-stack.yml portainer
```

**Giải thích các tham số:**

- `docker stack deploy`: Lệnh để triển khai một Docker Stack.
- `-c portainer-stack.yml`: Chỉ định file cấu hình Stack.
- `portainer`: Tên của Stack mà bạn muốn đặt (tên này sẽ xuất hiện trong Portainer và khi bạn dùng `docker stack ls`).

### 4. Kiểm tra trạng thái triển khai Portainer

Sau khi chạy lệnh deploy, Swarm sẽ mất một chút thời gian để kéo image Portainer và khởi động container. Bạn có thể kiểm tra trạng thái của service Portainer bằng lệnh sau trên Node Manager:

```bash
docker service ps portainer_portainer
```

Bạn sẽ thấy một đầu ra tương tự như sau, với trạng thái Running cho Portainer:

```text
ID             NAME                 IMAGE                          NODE      DESIRED STATE   CURRENT STATE
abc123def456   portainer_portainer.1   portainer/portainer-ce:latest   manager   Running         Running 2 minutes ago
```

Nếu CURRENT STATE chưa phải là Running, hãy đợi thêm một chút và chạy lại lệnh.

Bạn cũng có thể kiểm tra tất cả các stack đang chạy:

```bash
docker stack ls
```

### 5. Truy cập Giao diện Web của Portainer

Khi service Portainer đã Running, bạn có thể truy cập giao diện người dùng web của nó:

1. **Mở trình duyệt web** trên máy tính của bạn (máy đang truy cập được vào mạng chứa các VM).

2. **Nhập địa chỉ sau vào thanh địa chỉ:**

   ```text
   https://192.168.30.153:9443
   ```
   
   (Thay `192.168.30.153` bằng địa chỉ IP của Node Manager của bạn).

   **Lưu ý:** Bạn cũng có thể sử dụng HTTP trên port 9000:

   ```text
   http://192.168.30.153:9000
   ```

3. **Xử lý cảnh báo bảo mật:** Trình duyệt của bạn có thể hiển thị cảnh báo về chứng chỉ không an toàn (do Portainer sử dụng chứng chỉ tự ký mặc định). Hãy chấp nhận rủi ro và tiếp tục (hoặc chọn "Proceed to 192.168.30.153" tùy trình duyệt).

4. **Tạo tài khoản quản trị viên:** Lần đầu tiên truy cập, Portainer sẽ yêu cầu bạn tạo tài khoản quản trị viên. Hãy tạo một tên người dùng và mật khẩu mạnh.

5. **Kết nối với Docker Swarm:** Sau khi tạo tài khoản, đăng nhập. Portainer sẽ hỏi bạn muốn quản lý môi trường nào. Chọn "Docker Swarm" (hoặc "Local" nếu đó là lựa chọn cho môi trường Swarm) và nhấn "Connect".

### 6. Khám phá Portainer

Sau khi đăng nhập thành công, bạn sẽ thấy dashboard của Portainer với các thông tin về:

- **Nodes**: Danh sách tất cả các node trong cụm Swarm
- **Services**: Các service đang chạy trong Swarm
- **Stacks**: Các stack đã triển khai
- **Networks**: Mạng Docker
- **Volumes**: Các volume được tạo

Giờ đây bạn có thể quản lý toàn bộ cụm Docker Swarm thông qua giao diện web trực quan của Portainer!

---

**Lưu ý bảo mật:**

- Đảm bảo firewall chỉ cho phép truy cập port 9000/9443 từ các máy tin cậy
- Sử dụng mật khẩu mạnh cho tài khoản admin Portainer
- Cân nhắc cấu hình HTTPS với chứng chỉ hợp lệ cho môi trường production

---

> **Lưu ý quan trọng**: Hãy đảm bảo rằng tất cả các máy chủ đều có thể kết nối mạng với nhau và port 2377 (Swarm management port) không bị firewall chặn.
