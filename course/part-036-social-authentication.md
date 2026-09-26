# Part 036: Social Authentication (OAuth, django-allauth)

> **ขั้นตอนที่ 351-360 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เจาะลึกสิ่งที่ Part 031 ขั้นตอนที่ 309 เกริ่นไว้แบบผิวเผินให้
> ครบทุกมิติ คุณจะเข้าใจ **OAuth2 Authorization Code Flow** ตั้งแต่ทฤษฎีจนถึงการทำงาน
> จริงเบื้องหลังทุกครั้งที่กดปุ่ม "Sign in with Google" ติดตั้งและตั้งค่า
> **django-allauth** อย่างถูกต้องครบวงจร เชื่อมต่อ **Google Login** และ **GitHub
> Login** ด้วย OAuth credentials จริงที่ใช้งานได้ทันที จัดการ **Account Linking**
> เมื่อผู้ใช้มีทั้งบัญชี email/password และ social account ปรับแต่ง Template ของ
> allauth ให้ตรงกับดีไซน์เว็บของคุณ ใช้ Signal ของ allauth สร้าง Profile อัตโนมัติ
> เมื่อมีผู้ใช้สมัครใหม่ เข้าใจข้อควรระวังด้านความปลอดภัยของ OAuth (redirect URI,
> `state` parameter) และรู้จักทางเลือกอื่นในระบบนิเวศ Python เมื่อจบ Part นี้ คุณจะ
> implement ระบบ "Sign in with Google" ที่ทำงานได้จริงให้กับโปรเจกต์ blog ของคุณ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 351: ภาพรวมแนวคิด OAuth2 — Authorization Code Flow อธิบายทีละขั้นตอนพร้อม diagram
- ขั้นตอนที่ 352: ติดตั้งและตั้งค่า django-allauth เบื้องต้น (INSTALLED_APPS, MIDDLEWARE, SITE_ID, urls.py)
- ขั้นตอนที่ 353: เชื่อมต่อ Google Login จริง — สร้าง OAuth credentials ใน Google Cloud Console, ตั้งค่า `SocialApp` ใน Django Admin
- ขั้นตอนที่ 354: เชื่อมต่อ GitHub Login จริง — สร้าง OAuth App ใน GitHub Developer Settings
- ขั้นตอนที่ 355: การเชื่อม Social Account เข้ากับบัญชีที่มีอยู่แล้ว (Account Linking)
- ขั้นตอนที่ 356: Customize Template ของ allauth ให้ตรง design ของเว็บ
- ขั้นตอนที่ 357: จัดการ Signal ของ allauth เพื่อสร้าง Profile อัตโนมัติ
- ขั้นตอนที่ 358: ข้อควรระวังด้านความปลอดภัยของ OAuth
- ขั้นตอนที่ 359: ทางเลือกอื่น — python-social-auth/social-auth-app-django เปรียบเทียบกับ django-allauth
- ขั้นตอนที่ 360: สรุปและแบบฝึกหัด — implement Google login จริงสำหรับ blog project

---

## ขั้นตอนที่ 351: ภาพรวมแนวคิด OAuth2 — Authorization Code Flow อธิบายทีละขั้นตอนพร้อม diagram

### 351.1 ปัญหาที่ OAuth2 แก้ไข

ก่อน OAuth2 จะแพร่หลาย ถ้าเว็บไซต์ A ต้องการเข้าถึงข้อมูลของผู้ใช้ที่อยู่บนเว็บไซต์ B
(เช่น อ่านรายชื่อ contact จาก Google) วิธีเดียวที่ทำได้ในยุคก่อนคือให้ผู้ใช้ **บอก
username/password ของ Google ให้เว็บไซต์ A โดยตรง** ซึ่งเป็นแนวทางที่อันตรายมาก:

- เว็บไซต์ A รู้รหัสผ่าน Google ของผู้ใช้ทั้งหมด แล้วนำไปใช้ทำอะไรก็ได้ ไม่จำกัดสิทธิ์
- ผู้ใช้ไม่มีทางยกเลิกสิทธิ์เฉพาะเว็บไซต์ A โดยไม่เปลี่ยนรหัสผ่านทั้งระบบ
- ถ้าเว็บไซต์ A ถูกแฮ็ก รหัสผ่าน Google ของผู้ใช้ทุกคนก็รั่วไหลไปด้วย

**OAuth2** (Open Authorization 2.0) คือมาตรฐานเปิดที่แก้ปัญหานี้ด้วยแนวคิดหลักคือ
**"ให้สิทธิ์แบบจำกัดขอบเขต โดยไม่ต้องเปิดเผยรหัสผ่าน"** ผู้ใช้ login ที่เว็บของ
Google เอง (ไม่ใช่ที่เว็บไซต์ A) แล้ว Google จะออก **token** ที่มีอายุจำกัดและ
ขอบเขตสิทธิ์ (scope) ที่ชัดเจนให้เว็บไซต์ A ใช้แทน — เว็บไซต์ A ไม่เคยเห็นรหัสผ่าน
Google ของผู้ใช้เลยแม้แต่ตัวอักษรเดียว

**"Sign in with Google/GitHub"** ที่เราจะสร้างใน Part นี้ คือการนำ OAuth2 มาใช้เพื่อ
**ยืนยันตัวตน (authentication)** แทนการสร้างรหัสผ่านใหม่บนเว็บของเรา ไม่ใช่เพื่อขอ
สิทธิ์เข้าถึงข้อมูลอื่น (ซึ่งเป็นการใช้งานแบบ **authorization** ดั้งเดิมของ OAuth2)
— ในทางเทคนิค ส่วนที่ทำให้ "authenticate ด้วย OAuth2" เป็นมาตรฐานที่ปลอดภัยเรียกว่า
**OpenID Connect (OIDC)** ซึ่งสร้างต่อยอดบน OAuth2 อีกชั้นหนึ่ง (อธิบายในขั้นตอนที่
351.6)

### 351.2 ผู้เล่นทั้ง 4 ฝ่ายในระบบ OAuth2

| บทบาท (Role) | คือใครในบริบทของเรา | หน้าที่ |
|---|---|---|
| **Resource Owner** | ผู้ใช้ (คุณ) | เจ้าของบัญชี Google/GitHub ที่ต้องอนุญาตหรือปฏิเสธคำขอ |
| **Client** | เว็บ Django ของเรา (blog project) | แอปที่ต้องการยืนยันตัวตนผู้ใช้ผ่าน Google/GitHub |
| **Authorization Server** | Google/GitHub (ฝั่งที่ออก token) | ตรวจสอบตัวตนผู้ใช้ ขอความยินยอม แล้วออก authorization code/token |
| **Resource Server** | Google/GitHub API (เช่น `userinfo` endpoint) | เซิร์ฟเวอร์ที่เก็บข้อมูลผู้ใช้จริง (ชื่อ, อีเมล, รูปโปรไฟล์) ที่ client เข้าถึงได้ด้วย token |

ในกรณีของ Google และ GitHub, **Authorization Server** และ **Resource Server** มักเป็น
ระบบเดียวกันหรืออยู่ภายใต้องค์กรเดียวกัน (ต่างจากระบบ OAuth2 บางระบบที่แยกกันจริง ๆ)
แต่ในทางแนวคิด ทั้งสองบทบาทนี้ยังคงแยกจากกันเสมอ

### 351.3 Authorization Code Flow: แผนภาพทีละขั้นตอน

**Authorization Code Flow** คือรูปแบบ OAuth2 ที่ปลอดภัยที่สุดและเป็นมาตรฐานที่
django-allauth ใช้เมื่อเชื่อมต่อกับ Google/GitHub ขั้นตอนทั้งหมดมีดังนี้:

```
┌──────────┐                                          ┌────────────────┐
│  Browser │                                          │  Django (Client)│
│ (ผู้ใช้)  │                                          │  ของเรา         │
└────┬─────┘                                          └────────┬───────┘
     │                                                          │
     │  1. คลิก "Sign in with Google"                          │
     │ ────────────────────────────────────────────────────────>
     │                                                          │
     │  2. Django สร้าง `state` (ค่าสุ่มป้องกัน CSRF) เก็บไว้ใน  │
     │     session แล้ว redirect ไปหา Google พร้อม              │
     │     client_id, redirect_uri, scope, state                │
     │ <────────────────────────────────────────────────────────
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Google (Authorization Server)                   │
│  3. ผู้ใช้ login เข้า Google (ถ้ายังไม่เคย login)                    │
│  4. Google แสดงหน้า "อนุญาตให้ myblog.com เข้าถึงอีเมลและโปรไฟล์      │
│     ของคุณหรือไม่?" (consent screen)                                 │
│  5. ผู้ใช้กด "อนุญาต (Allow)"                                        │
└──────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    │ 6. Google redirect กลับไปที่
                                    │    redirect_uri พร้อม `code` และ `state`
                                    │    เดิม (ไม่ใช่ access token โดยตรง!)
                                    ▼
┌──────────┐                                          ┌────────────────┐
│  Browser │  7. Browser ถูก redirect ไปที่           │  Django (Client)│
│          │     /accounts/google/login/callback/     │                │
│          │     ?code=xxxx&state=yyyy                │                │
│          │ ────────────────────────────────────────>│                │
└──────────┘                                          │                │
                                                        │ 8. Django ตรวจสอบ
                                                        │    `state` ตรงกับที่
                                                        │    เก็บไว้ใน session
                                                        │    หรือไม่ (ป้องกัน CSRF)
                                                        │                │
                                                        │ 9. Django ส่ง `code`
                                                        │    ไปแลก access token
                                                        │    โดยตรงกับ Google
                                                        │    (server-to-server,
                                                        │    แนบ client_secret ด้วย)
                                                        └────────┬───────┘
                                                                 │
                                                                 ▼
                                                    ┌────────────────────┐
                                                    │  Google Token       │
                                                    │  Endpoint            │
                                                    │  10. ตรวจสอบ code +  │
                                                    │  client_secret ถูกต้อง│
                                                    │  → คืน access_token, │
                                                    │  id_token, (refresh) │
                                                    └────────┬───────────┘
                                                                 │
                                                                 ▼
                                                        ┌────────────────┐
                                                        │  Django (Client)│
                                                        │ 11. ใช้ access_  │
                                                        │ token เรียก      │
                                                        │ Google userinfo  │
                                                        │ endpoint เพื่อดึง │
                                                        │ email, name,     │
                                                        │ picture          │
                                                        │                 │
                                                        │ 12. สร้าง/หา User│
                                                        │ ในฐานข้อมูลของเรา │
                                                        │ แล้ว login()      │
                                                        │ (สร้าง session   │
                                                        │ ของ Django เอง)  │
                                                        └────────┬───────┘
                                                                 │
     ┌──────────┐                                                │
     │  Browser │  13. Redirect กลับไปหน้าแรกของเว็บ พร้อม        │
     │          │      session cookie ของ Django ที่ login แล้ว   │
     │          │ <─────────────────────────────────────────────┘
     └──────────┘
```

### 351.4 ตารางสรุปแต่ละขั้นตอนพร้อมชื่อเรียกทางเทคนิค

| ขั้น | เกิดอะไรขึ้น | ชื่อเรียกทางเทคนิค |
|---|---|---|
| 1-2 | ผู้ใช้กดปุ่ม, Django redirect ไป Google พร้อมพารามิเตอร์ | Authorization Request |
| 3-5 | ผู้ใช้ login และกด "อนุญาต" ที่ฝั่ง Google | User Consent |
| 6-7 | Google redirect กลับมาที่เว็บเราพร้อม `code` | Authorization Response / Redirect URI Callback |
| 8 | Django ตรวจสอบ `state` ว่าตรงกับที่ส่งไปตอนแรก | CSRF Protection ผ่าน `state` (ขั้นตอนที่ 358.2) |
| 9-10 | Django แลก `code` เป็น token โดยตรงกับ Google (ไม่ผ่าน browser) | Token Exchange / Access Token Request |
| 11 | Django ใช้ access token เรียก API เพื่อดึงข้อมูลผู้ใช้ | Resource Request |
| 12 | Django สร้าง/จับคู่ user ในฐานข้อมูลตัวเอง แล้วสร้าง session | Local Account Provisioning + Login |
| 13 | ผู้ใช้ login สำเร็จ กลับสู่เว็บของเรา | Redirect / Final Response |

### 351.5 ทำไมต้องมี "code" ก่อนแล้วค่อยแลกเป็น "token" (ไม่ส่ง token ตรง ๆ)

จุดที่มือใหม่มักสงสัย: ทำไม Google ไม่ส่ง access token กลับมาที่ `redirect_uri`
ตรง ๆ ในขั้นตอนที่ 6-7 เลย ต้องมี `code` มาคั่นกลางทำไม?

คำตอบคือเรื่องความปลอดภัย: การ redirect ผ่าน browser (ขั้นตอนที่ 6-7) ถือว่า
**ไม่ปลอดภัยเท่าการสื่อสาร server-to-server** เพราะ URL ที่ปรากฏใน browser
อาจถูกบันทึกไว้ใน browser history, server access log, หรือ referrer header ของ
หน้าเว็บถัดไป หาก access token (ซึ่งใช้เรียก API ได้ทันที) หลุดไปอยู่ในที่เหล่านี้
ก็จะเป็นอันตรายทันที

ในทางกลับกัน `code` เป็นค่าที่ **ใช้ได้เพียงครั้งเดียว มีอายุสั้นมาก (มักไม่เกิน
1 นาที)** และต้องใช้คู่กับ `client_secret` (ที่รู้เฉพาะฝั่งเซิร์ฟเวอร์ของ Django
เท่านั้น ไม่เคยส่งไปที่ browser) ในการแลกเป็น token จริง ๆ อีกที ผ่านการเชื่อมต่อ
server-to-server ที่ browser ไม่มีทางเห็น แม้ `code` จะรั่วไหลออกไปจาก browser
history ผู้โจมตีก็ไม่สามารถใช้มันได้เพราะไม่มี `client_secret`

### 351.6 OAuth2 ล้วน ๆ vs OpenID Connect (OIDC)

| ประเด็น | OAuth2 ล้วน ๆ | OpenID Connect (OIDC) |
|---|---|---|
| จุดประสงค์หลัก | **Authorization** — ให้สิทธิ์เข้าถึงทรัพยากร | **Authentication** — ยืนยันตัวตนผู้ใช้ |
| สิ่งที่ได้จาก token endpoint | `access_token` (และอาจมี `refresh_token`) | `access_token` + **`id_token`** (JWT ที่มีข้อมูลตัวตนผู้ใช้ เซ็นด้วย private key ของ provider) |
| ใครอ่านข้อมูลผู้ใช้ | Client ต้องเรียก API แยกต่างหาก (เช่น `/userinfo`) | อ่านได้ทันทีจาก `id_token` (เป็น JWT ที่ decode ได้เลย) โดยไม่ต้องเรียก API เพิ่ม |
| ตัวอย่างการใช้งาน | แอปที่ต้องการ "โพสต์ข้อความแทนผู้ใช้" (ต้องการ scope เข้าถึงทรัพยากร) | "Sign in with Google" ทั่วไปที่ต้องการแค่รู้ว่า "คุณคือใคร" |
| Scope ที่เกี่ยวข้อง | กำหนดเองตาม API (เช่น `read:user`, `repo`) | มี scope มาตรฐาน `openid`, `profile`, `email` |

**Google** รองรับทั้ง OAuth2 และ OIDC เต็มรูปแบบ (มี `id_token` เสมอเมื่อขอ scope
`openid`) ส่วน **GitHub ใช้ OAuth2 ล้วน ๆ ไม่มี OIDC** ต้องเรียก endpoint
`https://api.github.com/user` เพื่อดึงข้อมูลโปรไฟล์เสมอ — นี่คือความแตกต่างที่
django-allauth จัดการให้อัตโนมัติผ่านโมดูล provider ที่แตกต่างกัน (จะเห็นชัดเจนใน
ขั้นตอนที่ 353-354)

### 351.7 คำศัพท์ OAuth2 ที่ต้องจำให้แม่น (Glossary)

| คำศัพท์ | ความหมาย |
|---|---|
| `client_id` | รหัสประจำตัวแอปของเรา ที่ Google/GitHub ออกให้ตอนลงทะเบียนแอป (เปิดเผยได้ ฝัง ใน URL ได้) |
| `client_secret` | รหัสลับของแอป **ห้ามเปิดเผยเด็ดขาด** ใช้ตอนแลก code เป็น token เท่านั้น (server-side) |
| `redirect_uri` | URL ปลายทางที่ Google/GitHub จะ redirect กลับมาหลังผู้ใช้อนุญาต ต้องตรงกับที่ลงทะเบียนไว้เป๊ะ ๆ |
| `scope` | ขอบเขตสิทธิ์ที่ขอ เช่น `email`, `profile`, `read:user` — ยิ่งขอน้อย ยิ่งปลอดภัยและผู้ใช้ไว้ใจง่ายกว่า |
| `state` | ค่าสุ่มที่ client สร้างขึ้นก่อนเริ่ม flow ใช้ตรวจสอบว่า callback ที่ได้รับมาเป็น response ของ request ที่เราเริ่มไว้จริง (ป้องกัน CSRF — ขั้นตอนที่ 358.2) |
| `authorization code` (`code`) | รหัสชั่วคราวอายุสั้นที่ authorization server ออกให้ ใช้แลกเป็น token ได้ครั้งเดียว |
| `access_token` | token ที่ใช้เรียก API ของ resource server แทนผู้ใช้ มีอายุจำกัด (มักไม่กี่ชั่วโมง) |
| `refresh_token` | token อายุยาวกว่า ใช้ขอ access_token ใหม่โดยไม่ต้องให้ผู้ใช้ login ซ้ำ |
| `id_token` | JWT ที่มีข้อมูลตัวตนผู้ใช้ (เฉพาะ OIDC) เซ็นด้วย key ของ provider ตรวจสอบความถูกต้องได้ |
| `consent screen` | หน้าที่ผู้ใช้เห็นบนเว็บของ Google/GitHub เพื่ออนุญาตหรือปฏิเสธคำขอ |
| `PKCE` (Proof Key for Code Exchange) | กลไกเสริมความปลอดภัยของ Authorization Code Flow เดิมทีออกแบบมาสำหรับแอปมือถือ/SPA ที่เก็บ `client_secret` ไม่ได้ ปัจจุบันแนะนำให้ใช้แม้ใน web app ด้วย |

ตอนนี้คุณมีภาพรวมทฤษฎี OAuth2 ครบถ้วนแล้ว มาลงมือติดตั้ง django-allauth เพื่อให้
Django ทำทุกขั้นตอนที่อธิบายไปข้างต้นให้เราโดยอัตโนมัติกันต่อในขั้นตอนถัดไป

---

## ขั้นตอนที่ 352: ติดตั้งและตั้งค่า django-allauth เบื้องต้น (INSTALLED_APPS, MIDDLEWARE, SITE_ID, urls.py)

### 352.1 ติดตั้ง django-allauth

```bash
# ตรวจสอบว่า venv ถูก activate อยู่ก่อนเสมอ (ทบทวนจาก Part 001 ขั้นตอนที่ 8)
pip install django-allauth

# บันทึกลง requirements.txt ทันทีตามวินัยที่เรียนไปใน Part 001 ขั้นตอนที่ 9.3
pip freeze > requirements.txt
```

ตรวจสอบเวอร์ชันที่ติดตั้ง:

```bash
python -c "import allauth; print(allauth.__version__)"
```

หลักสูตรนี้เขียนโดยอ้างอิง django-allauth เวอร์ชัน **0.63 ขึ้นไป** ซึ่งเป็นเวอร์ชัน
ที่ปรับ settings หลายตัวให้อ่านง่ายขึ้นกว่ายุคก่อนหน้า (เช่น `ACCOUNT_LOGIN_METHODS`
แทนที่ `ACCOUNT_AUTHENTICATION_METHOD` เดิม) — ถ้าคุณเห็นบทความเก่าใช้ชื่อ setting
ต่างจากที่นี่ ให้ตรวจสอบ **CHANGELOG ทางการ** เสมอ เพราะ allauth เป็น package ที่
อัปเดต API การตั้งค่าค่อนข้างบ่อย

### 352.2 เพิ่ม `django.contrib.sites` และแอปของ allauth ใน `INSTALLED_APPS`

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django.contrib.sites",                          # <- allauth ต้องพึ่งพา Sites framework

    "allauth",                                        # <- core ของ allauth
    "allauth.account",                                # <- ระบบ local login/signup ของ allauth
    "allauth.socialaccount",                          # <- โครงสร้าง social login กลาง
    "allauth.socialaccount.providers.google",         # <- provider เฉพาะของ Google
    "allauth.socialaccount.providers.github",         # <- provider เฉพาะของ GitHub

    "accounts",
    "blog",
]

SITE_ID = 1   # <- บังคับต้องมี เพราะ allauth ผูก SocialApp เข้ากับ "Site" เสมอ (ขั้นตอนที่ 352.5)
```

| แอป | หน้าที่ |
|---|---|
| `django.contrib.sites` | เก็บข้อมูล "โดเมนของเว็บนี้คืออะไร" — allauth ใช้อ้างอิงว่า SocialApp แต่ละตัวใช้กับ site ไหน (รองรับ multi-site ในโปรเจกต์เดียวได้) |
| `allauth` | โมดูลกลางที่ให้ template tags, middleware, และ utility function ร่วมกัน |
| `allauth.account` | จัดการ login/logout/signup/password reset ของบัญชี local (คล้าย `django.contrib.auth` แต่ฟีเจอร์ครบกว่า เช่น email verification เต็มรูปแบบ) |
| `allauth.socialaccount` | โครงสร้างกลางสำหรับ social login ทุก provider เก็บ Model `SocialAccount`, `SocialApp`, `SocialToken` |
| `allauth.socialaccount.providers.<name>` | โค้ดเฉพาะของแต่ละ provider (Google, GitHub, Facebook, ...) — ต้องเพิ่มทีละตัวตามที่จะใช้จริง |

### 352.3 เพิ่ม `AccountMiddleware` ใน `MIDDLEWARE`

```python
# config/settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "allauth.account.middleware.AccountMiddleware",   # <- ต้องอยู่หลัง AuthenticationMiddleware เสมอ
]
```

`AccountMiddleware` (เพิ่มเข้ามาใน allauth เวอร์ชันใหม่ ๆ) ทำหน้าที่จัดการ flow
บางอย่างของ allauth ที่ต้องอาศัย request ระดับ middleware เช่น การตรวจจับว่าควร
บังคับผู้ใช้ยืนยันอีเมลก่อนเข้าถึงหน้าอื่นหรือไม่ — ถ้าลืมเพิ่ม middleware ตัวนี้
allauth เวอร์ชันใหม่จะแจ้ง `ImproperlyConfigured` ทันทีตอนรัน `runserver`

### 352.4 เพิ่ม `AUTHENTICATION_BACKENDS`

```python
# config/settings.py
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",       # <- ของเดิมจาก Django (เก็บไว้เผื่อ login ผ่าน Django Admin)
    "allauth.account.auth_backends.AuthenticationBackend",  # <- ของ allauth (ต้องเพิ่มเข้ามาใหม่)
]
```

ทบทวนจาก Part 031 ขั้นตอนที่ 303.3: `authenticate()` วนลูปผ่านทุก backend ใน
`AUTHENTICATION_BACKENDS` ตามลำดับ — การมีทั้งสองตัวพร้อมกันทำให้ทั้ง Django Admin
(ที่ใช้ `ModelBackend` มาตรฐาน) และหน้า login ของ allauth (ที่ต้องการ backend ของ
ตัวเองเพื่อรองรับ login ด้วย email/username พร้อมกัน) ทำงานได้ถูกต้องทั้งคู่

### 352.5 `SITE_ID` และการตั้งค่า Sites framework

หลังรัน `migrate` ครั้งแรก Django จะสร้าง Site record เริ่มต้นชื่อ
`example.com` ให้อัตโนมัติ (id เท่ากับ 1 ซึ่งตรงกับ `SITE_ID = 1` ที่ตั้งไว้)
**ต้องแก้ไข record นี้ให้ตรงกับโดเมนจริงของโปรเจกต์** ไม่เช่นนั้น allauth
จะสร้าง URL callback ผิดโดเมนตอน redirect กลับจาก Google/GitHub:

```bash
python manage.py migrate
python manage.py shell
```

```python
from django.contrib.sites.models import Site

site = Site.objects.get(pk=1)
site.domain = "127.0.0.1:8000"    # สำหรับ development
site.name = "My Django Blog (dev)"
site.save()
```

หรือแก้ผ่าน Django Admin ที่เมนู **Sites** ก็ได้เช่นกัน (ต้อง `createsuperuser`
ก่อนถ้ายังไม่มี — ทบทวนจาก Part 031 ขั้นตอนที่ 301.5) เมื่อ deploy ขึ้น production
จริง ต้องกลับมาแก้ `domain` เป็นโดเมนจริง เช่น `myblog.com` (จะเจาะลึกเรื่อง
environment-specific settings ใน Phase DevOps)

### 352.6 เพิ่ม URL ของ allauth ใน `urls.py`

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("accounts/", include("allauth.urls")),   # <- แทนที่ django.contrib.auth.urls เดิม
    path("blog/", include("blog.urls")),
]
```

> **ข้อควรระวังสำคัญ**: `allauth.urls` ครอบคลุมทุก URL ที่ `django.contrib.auth.urls`
> เคยให้ไว้แล้ว (login, logout, password reset ฯลฯ) **พร้อมเพิ่มเติม** URL สำหรับ
> social login และ signup ของตัวเอง ดังนั้นต้อง**เอา
> `path("accounts/", include("django.contrib.auth.urls"))` และ URL signup แยกต่าง
> หากที่เคยเพิ่มใน Part 031 ขั้นตอนที่ 306.1 ออก** เพื่อไม่ให้ URL pattern ชนกัน
> (`login`, `logout` เป็น URL name ที่ทั้งสองระบบใช้ชื่อเดียวกัน)

### 352.7 ตารางเปรียบเทียบ URL patterns สำคัญที่ `allauth.urls` เพิ่มเข้ามา

| URL name | Path เต็ม | หน้าที่ |
|---|---|---|
| `account_login` | `/accounts/login/` | หน้า login (local + ปุ่ม social login ถ้าตั้งค่าไว้) |
| `account_signup` | `/accounts/signup/` | หน้าสมัครสมาชิกด้วย email/password (มาแทน `UserCreationForm` ของ Part 031) |
| `account_logout` | `/accounts/logout/` | ออกจากระบบ (default บังคับ POST เหมือน Django ปกติ) |
| `account_email` | `/accounts/email/` | จัดการอีเมลของบัญชี (เพิ่ม/ลบ/ตั้ง primary) |
| `account_change_password` | `/accounts/password/change/` | เปลี่ยนรหัสผ่าน |
| `socialaccount_connections` | `/accounts/3rdparty/` | ดูรายการ social account ที่เชื่อมกับบัญชีนี้ (ขั้นตอนที่ 355) |
| `<provider>_login` | `/accounts/google/login/` | เริ่ม OAuth flow กับ provider นั้น ๆ |
| `<provider>_callback` | `/accounts/google/login/callback/` | URL ที่ provider redirect กลับมาหลังผู้ใช้อนุญาต (ต้องนำไปตั้งใน Google/GitHub Console) |

ดูรายการทั้งหมดได้จริงในเครื่องของคุณด้วยคำสั่ง:

```bash
python manage.py show_urls | grep account
# หรือถ้าไม่มี django-extensions ติดตั้งไว้ ให้เปิด shell แล้วตรวจสอบ
python manage.py shell -c "
from django.urls import get_resolver
for pattern in get_resolver().url_patterns:
    print(pattern)
"
```

### 352.8 เพิ่ม settings พื้นฐานของ `allauth.account`

```python
# config/settings.py
LOGIN_REDIRECT_URL = "blog:post-list"
ACCOUNT_LOGOUT_ON_GET = False          # บังคับ logout ผ่าน POST เท่านั้น (เหมือน Part 031 ขั้นตอนที่ 307.3)

ACCOUNT_LOGIN_METHODS = {"email"}      # ให้ login ด้วย email แทน username (allauth >= 0.63)
ACCOUNT_SIGNUP_FIELDS = ["email*", "username*", "password1*", "password2*"]
ACCOUNT_EMAIL_VERIFICATION = "optional"   # "mandatory" | "optional" | "none" — จะเจาะลึกเรื่อง email verification เต็มรูปแบบใน Part 035
ACCOUNT_USERNAME_MIN_LENGTH = 4

SOCIALACCOUNT_AUTO_SIGNUP = True       # ให้สมัครสมาชิกอัตโนมัติเมื่อ login ผ่าน social ครั้งแรก (ไม่ต้องกรอกฟอร์มเพิ่ม)
SOCIALACCOUNT_STORE_TOKENS = False     # ไม่เก็บ access/refresh token ของ provider ไว้ในฐานข้อมูล (ขั้นตอนที่ 358.5)
```

| Setting | ความหมาย |
|---|---|
| `ACCOUNT_LOGIN_METHODS` | ชุดของช่องทาง login ที่อนุญาต (`{"username"}`, `{"email"}`, หรือ `{"username", "email"}`) |
| `ACCOUNT_SIGNUP_FIELDS` | field ที่ต้องกรอกตอนสมัคร local account เครื่องหมาย `*` แปลว่า required |
| `ACCOUNT_EMAIL_VERIFICATION` | บังคับยืนยันอีเมลก่อนใช้งานหรือไม่ (`mandatory` ปลอดภัยสุดสำหรับ production) |
| `SOCIALACCOUNT_AUTO_SIGNUP` | `True` = สร้างบัญชีอัตโนมัติทันทีที่ login ผ่าน social สำเร็จ โดยไม่ต้องถามข้อมูลเพิ่ม |
| `SOCIALACCOUNT_STORE_TOKENS` | เก็บ access token ของ Google/GitHub ไว้ในตาราง `SocialToken` หรือไม่ (เปิดเมื่อต้องเรียก API ของ provider ต่อในภายหลังเท่านั้น) |

### 352.9 รัน migration และทดสอบว่า allauth ทำงาน

```bash
python manage.py migrate
python manage.py runserver
```

เปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/accounts/login/` ควรเห็นหน้า login
ของ allauth (หน้าตาเรียบ ๆ ตาม template default ที่ยังไม่ได้ปรับแต่ง — เราจะ
ปรับให้สวยงามตรงกับดีไซน์เว็บใน ขั้นตอนที่ 356) พร้อมลิงก์ไปหน้า signup

ตรวจสอบตารางที่ allauth สร้างในฐานข้อมูลผ่าน `python manage.py shell`:

```python
from allauth.socialaccount.models import SocialAccount, SocialApp, SocialToken
from allauth.account.models import EmailAddress

print(SocialApp.objects.count())    # 0 — ยังไม่ได้สร้าง provider เลยในขั้นตอนนี้
print(SocialAccount.objects.count())  # 0
print(EmailAddress.objects.count())   # 0
```

ยังไม่มี `SocialApp` แม้แต่ตัวเดียว เพราะเรายังไม่ได้สร้าง OAuth credentials จริง
กับ Google หรือ GitHub — นั่นคือสิ่งที่เราจะทำในสองขั้นตอนถัดไป

### 352.10 ตารางสรุป Model หลักที่ allauth เพิ่มเข้ามา

| Model | อยู่ในแอป | เก็บอะไร |
|---|---|---|
| `SocialApp` | `allauth.socialaccount` | credential ของแต่ละ OAuth provider (`client_id`, `secret`) ผูกกับ `Site` |
| `SocialAccount` | `allauth.socialaccount` | บัญชี social ที่เชื่อมกับ `User` ในระบบ (1 user เชื่อมได้หลาย provider) |
| `SocialToken` | `allauth.socialaccount` | access/refresh token ของแต่ละ `SocialAccount` (เก็บเมื่อ `SOCIALACCOUNT_STORE_TOKENS=True` เท่านั้น) |
| `EmailAddress` | `allauth.account` | อีเมลทั้งหมดของผู้ใช้ พร้อมสถานะ verified/primary (รองรับหลายอีเมลต่อ 1 user) |

---

## ขั้นตอนที่ 353: เชื่อมต่อ Google Login จริง — สร้าง OAuth credentials ใน Google Cloud Console, ตั้งค่า `SocialApp` ใน Django Admin

### 353.1 สร้างโปรเจกต์ใน Google Cloud Console

1. เปิด https://console.cloud.google.com/
2. คลิกเมนู **Select a project → New Project**
3. ตั้งชื่อโปรเจกต์ เช่น `django-blog-oauth` แล้วกด **Create**
4. รอสักครู่จนโปรเจกต์ถูกสร้าง แล้วเลือกโปรเจกต์นั้นให้ active อยู่มุมบนซ้าย

### 353.2 ตั้งค่า OAuth Consent Screen

1. ไปที่เมนู **APIs & Services → OAuth consent screen**
2. เลือก **User Type**: `External` (สำหรับให้ผู้ใช้ทั่วไป login ได้ ไม่ใช่แค่คนใน
   organization เดียวกัน)
3. กรอกข้อมูลที่จำเป็น:
   - **App name**: ชื่อแอปที่ผู้ใช้จะเห็นตอน consent screen เช่น `My Django Blog`
   - **User support email**: อีเมลของคุณ
   - **Developer contact information**: อีเมลของคุณอีกครั้ง
4. ในหน้า **Scopes** เพิ่ม scope `.../auth/userinfo.email` และ
   `.../auth/userinfo.profile` (allauth ขอ scope เหล่านี้เป็นค่าเริ่มต้นอยู่แล้ว
   แต่ควรตรวจสอบให้แน่ใจว่าอยู่ในรายการที่ Google อนุมัติ)
5. ในหน้า **Test users** (ถ้าแอปยังอยู่สถานะ "Testing") เพิ่มอีเมลของคุณเองและ
   ผู้ทดสอบคนอื่น ๆ ที่จะใช้ทดสอบ login ก่อนแอปได้รับการ verify จาก Google
   (แอปที่ยังไม่ verify จะจำกัดให้ login ได้เฉพาะอีเมลในรายการนี้เท่านั้น)

### 353.3 สร้าง OAuth Client ID

1. ไปที่เมนู **APIs & Services → Credentials**
2. คลิก **Create Credentials → OAuth client ID**
3. เลือก **Application type**: `Web application`
4. ตั้งชื่อ เช่น `django-blog-web-client`
5. ในช่อง **Authorized JavaScript origins** เพิ่ม:
   ```
   http://127.0.0.1:8000
   ```
6. ในช่อง **Authorized redirect URIs** เพิ่ม **ให้ตรงกับ URL callback ของ allauth
   เป๊ะ ๆ** (ทบทวนตารางในขั้นตอนที่ 352.7):
   ```
   http://127.0.0.1:8000/accounts/google/login/callback/
   ```
7. กด **Create** — Google จะแสดง **Client ID** และ **Client Secret** ให้ทันที
   คัดลอกเก็บไว้ (สามารถกลับมาดูซ้ำได้ทีหลังผ่านหน้า Credentials เช่นกัน)

> **ข้อควรระวังเรื่อง trailing slash**: `redirect_uri` ที่ allauth สร้างขึ้น
> **มี `/` ปิดท้ายเสมอ** ถ้าคุณลงทะเบียนใน Google Console แบบไม่มี `/` ปิดท้าย
> Google จะปฏิเสธ callback ด้วย error `redirect_uri_mismatch` ทันที เพราะ OAuth2
> spec กำหนดให้ `redirect_uri` ต้องตรงกันแบบ **exact string match** เท่านั้น
> (จะเจาะลึกเหตุผลด้านความปลอดภัยใน ขั้นตอนที่ 358.1)

### 353.4 เก็บ Client ID/Secret อย่างปลอดภัยด้วย environment variables

**ห้าม hardcode `client_secret` ลงในโค้ดหรือ commit ขึ้น Git เด็ดขาด** ให้ใช้
`django-environ` (หรือ `python-decouple`) ที่ควรเคยติดตั้งไว้แล้วตั้งแต่ตอนจัดการ
`SECRET_KEY` ของโปรเจกต์:

```bash
pip install django-environ
```

```bash
# .env  (ต้องอยู่ใน .gitignore เสมอ — ทบทวนจาก Part 001 ขั้นตอนที่ 8.5)
GOOGLE_OAUTH_CLIENT_ID=1234567890-abcdefg.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxxxxxxxxxx
```

```python
# config/settings.py
import environ

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")

GOOGLE_OAUTH_CLIENT_ID = env("GOOGLE_OAUTH_CLIENT_ID")
GOOGLE_OAUTH_CLIENT_SECRET = env("GOOGLE_OAUTH_CLIENT_SECRET")
```

### 353.5 ตั้งค่า `SocialApp` ผ่าน Django Admin

เปิด `http://127.0.0.1:8000/admin/` แล้ว login ด้วย superuser จากนั้นไปที่
**Social Applications → Add social application**:

| ช่อง | ค่าที่กรอก |
|---|---|
| **Provider** | `Google` |
| **Name** | `Google OAuth` (ชื่อเรียกภายใน ตั้งอะไรก็ได้) |
| **Client id** | ค่า `client_id` จาก Google Console |
| **Secret key** | ค่า `client_secret` จาก Google Console |
| **Sites** | เลือก site ที่ตั้งค่าไว้ในขั้นตอนที่ 352.5 (`127.0.0.1:8000`) แล้วกดปุ่มลูกศรย้ายไปช่อง "Chosen sites" |

หรือสร้างผ่านโค้ด (สะดวกกว่าเมื่อทำงานเป็นทีมและต้องการ reproducible setup)
ด้วย management command หรือ data migration:

```python
# accounts/management/commands/setup_google_oauth.py
from django.conf import settings
from django.contrib.sites.models import Site
from django.core.management.base import BaseCommand

from allauth.socialaccount.models import SocialApp


class Command(BaseCommand):
    help = "สร้างหรืออัปเดต SocialApp ของ Google จาก environment variables"

    def handle(self, *args, **options):
        site = Site.objects.get(pk=settings.SITE_ID)

        social_app, created = SocialApp.objects.update_or_create(
            provider="google",
            name="Google OAuth",
            defaults={
                "client_id": settings.GOOGLE_OAUTH_CLIENT_ID,
                "secret": settings.GOOGLE_OAUTH_CLIENT_SECRET,
            },
        )
        social_app.sites.add(site)

        action = "สร้างใหม่" if created else "อัปเดต"
        self.stdout.write(self.style.SUCCESS(f"{action} SocialApp สำหรับ Google เรียบร้อย"))
```

```bash
python manage.py setup_google_oauth
```

วิธีนี้ปลอดภัยกว่าและเหมาะกับ CI/CD เพราะไม่ต้องเข้า Admin ด้วยมือทุกครั้งที่ deploy
เครื่องใหม่ (ทบทวนแนวคิด management command จาก Part 032 ขั้นตอนที่ 318.6)

### 353.6 ทางเลือกใหม่: ตั้งค่า provider ผ่าน `SOCIALACCOUNT_PROVIDERS` โดยไม่ต้องใช้ Admin

django-allauth เวอร์ชันใหม่รองรับการตั้งค่า credential ผ่าน `settings.py` โดยตรง
(เรียกว่า **"provider apps" แบบ config-based**) ซึ่งเหมาะกับการจัดการผ่าน
environment variables ล้วน ๆ โดยไม่ต้องพึ่งฐานข้อมูลเลย:

```python
# config/settings.py
SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "APPS": [
            {
                "client_id": env("GOOGLE_OAUTH_CLIENT_ID"),
                "secret": env("GOOGLE_OAUTH_CLIENT_SECRET"),
                "key": "",
            }
        ],
        "SCOPE": ["profile", "email"],
        "AUTH_PARAMS": {"access_type": "online"},
    },
}
```

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| `SocialApp` ผ่าน Django Admin/ORM | จัดการหลาย environment ได้จากฐานข้อมูล เปลี่ยนได้โดยไม่ redeploy โค้ด รองรับ multi-tenant/multi-site ได้ยืดหยุ่น | ต้องเข้าฐานข้อมูลเพื่อตั้งค่า เพิ่มขั้นตอน deploy |
| `SOCIALACCOUNT_PROVIDERS["APPS"]` ผ่าน settings | ตั้งค่าเป็น environment variable ได้ล้วน ๆ เหมาะกับ 12-factor app และ container-based deployment | เปลี่ยน credential ต้อง redeploy เสมอ, จัดการ multi-site ยุ่งยากกว่า |

หลักสูตรนี้แนะนำแนวทาง `SocialApp` ผ่าน Admin/ORM เป็นค่าเริ่มต้น เพราะสอดคล้องกับ
Sites framework และยืดหยุ่นกว่าเมื่อโปรเจกต์เติบโต แต่ทั้งสองแนวทางใช้ร่วมกันได้
ไม่ขัดแย้งกัน (เลือกใช้แค่แบบเดียวต่อ provider ก็เพียงพอ)

### 353.7 ทดสอบ Google Login จริง

```bash
python manage.py runserver
```

1. เปิด `http://127.0.0.1:8000/accounts/login/`
2. ควรเห็นปุ่ม "Google" ปรากฏขึ้นมาแล้ว (เพราะตอนนี้มี `SocialApp` ที่ผูกกับ site
   ที่ถูกต้องแล้ว) คลิกปุ่มนั้น
3. ระบบจะ redirect ไปหน้า login ของ Google → login ด้วยบัญชี Google ของคุณ (ต้อง
   เป็นอีเมลที่อยู่ในรายการ Test users ถ้าแอปยังไม่ verify) → กด "Continue" ที่หน้า
   consent screen
4. Google redirect กลับมาที่เว็บของเรา ควร login สำเร็จทันทีและเห็นชื่อผู้ใช้ที่
   `base.html` (ทบทวน template จาก Part 031 ขั้นตอนที่ 308.2)

### 353.8 ตรวจสอบข้อมูลที่ได้จาก Google หลัง login สำเร็จ

```python
# python manage.py shell
from django.contrib.auth import get_user_model
from allauth.socialaccount.models import SocialAccount

User = get_user_model()
user = User.objects.latest("date_joined")
print(user.username, user.email)

social_account = SocialAccount.objects.get(user=user, provider="google")
print(social_account.extra_data)
```

ผลลัพธ์ตัวอย่างของ `extra_data` (ข้อมูลดิบทั้งหมดที่ Google ส่งกลับมา เก็บเป็น
`JSONField`):

```json
{
    "sub": "108234982340982340982",
    "name": "Somchai Jaidee",
    "given_name": "Somchai",
    "family_name": "Jaidee",
    "picture": "https://lh3.googleusercontent.com/a/xxxxx",
    "email": "somchai@gmail.com",
    "email_verified": true,
    "locale": "th"
}
```

field `extra_data` นี้คือสิ่งที่เราจะนำไปใช้สร้าง Profile อัตโนมัติผ่าน signal
ใน ขั้นตอนที่ 357 (เช่นดึง `picture` มาเก็บเป็นรูปโปรไฟล์เริ่มต้น)

---

## ขั้นตอนที่ 354: เชื่อมต่อ GitHub Login จริง — สร้าง OAuth App ใน GitHub Developer Settings

### 354.1 สร้าง OAuth App ใน GitHub

1. เปิด https://github.com/settings/developers (ต้อง login GitHub ก่อน)
2. ไปที่แท็บ **OAuth Apps → New OAuth App**
3. กรอกข้อมูล:

| ช่อง | ค่าที่กรอก |
|---|---|
| **Application name** | `My Django Blog (dev)` |
| **Homepage URL** | `http://127.0.0.1:8000` |
| **Application description** | (ไม่บังคับ) คำอธิบายสั้น ๆ |
| **Authorization callback URL** | `http://127.0.0.1:8000/accounts/github/login/callback/` |

4. กด **Register application**
5. หน้าถัดไปจะแสดง **Client ID** ทันที ส่วน **Client Secret** ต้องกดปุ่ม
   **Generate a new client secret** เพื่อสร้าง (แสดงให้เห็นแค่ครั้งเดียว
   ต้องคัดลอกเก็บทันที)

> **ข้อสังเกต**: GitHub อนุญาตให้ตั้งค่าได้แค่ **1 callback URL ต่อ 1 OAuth App**
> เท่านั้น (ต่างจาก Google ที่รองรับหลาย redirect URI ในแอปเดียว) ถ้าต้องการแยก
> credential สำหรับ development/staging/production ต้องสร้าง **OAuth App แยกกัน
> คนละตัว** สำหรับแต่ละ environment

### 354.2 เก็บ credential และตั้งค่า `SocialApp`

```bash
# .env
GITHUB_OAUTH_CLIENT_ID=Iv1.xxxxxxxxxxxxxxxx
GITHUB_OAUTH_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

```python
# config/settings.py
GITHUB_OAUTH_CLIENT_ID = env("GITHUB_OAUTH_CLIENT_ID")
GITHUB_OAUTH_CLIENT_SECRET = env("GITHUB_OAUTH_CLIENT_SECRET")
```

ขยาย management command จากขั้นตอนที่ 353.5 ให้รองรับหลาย provider:

```python
# accounts/management/commands/setup_oauth_providers.py
from django.conf import settings
from django.contrib.sites.models import Site
from django.core.management.base import BaseCommand

from allauth.socialaccount.models import SocialApp


class Command(BaseCommand):
    help = "สร้างหรืออัปเดต SocialApp ของทุก provider จาก environment variables"

    PROVIDERS = {
        "google": ("GOOGLE_OAUTH_CLIENT_ID", "GOOGLE_OAUTH_CLIENT_SECRET", "Google OAuth"),
        "github": ("GITHUB_OAUTH_CLIENT_ID", "GITHUB_OAUTH_CLIENT_SECRET", "GitHub OAuth"),
    }

    def handle(self, *args, **options):
        site = Site.objects.get(pk=settings.SITE_ID)

        for provider, (id_key, secret_key, name) in self.PROVIDERS.items():
            client_id = getattr(settings, id_key, None)
            client_secret = getattr(settings, secret_key, None)

            if not client_id or not client_secret:
                self.stdout.write(self.style.WARNING(f"ข้าม {provider}: ไม่พบ credential ใน settings"))
                continue

            social_app, created = SocialApp.objects.update_or_create(
                provider=provider,
                name=name,
                defaults={"client_id": client_id, "secret": client_secret},
            )
            social_app.sites.add(site)
            action = "สร้างใหม่" if created else "อัปเดต"
            self.stdout.write(self.style.SUCCESS(f"{action} SocialApp: {provider}"))
```

```bash
python manage.py setup_oauth_providers
```

### 354.3 GitHub และปัญหาอีเมลที่มองไม่เห็น (Private Email)

จุดที่ต่างจาก Google อย่างชัดเจน: ผู้ใช้ GitHub จำนวนมาก**ตั้งค่าอีเมลเป็น
private** ในโปรไฟล์ ทำให้ endpoint `/user` มาตรฐานของ GitHub API **คืนค่า
`email: null`** แม้ผู้ใช้จะ login สำเร็จแล้วก็ตาม django-allauth จัดการปัญหานี้
ให้อัตโนมัติโดยขอ scope เพิ่มเติมชื่อ `user:email` แล้วเรียก endpoint พิเศษ
`https://api.github.com/user/emails` แทน ซึ่งคืนรายการอีเมลทั้งหมด (รวม verified
status) แม้จะตั้งเป็น private ก็ตาม — ตราบใดที่คุณไม่ได้ override
`SOCIALACCOUNT_PROVIDERS["github"]["SCOPE"]` ทับด้วยค่าที่ไม่มี `user:email`
กลไกนี้จะทำงานให้อัตโนมัติ

```python
# config/settings.py — ตรวจสอบให้แน่ใจว่า scope นี้มีอยู่ (เป็นค่าเริ่มต้นของ allauth อยู่แล้ว)
SOCIALACCOUNT_PROVIDERS = {
    "github": {
        "SCOPE": ["user:email", "read:user"],
    },
}
```

### 354.4 ทดสอบ GitHub Login จริง

```bash
python manage.py runserver
```

เปิด `http://127.0.0.1:8000/accounts/login/` ควรเห็นปุ่มทั้ง "Google" และ "GitHub"
ปรากฏพร้อมกันแล้ว คลิกปุ่ม "GitHub" → login/authorize ที่ฝั่ง GitHub → redirect
กลับมาที่เว็บของเรา ควร login สำเร็จเช่นเดียวกับ Google

### 354.5 ตารางเปรียบเทียบ Google OAuth vs GitHub OAuth

| ประเด็น | Google | GitHub |
|---|---|---|
| มาตรฐานที่ใช้ | OAuth2 + OpenID Connect (มี `id_token`) | OAuth2 ล้วน ๆ (ไม่มี `id_token`) |
| จำนวน redirect URI ต่อแอป | หลายอันได้ในแอปเดียว | **1 อันต่อ 1 OAuth App เท่านั้น** ต้องแยกแอปตาม environment |
| การได้มาซึ่งอีเมล | มากับ `id_token`/`userinfo` โดยตรง พร้อม flag `email_verified` | ต้องขอ scope `user:email` แยก เพราะอีเมลอาจถูกตั้งเป็น private |
| Consent screen | ต้องผ่านขั้นตอน "verify app" ของ Google หากมีผู้ใช้เกิน 100 คน (unverified app) | ไม่มีขั้นตอน verify แยก ใช้งานได้ทันทีหลังสร้าง OAuth App |
| รูปโปรไฟล์ | field `picture` ใน `extra_data` | field `avatar_url` ใน `extra_data` |

### 354.6 ปุ่ม Login หลาย Provider ในหน้าเดียว

Template default ของ allauth จะแสดงปุ่ม provider ทุกตัวที่มี `SocialApp` ผูกกับ
site ปัจจุบันโดยอัตโนมัติ ผ่าน template tag `{% load socialaccount %}`:

```html
<!-- ตัวอย่างพื้นฐานก่อนปรับแต่งใน ขั้นตอนที่ 356 -->
{% load socialaccount %}

<div class="social-login-buttons">
    {% get_providers as socialaccount_providers %}
    {% for provider in socialaccount_providers %}
        <a href="{% provider_login_url provider.id process='login' %}">
            เข้าสู่ระบบด้วย {{ provider.name }}
        </a>
    {% endfor %}
</div>
```

โค้ดนี้จะแสดงปุ่ม "เข้าสู่ระบบด้วย Google" และ "เข้าสู่ระบบด้วย GitHub" โดยไม่ต้อง
เขียนโค้ดแยกทีละ provider — เมื่อเพิ่ม provider ใหม่ในอนาคต (เช่น Facebook)
ปุ่มจะปรากฏขึ้นเองโดยไม่ต้องแก้ template เลย

---

## ขั้นตอนที่ 355: การเชื่อม Social Account เข้ากับบัญชีที่มีอยู่แล้ว (Account Linking)

### 355.1 สถานการณ์ปัญหา: ผู้ใช้มีบัญชีอยู่แล้ว แล้วอยากเชื่อม Google ทีหลัง

ลองจินตนาการ flow นี้ซึ่งเกิดขึ้นจริงบ่อยมากในระบบจริง:

```
1. คุณสมชาย สมัครสมาชิกด้วย email/password ธรรมดา
   (somchai@gmail.com / password: xxxx) ผ่านหน้า signup ปกติ

2. สองสัปดาห์ต่อมา คุณสมชายลืมรหัสผ่าน แต่จำได้ว่ามีบัญชี Gmail
   ที่ใช้อีเมลเดียวกัน (somchai@gmail.com) เลยลองกดปุ่ม
   "Sign in with Google" แทน

3. คำถามสำคัญ: ระบบควร (ก) สร้างบัญชีใหม่ซ้อนขึ้นมาอีกบัญชี
   หรือ (ข) รู้ว่านี่คือคนเดียวกัน แล้ว "เชื่อม" Google account
   เข้ากับบัญชีเดิมที่มีอยู่แล้ว?
```

คำตอบที่ถูกต้องคือ **(ข)** เสมอ — การสร้างบัญชีซ้อนจะทำให้ผู้ใช้สับสน (มีบทความ/
ข้อมูลกระจายอยู่คนละบัญชี) และเป็นประสบการณ์ผู้ใช้ที่แย่มาก django-allauth มีกลไก
จัดการเรื่องนี้ให้อัตโนมัติในระดับหนึ่ง แต่ต้องเข้าใจเงื่อนไขให้ชัดเจนเพื่อไม่ให้
เกิดช่องโหว่ความปลอดภัย (อธิบายในขั้นตอนที่ 355.4)

### 355.2 พฤติกรรมเริ่มต้นของ allauth เมื่อพบอีเมลซ้ำ

ค่าเริ่มต้น allauth จะ**ไม่เชื่อมบัญชีให้อัตโนมัติทันที** แม้อีเมลจะตรงกัน
เพราะการเชื่อมบัญชีอัตโนมัติแบบไม่มีเงื่อนไขถือเป็นความเสี่ยงด้านความปลอดภัย
(อธิบายในขั้นตอนที่ 355.4) แต่จะแสดงหน้าแจ้งเตือนให้ผู้ใช้ทราบว่ามีบัญชีที่ใช้
อีเมลนี้อยู่แล้ว และแนะนำให้ login ด้วยวิธีเดิมก่อน แล้วค่อยไปเชื่อม social account
เพิ่มทีหลังผ่านหน้า **Connections**

### 355.3 หน้า "Connections" — เชื่อม Social Account ด้วยตัวเองตอน login อยู่แล้ว

allauth เตรียม URL `socialaccount_connections` (`/accounts/3rdparty/`) ไว้ให้
ผู้ใช้ที่ login อยู่แล้วสามารถเชื่อม/ยกเลิกการเชื่อม social account ได้เอง:

```html
<!-- templates/socialaccount/connections.html (override จาก default ของ allauth) -->
{% extends "base.html" %}
{% load socialaccount %}

{% block content %}
<h1>บัญชีที่เชื่อมต่ออยู่</h1>

{% if form.accounts %}
    <ul>
    {% for account in form.accounts %}
        <li>
            {{ account.get_provider.name }} — {{ account }}
            <form method="post" action="{% url 'socialaccount_connections' %}" style="display: inline;">
                {% csrf_token %}
                <input type="hidden" name="account" value="{{ account.id }}">
                <button type="submit" name="action_remove">ยกเลิกการเชื่อม</button>
            </form>
        </li>
    {% endfor %}
    </ul>
{% else %}
    <p>คุณยังไม่ได้เชื่อมบัญชี social ใด ๆ</p>
{% endif %}

<h2>เพิ่มการเชื่อมต่อใหม่</h2>
{% get_providers as socialaccount_providers %}
{% for provider in socialaccount_providers %}
    <a href="{% provider_login_url provider.id process='connect' %}">
        เชื่อมกับ {{ provider.name }}
    </a>
{% endfor %}
{% endblock %}
```

จุดสำคัญที่สุดในโค้ดนี้คือพารามิเตอร์ **`process='connect'`** (เทียบกับ
`process='login'` ที่ใช้ในหน้า login ปกติ) — allauth ใช้ค่านี้แยกแยะว่า:

| `process` | ความหมาย | เงื่อนไข |
|---|---|---|
| `login` | ผู้ใช้กำลังจะ **login** ด้วย social account นี้ (อาจสร้างบัญชีใหม่ถ้ายังไม่มี) | ผู้ใช้ยังไม่ login อยู่ |
| `connect` | ผู้ใช้ที่ **login อยู่แล้ว** ต้องการ**เพิ่ม** social account เข้ากับบัญชีปัจจุบันของตัวเอง | ผู้ใช้ต้อง login อยู่ก่อนเสมอ ระบบจะปฏิเสธถ้าไม่ได้ login |

ด้วยกลไกนี้ คุณสมชายจาก 355.1 สามารถ (1) login ด้วย email/password เดิมก่อน
(2) ไปที่หน้า Connections (3) กด "เชื่อมกับ Google" ซึ่งจะพาไปผ่าน OAuth flow
เหมือนเดิมทุกประการ แต่จบลงด้วยการ**เพิ่ม `SocialAccount` ใหม่ผูกกับ `User`
ที่ login อยู่**แทนการสร้าง user ใหม่

### 355.4 Auto-linking แบบมีเงื่อนไข: ทำไมต้องเช็ค "verified email" ก่อนเชื่อมอัตโนมัติ

บางระบบต้องการ **auto-link บัญชีอัตโนมัติทันที** โดยไม่ให้ผู้ใช้ต้องเข้าหน้า
Connections เอง (ลด friction) ทำได้โดย override `pre_social_login()` ใน
custom adapter — แต่ **ต้องตรวจสอบว่าอีเมลนั้นถูกยืนยัน (verified) โดย provider
แล้วเท่านั้น** ไม่เช่นนั้นจะเปิดช่องโหว่ร้ายแรง:

```
สถานการณ์อันตรายถ้า auto-link โดยไม่เช็ค verified:

1. เหยื่อสมัครบัญชีในเว็บเราด้วย victim@company.com (ยังไม่ verify)
2. ผู้โจมตีไปสร้างบัญชี GitHub ปลอมโดยตั้งอีเมลเป็น
   victim@company.com (GitHub ยอมให้ตั้งอีเมลอะไรก็ได้ตอนสมัคร
   ถ้ายังไม่ยืนยัน)
3. ผู้โจมตี login เว็บเราผ่าน GitHub ปลอมนั้น
4. ถ้าระบบ auto-link ตามอีเมลโดยไม่เช็ค verified
   → ผู้โจมตีจะได้สิทธิ์เข้าบัญชีของเหยื่อทันที!
   (Account Takeover ผ่าน Email Confusion Attack)
```

โค้ดที่ปลอดภัย ต้องเช็ค `email_verified` (Google) หรือ `verified` (GitHub) จาก
`extra_data` ก่อนเชื่อมบัญชีเสมอ:

```python
# accounts/adapters.py
from allauth.account.utils import filter_users_by_email
from allauth.socialaccount.adapter import DefaultSocialAccountAdapter


class SocialAccountAdapter(DefaultSocialAccountAdapter):
    def pre_social_login(self, request, sociallogin):
        """
        เรียกทุกครั้งก่อน allauth ตัดสินใจว่าจะสร้างบัญชีใหม่หรือ login
        เข้าบัญชีเดิม — ใช้จุดนี้ทำ auto-link แบบปลอดภัย
        """
        if sociallogin.is_existing:
            # มี SocialAccount นี้ผูกกับ user อยู่แล้ว ไม่ต้องทำอะไรเพิ่ม
            return

        email = sociallogin.account.extra_data.get("email")
        if not email:
            return

        # ตรวจสอบ verified flag ตาม provider แต่ละตัว
        provider = sociallogin.account.provider
        if provider == "google":
            is_verified = sociallogin.account.extra_data.get("email_verified", False)
        elif provider == "github":
            # GitHub ส่ง verified มาต่างรูปแบบ ต้องเช็คผ่าน socialaccount.extra_data
            # ที่ allauth ดึงมาจาก endpoint /user/emails (ขั้นตอนที่ 354.3)
            is_verified = sociallogin.account.extra_data.get("verified", False)
        else:
            is_verified = False

        if not is_verified:
            return   # ไม่เชื่อมอัตโนมัติถ้าอีเมลยังไม่ verified โดยเด็ดขาด

        existing_users = filter_users_by_email(email)
        if len(existing_users) == 1:
            # เจอผู้ใช้เดิมที่ verified email ตรงกันพอดี 1 คน — เชื่อมให้อัตโนมัติ
            sociallogin.connect(request, existing_users[0])
```

```python
# config/settings.py
SOCIALACCOUNT_ADAPTER = "accounts.adapters.SocialAccountAdapter"
```

### 355.5 ตารางสรุปกฎการทำ Account Linking อย่างปลอดภัย

| กฎ | เหตุผล |
|---|---|
| Auto-link ได้ **เฉพาะ** เมื่ออีเมลถูก verified โดย provider เท่านั้น | ป้องกัน Account Takeover ผ่าน Email Confusion Attack (ขั้นตอนที่ 355.4) |
| ถ้าเจอผู้ใช้ที่ตรงกันมากกว่า 1 คน (`len(existing_users) > 1`) ห้าม auto-link | กรณีนี้ไม่ควรเกิดถ้า `email` เป็น unique อยู่แล้ว (Part 032) แต่ถ้าเกิดขึ้นให้ปฏิเสธไว้ก่อนเพื่อความปลอดภัย |
| ให้ผู้ใช้ยืนยันด้วยรหัสผ่านเดิมก่อน connect ในกรณีอ่อนไหวสูง (เช่น ระบบการเงิน) | เพิ่มชั้นความปลอดภัย แม้อีเมลจะ verified แล้วก็ตาม |
| แสดง log/notification ทุกครั้งที่มีการเชื่อม/ยกเลิกเชื่อม social account | ให้ผู้ใช้ตรวจสอบย้อนหลังได้หากเกิดการเชื่อมที่ไม่ได้ตั้งใจ |
| อนุญาตให้ผู้ใช้ยกเลิกการเชื่อม (`unlink`) ได้เองเสมอ ผ่านหน้า Connections | สิทธิ์ควบคุมบัญชีของผู้ใช้เอง (data ownership) |

---

## ขั้นตอนที่ 356: Customize Template ของ allauth (หน้า login/signup ให้ตรง design ของเว็บ)

### 356.1 กลไก Template Resolution ของ allauth

allauth ทำงานตามหลักการเดียวกับที่เรียนไปใน Part 008: ถ้า Django หา template
เจอในโฟลเดอร์ `templates/` ระดับโปรเจกต์ก่อน (ตามลำดับ `DIRS` ใน `TEMPLATES`)
มันจะใช้ template นั้นแทน template default ที่มากับ package — เราจึง
**override ได้โดยไม่ต้องแก้โค้ดของ allauth เองเลยแม้แต่บรรทัดเดียว** เพียงสร้าง
ไฟล์ path เดียวกันไว้ในโปรเจกต์ของเรา

### 356.2 ตารางไฟล์ template หลักที่มักต้อง override

| Path ที่ต้องสร้าง | ใช้กับหน้าอะไร |
|---|---|
| `templates/account/login.html` | หน้า login (local + ปุ่ม social) |
| `templates/account/signup.html` | หน้าสมัครสมาชิกด้วย email/password |
| `templates/account/logout.html` | หน้ายืนยันก่อน logout |
| `templates/account/email.html` | หน้าจัดการอีเมลของบัญชี |
| `templates/account/password_change.html` | หน้าเปลี่ยนรหัสผ่าน |
| `templates/socialaccount/connections.html` | หน้ารายการ social account ที่เชื่อมอยู่ (ขั้นตอนที่ 355.3) |
| `templates/socialaccount/login.html` | หน้ายืนยันก่อนเริ่ม OAuth flow ("คุณกำลังจะไปที่ Google...") |
| `templates/socialaccount/signup.html` | หน้าให้กรอกข้อมูลเพิ่มเติมหลัง social login ครั้งแรก (ถ้า `SOCIALACCOUNT_AUTO_SIGNUP=False`) |
| `templates/socialaccount/snippets/provider_list.html` | ส่วนแสดงปุ่ม provider ทั้งหมด (ใช้ `{% include %}` ซ้ำได้หลายหน้า) |

### 356.3 ดูตัวอย่าง Template ต้นฉบับของ allauth เพื่อเป็นจุดเริ่มต้น

หาไฟล์ต้นฉบับที่ pip ติดตั้งไว้เพื่อดูโครงสร้าง context variable ที่มีให้ใช้:

```bash
python -c "import allauth; print(allauth.__path__[0])"
# แสดง path เช่น /path/to/venv/lib/python3.12/site-packages/allauth

# ดูไฟล์ template ต้นฉบับ
find $(python -c "import allauth; print(allauth.__path__[0])") -path "*templates/account/login.html"
```

### 356.4 Override `account/login.html` ให้ตรงดีไซน์เว็บ

```html
<!-- templates/account/login.html -->
{% extends "base.html" %}
{% load socialaccount %}

{% block title %}เข้าสู่ระบบ | My Django Blog{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>เข้าสู่ระบบ</h1>

    {% if form.non_field_errors %}
        <div class="alert alert-error">
            {% for error in form.non_field_errors %}
                <p>{{ error }}</p>
            {% endfor %}
        </div>
    {% endif %}

    <form method="post" action="{% url 'account_login' %}">
        {% csrf_token %}

        <div class="form-group">
            <label for="{{ form.login.id_for_label }}">อีเมล</label>
            {{ form.login }}
        </div>

        <div class="form-group">
            <label for="{{ form.password.id_for_label }}">รหัสผ่าน</label>
            {{ form.password }}
        </div>

        <div class="form-group form-check">
            {{ form.remember }}
            <label for="{{ form.remember.id_for_label }}">จำฉันไว้ในระบบ</label>
        </div>

        {% if redirect_field_value %}
            <input type="hidden" name="{{ redirect_field_name }}" value="{{ redirect_field_value }}">
        {% endif %}

        <button type="submit" class="btn btn-primary">เข้าสู่ระบบ</button>
    </form>

    <p><a href="{% url 'account_reset_password' %}">ลืมรหัสผ่าน?</a></p>

    <div class="divider"><span>หรือ</span></div>

    {% include "socialaccount/snippets/provider_list.html" with process="login" %}

    <p class="auth-footer">
        ยังไม่มีบัญชี? <a href="{% url 'account_signup' %}">สมัครสมาชิก</a>
    </p>
</div>
{% endblock %}
```

### 356.5 สร้าง snippet ปุ่ม provider ที่ใช้ซ้ำได้พร้อมไอคอน

```html
<!-- templates/socialaccount/snippets/provider_list.html -->
{% load socialaccount %}

<div class="social-login-buttons">
    {% get_providers as socialaccount_providers %}
    {% for provider in socialaccount_providers %}
        <a href="{% provider_login_url provider.id process=process %}"
           class="btn-social btn-social-{{ provider.id }}">
            {% if provider.id == "google" %}
                <svg class="icon-google" viewBox="0 0 24 24" width="20" height="20">
                    <path fill="currentColor" d="M12 11v2.4h6.7c-.3 1.6-2.1 4.6-6.7 4.6-4 0-7.3-3.3-7.3-7.4S8 3.2 12 3.2c2.3 0 3.8.9 4.7 1.8l3.2-3C17.8 0.3 15.2-.5 12-.5 5.7-.5.5 4.7.5 11S5.7 22.5 12 22.5c6.9 0 11.5-4.9 11.5-11.7 0-.8-.1-1.4-.2-2H12z"/>
                </svg>
            {% elif provider.id == "github" %}
                <svg class="icon-github" viewBox="0 0 24 24" width="20" height="20">
                    <path fill="currentColor" d="M12 .3a12 12 0 0 0-3.8 23.4c.6.1.8-.3.8-.6v-2c-3.3.7-4-1.6-4-1.6-.6-1.4-1.4-1.8-1.4-1.8-1.1-.8.1-.8.1-.8 1.2.1 1.9 1.3 1.9 1.3 1.1 1.8 2.8 1.3 3.5 1 .1-.8.4-1.3.7-1.6-2.7-.3-5.5-1.3-5.5-6a4.6 4.6 0 0 1 1.3-3.2 4.3 4.3 0 0 1 .1-3.2s1-.3 3.4 1.2a11.5 11.5 0 0 1 6 0C17.3 5 18.3 5.3 18.3 5.3a4.3 4.3 0 0 1 .1 3.2 4.6 4.6 0 0 1 1.3 3.2c0 4.7-2.8 5.7-5.5 6 .4.4.8 1.1.8 2.2v3.3c0 .3.2.7.8.6A12 12 0 0 0 12 .3"/>
                </svg>
            {% endif %}
            <span>เข้าสู่ระบบด้วย {{ provider.name }}</span>
        </a>
    {% endfor %}
</div>
```

การแยก snippet นี้ทำให้เรานำไปใช้ซ้ำได้ทั้งในหน้า `login.html`, `signup.html`,
และ `connections.html` โดยไม่ต้องเขียนโค้ด SVG ซ้ำ (ยึดหลัก DRY ที่เรียนไปตั้งแต่
Part 001 ขั้นตอนที่ 3.4)

### 356.6 Override `socialaccount/login.html` (หน้ายืนยันก่อนออกจากเว็บเรา)

ก่อน redirect ไปยัง Google/GitHub จริง ๆ allauth จะแสดงหน้ายืนยันสั้น ๆ ก่อน
(ป้องกันการ redirect โดยไม่ได้ตั้งใจผ่าน `GET` link ธรรมดา — เป็นหลักการเดียวกับ
Part 031 ขั้นตอนที่ 307.3 ที่บังคับ logout ผ่าน `POST`):

```html
<!-- templates/socialaccount/login.html -->
{% extends "base.html" %}

{% block content %}
<div class="auth-card">
    <h1>กำลังจะไปที่ {{ provider.name }}</h1>
    <p>คุณกำลังจะถูกนำไปยัง {{ provider.name }} เพื่อยืนยันตัวตน</p>

    <form method="post">
        {% csrf_token %}
        <button type="submit" class="btn btn-primary">ดำเนินการต่อ</button>
    </form>
</div>
{% endblock %}
```

### 356.7 ตารางสรุป context variable สำคัญที่แต่ละ template ได้รับ

| Template | Context variable สำคัญ |
|---|---|
| `account/login.html` | `form` (`AuthenticationForm` ของ allauth), `redirect_field_name`, `redirect_field_value` |
| `account/signup.html` | `form` (`SignupForm`) |
| `socialaccount/login.html` | `provider` (object ของ provider ที่กำลังเชื่อมต่อ) |
| `socialaccount/connections.html` | `form.accounts` (list ของ `SocialAccount` ที่เชื่อมอยู่) |
| `socialaccount/snippets/provider_list.html` | ต้องใช้ `{% get_providers %}` ดึงเองเสมอ ไม่ได้ inherit context จากหน้าที่ include |

---

## ขั้นตอนที่ 357: จัดการ Signal ของ allauth (`user_signed_up`, `social_account_added`) เพื่อสร้าง Profile อัตโนมัติ

### 357.1 ทบทวน Django Signals จาก Part 019

ทบทวนสั้น ๆ จาก Part 019: **Signal** คือกลไกให้โค้ดส่วนหนึ่ง "ประกาศ" ว่ามีเหตุการณ์
เกิดขึ้น (เช่น "มีผู้ใช้สมัครสมาชิกใหม่แล้ว") โดยไม่ต้องรู้ว่าใครจะมาฟังเหตุการณ์
นั้นบ้าง — allauth ส่ง signal ของตัวเองเพิ่มเติมจาก signal มาตรฐานของ Django
(`user_logged_in`, `user_logged_out` จาก Part 031 ขั้นตอนที่ 303.4/307.1)

### 357.2 ตารางสรุป Signal สำคัญของ allauth

| Signal | อยู่ในโมดูล | ส่งเมื่อไหร่ | ใช้ทำอะไรบ่อยที่สุด |
|---|---|---|---|
| `user_signed_up` | `allauth.account.signals` | ผู้ใช้สมัครสมาชิกสำเร็จ (ทั้ง local signup และ social signup ครั้งแรก) | สร้าง Profile, ส่ง welcome email, บันทึก analytics |
| `social_account_added` | `allauth.socialaccount.signals` | มีการเชื่อม social account ใหม่เข้ากับ user (ทั้งตอนสมัครครั้งแรกและตอน connect ทีหลังตามขั้นตอนที่ 355.3) | อัปเดต avatar, sync ข้อมูลจาก provider |
| `social_account_updated` | `allauth.socialaccount.signals` | login ผ่าน social account ที่เชื่อมอยู่แล้วซ้ำ (ข้อมูลจาก provider อาจเปลี่ยนแปลง) | อัปเดต avatar/ชื่อให้เป็นปัจจุบันเสมอ |
| `social_account_removed` | `allauth.socialaccount.signals` | ผู้ใช้ยกเลิกการเชื่อม social account (ขั้นตอนที่ 355.3) | ล้างข้อมูลที่ sync มาจาก provider นั้น |
| `email_confirmed` | `allauth.account.signals` | ผู้ใช้ยืนยันอีเมลสำเร็จ | ปลดล็อกฟีเจอร์ที่ต้องการอีเมล verified (จะเจาะลึกใน Part 035) |

### 357.3 สร้าง Model `Profile` เพื่อเก็บข้อมูลเสริมจาก OAuth provider

ทบทวนจาก Part 032 ขั้นตอนที่ 318.4: ต้องใช้ `settings.AUTH_USER_MODEL` เป็น string
ใน `models.py` เสมอ (โปรเจกต์นี้ตั้ง `AUTH_USER_MODEL = "accounts.CustomUser"`
ไว้แล้วตั้งแต่ Part 032):

```python
# accounts/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="profile",
    )
    avatar_url = models.URLField(blank=True)
    provider_bio = models.TextField(blank=True)
    signed_up_via = models.CharField(max_length=30, default="local")

    def __str__(self):
        return f"Profile ของ {self.user}"
```

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

### 357.4 เขียน Signal Receiver สร้าง Profile อัตโนมัติตอนสมัครสมาชิก

```python
# accounts/signals.py
from allauth.account.signals import user_signed_up
from allauth.socialaccount.signals import social_account_added, social_account_updated
from django.dispatch import receiver

from .models import Profile


@receiver(user_signed_up)
def create_profile_on_signup(request, user, **kwargs):
    """
    ทำงานทั้งกรณี signup ด้วย local email/password และกรณี social signup
    ครั้งแรก (allauth ส่ง signal นี้ให้ครอบคลุมทั้งสองเส้นทาง)
    """
    sociallogin = kwargs.get("sociallogin")
    signed_up_via = sociallogin.account.provider if sociallogin else "local"

    Profile.objects.update_or_create(
        user=user,
        defaults={"signed_up_via": signed_up_via},
    )


@receiver(social_account_added)
@receiver(social_account_updated)
def sync_profile_from_social_account(request, sociallogin, **kwargs):
    """
    ดึงรูปโปรไฟล์และข้อมูลจาก provider มาอัปเดต Profile ทุกครั้งที่
    เชื่อม social account ใหม่ หรือ login ซ้ำผ่าน social account เดิม
    """
    user = sociallogin.user
    extra_data = sociallogin.account.extra_data
    provider = sociallogin.account.provider

    if provider == "google":
        avatar_url = extra_data.get("picture", "")
        bio = extra_data.get("name", "")
    elif provider == "github":
        avatar_url = extra_data.get("avatar_url", "")
        bio = extra_data.get("bio", "") or ""
    else:
        avatar_url = ""
        bio = ""

    profile, _ = Profile.objects.get_or_create(user=user)
    updated_fields = []

    if avatar_url and profile.avatar_url != avatar_url:
        profile.avatar_url = avatar_url
        updated_fields.append("avatar_url")

    if bio and not profile.provider_bio:
        profile.provider_bio = bio
        updated_fields.append("provider_bio")

    if updated_fields:
        profile.save(update_fields=updated_fields)
```

### 357.5 เชื่อม signal receiver ผ่าน `apps.py` (ทบทวนหลักการจาก Part 019)

```python
# accounts/apps.py
from django.apps import AppConfig


class AccountsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "accounts"

    def ready(self):
        import accounts.signals  # noqa: F401  — import เพื่อลงทะเบียน @receiver เท่านั้น
```

> **ข้อควรระวังสำคัญ (ทบทวนจาก Part 019)**: ต้อง import module `signals.py`
> ใน `ready()` เท่านั้น **ห้าม import ที่ระดับบนสุดของ `models.py`** เพราะจะทำให้
> เกิด circular import หรือ `AppRegistryNotReady` ได้ง่ายมาก (หลักการเดียวกับที่
> อธิบายเรื่อง `get_user_model()` ใน Part 032 ขั้นตอนที่ 318.3)

### 357.6 ทดสอบ signal ทำงานจริง

```python
# python manage.py shell
from django.contrib.auth import get_user_model
from accounts.models import Profile

User = get_user_model()
user = User.objects.latest("date_joined")

profile = Profile.objects.get(user=user)
print(profile.signed_up_via)   # "google" ถ้าสมัครผ่าน Google
print(profile.avatar_url)      # URL รูปโปรไฟล์จาก Google
```

แสดงผลใน template:

```html
<!-- templates/base.html -->
{% if user.is_authenticated %}
    {% if user.profile.avatar_url %}
        <img src="{{ user.profile.avatar_url }}" alt="{{ user.username }}" class="avatar-small">
    {% endif %}
    <span>สวัสดี, {{ user.username }}!</span>
{% endif %}
```

### 357.7 ตารางสรุปว่า signal ไหนควรใช้ทำอะไร

| ต้องการทำอะไร | ควรใช้ signal ไหน |
|---|---|
| สร้าง Profile ครั้งแรกตอนสมัครสมาชิก (ไม่ว่าจะสมัครทางไหน) | `user_signed_up` |
| ดึง avatar/bio จาก provider มาอัปเดต Profile ทุกครั้งที่ login ผ่าน social | `social_account_updated` |
| แจ้งเตือนผู้ใช้ทาง email ว่า "มีการเชื่อมบัญชี Google ใหม่กับบัญชีคุณ" (ความปลอดภัย) | `social_account_added` |
| ล้าง avatar ที่เคย sync มา เมื่อผู้ใช้ยกเลิกเชื่อมบัญชี | `social_account_removed` |
| ปลดล็อกฟีเจอร์หลังยืนยันอีเมล | `email_confirmed` |

---

## ขั้นตอนที่ 358: ข้อควรระวังด้านความปลอดภัยของ OAuth

### 358.1 การ Validate Redirect URI: ทำไม Exact Match ถึงสำคัญ

ทบทวนจากขั้นตอนที่ 353.3: `redirect_uri` ต้องตรงกับที่ลงทะเบียนไว้ **แบบ exact
string match** เท่านั้น เหตุผลด้านความปลอดภัยคือถ้า authorization server ยอมรับ
`redirect_uri` แบบหลวม ๆ (เช่น ยอมรับทุก URL ที่ขึ้นต้นด้วยโดเมนที่ลงทะเบียนไว้)
ผู้โจมตีอาจใช้ช่องโหว่นี้ขโมย authorization code ได้:

```
สถานการณ์อันตรายถ้า redirect_uri ตรวจสอบแบบหลวม (prefix match แทน exact match):

1. แอปลงทะเบียน redirect_uri = https://myblog.com/accounts/google/login/callback/
2. authorization server ตรวจสอบแบบหลวม ยอมรับทุก URL ที่ขึ้นต้นด้วย
   https://myblog.com/... รวมถึง
   https://myblog.com/accounts/google/login/callback/../../evil-page/
3. ผู้โจมตีหลอกเหยื่อให้คลิกลิงก์ authorization request ที่ตั้ง
   redirect_uri เป็น URL หลอกนี้
4. authorization code รั่วไหลไปที่หน้าเว็บของผู้โจมตี (ผ่าน query
   string หรือ open redirect ที่ซ่อนอยู่ในเว็บ)
5. ผู้โจมตีนำ code ไปแลก token แทนเหยื่อได้ (ถ้ารู้ client_secret
   ด้วย หรือถ้าเป็น public client ที่ไม่ใช้ client_secret)
```

Google, GitHub, และ authorization server มาตรฐานทุกเจ้าในปัจจุบันบังคับ
**exact match** อยู่แล้ว (นี่คือเหตุผลที่ Google ปฏิเสธ error
`redirect_uri_mismatch` เมื่อ URL ไม่ตรงเป๊ะแม้จะต่างกันแค่ trailing slash)
สิ่งที่นักพัฒนาต้องรับผิดชอบเองคือ **ลงทะเบียนเฉพาะ redirect URI ที่จำเป็นจริง ๆ**
ไม่เผื่อ URL ที่ไม่ได้ใช้งาน และตรวจสอบให้แน่ใจว่าไม่มี open redirect ปรากฏอยู่ที่
path ใกล้เคียงกับ callback URL ของระบบ

### 358.2 `state` Parameter: เกราะป้องกัน CSRF ของ OAuth Flow

ย้อนกลับไปดูขั้นตอนที่ 8 ใน diagram ของขั้นตอนที่ 351.3 — allauth ตรวจสอบ `state`
ทุกครั้งที่ได้รับ callback กลับมา กลไกนี้ป้องกันการโจมตีที่เรียกว่า **CSRF ใน
OAuth Flow** (บางครั้งเรียก **Login CSRF**):

```
สถานการณ์อันตรายถ้าไม่มี state parameter:

1. ผู้โจมตีเริ่ม OAuth flow ด้วยบัญชี Google ของตัวเอง จนได้
   authorization code ของตัวเองมา (แต่ยังไม่ใช้ทันที)
2. ผู้โจมตีสร้างลิงก์ปลอมที่ชี้ไปยัง callback URL ของเว็บเป้าหมาย
   พร้อมแนบ code ของตัวเองเข้าไป:
   https://victim-site.com/accounts/google/login/callback/?code=ATTACKER_CODE
3. หลอกให้เหยื่อ (ที่ login เว็บ victim-site.com อยู่แล้วด้วยบัญชี
   อื่น) คลิกลิงก์นี้
4. ถ้าเว็บไม่ตรวจสอบ state เลย ระบบจะ "เชื่อม" บัญชี Google ของ
   ผู้โจมตี เข้ากับ session ปัจจุบันของเหยื่อ (เพราะ code ของ
   ผู้โจมตีถูกแลกเป็น token สำเร็จ)
5. ผู้โจมตีสามารถ login เข้าบัญชีของเหยื่อได้ทีหลัง โดยใช้บัญชี
   Google ของตัวเองที่ถูกเชื่อมเข้าไปแล้ว!
```

`state` ป้องกันการโจมตีนี้เพราะ authorization request ที่แท้จริงของเหยื่อ
(ที่เกิดตอนเหยื่อกดปุ่ม login เอง) จะสร้าง `state` แบบสุ่มเก็บไว้ใน **session
ของเหยื่อเอง** — เมื่อ callback ที่ผู้โจมตีปลอมขึ้นมาไม่มี `state` ที่ตรงกับใน
session ของเหยื่อ (เพราะผู้โจมตีไม่มีทางรู้ค่า `state` ที่ session ของเหยื่อสร้าง
ไว้) allauth จะปฏิเสธ callback นั้นทันทีด้วย error

```python
# django/contrib/auth/... (แนวคิดการตรวจสอบ state ของ allauth ย่อเพื่อความเข้าใจ)
def process_callback(request):
    state_from_provider = request.GET.get("state")
    state_from_session = request.session.pop("oauth_state", None)

    if not state_from_session or state_from_provider != state_from_session:
        raise PermissionDenied("Invalid OAuth state — อาจเป็นความพยายามโจมตีแบบ CSRF")

    # ดำเนินการแลก code เป็น token ต่อ เฉพาะเมื่อ state ตรงกันเท่านั้น
    ...
}
```

allauth จัดการเรื่องนี้ให้อัตโนมัติทั้งหมดโดยที่คุณไม่ต้องเขียนโค้ดตรวจสอบเอง
แม้แต่บรรทัดเดียว **ตราบใดที่คุณใช้ URL และ view ของ allauth ตามมาตรฐาน**
(ไม่ได้เขียน OAuth flow เองแบบขั้นตอนที่ 351 ทำมือ) — นี่คือเหตุผลสำคัญข้อหนึ่งที่
ควรใช้ library ที่ผ่านการตรวจสอบความปลอดภัยมาแล้วอย่าง django-allauth แทนการ
implement OAuth2 เองตั้งแต่ต้น

### 358.3 การจัดการ `client_secret` อย่างปลอดภัย

| กฎ | เหตุผล |
|---|---|
| ห้าม commit `client_secret` ลง Git เด็ดขาด | ทบทวนจากขั้นตอนที่ 353.4 — ใช้ `.env` + `.gitignore` เสมอ |
| ใช้ secret manager ในสภาพแวดล้อม production (AWS Secrets Manager, HashiCorp Vault, ฯลฯ) แทนไฟล์ `.env` ธรรมดา | ไฟล์ `.env` บน production server ยังมีความเสี่ยงถ้า server ถูกเจาะ |
| หมุนเวียน (rotate) `client_secret` เป็นระยะ โดยเฉพาะถ้าสงสัยว่ารั่วไหล | ลด attack window หากมีการรั่วไหลที่ยังไม่รู้ตัว |
| ใช้ credential คนละชุดสำหรับ development/staging/production เสมอ | ถ้า credential ของ dev รั่วไหล จะไม่กระทบ production |
| จำกัดสิทธิ์ (scope) ที่ขอให้น้อยที่สุดเท่าที่จำเป็น | ลดความเสียหายหาก token รั่วไหล (Principle of Least Privilege) |

### 358.4 อย่าเชื่อถือ Email จาก Provider โดยไม่เช็ค Verified Flag

ย้ำอีกครั้งจากขั้นตอนที่ 355.4: **ห้ามใช้ email จาก social login เพื่อ auto-link
บัญชีหรือให้สิทธิ์พิเศษ โดยไม่เช็ค flag verified ก่อนเด็ดขาด** เพราะ provider
บางเจ้า (โดยเฉพาะ GitHub) ยอมให้ผู้ใช้กรอกอีเมลอะไรก็ได้ตอนสมัครโดยยังไม่ต้อง
ยืนยัน — ถือเป็นกฎความปลอดภัยที่สำคัญที่สุดข้อหนึ่งของ Social Authentication

### 358.5 ความเสี่ยงของการเก็บ Access/Refresh Token (`SOCIALACCOUNT_STORE_TOKENS`)

ถ้าตั้งค่า `SOCIALACCOUNT_STORE_TOKENS = True` (เปิดเมื่อต้องเรียก Google/GitHub
API ต่อในภายหลัง เช่น ดึงรายชื่อ repository ของผู้ใช้จาก GitHub API) ต้องตระหนักว่า
`access_token`/`refresh_token` ที่เก็บในตาราง `SocialToken` **มีค่าเทียบเท่ากับ
กุญแจเข้าถึงบัญชี Google/GitHub ของผู้ใช้บางส่วน** ตามขอบเขต scope ที่ขอไว้

| แนวทางป้องกัน | รายละเอียด |
|---|---|
| เปิด `SOCIALACCOUNT_STORE_TOKENS` เฉพาะเมื่อจำเป็นจริง ๆ เท่านั้น | ถ้าแค่ต้องการ authenticate อย่างเดียว ไม่ต้องเก็บ token เลย (ค่า default `False` ปลอดภัยกว่า) |
| เข้ารหัส (encrypt) ค่า token ก่อนบันทึกลงฐานข้อมูล | ใช้ library เช่น `django-fernet-fields` หรือเข้ารหัสเองด้วย `cryptography` |
| จำกัดสิทธิ์เข้าถึงตาราง `SocialToken` ใน Django Admin เฉพาะ superuser | ป้องกัน staff ทั่วไปเห็น token ของผู้ใช้คนอื่น |
| ตั้งค่า token ให้หมดอายุและ refresh ตามรอบที่เหมาะสม | ลดความเสียหายหาก token รั่วไหล |

### 358.6 HTTPS บังคับใน Production

OAuth2 flow ทั้งหมด (โดยเฉพาะขั้นตอนแลก code เป็น token) **ต้องทำผ่าน HTTPS
เท่านั้นใน production** Google และ GitHub จะปฏิเสธการลงทะเบียน `redirect_uri`
แบบ `http://` สำหรับโดเมนจริง (อนุญาตเฉพาะ `http://127.0.0.1` และ `http://
localhost` สำหรับ development เท่านั้น) ตั้งค่า Django ให้บังคับ HTTPS ด้วย:

```python
# config/settings/production.py
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

(จะเจาะลึกการแยก settings ตาม environment และ security middleware เต็มรูปแบบใน
Phase DevOps)

### 358.7 ตารางสรุป Security Checklist สำหรับ Social Authentication

| ✅ | รายการตรวจสอบ |
|---|---|
| ☐ | `redirect_uri` ที่ลงทะเบียนตรงกับ URL จริงแบบ exact match ไม่มี URL ที่ไม่ได้ใช้ค้างอยู่ |
| ☐ | ใช้ allauth (หรือ library ที่ผ่านการตรวจสอบแล้ว) จัดการ `state` parameter แทนการเขียน OAuth เอง |
| ☐ | `client_secret` เก็บใน environment variable/secret manager ไม่ commit ขึ้น Git |
| ☐ | Auto-link บัญชีเฉพาะเมื่ออีเมล verified โดย provider เท่านั้น |
| ☐ | เปิด `SOCIALACCOUNT_STORE_TOKENS` เฉพาะเมื่อจำเป็น และเข้ารหัสถ้าเก็บ |
| ☐ | บังคับ HTTPS ทุก endpoint ที่เกี่ยวข้องกับ OAuth ใน production |
| ☐ | ขอ scope เท่าที่จำเป็นเท่านั้น ตรวจสอบ scope ที่ขอเป็นระยะ |
| ☐ | มี log การเชื่อม/ยกเลิกเชื่อม social account เพื่อ audit ย้อนหลังได้ |

---

## ขั้นตอนที่ 359: ทางเลือกอื่น — python-social-auth/social-auth-app-django เปรียบเทียบสั้น ๆ กับ django-allauth

### 359.1 ประวัติโดยย่อ: จาก python-social-auth สู่ social-auth-app-django

**python-social-auth** เป็น library ยอดนิยมในอดีตสำหรับทำ social login รองรับ
หลาย framework (Django, Flask, Pyramid) ไม่ใช่แค่ Django เท่านั้น แต่ตัวโปรเจกต์
เดิม**หยุดพัฒนาไปแล้ว** ปัจจุบันถูกแยกและสืบทอดต่อเป็น:

- **`social-auth-core`**: โมดูล core ที่ไม่ผูกกับ framework ใดโดยเฉพาะ
- **`social-auth-app-django`**: ส่วนขยายสำหรับ Django โดยเฉพาะ (ยังคง maintain
  อยู่จนถึงปัจจุบัน แม้ความถี่ในการอัปเดตจะน้อยกว่า django-allauth มาก)

### 359.2 ตัวอย่างการตั้งค่า `social-auth-app-django` (เพื่อเปรียบเทียบเท่านั้น)

```bash
pip install social-auth-app-django
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "social_django",
]

AUTHENTICATION_BACKENDS = [
    "social_core.backends.google.GoogleOAuth2",
    "social_core.backends.github.GithubOAuth2",
    "django.contrib.auth.backends.ModelBackend",
]

SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = env("GOOGLE_OAUTH_CLIENT_ID")
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = env("GOOGLE_OAUTH_CLIENT_SECRET")

SOCIAL_AUTH_GITHUB_KEY = env("GITHUB_OAUTH_CLIENT_ID")
SOCIAL_AUTH_GITHUB_SECRET = env("GITHUB_OAUTH_CLIENT_SECRET")
```

```python
# config/urls.py
urlpatterns = [
    path("oauth/", include("social_django.urls", namespace="social")),
]
```

สังเกตว่าแนวคิดโดยรวมคล้ายกับ django-allauth มาก (ตั้งค่า credential ผ่าน
settings, เพิ่ม authentication backend, include urls) เพราะทั้งคู่แก้ปัญหา
เดียวกัน แต่ปรัชญาการออกแบบต่างกันตรงที่ `social-auth-app-django` **ไม่มีระบบ
local account (username/password) มาให้ในตัว** ต้องใช้คู่กับ
`django.contrib.auth` เดิมหรือระบบ signup ของตัวเองเสมอ ต่างจาก allauth ที่ผูก
ทั้ง local account และ social account ไว้ในระบบเดียวกันหมด

### 359.3 ตารางเปรียบเทียบ django-allauth vs social-auth-app-django

| ประเด็น | django-allauth | social-auth-app-django |
|---|---|---|
| ระบบ local login/signup ในตัว | ✅ มีครบ (`allauth.account`) | ❌ ไม่มี ต้องพึ่ง `django.contrib.auth` เดิม |
| จำนวน provider ที่รองรับสำเร็จรูป | มากกว่า 80 provider | มากกว่า 50 provider (ครอบคลุมเจ้าใหญ่ ๆ ครบเช่นกัน) |
| Email verification workflow | ✅ มีครบวงจร | ❌ ไม่มี ต้องเขียนเอง |
| ความถี่ในการอัปเดต (ปี 2025-2026) | สูง ยังพัฒนาต่อเนื่อง | ต่ำกว่า อัปเดตเป็นครั้งคราว |
| Account linking (ขั้นตอนที่ 355) | มีกลไกในตัวผ่าน adapter | ต้องเขียน pipeline เอง (ยืดหยุ่นแต่ซับซ้อนกว่า) |
| ความซับซ้อนในการเริ่มต้น | ปานกลาง (ตั้งค่าหลายจุดแต่มีเอกสารครบ) | ปานกลาง-สูง (ต้องเข้าใจ concept "pipeline" ของตัวเอง) |
| เหมาะกับ | โปรเจกต์ Django ใหม่ที่ต้องการทั้ง local + social auth ครบวงจร | โปรเจกต์ที่มีระบบ auth เดิมอยู่แล้วและต้องการแค่เพิ่ม social login แบบ minimal |

### 359.4 ทางเลือกระดับต่ำกว่า: เขียนเองด้วย `requests-oauthlib` หรือ `Authlib`

สำหรับกรณีพิเศษที่ต้องการควบคุม OAuth flow เองแบบละเอียดที่สุด (เช่น provider
ที่ allauth ยังไม่รองรับ หรือต้องการ custom flow ที่ผิดมาตรฐานมาก) มีทางเลือก
ระดับต่ำกว่าคือเขียนเองด้วย library ที่จัดการเฉพาะ OAuth2 protocol โดยตรง:

```python
# ตัวอย่างคร่าว ๆ ด้วย Authlib (เพื่อความเข้าใจเท่านั้น ไม่แนะนำสำหรับ production
# ถ้า allauth รองรับ provider นั้นอยู่แล้ว)
from authlib.integrations.django_client import OAuth

oauth = OAuth()
oauth.register(
    name="google",
    client_id=settings.GOOGLE_OAUTH_CLIENT_ID,
    client_secret=settings.GOOGLE_OAUTH_CLIENT_SECRET,
    server_metadata_url="https://accounts.google.com/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email profile"},
)
```

| ทางเลือก | เหมาะกับ |
|---|---|
| django-allauth | งานส่วนใหญ่ 95%+ ของโปรเจกต์ Django ที่ต้องการ social login |
| social-auth-app-django | โปรเจกต์ที่มี auth เดิมซับซ้อนอยู่แล้วและต้องการ minimal addition |
| Authlib / requests-oauthlib | provider แปลกที่ไม่มีใครรองรับ หรือต้องการควบคุม OAuth flow ระดับ protocol เอง |
| เขียน OAuth2 client เองทั้งหมด | **ไม่แนะนำ** เว้นแต่กำลังศึกษาเพื่อความเข้าใจ (ตามขั้นตอนที่ 351) เพราะเสี่ยงพลาดรายละเอียดความปลอดภัยที่ library สำเร็จรูปจัดการให้แล้ว |

หลักสูตรนี้ยึด **django-allauth เป็นมาตรฐานหลัก** ตลอดทั้งหลักสูตร เพราะครอบคลุม
ทั้ง local และ social authentication ในระบบเดียวกัน มีเอกสารและชุมชนสนับสนุนที่
active ที่สุดในกลุ่ม Django ecosystem

---

## ขั้นตอนที่ 360: สรุปและแบบฝึกหัด — implement Google login จริงสำหรับ blog project

### 360.1 ประกอบทุกอย่างเข้าด้วยกัน: Checklist การตั้งค่าแบบสมบูรณ์

```python
# config/settings.py — สรุปการตั้งค่าทั้งหมดของ Part นี้ในที่เดียว
import environ

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django.contrib.sites",

    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",

    "accounts",
    "blog",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "allauth.account.middleware.AccountMiddleware",
]

SITE_ID = 1

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

LOGIN_REDIRECT_URL = "blog:post-list"
LOGOUT_REDIRECT_URL = "blog:post-list"
ACCOUNT_LOGOUT_ON_GET = False

ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "username*", "password1*", "password2*"]
ACCOUNT_EMAIL_VERIFICATION = "optional"
ACCOUNT_USERNAME_MIN_LENGTH = 4

SOCIALACCOUNT_AUTO_SIGNUP = True
SOCIALACCOUNT_STORE_TOKENS = False
SOCIALACCOUNT_ADAPTER = "accounts.adapters.SocialAccountAdapter"

SOCIALACCOUNT_PROVIDERS = {
    "github": {
        "SCOPE": ["user:email", "read:user"],
    },
}

GOOGLE_OAUTH_CLIENT_ID = env("GOOGLE_OAUTH_CLIENT_ID")
GOOGLE_OAUTH_CLIENT_SECRET = env("GOOGLE_OAUTH_CLIENT_SECRET")
GITHUB_OAUTH_CLIENT_ID = env("GITHUB_OAUTH_CLIENT_ID")
GITHUB_OAUTH_CLIENT_SECRET = env("GITHUB_OAUTH_CLIENT_SECRET")
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("accounts/", include("allauth.urls")),
    path("blog/", include("blog.urls")),
]
```

### 360.2 ขั้นตอนติดตั้งจริงแบบเรียงลำดับ (Runbook)

```bash
# 1. ติดตั้ง package
pip install django-allauth django-environ
pip freeze > requirements.txt

# 2. สร้าง OAuth credentials จริงตามขั้นตอนที่ 353.1-353.3 (Google)
#    และขั้นตอนที่ 354.1 (GitHub) แล้วบันทึกลง .env

# 3. รัน migration
python manage.py migrate

# 4. ตั้งค่า Site ให้ตรงกับ domain
python manage.py shell -c "
from django.contrib.sites.models import Site
site = Site.objects.get(pk=1)
site.domain = '127.0.0.1:8000'
site.name = 'My Django Blog (dev)'
site.save()
"

# 5. สร้าง SocialApp จาก environment variables
python manage.py setup_oauth_providers

# 6. รันเซิร์ฟเวอร์และทดสอบ
python manage.py runserver
```

### 360.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจ **OAuth2 Authorization Code Flow** ทีละขั้นตอน ตั้งแต่การ redirect
  ไปยัง Authorization Server จนถึงการแลก `code` เป็น `access_token` และเหตุผล
  ที่ต้องมี `code` มาคั่นกลางแทนการส่ง token ตรง ๆ
- แยกแยะ **OAuth2** (authorization) กับ **OpenID Connect** (authentication)
  ได้ และรู้ว่า Google รองรับ OIDC เต็มรูปแบบขณะที่ GitHub ใช้ OAuth2 ล้วน ๆ
- ติดตั้งและตั้งค่า **django-allauth** ครบทุกจุด (`INSTALLED_APPS`,
  `MIDDLEWARE`, `SITE_ID`, `AUTHENTICATION_BACKENDS`, `urls.py`)
- สร้าง OAuth credentials จริงกับ **Google Cloud Console** และ **GitHub
  Developer Settings** พร้อมตั้งค่า `SocialApp` ทั้งผ่าน Admin และผ่านโค้ด
- จัดการ **Account Linking** อย่างปลอดภัย โดยเชื่อมเฉพาะอีเมลที่ verified
  เท่านั้น ป้องกัน Account Takeover ผ่าน Email Confusion Attack
- ปรับแต่ง **Template** ของ allauth ให้ตรงดีไซน์เว็บ ทั้งหน้า login, signup,
  และปุ่ม social login พร้อมไอคอน
- ใช้ **Signal** ของ allauth (`user_signed_up`, `social_account_added`,
  `social_account_updated`) สร้างและอัปเดต Profile อัตโนมัติ
- เข้าใจความเสี่ยงด้านความปลอดภัยของ OAuth ทั้ง **redirect URI validation**,
  **`state` parameter** ป้องกัน CSRF, และการจัดการ `client_secret`/token
  อย่างปลอดภัย
- รู้จักทางเลือกอื่นในระบบนิเวศ Python (`social-auth-app-django`, `Authlib`)
  และเหตุผลที่หลักสูตรนี้เลือกใช้ django-allauth เป็นมาตรฐานหลัก

### 360.4 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบาย Authorization Code Flow ครบทั้ง 13 ขั้นตอนตาม diagram ในขั้นตอนที่ 351.3 ได้ด้วยคำพูดตัวเอง
- [ ] ติดตั้ง django-allauth และตั้งค่า `INSTALLED_APPS`/`MIDDLEWARE`/`SITE_ID` ได้ถูกต้องโดยไม่เปิดเอกสาร
- [ ] สร้าง Google OAuth credentials จริงและ login ผ่าน Google สำเร็จ
- [ ] สร้าง GitHub OAuth App จริงและ login ผ่าน GitHub สำเร็จ
- [ ] อธิบายได้ว่าทำไมการ auto-link บัญชีต้องเช็ค verified email ก่อนเสมอ
- [ ] Override template `account/login.html` ให้แสดงปุ่ม social login ตามดีไซน์ของตัวเองได้
- [ ] เขียน signal receiver สร้าง Profile อัตโนมัติเมื่อมีผู้ใช้สมัครใหม่ได้
- [ ] อธิบายบทบาทของ `state` parameter ในการป้องกัน CSRF ของ OAuth flow ได้
- [ ] ทำระบบ "Sign in with Google" ที่ทำงานได้จริงสำหรับ blog project สำเร็จ

### 360.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ทำตาม Runbook ในขั้นตอนที่ 360.2 ให้ครบทุกขั้นตอนกับโปรเจกต์
blog ของคุณเอง สร้าง Google OAuth credentials จริง ตั้งค่า `SocialApp` และ
ทดสอบ login ผ่าน Google จนสำเร็จ พร้อมถ่ายภาพหน้าจอ (screenshot) ทุกขั้นตอนสำคัญ
เก็บไว้เป็นหลักฐาน (consent screen, credentials page, หน้า login ที่มีปุ่ม
Google ปรากฏ, และหน้าที่ login สำเร็จแล้ว)

**แบบฝึกหัดที่ 2**: เพิ่ม GitHub Login เข้าไปในโปรเจกต์เดียวกัน (ตามขั้นตอนที่
354) แล้วทดสอบสถานการณ์ **Account Linking** ด้วยตัวเอง: (1) สมัครสมาชิกด้วย
email/password ก่อน โดยใช้อีเมลเดียวกับที่ผูกกับบัญชี GitHub ของคุณ (2) logout
(3) ลอง login ด้วย GitHub แล้วสังเกตพฤติกรรมของระบบ (4) เขียนสรุปว่าเกิดอะไรขึ้น
และตรงกับที่อธิบายในขั้นตอนที่ 355.2 หรือไม่

**แบบฝึกหัดที่ 3**: ปรับแต่ง custom adapter จากขั้นตอนที่ 355.4 ให้บันทึก log
ทุกครั้งที่มีการ auto-link บัญชีสำเร็จ (ใช้ `logging` module มาตรฐานของ Python)
โดยบันทึกข้อมูล username, provider, และเวลาที่เชื่อม แล้วทดสอบว่า log ปรากฏขึ้น
จริงเมื่อมีการ auto-link เกิดขึ้น

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ขยาย Model `Profile` จากขั้นตอนที่ 357.3 ให้มี
field `github_username` (ดึงจาก `extra_data["login"]` ของ GitHub) แล้วสร้างหน้า
`ProfileDetailView` ที่แสดงลิงก์ไปยังหน้า GitHub profile ของผู้ใช้โดยอัตโนมัติ
(`https://github.com/<github_username>`) เฉพาะกรณีที่ผู้ใช้เชื่อม GitHub account
ไว้เท่านั้น (ถ้าไม่ได้เชื่อม ไม่ต้องแสดงลิงก์นี้)

### 360.6 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม login ผ่าน Google แล้วเจอ error `redirect_uri_mismatch`?**
A: สาเหตุที่พบบ่อยที่สุดคือ URL ที่ลงทะเบียนใน Google Cloud Console ไม่ตรงกับ
callback URL จริงของ allauth แบบเป๊ะ ๆ (ทบทวนขั้นตอนที่ 353.3) จุดที่มักพลาด
คือลืมใส่ `/` ปิดท้าย หรือใช้ `http://localhost:8000` ในโค้ดแต่ลงทะเบียนไว้เป็น
`http://127.0.0.1:8000` ใน Console (สอง domain นี้ Google มองว่าต่างกัน แม้จะ
ชี้ไปที่เครื่องเดียวกันก็ตาม) ให้ตรวจสอบว่า `Site.domain` ที่ตั้งไว้ในขั้นตอนที่
352.5 ตรงกับ URL ที่คุณพิมพ์ในเบราว์เซอร์เป๊ะ ๆ

**Q: ทำไมปุ่ม Google/GitHub ไม่ปรากฏในหน้า login เลย?**
A: มักเกิดจาก 3 สาเหตุ: (1) ยังไม่ได้สร้าง `SocialApp` เลย (2) สร้างแล้วแต่ลืม
เพิ่ม site ที่ถูกต้องเข้าไปในช่อง "Chosen sites" ตอนสร้าง `SocialApp` (ทบทวน
ขั้นตอนที่ 353.5) หรือ (3) `SITE_ID` ใน settings ไม่ตรงกับ site ที่ผูกกับ
`SocialApp` นั้น ตรวจสอบด้วย `python manage.py shell` แล้ว query
`SocialApp.objects.all().values("provider", "sites__domain")` เพื่อดูว่า
ตรงกันหรือไม่

**Q: ใช้ django-allauth แล้วยังต้องเรียน `django.contrib.auth` จาก Part 031
อยู่ไหม?**
A: จำเป็นมาก เพราะ `allauth.account.auth_backends.AuthenticationBackend`
ทำงานอยู่**บนพื้นฐาน** ของ `django.contrib.auth` ทั้งหมด (ใช้ `User` model,
`Permission`, `Group`, session-based auth เดียวกัน) allauth เป็นเพียงชั้น
เสริมที่เพิ่มความสามารถ ไม่ได้แทนที่ระบบเดิมทั้งหมด ความเข้าใจเรื่อง
`request.user`, `AnonymousUser`, `LoginRequiredMixin` จาก Part 031 ยังคงใช้ได้
เหมือนเดิมทุกประการ

**Q: ถ้าอยากรองรับ Facebook Login หรือ Apple Sign In เพิ่ม ต้องทำอย่างไร?**
A: หลักการเหมือนกันทุกประการกับ Google/GitHub ที่เรียนไปใน Part นี้: (1) เพิ่ม
`allauth.socialaccount.providers.facebook` (หรือ `apple`) ใน `INSTALLED_APPS`
(2) สร้าง OAuth App ที่ Facebook Developer Portal/Apple Developer Portal
(3) ตั้งค่า `SocialApp` ผ่าน Admin หรือโค้ดเหมือนขั้นตอนที่ 353.5-354.2 — allauth
รองรับ provider มากกว่า 80 ตัว ดูรายชื่อทั้งหมดพร้อมคู่มือเฉพาะแต่ละตัวได้ที่
เอกสารทางการ https://docs.allauth.org/en/latest/socialaccount/providers/

**Q: SocialApp ควรตั้งค่าผ่าน Django Admin หรือผ่าน `SOCIALACCOUNT_PROVIDERS`
ใน settings ดี?**
A: ทั้งสองแบบใช้งานได้จริงตามที่เปรียบเทียบในขั้นตอนที่ 353.6 หลักสูตรนี้แนะนำ
แนวทาง `SocialApp` ผ่าน ORM/Admin เป็นค่าเริ่มต้น เพราะยืดหยุ่นกว่าเมื่อต้อง
จัดการหลาย site หรือเปลี่ยน credential โดยไม่ต้อง deploy ใหม่ แต่ถ้าโปรเจกต์ของ
คุณใช้แนวทาง infrastructure-as-code ที่ต้องการให้ทุกอย่างมาจาก environment
variable ล้วน ๆ (12-factor app) การใช้ `SOCIALACCOUNT_PROVIDERS["APPS"]` ก็เป็น
ทางเลือกที่สมเหตุสมผลไม่แพ้กัน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 037: Two-Factor Authentication** จะพาคุณไปเสริมความปลอดภัยอีกชั้นให้กับ
ระบบ authentication ที่สร้างเสร็จแล้วทั้ง local login (Part 031) และ social
login (Part นี้) คุณจะได้เรียนรู้หลักการ **TOTP (Time-based One-Time Password)**
ที่แอปอย่าง Google Authenticator ใช้งาน วิธีสร้าง QR code สำหรับผูกอุปกรณ์ การใช้
package `django-otp` ร่วมกับ django-allauth (ซึ่งรองรับ MFA มาให้ในตัวผ่าน
`allauth.mfa` เช่นกัน) การจัดการ **Backup Codes** เผื่อผู้ใช้ทำอุปกรณ์หาย และ
วิธีบังคับให้ staff/superuser ต้องเปิดใช้ 2FA เสมอเพื่อความปลอดภัยของระบบ Admin

เตรียมโทรศัพท์มือถือของคุณไว้ให้พร้อม (ติดตั้งแอป Google Authenticator หรือ
Authy ล่วงหน้า) เพราะ Part ถัดไปจะให้คุณทดสอบ 2FA จริงด้วยอุปกรณ์ของตัวเอง!
