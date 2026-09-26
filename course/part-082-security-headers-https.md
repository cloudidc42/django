# Part 082: Security Headers และ HTTPS

> **ขั้นตอนที่ 811-820** | Phase 10: Security

---

## ขั้นตอนที่ 811: Django SECURE_* Settings

### 811.1 การตั้งค่า Security ใน Production

```python
# config/settings/production.py

# บังคับ HTTPS
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS — บอก browser ให้ใช้ HTTPS เท่านั้น
SECURE_HSTS_SECONDS = 31536000        # 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Cookie ส่งผ่าน HTTPS เท่านั้น
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
SESSION_COOKIE_AGE = 1209600          # 2 สัปดาห์

# Clickjacking protection
X_FRAME_OPTIONS = 'DENY'

# ป้องกัน MIME-type sniffing
SECURE_CONTENT_TYPE_NOSNIFF = True

# Referrer Policy
SECURE_REFERRER_POLICY = 'strict-origin-when-cross-origin'
```

### 811.2 ตรวจสอบด้วย `check --deploy`

```bash
python manage.py check --deploy
```

ถ้าตั้งค่าครบจะไม่มี WARNING ใดเลย

---

## ขั้นตอนที่ 812: HTTP Security Headers

### 812.1 Headers ที่สำคัญ

| Header | หน้าที่ | ค่าที่แนะนำ |
|--------|---------|------------|
| `Strict-Transport-Security` | บังคับ HTTPS | `max-age=31536000; includeSubDomains` |
| `X-Frame-Options` | ป้องกัน Clickjacking | `DENY` หรือ `SAMEORIGIN` |
| `X-Content-Type-Options` | ป้องกัน MIME sniffing | `nosniff` |
| `Content-Security-Policy` | ควบคุมแหล่งที่มาของ resources | ดูข้างล่าง |
| `Referrer-Policy` | ควบคุม Referer header | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | ปิด browser features ที่ไม่ใช้ | `camera=(), microphone=()` |

### 812.2 django-csp สำหรับ Content Security Policy

```bash
pip install django-csp
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'csp',
]

MIDDLEWARE = [
    ...
    'csp.middleware.CSPMiddleware',
]

# CSP Policy
CSP_DEFAULT_SRC = ("'self'",)
CSP_STYLE_SRC = ("'self'", "https://fonts.googleapis.com")
CSP_FONT_SRC = ("'self'", "https://fonts.gstatic.com")
CSP_SCRIPT_SRC = ("'self'",)
CSP_IMG_SRC = ("'self'", "data:", "https:")
CSP_CONNECT_SRC = ("'self'",)
CSP_FRAME_ANCESTORS = ("'none'",)
CSP_REPORT_URI = '/csp-report/'
```

```python
# views.py — ใช้ decorator เมื่อต้องการ inline script
from csp.decorators import csp_update

@csp_update(SCRIPT_SRC=["'unsafe-inline'"])
def legacy_view(request):
    # view ที่ยังต้องใช้ inline script
    ...
```

---

## ขั้นตอนที่ 813: HTTPS ด้วย Let's Encrypt และ Certbot

### 813.1 ติดตั้ง Certificate บน Ubuntu

```bash
# ติดตั้ง Certbot
sudo apt install certbot python3-certbot-nginx

# ขอ Certificate (แทนที่ example.com ด้วย domain จริง)
sudo certbot --nginx -d example.com -d www.example.com

# ต่ออายุอัตโนมัติ
sudo systemctl enable certbot.timer
sudo certbot renew --dry-run
```

### 813.2 Nginx config หลัง Certbot

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/myapp/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /var/www/myapp/media/;
    }
}
```

---

## ขั้นตอนที่ 814: Custom Security Middleware

### 814.1 เพิ่ม Headers ที่ Django ไม่มีในตัว

```python
# apps/core/middleware.py


class SecurityHeadersMiddleware:
    """เพิ่ม security headers ที่ Django ยังไม่รองรับ built-in"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        response['Permissions-Policy'] = (
            'camera=(), microphone=(), geolocation=(), '
            'payment=(), usb=(), magnetometer=()'
        )
        response['Cross-Origin-Opener-Policy'] = 'same-origin'
        response['Cross-Origin-Embedder-Policy'] = 'require-corp'
        response['Cross-Origin-Resource-Policy'] = 'same-origin'

        return response
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'apps.core.middleware.SecurityHeadersMiddleware',
    ...
]
```

---

## ขั้นตอนที่ 815: SSL/TLS Configuration

### 815.1 ค่า SSL ที่แนะนำสำหรับ Nginx

```nginx
# /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DHE+AES128:!ADH:!AECDH:!MD5;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
```

### 815.2 ทดสอบ SSL Rating ด้วย SSL Labs

```bash
# ใช้ curl ทดสอบ headers
curl -I https://example.com

# ทดสอบ HSTS
curl -I https://example.com | grep -i strict

# ตรวจสอบ certificate expiry
echo | openssl s_client -servername example.com \
  -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates
```

---

## ขั้นตอนที่ 816: Subresource Integrity (SRI)

### 816.1 ป้องกัน CDN ถูก Tamper

```html
<!-- templates/base.html -->
<!-- ใช้ SRI hash เพื่อตรวจสอบ integrity ของ external scripts -->
<link rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css"
      integrity="sha384-9ndCyUaIbzAi2FUVXJi0CjmCapSmO7SnpJef0486qhLnuZ2cdeRhO02iuK6FUUVM"
      crossorigin="anonymous">

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-geWF76RCwLtnZ8qwWowPQNguL3RmwHVBC9FhGdlKrxdiJJigb/j/68SIy3Te4Bkz"
        crossorigin="anonymous"></script>
```

```python
# generate_sri.py — สร้าง SRI hash
import hashlib, base64, urllib.request

def get_sri_hash(url):
    with urllib.request.urlopen(url) as r:
        content = r.read()
    digest = hashlib.sha384(content).digest()
    return 'sha384-' + base64.b64encode(digest).decode()
```

---

## ขั้นตอนที่ 817: Cookie Security เพิ่มเติม

### 817.1 SameSite Cookie

```python
# settings.py
SESSION_COOKIE_SAMESITE = 'Lax'    # หรือ 'Strict'
CSRF_COOKIE_SAMESITE = 'Lax'

# ป้องกัน session fixation
SESSION_COOKIE_NAME = 'sessionid'   # เปลี่ยนชื่อถ้าต้องการ obscurity
```

### 817.2 Custom Session Backend ที่ Rotate Session ID

```python
# apps/accounts/views.py
from django.contrib.auth import login, authenticate
from django.middleware.csrf import rotate_token


def secure_login(request):
    if request.method == 'POST':
        user = authenticate(request, **request.POST)
        if user:
            # สร้าง session ID ใหม่หลัง login (ป้องกัน session fixation)
            request.session.cycle_key()
            rotate_token(request)
            login(request, user)
```

---

## ขั้นตอนที่ 818: CORS Configuration

### 818.1 django-cors-headers

```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # ต้องอยู่ก่อน CommonMiddleware
    'django.middleware.common.CommonMiddleware',
    ...
]

# อนุญาตเฉพาะ domain ที่รู้จัก
CORS_ALLOWED_ORIGINS = [
    'https://app.example.com',
    'https://admin.example.com',
]

# ห้ามใช้ CORS_ALLOW_ALL_ORIGINS = True ใน production
CORS_ALLOW_CREDENTIALS = True

CORS_ALLOW_METHODS = [
    'DELETE', 'GET', 'OPTIONS', 'PATCH', 'POST', 'PUT',
]

CORS_ALLOW_HEADERS = [
    'accept', 'authorization', 'content-type', 'x-csrftoken',
]
```

---

## ขั้นตอนที่ 819: Security Headers Testing

### 819.1 Unit Test สำหรับ Headers

```python
# tests/test_security_headers.py
import pytest
from django.test import Client


@pytest.mark.django_db
class TestSecurityHeaders:
    def setup_method(self):
        self.client = Client(enforce_csrf_checks=False)

    def test_x_frame_options(self):
        response = self.client.get('/')
        assert response['X-Frame-Options'] == 'DENY'

    def test_content_type_nosniff(self):
        response = self.client.get('/')
        assert response['X-Content-Type-Options'] == 'nosniff'

    def test_no_server_header_exposed(self):
        response = self.client.get('/')
        assert 'Server' not in response or 'nginx' not in response.get('Server', '')

    def test_csp_present(self):
        response = self.client.get('/')
        assert 'Content-Security-Policy' in response
```

### 819.2 Integration Test ด้วย SecurityHeaders.com

```bash
# ใช้ curl ตรวจสอบทุก header ที่สำคัญ
curl -s -I https://example.com | grep -iE \
  "(strict-transport|x-frame|x-content-type|content-security|referrer|permissions)"
```

---

## ขั้นตอนที่ 820: สรุปและแบบฝึกหัด

### 820.1 Security Headers Checklist

- [x] `SECURE_SSL_REDIRECT = True`
- [x] `SECURE_HSTS_SECONDS = 31536000`
- [x] `SESSION_COOKIE_SECURE = True`
- [x] `CSRF_COOKIE_SECURE = True`
- [x] `X_FRAME_OPTIONS = 'DENY'`
- [x] `SECURE_CONTENT_TYPE_NOSNIFF = True`
- [x] `SECURE_REFERRER_POLICY` ตั้งค่าแล้ว
- [x] CSP middleware ติดตั้งแล้ว
- [x] Nginx ส่ง HSTS header
- [x] SSL certificate ยังไม่หมดอายุ

### 820.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: เพิ่ม `Permissions-Policy` ปิด camera และ microphone ผ่าน custom middleware

**แบบฝึกหัดที่ 2**: ตั้งค่า CSP สำหรับโปรเจกต์ที่ใช้ Bootstrap CDN + Google Fonts โดยไม่ใช้ `unsafe-inline`

**แบบฝึกหัดที่ 3**: เขียน test ตรวจสอบว่าทุก response มี security headers ครบ
