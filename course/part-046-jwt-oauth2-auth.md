# Part 046: Authentication ขั้นสูง: JWT, Token, OAuth2

> **ขั้นตอนที่ 451-460 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: เข้าใจแนวคิด JWT (JSON Web Token) อย่างถ่องแท้ตั้งแต่โครงสร้าง
> ภายในจนถึงเหตุผลที่ API ระดับโลกส่วนใหญ่เลือกใช้ stateless authentication ติดตั้งและ
> ใช้งาน `djangorestframework-simplejwt` แบบเต็มรูปแบบ (obtain/refresh/verify, blacklisting,
> token rotation, custom claims) ต่อยอดไปถึงการทำให้ API ของคุณเองเป็น **OAuth2 Provider**
> ด้วย `django-oauth-toolkit` สำหรับให้ third-party app เชื่อมต่อ และ **API Key
> Authentication** ด้วย `djangorestframework-api-key` สำหรับการเชื่อมต่อแบบ
> machine-to-machine ปิดท้ายด้วยตารางเปรียบเทียบวิธี authentication ทั้งหมดที่เรียนมา
> ในหลักสูตรนี้ และ Security Best Practices ระดับ production พร้อมลงมือแทนที่
> Token Authentication เดิมของ `blog` API (จาก Part 045) ด้วย JWT แบบเต็มรูปแบบ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 451: แนวคิด JWT (JSON Web Token) — โครงสร้าง header.payload.signature และทำไม stateless auth ถึงเหมาะกับ API
- ขั้นตอนที่ 452: ติดตั้งและตั้งค่า `djangorestframework-simplejwt`
- ขั้นตอนที่ 453: Access Token vs Refresh Token flow เต็มรูปแบบ (obtain/refresh/verify)
- ขั้นตอนที่ 454: Token Blacklisting และ Token Rotation
- ขั้นตอนที่ 455: การใส่ Custom Claims เข้าไปใน JWT payload
- ขั้นตอนที่ 456: ทำ API ของตัวเองเป็น OAuth2 Provider ด้วย `django-oauth-toolkit`
- ขั้นตอนที่ 457: API Key Authentication ด้วย `djangorestframework-api-key`
- ขั้นตอนที่ 458: ตารางเปรียบเทียบวิธี Authentication ทั้งหมด — เมื่อไหร่ควรใช้อะไร
- ขั้นตอนที่ 459: Security Best Practices สำหรับ API Authentication
- ขั้นตอนที่ 460: สรุปและแบบฝึกหัด — ย้าย `blog` API ทั้งหมดไปใช้ JWT

---

## ขั้นตอนที่ 451: แนวคิด JWT (JSON Web Token) — โครงสร้าง header.payload.signature และทำไม stateless auth ถึงเหมาะกับ API

### 451.1 ทบทวน Authentication ที่เรียนมาแล้วใน Part 045

ใน Part 045 คุณตั้งค่า `blog` API ให้รองรับสอง authentication scheme พร้อมกัน:

```python
# config/settings.py (จาก Part 045)
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}
```

ทั้งสองแบบทำงานได้ดี แต่มีข้อจำกัดร่วมกันที่สำคัญ: **ทั้งคู่เป็น stateful** กล่าวคือทุกครั้ง
ที่ request เข้ามา Django ต้อง **query ฐานข้อมูล** เพื่อตรวจสอบว่า session key หรือ token
string นั้นมีอยู่จริงและยังไม่หมดอายุหรือไม่:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.authentication.TokenAuthentication
class TokenAuthentication(BaseAuthentication):
    def authenticate_credentials(self, key):
        model = self.get_model()
        try:
            token = model.objects.select_related('user').get(key=key)  # ← DB hit ทุก request!
        except model.DoesNotExist:
            raise exceptions.AuthenticationFailed('Invalid token.')
        return (token.user, token)
```

ทุก 1 request ที่มี header `Authorization: Token <key>` เข้ามา DRF ต้อง `SELECT` ตาราง
`authtoken_token` หนึ่งครั้งเสมอ ก่อนที่ handler ของ view จะได้ทำงานด้วยซ้ำ — เมื่อ API มี
traffic สูงมาก (หลักพัน-หลักหมื่น request ต่อวินาที) การ query DB ซ้ำ ๆ เพื่อ "แค่ยืนยันตัวตน"
กลายเป็นคอขวด (bottleneck) ที่ชัดเจน

### 451.2 JWT คืออะไรกันแน่

**JWT (JSON Web Token)** คือมาตรฐานเปิด (มาตรฐาน [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519))
สำหรับสร้าง token ที่ **บรรจุข้อมูลผู้ใช้ไว้ในตัวเอง** และ **เซ็นลายเซ็นดิจิทัล (digital
signature)** กำกับไว้ ทำให้ server สามารถ **ตรวจสอบความถูกต้องของ token ได้โดยไม่ต้อง
เปิดฐานข้อมูลเลย** — เพียงตรวจลายเซ็นด้วย secret key ที่ server มีอยู่แล้วเท่านั้น

นี่คือความแตกต่างเชิงปรัชญาที่สำคัญที่สุด:

| ลักษณะ | Session/Token Authentication | JWT |
|---|---|---|
| ข้อมูลผู้ใช้เก็บอยู่ที่ไหน | ในฐานข้อมูล (server เก็บ state) | ในตัว token เอง (client ถือ state) |
| การตรวจสอบแต่ละ request | Query DB เพื่อดู record | ตรวจลายเซ็นด้วย secret key (ไม่ query DB) |
| เรียกว่าแบบไหน | **Stateful** | **Stateless** |
| Scale ข้าม server หลายเครื่องได้ไหม | ต้องใช้ shared session store (Redis) | ได้ทันที ทุก server ตรวจลายเซ็นได้เอง |

### 451.3 โครงสร้างของ JWT: `header.payload.signature`

JWT ที่จริงคือ**สตริงข้อความธรรมดา**ยาว ๆ ที่ประกอบด้วย 3 ส่วน คั่นด้วยจุด (`.`) เสมอ:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6InNvbWNoYWkiLCJleHAiOjE3MzAwMDAwMDB9.4f8e2a1c9b3d7e5f6a0c8b2d4e6f1a3c5b7d9e0f2a4c6b8d0e2f4a6c8b0d2e4f
└──────────────── HEADER ────────────────┘ └────────────────────── PAYLOAD ──────────────────────┘ └──────────────────── SIGNATURE ────────────────────┘
```

แต่ละส่วนคือ **Base64URL-encoded JSON** (ไม่ใช่การเข้ารหัส/encryption แบบอ่านไม่ได้ —
เพียงแค่ "แปลงรูปแบบ" เท่านั้น ใครก็ตามที่มี token สามารถถอด header/payload อ่านได้เสมอ)

ลองถอดรหัสด้วย Python ใน `python manage.py shell` เพื่อพิสูจน์:

```python
>>> import base64
>>> import json
>>>
>>> def decode_jwt_part(part):
...     # เติม padding เพราะ Base64URL ตัด '=' ท้ายออก
...     padded = part + '=' * (-len(part) % 4)
...     return json.loads(base64.urlsafe_b64decode(padded))
...
>>> token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6InNvbWNoYWkiLCJleHAiOjE3MzAwMDAwMDB9.4f8e2a1c9b3d7e5f"
>>> header_part, payload_part, signature_part = token.split('.')
>>>
>>> decode_jwt_part(header_part)
{'alg': 'HS256', 'typ': 'JWT'}
>>>
>>> decode_jwt_part(payload_part)
{'user_id': 1, 'username': 'somchai', 'exp': 1730000000}
```

### 451.4 อธิบายแต่ละส่วนโดยละเอียด

#### Header

บอกว่า token นี้ใช้อัลกอริทึมอะไรในการเซ็นลายเซ็น:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

| Algorithm | ประเภท | ใช้อย่างไร |
|---|---|---|
| `HS256` (HMAC-SHA256) | Symmetric (secret เดียว) | server เดียวกันทั้งเซ็นและตรวจสอบด้วย secret key เดียวกัน — ค่า default ของ `simplejwt` |
| `RS256` (RSA-SHA256) | Asymmetric (public/private key) | server ที่ออก token ใช้ private key เซ็น ส่วน server อื่น ๆ ใช้แค่ public key ตรวจสอบได้ — เหมาะกับระบบ microservices ที่มีหลาย service ต้องตรวจ token แต่ไม่ควรมีสิทธิ์ออก token เอง |

#### Payload (Claims)

ข้อมูลจริงที่ฝังอยู่ใน token เรียกว่า **claims**:

```json
{
  "token_type": "access",
  "exp": 1730000000,
  "iat": 1729999700,
  "jti": "a1b2c3d4e5f6...",
  "user_id": 1
}
```

| Claim | ความหมาย |
|---|---|
| `exp` (expiration) | เวลาหมดอายุ (Unix timestamp) — server จะปฏิเสธ token ที่ `exp` ผ่านไปแล้วทันที |
| `iat` (issued at) | เวลาที่สร้าง token |
| `jti` (JWT ID) | รหัสเฉพาะของ token นี้ — ใช้ทำ blacklist ได้ (ขั้นตอนที่ 454) |
| `user_id` | claim ที่ `simplejwt` ใส่ให้อัตโนมัติ ระบุว่า token นี้เป็นของ user คนไหน |
| `token_type` | บอกว่าเป็น `access` หรือ `refresh` token (ขั้นตอนที่ 453) |

#### Signature

ส่วนที่สำคัญที่สุดในเชิงความปลอดภัย คำนวณจากสูตร:

```
signature = HMAC-SHA256(
    base64url(header) + "." + base64url(payload),
    secret_key
)
```

ถ้าใครแก้ไข payload แม้แต่ตัวอักษรเดียว (เช่น เปลี่ยน `"user_id": 1` เป็น `"user_id": 2`)
แล้วส่ง token กลับมา server จะคำนวณ signature ใหม่จาก payload ที่ถูกแก้ไข แล้วพบว่า
**ไม่ตรง** กับ signature เดิมที่แนบมา จึงปฏิเสธ token ทันที — นี่คือกลไกที่ทำให้ JWT
**ปลอมแปลงไม่ได้โดยไม่รู้ secret key** แม้ว่าใครก็ตามจะ "อ่าน" เนื้อหาข้างในได้ก็ตาม

**ข้อควรระวังที่สำคัญที่สุดของขั้นตอนนี้**: เพราะ payload อ่านได้เสมอ (แค่ base64 decode)
**ห้ามใส่ข้อมูลลับ** เช่น รหัสผ่าน, บัตรเครดิต, หรือข้อมูลส่วนตัวที่อ่อนไหวลงใน payload
เด็ดขาด — JWT ปกป้องแค่ **ความถูกต้อง (integrity)** ไม่ได้ปกป้อง **ความลับ (confidentiality)**

### 451.5 ทำไม Stateless Auth ถึงเหมาะกับ API เป็นพิเศษ

| เหตุผล | รายละเอียด |
|---|---|
| **Horizontal Scaling** | เพิ่ม server กี่เครื่องก็ได้โดยไม่ต้อง sync session store — ทุกเครื่องมี secret key เดียวกันก็ตรวจ token ได้เอง |
| **ไม่ต้อง query DB ทุก request** | ลด latency และภาระฐานข้อมูลได้มากในระบบที่มี traffic สูง |
| **เหมาะกับ Microservices** | Service B ตรวจสอบตัวตนผู้ใช้ที่ Service A ออก token ให้ได้ โดยไม่ต้องเรียก Service A กลับไปถามทุกครั้ง (ยิ่งชัดเจนเมื่อใช้ `RS256`) |
| **Mobile App / SPA friendly** | ไม่ต้องพึ่ง cookie ที่มีปัญหาเรื่อง cross-domain — ส่ง JWT ผ่าน `Authorization` header ได้ตรงไปตรงมา |
| **Cross-platform มาตรฐานเดียว** | JWT เป็นมาตรฐานเปิด รองรับแทบทุกภาษาโปรแกรมมิ่ง ไม่ผูกติดกับ Django เพียงอย่างเดียว |

### 451.6 ข้อเสียของ JWT ที่ต้องรู้ไว้ล่วงหน้า (จะแก้ในขั้นตอนที่ 454)

Stateless ไม่ได้มีแต่ข้อดี ข้อเสียที่สำคัญที่สุดคือ **การเพิกถอน (revoke) token ก่อนหมดอายุ
ทำได้ยากกว่า Session/Token แบบเดิม** เพราะ server ไม่ได้เก็บ record ของ token ไว้ที่ไหนเลย
ถ้า token รั่วไหลไปอยู่ในมือคนร้าย server จะยังคง "เชื่อ" token นั้นไปจนกว่าจะหมดอายุตาม
`exp` ที่กำหนดไว้ — ปัญหานี้คือเหตุผลที่ `djangorestframework-simplejwt` ต้องมีระบบ
**Blacklisting** เสริมเข้ามา ซึ่งเราจะเจาะลึกในขั้นตอนที่ 454

### 451.7 ตารางสรุปขั้นตอนที่ 451

| ประเด็น | สรุป |
|---|---|
| JWT คือ | Token ที่บรรจุข้อมูลผู้ใช้ในตัวเอง + เซ็นลายเซ็นดิจิทัลกำกับ |
| โครงสร้าง | `header.payload.signature` (แต่ละส่วนคือ Base64URL JSON) |
| Header/Payload | อ่านได้เสมอ ไม่ใช่การเข้ารหัส ห้ามใส่ข้อมูลลับ |
| Signature | ปลอมแปลงไม่ได้หากไม่รู้ secret key — ปกป้อง integrity ไม่ใช่ confidentiality |
| จุดแข็ง | Stateless, scale ง่าย, ไม่ query DB ทุก request, ข้าม service ได้ |
| จุดอ่อน | Revoke ก่อนหมดอายุยาก ต้องแก้ด้วย blacklist (ขั้นตอนที่ 454) |

---

## ขั้นตอนที่ 452: ติดตั้งและตั้งค่า `djangorestframework-simplejwt`

### 452.1 ทำไมเลือก `simplejwt` ไม่เขียน JWT เอง

แม้จะเข้าใจกลไกเบื้องหลังแล้วจากขั้นตอนที่ 451 แต่การเขียนระบบเซ็น/ตรวจสอบ JWT เอง
ในโปรเจกต์จริงเป็นความเสี่ยงด้านความปลอดภัยที่ไม่คุ้มค่า (เช่น ลืมตรวจ `exp`, เลือก
algorithm ไม่ปลอดภัย, จัดการ secret ผิดวิธี) `djangorestframework-simplejwt` คือ package
มาตรฐานที่ community DRF ใช้กันมากที่สุด ผ่านการตรวจสอบความปลอดภัยมานาน และผสาน
เข้ากับ DRF authentication framework ที่เรียนมาตั้งแต่ Part 045 ได้ทันที

### 452.2 ติดตั้ง Package

```bash
# ตรวจสอบให้แน่ใจว่า venv ถูก activate อยู่ก่อนเสมอ
pip install djangorestframework-simplejwt

# บันทึกลง requirements.txt ตามหลักที่เรียนมาตั้งแต่ Part 001
pip freeze | grep -i jwt >> requirements.txt
```

ตรวจสอบเวอร์ชัน:

```bash
python -c "import rest_framework_simplejwt; print(rest_framework_simplejwt.__version__)"
# ผลลัพธ์ที่คาดหวัง: 5.x.x ขึ้นไป (รองรับ Django 5.x)
```

### 452.3 ตั้งค่า `DEFAULT_AUTHENTICATION_CLASSES`

เพิ่ม `JWTAuthentication` เข้าไปในรายการ authentication classes ของ DRF (ยังคงเก็บ
`SessionAuthentication` ไว้สำหรับ Browsable API และ `TokenAuthentication` ไว้ชั่วคราว
เพื่อไม่ให้ client เดิมที่ยังใช้ Token อยู่พังทันที — เราจะถอด `TokenAuthentication` ออก
อย่างเป็นทางการในขั้นตอนที่ 460):

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}
```

**ข้อควรระวังเรื่องลำดับ**: DRF จะไล่ทดสอบ authentication class ทีละตัวตามลำดับที่ประกาศไว้
ในลิสต์ จนกว่าจะเจอตัวแรกที่ authenticate สำเร็จ (หรือลองครบทุกตัวแล้วไม่ผ่านเลยจะถือว่า
เป็น anonymous user) การวาง `JWTAuthentication` ไว้บนสุดไม่ได้แปลว่า "บังคับใช้ JWT
เท่านั้น" — client ที่ส่ง `Authorization: Token <key>` มาก็ยังผ่าน `TokenAuthentication`
ที่อยู่ถัดไปได้ปกติ เพราะแต่ละ class จะข้ามตัวเองไปเงียบ ๆ ถ้า header ไม่ตรงรูปแบบที่ตนรู้จัก

### 452.4 ตั้งค่า `SIMPLE_JWT` Settings Dictionary

`simplejwt` มี settings เฉพาะของตัวเองแยกจาก `REST_FRAMEWORK` เรียกว่า `SIMPLE_JWT`
มาดูค่าที่สำคัญที่สุดพร้อมคำอธิบายทีละตัว:

```python
# config/settings.py
from datetime import timedelta

SIMPLE_JWT = {
    # อายุของ access token — สั้น เพราะเป็นตัวที่ใช้เรียก API จริงทุกครั้ง
    # หากรั่วไหล ความเสียหายจะจำกัดอยู่ในกรอบเวลาสั้น ๆ นี้เท่านั้น
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),

    # อายุของ refresh token — ยาวกว่า เพราะใช้แค่ "ขอ access token ใหม่" ไม่ได้ใช้เรียก API ตรง ๆ
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),

    # เมื่อ refresh สำเร็จ ออก refresh token ใหม่ให้ทุกครั้ง (เจาะลึกในขั้นตอนที่ 454)
    'ROTATE_REFRESH_TOKENS': True,

    # เมื่อ rotate แล้ว ให้ blacklist refresh token เก่าทันที ป้องกันเอาไปใช้ซ้ำ
    'BLACKLIST_AFTER_ROTATION': True,

    # อัปเดตฟิลด์ last_login ของ User ทุกครั้งที่ obtain token สำเร็จ
    'UPDATE_LAST_LOGIN': True,

    # Algorithm ที่ใช้เซ็นลายเซ็น (HS256 = symmetric, ใช้ SECRET_KEY เดียวกับ Django)
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': None,  # None = ใช้ settings.SECRET_KEY ของ Django โดยอัตโนมัติ

    # Header ที่ client ต้องส่งมา
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',

    # claim ใน payload ที่เก็บ primary key ของ user
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',

    # class ของ token แต่ละประเภท (จะ override ในขั้นตอนที่ 455)
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),

    # claim สำหรับบอกประเภท token (access/refresh)
    'TOKEN_TYPE_CLAIM': 'token_type',

    # claim สำหรับ JWT ID (jti) — ใช้อ้างอิงตอน blacklist
    'JTI_CLAIM': 'jti',
}
```

### 452.5 เพิ่ม URL สำหรับ Obtain / Refresh / Verify

`simplejwt` มี view สำเร็จรูปให้ 3 ตัวหลัก ที่ตรงกับ endpoint มาตรฐานของระบบ JWT ทั่วโลก:

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),

    # JWT authentication endpoints
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### 452.6 ทดสอบว่าติดตั้งสำเร็จด้วย `curl`

```bash
# ขอ token คู่แรกด้วย username/password (ต้องมี user ในระบบอยู่แล้วจาก Part 011)
curl -X POST http://127.0.0.1:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "somchai", "password": "SecurePass123!"}'
```

ผลลัพธ์ที่คาดหวัง:

```json
{
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTczMDYwNDgwMCwiaWF0IjoxNzMwMDAwMDAwLCJqdGkiOiJhMWIyYzNkNCIsInVzZXJfaWQiOjF9.xxxxx",
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzMwMDAwOTAwLCJpYXQiOjE3MzAwMDAwMDAsImp0aSI6ImU1ZjZhN2I4IiwidXNlcl9pZCI6MX0.yyyyy"
}
```

ถ้าเห็นผลลัพธ์แบบนี้ แสดงว่าติดตั้งและตั้งค่าสำเร็จเรียบร้อย — เราจะเจาะลึกความหมายของ
token ทั้งสองตัวนี้และวิธีใช้งานเต็มรูปแบบในขั้นตอนที่ 453 ถัดไป

### 452.7 ตารางสรุป Settings ที่สำคัญที่สุด

| Setting | ค่า default ของหลักสูตรนี้ | ผลกระทบ |
|---|---|---|
| `ACCESS_TOKEN_LIFETIME` | 15 นาที | ยิ่งสั้น ยิ่งปลอดภัย แต่ client ต้อง refresh บ่อยขึ้น |
| `REFRESH_TOKEN_LIFETIME` | 7 วัน | กำหนดว่า user ต้อง login ใหม่ทุกกี่วัน |
| `ROTATE_REFRESH_TOKENS` | `True` | ป้องกัน refresh token เดิมถูกใช้ซ้ำหลายครั้ง |
| `BLACKLIST_AFTER_ROTATION` | `True` | ต้องติดตั้งแอป `token_blacklist` เพิ่ม (ขั้นตอนที่ 454) |
| `SIGNING_KEY` | `None` (ใช้ `SECRET_KEY`) | ใน production ควรแยก key เฉพาะ (ขั้นตอนที่ 459) |
| `ALGORITHM` | `HS256` | เพียงพอสำหรับระบบ single-service อย่าง `blog` API |

---

## ขั้นตอนที่ 453: Access Token vs Refresh Token flow เต็มรูปแบบ (obtain/refresh/verify)

### 453.1 ทำไมต้องมี Token 2 ตัว แทนที่จะมีตัวเดียว

หลักการออกแบบของ JWT authentication ที่ดีคือ**แยกหน้าที่ของ token ออกเป็น 2 ระดับ**:

| Token | อายุ | ใช้ทำอะไร | ส่งไปที่ไหน |
|---|---|---|---|
| **Access Token** | สั้น (15 นาทีในหลักสูตรนี้) | แนบไปกับทุก request เพื่อเรียก API จริง | ทุก endpoint ที่ต้องการ authentication |
| **Refresh Token** | ยาว (7 วัน) | ใช้ **แลก** access token ใหม่เมื่อตัวเก่าหมดอายุ | เฉพาะ endpoint `/api/token/refresh/` เท่านั้น |

เหตุผลที่แยกกัน: ถ้ามี token เดียวที่อายุยาว (เช่น 7 วัน) แล้วต้องแนบไปกับทุก request
ความเสี่ยงที่ token จะรั่วไหล (ถูกดักจับระหว่างทาง, หลุดจาก log, เก็บไม่ปลอดภัยฝั่ง client)
จะสูงขึ้นมาก และถ้ารั่วไหลจริง คนร้ายจะใช้งานได้นานถึง 7 วันเต็ม แต่ถ้าแยกเป็น 2 ระดับ
Access Token ที่ถูกส่งบ่อยที่สุดจะมีอายุสั้นมาก แม้รั่วไหลก็เสียหายจำกัดแค่ไม่กี่นาที
ส่วน Refresh Token ที่อายุยาวกว่าจะถูกส่งไปแค่ endpoint เดียวเท่านั้น (ความเสี่ยงถูกดักจับ
น้อยกว่ามาก)

### 453.2 แผนภาพวงจรการทำงานแบบเต็ม

```
┌────────┐  1. POST /api/token/ (username+password)   ┌─────────────┐
│ Client │ ──────────────────────────────────────────> │ Django API  │
└────────┘                                              └─────────────┘
     │  2. { access, refresh }  <──────────────────────────────│
     │
     │  3. เก็บ access + refresh ไว้ (ขั้นตอนที่ 459: เก็บอย่างไรให้ปลอดภัย)
     │
     ▼
┌────────┐  4. GET /api/posts/                        ┌─────────────┐
│ Client │  Authorization: Bearer <access>              │ Django API  │
└────────┘ ──────────────────────────────────────────> └─────────────┘
     │  5. 200 OK + ข้อมูล  <───────────────────────────────────│
     │
     │  ... เวลาผ่านไป 15 นาที access token หมดอายุ ...
     │
     ▼
┌────────┐  6. GET /api/posts/ (access หมดอายุแล้ว)   ┌─────────────┐
│ Client │ ──────────────────────────────────────────> │ Django API  │
└────────┘  7. 401 Unauthorized (token_not_valid)  <───└─────────────┘
     │
     ▼
┌────────┐  8. POST /api/token/refresh/ { refresh }   ┌─────────────┐
│ Client │ ──────────────────────────────────────────> │ Django API  │
└────────┘  9. { access ใหม่ (, refresh ใหม่) }  <─────└─────────────┘
     │
     ▼
┌────────┐  10. GET /api/posts/ (access ใหม่)          ┌─────────────┐
│ Client │ ──────────────────────────────────────────> │ Django API  │
└────────┘  11. 200 OK  <──────────────────────────────└─────────────┘
```

### 453.3 Endpoint ที่ 1: `/api/token/` (Obtain Pair)

```bash
curl -X POST http://127.0.0.1:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "somchai", "password": "SecurePass123!"}'
```

```json
{
  "refresh": "eyJhbGci...(refresh token)...xxxxx",
  "access": "eyJhbGci...(access token)...yyyyy"
}
```

เบื้องหลังคือ `TokenObtainPairSerializer` ที่ตรวจสอบ username/password ผ่าน Django's
authentication backend ปกติ (คนละเรื่องกับตัว JWT เอง) แล้วสร้าง token คู่ให้:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework_simplejwt.serializers
class TokenObtainPairSerializer(TokenObtainSerializer):
    @classmethod
    def get_token(cls, user):
        return RefreshToken.for_user(user)

    def validate(self, attrs):
        data = super().validate(attrs)   # ตรวจ username/password ก่อน
        refresh = self.get_token(self.user)
        data['refresh'] = str(refresh)
        data['access'] = str(refresh.access_token)   # สร้าง access token จาก refresh token
        return data
```

สังเกตว่า **access token ถูกสร้างมาจาก refresh token** เสมอ (`refresh.access_token`)
นี่คือเหตุผลที่ endpoint refresh (453.4) สามารถออก access token ใหม่ได้โดยไม่ต้องขอ
username/password ซ้ำ — ตราบใดที่ refresh token ยังใช้ได้อยู่

### 453.4 เรียกใช้ Access Token กับ API จริง

```bash
curl -X GET http://127.0.0.1:8000/api/posts/ \
  -H "Authorization: Bearer eyJhbGci...yyyyy"
```

```json
[
  {
    "id": 1,
    "author": "somchai",
    "title": "สวัสดี Django",
    "slug": "สวัสดี-django",
    "content": "...",
    "is_published": true,
    "created_at": "2026-09-01T10:00:00Z",
    "updated_at": "2026-09-01T10:00:00Z"
  }
]
```

**สังเกต header**: JWT ใช้คำนำหน้า `Bearer` (ไม่ใช่ `Token` แบบ `TokenAuthentication`
ที่เรียนใน Part 045) — นี่เป็นมาตรฐานสากลของ [RFC 6750 (Bearer Token Usage)](https://datatracker.ietf.org/doc/html/rfc6750)
ที่ใช้ร่วมกันทั้ง JWT และ OAuth2 (ขั้นตอนที่ 456)

ถ้า access token หมดอายุแล้ว จะได้ error มาตรฐานนี้:

```json
{
  "detail": "Given token not valid for any token type",
  "code": "token_not_valid",
  "messages": [
    {
      "token_class": "AccessToken",
      "token_type": "access",
      "message": "Token is invalid or expired"
    }
  ]
}
```

พร้อม status code `401 Unauthorized` — client (เช่น React/Vue frontend หรือ mobile app)
ควรตรวจจับ error นี้แล้วเรียก endpoint refresh อัตโนมัติ

### 453.5 Endpoint ที่ 2: `/api/token/refresh/`

```bash
curl -X POST http://127.0.0.1:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "eyJhbGci...(refresh token เดิม)...xxxxx"}'
```

ผลลัพธ์ (เมื่อ `ROTATE_REFRESH_TOKENS = True` ตามที่ตั้งไว้ในขั้นตอนที่ 452.4):

```json
{
  "access": "eyJhbGci...(access token ใหม่)...zzzzz",
  "refresh": "eyJhbGci...(refresh token ใหม่)...wwwww"
}
```

หากตั้ง `ROTATE_REFRESH_TOKENS = False` จะได้แค่ `access` กลับมาตัวเดียว ส่วน refresh
token เดิมยังใช้ซ้ำได้จนกว่าจะหมดอายุตาม `REFRESH_TOKEN_LIFETIME` — หลักสูตรนี้แนะนำให้
เปิด rotation ไว้เสมอด้วยเหตุผลด้านความปลอดภัยที่จะอธิบายเต็มรูปแบบในขั้นตอนที่ 454

### 453.6 Endpoint ที่ 3: `/api/token/verify/`

Endpoint นี้ใช้ตรวจสอบว่า token (จะเป็น access หรือ refresh ก็ได้) ยังใช้งานได้อยู่หรือไม่
โดยไม่ต้องถอดรหัสเอง เหมาะสำหรับกรณีที่ service อื่นต้องการเช็ค token ก่อนดำเนินการ:

```bash
curl -X POST http://127.0.0.1:8000/api/token/verify/ \
  -H "Content-Type: application/json" \
  -d '{"token": "eyJhbGci...yyyyy"}'
```

ถ้า token ใช้ได้: คืน `200 OK` พร้อม body ว่างเปล่า `{}`
ถ้า token หมดอายุหรือไม่ถูกต้อง: คืน `401 Unauthorized` พร้อม error message แบบเดียวกับ
ขั้นตอนที่ 453.4

### 453.7 ตารางสรุป Endpoint ทั้งหมดของ `simplejwt`

| Method | URL | Input | Output | ใช้เมื่อไหร่ |
|---|---|---|---|---|
| `POST` | `/api/token/` | `username`, `password` | `access`, `refresh` | ตอน login ครั้งแรก |
| `POST` | `/api/token/refresh/` | `refresh` | `access` (+ `refresh` ใหม่ถ้าเปิด rotation) | เมื่อ access token หมดอายุ |
| `POST` | `/api/token/verify/` | `token` | `200`/`401` | ตรวจสอบ token โดยไม่ต้อง decode เอง |
| — | ทุก endpoint ของ `blog` API | Header `Authorization: Bearer <access>` | ตามปกติของแต่ละ endpoint | ทุกครั้งที่เรียก API ที่ต้อง authenticate |

### 453.8 การใช้งานฝั่ง Python Client (ตัวอย่างจำลอง)

เพื่อให้เห็นภาพรวมทั้งวงจรในโค้ดเดียว มาดูตัวอย่าง client ฝั่ง Python ที่จัดการ refresh
อัตโนมัติ (แนวคิดเดียวกับที่ frontend framework อย่าง React/Vue ใช้ด้วย `axios interceptor`):

```python
# ตัวอย่างจำลอง client-side logic (ไม่ใช่ส่วนหนึ่งของ Django project)
import requests

BASE_URL = 'http://127.0.0.1:8000'


class JWTClient:
    def __init__(self, username, password):
        response = requests.post(f'{BASE_URL}/api/token/', json={
            'username': username, 'password': password,
        })
        response.raise_for_status()
        tokens = response.json()
        self.access = tokens['access']
        self.refresh = tokens['refresh']

    def _refresh_access_token(self):
        response = requests.post(f'{BASE_URL}/api/token/refresh/', json={
            'refresh': self.refresh,
        })
        response.raise_for_status()
        tokens = response.json()
        self.access = tokens['access']
        if 'refresh' in tokens:          # rotation เปิดอยู่ จะได้ refresh ใหม่มาด้วย
            self.refresh = tokens['refresh']

    def get(self, path):
        headers = {'Authorization': f'Bearer {self.access}'}
        response = requests.get(f'{BASE_URL}{path}', headers=headers)

        if response.status_code == 401:                # access token หมดอายุ
            self._refresh_access_token()
            headers = {'Authorization': f'Bearer {self.access}'}
            response = requests.get(f'{BASE_URL}{path}', headers=headers)

        return response
```

---

## ขั้นตอนที่ 454: Token Blacklisting และ Token Rotation

### 454.1 ปัญหาที่แท้จริง: refresh token ถูกขโมยแล้วใช้ซ้ำได้ตลอด 7 วัน

จากขั้นตอนที่ 451.6 เราทราบแล้วว่าข้อเสียใหญ่ที่สุดของ JWT คือ revoke ก่อนหมดอายุได้ยาก
ลองจินตนาการสถานการณ์นี้: refresh token ของผู้ใช้ถูกขโมยไป (เช่น จาก log ที่ไม่ได้ mask
ข้อมูล หรือจาก XSS attack) คนร้ายสามารถใช้ refresh token นั้น**แลก access token ใหม่ได้
เรื่อย ๆ นานถึง 7 วัน** แม้ผู้ใช้ตัวจริงจะเปลี่ยนรหัสผ่านไปแล้วก็ตาม (เพราะ JWT ไม่ query DB
มาตรวจสอบสถานะบัญชี — นี่คือ trade-off ของ stateless auth)

**Token Rotation + Blacklisting** คือกลไกที่ `simplejwt` ใช้บรรเทาปัญหานี้

### 454.2 กลไก Token Rotation ทำงานอย่างไร

เมื่อ `ROTATE_REFRESH_TOKENS = True` (ตั้งไว้แล้วในขั้นตอนที่ 452.4) ทุกครั้งที่เรียก
`/api/token/refresh/` สำเร็จ ระบบจะ:

1. ออก **refresh token ใหม่** ให้เสมอ (ไม่ใช่แค่ access token ใหม่)
2. ถ้า `BLACKLIST_AFTER_ROTATION = True` ด้วย: นำ refresh token **เก่า** (ตัวที่เพิ่งใช้
   ไป) เข้า blacklist ทันที ทำให้ใช้ซ้ำไม่ได้อีกต่อไป

```
วันที่ 1: Login          → ได้ refresh_v1
วันที่ 2: Refresh         → ใช้ refresh_v1 (ถูก blacklist ทันที) → ได้ refresh_v2
วันที่ 3: Refresh         → ใช้ refresh_v2 (ถูก blacklist ทันที) → ได้ refresh_v3
...
ถ้าคนร้ายขโมย refresh_v1 ไปตั้งแต่วันที่ 1 แล้วพยายามใช้ในวันที่ 3
→ ระบบปฏิเสธทันที เพราะ refresh_v1 อยู่ใน blacklist แล้ว (ถูกใช้ไปแล้วครั้งหนึ่ง)
```

นี่คือ **Reuse Detection**: ถ้า refresh token ตัวเดิมถูกใช้ซ้ำสองครั้ง (ทั้งจากผู้ใช้ตัวจริง
และคนร้ายที่ขโมยไป) ครั้งที่สองจะถูกปฏิเสธเสมอ เพราะครั้งแรกได้ทำให้มัน "ถูกใช้ไปแล้ว"
(blacklisted) ไปแล้ว — วิธีนี้ไม่ได้ป้องกันการขโมยได้ 100% แต่ลด window ของความเสียหายลง
อย่างมาก เพราะคนร้ายต้องแข่งกับผู้ใช้ตัวจริงว่าใครใช้ token ก่อนกัน

### 454.3 ติดตั้งแอป `token_blacklist`

`BLACKLIST_AFTER_ROTATION = True` ต้องอาศัยแอป Django เพิ่มเติมที่เก็บ record ของ token
ที่ถูก blacklist ไว้ในฐานข้อมูล (ใช่ครับ — ส่วนนี้ทำให้ JWT ไม่ 100% stateless อีกต่อไป
แต่เป็น trade-off ที่คุ้มค่าด้านความปลอดภัย):

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'rest_framework_simplejwt.token_blacklist',   # ← เพิ่มบรรทัดนี้
    'blog',
]
```

```bash
python manage.py migrate
```

Migration จะสร้าง 2 ตารางใหม่:

| ตาราง | เก็บอะไร |
|---|---|
| `token_blacklist_outstandingtoken` | บันทึกทุก refresh token ที่เคยออกไป (ใช้ตรวจสอบและยกเลิกได้ทีหลัง) |
| `token_blacklist_blacklistedtoken` | บันทึก refresh token ที่ถูกเพิกถอนแล้ว (ห้ามใช้ซ้ำ) |

### 454.4 สร้าง Logout Endpoint ที่ Blacklist Token ทันที

ค่า default ของ `simplejwt` จะ blacklist token ให้อัตโนมัติแค่ตอน **rotation** เท่านั้น
แต่กรณี "ผู้ใช้กด Logout" เราต้องการ blacklist refresh token **ทันที** โดยไม่ต้องรอ
rotation — สร้าง view เฉพาะสำหรับสิ่งนี้:

```python
# blog/api_views.py
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError


class LogoutAPIView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        try:
            refresh_token = request.data['refresh']
            token = RefreshToken(refresh_token)
            token.blacklist()   # ← ใส่ refresh token เข้า blacklist ทันที
            return Response(status=status.HTTP_205_RESET_CONTENT)
        except KeyError:
            return Response(
                {'detail': 'ต้องส่ง refresh token มาด้วย'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        except TokenError:
            return Response(
                {'detail': 'refresh token ไม่ถูกต้องหรือถูกเพิกถอนไปแล้ว'},
                status=status.HTTP_400_BAD_REQUEST,
            )
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
    path('logout/', api_views.LogoutAPIView.as_view(), name='logout'),
]
```

ทดสอบ:

```bash
curl -X POST http://127.0.0.1:8000/api/logout/ \
  -H "Authorization: Bearer <access token ที่ยังไม่หมดอายุ>" \
  -H "Content-Type: application/json" \
  -d '{"refresh": "<refresh token ที่ต้องการ logout>"}'
# ผลลัพธ์: 205 Reset Content (ไม่มี body)

# ลองใช้ refresh token เดิมอีกครั้งหลัง logout
curl -X POST http://127.0.0.1:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "<refresh token เดิมที่เพิ่ง logout ไป>"}'
```

```json
{
  "detail": "Token is blacklisted",
  "code": "token_not_valid"
}
```

**ข้อควรระวังสำคัญ**: การ blacklist refresh token ไม่ได้ทำให้ **access token** ที่ออก
ไปแล้วก่อนหน้าใช้งานไม่ได้ทันที เพราะ access token ไม่ผ่านการตรวจ blacklist (ตรวจแค่
ลายเซ็นและ `exp`) — access token ที่ยังไม่หมดอายุจะยังใช้เรียก API ได้ต่อไปจนกว่าจะครบ
15 นาทีตาม `ACCESS_TOKEN_LIFETIME` นี่คือเหตุผลสำคัญที่ควรตั้ง access token ให้มีอายุสั้น
ที่สุดเท่าที่ UX จะรับได้

### 454.5 คำสั่งจัดการ Blacklist ผ่าน Management Command

`simplejwt` มี management command สำเร็จรูปสำหรับลบ token ที่หมดอายุแล้วออกจากฐานข้อมูล
(ป้องกันตารางบวมในระยะยาว) เหมาะสำหรับตั้งเป็น cron job หรือ Celery periodic task
(จะเรียนเต็มรูปแบบใน Phase Background Tasks):

```bash
python manage.py flushexpiredtokens
```

### 454.6 ตารางสรุปขั้นตอนที่ 454

| กลไก | Setting ที่เกี่ยวข้อง | ผลลัพธ์ |
|---|---|---|
| Token Rotation | `ROTATE_REFRESH_TOKENS = True` | ทุกครั้งที่ refresh จะได้ refresh token ใหม่เสมอ |
| Auto-blacklist หลัง rotation | `BLACKLIST_AFTER_ROTATION = True` | refresh token เก่าใช้ซ้ำไม่ได้ทันทีหลัง rotate |
| Manual blacklist (logout) | `token.blacklist()` ใน view เอง | เพิกถอน refresh token ได้ทันทีโดยไม่ต้องรอ rotation |
| ทำความสะอาดฐานข้อมูล | `python manage.py flushexpiredtokens` | ลบ record ที่หมดอายุแล้วออกจากตาราง |

---

## ขั้นตอนที่ 455: การใส่ Custom Claims เข้าไปใน JWT payload

### 455.1 ทำไมต้องมี Custom Claims

ค่า default ของ `simplejwt` ใส่แค่ `user_id` ลงใน payload เท่านั้น แต่ในระบบจริงมักต้องการ
ให้ frontend รู้ข้อมูลเพิ่มเติมของผู้ใช้ทันทีโดยไม่ต้องยิง request แยกไปถาม `/api/me/`
เช่น **บทบาท (role)** สำหรับปรับ UI ที่แสดงผล หรือ **สถานะสมาชิกพรีเมียม (is_premium)**
สำหรับปลดล็อกฟีเจอร์บางอย่างในหน้าเว็บทันที

### 455.2 เตรียม Field ที่จะใช้เป็น Claim

สมมติว่า `blog` มี model `Profile` ที่ผูกกับ `User` แบบ one-to-one (ต่อยอดจาก Part 015-016):

```python
# blog/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    class Role(models.TextChoices):
        READER = 'reader', 'ผู้อ่าน'
        AUTHOR = 'author', 'นักเขียน'
        EDITOR = 'editor', 'บรรณาธิการ'

    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    role = models.CharField(max_length=10, choices=Role.choices, default=Role.READER)
    is_premium = models.BooleanField(default=False)

    def __str__(self):
        return f'{self.user.username} ({self.role})'
```

### 455.3 Override `TokenObtainPairSerializer` เพื่อเพิ่ม Claims

จุดที่ต้องแก้คือ `get_token()` classmethod ซึ่งเป็นตัวสร้าง `RefreshToken` (ทวนจากขั้นตอน
ที่ 453.3 ที่เห็นซอร์สโค้ดต้นฉบับมาแล้ว):

```python
# blog/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer


class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # เพิ่ม custom claims ลงใน payload ของทั้ง access และ refresh token
        token['username'] = user.username
        token['role'] = getattr(user.profile, 'role', 'reader')
        token['is_premium'] = getattr(user.profile, 'is_premium', False)

        return token
```

**หมายเหตุสำคัญ**: `token[key] = value` ใน `simplejwt` คือการเรียก `__setitem__` ของ
`Token` object ซึ่งภายในจะเก็บลง `self.payload[key] = value` — claims เหล่านี้จะติดไป
กับ **ทั้ง refresh และ access token** เพราะ `access_token` ถูกสร้างมาจาก `refresh` object
ตัวเดียวกัน (ทวนจากขั้นตอนที่ 453.3)

### 455.4 สร้าง View ที่ใช้ Serializer ที่ Override แล้ว

```python
# blog/api_views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import CustomTokenObtainPairSerializer


class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

```python
# config/urls.py
from blog.api_views import CustomTokenObtainPairView
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView

urlpatterns = [
    # ...
    path('api/token/', CustomTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### 455.5 ทดสอบผลลัพธ์

```bash
curl -X POST http://127.0.0.1:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "somchai", "password": "SecurePass123!"}'
```

ถอดรหัส access token ที่ได้กลับมาด้วยวิธีเดียวกับขั้นตอนที่ 451.3:

```python
>>> decode_jwt_part(payload_part)
{
    'token_type': 'access',
    'exp': 1730000900,
    'iat': 1730000000,
    'jti': 'e5f6a7b8',
    'user_id': 1,
    'username': 'somchai',
    'role': 'author',
    'is_premium': True,
}
```

Frontend สามารถถอด payload นี้ (ด้วย library เช่น `jwt-decode` ฝั่ง JavaScript) เพื่อ
แสดงผล UI ตาม `role`/`is_premium` ได้ทันที โดยไม่ต้องยิง request เพิ่มเลย

### 455.6 กับดักคลาสสิก: Claims ไม่อัปเดตทันทีเมื่อข้อมูลใน DB เปลี่ยน

**ข้อควรระวังที่สำคัญที่สุดของขั้นตอนนี้**: เพราะ JWT เป็น stateless (ทวนจากขั้นตอนที่
451.2) claims ที่ฝังไว้ตอน login **จะไม่อัปเดตอัตโนมัติ** แม้ข้อมูลในฐานข้อมูลจะเปลี่ยน
ไปแล้วก็ตาม เช่น ถ้า admin เปลี่ยน `role` ของผู้ใช้จาก `author` เป็น `editor` ผ่าน Django
Admin ระหว่างที่ access token เดิมยังไม่หมดอายุ — token นั้นจะยังคงมี claim `role: author`
เก่าอยู่จนกว่าจะหมดอายุ (หรือจนกว่า client จะ refresh token ใหม่)

ทางแก้เชิงปฏิบัติมี 2 แนวทาง:

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| ใช้ `ACCESS_TOKEN_LIFETIME` สั้น ๆ (เช่น 5-15 นาที) | claims จะ "สด" ขึ้นบ่อย เพราะต้อง refresh บ่อย | ต้อง refresh บ่อยขึ้น เพิ่ม request |
| อย่าใช้ claims สำหรับข้อมูลที่เปลี่ยนบ่อย/สำคัญต่อ authorization จริง ๆ ใช้แค่แสดงผล UI เท่านั้น แล้วเช็คสิทธิ์จริงจาก DB ใน permission class เสมอ | ปลอดภัยกว่า authorization ที่สำคัญไม่พึ่ง claims ที่อาจเก่า | ต้องเขียน custom permission เพิ่ม |

**หลักการสำคัญที่สุด**: ห้ามใช้ custom claims เป็นแหล่งความจริงเพียงแหล่งเดียวสำหรับ
การตัดสินใจด้าน **authorization** (เช่น อนุญาตให้ลบโพสต์หรือไม่) ควรใช้แค่เพื่อ**แสดงผล
UI** เท่านั้น ส่วนการตรวจสอบสิทธิ์จริงต้องเช็คจากฐานข้อมูลผ่าน permission class เสมอ
(ตามที่เรียนใน Part 045)

---

## ขั้นตอนที่ 456: ทำ API ของตัวเองเป็น OAuth2 Provider ด้วย `django-oauth-toolkit`

### 456.1 JWT/Token Auth vs OAuth2: ปัญหาคนละแบบ

จนถึงตอนนี้ทุกวิธีที่เรียนมา (Session, Token, JWT) ล้วนออกแบบมาสำหรับสถานการณ์ที่
**เจ้าของ API เขียน client เอง** (frontend ของตัวเอง, mobile app ของตัวเอง) แต่ถ้าคุณ
ต้องการให้ **third-party application** (ที่คุณไม่ได้เขียนเอง ไม่รู้จักผู้พัฒนาโดยตรง)
เชื่อมต่อเข้ามาขอสิทธิ์เข้าถึงข้อมูลของผู้ใช้แทนผู้ใช้ (เช่น "แอปจัดตารางเวลาขอสิทธิ์
อ่าน/เขียนบทความในบัญชี blog ของคุณ") จะเกิดปัญหาใหม่:

- คุณไม่อยากให้ third-party app รู้ password ของผู้ใช้เด็ดขาด
- ผู้ใช้ต้องเห็นหน้าจอ "อนุญาต/ปฏิเสธ" ก่อนให้สิทธิ์เสมอ (consent screen)
- ต้องกำหนดได้ว่า third-party app นี้เข้าถึง **ขอบเขต (scope)** ไหนได้บ้าง เช่น อ่านได้
  อย่างเดียว หรืออ่าน-เขียนได้ด้วย
- ผู้ใช้ต้องเพิกถอนสิทธิ์ที่ให้ third-party app ไปได้ทุกเมื่อ โดยไม่กระทบ session ของตัวเอง

นี่คือปัญหาที่ **OAuth2** (มาตรฐาน [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749))
ถูกออกแบบมาแก้โดยเฉพาะ — ไม่ใช่แค่ "อีกวิธีหนึ่งในการยืนยันตัวตน" แต่เป็น
**framework สำหรับมอบสิทธิ์ (authorization) แบบมีขอบเขตให้บุคคลที่สาม**

### 456.2 ติดตั้ง `django-oauth-toolkit`

```bash
pip install django-oauth-toolkit
pip freeze | grep -i oauth >> requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt.token_blacklist',
    'oauth2_provider',   # ← เพิ่มบรรทัดนี้
    'blog',
]

AUTHENTICATION_BACKENDS = [
    'oauth2_provider.backends.OAuth2Backend',
    'django.contrib.auth.backends.ModelBackend',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}
```

```bash
python manage.py migrate
```

Migration จะสร้างตารางสำหรับเก็บ `Application` (third-party app ที่ลงทะเบียนไว้),
`AccessToken`, `RefreshToken`, และ `Grant` ของ OAuth2 โดยเฉพาะ (แยกจากตารางของ
`simplejwt` โดยสิ้นเชิง)

### 456.3 เพิ่ม URL ของ `oauth2_provider`

```python
# config/urls.py
urlpatterns = [
    # ...
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
```

URL prefix `/o/` จะเปิด endpoint มาตรฐานของ OAuth2 ให้ทันที เช่น `/o/authorize/`,
`/o/token/`, `/o/revoke_token/`, `/o/applications/` (หน้าจัดการ third-party app ของ
ผู้ใช้เอง)

### 456.4 ลงทะเบียน Third-Party Application

เข้า `http://127.0.0.1:8000/o/applications/register/` (ต้อง login ก่อน) หรือผ่าน Django
Admin (`/admin/oauth2_provider/application/add/`) กรอกข้อมูล:

| ฟิลด์ | ตัวอย่างค่า | ความหมาย |
|---|---|---|
| Name | `Blog Scheduler App` | ชื่อ third-party app ที่จะแสดงในหน้า consent |
| Client type | `Confidential` | app ที่รันบน server (เก็บ secret ได้ปลอดภัย) เทียบกับ `Public` (SPA/mobile ที่เก็บ secret ไม่ได้) |
| Authorization grant type | `Authorization code` | รูปแบบการขอสิทธิ์ (อธิบายในขั้นตอนที่ 456.6) |
| Redirect URIs | `https://scheduler.example.com/callback/` | URL ที่ OAuth2 server จะ redirect กลับไปพร้อม authorization code |

ระบบจะสร้าง `Client ID` และ `Client Secret` ให้อัตโนมัติ — สอง ค่านี้คือสิ่งที่ third-party
app ต้องใช้ในการขอ token

### 456.5 กำหนด Scopes

```python
# config/settings.py
OAUTH2_PROVIDER = {
    'SCOPES': {
        'read': 'อ่านข้อมูลบทความได้',
        'write': 'สร้าง/แก้ไข/ลบบทความได้',
    },
    'ACCESS_TOKEN_EXPIRE_SECONDS': 3600,        # 1 ชั่วโมง
    'REFRESH_TOKEN_EXPIRE_SECONDS': 1209600,    # 14 วัน
}
```

### 456.6 Authorization Code Grant Flow แบบเต็ม (Grant Type ที่ปลอดภัยที่สุด)

```
┌──────────┐  1. Redirect ผู้ใช้ไปหน้า /o/authorize/?client_id=...&scope=read  ┌─────────────┐
│Third-party│ ───────────────────────────────────────────────────────────────> │ Django API  │
│   App    │                                                                    │ (ผู้ใช้เห็น  │
└──────────┘                                                                    │ หน้า consent)│
     │                                                                          └─────────────┘
     │  2. ผู้ใช้กด "อนุญาต" → redirect กลับไปที่ redirect_uri พร้อม ?code=xxxxx
     │  <───────────────────────────────────────────────────────────────────────────
     ▼
┌──────────┐  3. POST /o/token/ (code, client_id, client_secret, grant_type)  ┌─────────────┐
│Third-party│ ───────────────────────────────────────────────────────────────> │ Django API  │
│   App    │  4. { access_token, refresh_token, scope }  <───────────────────  └─────────────┘
└──────────┘
     │
     ▼
┌──────────┐  5. GET /api/posts/ (Authorization: Bearer <access_token>)       ┌─────────────┐
│Third-party│ ───────────────────────────────────────────────────────────────> │ Django API  │
│   App    │  6. 200 OK (เฉพาะข้อมูลที่ scope=read อนุญาต)  <──────────────────└─────────────┘
└──────────┘
```

ทดสอบขั้นตอนที่ 3 ด้วย `curl` (สมมติว่ามี authorization `code` มาแล้วจากขั้นตอนที่ 2):

```bash
curl -X POST http://127.0.0.1:8000/o/token/ \
  -d "grant_type=authorization_code" \
  -d "code=<authorization_code_ที่ได้มา>" \
  -d "redirect_uri=https://scheduler.example.com/callback/" \
  -d "client_id=<client_id>" \
  -d "client_secret=<client_secret>"
```

```json
{
  "access_token": "AbCdEf123456...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "read",
  "refresh_token": "GhIjKl789012..."
}
```

**สังเกต**: token ของ OAuth2 toolkit **ไม่ใช่ JWT** โดยค่า default (เป็น random opaque
string ธรรมดา ตรวจสอบด้วยการ query DB — คล้าย `TokenAuthentication` ในขั้นตอนที่ 451.1)
`django-oauth-toolkit` รองรับการออก JWT ผ่าน JWT-bearer extension ได้เช่นกัน แต่หลักสูตร
นี้จะเน้นรูปแบบ opaque token มาตรฐานเพื่อความชัดเจน

### 456.7 ตารางเปรียบเทียบ Grant Types หลักของ OAuth2

| Grant Type | เหมาะกับ | Client เห็น password ผู้ใช้ไหม |
|---|---|---|
| **Authorization Code** | Web app / mobile app ที่มี backend ของตัวเอง (แนะนำที่สุด) | ไม่เห็นเลย ผู้ใช้กรอกที่หน้า login ของ Django เอง |
| **Client Credentials** | Machine-to-machine ที่ไม่มีผู้ใช้เกี่ยวข้อง (server คุยกับ server) | ไม่เกี่ยวกับผู้ใช้เลย ใช้แค่ client_id/secret |
| **Password** (Resource Owner Password Credentials) | เฉพาะกรณี first-party app ที่ไว้ใจได้ 100% เท่านั้น (ไม่แนะนำสำหรับ third-party) | เห็น (client ต้องกรอก username/password ส่งตรงมาที่ token endpoint) |
| **Refresh Token** | ขอ access token ใหม่จาก refresh token (คล้ายขั้นตอนที่ 453) | ไม่เกี่ยว |

**คำแนะนำระดับมืออาชีพ**: สำหรับ third-party app ที่แท้จริง ให้ใช้ **Authorization Code**
เท่านั้น หลีกเลี่ยง **Password Grant** เด็ดขาด เพราะขัดกับหลักการพื้นฐานของ OAuth2 ที่ว่า
"third-party ไม่ควรเห็น password ของผู้ใช้เลย"

### 456.8 ปกป้อง View ด้วย Scope

```python
# blog/api_views.py
from oauth2_provider.contrib.rest_framework import TokenHasScope


class PostListAPIView(APIView):
    permission_classes = [TokenHasScope]
    required_scopes = ['read']   # ใช้เฉพาะ GET เท่านั้นในตัวอย่างนี้

    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
```

ถ้า access token ที่ third-party app ส่งมามี scope แค่ `read` แต่พยายามเรียก endpoint
ที่ต้องการ `write` (เช่น สร้างโพสต์ใหม่) จะได้ `403 Forbidden` ทันที แม้ token จะยังไม่
หมดอายุก็ตาม — นี่คือหัวใจของ **การมอบสิทธิ์แบบมีขอบเขต** ที่ Token/JWT ธรรมดาไม่มี
ให้โดยกำเนิด

### 456.9 ตารางสรุปขั้นตอนที่ 456

| ประเด็น | สรุป |
|---|---|
| ใช้เมื่อไหร่ | ให้ third-party application เชื่อมต่อ ไม่ใช่ client ของตัวเอง |
| หัวใจสำคัญ | ผู้ใช้เห็นหน้า consent และเลือก scope ที่จะอนุญาตได้ |
| Grant Type ที่แนะนำ | Authorization Code (ไม่แนะนำ Password Grant สำหรับ third-party) |
| Token ที่ได้ | Opaque token (ตรวจสอบผ่าน DB) ไม่ใช่ JWT โดย default |
| จัดการสิทธิ์ | ผ่าน `scope` + permission class เช่น `TokenHasScope` |

---

## ขั้นตอนที่ 457: API Key Authentication ด้วย `djangorestframework-api-key` สำหรับการเชื่อมต่อแบบ machine-to-machine

### 457.1 เมื่อไหร่ที่ JWT/OAuth2 "เกินความจำเป็น"

ทั้ง JWT และ OAuth2 ถูกออกแบบมาโดยมี **"ผู้ใช้ (user)"** เป็นศูนย์กลางเสมอ — ต้องมี
username/password, ต้องมี consent screen, ต้องมี concept ของ "user คนไหนกำลังเรียก
API" แต่ในโลกจริงมีสถานการณ์ที่**ไม่มีผู้ใช้เข้ามาเกี่ยวข้องเลย** เช่น:

- Cron job ภายในองค์กรที่ดึงข้อมูลสถิติจาก `blog` API ทุกเที่ยงคืน
- ระบบพาร์ทเนอร์ (partner server) ที่ sync ข้อมูลบทความเข้าระบบของตัวเองอัตโนมัติ
- Monitoring/health-check service ที่ต้องเรียก endpoint พิเศษเป็นระยะ

สถานการณ์เหล่านี้เรียกว่า **Machine-to-Machine (M2M)** — ไม่มี "คนล็อกอิน" แต่มี
"ระบบหนึ่งเรียกอีกระบบหนึ่ง" การสร้าง JWT/OAuth2 flow ที่ซับซ้อนสำหรับสิ่งนี้เกินความ
จำเป็นมาก **API Key** คือทางออกที่เรียบง่ายกว่ามาก: กุญแจสตริงยาว ๆ หนึ่งดอก ที่ออกให้
ระบบภายนอกใช้แนบมาในทุก request

### 457.2 ติดตั้ง `djangorestframework-api-key`

```bash
pip install djangorestframework-api-key
pip freeze | grep -i api-key >> requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_api_key',   # ← เพิ่มบรรทัดนี้
    'oauth2_provider',
    'blog',
]
```

```bash
python manage.py migrate
```

### 457.3 สร้าง API Key ผ่าน Django Admin

`rest_framework_api_key` มี `APIKey` model พร้อม admin สำเร็จรูปให้ทันที เข้า
`/admin/rest_framework_api_key/apikey/add/` แล้วกรอกชื่อ (เช่น `Partner Sync Service`)
กด Save — ระบบจะแสดง **key เต็ม ๆ ให้เห็นแค่ครั้งเดียว** (หลังจากนั้นฐานข้อมูลจะเก็บแค่
hash ของ key ไว้ ไม่สามารถดูค่าจริงย้อนหลังได้ — เหมือนหลักการเก็บรหัสผ่านของ Django เอง)

```
ตัวอย่าง key ที่ได้: qFqQqXYZ.aBcDeFgHiJkLmNoPqRsTuVwXyZ123456
                     └─prefix─┘└──────────── secret ────────────┘
```

**สำคัญมาก**: คัดลอก key นี้เก็บไว้ทันที เพราะเมื่อปิดหน้าจอไปแล้วจะไม่สามารถดูค่าเต็มได้
อีก (ต้อง revoke แล้วสร้างใหม่เท่านั้น)

### 457.4 สร้าง Custom Permission Class

```python
# blog/permissions.py
from rest_framework_api_key.permissions import BaseHasAPIKey
from .models import PartnerAPIKey


class HasPartnerAPIKey(BaseHasAPIKey):
    model = PartnerAPIKey   # ระบุ model ของ API key ที่จะใช้ตรวจสอบ
```

หากต้องการ scope key แยกตามพาร์ทเนอร์แต่ละราย (เช่น รู้ว่า key ไหนเป็นของพาร์ทเนอร์ใด)
สร้าง model สืบทอดจาก `AbstractAPIKey`:

```python
# blog/models.py
from rest_framework_api_key.models import AbstractAPIKey


class PartnerAPIKey(AbstractAPIKey):
    partner_name = models.CharField(max_length=100)
    can_write = models.BooleanField(default=False)

    class Meta(AbstractAPIKey.Meta):
        verbose_name = 'Partner API Key'
        verbose_name_plural = 'Partner API Keys'
```

```python
# blog/admin.py
from rest_framework_api_key.admin import APIKeyModelAdmin
from django.contrib import admin
from .models import PartnerAPIKey


@admin.register(PartnerAPIKey)
class PartnerAPIKeyAdmin(APIKeyModelAdmin):
    list_display = [*APIKeyModelAdmin.list_display, 'partner_name', 'can_write']
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

### 457.5 ใช้งานใน View

```python
# blog/api_views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from .permissions import HasPartnerAPIKey
from .models import Post
from .serializers import PostSerializer


class PartnerPostSyncAPIView(APIView):
    """
    Endpoint สำหรับพาร์ทเนอร์ภายนอก sync ข้อมูลบทความ
    ไม่มี user login เกี่ยวข้องเลย ตรวจสอบสิทธิ์ด้วย API Key เท่านั้น
    """
    authentication_classes = []          # ไม่ต้องใช้ authentication class ใด ๆ
    permission_classes = [HasPartnerAPIKey]

    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
```

```python
# blog/api_urls.py
urlpatterns = [
    # ...
    path('partner/posts/', api_views.PartnerPostSyncAPIView.as_view(), name='partner-post-sync'),
]
```

**สังเกต**: `authentication_classes = []` เพราะ `djangorestframework-api-key` ทำงานผ่าน
**permission class** ไม่ใช่ authentication class — มันไม่ได้ผูก request กับ `request.user`
คนใดคนหนึ่ง (เพราะไม่มี "user" ในความหมายนั้นตั้งแต่แรก) เพียงแค่ตรวจว่า key ที่แนบมา
ถูกต้องและยังไม่ถูก revoke เท่านั้น

### 457.6 ทดสอบด้วย `curl`

```bash
curl -X GET http://127.0.0.1:8000/api/partner/posts/ \
  -H "Authorization: Api-Key qFqQqXYZ.aBcDeFgHiJkLmNoPqRsTuVwXyZ123456"
```

ถ้า key ถูกต้อง: `200 OK` พร้อมข้อมูล
ถ้า key ผิดหรือถูก revoke แล้ว:

```json
{
  "detail": "Invalid API key."
}
```

พร้อม status `403 Forbidden`

### 457.7 Revoke API Key

เข้า Django Admin แล้วติ๊ก `Revoked` ที่ record ของ key นั้น — มีผลทันที ไม่ต้อง restart
server เพราะการตรวจสอบเป็นการ query DB สด ๆ ทุกครั้ง (คล้ายกับ `TokenAuthentication`
ในขั้นตอนที่ 451.1 คือ stateful แต่ที่นี่คือข้อดี เพราะ revoke ได้ทันที ต่างจากปัญหาของ
JWT ในขั้นตอนที่ 451.6)

### 457.8 ตารางสรุปขั้นตอนที่ 457

| ประเด็น | สรุป |
|---|---|
| ใช้เมื่อไหร่ | Machine-to-machine ที่ไม่มี "ผู้ใช้" เกี่ยวข้อง |
| กลไก | Header `Authorization: Api-Key <key>` ตรวจสอบผ่าน permission class |
| Revoke | ทันที ผ่าน Django Admin (ติ๊ก revoked) |
| ข้อจำกัด | ไม่มี concept ของ user/scope ละเอียดแบบ OAuth2 — เหมาะกับสิทธิ์แบบง่าย |
| ความปลอดภัย | เก็บ hash ใน DB เท่านั้น เห็นค่าเต็มแค่ตอนสร้างครั้งเดียว |

---

## ขั้นตอนที่ 458: ตารางเปรียบเทียบวิธี Authentication ทั้งหมด — เมื่อไหร่ควรใช้อะไร

### 458.1 ทบทวนทุกวิธีที่เรียนมาในหลักสูตรนี้

ตอนนี้ `blog` API ของคุณรู้จัก authentication scheme ถึง 5 แบบแล้ว มาสรุปเปรียบเทียบ
แบบละเอียดเพื่อให้ตัดสินใจถูกต้องในโปรเจกต์จริงได้:

| คุณสมบัติ | Session Auth | Token Auth | JWT | OAuth2 | API Key |
|---|---|---|---|---|---|
| Stateful/Stateless | Stateful | Stateful | Stateless (ไม่นับ blacklist) | Stateful | Stateful |
| Query DB ทุก request | ✅ ใช่ | ✅ ใช่ | ❌ ไม่ (เว้นแต่ตรวจ blacklist) | ✅ ใช่ | ✅ ใช่ |
| มี "ผู้ใช้" เกี่ยวข้อง | ✅ ใช่ | ✅ ใช่ | ✅ ใช่ | ✅ ใช่ | ❌ ไม่จำเป็น |
| เหมาะกับ Client ประเภทไหน | Browser (Django Template) | Mobile/SPA แบบง่าย | Mobile/SPA, Microservices | Third-party app | Machine-to-machine |
| ต้องมีหน้า consent | ❌ ไม่มี | ❌ ไม่มี | ❌ ไม่มี | ✅ มี | ❌ ไม่มี |
| กำหนด scope ละเอียดได้ | ❌ ไม่ได้ | ❌ ไม่ได้ | ⚠️ ทำเองผ่าน custom claims | ✅ ได้ในตัว | ⚠️ ทำเองผ่าน custom field |
| Revoke ก่อนหมดอายุ | ง่าย (`request.session.flush()`) | ง่าย (ลบ record) | ยาก (ต้องพึ่ง blacklist) | ง่าย (`/o/revoke_token/`) | ง่าย (revoke ผ่าน admin) |
| Scale ข้ามหลาย server | ต้องใช้ shared session store | ต้องใช้ shared DB | ทำได้ทันที (ยกเว้นตรวจ blacklist) | ต้องใช้ shared DB | ต้องใช้ shared DB |
| รองรับ CSRF protection | ✅ ต้องมี (มากับ cookie) | ❌ ไม่เกี่ยว | ❌ ไม่เกี่ยว | ❌ ไม่เกี่ยว | ❌ ไม่เกี่ยว |
| ความซับซ้อนในการ implement | ต่ำ (Django มีให้ในตัว) | ต่ำ | ปานกลาง | สูง | ต่ำ |
| มาตรฐานสากล | ไม่มี (เฉพาะ Django) | ไม่มี (เฉพาะ DRF) | ✅ RFC 7519 | ✅ RFC 6749 | ไม่มีมาตรฐานตายตัว |

### 458.2 Flowchart การตัดสินใจ

```
เริ่มต้น: API ของคุณจะถูกเรียกโดยใคร?
│
├─ เบราว์เซอร์ที่ render Template ของ Django เอง (ไม่ใช่ SPA)
│    → ใช้ SessionAuthentication (Part 045)
│
├─ Mobile App หรือ SPA (React/Vue) ที่เป็นของทีมคุณเอง
│    ├─ โปรเจกต์เล็ก/prototype ไม่ซับซ้อน
│    │    → TokenAuthentication ก็เพียงพอ (Part 045)
│    └─ โปรเจกต์ production จริง ต้องการ scale/สิทธิ์ที่มีอายุสั้น
│         → JWT (ขั้นตอนที่ 452-455)
│
├─ ระบบ Microservices หลาย service ต้องตรวจสอบ token ร่วมกัน
│    → JWT ด้วย RS256 (asymmetric key)
│
├─ Third-party application ที่ไม่ใช่ของทีมคุณ ต้องขอสิทธิ์จากผู้ใช้
│    → OAuth2 (ขั้นตอนที่ 456)
│
└─ ระบบอื่น/Partner server/Cron job ที่ไม่มี "ผู้ใช้" เกี่ยวข้องเลย
     → API Key (ขั้นตอนที่ 457)
```

### 458.3 ตัวอย่างระบบจริงที่ผสมหลายวิธีพร้อมกัน (แบบที่ `blog` API ของเรากำลังเป็น)

ระบบระดับ production จริงมักไม่ได้ใช้วิธีเดียวตลอดทั้งระบบ แต่ผสมกันตามแต่ละ client:

| Endpoint | Client | Authentication ที่ใช้ |
|---|---|---|
| `/api/posts/` (เรียกจาก Django Admin ผ่านเบราว์เซอร์) | เบราว์เซอร์ | `SessionAuthentication` |
| `/api/posts/` (เรียกจาก Mobile App ของบริษัทเอง) | Mobile App | `JWTAuthentication` |
| `/o/authorize/` (third-party scheduler app ขอสิทธิ์) | Third-party App | OAuth2 |
| `/api/partner/posts/` (พาร์ทเนอร์ sync ข้อมูล) | Partner Server | API Key |

`REST_FRAMEWORK['DEFAULT_AUTHENTICATION_CLASSES']` ที่ตั้งไว้ตั้งแต่ขั้นตอนที่ 456.2
รองรับสถานการณ์แบบนี้ได้อยู่แล้วในตัว เพราะ DRF ไล่ตรวจ authentication class ทีละตัว
จนกว่าจะเจอตัวที่เข้าใจ header ที่ client ส่งมา

---

## ขั้นตอนที่ 459: Security Best Practices สำหรับ API Authentication

### 459.1 บังคับใช้ HTTPS เสมอในทุก Environment ที่ไม่ใช่ Local

Token/JWT ที่ส่งผ่าน HTTP ธรรมดา (ไม่เข้ารหัส) สามารถถูกดักจับได้ง่ายด้วยเทคนิค
Man-in-the-Middle — ไม่ว่าจะ implement authentication scheme ดีแค่ไหน ถ้า transport
layer ไม่ปลอดภัย ทุกอย่างที่เรียนมาใน Part นี้ก็ไร้ความหมาย

```python
# config/settings.py — ตั้งค่าเฉพาะ production (แยกไฟล์ settings ตาม Part 089)
SECURE_SSL_REDIRECT = True          # redirect ทุก HTTP request ไป HTTPS อัตโนมัติ
SESSION_COOKIE_SECURE = True        # cookie ส่งได้เฉพาะผ่าน HTTPS เท่านั้น
CSRF_COOKIE_SECURE = True           # เช่นเดียวกันสำหรับ CSRF cookie
SECURE_HSTS_SECONDS = 31536000      # บอกเบราว์เซอร์ให้จำว่า "domain นี้ใช้ HTTPS เท่านั้น" 1 ปี
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')  # เมื่ออยู่หลัง reverse proxy
```

### 459.2 การเก็บ Token ฝั่ง Client อย่างปลอดภัย

นี่คือจุดที่นักพัฒนามือใหม่พลาดบ่อยที่สุด — การออกแบบ backend ดีแค่ไหนก็ไร้ประโยชน์ถ้า
frontend เก็บ token แบบเสี่ยงต่อการถูกขโมย:

| ที่เก็บ | เสี่ยงต่อ XSS | เสี่ยงต่อ CSRF | คำแนะนำ |
|---|---|---|---|
| `localStorage` | ⚠️ เสี่ยงสูง — JavaScript อ่านได้ตรง ๆ ถ้ามี XSS หลุดเข้ามาแม้แต่จุดเดียว ขโมย token ได้ทันที | ✅ ปลอดภัย (ไม่ได้แนบอัตโนมัติ) | หลีกเลี่ยงสำหรับ refresh token; ถ้าใช้กับ access token ต้องมั่นใจว่าไม่มีช่องโหว่ XSS เลย |
| `sessionStorage` | ⚠️ เสี่ยงสูงเหมือนกัน แต่หายเมื่อปิดแท็บ | ✅ ปลอดภัย | ดีกว่า `localStorage` เล็กน้อยสำหรับ access token อายุสั้น |
| `httpOnly` Cookie | ✅ ปลอดภัย — JavaScript **อ่านไม่ได้เลย** | ⚠️ เสี่ยง ต้องมี CSRF protection คู่กัน | **แนะนำที่สุดสำหรับ refresh token** ในสถาปัตยกรรมที่ backend ควบคุม cookie ได้ |
| In-memory (JavaScript variable, ไม่ persist) | ✅ ปลอดภัยที่สุดจาก XSS แบบอ่านไฟล์/storage | ✅ ปลอดภัย | ดีที่สุดสำหรับ access token แต่ต้อง re-login ทุกครั้งที่ปิดแท็บ (UX แย่กว่า) |

**แนวทางที่แนะนำสำหรับสถาปัตยกรรมระดับ production** (ผสมข้อดีของหลายวิธี):

```
1. Refresh Token  → เก็บใน httpOnly + Secure + SameSite=Strict cookie
                     (backend เป็นคน set cookie ให้ตอน login, JavaScript แตะต้องไม่ได้เลย)

2. Access Token   → เก็บใน memory (JavaScript variable ธรรมดา ไม่ persist)
                     (อายุสั้นมาก แม้ถูกขโมยผ่าน XSS ก็เสียหายจำกัดในกรอบเวลาสั้น ๆ)
```

ตัวอย่างการปรับ `TokenObtainPairView` ให้ set refresh token เป็น httpOnly cookie แทนที่
จะส่งกลับใน response body:

```python
# blog/api_views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework.response import Response


class CookieTokenObtainPairView(TokenObtainPairView):
    def post(self, request, *args, **kwargs):
        response = super().post(request, *args, **kwargs)

        if response.status_code == 200:
            refresh_token = response.data.pop('refresh')   # ไม่ส่ง refresh กลับใน body
            response.set_cookie(
                key='refresh_token',
                value=refresh_token,
                httponly=True,      # JavaScript อ่านไม่ได้
                secure=True,        # ส่งได้เฉพาะผ่าน HTTPS
                samesite='Strict',  # ป้องกัน CSRF ระดับหนึ่ง
                max_age=7 * 24 * 60 * 60,   # ตรงกับ REFRESH_TOKEN_LIFETIME
            )
        return response
```

### 459.3 ตั้งค่า `ACCESS_TOKEN_LIFETIME` ให้สั้นที่สุดเท่าที่ UX จะรับได้

ทวนจากขั้นตอนที่ 452.4 — 15 นาทีคือค่าที่สมดุลระหว่างความปลอดภัยและ UX สำหรับ `blog`
API แต่ระบบที่มีข้อมูลอ่อนไหวสูงกว่า (เช่น ระบบธนาคาร) มักตั้งไว้แค่ 5 นาทีหรือน้อยกว่า
ยิ่งสั้น ยิ่งลด window ของความเสียหายเมื่อ token รั่วไหล

### 459.4 แยก `SIGNING_KEY` ออกจาก `SECRET_KEY` หลักของ Django

ค่า default ของ `simplejwt` ใช้ `settings.SECRET_KEY` ตัวเดียวกับที่ Django ใช้เซ็น
session, CSRF token และอื่น ๆ ทั้งหมด ในระบบ production ที่ต้องการความปลอดภัยสูง
ควรแยก key เฉพาะสำหรับ JWT ออกมา เพื่อว่าถ้า key หนึ่งรั่วไหล จะไม่กระทบระบบอื่นทั้งหมด:

```python
# config/settings.py
import os

SIMPLE_JWT = {
    # ...
    'SIGNING_KEY': os.environ['JWT_SIGNING_KEY'],   # แยกจาก SECRET_KEY โดยเด็ดขาด
}
```

```bash
# สร้าง key ใหม่แบบสุ่มปลอดภัย
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

เก็บค่านี้ไว้ใน environment variable หรือ secret manager เท่านั้น (ตามหลักที่เรียนมาตั้งแต่
Part 030 เรื่อง `.env` และ `python-decouple`/`django-environ`) **ห้าม commit ลง Git
เด็ดขาด**

### 459.5 Rate Limiting บน Endpoint Login/Token Obtain

Endpoint `/api/token/` คือเป้าหมายอันดับหนึ่งของการโจมตีแบบ brute-force (ลองรหัสผ่าน
ซ้ำ ๆ) ควรจำกัดจำนวนครั้งที่เรียกได้ต่อ IP/ต่อ user (จะเจาะลึกกลไก Throttling เต็มรูปแบบ
ใน Part 048 แต่แสดงตัวอย่างเบื้องต้นไว้ก่อน):

```python
# blog/api_views.py
from rest_framework.throttling import AnonRateThrottle
from rest_framework_simplejwt.views import TokenObtainPairView


class LoginRateThrottle(AnonRateThrottle):
    scope = 'login'


class ThrottledTokenObtainPairView(TokenObtainPairView):
    throttle_classes = [LoginRateThrottle]
```

```python
# config/settings.py
REST_FRAMEWORK = {
    # ...
    'DEFAULT_THROTTLE_RATES': {
        'login': '5/min',   # จำกัดแค่ 5 ครั้งต่อนาทีต่อ IP
    },
}
```

### 459.6 ตรวจสอบ Refresh Token Reuse และแจ้งเตือน

ทวนจากขั้นตอนที่ 454.2 เรื่อง Reuse Detection — ในระบบระดับ production ควร log และ
แจ้งเตือนเมื่อพบว่า refresh token ที่ถูก blacklist แล้วถูกพยายามใช้ซ้ำ เพราะนั่นคือ
สัญญาณบ่งชี้ที่ชัดเจนว่า token น่าจะรั่วไหลไปอยู่ในมือคนร้าย:

```python
# blog/api_views.py
import logging
from rest_framework_simplejwt.views import TokenRefreshView
from rest_framework_simplejwt.exceptions import TokenError

logger = logging.getLogger('security')


class MonitoredTokenRefreshView(TokenRefreshView):
    def post(self, request, *args, **kwargs):
        try:
            return super().post(request, *args, **kwargs)
        except TokenError as exc:
            if 'blacklisted' in str(exc).lower():
                logger.warning(
                    'ตรวจพบการพยายามใช้ refresh token ที่ถูก blacklist แล้วซ้ำ — '
                    'อาจเป็นสัญญาณว่า token รั่วไหล IP=%s',
                    request.META.get('REMOTE_ADDR'),
                )
            raise
```

### 459.7 Checklist สรุป Security Best Practices

| หัวข้อ | ทำแล้วหรือยัง |
|---|---|
| บังคับ HTTPS ทุก environment ที่ไม่ใช่ local (`SECURE_SSL_REDIRECT`) | ☐ |
| Refresh token เก็บใน `httpOnly` + `Secure` + `SameSite` cookie | ☐ |
| Access token เก็บใน memory เท่านั้น ไม่ persist ลง storage | ☐ |
| `ACCESS_TOKEN_LIFETIME` สั้นที่สุดเท่าที่ UX รับได้ | ☐ |
| เปิด `ROTATE_REFRESH_TOKENS` + `BLACKLIST_AFTER_ROTATION` | ☐ |
| แยก `SIGNING_KEY` ออกจาก `SECRET_KEY` และเก็บใน secret manager | ☐ |
| Rate limit endpoint `/api/token/` ป้องกัน brute-force | ☐ |
| Log และแจ้งเตือนเมื่อพบ refresh token reuse | ☐ |
| ไม่ใส่ข้อมูลอ่อนไหว/ลับลงใน JWT payload | ☐ |
| Revoke API Key/OAuth2 token ได้ทันทีเมื่อสงสัยว่ารั่วไหล | ☐ |

---

## ขั้นตอนที่ 460: สรุปและแบบฝึกหัด — ย้าย `blog` API ทั้งหมดไปใช้ JWT

### 460.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจโครงสร้าง JWT (`header.payload.signature`) และเหตุผลเชิงสถาปัตยกรรมที่ทำให้
  stateless authentication เหมาะกับ API มากกว่า Session/Token แบบเดิม
- ✅ ติดตั้งและตั้งค่า `djangorestframework-simplejwt` ครบทุก setting ที่สำคัญ
- ✅ เข้าใจวงจร Access Token / Refresh Token แบบเต็มรูปแบบ พร้อม endpoint obtain/
  refresh/verify
- ✅ ป้องกันปัญหา refresh token ถูกใช้ซ้ำด้วย Token Rotation + Blacklisting
- ✅ เพิ่ม Custom Claims (`role`, `is_premium`) ลงใน JWT payload พร้อมรู้ข้อจำกัดเรื่อง
  ข้อมูลไม่อัปเดตทันที
- ✅ ทำ API ของตัวเองเป็น OAuth2 Provider ด้วย `django-oauth-toolkit` สำหรับ third-party
  application พร้อมเข้าใจ Authorization Code Grant Flow
- ✅ ใช้ API Key Authentication สำหรับการเชื่อมต่อแบบ machine-to-machine ที่ไม่มีผู้ใช้
  เกี่ยวข้อง
- ✅ เปรียบเทียบวิธี authentication ทั้ง 5 แบบ และรู้ว่าเมื่อไหร่ควรเลือกแบบไหน
- ✅ ปฏิบัติตาม Security Best Practices ระดับ production สำหรับระบบ authentication ของ API

### 460.2 ลงมือจริง: แทนที่ Token Authentication เดิมด้วย JWT แบบเต็มรูปแบบ

ถึงเวลาสรุปทุกอย่างที่เรียนมาให้เป็นการเปลี่ยนแปลงจริงบน `blog` API ทำตามลำดับนี้:

**ขั้นที่ 1 — อัปเดต `settings.py` ให้ JWT เป็นค่า default และถอด `TokenAuthentication` ออก**

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework_simplejwt.token_blacklist',
    'oauth2_provider',
    'rest_framework_api_key',
    'blog',
    # หมายเหตุ: ลบ 'rest_framework.authtoken' ออกได้ถ้าไม่มี client เดิมพึ่งพาอยู่แล้ว
    # แนะนำให้ตรวจสอบ log การใช้งานจริงก่อน 1-2 sprint ก่อนลบทิ้งถาวร
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework.authentication.SessionAuthentication',
        # ลบ 'rest_framework.authentication.TokenAuthentication' ออกแล้ว
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'login': '5/min',
    },
}

from datetime import timedelta
import os

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': os.environ.get('JWT_SIGNING_KEY', None),
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

**ขั้นที่ 2 — รวม URL ทั้งหมดของระบบ authentication เข้าด้วยกัน**

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView
from blog.api_views import (
    CustomTokenObtainPairView,
    LogoutAPIView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),

    # JWT authentication (ขั้นตอนที่ 452-455)
    path('api/token/', CustomTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),

    # OAuth2 สำหรับ third-party app (ขั้นตอนที่ 456)
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
```

**ขั้นที่ 3 — รวม `blog/api_views.py` และ `blog/api_urls.py` ฉบับสมบูรณ์**

```python
# blog/api_views.py
from django.shortcuts import get_object_or_404
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_api_key.permissions import BaseHasAPIKey

from .models import Post, PartnerAPIKey
from .serializers import CustomTokenObtainPairSerializer, PostSerializer


class CustomTokenObtainPairView(TokenObtainPairView):
    """Obtain JWT พร้อม custom claims (role, is_premium) — ขั้นตอนที่ 455"""
    serializer_class = CustomTokenObtainPairSerializer


class LogoutAPIView(APIView):
    """Blacklist refresh token ทันทีเมื่อผู้ใช้ logout — ขั้นตอนที่ 454"""
    permission_classes = [IsAuthenticated]

    def post(self, request):
        try:
            token = RefreshToken(request.data['refresh'])
            token.blacklist()
            return Response(status=status.HTTP_205_RESET_CONTENT)
        except KeyError:
            return Response(
                {'detail': 'ต้องส่ง refresh token มาด้วย'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        except TokenError:
            return Response(
                {'detail': 'refresh token ไม่ถูกต้องหรือถูกเพิกถอนไปแล้ว'},
                status=status.HTTP_400_BAD_REQUEST,
            )


class HasPartnerAPIKey(BaseHasAPIKey):
    model = PartnerAPIKey


class PostListAPIView(APIView):
    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetailAPIView(APIView):
    def get_object(self, slug):
        return get_object_or_404(Post, slug=slug)

    def get(self, request, slug):
        serializer = PostSerializer(self.get_object(slug))
        return Response(serializer.data)

    def put(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def patch(self, request, slug):
        post = self.get_object(slug)
        serializer = PostSerializer(post, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, slug):
        self.get_object(slug).delete()
        return Response(status=status.HTTP_204_NO_CONTENT)


class PartnerPostSyncAPIView(APIView):
    """Machine-to-machine endpoint สำหรับพาร์ทเนอร์ภายนอก — ขั้นตอนที่ 457"""
    authentication_classes = []
    permission_classes = [HasPartnerAPIKey]

    def get(self, request):
        posts = Post.objects.filter(is_published=True)
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
    path('logout/', api_views.LogoutAPIView.as_view(), name='logout'),
    path('partner/posts/', api_views.PartnerPostSyncAPIView.as_view(), name='partner-post-sync'),
]
```

**ขั้นที่ 4 — ทดสอบวงจรทั้งหมดแบบ end-to-end**

```bash
# 1. Login ได้ access + refresh พร้อม custom claims
curl -s -X POST http://127.0.0.1:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "somchai", "password": "SecurePass123!"}' | python -m json.tool

# 2. เรียก API ด้วย access token
curl -s http://127.0.0.1:8000/api/posts/ \
  -H "Authorization: Bearer <access>" | python -m json.tool

# 3. Refresh เมื่อ access หมดอายุ
curl -s -X POST http://127.0.0.1:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "<refresh>"}' | python -m json.tool

# 4. Logout (blacklist refresh token ทันที)
curl -s -X POST http://127.0.0.1:8000/api/logout/ \
  -H "Authorization: Bearer <access>" \
  -H "Content-Type: application/json" \
  -d '{"refresh": "<refresh>"}'
```

### 460.3 Checklist ก่อนไป Part ถัดไป

- [ ] เข้าใจโครงสร้าง `header.payload.signature` ของ JWT และถอดรหัส payload ด้วยมือได้
- [ ] ติดตั้งและตั้งค่า `djangorestframework-simplejwt` สำเร็จ พร้อม `SIMPLE_JWT` settings
- [ ] เรียก `/api/token/`, `/api/token/refresh/`, `/api/token/verify/` ได้ครบทั้ง 3 endpoint
- [ ] เปิด `ROTATE_REFRESH_TOKENS` + `BLACKLIST_AFTER_ROTATION` และติดตั้งแอป
      `token_blacklist` สำเร็จ
- [ ] สร้าง Logout endpoint ที่ blacklist refresh token ได้จริง
- [ ] เพิ่ม custom claims (`role`, `is_premium`) ลงใน JWT payload สำเร็จ
- [ ] ติดตั้ง `django-oauth-toolkit` และเข้าใจ Authorization Code Grant Flow
- [ ] ติดตั้ง `djangorestframework-api-key` และสร้าง machine-to-machine endpoint สำเร็จ
- [ ] อธิบายได้ว่าเมื่อไหร่ควรใช้ Session, Token, JWT, OAuth2, หรือ API Key
- [ ] ย้าย `blog` API ทั้งหมดไปใช้ JWT เป็น authentication หลักสำเร็จ

### 460.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียนสคริปต์ Python (ไม่ใช้ library `PyJWT` หรือ `simplejwt` — ใช้แค่
`base64`, `hashlib`/`hmac`, และ `json` เท่านั้น) ที่สร้าง JWT ขึ้นมาเองตั้งแต่ต้นจาก
payload `{"user_id": 1, "username": "test"}` ด้วย algorithm `HS256` แล้วตรวจสอบว่า
signature ที่คำนวณได้ตรงกับที่ `simplejwt` สร้างจริงหรือไม่ (ใช้ secret key เดียวกัน)
เพื่อพิสูจน์ความเข้าใจกลไกจากขั้นตอนที่ 451.4 อย่างถ่องแท้

**แบบฝึกหัดที่ 2**: เพิ่ม endpoint ใหม่ `/api/posts/<slug>/like/` ที่อนุญาตเฉพาะผู้ใช้ที่
`role` เป็น `reader` ขึ้นไป (ทุก role) ให้กด like ได้ แต่เฉพาะ `role` เป็น `editor`
เท่านั้นที่ pin โพสต์ขึ้นบนสุดได้ (`is_pinned = True`) โดยต้องตรวจสอบ `role` **จาก
ฐานข้อมูลจริงผ่าน permission class** ไม่ใช่จาก custom claims ใน JWT (ทวนเหตุผลจาก
ขั้นตอนที่ 455.6)

**แบบฝึกหัดที่ 3**: ตั้งค่า `django-oauth-toolkit` ให้รองรับ **Client Credentials Grant**
(นอกเหนือจาก Authorization Code ที่ทำในขั้นตอนที่ 456) สำหรับ use case ที่ partner
server ต้องการ sync ข้อมูลแบบ machine-to-machine ผ่าน OAuth2 แทนที่จะใช้ API Key แล้ว
เขียนอธิบายเปรียบเทียบว่ากรณีไหนควรใช้ OAuth2 Client Credentials กับกรณีไหนควรใช้
API Key ธรรมดา (ขั้นตอนที่ 457) แทน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ปรับ `CustomTokenObtainPairView` ให้ set refresh token
เป็น httpOnly cookie ตามตัวอย่างในขั้นตอนที่ 459.2 ทั้งหมด แล้วปรับ
`TokenRefreshView` ให้อ่าน refresh token จาก cookie แทนที่จะรับจาก request body
(ต้อง override `get_serializer` หรือเขียน view ใหม่ที่ดึงค่าจาก `request.COOKIES`
ก่อนส่งต่อให้ serializer เดิมประมวลผล) ทดสอบว่า JavaScript ฝั่ง client อ่านค่า
refresh token จาก cookie ไม่ได้จริง (เปิด browser DevTools Console แล้วลองรัน
`document.cookie` ดูผลลัพธ์)

### 460.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ JWT แทน Session Authentication สำหรับเว็บที่ยัง render Template ของ Django
เองอยู่หรือไม่?**
A: ไม่ควร ถ้าเว็บของคุณยัง render HTML ผ่าน Django Template (ไม่ใช่ SPA แยกส่วน)
`SessionAuthentication` ยังคงเป็นตัวเลือกที่ดีที่สุด เพราะ Django จัดการเรื่อง cookie,
CSRF protection ให้ครบอยู่แล้ว การเปลี่ยนไปใช้ JWT ในกรณีนี้เพิ่มความซับซ้อนโดยไม่ได้
ประโยชน์อะไรเพิ่มเลย — JWT เหมาะกับสถานการณ์ที่ frontend/backend แยกจากกันอย่างชัดเจน
(SPA, Mobile App) เท่านั้น

**Q: ทำไมไม่ตั้ง `ACCESS_TOKEN_LIFETIME` ให้ยาวเท่ากับ `REFRESH_TOKEN_LIFETIME` ไปเลย
จะได้ไม่ต้อง refresh บ่อย ๆ?**
A: เพราะจะเสียหลักการสำคัญที่สุดของการแยก token 2 ระดับ (ขั้นตอนที่ 453.1) — access
token ถูกส่งไปกับทุก request จึงมีโอกาสรั่วไหลสูงกว่า refresh token ที่ส่งไปแค่ endpoint
เดียว ถ้าตั้งอายุยาวเท่ากัน ความเสี่ยงเมื่อรั่วไหลจะเท่ากับใช้ token เดี่ยวอายุยาวตัวเดียว
ซึ่งเสียจุดประสงค์ของการออกแบบทั้งหมดไป

**Q: จำเป็นต้องใช้ทั้ง JWT, OAuth2, และ API Key พร้อมกันในทุกโปรเจกต์หรือไม่?**
A: ไม่จำเป็นเลย ส่วนใหญ่โปรเจกต์ระดับกลาง-เล็กใช้แค่ JWT อย่างเดียวก็เพียงพอสำหรับ
client ของตัวเอง (mobile app, SPA) OAuth2 Provider ควรทำเมื่อมีความต้องการให้
third-party เชื่อมต่อจริง ๆ เท่านั้น (เพราะซับซ้อนและดูแลยากกว่ามาก) ส่วน API Key
เหมาะกับกรณี machine-to-machine ที่เจาะจงจริง ๆ — เลือกใช้ตามความจำเป็นจริงของระบบ
ไม่ใช่ใช้ทุกอย่างเพราะ "เผื่อไว้"

**Q: ถ้าไม่ได้ติดตั้งแอป `token_blacklist` จะเกิดอะไรขึ้นถ้าตั้ง
`BLACKLIST_AFTER_ROTATION = True` ไว้?**
A: จะเกิด error ทันทีตอนเรียก `/api/token/refresh/` เพราะ `simplejwt` จะพยายามเขียน
record ลงตาราง `token_blacklist_blacklistedtoken` ที่ไม่มีอยู่จริง (Django จะแจ้ง
`django.db.utils.ProgrammingError: relation does not exist` หรือคล้ายกัน) ต้องเพิ่ม
`'rest_framework_simplejwt.token_blacklist'` ใน `INSTALLED_APPS` และรัน `migrate`
ตามขั้นตอนที่ 454.3 ก่อนเปิดใช้ setting นี้เสมอ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 047: Filtering, Searching, Pagination ใน DRF** จะพาไปเรียนรู้วิธีทำให้ endpoint
`/api/posts/` ของเรารองรับการกรองข้อมูล (เช่น `?is_published=true`), การค้นหา (เช่น
`?search=django`), การเรียงลำดับ (`?ordering=-created_at`), และการแบ่งหน้า (pagination)
ด้วย `django-filter` และ DRF's built-in pagination classes เพื่อให้ API ของเราพร้อม
รองรับข้อมูลจำนวนมากในระดับ production จริง โดยจะใช้ endpoint ที่มี JWT authentication
ครบถ้วนจาก Part นี้เป็นฐานต่อยอดทันที

เตรียม `blog` API ที่ใช้งาน JWT ได้ครบวงจรของคุณไว้ให้พร้อม แล้วไปต่อกันเลย!
