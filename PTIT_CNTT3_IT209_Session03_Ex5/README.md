# Bài 5: Thiết lập thư mục Web an toàn tại phân vùng hệ thống (`/opt/`)

## 1. Mục tiêu

- Cô lập mã nguồn web tĩnh và logs **ra khỏi phân vùng mặc định** của hệ điều hành (`/var/www`, `/var/log/nginx`), đưa về phân vùng add-on `/opt/`.
- Phân quyền tệp tin/thư mục nâng cao, tách vai trò rõ ràng:
  - user thường **`devops`** (lập trình viên) toàn quyền đọc/ghi mã nguồn **không cần `sudo`**;
  - tiến trình **Nginx** chạy dưới **`www-data`** đọc được web (không `403`) và **ghi được log** vào thư mục riêng.
- Cấu hình Server Block Nginx trỏ `root` vào `/opt/my-app/html/` và ghi `access_log` / `error_log` trực tiếp vào `/opt/my-app/logs/`.

## 2. Môi trường thực hiện

| Thành phần | Giá trị |
|---|---|
| Hệ điều hành | Ubuntu (Linux) |
| Web server | Nginx 1.24.0 |
| User chạy Nginx | `www-data` (uid=33, gid=33) |
| User chỉnh sửa code | `devops` (uid=1001, gid=1001) |
| Thư mục dự án | `/opt/my-app/` |
| Web root | `/opt/my-app/html/` |
| Log dir | `/opt/my-app/logs/` |

## 3. Cấu trúc thư mục đích

```
/opt/my-app/
├── html/                 # web root (mã nguồn tĩnh)
│   └── index.html
└── logs/                 # log riêng của dự án
    ├── access.log
    └── error.log
```

Sở hữu và quyền mục tiêu:

| Đối tượng | Owner | Group | Quyền | Ghi chú |
|---|---|---|---|---|
| `/opt/my-app` | `devops` | `www-data` | `750` | `drwxr-x---` |
| `/opt/my-app/html` | `devops` | `www-data` | `750` | group cần `x` để "đi qua" |
| `/opt/my-app/html/index.html` | `devops` | `www-data` | `640` | group chỉ đọc |
| `/opt/my-app/logs` | `devops` | `www-data` | `775` | group **ghi được** log |
| `/opt/my-app/logs/*.log` | `devops` | `www-data` | `664` | group ghi được |

## 4. Thực hiện phân quyền

Thiết lập thư mục và phân quyền (chạy bằng `sudo`):

```bash
APP_ROOT=/opt/my-app
APP_USER=devops
WEB_GROUP=www-data

# 1. Tạo cây thư mục
sudo mkdir -p "$APP_ROOT/html" "$APP_ROOT/logs"

# 2. Chuyển quyền sở hữu đệ quy về devops:www-data
sudo chown -R "$APP_USER:$WEB_GROUP" "$APP_ROOT"

# 3. Thư mục 750, tệp tin 640
sudo find "$APP_ROOT" -type d -exec chmod 750 {} \;
sudo find "$APP_ROOT" -type f -exec chmod 640 {} \;

# 4. Cấp quyền ghi log cho group www-data trên logs/ (775)
sudo chmod 775 "$APP_ROOT/logs"

# 5. Tạo sẵn file log và phân quyền 664 (owner + group ghi được)
sudo touch "$APP_ROOT/logs/access.log" "$APP_ROOT/logs/error.log"
sudo chown "$APP_USER:$WEB_GROUP" "$APP_ROOT/logs/"*.log
sudo chmod 664 "$APP_ROOT/logs/"*.log
```

Tạo web tĩnh trong web root rồi phân quyền:

```bash
sudo tee /opt/my-app/html/index.html >/dev/null <<'HTML'
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>my-app - /opt/my-app/html</title>
</head>
<body>
    <h1>my-app</h1>
    <p>Static site served from <code>/opt/my-app/html/</code></p>
</body>
</html>
HTML

sudo chown devops:www-data /opt/my-app/html/index.html
sudo chmod 640 /opt/my-app/html/index.html
```

> **Vì sao tách `-type d` (750) và `-type f` (640) thay vì `chmod -R 750`?**
> `chmod -R 750` sẽ gán luôn bit thực thi (`x`) cho mọi **file**, biến các file `.html/.css/.js`
> thành file thực thi — sai ngữ nghĩa và kém an toàn. Cách viết tương đương bằng một lệnh
> (dùng `X` hoa = chỉ thêm `x` cho thư mục):
> `sudo chmod -R u=rwX,g=rX,o= /opt/my-app`

## 5. Cấu hình Server Block Nginx

Tệp [`nginx/my-app.conf`](./nginx/my-app.conf) — triển khai thành `/etc/nginx/sites-available/my-app`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name my-app.local;

    # Mã nguồn web tĩnh nằm ngoài phân vùng mặc định (/var/www)
    root /opt/my-app/html;
    index index.html index.htm;

    # Log riêng của dự án, tách khỏi /var/log/nginx/
    access_log /opt/my-app/logs/access.log;
    error_log  /opt/my-app/logs/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Kích hoạt:

```bash
sudo cp nginx/my-app.conf /etc/nginx/sites-available/my-app
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/my-app
sudo nginx -t && sudo systemctl reload nginx
```

```console
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 6. Kiểm tra kết quả

### 6.1. Phân quyền — `ls -la /opt/my-app/`

```console
$ ls -la /opt/my-app/
total 16
drwxr-x--- 4 devops www-data 4096 Oct  1 08:36 .
drwxr-xr-x 3 root   root     4096 Oct  1 08:36 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 08:36 html
drwxrwxr-x 2 devops www-data 4096 Oct  1 08:36 logs

$ ls -la /opt/my-app/html/
total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 08:36 .
drwxr-x--- 4 devops www-data 4096 Oct  1 08:36 ..
-rw-r----- 1 devops www-data  272 Oct  1 08:36 index.html

$ ls -la /opt/my-app/logs/
total 12
drwxrwxr-x 2 devops www-data 4096 Oct  1 08:36 .
drwxr-x--- 4 devops www-data 4096 Oct  1 08:36 ..
-rw-rw-r-- 1 devops www-data  255 Oct  1 08:36 access.log
-rw-rw-r-- 1 devops www-data    0 Oct  1 08:36 error.log
```

- Toàn bộ cây `/opt/my-app/` thuộc sở hữu **`devops:www-data`** (đệ quy).
- Thư mục `drwxr-x---` (**750**), tệp `-rw-r-----` (**640**); riêng `logs/` là `drwxrwxr-x` (**775**).

### 6.2. Ghi file bằng user `devops` **không dùng `sudo`** — phải thành công

```console
$ su - devops -c 'whoami'
devops

$ su - devops -c 'echo "Update Test" >> /opt/my-app/html/index.html'
-> WRITE OK (exit 0)

$ tail -3 /opt/my-app/html/index.html
</body>
</html>
Update Test - Thu Oct  1 08:36:51 UTC 2026
```

→ `devops` ghi trực tiếp vào file **không cần `sudo`** nhờ bit `rw` của owner.

### 6.3. Nginx phục vụ web — HTTP 200, không lỗi `403`

```console
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" -H "Host: my-app.local" http://127.0.0.1/
HTTP 200

$ curl -s -H "Host: my-app.local" http://127.0.0.1/ | tail -3
</body>
</html>
Update Test - Thu Oct  1 08:36:51 UTC 2026
```

### 6.4. Nginx tự động ghi log vào `/opt/my-app/logs/`

```console
$ tail -f /opt/my-app/logs/access.log
127.0.0.1 - - [01/Oct/2026:08:36:50 +0000] "GET / HTTP/1.1" 200 229 "-" "curl/8.5.0"
127.0.0.1 - - [01/Oct/2026:08:36:50 +0000] "GET / HTTP/1.1" 200 229 "-" "curl/8.5.0"
127.0.0.1 - - [01/Oct/2026:08:36:55 +0000] "GET / HTTP/1.1" 200 272 "-" "curl/8.5.0"
```

```console
$ cat /opt/my-app/logs/error.log
(trống — không có lỗi 403/Permission denied)
```

→ Nginx (worker chạy dưới `www-data`) ghi được log nhờ group `www-data` có quyền ghi trên `logs/`.

```console
$ ps -o user,pid,cmd -C nginx
USER         PID CMD
root       3807 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  43027 nginx: worker process
www-data  43028 nginx: worker process
```

### 6.5. Người dùng khác (others) — không truy cập được mã nguồn

```console
$ id dun
uid=1000(dun) gid=1000(dun) groups=1000(dun),4(adm),...,27(sudo)   # KHÔNG thuộc www-data

$ su - dun -c 'cat /opt/my-app/html/index.html'
cat: /opt/my-app/html/index.html: Permission denied
```

→ Đúng thiết kế: `others` (`---`) bị chặn, mã nguồn được cô lập.

## 7. Giải thích cơ chế phân quyền

| Đối tượng | Vai trò | Quyền cần | Ý nghĩa |
|---|---|---|---|
| `devops` (owner) | Lập trình viên | `rw` trên file, `rwx` trên thư mục | Đọc + sửa + tạo/xóa file, không cần `sudo` |
| `www-data` (group) | Tiến trình Nginx | `r` trên file, `r-x` trên thư mục web, `rwx` trên `logs/` | Đọc web để phục vụ; ghi log vào `logs/` |
| others | Khách khác | không (`---`) | Không truy cập trực tiếp vào mã nguồn |

Điểm mấu chốt:
- **Chỉ cần đổi owner → `devops`** là giải quyết yêu cầu "sửa file không cần root".
- **Bit `x` trên thư mục** là điều kiện bắt buộc để Nginx truy cập vào trong; thiếu `x` sẽ gây `403 Forbidden` dù file có `r`.
- **Vì sao `logs/` phải khác (775/664 thay vì 750/640)?** Nginx cần **quyền ghi** trên thư mục logs.
  Group `www-data` được cấp `w` trên `logs/` (và trên file log `664`), nên Nginx ghi log được
  mà **không** phải mở quyền cho `others`. Giữ `others = ---` đảm bảo log nội bộ không bị đọc trái phép.
- **Cô lập phân vùng**: đặt tại `/opt/my-app/` (phân vùng add-on) tách khỏi `/var/www` và `/var/log/nginx`,
  hạn chế ảnh hưởng chéo và dễ sao lưu/quản lý theo dự án.

## 8. Kết luận

- Đã tạo và phân quyền `/opt/my-app/` với owner **`devops:www-data`** (đệ quy).
- Mã nguồn web tĩnh đặt tại `/opt/my-app/html/`; logs đặt tại `/opt/my-app/logs/`.
- **`devops` ghi file không cần `sudo`** (kiểm tra thành công, exit 0).
- **Nginx trả về HTTP 200, không có `403 Forbidden`**, tự động ghi `access.log`/`error.log`
  vào đúng thư mục mới.
- Mã nguồn và logs được **cô lập** khỏi phân vùng hệ thống; `others` bị chặn truy cập.

## Phụ lục — Danh sách tệp nộp bài

```
homework/session_03/ex5/
├── README.md              # báo cáo: log kiểm tra + trạng thái phân quyền
└── nginx/
    └── my-app.conf        # Server Block Nginx
```

### Tóm tắt lệnh đã dùng

```bash
# 1. Thiết lập thư mục + phân quyền (xem mục 4)

# 2. Kích hoạt Nginx
sudo cp nginx/my-app.conf /etc/nginx/sites-available/my-app
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/my-app
sudo nginx -t && sudo systemctl reload nginx

# 3. Kiểm tra (theo đề bài)
ls -la /opt/my-app/
su - devops -c 'echo "Update Test" >> /opt/my-app/html/index.html'
tail -f /opt/my-app/logs/access.log
curl -s -o /dev/null -w "HTTP %{http_code}\n" -H "Host: my-app.local" http://127.0.0.1/
```
