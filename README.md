# Cài đặt Odoo 18 với hệ điều hành Ubuntu 25.04 trên Google Cloud Platfrom

Việc cài đặt này sẽ được triển khai bằng Docker và Nginx.

## 1. Chuẩn bị hệ thống
### 1.1. Chuẩn bị các dependencies cần thiết
- Refresh các repository và cập nhật một vài Linux modules:
```bash
sudo apt update && sudo apt upgrade -y
```
- Cài đặt dependencies:
```bash
sudo apt install apt-transport-https ca-certificates curl nano gnupg gpg lsb-release -y
```
- Thêm Docker GPG key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
- Thêm Docker repository
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### 1.2. Cài đặt Docker
- Cài đặt Docker
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

- Khởi động Docker service
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

- Thêm user hiện tại của hệ thống vào docker group
```bash
sudo usermod -aG docker $USER
```
### 1.3. Cài đặt Docker Compose
- Tải Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

- Cấp quyền thực thi
```bash
sudo chmod +x /usr/local/bin/docker-compose
```

- Kiểm tra cài đặt
```bash
docker-compose --version
```
## 2. Tạo cấu trúc thư mục cho Odoo
### 2.1. Tạo các cấu trúc thư mục
- Tạo thư mục dự án
```bash
mkdir -p ~/odoo-docker
cd ~/odoo-docker
```

- Tạo các thư mục cần thiết khác
```bash
mkdir -p odoo-web-data odoo-db-data nginx/conf.d odoo/addons odoo/etc odoo/custom-addons
```
### 2.2. Tạo thư mục session (quan trọng)
- Thay đổi đối tượng và quyền truy cập của thư mục ```odoo-web-data```
```bash
sudo chown -R 101:101 odoo-web-data
sudo chmod -R 755 odoo-web-data
```
- Tạo thư mục session và thay đổi đối tượng truy cập
```bash
mkdir -p odoo-web-data/sessions
sudo chown -R 101:101 odoo-web-data/sessions
```

## 3. Tạo cấu hình Odoo
### 3.1. Tạo file config cho Odoo
```bash
cat > odoo/etc/odoo.conf << 'EOF'
[options]
addons_path = /mnt/extra-addons,/mnt/custom-addons
data_dir = /var/lib/odoo
db_host = db
db_port = 5432
db_user = odoo
db_password = myodoo123
db_maxconn = 64
db_template = template0
list_db = True
admin_passwd = admin123
xmlrpc_port = 8069
longpolling_port = 8072
proxy_mode = True
EOF
```
### 3.2. Tạo file Docker-Compose
- Khuyến nghị cài đặt PostgreSQL 16 để đảm bảo hiệu suất
```bash
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: postgres:16
    container_name: odoo_db
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=myodoo123
      - POSTGRES_USER=odoo
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - ./odoo-db-data:/var/lib/postgresql/data/pgdata
    restart: unless-stopped
    networks:
      - odoo-network

  odoo:
    image: odoo:18.0
    container_name: odoo_web
    user: "101:101"
    depends_on:
      - db
    ports:
      - "8069:8069"
      - "8072:8072"
    volumes:
      - ./odoo-web-data:/var/lib/odoo
      - ./odoo/etc:/etc/odoo
      - ./odoo/addons:/mnt/extra-addons
      - ./odoo/custom-addons:/mnt/custom-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=myodoo123
    restart: unless-stopped
    networks:
      - odoo-network

networks:
  odoo-network:
    driver: bridge

volumes:
  odoo-web-data:
  odoo-db-data:
EOF
```
## 4. Cài đặt Nginx và cấu hình Reverse Proxy để dùng trước ở giao thức HTTP
### 4.1. Cài đặt Nginx
```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```
### 4.2. Tạo file cấu hình Nginx cho Odoo
- Nếu chưa có tên miền:
```bash
sudo cat > /etc/nginx/sites-available/odoo << 'EOF'
server {
    listen 80;
    server_name 127.0.0.1:8069;

    # Log files
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Client settings
    client_max_body_size 100M;

    # Proxy timeout settings
    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    # Proxy headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Odoo web server
    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_redirect off;
    }

    # Odoo web client
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://127.0.0.1:8069;
    }

    # Longpolling
    location /longpolling {
        proxy_pass http://127.0.0.1:8072;
    }

    # Gzip compression
    gzip on;
    gzip_types text/css text/javascript application/javascript application/json;
    gzip_min_length 1000;
}
EOF
```
- Nếu đã có tên miền:
```bash
sudo cat > /etc/nginx/sites-available/<tên_miền_của_bạn> << 'EOF'
server {
    listen 80;
    server_name <tên_miền_của_bạn>;

    # Log files
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Client settings
    client_max_body_size 100M;

    # Proxy timeout settings
    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    # Proxy headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Odoo web server
    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_redirect off;
    }

    # Odoo web client
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://127.0.0.1:8069;
    }

    # Longpolling
    location /longpolling {
        proxy_pass http://127.0.0.1:8072;
    }

    # Gzip compression
    gzip on;
    gzip_types text/css text/javascript application/javascript application/json;
    gzip_min_length 1000;
}
EOF
```
### 4.3. Kích hoạt cấu hình Nginx
- Tạo symbolic link sang thư mục `site-enabled`
```bash
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/
```
hoặc nếu đã có tên miền:
```bash
sudo ln -s /etc/nginx/sites-available/<tên_miền_của_bạn> /etc/nginx/sites-enabled/
```
- Xóa file default ở thư mục `sites-enabled`
```bash
sudo rm /etc/nginx/sites-enabled/default
```

- Kiểm tra cấu hình xem chạy có dính lỗi gì không
```bash
sudo nginx -t
```

- Khởi động lại Nginx service
```bash
sudo systemctl restart nginx
```
### 4.4. Cấu hình tường lửa
- Mở luồng HTTP và HTTPS bằng cách gắn tag `http-server` và `https-server` vào máy chủ cài đặt Odoo trong cài đặt của Google Cloud
- Cho phép HTTP, HTTPS và kích hoạt firewall
```bash
sudo ufw allow 'Nginx Full' &&& sudo ufw enable
```
- Kiểm tra trạng thái
```bash
sudo ufw status
```

Nếu trong kết quả trả ra có 2 dòng dưới là đã thành công:
```bash
Nginx Full                 ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere (v6)
```

- **Nhớ:** Mở cổng 8069 và 8072 trên Google Cloud
- Đến đây, có thể truy cập được ở dạng HTTP rồi, có thể dùng bình thường, tuy nhiên, nếu muốn sử dụng giao thức HTTPS, tham khảo mục số 5
## 5. Cấu hình giao thức HTTPS

Nếu đã có tên miền thì có thể cấu hình để sử dụng HTTPS bằng các bước sau:

### 5.1. Cấu hình IP tĩnh trên Google Cloud
- Từ navigation menu ☰ chọn **VPC Network** > **IP addresses** > **Reserve external static IP address**
- Đặt tên tuỳ ý tại trường **Name**
- Ở trường **Attached to** chọn máy ảo sẽ gắn IP tĩnh
- Nhấn **Create** để hoàn thành
- Gắn địa chỉ IP tĩnh nhận được vào nơi cung cấp tên miền đã sở hữu
- Cấu hình file Nginx đã đề cập tại mục 4 với ở trường hợp đã có tên miền
### 5.2. Cấu hình HTTPS/SSL
- Cài đặt module tạo SSL certificates
```bash
sudo apt install certbot python3-certbot-nginx -y
```
- Cấu hình SSL certificates và cập nhật vào Nginx
```bash
sudo certbot --nginx -d <tên_miền_của_bạn>
```
- Ở bước này, nhập email của bạn vào, tốt nhất là email mà bạn đã dùng để đăng ký với máy chủ cung cấp nơi tên miền đã được cấp, sau đó chờ 1 lúc, hệ thống sẽ tự động tạo file certificate và SSL key đồng thời file Nginx config tương ứng sẽ được tự động cấu hình lại.
## 6. Cài đặt Odoo Module giao diện Odoo Enterprise Home Screen (tuỳ chọn)

Module này có hỗ trợ Dark Mode, và khá xịn ở chỗ có Home Menu Screen trực quan, dễ nhìn và dễ tuỳ chính, sắp xếp, và cung cấp giao diện tổng thể đẹp như người yêu của bạn vậy. Các bước làm như sau:

- Quay trở ra thư mục `/home/$USER` và clone repository về 
```bash
cd ~
git clone https://github.com/hrmuwanika/odooapps18.git
```
- Copy vào thư mục `custom-addons` đã tạo từ trước đó:
```bash
sudo cp -rf odooapps18/* odoo-docker/odoo/custom-addons 
```
- Khởi động lại container `odoo`:
```bash
cd ~/odoo-docker
docker-compose restart odoo
```
- Sau đó refresh danh sách ứng dụng và tìm module tên **ICA Web Responsive**. Xong, bạn đã có ngay 1 giao diện đẹp như mơ.
## 7. Cài đặt module WKHTMLTOPDF

Đây là một module nền hỗ trợ cho việc trích xuất báo cáo, các bước như sau:

- Download bản release từ GitHub:
```bash
cd ~ && sudo wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
```
- Cài đặt module:
```bash
sudo dpkg -i wkhtmltox_0.12.6.1-2.jammy_amd64.deb
sudo apt install -f
```
- Kiểm tra module xem đã cài đặt thành công chưa
```bash
wkhtmltopdf --version
```
