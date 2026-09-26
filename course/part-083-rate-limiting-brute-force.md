# Part 083: Rate Limiting และ Brute Force Protection

> **ขั้นตอนที่ 821-830** | Phase 10: Security

---

## ขั้นตอนที่ 821: Rate Limiting ด้วย django-ratelimit

### 821.1 ติดตั้งและตั้งค่า

```bash
pip install django-ratelimit
```

```python
# apps/accounts/views.py
from django_ratelimit.decorators import ratelimit
from django_ratelimit.exceptions import Ratelimited
from django.shortcuts import render


@ratelimit(key='ip', rate='5/m', method='POST', block=True)
def login_view(request):
    """จำกัด login 5 ครั้งต่อนาทีต่อ IP"""
    if request.method == 'POST':
        ...
    return render(request, 'accounts/login.html')


@ratelimit(key='user', rate='100/h', method='GET', block=False)
def api_list_view(request):
    """จำกัด 100 ครั้งต่อชั่วโมงต่อ user"""
    was_limited = getattr(request, 'limited', False)
    if was_limited:
        return JsonResponse({'error': 'rate limit exceeded'}, status=429)
    ...
```

### 821.2 Handler สำหรับ Ratelimited Exception

```python
# apps/core/views.py
from django.shortcuts import render


def ratelimited_error(request, exception):
    return render(request, '429.html', status=429)
```

```python
# config/urls.py
handler429 = 'apps.core.views.ratelimited_error'
```

```html
<!-- templates/429.html -->
<!DOCTYPE html>
<html lang="th">
<head><title>429 Too Many Requests</title></head>
<body>
  <h1>คุณส่งคำขอมากเกินไป</h1>
  <p>กรุณารอสักครู่แล้วลองใหม่</p>
</body>
</html>
```

---

## ขั้นตอนที่ 822: django-axes สำหรับ Brute Force Protection

### 822.1 ติดตั้งและตั้งค่า

```bash
pip install django-axes
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'axes',
]

MIDDLEWARE = [
    ...
    'axes.middleware.AxesMiddleware',
]

AUTHENTICATION_BACKENDS = [
    'axes.backends.AxesStandaloneBackend',
    'django.contrib.auth.backends.ModelBackend',
]

# django-axes config
AXES_FAILURE_LIMIT = 5          # ล้มเหลวได้สูงสุด 5 ครั้ง
AXES_COOLOFF_TIME = 1           # lock 1 ชั่วโมง
AXES_RESET_ON_SUCCESS = True    # reset counter เมื่อ login สำเร็จ
AXES_LOCKOUT_URL = '/accounts/lockout/'
AXES_USE_USER_AGENT = True      # track user agent ด้วย
AXES_IP_BLACKLIST = []          # IP ที่ block ถาวร
AXES_IP_WHITELIST = ['127.0.0.1']  # IP ที่ไม่ต้อง lock
```

```bash
python manage.py migrate
```

### 822.2 ดู Access Attempts

```python
# python manage.py shell
from axes.models import AccessAttempt, AccessLog

# ดู attempts ทั้งหมด
AccessAttempt.objects.all()

# reset attempt สำหรับ IP นึง
from axes.utils import reset
reset(ip='1.2.3.4')
```

---

## ขั้นตอนที่ 823: Rate Limiting บน DRF (Django REST Framework)

### 823.1 Built-in DRF Throttling

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },
}
```

### 823.2 Custom Throttle Class

```python
# apps/api/throttling.py
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle


class LoginRateThrottle(AnonRateThrottle):
    scope = 'login'
    rate = '5/minute'


class SensitiveEndpointThrottle(UserRateThrottle):
    scope = 'sensitive'
    rate = '10/minute'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'login': '5/minute',
        'sensitive': '10/minute',
    },
}
```

```python
# apps/api/views.py
from rest_framework.views import APIView
from apps.api.throttling import LoginRateThrottle, SensitiveEndpointThrottle


class LoginAPIView(APIView):
    throttle_classes = [LoginRateThrottle]

    def post(self, request):
        ...


class ChangePasswordView(APIView):
    throttle_classes = [SensitiveEndpointThrottle]

    def post(self, request):
        ...
```

---

## ขั้นตอนที่ 824: Redis-based Rate Limiting

### 824.1 ใช้ Redis แทน Database

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}

# django-ratelimit ใช้ cache backend อัตโนมัติ
RATELIMIT_USE_CACHE = 'default'
```

### 824.2 Rate Limiter ด้วย Redis โดยตรง

```python
# apps/core/rate_limiter.py
import time
from django.core.cache import cache


class SlidingWindowRateLimiter:
    """Sliding window rate limiter ด้วย Redis sorted set"""

    def __init__(self, key: str, limit: int, window: int):
        self.key = f'ratelimit:{key}'
        self.limit = limit
        self.window = window  # seconds

    def is_allowed(self) -> tuple[bool, int]:
        now = time.time()
        window_start = now - self.window

        # ตัวอย่างด้วย redis-py โดยตรง
        import redis
        r = redis.Redis(host='localhost', port=6379, db=1)
        pipe = r.pipeline()
        pipe.zremrangebyscore(self.key, 0, window_start)
        pipe.zadd(self.key, {str(now): now})
        pipe.zcard(self.key)
        pipe.expire(self.key, self.window)
        results = pipe.execute()

        count = results[2]
        remaining = max(0, self.limit - count)
        return count <= self.limit, remaining
```

---

## ขั้นตอนที่ 825: Account Lockout และ Progressive Delay

### 825.1 Progressive Delay หลัง Login ล้มเหลว

```python
# apps/accounts/backends.py
import time
from django.contrib.auth.backends import ModelBackend
from django.core.cache import cache


class ProgressiveDelayBackend(ModelBackend):
    """เพิ่ม delay หลัง login ล้มเหลว"""

    def authenticate(self, request, username=None, password=None, **kwargs):
        if username is None:
            return None

        attempt_key = f'login_attempt:{username}'
        attempts = cache.get(attempt_key, 0)

        if attempts > 0:
            delay = min(2 ** (attempts - 1), 30)  # max 30 วินาที
            time.sleep(delay)

        user = super().authenticate(request, username=username, password=password, **kwargs)

        if user is None:
            cache.set(attempt_key, attempts + 1, timeout=3600)
        else:
            cache.delete(attempt_key)

        return user
```

### 825.2 Account Lock หลัง N ครั้ง

```python
# apps/accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.utils import timezone
from datetime import timedelta


class User(AbstractUser):
    failed_login_attempts = models.IntegerField(default=0)
    locked_until = models.DateTimeField(null=True, blank=True)

    def is_locked(self) -> bool:
        if self.locked_until and timezone.now() < self.locked_until:
            return True
        return False

    def record_failed_login(self):
        self.failed_login_attempts += 1
        if self.failed_login_attempts >= 5:
            self.locked_until = timezone.now() + timedelta(hours=1)
        self.save(update_fields=['failed_login_attempts', 'locked_until'])

    def reset_failed_logins(self):
        self.failed_login_attempts = 0
        self.locked_until = None
        self.save(update_fields=['failed_login_attempts', 'locked_until'])
```

---

## ขั้นตอนที่ 826: CAPTCHA Integration

### 826.1 django-simple-captcha

```bash
pip install django-simple-captcha
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'captcha',
]
```

```python
# apps/accounts/forms.py
from django import forms
from captcha.fields import CaptchaField


class LoginForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
    captcha = CaptchaField()  # แสดงเมื่อ failed attempts > 3


class CaptchaLoginForm(forms.Form):
    """แสดง CAPTCHA เสมอสำหรับ endpoint ที่ sensitive"""
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
    captcha = CaptchaField()
```

### 826.2 hCaptcha / reCAPTCHA

```bash
pip install django-hcaptcha
```

```python
# settings.py
HCAPTCHA_SITEKEY = 'your-site-key'
HCAPTCHA_SECRET = 'your-secret-key'
```

```python
# apps/accounts/forms.py
from hcaptcha.fields import hCaptchaField

class RegisterForm(forms.Form):
    ...
    hcaptcha = hCaptchaField()
```

---

## ขั้นตอนที่ 827: IP Blocking และ Geo-blocking

### 827.1 IP Block ด้วย Middleware

```python
# apps/core/middleware.py
from django.core.cache import cache
from django.http import HttpResponseForbidden
import ipaddress


BLOCKED_IP_RANGES = [
    # เพิ่ม CIDR ranges ที่ต้องการ block
]


class IPBlockingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.blocked_ranges = [ipaddress.ip_network(r) for r in BLOCKED_IP_RANGES]

    def __call__(self, request):
        ip = self._get_client_ip(request)

        # ตรวจสอบ dynamic block list จาก cache
        if cache.get(f'blocked_ip:{ip}'):
            return HttpResponseForbidden('Access denied')

        # ตรวจสอบ static block ranges
        try:
            client_ip = ipaddress.ip_address(ip)
            for network in self.blocked_ranges:
                if client_ip in network:
                    return HttpResponseForbidden('Access denied')
        except ValueError:
            pass

        return self.get_response(request)

    def _get_client_ip(self, request) -> str:
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR', '')


def block_ip(ip: str, duration: int = 3600):
    """Block IP ชั่วคราว (default 1 ชั่วโมง)"""
    cache.set(f'blocked_ip:{ip}', True, timeout=duration)
```

---

## ขั้นตอนที่ 828: Monitoring และ Alerting

### 828.1 Log Failed Login Attempts

```python
# apps/accounts/signals.py
from django.contrib.auth.signals import user_login_failed, user_logged_in
from django.dispatch import receiver
import logging

logger = logging.getLogger('security')


@receiver(user_login_failed)
def log_failed_login(sender, credentials, request, **kwargs):
    ip = request.META.get('REMOTE_ADDR')
    username = credentials.get('username', 'unknown')
    logger.warning(
        'Failed login attempt',
        extra={
            'ip': ip,
            'username': username,
            'user_agent': request.META.get('HTTP_USER_AGENT', ''),
        }
    )


@receiver(user_logged_in)
def log_successful_login(sender, request, user, **kwargs):
    ip = request.META.get('REMOTE_ADDR')
    logger.info(
        'Successful login',
        extra={'ip': ip, 'user_id': user.pk, 'username': user.username}
    )
```

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'security_file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': '/var/log/django/security.log',
        },
    },
    'loggers': {
        'security': {
            'handlers': ['security_file'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}
```

---

## ขั้นตอนที่ 829: Rate Limit Headers

### 829.1 ส่ง Rate Limit Headers ใน Response

```python
# apps/api/middleware.py
from django_ratelimit.exceptions import Ratelimited


class RateLimitHeadersMiddleware:
    """เพิ่ม X-RateLimit headers ใน response"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        return response

    def process_exception(self, request, exception):
        if isinstance(exception, Ratelimited):
            from django.http import JsonResponse
            response = JsonResponse(
                {'error': 'rate_limit_exceeded', 'message': 'Too many requests'},
                status=429
            )
            response['X-RateLimit-Limit'] = '5'
            response['X-RateLimit-Remaining'] = '0'
            response['Retry-After'] = '60'
            return response
```

---

## ขั้นตอนที่ 830: สรุปและแบบฝึกหัด

### 830.1 Rate Limiting Strategy

| Endpoint | Rate | Key | Action |
|----------|------|-----|--------|
| Login | 5/min | IP | Block + CAPTCHA |
| Register | 3/hour | IP | Block |
| Password Reset | 3/hour | IP + Email | Block |
| API (anon) | 100/day | IP | 429 Response |
| API (user) | 1000/day | User | 429 Response |
| Admin | 10/min | IP | Block |

### 830.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: ติดตั้ง django-axes และกำหนดให้ lock account 30 นาทีหลัง login ผิด 5 ครั้ง แล้วเขียน test ตรวจสอบ

**แบบฝึกหัดที่ 2**: สร้าง custom DRF throttle class ที่มี rate ต่างกันสำหรับ free/premium users

**แบบฝึกหัดที่ 3**: เขียน middleware ที่ auto-block IP ที่ trigger rate limit เกิน 3 ครั้งใน 10 นาที
