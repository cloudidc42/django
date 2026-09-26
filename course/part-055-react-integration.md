# Part 055: Django กับ React (Django เป็น API Backend)

> **ขั้นตอนที่ 541-550 ของหลักสูตร** | Phase 6: Frontend Integration
>
> เป้าหมายของ Part นี้: ก้าวข้ามการ "โรย JavaScript" แบบ Part 052-054 ไปสู่การสร้าง
> **Single Page Application (SPA)** เต็มรูปแบบด้วย React โดยให้ Django ทำหน้าที่เป็น
> **API Backend ล้วน ๆ** ผ่าน Django REST Framework ที่คุณสร้างไว้ตลอด Phase 5
> (โดยเฉพาะ Blog API ด้วย ViewSets/Router จาก Part 044 และ JWT Authentication จาก
> Part 046) คุณจะตั้งค่า CORS ด้วย `django-cors-headers`, สร้างโปรเจกต์ React ด้วย
> Vite, เขียน JWT Authentication Flow ฝั่ง React ที่ refresh token อัตโนมัติ, ทำ
> navigation ด้วย React Router, เข้าใจสองแนวทางการ deploy (single deployment ผ่าน
> Django static files เทียบกับแยก host), จัดการ state ด้วย Context API, เข้าใจ
> CSRF/CORS สำหรับ SPA อย่างถ่องแท้ และตั้ง workflow การพัฒนาที่รัน Django กับ React
> dev server พร้อมกัน เมื่อจบ Part นี้คุณจะมี React frontend ที่สมบูรณ์สำหรับ Blog API
> ครบทั้งหน้า list, detail, login, และ create post

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 541: สถาปัตยกรรม Django REST API + React SPA แยกกัน เทียบกับ Server-rendered
- ขั้นตอนที่ 542: ตั้งค่า CORS ด้วย `django-cors-headers` เพื่อให้ React เรียก API ได้
- ขั้นตอนที่ 543: สร้างโปรเจกต์ React ด้วย Vite และดึงข้อมูลจาก Blog API มาแสดง
- ขั้นตอนที่ 544: JWT Authentication Flow ใน React (เก็บ/ใช้ token, refresh อัตโนมัติ)
- ขั้นตอนที่ 545: React Router สำหรับ navigation ใน SPA (list, detail, login)
- ขั้นตอนที่ 546: Deploy React build ผ่าน Django static files เทียบกับแยก host
- ขั้นตอนที่ 547: State Management เบื้องต้นด้วย React Context API
- ขั้นตอนที่ 548: ข้อควรพิจารณาด้าน CSRF สำหรับ SPA — Session-based vs JWT-based + CORS
- ขั้นตอนที่ 549: Workflow การพัฒนา — รัน Django + React dev server พร้อมกัน (Vite proxy)
- ขั้นตอนที่ 550: สรุปและแบบฝึกหัด — สร้าง React frontend ที่สมบูรณ์สำหรับ Blog API

---

## ขั้นตอนที่ 541: สถาปัตยกรรม Django REST API + React SPA แยกกัน เทียบกับ Server-rendered

### 541.1 ทบทวนเส้นทางที่พาเรามาถึงจุดนี้

ตลอด Phase 6 คุณได้เรียนรู้วิธี "เสริม" ฝั่ง Frontend เข้ากับ Django Template แบบ
server-rendered เป็นลำดับขั้น:

- **Part 051**: Bootstrap/CSS Framework — จัดหน้าตาให้สวยงาม ยังคงเป็น server-rendered
- **Part 052**: JavaScript + Fetch API — เริ่มยิง request แบบ async จาก Template เดิม
- **Part 053**: HTMX — ให้ Django ส่ง HTML fragment กลับมาแทน JSON (server-driven UI)
- **Part 054**: Alpine.js — เพิ่ม reactivity เล็ก ๆ น้อย ๆ ที่ฝั่ง client โดยไม่มี build step

ทุก Part ข้างต้นมีจุดร่วมกัน: **Django ยังคงเป็นผู้ render HTML** ไม่ว่าจะ render
ทั้งหน้าแบบ Part 001-050 หรือ render แค่บางส่วนแบบ HTMX ของ Part 053 ก็ตาม

Part นี้ตั้งคำถามที่ต่างออกไปโดยสิ้นเชิง: **ถ้า Django ไม่ต้อง render HTML เลยแม้แต่
บรรทัดเดียว** และปล่อยให้ Browser ฝั่ง client รัน JavaScript framework เต็มรูปแบบ
เพื่อสร้าง HTML ทั้งหมดเองล่ะ? นี่คือแนวคิดของ **Single Page Application (SPA)**

### 541.2 เปรียบเทียบสถาปัตยกรรมทั้งสองแบบ

```
┌─────────────────────────── แบบ Server-Rendered (Part 001-054) ───────────────────────────┐
│                                                                                             │
│   Browser                              Django Server                                       │
│   ┌──────────┐   GET /posts/           ┌─────────────────────────┐                        │
│   │          │ ──────────────────────> │ urls.py -> views.py     │                        │
│   │  แสดงผล   │                         │   -> models.py (Query)  │                        │
│   │  HTML     │ <────────────────────── │   -> templates/*.html   │                        │
│   │  ที่ได้รับ │   HTML ที่ Render        │      (Render สมบูรณ์)   │                        │
│   └──────────┘   สมบูรณ์แล้วทั้งหน้า      └─────────────────────────┘                        │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────── แบบ SPA + API Backend (Part นี้เป็นต้นไป) ─────────────────────┐
│                                                                                             │
│   Browser                                          Django Server (API เท่านั้น)            │
│   ┌────────────────────┐  1. GET / (ครั้งแรก)      ┌─────────────────────────┐            │
│   │ React App (JS Bundle)│ ───────────────────────> │  Static file server     │            │
│   │  - ประกอบ HTML เอง   │ <─────────────────────── │  (index.html + JS/CSS   │            │
│   │  - จัดการ routing    │  index.html + bundle.js  │   ที่ build ไว้แล้ว)     │            │
│   │  - เก็บ state         │                          └─────────────────────────┘            │
│   └────────────────────┘                                                                    │
│           │  2. GET /api/posts/ (ทุกครั้งที่ต้องข้อมูล)  ┌─────────────────────────┐        │
│           │ ──────────────────────────────────────────> │ urls.py -> ViewSet      │        │
│           │ <────────────────────────────────────────── │   -> Serializer -> JSON │        │
│           │              JSON เท่านั้น (ไม่มี HTML)        │   (Part 044, 046)       │        │
│           ▼                                              └─────────────────────────┘        │
│   React render HTML จาก JSON เอง ฝั่ง client ทั้งหมด                                        │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 541.3 ตารางเปรียบเทียบเชิงลึก

| ประเด็น | Server-Rendered (Django Template) | SPA + API Backend (Django REST + React) |
|---|---|---|
| ใครสร้าง HTML | Django (server) | Browser (client) ผ่าน JavaScript |
| Django ส่งอะไรกลับ | HTML สมบูรณ์ทุก request | JSON เท่านั้น (ยกเว้น request แรกที่ส่ง `index.html` + JS bundle) |
| จำนวน "หน้า" ที่โหลดจริง | 1 request = 1 หน้าเต็ม (full page reload) | โหลด HTML shell ครั้งเดียว แล้ว "เปลี่ยนหน้า" ด้วย JavaScript ล้วน ๆ |
| SEO (Search Engine Optimization) | ดีมาก (HTML พร้อมให้ crawler อ่านทันที) | ต้องทำ SSR/Prerendering เพิ่ม (React ล้วนอ่อนด้าน SEO) |
| First Contentful Paint | เร็ว (HTML มาพร้อมข้อมูล) | ช้ากว่า (ต้องโหลด JS bundle ก่อน แล้วค่อยยิง API) |
| ประสบการณ์ระหว่างเปลี่ยนหน้า | Full reload (กระพริบขาว ๆ สั้น ๆ) | ลื่นไหลแบบ native app (ไม่ reload) |
| ความซับซ้อนของ Deployment | เซิร์ฟเวอร์เดียว | อาจต้องดูแล 2 ระบบ (Django + React) หรือรวมกันแบบ Part 546 |
| ทีมงานที่เหมาะสม | ทีมเล็ก, Full-stack developer คนเดียวจัดการได้ | ทีมที่แยก Frontend/Backend developer ชัดเจน |
| นำ API เดียวไปใช้กับ Mobile App ได้ไหม | ทำไม่ได้โดยตรง (HTML ผูกกับหน้าเว็บ) | ได้ทันที (API เดียวกันเสิร์ฟทั้ง React, iOS, Android) |
| ตัวอย่างงานจริง | เว็บบล็อก, เว็บบริษัท, CMS | Dashboard, Admin Panel ซับซ้อน, แอปที่มี interaction สูง |

### 541.4 ทำไม Blog API จาก Part 044/046 ถึงพร้อมใช้กับ React ทันที

ข่าวดีคือ Django REST Framework ที่คุณสร้างมาตลอด Phase 5 **ไม่รู้จักและไม่สนใจ**
ว่าใครเป็นผู้เรียก API — จะเป็น `curl`, Postman, Mobile App, หรือ React ก็เรียกผ่าน
HTTP + JSON เหมือนกันทุกประการ ตัวอย่าง `PostViewSet` จาก Part 044 (`ModelViewSet`)
และ JWT endpoints จาก Part 046 (`/api/token/`, `/api/token/refresh/`) จะถูกใช้ตลอด
ทั้ง Part นี้โดย **ไม่ต้องแก้โค้ด Django ฝั่ง API เลยแม้แต่บรรทัดเดียว** ยกเว้นการเพิ่ม
CORS (ขั้นตอนที่ 542) ซึ่งเป็นเรื่องของ "ใครมีสิทธิ์เรียก" ไม่ใช่ "API ทำงานอย่างไร"

นี่คือข้อพิสูจน์เชิงสถาปัตยกรรมที่สำคัญที่สุดของ Part นี้: **การออกแบบ Backend เป็น
REST API ที่ดีตั้งแต่ต้น (Phase 5) ทำให้เปลี่ยน Frontend จาก Django Template ไปเป็น
React (หรือ Vue ใน Part 056, หรือ Mobile App ในอนาคต) ได้โดยแทบไม่ต้องแตะ Backend เลย**

### 541.5 เมื่อไหร่ควรเลือกแนวทางไหน

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| เว็บบล็อก/ข่าว ที่ SEO สำคัญมาก | Server-rendered (Part 001-050) |
| Interactive UI เล็กน้อย บนหน้า server-rendered เดิม | HTMX/Alpine.js (Part 053-054) |
| Dashboard/Admin ที่มี interaction ซับซ้อนมาก ไม่ต้องพึ่ง SEO | React SPA + API (Part นี้) |
| ต้องรองรับทั้งเว็บและ Mobile App ด้วย Backend เดียวกัน | React/Vue SPA + API เต็มรูปแบบ |
| ทีมมีทั้ง Frontend และ Backend developer แยกกันชัดเจน | React SPA + API (ทำงานคู่ขนานกันได้อิสระ) |

### 541.6 โครงสร้างโปรเจกต์ที่จะใช้ตลอด Part นี้

```
django-mastery-course/
├── config/                  # Django project (จาก Part 001)
├── blog/                    # Django app: models, serializers, viewsets (Part 012, 044, 046)
├── manage.py
├── requirements.txt
└── frontend/                # โปรเจกต์ React ใหม่ (สร้างในขั้นตอนที่ 543)
    ├── src/
    │   ├── api/
    │   ├── components/
    │   ├── pages/
    │   ├── context/
    │   └── main.jsx
    ├── package.json
    └── vite.config.js
```

เราจะวาง `frontend/` ไว้เป็นโฟลเดอร์ย่อยของโปรเจกต์ Django เดิม (monorepo แบบง่าย)
เพื่อให้จัดการง่ายในหลักสูตรนี้ — ในทีมงานจริงหลายทีมเลือกแยกเป็นคนละ Git repository
กันไปเลย (React repo กับ Django repo แยกกัน) ซึ่งทำได้เหมือนกัน เพียงแค่ต้องดูแล
CORS (ขั้นตอนที่ 542) ให้ถูกต้อง

---

## ขั้นตอนที่ 542: ตั้งค่า CORS ด้วย `django-cors-headers` เพื่อให้ React เรียก API ได้

### 542.1 ทำไมจู่ ๆ API ที่เคยใช้ได้กับ `curl` ถึงถูก Browser บล็อก

ทดลองสถานการณ์นี้: React dev server รันอยู่ที่ `http://localhost:5173` (ค่าเริ่มต้นของ
Vite) และพยายามยิง `fetch('http://localhost:8000/api/posts/')` ไปยัง Django ที่รันอยู่
คนละพอร์ต ผลลัพธ์ที่ได้ใน Console ของ Browser จะเป็น:

```
Access to fetch at 'http://localhost:8000/api/posts/' from origin 'http://localhost:5173'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on
the requested resource.
```

นี่**ไม่ใช่บั๊กของ Django** แต่เป็นกลไกความปลอดภัยของ **Browser เอง** ที่เรียกว่า
**Same-Origin Policy (SOP)** — Browser จะบล็อก JavaScript ที่พยายามอ่าน response จาก
**คนละ origin** กับหน้าเว็บที่กำลังรันอยู่ เว้นแต่ server ปลายทางจะอนุญาตอย่างชัดเจน
ผ่าน CORS headers

### 542.2 Origin คืออะไรกันแน่

**Origin** ประกอบด้วย 3 ส่วน: **scheme (protocol) + host + port** ต้องตรงกันครบทั้ง 3
ส่วนถึงจะถือว่าเป็น origin เดียวกัน:

| URL 1 | URL 2 | Origin เดียวกันไหม | เหตุผล |
|---|---|---|---|
| `http://localhost:5173` | `http://localhost:8000` | ❌ ไม่เหมือนกัน | คนละ port |
| `http://localhost:8000` | `https://localhost:8000` | ❌ ไม่เหมือนกัน | คนละ scheme (http vs https) |
| `http://localhost:8000` | `http://127.0.0.1:8000` | ❌ ไม่เหมือนกัน | คนละ host (แม้จะชี้ไปที่เครื่องเดียวกัน) |
| `http://example.com:8000` | `http://example.com:8000/api/` | ✅ เหมือนกัน | path ไม่นับเป็นส่วนหนึ่งของ origin |

ในการพัฒนา React (พอร์ต 5173 จาก Vite) กับ Django (พอร์ต 8000) จึงเป็นคนละ origin
เสมอ แม้จะรันบนเครื่องเดียวกันก็ตาม

### 542.3 CORS (Cross-Origin Resource Sharing) คือทางออก

**CORS** คือมาตรฐานที่ทำให้ server ฝั่ง Django สามารถ "อนุญาต" ให้ origin ที่ระบุไว้
ล่วงหน้าเรียก API ข้าม origin ได้ โดยแนบ HTTP header `Access-Control-Allow-Origin`
กลับไปกับ response — เมื่อ Browser เห็น header นี้ที่ตรงกับ origin ของหน้าเว็บ มันจะ
อนุญาตให้ JavaScript อ่าน response ได้ตามปกติ

### 542.4 ติดตั้ง `django-cors-headers`

```bash
pip install django-cors-headers
pip freeze > requirements.txt
```

### 542.5 ตั้งค่าใน `settings.py`

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'corsheaders',                              # ← เพิ่มแอปนี้
    'rest_framework',
    'rest_framework_simplejwt.token_blacklist',
    'blog',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'corsheaders.middleware.CorsMiddleware',    # ← ต้องอยู่สูงสุดเท่าที่ทำได้
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

**ข้อควรระวังเรื่องลำดับ Middleware ที่สำคัญมาก**: `CorsMiddleware` ต้องอยู่**ก่อน**
middleware อื่น ๆ ที่อาจสร้าง response กลับไปเลยโดยไม่ผ่านลำดับปกติ (เช่น
`CommonMiddleware`) เพื่อให้ header CORS ถูกแนบไปกับ**ทุก** response รวมถึง response
ที่เป็น error (403, 404, 500) ด้วย เอกสารทางการของ `django-cors-headers` แนะนำให้วางไว้
สูงที่สุดเท่าที่จะทำได้ในลิสต์ โดยเฉพาะให้อยู่**ก่อน** `CommonMiddleware`

### 542.6 ระบุ Origin ที่อนุญาตแบบเจาะจง (วิธีที่แนะนำ)

```python
# config/settings.py
CORS_ALLOWED_ORIGINS = [
    'http://localhost:5173',      # Vite dev server
    'http://127.0.0.1:5173',
]
```

**หลักการสำคัญที่สุดของขั้นตอนนี้**: **ห้ามใช้ `CORS_ALLOW_ALL_ORIGINS = True` ใน
production เด็ดขาด** เพราะเท่ากับอนุญาตให้ **เว็บไซต์ใดก็ได้ในโลก** เรียก API ของคุณ
ข้าม origin ได้ ซึ่งเปิดช่องให้เว็บอันตรายฝัง JavaScript เพื่อยิง request แทนผู้ใช้ที่
หลงเข้าไปเปิดเว็บนั้น (โดยเฉพาะอันตรายมากถ้าใช้ร่วมกับ Session-based auth ตามที่จะ
อธิบายในขั้นตอนที่ 548) ควรระบุ origin ที่อนุญาตแบบเจาะจงเสมอ และแยกค่าตาม environment:

```python
# config/settings.py
import os

if os.environ.get('DJANGO_ENV') == 'production':
    CORS_ALLOWED_ORIGINS = [
        'https://myblog.com',
        'https://www.myblog.com',
    ]
else:
    CORS_ALLOWED_ORIGINS = [
        'http://localhost:5173',
        'http://127.0.0.1:5173',
    ]
```

### 542.7 ตัวเลือกเพิ่มเติมที่ควรรู้

| Setting | ความหมาย | ใช้เมื่อไหร่ |
|---|---|---|
| `CORS_ALLOWED_ORIGINS` | list ของ origin ที่อนุญาตแบบเจาะจง | มาตรฐานที่แนะนำเสมอ |
| `CORS_ALLOWED_ORIGIN_REGEXES` | list ของ regex pattern สำหรับ origin ที่อนุญาต | เมื่อมี subdomain แบบไดนามิก เช่น `https://.*\.myapp\.com$` |
| `CORS_ALLOW_ALL_ORIGINS` | อนุญาตทุก origin (`*`) | **Public API เท่านั้น ที่ตั้งใจให้เว็บไซต์ไหนก็เรียกได้ ไม่มี credential** |
| `CORS_ALLOW_CREDENTIALS` | อนุญาตให้ส่ง cookie/credential ข้าม origin ได้ | จำเป็นถ้าใช้ Session-based auth กับ SPA (ขั้นตอนที่ 548) |
| `CORS_ALLOW_METHODS` | HTTP method ที่อนุญาต | ปกติใช้ค่า default (ครบทุก method) ก็เพียงพอ |
| `CORS_ALLOW_HEADERS` | Header ที่ client ส่งมาได้ | ต้องเพิ่ม `authorization` ถ้าใช้ JWT (ขั้นตอนที่ 544) |

สำหรับหลักสูตรนี้ที่ใช้ JWT Authentication (ส่ง token ผ่าน header ไม่ใช่ cookie) การ
ตั้งค่าเพิ่มเติมที่จำเป็นมีเท่านี้:

```python
# config/settings.py
CORS_ALLOW_CREDENTIALS = False   # ไม่จำเป็นเพราะ JWT ไม่ได้พึ่ง cookie (ต่างจาก Session)
```

(เราจะกลับมาอธิบายกรณี Session-based ที่ต้องตั้งเป็น `True` อย่างละเอียดในขั้นตอนที่ 548)

### 542.8 ทดสอบว่า CORS ทำงานถูกต้อง

```bash
curl -i -X OPTIONS http://127.0.0.1:8000/api/posts/ \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET"
```

ผลลัพธ์ที่คาดหวังต้องมี header เหล่านี้ปนมาด้วย:

```
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: DELETE, GET, OPTIONS, PATCH, POST, PUT
```

Request แบบ `OPTIONS` นี้เรียกว่า **Preflight Request** — Browser จะส่งมาก่อนอัตโนมัติ
เมื่อ request จริงมีเงื่อนไขที่ "ซับซ้อน" กว่าพื้นฐาน (เช่น มี custom header อย่าง
`Authorization`, หรือ `Content-Type: application/json`) เพื่อ "ถาม" server ก่อนว่า
อนุญาตให้ยิง request จริงหรือไม่ ก่อนจะยิง request จริงตามมา — กลไกนี้เกิดขึ้นอัตโนมัติ
โดย Browser ทั้งหมด ไม่ต้องเขียนโค้ดจัดการเองฝั่ง React

### 542.9 ตารางสรุปขั้นตอนที่ 542

| ประเด็น | สรุป |
|---|---|
| ปัญหา | Browser บล็อก JS ที่เรียก API ข้าม origin ตาม Same-Origin Policy |
| ทางแก้ | ติดตั้ง `django-cors-headers` ให้ Django แนบ CORS header กลับไป |
| ตำแหน่ง Middleware | `CorsMiddleware` ต้องอยู่สูงสุด (ก่อน `CommonMiddleware`) |
| การตั้งค่าที่ปลอดภัย | ระบุ `CORS_ALLOWED_ORIGINS` แบบเจาะจงเสมอ ห้ามใช้ `CORS_ALLOW_ALL_ORIGINS` ใน production |
| Preflight Request | Browser ส่ง `OPTIONS` อัตโนมัติก่อน request จริงที่ "ซับซ้อน" |

---

## ขั้นตอนที่ 543: สร้างโปรเจกต์ React ด้วย Vite และดึงข้อมูลจาก Blog API มาแสดง

### 543.1 ทำไมเลือก Vite แทน Create React App

**Create React App (CRA)** เคยเป็นมาตรฐานสำหรับสร้างโปรเจกต์ React มานานหลายปี แต่
ทีมงาน React ประกาศเลิกดูแล CRA อย่างเป็นทางการแล้ว **Vite** (จากผู้สร้าง Vue.js)
กลายเป็นเครื่องมือมาตรฐานใหม่ที่ชุมชน React ทั้งหมดแนะนำแทน เพราะ:

| คุณสมบัติ | Vite | Create React App (เดิม) |
|---|---|---|
| ความเร็วในการ start dev server | เร็วมาก (ใช้ native ES modules, ไม่ bundle ตอน dev) | ช้ากว่ามาก (bundle ทั้งหมดก่อน serve) |
| ความเร็วในการ build | เร็ว (ใช้ esbuild + Rollup) | ช้ากว่า (ใช้ Webpack) |
| การดูแลรักษา (Maintenance) | Active development ต่อเนื่อง | หยุดพัฒนาอย่างเป็นทางการ |
| ขนาด config เริ่มต้น | เล็ก กำหนดเองง่าย | ซ่อน config ไว้ลึก (ต้อง eject) |

### 543.2 สร้างโปรเจกต์ React ด้วย Vite

ต้องมี **Node.js** ติดตั้งไว้ก่อน (แนะนำเวอร์ชัน LTS ล่าสุด 20.x ขึ้นไป):

```bash
node --version    # ควรเห็น v20.x ขึ้นไป
npm --version

# สร้างโปรเจกต์ React ในโฟลเดอร์ frontend/ (รันจาก root ของโปรเจกต์ Django)
npm create vite@latest frontend -- --template react

cd frontend
npm install
```

โครงสร้างที่ได้:

```
frontend/
├── public/
├── src/
│   ├── App.jsx
│   ├── App.css
│   ├── main.jsx
│   └── index.css
├── index.html
├── package.json
└── vite.config.js
```

### 543.3 รัน Dev Server ครั้งแรก

```bash
npm run dev
```

เปิด `http://localhost:5173` ควรเห็นหน้า React เริ่มต้น (โลโก้ Vite + React, ปุ่มนับ
จำนวนคลิก) — นี่คือสัญญาณว่า React ทำงานได้แล้ว

### 543.4 ล้าง Boilerplate และเตรียมโครงสร้างโฟลเดอร์

```bash
# ลบไฟล์ demo ที่ไม่ใช้
rm src/App.css
```

จัดโครงสร้างโฟลเดอร์ที่จะใช้ตลอด Part นี้:

```
frontend/src/
├── api/
│   └── client.js          # ตั้งค่า fetch/axios กลาง (ขั้นตอนที่ 544)
├── components/
│   └── PostCard.jsx
├── pages/
│   └── PostListPage.jsx
├── context/                # (ขั้นตอนที่ 547)
├── App.jsx
├── main.jsx
└── index.css
```

```bash
mkdir -p src/api src/components src/pages src/context
```

### 543.5 ติดตั้ง Axios สำหรับเรียก API

แม้ `fetch()` ในตัว Browser (ที่เรียนใน Part 052) ใช้งานได้ แต่ **Axios** เป็นที่นิยม
มากในระบบนิเวศ React เพราะมี interceptor (จำเป็นสำหรับ auto-refresh token ในขั้นตอน
ที่ 544), แปลง JSON อัตโนมัติ, และ error handling ที่สะดวกกว่า:

```bash
npm install axios
```

### 543.6 สร้าง API Client กลาง

```javascript
// frontend/src/api/client.js
import axios from 'axios';

// อ่านค่า base URL จาก environment variable ของ Vite (ขั้นตอนที่ 549 จะอธิบาย
// การตั้งค่านี้ให้ต่างกันระหว่าง dev/production)
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

const apiClient = axios.create({
    baseURL: API_BASE_URL,
    headers: {
        'Content-Type': 'application/json',
    },
});

export default apiClient;
```

สร้างไฟล์ `.env` ที่ root ของโฟลเดอร์ `frontend/`:

```bash
# frontend/.env
VITE_API_BASE_URL=http://localhost:8000
```

**ข้อควรระวังเรื่อง Environment Variable ของ Vite**: ต่างจาก Node.js ทั่วไปที่ใช้
`process.env`, Vite กำหนดให้ environment variable ที่จะเปิดเผยให้โค้ดฝั่ง client เห็น
ได้ **ต้องขึ้นต้นด้วย `VITE_` เท่านั้น** (ค่าที่ไม่มี prefix นี้จะถูกซ่อนไว้โดยเจตนา
เพื่อป้องกัน secret หลุดไปอยู่ใน JS bundle ที่ผู้ใช้ทุกคนดาวน์โหลดได้) และเข้าถึงผ่าน
`import.meta.env.VITE_XXX` ไม่ใช่ `process.env.VITE_XXX`

### 543.7 ดึงรายการ Post จาก API มาแสดงผล

```jsx
// frontend/src/components/PostCard.jsx
export default function PostCard({ post }) {
    return (
        <article className="post-card">
            <h2>{post.title}</h2>
            <p className="post-meta">
                โดย {post.author} · {new Date(post.created_at).toLocaleDateString('th-TH')}
            </p>
            <p>{post.content.slice(0, 150)}...</p>
            {post.category_name && (
                <span className="post-category">{post.category_name}</span>
            )}
        </article>
    );
}
```

```jsx
// frontend/src/pages/PostListPage.jsx
import { useEffect, useState } from 'react';
import apiClient from '../api/client';
import PostCard from '../components/PostCard';

export default function PostListPage() {
    const [posts, setPosts] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        let isMounted = true;

        async function fetchPosts() {
            try {
                // เรียก endpoint /api/posts/ ที่ PostViewSet (Part 044) สร้างให้ผ่าน Router
                const response = await apiClient.get('/api/posts/');
                if (isMounted) {
                    setPosts(response.data);
                }
            } catch (err) {
                if (isMounted) {
                    setError('ไม่สามารถโหลดรายการบทความได้');
                }
            } finally {
                if (isMounted) {
                    setLoading(false);
                }
            }
        }

        fetchPosts();

        // cleanup function ป้องกัน setState หลัง component ถูก unmount ไปแล้ว
        return () => {
            isMounted = false;
        };
    }, []);

    if (loading) return <p>กำลังโหลด...</p>;
    if (error) return <p className="error">{error}</p>;

    return (
        <div className="post-list">
            <h1>บทความทั้งหมด</h1>
            {posts.length === 0 ? (
                <p>ยังไม่มีบทความ</p>
            ) : (
                posts.map((post) => <PostCard key={post.id} post={post} />)
            )}
        </div>
    );
}
```

```jsx
// frontend/src/App.jsx
import PostListPage from './pages/PostListPage';
import './index.css';

function App() {
    return (
        <div className="app">
            <header className="app-header">
                <h1>My Django + React Blog</h1>
            </header>
            <main>
                <PostListPage />
            </main>
        </div>
    );
}

export default App;
```

### 543.8 รันทั้งสองฝั่งพร้อมกันเพื่อทดสอบ

```bash
# Terminal 1: Django (จาก root ของโปรเจกต์)
python manage.py runserver

# Terminal 2: React (จากโฟลเดอร์ frontend/)
cd frontend
npm run dev
```

เปิด `http://localhost:5173` ควรเห็นรายการบทความจริงที่ดึงมาจาก Django API — ถ้าเห็น
error CORS ใน Console ให้ย้อนกลับไปตรวจสอบขั้นตอนที่ 542 อีกครั้ง

### 543.9 ตารางสรุปขั้นตอนที่ 543

| สิ่งที่ทำ | เครื่องมือ |
|---|---|
| สร้างโปรเจกต์ React | `npm create vite@latest frontend -- --template react` |
| เรียก API | Axios (`apiClient.get('/api/posts/')`) |
| จัดการ Environment Variable | ไฟล์ `.env` + prefix `VITE_` + `import.meta.env` |
| แสดงข้อมูล async | `useState` + `useEffect` + cleanup function |

---

## ขั้นตอนที่ 544: JWT Authentication Flow ใน React (เก็บ/ใช้ token, refresh อัตโนมัติ)

### 544.1 ทบทวน JWT Flow จาก Part 046 ที่จะนำมาใช้จริงฝั่ง Client

Part 046 อธิบาย flow ของ JWT ไว้ครบแล้ว (obtain → ใช้ access token → หมดอายุ →
refresh) และถึงกับมีตัวอย่าง `JWTClient` เวอร์ชัน Python จำลอง (ขั้นตอนที่ 453.8)
ไว้ล่วงหน้า Part นี้จะนำแนวคิดเดียวกันมาเขียนเป็น **Axios interceptor** ฝั่ง React
ซึ่งเป็นรูปแบบมาตรฐานที่ frontend framework สมัยใหม่ทุกตัวใช้

### 544.2 ประเด็นสำคัญที่สุดก่อนเริ่ม: เก็บ Token ไว้ที่ไหนถึงปลอดภัย

| ที่เก็บ | ข้อดี | ข้อเสีย |
|---|---|---|
| `localStorage` | เขียนง่าย อ่านง่าย ข้าม tab ได้ | เข้าถึงได้จาก JavaScript ทุกตัวในหน้า รวมถึง XSS payload — เสี่ยงถูกขโมย |
| `sessionStorage` | เหมือน `localStorage` แต่หายเมื่อปิด tab | ยังเสี่ยง XSS เหมือนกัน |
| In-memory (React state/Context) | XSS ขโมยยากกว่า (ไม่ persist ใน DOM/storage) | หายทันทีที่ refresh หน้า ต้อง login ใหม่ทุกครั้ง |
| HttpOnly Cookie (ตั้งค่าโดย server) | JavaScript **อ่านไม่ได้เลย** ป้องกัน XSS ได้ดีที่สุด | เสี่ยง CSRF แทน (ต้องป้องกันเพิ่ม ตามขั้นตอนที่ 548) ต้องตั้งค่า cookie จาก Django |

**คำแนะนำระดับมืออาชีพ**: สำหรับ SPA ที่ต้องการความปลอดภัยสูงสุด แนวทางที่ดีที่สุด
คือเก็บ **access token ไว้ใน memory เท่านั้น** (ไม่ persist) และเก็บ **refresh token
ไว้ใน HttpOnly Cookie** ที่ Django เป็นคนตั้งให้ (ต้องปรับ `simplejwt` ให้รองรับ ซึ่งมี
package เสริมอย่าง `django-rest-framework-simplejwt` cookie flavor หรือเขียน view
เองเพิ่มเติม) แต่แนวทางนี้ซับซ้อนกว่ามาก

เพื่อความเรียบง่ายและตรงกับที่ Part 046 สอนไว้ (JWT ส่งผ่าน `Authorization` header
ไม่ใช่ cookie) **หลักสูตรนี้จะใช้ `localStorage`** เป็นทางเลือกเริ่มต้นสำหรับผู้เรียน
พร้อมเตือนความเสี่ยง XSS อย่างชัดเจน และแนะนำให้ทำ **Content Security Policy (CSP)**
ควบคู่ไปด้วยเสมอในระบบ production จริงเพื่อลดความเสี่ยงจาก XSS ที่จะมาขโมย token ใน
`localStorage`

### 544.3 สร้าง Auth Service สำหรับจัดการ Token

```javascript
// frontend/src/api/auth.js
const ACCESS_TOKEN_KEY = 'access_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

export function saveTokens({ access, refresh }) {
    localStorage.setItem(ACCESS_TOKEN_KEY, access);
    if (refresh) {
        localStorage.setItem(REFRESH_TOKEN_KEY, refresh);
    }
}

export function getAccessToken() {
    return localStorage.getItem(ACCESS_TOKEN_KEY);
}

export function getRefreshToken() {
    return localStorage.getItem(REFRESH_TOKEN_KEY);
}

export function clearTokens() {
    localStorage.removeItem(ACCESS_TOKEN_KEY);
    localStorage.removeItem(REFRESH_TOKEN_KEY);
}

export function isAuthenticated() {
    return Boolean(getAccessToken());
}
```

### 544.4 ฟังก์ชัน Login ที่เรียก `/api/token/`

```javascript
// frontend/src/api/auth.js (ต่อจากด้านบน)
import axios from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

export async function login(username, password) {
    // ใช้ axios ตรง ๆ (ไม่ใช่ apiClient จากขั้นตอนที่ 543.6) เพราะ endpoint นี้
    // ไม่ต้องแนบ Authorization header เดิม (กำลังจะขอตัวใหม่)
    const response = await axios.post(`${API_BASE_URL}/api/token/`, {
        username,
        password,
    });
    saveTokens(response.data);
    return response.data;
}

export function logout() {
    clearTokens();
}
```

### 544.5 Axios Interceptor: แนบ Access Token อัตโนมัติทุก Request

```javascript
// frontend/src/api/client.js
import axios from 'axios';
import { getAccessToken } from './auth';

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000';

const apiClient = axios.create({
    baseURL: API_BASE_URL,
    headers: {
        'Content-Type': 'application/json',
    },
});

// Request Interceptor: ทำงานก่อนทุก request ถูกส่งออกไปจริง
apiClient.interceptors.request.use((config) => {
    const token = getAccessToken();
    if (token) {
        // ตรงกับรูปแบบ "Bearer <token>" ตามที่ JWTAuthentication ของ simplejwt
        // คาดหวัง (Part 046 ขั้นตอนที่ 453.4)
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

export default apiClient;
```

### 544.6 Axios Interceptor: Refresh Token อัตโนมัติเมื่อได้ 401

นี่คือหัวใจของขั้นตอนนี้ — เมื่อ access token หมดอายุ (Part 046 ขั้นตอนที่ 453.4 อธิบาย
ว่าจะได้ `401` พร้อม `code: "token_not_valid"") เราต้องการให้ React **ขอ access token
ใหม่โดยอัตโนมัติ แล้วลอง request เดิมซ้ำ** โดยผู้ใช้ไม่รู้สึกถึงการ interrupt ใด ๆ เลย:

```javascript
// frontend/src/api/client.js (ต่อจากด้านบน)
import { getRefreshToken, saveTokens, clearTokens } from './auth';

let isRefreshing = false;
let pendingRequests = [];

function resolvePendingRequests(newAccessToken) {
    pendingRequests.forEach((callback) => callback(newAccessToken));
    pendingRequests = [];
}

apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        // เฉพาะกรณี 401 และยังไม่เคย retry request นี้มาก่อน (ป้องกัน infinite loop)
        if (error.response?.status === 401 && !originalRequest._retry) {
            if (isRefreshing) {
                // ถ้ามี request อื่นกำลัง refresh อยู่แล้ว ให้ "ต่อคิว" รอผลลัพธ์
                // แทนที่จะยิง /api/token/refresh/ ซ้ำซ้อนหลายครั้งพร้อมกัน
                return new Promise((resolve) => {
                    pendingRequests.push((newToken) => {
                        originalRequest.headers.Authorization = `Bearer ${newToken}`;
                        resolve(apiClient(originalRequest));
                    });
                });
            }

            originalRequest._retry = true;
            isRefreshing = true;

            try {
                const refreshToken = getRefreshToken();
                if (!refreshToken) {
                    throw new Error('ไม่มี refresh token');
                }

                // เรียก endpoint refresh จาก Part 046 ขั้นตอนที่ 453.5
                const response = await axios.post(`${API_BASE_URL}/api/token/refresh/`, {
                    refresh: refreshToken,
                });

                saveTokens(response.data);
                resolvePendingRequests(response.data.access);

                originalRequest.headers.Authorization = `Bearer ${response.data.access}`;
                return apiClient(originalRequest);
            } catch (refreshError) {
                // refresh token ก็หมดอายุ/ถูก blacklist แล้ว (Part 046 ขั้นตอนที่ 454)
                // ต้องบังคับให้ผู้ใช้ login ใหม่
                clearTokens();
                window.location.href = '/login';
                return Promise.reject(refreshError);
            } finally {
                isRefreshing = false;
            }
        }

        return Promise.reject(error);
    }
);

export default apiClient;
```

### 544.6.1 อธิบายกลไก "ต่อคิว" (`pendingRequests`)

ถ้าหน้าเว็บยิงหลาย request พร้อมกัน (เช่น หน้า Dashboard ที่โหลดทั้ง posts, categories,
comments พร้อมกัน) แล้ว access token หมดอายุพอดี ทุก request จะได้ `401` พร้อมกันหมด
ถ้าไม่มีกลไกป้องกัน แต่ละ request จะพยายามเรียก `/api/token/refresh/` **แยกกัน**
ซ้ำซ้อนโดยไม่จำเป็น (และอาจชนกับกลไก Token Rotation จาก Part 046 ขั้นตอนที่ 454 ที่
blacklist refresh token เก่าทันทีหลังใช้ครั้งแรก ทำให้ request ที่สองใช้ refresh token
เดิมไม่ได้อีกต่อไป) กลไก `isRefreshing` + `pendingRequests` แก้ปัญหานี้โดยให้ **แค่
request แรกเท่านั้น** ที่ไปเรียก refresh จริง ส่วน request อื่น ๆ ที่ตามมาจะ "ต่อคิว"
รอผลลัพธ์แล้วใช้ access token ใหม่ตัวเดียวกัน

### 544.7 สร้างหน้า Login

```jsx
// frontend/src/pages/LoginPage.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { login } from '../api/auth';

export default function LoginPage() {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const [error, setError] = useState(null);
    const [submitting, setSubmitting] = useState(false);
    const navigate = useNavigate();

    async function handleSubmit(event) {
        event.preventDefault();
        setError(null);
        setSubmitting(true);

        try {
            await login(username, password);
            navigate('/');
        } catch (err) {
            setError('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง');
        } finally {
            setSubmitting(false);
        }
    }

    return (
        <div className="login-page">
            <h1>เข้าสู่ระบบ</h1>
            <form onSubmit={handleSubmit}>
                <div className="form-field">
                    <label htmlFor="username">ชื่อผู้ใช้</label>
                    <input
                        id="username"
                        type="text"
                        value={username}
                        onChange={(e) => setUsername(e.target.value)}
                        required
                    />
                </div>
                <div className="form-field">
                    <label htmlFor="password">รหัสผ่าน</label>
                    <input
                        id="password"
                        type="password"
                        value={password}
                        onChange={(e) => setPassword(e.target.value)}
                        required
                    />
                </div>
                {error && <p className="error">{error}</p>}
                <button type="submit" disabled={submitting}>
                    {submitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
                </button>
            </form>
        </div>
    );
}
```

### 544.8 ตารางสรุปขั้นตอนที่ 544

| ส่วนประกอบ | หน้าที่ |
|---|---|
| `saveTokens()` / `getAccessToken()` / `clearTokens()` | จัดการ token ใน `localStorage` |
| Request Interceptor | แนบ `Authorization: Bearer <access>` อัตโนมัติทุก request |
| Response Interceptor | ตรวจจับ `401` แล้วเรียก `/api/token/refresh/` อัตโนมัติ พร้อม retry request เดิม |
| กลไกต่อคิว (`isRefreshing`) | ป้องกันการเรียก refresh ซ้ำซ้อนเมื่อหลาย request พร้อมกัน |
| ความเสี่ยง | `localStorage` เสี่ยง XSS — ต้องทำ CSP และ sanitize input ควบคู่กันเสมอ |

---

## ขั้นตอนที่ 545: React Router สำหรับ navigation ใน SPA (list, detail, login)

### 545.1 ทำไม SPA ต้องมี Router ของตัวเอง

ใน Server-rendered Django (Part 001-050) การ "เปลี่ยนหน้า" คือการที่ Browser ยิง HTTP
request ใหม่ไปยัง URL ใหม่ แล้ว Django's `urls.py` (Part 001 ขั้นตอนที่ 3.3) จับคู่ URL
กับ View ที่เหมาะสม — Browser **reload หน้าใหม่ทั้งหมด** ทุกครั้ง

ใน SPA เราต้องการ **หลีกเลี่ยง** การ reload เต็มหน้า (เพื่อความลื่นไหลตามที่อธิบายไว้ใน
ขั้นตอนที่ 541.3) แต่ผู้ใช้ยังคงต้องการ:
- URL ที่เปลี่ยนตามหน้าที่กำลังดู (`/posts/`, `/posts/my-first-post/`, `/login`)
- ปุ่ม Back/Forward ของ Browser ทำงานถูกต้อง
- Bookmark/แชร์ลิงก์ตรงไปยังหน้าที่ต้องการได้

**React Router** คือ library มาตรฐานที่ทำสิ่งเหล่านี้โดยใช้ **Browser History API**
(`pushState`/`popState`) แทนการยิง request ใหม่จริง ๆ

### 545.2 ติดตั้ง React Router

```bash
npm install react-router-dom
```

### 545.3 กำหนดเส้นทางทั้งหมดของแอป

```jsx
// frontend/src/main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <BrowserRouter>
            <App />
        </BrowserRouter>
    </StrictMode>
);
```

```jsx
// frontend/src/App.jsx
import { Routes, Route } from 'react-router-dom';
import Layout from './components/Layout';
import PostListPage from './pages/PostListPage';
import PostDetailPage from './pages/PostDetailPage';
import LoginPage from './pages/LoginPage';
import CreatePostPage from './pages/CreatePostPage';
import ProtectedRoute from './components/ProtectedRoute';
import './index.css';

function App() {
    return (
        <Routes>
            <Route path="/" element={<Layout />}>
                <Route index element={<PostListPage />} />
                <Route path="posts/:slug" element={<PostDetailPage />} />
                <Route path="login" element={<LoginPage />} />
                <Route
                    path="posts/new"
                    element={
                        <ProtectedRoute>
                            <CreatePostPage />
                        </ProtectedRoute>
                    }
                />
            </Route>
        </Routes>
    );
}

export default App;
```

### 545.4 สร้าง Layout พร้อม Navigation Bar

```jsx
// frontend/src/components/Layout.jsx
import { Outlet, Link, useNavigate } from 'react-router-dom';
import { isAuthenticated, logout } from '../api/auth';

export default function Layout() {
    const navigate = useNavigate();
    const authenticated = isAuthenticated();

    function handleLogout() {
        logout();
        navigate('/login');
    }

    return (
        <div className="app">
            <header className="app-header">
                <nav>
                    {/* <Link> ของ React Router ไม่ทำให้ Browser reload หน้าใหม่
                        ต่างจาก <a href> ธรรมดาที่จะเป็นการยิง request ใหม่เต็มรูปแบบ */}
                    <Link to="/">หน้าแรก</Link>
                    {authenticated ? (
                        <>
                            <Link to="/posts/new">เขียนบทความใหม่</Link>
                            <button onClick={handleLogout}>ออกจากระบบ</button>
                        </>
                    ) : (
                        <Link to="/login">เข้าสู่ระบบ</Link>
                    )}
                </nav>
            </header>
            <main>
                {/* <Outlet /> คือจุดที่ React Router จะ render component
                    ของ child route ที่ตรงกับ URL ปัจจุบัน */}
                <Outlet />
            </main>
        </div>
    );
}
```

### 545.5 หน้า Post Detail ที่อ่าน Parameter จาก URL

```jsx
// frontend/src/pages/PostDetailPage.jsx
import { useEffect, useState } from 'react';
import { useParams, Link } from 'react-router-dom';
import apiClient from '../api/client';

export default function PostDetailPage() {
    // useParams() อ่านค่าจาก path pattern 'posts/:slug' ที่กำหนดไว้ใน App.jsx
    const { slug } = useParams();
    const [post, setPost] = useState(null);
    const [loading, setLoading] = useState(true);
    const [notFound, setNotFound] = useState(false);

    useEffect(() => {
        let isMounted = true;

        async function fetchPost() {
            try {
                // ตรงกับ PostViewSet ที่ตั้ง lookup_field = 'slug' ไว้ (Part 044 ขั้นตอนที่ 432.2)
                const response = await apiClient.get(`/api/posts/${slug}/`);
                if (isMounted) setPost(response.data);
            } catch (err) {
                if (isMounted && err.response?.status === 404) {
                    setNotFound(true);
                }
            } finally {
                if (isMounted) setLoading(false);
            }
        }

        fetchPost();
        return () => { isMounted = false; };
    }, [slug]);   // ถ้า slug เปลี่ยน (คลิกไปโพสต์อื่น) ให้ fetch ใหม่

    if (loading) return <p>กำลังโหลด...</p>;
    if (notFound) return <p>ไม่พบบทความนี้</p>;

    return (
        <article className="post-detail">
            <Link to="/">&larr; กลับไปหน้ารายการ</Link>
            <h1>{post.title}</h1>
            <p className="post-meta">โดย {post.author}</p>
            <div className="post-content">{post.content}</div>
        </article>
    );
}
```

### 545.6 `ProtectedRoute`: ป้องกันหน้าที่ต้อง Login ก่อน

```jsx
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { isAuthenticated } from '../api/auth';

export default function ProtectedRoute({ children }) {
    if (!isAuthenticated()) {
        // Navigate ทำหน้าที่เหมือน redirect ฝั่ง client โดยไม่ reload หน้า
        return <Navigate to="/login" replace />;
    }
    return children;
}
```

**ข้อควรระวังสำคัญ**: `ProtectedRoute` นี้เป็นแค่การป้องกันระดับ **UX** (ซ่อนหน้าจาก
ผู้ใช้ที่ยังไม่ login) เท่านั้น **ไม่ใช่การรักษาความปลอดภัยจริง** เพราะโค้ด JavaScript
ทั้งหมดถูกส่งไปยัง Browser ของผู้ใช้อยู่แล้ว ใครก็ตามที่เปิด DevTools สามารถแก้ไข
`localStorage` หรือข้าม check นี้ได้ไม่ยาก **การตรวจสอบสิทธิ์ที่แท้จริงต้องเกิดขึ้นที่
ฝั่ง Django API เสมอ** ผ่าน `permission_classes` ที่เรียนมาใน Part 045 — `ProtectedRoute`
แค่ทำให้ UX ดีขึ้น (ไม่ต้องให้ผู้ใช้เห็น form แล้วกดส่งแล้วเจอ error 401 ทีหลัง)

### 545.7 ตั้งค่า Vite ให้รองรับ Client-Side Routing เมื่อ Refresh หน้า

ปัญหาคลาสสิกของ SPA: ถ้าผู้ใช้กด Refresh ที่หน้า `/posts/my-first-post/` โดยตรง (ไม่ได้
คลิกผ่าน `<Link>`) Browser จะยิง HTTP request จริงไปยัง server เพื่อขอ path นั้น ถ้า
server ไม่รู้จัก path นี้ (เพราะมันมีอยู่แค่ในความรู้ของ React Router) จะได้ `404` ทันที
วิธีแก้คือให้ dev server ของ Vite (และ web server จริงตอน deploy — ขั้นตอนที่ 546) ส่ง
`index.html` กลับไปเสมอสำหรับทุก path ที่ไม่ตรงกับไฟล์ static จริง แล้วปล่อยให้ React
Router จัดการ routing ต่อฝั่ง client เอง — Vite dev server ทำสิ่งนี้ให้อัตโนมัติอยู่แล้ว
โดยไม่ต้องตั้งค่าเพิ่ม

### 545.8 ตารางสรุป Route ทั้งหมดของแอป

| Path | Component | ต้อง Login ก่อนไหม |
|---|---|---|
| `/` | `PostListPage` | ❌ |
| `/posts/:slug` | `PostDetailPage` | ❌ |
| `/login` | `LoginPage` | ❌ |
| `/posts/new` | `CreatePostPage` | ✅ (ผ่าน `ProtectedRoute`) |

---

## ขั้นตอนที่ 546: Deploy React build ผ่าน Django static files เทียบกับแยก host

### 546.1 สอง Deployment Strategy หลัก

เมื่อพัฒนาเสร็จแล้ว มี 2 แนวทางหลักในการนำ React SPA ไปใช้งานจริงคู่กับ Django API:

```
แนวทางที่ 1: Single Deployment (Django serve ทั้งหมด)
┌─────────────────────────────────────────────┐
│              Django Server (1 เครื่อง)        │
│  ┌───────────────┐   ┌────────────────────┐ │
│  │ /api/*         │   │ / , /posts/*, ...   │ │
│  │ -> DRF ViewSet │   │ -> React build      │ │
│  │    (JSON)      │   │    (staticfiles)    │ │
│  └───────────────┘   └────────────────────┘ │
└─────────────────────────────────────────────┘
        ผู้ใช้เข้าโดเมนเดียว ไม่มีปัญหา CORS เลย

แนวทางที่ 2: Separate Hosting (แยกกันคนละที่)
┌─────────────────────┐        ┌─────────────────────────┐
│  CDN/Static Host      │        │   Django Server          │
│  (Vercel/Netlify/S3)  │  API   │   (Railway/Render/EC2)   │
│  myapp.com             │ ─────> │   api.myapp.com          │
│  -> React build        │        │   -> DRF ViewSet เท่านั้น │
└─────────────────────┘        └─────────────────────────┘
        ต้องตั้งค่า CORS (ขั้นตอนที่ 542) ให้ตรงกัน
```

### 546.2 แนวทางที่ 1: Build React แล้วให้ Django Serve เป็น Static Files

```bash
# frontend/vite.config.js — ตั้งค่าให้ build ออกไปโฟลเดอร์ที่ Django มองเห็น
```

```javascript
// frontend/vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    build: {
        // build ไปไว้ในโฟลเดอร์ static ของ Django app โดยตรง
        outDir: '../blog/static/react_build',
        emptyOutDir: true,
    },
    base: '/static/react_build/',   // ต้องตรงกับ STATIC_URL ของ Django
});
```

```bash
cd frontend
npm run build
# ได้ไฟล์ index.html, assets/*.js, assets/*.css ในโฟลเดอร์ blog/static/react_build/
```

```python
# config/settings.py
import os

STATIC_URL = '/static/'
STATICFILES_DIRS = []   # blog/static/ ถูกรวมอัตโนมัติเพราะเป็น static dir ของแอป blog
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')   # ทบทวนจาก Phase Deployment
```

```python
# blog/views.py
from django.views.generic import TemplateView


class ReactAppView(TemplateView):
    """
    View นี้แค่ส่ง index.html ที่ Vite build ไว้กลับไป แล้วปล่อยให้ React Router
    (ขั้นตอนที่ 545) จัดการ routing ฝั่ง client ทั้งหมดเอง
    """
    template_name = 'react_build/index.html'
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path, re_path
from blog.views import ReactAppView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
    path('api/token/', ...),   # endpoint JWT จาก Part 046

    # ทุก path อื่น ๆ ที่ไม่ตรงกับด้านบน ให้ React Router จัดการเอง
    # (แก้ปัญหา refresh หน้าที่ path ลึก ๆ ตามที่อธิบายในขั้นตอนที่ 545.7)
    re_path(r'^(?!api/|admin/).*$', ReactAppView.as_view(), name='react-app'),
]
```

**ข้อควรระวังเรื่องลำดับ URL Pattern**: `re_path` ที่จับ "ทุกอย่างที่เหลือ" ต้องอยู่
**ล่างสุด** ของ `urlpatterns` เสมอ (Django ไล่ตรวจ pattern จากบนลงล่าง หยุดที่ตัวแรก
ที่ match) และต้อง `(?!api/|admin/)` (negative lookahead) กันไม่ให้ path ของ API/Admin
หลุดไปโดน view นี้จับแทน

### 546.3 ข้อดี-ข้อเสียของแนวทางที่ 1

| ข้อดี | ข้อเสีย |
|---|---|
| ไม่มีปัญหา CORS เลย (origin เดียวกันทั้งหมด) | Django server ต้องรับภาระ serve static file ด้วย (แม้จะใช้ Whitenoise/Nginx ช่วยได้) |
| Deploy ที่เดียว จัดการง่ายสำหรับทีมเล็ก | Build React ทุกครั้งต้องรัน `npm run build` ก่อน deploy Django |
| ไม่ต้องดูแล DNS/domain แยก | Frontend/Backend deploy ผูกกันแน่น — deploy แยก cycle กันไม่ได้ |
| เหมาะกับ MVP หรือโปรเจกต์ขนาดเล็ก-กลาง | ไม่ได้ประโยชน์จาก CDN ระดับโลกสำหรับ static asset เต็มที่ |

### 546.4 แนวทางที่ 2: แยก Host React ต่างหาก (Vercel/Netlify) + Django API แยก

```bash
# Build React เป็น static file ธรรมดา
cd frontend
npm run build
# ได้โฟลเดอร์ dist/ พร้อม deploy ขึ้น Vercel/Netlify/Cloudflare Pages/S3+CloudFront
```

```javascript
// frontend/.env.production
VITE_API_BASE_URL=https://api.myblog.com
```

```python
# config/settings.py (ฝั่ง Django ที่ deploy แยก เช่นที่ api.myblog.com)
CORS_ALLOWED_ORIGINS = [
    'https://myblog.com',
    'https://www.myblog.com',
]
```

### 546.5 ข้อดี-ข้อเสียของแนวทางที่ 2

| ข้อดี | ข้อเสีย |
|---|---|
| React ได้ประโยชน์เต็มที่จาก CDN ทั่วโลก (โหลดเร็วทุกที่) | ต้องตั้งค่า CORS ให้ถูกต้องเสมอ (ขั้นตอนที่ 542) |
| Deploy Frontend/Backend แยกอิสระ คนละทีมคนละ cycle ได้ | ต้องดูแล DNS/domain 2 ชุด (`myblog.com`, `api.myblog.com`) |
| Scale แต่ละส่วนอิสระตามโหลดจริง | ความซับซ้อนของ infrastructure สูงกว่า |
| รองรับ Mobile App เรียก API เดียวกันได้ทันที (Backend ไม่ผูกกับ Frontend ใด ๆ) | มี moving part มากกว่า ต้อง monitor 2 ระบบ |

### 546.6 ตารางสรุป: เมื่อไหร่ควรเลือกแนวทางไหน

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| โปรเจกต์ขนาดเล็ก-กลาง ทีมเดียวดูแลทั้งหมด | แนวทางที่ 1 (Single Deployment) |
| ต้องการ deploy Frontend บ่อยกว่า Backend มาก (หรือกลับกัน) | แนวทางที่ 2 (Separate Hosting) |
| มี Mobile App ที่ต้องใช้ API เดียวกันอยู่แล้ว | แนวทางที่ 2 (Backend เป็น pure API service) |
| ต้องการ SEO ที่ดีขึ้นจาก CDN + Edge caching | แนวทางที่ 2 |
| งบประมาณ/ทีมงานจำกัด ต้องการความง่ายสูงสุด | แนวทางที่ 1 |

เราจะเจาะลึกการ deploy จริงทั้งสองแนวทางบน infrastructure ระดับ production (Docker,
Nginx, CI/CD) ใน Phase DevOps ต่อไป

---

## ขั้นตอนที่ 547: State Management เบื้องต้นด้วย React Context API

### 547.1 ปัญหา "Prop Drilling" ที่จะเกิดขึ้นเมื่อแอปโตขึ้น

จากขั้นตอนที่ 545 สังเกตว่า `Layout.jsx` ต้องเรียก `isAuthenticated()` เอง และแต่ละ
หน้าที่ต้องรู้ว่า "ผู้ใช้ login อยู่หรือไม่" ก็ต้อง import ฟังก์ชันจาก `api/auth.js`
มาเรียกเองซ้ำ ๆ ทุกที่ — เมื่อแอปโตขึ้น (มี component ซ้อนกันหลายชั้น) การส่งข้อมูล
"ผู้ใช้คนปัจจุบันคือใคร" ผ่าน props ลงไปทีละชั้นจะกลายเป็นปัญหาที่เรียกว่า **Prop
Drilling** — ต้องส่ง prop ผ่าน component ตัวกลางที่ไม่ได้ใช้งานมันเลย เพียงเพื่อส่งต่อ
ให้ component ลูกหลานที่อยู่ลึกลงไป

### 547.2 React Context API คือทางออกสำหรับปัญหานี้

**Context** ให้ component ใด ๆ ในต้นไม้ "สมัครรับข้อมูล" จากจุดศูนย์กลางได้โดยตรง
โดยไม่ต้องผ่าน props ทีละชั้น เหมาะสำหรับข้อมูลที่ "หลายที่ในแอปต้องใช้ร่วมกัน" เช่น
ผู้ใช้ที่ login อยู่, ธีมสี, ภาษาที่เลือก

### 547.3 สร้าง `AuthContext`

```jsx
// frontend/src/context/AuthContext.jsx
import { createContext, useContext, useState, useEffect } from 'react';
import * as authApi from '../api/auth';
import apiClient from '../api/client';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        async function loadCurrentUser() {
            if (!authApi.isAuthenticated()) {
                setLoading(false);
                return;
            }
            try {
                // เรียก endpoint /api/me/ (สมมติว่ามีอยู่แล้วจาก Part 045)
                // ที่คืนข้อมูลผู้ใช้ปัจจุบันตาม access token ที่แนบไป
                const response = await apiClient.get('/api/me/');
                setUser(response.data);
            } catch (err) {
                authApi.clearTokens();
            } finally {
                setLoading(false);
            }
        }

        loadCurrentUser();
    }, []);

    async function login(username, password) {
        const data = await authApi.login(username, password);
        const response = await apiClient.get('/api/me/');
        setUser(response.data);
        return data;
    }

    function logout() {
        authApi.logout();
        setUser(null);
    }

    const value = {
        user,
        isAuthenticated: Boolean(user),
        loading,
        login,
        logout,
    };

    return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Custom Hook: จุดเดียวที่ component อื่นใช้เข้าถึง AuthContext
// ทำให้ import สั้นลงและใส่ error guard ไว้ที่เดียว
export function useAuth() {
    const context = useContext(AuthContext);
    if (context === null) {
        throw new Error('useAuth ต้องถูกเรียกภายใน <AuthProvider> เท่านั้น');
    }
    return context;
}
```

### 547.4 ห่อทั้งแอปด้วย `AuthProvider`

```jsx
// frontend/src/main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <BrowserRouter>
            <AuthProvider>
                <App />
            </AuthProvider>
        </BrowserRouter>
    </StrictMode>
);
```

### 547.5 ใช้ `useAuth()` แทนการเรียก `isAuthenticated()` กระจัดกระจาย

```jsx
// frontend/src/components/Layout.jsx (ปรับปรุงจากขั้นตอนที่ 545.4)
import { Outlet, Link, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function Layout() {
    const navigate = useNavigate();
    const { user, isAuthenticated, logout } = useAuth();

    function handleLogout() {
        logout();
        navigate('/login');
    }

    return (
        <div className="app">
            <header className="app-header">
                <nav>
                    <Link to="/">หน้าแรก</Link>
                    {isAuthenticated ? (
                        <>
                            <span>สวัสดี, {user.username}</span>
                            <Link to="/posts/new">เขียนบทความใหม่</Link>
                            <button onClick={handleLogout}>ออกจากระบบ</button>
                        </>
                    ) : (
                        <Link to="/login">เข้าสู่ระบบ</Link>
                    )}
                </nav>
            </header>
            <main>
                <Outlet />
            </main>
        </div>
    );
}
```

```jsx
// frontend/src/components/ProtectedRoute.jsx (ปรับปรุงให้ใช้ Context)
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function ProtectedRoute({ children }) {
    const { isAuthenticated, loading } = useAuth();

    if (loading) return <p>กำลังตรวจสอบสิทธิ์...</p>;
    if (!isAuthenticated) return <Navigate to="/login" replace />;

    return children;
}
```

### 547.6 ตารางเปรียบเทียบ: Props เดิม vs Context

| ประเด็น | ส่ง Props ทีละชั้น (เดิม) | React Context (ขั้นตอนนี้) |
|---|---|---|
| Component ตัวกลางที่ไม่ได้ใช้ข้อมูลนี้ | ต้องรับ-ส่ง prop ต่อไปเรื่อย ๆ (prop drilling) | ไม่ต้องรู้จัก Context เลย |
| แก้ไข/เพิ่ม field ใหม่ในข้อมูลผู้ใช้ | ต้องแก้ signature หลาย component | แก้แค่ `AuthContext.jsx` ที่เดียว |
| เข้าถึงจาก component ที่อยู่ลึกมาก | ยุ่งยากมาก ต้องผ่านหลายชั้น | เรียก `useAuth()` ได้ทันทีจากที่ไหนก็ได้ในต้นไม้ |
| เหมาะกับข้อมูลที่เปลี่ยนบ่อยมาก (ทุก keystroke) | เหมาะกว่า (Context re-render ทุก consumer เมื่อ value เปลี่ยน) | ควรใช้ library เฉพาะทาง (Redux, Zustand) แทนถ้าซับซ้อนมาก |

### 547.7 ข้อจำกัดของ Context API ที่ควรรู้ไว้ล่วงหน้า

Context เหมาะกับข้อมูล "global" ที่เปลี่ยนไม่บ่อยนัก เช่น ผู้ใช้ปัจจุบัน, ธีม, ภาษา
แต่**ไม่เหมาะ**กับ state ที่ซับซ้อนมาก มีการอัปเดตบ่อยและกระทบ performance (เพราะทุก
component ที่เรียก `useContext()` จะ re-render ใหม่ทุกครั้งที่ value ของ Context
เปลี่ยน แม้จะสนใจแค่บางส่วนของ value ก็ตาม) สำหรับแอปขนาดใหญ่ที่มี state ซับซ้อนมาก
ทีมงานมืออาชีพมักหันไปใช้ library เฉพาะทางอย่าง **Redux Toolkit**, **Zustand**, หรือ
**Jotai** ซึ่งอยู่นอกขอบเขตของหลักสูตรนี้ แต่แนวคิดพื้นฐาน (state แบบรวมศูนย์, ผู้บริโภค
สมัครรับข้อมูล) ที่เรียนจาก Context นี้เป็นรากฐานเดียวกัน

---

## ขั้นตอนที่ 548: ข้อควรพิจารณาด้าน CSRF สำหรับ SPA — Session-based vs JWT-based auth กับ CORS

### 548.1 ทบทวน CSRF จากช่วงต้นหลักสูตร

Django มีระบบป้องกัน **CSRF (Cross-Site Request Forgery)** มาให้ในตัวโดยอัตโนมัติ
(Part 001 ขั้นตอนที่ 2.1 กล่าวถึงไว้แล้วว่าเป็นหนึ่งใน "Batteries Included") กลไกนี้
ป้องกันไม่ให้เว็บไซต์อันตรายหลอกให้ Browser ของผู้ใช้ที่ login อยู่แล้ว (มี session
cookie ติดอยู่) ยิง request ที่มีผลเปลี่ยนแปลงข้อมูล (POST/PUT/DELETE) ไปยัง Django
โดยที่เจ้าของบัญชีไม่รู้ตัว

### 548.2 ทำไม JWT ถึง "รอด" จากปัญหา CSRF โดยธรรมชาติ

**นี่คือหลักการที่สำคัญที่สุดของขั้นตอนนี้**: CSRF โจมตีได้เพราะ **Browser แนบ Cookie
ไปกับทุก request โดยอัตโนมัติ** แม้ request นั้นจะถูกสร้างโดยเว็บไซต์อื่นก็ตาม (เพราะ
Cookie ผูกกับ **domain ปลายทาง** ไม่ใช่ domain ต้นทาง) แต่ **JWT ที่ส่งผ่าน
`Authorization` header** (ตามที่ตั้งค่าไว้ตลอด Part 046 และ 544) **ไม่ได้ถูกแนบไป
อัตโนมัติโดย Browser เลย** — ต้องมี JavaScript ของหน้าเว็บนั้นเอง (`axios.interceptors`
ในขั้นตอนที่ 544.5) เป็นคนเขียน header นี้ลงไปเอง

```
CSRF โจมตีสำเร็จได้เมื่อ:                    ทำไม JWT (header) ถึงรอด:
┌─────────────────────────────┐              ┌─────────────────────────────┐
│ เว็บอันตราย (evil.com)        │              │ เว็บอันตราย (evil.com)        │
│  <form action="django.com">  │              │  fetch('django.com/api/...',│
│  <script>submit()</script>   │              │    { headers: {              │
│                                │              │      Authorization: '???'  │  <- ไม่รู้ token!
│  Browser แนบ Cookie ของ       │              │    }})                       │
│  django.com ให้อัตโนมัติ       │              │  evil.com เข้าไม่ถึง          │
│  (แม้ request มาจาก evil.com) │              │  localStorage ของ            │
│  -> CSRF สำเร็จ!               │              │  django.com (คนละ origin)    │
└─────────────────────────────┘              │  -> ไม่มี token ให้แนบ -> ล้มเหลว │
                                              └─────────────────────────────┘
```

evil.com ไม่สามารถอ่าน `localStorage` ของ `myblog.com` ได้เลย (ป้องกันโดย Same-Origin
Policy ที่อธิบายในขั้นตอนที่ 542.1) จึงไม่มีทางรู้ค่า JWT ที่จะเอาไปแนบใน header ได้ —
นี่คือเหตุผลที่ **JWT-based authentication ที่ส่งผ่าน header ไม่ต้องพึ่ง CSRF token
เลย**

### 548.3 ตารางเปรียบเทียบ Session-based vs JWT-based สำหรับ SPA

| ประเด็น | Session-based (Cookie) | JWT-based (Authorization Header) |
|---|---|---|
| แนบไปกับ request โดยใคร | Browser แนบ Cookie อัตโนมัติเสมอ | JavaScript ของแอปต้องเขียน header เอง |
| เสี่ยง CSRF ไหม | ✅ เสี่ยง ต้องป้องกันด้วย CSRF token | ❌ ไม่เสี่ยง (ไม่มีการแนบอัตโนมัติ) |
| เสี่ยง XSS ไหม | ต่ำกว่า ถ้าตั้ง Cookie เป็น `HttpOnly` (JS อ่านไม่ได้) | สูงกว่า ถ้าเก็บใน `localStorage` (ขั้นตอนที่ 544.2) |
| ต้องตั้ง `CORS_ALLOW_CREDENTIALS` ไหม | ✅ ต้องตั้งเป็น `True` เพื่อให้ cookie ข้าม origin ได้ | ❌ ไม่ต้อง (ไม่ได้พึ่ง cookie เลย) |
| Stateless (scale ง่าย) | ❌ ไม่ (server ต้องเก็บ session state) | ✅ ใช่ (Part 046 ขั้นตอนที่ 451.5) |
| เหมาะกับ | SPA ที่ deploy รวมกับ Django (แนวทางที่ 1 ขั้นตอนที่ 546) | SPA ที่แยก host หรือต้องรองรับ Mobile App ด้วย |

### 548.4 ถ้าเลือกใช้ Session-based Authentication กับ React แทน (ทางเลือกที่ควรรู้)

แม้หลักสูตรนี้เลือกใช้ JWT ตลอดทั้ง Part แต่บางทีมเลือกใช้ **Session Authentication**
ของ Django ร่วมกับ React ก็ได้เช่นกัน (โดยเฉพาะเมื่อ deploy รวมกันแบบแนวทางที่ 1) ซึ่ง
ต้องตั้งค่าเพิ่มดังนี้:

```python
# config/settings.py — กรณีเลือกใช้ Session-based กับ SPA แทน JWT
CORS_ALLOWED_ORIGINS = [
    'http://localhost:5173',
]
CORS_ALLOW_CREDENTIALS = True   # จำเป็น เพื่อให้ Browser ส่ง Cookie ข้าม origin ได้

CSRF_TRUSTED_ORIGINS = [
    'http://localhost:5173',
]

# Cookie ต้องตั้ง SameSite ให้เหมาะสมเมื่อ frontend/backend คนละ origin กันตอน dev
CSRF_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_SAMESITE = 'Lax'
```

```javascript
// frontend/src/api/client.js — ต้องตั้ง withCredentials และแนบ CSRF token เอง
const apiClient = axios.create({
    baseURL: API_BASE_URL,
    withCredentials: true,   // ให้ axios แนบ cookie ไปด้วยทุก request ข้าม origin
});

function getCookie(name) {
    const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`));
    return match ? match[2] : null;
}

apiClient.interceptors.request.use((config) => {
    // Django ตั้ง cookie ชื่อ csrftoken ไว้ให้ ต้องอ่านแล้วแนบเป็น header เอง
    // สำหรับ method ที่เปลี่ยนแปลงข้อมูล (POST/PUT/PATCH/DELETE)
    const csrfToken = getCookie('csrftoken');
    if (csrfToken && !['get', 'head', 'options'].includes(config.method)) {
        config.headers['X-CSRFToken'] = csrfToken;
    }
    return config;
});
```

สังเกตว่าแนวทางนี้ซับซ้อนกว่า JWT อย่างชัดเจน (ต้องจัดการทั้ง `withCredentials`,
`CORS_ALLOW_CREDENTIALS`, `CSRF_TRUSTED_ORIGINS`, และอ่าน CSRF cookie เอง) นี่คือ
เหตุผลหลักที่ทีมงานจำนวนมากเลือก **JWT เป็นค่าเริ่มต้นสำหรับ SPA + API** แม้จะต้อง
แลกกับความเสี่ยง XSS ที่ต้องป้องกันด้วยวิธีอื่น (CSP, sanitize input) แทน

### 548.5 หลักการสรุปด้านความปลอดภัยที่สำคัญที่สุดของขั้นตอนนี้

| ภัยคุกคาม | Session-based | JWT-based (header) |
|---|---|---|
| CSRF | ต้องป้องกันเชิงรุก (`{% csrf_token %}`, `CSRF_TRUSTED_ORIGINS`) | ป้องกันได้โดยธรรมชาติของกลไก header |
| XSS | ผลกระทบจำกัดกว่า (ถ้า cookie เป็น `HttpOnly`) | ผลกระทบรุนแรงกว่า (token ใน `localStorage` ถูกขโมยได้ตรง ๆ) |
| แนวทางป้องกันเสริมที่ต้องทำเสมอ | Escape output ทุกจุด (Django Template ทำให้อัตโนมัติ) | Content Security Policy (CSP) + sanitize ทุก input ที่แสดงผลใน React |

**ไม่มีวิธีไหน "ปลอดภัย 100%" โดยตัวมันเอง** — ทั้งสองแนวทางต้องอาศัยแนวปฏิบัติด้าน
ความปลอดภัยเสริมเสมอ (ตามที่ Part 046 ขั้นตอนที่ 459 อธิบาย Security Best Practices
ไว้แล้ว) การเลือกใช้ JWT ไม่ได้แปลว่า "ปลอดภัยกว่า" โดยอัตโนมัติ เพียงแต่ **ย้ายความ
เสี่ยงจากภัยประเภทหนึ่ง (CSRF) ไปเป็นอีกประเภทหนึ่ง (XSS)** ซึ่งต้องป้องกันคนละวิธี

---

## ขั้นตอนที่ 549: Workflow การพัฒนา — รัน Django + React dev server พร้อมกัน (Vite proxy)

### 549.1 ปัญหาของการรัน 2 Dev Server แยกกัน (ที่เจอมาตลอด Part นี้)

ตลอดขั้นตอนที่ผ่านมา เราต้องพิมพ์ URL เต็ม (`http://localhost:8000/api/posts/`) ทุก
ครั้งที่เรียก API จาก React — วิธีนี้ใช้งานได้ แต่มีข้อเสียเชิง workflow:

- ต้องจำ/ตั้งค่า `VITE_API_BASE_URL` ให้ตรงกับ Django เสมอ
- ถ้าเปลี่ยนพอร์ต Django (เช่นมีหลายโปรเจกต์รันพร้อมกัน) ต้องแก้ `.env` ตาม
- Header/Cookie บางอย่างอาจมีพฤติกรรมต่างกันเล็กน้อยระหว่าง cross-origin request (dev)
  กับ same-origin request (production ที่ deploy รวมกันตามขั้นตอนที่ 546.2)

### 549.2 ทางออก: ตั้งค่า Vite Dev Server ให้ทำหน้าที่เป็น Proxy

Vite มีความสามารถในตัวที่ให้ dev server **ส่งต่อ (proxy)** request ที่ขึ้นต้นด้วย
`/api` ไปยัง Django โดยอัตโนมัติ ทำให้โค้ด React **ไม่ต้องรู้จัก URL เต็มของ Django
เลย** — เรียกแค่ `/api/posts/` เหมือนกับว่า Django กับ React อยู่ origin เดียวกัน

```javascript
// frontend/vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    server: {
        port: 5173,
        proxy: {
            // ทุก request ที่ path ขึ้นต้นด้วย /api จะถูกส่งต่อไปยัง Django
            '/api': {
                target: 'http://localhost:8000',
                changeOrigin: true,
            },
        },
    },
});
```

### 549.3 ปรับ API Client ให้ใช้ Relative Path แทน Absolute URL

```javascript
// frontend/src/api/client.js
import axios from 'axios';
import { getAccessToken } from './auth';

const apiClient = axios.create({
    // ไม่ต้องระบุ baseURL เต็มอีกต่อไปตอน dev เพราะ proxy จัดการให้แล้ว
    // (ตอน production หลังตั้ง VITE_API_BASE_URL ใน .env.production จะกลับมาใช้ URL เต็ม)
    baseURL: import.meta.env.VITE_API_BASE_URL || '',
    headers: {
        'Content-Type': 'application/json',
    },
});

apiClient.interceptors.request.use((config) => {
    const token = getAccessToken();
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

export default apiClient;
```

```bash
# frontend/.env (สำหรับ dev — ปล่อยว่างไว้ ให้ proxy จัดการแทน)
# VITE_API_BASE_URL=

# frontend/.env.production (สำหรับ production ที่แยก host จริง)
VITE_API_BASE_URL=https://api.myblog.com
```

### 549.4 ผลลัพธ์: ไม่ต้องพึ่ง CORS เลยตอน Dev (ถ้าใช้ Proxy)

**ข้อสังเกตที่น่าสนใจ**: เมื่อใช้ Vite proxy, request จาก Browser จริง ๆ แล้วจะยิงไปที่
`http://localhost:5173/api/posts/` (origin เดียวกับหน้าเว็บ) แล้ว **Vite dev server
เองต่างหาก** ที่เป็นคนส่งต่อ request นั้นไปยัง Django อีกที (server-to-server ไม่ผ่าน
Browser) ดังนั้นในมุมมองของ Browser จึงไม่มี cross-origin request เกิดขึ้นเลยตอน dev
— **การตั้งค่า CORS ในขั้นตอนที่ 542 จึงไม่จำเป็นตอน dev ถ้าใช้ proxy** (แต่ยังคง
จำเป็นเสมอสำหรับตอน production ถ้าเลือก deploy แบบแยก host ตามขั้นตอนที่ 546.4)
หลักสูตรนี้แนะนำให้ตั้งค่า CORS ไว้ตั้งแต่ต้นเพื่อความเข้าใจกลไกที่ถูกต้อง แต่ในการ
พัฒนาประจำวันสามารถพึ่ง proxy เพื่อความสะดวกได้เช่นกัน

### 549.5 ตั้ง `package.json` Script ให้รันสะดวกขึ้น

```json
// frontend/package.json (เฉพาะส่วน scripts)
{
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview",
        "lint": "eslint ."
    }
}
```

### 549.6 ใช้ `concurrently` เพื่อรันทั้งสอง Server ด้วยคำสั่งเดียว (ทางเลือกเสริม)

สำหรับทีมที่ต้องการรันทั้ง Django และ React ด้วยคำสั่งเดียวจาก root ของโปรเจกต์:

```bash
npm install --save-dev concurrently --prefix frontend
```

```json
// package.json ที่ root ของโปรเจกต์ (สร้างใหม่ถ้ายังไม่มี)
{
    "name": "django-mastery-course",
    "private": true,
    "scripts": {
        "dev": "concurrently -n DJANGO,REACT -c blue,green \"python manage.py runserver\" \"npm run dev --prefix frontend\""
    },
    "devDependencies": {
        "concurrently": "^9.0.0"
    }
}
```

```bash
# รันทั้ง Django และ React พร้อมกันด้วยคำสั่งเดียว
npm run dev
```

Output ที่ได้จะแสดง log ของทั้งสอง process แยกสี (`DJANGO` สีน้ำเงิน, `REACT` สีเขียว)
ในหน้าต่าง Terminal เดียว สะดวกมากสำหรับการพัฒนาประจำวัน

### 549.7 ตารางสรุป Workflow การพัฒนาที่แนะนำ

| ขั้นตอน | คำสั่ง |
|---|---|
| Activate venv ของ Django | `source venv/bin/activate` |
| ติดตั้ง dependency ครั้งแรก | `pip install -r requirements.txt` และ `npm install --prefix frontend` |
| รันทั้งสอง server พร้อมกัน | `npm run dev` (จาก root, ผ่าน `concurrently`) |
| หรือรันแยก 2 terminal | `python manage.py runserver` และ `npm run dev --prefix frontend` |
| เข้าถึงแอประหว่างพัฒนา | เปิด `http://localhost:5173` (React dev server พร้อม proxy ไปยัง Django) |
| Build สำหรับ production | `npm run build --prefix frontend` (ดูขั้นตอนที่ 546 สำหรับ deploy ต่อ) |

---

## ขั้นตอนที่ 550: สรุปและแบบฝึกหัด — สร้าง React frontend ที่สมบูรณ์สำหรับ Blog API

### 550.1 ประกอบทุกอย่างเข้าด้วยกัน: โครงสร้างไฟล์สุดท้ายของ `frontend/`

```
frontend/
├── .env
├── .env.production
├── vite.config.js
├── package.json
├── index.html
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── index.css
    ├── api/
    │   ├── client.js       # ขั้นตอนที่ 543, 544, 549
    │   └── auth.js         # ขั้นตอนที่ 544
    ├── context/
    │   └── AuthContext.jsx # ขั้นตอนที่ 547
    ├── components/
    │   ├── Layout.jsx       # ขั้นตอนที่ 545, 547
    │   ├── PostCard.jsx     # ขั้นตอนที่ 543
    │   └── ProtectedRoute.jsx  # ขั้นตอนที่ 545, 547
    └── pages/
        ├── PostListPage.jsx    # ขั้นตอนที่ 543
        ├── PostDetailPage.jsx  # ขั้นตอนที่ 545
        ├── LoginPage.jsx       # ขั้นตอนที่ 544
        └── CreatePostPage.jsx  # ขั้นตอนนี้ (550.2)
```

### 550.2 หน้าสุดท้ายที่ยังขาด: `CreatePostPage`

```jsx
// frontend/src/pages/CreatePostPage.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import apiClient from '../api/client';

export default function CreatePostPage() {
    const [title, setTitle] = useState('');
    const [content, setContent] = useState('');
    const [categoryId, setCategoryId] = useState('');
    const [error, setError] = useState(null);
    const [submitting, setSubmitting] = useState(false);
    const navigate = useNavigate();

    async function handleSubmit(event) {
        event.preventDefault();
        setError(null);
        setSubmitting(true);

        try {
            // เรียก POST /api/posts/ ที่ PostViewSet.create() จัดการ (Part 044 ขั้นตอนที่ 432.2)
            // perform_create() ฝั่ง Django จะผูก author = request.user ให้อัตโนมัติ
            // จาก JWT ที่แนบมาผ่าน Authorization header (Part 546)
            const response = await apiClient.post('/api/posts/', {
                title,
                content,
                category: categoryId || null,
            });
            navigate(`/posts/${response.data.slug}`);
        } catch (err) {
            if (err.response?.status === 401) {
                setError('กรุณาเข้าสู่ระบบใหม่อีกครั้ง');
            } else {
                setError('เกิดข้อผิดพลาด กรุณาตรวจสอบข้อมูลที่กรอก');
            }
        } finally {
            setSubmitting(false);
        }
    }

    return (
        <div className="create-post-page">
            <h1>เขียนบทความใหม่</h1>
            <form onSubmit={handleSubmit}>
                <div className="form-field">
                    <label htmlFor="title">หัวข้อ</label>
                    <input
                        id="title"
                        type="text"
                        value={title}
                        onChange={(e) => setTitle(e.target.value)}
                        required
                    />
                </div>
                <div className="form-field">
                    <label htmlFor="content">เนื้อหา</label>
                    <textarea
                        id="content"
                        rows={10}
                        value={content}
                        onChange={(e) => setContent(e.target.value)}
                        required
                    />
                </div>
                {error && <p className="error">{error}</p>}
                <button type="submit" disabled={submitting}>
                    {submitting ? 'กำลังบันทึก...' : 'เผยแพร่บทความ'}
                </button>
            </form>
        </div>
    );
}
```

### 550.3 CSS พื้นฐานเพื่อให้แอปดูใช้งานได้จริง

```css
/* frontend/src/index.css */
* {
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', 'Noto Sans Thai', sans-serif;
    margin: 0;
    background-color: #f5f5f5;
    color: #1a1a1a;
}

.app-header {
    background-color: #1a1a2e;
    padding: 1rem 2rem;
}

.app-header nav {
    display: flex;
    align-items: center;
    gap: 1.5rem;
}

.app-header nav a,
.app-header nav span {
    color: #ffffff;
    text-decoration: none;
}

.app-header nav button {
    background: none;
    border: 1px solid #ffffff;
    color: #ffffff;
    padding: 0.3rem 0.8rem;
    border-radius: 4px;
    cursor: pointer;
}

main {
    max-width: 720px;
    margin: 2rem auto;
    padding: 0 1rem;
}

.post-card {
    background: #ffffff;
    padding: 1.5rem;
    border-radius: 8px;
    margin-bottom: 1rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.post-category {
    display: inline-block;
    background: #e0e0ff;
    padding: 0.2rem 0.6rem;
    border-radius: 4px;
    font-size: 0.85rem;
}

.form-field {
    margin-bottom: 1rem;
}

.form-field label {
    display: block;
    margin-bottom: 0.3rem;
    font-weight: 600;
}

.form-field input,
.form-field textarea {
    width: 100%;
    padding: 0.6rem;
    border: 1px solid #cccccc;
    border-radius: 4px;
}

.error {
    color: #d32f2f;
}
```

### 550.4 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจสถาปัตยกรรม SPA + REST API และเปรียบเทียบกับ Server-rendered ที่เรียนมาก่อน
  หน้า พร้อมรู้ว่าเมื่อไหร่ควรเลือกแนวทางไหน
- ตั้งค่า CORS ด้วย `django-cors-headers` ให้ React เรียก Django API ข้าม origin ได้
  อย่างปลอดภัย (ระบุ origin เจาะจง ไม่เปิดกว้างเกินจำเป็น)
- สร้างโปรเจกต์ React ด้วย Vite และดึงข้อมูลจริงจาก Blog API (Part 044) มาแสดงผล
- เขียน JWT Authentication Flow เต็มรูปแบบฝั่ง React ด้วย Axios Interceptor ที่แนบ
  token อัตโนมัติและ refresh token เมื่อหมดอายุโดยไม่ต้องรบกวนผู้ใช้
- ใช้ React Router จัดการ navigation แบบ SPA (list, detail, login, protected route)
- เข้าใจสองแนวทาง deploy (Single Deployment ผ่าน Django static files vs Separate
  Hosting) พร้อมข้อดี-ข้อเสียของแต่ละแบบ
- จัดการ state ร่วมของแอปด้วย React Context API แก้ปัญหา Prop Drilling
- เข้าใจความสัมพันธ์เชิงลึกระหว่าง CSRF, Session-based auth, JWT-based auth, และ CORS
  — และทำไม JWT ผ่าน header ถึงรอดพ้นจาก CSRF โดยธรรมชาติ (แต่ต้องระวัง XSS แทน)
- ตั้ง Workflow การพัฒนาที่รัน Django และ React dev server พร้อมกันผ่าน Vite proxy
- ประกอบทุกอย่างเป็น React frontend ที่สมบูรณ์สำหรับ Blog API: list, detail, login,
  create post

### 550.5 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้งและตั้งค่า `django-cors-headers` สำเร็จ ทดสอบ Preflight Request ด้วย `curl` ได้
- [ ] สร้างโปรเจกต์ React ด้วย Vite และดึงรายการ Post จาก API มาแสดงผลได้จริง
- [ ] Login ผ่านฟอร์ม React แล้วได้ access/refresh token เก็บใน `localStorage`
- [ ] ทดสอบว่า Axios Interceptor refresh token อัตโนมัติเมื่อ access token หมดอายุ
      (ลองตั้ง `ACCESS_TOKEN_LIFETIME` เป็น 30 วินาทีชั่วคราวเพื่อทดสอบ)
- [ ] Navigate ระหว่างหน้า list/detail/login ด้วย React Router โดยไม่มี full page reload
- [ ] เข้าใจความแตกต่างระหว่าง Single Deployment กับ Separate Hosting และเลือกได้ว่า
      โปรเจกต์แบบไหนควรใช้แนวทางไหน
- [ ] ใช้ `AuthContext` แทนการเรียก `isAuthenticated()` กระจัดกระจายในหลายไฟล์
- [ ] อธิบายได้ว่าทำไม JWT-based auth ถึงไม่ต้องพึ่ง CSRF token ในขณะที่ Session-based
      auth ต้องพึ่ง
- [ ] รัน Django และ React dev server พร้อมกันผ่าน `npm run dev` (ด้วย `concurrently`)
- [ ] สร้างบทความใหม่ผ่านฟอร์ม React สำเร็จ และเห็นบทความปรากฏในหน้า list ทันที

### 550.6 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่มฟีเจอร์ **Pagination** ในหน้า `PostListPage` โดยใช้ query
parameter `?page=` ที่ DRF's `PageNumberPagination` รองรับอยู่แล้ว (ทบทวนจาก Part 043)
สร้างปุ่ม "หน้าถัดไป"/"หน้าก่อนหน้า" ที่อัปเดต URL ผ่าน React Router (เช่น
`/?page=2`) และ fetch ข้อมูลหน้าที่ถูกต้องจาก API

**แบบฝึกหัดที่ 2**: เพิ่มหน้า **แก้ไขบทความ** (`/posts/:slug/edit`) ที่ใช้
`PATCH /api/posts/{slug}/` (ตรงกับ `partial_update()` ของ `ModelViewSet` จาก Part 044)
โดยต้องโหลดข้อมูลเดิมมาแสดงในฟอร์มก่อน แล้วค่อยส่งเฉพาะ field ที่แก้ไข พร้อมป้องกัน
ด้วย `ProtectedRoute` และตรวจสอบว่าเฉพาะเจ้าของบทความเท่านั้นที่เห็นปุ่มแก้ไข (เทียบ
`post.author` กับ `user.username` จาก `AuthContext`)

**แบบฝึกหัดที่ 3**: เพิ่ม **Toast Notification** (เช่นด้วย library `react-hot-toast`)
ที่แสดงข้อความ "กำลังต่ออายุ session..." ทุกครั้งที่ Axios Interceptor ในขั้นตอนที่
544.6 กำลัง refresh token อยู่เบื้องหลัง เพื่อให้ผู้ใช้เห็นว่าเกิดอะไรขึ้น (ปกติแล้ว
กลไกนี้ทำงานเงียบ ๆ โดยผู้ใช้ไม่รู้ตัว แต่การแสดงผลลัพธ์ทำให้เข้าใจ flow ชัดเจนขึ้น
สำหรับจุดประสงค์การเรียนรู้)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ทดลอง deploy โปรเจกต์ทั้งหมดด้วยทั้งสองแนวทางจาก
ขั้นตอนที่ 546: (ก) build React แล้วให้ Django serve เป็น static file บนเครื่องเดียว
(ทดสอบด้วย `python manage.py runserver` หลัง `npm run build`), และ (ข) รัน React
build ผ่าน `npm run preview` แยกพอร์ตต่างหาก แล้วตั้งค่า `CORS_ALLOWED_ORIGINS` ให้
ตรงกับพอร์ตนั้น เปรียบเทียบว่าแนวทางไหนตั้งค่าง่ายกว่าในบริบทของคุณ และบันทึกปัญหา
ที่พบเจอระหว่างทาง (ถ้ามี)

### 550.7 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเขียน Django Template ทิ้งไปเลยหรือไม่ เมื่อเปลี่ยนมาใช้ React?**
A: ไม่จำเป็นเสมอไป หลายโปรเจกต์จริงใช้ทั้งสองแนวทางผสมกัน เช่น หน้า Marketing/Landing
Page และหน้าที่ต้องการ SEO สูง (บทความ, สินค้า) ยังคงใช้ Django Template แบบ
server-rendered (Part 001-050) ส่วนหน้า Dashboard หรือ Admin Panel ภายในที่มี
interaction ซับซ้อนและไม่ต้องพึ่ง SEO ค่อยทำเป็น React SPA แยกต่างหาก การเลือก
สถาปัตยกรรมควรพิจารณาเป็นหน้าต่อหน้า ไม่ใช่ตัดสินใจครั้งเดียวทั้งโปรเจกต์

**Q: TypeScript จำเป็นไหมสำหรับโปรเจกต์ React ระดับมืออาชีพ?**
A: ไม่ได้บังคับ แต่ **แนะนำอย่างยิ่ง** สำหรับโปรเจกต์ขนาดกลาง-ใหญ่ที่มีทีมงานหลายคน
เพราะ TypeScript ช่วยจับ error เรื่อง type ตั้งแต่ตอนเขียนโค้ด (เช่น ลืมส่ง prop ที่
จำเป็น, สะกดชื่อ field ผิด) ก่อนที่จะกลายเป็นบั๊กตอน runtime หลักสูตรนี้ใช้ JavaScript
(JSX) ล้วนเพื่อโฟกัสที่แนวคิดหลักของการเชื่อม React กับ Django ก่อน แต่ทุกโค้ดตัวอย่าง
ใน Part นี้แปลงเป็น TypeScript (`.tsx`) ได้โดยตรงเมื่อพร้อม

**Q: ทำไมไม่ใช้ Next.js แทน Vite + React ธรรมดา ในเมื่อ Next.js แก้ปัญหา SEO ได้ด้วย SSR?**
A: Next.js เป็นตัวเลือกที่ดีมากเมื่อ SEO สำคัญ (เพราะรองรับ Server-Side Rendering และ
Static Site Generation ในตัว) แต่มันมาพร้อมแนวคิดและความซับซ้อนเพิ่มเติมมาก (App
Router, Server Components, การแยกว่าโค้ดไหนรันที่ server/client) หลักสูตรนี้เลือก
Vite + React ล้วนก่อน เพื่อให้เห็นกลไกพื้นฐานของ "SPA คุยกับ REST API" อย่างชัดเจน
ที่สุดโดยไม่มีเลเยอร์เพิ่มเติมมาบดบัง เมื่อเข้าใจแนวคิดนี้แน่นแล้ว การต่อยอดไปเรียน
Next.js (หรือ Remix) จะง่ายขึ้นมาก เพราะหลักการเรียก API, จัดการ JWT, และ routing
พื้นฐานเหมือนกัน

**Q: ควรเก็บ `frontend/` กับ Django ไว้ใน Git repository เดียวกัน (monorepo) หรือแยก repo?**
A: ทั้งสองแบบใช้ได้ในงานจริง **Monorepo** (แบบที่ Part นี้ใช้) สะดวกสำหรับทีมเล็กที่
ดูแลทั้ง Frontend/Backend คนเดียวกัน หรือทีมที่ deploy ทั้งสองส่วนพร้อมกันเสมอ (แนวทาง
Single Deployment จากขั้นตอนที่ 546.2) ส่วน **Separate Repository** เหมาะกับทีมที่แยก
Frontend/Backend developer ชัดเจน มี deploy cycle อิสระจากกัน (แนวทาง Separate
Hosting จากขั้นตอนที่ 546.4) ไม่มีคำตอบที่ถูกต้องเพียงหนึ่งเดียว ขึ้นอยู่กับโครงสร้าง
ทีมและความถี่ในการ deploy ของแต่ละส่วน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 056: Django กับ Vue.js Integration** จะพาคุณสำรวจอีกหนึ่งทางเลือกยอดนิยมสำหรับ
การสร้าง Frontend แยกจาก Django คือ **Vue.js** ซึ่งมีปรัชญาการเขียนที่ต่างจาก React
พอสมควร (Template syntax ที่ใกล้เคียง HTML ธรรมดามากกว่า, Reactivity System ในตัวที่
ไม่ต้องพึ่ง `useState`/`useEffect`) แต่แนวคิดหลักในการเชื่อมกับ Django REST API — CORS,
JWT Authentication Flow, Client-side Routing, State Management — จะคล้ายกับที่คุณ
เพิ่งเรียนใน Part นี้อย่างมาก จนสามารถเทียบเคียงกันได้บรรทัดต่อบรรทัดในหลายจุด คุณจะ
ได้เห็นว่าหลักการที่เรียนรู้จาก Part 055 (การออกแบบ Backend เป็น pure API ที่ไม่ผูกกับ
Frontend framework ใดวันหนึ่ง) นำไปใช้ซ้ำได้กับ Frontend framework ตัวไหนก็ได้จริง ๆ

เตรียมทบทวนแนวคิด Component-based UI, Reactive State, และ JWT Flow จาก Part นี้ให้
แม่น เพราะ Part 056 จะสร้าง Vue.js เวอร์ชันของแอปเดียวกันนี้ เพื่อให้เห็นการเปรียบเทียบ
ที่ชัดเจนระหว่างสอง Frontend Framework ยอดนิยมที่สุดในโลกปัจจุบัน
