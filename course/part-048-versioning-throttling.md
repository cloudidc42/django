# Part 048: API Versioning และ Throttling

> **ขั้นตอนที่ 471-480 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: เข้าใจว่าทำไม API ระดับโลกทุกตัวต้องมีระบบ **Versioning**
> ตั้งแต่วันแรกที่มี client ภายนอกใช้งานจริง เปรียบเทียบ 4 กลยุทธ์ versioning ที่ DRF
> มีให้ในตัว (`URLPathVersioning`, `NamespaceVersioning`, `AcceptHeaderVersioning`,
> `QueryParameterVersioning`) แล้วลงมือ implement `/api/v1/posts/` และ `/api/v2/posts/`
> ที่มีโครงสร้าง response ต่างกันจริงบน `blog` API วางกลยุทธ์ deprecation ของเวอร์ชันเก่า
> อย่างมืออาชีพ จากนั้นเจาะลึกระบบ **Throttling** ของ DRF ตั้งแต่ class สำเร็จรูป
> (`AnonRateThrottle`, `UserRateThrottle`, `ScopedRateThrottle`) ไปจนถึงการเขียน
> Custom Throttle Class เอง ตั้งค่า throttle เฉพาะ endpoint ที่อ่อนไหว (เช่น login)
> ให้เข้มกว่าปกติ ต่อยอด `LoginRateThrottle` ที่เกริ่นไว้ใน Part 046 ให้สมบูรณ์ พร้อมเข้าใจ
> ความสัมพันธ์ระหว่าง throttling กับ caching backend เบื้องต้น (จะเจาะลึกเต็มรูปแบบใน
> Part 068) และปิดท้ายด้วยการเขียนเทสสำหรับพฤติกรรม throttling อย่างถูกต้อง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 471: ทำไม API Versioning ถึงสำคัญ — ปัญหาที่เกิดถ้าไม่มี versioning เมื่อ API ต้อง breaking change
- ขั้นตอนที่ 472: เปรียบเทียบ `URLPathVersioning`, `NamespaceVersioning`, `AcceptHeaderVersioning`, `QueryParameterVersioning`
- ขั้นตอนที่ 473: Implement versioned Serializer/View จริง (`/api/v1/posts/` vs `/api/v2/posts/` ที่มี field ต่างกัน)
- ขั้นตอนที่ 474: กลยุทธ์ Deprecation ของ API version เก่า (deprecation header, sunset date, การสื่อสารกับผู้ใช้ API)
- ขั้นตอนที่ 475: Throttling Class — `AnonRateThrottle`, `UserRateThrottle`
- ขั้นตอนที่ 476: เขียน Custom Throttle Class เอง (`ScopedRateThrottle` และ throttle แบบกำหนดเอง)
- ขั้นตอนที่ 477: ตั้งค่า Throttle เฉพาะ endpoint (เช่น login endpoint throttle เข้มกว่าปกติ)
- ขั้นตอนที่ 478: ผสาน Throttling กับ Caching (เกริ่นสั้น ๆ เจาะลึกเต็มใน Part 068)
- ขั้นตอนที่ 479: การเขียน Test สำหรับพฤติกรรม Throttling
- ขั้นตอนที่ 480: สรุปและแบบฝึกหัด — เพิ่ม versioning และ throttling เต็มรูปแบบให้ Blog API

---

## ขั้นตอนที่ 471: ทำไม API Versioning ถึงสำคัญ — ปัญหาที่เกิดถ้าไม่มี versioning เมื่อ API ต้อง breaking change

### 471.1 ทวนสถานะปัจจุบันของ `blog` API

ตั้งแต่ Part 040 เป็นต้นมา คุณสร้าง `blog` API ขึ้นมาเรื่อย ๆ จนถึง Part 046 ที่มีระบบ
authentication ครบวงจร (JWT, OAuth2, API Key) endpoint หลักตอนนี้คือ:

```
GET  /api/posts/
GET  /api/posts/<slug>/
POST /api/posts/
```

Response ของ `GET /api/posts/` มีหน้าตาแบบนี้มาตลอด:

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

สมมติว่าตอนนี้ `blog` API ของคุณถูกใช้งานจริงแล้วโดย 3 ฝ่าย: **มือถือ iOS app**, **มือถือ
Android app**, และ **partner ภายนอก** ที่เชื่อมผ่าน API Key (จาก Part 046 ขั้นตอนที่ 457)
ทั้ง 3 ฝ่ายนี้ต่างพัฒนาโค้ดของตัวเองโดยอิงจากโครงสร้าง JSON ด้านบน — เช่น iOS app มีโค้ด
Swift ที่ทำ `json["author"] as? String` โดยตรง

### 471.2 สถานการณ์จำลอง: ทีมตัดสินใจปรับปรุง API โดยไม่มี Versioning

ทีมพัฒนาต้องการปรับปรุง API ให้ดีขึ้น 2 อย่าง:

1. เปลี่ยน `author` จาก string (username) ให้เป็น **nested object** ที่มีทั้ง `id`,
   `username`, `role` เพื่อให้ frontend ไม่ต้องยิง request แยกไปถามข้อมูลผู้เขียนเพิ่ม
2. เปลี่ยน `is_published` (boolean) ให้เป็น `status` (string enum: `draft`/`published`/
   `archived`) เพื่อรองรับสถานะที่มากกว่าสองค่าในอนาคต

ถ้าทีม deploy การเปลี่ยนแปลงนี้ทับ endpoint เดิม `/api/posts/` ตรง ๆ โดยไม่มี versioning
ผลลัพธ์ที่ผู้ใช้จริงจะเห็นทันทีคือ:

```
iOS app (เวอร์ชันที่ user ยังไม่อัปเดต):
  json["author"] as? String  → ได้ nil เพราะตอนนี้ author เป็น Dictionary ไม่ใช่ String
  → แอปแสดงชื่อผู้เขียนเป็นค่าว่าง หรือ crash ทันทีถ้าไม่ได้ทำ optional handling ดีพอ

Partner ภายนอก:
  if data['is_published']:  → KeyError เพราะ field นี้หายไปแล้ว ถูกแทนที่ด้วย 'status'
  → ระบบ sync ข้อมูลของ partner ล่มทั้งระบบทันทีที่ deploy เสร็จ
```

**ปัญหาที่ร้ายแรงที่สุด**: mobile app **ไม่เหมือนเว็บไซต์** ที่ deploy แล้วผู้ใช้เห็นเวอร์ชัน
ใหม่ทันที การอัปเดต mobile app ต้องผ่านกระบวนการ **App Store / Play Store review**
(อาจใช้เวลาหลายวันถึงหลายสัปดาห์) และผู้ใช้จำนวนมาก**ไม่กดอัปเดต**เป็นเวลาหลายเดือนหรือ
เป็นปีด้วยซ้ำ นั่นแปลว่า ณ วินาทีที่ backend เปลี่ยน แอปเวอร์ชันเก่านับล้าน install ที่ยังอยู่
ในมือผู้ใช้จะพังพร้อมกันทันที โดยที่ทีมพัฒนาไม่มีทางแก้ไขฝั่ง client ได้เร็วพอ

### 471.3 ตารางสรุปประเภทของ Breaking Change ที่พบบ่อยที่สุด

| ประเภทการเปลี่ยนแปลง | ตัวอย่าง | ทำไมถึง breaking |
|---|---|---|
| เปลี่ยนชนิดข้อมูลของ field | `author` จาก `string` → `object` | Client ที่ cast type ตรง ๆ จะพังหรือได้ค่าผิด |
| เปลี่ยนชื่อ field | `is_published` → `status` | Client เข้าถึง field เดิมด้วยชื่อเก่าจะได้ `KeyError`/`undefined` |
| ลบ field ออก | เอา `content` ออกจาก list endpoint | Client ที่ต้องใช้ field นั้นแสดงผลผิดหรือ error |
| เพิ่ม field ที่ **required** ตอนส่งข้อมูล (request) | บังคับให้ต้องส่ง `category_id` ทุกครั้งตอนสร้างโพสต์ | Client เดิมที่ไม่รู้จัก field นี้จะถูกปฏิเสธด้วย `400 Bad Request` |
| เปลี่ยนโครงสร้าง response ทั้งชุด | จาก `[]` (array ตรง ๆ) เป็น `{"results": [], "count": N}` (paginated) | Client ที่วน `for item in response` จะวนผิดโครงสร้างทันที |
| ลบ endpoint ทิ้งไปเลย | เลิกใช้ `/api/posts/<slug>/` เปลี่ยนเป็น `/api/posts/<id>/` | Client ที่เก็บ URL เดิมไว้เรียกไม่ได้อีกต่อไป |
| เปลี่ยน HTTP status code ที่คืนกลับ | เปลี่ยนจาก `200 OK` เป็น `201 Created` ตอนสร้างสำเร็จ | Client ที่เช็ค status code ตรง ๆ (`if status == 200`) จะเข้าใจผิดว่าล้มเหลว |

การเปลี่ยนแปลงในตารางนี้ **ไม่ใช่เรื่องแปลก** — มันคือวิวัฒนาการปกติของ API ที่ดีขึ้นเรื่อย ๆ
ตามเวลา ปัญหาไม่ได้อยู่ที่ "ห้ามเปลี่ยน" แต่อยู่ที่ **"เปลี่ยนอย่างไรไม่ให้ client เดิมพังทันที"**
ซึ่งคำตอบคือ **API Versioning**

### 471.4 API Versioning คืออะไรกันแน่ (แยกจาก Semantic Versioning ของซอฟต์แวร์)

หลายคนสับสนระหว่าง **Semantic Versioning** (SemVer, เช่น `v2.3.1` ของตัวแอปพลิเคชันหรือ
library) กับ **API Versioning** ซึ่งเป็นคนละเรื่องกัน:

| ประเด็น | Semantic Versioning (SemVer) | API Versioning |
|---|---|---|
| ใช้กับ | ตัวซอฟต์แวร์/library ทั้งก้อน (`django==5.1.2`) | **สัญญา (contract)** ระหว่าง client-server ของ endpoint หนึ่ง ๆ |
| รูปแบบ | `MAJOR.MINOR.PATCH` (3 ตัวเลข) | มักใช้แค่ `v1`, `v2` (เลขเดียว) หรือ date-based (`2026-01-15`) |
| จุดประสงค์ | บอกว่าโค้ดเปลี่ยนมากแค่ไหน | บอกว่า **response/request shape** ของ endpoint นี้เป็นแบบไหน |
| ใครเป็นคนดู | นักพัฒนาที่ติดตั้ง dependency | Client ที่เรียก API (เลือกว่าจะคุยกับ backend ด้วยสัญญาแบบไหน) |
| เปลี่ยนบ่อยแค่ไหน | ทุกครั้งที่ release โค้ดใหม่ | เปลี่ยนเฉพาะตอนมี **breaking change** เท่านั้น (ปีละ 1-2 ครั้งก็ถือว่าถี่แล้ว) |

**หลักการสำคัญที่สุดของขั้นตอนนี้**: การเพิ่ม field ใหม่ที่ **optional** (ไม่บังคับ) หรือเพิ่ม
endpoint ใหม่ **ไม่ใช่ breaking change** และ **ไม่จำเป็นต้องขึ้น version ใหม่** — ควรขึ้น
version ใหม่เฉพาะเมื่อมีการเปลี่ยนแปลงที่ทำให้ client เดิมที่ทำงานถูกต้องอยู่แล้ว **จะพัง**
เท่านั้น การขึ้น version พร่ำเพรื่อทำให้ต้อง maintain โค้ดหลายเวอร์ชันพร้อมกันโดยไม่จำเป็น

### 471.5 ตัวอย่างจากอุตสาหกรรมจริง

| บริษัท | กลยุทธ์ Versioning | ตัวอย่าง |
|---|---|---|
| **Stripe** | Date-based version ผ่าน header, ผูก account กับ version ที่ "pin" ไว้ตอนสร้าง API key | `Stripe-Version: 2024-06-20` |
| **GitHub** | Date-based version ผ่าน header, มี default version ถ้าไม่ระบุ | `X-GitHub-Api-Version: 2022-11-28` |
| **Twitter/X (ยุค API v1.1 → v2)** | URL path versioning, เคยมีดราม่าใหญ่ตอนบังคับ sunset v1.1 กะทันหันจนนักพัฒนาจำนวนมากได้รับผลกระทบ | `/1.1/statuses/...` → `/2/tweets/...` |
| **Google Maps Platform** | URL path versioning ผสมกับ discovery document | `/maps/api/geocode/v1/json` |

บทเรียนจากกรณี Twitter คือ **การมี versioning อย่างเดียวไม่พอ** ต้องมี**กลยุทธ์ deprecation**
ที่ให้เวลา client เพียงพอด้วย (เราจะเรียนเรื่องนี้ในขั้นตอนที่ 474)

### 471.6 ตารางสรุปขั้นตอนที่ 471

| ประเด็น | สรุป |
|---|---|
| ปัญหาถ้าไม่มี versioning | Breaking change ทับ endpoint เดิมทำให้ client เก่าพังทันที โดยเฉพาะ mobile app ที่อัปเดตช้า |
| Breaking change คืออะไร | เปลี่ยน type/ชื่อ field, ลบ field/endpoint, เปลี่ยนโครงสร้าง response ที่ client เดิมพึ่งพาอยู่ |
| API Versioning ต่างจาก SemVer | SemVer วัดความเปลี่ยนแปลงของโค้ด, API Versioning คือสัญญาของ response shape |
| ควรขึ้น version ใหม่เมื่อไหร่ | เฉพาะตอนมี breaking change เท่านั้น ไม่ใช่ทุกครั้งที่แก้โค้ด |
| บทเรียนจากอุตสาหกรรม | Versioning ต้องมาคู่กับกลยุทธ์ deprecation ที่ให้เวลาผู้ใช้ API เพียงพอ |

---

## ขั้นตอนที่ 472: เปรียบเทียบ `URLPathVersioning`, `NamespaceVersioning`, `AcceptHeaderVersioning`, `QueryParameterVersioning`

### 472.1 กลไก Versioning ของ DRF ทำงานอย่างไรในภาพรวม

DRF มี versioning scheme สำเร็จรูปให้ 5 แบบ (4 แบบหลักที่เราจะเรียนในขั้นตอนนี้ บวก
`HostNameVersioning` ที่พบน้อยกว่ามาก) ทุกแบบทำหน้าที่เดียวกันคือ: **อ่านค่าเวอร์ชันจาก
request แล้วเซ็ตไว้ที่ `request.version`** ให้ view เอาไปใช้ตัดสินใจต่อว่าจะ serialize
ข้อมูลแบบไหน

ตั้งค่าเริ่มต้นแบบ global ได้ที่ `settings.py`:

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],

    # ตั้งค่า Versioning
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',                 # ใช้เมื่อ client ไม่ระบุเวอร์ชันมาเลย
    'ALLOWED_VERSIONS': ['v1', 'v2'],        # เวอร์ชันที่ยอมรับ เวอร์ชันอื่นจะได้ 404
    'VERSION_PARAM': 'version',              # ชื่อ parameter (ใช้กับบาง scheme)
}
```

หรือกำหนดเฉพาะ view เดียวก็ได้ผ่าน attribute `versioning_class` โดยไม่กระทบ view อื่น:

```python
# blog/api_views.py
from rest_framework.versioning import AcceptHeaderVersioning
from rest_framework.views import APIView


class PostListAPIView(APIView):
    versioning_class = AcceptHeaderVersioning
```

### 472.2 แบบที่ 1: `URLPathVersioning`

เวอร์ชันฝังอยู่ใน **path ของ URL เอง** เป็นแบบที่ **เห็นชัดที่สุดและใช้กันมากที่สุดใน
อุตสาหกรรม**:

```python
# config/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/<str:version>/', include('blog.api_urls')),
]
```

```bash
curl http://127.0.0.1:8000/api/v1/posts/
curl http://127.0.0.1:8000/api/v2/posts/
```

DRF จะดึงค่า `version` จาก URL keyword argument ที่ชื่อตรงกับ `VERSION_PARAM`
(ค่า default คือ `'version'`) มาเซ็ตเป็น `request.version` โดยอัตโนมัติ

### 472.3 แบบที่ 2: `NamespaceVersioning`

แนวคิดคล้าย `URLPathVersioning` (เวอร์ชันยังอยู่ใน URL) แต่ใช้กลไก **URL namespace**
ของ Django แทนการอ่านจาก URL kwarg ตรง ๆ — เหมาะเมื่อแต่ละเวอร์ชันมี `urls.py` แยกไฟล์
กันชัดเจน:

```python
# blog/api_urls_v1.py
from django.urls import path
from . import api_views_v1 as views

app_name = 'blog'   # ต้องตรงกับที่ include namespace ไว้

urlpatterns = [
    path('posts/', views.PostListAPIViewV1.as_view(), name='post-list'),
    path('posts/<slug:slug>/', views.PostDetailAPIViewV1.as_view(), name='post-detail'),
]
```

```python
# blog/api_urls_v2.py
from django.urls import path
from . import api_views_v2 as views

app_name = 'blog'

urlpatterns = [
    path('posts/', views.PostListAPIViewV2.as_view(), name='post-list'),
    path('posts/<slug:slug>/', views.PostDetailAPIViewV2.as_view(), name='post-detail'),
]
```

```python
# config/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/v1/', include(('blog.api_urls_v1', 'blog'), namespace='v1')),
    path('api/v2/', include(('blog.api_urls_v2', 'blog'), namespace='v2')),
]
```

`NamespaceVersioning` อ่านค่าเวอร์ชันจาก `request.resolver_match.namespace` (คือ `v1`
หรือ `v2` ตามที่ตั้งไว้ตอน `include(...)`) ข้อดีคือแยกไฟล์ view ของแต่ละเวอร์ชันออกจาก
กันชัดเจนตั้งแต่ระดับ URL config ทำให้ไม่มีทาง "ลืม if version == 'v2'" ในโค้ดเดียวปนกัน

### 472.4 แบบที่ 3: `AcceptHeaderVersioning`

เวอร์ชันไม่อยู่ใน URL เลย แต่ฝังอยู่ใน HTTP header `Accept` ตามหลัก **Content
Negotiation** ของ HTTP (แนวทางที่ Stripe และ GitHub ใช้ แม้จะใช้ header คนละชื่อ):

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': '1.0',
    'ALLOWED_VERSIONS': ['1.0', '2.0'],
}
```

```python
# config/urls.py — ไม่ต้องมี version ใน path เลย
urlpatterns = [
    path('api/', include('blog.api_urls')),
]
```

```bash
curl http://127.0.0.1:8000/api/posts/ \
  -H "Accept: application/json; version=2.0"
```

ข้อดีคือ URL สะอาด (มีแค่ `/api/posts/` ตัวเดียวตลอดกาล) ตรงตามหลัก REST ที่ว่า **URL
ควรระบุ "ทรัพยากร" ไม่ใช่ "เวอร์ชัน"** แต่ข้อเสียคือทดสอบยากกว่า (พิมพ์ URL ใน browser
เฉย ๆ ไม่ได้ ต้องตั้ง header เสมอ) และ **cache ยากกว่า** เพราะ URL เดียวกันคืนผลต่างกัน
ตาม header (ต้องตั้ง `Vary: Accept` ให้ถูกต้องที่ระบบ cache/CDN ไม่งั้นจะแคชผิดเวอร์ชัน
ปนกัน)

### 472.5 แบบที่ 4: `QueryParameterVersioning`

เวอร์ชันอยู่ใน **query string** ท้าย URL:

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

```bash
curl "http://127.0.0.1:8000/api/posts/?version=v2"
```

ง่ายต่อการทดสอบด้วย browser ตรง ๆ (แค่แปะท้าย URL) แต่ในทางปฏิบัติ **ไม่ค่อยนิยมใน
production จริง** เพราะ query parameter ปนกับ parameter อื่น ๆ ที่ใช้ filter/search/
pagination (จาก Part 047) ทำให้ URL ดูรกและสับสนว่าอันไหนคือ "เวอร์ชัน" อันไหนคือ
"เงื่อนไขค้นหา"

### 472.6 ตารางเปรียบเทียบทั้ง 4 แบบแบบเต็ม

| ประเด็น | `URLPathVersioning` | `NamespaceVersioning` | `AcceptHeaderVersioning` | `QueryParameterVersioning` |
|---|---|---|---|---|
| ตำแหน่งเวอร์ชัน | ใน path (`/api/v2/...`) | ใน path (ผ่าน namespace) | ใน HTTP header `Accept` | ใน query string (`?version=v2`) |
| ทดสอบง่ายด้วย browser/curl เปล่า ๆ | ✅ ง่ายมาก | ✅ ง่ายมาก | ❌ ต้องตั้ง header เสมอ | ✅ ง่ายมาก |
| Cache-friendly (CDN/browser cache) | ✅ ดีมาก (URL ต่างกัน = cache key ต่างกันเอง) | ✅ ดีมาก | ⚠️ ต้องตั้ง `Vary: Accept` ให้ถูก ไม่งั้นแคชปน | ✅ ดี (แต่ query string บาง CDN ไม่แคชแยก) |
| ตรงตามหลัก REST purist (URL = ทรัพยากรเท่านั้น) | ❌ ถือว่าไม่ purist (version ไม่ใช่ทรัพยากร) | ❌ เช่นกัน | ✅ purist ที่สุด | ❌ เช่นกัน |
| แยกโค้ด view ของแต่ละเวอร์ชันออกจากกันได้ชัดเจนที่สุด | ⚠️ ทำได้แต่ต้อง if/else หรือ dict mapping เอง | ✅ แยกเป็นไฟล์ url/view คนละชุดเลย | ⚠️ ทำได้แต่ต้อง if/else เอง | ⚠️ ทำได้แต่ต้อง if/else เอง |
| ความนิยมในอุตสาหกรรมจริง | ⭐⭐⭐⭐⭐ นิยมที่สุด | ⭐⭐⭐ นิยมในโปรเจกต์ที่แยก app ชัดเจน | ⭐⭐⭐⭐ นิยมใน API ระดับ enterprise (Stripe, GitHub) | ⭐⭐ พบน้อยสุดใน production |
| Browsable API ของ DRF ใช้งานสะดวก | ✅ คลิกลิงก์ตรง ๆ ได้เลย | ✅ เช่นกัน | ❌ ต้องพิมพ์ header เอง | ✅ พิมพ์ query ต่อท้ายได้ |

### 472.7 หลักสูตรนี้เลือกใช้อะไรกับ `blog` API

หลักสูตรนี้เลือก **`URLPathVersioning`** เป็นหลักสำหรับ `blog` API ด้วยเหตุผล:

1. ทดสอบง่ายที่สุดสำหรับผู้เริ่มต้น (วาง URL ใน browser ก็เห็นผลทันที)
2. Cache-friendly โดยธรรมชาติ ไม่ต้องกังวลเรื่อง `Vary` header เพิ่ม
3. เป็นแบบที่พบเจอบ่อยที่สุดเมื่อไปทำงานจริงกับทีมอื่น หรืออ่านเอกสาร API สาธารณะทั่วไป
4. Browsable API ของ DRF (จาก Part 040) ยังคลิกไปมาระหว่างเวอร์ชันได้สะดวก

แต่โค้ดตัวอย่างในขั้นตอนที่ 473 จะออกแบบให้ **สลับไปใช้ scheme อื่นได้ง่าย** เพราะ logic
การเลือก serializer จะแยกออกจาก URL routing อย่างชัดเจน

### 472.8 ตารางสรุปขั้นตอนที่ 472

| Scheme | Class | Attribute ที่ต้องตั้ง |
|---|---|---|
| URL Path | `rest_framework.versioning.URLPathVersioning` | `<str:version>/` ใน urls.py |
| Namespace | `rest_framework.versioning.NamespaceVersioning` | `namespace=` ตอน `include()` |
| Accept Header | `rest_framework.versioning.AcceptHeaderVersioning` | Client ส่ง header `Accept: ...; version=X` |
| Query Parameter | `rest_framework.versioning.QueryParameterVersioning` | `?version=X` |

---

## ขั้นตอนที่ 473: Implement versioned Serializer/View จริง (`/api/v1/posts/` vs `/api/v2/posts/` ที่มี field ต่างกัน)

### 473.1 ออกแบบความแตกต่างระหว่าง v1 กับ v2

ทวนจากสถานการณ์จำลองในขั้นตอนที่ 471.2 — ตอนนี้เราจะ implement การเปลี่ยนแปลงนั้นจริง
โดยให้ **v1 ยังทำงานเหมือนเดิมทุกประการ** (ไม่ให้ client เก่าพัง) และ **v2 มีโครงสร้าง
ใหม่ที่ดีขึ้น**:

| Field | v1 | v2 |
|---|---|---|
| `author` | string (username เฉย ๆ) | object `{"id": 1, "username": "somchai"}` |
| `is_published` / `status` | `is_published`: boolean | `status`: string enum (`draft`/`published`) |
| `reading_time_minutes` | ไม่มี | มี (คำนวณจากจำนวนคำใน `content`) |
| `content` | มีเต็ม | มีเต็ม (ไม่เปลี่ยน) |

### 473.2 Serializer ของ v1 (เหมือนเดิมทุกประการจาก Part 046)

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post


class PostSerializerV1(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug', 'content',
            'is_published', 'created_at', 'updated_at',
        ]
```

### 473.3 Serializer ของ v2 (โครงสร้างใหม่)

```python
# blog/serializers.py (ต่อจากด้านบน)
from django.contrib.auth import get_user_model

User = get_user_model()


class AuthorSummarySerializer(serializers.ModelSerializer):
    """Nested serializer แสดงข้อมูลผู้เขียนแบบย่อ — ใช้เฉพาะใน v2"""

    class Meta:
        model = User
        fields = ['id', 'username']


class PostSerializerV2(serializers.ModelSerializer):
    author = AuthorSummarySerializer(read_only=True)
    status = serializers.SerializerMethodField()
    reading_time_minutes = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug', 'content',
            'status', 'reading_time_minutes', 'created_at', 'updated_at',
        ]

    def get_status(self, obj):
        # ในเวอร์ชันนี้ยังคำนวณจาก field is_published เดิมในฐานข้อมูล
        # (ยังไม่ต้องแก้ Model — เปลี่ยนแค่ "หน้าตา" ที่ client เห็นเท่านั้น)
        return 'published' if obj.is_published else 'draft'

    def get_reading_time_minutes(self, obj):
        word_count = len(obj.content.split())
        # สมมติฐานความเร็วการอ่านเฉลี่ย 200 คำ/นาที ปัดขึ้นอย่างน้อย 1 นาทีเสมอ
        return max(1, round(word_count / 200))
```

**สังเกตสิ่งสำคัญ**: `PostSerializerV2` **ไม่ได้แก้ `Post` model เลย** — `status` และ
`reading_time_minutes` คำนวณจาก field เดิม (`is_published`, `content`) ที่มีอยู่แล้ว
นี่คือรูปแบบที่พบบ่อยที่สุดของการทำ versioning ในทางปฏิบัติ: **Model และฐานข้อมูลเป็น
"ความจริงหนึ่งเดียว" (single source of truth) ส่วน Serializer แต่ละเวอร์ชันคือ "มุมมอง
(view/projection)" ที่ต่างกันของความจริงเดียวกันนั้น**

### 473.4 View ที่เลือก Serializer ตาม `request.version`

```python
# blog/api_views.py
from django.shortcuts import get_object_or_404
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView

from .models import Post
from .serializers import PostSerializerV1, PostSerializerV2


class VersionedSerializerMixin:
    """Mixin กลาง ใช้ซ้ำได้ทุก view ที่ต้องเลือก serializer ตามเวอร์ชัน"""

    serializer_class_by_version = {}   # ต้อง override ใน subclass

    def get_serializer_class(self):
        version = getattr(self.request, 'version', None)
        try:
            return self.serializer_class_by_version[version]
        except KeyError:
            # เวอร์ชันที่ไม่รู้จัก (ปกติ ALLOWED_VERSIONS จะกันไว้ตั้งแต่ต้นทางแล้ว)
            # แต่ใส่ fallback ไว้กันเหตุการณ์ไม่คาดฝันเสมอ
            return self.serializer_class_by_version[self.default_version]

    default_version = 'v1'


class PostListAPIView(VersionedSerializerMixin, APIView):
    serializer_class_by_version = {
        'v1': PostSerializerV1,
        'v2': PostSerializerV2,
    }

    def get(self, request):
        posts = Post.objects.filter(is_published=True).select_related('author')
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(posts, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetailAPIView(VersionedSerializerMixin, APIView):
    serializer_class_by_version = {
        'v1': PostSerializerV1,
        'v2': PostSerializerV2,
    }

    def get_object(self, slug):
        return get_object_or_404(Post.objects.select_related('author'), slug=slug)

    def get(self, request, slug):
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(self.get_object(slug))
        return Response(serializer.data)
```

### 473.5 ตั้งค่า URL Routing ด้วย `URLPathVersioning`

```python
# config/settings.py
REST_FRAMEWORK = {
    # ... (authentication/permission classes จาก Part 046)
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/<str:version>/', include('blog.api_urls')),

    # JWT authentication endpoints ไม่ต้องมี version (ไม่ใช่ resource ที่เปลี่ยนบ่อย)
    path('api/token/', include('blog.auth_urls')),
]
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

### 473.6 ทดสอบผลลัพธ์จริงด้วย `curl`

```bash
# v1 — โครงสร้างเดิม ไม่เปลี่ยนแปลง
curl -s http://127.0.0.1:8000/api/v1/posts/ | python -m json.tool
```

```json
[
  {
    "id": 1,
    "author": "somchai",
    "title": "สวัสดี Django",
    "slug": "สวัสดี-django",
    "content": "เนื้อหาโพสต์ตัวอย่างที่มีความยาวประมาณสี่ร้อยคำสำหรับทดสอบ...",
    "is_published": true,
    "created_at": "2026-09-01T10:00:00Z",
    "updated_at": "2026-09-01T10:00:00Z"
  }
]
```

```bash
# v2 — โครงสร้างใหม่
curl -s http://127.0.0.1:8000/api/v2/posts/ | python -m json.tool
```

```json
[
  {
    "id": 1,
    "author": {
      "id": 3,
      "username": "somchai"
    },
    "title": "สวัสดี Django",
    "slug": "สวัสดี-django",
    "content": "เนื้อหาโพสต์ตัวอย่างที่มีความยาวประมาณสี่ร้อยคำสำหรับทดสอบ...",
    "status": "published",
    "reading_time_minutes": 2,
    "created_at": "2026-09-01T10:00:00Z",
    "updated_at": "2026-09-01T10:00:00Z"
  }
]
```

```bash
# เวอร์ชันที่ไม่รู้จัก → 404 ทันที เพราะไม่อยู่ใน ALLOWED_VERSIONS
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/api/v3/posts/
# ผลลัพธ์: 404
```

### 473.7 ตารางสรุปขั้นตอนที่ 473

| องค์ประกอบ | v1 | v2 |
|---|---|---|
| Serializer | `PostSerializerV1` | `PostSerializerV2` |
| `author` | string | nested object |
| สถานะเผยแพร่ | `is_published` (boolean) | `status` (enum string) |
| Field พิเศษ | — | `reading_time_minutes` |
| Model เปลี่ยนหรือไม่ | ❌ ไม่แตะ | ❌ ไม่แตะ (คำนวณจาก field เดิม) |
| กลไกเลือก serializer | `VersionedSerializerMixin.get_serializer_class()` อ่านจาก `request.version` |

---

## ขั้นตอนที่ 474: กลยุทธ์ Deprecation ของ API version เก่า (deprecation header, sunset date, การสื่อสารกับผู้ใช้ API)

### 474.1 ทำไมแค่ "มี v2 แล้ว" ยังไม่พอ

เมื่อ `blog` API มี v2 พร้อมใช้งานจริงแล้ว (จากขั้นตอนที่ 473) เป้าหมายระยะยาวคือให้ทุก
client ย้ายจาก v1 ไป v2 แล้ว**ปลด v1 ทิ้งในที่สุด** (ไม่งั้นทีมต้อง maintain โค้ด 2
เวอร์ชันตลอดไป เพิ่มภาระทดสอบและความเสี่ยง bug เป็นสองเท่า) แต่การ**ปิด v1 กะทันหัน**
โดยไม่แจ้งล่วงหน้าคือสิ่งที่ทำให้เกิดดราม่าแบบกรณี Twitter API v1.1 ในขั้นตอนที่ 471.5
— บทเรียนคือ **ต้องมีกระบวนการ deprecation ที่ชัดเจนและให้เวลาเพียงพอ**

### 474.2 มาตรฐาน HTTP สำหรับประกาศ Deprecation: RFC 8594

มี HTTP header มาตรฐานสากล ([RFC 8594 — The Sunset HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc8594))
ที่ใช้ประกาศว่า resource หนึ่งกำลังจะถูกปลดระวาง:

| Header | ความหมาย | ตัวอย่างค่า |
|---|---|---|
| `Deprecation` | บอกว่า endpoint/version นี้ถูก deprecate แล้ว (บาง draft ใช้ `true` บางระบบใช้วันที่ที่เริ่ม deprecate) | `Deprecation: true` |
| `Sunset` | วันที่ที่ endpoint นี้จะ**หยุดทำงานจริง** (ตามมาตรฐาน HTTP-date) | `Sunset: Sat, 31 Dec 2026 23:59:59 GMT` |
| `Link` (rel="successor-version") | ชี้ไปยัง endpoint/version ใหม่ที่ควรย้ายไปใช้แทน | `Link: </api/v2/posts/>; rel="successor-version"` |

### 474.3 Implement ผ่าน Mixin ที่ใส่ Header อัตโนมัติ

```python
# blog/api_views.py
from django.utils.http import http_date
import datetime


class DeprecatedVersionMixin:
    """
    แปะ header มาตรฐาน RFC 8594 ให้ทุก response ของ view ที่เวอร์ชันถูก deprecate แล้ว
    ใช้ผ่าน dict ที่ map version → วันที่ sunset (None = ยังไม่ deprecate)
    """

    deprecated_versions = {}   # เช่น {'v1': datetime.date(2027, 6, 30)}
    successor_version_url = None   # เช่น '/api/v2/posts/'

    def finalize_response(self, request, response, *args, **kwargs):
        response = super().finalize_response(request, response, *args, **kwargs)
        sunset_date = self.deprecated_versions.get(getattr(request, 'version', None))
        if sunset_date is not None:
            sunset_datetime = datetime.datetime.combine(
                sunset_date, datetime.time.min, tzinfo=datetime.timezone.utc
            )
            response['Deprecation'] = 'true'
            response['Sunset'] = http_date(sunset_datetime.timestamp())
            if self.successor_version_url:
                response['Link'] = f'<{self.successor_version_url}>; rel="successor-version"'
        return response
```

**ข้อควรระวังสำคัญ**: `http_date()` ของ Django ต้องการ **Unix timestamp** (ตัวเลข
`float`/`int`) ไม่ใช่ `datetime` object ตรง ๆ จึงต้องเรียก `.timestamp()` ปิดท้ายเสมอ —
และ `datetime.datetime.combine(..., tzinfo=...)` ต้องระบุ `timezone.utc` ให้ครบ ไม่งั้น
จะได้ **naive datetime** ที่ไม่มี timezone กำกับ ซึ่งจะคำนวณ timestamp ผิดเพี้ยนไปตาม
timezone ของเครื่อง server แทนที่จะเป็น UTC ตามมาตรฐาน HTTP-date

นำไปใช้กับ view จริง:

```python
# blog/api_views.py
import datetime


class PostListAPIView(DeprecatedVersionMixin, VersionedSerializerMixin, APIView):
    serializer_class_by_version = {
        'v1': PostSerializerV1,
        'v2': PostSerializerV2,
    }
    deprecated_versions = {
        'v1': datetime.date(2027, 6, 30),   # ประกาศ sunset v1 ล่วงหน้า
    }
    successor_version_url = '/api/v2/posts/'

    # ... get()/post() เหมือนเดิมจากขั้นตอนที่ 473.4
```

ทดสอบด้วย `curl -i` (แสดง header):

```bash
curl -si http://127.0.0.1:8000/api/v1/posts/ | head -n 12
```

```
HTTP/1.1 200 OK
...
Deprecation: true
Sunset: Wed, 30 Jun 2027 00:00:00 GMT
Link: </api/v2/posts/>; rel="successor-version"
Content-Type: application/json
```

```bash
curl -si http://127.0.0.1:8000/api/v2/posts/ | head -n 12
# ไม่มี header Deprecation/Sunset เลย เพราะ v2 ยังไม่ถูก deprecate
```

### 474.4 กระบวนการ Deprecation แบบมืออาชีพ (Timeline)

| ระยะ | ระยะเวลาแนะนำ | สิ่งที่ต้องทำ |
|---|---|---|
| **1. ประกาศ (Announce)** | ทันทีที่ v2 พร้อมใช้งานจริง | เขียน changelog, ส่งอีเมลถึงผู้ถือ API Key ทุกราย, ประกาศใน developer portal/docs |
| **2. เตือนใน Response (Warning Headers)** | ตลอดช่วง grace period (แนะนำอย่างน้อย 6-12 เดือนสำหรับ public API) | แปะ `Deprecation`, `Sunset`, `Link` header ทุก response ของ v1 ตามขั้นตอนที่ 474.3 |
| **3. ติดตามการใช้งาน (Monitor Usage)** | ตลอดช่วง grace period | Log ทุก request ที่ยังเรียก v1 พร้อม identity ของ client (API Key/user) เพื่อดูว่าใครยังไม่ย้าย |
| **4. เตือนเชิงรุก (Proactive Outreach)** | 1-2 เดือนก่อนถึง sunset date | ติดต่อ client ที่ log แสดงว่ายังใช้ v1 อยู่โดยตรง (อีเมล/webhook แจ้งเตือน) |
| **5. Hard Sunset** | ถึงวันที่ตั้งไว้ใน header `Sunset` | v1 คืน `410 Gone` แทนข้อมูลจริง พร้อม body อธิบายว่าให้ย้ายไป v2 |
| **6. ลบโค้ดทิ้งถาวร** | หลัง Hard Sunset ผ่านไปอีกระยะหนึ่ง (เผื่อ client ที่ยัง cache request เก่า) | ลบ `PostSerializerV1`, view/url ของ v1 ออกจากโค้ดจริง |

### 474.5 Implement ระยะที่ 5 (Hard Sunset): คืน `410 Gone`

```python
# blog/api_views.py
from rest_framework.exceptions import APIException
from rest_framework import status


class VersionSunsetted(APIException):
    status_code = status.HTTP_410_GONE
    default_detail = (
        'API เวอร์ชันนี้ถูกปลดระวางแล้ว กรุณาย้ายไปใช้ /api/v2/ '
        'อ่านคู่มือการย้ายได้ที่ https://docs.example.com/migration/v1-to-v2'
    )
    default_code = 'version_sunsetted'


class SunsetCheckMixin:
    sunsetted_versions = set()   # เช่น {'v1'} หลังผ่าน hard sunset date แล้ว

    def initial(self, request, *args, **kwargs):
        super().initial(request, *args, **kwargs)
        if getattr(request, 'version', None) in self.sunsetted_versions:
            raise VersionSunsetted()
```

```python
# blog/api_views.py
class PostListAPIView(SunsetCheckMixin, DeprecatedVersionMixin, VersionedSerializerMixin, APIView):
    sunsetted_versions = set()   # ยังว่างอยู่จนกว่าจะถึงวันที่ 2027-06-30 จริง
    # ...
```

เมื่อถึงวันจริง ทีมแค่เพิ่ม `'v1'` เข้า `sunsetted_versions` (ผ่าน environment variable
หรือ feature flag ก็ได้ ไม่ต้อง deploy โค้ดใหม่) แล้ว client ที่ยังเรียก v1 อยู่จะได้
`410 Gone` พร้อมคำอธิบายที่ชัดเจนแทนที่จะได้ error กำกวมหรือข้อมูลผิดโครงสร้าง

### 474.6 การสื่อสารกับผู้ใช้ API นอกเหนือจาก HTTP Header

Header เป็นการสื่อสารกับ**โค้ด**ของ client แต่ยังต้องสื่อสารกับ**คนที่ดูแล**โค้ดนั้นด้วย:

- **Changelog สาธารณะ**: หน้าเว็บหรือไฟล์ `CHANGELOG.md` ที่บันทึกทุกการเปลี่ยนแปลงของ
  API พร้อมวันที่ ตัวอย่างรูปแบบที่ดี:

```markdown
## API Changelog

### 2026-09-15 — v2 Released
- `author` เปลี่ยนจาก string เป็น nested object `{id, username}`
- `is_published` (boolean) ถูกแทนที่ด้วย `status` (enum: draft/published)
- เพิ่ม field `reading_time_minutes`
- **v1 จะถูก sunset วันที่ 2027-06-30** กรุณาวางแผนย้ายก่อนวันดังกล่าว
```

- **อีเมลถึงผู้ถือ API Key**: ใช้ตาราง `PartnerAPIKey` จาก Part 046 (ขั้นตอนที่ 457) ส่ง
  แจ้งเตือนอัตโนมัติผ่าน Celery periodic task (จะเรียนเต็มรูปแบบใน Phase Background
  Tasks) ไปยังอีเมลที่ผูกกับแต่ละ API Key
- **Developer Portal / API Documentation**: อัปเดตเอกสาร (จะเรียนเครื่องมือ
  `drf-spectacular` ใน Part 049) ให้ขึ้นแบนเนอร์เตือนชัดเจนบนหน้า v1

### 474.7 ตารางสรุปขั้นตอนที่ 474

| เครื่องมือ/แนวคิด | หน้าที่ |
|---|---|
| `Deprecation: true` header | บอก client (โปรแกรม) ว่าเวอร์ชันนี้เลิกพัฒนาต่อแล้ว |
| `Sunset: <date>` header | บอกวันที่ที่จะหยุดทำงานจริง (RFC 8594) |
| `Link: rel="successor-version"` | ชี้ทางไปเวอร์ชันใหม่ให้อัตโนมัติ |
| Grace period 6-12 เดือน | เวลาที่สมเหตุสมผลให้ client ภายนอกย้ายเวอร์ชัน |
| `410 Gone` หลัง hard sunset | ปฏิเสธ request ของเวอร์ชันเก่าแบบชัดเจน ไม่ทิ้งให้ error กำกวม |
| Changelog + อีเมล + developer portal | สื่อสารกับ**คน** ไม่ใช่แค่โค้ด |

---

## ขั้นตอนที่ 475: Throttling Class — `AnonRateThrottle`, `UserRateThrottle`

### 475.1 Throttling ต่างจาก Rate Limiting ทั่วไปอย่างไร

หลายคนใช้คำว่า "throttling" กับ "rate limiting" สลับกันไปมา ในบริบทของ DRF **throttling**
หมายถึงกลไกที่ **จำกัดจำนวนครั้งที่ client หนึ่ง ๆ เรียก API ได้ในช่วงเวลาหนึ่ง** เพื่อ
ป้องกัน:

- **Brute-force attack** บน endpoint ที่อ่อนไหว เช่น login (ทวนจาก Part 046 ขั้นตอนที่
  459.5)
- **Denial of Service** จาก client ที่ยิง request ถี่เกินไป (ตั้งใจหรือไม่ตั้งใจก็ตาม
  เช่น bug ใน client ที่ loop ยิง request ไม่หยุด)
- **การใช้ทรัพยากรฝ่ายเดียวมากเกินไป** จน client รายอื่นได้รับผลกระทบ (fair usage)

### 475.2 กลไกภายในของ `SimpleRateThrottle`

Throttle class ทั้งหมดของ DRF สืบทอดจาก `SimpleRateThrottle` ซึ่งใช้ **Django's cache
framework** (จาก Part 021) เป็นที่เก็บ "ประวัติการเรียก" ของแต่ละ client:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.throttling.SimpleRateThrottle
class SimpleRateThrottle(BaseThrottle):
    cache = default_cache          # ใช้ cache alias 'default' โดย default
    timer = time.time
    cache_format = 'throttle_%(scope)s_%(ident)s'

    def allow_request(self, request, view):
        if self.rate is None:
            return True

        self.key = self.get_cache_key(request, view)
        if self.key is None:
            return True   # ไม่มี cache key แปลว่าไม่ต้อง throttle request นี้

        self.history = self.cache.get(self.key, [])
        self.now = self.timer()

        # ทิ้ง timestamp ที่เก่าเกิน duration ออกจาก history
        while self.history and self.history[-1] <= self.now - self.duration:
            self.history.pop()

        if len(self.history) >= self.num_requests:
            return self.throttle_failure()
        return self.throttle_success()

    def throttle_success(self):
        self.history.insert(0, self.now)
        self.cache.set(self.key, self.history, self.duration)
        return True
```

สรุปเป็นภาษาคนคือ: ทุกครั้งที่มี request เข้ามา DRF จะดู "ประวัติเวลาที่ client นี้เคย
เรียกมาก่อน" ที่เก็บไว้ใน cache ถ้าจำนวนครั้งภายในช่วงเวลาที่กำหนด (`duration`) ยังไม่ถึง
เพดาน (`num_requests`) ก็อนุญาตให้ผ่าน แล้วบันทึกเวลาปัจจุบันเพิ่มเข้าไปใน history —
**นี่คือเหตุผลที่ขั้นตอนที่ 478 (throttling + caching) สำคัญมาก**: ถ้า cache backend
ไม่แชร์ข้อมูลข้ามหลาย process/server การนับจะไม่แม่นยำ

### 475.3 `AnonRateThrottle`: จำกัดตาม IP สำหรับผู้ใช้ที่ไม่ login

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.throttling
class AnonRateThrottle(SimpleRateThrottle):
    scope = 'anon'

    def get_cache_key(self, request, view):
        if request.user and request.user.is_authenticated:
            return None   # ผู้ใช้ที่ login แล้ว ไม่ต้องถูก throttle ด้วยคลาสนี้
        return self.cache_format % {
            'scope': self.scope,
            'ident': self.get_ident(request),   # ระบุตัวตนด้วย IP address
        }
```

`get_ident(request)` อ่านค่า IP จาก header `X-Forwarded-For` (ถ้ามี, กรณีอยู่หลัง
reverse proxy/load balancer) หรือ `REMOTE_ADDR` เป็นค่า fallback

### 475.4 `UserRateThrottle`: จำกัดตาม User ID สำหรับผู้ใช้ที่ login แล้ว

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.throttling
class UserRateThrottle(SimpleRateThrottle):
    scope = 'user'

    def get_cache_key(self, request, view):
        if request.user and request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)
        return self.cache_format % {
            'scope': self.scope,
            'ident': ident,
        }
```

สังเกตว่า `UserRateThrottle` ทำงาน**ทั้งกับผู้ใช้ที่ login แล้วและยังไม่ได้ login**
(ใช้ IP แทนตอนยังไม่ login) จึงนิยมตั้งให้ทำงานคู่กับ `AnonRateThrottle` เสมอเพื่อให้
ครอบคลุมทั้งสองกรณีด้วย rate ที่ต่างกัน (ผู้ใช้ที่ login แล้วมักได้ rate ที่สูงกว่า
เพราะระบุตัวตนได้ชัดเจนกว่า และมักไม่ใช่ bot)

### 475.5 ตั้งค่าแบบ Global ใน `settings.py`

```python
# config/settings.py
REST_FRAMEWORK = {
    # ... (versioning settings จากขั้นตอนที่ 473)
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',     # ผู้ใช้ไม่ login: 100 ครั้ง/วัน ต่อ IP
        'user': '1000/day',    # ผู้ใช้ login แล้ว: 1000 ครั้ง/วัน ต่อ user
    },
}
```

รูปแบบของ rate string คือ `'<จำนวนครั้ง>/<หน่วยเวลา>'` โดยหน่วยเวลาที่รองรับ:

| ตัวย่อ | ความหมาย |
|---|---|
| `s` หรือ `sec` | วินาที |
| `m` หรือ `min` | นาที |
| `h` หรือ `hour` | ชั่วโมง |
| `d` หรือ `day` | วัน |

ตัวอย่าง: `'5/min'`, `'100/hour'`, `'1000/day'`

### 475.6 ผลลัพธ์เมื่อถูก Throttle: `429 Too Many Requests`

เมื่อ client เรียกเกินเพดานที่กำหนด DRF จะคืน response แบบนี้อัตโนมัติ:

```bash
curl -si http://127.0.0.1:8000/api/v2/posts/
```

```
HTTP/1.1 429 Too Many Requests
Retry-After: 3600
Content-Type: application/json

{"detail":"Request was throttled. Expected available in 3600 seconds."}
```

Header `Retry-After` (หน่วยวินาที) บอก client ว่าต้องรออีกเท่าไหร่ถึงจะเรียกได้อีกครั้ง
— client ที่ดีควรอ่านค่านี้แล้วหยุดยิง request ซ้ำจนกว่าจะครบเวลา แทนที่จะยิงรัว ๆ ต่อไป
(ซึ่งจะยิ่งโดน throttle ซ้ำไม่มีที่สิ้นสุด)

### 475.7 ตั้งค่าเฉพาะ View เดียว (Override ค่า Global)

```python
# blog/api_views.py
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle


class PostListAPIView(APIView):
    throttle_classes = [AnonRateThrottle, UserRateThrottle]
    # ...
```

ถ้า view ไหนไม่ต้องการ throttle เลย (เช่น health-check endpoint ที่ monitoring
เรียกถี่มากโดยเจตนา) ตั้งเป็น list ว่างได้:

```python
class HealthCheckAPIView(APIView):
    throttle_classes = []   # ปิด throttling สำหรับ view นี้โดยเฉพาะ
    permission_classes = []
    authentication_classes = []

    def get(self, request):
        return Response({'status': 'ok'})
```

### 475.8 ตารางสรุปขั้นตอนที่ 475

| Throttle Class | ใช้กับใคร | ระบุตัวตนด้วย | Scope ค่า default |
|---|---|---|---|
| `AnonRateThrottle` | ผู้ใช้ที่ยังไม่ login เท่านั้น | IP address (`get_ident`) | `'anon'` |
| `UserRateThrottle` | ทั้งผู้ใช้ที่ login แล้วและยังไม่ login | User PK (login แล้ว) หรือ IP (ยังไม่ login) | `'user'` |
| ผลลัพธ์เมื่อเกินเพดาน | `429 Too Many Requests` + header `Retry-After` (วินาที) |
| ที่เก็บ "ประวัติการเรียก" | Django cache framework (ตาม `cache` attribute ของ throttle class) |

---

## ขั้นตอนที่ 476: เขียน Custom Throttle Class เอง (`ScopedRateThrottle` และ throttle แบบกำหนดเอง)

### 476.1 ปัญหาของ `AnonRateThrottle`/`UserRateThrottle`: Rate เดียวใช้กับทุก endpoint

`AnonRateThrottle` และ `UserRateThrottle` ใช้ scope คงที่ (`'anon'`, `'user'`) ตายตัว
หมายความว่าถ้าตั้ง `'user': '1000/day'` ไว้ ทุก endpoint ที่ user คนนั้นเรียก **จะแชร์
เพดานเดียวกันทั้งหมด** — แต่ในความเป็นจริง endpoint ต่าง ๆ มีความอ่อนไหวไม่เท่ากัน เช่น
`GET /api/posts/` ควรเรียกได้บ่อยกว่า `POST /api/token/` (login) มาก DRF จึงมี
`ScopedRateThrottle` มาแก้ปัญหานี้โดยเฉพาะ

### 476.2 `ScopedRateThrottle`: กำหนด Scope ต่อ View ได้อิสระ

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.throttling.ScopedRateThrottle
class ScopedRateThrottle(SimpleRateThrottle):
    scope_attr = 'throttle_scope'

    def allow_request(self, request, view):
        self.scope = getattr(view, self.scope_attr, None)
        if not self.scope:
            return True   # view ไม่ได้กำหนด throttle_scope ไว้ = ไม่ throttle
        self.rate = self.get_rate()
        self.num_requests, self.duration = self.parse_rate(self.rate)
        return super().allow_request(request, view)

    def get_cache_key(self, request, view):
        if request.user and request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)
        return self.cache_format % {'scope': self.scope, 'ident': ident}
```

ใช้งานโดยกำหนด attribute `throttle_scope` ที่ view แล้วตั้ง rate แยกในตาราง
`DEFAULT_THROTTLE_RATES` ตาม scope นั้น:

```python
# blog/api_views.py
from rest_framework.throttling import ScopedRateThrottle


class PostListAPIView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'posts'
    # ...


class PostSearchAPIView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'search'
    # ...
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'posts': '500/hour',    # endpoint อ่านข้อมูลทั่วไป เรียกได้ถี่กว่า
        'search': '30/min',     # endpoint ค้นหา (มักหนักกว่าเพราะ query ซับซ้อน)
    },
}
```

**ข้อควรระวังสำคัญ**: `ScopedRateThrottle` ใช้ **แทนที่** `AnonRateThrottle`/
`UserRateThrottle` ไม่ได้ใช้ร่วมกันในความหมายที่ครอบคลุมกรณีเดียวกัน — ถ้าต้องการทั้ง
เพดานรวมของทั้งระบบ (global) และเพดานเฉพาะ endpoint (scoped) ต้องใส่ throttle class
หลายตัวพร้อมกันใน `throttle_classes` (DRF จะเช็คทุกตัว ถ้าตัวใดตัวหนึ่งไม่ผ่านคือ throttle
ทันที)

### 476.3 เขียน Custom Throttle Class เองแบบเต็มรูปแบบ: Burst + Sustained Rate

บาง endpoint ต้องการเพดาน **2 ระดับพร้อมกัน**: จำกัดไม่ให้ยิงถี่เกินไปในช่วงสั้น ๆ
(burst) **และ** จำกัดจำนวนรวมทั้งวันด้วย (sustained) — แนวทางนี้คือสิ่งที่ API ระดับโลก
อย่าง GitHub ใช้จริง:

```python
# blog/throttles.py
from rest_framework.throttling import UserRateThrottle


class BurstRateThrottle(UserRateThrottle):
    """จำกัดไม่ให้ยิงถี่เกินไปในช่วงเวลาสั้น ๆ (ป้องกัน spike กะทันหัน)"""
    scope = 'burst'


class SustainedRateThrottle(UserRateThrottle):
    """จำกัดจำนวนรวมทั้งวัน (ป้องกันการใช้งานเกินโควตาระยะยาว)"""
    scope = 'sustained'
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',       # ยิงได้ไม่เกิน 60 ครั้งใน 1 นาทีใด ๆ
        'sustained': '1000/day', # รวมทั้งวันไม่เกิน 1000 ครั้ง
    },
}
```

```python
# blog/api_views.py
from .throttles import BurstRateThrottle, SustainedRateThrottle


class PostListAPIView(APIView):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
    # ...
```

Client ที่ยิง 100 request รวดในนาทีแรกจะโดน `BurstRateThrottle` บล็อกทันทีตั้งแต่ครั้งที่
61 แม้ว่าโควตารวมทั้งวัน (1000 ครั้ง) จะยังเหลืออีกมากก็ตาม — นี่คือการป้องกัน "burst
traffic" ที่อาจทำให้ server รับภาระหนักเกินไปในช่วงเวลาสั้น ๆ แม้ปริมาณรวมทั้งวันจะไม่
มากก็ตาม

### 476.4 เขียน Custom Throttle Class ที่ปรับ Rate ตาม Role ของผู้ใช้ (Dynamic Rate)

ต่อยอดจาก custom claims (`role`, `is_premium`) ที่เพิ่มเข้า JWT payload ใน Part 046
ขั้นตอนที่ 455 — สมมติต้องการให้ผู้ใช้ที่เป็นสมาชิกพรีเมียม (`is_premium=True`) ได้เพดาน
การเรียก API สูงกว่าผู้ใช้ทั่วไป การเขียน throttle class เองแบบเต็มรูปแบบทำได้โดย
override `allow_request()` เพื่อสลับ `scope`/`rate` แบบไดนามิกตามข้อมูลจริงในฐานข้อมูล
(ย้ำหลักการจาก Part 046 ขั้นตอนที่ 455.6: **ห้ามเชื่อ custom claims ใน JWT เพื่อการ
ตัดสินใจที่สำคัญ** จึงต้องอ่าน `request.user.profile.is_premium` จาก DB จริง ไม่ใช่จาก
payload ของ token):

```python
# blog/throttles.py
from rest_framework.throttling import UserRateThrottle


class RoleAwareUserRateThrottle(UserRateThrottle):
    """
    ปรับ scope แบบไดนามิกตามสถานะสมาชิกของผู้ใช้จริงในฐานข้อมูล
    (ไม่ใช่จาก JWT custom claims — ตามหลักการจาก Part 046 ขั้นตอนที่ 455.6)
    """

    def allow_request(self, request, view):
        if request.user and request.user.is_authenticated:
            is_premium = getattr(
                getattr(request.user, 'profile', None), 'is_premium', False
            )
            self.scope = 'user_premium' if is_premium else 'user_standard'
            self.rate = self.get_rate()
            self.num_requests, self.duration = self.parse_rate(self.rate)
        return super().allow_request(request, view)
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'user_standard': '1000/day',
        'user_premium': '10000/day',   # สมาชิกพรีเมียมได้โควตาสูงกว่า 10 เท่า
    },
}
```

**อธิบายกลไก**: `get_rate()` ของ `SimpleRateThrottle` อ่านค่าจาก
`settings.DEFAULT_THROTTLE_RATES[self.scope]` ดังนั้นการเปลี่ยน `self.scope` ก่อนเรียก
`super().allow_request()` จึงทำให้ throttle ไปอ่าน rate คนละค่ากันได้ตามเงื่อนไขที่เขียน
เอง — นี่คือรูปแบบที่ยืดหยุ่นที่สุดเมื่อ `ScopedRateThrottle` (ที่ผูก scope ตายตัวกับ
view) ไม่พอสำหรับ requirement ที่ซับซ้อนกว่านั้น

### 476.5 เขียน Custom Throttle Class โดยไม่พึ่งฐาน `UserRateThrottle`/`AnonRateThrottle` เลย: Throttle ตาม API Key

Endpoint สำหรับ partner ภายนอกที่ใช้ API Key (จาก Part 046 ขั้นตอนที่ 457) ไม่มี
`request.user` ที่ authenticated ในความหมายปกติ (เพราะ `HasPartnerAPIKey` เป็น
permission class ไม่ใช่ authentication class) จึงต้องเขียน `get_cache_key()` เองที่
ระบุตัวตนจาก API Key header โดยตรง:

```python
# blog/throttles.py
from rest_framework.throttling import SimpleRateThrottle


class PartnerAPIKeyThrottle(SimpleRateThrottle):
    """Throttle แยกโควตาตาม API Key แต่ละดวง แทนที่จะรวมกันทั้งหมดเป็น IP เดียว"""

    scope = 'partner_api_key'

    def get_cache_key(self, request, view):
        api_key = request.META.get('HTTP_X_API_KEY') or request.META.get(
            'HTTP_AUTHORIZATION', ''
        ).removeprefix('Api-Key ')
        if not api_key:
            return None   # ไม่มี API Key ส่งมา ให้ permission class จัดการปฏิเสธแทน
        return self.cache_format % {
            'scope': self.scope,
            'ident': api_key,   # ใช้ตัว key เองเป็น identity (แต่ละ partner แยกโควตากัน)
        }
```

```python
# blog/api_views.py
from .throttles import PartnerAPIKeyThrottle


class PartnerPostSyncAPIView(APIView):
    authentication_classes = []
    permission_classes = [HasPartnerAPIKey]
    throttle_classes = [PartnerAPIKeyThrottle]
    # ...
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'partner_api_key': '10000/day',   # ต่อ 1 API Key
    },
}
```

### 476.6 ตารางสรุปขั้นตอนที่ 476

| Throttle Class | ปัญหาที่แก้ | Scope กำหนดอย่างไร |
|---|---|---|
| `ScopedRateThrottle` (built-in) | Rate ต่างกันตาม endpoint โดยไม่ต้องเขียน class ใหม่ทุกครั้ง | attribute `throttle_scope` บน view |
| `BurstRateThrottle` + `SustainedRateThrottle` (custom) | ต้องการเพดาน 2 ระดับพร้อมกัน (สั้น+ยาว) | Scope คงที่ 2 ค่า ใช้พร้อมกันใน `throttle_classes` |
| `RoleAwareUserRateThrottle` (custom) | Rate ต้องต่างกันตาม role/สถานะสมาชิกที่เก็บใน DB | Override `allow_request()` สลับ `self.scope` แบบไดนามิก |
| `PartnerAPIKeyThrottle` (custom) | ไม่มี `request.user` ที่ authenticated แต่ต้องแยกโควตาตาม identity อื่น | Override `get_cache_key()` อ่านจาก header เอง |

---

## ขั้นตอนที่ 477: ตั้งค่า Throttle เฉพาะ endpoint (เช่น login endpoint throttle เข้มกว่าปกติ)

### 477.1 ทวนจาก Part 046 ขั้นตอนที่ 459.5

ใน Part 046 เราเกริ่นไว้สั้น ๆ ว่า endpoint `/api/token/` (login) ควรถูกจำกัดเข้มกว่า
endpoint ทั่วไปเพราะเป็นเป้าหมายอันดับหนึ่งของการโจมตีแบบ brute-force ตอนนี้เราจะ
implement ให้สมบูรณ์โดยใช้ทุกแนวคิดที่เรียนมาในขั้นตอนที่ 475-476

```python
# blog/throttles.py
from rest_framework.throttling import AnonRateThrottle


class LoginRateThrottle(AnonRateThrottle):
    """
    Throttle เข้มเป็นพิเศษสำหรับ endpoint login — ป้องกัน brute-force
    ทวนจาก Part 046 ขั้นตอนที่ 459.5
    """
    scope = 'login'
```

```python
# blog/api_views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import CustomTokenObtainPairSerializer
from .throttles import LoginRateThrottle


class ThrottledTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
    throttle_classes = [LoginRateThrottle]
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'login': '5/min',        # ป้องกัน brute-force: แค่ 5 ครั้ง/นาที ต่อ IP
        'password_reset': '3/hour',
        'search': '30/min',
    },
}
```

**เหตุผลที่ใช้ `AnonRateThrottle` เป็นฐานแทน `UserRateThrottle`**: ตอนที่ผู้ใช้ยังไม่
login สำเร็จ ระบบยังไม่รู้ว่าเป็น user คนไหน (นั่นคือสิ่งที่ endpoint นี้กำลังพยายาม
พิสูจน์) จึงต้องระบุตัวตนด้วย **IP address** เท่านั้น ซึ่งเป็นสิ่งเดียวที่มีอยู่ก่อน
authentication จะสำเร็จ

### 477.2 Throttle ต่างกันตาม HTTP Method ในหน้าเดียวกัน (`get_throttles()`)

บาง endpoint มี method ที่ความเสี่ยงต่างกันมาก เช่น `GET /api/v2/posts/` (แค่อ่าน)
ควรเรียกได้บ่อยกว่ามาก เทียบกับ `POST /api/v2/posts/` (เขียนข้อมูลใหม่ ซึ่งกิน
ทรัพยากรฐานข้อมูลมากกว่าและเสี่ยงต่อ spam content) DRF อนุญาตให้ override
`get_throttles()` เพื่อเลือก throttle class ตาม `self.request.method` ได้:

```python
# blog/api_views.py
from .throttles import BurstRateThrottle, SustainedRateThrottle
from rest_framework.throttling import ScopedRateThrottle


class PostListAPIView(VersionedSerializerMixin, APIView):
    serializer_class_by_version = {'v1': PostSerializerV1, 'v2': PostSerializerV2}

    def get_throttles(self):
        if self.request.method == 'POST':
            # เขียนข้อมูลใหม่ — throttle เข้มกว่า
            self.throttle_scope = 'posts_write'
            throttle_classes = [ScopedRateThrottle]
        else:
            # อ่านข้อมูล — throttle หลวมกว่า
            self.throttle_scope = 'posts_read'
            throttle_classes = [ScopedRateThrottle]
        return [throttle() for throttle in throttle_classes]

    def get(self, request):
        # ... เหมือนขั้นตอนที่ 473.4
        ...

    def post(self, request):
        # ... เหมือนขั้นตอนที่ 473.4
        ...
```

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'posts_read': '500/hour',
        'posts_write': '20/hour',   # เขียนได้จำกัดกว่าอ่านมาก
    },
}
```

**ข้อควรระวัง**: `ScopedRateThrottle.allow_request()` อ่านค่า `self.scope` จาก
`getattr(view, self.scope_attr, None)` (คือ `view.throttle_scope`) ทุกครั้งที่ถูกเรียก
ดังนั้นการตั้ง `self.throttle_scope = '...'` **ก่อน** สร้าง instance ของ
`ScopedRateThrottle` ใน `get_throttles()` ด้านบนจึงทำงานถูกต้อง เพราะ DRF เรียก
`get_throttles()` ก่อนเสมอในขั้นตอน `initial()` ของ request lifecycle

### 477.3 ตารางสรุปเพดาน Throttle ทั้งหมดของ `blog` API

| Endpoint | Method | Throttle Class | Rate | เหตุผล |
|---|---|---|---|---|
| `/api/token/` | `POST` | `LoginRateThrottle` (scope `login`) | `5/min` ต่อ IP | ป้องกัน brute-force รหัสผ่าน |
| `/api/token/refresh/` | `POST` | `AnonRateThrottle` (scope `anon`) | `100/day` ต่อ IP | ป้องกันการยิง refresh รัว ๆ ผิดปกติ |
| `/api/v2/posts/` | `GET` | `ScopedRateThrottle` (scope `posts_read`) | `500/hour` | อ่านข้อมูลทั่วไป เรียกถี่ได้ |
| `/api/v2/posts/` | `POST` | `ScopedRateThrottle` (scope `posts_write`) | `20/hour` | เขียนข้อมูลใหม่ กินทรัพยากรมากกว่า |
| `/api/v2/posts/search/` (Part 047) | `GET` | `ScopedRateThrottle` (scope `search`) | `30/min` | query ค้นหามักหนักกว่า query ทั่วไป |
| `/api/partner/posts/` | `GET` | `PartnerAPIKeyThrottle` | `10000/day` ต่อ API Key | Machine-to-machine โควตาสูงแต่แยกตาม partner |
| ทุก endpoint ที่เหลือ (default) | ทุก method | `AnonRateThrottle` + `UserRateThrottle` | `100/day` (anon), `1000/day` (user) | เพดานพื้นฐานทั่วทั้งระบบ |

### 477.4 ทดสอบว่า Throttle เข้มขึ้นจริงบน Login Endpoint

```bash
# ยิง login ผิดรหัสผ่านซ้ำ ๆ 6 ครั้งติดกัน (rate จำกัดไว้แค่ 5/min)
for i in $(seq 1 6); do
  echo "ครั้งที่ $i:"
  curl -s -o /dev/null -w "  status: %{http_code}\n" \
    -X POST http://127.0.0.1:8000/api/token/ \
    -H "Content-Type: application/json" \
    -d '{"username": "somchai", "password": "wrong-password"}'
done
```

```
ครั้งที่ 1:
  status: 401
ครั้งที่ 2:
  status: 401
ครั้งที่ 3:
  status: 401
ครั้งที่ 4:
  status: 401
ครั้งที่ 5:
  status: 401
ครั้งที่ 6:
  status: 429
```

สังเกตว่าครั้งที่ 1-5 ยังคืน `401 Unauthorized` (รหัสผ่านผิดจริง) แต่พอถึงครั้งที่ 6
กลไก throttle ตัดหน้าก่อนที่ view จะได้ประมวลผล username/password ด้วยซ้ำ — คืน
`429 Too Many Requests` ทันที นี่คือพฤติกรรมที่ถูกต้อง เพราะ**การตรวจ throttle เกิดขึ้น
ก่อนการตรวจ authentication เสมอ** (ใน `APIView.initial()` ลำดับคือ
`perform_authentication()` → `check_permissions()` → `check_throttles()`... ที่จริงแล้ว
throttle ถูกเช็คเป็นขั้นตอนสุดท้ายใน `initial()` แต่ยังคง**ก่อน**ที่ `handler()` ของ
view เช่น `post()` จะถูกเรียก ทำให้ป้องกัน brute-force ได้ตั้งแต่ก่อนแตะ logic ภายในเลย)

### 477.5 ตารางสรุปขั้นตอนที่ 477

| แนวคิด | สรุป |
|---|---|
| Throttle ต่างกันตาม endpoint | ใช้ `ScopedRateThrottle` + `throttle_scope` attribute ต่อ view |
| Throttle ต่างกันตาม method ในหน้าเดียว | Override `get_throttles()` เช็ค `self.request.method` |
| Login endpoint ต้องเข้มที่สุด | `LoginRateThrottle` scope `'login'` rate ต่ำมาก (`5/min`) |
| ลำดับการตรวจสอบใน request lifecycle | Throttle ถูกเช็คใน `initial()` ก่อนเรียก handler เสมอ — บล็อกได้ก่อนแตะ logic |

---

## ขั้นตอนที่ 478: ผสาน Throttling กับ Caching (เกริ่นสั้น ๆ เจาะลึกเต็มใน Part 068)

### 478.1 ปัญหาที่ซ่อนอยู่: Cache Backend เริ่มต้นของ Django ไม่เหมาะกับ Production

ทวนจากขั้นตอนที่ 475.2 — throttle เก็บ "ประวัติการเรียก" ไว้ใน **Django cache
framework** ถ้าโปรเจกต์ยังไม่ได้ตั้งค่า `CACHES` เอง Django จะใช้
`LocMemCache` (`django.core.cache.backends.locmem.LocMemCache`) เป็นค่า default ซึ่งเก็บ
ข้อมูลไว้ **ในหน่วยความจำของ process เดียวเท่านั้น**

ปัญหาเกิดขึ้นทันทีที่ deploy จริงด้วย **หลาย worker process** (เช่น Gunicorn ที่รันด้วย
`--workers 4`) หรือ **หลายเครื่อง server** พร้อมกัน (horizontal scaling ตามที่เรียนใน
Part 001 เรื่องข้อดีของ stateless architecture):

```
Worker Process 1 (LocMemCache ของตัวเอง) → เห็น user A เรียกไปแล้ว 3 ครั้ง
Worker Process 2 (LocMemCache ของตัวเอง) → เห็น user A เรียกไปแล้ว 0 ครั้ง (คนละ memory!)
Worker Process 3 (LocMemCache ของตัวเอง) → เห็น user A เรียกไปแล้ว 0 ครั้ง (คนละ memory!)

ถ้าตั้ง rate ไว้ที่ 5/min และ nginx/load balancer กระจาย request ของ user A
ไปยัง worker ต่างกันแบบสุ่ม → user A จะเรียกได้จริง ๆ ถึง 5 × 4 = 20 ครั้ง/นาที
(เพราะแต่ละ worker นับแยกกันเป็นอิสระ) — throttle ไม่แม่นยำอีกต่อไป!
```

### 478.2 ทางแก้: ใช้ Cache Backend แบบ Shared State เช่น Redis

การแก้ปัญหานี้คือให้ **ทุก worker/server ใช้ cache backend ตัวเดียวกันที่แยกตัวออกมา
ต่างหาก** (external cache service) ที่นิยมที่สุดคือ **Redis** เพราะเร็วมากและรองรับ
การหมดอายุอัตโนมัติ (TTL) ได้ในตัว ตรงกับความต้องการของ throttle พอดี:

```bash
pip install django-redis
pip freeze | grep -i redis >> requirements.txt
```

```python
# config/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
    },
}
```

เพียงเท่านี้ throttle ทุกตัวที่เรียนมาในขั้นตอนที่ 475-477 จะเริ่มใช้ Redis เป็นที่เก็บ
ประวัติการเรียกโดยอัตโนมัติทันที **โดยไม่ต้องแก้โค้ด throttle class เลยสักบรรทัด**
เพราะ `SimpleRateThrottle.cache` ชี้ไปที่ `caches['default']` อยู่แล้ว

### 478.3 แยก Cache Alias เฉพาะสำหรับ Throttling (แนวทางขั้นสูง)

ในระบบที่มีทั้ง **page caching** (แคชผลลัพธ์ query หนัก ๆ) และ **throttling** พร้อมกัน
การให้ทั้งสองใช้ cache alias เดียวกัน (`'default'`) อาจทำให้ throttle counter ปนกับ
cache key อื่น ๆ จำนวนมาก (แม้จะมี prefix `throttle_` กันชนกันอยู่แล้วก็ตาม) ระบบระดับ
production มักแยก Redis database index หรือแยก instance ไปเลยสำหรับ throttling
โดยเฉพาะ เพื่อให้ monitor และ debug ง่ายขึ้น:

```python
# config/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',   # สำหรับ page/query caching
    },
    'throttle': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/2',   # DB index แยกต่างหากสำหรับ throttle
    },
}
```

```python
# blog/throttles.py
from django.core.cache import caches
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle


class ThrottleCacheMixin:
    cache = caches['throttle']


class AppUserRateThrottle(ThrottleCacheMixin, UserRateThrottle):
    pass


class AppAnonRateThrottle(ThrottleCacheMixin, AnonRateThrottle):
    pass
```

จากนั้นใช้ `AppUserRateThrottle`/`AppAnonRateThrottle` แทนตัวต้นฉบับของ DRF ทุกที่ใน
โปรเจกต์ (รวมถึง `LoginRateThrottle` ในขั้นตอนที่ 477.1 ก็ควรเปลี่ยนให้สืบทอดจาก
`AppAnonRateThrottle` แทน `AnonRateThrottle` ตรง ๆ)

### 478.4 ทำไมเรื่องนี้ถึงแค่ "เกริ่น" ในขั้นตอนนี้

หัวข้อ **Caching Framework แบบเต็มรูปแบบของ Django** — การตั้งค่า Redis อย่างละเอียด,
`cache_page` decorator, low-level cache API, cache invalidation strategy, per-object
caching, และการผสาน caching เข้ากับ DRF ViewSet — เป็นหัวข้อที่ใหญ่พอที่จะมี **Part
068** เป็นของตัวเองในหลักสูตรนี้ สิ่งที่ต้องจำจากขั้นตอนนี้มีแค่ 3 ข้อ:

1. Throttle counter เก็บอยู่ใน Django cache framework เสมอ
2. `LocMemCache` (ค่า default) **ใช้ไม่ได้จริง** กับ production ที่มีหลาย
   process/server เพราะแต่ละ process มี memory แยกกัน
3. ต้องเปลี่ยนไปใช้ cache backend แบบ shared state (Redis เป็นตัวเลือกมาตรฐาน) ก่อน
   deploy จริงเสมอ — รายละเอียดวิธีตั้งค่า Redis แบบเต็มรูปแบบ (persistence, cluster,
   monitoring) รอเรียนใน Part 068

### 478.5 ตารางสรุปขั้นตอนที่ 478

| ประเด็น | สรุป |
|---|---|
| Throttle เก็บข้อมูลไว้ที่ไหน | Django cache framework (attribute `cache` ของ throttle class) |
| ปัญหาของ `LocMemCache` | แยก memory ต่อ process → throttle นับไม่แม่นยำเมื่อมีหลาย worker/server |
| ทางแก้มาตรฐาน | เปลี่ยน `CACHES['default']` เป็น Redis (ผ่าน `django-redis`) |
| แนวทางขั้นสูง | แยก cache alias เฉพาะสำหรับ throttle ออกจาก cache ทั่วไป |
| เจาะลึกเต็มรูปแบบที่ไหน | Part 068 (Caching Framework) |

---

## ขั้นตอนที่ 479: การเขียน Test สำหรับพฤติกรรม Throttling

### 479.1 กับดักสำคัญที่สุดของการเทส Throttling: Cache ค้างข้าม Test

เพราะ throttle เก็บ state ไว้ใน cache (ทวนจากขั้นตอนที่ 475.2/478) ถ้าไม่ล้าง cache
ระหว่าง test แต่ละตัว **ผลของ test ก่อนหน้าจะรั่วไหลมาปนกับ test ถัดไป** ทำให้ test
ล้มเหลวแบบสุ่ม (flaky test) ที่ debug ยากมาก — นี่คือกับดักคลาสสิกที่นักพัฒนา DRF
มือใหม่เกือบทุกคนเจอ

```python
# blog/tests/test_throttling.py
from django.core.cache import cache
from django.test import override_settings
from django.contrib.auth import get_user_model
from rest_framework.test import APITestCase
from rest_framework import status

User = get_user_model()


class LoginThrottleTests(APITestCase):
    def setUp(self):
        # ล้าง cache ก่อนทุก test เสมอ — ป้องกัน throttle history รั่วไหลข้าม test
        cache.clear()
        self.user = User.objects.create_user(
            username='somchai', password='SecurePass123!'
        )

    def tearDown(self):
        # ล้างซ้ำอีกครั้งหลัง test จบ กันกรณี test ถัดไปรันเร็วมากจน cache ยังไม่ expire
        cache.clear()
```

### 479.2 Test ว่า Login Throttle บล็อกจริงหลังเกินเพดาน

ใช้ `override_settings` เพื่อกำหนด rate ที่**คาดเดาได้และเทสเร็ว** แทนค่า production
จริง (ถ้าเทสด้วย rate จริงอย่าง `100/day` จะต้องยิง 101 request ถึงจะเห็นผล ซึ่งช้า
เกินไปสำหรับ test suite):

```python
# blog/tests/test_throttling.py (ต่อจากด้านบน)
@override_settings(
    REST_FRAMEWORK={
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'rest_framework_simplejwt.authentication.JWTAuthentication',
        ],
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.IsAuthenticatedOrReadOnly',
        ],
        'DEFAULT_THROTTLE_RATES': {
            'login': '3/min',   # ตั้งให้ต่ำมากเพื่อเทสได้เร็วโดยไม่ต้องยิงหลายร้อยครั้ง
        },
    }
)
class LoginThrottleRateTests(APITestCase):
    def setUp(self):
        cache.clear()
        self.user = User.objects.create_user(
            username='somchai', password='SecurePass123!'
        )
        self.url = '/api/token/'

    def tearDown(self):
        cache.clear()

    def test_allows_requests_within_limit(self):
        for _ in range(3):
            response = self.client.post(
                self.url,
                {'username': 'somchai', 'password': 'wrong-password'},
                format='json',
            )
            # รหัสผ่านผิดตั้งใจ แต่ยังไม่ควรโดน throttle เพราะยังไม่เกิน 3 ครั้ง
            self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_blocks_after_exceeding_limit(self):
        for _ in range(3):
            self.client.post(
                self.url,
                {'username': 'somchai', 'password': 'wrong-password'},
                format='json',
            )

        # ครั้งที่ 4 ต้องถูก throttle แล้ว
        response = self.client.post(
            self.url,
            {'username': 'somchai', 'password': 'wrong-password'},
            format='json',
        )
        self.assertEqual(response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
        self.assertIn('Retry-After', response.headers)

    def test_successful_login_still_counts_toward_limit(self):
        """แม้ login สำเร็จ ก็ยังนับรวมในโควตาเดียวกัน (ป้องกัน account enumeration
        ผ่านการเดาว่า throttle ทำงานเฉพาะตอน login ผิดเท่านั้น)"""
        for _ in range(3):
            self.client.post(
                self.url,
                {'username': 'somchai', 'password': 'SecurePass123!'},
                format='json',
            )

        response = self.client.post(
            self.url,
            {'username': 'somchai', 'password': 'SecurePass123!'},
            format='json',
        )
        self.assertEqual(response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
```

### 479.3 Test ว่า Throttle แยกกันจริงตาม IP/Identity (ไม่ปนกันข้าม client)

```python
# blog/tests/test_throttling.py (ต่อจากด้านบน)
@override_settings(
    REST_FRAMEWORK={
        'DEFAULT_THROTTLE_RATES': {'login': '2/min'},
    }
)
class LoginThrottlePerIPTests(APITestCase):
    def setUp(self):
        cache.clear()
        self.user = User.objects.create_user(
            username='somchai', password='SecurePass123!'
        )

    def tearDown(self):
        cache.clear()

    def test_different_ips_have_independent_quota(self):
        url = '/api/token/'
        payload = {'username': 'somchai', 'password': 'wrong-password'}

        # ยิงจาก IP แรกจนเกินโควตา
        for _ in range(2):
            self.client.post(url, payload, format='json', REMOTE_ADDR='10.0.0.1')
        blocked_response = self.client.post(
            url, payload, format='json', REMOTE_ADDR='10.0.0.1'
        )
        self.assertEqual(blocked_response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)

        # IP อื่นต้องยังเรียกได้ตามปกติ เพราะโควตาแยกกันตาม IP
        other_ip_response = self.client.post(
            url, payload, format='json', REMOTE_ADDR='10.0.0.2'
        )
        self.assertEqual(other_ip_response.status_code, status.HTTP_401_UNAUTHORIZED)
```

### 479.4 Test ว่า Versioned Endpoint คืนโครงสร้างถูกต้องตามเวอร์ชัน

แม้หัวข้อหลักของขั้นตอนนี้คือ throttling แต่ควรเทส versioning ควบคู่กันเสมอ เพราะทั้งคู่
เป็น "สัญญา" ที่ผิดพลาดแล้วกระทบ client ภายนอกโดยตรงเหมือนกัน:

```python
# blog/tests/test_versioning.py
from django.contrib.auth import get_user_model
from rest_framework.test import APITestCase
from rest_framework import status
from blog.models import Post

User = get_user_model()


class PostVersioningTests(APITestCase):
    def setUp(self):
        self.author = User.objects.create_user(username='somchai', password='x')
        self.post = Post.objects.create(
            author=self.author,
            title='ทดสอบ Versioning',
            slug='test-versioning',
            content=' '.join(['คำ'] * 250),  # ~250 คำ ให้คำนวณ reading_time ได้ >1
            is_published=True,
        )

    def test_v1_returns_flat_author_string(self):
        response = self.client.get('/api/v1/posts/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data[0]['author'], 'somchai')
        self.assertIn('is_published', response.data[0])
        self.assertNotIn('status', response.data[0])

    def test_v2_returns_nested_author_object(self):
        response = self.client.get('/api/v2/posts/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data[0]['author']['username'], 'somchai')
        self.assertEqual(response.data[0]['status'], 'published')
        self.assertNotIn('is_published', response.data[0])

    def test_v2_reading_time_calculated_correctly(self):
        response = self.client.get('/api/v2/posts/')
        # 250 คำ / 200 คำต่อนาที = 1.25 ปัดเป็น 1
        self.assertEqual(response.data[0]['reading_time_minutes'], 1)

    def test_unknown_version_returns_404(self):
        response = self.client.get('/api/v99/posts/')
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)

    def test_deprecated_version_has_sunset_headers(self):
        response = self.client.get('/api/v1/posts/')
        self.assertEqual(response.headers.get('Deprecation'), 'true')
        self.assertIn('Sunset', response.headers)
        self.assertIn('successor-version', response.headers.get('Link', ''))
```

### 479.5 การเทส Throttle ที่ต้องพึ่ง "เวลาผ่านไป" (Time-based Reset)

การเทสว่า throttle **รีเซ็ตเมื่อเวลาผ่านไปครบ `duration`** เป็นเรื่องยากถ้ารอเวลาจริง
(เทส 1 นาทีก็ทำให้ test suite ช้าโดยไม่จำเป็น) วิธีที่มืออาชีพใช้คือ mock ฟังก์ชัน
`timer` ของ throttle class ให้คืนค่าที่ควบคุมได้เอง:

```python
# blog/tests/test_throttling.py (ต่อจากด้านบน)
from unittest.mock import patch


@override_settings(REST_FRAMEWORK={'DEFAULT_THROTTLE_RATES': {'login': '2/min'}})
class LoginThrottleResetTests(APITestCase):
    def setUp(self):
        cache.clear()
        self.user = User.objects.create_user(username='somchai', password='x')

    def tearDown(self):
        cache.clear()

    def test_quota_resets_after_duration_passes(self):
        url = '/api/token/'
        payload = {'username': 'somchai', 'password': 'wrong'}

        fake_time = [1_700_000_000.0]   # ใช้ list เพื่อแก้ไขค่าจากใน closure ได้

        def mock_timer():
            return fake_time[0]

        with patch(
            'rest_framework.throttling.SimpleRateThrottle.timer',
            side_effect=mock_timer,
        ):
            # ใช้โควตาจนหมด (2 ครั้ง)
            self.client.post(url, payload, format='json')
            self.client.post(url, payload, format='json')
            blocked = self.client.post(url, payload, format='json')
            self.assertEqual(blocked.status_code, status.HTTP_429_TOO_MANY_REQUESTS)

            # เลื่อนเวลาไปข้างหน้า 61 วินาที (เกิน duration ของ '2/min' คือ 60 วินาที)
            fake_time[0] += 61

            recovered = self.client.post(url, payload, format='json')
            self.assertEqual(recovered.status_code, status.HTTP_401_UNAUTHORIZED)
```

**ทางเลือกอื่นในโปรเจกต์จริง**: ถ้าใช้ library `freezegun` (`pip install freezegun`)
จะเขียนโค้ดลักษณะนี้ได้สะดวกกว่ามาก ด้วย `freeze_time(...)` context manager ที่ควบคุม
เวลาของทั้งระบบพร้อมกันในทีเดียว แทนการ patch เฉพาะ `timer` ของ throttle class ตรง ๆ
แบบด้านบน — เหมาะกับโปรเจกต์ที่มีการเทสเรื่องเวลา (expiry, scheduling) หลายจุดพร้อมกัน

### 479.6 ตารางสรุปขั้นตอนที่ 479

| หลักการ | รายละเอียด |
|---|---|
| ล้าง cache ทุก test | `cache.clear()` ใน `setUp()`/`tearDown()` เสมอ ป้องกัน throttle history รั่วไหลข้าม test |
| ใช้ `override_settings` กำหนด rate เฉพาะ test | หลีกเลี่ยงรอ 100+ request เพื่อเทส rate จริงที่สูงมาก |
| เทสทั้งกรณีผ่านและกรณีถูกบล็อก | ต้องยืนยันทั้ง "ไม่ throttle ก่อนถึงเพดาน" และ "throttle เมื่อเกินเพดาน" |
| เทสว่าโควตาแยกกันตาม identity | ส่ง `REMOTE_ADDR` ต่างกันเพื่อพิสูจน์ว่า IP หนึ่งไม่กระทบโควตาของอีก IP |
| เทส time-based reset | `patch()` เมธอด `timer` ของ throttle class หรือใช้ library `freezegun` |
| เทส versioning คู่กับ throttling | ทั้งคู่คือ "สัญญา" ของ API ที่ผิดแล้วกระทบ client โดยตรงเหมือนกัน |

---

## ขั้นตอนที่ 480: สรุปและแบบฝึกหัด — เพิ่ม versioning และ throttling เต็มรูปแบบให้ Blog API

### 480.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่าทำไม API Versioning ถึงจำเป็น โดยเฉพาะกับ mobile app ที่อัปเดตช้ากว่าเว็บ
  และรู้จักประเภทของ breaking change ที่พบบ่อยที่สุด
- ✅ เปรียบเทียบ 4 versioning scheme ของ DRF (`URLPathVersioning`,
  `NamespaceVersioning`, `AcceptHeaderVersioning`, `QueryParameterVersioning`) และรู้ว่า
  เมื่อไหร่ควรเลือกแบบไหน
- ✅ Implement `/api/v1/posts/` และ `/api/v2/posts/` ที่มีโครงสร้าง response ต่างกันจริง
  โดยไม่ต้องแก้ Model เลย (Model คือความจริงหนึ่งเดียว, Serializer คือมุมมองที่ต่างกัน)
- ✅ วางกลยุทธ์ deprecation แบบมืออาชีพด้วย `Deprecation`/`Sunset`/`Link` header
  (RFC 8594) พร้อม timeline 6 ระยะตั้งแต่ประกาศจนถึงลบโค้ดทิ้ง
- ✅ ใช้ `AnonRateThrottle`/`UserRateThrottle` และเข้าใจกลไกภายในที่พึ่งพา Django cache
  framework
- ✅ เขียน Custom Throttle Class เองได้หลายรูปแบบ: `ScopedRateThrottle`, burst+sustained,
  role-based dynamic rate, และ throttle ตาม API Key
- ✅ ตั้งค่า throttle เฉพาะ endpoint ที่อ่อนไหว (login) ให้เข้มกว่าปกติ และแยก throttle
  ตาม HTTP method ในหน้าเดียวกันได้
- ✅ เข้าใจว่าทำไม `LocMemCache` ใช้ไม่ได้กับ production ที่มีหลาย worker/server และรู้
  แนวทางแก้ด้วย Redis (รอเจาะลึกเต็มใน Part 068)
- ✅ เขียนเทสสำหรับพฤติกรรม throttling ได้ถูกต้อง รวมถึงหลีกเลี่ยงกับดัก cache รั่วไหล
  ข้าม test

### 480.2 ลงมือจริง: รวมทุกอย่างเข้าเป็น `blog` API เวอร์ชันสมบูรณ์

**ขั้นที่ 1 — `config/settings.py` ฉบับรวม**

```python
# config/settings.py
from datetime import timedelta
import os

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],

    # Versioning (ขั้นตอนที่ 472-473)
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',

    # Throttling (ขั้นตอนที่ 475-477)
    'DEFAULT_THROTTLE_CLASSES': [
        'blog.throttles.AppAnonRateThrottle',
        'blog.throttles.AppUserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'user_standard': '1000/day',
        'user_premium': '10000/day',
        'login': '5/min',
        'posts_read': '500/hour',
        'posts_write': '20/hour',
        'search': '30/min',
        'partner_api_key': '10000/day',
    },
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'SIGNING_KEY': os.environ.get('JWT_SIGNING_KEY', None),
}

# Cache backend สำหรับ throttling (ขั้นตอนที่ 478) — Redis ใน production
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/1'),
    },
    'throttle': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_THROTTLE_URL', 'redis://127.0.0.1:6379/2'),
    },
}
```

**ขั้นที่ 2 — `blog/throttles.py` ฉบับรวม**

```python
# blog/throttles.py
from django.core.cache import caches
from rest_framework.throttling import (
    AnonRateThrottle,
    UserRateThrottle,
    SimpleRateThrottle,
)


class ThrottleCacheMixin:
    """แยก cache alias ของ throttle ออกจาก cache ทั่วไปของระบบ — ขั้นตอนที่ 478.3"""
    cache = caches['throttle']


class AppAnonRateThrottle(ThrottleCacheMixin, AnonRateThrottle):
    pass


class AppUserRateThrottle(ThrottleCacheMixin, UserRateThrottle):
    pass


class LoginRateThrottle(ThrottleCacheMixin, AnonRateThrottle):
    """ขั้นตอนที่ 477.1 — ป้องกัน brute-force บน endpoint login"""
    scope = 'login'


class RoleAwareUserRateThrottle(ThrottleCacheMixin, UserRateThrottle):
    """ขั้นตอนที่ 476.4 — rate สูงกว่าสำหรับสมาชิกพรีเมียม ตรวจจาก DB จริงเสมอ"""

    def allow_request(self, request, view):
        if request.user and request.user.is_authenticated:
            is_premium = getattr(
                getattr(request.user, 'profile', None), 'is_premium', False
            )
            self.scope = 'user_premium' if is_premium else 'user_standard'
            self.rate = self.get_rate()
            self.num_requests, self.duration = self.parse_rate(self.rate)
        return super().allow_request(request, view)


class PartnerAPIKeyThrottle(ThrottleCacheMixin, SimpleRateThrottle):
    """ขั้นตอนที่ 476.5 — throttle แยกโควตาตาม API Key ของ partner แต่ละราย"""

    scope = 'partner_api_key'

    def get_cache_key(self, request, view):
        api_key = request.META.get('HTTP_X_API_KEY')
        if not api_key:
            return None
        return self.cache_format % {'scope': self.scope, 'ident': api_key}
```

**ขั้นที่ 3 — `blog/serializers.py`**

`PostSerializerV1` และ `PostSerializerV2` ใช้โค้ดตัวเต็มจากขั้นตอนที่ 473.2 และ 473.3
โดยตรง ไม่มีอะไรเปลี่ยนเพิ่มในขั้นตอนสรุปนี้ (ทวนหลักการสำคัญ: Model ไม่เคยถูกแก้เลย
ตลอดทั้ง Part นี้ — เปลี่ยนแค่ "มุมมอง" ที่ Serializer แต่ละเวอร์ชันนำเสนอเท่านั้น)

**ขั้นที่ 4 — `blog/api_views.py` ฉบับรวม**

```python
# blog/api_views.py
import datetime
from django.shortcuts import get_object_or_404
from django.utils.http import http_date
from rest_framework import status
from rest_framework.exceptions import APIException
from rest_framework.response import Response
from rest_framework.throttling import ScopedRateThrottle
from rest_framework.views import APIView

from .models import Post
from .serializers import PostSerializerV1, PostSerializerV2
from .throttles import AppAnonRateThrottle, AppUserRateThrottle


class VersionedSerializerMixin:
    serializer_class_by_version = {}
    default_version = 'v1'

    def get_serializer_class(self):
        version = getattr(self.request, 'version', None)
        return self.serializer_class_by_version.get(
            version, self.serializer_class_by_version[self.default_version]
        )


class VersionSunsetted(APIException):
    status_code = status.HTTP_410_GONE
    default_detail = 'API เวอร์ชันนี้ถูกปลดระวางแล้ว กรุณาย้ายไปใช้ /api/v2/'
    default_code = 'version_sunsetted'


class SunsetCheckMixin:
    sunsetted_versions = set()

    def initial(self, request, *args, **kwargs):
        super().initial(request, *args, **kwargs)
        if getattr(request, 'version', None) in self.sunsetted_versions:
            raise VersionSunsetted()


class DeprecatedVersionMixin:
    deprecated_versions = {}
    successor_version_url = None

    def finalize_response(self, request, response, *args, **kwargs):
        response = super().finalize_response(request, response, *args, **kwargs)
        sunset_date = self.deprecated_versions.get(getattr(request, 'version', None))
        if sunset_date is not None:
            sunset_datetime = datetime.datetime.combine(
                sunset_date, datetime.time.min, tzinfo=datetime.timezone.utc
            )
            response['Deprecation'] = 'true'
            response['Sunset'] = http_date(sunset_datetime.timestamp())
            if self.successor_version_url:
                response['Link'] = f'<{self.successor_version_url}>; rel="successor-version"'
        return response


class PostListAPIView(
    SunsetCheckMixin, DeprecatedVersionMixin, VersionedSerializerMixin, APIView
):
    serializer_class_by_version = {
        'v1': PostSerializerV1,
        'v2': PostSerializerV2,
    }
    deprecated_versions = {
        'v1': datetime.date(2027, 6, 30),
    }
    successor_version_url = '/api/v2/posts/'
    sunsetted_versions = set()   # เปลี่ยนเป็น {'v1'} เมื่อถึงวัน hard sunset จริง

    def get_throttles(self):
        if self.request.method == 'POST':
            self.throttle_scope = 'posts_write'
        else:
            self.throttle_scope = 'posts_read'
        return [ScopedRateThrottle(), AppAnonRateThrottle(), AppUserRateThrottle()]

    def get(self, request):
        posts = Post.objects.filter(is_published=True).select_related('author')
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(posts, many=True, context={'request': request})
        return Response(serializer.data)

    def post(self, request):
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(data=request.data, context={'request': request})
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetailAPIView(
    SunsetCheckMixin, DeprecatedVersionMixin, VersionedSerializerMixin, APIView
):
    serializer_class_by_version = {
        'v1': PostSerializerV1,
        'v2': PostSerializerV2,
    }
    deprecated_versions = {'v1': datetime.date(2027, 6, 30)}
    successor_version_url = '/api/v2/posts/'
    throttle_classes = [AppAnonRateThrottle, AppUserRateThrottle]

    def get_object(self, slug):
        return get_object_or_404(Post.objects.select_related('author'), slug=slug)

    def get(self, request, slug):
        serializer_class = self.get_serializer_class()
        serializer = serializer_class(self.get_object(slug), context={'request': request})
        return Response(serializer.data)
```

**ขั้นที่ 5 — `config/urls.py` และ `blog/api_urls.py` ฉบับรวม**

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView
from blog.api_views_auth import ThrottledTokenObtainPairView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/<str:version>/', include('blog.api_urls')),
    path('api/token/', ThrottledTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

```python
# blog/api_urls.py
from django.urls import path
from . import api_views

app_name = 'blog_api'

urlpatterns = [
    path('posts/', api_views.PostListAPIView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', api_views.PostDetailAPIView.as_view(), name='post-detail'),
]
```

```python
# blog/api_views_auth.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers_auth import CustomTokenObtainPairSerializer   # จาก Part 046
from .throttles import LoginRateThrottle


class ThrottledTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
    throttle_classes = [LoginRateThrottle]
```

**ขั้นที่ 6 — ทดสอบ end-to-end ด้วย `curl`**

```bash
# 1. v1 ยังทำงานเหมือนเดิม พร้อม deprecation header
curl -si http://127.0.0.1:8000/api/v1/posts/ | grep -E "Deprecation|Sunset|Link|HTTP"

# 2. v2 มีโครงสร้างใหม่ ไม่มี deprecation header
curl -s http://127.0.0.1:8000/api/v2/posts/ | python -m json.tool

# 3. ยิง login เกินเพดานจนโดน throttle
for i in $(seq 1 6); do
  curl -s -o /dev/null -w "ครั้งที่ $i: %{http_code}\n" \
    -X POST http://127.0.0.1:8000/api/token/ \
    -H "Content-Type: application/json" \
    -d '{"username": "somchai", "password": "wrong"}'
done

# 4. เวอร์ชันที่ไม่รู้จักคืน 404
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/api/v9/posts/
```

### 480.3 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่าทำไม API Versioning ถึงจำเป็น พร้อมยกตัวอย่าง breaking change อย่างน้อย
      3 ประเภท
- [ ] เปรียบเทียบข้อดี-ข้อเสียของ 4 versioning scheme ได้ และบอกได้ว่าโปรเจกต์แบบไหน
      ควรเลือกแบบไหน
- [ ] Implement `/api/v1/posts/` และ `/api/v2/posts/` ที่มีโครงสร้าง response ต่างกันจริง
      สำเร็จ โดยไม่แก้ Model
- [ ] เขียน `DeprecatedVersionMixin` ที่แปะ `Deprecation`/`Sunset`/`Link` header ได้ถูกต้อง
- [ ] เข้าใจกลไกภายในของ `AnonRateThrottle`/`UserRateThrottle` ว่าพึ่งพา cache framework
      อย่างไร
- [ ] เขียน Custom Throttle Class ได้อย่างน้อย 2 แบบ (เช่น scoped + role-based)
- [ ] ตั้งค่า throttle เฉพาะ endpoint login ให้เข้มกว่าปกติสำเร็จ และทดสอบด้วย `curl`
      เห็น `429` จริง
- [ ] อธิบายได้ว่าทำไม `LocMemCache` ใช้ไม่ได้กับ production หลาย worker/server
- [ ] เขียนเทส throttling ที่ล้าง cache ถูกต้องระหว่าง test แต่ละตัว ไม่มี flaky test
- [ ] รวม versioning + throttling เข้ากับ `blog` API ทั้งระบบสำเร็จ

### 480.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่มเวอร์ชัน `v3` ให้ `blog` API ที่เปลี่ยนโครงสร้าง response จาก
array ตรง ๆ (`[...]`) ให้เป็น envelope แบบ `{"data": [...], "meta": {"version": "v3",
"count": N}}` โดยต้องไม่กระทบ `v1` และ `v2` ที่มีอยู่เดิมเลย และต้องเพิ่ม `'v3'` เข้า
`ALLOWED_VERSIONS` ให้ถูกต้อง

**แบบฝึกหัดที่ 2**: เขียน management command ชื่อ `check_deprecated_usage` ที่ query
log การเรียก API (สมมติว่ามีการบันทึกไว้ในตาราง `APIRequestLog` ที่มีฟิลด์ `version`,
`api_key`, `timestamp`) แล้วพิมพ์รายชื่อ partner ที่ยังเรียก `v1` อยู่ในช่วง 7 วันล่าสุด
ออกมาเป็นตาราง เพื่อใช้ในระยะที่ 4 (Proactive Outreach) ของ deprecation timeline จาก
ขั้นตอนที่ 474.4

**แบบฝึกหัดที่ 3**: เขียน Custom Throttle Class ชื่อ `SlidingWindowRateThrottle` ที่ใช้
หลักการ **sliding window** อย่างเคร่งครัด (ไม่ใช่ fixed window) กล่าวคือแทนที่จะนับจาก
"เวลาที่ผ่านมา N หน่วยจากตอนนี้" ให้ปัดเศษเวลาปัจจุบันลงเป็นช่วง (bucket) คงที่ก่อนคำนวณ
แล้วเขียนเทสเปรียบเทียบพฤติกรรมกับ `SimpleRateThrottle` แบบปกติที่เรียนในขั้นตอนที่ 475.2
ว่าต่างกันอย่างไรตรงขอบเขตของช่วงเวลา (edge case ตอนรอยต่อของแต่ละ window)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ตั้งค่า `CACHES['throttle']` ให้ชี้ไปที่ Redis จริง
(ติดตั้ง Redis ผ่าน Docker: `docker run -d -p 6379:6379 redis:7-alpine`) แล้วเขียนสคริปต์
ทดสอบด้วย `concurrent.futures.ThreadPoolExecutor` ยิง request พร้อมกัน 20 thread ไปยัง
endpoint ที่ throttle ไว้ที่ `10/min` แล้ววัดผลว่าจำนวน request ที่ผ่านจริงตรงกับ `10`
พอดีหรือไม่ (ทดสอบเรื่อง race condition ของการนับ throttle แบบ concurrent เปรียบเทียบ
ระหว่างใช้ `LocMemCache` กับ Redis)

### 480.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องทำ versioning ตั้งแต่ API เวอร์ชันแรกเลยหรือไม่ หรือรอให้มี breaking change
ก่อนค่อยทำ?**
A: แนะนำให้ตั้ง URL/settings ให้รองรับ versioning ไว้ตั้งแต่ต้น (เช่น `/api/v1/...`
ตั้งแต่ endpoint แรกที่เขียน) แม้จะยังไม่มี `v2` ก็ตาม เพราะการเพิ่ม prefix `v1/` เข้าไป
ทีหลังหลังจาก client ภายนอกเริ่มใช้ `/api/posts/` ไปแล้ว **ก็คือ breaking change ในตัว
มันเอง** — การเตรียมโครงสร้างไว้ตั้งแต่แรกไม่มีต้นทุนเพิ่มเลย แต่ป้องกันปัญหาใหญ่ใน
อนาคตได้เสมอ

**Q: Throttling กับ Rate Limiting ที่ทำใน Nginx/API Gateway (เช่น Kong, AWS API
Gateway) ต่างกันอย่างไร ต้องทำทั้งสองระดับหรือไม่?**
A: ต่างกันที่ "ชั้น" (layer) — Rate limiting ระดับ infrastructure (Nginx, API Gateway,
CDN) ทำงาน**ก่อน**ที่ request จะเข้าถึงโค้ด Django เลยด้วยซ้ำ เหมาะสำหรับป้องกัน DDoS
ขนาดใหญ่ที่ยิงมาจำนวนมหาศาลจนไม่อยากให้ request ไปถึง application server เลย ส่วน DRF
Throttling ทำงานที่ระดับ application ทำให้เข้าถึงข้อมูล **business logic** ได้ลึกกว่า
(เช่น รู้ว่า user คนนี้เป็นสมาชิกพรีเมียมหรือไม่ ตามขั้นตอนที่ 476.4) ระบบระดับ production
จริงมักทำ**ทั้งสองชั้นพร้อมกัน**: ชั้นนอกกันการโจมตีหยาบ ๆ ปริมาณมาก ชั้นในควบคุมโควตา
แบบละเอียดตาม business rule

**Q: ทำไม Throttle ถึงไม่ใช้ Session/Database แทน Cache ไปเลย จะได้ไม่ต้องพึ่ง Redis?**
A: เพราะ throttle ต้องเช็ค**ทุก request**ที่เข้ามา (คล้ายกับปัญหา query DB ทุก request
ของ `TokenAuthentication` ที่เรียนใน Part 046 ขั้นตอนที่ 451.1) ถ้าใช้ฐานข้อมูลจะเพิ่ม
ภาระ query มหาศาลและช้ากว่ามาก Cache (โดยเฉพาะ Redis ที่เก็บใน memory ทั้งหมด) ตอบสนอง
เร็วกว่าฐานข้อมูลหลายเท่าตัว และมีกลไก TTL (auto-expire) ในตัวที่เหมาะกับข้อมูลแบบ
"ประวัติการเรียกในช่วงเวลาสั้น ๆ" พอดี

**Q: ถ้า client ส่ง header `Accept` ผิดรูปแบบตอนใช้ `AcceptHeaderVersioning` หรือใส่
version ที่ไม่มีใน `ALLOWED_VERSIONS` จะเกิดอะไรขึ้น?**
A: DRF จะ raise `rest_framework.exceptions.NotAcceptable` (คืน `406 Not Acceptable`)
สำหรับ `AcceptHeaderVersioning` หรือ `NotFound` (`404`) สำหรับ scheme อื่น ๆ ที่เวอร์ชัน
อยู่ใน URL (`URLPathVersioning`, `NamespaceVersioning`, `QueryParameterVersioning`)
โดยอัตโนมัติ ไม่ต้องเขียนโค้ดตรวจสอบเองเลย ตราบใดที่ตั้งค่า `ALLOWED_VERSIONS` ไว้ถูกต้อง
ตามขั้นตอนที่ 472.1

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 049: API Documentation ด้วย `drf-spectacular`, Swagger, OpenAPI** จะพาไปสร้าง
เอกสาร API แบบอัตโนมัติจากโค้ดจริงของ `blog` API ที่มีทั้ง versioning (v1/v2 จาก Part
นี้) และ throttling ครบถ้วน ให้ออกมาเป็นมาตรฐาน **OpenAPI Schema** พร้อม **Swagger UI**
และ **ReDoc** ที่ผู้ใช้ API ภายนอกเปิดดูและทดลองยิง request ได้เองผ่านเบราว์เซอร์ รวมถึง
วิธีเอกสารให้แสดงผลต่างกันตามแต่ละเวอร์ชัน และแสดง throttle rate ของแต่ละ endpoint ไว้
ในเอกสารให้ผู้ใช้ API เห็นชัดเจนตั้งแต่แรก ไม่ต้องมาเจอ `429` โดยไม่รู้ตัวอีกต่อไป

เตรียม `blog` API เวอร์ชัน v1/v2 พร้อมระบบ throttling เต็มรูปแบบของคุณไว้ให้พร้อม
แล้วไปต่อกันเลย!
