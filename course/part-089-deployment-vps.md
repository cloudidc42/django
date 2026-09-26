# Part 089: Deployment บน VPS ด้วย Gunicorn และ Nginx

> **ขั้นตอนที่ 881-890** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 881: เตรียม VPS

### 881.1 ตั้งค่า Server เบื้องต้น (Ubuntu 22.04)

```bash
# 1. อัพเดต packages
sudo apt update && sudo apt upgrade -y

# 2. ติดตั้ง dependencies
sudo apt install -y python3.12 python3.12-venv python3-pip \
    postgresql postgresql-contrib nginx certbot python3-certbot-nginx \
    redis-server supervisor git curl

# 3. สร้าง user สำหรับ app
sudo useradd --create-home --shell /bin/bash deploy
sudo usermod -aG www-data deploy

# 4. ตั้งค่า firewall
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable

# 5. SSH key authentication (ปิด password auth)
# เพิ่ม public key ใน /home/deploy/.ssh/authorized_keys
sudo mkdir -p /home/deploy/.ssh
sudo chown deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
```

---

## ขั้นตอนที่ 882: ตั้งค่า PostgreSQL

### 882.1 สร้าง Database และ User

```bash
sudo -u postgres psql

-- สร้าง user
CREATE USER myapp WITH PASSWORD 'strongpassword';

-- สร้าง database
CREATE DATABASE myapp_production OWNER myapp;

-- ให้สิทธิ์
GRANT ALL PRIVILEGES ON DATABASE myapp_production TO myapp;

-- ออก
\q
```

```bash
# ทดสอบ connection
psql -U myapp -d myapp_production -h localhost
```

---

## ขั้นตอนที่ 883: Deploy Application

### 883.1 โครงสร้าง Directory

```
/opt/myapp/
├── app/           # source code
├── venv/          # virtual environment
├── logs/          # application logs
└── media/         # uploaded files
```

```bash
# สร้าง directory structure
sudo mkdir -p /opt/myapp/{app,logs,media}
sudo chown -R deploy:deploy /opt/myapp

# Switch to deploy user
sudo su - deploy

# Clone repository
cd /opt/myapp
git clone git@github.com:username/myapp.git app
cd app

# สร้าง virtual environment
python3.12 -m venv /opt/myapp/venv
source /opt/myapp/venv/bin/activate

# ติดตั้ง dependencies
pip install -r requirements.txt
pip install gunicorn
```

### 883.2 Environment Variables

```bash
# สร้าง .env file
cat > /opt/myapp/app/.env << 'EOF'
DEBUG=False
SECRET_KEY=your-very-long-secret-key-here
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgres://myapp:strongpassword@localhost:5432/myapp_production
REDIS_URL=redis://localhost:6379/0
STATIC_ROOT=/opt/myapp/app/staticfiles
MEDIA_ROOT=/opt/myapp/media
EOF
chmod 600 /opt/myapp/app/.env
```

```bash
# รัน migrations และ collectstatic
cd /opt/myapp/app
source /opt/myapp/venv/bin/activate

python manage.py migrate --noinput
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

---

## ขั้นตอนที่ 884: Gunicorn Configuration

### 884.1 gunicorn.conf.py

```python
# /opt/myapp/app/gunicorn.conf.py
import multiprocessing

# Socket
bind = 'unix:/opt/myapp/gunicorn.sock'

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'gthread'
threads = 2
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50

# Timeouts
timeout = 30
graceful_timeout = 30
keepalive = 5

# Logging
accesslog = '/opt/myapp/logs/gunicorn-access.log'
errorlog = '/opt/myapp/logs/gunicorn-error.log'
loglevel = 'warning'

# Process naming
proc_name = 'myapp'

# Security
limit_request_line = 4094
limit_request_fields = 100
limit_request_field_size = 8190
```

---

## ขั้นตอนที่ 885: Systemd Service

### 885.1 gunicorn.service

```ini
# /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn daemon for MyApp
After=network.target postgresql.service redis.service
Requires=gunicorn.socket

[Service]
Type=notify
User=deploy
Group=www-data
WorkingDirectory=/opt/myapp/app
ExecStart=/opt/myapp/venv/bin/gunicorn \
    --config /opt/myapp/app/gunicorn.conf.py \
    config.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/gunicorn.socket
[Unit]
Description=Gunicorn socket

[Socket]
ListenStream=/opt/myapp/gunicorn.sock
SocketUser=www-data

[Install]
WantedBy=sockets.target
```

```bash
# เปิดใช้งาน
sudo systemctl daemon-reload
sudo systemctl enable gunicorn.socket gunicorn.service
sudo systemctl start gunicorn.socket
sudo systemctl status gunicorn
```

---

## ขั้นตอนที่ 886: Nginx Configuration

### 886.1 /etc/nginx/sites-available/myapp

```nginx
# /etc/nginx/sites-available/myapp
upstream gunicorn {
    server unix:/opt/myapp/gunicorn.sock fail_timeout=0;
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL (จัดการโดย Certbot)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    client_max_body_size 10M;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;

    location /static/ {
        alias /opt/myapp/app/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location /media/ {
        alias /opt/myapp/media/;
        expires 30d;
    }

    location / {
        proxy_pass http://gunicorn;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_buffering off;
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# ขอ SSL certificate
sudo certbot --nginx -d example.com -d www.example.com
```

---

## ขั้นตอนที่ 887: Celery ด้วย Supervisor

### 887.1 Supervisor config

```ini
# /etc/supervisor/conf.d/celery.conf
[program:celery_worker]
command=/opt/myapp/venv/bin/celery -A config worker -l warning -c 4
directory=/opt/myapp/app
user=deploy
autostart=true
autorestart=true
stdout_logfile=/opt/myapp/logs/celery-worker.log
stderr_logfile=/opt/myapp/logs/celery-worker-error.log
environment=DJANGO_SETTINGS_MODULE="config.settings.production"

[program:celery_beat]
command=/opt/myapp/venv/bin/celery -A config beat -l warning --scheduler django_celery_beat.schedulers:DatabaseScheduler
directory=/opt/myapp/app
user=deploy
autostart=true
autorestart=true
stdout_logfile=/opt/myapp/logs/celery-beat.log
stderr_logfile=/opt/myapp/logs/celery-beat-error.log
environment=DJANGO_SETTINGS_MODULE="config.settings.production"
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

---

## ขั้นตอนที่ 888: Deploy Script

### 888.1 scripts/deploy.sh

```bash
#!/bin/bash
# scripts/deploy.sh — รันบน server หลัง git pull

set -e

APP_DIR="/opt/myapp/app"
VENV="/opt/myapp/venv"

echo "=== Starting deployment ==="

# Pull latest code
cd "$APP_DIR"
git pull origin main

# Activate venv
source "$VENV/bin/activate"

# Install/update dependencies
pip install -r requirements.txt -q

# Migrations
echo "Running migrations..."
python manage.py migrate --noinput

# Collect static files
echo "Collecting static files..."
python manage.py collectstatic --noinput -v 0

# Django checks
python manage.py check --deploy

# Restart application
echo "Restarting services..."
sudo systemctl restart gunicorn
sudo supervisorctl restart celery_worker celery_beat

echo "=== Deployment complete ==="
```

```bash
chmod +x scripts/deploy.sh
# อนุญาต deploy user ใช้ systemctl restart gunicorn โดยไม่ต้องใส่ password
# /etc/sudoers.d/deploy:
# deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart gunicorn
```

---

## ขั้นตอนที่ 889: Log Rotation

### 889.1 Logrotate สำหรับ Application Logs

```
# /etc/logrotate.d/myapp
/opt/myapp/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        systemctl reload gunicorn > /dev/null 2>&1 || true
        supervisorctl signal HUP celery_worker > /dev/null 2>&1 || true
    endscript
}
```

---

## ขั้นตอนที่ 890: สรุปและแบบฝึกหัด

### 890.1 Production Deployment Checklist

- [x] VPS อัพเดตและ firewall ตั้งค่าแล้ว
- [x] Non-root user สำหรับ application
- [x] PostgreSQL database + user สร้างแล้ว
- [x] `.env` file อยู่นอก git และ permissions 600
- [x] Gunicorn socket (ไม่ใช้ port) + systemd service
- [x] Nginx reverse proxy + SSL certificate
- [x] `SECURE_SSL_REDIRECT = True` ใน settings
- [x] Celery + Celery Beat ด้วย Supervisor
- [x] Log rotation ตั้งค่าแล้ว
- [x] Deploy script ทำงานได้

### 890.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: Deploy Django app บน VPS จริง (หรือ VM) ตาม steps ข้างต้นจนเข้าถึงได้ผ่าน HTTPS

**แบบฝึกหัดที่ 2**: ตั้งค่า Celery worker + beat ด้วย Supervisor และทดสอบว่า task ทำงานได้

**แบบฝึกหัดที่ 3**: สร้าง deploy script ที่ zero-downtime ด้วย Gunicorn graceful reload (`kill -HUP`)
