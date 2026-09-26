# Part 094: Microservices Architecture กับ Django

> **ขั้นตอนที่ 931-940** | Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ

---

## ขั้นตอนที่ 931: Monolith vs Microservices

### 931.1 ทำความเข้าใจ Monolith

โปรเจกต์ blog ที่สร้างมาตลอดหลักสูตรคือ **Monolith** — แอปพลิเคชันเดียวที่ทำทุกอย่าง:

```
myblog (Monolith)
├── blog/       → ระบบบทความ
├── accounts/   → ระบบ user/auth
├── payments/   → ระบบชำระเงิน
├── search/     → ระบบค้นหา
└── notifications/ → ระบบแจ้งเตือน
```

**ข้อดี Monolith**: ง่ายต่อการพัฒนา, debug, deploy, และ test ในช่วงแรก
**ข้อเสีย**: scale บางส่วนไม่ได้, ทีมใหญ่ขัดแย้งกัน, deploy ช้าเพราะต้อง deploy ทั้งก้อน

### 931.2 Microservices Architecture

```
Client
  │
API Gateway (Kong / AWS API Gateway)
  │
  ├── User Service (Django) → PostgreSQL_users
  ├── Blog Service (Django) → PostgreSQL_blog
  ├── Search Service (FastAPI) → Elasticsearch
  ├── Payment Service (Django) → PostgreSQL_payments
  └── Notification Service (FastAPI) → Redis + SMTP

  ← Services communicate via HTTP/gRPC/Message Queue →
```

### 931.3 เมื่อไหรควร Migrate ไป Microservices

**ยังไม่ควร migrate ถ้า**:
- ทีม < 20 คน
- Traffic < 100K requests/วัน
- Monolith ยัง deploy ได้ภายใน 30 นาที
- ยังไม่มีปัญหา performance ที่ชัดเจน

**ควร migrate ถ้า**:
- แต่ละ module ต้องการ scale ต่างกัน (Search เยอะกว่า Payments 100x)
- ทีมต่างกัน owns ต่างโมดูล และขัดแย้งกัน
- ต้องการใช้ tech stack ที่ต่างกัน (ML service ใช้ Python, real-time service ใช้ Go)

---

## ขั้นตอนที่ 932: Django เป็น Microservice

### 932.1 ออกแบบ Service Boundaries

```
E-Commerce Platform:
┌─────────────────────────────────────────────────────────┐
│ Service        │ ความรับผิดชอบ              │ Django app │
├────────────────┼───────────────────────────┼────────────┤
│ users-service  │ Auth, profiles, sessions   │ accounts/  │
│ products-service│ Catalog, inventory        │ products/  │
│ orders-service │ Cart, orders, checkout    │ orders/    │
│ payments-service│ Payment processing        │ payments/  │
│ notifications  │ Email, SMS, push          │ notifs/    │
└─────────────────────────────────────────────────────────┘
```

### 932.2 ตัวอย่าง: Django Microservice อย่างง่าย

```python
# users_service/config/settings.py
# Settings สำหรับ micro service ที่เล็กที่สุด

from pathlib import Path
import environ

env = environ.Env()
BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = env('SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

INSTALLED_APPS = [
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'rest_framework',
    'corsheaders',
    'users',     # เฉพาะ app นี้เท่านั้น
]

DATABASES = {'default': env.db('DATABASE_URL')}
ROOT_URLCONF = 'config.urls'

# Minimal middleware
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
]
```

### 932.3 Service Communication ด้วย HTTP

```python
# orders_service/services/user_client.py
import httpx
from django.conf import settings


class UserServiceClient:
    """Client สำหรับ call Users Service"""

    BASE_URL = settings.USERS_SERVICE_URL  # 'http://users-service:8001'

    @classmethod
    def get_user(cls, user_id: int, auth_token: str) -> dict:
        """ดึงข้อมูล user จาก Users Service"""
        with httpx.Client(timeout=5.0) as client:
            response = client.get(
                f"{cls.BASE_URL}/api/v1/users/{user_id}/",
                headers={"Authorization": f"Bearer {auth_token}"},
            )
            response.raise_for_status()
            return response.json()

    @classmethod
    def verify_token(cls, token: str) -> dict:
        """ตรวจสอบ JWT token กับ Users Service"""
        with httpx.Client(timeout=3.0) as client:
            response = client.post(
                f"{cls.BASE_URL}/api/v1/auth/verify/",
                json={"token": token},
            )
            if response.status_code == 200:
                return response.json()
            raise ValueError("Invalid token")
```

---

## ขั้นตอนที่ 933: Message Queue สำหรับ Async Communication

### 933.1 Event-Driven Architecture

```
Order Service                          Notification Service
     │                                       │
     │ publish "order.created" event         │
     └─────────────────┬────────────────────►│
                       │                     │
               RabbitMQ/Kafka                │
               (Message Broker)              │
                                     (subscribe "order.created")
                                       → ส่ง email confirmation
```

### 933.2 ใช้ Celery เป็น Event Bus

```python
# orders_service/tasks.py
from celery import shared_task
from kombu import Exchange, Queue

# กำหนด routing
ORDER_EXCHANGE = Exchange('orders', type='topic')

CELERY_TASK_ROUTES = {
    'orders.order_created': {'queue': 'order_events'},
    'orders.order_paid': {'queue': 'order_events'},
}

@shared_task(name='orders.order_created')
def publish_order_created(order_id: int):
    """Publish event เมื่อ order ถูกสร้าง"""
    from orders.models import Order
    order = Order.objects.get(id=order_id)
    # Notification service จะ subscribe event นี้
    return {
        'order_id': order.id,
        'user_id': order.user_id,
        'total': str(order.total),
        'items': list(order.items.values('product_id', 'quantity', 'price')),
    }


# orders_service/views.py
def create_order(request):
    order = Order.objects.create(user=request.user, ...)
    # publish event async
    publish_order_created.delay(order.id)
    return Response({'id': order.id}, status=201)
```

### 933.3 Notification Service Subscribe Event

```python
# notifications_service/tasks.py
from celery import Celery

app = Celery('notifications')

@app.task(queue='order_events', name='orders.order_created')
def handle_order_created(order_data: dict):
    """Subscribe และจัดการ order.created event จาก orders service"""
    from notifications.email import send_order_confirmation
    send_order_confirmation(
        user_id=order_data['user_id'],
        order_id=order_data['order_id'],
        total=order_data['total'],
    )
```

---

## ขั้นตอนที่ 934: API Gateway

### 934.1 ทำไมต้องใช้ API Gateway

```
ก่อน API Gateway:
Client → users-service:8001
Client → products-service:8002
Client → orders-service:8003
→ Client ต้องรู้ address ของทุก service
→ ต้องจัดการ auth ในทุก service

หลัง API Gateway:
Client → API Gateway:80
         ├── /api/users/* → users-service:8001
         ├── /api/products/* → products-service:8002
         └── /api/orders/* → orders-service:8003
→ Single entry point, centralized auth, rate limiting, logging
```

### 934.2 Nginx เป็น Simple API Gateway

```nginx
# nginx.conf — Simple API Gateway

upstream users_service {
    server users-service:8001;
}

upstream products_service {
    server products-service:8002;
}

upstream orders_service {
    server orders-service:8003;
}

server {
    listen 80;

    # Route ตาม URL prefix
    location /api/v1/users/ {
        proxy_pass http://users_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/v1/products/ {
        proxy_pass http://products_service;
        proxy_set_header Host $host;
    }

    location /api/v1/orders/ {
        # Auth check ก่อน route
        auth_request /auth/verify;
        proxy_pass http://orders_service;
    }

    # Auth verification endpoint
    location = /auth/verify {
        internal;
        proxy_pass http://users_service/api/v1/auth/verify/;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
}
```

### 934.3 Kong API Gateway (Enterprise-grade)

```yaml
# kong-config.yml (declarative config)
_format_version: "3.0"

services:
  - name: users-service
    url: http://users-service:8001
    routes:
      - name: users-route
        paths: ["/api/v1/users"]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local

  - name: orders-service
    url: http://orders-service:8003
    routes:
      - name: orders-route
        paths: ["/api/v1/orders"]
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
```

---

## ขั้นตอนที่ 935: Service Discovery

### 935.1 ใน Docker Compose (Local Development)

```yaml
# docker-compose.yml — services ค้นหากันผ่านชื่อ service
services:
  users-service:
    hostname: users-service
    ports: ["8001:8000"]

  orders-service:
    hostname: orders-service
    ports: ["8003:8000"]
    environment:
      USERS_SERVICE_URL: http://users-service:8000
```

### 935.2 ใน Kubernetes (Production)

```yaml
# k8s/service-users.yaml
apiVersion: v1
kind: Service
metadata:
  name: users-service    # DNS name ภายใน cluster
  namespace: ecommerce
spec:
  selector:
    app: users-service
  ports:
    - port: 80
      targetPort: 8000
```

```python
# orders_service/config/settings.py
# K8s DNS: service-name.namespace.svc.cluster.local
USERS_SERVICE_URL = 'http://users-service.ecommerce.svc.cluster.local'
```

---

## ขั้นตอนที่ 936: Distributed Tracing

### 936.1 OpenTelemetry สำหรับ Django

```bash
pip install opentelemetry-distro opentelemetry-exporter-jaeger
opentelemetry-bootstrap -a install
```

```python
# config/settings.py
from opentelemetry import trace
from opentelemetry.instrumentation.django import DjangoInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# ตั้งค่า OpenTelemetry ส่ง traces ไป Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
trace.set_tracer_provider(provider)

# Auto-instrument Django, requests, psycopg2
DjangoInstrumentor().instrument()
RequestsInstrumentor().instrument()
Psycopg2Instrumentor().instrument()
```

---

## ขั้นตอนที่ 937: Data Consistency ใน Microservices

### 937.1 Saga Pattern สำหรับ Distributed Transactions

```
ปัญหา: ต้องการ atomic transaction ข้าม service หลายตัว
Order flow: Create Order → Deduct Inventory → Process Payment → Send Notification
ถ้า Payment ล้มเหลว: ต้อง rollback Inventory และ Cancel Order

Saga Orchestration:
Order Service (Orchestrator)
    → Inventory Service: reserve_items
    → Payment Service: process_payment
    → ถ้า Payment ล้มเหลว:
        → Inventory Service: release_items (compensating transaction)
        → Order Service: cancel_order
```

```python
# orders/sagas.py
from celery import chain, chord

def create_order_saga(order_id: int):
    """Saga orchestrator ใช้ Celery chain"""
    saga = chain(
        reserve_inventory.si(order_id),
        process_payment.si(order_id),
        send_order_confirmation.si(order_id),
    )

    result = saga.apply_async()
    return result


@shared_task(bind=True, max_retries=3)
def reserve_inventory(self, order_id: int):
    try:
        response = inventory_client.reserve(order_id)
        return response
    except Exception as e:
        # Compensating transaction ถ้า retry หมดแล้ว
        if self.request.retries >= self.max_retries:
            cancel_order.delay(order_id, reason='inventory_unavailable')
        raise self.retry(exc=e, countdown=2 ** self.request.retries)
```

---

## ขั้นตอนที่ 938: Testing Microservices

### 938.1 Contract Testing ด้วย Pact

```bash
pip install pact-python
```

```python
# tests/test_user_client_contract.py
from pact import Consumer, Provider

pact = Consumer('orders-service').has_pact_with(
    Provider('users-service'), host_name='localhost', port=8001
)


def test_get_user_contract():
    """ทดสอบว่า orders-service เรียก users-service ถูกต้อง"""
    expected = {
        'id': 1,
        'username': 'testuser',
        'email': 'test@example.com',
    }

    (pact
     .given('user with id 1 exists')
     .upon_receiving('a request for user 1')
     .with_request('GET', '/api/v1/users/1/')
     .will_respond_with(200, body=expected))

    with pact:
        result = UserServiceClient.get_user(user_id=1, auth_token='test-token')
        assert result['id'] == 1
```

### 938.2 Integration Test ด้วย Docker Compose

```yaml
# docker-compose.test.yml
services:
  users-service:
    image: users-service:test
    environment:
      DATABASE_URL: postgresql://test:test@users-db:5432/test

  orders-service:
    image: orders-service:test
    environment:
      USERS_SERVICE_URL: http://users-service:8000

  test-runner:
    image: orders-service:test
    command: pytest tests/integration/ -v
    depends_on:
      - users-service
      - orders-service
```

---

## ขั้นตอนที่ 939: Strangler Fig Pattern — Migrate จาก Monolith

### 939.1 ค่อย ๆ ย้ายทีละ Service

```
ขั้นตอน:

1. เริ่มจาก Monolith ทำงานทุกอย่าง
   → ค่อย ๆ ย้ายทีละ feature ออกเป็น microservice

2. ตั้ง API Gateway ข้างหน้า Monolith
   GET /api/products/* → ยังคงไปที่ Monolith (เดิม)
   GET /api/users/* → ยังคงไปที่ Monolith

3. สร้าง Users Microservice ใหม่
   GET /api/users/* → Users Microservice (ใหม่)
   GET /api/products/* → ยังคงไปที่ Monolith

4. ย้ายทีละ service จนหมด
5. Monolith หายไปเอง
```

```python
# config/settings/production.py — Monolith กำลัง migrate
# Feature flag สำหรับ redirect traffic ไป microservice

USE_USERS_MICROSERVICE = env.bool('USE_USERS_MICROSERVICE', default=False)

if USE_USERS_MICROSERVICE:
    # ใช้ Users Microservice
    AUTH_USER_MODEL_URL = env('USERS_SERVICE_URL')
else:
    # ยังคงใช้ local models
    AUTH_USER_MODEL = 'accounts.User'
```

---

## ขั้นตอนที่ 940: สรุปและแบบฝึกหัด

### 940.1 Microservices Checklist

- [ ] Service boundaries กำหนดชัดเจน (1 service = 1 domain)
- [ ] Services communicate ผ่าน HTTP API หรือ Message Queue เท่านั้น
- [ ] แต่ละ service มี database เป็นของตัวเอง (no shared DB)
- [ ] API Gateway เป็น single entry point
- [ ] Distributed tracing ตั้งค่าแล้ว (Jaeger/Zipkin)
- [ ] Contract testing ระหว่าง services
- [ ] Health check ทุก service
- [ ] Circuit breaker ป้องกัน cascade failures

### 940.2 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: แยก `accounts` app ออกจาก blog monolith เป็น Users Microservice
แยก Django project, แยก database, expose REST API สำหรับ auth/profile

**แบบฝึกหัดที่ 2**: ตั้งค่า Nginx เป็น API Gateway ที่ route `/api/users/*` ไป Users
Service และ `/api/blog/*` ไป Blog Service ใน docker-compose ทดสอบว่า routing ถูกต้อง

**แบบฝึกหัดที่ 3**: ใช้ Celery เป็น event bus ส่ง event `user.registered` จาก Users
Service แล้วให้ Blog Service subscribe และสร้าง welcome post อัตโนมัติ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ติดตั้ง Jaeger สำหรับ distributed tracing และทดสอบว่า
เมื่อ order-service call user-service, trace ปรากฏใน Jaeger UI ครบทั้ง 2 services

### 940.3 เตรียมตัวสำหรับ Part ถัดไป

**Part 095: Multi-tenant Django Applications** จะพาไปสร้างระบบ SaaS แบบ multi-tenant
ที่ให้ customers หลาย organizations ใช้ระบบเดียวกันโดยข้อมูลแยกกัน — เรียนรู้ทั้ง
shared-schema, row-level, และ database-per-tenant approaches
