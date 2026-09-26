# Part 010: Django Settings และ Environment Configuration

> **ขั้นตอนที่ 91-100 ของหลักสูตร** | Phase 1: รากฐาน Python & Django (Part สุดท้ายของ Phase นี้)
>
> เป้าหมายของ Part นี้: เจาะลึก `settings.py` ให้ครบทุกมุมที่ยังไม่ได้พูดถึงใน Part 004
> เรียนรู้วิธีแยก settings เป็นหลายไฟล์แบบที่ทีมมืออาชีพใช้จริงใน production, ใช้
> `django-environ` ในระดับขั้นสูง, เข้าใจความเสี่ยงของ `DEBUG=True`, จัดการ
> `SECRET_KEY` อย่างปลอดภัย, ตั้งค่า Logging, จัดการหลาย environment ผ่าน
> `DJANGO_SETTINGS_MODULE`, รู้จักทางเลือกอย่าง `python-decouple`, และตรวจสอบ
> ความปลอดภัยด้วย `manage.py check --deploy` เมื่อจบ Part นี้ คุณจะสามารถตั้งค่า
> Django project ให้พร้อมสำหรับทั้ง dev, staging และ production ได้อย่างมืออาชีพ
> และ Part นี้จะปิดท้ายด้วย **การสรุปภาพรวมทั้ง Phase 1** พร้อมแบบทดสอบและ
> แบบฝึกหัดใหญ่ที่รวบยอดทุกอย่างที่เรียนมาตั้งแต่ Part 001

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 91: settings.py ทั้งไฟล์แบบเจาะลึกครบ (setting สำคัญที่ยังไม่ได้พูดถึงลึก ๆ)
- ขั้นตอนที่ 92: แยก settings เป็นหลายไฟล์แบบมืออาชีพ (base/dev/staging/prod/test)
- ขั้นตอนที่ 93: django-environ ขั้นสูง (env.db(), env.cache(), env.list() และ type casting)
- ขั้นตอนที่ 94: ความเสี่ยงของ DEBUG=True ใน production และ ALLOWED_HOSTS ที่ถูกต้อง
- ขั้นตอนที่ 95: SECRET_KEY management — generate, rotate, และทำไมห้าม commit
- ขั้นตอนที่ 96: ตั้งค่า Logging เบื้องต้นใน settings.py
- ขั้นตอนที่ 97: จัดการหลาย environment ผ่าน DJANGO_SETTINGS_MODULE
- ขั้นตอนที่ 98: python-decouple เป็นทางเลือกแทน django-environ
- ขั้นตอนที่ 99: Checklist ความปลอดภัยด้วย `python manage.py check --deploy`
- ขั้นตอนที่ 100: สรุป Phase 1 ทั้งหมด + Quiz + แบบฝึกหัดใหญ่ปิดท้าย Phase

---

## ขั้นตอนที่ 91: settings.py ทั้งไฟล์แบบเจาะลึกครบ

### 91.1 ทบทวนสิ่งที่เรียนไปแล้วใน Part 004

ใน Part 004 เราเจาะลึก setting พื้นฐานที่สุดที่ `django-admin startproject` สร้างให้แล้ว:

| Setting ที่เรียนไปแล้ว | เรียนใน |
|---|---|
| `BASE_DIR`, `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` | Part 004 ขั้นตอนที่ 33.1-33.4 |
| `INSTALLED_APPS`, `MIDDLEWARE` | Part 004 ขั้นตอนที่ 33.5-33.6 |
| `ROOT_URLCONF`, `WSGI_APPLICATION`, `TEMPLATES` | Part 004 ขั้นตอนที่ 33.7-33.8 |
| `DATABASES`, `AUTH_PASSWORD_VALIDATORS` | Part 004 ขั้นตอนที่ 33.9-33.10 |
| `LANGUAGE_CODE`, `TIME_ZONE`, `USE_I18N`, `USE_TZ` | Part 004 ขั้นตอนที่ 33.11 |
| `STATIC_URL`, `DEFAULT_AUTO_FIELD` | Part 004 ขั้นตอนที่ 33.12 |
| `MEDIA_URL`, `MEDIA_ROOT`, `STATICFILES_DIRS`, `STATIC_ROOT` | Part 009 |

ขั้นตอนนี้เราจะเจาะลึก setting ที่เหลือ ซึ่งเป็น setting ที่นักพัฒนา Django มืออาชีพต้อง
เข้าใจก่อนจะนำโปรเจกต์ขึ้น production จริง แม้ว่า Django จะมี **default value** ให้
ทุกตัวอยู่แล้ว แต่การไม่เข้าใจความหมายของมันคือช่องโหว่ด้าน security และ UX ที่พบบ่อย
ที่สุดในโปรเจกต์จริง

### 91.2 `CACHES` — ระบบ Cache Framework

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "unique-snowflake",
    }
}
```

`CACHES` กำหนดว่า Django จะเก็บข้อมูลที่ cache ไว้ที่ไหน คีย์ `"default"` คือ cache
หลักที่ระบบใช้เมื่อไม่ระบุชื่อ (สามารถมีหลาย cache backend พร้อมกันได้ เช่น `"default"`
กับ `"sessions"`) ค่าเริ่มต้นของ Django (ถ้าไม่ตั้งค่าเลย) คือ
`LocMemCache` (เก็บใน memory ของ process นั้น ๆ เท่านั้น เหมาะกับ dev/test)

```python
# ตัวอย่าง backend ที่จะได้ใช้จริงเมื่อถึง Phase 8 (Redis)
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
    }
}
```

เราจะเจาะลึกการใช้ cache จริงใน Part 068-069 (Phase 8) ตอนนี้ขอให้รู้แค่ว่านี่คือ
setting ที่บอก Django ว่า "จะเก็บ cache ไว้ที่ไหน"

### 91.3 `EMAIL_*` — การตั้งค่าระบบส่งอีเมล

```python
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.gmail.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "noreply@example.com"
EMAIL_HOST_PASSWORD = "app-specific-password"
DEFAULT_FROM_EMAIL = "ทีมงาน MySite <noreply@example.com>"
SERVER_EMAIL = "server-errors@example.com"
```

| Setting | หน้าที่ |
|---|---|
| `EMAIL_BACKEND` | เลือกวิธีส่งอีเมล (SMTP จริง, console, file, ในหน่วยความจำสำหรับ test) |
| `EMAIL_HOST` / `EMAIL_PORT` | ที่อยู่และพอร์ตของ SMTP server |
| `EMAIL_USE_TLS` / `EMAIL_USE_SSL` | เข้ารหัสการเชื่อมต่อ (เลือกอย่างใดอย่างหนึ่ง ไม่ใช้พร้อมกัน) |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | บัญชีสำหรับ login เข้า SMTP server |
| `DEFAULT_FROM_EMAIL` | ชื่อผู้ส่งเริ่มต้นเมื่อไม่ได้ระบุ `from_email` |
| `SERVER_EMAIL` | ผู้ส่งของอีเมล error ที่ Django ส่งให้ `ADMINS` อัตโนมัติ |

สำหรับการพัฒนา ไม่ควรส่งอีเมลจริงออกไปทุกครั้งที่ทดสอบ Django จึงมี **console backend**
ที่พิมพ์เนื้อหาอีเมลออกมาที่ terminal แทนการส่งจริง:

```python
# ใช้เฉพาะตอนพัฒนา (dev.py)
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

และมี **file backend** ที่เขียนอีเมลลงไฟล์แทน เหมาะกับการ debug แบบเก็บหลักฐาน:

```python
EMAIL_BACKEND = "django.core.mail.backends.filebased.EmailBackend"
EMAIL_FILE_PATH = BASE_DIR / "sent_emails"
```

เราจะเจาะลึกการส่งอีเมลจริง (password reset, verification) ใน Phase 4

### 91.4 `SESSION_*` — พฤติกรรมของ Session และ Cookie

```python
SESSION_ENGINE = "django.contrib.sessions.backends.db"
SESSION_COOKIE_NAME = "sessionid"
SESSION_COOKIE_AGE = 1209600  # 2 สัปดาห์ (หน่วยเป็นวินาที)
SESSION_COOKIE_SECURE = False   # ต้องเป็น True ใน production ที่ใช้ HTTPS
SESSION_COOKIE_HTTPONLY = True
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
SESSION_SAVE_EVERY_REQUEST = False
```

| Setting | ความหมาย |
|---|---|
| `SESSION_ENGINE` | เก็บ session ไว้ที่ไหน (`db` = ตารางฐานข้อมูล, `cache`, `cached_db`, `file`, `signed_cookies`) |
| `SESSION_COOKIE_AGE` | อายุของ session cookie (วินาที) ก่อนหมดอายุ |
| `SESSION_COOKIE_SECURE` | ถ้า `True` cookie จะถูกส่งผ่าน HTTPS เท่านั้น (**ต้องเปิดใน production เสมอ**) |
| `SESSION_COOKIE_HTTPONLY` | ป้องกัน JavaScript อ่าน cookie นี้ได้ (ป้องกัน XSS ขโมย session) |
| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | ถ้า `True` session จะหมดอายุทันทีที่ปิดเบราว์เซอร์ |

เราจะเจาะลึกเรื่อง Session แบบเต็มรูปแบบใน Part 034 (Phase 4)

### 91.5 `CSRF_*` — การป้องกัน Cross-Site Request Forgery

```python
CSRF_COOKIE_NAME = "csrftoken"
CSRF_COOKIE_SECURE = False   # True ใน production ที่ใช้ HTTPS
CSRF_COOKIE_HTTPONLY = False  # ปกติต้องเป็น False เพราะ JS ต้องอ่านไปใส่ header เอง
CSRF_TRUSTED_ORIGINS = []
CSRF_USE_SESSIONS = False
```

`CsrfViewMiddleware` (ที่เราเห็นใน `MIDDLEWARE` ตั้งแต่ Part 004) ใช้ setting กลุ่มนี้
ควบคุมพฤติกรรม ที่สำคัญที่สุดสำหรับมือใหม่คือ `CSRF_TRUSTED_ORIGINS` ซึ่งตั้งแต่ Django 4.0
เป็นต้นมา **ต้องระบุ scheme (`https://`) เสมอ** ไม่ใช่แค่ domain เฉย ๆ:

```python
CSRF_TRUSTED_ORIGINS = [
    "https://example.com",
    "https://*.example.com",  # รองรับ wildcard subdomain ได้ตั้งแต่ Django 4.0
]
```

setting นี้จำเป็นเมื่อ frontend กับ backend อยู่คนละ domain กัน หรือเมื่อมี reverse
proxy (เช่น Nginx) อยู่หน้า Django และ POST request มาจาก origin ที่ Django ไม่รู้จัก
จะเจอ error `403 Forbidden (CSRF verification failed)` ถ้าไม่ได้ตั้งค่านี้ให้ถูกต้อง

### 91.6 `SECURE_*` — HTTP Security Headers

กลุ่ม setting นี้ควบคุม `SecurityMiddleware` (ตัวแรกใน `MIDDLEWARE`) ซึ่งจัดการ
HTTP response header ด้าน security โดยตรง:

```python
SECURE_SSL_REDIRECT = False           # True ใน production: บังคับ redirect HTTP → HTTPS
SECURE_HSTS_SECONDS = 0                # production ควรตั้งเป็น 31536000 (1 ปี)
SECURE_HSTS_INCLUDE_SUBDOMAINS = False
SECURE_HSTS_PRELOAD = False
SECURE_CONTENT_TYPE_NOSNIFF = True    # ป้องกันเบราว์เซอร์เดา content-type เอง
SECURE_REFERRER_POLICY = "same-origin"
SECURE_PROXY_SSL_HEADER = None        # ตั้งเมื่ออยู่หลัง reverse proxy เช่น Nginx/ELB
```

| Setting | ป้องกันอะไร |
|---|---|
| `SECURE_SSL_REDIRECT` | บังคับทุก request ให้วิ่งผ่าน HTTPS เท่านั้น |
| `SECURE_HSTS_SECONDS` | บอกเบราว์เซอร์ให้จำไว้ว่าต้องใช้ HTTPS เท่านั้นในอนาคต (HSTS header) |
| `SECURE_CONTENT_TYPE_NOSNIFF` | ป้องกันการโจมตีที่อาศัยเบราว์เซอร์เดา MIME type ผิด |
| `SECURE_PROXY_SSL_HEADER` | บอก Django ว่า request ที่มาจาก proxy ตัวไหนถือว่าเป็น HTTPS จริง |

`SECURE_PROXY_SSL_HEADER` สำคัญมากเมื่อ deploy หลัง Nginx หรือ Load Balancer เพราะ
connection จริงระหว่าง proxy กับ Django มักเป็น HTTP ธรรมดา (เข้ารหัสแค่ช่วง
proxy-เบราว์เซอร์) ถ้าไม่ตั้งค่านี้ Django จะเข้าใจผิดว่าทุก request เป็น HTTP เสมอ:

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

เราจะกลับมาตั้งค่ากลุ่มนี้อย่างเต็มรูปแบบใน Phase 10 (Security) และในขั้นตอนที่ 99
ของ Part นี้ เมื่อพูดถึง `check --deploy`

### 91.7 การตั้งค่าไฟล์อัปโหลด

```python
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440       # 2.5 MB — เกินนี้จะเขียนลงดิสก์แทน RAM
DATA_UPLOAD_MAX_MEMORY_SIZE = 2621440
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1000        # จำกัดจำนวน field สูงสุดใน POST/GET
FILE_UPLOAD_PERMISSIONS = 0o644
FILE_UPLOAD_TEMP_DIR = None                  # None = ใช้ temp directory ของระบบปฏิบัติการ
```

`DATA_UPLOAD_MAX_NUMBER_FIELDS` เป็น setting ด้าน security ที่มือใหม่มักไม่รู้จัก
มันป้องกันการโจมตีแบบส่งฟอร์มที่มี field เป็นหมื่นเป็นแสนตัวเพื่อทำให้ server ค้าง
(a denial-of-service ผ่านการ parse form) เราเจาะลึกเรื่องไฟล์อัปโหลดแบบเต็มใน Part 009
และ Part 058 (Phase 6)

### 91.8 Setting ที่เกี่ยวกับ Authentication redirect

```python
LOGIN_URL = "/accounts/login/"
LOGIN_REDIRECT_URL = "/"
LOGOUT_REDIRECT_URL = None
```

- `LOGIN_URL` — URL ที่ decorator `@login_required` จะ redirect ผู้ใช้ไปเมื่อยังไม่ล็อกอิน
- `LOGIN_REDIRECT_URL` — URL ปลายทางหลังล็อกอินสำเร็จ (ถ้าไม่มี `?next=` ระบุมา)
- `LOGOUT_REDIRECT_URL` — URL ปลายทางหลัง logout

เราจะใช้ setting กลุ่มนี้จริงจังใน Part 031 (Phase 4) ตอนสร้างระบบ Authentication

### 91.9 `ADMINS`, `MANAGERS` — แจ้งเตือนแอดมินเมื่อเกิด error

```python
ADMINS = [
    ("Somchai Dev", "somchai@example.com"),
]
MANAGERS = ADMINS
```

เมื่อ `DEBUG = False` และเกิด **500 Internal Server Error** ในระบบ Django จะส่งอีเมล
แจ้งเตือนไปยังทุกคนใน `ADMINS` โดยอัตโนมัติ (ผ่าน `AdminEmailHandler` ที่เราจะเห็นอีกครั้ง
ในขั้นตอนที่ 96 เรื่อง Logging) พร้อม traceback แบบเต็ม — เป็นกลไกที่ช่วยให้ทีมรู้ตัวว่า
มี error เกิดขึ้นจริงบน production โดยไม่ต้องรอผู้ใช้แจ้งเข้ามาเอง

### 91.10 เบ็ดเตล็ดที่ควรรู้

```python
INTERNAL_IPS = ["127.0.0.1"]     # IP ที่ถือว่าเป็น "internal" (ใช้กับ Debug Toolbar)
APPEND_SLASH = True               # เติม / ท้าย URL อัตโนมัติถ้าไม่ตรง pattern แล้วลองใหม่
X_FRAME_OPTIONS = "DENY"          # ป้องกัน Clickjacking โดยห้าม embed เว็บใน <iframe> เลย
SILENCED_SYSTEM_CHECKS = []       # รายชื่อ warning code ที่ยอมรับความเสี่ยงแล้ว ไม่อยากเห็นซ้ำ
```

- `APPEND_SLASH` ทำงานร่วมกับ `CommonMiddleware`: ถ้าผู้ใช้เข้า `/products` (ไม่มี `/`
  ท้าย) และไม่มี URL pattern ตรงกันพอดี แต่ `/products/` มี Django จะ redirect ให้อัตโนมัติ
- `X_FRAME_OPTIONS` ค่าเริ่มต้นคือ `"DENY"` ปลอดภัยที่สุด ถ้าต้องการให้หน้าเว็บถูก embed
  ใน iframe ได้ (เช่น widget) ต้องเปลี่ยนเป็น `"SAMEORIGIN"` หรือปรับเฉพาะ view นั้น
- `SILENCED_SYSTEM_CHECKS` เราจะกลับมาใช้ในขั้นตอนที่ 99

### 91.11 ตารางสรุปรวม setting ทั้งหมดที่เจาะลึกใน Part นี้

| กลุ่ม Setting | ตัวอย่างชื่อ | เจาะลึกใน |
|---|---|---|
| Cache | `CACHES` | 91.2, Phase 8 |
| Email | `EMAIL_BACKEND`, `DEFAULT_FROM_EMAIL` | 91.3, Phase 4 |
| Session | `SESSION_COOKIE_AGE`, `SESSION_ENGINE` | 91.4, Part 034 |
| CSRF | `CSRF_TRUSTED_ORIGINS` | 91.5, Phase 10 |
| Security Headers | `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS` | 91.6, 99, Phase 10 |
| File Upload | `DATA_UPLOAD_MAX_NUMBER_FIELDS` | 91.7, Part 058 |
| Auth Redirect | `LOGIN_URL`, `LOGIN_REDIRECT_URL` | 91.8, Part 031 |
| Error Notification | `ADMINS`, `SERVER_EMAIL` | 91.9, 96 |
| เบ็ดเตล็ด | `X_FRAME_OPTIONS`, `APPEND_SLASH` | 91.10 |

ตอนนี้คุณเห็นภาพรวมของ `settings.py` ครบทุกมุมแล้ว ขั้นตอนถัดไปคือการจัดการไฟล์เดียวที่
กำลังจะยาวขึ้นเรื่อย ๆ นี้ ให้เป็นระบบระดับมืออาชีพ

---

## ขั้นตอนที่ 92: แยก settings เป็นหลายไฟล์แบบมืออาชีพ

### 92.1 ปัญหาของ `settings.py` ไฟล์เดียว

เมื่อโปรเจกต์เติบโตขึ้น ไฟล์ `settings.py` ไฟล์เดียวจะเริ่มมีปัญหาเหล่านี้:

1. **โค้ด if-else กระจัดกระจาย**: นักพัฒนามือใหม่มักแก้ปัญหาแบบ
   `if env("DJANGO_ENV") == "production": DEBUG = False` ปนอยู่ทั่วไฟล์ ทำให้อ่านยาก
   และเสี่ยงต่อการลืมเงื่อนไขบางจุด
2. **ไฟล์ยาวเกินไป**: settings จริงของโปรเจกต์ขนาดกลางมักมีหลายร้อยบรรทัด รวมทุก
   environment ไว้ในไฟล์เดียวทำให้ merge conflict บ่อยเมื่อทำงานเป็นทีม
3. **เสี่ยงเผลอใช้ค่าผิด environment**: การสลับ environment ด้วยตัวแปรใน if-else
   ทำให้ตรวจสอบยากว่า production จริง ๆ ใช้ค่าไหนอยู่ ต่างจากการมีไฟล์แยกที่เห็นชัดเจน
4. **Test settings ปนกับของจริง**: การรัน automated test ต้องการค่าพิเศษ (เช่น password
   hasher ที่เร็วขึ้น) ซึ่งไม่ควรปนกับ settings ของ dev/prod

**แนวทางมืออาชีพ**: แยก `settings.py` ออกเป็น **package** (โฟลเดอร์ที่มี `__init__.py`)
แทนที่จะเป็นไฟล์เดียว โดยมีไฟล์ฐาน (`base.py`) ที่เก็บค่าที่ใช้ร่วมกันทุก environment
แล้วให้แต่ละ environment มีไฟล์ของตัวเองที่ import ค่าจาก base มา override เฉพาะจุด
ที่ต่างกัน

### 92.2 โครงสร้างเป้าหมาย

```
django-mastery-course/
├── config/
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── dev.py
│   │   ├── staging.py
│   │   ├── prod.py
│   │   └── test.py
│   ├── __init__.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── .env.dev
├── .env.staging
├── .env.prod
├── manage.py
└── requirements.txt
```

สังเกตว่า `config/settings.py` (ไฟล์เดี่ยว) หายไป กลายเป็นโฟลเดอร์ `config/settings/`
(package) แทน — Python สามารถ `import config.settings.dev` ได้เหมือน module ปกติ
เพราะมี `__init__.py` อยู่ในโฟลเดอร์นั้น

### 92.3 ย้ายไฟล์และสร้าง `base.py`

```bash
mkdir config/settings
git mv config/settings.py config/settings/base.py
touch config/settings/__init__.py
touch config/settings/dev.py config/settings/staging.py
touch config/settings/prod.py config/settings/test.py
```

`base.py` เก็บ **ทุกอย่างที่เหมือนกันในทุก environment**: `BASE_DIR`, `INSTALLED_APPS`,
`MIDDLEWARE`, `TEMPLATES`, `AUTH_PASSWORD_VALIDATORS`, `LANGUAGE_CODE`, `TIME_ZONE`,
`STATIC_URL`, `DEFAULT_AUTO_FIELD` เป็นต้น สังเกตว่า `BASE_DIR` ต้องขยับขึ้นอีก 1
ระดับ เพราะไฟล์นี้ลึกลงไปอีกชั้นหนึ่ง (`config/settings/base.py` แทนที่จะเป็น
`config/settings.py`):

```python
# config/settings/base.py
"""
Base settings — ค่าที่ใช้ร่วมกันทุก environment
ห้ามใส่ค่าที่ sensitive หรือค่าที่ต่างกันระหว่าง environment ไว้ที่นี่
"""
from pathlib import Path

import environ

# เดินขึ้น 3 ระดับ: base.py -> settings/ -> config/ -> รากโปรเจกต์
BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env()

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

WSGI_APPLICATION = "config.wsgi.application"

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]

LANGUAGE_CODE = "th"
TIME_ZONE = "Asia/Bangkok"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
STATICFILES_DIRS = [BASE_DIR / "static"]
STATIC_ROOT = BASE_DIR / "staticfiles"

MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

### 92.4 `dev.py` — environment สำหรับพัฒนาในเครื่อง

```python
# config/settings/dev.py
from .base import *  # noqa: F401,F403
from .base import BASE_DIR, env

# อ่านค่าจาก .env.dev เฉพาะไฟล์นี้เท่านั้น
environ.Env.read_env(BASE_DIR / ".env.dev")

DEBUG = True
SECRET_KEY = env("SECRET_KEY")
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

# แสดง error page แบบละเอียดของ Django เองเสมอตอน dev (ไม่ redirect ปิดบัง error)
CORS_ALLOW_ALL_ORIGINS = True if "corsheaders" in INSTALLED_APPS else False
```

ต้อง `import environ` ด้วยในไฟล์นี้เพื่อเรียก `environ.Env.read_env(...)` — ใส่ไว้บนสุด
ของไฟล์ (ตัดออกจากตัวอย่างข้างต้นเพื่อความกระชับ แต่ในไฟล์จริงต้องมี
`import environ` ก่อนใช้งาน)

### 92.5 `staging.py` — environment จำลอง production เพื่อทดสอบก่อนขึ้นจริง

```python
# config/settings/staging.py
from .base import *  # noqa: F401,F403
from .base import BASE_DIR, env

environ.Env.read_env(BASE_DIR / ".env.staging")

DEBUG = False
SECRET_KEY = env("SECRET_KEY")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")

DATABASES = {"default": env.db("DATABASE_URL")}

EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = env("EMAIL_HOST")
EMAIL_PORT = env.int("EMAIL_PORT", default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = env("EMAIL_HOST_PASSWORD")

# staging ควรเปิด security header เหมือน production เกือบทั้งหมด เพื่อทดสอบจริง
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

**Staging** คือ environment ที่จำลอง production ให้เหมือนที่สุด (ใช้ PostgreSQL จริง,
HTTPS จริง, ตั้งค่า security เหมือนจริง) แต่แยกฐานข้อมูลและโดเมนออกจาก production
เพื่อให้ทีมทดสอบ feature ใหม่ได้อย่างปลอดภัยก่อนปล่อยให้ผู้ใช้จริงเห็น

### 92.6 `prod.py` — environment สำหรับ production จริง

```python
# config/settings/prod.py
from .base import *  # noqa: F401,F403
from .base import BASE_DIR, env

environ.Env.read_env(BASE_DIR / ".env.prod")

DEBUG = False
SECRET_KEY = env("SECRET_KEY")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")

DATABASES = {"default": env.db("DATABASE_URL")}

CACHES = {"default": env.cache("CACHE_URL")}

EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = env("EMAIL_HOST")
EMAIL_PORT = env.int("EMAIL_PORT", default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = env("EMAIL_HOST_PASSWORD")
DEFAULT_FROM_EMAIL = env("DEFAULT_FROM_EMAIL")

ADMINS = [("Dev Team", env("ADMIN_EMAIL"))]
MANAGERS = ADMINS

# --- Security hardening เต็มรูปแบบ ---
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
CSRF_TRUSTED_ORIGINS = env.list("CSRF_TRUSTED_ORIGINS")
```

### 92.7 `test.py` — environment สำหรับรัน automated test

```python
# config/settings/test.py
from .base import *  # noqa: F401,F403
from .base import BASE_DIR

DEBUG = False
SECRET_KEY = "test-secret-key-not-used-anywhere-real"
ALLOWED_HOSTS = ["testserver"]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",   # ฐานข้อมูลอยู่ใน RAM ทั้งหมด รันเทสต์เร็วขึ้นมาก
    }
}

# ทำให้ hash รหัสผ่านเร็วขึ้นมาก (ปกติ PBKDF2 ตั้งใจให้ช้าเพื่อความปลอดภัย
# แต่ตอนรัน test หลายพันเคส ความช้านี้จะทำให้ test suite ใช้เวลานานเกินจำเป็น)
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]

EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
```

**ข้อควรระวังสำคัญ**: `test.py` ใช้ `SECRET_KEY` แบบ hardcode ได้ (ไม่ต้องอ่านจาก `.env`)
เพราะไม่มีข้อมูลจริงใด ๆ เกี่ยวข้อง และ `MD5PasswordHasher` ใช้ได้เฉพาะตอนรัน test
เท่านั้น **ห้ามใช้ใน dev/staging/prod เด็ดขาด** เพราะ MD5 ไม่ปลอดภัยพอสำหรับเก็บรหัสผ่าน
จริง เราจะเจาะลึกเรื่อง test settings อีกครั้งใน Part 059-061 (Phase 7)

### 92.8 `__init__.py` — pattern การเลือก settings อัตโนมัติ

มีสองแนวทางหลักที่ทีมมืออาชีพใช้เลือกว่าจะโหลด settings ไฟล์ไหน:

**แนวทาง A: ให้ `__init__.py` เลือกเองจากตัวแปร `DJANGO_ENV`**

```python
# config/settings/__init__.py
import os

_env = os.environ.get("DJANGO_ENV", "dev")

if _env == "prod":
    from .prod import *  # noqa: F401,F403
elif _env == "staging":
    from .staging import *  # noqa: F401,F403
elif _env == "test":
    from .test import *  # noqa: F401,F403
else:
    from .dev import *  # noqa: F401,F403
```

ข้อดีของแนวทางนี้คือ `manage.py`/`wsgi.py`/`asgi.py` ไม่ต้องรู้เรื่อง environment เลย —
ยังชี้ไปที่ `config.settings` เหมือนเดิม (module เดียวไม่เปลี่ยน) แค่ตั้งตัวแปร
`DJANGO_ENV` ให้ถูกต้องก่อนรัน ข้อเสียคือมี "เลเยอร์ตัวแปรเพิ่ม" (`DJANGO_ENV`)
ที่ไม่ใช่ค่ามาตรฐานของ Django เอง (Django เองรู้จักแค่ `DJANGO_SETTINGS_MODULE`)

**แนวทาง B: ให้ `DJANGO_SETTINGS_MODULE` ชี้ตรงไปที่ไฟล์นั้นเลย** (มาตรฐานของ Django เอง)

```python
# config/settings/__init__.py
# (ปล่อยว่างไว้ แค่ทำให้ settings/ เป็น Python package)
```

```bash
# แทนที่จะพึ่ง DJANGO_ENV เราชี้ DJANGO_SETTINGS_MODULE ตรง ๆ ไปที่ไฟล์ที่ต้องการ
export DJANGO_SETTINGS_MODULE=config.settings.prod
python manage.py runserver
```

หลักสูตรนี้แนะนำ **แนวทาง B** เป็นค่าเริ่มต้น เพราะเป็นมาตรฐานที่ตัว Django framework
เองออกแบบมาให้ใช้งานโดยตรง (ไม่ต้องพึ่งตัวแปรที่ทีมประดิษฐ์เอง) และจะอธิบายอย่างเต็มรูปแบบ
ในขั้นตอนที่ 97 — ส่วนแนวทาง A ก็ยังเป็นทางเลือกที่ใช้ได้ดีในบางทีมที่ต้องการความสะดวก
ไม่ต้องเปลี่ยนค่า `DJANGO_SETTINGS_MODULE` เลย

### 92.9 อัปเดต `manage.py`, `wsgi.py`, `asgi.py`

เมื่อ settings กลายเป็น package ต้องอัปเดตทั้ง 3 ไฟล์ entry point ให้ชี้ไปยัง module
ย่อยที่ถูกต้อง (ตัวอย่างนี้ตั้งค่า default เป็น `dev` เพื่อความสะดวกตอนรันในเครื่อง):

```python
# manage.py
def main():
    """Run administrative tasks."""
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
    ...
```

```python
# config/wsgi.py
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.prod")
application = get_wsgi_application()
```

```python
# config/asgi.py
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.prod")
application = get_asgi_application()
```

สังเกตว่า `manage.py` ตั้ง default เป็น `dev` (เพราะนักพัฒนามักรันคำสั่งนี้ในเครื่องของ
ตัวเอง) ในขณะที่ `wsgi.py`/`asgi.py` ตั้ง default เป็น `prod` (เพราะเป็นจุดเข้าที่ web
server จริงใช้ตอน deploy) — `os.environ.setdefault()` จะใช้ค่าเหล่านี้ **ก็ต่อเมื่อ**
ยังไม่มีตัวแปร `DJANGO_SETTINGS_MODULE` ถูกตั้งไว้ในระบบมาก่อน ถ้ามีการ `export`
ตัวแปรนี้ไว้แล้ว (ตามที่จะสอนในขั้นตอนที่ 97) ค่านั้นจะถูกใช้แทนเสมอ

### 92.10 ทดสอบว่าระบบยังทำงานถูกต้อง

```bash
# รันด้วย dev settings (ค่า default จาก manage.py)
python manage.py check

# บังคับใช้ settings อื่นแบบชัดเจนด้วย --settings flag
python manage.py check --settings=config.settings.prod
python manage.py runserver --settings=config.settings.staging
```

`--settings` เป็น flag ที่ใช้ได้กับทุกคำสั่งของ `manage.py` และมีความสำคัญกว่า
ทั้งค่า default ใน `manage.py` และตัวแปร environment `DJANGO_SETTINGS_MODULE`
(ลำดับความสำคัญ: `--settings` flag > environment variable > default ใน `manage.py`)

### 92.11 ข้อดี/ข้อเสียเทียบกับไฟล์เดียว

| ประเด็น | ไฟล์เดียว (`settings.py`) | แยกไฟล์ (`settings/` package) |
|---|---|---|
| ความชัดเจนของแต่ละ environment | ต่ำ (if-else ปนกัน) | สูงมาก (แยกไฟล์ชัดเจน) |
| ความเสี่ยง merge conflict ในทีม | สูง | ต่ำกว่า (แก้คนละไฟล์กันได้) |
| เหมาะกับโปรเจกต์เล็ก/เรียนรู้ | ✅ เหมาะมาก | อาจซับซ้อนเกินความจำเป็น |
| เหมาะกับโปรเจกต์ทีม/production | ❌ ไม่แนะนำ | ✅ มาตรฐานอุตสาหกรรม |
| ใช้ใน cookiecutter-django (template ยอดนิยม) | ❌ | ✅ ใช้รูปแบบนี้เป็นค่าเริ่มต้น |

ตั้งแต่ Part นี้เป็นต้นไป หลักสูตรจะอ้างอิงโครงสร้าง `config/settings/` แบบนี้เมื่อพูดถึง
การ deploy หรือแยก environment แม้ว่าใน Part ต้น ๆ ก่อนหน้านี้จะใช้ `settings.py`
ไฟล์เดียวเพื่อความง่ายในการเรียนรู้ก็ตาม คุณสามารถเลือกแปลงโปรเจกต์ของตัวเองตอนนี้
หรือรอจนกว่าจะถึง Phase 11 (Deployment) ก็ได้ตามความสะดวก

---

## ขั้นตอนที่ 93: django-environ ขั้นสูง

### 93.1 ทบทวนพื้นฐานจาก Part 004

ใน Part 004 เราใช้ `django-environ` แค่ระดับพื้นฐาน: `env("SECRET_KEY")`, `env("DEBUG")`,
และ `env.list("ALLOWED_HOSTS")` ขั้นตอนนี้เราจะเจาะลึกความสามารถระดับขั้นสูงที่ทำให้
`django-environ` เป็น library ยอดนิยมอันดับหนึ่งของวงการ Django

### 93.2 `env.db()` — Parse Database URL เป็น dict ให้อัตโนมัติ

แทนที่จะเขียน `DATABASES` เป็น dict ยาว ๆ ทีละ key เราสามารถเก็บ connection string
เดียวไว้ใน `.env` แล้วให้ `django-environ` แปลงเป็น dict ที่ Django ต้องการให้อัตโนมัติ:

```
# .env.prod
DATABASE_URL=postgres://myuser:mypassword@db.example.com:5432/mydb
```

```python
DATABASES = {
    "default": env.db("DATABASE_URL"),
}
```

ผลลัพธ์ของ `env.db("DATABASE_URL")` จะเทียบเท่ากับการเขียนเองแบบนี้:

```python
{
    "ENGINE": "django.db.backends.postgresql",
    "NAME": "mydb",
    "USER": "myuser",
    "PASSWORD": "mypassword",
    "HOST": "db.example.com",
    "PORT": 5432,
}
```

`env.db()` รองรับ URL scheme หลายแบบ:

| Scheme ใน URL | แปลงเป็น `ENGINE` |
|---|---|
| `postgres://` หรือ `postgresql://` | `django.db.backends.postgresql` |
| `mysql://` | `django.db.backends.mysql` |
| `sqlite:///` | `django.db.backends.sqlite3` |
| `postgis://` | `django.contrib.gis.db.backends.postgis` |

นี่คือรูปแบบที่ **แพลตฟอร์ม Cloud ยอดนิยม** (Heroku, Railway, Render) ใช้ส่งค่า
`DATABASE_URL` มาให้แอปของคุณโดยอัตโนมัติ ทำให้ `env.db()` มีประโยชน์อย่างยิ่งตอน deploy

### 93.3 `env.cache()` — Parse Cache URL เช่นเดียวกัน

```
# .env.prod
CACHE_URL=redis://127.0.0.1:6379/1
```

```python
CACHES = {
    "default": env.cache("CACHE_URL"),
}
```

เทียบเท่ากับ:

```python
{
    "BACKEND": "django.core.cache.backends.redis.RedisCache",
    "LOCATION": "redis://127.0.0.1:6379/1",
}
```

### 93.4 `env.list()`, `env.tuple()`, `env.dict()`

```
# .env
ALLOWED_HOSTS=example.com,www.example.com,api.example.com
CORS_METHODS=GET,POST,PUT
FEATURE_FLAGS=new_ui:true,beta_search:false
```

```python
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")
# → ['example.com', 'www.example.com', 'api.example.com']

CORS_METHODS = env.tuple("CORS_METHODS")
# → ('GET', 'POST', 'PUT')  — tuple แทน list (immutable)

FEATURE_FLAGS = env.dict("FEATURE_FLAGS", cast=bool)
# → {'new_ui': True, 'beta_search': False}
```

`env.list()` และ `env.tuple()` รับ parameter `cast` เพื่อแปลงชนิดของสมาชิกแต่ละตัวได้
เช่น `env.list("PORTS", cast=int)` จะได้ `list` ของ `int` แทน `str`

### 93.5 Type Casting ที่ครบถ้วน

| เมธอด | แปลงจาก string เป็น | ตัวอย่าง |
|---|---|---|
| `env.str("KEY")` | `str` (เหมือน `env("KEY")`) | `"hello"` → `"hello"` |
| `env.bool("KEY")` | `bool` | `"True"`/`"1"`/`"yes"` → `True` |
| `env.int("KEY")` | `int` | `"8000"` → `8000` |
| `env.float("KEY")` | `float` | `"3.14"` → `3.14` |
| `env.json("KEY")` | `dict`/`list` (parse JSON) | `'{"a": 1}'` → `{"a": 1}` |
| `env.path("KEY")` | `pathlib.Path` | `"/var/www"` → `Path("/var/www")` |
| `env.url("KEY")` | `urllib.parse.ParseResult` | เช่นใช้ parse webhook URL ภายนอก |
| `env.email("KEY")` (validate) | `str` ที่ตรวจสอบรูปแบบอีเมล | `"a@b.com"` → `"a@b.com"` |

ตัวอย่างการใช้ `env.json()` สำหรับค่าที่ซับซ้อนเป็นโครงสร้าง:

```
# .env
THIRD_PARTY_CONFIG={"timeout": 30, "retries": 3, "endpoints": ["a.com", "b.com"]}
```

```python
THIRD_PARTY_CONFIG = env.json("THIRD_PARTY_CONFIG")
# → {'timeout': 30, 'retries': 3, 'endpoints': ['a.com', 'b.com']}
```

### 93.6 จัดการหลาย `.env` file ต่อ environment

ตามโครงสร้างที่วางไว้ในขั้นตอนที่ 92 เรามีไฟล์ `.env.dev`, `.env.staging`, `.env.prod`
แยกกัน — สังเกตว่าแต่ละไฟล์ settings (`dev.py`, `staging.py`, `prod.py`) เรียก
`environ.Env.read_env(BASE_DIR / ".env.<environment>")` ของตัวเอง ทำให้ **แต่ละ
environment อ่านไฟล์ค่าคนละไฟล์กันโดยอัตโนมัติ** ไม่มีทางอ่านผิดไฟล์ เพราะไฟล์
settings ระบุชื่อไฟล์ `.env` ของตัวเองตรง ๆ:

```python
# config/settings/dev.py
environ.Env.read_env(BASE_DIR / ".env.dev")

# config/settings/staging.py
environ.Env.read_env(BASE_DIR / ".env.staging")

# config/settings/prod.py
environ.Env.read_env(BASE_DIR / ".env.prod")
```

อย่าลืมเพิ่มไฟล์เหล่านี้ทั้งหมดใน `.gitignore` (ยกเว้นไฟล์ `.env.example` ที่ไม่มีค่า
sensitive จริง):

```
# .gitignore
.env
.env.dev
.env.staging
.env.prod
!.env.example
```

**ข้อควรรู้สำหรับ production จริง**: ในทางปฏิบัติ ทีมส่วนใหญ่ **ไม่ได้เก็บไฟล์
`.env.prod` ไว้บนเครื่อง server เลย** แต่ให้ระบบ orchestration (Docker, Kubernetes,
CI/CD pipeline) เป็นผู้ set environment variable ให้โดยตรงตอน deploy (เราจะเรียน
เรื่องนี้เต็มรูปแบบใน Phase 11) — ไฟล์ `.env.*` มีประโยชน์มากที่สุดตอนพัฒนาในเครื่อง
และตอนทดสอบ staging เป็นหลัก

### 93.7 ตัวอย่างการรวมทุกเทคนิคไว้ในไฟล์เดียว

```python
# config/settings/prod.py (ตัวอย่างสมบูรณ์ที่รวมเทคนิคจากขั้นตอนนี้)
import environ

from .base import *  # noqa: F401,F403
from .base import BASE_DIR

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)
environ.Env.read_env(BASE_DIR / ".env.prod")

DEBUG = env("DEBUG")
SECRET_KEY = env("SECRET_KEY")
ALLOWED_HOSTS = env("ALLOWED_HOSTS")

DATABASES = {"default": env.db("DATABASE_URL")}
CACHES = {"default": env.cache("CACHE_URL")}

FEATURE_FLAGS = env.json("FEATURE_FLAGS", default={})
```

สังเกตว่า `environ.Env(...)` รับ dict ของ default schema ได้ตั้งแต่ตอนสร้าง object
(`DEBUG=(bool, False)` หมายถึง "ถ้าเรียก `env("DEBUG")` ให้แปลงเป็น `bool` และถ้าไม่พบ
ค่าใน `.env` เลยให้ใช้ `False` เป็นค่า default") วิธีนี้ทำให้ไม่ต้องเขียน `env.bool(...)`
ซ้ำทุกครั้งที่เรียกตัวแปรเดิม

---

## ขั้นตอนที่ 94: ความเสี่ยงของ DEBUG=True ใน production และ ALLOWED_HOSTS ที่ถูกต้อง

### 94.1 สิ่งที่หน้า Debug Page ของ Django เผยออกมา

เมื่อ `DEBUG = True` และเกิด exception ที่ไม่ถูกจัดการ (unhandled exception) Django
จะแสดงหน้า error สีเหลืองที่มีข้อมูลรายละเอียดสูงมาก ซึ่งมีประโยชน์มากตอนพัฒนา แต่
**เป็นหายนะด้าน security ถ้าเกิดขึ้นบน production ที่เปิดให้สาธารณะเข้าถึงได้**
หน้านี้เผยข้อมูลต่อไปนี้ให้ใครก็ตามที่ทำให้เกิด error ได้:

| ข้อมูลที่รั่วไหล | ผลกระทบ |
|---|---|
| **Full Python traceback** พร้อมชื่อไฟล์และเลขบรรทัดจริงบนเซิร์ฟเวอร์ | เผยโครงสร้างโค้ดภายในทั้งหมด ช่วยผู้โจมตีวางแผนต่อ |
| **ค่าตัวแปรทุกตัวในทุก stack frame** (local variables) | อาจรวมถึงรหัสผ่านที่ถูกส่งเข้ามาใน form, token ชั่วคราว |
| **รายการ setting ทั้งหมดใน `settings.py`** (ยกเว้นบางตัวที่ Django กรองให้อัตโนมัติ เช่น `SECRET_KEY`, `PASSWORD`) | เผย path ของระบบไฟล์, ชื่อ database, hostname ภายใน |
| **SQL query ล่าสุดที่รันไป** (ถ้าใช้ ORM) | เผยโครงสร้างตารางฐานข้อมูลและชื่อคอลัมน์ |
| **เวอร์ชันของ Django และ Python ที่ใช้** | ช่วยผู้โจมตีค้นหาช่องโหว่ (CVE) ที่ตรงกับเวอร์ชันนั้น ๆ |
| **HTTP headers ทั้งหมดของ request** | อาจเผย cookie หรือ token ที่ยังไม่ถูก mask |

แม้ Django จะพยายาม **กรอง** ค่าที่ดูเหมือนจะ sensitive อัตโนมัติ (เช่นตัวแปรที่ชื่อมีคำว่า
`password`, `secret`, `token`, `api_key` จะถูกแทนที่ด้วย `********`) แต่การกรองนี้
อาศัยการ**เดาจากชื่อตัวแปร**เท่านั้น ไม่ได้ครอบคลุมทุกกรณี — จึงไม่ควรพึ่งพากลไกนี้
เป็นเกราะป้องกันสุดท้าย

### 94.2 บทเรียนจากอุตสาหกรรมจริง

การเปิด `DEBUG = True` บน production เป็นหนึ่งใน **misconfiguration ที่พบบ่อยที่สุด**
ในรายงานด้าน security ของเว็บแอปพลิเคชันทั่วโลก (ทั้งใน Django และ framework อื่น
ที่มีโหมด debug คล้ายกัน) เพราะมักเกิดจากความสะเพร่า เช่น:

- นักพัฒนาเปิด `DEBUG = True` เพื่อ debug ปัญหาเร่งด่วนบน production แล้วลืมปิดกลับ
- ไม่มีระบบตรวจสอบอัตโนมัติ (เช่น `check --deploy` ในขั้นตอนที่ 99) ก่อน deploy จริง
- Deploy script ไม่ได้แยก environment variable ระหว่าง staging กับ production
  อย่างชัดเจน ทำให้ `.env` ของ dev หลุดไปใช้ที่ production โดยไม่ตั้งใจ

**กฎเหล็กของหลักสูตรนี้ที่ต้องจำให้ขึ้นใจ**: `DEBUG` ต้องมาจาก environment variable
เสมอ (ไม่ hardcode) และค่า default เมื่อไม่มีตัวแปรนี้เลยต้องเป็น **`False` เสมอ**
(fail-safe แบบปลอดภัยไว้ก่อน) ดังที่เราตั้งค่าไว้ตั้งแต่ Part 004:

```python
env = environ.Env(
    DEBUG=(bool, False),  # ถ้าลืมตั้งค่า .env จริง ๆ ระบบจะ "ปลอดภัยไว้ก่อน" เป็น False
)
```

### 94.3 สิ่งที่ควรมีแทนหน้า Debug Page บน production

เมื่อ `DEBUG = False` Django จะแสดงหน้า error ทั่วไป (generic) แทน ไม่เผยรายละเอียดใด ๆ
คุณสามารถสร้าง template ของตัวเองมาแทนที่หน้า default ได้ โดยวางไฟล์ไว้ที่ราก
โฟลเดอร์ template (ตาม `DIRS` ใน `TEMPLATES` ที่ตั้งไว้):

```html
<!-- templates/404.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ไม่พบหน้าที่คุณค้นหา</title>
</head>
<body>
    <h1>404 — ไม่พบหน้านี้</h1>
    <p>ขออภัย เราไม่พบหน้าที่คุณกำลังค้นหา</p>
    <a href="/">กลับไปหน้าหลัก</a>
</body>
</html>
```

```html
<!-- templates/500.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>เกิดข้อผิดพลาดบางอย่าง</title>
</head>
<body>
    <h1>500 — เกิดข้อผิดพลาดภายในระบบ</h1>
    <p>ทีมงานได้รับแจ้งเหตุการณ์นี้แล้ว กรุณาลองใหม่อีกครั้งภายหลัง</p>
</body>
</html>
```

Django จะใช้ template เหล่านี้อัตโนมัติเมื่อ `DEBUG = False` โดยไม่ต้องเขียน view หรือ
`urls.py` เพิ่มเติมใด ๆ (Django ค้นหาไฟล์ชื่อ `404.html`/`500.html` ที่รากของ template
directory ให้เองตามธรรมเนียม)

### 94.4 `ALLOWED_HOSTS` ที่ถูกต้องสำหรับ production

ทบทวนจาก Part 004: `ALLOWED_HOSTS` ป้องกันการโจมตีแบบ **HTTP Host Header Injection**
— ผู้โจมตีปลอมแปลง header `Host:` ของ request เพื่อหลอกให้ Django สร้าง URL ที่ชี้ไป
โดเมนอื่น (เช่น ใน password reset email ที่ฝัง link) ตัวอย่างการตั้งค่าที่ถูกต้อง:

```python
# ระบุ domain ที่ถูกต้องแบบเจาะจงเสมอ (ปลอดภัยที่สุด)
ALLOWED_HOSTS = ["example.com", "www.example.com", "api.example.com"]

# รองรับทุก subdomain ด้วย wildcard (ระวังใช้เฉพาะเมื่อจำเป็นจริง ๆ)
ALLOWED_HOSTS = [".example.com"]  # จุดนำหน้า = ครอบคลุมทุก subdomain ของ example.com

# ตอน deploy บน Docker/Kubernetes ที่ health check เข้าผ่าน internal IP
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")  # ตั้งค่าจริงผ่าน environment variable
```

**ข้อผิดพลาดที่พบบ่อยและอันตราย**: การตั้ง `ALLOWED_HOSTS = ["*"]` เพื่อ "แก้ปัญหา
ชั่วคราว" — นี่คือการ **ปิดการป้องกันทั้งหมด** ยอมให้ request จาก Host header ใดก็ได้
ผ่านเข้ามา ซึ่งเปิดช่องให้เกิดการโจมตีต่าง ๆ เช่น cache poisoning และการปลอมแปลง link
ใน password reset email **ห้ามใช้ `"*"` ใน production เด็ดขาด**

### 94.5 ความสัมพันธ์ระหว่าง `ALLOWED_HOSTS` และ `CSRF_TRUSTED_ORIGINS`

ทั้งสอง setting ทำหน้าที่คล้ายกันแต่ป้องกันคนละชั้น:

| Setting | ตรวจสอบอะไร | รูปแบบค่า |
|---|---|---|
| `ALLOWED_HOSTS` | header `Host` ของทุก request ที่เข้ามา | แค่ hostname เช่น `"example.com"` |
| `CSRF_TRUSTED_ORIGINS` | header `Origin`/`Referer` เฉพาะตอนทำ POST/PUT/DELETE ที่ต้องใช้ CSRF token | ต้องมี scheme เช่น `"https://example.com"` |

โปรเจกต์ที่ตั้งค่า `ALLOWED_HOSTS` ถูกต้องแต่ลืมตั้ง `CSRF_TRUSTED_ORIGINS` เมื่อมี
frontend แยก domain (เช่น React บน `app.example.com` เรียก API บน `api.example.com`)
มักเจอปัญหา `403 CSRF verification failed` — ต้องตั้งทั้งสอง setting ให้สอดคล้องกัน

### 94.6 `DEBUG_PROPAGATE_EXCEPTIONS` (เกร็ดความรู้ขั้นสูง)

```python
DEBUG_PROPAGATE_EXCEPTIONS = True
```

setting นี้ใช้ในสถานการณ์พิเศษ เช่น การรัน automated test ที่ต้องการให้ exception
"หลุด" ออกมาให้ test runner จับได้โดยตรง แทนที่ Django จะจับแล้วแปลงเป็น
`HttpResponseServerError` ให้เองแม้ตอน `DEBUG = False` — ใช้น้อยมากในงานทั่วไป
แต่มีประโยชน์เวลา debug ปัญหาที่ error handler ของ Django เองบดบังรายละเอียดที่ต้องการ

---

## ขั้นตอนที่ 95: SECRET_KEY management

### 95.1 ทบทวนบทบาทของ `SECRET_KEY`

จาก Part 004 เรารู้ว่า `SECRET_KEY` ใช้เซ็น (cryptographically sign) หลายจุดในระบบ:
session cookie, CSRF token, password reset token, และข้อมูลใด ๆ ที่ผ่าน
`django.core.signing` ขั้นตอนนี้เราจะเจาะลึกวิธี **จัดการวงจรชีวิต** ของค่านี้อย่าง
ปลอดภัย ตั้งแต่การสร้าง การหมุนเวียน (rotation) ไปจนถึงการจัดเก็บระดับองค์กร

### 95.2 วิธี Generate SECRET_KEY ที่ปลอดภัย

```bash
# วิธีที่ 1: ใช้ฟังก์ชันของ Django เอง (แนะนำที่สุด เพราะรับประกันความเข้ากันได้)
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"

# วิธีที่ 2: ใช้ module secrets มาตรฐานของ Python โดยตรง
python -c "import secrets; print(secrets.token_urlsafe(50))"

# วิธีที่ 3: ใช้ openssl (มีมาให้ในระบบปฏิบัติการส่วนใหญ่)
openssl rand -base64 48
```

ทั้งสามวิธีให้ผลลัพธ์ที่ปลอดภัยเพียงพอ (สุ่มด้วย cryptographically secure random
generator ไม่ใช่ `random` module ธรรมดาที่คาดเดาได้ง่ายกว่า) แต่หลักสูตรนี้แนะนำ
**วิธีที่ 1** เป็นค่าเริ่มต้น เพราะ `get_random_secret_key()` สร้างค่าที่มีรูปแบบและ
ความยาวตรงตามที่ Django framework คาดหวังพอดี

### 95.3 ทำไมห้าม commit SECRET_KEY ลง Git เด็ดขาด

ถ้า `SECRET_KEY` รั่วไหล (ไม่ว่าจะจาก Git history, ไฟล์ backup, หรือ log ที่หลุด)
ผู้โจมตีสามารถทำสิ่งเหล่านี้ได้ทันที:

1. **ปลอมแปลง session cookie**: สร้าง cookie ที่อ้างว่าเป็นผู้ใช้คนไหนก็ได้ รวมถึง
   superuser โดยไม่ต้องรู้รหัสผ่านจริงเลย
2. **บายพาส CSRF protection**: สร้าง CSRF token ปลอมที่ผ่านการตรวจสอบได้
3. **ปลอมแปลง password reset token**: สร้าง link รีเซ็ตรหัสผ่านของผู้ใช้คนใดก็ได้
   โดยไม่ต้องมีสิทธิ์เข้าถึงอีเมลของเขาเลย
4. **ถอดรหัสข้อมูลที่เซ็นด้วย `django.core.signing`**: ถ้าแอปใช้ signing เก็บข้อมูล
   sensitive อื่น ๆ (เช่น token ยืนยันตัวตนชั่วคราว)

นี่คือเหตุผลที่ **ทุก guideline ด้าน security ของ Django ระบุชัดเจนว่า SECRET_KEY
ต้องถูกเก็บเป็นความลับตลอดเวลา** เทียบเท่ากับรหัสผ่านฐานข้อมูลหรือ API key ระดับสูงสุด

**ถ้าคุณเคย commit SECRET_KEY ขึ้น Git โดยไม่ตั้งใจ** การแก้ไขไฟล์แล้ว commit ใหม่
**ไม่เพียงพอ** เพราะค่าเก่ายังอยู่ใน Git history ต้องทำ 2 อย่างพร้อมกัน:

1. **สร้าง SECRET_KEY ใหม่ทันทีและ deploy ไปแทนที่ค่าเดิม** (ค่าเก่าถือว่าถูกเผาไปแล้ว)
2. ใช้เครื่องมือเช่น `git filter-repo` หรือ BFG Repo-Cleaner ลบค่าที่รั่วไหลออกจาก
   ประวัติ Git ทั้งหมด (แม้จะลบออกจาก history ได้ แต่ต้อง treat ค่านั้นว่า "ถูกเผาไปแล้ว
   ตลอดกาล" อยู่ดี เพราะไม่มีทางรู้แน่ชัดว่ามีใครดาวน์โหลด repo ไปตอนที่ยังมีค่านั้นอยู่
   หรือไม่)

### 95.4 การ Rotate SECRET_KEY โดยไม่ทำให้ผู้ใช้ทุกคน logout พร้อมกัน

ปัญหาของการเปลี่ยน `SECRET_KEY` แบบตรง ๆ คือ **session cookie และ token ทั้งหมดที่
เซ็นด้วยคีย์เก่าจะใช้งานไม่ได้ทันที** ทำให้ผู้ใช้ทุกคนถูก logout พร้อมกันในทันทีที่
deploy ตั้งแต่ Django 4.1 เป็นต้นมา มี setting `SECRET_KEY_FALLBACKS` ที่แก้ปัญหานี้
โดยเฉพาะ:

```python
# config/settings/prod.py
SECRET_KEY = env("SECRET_KEY")  # คีย์ใหม่ที่ใช้เซ็นข้อมูลใหม่ทั้งหมด

SECRET_KEY_FALLBACKS = [
    env("SECRET_KEY_OLD"),  # คีย์เก่า — ยังใช้ "ตรวจสอบ" ข้อมูลเก่าได้ แต่ไม่ใช้เซ็นใหม่
]
```

Django จะใช้ `SECRET_KEY` (คีย์ใหม่) ในการเซ็นข้อมูลใหม่ทุกอย่างเสมอ แต่เวลา
**ตรวจสอบ** (verify) session/token ที่มีอยู่แล้ว จะลองไล่ตรวจกับคีย์ใน
`SECRET_KEY_FALLBACKS` ด้วย ถ้าตรงกับคีย์เก่าก็ยังถือว่าถูกต้อง — ทำให้ session เดิม
ของผู้ใช้ยังใช้งานต่อได้จนกว่าจะหมดอายุตามปกติ (`SESSION_COOKIE_AGE`) โดยไม่ต้อง
บังคับ logout ทันที

### 95.5 ขั้นตอนการ Rotate SECRET_KEY แบบปลอดภัยในทางปฏิบัติ

1. Generate คีย์ใหม่ด้วยคำสั่งในข้อ 95.2
2. ย้ายคีย์ปัจจุบันไปไว้ที่ `SECRET_KEY_FALLBACKS` (เป็นค่าแรกในลิสต์)
3. ตั้ง `SECRET_KEY` เป็นคีย์ใหม่ที่เพิ่ง generate
4. Deploy การเปลี่ยนแปลงนี้ (ผู้ใช้ปัจจุบันยังไม่ถูก logout เพราะมี fallback รองรับ)
5. รอจนกว่า session เก่าทั้งหมดที่เซ็นด้วยคีย์เก่าจะหมดอายุตามธรรมชาติ (เท่ากับค่า
   `SESSION_COOKIE_AGE` ที่ตั้งไว้ เช่น 2 สัปดาห์)
6. หลังจากนั้น ลบคีย์เก่าออกจาก `SECRET_KEY_FALLBACKS` ได้อย่างปลอดภัย

ควรทำการ rotate นี้เป็นประจำตามนโยบายความปลอดภัยขององค์กร (เช่น ทุก 6-12 เดือน) และ
**ต้องทำทันที** หากสงสัยว่าคีย์อาจรั่วไหล (พนักงานลาออกที่เคยเข้าถึง production secret,
เซิร์ฟเวอร์ถูกบุกรุก เป็นต้น)

### 95.6 การเก็บ SECRET_KEY ในระดับองค์กร (Secret Manager)

สำหรับทีมขนาดใหญ่ที่มีหลายเซิร์ฟเวอร์ การเก็บ secret ไว้ในไฟล์ `.env` บนแต่ละเครื่อง
เริ่มจัดการยาก จึงนิยมใช้ **Secret Manager** ระดับองค์กรแทน:

| เครื่องมือ | ผู้ให้บริการ |
|---|---|
| AWS Secrets Manager / Parameter Store | Amazon Web Services |
| Google Secret Manager | Google Cloud Platform |
| Azure Key Vault | Microsoft Azure |
| HashiCorp Vault | Self-hosted หรือ HCP Vault |

ระบบเหล่านี้ทำหน้าที่เก็บ secret แบบเข้ารหัสไว้ส่วนกลาง มีระบบ audit log ว่าใครเข้าถึง
เมื่อไร และรองรับการ rotate อัตโนมัติ ในทางปฏิบัติ แอป Django ยังคงอ่านค่าผ่าน
environment variable เหมือนเดิม (ผ่าน `django-environ` เหมือนที่เรียนมา) เพียงแต่
**ผู้ที่ set ตัวแปร environment variable นั้นให้ container/server ไม่ใช่ไฟล์ `.env`
บนดิสก์อีกต่อไป แต่เป็น secret manager ที่ดึงค่ามา inject ให้ตอน container เริ่มทำงาน**:

```python
# โค้ด Django ไม่เปลี่ยนแปลงเลย ไม่ว่าค่าจะมาจาก .env หรือ Secret Manager
SECRET_KEY = env("SECRET_KEY")
```

เราจะเจาะลึกการเชื่อมต่อกับ Secret Manager จริงเมื่อถึง Phase 11 (Deployment) และ
Phase 12 (DevOps/Cloud) — ตอนนี้ขอให้เข้าใจแค่หลักการว่า **โค้ดของแอปไม่ควรสนใจว่า
ค่ามาจากไหน ขอแค่มันมาอยู่ใน environment variable ให้ `env()` อ่านได้ก็พอ** นี่คือ
พลังของการยึดหลัก Twelve-Factor App ที่เรียนมาตั้งแต่ Part 004

### 95.7 สรุปกฎเหล็กเรื่อง SECRET_KEY

| กฎ | เหตุผล |
|---|---|
| ห้าม hardcode ในโค้ดเด็ดขาด | ป้องกันการรั่วไหลผ่าน Git |
| ห้าม commit ไฟล์ `.env` ที่มีค่าจริงขึ้น Git | เหตุผลเดียวกัน |
| Generate ด้วยเครื่องมือที่ปลอดภัยเท่านั้น | ป้องกันค่าที่คาดเดาได้ |
| Rotate เป็นระยะและทันทีเมื่อสงสัยว่ารั่วไหล | ลดผลกระทบเมื่อเกิดเหตุ |
| ใช้ `SECRET_KEY_FALLBACKS` ตอน rotate | ไม่ต้องบังคับ logout ผู้ใช้ทุกคนพร้อมกัน |
| ใช้ค่าคนละตัวกันระหว่าง dev/staging/prod เสมอ | จำกัดความเสียหายถ้า dev key รั่วไหล |

---

## ขั้นตอนที่ 96: ตั้งค่า Logging เบื้องต้นใน settings.py

### 96.1 ทำไมต้องใช้ Logging แทน `print()`

ระหว่างเรียนที่ผ่านมา คุณอาจเคยใช้ `print()` เพื่อ debug ค่าตัวแปรระหว่างพัฒนา วิธีนี้
ใช้ได้ตอนเรียนรู้ แต่ **ไม่เหมาะกับระบบจริง** ด้วยเหตุผลสำคัญ:

- `print()` เขียนไปที่ `stdout` เสมอ ไม่มีระดับความสำคัญ (info/warning/error) ให้กรอง
- ไม่มีการบันทึกเวลา (timestamp), ชื่อไฟล์/บรรทัดที่เกิดเหตุการณ์ให้อัตโนมัติ
- ไม่สามารถส่งไปหลายปลายทางพร้อมกันได้ (เช่น เขียนลงไฟล์ **และ** ส่งอีเมลแจ้งแอดมิน
  **และ** แสดงที่ console พร้อมกัน)
- เมื่อ deploy จริง `print()` มักหายไปเงียบ ๆ (ไม่มีใครเห็น) เพราะไม่มีระบบเก็บ log
  ที่เป็นมาตรฐาน

Python มี module มาตรฐานชื่อ `logging` ที่ Django integrate เข้ากับ `settings.py`
โดยตรงผ่าน dict ที่ชื่อ `LOGGING`

### 96.2 โครงสร้างของ `LOGGING` dict

`LOGGING` ใช้รูปแบบ **`dictConfig`** ของ Python (ไม่ใช่รูปแบบเฉพาะของ Django) มี
4 ส่วนหลัก:

```
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": { ... },   # กำหนดรูปแบบข้อความ log
    "handlers": { ... },     # กำหนดว่า log จะถูกส่งไปที่ไหน
    "loggers": { ... },      # กำหนดว่า logger ตัวไหนใช้ handler ไหน ที่ level ไหน
}
```

| ส่วน | หน้าที่ |
|---|---|
| `version` | ต้องเป็น `1` เสมอ (เวอร์ชันของ schema นี้ ปัจจุบันมีแค่เวอร์ชันเดียว) |
| `disable_existing_loggers` | ถ้า `True` จะปิด logger อื่นที่ third-party library ตั้งไว้ก่อนหน้า (ปกติตั้ง `False`) |
| `formatters` | เทมเพลตของข้อความ log แต่ละบรรทัด (จะใส่ timestamp, level, ชื่อ module ไหมเป็นต้น) |
| `handlers` | ปลายทางของ log (console, file, ส่งอีเมล ฯลฯ) |
| `loggers` | mapping ว่า logger ชื่อไหน (`"django"`, `"myapp"`) ใช้ handler ไหน ที่ level ไหน |

### 96.3 ตัวอย่าง `LOGGING` config เต็มรูปแบบ

```python
# config/settings/base.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "[{asctime}] {levelname} {name} {module}.{funcName}:{lineno} — {message}",
            "style": "{",
        },
        "simple": {
            "format": "{levelname}: {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": BASE_DIR / "logs" / "django.log",
            "maxBytes": 1024 * 1024 * 10,   # 10 MB ต่อไฟล์
            "backupCount": 5,                # เก็บไฟล์เก่าสูงสุด 5 ไฟล์
            "formatter": "verbose",
        },
        "mail_admins": {
            "class": "django.utils.log.AdminEmailHandler",
            "level": "ERROR",
            "formatter": "verbose",
        },
    },
    "loggers": {
        "django": {
            "handlers": ["console", "file"],
            "level": "INFO",
            "propagate": True,
        },
        "django.request": {
            "handlers": ["mail_admins", "file"],
            "level": "ERROR",
            "propagate": False,
        },
        "django.security": {
            "handlers": ["file", "mail_admins"],
            "level": "WARNING",
            "propagate": False,
        },
        "myapp": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}
```

**ข้อควรระวัง**: `RotatingFileHandler` เขียนไฟล์ลงโฟลเดอร์ `logs/` ซึ่งต้องมีอยู่จริง
ก่อนรันเซิร์ฟเวอร์ (Django ไม่สร้างโฟลเดอร์ให้อัตโนมัติ) ให้สร้างไว้ล่วงหน้าและเพิ่มใน
`.gitignore`:

```bash
mkdir logs
touch logs/.gitkeep
```

```
# .gitignore (เพิ่ม)
logs/*.log
```

### 96.4 Logger สำคัญที่ Django สร้างไว้ให้อัตโนมัติ

| ชื่อ Logger | บันทึกเหตุการณ์อะไร |
|---|---|
| `django` | logger แม่ (parent) ของทุก logger ย่อยของ Django เอง |
| `django.request` | error ระดับ 500 ทุกครั้งที่เกิดในการประมวลผล request |
| `django.server` | log ของ development server (`runserver`) เช่น request ที่เข้ามาแต่ละครั้ง |
| `django.template` | ข้อผิดพลาดขณะ render template |
| `django.db.backends` | SQL query ทุกตัวที่ ORM รันจริง (มีประโยชน์มากตอน debug performance แต่ **ควรปิดใน production** เพราะปริมาณ log จะมหาศาลและอาจโชว์ query ที่มีข้อมูล sensitive) |
| `django.security` | เหตุการณ์ด้าน security เช่น request ที่ถูกปฏิเสธเพราะ CSRF ไม่ตรง หรือ suspicious operation |

### 96.5 การใช้ Logging ในโค้ดแอปของตัวเอง

```python
# blog/views.py
import logging

from django.http import HttpResponse

logger = logging.getLogger(__name__)  # ชื่อ logger จะเป็น "blog.views" อัตโนมัติ


def article_detail(request, article_id):
    logger.debug("กำลังโหลดบทความ id=%s", article_id)
    try:
        # ... โค้ดดึงข้อมูลบทความ (จะเรียนจริงใน Phase 2 หลังมี Model) ...
        logger.info("โหลดบทความ id=%s สำเร็จ", article_id)
        return HttpResponse(f"บทความ #{article_id}")
    except Exception:
        logger.exception("เกิดข้อผิดพลาดขณะโหลดบทความ id=%s", article_id)
        raise
```

**ข้อสังเกตสำคัญ**: `logging.getLogger(__name__)` ใช้ `__name__` ของ module นั้น ๆ
เป็นชื่อ logger เสมอ (เช่น `blog.views`, `blog.models`) — นี่คือ **ธรรมเนียมมาตรฐาน**
ของ Python logging ทำให้เมื่อดู log สามารถรู้ทันทีว่าข้อความนั้นมาจากไฟล์ไหนของระบบ
และยังตั้งค่า `LOGGING["loggers"]` ให้ครอบคลุมทั้งแอป (`"blog": {...}`) หรือเจาะจง
เฉพาะโมดูลย่อยก็ได้ (`"blog.views": {...}`)

`logger.exception(...)` เป็นเมธอดพิเศษที่ **ต้องเรียกภายใน `except` block เท่านั้น**
มันจะบันทึก traceback แบบเต็มให้อัตโนมัติ เทียบเท่ากับเรียก `logger.error(..., exc_info=True)`

### 96.6 ระดับความสำคัญของ Log (Log Levels)

| Level | ตัวเลข | ใช้เมื่อไร |
|---|---|---|
| `DEBUG` | 10 | ข้อมูลละเอียดสำหรับ debug เท่านั้น (ปิดใน production) |
| `INFO` | 20 | เหตุการณ์ปกติที่น่าบันทึกไว้ เช่น "ผู้ใช้ login สำเร็จ" |
| `WARNING` | 30 | สิ่งผิดปกติแต่ระบบยังทำงานต่อได้ เช่น "การเชื่อมต่อ API ภายนอกช้าผิดปกติ" |
| `ERROR` | 40 | เกิดข้อผิดพลาดที่กระทบการทำงานของ request นั้น ๆ |
| `CRITICAL` | 50 | ข้อผิดพลาดร้ายแรงระดับระบบทั้งหมดอาจล่ม |

logger จะบันทึกเฉพาะข้อความที่มี level **เท่ากับหรือสูงกว่า** ค่าที่ตั้งไว้ใน `"level"`
เท่านั้น เช่น ถ้าตั้ง `"level": "INFO"` ข้อความระดับ `DEBUG` จะถูกข้ามไปเงียบ ๆ

### 96.7 แยก LOGGING ต่อ environment

```python
# config/settings/dev.py — เห็นทุกอย่างแบบละเอียดที่สุด รวมถึง SQL query
LOGGING["loggers"]["django.db.backends"] = {
    "handlers": ["console"],
    "level": "DEBUG",
    "propagate": False,
}
LOGGING["loggers"]["myapp"]["level"] = "DEBUG"

# config/settings/prod.py — เงียบกว่า เน้นเก็บลงไฟล์ + แจ้งเตือนแอดมินเมื่อ error เท่านั้น
LOGGING["loggers"]["django"]["level"] = "WARNING"
LOGGING["handlers"]["console"]["level"] = "WARNING"
```

รูปแบบนี้ใช้เทคนิคเดียวกับ setting อื่น ๆ ที่เรียนในขั้นตอนที่ 92: import `LOGGING`
(ทั้ง dict) มาจาก `base.py` ก่อน แล้วแก้ไขเฉพาะ key ที่ต้องต่างกันในแต่ละ environment
เมื่อถึง Phase 10 (Security) เราจะเพิ่ม handler สำหรับส่ง log ไปยังบริการภายนอกอย่าง
Sentry เพื่อติดตาม error แบบ real-time ในระดับ production จริง

---

## ขั้นตอนที่ 97: จัดการหลาย environment ผ่าน DJANGO_SETTINGS_MODULE

### 97.1 `DJANGO_SETTINGS_MODULE` คืออะไร

`DJANGO_SETTINGS_MODULE` คือ environment variable ที่ **ตัว Django framework เอง**
อ่านค่าเพื่อรู้ว่าจะ import settings จาก module ไหน เราเห็นค่านี้มาตั้งแต่ Part 004
ใน `manage.py`, `wsgi.py`, `asgi.py`:

```python
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
```

`os.environ.setdefault(key, value)` จะตั้งค่า `key = value` **ก็ต่อเมื่อ** `key` นั้น
ยังไม่มีอยู่ใน environment ของ process มาก่อน ซึ่งหมายความว่า **ถ้าคุณ export ตัวแปรนี้
ไว้ในระบบล่วงหน้า ค่านั้นจะชนะเสมอ** ไม่ว่าใน `manage.py` จะเขียน default เป็นอะไรไว้ก็ตาม
นี่คือกลไกที่ทำให้เราเปลี่ยน environment ได้โดยไม่ต้องแก้โค้ดเลยแม้แต่บรรทัดเดียว

### 97.2 วิธีตั้งค่า `DJANGO_SETTINGS_MODULE` ในสถานการณ์ต่าง ๆ

**ตั้งค่าชั่วคราวสำหรับ 1 คำสั่ง (macOS/Linux):**

```bash
DJANGO_SETTINGS_MODULE=config.settings.staging python manage.py migrate
```

**ตั้งค่าให้ทั้ง session ของ terminal (macOS/Linux):**

```bash
export DJANGO_SETTINGS_MODULE=config.settings.prod
python manage.py check
python manage.py collectstatic
# ทุกคำสั่งหลังจากนี้ในหน้าต่าง terminal เดียวกันจะใช้ prod settings จนกว่าจะปิด terminal
```

**Windows PowerShell:**

```powershell
$env:DJANGO_SETTINGS_MODULE = "config.settings.prod"
python manage.py check
```

**เก็บไว้ถาวรในไฟล์ `.env` แล้วให้ shell โหลดอัตโนมัติ** (สะดวกตอนพัฒนา):

```bash
# .envrc (ใช้กับเครื่องมือ direnv — โหลดอัตโนมัติเมื่อ cd เข้าโฟลเดอร์นี้)
export DJANGO_SETTINGS_MODULE=config.settings.dev
```

### 97.3 ตัวอย่างการรันแต่ละ environment ให้ครบวงจร

```bash
# --- Development (ในเครื่องตัวเอง) ---
export DJANGO_SETTINGS_MODULE=config.settings.dev
python manage.py runserver

# --- Staging (ทดสอบก่อนขึ้นจริง) ---
export DJANGO_SETTINGS_MODULE=config.settings.staging
python manage.py migrate
python manage.py collectstatic --noinput
gunicorn config.wsgi:application --bind 0.0.0.0:8000

# --- Production ---
export DJANGO_SETTINGS_MODULE=config.settings.prod
python manage.py migrate
python manage.py collectstatic --noinput
gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4

# --- Test (มักไม่ต้อง export เอง เพราะ pytest-django ตั้งให้อัตโนมัติผ่าน pytest.ini) ---
export DJANGO_SETTINGS_MODULE=config.settings.test
python manage.py test
```

### 97.4 `--settings` flag: override เฉพาะครั้งโดยไม่ต้อง export

ทุกคำสั่งของ `manage.py` รองรับ flag `--settings` ซึ่งมีความสำคัญ **สูงสุด** (แม้จะ
`export DJANGO_SETTINGS_MODULE` ไว้แล้วก็ตาม):

```bash
python manage.py shell --settings=config.settings.prod
python manage.py dbshell --settings=config.settings.staging
```

มีประโยชน์มากเวลาต้องการรันคำสั่งเดียวกับ environment อื่นชั่วคราว โดยไม่อยากไปยุ่งกับ
ตัวแปร environment ที่ตั้งไว้ทั้ง session

### 97.5 ตัวอย่างการตั้งค่าจริงตอน Deploy

**systemd service file** (สำหรับรัน Gunicorn เป็น background service บน Linux server):

```ini
# /etc/systemd/system/myproject.service
[Unit]
Description=My Django Project (Gunicorn)
After=network.target

[Service]
User=www-data
WorkingDirectory=/var/www/myproject
Environment="DJANGO_SETTINGS_MODULE=config.settings.prod"
EnvironmentFile=/var/www/myproject/.env.prod
ExecStart=/var/www/myproject/venv/bin/gunicorn config.wsgi:application --bind 0.0.0.0:8000

[Install]
WantedBy=multi-user.target
```

**Docker Compose:**

```yaml
# docker-compose.yml
services:
  web:
    build: .
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.prod
    env_file:
      - .env.prod
```

**Dockerfile (ตั้งค่า default ไว้ที่ image เอง):**

```dockerfile
ENV DJANGO_SETTINGS_MODULE=config.settings.prod
```

เราจะเจาะลึกทั้ง systemd และ Docker แบบเต็มรูปแบบใน Phase 11-12 ตอนนี้ขอให้เห็นภาพว่า
ไม่ว่าจะ deploy ด้วยเครื่องมือไหน หลักการเดียวกันคือ **ตั้ง `DJANGO_SETTINGS_MODULE`
ให้ถูกต้องก่อนที่แอปจะเริ่มทำงานเสมอ**

### 97.6 ข้อควรระวังที่พบบ่อยที่สุด

**อุบัติเหตุที่เกิดขึ้นบ่อยในทีมจริง**: ลืมตั้ง `DJANGO_SETTINGS_MODULE` ให้ถูกต้องตอน
deploy ทำให้ production รันด้วย `dev` settings โดยไม่ตั้งใจ (ซึ่งหมายความว่า
`DEBUG = True` รั่วไหลไปอยู่บน production ทันที ตามความเสี่ยงที่อธิบายในขั้นตอนที่ 94)
วิธีป้องกันที่ดีที่สุดคือ:

1. **ไม่ตั้งค่า default ที่ไม่ปลอดภัย** — ถ้าเป็นไปได้ ให้ `wsgi.py`/`asgi.py` (จุดเข้า
   ที่ production ใช้จริง) มี default เป็น `prod` เสมอ (ตามที่ทำในขั้นตอนที่ 92.9)
   เพื่อให้ "พลาดแล้วปลอดภัยไว้ก่อน" (fail-safe) แทนที่จะ "พลาดแล้วเสี่ยง" (fail-open)
2. ใช้คำสั่ง `python manage.py check --deploy` (ขั้นตอนที่ 99) เป็นส่วนหนึ่งของ
   CI/CD pipeline เพื่อดักจับการตั้งค่าผิดก่อนที่จะ deploy จริงเสมอ
3. Log ค่า `DJANGO_SETTINGS_MODULE` ที่ใช้จริงออกมาตอนเริ่มระบบ เพื่อยืนยันด้วยตาเปล่า
   ทุกครั้งที่ deploy

---

## ขั้นตอนที่ 98: python-decouple เป็นทางเลือกแทน django-environ

### 98.1 ทำความรู้จัก python-decouple

`python-decouple` เป็นอีกหนึ่ง library ยอดนิยมสำหรับแยก config ออกจากโค้ด มีปรัชญา
คล้ายกับ `django-environ` แต่มี API ที่ต่างกันเล็กน้อยและเรียบง่ายกว่าในบางแง่มุม

```bash
pip install python-decouple
```

### 98.2 การใช้งานพื้นฐานเทียบเคียงกับ django-environ

```
# .env (รูปแบบไฟล์เหมือนกันทุกประการ ไม่ต้องเปลี่ยน)
DEBUG=True
SECRET_KEY=your-secret-key-here
ALLOWED_HOSTS=127.0.0.1,localhost
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
```

```python
# settings.py แบบ python-decouple
from decouple import Csv, config

SECRET_KEY = config("SECRET_KEY")
DEBUG = config("DEBUG", default=False, cast=bool)
ALLOWED_HOSTS = config("ALLOWED_HOSTS", default="", cast=Csv())
```

เทียบเคียงกับ `django-environ` แบบเดียวกัน:

```python
# settings.py แบบ django-environ (ที่เราใช้มาตลอด)
import environ

env = environ.Env(DEBUG=(bool, False))
environ.Env.read_env()

SECRET_KEY = env("SECRET_KEY")
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=[])
```

ความแตกต่างหลักคือ `python-decouple` ใช้ **ฟังก์ชัน `config()` เดียว** พร้อม parameter
`cast` เพื่อแปลงชนิดข้อมูล (ต้องใช้ `Csv()` object แทนเมธอดแยกแบบ `env.list()`)
ในขณะที่ `django-environ` มี **เมธอดเฉพาะทาง** สำหรับแต่ละชนิดข้อมูล (`env.bool()`,
`env.list()`, `env.db()`) ทำให้สั้นกว่าในหลายกรณี

### 98.3 python-decouple ไม่มี `env.db()` หรือ `env.cache()` ในตัว

นี่คือข้อจำกัดสำคัญที่ทำให้ `django-environ` ได้เปรียบเมื่อทำงานกับ Cloud platform
ที่ส่ง `DATABASE_URL` มาให้ ถ้าใช้ `python-decouple` ต้องติดตั้ง library เสริมอย่าง
`dj-database-url` เพื่อ parse URL เป็น dict เอง:

```bash
pip install dj-database-url
```

```python
import dj_database_url
from decouple import config

DATABASES = {
    "default": dj_database_url.config(default=config("DATABASE_URL")),
}
```

### 98.4 ตารางเปรียบเทียบทางเลือกทั้งหมด

| Library | จุดเด่น | ข้อจำกัด |
|---|---|---|
| **django-environ** | ครบเครื่องที่สุด มี `env.db()`, `env.cache()`, `env.json()` ในตัว | API มีหลายเมธอดต้องจำ |
| **python-decouple** | เรียบง่าย เข้าใจง่ายมากสำหรับมือใหม่ ใช้ `config()` ฟังก์ชันเดียว | ต้องพึ่ง library เสริมสำหรับ parse URL |
| **python-dotenv** | เบาที่สุด (แค่โหลด `.env` เข้า `os.environ`) เป็นรากฐานที่หลาย library อื่นใช้ | ไม่มีระบบ type casting ในตัว ต้องแปลงเองทั้งหมด |
| **pydantic-settings** | Type-safe เต็มรูปแบบด้วย Pydantic model, validate error ชัดเจนมาก | ต้องเรียนรู้ Pydantic เพิ่ม, overhead สำหรับโปรเจกต์เล็ก |

### 98.5 คำแนะนำเลือกใช้

- **โปรเจกต์ที่ต้องเชื่อมต่อ Cloud platform (Heroku, Railway) บ่อย** → `django-environ`
  เพราะ `env.db()`/`env.cache()` สะดวกที่สุด (หลักสูตรนี้เลือกใช้ตัวนี้เป็นหลัก)
- **ทีมที่ต้องการความเรียบง่ายสูงสุด ไม่ยุ่งกับ database URL parsing** →
  `python-decouple` ก็เพียงพอและอ่านง่ายมาก
- **ทีมที่ให้ความสำคัญกับ type safety สูงมาก และคุ้นเคยกับ Pydantic อยู่แล้ว** →
  `pydantic-settings` (นิยมมากขึ้นเรื่อย ๆ ในโปรเจกต์ FastAPI/Django ผสมกัน)

ไม่ว่าจะเลือกเครื่องมือไหน **หลักการที่สำคัญที่สุดคือการแยก config ออกจากโค้ดเสมอ**
ตามที่เน้นย้ำมาตั้งแต่ Part 004 — เครื่องมือเป็นเพียงรายละเอียดปลีกย่อยที่เปลี่ยนได้

---

## ขั้นตอนที่ 99: Checklist ความปลอดภัยด้วย `python manage.py check --deploy`

### 99.1 รันคำสั่งและดูผลลัพธ์

Django มีคำสั่งตรวจสอบความพร้อมด้าน security สำหรับ production มาให้ในตัว ไม่ต้อง
ติดตั้ง library เพิ่มเติมใด ๆ:

```bash
python manage.py check --deploy --settings=config.settings.prod
```

ถ้ารันด้วย settings ที่ยังไม่ได้ปรับให้ปลอดภัย (เช่นค่า default ที่ `startproject`
สร้างให้) จะเห็น warning ประมาณนี้:

```
System check identified some issues:

WARNINGS:
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting.
    If your site is served exclusively over SSL, you may want to consider setting a
    value and enabling HTTP Strict Transport Security.
?: (security.W008) Your SECURE_SSL_REDIRECT setting is not set to True.
    Unless your site should be available over both SSL and non-SSL connections,
    you may want to either set this setting True or configure a load balancer or
    reverse-proxy server to redirect all connections to HTTPS.
?: (security.W009) Your SECRET_KEY has less than 50 characters, less than 5 unique
    characters, or it's prefixed with 'django-insecure-'.
?: (security.W012) SESSION_COOKIE_SECURE is not set to True.
?: (security.W016) CSRF_COOKIE_SECURE is not set to True.
?: (security.W018) You should not have DEBUG set to True in deployment.
?: (security.W019) You have not set the X_FRAME_OPTIONS setting.
?: (security.W021) You have not set the SECURE_HSTS_INCLUDE_SUBDOMAINS setting.

System check identified 8 issues (0 silenced).
```

### 99.2 ตารางอธิบายแต่ละ Warning Code พร้อมวิธีแก้

| Code | ปัญหา | วิธีแก้ |
|---|---|---|
| `security.W004` | ไม่ได้ตั้ง `SECURE_HSTS_SECONDS` | `SECURE_HSTS_SECONDS = 31536000` (1 ปี) |
| `security.W008` | `SECURE_SSL_REDIRECT` ไม่เป็น `True` | `SECURE_SSL_REDIRECT = True` |
| `security.W009` | `SECRET_KEY` สั้นเกินไปหรือยังใช้ค่า `django-insecure-` เริ่มต้น | Generate คีย์ใหม่ตามขั้นตอนที่ 95.2 |
| `security.W012` | `SESSION_COOKIE_SECURE` ไม่เป็น `True` | `SESSION_COOKIE_SECURE = True` |
| `security.W016` | `CSRF_COOKIE_SECURE` ไม่เป็น `True` | `CSRF_COOKIE_SECURE = True` |
| `security.W018` | `DEBUG = True` | ตั้ง `DEBUG = False` ใน production settings |
| `security.W019` | ไม่ได้ตั้ง `X_FRAME_OPTIONS` | `X_FRAME_OPTIONS = "DENY"` |
| `security.W021` | ไม่ได้ตั้ง `SECURE_HSTS_INCLUDE_SUBDOMAINS` | `SECURE_HSTS_INCLUDE_SUBDOMAINS = True` |
| `security.W022` | ไม่ได้ตั้ง `SECURE_HSTS_PRELOAD` | `SECURE_HSTS_PRELOAD = True` |

หลังจากใส่ค่าทั้งหมดตามที่แนะนำ (ซึ่งตรงกับ `prod.py` ที่เราเขียนไว้แล้วในขั้นตอนที่
92.6) ลองรันใหม่:

```bash
python manage.py check --deploy --settings=config.settings.prod
```

```
System check identified no issues (0 silenced).
```

### 99.3 นำไปใช้ใน CI/CD Pipeline เพื่อ Block การ Deploy ที่ไม่ปลอดภัย

`manage.py check` จะจบด้วย **exit code ที่ไม่ใช่ 0** เมื่อพบปัญหาระดับ error (ไม่ใช่
แค่ warning) ทำให้สามารถใช้เป็นเงื่อนไข block การ deploy อัตโนมัติได้:

```yaml
# ตัวอย่างขั้นตอนใน GitHub Actions (เจาะลึกเต็มรูปแบบใน Part 088)
- name: ตรวจสอบความปลอดภัยก่อน deploy
  run: |
    python manage.py check --deploy --settings=config.settings.prod --fail-level WARNING
```

flag `--fail-level WARNING` (เพิ่มมาตั้งแต่ Django 4.1) บอกให้คำสั่งนี้จบด้วย exit
code ที่ล้มเหลวแม้เจอแค่ระดับ **warning** (ปกติ `check` จะ fail เฉพาะระดับ `ERROR`
ขึ้นไปเท่านั้น) ทำให้ pipeline หยุดการ deploy ทันทีถ้ายังมี security warning ค้างอยู่
แม้แต่ข้อเดียว — นี่คือวิธีที่ทีมมืออาชีพป้องกัน "ลืมตั้งค่า" ไม่ให้หลุดไปถึง production จริง

### 99.4 `SILENCED_SYSTEM_CHECKS` — ปิด Warning ที่ยอมรับความเสี่ยงแล้วอย่างมีเหตุผล

บางครั้งทีมอาจตัดสินใจ **ยอมรับความเสี่ยง** บาง warning โดยมีเหตุผลรองรับชัดเจน เช่น
ระบบมี reverse proxy จัดการ HSTS header ให้แล้วในระดับ infrastructure ไม่จำเป็นต้อง
ให้ Django ตั้งซ้ำ ในกรณีนี้สามารถ silence warning เฉพาะตัวได้:

```python
SILENCED_SYSTEM_CHECKS = ["security.W004"]
```

**คำเตือนสำคัญ**: ใช้ setting นี้อย่างระมัดระวังที่สุด และควรมี comment อธิบายเหตุผล
กำกับไว้เสมอทุกครั้งที่ silence อะไรก็ตาม เพราะการ silence โดยไม่มีเหตุผลรองรับ
เท่ากับ **ปิดหูปิดตาตัวเองจากความเสี่ยงจริง** ไม่ใช่การแก้ปัญหา:

```python
SILENCED_SYSTEM_CHECKS = [
    # W004: HSTS header ถูกตั้งค่าโดย Nginx reverse proxy อยู่แล้วในระดับ infra
    # (ดูรายละเอียดที่ infra/nginx/security-headers.conf) — ตรวจสอบล่าสุดวันที่ 2026-01-15
    "security.W004",
]
```

---

## ขั้นตอนที่ 100: สรุป Phase 1 ทั้งหมด

ยินดีด้วย! คุณเดินทางมาถึงจุดสิ้นสุดของ **Phase 1: รากฐาน Python & Django** แล้ว
นี่คือขั้นตอนที่ 100 จาก 1000 ขั้นตอนของหลักสูตรทั้งหมด — ก่อนจะก้าวเข้าสู่ Phase 2
เราจะทบทวนภาพรวมทุกอย่างที่ผ่านมา ทดสอบความเข้าใจด้วย quiz และปิดท้ายด้วยแบบฝึกหัด
ใหญ่ที่รวบยอดทุกทักษะที่เรียนมาทั้ง 10 Part

### 100.1 ทบทวนภาพรวม Part 001-010

| Part | หัวข้อหลัก | สิ่งที่ได้เรียนรู้สำคัญที่สุด |
|---|---|---|
| **001** | บทนำสู่ Django และเตรียมเครื่องมือ | สถาปัตยกรรม MTV, ติดตั้ง Python/VS Code/Git/venv/Django |
| **002** | ทบทวน Python ที่จำเป็น | OOP, Decorators, Context Managers, Comprehension, Type Hints, `*args`/`**kwargs` |
| **003** | Virtual Environment และ pip | จัดการ dependency แยกต่อโปรเจกต์, `requirements.txt`, ทางเลือกอย่าง `uv` |
| **004** | สร้างโปรเจกต์ Django แรก | `startproject`, เจาะลึก `settings.py` เบื้องต้น, WSGI/ASGI, Project vs App, `.env` เบื้องต้น |
| **005** | Django Apps และการจัดระเบียบโค้ด | `startapp`, ไฟล์ในแอป, `INSTALLED_APPS`, การออกแบบขอบเขตของแอป |
| **006** | URL Routing และ URLconf | `path()`, `include()`, path converters, การตั้งชื่อ URL และ `reverse()` |
| **007** | Views แบบ Function-Based | FBV, `HttpRequest`/`HttpResponse`, HTTP methods, `render()`, `redirect()` |
| **008** | Django Template Language | Template inheritance, tags, filters, context |
| **009** | Static Files และ Media Files | `STATIC_URL`, `STATICFILES_DIRS`, `collectstatic`, การจัดการไฟล์อัปโหลด |
| **010** | Settings และ Environment Configuration | แยก settings หลายไฟล์, `django-environ` ขั้นสูง, security ของ `DEBUG`/`SECRET_KEY`, Logging, `check --deploy` |

### 100.2 แผนภาพความสัมพันธ์ของทุกสิ่งที่เรียนมา

```
                         ┌─────────────────────────────┐
                         │   config/settings/*.py       │  ← Part 004, 010
                         │   (.env per environment)     │
                         └───────────────┬───────────────┘
                                         │ ควบคุมพฤติกรรมทั้งหมด
                                         ▼
┌──────────────┐   HTTP Request   ┌─────────────────┐
│   Browser    │ ───────────────> │  config/urls.py  │  ← Part 006 (URL Routing)
│   (Client)   │                  └────────┬─────────┘
│              │ <─────────────── │         │ จับคู่ path
└──────────────┘   HTTP Response  │         ▼
                                   │  app/views.py     │  ← Part 005, 007 (Apps, FBV)
                                   │  (Function-Based) │
                                   └────────┬───────────┘
                                            │
                              ┌─────────────┼──────────────┐
                              ▼                             ▼
                    app/templates/*.html          static/ , media/
                    (Django Template Language)    (STATIC_URL, MEDIA_URL)
                        ← Part 008                    ← Part 009
```

สังเกตว่า **`settings` อยู่เหนือทุกอย่าง** เพราะทุกองค์ประกอบ (URL routing, views,
templates, static files) ล้วนถูกควบคุมพฤติกรรมโดย setting ใดสัก setting หนึ่งเสมอ
นี่คือเหตุผลที่ Part นี้ (Part 010) ถูกวางไว้เป็น Part สุดท้ายของ Phase 1 — เพื่อให้
คุณเห็นภาพรวมทั้งหมดก่อนที่จะเรียนรู้ Model และ ORM ใน Phase 2 ซึ่งเป็นองค์ประกอบ
สุดท้ายของสถาปัตยกรรม MTV ที่ยังไม่ได้พูดถึงในเชิงลึก

### 100.3 แบบทดสอบความเข้าใจรวม (Quiz)

ลองตอบคำถามต่อไปนี้ด้วยตัวเองก่อนเปิดดูเฉลย เพื่อประเมินว่าคุณพร้อมสำหรับ Phase 2
หรือยัง:

**1.** อธิบายความแตกต่างระหว่าง MVC และ MTV และบอกว่าคำว่า "View" ใน Django หมายถึง
อะไรในความหมายของ MVC ทั่วไป

**2.** ทำไมหลักสูตรนี้ (และทีมมืออาชีพจำนวนมาก) ถึงแนะนำให้ตั้งชื่อ Django project
package ว่า `config` แทนที่จะใช้ชื่อโปรเจกต์จริง

**3.** อธิบายบทบาทของไฟล์ `manage.py`, `wsgi.py`, และ `asgi.py` — แต่ละไฟล์ต่างกัน
อย่างไร และใช้ในสถานการณ์ไหน

**4.** `INSTALLED_APPS` กับ `MIDDLEWARE` ต่างกันอย่างไร ทำไมลำดับใน `MIDDLEWARE`
ถึงสำคัญ แต่ลำดับใน `INSTALLED_APPS` มีผลน้อยกว่ามาก

**5.** ทำไม `SECRET_KEY` ถึงห้าม hardcode ในโค้ดหรือ commit ขึ้น Git และถ้ารั่วไหล
ไปแล้วจะเกิดความเสียหายอะไรได้บ้าง

**6.** `ALLOWED_HOSTS = ["*"]` มีความเสี่ยงอย่างไร และควรตั้งค่าที่ถูกต้องแบบไหนแทน

**7.** อธิบายความแตกต่างระหว่าง `env("KEY")`, `env.list("KEY")`, และ `env.db("KEY")`
ของ `django-environ` พร้อมยกตัวอย่างการใช้งานแต่ละแบบ

**8.** เพราะเหตุใดการแยก `settings.py` เป็นหลายไฟล์ (`base.py`, `dev.py`, `prod.py`)
ถึงดีกว่าการใช้ if-else ในไฟล์เดียวสำหรับโปรเจกต์ทีม

**9.** `DJANGO_SETTINGS_MODULE` คืออะไร และมีลำดับความสำคัญเทียบกับ `--settings`
flag ของ `manage.py` อย่างไร

**10.** คำสั่ง `python manage.py check --deploy` ใช้ตรวจสอบอะไร และทำไมควรนำไปใส่
ไว้ใน CI/CD pipeline แทนที่จะรันด้วยมือเป็นครั้งคราว

**11. (ขั้นสูง)** อธิบายวิธีการ rotate `SECRET_KEY` โดยไม่ทำให้ผู้ใช้ทุกคน logout
พร้อมกันทันที โดยใช้ setting อะไร

**12. (ขั้นสูง)** ทำไม static files (`STATIC_URL`) กับ media files (`MEDIA_URL`)
ถึงต้องแยกกันอย่างชัดเจน และแต่ละอย่างควรถูก serve อย่างไรใน production

<details>
<summary>คลิกเพื่อดูเฉลยแบบย่อ</summary>

1. MVC = Model-View-Controller, MTV = Model-Template-View ของ Django "Template"
   ใน MTV เทียบเท่า "View" ใน MVC (สิ่งที่ผู้ใช้เห็น) ส่วน "View" ใน MTV เทียบเท่า
   "Controller" ใน MVC (ตรรกะรับ request/ส่ง response)
2. เพื่อป้องกันความสับสนกับชื่อแอปที่จะสร้างในอนาคต และสื่อความหมายชัดเจนว่าเป็น
   ที่เก็บการตั้งค่า ไม่ใช่ฟีเจอร์ของระบบ
3. `manage.py` = ประตูรันคำสั่งจัดการทั้งหมด, `wsgi.py` = จุดเข้าสำหรับ deploy แบบ
   synchronous (Gunicorn), `asgi.py` = จุดเข้าสำหรับ deploy แบบ asynchronous
   (Uvicorn, รองรับ WebSocket)
4. `INSTALLED_APPS` คือรายชื่อแอปที่ Django รู้จัก ลำดับมีผลน้อยในกรณีทั่วไป
   `MIDDLEWARE` คือชั้นประมวลผล request/response ที่ทำงานตามลำดับจริง (บนลงล่างตอน
   request, ล่างขึ้นบนตอน response) ลำดับผิดอาจทำให้ระบบทำงานผิดพลาด
5. เพราะ `SECRET_KEY` ใช้เซ็น session/CSRF/password reset token ถ้ารั่วไหล ผู้โจมตี
   สามารถปลอมแปลง session เป็นผู้ใช้ใดก็ได้ รวมถึง superuser
6. `"*"` อนุญาตทุก Host header เปิดช่องให้เกิด Host Header Injection ควรระบุ domain
   จริงแบบเจาะจง หรือใช้ wildcard subdomain แบบจำกัด เช่น `.example.com`
7. `env("KEY")` อ่านค่า string ธรรมดา, `env.list("KEY")` แยกค่าที่คั่นด้วย comma เป็น
   list, `env.db("KEY")` parse connection string เป็น dict ของ `DATABASES`
8. เพราะไฟล์แยกทำให้อ่านง่ายกว่า ลด merge conflict และลดความเสี่ยงที่จะเผลอใช้ค่าผิด
   environment เทียบกับ if-else ที่ปนกันในไฟล์เดียว
9. เป็น environment variable ที่บอก Django ว่าจะ import settings จาก module ไหน
   `--settings` flag มีความสำคัญสูงกว่าเสมอ (override ค่าที่ export ไว้)
10. ตรวจสอบความพร้อมด้าน security ของ settings ก่อน deploy production ควรใส่ใน
    CI/CD เพื่อป้องกันความผิดพลาดจากมนุษย์ (ลืมตั้งค่า) ไม่ให้หลุดไปถึงระบบจริง
11. ใช้ `SECRET_KEY_FALLBACKS` (Django 4.1+) เก็บคีย์เก่าไว้สำหรับ "ตรวจสอบ" session
    เดิม ในขณะที่คีย์ใหม่ใช้ "เซ็น" ข้อมูลใหม่ทั้งหมด
12. Static files คือไฟล์ที่มากับโค้ด (CSS/JS) ไม่เปลี่ยนบ่อย ส่วน Media files คือ
    ไฟล์ที่ผู้ใช้อัปโหลด (เปลี่ยนตลอดเวลา) ควร serve คนละที่กันเพื่อความปลอดภัยและ
    ประสิทธิภาพ (production มักใช้ CDN/Object Storage แยกสำหรับ media)

</details>

### 100.4 แบบฝึกหัดใหญ่ปิดท้าย Phase 1: สร้างโปรเจกต์ "Mini Blog" ให้สมบูรณ์

นี่คือแบบฝึกหัดที่รวบยอดทุกทักษะจาก Part 001-010 เข้าด้วยกัน **ข้อสังเกตสำคัญ**:
แม้ Part 005-009 จะพาคุณสร้าง `Post` model และฐานข้อมูลจริงไปแล้วสำหรับแอป `blog`
เดิม แต่แบบฝึกหัดปิดท้าย Phase นี้ให้คุณสร้างโปรเจกต์แยกต่างหากชื่อ `mini_blog`
โดยตั้งใจใช้ **โครงสร้างข้อมูลแบบ Python list/dict ธรรมดา** แทนฐานข้อมูลไปก่อน
เพื่อฝึกแยกชั้นข้อมูล (data layer) ออกจาก view/template/URL ให้ชัดเจน โดยไม่พึ่ง ORM
เลยแม้แต่น้อย — จะได้เห็นว่า Django ทำงานได้แม้ไม่มีฐานข้อมูลเลยก็ตาม เมื่อเรียนจบ
Phase 2 (เริ่ม Part 011) คุณจะย้อนกลับมาแปลงโปรเจกต์ `mini_blog` นี้ให้ใช้ Model และ
ฐานข้อมูลจริงแทน list/dict ได้ทันที เป็นแบบฝึกหัดเปรียบเทียบที่ดีระหว่างสองแนวทาง

**โจทย์**: สร้างเว็บบล็อกขนาดเล็กชื่อ `mini_blog` ที่มีคุณสมบัติครบตามนี้:

**ข้อกำหนดด้านโครงสร้างโปรเจกต์:**

1. สร้างโปรเจกต์ Django ใหม่ชื่อ `config` (ใช้ `django-admin startproject config .`)
2. แยก `settings` เป็นหลายไฟล์ตามที่เรียนในขั้นตอนที่ 92:
   `config/settings/{base,dev,staging,prod,test}.py`
3. สร้างแอปชื่อ `blog` ด้วย `python manage.py startapp blog` และลงทะเบียนใน
   `INSTALLED_APPS`

**ข้อกำหนดด้าน Views และข้อมูล:**

4. ในไฟล์ `blog/views.py` สร้างตัวแปร `POSTS` เป็น list ของ dict อย่างน้อย 5 รายการ
   แต่ละรายการมี key: `slug`, `title`, `content`, `author`, `published_date`

```python
# blog/views.py (ตัวอย่างโครงสร้างข้อมูลเริ่มต้น)
POSTS = [
    {
        "slug": "hello-django",
        "title": "สวัสดี Django",
        "content": "นี่คือบทความแรกของบล็อกนี้...",
        "author": "สมชาย",
        "published_date": "2026-01-10",
    },
    # ... เพิ่มอีกอย่างน้อย 4 รายการ ...
]
```

5. เขียน view function อย่างน้อย 3 ตัว:
   - `post_list(request)` — แสดงรายการบทความทั้งหมด
   - `post_detail(request, slug)` — แสดงเนื้อหาบทความเดียวตาม `slug` (ใช้ path
     converter `<str:slug>` ที่เรียนใน Part 006) ถ้าไม่พบให้คืน `Http404`
     (`from django.http import Http404`)
   - `about(request)` — หน้าแนะนำบล็อกสั้น ๆ

**ข้อกำหนดด้าน URL และ Template:**

6. เชื่อม URL ทั้งหมดผ่าน `blog/urls.py` แล้ว `include()` เข้า `config/urls.py`
   ตามรูปแบบที่เรียนใน Part 006 พร้อมตั้งชื่อ URL ทุกเส้นทาง (`name=...`)
7. สร้าง template แบบมี inheritance (Part 008): `templates/base.html` เป็นโครงหลัก
   แล้วให้ `post_list.html`, `post_detail.html`, `about.html` extends จากมัน
8. ใน `post_list.html` ใช้ `{% for %}` วนแสดงหัวข้อบทความทุกอันพร้อมลิงก์ไปหน้า
   detail (ใช้ `{% url %}` tag ไม่ hardcode path)

**ข้อกำหนดด้าน Static Files:**

9. สร้างไฟล์ `static/css/style.css` อย่างน้อย 1 ไฟล์ และเชื่อมเข้ากับ `base.html`
   ด้วย `{% load static %}` และ `{% static %}` ตามที่เรียนใน Part 009

**ข้อกำหนดด้าน Settings/Environment (จาก Part นี้):**

10. สร้างไฟล์ `.env.dev` และ `.env.prod` ที่มี `SECRET_KEY` คนละค่ากัน (generate
    ด้วยคำสั่งในขั้นตอนที่ 95.2) และตรวจสอบว่าไฟล์เหล่านี้อยู่ใน `.gitignore` แล้ว
11. ตั้งค่า `LOGGING` ใน `base.py` ให้บันทึกทุกครั้งที่มีคนเข้าดูบทความ (เรียก
    `logger.info(...)` ใน `post_detail` view) ลงไฟล์ `logs/django.log`
12. รัน `python manage.py check --deploy --settings=config.settings.prod` และแก้ไข
    ทุก warning จนกว่าจะเห็นข้อความ `System check identified no issues`
13. สร้าง template `templates/404.html` ของตัวเอง (ตามขั้นตอนที่ 94.3) แล้วทดสอบ
    ด้วยการตั้ง `DEBUG = False` ชั่วคราวและเข้า URL ที่ไม่มีอยู่จริง

**เกณฑ์ตรวจสอบความสำเร็จ (Checklist ปิดท้าย Phase 1):**

- [ ] `python manage.py runserver --settings=config.settings.dev` รันได้ไม่มี error
- [ ] เข้าหน้ารายการบทความ เห็นบทความทั้ง 5 รายการพร้อมลิงก์ที่คลิกได้จริง
- [ ] คลิกลิงก์แล้วเข้าหน้า detail ของบทความนั้นได้ถูกต้องตาม `slug`
- [ ] เข้า URL บทความที่ไม่มีอยู่จริง (เช่น `/posts/ไม่มีจริง/`) แล้วเห็นหน้า 404
      ที่ออกแบบเอง (ไม่ใช่หน้า debug ของ Django)
- [ ] หน้าเว็บมี CSS ที่โหลดจาก `static/` ทำงานถูกต้อง (พื้นหลัง/สี/ font เปลี่ยนไป)
- [ ] ไฟล์ `logs/django.log` มีบันทึกทุกครั้งที่เข้าหน้า detail
- [ ] `.env.dev` และ `.env.prod` ไม่ปรากฏใน `git status` (ถูก `.gitignore` ดักไว้)
- [ ] `python manage.py check --deploy --settings=config.settings.prod` ผ่านโดยไม่มี
      warning ใด ๆ เหลืออยู่
- [ ] push โค้ดทั้งหมดขึ้น GitHub สำเร็จ (ยกเว้นไฟล์ `.env.*` และ `venv/`)

**ระดับขั้นสูง (โบนัส)**: ลองเขียนฟังก์ชันค้นหาบทความง่าย ๆ ด้วย query parameter เช่น
`/posts/?q=django` ที่กรอง `POSTS` list ด้วย Python list comprehension (ทบทวนจาก
Part 002) แสดงเฉพาะบทความที่ title มีคำค้นหานั้นอยู่ — นี่คือการซ้อมมือก่อนที่จะได้
เรียนการค้นหาด้วย Django ORM จริงใน Phase 2

### 100.5 คำนำสู่ Phase 2: Models, ORM และ Admin

แบบฝึกหัดใหญ่ที่ทำไปข้างต้นน่าจะทำให้คุณรู้สึกถึง **ข้อจำกัดที่ชัดเจน** ของการเก็บข้อมูล
เป็น Python list ธรรมดา: ข้อมูลหายไปทุกครั้งที่รีสตาร์ทเซิร์ฟเวอร์, ไม่มีทางเพิ่ม/แก้ไข/
ลบบทความผ่านหน้าเว็บได้จริง, ค้นหาข้อมูลก็ต้องเขียน loop เองทุกครั้ง — นี่คือปัญหาที่
**Django Model และ ORM (Object-Relational Mapping)** จะเข้ามาแก้ไขทั้งหมด

**Phase 2: Models, ORM และ Admin (Part 011-020 | ขั้นตอนที่ 101-200)** จะพาคุณ:

- **Part 011**: เรียนรู้ Django Models เบื้องต้น — แปลง `POSTS` list ที่เขียนไว้ในแบบ
  ฝึกหัดนี้ให้กลายเป็น Model class จริงที่เชื่อมกับฐานข้อมูล พร้อมระบบ Migration
  ที่ติดตามการเปลี่ยนแปลงโครงสร้างตาราง
- **Part 012**: ความสัมพันธ์ระหว่างโมเดล (ForeignKey, ManyToMany) เช่น เชื่อม
  บทความเข้ากับผู้เขียนและหมวดหมู่จริง ๆ
- **Part 013-014**: Django ORM QuerySet ขั้นสูง, Aggregation และ Q/F Expressions
  แทนที่การเขียน list comprehension เอง
- **Part 017-018**: Django Admin — เปิดหน้าจัดการบทความผ่านเว็บได้ทันทีโดยไม่ต้อง
  เขียนฟอร์มเองสักตัว (ฟีเจอร์ที่ทำให้ Django โดดเด่นเหนือ framework อื่นตามที่เรียน
  ใน Part 001)

จากนี้ไป **ฐานข้อมูล PostgreSQL** จะเข้ามาแทนที่ SQLite ตามที่ระบุไว้ในคำแนะนำของ
Part 001 (ขั้นตอนที่ 9.4) และทุกอย่างที่เรียนใน Phase 1 — การตั้งค่า settings แบบ
แยกไฟล์, `django-environ`, การจัดการ `SECRET_KEY`, Logging — จะยังคงถูกใช้งานต่อเนื่อง
และมีความสำคัญมากขึ้นเรื่อย ๆ เมื่อโปรเจกต์เริ่มมีข้อมูลจริงที่ต้องปกป้อง

เตรียม virtual environment ให้พร้อม (ยัง activate อยู่เสมอ) ตรวจสอบว่า PostgreSQL
ถูกติดตั้งในเครื่องแล้ว (เราจะสอนวิธีติดตั้งใน Part 011) และเตรียมใจสำหรับหัวใจสำคัญ
ที่สุดอย่างหนึ่งของ Django — **Object-Relational Mapping** — ที่กำลังจะเปลี่ยนวิธีที่
คุณมองข้อมูลในเว็บแอปพลิเคชันไปตลอดกาล

ยินดีด้วยอีกครั้งที่จบ Phase 1 แล้ว! เจอกันที่ Part 011 🎉
