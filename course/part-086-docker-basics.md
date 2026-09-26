# Part 086: Docker เบื้องต้นสำหรับ Django

> **ขั้นตอนที่ 851-860** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 851: Docker คืออะไร

### 851.1 แนวคิดพื้นฐาน

**Docker** คือเครื่องมือที่ทำให้ application "บรรจุ" (containerize) ไว้พร้อม dependencies ทั้งหมด ทำให้รันได้เหมือนกันทุก environment

| คำศัพท์ | ความหมาย |
|---------|----------|
| **Image** | Blueprint ของ container (เหมือน class) |
| **Container** | Instance ที่กำลังรันอยู่จาก image (เหมือน object) |
| **Dockerfile** | สูตรสร้าง image |
| **Registry** | ที่เก็บ images (Docker Hub, ECR, GCR) |
| **Volume** | พื้นที่เก็บข้อมูลที่ persist ได้ |

```bash
# ติดตั้ง Docker
# Ubuntu:
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# ตรวจสอบ
docker --version
docker run hello-world
```

---

## ขั้นตอนที่ 852: Dockerfile สำหรับ Django

### 852.1 Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
FROM python:3.12-slim

# ตั้งค่า environment
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Working directory
WORKDIR /app

# ติดตั้ง system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้ง Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy source code
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

# สร้าง non-root user
RUN useradd --create-home appuser && chown -R appuser /app
USER appuser

EXPOSE 8000

CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3"]
```

### 852.2 Multi-stage Build (ประหยัด image size)

```dockerfile
# Dockerfile.prod
# Stage 1: Builder
FROM python:3.12-slim AS builder

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip wheel --no-deps --wheel-dir /wheels -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim AS runtime

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && rm -rf /var/lib/apt/lists/*

COPY --from=builder /wheels /wheels
RUN pip install --no-index --find-links=/wheels /wheels/*.whl

COPY . .
RUN python manage.py collectstatic --noinput

RUN useradd --create-home appuser && chown -R appuser /app
USER appuser

EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

## ขั้นตอนที่ 853: .dockerignore

### 853.1 ไม่ copy ไฟล์ที่ไม่จำเป็น

```dockerignore
# .dockerignore

# Git
.git/
.gitignore

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/

# Virtual environments
.venv/
venv/
env/

# Local development
.env
.env.*
!.env.example
db.sqlite3
media/

# Testing
.pytest_cache/
.coverage
htmlcov/

# Documentation
docs/
*.md

# IDE
.vscode/
.idea/

# CI/CD
.github/
Makefile
```

---

## ขั้นตอนที่ 854: Build และ Run Container

### 854.1 คำสั่งพื้นฐาน

```bash
# Build image
docker build -t myapp:latest .
docker build -t myapp:1.0.0 -f Dockerfile.prod .

# Run container
docker run -p 8000:8000 myapp:latest

# Run พร้อม environment variables
docker run -p 8000:8000 \
  -e SECRET_KEY=mysecretkey \
  -e DATABASE_URL=postgres://user:pass@host/db \
  myapp:latest

# Run แบบ detached (background)
docker run -d --name myapp -p 8000:8000 myapp:latest

# ดู logs
docker logs myapp
docker logs -f myapp  # follow

# เข้าไปใน container
docker exec -it myapp bash

# หยุดและลบ
docker stop myapp
docker rm myapp
```

### 854.2 ตรวจสอบ Container

```bash
# รายการ containers ที่รันอยู่
docker ps
docker ps -a  # ทั้งหมดรวม stopped

# ตรวจสอบ resource usage
docker stats

# ดูรายละเอียด container
docker inspect myapp

# ดู image layers
docker history myapp:latest

# Image size
docker images myapp
```

---

## ขั้นตอนที่ 855: Volume และ Data Persistence

### 855.1 Bind Mount สำหรับ Development

```bash
# Mount source code จาก host (hot reload)
docker run -p 8000:8000 \
  -v $(pwd):/app \
  -e DEBUG=True \
  myapp:latest \
  python manage.py runserver 0.0.0.0:8000
```

### 855.2 Named Volume สำหรับ Production

```bash
# สร้าง named volume
docker volume create myapp_media

# ใช้ volume
docker run -p 8000:8000 \
  -v myapp_media:/app/media \
  myapp:latest

# ดูรายการ volumes
docker volume ls

# ดูรายละเอียด
docker volume inspect myapp_media
```

---

## ขั้นตอนที่ 856: Django settings สำหรับ Docker

### 856.1 Settings ที่ต้องปรับ

```python
# config/settings/docker.py
from .base import *
import os

DEBUG = os.environ.get('DEBUG', 'False') == 'True'
SECRET_KEY = os.environ['SECRET_KEY']
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '*').split(',')

# Database
import dj_database_url
DATABASES = {
    'default': dj_database_url.config(
        default=os.environ.get('DATABASE_URL'),
        conn_max_age=600,
        conn_health_checks=True,
    )
}

# Redis
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://redis:6379/0'),
    }
}

# Static files
STATIC_ROOT = '/app/staticfiles'
MEDIA_ROOT = '/app/media'
```

### 856.2 entrypoint.sh สำหรับ startup tasks

```bash
#!/bin/bash
# entrypoint.sh
set -e

echo "Waiting for database..."
while ! python -c "
import os, django
os.environ['DJANGO_SETTINGS_MODULE'] = 'config.settings.docker'
django.setup()
from django.db import connection
connection.ensure_connection()
" 2>/dev/null; do
  sleep 1
done

echo "Running migrations..."
python manage.py migrate --noinput

echo "Starting application..."
exec "$@"
```

```dockerfile
# Dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

## ขั้นตอนที่ 857: Docker สำหรับ Development

### 857.1 Dockerfile.dev

```dockerfile
# Dockerfile.dev
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt requirements-dev.txt ./
RUN pip install -r requirements.txt -r requirements-dev.txt

# ไม่ copy source code — mount จาก host แทน

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

---

## ขั้นตอนที่ 858: Image Registry

### 858.1 Push ไป Docker Hub

```bash
# Login
docker login

# Tag image
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest username/myapp:1.0.0

# Push
docker push username/myapp:latest
docker push username/myapp:1.0.0

# Pull
docker pull username/myapp:latest
```

### 858.2 GitHub Container Registry (GHCR)

```bash
# Login ด้วย GitHub token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag
docker tag myapp:latest ghcr.io/username/myapp:latest

# Push
docker push ghcr.io/username/myapp:latest
```

---

## ขั้นตอนที่ 859: Security Best Practices

### 859.1 Checklist

```dockerfile
# ✅ ใช้ specific tag แทน latest
FROM python:3.12.1-slim

# ✅ Non-root user
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# ✅ ไม่ใช้ ADD ถ้าไม่จำเป็น — ใช้ COPY แทน
COPY requirements.txt .

# ✅ READ-ONLY filesystem
# docker run --read-only --tmpfs /tmp myapp:latest
```

```bash
# Scan image หา vulnerabilities
docker scout cves myapp:latest

# หรือใช้ trivy
trivy image myapp:latest
```

---

## ขั้นตอนที่ 860: สรุปและแบบฝึกหัด

### 860.1 Docker Cheat Sheet

```bash
# Build
docker build -t myapp:latest .

# Run
docker run -d -p 8000:8000 --name myapp myapp:latest

# Logs
docker logs -f myapp

# Shell
docker exec -it myapp bash

# Stop/Remove
docker stop myapp && docker rm myapp

# Cleanup
docker system prune -a
```

### 860.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: สร้าง Dockerfile สำหรับโปรเจกต์ Django ของคุณ รัน `docker build` ให้สำเร็จ และทดสอบว่า `docker run` รันได้โดยไม่มี error

**แบบฝึกหัดที่ 2**: เปลี่ยนเป็น multi-stage build เปรียบเทียบ image size ก่อนและหลัง

**แบบฝึกหัดที่ 3**: สร้าง `entrypoint.sh` ที่รอ database พร้อมก่อน migrate แล้วจึงเริ่ม server
