# Part 049: API Documentation: drf-spectacular, Swagger, OpenAPI

> **ขั้นตอนที่ 481-490 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: ทำให้ Blog API ที่คุณสร้างมาตั้งแต่ Part 039 (Post, Category,
> Comment พร้อม ViewSet, Router, Permission, JWT Authentication, Versioning) มี
> **เอกสารประกอบ (documentation) ที่สมบูรณ์แบบมืออาชีพ** โดยไม่ต้องเขียนเอกสารด้วยมือแม้แต่
> บรรทัดเดียว คุณจะติดตั้ง `drf-spectacular` เพื่อ generate **OpenAPI Specification**
> จากโค้ด Serializer/ViewSet ที่มีอยู่แล้วโดยอัตโนมัติ ปรับแต่งรายละเอียดด้วย
> `@extend_schema`, เปิด **Swagger UI** และ **Redoc** ให้ทีม frontend/mobile ทดลองยิง API
> ได้จริงจากเบราว์เซอร์, ทำให้เอกสาร "รู้จัก" JWT Authentication, จัดการเอกสารสำหรับ API
> หลายเวอร์ชันพร้อมกัน, เกริ่นการสร้าง client SDK จาก schema อัตโนมัติ และเข้าใจว่าทำไม
> อุตสาหกรรมย้ายจาก `drf-yasg` มาเป็น `drf-spectacular`

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 481: ทำไม API Documentation สำคัญ — ภาพรวม OpenAPI Specification (Swagger) มาตรฐาน
- ขั้นตอนที่ 482: ติดตั้งและตั้งค่า `drf-spectacular`
- ขั้นตอนที่ 483: การ generate schema อัตโนมัติจาก Serializer/View ที่มีอยู่แล้ว
- ขั้นตอนที่ 484: ปรับแต่ง schema ด้วย `@extend_schema` decorator (เพิ่ม description, example, custom response)
- ขั้นตอนที่ 485: ติดตั้ง Swagger UI และ Redoc เพื่อแสดงเอกสารแบบ interactive
- ขั้นตอนที่ 486: การ document Authentication scheme ใน schema (JWT, Token)
- ขั้นตอนที่ 487: จัดการเอกสารสำหรับ API หลาย version พร้อมกัน (เชื่อมกับ Part 048)
- ขั้นตอนที่ 488: เกริ่นการ generate client SDK จาก OpenAPI spec (openapi-generator)
- ขั้นตอนที่ 489: เปรียบเทียบ `drf-spectacular` กับ `drf-yasg` (ทางเลือกเก่ากว่า)
- ขั้นตอนที่ 490: สรุปและแบบฝึกหัด — สร้างเอกสาร API ที่สมบูรณ์สำหรับ Blog API พร้อม Swagger UI

---

## ขั้นตอนที่ 481: ทำไม API Documentation สำคัญ — ภาพรวม OpenAPI Specification (Swagger) มาตรฐาน

### 481.1 ปัญหาที่เกิดขึ้นจริงเมื่อ API ไม่มีเอกสาร

ทบทวนสิ่งที่คุณสร้างมาตั้งแต่ Part 039: Blog API ตอนนี้มี endpoint สำหรับ `Post`,
`Category`, `Comment` ครบ CRUD, มี custom action (`publish`, `unpublish`, `mine`), มี
permission ที่ซับซ้อน (Part 045), มี JWT authentication (Part 046), มี filtering/
pagination (Part 047) และ versioning/throttling (Part 048) ลองจินตนาการว่าคุณเป็น
นักพัฒนา frontend หรือ mobile ที่เพิ่งได้รับมอบหมายให้เชื่อมต่อกับ API นี้เป็นครั้งแรก
คำถามที่คุณจะต้องเจอทันที:

- `POST /api/posts/` ต้องส่ง field อะไรบ้าง? field ไหน required, field ไหน optional?
- `category` ต้องส่งเป็น ID ตัวเลข หรือ slug?
- ถ้า login ไม่ผ่านจะได้ error response หน้าตาเป็นอย่างไร (401 หรือ 403? มี field
  `detail` หรือ `errors`)?
- endpoint `/api/posts/{slug}/publish/` รับ HTTP method อะไร ต้องใส่ body หรือไม่?
- ต้องใส่ JWT token ตรงไหนของ request (header ชื่ออะไร format แบบไหน)?

ถ้าไม่มีเอกสาร คำตอบของคำถามเหล่านี้จะอยู่ใน **หัวของนักพัฒนา backend คนเดียว** ทีมอื่น
ต้องมาถามซ้ำ ๆ ทาง Slack/Line ทุกวัน หรือแย่กว่านั้นคือต้องไปเปิดอ่านซอร์สโค้ด Django
เอง ซึ่งเป็นสิ่งที่ทีม frontend/mobile ไม่ควรต้องทำเลย — นี่คือต้นทุนแฝง (hidden cost)
ที่ทีมจำนวนมากประเมินต่ำเกินไปตอนเริ่มโปรเจกต์

### 481.2 OpenAPI Specification คืออะไร

**OpenAPI Specification (OAS)** คือมาตรฐานเปิด (open standard) สำหรับอธิบาย RESTful API
ในรูปแบบที่ **ทั้งมนุษย์และเครื่องอ่านได้** โดยเขียนเป็นไฟล์ JSON หรือ YAML เดียวที่ระบุ
ครบทุกอย่างเกี่ยวกับ API:

- มี endpoint (path) อะไรบ้าง แต่ละ endpoint รองรับ HTTP method ไหน
- แต่ละ endpoint รับ parameter/request body แบบไหน (field, type, required/optional)
- แต่ละ endpoint คืน response แบบไหนในแต่ละ status code (200, 400, 404, ...)
- API ใช้ authentication scheme แบบไหน (Bearer token, API key, Basic auth, ...)
- โครงสร้างข้อมูล (schema) ของแต่ละ object ที่ใช้ร่วมกันหลาย endpoint

เพราะไฟล์นี้เป็น **มาตรฐาน** (ไม่ใช่ format เฉพาะของเครื่องมือใดเครื่องมือหนึ่ง) เครื่องมือ
นับพันตัวในระบบนิเวศจึงอ่านไฟล์เดียวกันนี้แล้วทำสิ่งต่าง ๆ ได้อัตโนมัติ: แสดงผลเป็นหน้าเว็บ
เอกสารสวยงาม (Swagger UI, Redoc), generate client SDK หลายภาษา (ขั้นตอนที่ 488),
สร้าง mock server สำหรับทดสอบ, ตรวจสอบ contract testing อัตโนมัติ

### 481.3 ประวัติโดยย่อ: จาก Swagger สู่ OpenAPI

| ปี | เหตุการณ์ |
|---|---|
| 2010-2011 | บริษัท Wordnik สร้าง **Swagger Specification** เพื่อจัดการเอกสาร API ภายในตัวเอง |
| 2015 | SmartBear (เจ้าของ Swagger) บริจาค spec ให้ **Linux Foundation** เปลี่ยนชื่อเป็น **OpenAPI Specification (OAS)** ภายใต้ **OpenAPI Initiative (OAI)** |
| 2017 | OpenAPI 3.0 เปิดตัว — ปรับโครงสร้างใหม่ทั้งหมด ยืดหยุ่นกว่า Swagger 2.0 เดิมมาก |
| 2021 | OpenAPI 3.1 เปิดตัว — รองรับ JSON Schema เต็มรูปแบบ (เข้ากันได้กับ JSON Schema 2020-12) |
| 2024-2026 | OpenAPI 3.1 กลายเป็นมาตรฐานหลักที่เครื่องมือใหม่ ๆ (รวมถึง `drf-spectacular`) ใช้เป็นค่าเริ่มต้น |

**คำว่า "Swagger" ในปัจจุบันมักหมายถึงชุดเครื่องมือ** (Swagger UI, Swagger Editor,
Swagger Codegen) ที่ SmartBear ยังคงดูแลต่อ ส่วน **"OpenAPI"** คือชื่อของตัว
**specification (มาตรฐานไฟล์)** เอง — คนทั่วไปมักใช้สองคำนี้แทนกันได้ในบทสนทนา แต่ในทาง
เทคนิคควรแยกให้ถูก: เขียน spec ตาม **OpenAPI**, แสดงผลด้วยเครื่องมือที่ชื่อ **Swagger UI**

### 481.4 ตัวอย่างหน้าตาไฟล์ OpenAPI (YAML) แบบย่อ

```yaml
openapi: 3.1.0
info:
  title: Blog API
  version: 1.0.0
paths:
  /api/posts/:
    get:
      summary: รายการบทความทั้งหมด
      responses:
        '200':
          description: รายการบทความ
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Post'
    post:
      summary: สร้างบทความใหม่
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PostRequest'
      responses:
        '201':
          description: สร้างสำเร็จ
components:
  schemas:
    Post:
      type: object
      properties:
        id:
          type: integer
        title:
          type: string
          maxLength: 200
        is_published:
          type: boolean
      required:
        - id
        - title
```

ไฟล์นี้เพียงไฟล์เดียวสามารถอธิบาย API ทั้งระบบได้ครบถ้วน — **เป้าหมายของ Part นี้คือ
ทำให้ Django สร้างไฟล์แบบนี้ให้เราโดยอัตโนมัติ** จากโค้ด `models.py`,
`serializers.py`, `viewsets.py` ที่มีอยู่แล้ว โดยไม่ต้องเขียน YAML เองแม้แต่บรรทัดเดียว

### 481.5 ระบบนิเวศเครื่องมือที่ทำงานกับ OpenAPI ได้

| เครื่องมือ | หน้าที่ |
|---|---|
| **Swagger UI** | แสดง schema เป็นหน้าเว็บ interactive กดทดลองยิง request ได้จริง (ขั้นตอนที่ 485) |
| **Redoc** | แสดง schema เป็นหน้าเว็บอ่านง่าย เน้นการอ่านเอกสารมากกว่าทดลองยิง (ขั้นตอนที่ 485) |
| **openapi-generator** | generate client SDK หลายภาษา (Python, TypeScript, Swift, Kotlin, ...) จาก schema (ขั้นตอนที่ 488) |
| **Postman** | import schema เข้ามาเป็น Collection สำเร็จรูปพร้อมทดสอบทันที |
| **Stoplight Studio** | เครื่องมือแก้ไข/ออกแบบ OpenAPI แบบ visual (design-first workflow) |
| **Spectral** | linter สำหรับตรวจสอบว่า schema เขียนถูกมาตรฐานและ convention ของทีมหรือไม่ |

### 481.6 ตารางเปรียบเทียบวิธี document API 3 แบบที่ใช้กับ Django ได้

| วิธี | วิธีทำงาน | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **เขียน Markdown มือ** | เขียนเอกสารแยกจากโค้ด (เช่น ไฟล์ `API.md`) | ควบคุมเนื้อหาได้เต็มที่ ไม่ต้องพึ่ง library | เอกสารกับโค้ดไม่ sync กัน (ลืมอัปเดตบ่อยมาก), ไม่มี "Try it out" |
| **`drf-yasg`** | อ่าน docstring + decorator `swagger_auto_schema` แล้ว generate Swagger 2.0 | เคยเป็นมาตรฐานยอดนิยม, มี Swagger UI ในตัว | ดูแลน้อยลงมาก, รองรับแค่ OpenAPI/Swagger **2.0** เท่านั้น (ขั้นตอนที่ 489) |
| **`drf-spectacular`** | อ่าน type ของ Serializer field, `queryset`, permission ฯลฯ โดยอัตโนมัติ (static analysis) แล้ว generate OpenAPI 3.0/3.1 | Sync กับโค้ดเสมอ, รองรับ OpenAPI รุ่นล่าสุด, ดูแลต่อเนื่องแอคทีฟ | ต้องเรียนรู้วิธี override ด้วย `@extend_schema` สำหรับกรณีที่เดาไม่ได้ |

### 481.7 ทำไมหลักสูตรนี้เลือก `drf-spectacular`

`drf-spectacular` เป็นเครื่องมือที่เอกสารทางการของ Django REST Framework แนะนำอย่าง
เป็นทางการในหน้า "Schema" (https://www.django-rest-framework.org/api-guide/schemas/)
มาตั้งแต่ปี 2021 ด้วยเหตุผลหลัก:

1. **Sync กับโค้ดเสมอโดยธรรมชาติ**: เพราะมันอ่านโครงสร้างจริงจาก `Serializer` และ
   `ViewSet` ที่คุณเขียนอยู่แล้ว ไม่ใช่เอกสารแยกที่ต้องจำอัปเดตเอง
2. **รองรับ OpenAPI 3.0/3.1** เต็มรูปแบบ ต่างจาก `drf-yasg` ที่ค้างอยู่ที่ Swagger 2.0
3. **มี contrib module** รองรับ package ยอดนิยมอย่าง `djangorestframework-simplejwt`
   (ขั้นตอนที่ 486), `django-filter` (Part 047), `djangorestframework-camel-case`
   ในตัวโดยไม่ต้องตั้งค่าเพิ่ม
4. ยังคงได้รับการดูแลและอัปเดตอย่างต่อเนื่องในปี 2026 ขณะที่ `drf-yasg` แทบไม่มี commit
   ใหม่มาหลายปีแล้ว (รายละเอียดเปรียบเทียบเต็มรูปแบบในขั้นตอนที่ 489)

---

## ขั้นตอนที่ 482: ติดตั้งและตั้งค่า `drf-spectacular`

### 482.1 ติดตั้งผ่าน pip

```bash
pip install drf-spectacular
pip freeze > requirements.txt
```

ตรวจสอบเวอร์ชัน:

```bash
python -c "import drf_spectacular; print(drf_spectacular.__version__)"
```

### 482.2 เพิ่มใน `INSTALLED_APPS`

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
    'rest_framework_simplejwt',
    'django_filters',          # จาก Part 047
    'drf_spectacular',         # ← เพิ่มบรรทัดนี้
    'blog',
]
```

### 482.3 ตั้ง `DEFAULT_SCHEMA_CLASS` ใน `REST_FRAMEWORK`

นี่คือจุดที่บอก DRF ว่า "เวลาสร้าง schema ให้ใช้กลไกของ `drf-spectacular` แทนกลไก
`AutoSchema` เริ่มต้นของ DRF เอง":

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    # ← บรรทัดสำคัญที่สุดของขั้นตอนนี้
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

**ถ้าลืมตั้งค่านี้**: `drf-spectacular` จะติดตั้งอยู่ในระบบ แต่ DRF จะยังคงใช้กลไก
`coreapi`/`AutoSchema` เดิมของตัวเองในการ generate schema (ซึ่งหยาบและไม่ค่อยแม่นยำ)
ดังนั้น **ทุกโปรเจกต์ที่ใช้ `drf-spectacular` ต้องตั้ง `DEFAULT_SCHEMA_CLASS` เสมอ**
ไม่เช่นนั้นการติดตั้งจะไม่มีผลอะไรเลย

### 482.4 ตั้งค่าข้อมูลทั่วไปของ API ผ่าน `SPECTACULAR_SETTINGS`

```python
# config/settings.py
SPECTACULAR_SETTINGS = {
    'TITLE': 'Blog API',
    'DESCRIPTION': (
        'REST API สำหรับระบบบล็อก รองรับการจัดการบทความ (Post), '
        'หมวดหมู่ (Category) และความคิดเห็น (Comment) '
        'พร้อมระบบ Authentication แบบ JWT'
    ),
    'VERSION': '1.0.0',
    'CONTACT': {
        'name': 'ทีม Backend',
        'email': 'backend-team@example.com',
    },
    'LICENSE': {'name': 'MIT License'},
    # ป้องกันไม่ให้ /api/schema/ ถูกรวมเข้าไปเป็น path หนึ่งใน schema ของตัวมันเอง
    'SERVE_INCLUDE_SCHEMA': False,
    # จัดกลุ่ม endpoint ตาม path แทนที่จะจัดตาม operationId
    'COMPONENT_SPLIT_REQUEST': True,
}
```

- `TITLE`, `DESCRIPTION`, `VERSION` จะไปปรากฏที่หัวหน้าเอกสารทั้ง Swagger UI และ Redoc
- `SERVE_INCLUDE_SCHEMA: False` ป้องกันปัญหา schema อ้างอิงตัวเองแบบไม่รู้จบ
- `COMPONENT_SPLIT_REQUEST: True` แยก schema สำหรับ request body ออกจาก response body
  (เช่น `PostRequest` แยกจาก `Post`) ซึ่งสำคัญมากเพราะ field อย่าง `id`, `created_at`
  เป็น `read_only` — ไม่ควรปรากฏใน request schema เลย

### 482.5 เพิ่ม endpoint สำหรับ schema ใน `urls.py`

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path
from drf_spectacular.views import SpectacularAPIView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
]
```

### 482.6 ทดสอบว่าติดตั้งสำเร็จ

```bash
python manage.py runserver
```

```bash
curl http://127.0.0.1:8000/api/schema/
```

ควรได้ผลลัพธ์เป็นไฟล์ YAML ยาว ๆ ที่ขึ้นต้นด้วย `openapi: 3.0.3` หรือ `3.1.0` ตามด้วย
`info:`, `paths:`, `components:` — ถ้าเจอ error `AssertionError:
"DEFAULT_SCHEMA_CLASS" ...` แปลว่าลืมทำขั้นตอนที่ 482.3

---

## ขั้นตอนที่ 483: การ generate schema อัตโนมัติจาก Serializer/View ที่มีอยู่แล้ว

### 483.1 `drf-spectacular` อ่านอะไรจากโค้ดของคุณบ้าง

จุดแข็งที่สุดของ `drf-spectacular` คือมันไม่ต้องการให้คุณเขียนอะไรเพิ่มเลยสำหรับกรณี
พื้นฐาน เพราะมัน **introspect (ตรวจสอบ)** โครงสร้างโค้ดที่มีอยู่แล้วโดยตรง:

| อ่านจาก | ได้ข้อมูลอะไร |
|---|---|
| `serializer.Meta.model` + field types | ชนิดข้อมูล (`string`, `integer`, `boolean`, ...) ของแต่ละ field |
| `CharField(max_length=200)` | `maxLength: 200` ใน schema |
| `read_only_fields` / `ReadOnlyField` | field นั้นไม่ปรากฏใน request schema (ถ้าตั้ง `COMPONENT_SPLIT_REQUEST`) |
| `required=False` / `blank=True` บน model field | field นั้นไม่อยู่ใน `required: [...]` |
| `viewset.queryset` / `get_queryset()` | ใช้เดาชนิดของ resource (สำหรับตั้ง tag/basename) |
| `permission_classes` | ใช้ประกอบ security requirement (ขั้นตอนที่ 486) |
| `filter_backends` + `filterset_fields` (Part 047) | สร้าง query parameter ใน schema อัตโนมัติ |
| `pagination_class` | สร้าง parameter `page`, `page_size` และ wrap response เป็น paginated schema |
| docstring ของ ViewSet/method | ใช้เป็น `description` เริ่มต้นถ้าไม่ได้ override ด้วย `@extend_schema` |

### 483.2 ตัวอย่าง: generate schema จาก `PostViewSet` ที่มีอยู่แล้วโดยไม่แก้อะไรเลย

นี่คือ `PostViewSet` ฉบับสมบูรณ์จาก Part 044-048 (ไม่มีการแก้ไขใด ๆ เพื่อ documentation
เลยในขั้นตอนนี้):

```python
# blog/viewsets.py (จาก Part 048 — ยังไม่แก้ไขอะไรเพื่อ documentation)
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.permissions import IsAdminUser, IsAuthenticated, IsAuthenticatedOrReadOnly
from rest_framework.response import Response
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.filters import OrderingFilter, SearchFilter
from .models import Post
from .serializers import PostSerializer
from .permissions import IsOwnerOrReadOnly


class PostViewSet(viewsets.ModelViewSet):
    """
    ViewSet สำหรับจัดการบทความในระบบบล็อก รองรับ CRUD เต็มรูปแบบ
    พร้อม custom action สำหรับเผยแพร่/ยกเลิกเผยแพร่บทความ
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    lookup_field = 'slug'
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['category__slug', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, slug=None):
        post = self.get_object()
        post.is_published = True
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)
```

รันคำสั่งต่อไปนี้เพื่อ generate schema เป็นไฟล์ (แทนที่จะดูผ่าน `curl`):

```bash
python manage.py spectacular --file schema.yaml
```

### 483.3 ผลลัพธ์ที่ `drf-spectacular` generate ให้อัตโนมัติ (ส่วนของ `Post`)

```yaml
paths:
  /api/posts/:
    get:
      operationId: posts_list
      description: >-
        ViewSet สำหรับจัดการบทความในระบบบล็อก รองรับ CRUD เต็มรูปแบบ
        พร้อม custom action สำหรับเผยแพร่/ยกเลิกเผยแพร่บทความ
      parameters:
        - name: category__slug
          in: query
          schema:
            type: string
        - name: is_published
          in: query
          schema:
            type: boolean
        - name: search
          in: query
          schema:
            type: string
          description: A search term.
        - name: ordering
          in: query
          schema:
            type: string
          description: Which field to use when ordering the results.
        - name: page
          in: query
          schema:
            type: integer
      tags: [posts]
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedPostList'
  /api/posts/{slug}/publish/:
    post:
      operationId: posts_publish_create
      parameters:
        - name: slug
          in: path
          required: true
          schema:
            type: string
      tags: [posts]
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Post'
components:
  schemas:
    Post:
      type: object
      properties:
        id:
          type: integer
          readOnly: true
        author:
          type: string
          readOnly: true
        title:
          type: string
          maxLength: 200
        slug:
          type: string
          readOnly: true
        content:
          type: string
        category:
          type: integer
          nullable: true
        is_published:
          type: boolean
      required: [id, author, title, content]
```

สังเกตสิ่งที่ `drf-spectacular` **เดาได้ถูกต้องเองทั้งหมด** โดยที่โค้ดใน 483.2 ไม่มี
การตกแต่งเพื่อ documentation เลยแม้แต่บรรทัดเดียว:

- Query parameter `category__slug`, `is_published`, `search`, `ordering`, `page`
  ถูกสร้างจาก `filterset_fields`, `search_fields`, `ordering_fields`, `pagination_class`
  โดยอัตโนมัติทั้งหมด
- `slug` ใน URL path ถูกตรวจจับจาก `lookup_field = 'slug'`
- Response ของ `list()` ถูก wrap เป็น `PaginatedPostList` (มี `count`, `next`,
  `previous`, `results`) เพราะเห็นว่ามี `pagination_class` ตั้งไว้
- `id`, `author`, `slug` มี `readOnly: true` เพราะเห็นจาก `ReadOnlyField` และ
  `read_only_fields` ใน Serializer
- docstring ของคลาส `PostViewSet` ถูกใช้เป็น `description` อัตโนมัติ

### 483.4 สิ่งที่ `drf-spectacular` **เดาไม่ได้** และต้องใช้ `@extend_schema` ช่วย

แม้จะเดาได้เก่งมาก แต่มีหลายกรณีที่การ introspect อย่างเดียวไม่พอ:

1. **custom action ที่ไม่คืนค่า serializer มาตรฐาน** — เช่น `publish()` ใน 483.2 ที่
   คืน `self.get_serializer(post).data` มัน**เดาถูก**เพราะเห็นเรียก `get_serializer`
   แต่ถ้า custom action คืน `Response({'status': 'ok'})` ตรง ๆ (ไม่ผ่าน serializer)
   `drf-spectacular` จะไม่รู้ shape ของ response เลย
2. **คำอธิบายภาษาไทยที่เจาะจงกว่า docstring ทั่วไป** — เช่น ต้องการอธิบายว่า field
   `category` รับเฉพาะ ID ของหมวดหมู่ที่ `is_active=True` เท่านั้น ซึ่งเป็น business
   rule ที่โค้ดไม่ได้สื่อออกมาตรง ๆ
3. **ตัวอย่างข้อมูล (example)** — schema เปล่า ๆ บอกแค่ type แต่ไม่มีตัวอย่างค่าจริง
   ที่ช่วยให้ทีม frontend เห็นภาพได้เร็วขึ้น
4. **error response ที่ไม่ได้มาจาก serializer validation ปกติ** — เช่น 404 แบบ
   custom message, 429 จาก throttling (Part 048)

ปัญหาทั้ง 4 ข้อนี้คือเหตุผลที่มีขั้นตอนที่ 484 ต่อไป

### 483.5 ตรวจสอบ warning จากคำสั่ง `spectacular`

```bash
python manage.py spectacular --file schema.yaml --validate
```

ถ้า `drf-spectacular` เจอจุดที่เดาไม่ได้ มันจะพิมพ์ warning ออกมาที่ terminal เช่น:

```
Warning: operation_id could not be derived automatically...
Warning: could not resolve serializer for "publish" action, defaulting to unknown type.
```

**คำแนะนำระดับมืออาชีพ**: รันคำสั่งนี้เป็นส่วนหนึ่งของ CI/CD pipeline (Part 088) เสมอ
เพื่อจับ warning เหล่านี้ตั้งแต่ก่อน merge — ทีมที่ไม่ตรวจสอบ warning มักจะมี schema
ที่ไม่สมบูรณ์สะสมไปเรื่อย ๆ โดยไม่รู้ตัว

---

## ขั้นตอนที่ 484: ปรับแต่ง schema ด้วย `@extend_schema` decorator

### 484.1 นำเข้าเครื่องมือที่จำเป็น

```python
from drf_spectacular.utils import (
    extend_schema,
    extend_schema_view,
    extend_schema_field,
    OpenApiParameter,
    OpenApiExample,
    OpenApiResponse,
    OpenApiTypes,
)
```

### 484.2 `@extend_schema_view`: ปรับแต่ง action มาตรฐาน (list/create/retrieve/...) ของ `ModelViewSet`

`@extend_schema` ใช้ตกแต่ง method เดี่ยว ๆ ได้ตรง ๆ แต่ action มาตรฐานของ
`ModelViewSet` (`list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`)
มาจาก Mixin ของ DRF เอง (Part 043-044) ไม่ใช่ method ที่เราเขียนเอง จึงต้องใช้
`@extend_schema_view` ครอบทั้งคลาสแทน:

```python
# blog/viewsets.py
@extend_schema_view(
    list=extend_schema(
        summary='รายการบทความทั้งหมด',
        description=(
            'คืนรายการบทความแบบแบ่งหน้า (pagination) '
            'กรองได้ด้วย query parameter `category__slug`, `is_published`, '
            '`search`, และเรียงลำดับด้วย `ordering`'
        ),
        tags=['posts'],
    ),
    create=extend_schema(
        summary='สร้างบทความใหม่',
        description='ผู้ใช้ที่ login แล้วเท่านั้นที่สร้างได้ ระบบจะตั้ง `author` เป็นผู้ใช้ปัจจุบันอัตโนมัติ',
    ),
    retrieve=extend_schema(summary='ดูรายละเอียดบทความ 1 ชิ้นตาม slug'),
    update=extend_schema(summary='แก้ไขบทความทั้งหมด (ต้องส่งครบทุก field)'),
    partial_update=extend_schema(summary='แก้ไขบทความบางส่วน'),
    destroy=extend_schema(summary='ลบบทความ (เจ้าของหรือ staff เท่านั้น)'),
)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    lookup_field = 'slug'
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['category__slug', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

### 484.3 ตกแต่ง custom action ด้วย `@extend_schema` ตรง ๆ

```python
    @extend_schema(
        summary='เผยแพร่บทความ',
        description='เปลี่ยนสถานะ `is_published` เป็น `true` เฉพาะเจ้าของบทความหรือ staff เท่านั้น',
        request=None,  # action นี้ไม่รับ request body
        responses={
            200: PostSerializer,
            403: OpenApiResponse(description='ไม่ใช่เจ้าของบทความ'),
            404: OpenApiResponse(description='ไม่พบบทความตาม slug ที่ระบุ'),
        },
        examples=[
            OpenApiExample(
                'ตัวอย่าง response สำเร็จ',
                value={
                    'id': 1,
                    'title': 'ทดสอบ ViewSet',
                    'slug': 'ทดสอบ-viewset',
                    'is_published': True,
                },
                response_only=True,
                status_codes=['200'],
            ),
        ],
    )
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, slug=None):
        post = self.get_object()
        post.is_published = True
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)
```

พารามิเตอร์สำคัญของ `@extend_schema`:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `summary` | หัวข้อสั้น ๆ (1 บรรทัด) แสดงในรายการ endpoint ของ Swagger UI |
| `description` | คำอธิบายยาว รองรับ Markdown เต็มรูปแบบ |
| `request` | ระบุ serializer/type ที่ใช้เป็น request body เอง (หรือ `None` ถ้าไม่มี body) |
| `responses` | dict ของ `{status_code: serializer/OpenApiResponse}` ระบุ response แต่ละสถานะ |
| `parameters` | list ของ `OpenApiParameter` สำหรับ query/path/header parameter เพิ่มเติม |
| `examples` | list ของ `OpenApiExample` ใส่ตัวอย่างค่าจริงให้ทีมอื่นเห็นภาพ |
| `tags` | จัดกลุ่ม endpoint ใน Swagger UI (ค่าเริ่มต้นมาจากชื่อ resource) |
| `deprecated` | `True` เพื่อบอกว่า endpoint นี้เลิกใช้แล้ว (ใช้ในขั้นตอนที่ 487) |

### 484.4 `OpenApiParameter`: เพิ่ม query parameter ที่ introspect ไม่เห็น

บางครั้งมี query parameter ที่ประมวลผลเองใน view โดยไม่ผ่าน `filter_backends`
มาตรฐาน (เช่น อ่านจาก `request.query_params` ตรง ๆ) ซึ่ง `drf-spectacular` ไม่มีทาง
รู้ได้เองว่ามี parameter นี้อยู่:

```python
    @extend_schema(
        summary='ค้นหาบทความตามหมวดหมู่',
        parameters=[
            OpenApiParameter(
                name='category',
                type=OpenApiTypes.STR,
                location=OpenApiParameter.QUERY,
                description='slug ของหมวดหมู่ที่ต้องการกรอง',
                required=True,
                examples=[
                    OpenApiExample('ตัวอย่างหมวดหมู่ Django', value='django'),
                ],
            ),
        ],
        responses={
            200: PostSerializer(many=True),
            400: OpenApiResponse(description='ไม่ได้ส่ง query parameter `category` มา'),
        },
    )
    @action(detail=False)
    def by_category(self, request):
        category_slug = request.query_params.get('category')
        if not category_slug:
            return Response({'detail': 'ต้องระบุ query parameter category'}, status=400)
        queryset = self.get_queryset().filter(category__slug=category_slug)
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

### 484.5 `extend_schema_field`: กำหนด type ของ `SerializerMethodField` เอง

`SerializerMethodField` เป็นจุดที่ `drf-spectacular` เดา type ไม่ได้เลย เพราะมันคือ
ฟังก์ชัน Python ธรรมดาที่คืนอะไรก็ได้:

```python
# blog/serializers.py
from drf_spectacular.utils import extend_schema_field
from rest_framework import serializers
from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')
    comment_count = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = [
            'id', 'author', 'title', 'slug', 'content', 'category',
            'comment_count', 'is_published', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']

    @extend_schema_field(OpenApiTypes.INT)
    def get_comment_count(self, obj) -> int:
        return obj.comments.count()
```

ถ้าไม่ใส่ `@extend_schema_field(OpenApiTypes.INT)` ตรงนี้ `drf-spectacular` จะใส่
`type: string` ให้เป็นค่า default (fallback ที่ปลอดภัยที่สุดเมื่อเดาไม่ได้) ซึ่งผิด
จากความเป็นจริงที่ `comment_count` เป็นตัวเลข — เอกสารที่ผิดแบบนี้อันตรายกว่าไม่มี
เอกสารเลยด้วยซ้ำ เพราะทีม frontend จะเขียนโค้ด parse เป็น string โดยเข้าใจผิด

### 484.6 ตรวจสอบผลลัพธ์หลัง `@extend_schema`

```bash
python manage.py spectacular --file schema.yaml --validate
# ไม่ควรมี warning เกี่ยวกับ publish, by_category, comment_count อีกต่อไป
```

เปิดไฟล์ `schema.yaml` แล้วค้นหา `publish` — ตอนนี้ควรเห็น `summary`, `description`,
`responses` ครบทั้ง 200/403/404 พร้อมตัวอย่าง (`examples`) ที่เราเพิ่มไปแทนที่จะเห็น
เฉพาะ response 200 เดา ๆ แบบในขั้นตอนที่ 483.4

---

## ขั้นตอนที่ 485: ติดตั้ง Swagger UI และ Redoc เพื่อแสดงเอกสารแบบ interactive

### 485.1 ไฟล์ YAML อย่างเดียวยังไม่พอสำหรับทีมที่ไม่ใช่ backend

ไฟล์ `schema.yaml` ที่ได้จากขั้นตอนที่แล้วถูกต้องครบถ้วน แต่ **อ่านยากมากสำหรับมนุษย์**
ทีม frontend/mobile ต้องการหน้าเว็บที่แปลงไฟล์นี้ให้อ่านง่าย และที่สำคัญที่สุดคือ
**กดทดลองยิง API ได้จริงจากเบราว์เซอร์โดยไม่ต้องเปิด Postman** — นี่คือหน้าที่ของ
**Swagger UI** และ **Redoc**

### 485.2 เพิ่ม view สำหรับ Swagger UI และ Redoc

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularRedocView,
    SpectacularSwaggerView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path(
        'api/docs/',
        SpectacularSwaggerView.as_view(url_name='schema'),
        name='swagger-ui',
    ),
    path(
        'api/redoc/',
        SpectacularRedocView.as_view(url_name='schema'),
        name='redoc',
    ),
]
```

เปิดเบราว์เซอร์ไปที่:

- **`http://127.0.0.1:8000/api/docs/`** → หน้า Swagger UI — แสดงทุก endpoint พร้อมปุ่ม
  "Try it out" ที่กดกรอกข้อมูลแล้วยิง request จริงจากหน้าเว็บได้เลย เห็น response
  จริงพร้อม status code และเวลาที่ใช้
- **`http://127.0.0.1:8000/api/redoc/`** → หน้า Redoc — เน้นการอ่านเอกสารแบบ 3 คอลัมน์
  (เมนู, รายละเอียด, ตัวอย่างโค้ด) อ่านง่ายกว่า Swagger UI มาก แต่ **ไม่มีปุ่มทดลองยิง
  request**

### 485.3 เมื่อไหร่ควรใช้ Swagger UI เมื่อไหร่ควรใช้ Redoc

| สถานการณ์ | เครื่องมือที่เหมาะสม |
|---|---|
| นักพัฒนา frontend/mobile กำลังเขียนโค้ดเชื่อมต่อ ต้องการทดลองยิง API จริง | **Swagger UI** |
| เอกสารสาธารณะสำหรับลูกค้า/partner ภายนอกที่แค่ต้องการอ่านทำความเข้าใจ | **Redoc** |
| QA ทดสอบ endpoint แบบรวดเร็วโดยไม่เปิด Postman | **Swagger UI** |
| ทีม Business/PM ต้องการดูภาพรวมว่า API มีอะไรบ้าง | **Redoc** (อ่านง่ายกว่า) |

โปรเจกต์จริงจำนวนมากเปิดให้ใช้งานทั้งสองตัวพร้อมกัน (ตามที่ตั้งค่าใน 485.2) แล้วให้
แต่ละทีมเลือกใช้ตามความถนัด

### 485.4 ปิดการเข้าถึงเอกสารใน production (สำคัญมาก!)

การเปิด Swagger UI แบบสาธารณะใน production เป็นความเสี่ยงด้าน security เพราะเผย
โครงสร้าง endpoint ทั้งหมดให้ผู้ไม่หวังดีเห็น ควรจำกัดการเข้าถึงด้วย permission:

```python
# config/urls.py
from django.conf import settings
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView
from rest_framework.permissions import IsAdminUser

urlpatterns = [
    # ...
]

if settings.DEBUG:
    # เปิดเอกสารแบบสาธารณะเฉพาะตอนพัฒนา (DEBUG=True) เท่านั้น
    urlpatterns += [
        path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
        path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    ]
else:
    # ใน production ให้เฉพาะ staff เข้าดูเอกสารได้ (ต้อง login ผ่าน admin ก่อน)
    urlpatterns += [
        path(
            'api/schema/',
            SpectacularAPIView.as_view(permission_classes=[IsAdminUser]),
            name='schema',
        ),
        path(
            'api/docs/',
            SpectacularSwaggerView.as_view(url_name='schema', permission_classes=[IsAdminUser]),
            name='swagger-ui',
        ),
    ]
```

**คำแนะนำระดับมืออาชีพ**: สำหรับ API สาธารณะที่ตั้งใจให้ third-party developer เข้าถึง
ได้ (เช่น API แบบ Stripe, Twilio) การเปิด Swagger UI แบบสาธารณะคือความตั้งใจ ไม่ใช่
ความเสี่ยง — แต่สำหรับ internal API ของบริษัทที่ไม่ต้องการให้คนนอกรู้โครงสร้างระบบ ควร
จำกัดสิทธิ์เสมอตามตัวอย่างข้างต้น

### 485.5 ใช้ asset แบบ offline ด้วย `drf-spectacular-sidecar` (ขั้นสูง)

ค่าเริ่มต้น Swagger UI และ Redoc โหลดไฟล์ CSS/JavaScript จาก CDN สาธารณะ ซึ่งมีปัญหา
สองอย่าง: (1) ใช้งานไม่ได้ถ้าเครื่อง/network ไม่มีอินเทอร์เน็ตออกนอก (เช่น intranet
องค์กร), (2) เสี่ยงด้าน security ถ้า CDN ถูกโจมตี (supply chain attack) แก้ได้ด้วย
package เสริม:

```bash
pip install drf-spectacular[sidecar]
```

```python
# config/settings.py
INSTALLED_APPS = [
    ...,
    'drf_spectacular',
    'drf_spectacular_sidecar',  # ← เพิ่มบรรทัดนี้ ต้องอยู่หลัง drf_spectacular
]

SPECTACULAR_SETTINGS = {
    # ...
    'SWAGGER_UI_DIST': 'SIDECAR',   # โหลด asset จากไฟล์ในเครื่องแทน CDN
    'SWAGGER_UI_FAVICON_HREF': 'SIDECAR',
    'REDOC_DIST': 'SIDECAR',
}
```

```python
# config/urls.py — ต้องรวม static ของ sidecar เข้าไปด้วย (development เท่านั้น)
from django.urls import include, path

urlpatterns = [
    # ...
    path('', include('drf_spectacular_sidecar.urls')),
]
```

### 485.6 ตั้งค่า `SWAGGER_UI_SETTINGS` เพิ่มเติมที่ใช้บ่อย

```python
SPECTACULAR_SETTINGS = {
    # ...
    'SWAGGER_UI_SETTINGS': {
        # จำ token ที่กรอกใน Authorize ไว้แม้ refresh หน้าเว็บ (สำคัญมากเวลาทดสอบ JWT)
        'persistAuthorization': True,
        # เรียง endpoint ตามลำดับตัวอักษรแทนลำดับที่เจอในโค้ด
        'operationsSorter': 'alpha',
        # เปิด/ปิดส่วนแสดงโครงสร้าง schema แบบละเอียดของแต่ละ endpoint
        'docExpansion': 'list',
    },
}
```

`persistAuthorization: True` มีประโยชน์มากในขั้นตอนถัดไปที่เราจะทดสอบ endpoint ที่
ต้องมี JWT token — ถ้าไม่เปิดค่านี้ ทุกครั้งที่ refresh หน้า Swagger UI จะต้องกรอก
token ใหม่ทุกครั้ง

---

## ขั้นตอนที่ 486: การ document Authentication scheme ใน schema (JWT, Token)

### 486.1 ปัญหา: Swagger UI ยังไม่รู้ว่าต้องส่ง JWT อย่างไร

แม้จะมี Swagger UI แล้ว ถ้าลองกดปุ่ม "Try it out" กับ endpoint `POST /api/posts/`
(ที่ต้อง login ตาม `IsAuthenticatedOrReadOnly` จาก Part 045) จะได้ `401 Unauthorized`
ทันที เพราะ Swagger UI ยังไม่รู้วิธีแนบ JWT token เข้ากับ request — ต้องบอกให้
`drf-spectacular` รู้จัก **Security Scheme** ของ authentication class ที่ใช้อยู่ก่อน

### 486.2 `SessionAuthentication`/`BasicAuthentication`/`TokenAuthentication`: รองรับในตัวอยู่แล้ว

ถ้าโปรเจกต์ใช้ authentication class มาตรฐานของ DRF เอง (`SessionAuthentication`,
`BasicAuthentication`) หรือ `TokenAuthentication` จาก `rest_framework.authtoken`
`drf-spectacular` จะสร้าง Security Scheme ให้อัตโนมัติทันทีที่เห็นค่าใน
`DEFAULT_AUTHENTICATION_CLASSES` **โดยไม่ต้องตั้งค่าเพิ่มเลย**

### 486.3 `djangorestframework-simplejwt`: รองรับผ่าน contrib module อัตโนมัติ

โปรเจกต์ของเราใช้ `JWTAuthentication` จาก `rest_framework_simplejwt` (ติดตั้งไปแล้ว
ใน Part 046) `drf-spectacular` มาพร้อม **contrib extension** สำหรับ package นี้
โดยเฉพาะ ซึ่งจะถูกโหลดโดยอัตโนมัติทันทีที่เห็นว่าทั้งสอง package (`drf_spectacular`
และ `rest_framework_simplejwt`) ติดตั้งอยู่ในโปรเจกต์เดียวกัน — **ไม่ต้องตั้งค่าอะไร
เพิ่มเลย** เพียงแค่:

```python
# config/settings.py — ตั้งค่านี้ไว้แล้วตั้งแต่ Part 046
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

รัน `python manage.py spectacular --file schema.yaml` อีกครั้ง แล้วเปิดดูส่วน
`components.securitySchemes` จะเห็น:

```yaml
components:
  securitySchemes:
    jwtAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

และทุก endpoint ที่มี permission ต้อง authenticate จะมี:

```yaml
security:
  - jwtAuth: []
```

### 486.4 ทดลองยิง API ที่ต้อง login ผ่าน Swagger UI

1. เปิด `http://127.0.0.1:8000/api/docs/`
2. ยิง `POST /api/token/` (จาก Part 046) ด้วย username/password เพื่อขอ
   `access` token
3. คัดลอกค่า `access` token ที่ได้
4. กดปุ่ม **"Authorize"** มุมขวาบนของหน้า Swagger UI (มีรูปกุญแจ)
5. เลือกช่อง `jwtAuth` แล้วพิมพ์ `Bearer <access-token-ที่คัดลอกมา>` (ต้องมีคำว่า
   `Bearer` นำหน้าเสมอ) แล้วกด Authorize
6. ปิดหน้าต่าง แล้วลองยิง `POST /api/posts/` — ตอนนี้ Swagger UI จะแนบ header
   `Authorization: Bearer <token>` ให้อัตโนมัติทุก request ที่กดทดสอบ

เพราะเราตั้ง `persistAuthorization: True` ไว้ในขั้นตอนที่ 485.6 การ Authorize นี้จะ
คงอยู่แม้ refresh หน้าเว็บ

### 486.5 document custom authentication class ที่ไม่มีใน contrib module

ถ้าโปรเจกต์มี authentication class ที่เขียนเองทั้งหมด (เช่น อ่าน API key จาก header
ชื่อเฉพาะของบริษัท) `drf-spectacular` จะเดา security scheme ไม่ได้เลย ต้องเขียน
`OpenApiAuthenticationExtension` เอง:

```python
# blog/authentication.py
from rest_framework.authentication import BaseAuthentication


class ApiKeyAuthentication(BaseAuthentication):
    """Authentication แบบ custom ที่อ่าน API key จาก header X-API-Key"""

    def authenticate(self, request):
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            return None
        # ... ตรรกะตรวจสอบ api_key จริง (ตัดออกเพื่อความกระชับ)
        return (user, None)
```

```python
# blog/schema.py
from drf_spectacular.extensions import OpenApiAuthenticationExtension


class ApiKeyAuthenticationScheme(OpenApiAuthenticationExtension):
    # ระบุ path แบบ string ไปยัง authentication class เป้าหมาย
    target_class = 'blog.authentication.ApiKeyAuthentication'
    name = 'apiKeyAuth'  # ชื่อที่จะไปปรากฏใน securitySchemes

    def get_security_definition(self, auto_schema):
        return {
            'type': 'apiKey',
            'in': 'header',
            'name': 'X-API-Key',
            'description': 'ใส่ API key ที่ได้รับจากทีม Backend',
        }
```

ต้อง import ไฟล์ `blog/schema.py` นี้ให้ Django โหลดตอน startup (เช่นใน `apps.py`
ของแอป `blog` ผ่าน `ready()`) ไม่เช่นนั้น `drf-spectacular` จะไม่รู้จัก extension นี้
เพราะยังไม่เคย import class ที่สืบทอดจาก `OpenApiAuthenticationExtension` เลย

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        import blog.schema  # noqa: F401 — ให้ drf-spectacular เห็น extension นี้
```

### 486.6 ตารางสรุป Authentication class ที่ `drf-spectacular` รองรับ

| Authentication Class | รองรับอัตโนมัติหรือไม่ | หมายเหตุ |
|---|---|---|
| `SessionAuthentication` | ✅ อัตโนมัติ | ใช้ cookie-based session |
| `BasicAuthentication` | ✅ อัตโนมัติ | HTTP Basic Auth |
| `TokenAuthentication` (`rest_framework.authtoken`) | ✅ อัตโนมัติ | Bearer token แบบง่าย |
| `JWTAuthentication` (`rest_framework_simplejwt`) | ✅ อัตโนมัติผ่าน contrib module | ต้องติดตั้งทั้งสอง package คู่กัน |
| `OAuth2Authentication` (`django-oauth-toolkit`) | ✅ อัตโนมัติผ่าน contrib module | ต้องติดตั้ง `django-oauth-toolkit` |
| Authentication class ที่เขียนเอง | ❌ ต้องเขียน `OpenApiAuthenticationExtension` เอง | ตามขั้นตอนที่ 486.5 |

---

## ขั้นตอนที่ 487: จัดการเอกสารสำหรับ API หลาย version พร้อมกัน (เชื่อมกับ Part 048)

### 487.1 ทบทวน Versioning จาก Part 048

Part 048 แนะนำการทำ **API Versioning** ด้วย `URLPathVersioning` เพื่อรองรับการ
เปลี่ยนแปลงแบบ breaking change โดยไม่กระทบผู้ใช้เดิม:

```python
# config/settings.py (จาก Part 048)
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'DEFAULT_VERSION': 'v1',
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

```python
# config/urls.py (จาก Part 048)
from django.urls import include, path

urlpatterns = [
    path('api/<str:version>/', include('blog.api_urls')),
]
```

สมมติว่า **v2** เปลี่ยนชื่อ field `content` ของ `Post` เป็น `body` (breaking change)
ในขณะที่ **v1** ยังต้องใช้ต่อไปสำหรับผู้ใช้เก่าที่ยังไม่ได้อัปเกรด — ปัญหาคือ ถ้า
generate schema เดียวรวมทั้ง v1 และ v2 เข้าด้วยกัน ทีม frontend จะสับสนว่า field ไหน
เป็นของเวอร์ชันไหน

### 487.2 แยก schema endpoint ต่อ version ด้วย `api_version`

`SpectacularAPIView` รับ parameter `api_version` (ส่งผ่าน `.as_view()`) เพื่อกรองว่า
ให้ generate schema เฉพาะ endpoint ของ version นั้น ๆ เท่านั้น โดยเทียบกับ
`versioning_class` ที่ตั้งไว้ในระบบ:

```python
# config/urls.py
from django.urls import include, path
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularRedocView,
    SpectacularSwaggerView,
)

urlpatterns = [
    path('api/<str:version>/', include('blog.api_urls')),

    # Schema + เอกสารแยกต่างหากสำหรับแต่ละ version
    path(
        'api/v1/schema/',
        SpectacularAPIView.as_view(api_version='v1'),
        name='schema-v1',
    ),
    path(
        'api/v1/docs/',
        SpectacularSwaggerView.as_view(url_name='schema-v1'),
        name='swagger-v1',
    ),
    path(
        'api/v2/schema/',
        SpectacularAPIView.as_view(api_version='v2'),
        name='schema-v2',
    ),
    path(
        'api/v2/docs/',
        SpectacularSwaggerView.as_view(url_name='schema-v2'),
        name='swagger-v2',
    ),
]
```

ตอนนี้:

- `http://127.0.0.1:8000/api/v1/docs/` แสดงเฉพาะ endpoint และ field ของ v1
  (`content`)
- `http://127.0.0.1:8000/api/v2/docs/` แสดงเฉพาะ endpoint และ field ของ v2 (`body`)

ทีม frontend ที่ยังใช้ v1 อยู่จะไม่เห็น field `body` ของ v2 มาสับสนเลย และในทาง
กลับกันทีมที่ย้ายไป v2 แล้วก็จะไม่เห็น field `content` เก่าที่เลิกใช้แล้ว

### 487.3 ทำเครื่องหมาย field/endpoint ที่กำลังจะถูกเลิกใช้ด้วย `deprecated`

ระหว่างช่วงเปลี่ยนผ่านจาก v1 ไป v2 การบอกทีม frontend ล่วงหน้าว่า endpoint ไหนกำลัง
จะถูกถอดออกเป็นมารยาทสำคัญของ API design ที่ดี:

```python
# blog/viewsets_v1.py — ViewSet ของ v1 ที่กำลังจะถูกแทนที่
@extend_schema_view(
    list=extend_schema(
        deprecated=True,
        description=(
            'เวอร์ชันนี้ (v1) จะถูกถอดออกในวันที่ 1 มกราคม 2027 '
            'กรุณาย้ายไปใช้ `/api/v2/posts/` แทน field `content` เปลี่ยนชื่อเป็น `body`'
        ),
    ),
)
class PostViewSetV1(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializerV1
    versioning_class = URLPathVersioning
```

เมื่อ endpoint ถูกทำเครื่องหมาย `deprecated=True` Swagger UI จะแสดงแถบสีเทาขีดฆ่า
พร้อมข้อความเตือนที่หัวข้อ endpoint นั้นโดยอัตโนมัติ ทำให้ทีม frontend สังเกตเห็นได้
ทันทีโดยไม่ต้องอ่านอีเมลประกาศแยกต่างหาก

### 487.4 ตาราง: กลยุทธ์จัดการเอกสารหลาย version

| กลยุทธ์ | วิธีทำ | เหมาะกับ |
|---|---|---|
| **Schema แยกต่อ version** (487.2) | `SpectacularAPIView.as_view(api_version='v1')` คนละ URL | โปรเจกต์ที่มี breaking change ชัดเจนระหว่าง version |
| **Schema เดียว + `deprecated`** (487.3) | ใช้ schema เดียว ทำเครื่องหมาย field/endpoint เก่า | ช่วงเปลี่ยนผ่านสั้น ๆ ที่ยังไม่อยาก maintain 2 ชุดเอกสารเต็มรูปแบบ |
| **แยก sub-domain** (เช่น `v1-api.example.com`) | deploy แยก instance ต่อ version | องค์กรใหญ่ที่ v1/v2 มี codebase แยกกันจริง ๆ |

หลักสูตรนี้แนะนำกลยุทธ์แรก (487.2) เป็นค่าเริ่มต้น เพราะให้ความชัดเจนสูงสุดโดยไม่ต้อง
แยก deployment จริง ๆ

---

## ขั้นตอนที่ 488: เกริ่นการ generate client SDK จาก OpenAPI spec (openapi-generator)

### 488.1 ปัญหาที่ client SDK generation แก้ได้

เมื่อมี schema ที่สมบูรณ์แล้ว (จากขั้นตอนที่ 481-487) ทีม frontend/mobile ไม่จำเป็น
ต้องเขียนโค้ดเชื่อมต่อ API ด้วยมืออีกต่อไป (เช่น เขียน `fetch()`/`axios` เรียก
`/api/posts/` เอง, สร้าง TypeScript interface ของ `Post` เอง) เพราะ **openapi-generator**
สามารถอ่าน `schema.yaml` แล้วสร้างโค้ด client (SDK) ที่มี type ครบถ้วนให้อัตโนมัติ

### 488.2 ติดตั้ง `openapi-generator-cli`

```bash
# ผ่าน npm (ต้องมี Node.js ติดตั้งอยู่)
npm install @openapitools/openapi-generator-cli -g

# หรือผ่าน Docker (ไม่ต้องติดตั้ง Node.js เลย)
docker pull openapitools/openapi-generator-cli
```

### 488.3 Export schema ล่าสุดก่อน generate

```bash
python manage.py spectacular --file schema.yaml --validate
```

**คำแนะนำระดับมืออาชีพ**: ควร generate schema จาก server ที่รันจริง (ไม่ใช่แค่จากไฟล์
สแตติก) เพื่อให้แน่ใจว่า schema ตรงกับโค้ดล่าสุดเสมอ:

```bash
curl http://127.0.0.1:8000/api/v1/schema/ -o schema.yaml
```

### 488.4 Generate client TypeScript สำหรับทีม frontend

```bash
openapi-generator-cli generate \
  -i schema.yaml \
  -g typescript-axios \
  -o client/typescript \
  --additional-properties=supportsES6=true,npmName=blog-api-client
```

ผลลัพธ์คือโฟลเดอร์ `client/typescript/` ที่มี TypeScript class/interface ครบถ้วน
ทีม frontend เพียง `import` แล้วเรียกใช้ได้ทันทีโดยได้ type-checking เต็มรูปแบบ:

```typescript
import { PostsApi, Configuration } from 'blog-api-client';

const config = new Configuration({
    basePath: 'https://api.example.com',
    accessToken: 'eyJhbGciOi...',  // JWT access token จาก Part 046
});

const postsApi = new PostsApi(config);

// TypeScript รู้ shape ของ response และรู้ field ที่ต้องส่งครบทุกตัว
// จาก schema ที่ generate มา — ผิด field จะขึ้น error ตอน compile ทันที
const response = await postsApi.postsList({ categorySlug: 'django' });
console.log(response.data.results[0].title);
```

### 488.5 Generate client Python สำหรับทีม backend อื่นหรือ script อัตโนมัติ

```bash
openapi-generator-cli generate \
  -i schema.yaml \
  -g python \
  -o client/python \
  --package-name blog_api_client
```

```python
import blog_api_client
from blog_api_client.api import posts_api

configuration = blog_api_client.Configuration(host='https://api.example.com')
configuration.access_token = 'eyJhbGciOi...'

with blog_api_client.ApiClient(configuration) as api_client:
    api_instance = posts_api.PostsApi(api_client)
    posts = api_instance.posts_list(category_slug='django')
    print(posts.results[0].title)
```

### 488.6 Generator ยอดนิยมอื่น ๆ ที่ใช้บ่อยในงานจริง

| ภาษา/Platform | ชื่อ generator ที่ใช้กับ `-g` |
|---|---|
| TypeScript (fetch) | `typescript-fetch` |
| TypeScript (axios) | `typescript-axios` |
| Swift (iOS) | `swift5` |
| Kotlin (Android) | `kotlin` |
| Java | `java` |
| Go | `go` |
| C# (.NET) | `csharp` |
| PHP | `php` |

### 488.7 ข้อควรระวังของการใช้ client SDK ที่ generate อัตโนมัติ

- ควร **generate ใหม่ทุกครั้ง** ที่ schema เปลี่ยน ไม่ควรแก้โค้ดใน client ที่ generate
  มาด้วยมือ (เพราะจะถูกทับทิ้งตอน generate รอบถัดไป)
- ควรใส่ขั้นตอน generate client เป็นส่วนหนึ่งของ CI/CD (Part 088) เพื่อให้ client
  ไม่ตกรุ่นจาก API จริง
- ชื่อ field ในโค้ดที่ generate อาจถูกแปลงรูปแบบอัตโนมัติ (เช่น `is_published` ใน
  Python กลายเป็น `isPublished` ใน TypeScript) ตาม convention ของแต่ละภาษา — ควร
  ตรวจสอบให้แน่ใจว่าทีมเข้าใจการแปลงชื่อนี้

---

## ขั้นตอนที่ 489: เปรียบเทียบ `drf-spectacular` กับ `drf-yasg` (ทางเลือกเก่ากว่า)

### 489.1 `drf-yasg` คืออะไร

**`drf-yasg`** ("Yet Another Swagger Generator") เป็นเครื่องมือ generate เอกสาร API
สำหรับ DRF ที่ได้รับความนิยมสูงมากในช่วงปี 2018-2021 ก่อนที่ `drf-spectacular` จะ
กลายเป็นตัวเลือกหลัก โปรเจกต์เก่าจำนวนมากที่ยังคงใช้งานอยู่ทุกวันนี้จึงมักเจอ `drf-yasg`
ในโค้ด — หลักสูตรนี้จึงต้องรู้จักไว้เผื่อต้องไป maintain โปรเจกต์เก่า

### 489.2 หน้าตาโค้ดของ `drf-yasg` เพื่อเปรียบเทียบ

```python
# ตัวอย่างการใช้ drf-yasg (สำหรับเปรียบเทียบเท่านั้น — หลักสูตรนี้ไม่ใช้)
from drf_yasg.utils import swagger_auto_schema
from drf_yasg import openapi


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @swagger_auto_schema(
        operation_summary='เผยแพร่บทความ',
        responses={200: PostSerializer, 404: 'ไม่พบบทความ'},
        manual_parameters=[
            openapi.Parameter(
                'slug', openapi.IN_PATH, type=openapi.TYPE_STRING, required=True,
            ),
        ],
    )
    @action(detail=True, methods=['post'])
    def publish(self, request, slug=None):
        ...
```

สังเกตว่า syntax คล้ายกับ `@extend_schema` ของ `drf-spectacular` มาก (เพราะ
`drf-spectacular` ได้รับแรงบันดาลใจการออกแบบ API มาจาก `drf-yasg` โดยตรง) แต่ใช้
class `openapi.Parameter` แทน `OpenApiParameter` และใช้ `swagger_auto_schema` แทน
`extend_schema`

### 489.3 ตารางเปรียบเทียบเต็มรูปแบบ

| หัวข้อ | `drf-spectacular` | `drf-yasg` |
|---|---|---|
| **OpenAPI version ที่ generate ได้** | 3.0 และ 3.1 | **2.0 (Swagger) เท่านั้น** ไม่รองรับ OpenAPI 3 |
| **สถานะการดูแล (ปี 2026)** | Active — อัปเดตรองรับ DRF/Django เวอร์ชันใหม่สม่ำเสมอ | แทบไม่มี release ใหม่มาหลายปี ถือว่าอยู่ในสถานะ maintenance เท่านั้น |
| **กลไก generate schema หลัก** | Static introspection (อ่านโครงสร้างโค้ดจริง) + override ด้วย decorator เมื่อจำเป็น | พึ่งพา `swagger_auto_schema` decorator เป็นหลักมากกว่า introspect เอง |
| **รองรับ JWT (`simplejwt`) ในตัว** | ✅ ผ่าน contrib module อัตโนมัติ | ❌ ต้องเขียน security definition เองทั้งหมด |
| **รองรับ `django-filter` ในตัว** | ✅ อ่าน `filterset_fields` อัตโนมัติ | ต้องประกาศ `manual_parameters` เองทุกตัว |
| **การจัดการ API versioning** | มี `api_version` kwarg ในตัว (ขั้นตอนที่ 487) | ต้องเขียนกลไกแยกเอง |
| **คำแนะนำในเอกสารทางการของ DRF** | ✅ เป็นตัวเลือกที่แนะนำในหน้า Schema ของเอกสาร DRF | เคยแนะนำในอดีต ปัจจุบันไม่ใช่ตัวเลือกแรกแล้ว |
| **Bundle Swagger UI/Redoc แบบ offline** | ✅ ผ่าน `drf-spectacular-sidecar` (ขั้นตอนที่ 485.5) | มีในตัวแต่ config ยุ่งยากกว่า |
| **ความเหมาะสมสำหรับโปรเจกต์ใหม่ (2024+)** | ✅ แนะนำเป็นค่าเริ่มต้น | ไม่แนะนำสำหรับโปรเจกต์ใหม่ |

### 489.4 กรณีที่ยังอาจเจอ `drf-yasg` อยู่

- โปรเจกต์เก่าที่สร้างก่อนปี 2021 และยังไม่มีเวลา migrate
- ทีมที่ยังต้องการ Swagger 2.0 spec ตรง ๆ เพราะเครื่องมือภายในองค์กร (internal
  tooling) รองรับเฉพาะ Swagger 2.0 เท่านั้น (พบได้น้อยลงเรื่อย ๆ ในปี 2026)

### 489.5 แนวทาง migrate จาก `drf-yasg` ไปยัง `drf-spectacular`

1. ติดตั้ง `drf-spectacular` คู่ขนานกับ `drf-yasg` ก่อน (ไม่ต้องถอด `drf-yasg`
   ออกทันที)
2. ตั้ง `DEFAULT_SCHEMA_CLASS` เป็นของ `drf-spectacular` ตามขั้นตอนที่ 482.3
3. เปลี่ยน `@swagger_auto_schema` เป็น `@extend_schema` ทีละ ViewSet — เพราะ syntax
   คล้ายกันมาก (489.2) การแปลงส่วนใหญ่ทำได้แบบ 1:1
4. รัน `python manage.py spectacular --validate` เพื่อเช็คว่าไม่มี warning ตกหล่น
5. เมื่อแปลงครบทุก ViewSet แล้ว ค่อยถอด `drf-yasg` ออกจาก `requirements.txt` และ
   `INSTALLED_APPS`

**คำแนะนำระดับมืออาชีพ**: ไม่ควร migrate ทั้งโปรเจกต์ในครั้งเดียว โดยเฉพาะโปรเจกต์
ขนาดใหญ่ที่มี ViewSet เป็นร้อยตัว ให้ทำทีละแอป (app) แล้ว merge เป็น PR ย่อย ๆ
เพื่อลดความเสี่ยงและง่ายต่อการ review

---

## ขั้นตอนที่ 490: สรุปและแบบฝึกหัด — สร้างเอกสาร API ที่สมบูรณ์สำหรับ Blog API พร้อม Swagger UI

### 490.1 ประกอบทุกอย่างเข้าด้วยกัน: `config/settings.py` ฉบับสมบูรณ์

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
    'rest_framework_simplejwt',
    'django_filters',
    'drf_spectacular',
    'blog',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'ALLOWED_VERSIONS': ['v1'],
    'DEFAULT_VERSION': 'v1',
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'Blog API',
    'DESCRIPTION': (
        'REST API สำหรับระบบบล็อก รองรับการจัดการบทความ (Post), '
        'หมวดหมู่ (Category) และความคิดเห็น (Comment) พร้อม JWT Authentication'
    ),
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'COMPONENT_SPLIT_REQUEST': True,
    'SWAGGER_UI_SETTINGS': {
        'persistAuthorization': True,
        'operationsSorter': 'alpha',
    },
}
```

### 490.2 `blog/viewsets.py` ฉบับสมบูรณ์พร้อมเอกสารครบทุก endpoint

```python
# blog/viewsets.py
from django_filters.rest_framework import DjangoFilterBackend
from drf_spectacular.utils import (
    OpenApiExample,
    OpenApiParameter,
    OpenApiResponse,
    OpenApiTypes,
    extend_schema,
    extend_schema_view,
)
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.filters import OrderingFilter, SearchFilter
from rest_framework.permissions import IsAdminUser, IsAuthenticated, IsAuthenticatedOrReadOnly
from rest_framework.response import Response

from .models import Category, Comment, Post
from .permissions import IsOwnerOrReadOnly
from .serializers import CategorySerializer, CommentSerializer, PostSerializer


@extend_schema_view(
    list=extend_schema(
        summary='รายการบทความทั้งหมด',
        description='รองรับ filter ผ่าน `category__slug`, `is_published`, `search`, `ordering`',
        tags=['posts'],
    ),
    create=extend_schema(summary='สร้างบทความใหม่', tags=['posts']),
    retrieve=extend_schema(summary='ดูบทความ 1 ชิ้นตาม slug', tags=['posts']),
    update=extend_schema(summary='แก้ไขบทความทั้งหมด', tags=['posts']),
    partial_update=extend_schema(summary='แก้ไขบทความบางส่วน', tags=['posts']),
    destroy=extend_schema(summary='ลบบทความ', tags=['posts']),
)
class PostViewSet(viewsets.ModelViewSet):
    """ViewSet สำหรับจัดการบทความในระบบบล็อก"""

    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    lookup_field = 'slug'
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['category__slug', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @extend_schema(
        summary='เผยแพร่บทความ',
        request=None,
        responses={
            200: PostSerializer,
            403: OpenApiResponse(description='ไม่ใช่เจ้าของบทความ'),
        },
        examples=[
            OpenApiExample(
                'ตัวอย่างสำเร็จ',
                value={'id': 1, 'title': 'ทดสอบ', 'is_published': True},
                response_only=True,
                status_codes=['200'],
            ),
        ],
        tags=['posts'],
    )
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, slug=None):
        post = self.get_object()
        post.is_published = True
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)

    @extend_schema(
        summary='ยกเลิกเผยแพร่บทความ',
        request=None,
        responses={200: PostSerializer},
        tags=['posts'],
    )
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def unpublish(self, request, slug=None):
        post = self.get_object()
        post.is_published = False
        post.save(update_fields=['is_published', 'updated_at'])
        return Response(self.get_serializer(post).data)

    @extend_schema(
        summary='ค้นหาบทความตามหมวดหมู่',
        parameters=[
            OpenApiParameter(
                name='category',
                type=OpenApiTypes.STR,
                location=OpenApiParameter.QUERY,
                required=True,
                description='slug ของหมวดหมู่',
            ),
        ],
        responses={
            200: PostSerializer(many=True),
            400: OpenApiResponse(description='ไม่ได้ส่ง query parameter category'),
        },
        tags=['posts'],
    )
    @action(detail=False)
    def by_category(self, request):
        category_slug = request.query_params.get('category')
        if not category_slug:
            return Response({'detail': 'ต้องระบุ query parameter category'}, status=400)
        queryset = self.get_queryset().filter(category__slug=category_slug)
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)


@extend_schema_view(
    list=extend_schema(summary='รายการหมวดหมู่ทั้งหมด', tags=['categories']),
    retrieve=extend_schema(summary='ดูหมวดหมู่ 1 รายการ', tags=['categories']),
)
class CategoryViewSet(viewsets.ModelViewSet):
    """ViewSet สำหรับจัดการหมวดหมู่บทความ — สร้าง/แก้ไข/ลบได้เฉพาะ staff เท่านั้น"""

    queryset = Category.objects.all()
    serializer_class = CategorySerializer

    def get_permissions(self):
        if self.action in ('create', 'update', 'partial_update', 'destroy'):
            return [IsAdminUser()]
        return [IsAuthenticatedOrReadOnly()]


@extend_schema_view(
    list=extend_schema(summary='รายการความคิดเห็นของบทความหนึ่ง ๆ', tags=['comments']),
    create=extend_schema(summary='แสดงความคิดเห็นใหม่', tags=['comments']),
)
class CommentViewSet(viewsets.ModelViewSet):
    """ViewSet ของ Comment — เป็น nested resource ของ Post (Part 044)"""

    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    def get_queryset(self):
        return Comment.objects.filter(post_id=self.kwargs['post_pk'])

    def perform_create(self, serializer):
        serializer.save(post_id=self.kwargs['post_pk'], author=self.request.user)
```

### 490.3 `config/urls.py` ฉบับสมบูรณ์

```python
# config/urls.py
from django.conf import settings
from django.contrib import admin
from django.urls import include, path
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularRedocView,
    SpectacularSwaggerView,
)
from rest_framework.permissions import IsAdminUser

schema_permission = [] if settings.DEBUG else [IsAdminUser]

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.api_urls')),
    path(
        'api/schema/',
        SpectacularAPIView.as_view(permission_classes=schema_permission),
        name='schema',
    ),
    path(
        'api/docs/',
        SpectacularSwaggerView.as_view(url_name='schema', permission_classes=schema_permission),
        name='swagger-ui',
    ),
    path(
        'api/redoc/',
        SpectacularRedocView.as_view(url_name='schema', permission_classes=schema_permission),
        name='redoc',
    ),
]
```

### 490.4 ตรวจสอบผลลัพธ์สุดท้าย

```bash
# 1. ตรวจว่า schema ถูกต้องไม่มี warning
python manage.py spectacular --file schema.yaml --validate

# 2. เปิด Swagger UI ทดลองยิงทุก endpoint
python manage.py runserver
# เปิด http://127.0.0.1:8000/api/docs/

# 3. เปิด Redoc เพื่อดูมุมมองอ่านง่าย
# เปิด http://127.0.0.1:8000/api/redoc/

# 4. Generate client SDK ให้ทีม frontend ใช้ทดสอบ
openapi-generator-cli generate -i schema.yaml -g typescript-axios -o client/typescript
```

### 490.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า OpenAPI Specification คือมาตรฐานเปิดสำหรับอธิบาย REST API และ
  แตกต่างจาก "Swagger" (ชุดเครื่องมือ) อย่างไร
- ✅ ติดตั้งและตั้งค่า `drf-spectacular` ผ่าน `DEFAULT_SCHEMA_CLASS` และ
  `SPECTACULAR_SETTINGS`
- ✅ เข้าใจว่า `drf-spectacular` generate schema จากการ introspect
  Serializer/ViewSet ที่มีอยู่แล้วโดยอัตโนมัติ และรู้ว่าอะไรที่มันเดาไม่ได้
- ✅ ใช้ `@extend_schema`, `@extend_schema_view`, `@extend_schema_field` ปรับแต่ง
  schema ให้ครบถ้วนแม่นยำ
- ✅ เปิด Swagger UI และ Redoc ให้ทีมอื่นทดลองยิง API ได้จริงจากเบราว์เซอร์
- ✅ ทำให้ Swagger UI รู้จัก JWT Authentication ผ่าน contrib module และทดสอบ
  endpoint ที่ต้อง login ได้จริง
- ✅ แยกเอกสารสำหรับ API หลาย version พร้อมกันด้วย `api_version` kwarg และทำ
  เครื่องหมาย `deprecated`
- ✅ เข้าใจภาพรวมการ generate client SDK จาก OpenAPI spec ด้วย `openapi-generator`
- ✅ เปรียบเทียบ `drf-spectacular` กับ `drf-yasg` และรู้แนวทาง migrate
- ✅ ประกอบเอกสาร API ที่สมบูรณ์สำหรับ Blog API ทั้งระบบพร้อม Swagger UI ใช้งานจริง

### 490.6 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `drf-spectacular` และตั้ง `DEFAULT_SCHEMA_CLASS` สำเร็จ
- [ ] รัน `python manage.py spectacular --file schema.yaml --validate` แล้วไม่มี
      warning
- [ ] เปิด `/api/docs/` (Swagger UI) และ `/api/redoc/` (Redoc) ได้จริงบนเครื่องของคุณ
- [ ] ใช้ `@extend_schema_view` ตกแต่ง action มาตรฐานของ `ModelViewSet` ได้อย่างน้อย
      1 ViewSet
- [ ] ใช้ `@extend_schema` ตกแต่ง custom action (`@action`) พร้อมระบุ `responses`
      หลาย status code ได้
- [ ] ทดสอบ login ผ่าน Swagger UI ด้วยปุ่ม "Authorize" แล้วยิง endpoint ที่ต้อง
      authenticate ได้สำเร็จ
- [ ] อธิบายความแตกต่างระหว่าง `drf-spectacular` กับ `drf-yasg` ได้อย่างน้อย 3 ข้อ
      โดยไม่ต้องเปิดตารางเปรียบเทียบดู
- [ ] จำกัดสิทธิ์การเข้าถึงเอกสาร API ให้เฉพาะ staff เมื่อ `DEBUG = False`

### 490.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เพิ่ม `@extend_schema_view` ให้กับ `CommentViewSet`
ใน 490.2 ให้ครบทั้ง `list`, `create`, `retrieve`, `update`, `partial_update`,
`destroy` พร้อม `summary` ภาษาไทยที่สื่อความหมายชัดเจนสำหรับแต่ละ action แล้ว
ตรวจสอบผลลัพธ์ผ่าน Swagger UI ว่าแสดงข้อความที่ตั้งใจไว้ครบถูกต้อง

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม `SerializerMethodField` ชื่อ `is_owner` ใน
`PostSerializer` ที่คืนค่า `True`/`False` ว่าผู้ใช้ปัจจุบัน (`request.user`) เป็น
เจ้าของบทความนั้นหรือไม่ แล้วใช้ `@extend_schema_field(OpenApiTypes.BOOL)` กำหนด
type ให้ถูกต้อง ตรวจสอบว่า schema ที่ generate ออกมาระบุ `type: boolean` (ไม่ใช่
`string` ที่เป็นค่า fallback)

**แบบฝึกหัดที่ 3 (Versioning)**: ตั้งค่าโปรเจกต์ของคุณให้มี 2 version (`v1`, `v2`)
ตามขั้นตอนที่ 487 โดยให้ v2 เพิ่ม field ใหม่ชื่อ `reading_time_minutes` ใน
`PostSerializer` (คำนวณจากความยาว `content`) ที่ v1 ไม่มี แล้วสร้าง schema endpoint
แยกกันสำหรับทั้งสอง version ตรวจสอบว่า `/api/v1/docs/` ไม่มี field นี้ปรากฏ แต่
`/api/v2/docs/` มี

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน `OpenApiAuthenticationExtension` ของคุณเองสำหรับ
authentication class สมมติชื่อ `WebhookSignatureAuthentication` ที่ตรวจสอบ header
ชื่อ `X-Webhook-Signature` (ไม่ต้องเขียนตรรกะตรวจสอบลายเซ็นจริงก็ได้ แค่ให้
`authenticate()` คืน `None` เสมอเพื่อทดสอบ schema) แล้วตรวจสอบว่า
`components.securitySchemes` ใน `schema.yaml` มี security scheme ใหม่นี้ปรากฏถูกต้อง
พร้อมชื่อ header ที่ตั้งไว้

### 490.8 คำถามที่พบบ่อย (FAQ)

**Q: ต้องรัน `python manage.py spectacular --file schema.yaml` ทุกครั้งที่แก้โค้ด
หรือไม่?**
A: ไม่จำเป็นระหว่างพัฒนา เพราะ `SpectacularAPIView` ที่ `/api/schema/` จะ generate
schema แบบ **สด (on-the-fly)** ทุกครั้งที่มีการเรียกดู ไม่ต้องสร้างไฟล์ล่วงหน้า
คำสั่ง `spectacular --file` มีประโยชน์เมื่อต้องการไฟล์นิ่ง ๆ ไปใช้กับ
`openapi-generator` (ขั้นตอนที่ 488) หรือเก็บไว้ตรวจสอบใน CI/CD

**Q: ทำไม Swagger UI แสดง field บาง field เป็น `string` ทั้งที่ในโค้ดเป็นตัวเลข?**
A: มักเกิดจาก `SerializerMethodField` ที่ไม่ได้ใส่ `@extend_schema_field` กำกับไว้
(ขั้นตอนที่ 484.5) `drf-spectacular` จะ fallback เป็น `string` เสมอเมื่อเดา type
ของฟังก์ชัน Python ไม่ได้ — แก้ได้ด้วยการระบุ `@extend_schema_field(OpenApiTypes.INT)`
หรือใช้ type hint ที่ฟังก์ชันคืนค่าให้ชัดเจน (`-> int`) ซึ่ง `drf-spectacular`
เวอร์ชันใหม่อ่าน type hint ได้โดยตรงเช่นกัน

**Q: `drf-spectacular` ทำให้ระบบช้าลงหรือไม่ เพราะต้อง introspect ทุกครั้ง?**
A: การ introspect เกิดขึ้นเฉพาะตอนเรียก `/api/schema/` เท่านั้น (ไม่ใช่ทุก request
ของ API ปกติ) และ `drf-spectacular` มี built-in caching ผ่าน `SPECTACULAR_SETTINGS`
(`'SCHEMA_PATH_PREFIX'` และการตั้งค่า cache เพิ่มเติม) สำหรับ production ที่ schema
ไม่เปลี่ยนบ่อย ควร cache response ของ `/api/schema/` ไว้ด้วย Django cache framework
(Part 072) เพื่อลดภาระการ generate ซ้ำทุกครั้ง

**Q: จำเป็นต้อง generate client SDK ทุกโปรเจกต์หรือไม่?**
A: ไม่จำเป็น โปรเจกต์เล็กที่มีทีม frontend คนเดียวหรือทีมเดียวกับ backend อาจไม่คุ้มที่
จะตั้ง pipeline generate SDK แยก แต่สำหรับองค์กรที่มีทีม frontend/mobile/partner
ภายนอกหลายทีมใช้ API เดียวกัน การ generate SDK อัตโนมัติช่วยลดความคลาดเคลื่อนระหว่าง
เอกสารกับโค้ดจริงได้มาก และลดเวลาที่แต่ละทีมต้องเขียนโค้ดเชื่อมต่อเองซ้ำ ๆ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 050: Testing REST APIs** จะพา Blog API ที่เพิ่งมีเอกสารสมบูรณ์แบบใน Part นี้
ไปสู่ขั้นตอนสำคัญถัดไป: การเขียนชุดทดสอบอัตโนมัติ (automated test suite) ครอบคลุม
ทุก endpoint ด้วย `APITestCase` ของ DRF ทดสอบทั้ง permission, authentication,
serializer validation, custom action และ pagination ที่เรียนมาตลอด Phase 5 คุณจะได้
เรียนรู้การใช้ `APIClient` จำลอง request จริง, `force_authenticate()` เพื่อทดสอบโดย
ไม่ต้องขอ JWT token จริงทุกครั้ง, และเทคนิค mock ข้อมูลด้วย `factory_boy` เพื่อให้ชุด
ทดสอบรันเร็วและอ่านง่าย

เตรียมเปิด `blog/tests/` และไฟล์ `schema.yaml` ที่เพิ่ง generate ไว้ในขั้นตอนนี้ให้
พร้อม เพราะ Part 050 จะใช้เอกสาร OpenAPI เป็นข้อมูลอ้างอิงประกอบการออกแบบ test case
ให้ครอบคลุมทุก endpoint ที่ประกาศไว้จริง!
