# Part 084: Secrets Management และ Environment Variables

> **ขั้นตอนที่ 831-840** | Phase 10: Security

---

## ขั้นตอนที่ 831: ปัญหาของ Hardcoded Secrets

### 831.1 ทำไมต้องจัดการ Secrets อย่างถูกต้อง

Secrets ที่ไม่ควร hardcode ในโค้ด:
- `SECRET_KEY`
- Database password
- API keys (Stripe, SendGrid, AWS)
- OAuth client secrets
- Encryption keys
- JWT secrets

**ตรวจสอบว่า code ปัจจุบัน leak secrets หรือเปล่า**

```bash
# หา secrets ที่อาจหลุดใน git history
git log --all --full-history -- "*.env"
git grep -i "password\|secret\|api_key\|token" -- "*.py" | grep -v test
```

---

## ขั้นตอนที่ 832: python-decouple

### 832.1 ติดตั้งและตั้งค่า

```bash
pip install python-decouple
```

```ini
# .env (ไม่ commit ไฟล์นี้เข้า git)
DEBUG=False
SECRET_KEY=your-very-secret-key-here-change-this
DATABASE_URL=postgres://user:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379/0
ALLOWED_HOSTS=example.com,www.example.com
EMAIL_HOST_PASSWORD=smtp-password-here
STRIPE_SECRET_KEY=sk_live_...
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```python
# config/settings/base.py
from decouple import config, Csv

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost', cast=Csv())

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME', default='mydb'),
        'USER': config('DB_USER', default='postgres'),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
    }
}
```

### 832.2 .env.example สำหรับ Developer ใหม่

```ini
# .env.example — commit ไฟล์นี้เข้า git (ไม่มีค่าจริง)
DEBUG=True
SECRET_KEY=change-this-to-a-real-secret-key
DATABASE_URL=postgres://postgres:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379/0
ALLOWED_HOSTS=localhost,127.0.0.1
EMAIL_HOST_PASSWORD=
STRIPE_SECRET_KEY=sk_test_...
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

---

## ขั้นตอนที่ 833: django-environ

### 833.1 อีกตัวเลือกที่นิยม

```bash
pip install django-environ
```

```python
# config/settings/base.py
import environ
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, ['localhost']),
)

# อ่านไฟล์ .env
environ.Env.read_env(BASE_DIR / '.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')

DATABASES = {
    'default': env.db('DATABASE_URL', default='sqlite:///db.sqlite3')
}

CACHES = {
    'default': env.cache('REDIS_URL', default='redis://localhost:6379/0')
}

EMAIL_CONFIG = env.email('EMAIL_URL', default='console://')
vars().update(EMAIL_CONFIG)
```

---

## ขั้นตอนที่ 834: .gitignore ที่ถูกต้อง

### 834.1 ไฟล์ที่ต้องอยู่ใน .gitignore

```gitignore
# .gitignore

# Secrets
.env
.env.*
!.env.example
*.pem
*.key
credentials.json
service-account.json

# Django
*.pyc
__pycache__/
db.sqlite3
media/
staticfiles/

# Python
.venv/
venv/
*.egg-info/
dist/
build/

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db
```

### 834.2 ล้าง Secrets ที่ Commit ไปแล้ว

```bash
# ถ้า commit ไฟล์ .env ไปแล้ว ต้องทำดังนี้:

# 1. เพิ่ม .env ใน .gitignore
echo ".env" >> .gitignore

# 2. ลบออกจาก git tracking (แต่เก็บไฟล์ไว้)
git rm --cached .env

# 3. Commit การเปลี่ยนแปลง
git commit -m "Remove .env from tracking"

# 4. Rotate ทุก secrets ที่หลุดออกไปทันที!
# (Secret key ที่ commit ไปแล้วถือว่า compromised)
```

---

## ขั้นตอนที่ 835: Django SECRET_KEY ที่ปลอดภัย

### 835.1 สร้าง SECRET_KEY ที่แข็งแกร่ง

```python
# สร้าง secret key ใหม่
python -c "
from django.core.management.utils import get_random_secret_key
print(get_random_secret_key())
"

# หรือใช้ secrets module
python -c "
import secrets
print(secrets.token_urlsafe(50))
"
```

### 835.2 หมุน SECRET_KEY (Key Rotation)

```python
# config/settings/base.py — รองรับ multiple secret keys
from django.conf import settings

# Key ปัจจุบัน (ใช้ sign sessions/cookies ใหม่)
SECRET_KEY = config('SECRET_KEY')

# Keys เก่าที่ยังยอมรับ (validate sessions เก่า)
SECRET_KEY_FALLBACKS = config('SECRET_KEY_FALLBACKS', default='', cast=Csv())
```

---

## ขั้นตอนที่ 836: AWS Secrets Manager

### 836.1 ดึง Secrets จาก AWS

```bash
pip install boto3
```

```python
# apps/core/secrets.py
import boto3
import json
from functools import lru_cache


@lru_cache(maxsize=None)
def get_secret(secret_name: str, region_name: str = 'ap-southeast-1') -> dict:
    """ดึง secret จาก AWS Secrets Manager (cached)"""
    client = boto3.client('secretsmanager', region_name=region_name)
    response = client.get_secret_value(SecretId=secret_name)

    if 'SecretString' in response:
        return json.loads(response['SecretString'])
    raise ValueError(f"Secret {secret_name} not found")
```

```python
# config/settings/production.py
from apps.core.secrets import get_secret

db_secret = get_secret('myapp/production/database')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': db_secret['dbname'],
        'USER': db_secret['username'],
        'PASSWORD': db_secret['password'],
        'HOST': db_secret['host'],
        'PORT': db_secret['port'],
    }
}
```

---

## ขั้นตอนที่ 837: HashiCorp Vault

### 837.1 ใช้งาน Vault กับ Django

```bash
pip install hvac
```

```python
# apps/core/vault.py
import hvac
import os


def get_vault_client() -> hvac.Client:
    client = hvac.Client(
        url=os.environ.get('VAULT_ADDR', 'http://localhost:8200'),
        token=os.environ.get('VAULT_TOKEN'),
    )
    assert client.is_authenticated(), "Vault authentication failed"
    return client


def get_secret_from_vault(path: str) -> dict:
    client = get_vault_client()
    secret = client.secrets.kv.v2.read_secret_version(path=path)
    return secret['data']['data']
```

```python
# config/settings/production.py
from apps.core.vault import get_secret_from_vault

secrets = get_secret_from_vault('myapp/production')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PASSWORD': secrets['db_password'],
        ...
    }
}
```

---

## ขั้นตอนที่ 838: Environment-Specific Settings

### 838.1 โครงสร้าง Settings หลายไฟล์

```
config/
  settings/
    __init__.py
    base.py          # ค่าพื้นฐานทั้งหมด
    development.py   # development overrides
    production.py    # production overrides
    testing.py       # test overrides
```

```python
# config/settings/development.py
from .base import *

DEBUG = True
SECRET_KEY = 'dev-secret-key-not-for-production'
ALLOWED_HOSTS = ['*']

# ใช้ sqlite ใน development
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Email ออก console
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# django-debug-toolbar
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
```

```python
# config/settings/testing.py
from .base import *

DEBUG = False
SECRET_KEY = 'test-secret-key'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',
    }
}

PASSWORD_HASHERS = ['django.contrib.auth.hashers.MD5PasswordHasher']
EMAIL_BACKEND = 'django.core.mail.backends.locmem.EmailBackend'
```

---

## ขั้นตอนที่ 839: Docker Secrets

### 839.1 ใช้ Docker Secrets ใน Production

```yaml
# docker-compose.yml (production)
version: '3.8'

secrets:
  db_password:
    external: true
  django_secret_key:
    external: true

services:
  web:
    image: myapp:latest
    secrets:
      - db_password
      - django_secret_key
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      SECRET_KEY_FILE: /run/secrets/django_secret_key
```

```python
# apps/core/docker_secrets.py
import os


def read_secret(env_var: str, secret_file_var: str = None) -> str:
    """อ่าน secret จาก Docker secret file หรือ env var"""
    if secret_file_var:
        file_path = os.environ.get(secret_file_var)
        if file_path and os.path.exists(file_path):
            with open(file_path, 'r') as f:
                return f.read().strip()
    return os.environ.get(env_var, '')
```

---

## ขั้นตอนที่ 840: สรุปและแบบฝึกหัด

### 840.1 Secrets Management Checklist

- [x] ไม่มี hardcoded secrets ในโค้ด
- [x] `.env` อยู่ใน `.gitignore`
- [x] `.env.example` อยู่ใน git (ไม่มีค่าจริง)
- [x] `SECRET_KEY` มีความยาวอย่างน้อย 50 ตัวอักษร
- [x] Production ใช้ secrets manager (AWS/Vault/Docker)
- [x] Secrets rotation plan มีอยู่
- [x] ไม่มี secrets ใน environment variables ที่ log ได้

### 840.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: แปลงโปรเจกต์ที่มี hardcoded settings ให้ใช้ `python-decouple` + `.env` file

**แบบฝึกหัดที่ 2**: ตั้งค่า multiple settings files (base/development/production) พร้อม environment variable `DJANGO_SETTINGS_MODULE`

**แบบฝึกหัดที่ 3**: เขียน script ที่ scan Python files ทั้งหมดหาค่าที่อาจเป็น secrets (regex สำหรับ AWS key format, password patterns)
