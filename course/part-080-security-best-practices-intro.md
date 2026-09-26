# Part 080: Django Security Best Practices เบื้องต้น

## ขั้นตอนที่ 791-800

---

## ภาพรวม Phase 10: Security

ถึง **Phase 10: Security** แล้ว ซึ่งเป็นหนึ่งใน Phase ที่สำคัญที่สุดของหลักสูตรนี้
Django ถูกออกแบบมาพร้อมกลไกความปลอดภัยในตัวที่ครอบคลุม แต่การกำหนดค่าที่ถูกต้องและ
การเข้าใจแต่ละ Setting อย่างถ่องแท้คือหน้าที่ของ Developer

Phase นี้ครอบคลุม:

| Part | หัวข้อ | ขั้นตอน |
|------|--------|---------|
| 080 | Django Security Best Practices เบื้องต้น | 791-800 |
| 081 | CSRF, XSS และ SQL Injection Prevention | 801-810 |
| 082 | Security Headers และ HTTPS | 811-820 |
| 083 | Rate Limiting และ Brute Force Protection | 821-830 |
| 084 | Secrets Management และ Environment Variables | 831-840 |
| 085 | Security Auditing และ Penetration Testing เบื้องต้น | 841-850 |

---

## ขั้นตอนที่ 791: Django Security Philosophy — "Batteries Included, Properly Configured"

Django ยึดหลัก **"Secure by Default"** ซึ่งหมายความว่า กลไกความปลอดภัยส่วนใหญ่
เปิดใช้งานอยู่ในตัวตั้งแต่ต้น แต่ต้องการการกำหนดค่าที่ถูกต้องสำหรับสภาพแวดล้อม Production

### หลักการพื้นฐาน

**1. Defense in Depth (การป้องกันแบบหลายชั้น)**

```
Request ─► SecurityMiddleware ─► CsrfViewMiddleware ─► AuthenticationMiddleware
           ↓                     ↓                      ↓
        HTTPS Redirect         Token Check           User Attachment
        HSTS Header            (ก่อน View)           (ใช้ใน View)
```

Django ไม่ได้พึ่งพาการป้องกันชั้นเดียว แต่มีหลายชั้นที่ทำงานร่วมกัน

**2. กลไกที่เปิดใช้งานโดยอัตโนมัติ**

| กลไก | Middleware/Setting | สถานะเริ่มต้น |
|------|-------------------|--------------|
| CSRF Protection | `CsrfViewMiddleware` | เปิด |
| XSS Prevention via Autoescape | Template Engine | เปิด |
| SQL Parameterization | ORM | เปิดเสมอ |
| Click-jacking Protection | `XFrameOptionsMiddleware` | เปิด |
| Content-Type Sniffing Prevention | `SecurityMiddleware` | ต้องตั้งค่า |
| HTTPS Redirect | `SecurityMiddleware` | ต้องตั้งค่า |

**3. Django Security Release Policy**

Django ออก Security Releases ตาม [Django Security Policy](https://docs.djangoproject.com/en/stable/internals/security/)
เมื่อมีการค้นพบ vulnerability ใน Django หรือ library ที่ Django ใช้

```bash
# ติดตาม Django Security Announcements
# Subscribe ที่: https://groups.google.com/g/django-announce

# ตรวจสอบ Security Updates ด้วย pip-audit
pip install pip-audit
pip-audit
```

---

## ขั้นตอนที่ 792: `python manage.py check --deploy` — ทำความเข้าใจทุก Warning

Django มี built-in deployment checklist ที่รันผ่าน management command นี้
ควรรันทุกครั้งก่อน deploy ขึ้น Production

### วิธีรัน

```bash
# รันใน local แต่จำลอง Production settings
DJANGO_SETTINGS_MODULE=myproject.settings.production \
    python manage.py check --deploy

# หรือถ้าใช้ environment variable
python manage.py check --deploy --settings=myproject.settings.production
```

### ผลลัพธ์ตัวอย่าง (เมื่อยังไม่ได้กำหนดค่า)

```
System check identified some issues:

WARNINGS:
?: (security.W001) You do not have 'django.middleware.security.SecurityMiddleware'
   in your MIDDLEWARE so the SECURE_HSTS_SECONDS,
   SECURE_HSTS_PRELOAD, SECURE_HSTS_INCLUDE_SUBDOMAINS,
   SECURE_CONTENT_TYPE_NOSNIFF, SECURE_BROWSER_XSS_FILTER,
   SECURE_SSL_REDIRECT, and SECURE_REDIRECT_EXEMPT settings will have no effect.
   
?: (security.W002) You do not have 'django.middleware.clickjacking.XFrameOptionsMiddleware'
   in your MIDDLEWARE, so your pages will not be served with an
   'x-frame-options' header. Unless there is another part of your stack that
   provides clickjacking protection, we strongly encourage you to add it.
   
?: (security.W003) You don't appear to be using Django's built-in cross-site
   request forgery protection via the middleware
   ('django.middleware.csrf.CsrfViewMiddleware' is not in your MIDDLEWARE).
   Enabling CSRF protection is strongly encouraged.
   
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting.
   If your entire site is served only over SSL, we suggest setting a value and
   enabling HTTP Strict Transport Security. Be sure to read the documentation
   first; enabling HSTS carelessly can cause serious, irreversible problems.
   
?: (security.W006) Your SECURE_CONTENT_TYPE_NOSNIFF setting is not set to True,
   so your pages will not be served with an 'X-Content-Type-Options: nosniff'
   header.
   
?: (security.W007) Your SECURE_BROWSER_XSS_FILTER setting is not set to True,
   so your pages will not be served with an 'X-XSS-Protection: 1; mode=block'
   header.

?: (security.W008) Your SECRET_KEY has less than 50 characters, less than 5
   unique characters, or it is a well-known value like 'django-insecure-'.
   Please generate a long and random value, otherwise many of Django's
   security-critical features will be vulnerable.
   
?: (security.W009) Your SECRET_KEY_FALLBACKS contains a value that has less
   than 50 characters...

?: (security.W012) SESSION_COOKIE_SECURE is not set to True. Using a
   secure-only session cookie makes it more difficult for network traffic
   sniffers to hijack user sessions.

?: (security.W016) You have 'django.middleware.csrf.CsrfViewMiddleware' in your
   MIDDLEWARE, but you have not set CSRF_COOKIE_SECURE to True. Using a
   secure-only CSRF cookie makes it more difficult for network traffic
   sniffers to steal the CSRF token.

?: (security.W018) You should not have DEBUG set to True in deployment.

?: (security.W019) You have 'django.middleware.clickjacking.XFrameOptionsMiddleware'
   in your MIDDLEWARE, but X_FRAME_OPTIONS is not set to 'DENY'.
   The default is 'SAMEORIGIN', but unless there is a good reason for your
   site to serve other parts of itself in a frame, you should change it to
   'DENY'.
   
System check identified 12 issues (silenced: 0).
```

### รหัส Warning ทั้งหมดและความหมาย

| รหัส | ความหมาย | การแก้ไข |
|------|---------|----------|
| `security.W001` | ไม่มี `SecurityMiddleware` | เพิ่มใน MIDDLEWARE |
| `security.W002` | ไม่มี `XFrameOptionsMiddleware` | เพิ่มใน MIDDLEWARE |
| `security.W003` | ไม่มี `CsrfViewMiddleware` | เพิ่มใน MIDDLEWARE |
| `security.W004` | ไม่ได้ตั้ง `SECURE_HSTS_SECONDS` | ตั้งค่า (ดู Part 082) |
| `security.W005` | `SECURE_HSTS_INCLUDE_SUBDOMAINS` = False | ตั้งค่าตาม domain structure |
| `security.W006` | `SECURE_CONTENT_TYPE_NOSNIFF` = False | ตั้งเป็น True |
| `security.W007` | `SECURE_BROWSER_XSS_FILTER` = False | ตั้งเป็น True |
| `security.W008` | SECRET_KEY สั้นหรืออ่อนแอ | สร้าง SECRET_KEY ใหม่ |
| `security.W009` | SECRET_KEY_FALLBACKS มีค่าอ่อนแอ | ตรวจสอบ fallback keys |
| `security.W012` | `SESSION_COOKIE_SECURE` = False | ตั้งเป็น True |
| `security.W016` | `CSRF_COOKIE_SECURE` = False | ตั้งเป็น True |
| `security.W018` | `DEBUG` = True | ตั้งเป็น False |
| `security.W019` | `X_FRAME_OPTIONS` ≠ 'DENY' | พิจารณาตั้งเป็น 'DENY' |
| `security.W020` | `ALLOWED_HOSTS` ว่างเปล่า | กำหนด hosts ที่อนุญาต |
| `security.W021` | `SECURE_SSL_REDIRECT` = False | ตั้งเป็น True (ถ้าใช้ HTTPS) |
| `security.E001` | `SILENCED_SYSTEM_CHECKS` มีรหัส security | ตรวจสอบและลบออก |

### ทำให้ผ่านทุก Check

```bash
# เป้าหมาย: ไม่มี warning ใด ๆ
python manage.py check --deploy

# System check identified no issues (silenced: 0).
```

---

## ขั้นตอนที่ 793: SECRET_KEY — การจัดการและการสร้าง Key ที่ปลอดภัย

### SECRET_KEY ใช้ทำอะไร

Django ใช้ `SECRET_KEY` สำหรับการดำเนินการ cryptographic หลายอย่าง:

| การใช้งาน | รายละเอียด |
|----------|------------|
| Session signing | เซ็น session cookies เพื่อป้องกันการปลอมแปลง |
| CSRF tokens | สร้างและตรวจสอบ CSRF tokens |
| Password reset tokens | สร้าง one-time tokens สำหรับ password reset |
| Email confirmation tokens | สร้าง tokens สำหรับ email verification |
| `django.contrib.messages` | เซ็น message cookies |
| `signing.dumps()` / `signing.loads()` | สำหรับ custom signed data |

### ข้อกำหนดของ SECRET_KEY

จาก Django documentation:

- **ความยาว**: ต้องมีอย่างน้อย 50 ตัวอักษร
- **ความหลากหลาย**: ต้องมีอักขระ unique อย่างน้อย 5 ตัว
- **ค่าที่ห้ามใช้**: ค่าที่ขึ้นต้นด้วย `django-insecure-` (ค่าที่ Django สร้างให้ใน development)
- **ความลับ**: ไม่ควร commit ลงใน version control

### การสร้าง SECRET_KEY ใหม่

```python
# วิธีที่ 1: ใช้ Django built-in
from django.core.management.utils import get_random_secret_key
print(get_random_secret_key())
# ตัวอย่าง output:
# 9@d$8^2#k+p5v7m!nqr0s1t3u4w6x8y=z-abc

# วิธีที่ 2: ใช้ Python secrets module
import secrets
import string
alphabet = string.ascii_letters + string.digits + string.punctuation
secret_key = ''.join(secrets.choice(alphabet) for _ in range(64))
print(secret_key)

# วิธีที่ 3: ใช้ Django management command
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

### SECRET_KEY_FALLBACKS — การ Rotate Key

ตั้งแต่ Django 4.1 เป็นต้นมา Django รองรับการ rotate SECRET_KEY โดยไม่ทำให้ session ของ user ที่ login อยู่หมดอายุทันที

```python
# settings/production.py

# Key ปัจจุบัน (ใหม่)
SECRET_KEY = env('DJANGO_SECRET_KEY')

# Key เก่า (ใช้ verify ของที่ signed ด้วย key เก่า)
SECRET_KEY_FALLBACKS = [
    env('DJANGO_SECRET_KEY_FALLBACK_1', default=''),
]
# เมื่อ user request ครั้งถัดไป Django จะ resign ด้วย key ใหม่
# หลังจากนั้นสักพัก ลบ key เก่าออกจาก fallbacks ได้
```

### ขั้นตอนการ Rotate SECRET_KEY อย่างปลอดภัย

```
1. สร้าง SECRET_KEY ใหม่
2. ย้าย SECRET_KEY เดิม → SECRET_KEY_FALLBACKS
3. ตั้ง SECRET_KEY ใหม่
4. Deploy
5. รอให้ session ทั้งหมด expire (หรือ force logout ทุก user)
6. ลบ key เดิมออกจาก SECRET_KEY_FALLBACKS
7. Deploy อีกครั้ง
```

---

## ขั้นตอนที่ 794: DEBUG และ ALLOWED_HOSTS สำหรับ Production

### DEBUG = False

การตั้ง `DEBUG = True` ใน Production เป็นความเสี่ยงสำคัญ:

| ข้อมูลที่เปิดเผยเมื่อ DEBUG=True | ผลกระทบ |
|--------------------------------|---------|
| Full stack traces พร้อม local variables | เปิดเผย internal code structure |
| Database queries ทั้งหมด | เปิดเผย data model และ query patterns |
| Settings ที่ใช้อยู่ | เปิดเผย configuration (แต่ Django ซ่อน SECRET_KEY) |
| Template paths | เปิดเผย server file structure |
| Installed apps และ middleware | เปิดเผย stack information |

**เมื่อ `DEBUG = False`:** Django แสดง generic error pages (404, 500) แทน
Django ยังส่ง email ไปยัง `ADMINS` เมื่อเกิด 500 error (ถ้าตั้งค่า email)

```python
# settings/production.py
DEBUG = False

# ต้องตั้ง ADMINS เพื่อรับ error emails
ADMINS = [
    ('Admin Name', 'admin@example.com'),
]

# Server email ที่ใช้ส่ง error notification
SERVER_EMAIL = 'django-errors@example.com'
```

### ALLOWED_HOSTS

`ALLOWED_HOSTS` เป็น whitelist ของ hostnames ที่ Django จะ serve requests ให้
หากไม่ตั้งค่า (หรือตั้งเป็น `[]`) Django จะ reject ทุก request เมื่อ `DEBUG = False`

```python
# settings/production.py

# ตั้งค่าพื้นฐาน
ALLOWED_HOSTS = [
    'www.example.com',
    'example.com',
    'api.example.com',
]

# รับจาก environment variable (แนะนำ)
import os
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')
# ตั้งค่าใน .env:
# ALLOWED_HOSTS=www.example.com,example.com

# สำหรับ Kubernetes หรือ load balancer ที่มี internal health checks
ALLOWED_HOSTS = [
    'www.example.com',
    '10.0.0.0/24',      # *** Django 4.x: รองรับ CIDR notation (แต่ใช้ระวัง)
]
```

**Note เรื่อง wildcard:** `ALLOWED_HOSTS = ['*']` เป็นการปิดการตรวจสอบ ห้ามใช้ใน Production
ยกเว้นกรณีที่มี reverse proxy ด้านหน้าที่ตรวจสอบ Host header แทนอยู่แล้ว

---

## ขั้นตอนที่ 795: Django Security Middleware Stack

`django.middleware.security.SecurityMiddleware` คือ middleware ที่จัดการกับ
HTTP security headers และ redirect policies ต่าง ๆ

### ตำแหน่งใน MIDDLEWARE

```python
# settings/base.py
MIDDLEWARE = [
    # SecurityMiddleware ต้องอยู่ก่อน middleware อื่น ๆ ทั้งหมด
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

### Security-related Middleware และหน้าที่

| Middleware | หน้าที่หลัก |
|-----------|-------------|
| `SecurityMiddleware` | HTTPS redirect, HSTS, content-type nosniff, XSS filter header |
| `SessionMiddleware` | Session management |
| `CsrfViewMiddleware` | CSRF token validation |
| `AuthenticationMiddleware` | Attach `request.user` จาก session |
| `XFrameOptionsMiddleware` | ส่ง X-Frame-Options header |

### SecurityMiddleware ทำอะไรบ้าง

**Source code ของ SecurityMiddleware (สรุป):**

```python
# django/middleware/security.py (simplified)
class SecurityMiddleware(MiddlewareMixin):
    def process_request(self, request):
        # 1. HTTPS redirect
        if settings.SECURE_SSL_REDIRECT and not request.is_secure():
            if request.path not in settings.SECURE_REDIRECT_EXEMPT:
                return HttpResponsePermanentRedirect(
                    "https://" + request.get_host() + request.get_full_path()
                )
    
    def process_response(self, request, response):
        # 2. HSTS header
        if settings.SECURE_HSTS_SECONDS and request.is_secure():
            response['Strict-Transport-Security'] = (
                f'max-age={settings.SECURE_HSTS_SECONDS}'
                + ('; includeSubDomains' if settings.SECURE_HSTS_INCLUDE_SUBDOMAINS else '')
                + ('; preload' if settings.SECURE_HSTS_PRELOAD else '')
            )
        
        # 3. X-Content-Type-Options
        if settings.SECURE_CONTENT_TYPE_NOSNIFF:
            response.setdefault('X-Content-Type-Options', 'nosniff')
        
        # 4. Referrer-Policy (Django 3.x+)
        if settings.SECURE_REFERRER_POLICY:
            response.setdefault('Referrer-Policy', settings.SECURE_REFERRER_POLICY)
        
        # 5. Cross-Origin-Opener-Policy (Django 4.x+)
        if settings.SECURE_CROSS_ORIGIN_OPENER_POLICY:
            response.setdefault(
                'Cross-Origin-Opener-Policy',
                settings.SECURE_CROSS_ORIGIN_OPENER_POLICY,
            )
        
        return response
```

---

## ขั้นตอนที่ 796: SECURE_* Settings — Reference ครบทุก Setting

### SECURE_SSL_REDIRECT

```python
# Redirect HTTP → HTTPS
SECURE_SSL_REDIRECT = True  # default: False

# Exempt บาง URL จาก redirect (เช่น health check endpoint ที่ต้องรับ HTTP)
SECURE_REDIRECT_EXEMPT = [
    r'^health/$',
    r'^ping/$',
]

# ถ้าใช้ Load Balancer / Reverse Proxy ที่ terminate SSL แทน
# Django จะเห็น request เป็น HTTP เสมอ → ต้องบอก Django ว่า proxy บอกว่า HTTPS ผ่าน header
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
# ตรงกับ Nginx config: proxy_set_header X-Forwarded-Proto $scheme;
```

### SECURE_HSTS_SECONDS

```python
# HTTP Strict Transport Security
# บอก browser ให้ใช้ HTTPS เท่านั้นตลอด max-age วินาที
SECURE_HSTS_SECONDS = 31536000  # 1 ปี (recommended)

# เริ่มต้นทดสอบด้วยค่าน้อย ๆ ก่อน
SECURE_HSTS_SECONDS = 300  # 5 นาที สำหรับ testing

# *** Warning: เมื่อตั้งแล้ว browser จะจำค่านี้ไว้
# ถ้าต้องการกลับมาใช้ HTTP ต้องรอ max-age หมดก่อน
# ดู Part 082 สำหรับรายละเอียด HSTS
```

### SECURE_HSTS_INCLUDE_SUBDOMAINS

```python
# รวม subdomains ทั้งหมดใน HSTS policy
# ต้องแน่ใจว่า HTTPS ทุก subdomain ก่อนตั้งค่านี้
SECURE_HSTS_INCLUDE_SUBDOMAINS = True  # default: False
```

### SECURE_HSTS_PRELOAD

```python
# ส่งไปยัง HSTS Preload List (browser built-in list)
# ต้องมี SECURE_HSTS_SECONDS >= 31536000 และ SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True  # default: False

# *** Irreversible: เมื่อ submit ไปยัง preload list แล้ว
# กระบวนการลบออกใช้เวลาหลายเดือน
```

### SECURE_CONTENT_TYPE_NOSNIFF

```python
# ส่ง header: X-Content-Type-Options: nosniff
# ป้องกัน browser จาก MIME type sniffing
SECURE_CONTENT_TYPE_NOSNIFF = True  # default: True (ตั้งแต่ Django 3.0)
```

### SECURE_REFERRER_POLICY

```python
# ควบคุม Referer header ที่ browser ส่งเมื่อ navigate
SECURE_REFERRER_POLICY = 'strict-origin-when-cross-origin'  # default (Django 3.1+)

# ค่าที่เป็นไปได้:
# 'no-referrer'                     — ไม่ส่ง Referer เลย
# 'no-referrer-when-downgrade'      — ไม่ส่งเมื่อ HTTPS → HTTP
# 'origin'                          — ส่งแค่ origin เท่านั้น
# 'origin-when-cross-origin'        — ส่ง full URL ภายใน origin, origin เท่านั้น cross-origin
# 'same-origin'                     — ส่งเฉพาะ same-origin requests
# 'strict-origin'                   — ส่ง origin เฉพาะ HTTPS→HTTPS
# 'strict-origin-when-cross-origin' — recommended
# 'unsafe-url'                      — ส่ง full URL เสมอ (ไม่แนะนำ)
```

### SECURE_CROSS_ORIGIN_OPENER_POLICY

```python
# Cross-Origin-Opener-Policy header (Django 4.0+)
# แยก browsing context เพื่อป้องกัน cross-origin information leaks
SECURE_CROSS_ORIGIN_OPENER_POLICY = 'same-origin'  # default (Django 4.0+)

# ค่าที่เป็นไปได้:
# 'same-origin'           — ไม่ share browsing context กับ cross-origin pages
# 'same-origin-allow-popups'  — อนุญาต popups
# 'unsafe-none'           — ไม่มีการแยก (default behavior เก่า)
```

---

## ขั้นตอนที่ 797: Cookie Security Settings

Cookie ที่ไม่ปลอดภัยเปิดช่องให้ถูก intercept หรือ steal ได้

### Session Cookie Settings

```python
# settings/production.py

# ส่ง session cookie ผ่าน HTTPS เท่านั้น
SESSION_COOKIE_SECURE = True  # default: False

# ป้องกัน JavaScript เข้าถึง session cookie (HttpOnly flag)
# Django ตั้งค่านี้เป็น True ตั้งแต่ต้น
SESSION_COOKIE_HTTPONLY = True  # default: True

# SameSite policy สำหรับ session cookie
# 'Strict' — ส่ง cookie เฉพาะ same-site requests (browser navigation)
# 'Lax'    — ส่ง cookie สำหรับ top-level navigation, ไม่ส่งสำหรับ cross-site sub-requests (default)
# 'None'   — ส่งทุก request (ต้องใช้ร่วมกับ Secure flag)
# False    — ไม่ส่ง SameSite header (ไม่แนะนำ)
SESSION_COOKIE_SAMESITE = 'Lax'  # default: 'Lax' (Django 3.1+)

# ชื่อ session cookie (อาจเปลี่ยนเพื่อลด fingerprinting)
SESSION_COOKIE_NAME = 'sessionid'  # default

# อายุของ session cookie (วินาที) ถ้าไม่ตั้งจะใช้ browser session
SESSION_COOKIE_AGE = 1209600  # 2 สัปดาห์ (default)

# ถ้า True → cookie ถูก update ทุก request (ต่อ expiry)
SESSION_SAVE_EVERY_REQUEST = False  # default

# ถ้า True → ใช้ browser session (session หายเมื่อปิด browser)
SESSION_EXPIRE_AT_BROWSER_CLOSE = False  # default
```

### CSRF Cookie Settings

```python
# ส่ง CSRF cookie ผ่าน HTTPS เท่านั้น
CSRF_COOKIE_SECURE = True  # default: False

# ป้องกัน JavaScript เข้าถึง CSRF cookie
CSRF_COOKIE_HTTPONLY = False  # default: False (ต้องให้ JS อ่านได้สำหรับ AJAX)

# SameSite policy สำหรับ CSRF cookie
CSRF_COOKIE_SAMESITE = 'Lax'  # default: 'Lax'

# อายุ CSRF cookie (วินาที)
CSRF_COOKIE_AGE = 31449600  # 1 ปี (default)

# ชื่อ CSRF cookie
CSRF_COOKIE_NAME = 'csrftoken'  # default

# Domain ที่ CSRF cookie ใช้ได้
CSRF_COOKIE_DOMAIN = None  # default: ใช้ domain ปัจจุบัน
# ถ้าต้องการ share CSRF cookie ระหว่าง subdomain:
# CSRF_COOKIE_DOMAIN = '.example.com'  # note: ขึ้นต้นด้วย dot
```

---

## ขั้นตอนที่ 798: X_FRAME_OPTIONS และ Clickjacking Protection

`XFrameOptionsMiddleware` ส่ง `X-Frame-Options` header เพื่อควบคุมว่า
page นี้สามารถแสดงใน `<iframe>`, `<frame>`, หรือ `<object>` ได้หรือไม่

### Settings

```python
# settings/production.py

# X-Frame-Options header value
X_FRAME_OPTIONS = 'DENY'         # ห้ามทุก framing (แนะนำสำหรับส่วนใหญ่)
# X_FRAME_OPTIONS = 'SAMEORIGIN'  # อนุญาตเฉพาะ same-origin (default)
```

### Per-View Override

บางครั้งต้องการให้บาง page ฝังใน iframe ได้ (เช่น embedded widget):

```python
from django.views.decorators.clickjacking import (
    xframe_options_deny,
    xframe_options_sameorigin,
    xframe_options_exempt,
)

# ห้ามทุก framing สำหรับ view นี้โดยเฉพาะ
@xframe_options_deny
def sensitive_page(request):
    ...

# อนุญาตให้ same-origin embed
@xframe_options_sameorigin
def embeddable_widget(request):
    ...

# ยกเว้น view นี้จาก X-Frame-Options header
# (ใช้เมื่อต้องการให้ third-party embed ได้)
@xframe_options_exempt
def publicly_embeddable(request):
    ...
```

### Content Security Policy (CSP) สำหรับ Framing

สำหรับ browser รุ่นใหม่ `frame-ancestors` ใน CSP มีความสามารถมากกว่า X-Frame-Options:

```python
# ใช้ django-csp (ดู Part 082 สำหรับรายละเอียดเต็ม)
CONTENT_SECURITY_POLICY = {
    "DIRECTIVES": {
        "frame-ancestors": ["'self'"],  # แทน X-Frame-Options: SAMEORIGIN
        # "frame-ancestors": ["'none'"],  # แทน X-Frame-Options: DENY
    }
}
```

---

## ขั้นตอนที่ 799: Django Security Checklist สำหรับ Code Review

### Pre-deployment Checklist

ใช้ checklist นี้ก่อน deploy ทุกครั้ง:

```
Django Security Deployment Checklist
=====================================

[ ] python manage.py check --deploy → ไม่มี warning ใด ๆ
[ ] DEBUG = False
[ ] SECRET_KEY สร้างด้วย get_random_secret_key() ไม่ได้ขึ้นต้นด้วย 'django-insecure-'
[ ] SECRET_KEY ไม่ได้ commit ใน version control
[ ] SECRET_KEY ยาวกว่า 50 ตัวอักษร มี unique characters อย่างน้อย 5 ตัว
[ ] ALLOWED_HOSTS ไม่มี wildcard '*'
[ ] SECURE_SSL_REDIRECT = True หรือ reverse proxy ทำ redirect แทน
[ ] SECURE_HSTS_SECONDS ≥ 300 (production: ≥ 31536000)
[ ] SESSION_COOKIE_SECURE = True
[ ] CSRF_COOKIE_SECURE = True
[ ] SECURE_CONTENT_TYPE_NOSNIFF = True
[ ] X_FRAME_OPTIONS = 'DENY' หรือ 'SAMEORIGIN'
[ ] ไม่ commit ไฟล์ .env ใน git
[ ] ใช้ environment variables สำหรับ credentials ทั้งหมด
[ ] DATABASES password ไม่ hard-code ใน settings
[ ] Email backend ไม่ใช้ ConsoleEmailBackend ใน production
[ ] STATIC_ROOT และ MEDIA_ROOT อยู่นอก project directory
[ ] Logging configuration ไม่ log sensitive data
```

### Code Review Checklist

```
Django Code Security Review
============================

[ ] ทุก view ที่ modify data มี CSRF protection (ใช้ @csrf_protect หรือ CsrfViewMiddleware)
[ ] ทุก view ที่ต้องการ auth มี @login_required หรือ LoginRequiredMixin
[ ] ไม่ใช้ raw() หรือ extra() โดยไม่มีเหตุผลจำเป็น
[ ] ถ้าใช้ raw()/extra() → ใช้ parameterized queries เสมอ (ไม่ใช้ string formatting)
[ ] Template autoescape เปิดอยู่ (default) ไม่ได้ปิดด้วย {% autoescape off %}
[ ] ไม่ใช้ {{ variable|safe }} โดยไม่ตรวจสอบ
[ ] ไม่ใช้ mark_safe() กับ user input
[ ] File upload validation มีการตรวจสอบ MIME type และ extension
[ ] Media files ไม่ serve จาก URL ที่ execute ได้
[ ] User input ไม่นำไปใช้ใน redirect URL โดยตรง (ต้องตรวจสอบ domain)
[ ] ไม่ log passwords, tokens, หรือ sensitive data
[ ] Permission check ทำที่ view layer ไม่ใช่แค่ template
```

---

## ขั้นตอนที่ 800: Capstone — สร้าง Hardened settings.py สำหรับ Production

### โครงสร้าง Settings ที่แนะนำ

```
myproject/
└── settings/
    ├── __init__.py       (ว่างเปล่า หรือ import base)
    ├── base.py           (settings พื้นฐานที่ใช้ทุก environment)
    ├── development.py    (settings สำหรับ local development)
    ├── production.py     (settings สำหรับ production)
    └── testing.py        (settings สำหรับ test suite)
```

### base.py — Settings พื้นฐาน

```python
# settings/base.py
from pathlib import Path
import os
import environ

BASE_DIR = Path(__file__).resolve().parent.parent.parent

# django-environ สำหรับอ่าน environment variables
env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)

# อ่าน .env file (ถ้ามี)
environ.Env.read_env(BASE_DIR / '.env')

# --- Core Settings ---
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third-party
    # Local apps
    'blog',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'myproject.urls'
WSGI_APPLICATION = 'myproject.wsgi.application'
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# --- Templates ---
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
            # autoescape = True เป็น default (ไม่ต้องระบุ)
            # ห้ามตั้ง 'autoescape': False ที่นี่
        },
    },
]

# --- Password Validation ---
# ใช้ Django's built-in validators ทั้งหมด
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12},
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# --- Authentication ---
AUTH_USER_MODEL = 'blog.CustomUser'
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

### production.py — Hardened Production Settings

```python
# settings/production.py
from .base import *

# --- Security Fundamentals ---
DEBUG = False
SECRET_KEY = env('DJANGO_SECRET_KEY')
SECRET_KEY_FALLBACKS = env.list('DJANGO_SECRET_KEY_FALLBACKS', default=[])
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# --- HTTPS / TLS ---
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# --- HSTS (เริ่มจากค่าน้อย ๆ แล้วค่อยเพิ่ม) ---
SECURE_HSTS_SECONDS = 31536000          # 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True              # เพิ่มเมื่อพร้อม 100%

# --- Security Headers ---
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_REFERRER_POLICY = 'strict-origin-when-cross-origin'
SECURE_CROSS_ORIGIN_OPENER_POLICY = 'same-origin'

# --- Cookie Security ---
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_AGE = 1209600            # 2 สัปดาห์

CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = False            # ต้อง False เพื่อให้ AJAX ทำงาน
CSRF_COOKIE_SAMESITE = 'Lax'

# --- Clickjacking ---
X_FRAME_OPTIONS = 'DENY'

# --- Database ---
DATABASES = {
    'default': env.db('DATABASE_URL')
    # Django-environ parse DATABASE_URL format:
    # postgres://user:password@host:5432/dbname
}
DATABASES['default']['CONN_MAX_AGE'] = 60

# --- Cache (Redis) ---
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL', default='redis://localhost:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PASSWORD': env('REDIS_PASSWORD', default=None),
        },
    }
}

# --- Email ---
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST')
EMAIL_PORT = env.int('EMAIL_PORT', default=587)
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
EMAIL_USE_TLS = True
EMAIL_USE_SSL = False           # ใช้ TLS หรือ SSL อย่างใดอย่างหนึ่ง ไม่ใช้ทั้งคู่
DEFAULT_FROM_EMAIL = env('DEFAULT_FROM_EMAIL')
SERVER_EMAIL = env('SERVER_EMAIL', default='server@example.com')
ADMINS = [tuple(admin.split(':')) for admin in env.list('DJANGO_ADMINS', default=[])]
# ตั้งค่าใน .env: DJANGO_ADMINS=Admin Name:admin@example.com

# --- Static & Media Files ---
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATIC_URL = '/static/'
MEDIA_ROOT = BASE_DIR / 'mediafiles'
MEDIA_URL = '/media/'

# Whitenoise สำหรับ serve static files (ถ้าไม่ใช้ Nginx/CDN)
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
# เพิ่ม whitenoise ใน MIDDLEWARE หลัง SecurityMiddleware:
# 'whitenoise.middleware.WhiteNoiseMiddleware',

# --- Logging ---
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
            'include_html': False,  # False เพื่อป้องกัน HTML injection ใน error emails
        },
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'mail_admins'],
            'level': 'INFO',
        },
        'django.security': {
            'handlers': ['console', 'mail_admins'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}
```

### development.py — Settings สำหรับ Development

```python
# settings/development.py
from .base import *

DEBUG = True
SECRET_KEY = 'django-insecure-development-only-do-not-use-in-production'
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '0.0.0.0']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Development tools
INSTALLED_APPS += [
    'debug_toolbar',
]
MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware'] + MIDDLEWARE

INTERNAL_IPS = ['127.0.0.1']

EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Development: ไม่บังคับ HTTPS
SECURE_SSL_REDIRECT = False
SESSION_COOKIE_SECURE = False
CSRF_COOKIE_SECURE = False
```

### testing.py — Settings สำหรับ Test Suite

```python
# settings/testing.py
from .base import *

DEBUG = False
SECRET_KEY = 'test-secret-key-not-for-production'
ALLOWED_HOSTS = ['testserver', 'localhost']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',  # in-memory database เพื่อ speed
    }
}

# ปิด password hashing เพื่อทดสอบเร็วขึ้น
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.MD5PasswordHasher',
]

# ปิด migrations เพื่อ test เร็วขึ้น (ไม่แนะนำสำหรับ integration test)
# class DisableMigrations:
#     def __contains__(self, item): return True
#     def __getitem__(self, item): return None
# MIGRATION_MODULES = DisableMigrations()

EMAIL_BACKEND = 'django.core.mail.backends.locmem.EmailBackend'

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}

# ปิด rate limiting ใน tests
AXES_ENABLED = False
```

### .env ตัวอย่างสำหรับ Production

```bash
# .env (ไม่ commit ไฟล์นี้ใน git)
# เพิ่มใน .gitignore: .env

DJANGO_SETTINGS_MODULE=myproject.settings.production

# Security
DJANGO_SECRET_KEY=your-very-long-random-secret-key-here-at-least-50-chars
DJANGO_SECRET_KEY_FALLBACKS=
ALLOWED_HOSTS=www.example.com,example.com

# Database
DATABASE_URL=postgres://dbuser:strong-password@localhost:5432/mydb

# Redis
REDIS_URL=redis://:redis-password@localhost:6379/1
REDIS_PASSWORD=your-redis-password

# Email
EMAIL_HOST=smtp.sendgrid.net
EMAIL_PORT=587
EMAIL_HOST_USER=apikey
EMAIL_HOST_PASSWORD=your-sendgrid-api-key
DEFAULT_FROM_EMAIL=noreply@example.com
SERVER_EMAIL=server@example.com
DJANGO_ADMINS=Admin Name:admin@example.com
```

### .gitignore ที่ต้องมี

```gitignore
# Environment variables — ห้าม commit เด็ดขาด
.env
.env.*
!.env.example     # ยกเว้นไฟล์ example ที่ไม่มี secrets จริง

# Python
__pycache__/
*.py[cod]
*.pyo
.pytest_cache/
.coverage
htmlcov/
*.egg-info/

# Django
db.sqlite3
staticfiles/
mediafiles/
*.log

# Virtual environments
venv/
.venv/
env/
```

### การตรวจสอบขั้นสุดท้าย

```bash
# 1. ตรวจสอบ Django deployment checklist
DJANGO_SETTINGS_MODULE=myproject.settings.production \
    python manage.py check --deploy
# Expected: System check identified no issues (silenced: 0).

# 2. ตรวจสอบ Python package vulnerabilities
pip install pip-audit
pip-audit
# Expected: No known vulnerabilities found

# 3. ตรวจสอบว่าไม่มี secret ใน git history
git log --oneline | head -20
# ใช้ trufflehog หรือ git-secrets scan

# 4. ตรวจสอบ HTTPS configuration
# เปิด https://www.ssllabs.com/ssltest/ แล้วใส่ domain
# Target: Grade A หรือ A+
```

---

## สรุป Part 080

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|---------------|
| Security Philosophy | Defense in depth, กลไกที่เปิดโดยอัตโนมัติ |
| `check --deploy` | รหัส warning ทุกรายการและวิธีแก้ |
| SECRET_KEY | การสร้าง, ความยาว, การ rotate ด้วย fallbacks |
| DEBUG & ALLOWED_HOSTS | ทำไมถึงสำคัญใน production |
| Security Middleware | ลำดับใน MIDDLEWARE, หน้าที่แต่ละตัว |
| SECURE_* Settings | ทุก setting พร้อม default value |
| Cookie Security | SESSION และ CSRF cookie settings ทั้งหมด |
| X_FRAME_OPTIONS | DENY vs SAMEORIGIN, per-view override |
| Hardened settings.py | base/development/production/testing structure |

ใน **Part 081** เราจะดูกลไกเฉพาะของ Django สำหรับป้องกัน CSRF, XSS
และ SQL Injection ในรูปแบบ configuration reference อย่างละเอียด

---

*หลักสูตร Django ฉบับสมบูรณ์ — Part 080/100*
