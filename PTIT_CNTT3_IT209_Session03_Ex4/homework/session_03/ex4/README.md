# Bài 4: Chạy song song nhiều cổng dịch vụ (Nginx Virtual Hosts)

## Mục tiêu

Cấu hình Nginx chạy song song hai trang web tĩnh độc lập trên cùng một Droplet thông qua hai cổng dịch vụ **8080** (Beta App) và **8090** (Internal App).

---

## Cấu trúc thư mục nộp bài

```
homework/session_03/ex4/
├── multi-port.conf          ← Cấu hình Nginx (chứa cả 2 server block)
├── beta-app/
│   └── index.html           ← Trang Beta App (cổng 8080)
├── internal-app/
│   └── index.html           ← Trang Internal App (cổng 8090)
└── README.md                ← Hướng dẫn triển khai
```

---

## Hướng dẫn triển khai trên Droplet

### Bước 1 – Tạo thư mục web root

```bash
sudo mkdir -p /var/www/beta-app/html
sudo mkdir -p /var/www/internal-app/html

# Cấp quyền sở hữu cho user hiện tại
sudo chown -R $USER:$USER /var/www/beta-app/html
sudo chown -R $USER:$USER /var/www/internal-app/html
```

### Bước 2 – Sao chép file HTML lên server

Từ máy local, sử dụng `scp` hoặc copy trực tiếp:

```bash
# Nếu dùng scp từ máy local:
scp beta-app/index.html    root@<IP_ADDRESS>:/var/www/beta-app/html/index.html
scp internal-app/index.html root@<IP_ADDRESS>:/var/www/internal-app/html/index.html

# Hoặc tạo trực tiếp trên server:
nano /var/www/beta-app/html/index.html
nano /var/www/internal-app/html/index.html
```

### Bước 3 – Cài đặt file cấu hình Nginx

```bash
# Copy file cấu hình vào sites-available
sudo cp multi-port.conf /etc/nginx/sites-available/multi-port.conf

# Tạo symbolic link để kích hoạt site
sudo ln -s /etc/nginx/sites-available/multi-port.conf \
           /etc/nginx/sites-enabled/multi-port.conf

# Kiểm tra cú pháp cấu hình
sudo nginx -t

# Reload Nginx để áp dụng
sudo systemctl reload nginx
```

### Bước 4 – Mở cổng trên UFW (tường lửa hệ thống)

```bash
sudo ufw allow 8080/tcp
sudo ufw allow 8090/tcp
sudo ufw status
```

Kết quả mong đợi:
```
Status: active
...
8080/tcp                   ALLOW       Anywhere
8090/tcp                   ALLOW       Anywhere
```

### Bước 5 – Mở cổng trên DigitalOcean Cloud Firewall

1. Đăng nhập **DigitalOcean** → **Networking** → **Firewalls**
2. Chọn Firewall đang áp dụng cho Droplet của bạn
3. Vào tab **Inbound Rules** → **Add Rule**:
   - **Type**: Custom | **Protocol**: TCP | **Port Range**: `8080` → Save
   - **Type**: Custom | **Protocol**: TCP | **Port Range**: `8090` → Save

---

## Kiểm tra kết quả

```bash
# Kiểm tra Beta App (cổng 8080)
curl http://<IP_ADDRESS_DROPLET>:8080

# Kiểm tra Internal App (cổng 8090)
curl http://<IP_ADDRESS_DROPLET>:8090
```

Hoặc mở trình duyệt:
- `http://<IP_ADDRESS_DROPLET>:8080` → Trang **Beta App** (nền tối vàng/cam)
- `http://<IP_ADDRESS_DROPLET>:8090` → Trang **Internal App** (nền tối xanh dương)

---

## Giải thích cấu hình

| Thành phần | Mô tả |
|---|---|
| `listen 8080;` | Nginx lắng nghe kết nối TCP trên cổng 8080 |
| `listen 8090;` | Nginx lắng nghe kết nối TCP trên cổng 8090 |
| `server_name _;` | Nhận mọi request bất kể tên miền hay IP |
| `root /var/www/beta-app/html;` | Thư mục gốc phục vụ file tĩnh cho cổng 8080 |
| `root /var/www/internal-app/html;` | Thư mục gốc phục vụ file tĩnh cho cổng 8090 |
| `try_files $uri $uri/ =404;` | Tìm file → tìm thư mục → trả 404 nếu không có |
| `access_log / error_log` | Log riêng biệt cho từng virtual host |

---

## Troubleshooting

```bash
# Kiểm tra Nginx có đang chạy không
sudo systemctl status nginx

# Xem log lỗi nếu nginx -t báo lỗi
sudo journalctl -xe | grep nginx

# Kiểm tra các cổng đang lắng nghe
sudo ss -tlnp | grep nginx

# Xem log truy cập
tail -f /var/log/nginx/beta-app.access.log
tail -f /var/log/nginx/internal-app.access.log
```
