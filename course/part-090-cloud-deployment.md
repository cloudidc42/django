# Part 090: Deployment บน Cloud: AWS, GCP, Azure

> **ขั้นตอนที่ 891-900** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 891: AWS Deployment Architecture

### 891.1 Overview

```
Internet
   │
   ▼
[Route 53 DNS]
   │
   ▼
[ALB (Application Load Balancer)]
   │
   ├─────────────────┐
   ▼                 ▼
[EC2/ECS Task]  [EC2/ECS Task]
      │
      ├── [RDS PostgreSQL]
      ├── [ElastiCache Redis]
      └── [S3 Static/Media Files]
```

### 891.2 Services ที่ใช้

| Service | หน้าที่ |
|---------|----------|
| EC2 / ECS | รัน Django application |
| RDS (PostgreSQL) | Database |
| ElastiCache (Redis) | Cache + Celery broker |
| S3 | Static files + Media files |
| CloudFront | CDN สำหรับ static files |
| ALB | Load balancer |
| Route 53 | DNS management |
| ACM | SSL/TLS certificates |
| IAM | Permission management |

---

## ขั้นตอนที่ 892: S3 สำหรับ Static และ Media Files

### 892.1 django-storages กับ S3

```bash
pip install django-storages[boto3]
```

```python
# config/settings/production.py
import os

AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
AWS_S3_REGION_NAME = 'ap-southeast-1'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'

AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',
}

AWS_QUERYSTRING_AUTH = False  # สำหรับ public files

# Static files → S3
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'
STATIC_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/static/'

# Media files → S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
MEDIA_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'
```

### 892.2 S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/static/*"
    }
  ]
}
```

---

## ขั้นตอนที่ 893: AWS ECS (Elastic Container Service)

### 893.1 Task Definition

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "ghcr.io/username/myapp:latest",
      "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
      "environment": [
        {"name": "DJANGO_SETTINGS_MODULE", "value": "config.settings.production"}
      ],
      "secrets": [
        {"name": "SECRET_KEY", "valueFrom": "arn:aws:secretsmanager:region:account:secret:myapp/SECRET_KEY"},
        {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:region:account:secret:myapp/DATABASE_URL"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

## ขั้นตอนที่ 894: RDS PostgreSQL

### 894.1 ตั้งค่า Django กับ RDS

```python
# config/settings/production.py
import dj_database_url

DATABASES = {
    'default': dj_database_url.config(
        default=os.environ['DATABASE_URL'],
        conn_max_age=600,
        conn_health_checks=True,
        ssl_require=True,  # RDS ต้องการ SSL
    )
}
```

### 894.2 Connection Pooling ด้วย PgBouncer

```python
# ถ้าใช้ PgBouncer
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp',
        'HOST': 'pgbouncer-endpoint',
        'PORT': '6432',
        'CONN_MAX_AGE': 0,  # ปิด persistent connections เมื่อใช้ PgBouncer
        'OPTIONS': {'sslmode': 'require'},
    }
}
```

---

## ขั้นตอนที่ 895: GCP Cloud Run

### 895.1 Deploy ไป Cloud Run

```bash
# สร้าง Docker image
docker build -t gcr.io/PROJECT_ID/myapp:latest .

# Push ไป GCR
gcloud auth configure-docker
docker push gcr.io/PROJECT_ID/myapp:latest

# Deploy ไป Cloud Run
gcloud run deploy myapp \
  --image gcr.io/PROJECT_ID/myapp:latest \
  --platform managed \
  --region asia-southeast1 \
  --allow-unauthenticated \
  --set-env-vars DEBUG=False \
  --set-secrets SECRET_KEY=django-secret-key:latest \
  --set-secrets DATABASE_URL=database-url:latest \
  --min-instances 1 \
  --max-instances 10 \
  --memory 512Mi \
  --cpu 1
```

### 895.2 Cloud Run กับ Cloud SQL (PostgreSQL)

```python
# settings.py — Cloud SQL Unix socket
import os

if os.environ.get('K_SERVICE'):  # Cloud Run
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'HOST': f'/cloudsql/{os.environ["INSTANCE_CONNECTION_NAME"]}',
            'NAME': os.environ['DB_NAME'],
            'USER': os.environ['DB_USER'],
            'PASSWORD': os.environ['DB_PASSWORD'],
        }
    }
```

---

## ขั้นตอนที่ 896: Azure App Service

### 896.1 Deploy ไป Azure

```bash
# Login
az login

# สร้าง Resource Group
az group create --name myapp-rg --location southeastasia

# สร้าง App Service Plan
az appservice plan create \
  --name myapp-plan \
  --resource-group myapp-rg \
  --sku B2 \
  --is-linux

# สร้าง Web App
az webapp create \
  --resource-group myapp-rg \
  --plan myapp-plan \
  --name myapp-unique-name \
  --deployment-container-image-name ghcr.io/username/myapp:latest

# ตั้งค่า environment variables
az webapp config appsettings set \
  --resource-group myapp-rg \
  --name myapp-unique-name \
  --settings \
    SECRET_KEY="your-secret-key" \
    DATABASE_URL="postgres://..." \
    WEBSITES_PORT="8000"
```

---

## ขั้นตอนที่ 897: Auto-scaling

### 897.1 AWS Auto Scaling Group

```bash
# Policy: scale up เมื่อ CPU > 70%
aws application-autoscaling put-scaling-policy \
  --policy-name cpu-scale-out \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

---

## ขั้นตอนที่ 898: Infrastructure as Code ด้วย Terraform

### 898.1 ตัวอย่าง Terraform สำหรับ AWS

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_db_instance" "postgres" {
  identifier        = "myapp-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  db_name  = "myapp_production"
  username = "myapp"
  password = var.db_password

  skip_final_snapshot = false
  deletion_protection = true

  tags = {
    Environment = "production"
    Project     = "myapp"
  }
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "myapp-redis"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  port                 = 6379
}
```

---

## ขั้นตอนที่ 899: Health Check Endpoint

### 899.1 Health Check View

```python
# apps/core/views.py
from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache
import time


def health_check(request):
    """Health check endpoint สำหรับ Load Balancer"""
    checks = {}
    status = 200

    # Database check
    try:
        start = time.time()
        connection.ensure_connection()
        checks['database'] = {'status': 'ok', 'latency_ms': round((time.time() - start) * 1000)}
    except Exception as e:
        checks['database'] = {'status': 'error', 'error': str(e)}
        status = 503

    # Cache check
    try:
        start = time.time()
        cache.set('health_check', '1', timeout=5)
        assert cache.get('health_check') == '1'
        checks['cache'] = {'status': 'ok', 'latency_ms': round((time.time() - start) * 1000)}
    except Exception as e:
        checks['cache'] = {'status': 'error', 'error': str(e)}
        status = 503

    return JsonResponse({'status': 'healthy' if status == 200 else 'unhealthy', 'checks': checks}, status=status)
```

```python
# urls.py
from apps.core.views import health_check

urlpatterns = [
    path('health/', health_check, name='health-check'),
]
```

---

## ขั้นตอนที่ 900: สรุปและแบบฝึกหัด

### 900.1 เปรียบเทียบ Cloud Providers

| Feature | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Django-friendly | ECS Fargate | Cloud Run | App Service |
| Managed DB | RDS | Cloud SQL | Azure Database for PostgreSQL |
| Redis | ElastiCache | Memorystore | Azure Cache for Redis |
| Storage | S3 | Cloud Storage | Azure Blob Storage |
| Free tier | t2.micro | e2-micro | F1 |
| Thai data center | ap-southeast-1 | asia-southeast1 | eastasia |

### 900.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: Deploy Django app ไป GCP Cloud Run โดยใช้ Cloud SQL เป็น database และ GCS เป็น static storage

**แบบฝึกหัดที่ 2**: ตั้งค่า S3 + CloudFront สำหรับ static files ด้วย `django-storages` และ measure latency ก่อน/หลัง

**แบบฝึกหัดที่ 3**: สร้าง `/health/` endpoint ที่ตรวจสอบ database + cache แล้วตั้งค่าเป็น health check สำหรับ AWS ALB
