# -Reverse-Proxy-Public-v-i-TLS-cho-Domain
 Reverse Proxy Public với TLS cho Domain

# -Thành phần
Domain: vscv.click
Cloudflare DNS: quản lý bản ghi DNS và cấp API cho DNS challenge
Reverse Proxy Server: 13.251.110.182
Upstream Server: 13.214.129.181
Let’s Encrypt: cấp chứng chỉ SSL/TLS miễn phí

# -Lab variables
DOMAIN=vscv.click
REVERSE_PROXY_IP=13.251.110.182
UPSTREAM_IP=13.214.129.181
APP_PORT=8000

Bước 1: Cấu hình DNS cho domain
1.1 Thêm domain vào Cloudflare
Đăng nhập Cloudflare
Chọn Add site
Nhập domain của bạn
Chọn gói Free nếu chỉ dùng lab
1.2 Đổi nameserver
Cloudflare sẽ cung cấp 2 nameserver
Vào nhà đăng ký domain
Đổi nameserver sang 2 giá trị Cloudflare cung cấp
1.3 Tạo bản ghi DNS
Tạo các bản ghi sau:

Type	Name	Content	Proxy
A	@ hoặc vscv.click	13.251.110.182	DNS only
