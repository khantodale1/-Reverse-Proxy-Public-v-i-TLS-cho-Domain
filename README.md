# README.md — Lab Reverse Proxy Public + TLS (Let’s Encrypt) cho WordPress qua Cloudflare DNS

Tài liệu này hướng dẫn dựng mô hình **2 máy EC2**:

- **Upstream** chạy WordPress + MySQL bằng Docker Compose
- **Reverse Proxy** chạy Nginx public (port 80/443) + TLS Let’s Encrypt (DNS-01 qua Cloudflare)

Sau khi hoàn tất, bạn truy cập WordPress bằng:

- `https://vscv.click`

---

## 1) Thông tin lab (theo setup thực tế)

| Thành phần | Giá trị |
|---|---|
| Domain | `vscv.click` |
| Reverse proxy public IP | `13.251.110.182` |
| Upstream public IP | `13.214.129.181` |
| Upstream app port | `8000` (WordPress container map `8000 -> 80`) |
| DNS provider | Cloudflare (DNS only trong lúc lab) |

Bạn có thể export biến để tiện copy/paste:

```bash
export DOMAIN="vscv.click"
export REVERSE_PROXY_IP="13.251.110.182"
export UPSTREAM_IP="13.214.129.181"
export APP_PORT="8000"
```

---

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

## 3) Yêu cầu trước khi bắt đầu

- 2 EC2 Ubuntu (khuyến nghị Ubuntu 20.04/22.04)
- Domain đã mua
- Tài khoản Cloudflare
- Domain đã trỏ Nameserver về Cloudflare
- Quyền SSH vào 2 EC2
- Mở Security Group đúng

---

## 4) Security Group / Firewall (cực quan trọng)

### 4.1 Reverse proxy (13.251.110.182)
Mở inbound:

- TCP `22` từ IP của bạn
- TCP `80` từ `0.0.0.0/0`
- TCP `443` từ `0.0.0.0/0`

### 4.2 Upstream (13.214.129.181)
Mở inbound:

- TCP `22` từ IP của bạn
- TCP `8000` **chỉ từ reverse proxy** (khuyến nghị)
  - Nếu làm nhanh cho lab, có thể mở `8000` từ `0.0.0.0/0` nhưng không an toàn.

> Gợi ý: nếu 2 EC2 cùng VPC, nên dùng **private IP** thay vì public IP khi reverse proxy gọi upstream.

---

## 5) Bước 1 — Cấu hình Cloudflare DNS cho domain

### 5.1 Add site và đổi Nameserver
1. Đăng nhập Cloudflare
2. Add site: `vscv.click`
3. Cloudflare cấp 2 nameserver
4. Vào trang quản lý domain (registrar) đổi Nameserver theo Cloudflare

Chờ DNS propagate (thường vài phút đến vài giờ).

### 5.2 Tạo DNS record
Vào **Cloudflare → DNS → Records**, tạo:

- A record:
  - **Name**: `@` (hoặc `vscv.click`)
  - **Content**: `13.251.110.182`
  - **Proxy**: **DNS only** (tắt mây cam)

- A record:
  - **Name**: `www`
  - **Content**: `13.251.110.182`
  - **Proxy**: **DNS only**

### 5.3 Kiểm tra DNS từ máy local (Windows)
PowerShell:

```powershell
nslookup vscv.click
nslookup -type=aaaa vscv.click
```

Kỳ vọng:
- A record trả về `13.251.110.182`
- Nếu có AAAA record mà bạn không dùng IPv6, nên xóa AAAA để tránh lỗi truy cập do ưu tiên IPv6.

---

## 6) Bước 2 — Cài Docker & Docker Compose (cả 2 máy)

Chạy trên **Upstream** và **Reverse Proxy**:

### 6.1 Cài Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

### 6.2 Cài docker-compose (v1)
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 6.3 Thêm user vào group docker (khuyến nghị)
```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Nếu vẫn chưa được, logout/login SSH lại rồi test:

```bash
docker ps
docker-compose --version
```

> Nếu gặp lỗi `PermissionError: [Errno 13] Permission denied` khi chạy docker/compose, gần như chắc chắn do chưa vào group docker (hoặc chưa dùng sudo).

---

## 7) Bước 3 — Triển khai WordPress trên Upstream (13.214.129.181)

### 7.1 Clone repo sample
```bash
git clone https://github.com/thaihust/docker-compose.git
cd docker-compose/wordpress-mysql
```

### 7.2 Start WordPress
```bash
sudo docker-compose up -d
```

### 7.3 Kiểm tra container
```bash
sudo docker ps
```

Bạn sẽ thấy dạng:
- `wordpress-mysql_wordpress_1` map `0.0.0.0:8000->80/tcp`
- `wordpress-mysql_db_1` MySQL

### 7.4 Kiểm tra upstream local
```bash
curl -I http://127.0.0.1:8000
```

### 7.5 Kiểm tra reverse proxy có thể gọi upstream
Từ máy reverse proxy:

```bash
curl -I http://13.214.129.181:8000
```

Nếu fail:
- kiểm tra Security Group upstream port 8000
- kiểm tra WordPress container có map port 8000
- kiểm tra route/network

---

## 8) Bước 4 — Lấy Cloudflare API key (để xin cert DNS-01)

Cloudflare Dashboard:
- My Profile → API Tokens → API Keys → **Global API Key** (View)

Bạn cần:
- **Cloudflare email**
- **Global API Key**

---

## 9) Bước 5 — Xin chứng chỉ Let’s Encrypt trên Reverse Proxy (DNS-01)

Chạy trên máy reverse proxy `13.251.110.182`.

### 9.1 Tạo file credentials (tạm thời)
> Thay email và key của bạn vào đây.

```bash
cat << EOF > /tmp/cloudflare.ini
dns_cloudflare_email = your-email@example.com
dns_cloudflare_api_key = your-cloudflare-global-api-key
EOF
```

### 9.2 Tạo thư mục lưu cert
```bash
sudo mkdir -p /etc/letsencrypt /var/lib/letsencrypt
```

### 9.3 Chạy certbot (dns-cloudflare)
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

### 9.4 Xóa file credentials
```bash
rm -f /tmp/cloudflare.ini
```

### 9.5 Kiểm tra cert tồn tại
```bash
sudo ls -la /etc/letsencrypt/live/vscv.click/
```

---

## 10) Bước 6 — Dựng Nginx reverse proxy bằng Docker Compose

### 10.1 Tạo thư mục cấu hình
```bash
sudo mkdir -p /opt/nginx/conf.d /opt/nginx/log
```

### 10.2 Tạo file docker-compose cho Nginx
> Lưu ý: **KHÔNG dùng `network_mode: host` chung với `ports:`** vì sẽ lỗi.

```bash
cat << EOF | sudo tee /opt/nginx/docker-compose.yaml
version: '3'

services:
  reverse-proxy:
    container_name: reverse-proxy
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

### 10.3 Start Nginx
```bash
sudo docker-compose -f /opt/nginx/docker-compose.yaml up -d
```

### 10.4 Kiểm tra container
```bash
sudo docker ps
```

Kỳ vọng có:
- `0.0.0.0:80->80/tcp`
- `0.0.0.0:443->443/tcp`

---

## 11) Bước 7 — Cấu hình Nginx reverse proxy cho domain

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
   ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";

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

     # Quan trọng: giúp WordPress biết request gốc là HTTPS
     proxy_set_header X-Forwarded-Proto https;

     proxy_set_header X-Forwarded-Host $host;
     proxy_set_header X-Forwarded-Port $server_port;
   }
}
EOF
```

---

## 12) Bước 8 — Reload Nginx và kiểm tra

### 12.1 Test config
```bash
sudo docker exec reverse-proxy nginx -t
```

### 12.2 Reload
```bash
sudo docker exec reverse-proxy nginx -s reload
```

### 12.3 Test từ reverse proxy
```bash
curl -I https://vscv.click
```

Kỳ vọng: `HTTP/2 200`

---

## 13) Bước 9 — Kiểm tra từ máy local (Windows)

### 13.1 Kiểm tra DNS
```powershell
nslookup vscv.click
```

Kỳ vọng: `13.251.110.182`

### 13.2 Truy cập website
Mở trình duyệt:
- `https://vscv.click`

> Nếu tab thường báo lỗi nhưng **ẩn danh vào được**, đó thường là do **cache / HSTS / DNS cache** trên trình duyệt.

### 13.3 Xử lý lỗi cache (nếu gặp)
- Mở ẩn danh: OK nhưng tab thường fail → xóa cache site
- Flush DNS Windows:

```powershell
ipconfig /flushdns
```

- Xóa dữ liệu site trên Chrome/Edge:
  - Settings → Privacy → Clear browsing data
  - hoặc xóa riêng site `vscv.click`

---

## 14) Bước 10 — Tự động gia hạn cert (cron)

### 14.1 Tạo script renew
> Thay email + API key của bạn.

```bash
cat << 'EOF' | sudo tee /usr/local/bin/renew-cert.sh
#!/bin/bash
set -e

cat << EOT > /tmp/cloudflare.ini
dns_cloudflare_email = your-email@example.com
dns_cloudflare_api_key = your-cloudflare-global-api-key
EOT

docker run --rm \
  -v "/etc/letsencrypt:/etc/letsencrypt" \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  -v "/tmp:/tmp" \
  certbot/dns-cloudflare renew \
  --dns-cloudflare \
  --dns-cloudflare-credentials /tmp/cloudflare.ini

rm -f /tmp/cloudflare.ini

docker exec reverse-proxy nginx -s reload
EOF

sudo chmod +x /usr/local/bin/renew-cert.sh
```

### 14.2 Thêm cronjob
```bash
crontab -e
```

Thêm dòng:

```cron
0 3 * * * /usr/local/bin/renew-cert.sh >> /var/log/renew-cert.log 2>&1
```

---

## 15) Troubleshooting (các lỗi đã gặp trong quá trình làm)

### 15.1 Lỗi Docker permission (`PermissionError: 13 Permission denied`)
**Nguyên nhân:** user không có quyền truy cập Docker socket.  
**Fix:**
```bash
sudo usermod -aG docker ubuntu
newgrp docker
# hoặc logout/login lại
```

### 15.2 Lỗi Docker Compose: `"host" network_mode is incompatible with port_bindings`
**Nguyên nhân:** dùng `network_mode: host` cùng với `ports:`.  
**Fix:** bỏ `network_mode: host` hoặc bỏ `ports:` (khuyến nghị bỏ `network_mode: host`).

### 15.3 WordPress redirect sai sang `:8000`
**Nguyên nhân:** WordPress hiểu URL site là có port 8000 hoặc thiếu header proxy.  
**Fix:**
- Đảm bảo Nginx có:
  - `proxy_set_header X-Forwarded-Proto https;`
- (Nếu vẫn bị) sửa `home/siteurl` trong DB về `https://vscv.click`

### 15.4 `docker-compose up -d` bị timeout
**Fix:**
```bash
export COMPOSE_HTTP_TIMEOUT=200
sudo -E docker-compose up -d
```

### 15.5 Domain không vào được nhưng IP vào được
Checklist:
- DNS A record đúng IP reverse proxy chưa?
- Có AAAA record sai không?
- Mở port 443 inbound SG chưa?
- Trình duyệt bị cache/HSTS không? (thử ẩn danh)

---

## 16) Kết quả cuối cùng
Sau khi hoàn tất:
- Domain `vscv.click` truy cập HTTPS được
- Nginx reverse proxy chạy public ở `13.251.110.182`
- WordPress chạy upstream ở `13.214.129.181:8000`
- TLS do Let’s Encrypt cấp qua Cloudflare DNS-01
- Có cron tự động renew certificate

---

## 17) Tài liệu tham khảo
- Nginx reverse proxy
- Let’s Encrypt (certbot)
- Cloudflare DNS API
- Docker / Docker Compose
