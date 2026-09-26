# Part 093: Monitoring และ Logging: Sentry, Prometheus, ELK

> **ขั้นตอนที่ 921-930** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 921: Sentry Error Tracking

### 921.1 ติดตั้งและตั้งค่า

```bash
pip install sentry-sdk[django]
```

```python
# config/settings/production.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
from sentry_sdk.integrations.redis import RedisIntegration

sentry_sdk.init(
    dsn=os.environ.get('SENTRY_DSN'),
    integrations=[
        DjangoIntegration(
            transaction_style='url',
            middleware_spans=True,
        ),
        CeleryIntegration(),
        RedisIntegration(),
    ],
    traces_sample_rate=0.1,    # 10% ของ requests สำหรับ performance
    profiles_sample_rate=0.1,
    send_default_pii=False,    # ไม่ส่ง PII
    environment=os.environ.get('ENVIRONMENT', 'production'),
    release=os.environ.get('GIT_COMMIT_SHA', 'unknown'),
)
```

### 921.2 Custom Error Context

```python
# apps/core/middleware.py
import sentry_sdk


class SentryContextMiddleware:
    """เพิ่ม user context ใน Sentry"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated:
            sentry_sdk.set_user({
                'id': str(request.user.pk),
                'username': request.user.username,
                'email': request.user.email,
            })
        return self.get_response(request)
```

### 921.3 Manual Error Capture

```python
import sentry_sdk

try:
    process_payment(order)
except Exception as e:
    with sentry_sdk.push_scope() as scope:
        scope.set_extra('order_id', order.pk)
        scope.set_extra('amount', str(order.total))
        scope.set_tag('payment_method', order.payment_method)
        sentry_sdk.capture_exception(e)
    raise
```

---

## ขั้นตอนที่ 922: Structured Logging ด้วย structlog

### 922.1 ติดตั้งและตั้งค่า

```bash
pip install structlog
```

```python
# config/logging.py
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.dev.ConsoleRenderer() if DEBUG else structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)
```

```python
# apps/orders/views.py
import structlog

logger = structlog.get_logger(__name__)


def create_order(request):
    log = logger.bind(user_id=request.user.pk, request_id=request.META.get('HTTP_X_REQUEST_ID'))
    log.info("order.create.started")

    try:
        order = Order.objects.create(user=request.user)
        log.info("order.create.completed", order_id=order.pk, amount=str(order.total))
    except Exception as e:
        log.error("order.create.failed", error=str(e))
        raise

    return JsonResponse({'order_id': order.pk})
```

---

## ขั้นตอนที่ 923: Django Logging Configuration

### 923.1 settings.py Logging

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/app.log',
            'maxBytes': 10 * 1024 * 1024,  # 10MB
            'backupCount': 5,
            'formatter': 'json',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
    'loggers': {
        'django.request': {
            'handlers': ['console', 'file'],
            'level': 'ERROR',
            'propagate': False,
        },
        'apps': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

---

## ขั้นตอนที่ 924: Prometheus Metrics

### 924.1 django-prometheus

```bash
pip install django-prometheus
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_prometheus',
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    ...
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]
```

```python
# urls.py
urlpatterns = [
    path('', include('django_prometheus.urls')),  # expose /metrics
    ...
]
```

### 924.2 Custom Metrics

```python
# apps/core/metrics.py
from prometheus_client import Counter, Histogram, Gauge

order_created_total = Counter(
    'order_created_total',
    'Total orders created',
    ['payment_method', 'status'],
)

order_processing_seconds = Histogram(
    'order_processing_seconds',
    'Time spent processing orders',
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0],
)

active_users = Gauge(
    'active_users_current',
    'Currently active users',
)
```

```python
# apps/orders/views.py
import time
from apps.core.metrics import order_created_total, order_processing_seconds


def create_order(request):
    start = time.time()
    try:
        order = Order.objects.create(...)
        order_created_total.labels(payment_method=order.payment_method, status='success').inc()
        return JsonResponse({'order_id': order.pk})
    except Exception:
        order_created_total.labels(payment_method='unknown', status='failed').inc()
        raise
    finally:
        order_processing_seconds.observe(time.time() - start)
```

---

## ขั้นตอนที่ 925: Prometheus + Grafana ด้วย Docker Compose

### 925.1 docker-compose.monitoring.yml

```yaml
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'django'
    static_configs:
      - targets: ['web:8000']
    metrics_path: '/metrics'
```

---

## ขั้นตอนที่ 926: ELK Stack Overview

### 926.1 Elasticsearch + Logstash + Kibana

```yaml
# docker-compose.elk.yml (simplified)
version: '3.9'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./monitoring/logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

```
# monitoring/logstash.conf
input {
  file {
    path => "/var/log/django/*.log"
    codec => json
  }
}

filter {
  if [levelname] == "ERROR" {
    mutate { add_tag => ["error"] }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "django-logs-%{+YYYY.MM.dd}"
  }
}
```

---

## ขั้นตอนที่ 927: Health Check Endpoint

### 927.1 Comprehensive Health Check

```python
# apps/core/views.py
from django.http import JsonResponse
from django.db import connection, DatabaseError
from django.core.cache import cache
from django.utils import timezone
import time


def health_check(request):
    """Health check สำหรับ load balancer และ monitoring"""
    checks = {}
    overall_healthy = True

    # Database
    try:
        start = time.monotonic()
        connection.ensure_connection()
        latency = round((time.monotonic() - start) * 1000, 2)
        checks['database'] = {'status': 'healthy', 'latency_ms': latency}
    except (DatabaseError, Exception) as e:
        checks['database'] = {'status': 'unhealthy', 'error': str(e)}
        overall_healthy = False

    # Cache
    try:
        start = time.monotonic()
        test_key = f'health_check_{int(time.time())}'
        cache.set(test_key, 'ok', timeout=5)
        assert cache.get(test_key) == 'ok'
        cache.delete(test_key)
        latency = round((time.monotonic() - start) * 1000, 2)
        checks['cache'] = {'status': 'healthy', 'latency_ms': latency}
    except Exception as e:
        checks['cache'] = {'status': 'unhealthy', 'error': str(e)}
        overall_healthy = False

    response_data = {
        'status': 'healthy' if overall_healthy else 'unhealthy',
        'timestamp': timezone.now().isoformat(),
        'checks': checks,
    }

    status_code = 200 if overall_healthy else 503
    return JsonResponse(response_data, status=status_code)
```

---

## ขั้นตอนที่ 928: Alerting

### 928.1 Prometheus Alerting Rules

```yaml
# monitoring/alerts.yml
groups:
  - name: django
    rules:
      - alert: DjangoHighErrorRate
        expr: rate(django_http_responses_total_by_status_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Django error rate สูง"
          description: "Error rate: {{ $value }} errors/sec"

      - alert: DjangoSlowRequests
        expr: histogram_quantile(0.95, rate(django_http_requests_latency_seconds_bucket[5m])) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Django response time ช้า (P95 > 2s)"
```

### 928.2 Uptime Monitoring

```python
# management/commands/check_uptime.py — รันด้วย cron
from django.core.management.base import BaseCommand
from django.core.mail import mail_admins
import urllib.request


class Command(BaseCommand):
    help = 'Check application uptime'

    def handle(self, *args, **options):
        try:
            with urllib.request.urlopen('https://example.com/health/', timeout=10) as r:
                if r.status != 200:
                    mail_admins('Uptime Alert', f'Health check returned {r.status}')
        except Exception as e:
            mail_admins('Uptime Alert', f'Health check failed: {e}')
```

---

## ขั้นตอนที่ 929: Performance Monitoring

### 929.1 django-silk สำหรับ Profiling

```bash
pip install django-silk
```

```python
# settings.py (development only)
INSTALLED_APPS = [
    ...
    'silk',
]

MIDDLEWARE = [
    ...
    'silk.middleware.SilkyMiddleware',
]

SILKY_PYTHON_PROFILER = True
SILKY_AUTHENTICATION = True
SILKY_AUTHORISATION = True
```

```python
# urls.py
urlpatterns = [
    path('silk/', include('silk.urls', namespace='silk')),
    ...
]
```

---

## ขั้นตอนที่ 930: สรุปและแบบฝึกหัด

### 930.1 Monitoring Stack

```
[Django Application]
   │
   ├── [Sentry] ← Error tracking + Performance
   ├── [/metrics] ← Prometheus scrapes
   │       └── [Grafana] ← Dashboards + Alerts
   └── [Log files] → [Logstash] → [Elasticsearch] → [Kibana]
```

### 930.2 Monitoring Checklist

- [x] Sentry ติดตั้งและ DSN ตั้งค่าแล้ว
- [x] Structured logging (JSON format) ตั้งค่าแล้ว
- [x] `/health/` endpoint ตรวจสอบ DB + Cache
- [x] Prometheus metrics ครอบคลุม requests + errors
- [x] Grafana dashboard สำหรับ key metrics
- [x] Alert เมื่อ error rate > threshold
- [x] Log rotation ตั้งค่าแล้ว
- [x] Uptime monitoring มีอยู่

### 930.3 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: ติดตั้ง Sentry SDK ใน Django project และทดสอบโดยเจตนา raise exception ดูว่า Sentry รับข้อมูลได้

**แบบฝึกหัดที่ 2**: สร้าง Prometheus custom metric ที่นับจำนวน user login และแสดงใน Grafana dashboard

**แบบฝึกหัดที่ 3**: สร้าง `/health/` endpoint ที่ return 503 เมื่อ database ไม่พร้อม และเขียน test ทดสอบทั้ง healthy + unhealthy cases
