# README.md — Lab Reverse Proxy Public + TLS (Let’s Encrypt) cho WordPress qua Cloudflare DNS

Tài liệu này hướng dẫn dựng mô hình **2 máy EC2**:

- **Upstream** chạy WordPress + MySQL bằng Docker Compose
- **Reverse Proxy** chạy Nginx public (port 80/443) + TLS Let’s Encrypt (DNS-01 qua Cloudflare)

Sau khi hoàn tất, truy cập WordPress bằng:

- `https://vscv.click`

---

## 1) Thông tin lab (theo setup thực tế)

| Thành phần | Giá trị |
|---|---|
| Domain | `vscv.click` |
| Reverse proxy public IP | `13.251.110.182` |
| Upstream public IP | `13.214.129.181` |
| Upstream app port | `8000` (WordPress container map `8000 -> 80`) |


## 2) Kiến trúc tổng quan

```text
Client (Browser)
   |
   |  https://vscv.click (DNS -> 13.251.110.182)
   v
Reverse Proxy (Nginx + TLS)
   |
   |  http://13.214.129.181:8000
   v
Upstream (WordPress)
```

---

## 3) Bước 1 — Cấu hình Cloudflare DNS cho domain

### 3.1 Add site và đổi Nameserver
1. Đăng nhập Cloudflare
2. Add site: `vscv.click`
3. Cloudflare cấp 2 nameserver
4. Vào trang quản lý domain (registrar) đổi Nameserver theo Cloudflare

Chờ DNS propagate (thường vài phút đến vài giờ).

### 3.2 Tạo DNS record
Vào **Cloudflare → DNS → Records**, tạo:

- A record:
  - **Name**: `@` (hoặc `vscv.click`)
  - **Content**: `13.251.110.182`
  - **Proxy**: **DNS only** 


### 3.3 Kiểm tra DNS từ máy local
PowerShell:

```powershell
nslookup vscv.click
nslookup -type=aaaa vscv.click
```

Kỳ vọng:
- A record trả về `13.251.110.182`
<img width="362" height="137" alt="image" src="https://github.com/user-attachments/assets/a6445864-345d-41a3-9568-c6494bcd552f" />

---

## 4) Bước 2 — Cài Docker & Docker Compose (cả 2 máy)

Chạy trên **Upstream** và **Reverse Proxy**:

### 4.1 Cài Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

### 4.2 Cài docker-compose (v1)
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 4.3 Thêm user vào group docker (nên)
```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Nếu vẫn chưa được, logout/login SSH lại rồi test:

```bash
docker ps
docker-compose --version
```
---

## 5) Bước 3 — Triển khai WordPress trên Upstream (13.214.129.181)

### 5.1 Clone repo sample
```bash
git clone https://github.com/thaihust/docker-compose.git
cd docker-compose/wordpress-mysql
```

### 5.2 Start WordPress
```bash
sudo docker-compose up -d
```

### 5.3 Kiểm tra container
```bash
sudo docker ps
```
<img width="1412" height="77" alt="image" src="https://github.com/user-attachments/assets/d4112c07-f53c-4f3c-9e7f-cfc6ab3df148" />

### 5.4 Kiểm tra upstream local
```bash
curl -I http://127.0.0.1:8000
```

## 6) Bước 4 — Lấy Cloudflare API key (để xin cert DNS-01)

Cloudflare Dashboard:
- My Profile → API Tokens → API Keys → **Global API Key** (View)

Cần:
- **Cloudflare email**
- **Global API Key**

---

## 7) Bước 5 — Xin chứng chỉ Let’s Encrypt trên Reverse Proxy (DNS-01)

Chạy trên máy reverse proxy `13.251.110.182`.

### 7.1 Tạo file credentials (tạm thời)
> Thay email và key của bạn vào đây.

```bash
cat << EOF > /tmp/cloudflare.ini
dns_cloudflare_email = your-email@example.com
dns_cloudflare_api_key = your-cloudflare-global-api-key
EOF
```

### 7.2 Tạo thư mục lưu cert
```bash
sudo mkdir -p /etc/letsencrypt /var/lib/letsencrypt
```

### 7.3 Chạy certbot (dns-cloudflare)
```bash
sudo docker pull certbot/dns-cloudflare

sudo docker run -it --rm --name certbot \
  -v "/etc/letsencrypt:/etc/letsencrypt" \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  -v "/tmp:/tmp" \
  certbot/dns-cloudflare certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /tmp/cloudflare.ini \
  -d vscv.click -d www.vscv.click
```
<img width="975" height="520" alt="image" src="https://github.com/user-attachments/assets/e4f7459d-4473-4a85-80f4-7d64e4c3878e" />
<img width="975" height="245" alt="image" src="https://github.com/user-attachments/assets/13eebd5b-b00e-44ab-8b79-5bd1052f5715" />

### 7.4 Xóa file credentials
```bash
rm -f /tmp/cloudflare.ini
```

### 7.5 Kiểm tra cert tồn tại
```bash
sudo ls -la /etc/letsencrypt/live/vscv.click/
```

---

## 8) Bước 6 — Dựng Nginx reverse proxy bằng Docker Compose

### 8.1 Tạo thư mục cấu hình
```bash
sudo mkdir -p /opt/nginx/conf.d /opt/nginx/log
```

### 8.2 Tạo file docker-compose cho Nginx

```bash
sudo tee /opt/nginx/docker-compose.yaml > /dev/null << EOF
version: '3'

services:
  reverse-proxy:
    container_name: reverse-proxy
    hostname: reverse-proxy
    image: nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - /opt/nginx/conf.d:/etc/nginx/conf.d
      - /opt/nginx/log:/var/log/nginx
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/lib/letsencrypt:/var/lib/letsencrypt
EOF
```

### 8.3 Start Nginx
```bash
sudo docker-compose -f /opt/nginx/docker-compose.yaml up -d
```

### 8.4 Kiểm tra container
```bash
sudo docker ps
```
<img width="1550" height="60" alt="image" src="https://github.com/user-attachments/assets/426c49e0-943b-4863-88bf-ed758237ee47" />

---

## 9) Bước 7 — Cấu hình Nginx reverse proxy cho domain

Tạo file `/opt/nginx/conf.d/vscv.click.conf`:

```bash
cat << 'EOF' | sudo tee /opt/nginx/conf.d/vscv.click.conf
server {
   listen 80;
   listen [::]:80;
   server_name vscv.click www.vscv.click;
   access_log /var/log/nginx/access-vscv.click.log;
   error_log /var/log/nginx/error-vscv.click.log;

   return 301 https://$host$request_uri;
}

server {
   listen 443 ssl http2;
   listen [::]:443 ssl http2;
   server_name vscv.click www.vscv.click;
   access_log /var/log/nginx/access-vscv.click.log;
   error_log /var/log/nginx/error-vscv.click.log;

   ssl_certificate      /etc/letsencrypt/live/vscv.click/fullchain.pem;
   ssl_certificate_key  /etc/letsencrypt/live/vscv.click/privkey.pem;

   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout 10m;

   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_prefer_server_ciphers on;
   ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";

   add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
   add_header X-Frame-Options DENY always;
   add_header X-Content-Type-Options nosniff always;
   add_header X-XSS-Protection "1; mode=block" always;

   ssl_stapling on;
   ssl_stapling_verify on;
   ssl_trusted_certificate /etc/letsencrypt/live/vscv.click/fullchain.pem;
   resolver 1.1.1.1 1.0.0.1 valid=300s;
   resolver_timeout 5s;

   location / {
     proxy_pass http://13.214.129.181:8000;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header X-Forwarded-Proto $scheme;
     proxy_set_header X-Forwarded-Host $host;
     proxy_set_header X-Forwarded-Port $server_port;
   }
EOF
```

---

## 10) Bước 8 — Reload Nginx và kiểm tra

### 10.1 Test config
```bash
sudo docker exec reverse-proxy nginx -t
```

### 10.2 Reload
```bash
sudo docker exec reverse-proxy nginx -s reload
```

### 10.3 Test từ reverse proxy
```bash
curl -I https://vscv.click
```

<img width="602" height="212" alt="image" src="https://github.com/user-attachments/assets/b0784f83-0182-4c8e-a259-2504645afb3c" />

---

## 11) Bước 9 — Kiểm tra từ máy local 

### 11.1 Kiểm tra DNS
```powershell
nslookup vscv.click
```
<img width="357" height="130" alt="image" src="https://github.com/user-attachments/assets/c835c17b-3a8b-420c-b032-481aa3828f70" />

### 13.2 Truy cập website
Mở trình duyệt:
- `https://vscv.click`
<img width="1907" height="1030" alt="image" src="https://github.com/user-attachments/assets/6bee344f-8ca4-4ab6-b864-bb0509d6510f" />

---

## 12) Kết quả cuối cùng
Sau khi hoàn tất:
- Domain `vscv.click` truy cập HTTPS được
- Nginx reverse proxy chạy public ở `13.251.110.182`
- WordPress chạy upstream ở `13.214.129.181:8000`
- TLS do Let’s Encrypt cấp qua Cloudflare DNS-01

---

