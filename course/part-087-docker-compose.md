# Part 087: Docker Compose: Django + PostgreSQL + Redis

> **ขั้นตอนที่ 861-870** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 861: Docker Compose คืออะไร

### 861.1 แนวคิด

**Docker Compose** ใช้ YAML file จัดการหลาย containers ให้ทำงานร่วมกัน

```
myapp/
  docker-compose.yml         # development
  docker-compose.prod.yml    # production overrides
  docker-compose.test.yml    # testing
  .env
  Dockerfile
  Dockerfile.dev
```

---

## ขั้นตอนที่ 862: docker-compose.yml สำหรับ Development

### 862.1 Config พื้นฐาน

```yaml
# docker-compose.yml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp_dev"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app            # hot reload
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    ports:
      - "8000:8000"
    environment:
      DEBUG: "True"
      SECRET_KEY: dev-secret-key-change-in-production
      DATABASE_URL: postgres://myapp:password@db:5432/myapp_dev
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  celery:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: celery -A config worker -l info
    volumes:
      - .:/app
    environment:
      DATABASE_URL: postgres://myapp:password@db:5432/myapp_dev
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
    depends_on:
      - web
      - redis

  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: celery -A config beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    volumes:
      - .:/app
    environment:
      DATABASE_URL: postgres://myapp:password@db:5432/myapp_dev
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
    depends_on:
      - celery

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

---

## ขั้นตอนที่ 863: คำสั่ง Docker Compose

### 863.1 คำสั่งที่ใช้บ่อย

```bash
# เริ่ม services ทั้งหมด
docker compose up

# เริ่มแบบ background
docker compose up -d

# ดู logs
docker compose logs -f
docker compose logs -f web

# หยุด
docker compose down

# หยุดและลบ volumes
docker compose down -v

# รัน Django management commands
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
docker compose exec web python manage.py shell

# Build ใหม่หลังแก้ Dockerfile
docker compose build
docker compose up --build

# ดู status
docker compose ps
```

---

## ขั้นตอนที่ 864: docker-compose.prod.yml

### 864.1 Production Override

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  db:
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  redis:
    restart: unless-stopped

  web:
    build:
      context: .
      dockerfile: Dockerfile.prod
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4 --timeout 120
    restart: unless-stopped
    environment:
      DEBUG: "False"
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgres://myapp:${POSTGRES_PASSWORD}@db:5432/myapp_prod
      REDIS_URL: redis://redis:6379/0
      ALLOWED_HOSTS: ${ALLOWED_HOSTS}
    ports: []   # ไม่ expose โดยตรง — ผ่าน Nginx

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - static_volume:/var/www/static
      - media_volume:/var/www/media
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - web
    restart: unless-stopped

  celery:
    build:
      context: .
      dockerfile: Dockerfile.prod
    command: celery -A config worker -l warning -c 4
    restart: unless-stopped
    environment:
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgres://myapp:${POSTGRES_PASSWORD}@db:5432/myapp_prod
      CELERY_BROKER_URL: redis://redis:6379/1
```

```bash
# รัน production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## ขั้นตอนที่ 865: Nginx config สำหรับ Docker

### 865.1 nginx/nginx.conf

```nginx
# nginx/nginx.conf
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name example.com www.example.com;

    client_max_body_size 10M;

    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /var/www/media/;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60;
        proxy_read_timeout 60;
    }
}
```

---

## ขั้นตอนที่ 866: Environment Variables กับ Docker Compose

### 866.1 .env file

```ini
# .env (ใช้กับ docker-compose)
POSTGRES_PASSWORD=supersecretpassword
SECRET_KEY=your-production-secret-key
ALLOWED_HOSTS=example.com,www.example.com
DJANGO_SETTINGS_MODULE=config.settings.production
```

```yaml
# docker-compose.yml — อ่านจาก .env อัตโนมัติ
services:
  web:
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - DATABASE_URL=postgres://myapp:${POSTGRES_PASSWORD}@db:5432/myapp

# หรืออ่านไฟล์ .env โดยตรง
    env_file:
      - .env
```

---

## ขั้นตอนที่ 867: Health Checks และ Dependency Order

### 867.1 Service Dependencies ที่ถูกต้อง

```yaml
services:
  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
```

---

## ขั้นตอนที่ 868: Makefile สำหรับ Docker Commands

### 868.1 ทำให้ใช้งานง่ายขึ้น

```makefile
# Makefile
.PHONY: up down build migrate shell logs test

up:
	docker compose up -d

down:
	docker compose down

build:
	docker compose build

migrate:
	docker compose exec web python manage.py migrate

makemigrations:
	docker compose exec web python manage.py makemigrations

shell:
	docker compose exec web python manage.py shell

bash:
	docker compose exec web bash

logs:
	docker compose logs -f web

test:
	docker compose exec web pytest

superuser:
	docker compose exec web python manage.py createsuperuser

collectstatic:
	docker compose exec web python manage.py collectstatic --noinput

restart:
	docker compose restart web

prod-up:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

prod-down:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml down
```

```bash
# ใช้งาน
make up
make migrate
make shell
make test
```

---

## ขั้นตอนที่ 869: Docker Compose สำหรับ Testing

### 869.1 docker-compose.test.yml

```yaml
# docker-compose.test.yml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    tmpfs:
      - /var/lib/postgresql/data  # ใช้ RAM — เร็วกว่า

  test:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: pytest --tb=short -q
    environment:
      DATABASE_URL: postgres://test:test@db:5432/test_db
      DJANGO_SETTINGS_MODULE: config.settings.testing
      SECRET_KEY: test-secret-key
    depends_on:
      - db
    volumes:
      - .:/app
```

```bash
# รัน tests ใน Docker
docker compose -f docker-compose.test.yml run --rm test

# รัน test เฉพาะ app
docker compose -f docker-compose.test.yml run --rm test pytest apps/accounts/
```

---

## ขั้นตอนที่ 870: สรุปและแบบฝึกหัด

### 870.1 Docker Compose Architecture

```
Internet
   │
   ▼
[Nginx :80/:443]
   │
   ▼
[Django/Gunicorn :8000] ↔ [Redis :6379]
   │                              │
   ▼                              ▼
[PostgreSQL :5432]         [Celery Worker]
                           [Celery Beat]
```

### 870.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: สร้าง `docker-compose.yml` ที่มี web + db + redis + celery ทั้งหมดทำงานได้จริง ทดสอบด้วย `docker compose up` และ `docker compose exec web python manage.py migrate`

**แบบฝึกหัดที่ 2**: เพิ่ม Nginx service ใน docker-compose.prod.yml ให้ serve static files โดยตรงไม่ผ่าน Django

**แบบฝึกหัดที่ 3**: สร้าง Makefile ที่มีคำสั่ง `make up`, `make test`, `make deploy` ใช้งานได้จริง
