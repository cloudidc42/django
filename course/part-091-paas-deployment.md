# Part 091: Deployment บน PaaS: Railway, Render, Heroku

> **ขั้นตอนที่ 901-910** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 901: PaaS คืออะไร

### 901.1 Platform as a Service

**PaaS** ทำให้ deploy ง่ายขึ้นโดยไม่ต้องจัดการ server เอง ใช้ได้เลยจาก git push

| Platform | ราคา | Free Tier | ความง่าย |
|----------|------|-----------|----------|
| **Railway** | $5/month credit | ✅ 500 ชม/เดือน | ⭐⭐⭐⭐⭐ |
| **Render** | $7/เดือน | ✅ (sleep หลัง 15 นาที) | ⭐⭐⭐⭐ |
| **Heroku** | $5+/เดือน | ❌ (ยกเลิก free tier) | ⭐⭐⭐⭐ |
| **Fly.io** | Pay-per-use | ✅ 3 machines | ⭐⭐⭐ |

---

## ขั้นตอนที่ 902: Deploy ไป Railway

### 902.1 ตั้งค่าโปรเจกต์

```bash
# ติดตั้ง Railway CLI
npm install -g @railway/cli
# หรือ
curl -fsSL https://railway.app/install.sh | sh

# Login
railway login

# สร้าง project จาก directory
railway init
```

### 902.2 railway.toml

```toml
# railway.toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "gunicorn config.wsgi:application --bind 0.0.0.0:$PORT"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 5
```

### 902.3 Procfile (สำหรับ Railway แบบ Nixpacks)

```procfile
# Procfile
web: gunicorn config.wsgi:application --bind 0.0.0.0:$PORT
worker: celery -A config worker -l warning
beat: celery -A config beat -l warning --scheduler django_celery_beat.schedulers:DatabaseScheduler
release: python manage.py migrate --noinput
```

---

## ขั้นตอนที่ 903: Deploy ไป Render

### 903.1 render.yaml (Infrastructure as Code)

```yaml
# render.yaml
services:
  - type: web
    name: myapp
    env: python
    region: singapore
    plan: starter
    buildCommand: pip install -r requirements.txt && python manage.py collectstatic --noinput
    startCommand: gunicorn config.wsgi:application --bind 0.0.0.0:$PORT
    envVars:
      - key: DJANGO_SETTINGS_MODULE
        value: config.settings.production
      - key: SECRET_KEY
        generateValue: true
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          type: redis
          name: myapp-redis
          property: connectionString

  - type: worker
    name: celery-worker
    env: python
    plan: starter
    buildCommand: pip install -r requirements.txt
    startCommand: celery -A config worker -l warning
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString

databases:
  - name: myapp-db
    databaseName: myapp
    user: myapp
    plan: free

  - name: myapp-redis
    plan: free
```

### 903.2 Django Settings สำหรับ Render

```python
# config/settings/render.py
from .base import *
import os

DEBUG = False
SECRET_KEY = os.environ['SECRET_KEY']
ALLOWED_HOSTS = [os.environ.get('RENDER_EXTERNAL_HOSTNAME', '*')]

# WhiteNoise สำหรับ static files (ไม่ต้องใช้ S3)
MIDDLEWARE.insert(1, 'whitenoise.middleware.WhiteNoiseMiddleware')
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

---

## ขั้นตอนที่ 904: WhiteNoise สำหรับ Static Files บน PaaS

### 904.1 ติดตั้งและตั้งค่า

```bash
pip install whitenoise
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # ต้องอยู่หลัง SecurityMiddleware
    ...
]

STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

WhiteNoise จะ serve static files โดยตรงจาก Gunicorn โดยไม่ต้องใช้ Nginx หรือ S3

---

## ขั้นตอนที่ 905: Database URL กับ dj-database-url

### 905.1 Parse DATABASE_URL อัตโนมัติ

```bash
pip install dj-database-url
```

```python
# settings.py
import dj_database_url
import os

DATABASES = {
    'default': dj_database_url.config(
        default=os.environ.get('DATABASE_URL', 'sqlite:///db.sqlite3'),
        conn_max_age=600,
        conn_health_checks=True,
    )
}
```

รองรับ format:
- PostgreSQL: `postgres://user:pass@host:5432/db`
- SQLite: `sqlite:///path/to/db.sqlite3`

---

## ขั้นตอนที่ 906: Deploy ไป Heroku

### 906.1 Procfile และคำสั่ง Heroku

```procfile
web: gunicorn config.wsgi:application --bind 0.0.0.0:$PORT --log-level warning
worker: celery -A config worker --loglevel=warning
release: python manage.py migrate
```

```bash
heroku login
heroku create myapp-unique-name
heroku config:set SECRET_KEY=your-secret-key DEBUG=False
heroku addons:create heroku-postgresql:mini
heroku addons:create heroku-redis:mini
git push heroku main
heroku logs --tail
```

---

## ขั้นตอนที่ 907: Fly.io

### 907.1 fly.toml

```toml
# fly.toml
app = "myapp"
primary_region = "sin"  # Singapore

[build]
  dockerfile = "Dockerfile"

[env]
  PORT = "8000"
  DJANGO_SETTINGS_MODULE = "config.settings.production"

[http_service]
  internal_port = 8000
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1

[processes]
  app = "gunicorn config.wsgi:application --bind 0.0.0.0:$PORT"
  worker = "celery -A config worker -l warning"
```

```bash
curl -L https://fly.io/install.sh | sh
fly auth login
fly launch
fly deploy
```

---

## ขั้นตอนที่ 908: Custom Domain บน PaaS

### 908.1 Railway และ Render Custom Domain

```bash
# Railway: Settings > Domains > Custom Domain
# CNAME: myapp.example.com → xxx.railway.app
ALLOWED_HOSTS = ['myapp.example.com', '*.railway.app']

# Render: Settings > Custom Domains
# CNAME: myapp.example.com → xxx.onrender.com
ALLOWED_HOSTS = [os.environ.get('RENDER_EXTERNAL_HOSTNAME'), 'myapp.example.com']
```

---

## ขั้นตอนที่ 909: Auto-detect Platform

### 909.1 ตรวจสอบ Platform อัตโนมัติ

```python
# config/settings/base.py
import os


def detect_environment():
    if os.environ.get('RAILWAY_ENVIRONMENT'):
        return 'railway'
    if os.environ.get('RENDER'):
        return 'render'
    if os.environ.get('DYNO'):  # Heroku
        return 'heroku'
    if os.environ.get('FLY_APP_NAME'):
        return 'fly'
    if os.environ.get('K_SERVICE'):  # Cloud Run
        return 'cloudrun'
    return 'development'


PLATFORM = detect_environment()
```

---

## ขั้นตอนที่ 910: สรุปและแบบฝึกหัด

### 910.1 Checklist สำหรับ PaaS Deployment

- [x] `Procfile` หรือ `railway.toml` / `render.yaml` สร้างแล้ว
- [x] `SECRET_KEY` ตั้งค่าเป็น env variable (ไม่ hardcode)
- [x] `DEBUG=False` ใน production
- [x] `ALLOWED_HOSTS` รวม PaaS domain แล้ว
- [x] `whitenoise` ติดตั้งสำหรับ static files
- [x] `dj-database-url` parse `DATABASE_URL` ได้
- [x] `migrate` รันใน release phase
- [x] `collectstatic` รันใน build phase

### 910.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: Deploy Django app ไป Railway ให้ทำงานได้จริง โดยใช้ PostgreSQL service ของ Railway

**แบบฝึกหัดที่ 2**: ตั้งค่า WhiteNoise ให้ serve static files พร้อม compression และ cache headers ที่ถูกต้อง

**แบบฝึกหัดที่ 3**: สร้าง `render.yaml` ที่ deploy Django web service + Celery worker + PostgreSQL database พร้อมกัน
