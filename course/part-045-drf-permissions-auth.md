# Part 045: DRF Permissions และ Authentication

> **ขั้นตอนที่ 441-450 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: ปิดช่องว่างที่ Part 039 (ขั้นตอนที่ 385.3) ทิ้งไว้ตรง ๆ ว่า
> `DEFAULT_PERMISSION_CLASSES` และ `DEFAULT_AUTHENTICATION_CLASSES` จะ "เจาะลึกเต็มรูปแบบ
> ใน Part 045" คุณจะเรียนรู้ Permission class มาตรฐานของ DRF ทั้งหมด เขียน Custom
> Permission Class ของตัวเอง (`IsOwnerOrReadOnly`) เจาะลึก Authentication class หลัก
> สามตัว (`SessionAuthentication`, `BasicAuthentication`, `TokenAuthentication`) วิธีผสาน
> หลาย Authentication Class เข้าด้วยกันอย่างถูกลำดับ เชื่อม **django-guardian** จาก
> Part 038 เข้ากับ DRF ผ่าน `DjangoObjectPermissions` เขียน Test ครอบคลุม Permission
> Logic ทั้งหมด และเข้าใจกับดักเรื่อง CSRF ที่มือใหม่เกือบทุกคนเจอเมื่อผสม
> `SessionAuthentication` กับ API เมื่อจบ Part นี้คุณจะมีระบบ permission ระดับ production
> พร้อมสำหรับ Part 046 ที่จะพา JWT และ OAuth2 มาเสริมทัพ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 441: DRF Permission Classes มาตรฐาน — `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`
- ขั้นตอนที่ 442: เขียน Custom Permission Class เอง — `IsOwnerOrReadOnly`
- ขั้นตอนที่ 443: `DEFAULT_PERMISSION_CLASSES` ใน settings และการ override เฉพาะ view
- ขั้นตอนที่ 444: DRF Authentication Classes — `SessionAuthentication` เทียบกับ `BasicAuthentication`
- ขั้นตอนที่ 445: `TokenAuthentication` — `obtain_auth_token` และสร้าง Token ให้ user
- ขั้นตอนที่ 446: การผสมหลาย Authentication Class พร้อมกัน (ลำดับความสำคัญ)
- ขั้นตอนที่ 447: เชื่อม `DjangoObjectPermissions`/django-guardian จาก Part 038 เข้ากับ DRF ViewSet
- ขั้นตอนที่ 448: การเขียน Test สำหรับ Permission Logic ของ API
- ขั้นตอนที่ 449: ข้อควรระวังเรื่อง CSRF เมื่อใช้ `SessionAuthentication` กับ API
- ขั้นตอนที่ 450: สรุปและแบบฝึกหัด — ระบบ permission เต็มรูปแบบสำหรับ blog API

---

## ขั้นตอนที่ 441: DRF Permission Classes มาตรฐาน

### 441.1 ทบทวนเส้นทางที่พาเรามาถึงจุดนี้

Part 039 (ขั้นตอนที่ 385.4) ตั้งใจ**ไม่**ตั้งค่า `DEFAULT_PERMISSION_CLASSES` เพื่อให้
endpoint `/api/ping/` เปิดสาธารณะไว้ก่อน พร้อมทิ้งท้ายไว้ว่า:

> "ไม่เหมาะกับ endpoint ที่จัดการข้อมูลจริงของ blog เราจะกลับมาตั้งค่านี้อย่างรัดกุมใน
> Part 045"

ตอนนี้ถึงเวลานั้นแล้ว ก่อนไปต่อ มาทบทวนภาพรวมว่า Permission ของ DRF อยู่ตรงไหนใน
วงจรชีวิตของ request หนึ่งตัว โดยเทียบกับ `dispatch()` ที่เรียนละเอียดใน Part 042
(ขั้นตอนที่ 412.4):

```
HTTP Request
     │
     ▼
APIView.dispatch()
     │
     ├─ 1. initialize_request()      → สร้าง DRF Request, รู้ authentication_classes
     │
     ├─ 2. perform_authentication()  → เรียก request.user (lazy) → รัน Authentication
     │                                  Classes ทีละตัวจนกว่าจะได้ user หรือ AnonymousUser
     │
     ├─ 3. check_permissions()       → รัน Permission Classes ทีละตัว "ทุกตัวต้องผ่าน"
     │                                  (AND logic) ไม่ผ่านตัวไหนตัวหนึ่ง = 403 ทันที
     │
     ├─ 4. check_throttles()         → (เจาะลึกใน Part 048)
     │
     ▼
handler(request, *args, **kwargs)    → get()/post()/... ของคุณทำงานจริง
```

**หลักการสำคัญที่ต้องแยกให้ชัดตั้งแต่ต้น Part นี้**:

| แนวคิด | ตอบคำถามว่า | ล้มเหลวแล้วได้ status code |
|---|---|---|
| **Authentication** | "คุณเป็นใคร" (identify) | `401 Unauthorized` (ถ้าไม่ได้ authenticate และ permission ต้องการ) |
| **Permission** | "คุณทำสิ่งนี้ได้ไหม" (authorize) | `403 Forbidden` (authenticate แล้ว แต่ไม่มีสิทธิ์) |

ข้อนี้คือกฎเหล็กของ DRF: **Authentication ไม่เคยปฏิเสธ request เอง** มันแค่พยายามระบุ
ตัวตนแล้วเซ็ต `request.user`/`request.auth` ให้ ส่วน**การตัดสินใจอนุญาตหรือปฏิเสธเป็น
หน้าที่ของ Permission เท่านั้น** — ถ้าคุณเห็น 403 ทั้งที่ยังไม่ได้ login เลย นั่นเป็นเพราะ
Permission class ตรวจแล้วว่า `request.user.is_authenticated` เป็น `False` (ซึ่งจะกลาย
เป็น `AnonymousUser` เสมอเมื่อไม่มี authenticator ไหนยืนยันตัวตนได้) แล้วเลือกคืน 403
เอง (DRF จะเปลี่ยนเป็น 401 ให้อัตโนมัติถ้ามี authenticator ที่รองรับ `WWW-Authenticate`
header อยู่ในระบบ — รายละเอียดเต็มอยู่ในขั้นตอนที่ 446.3)

### 441.2 กายวิภาคของ `BasePermission`

Permission class ทุกตัวใน DRF สืบทอดจาก `rest_framework.permissions.BasePermission`
ซึ่งมี method หลักสองตัว:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.permissions.BasePermission
class BasePermission:
    def has_permission(self, request, view):
        """
        เช็คระดับ "view" ทั้งก้อน (ไม่รู้จัก object เฉพาะเจาะจง)
        เรียกทุกครั้งก่อน handler ทำงาน (ทั้ง list และ detail endpoint)
        ค่า default คืน True เสมอ (ไม่บล็อกอะไร ถ้าไม่ override)
        """
        return True

    def has_object_permission(self, request, view, obj):
        """
        เช็คระดับ "object" เฉพาะเจาะจง — เรียก **เฉพาะ** ตอนที่ view เรียก
        self.get_object() (เช่น RetrieveAPIView, UpdateAPIView, DestroyAPIView)
        และ get_object() ของ DRF จะเรียก check_object_permissions() ให้อัตโนมัติเสมอ
        ค่า default คืน True เสมอเช่นกัน
        """
        return True
```

จุดที่มือใหม่สับสนบ่อยที่สุด: **`has_object_permission()` ไม่ถูกเรียกใน list endpoint**
(เช่น `GET /api/posts/`) เพราะไม่มี object เดี่ยว ๆ ให้เช็ค คุณต้องกรอง queryset เองใน
`get_queryset()` ถ้าต้องการจำกัดว่า list เห็นอะไรได้บ้าง (จะเจาะลึกเรื่องนี้อีกครั้งใน
ขั้นตอนที่ 447.5)

### 441.3 Permission Classes มาตรฐาน 4 ตัวที่ใช้บ่อยที่สุด

DRF มากับ Permission class สำเร็จรูปใน `rest_framework.permissions` ให้ใช้ได้ทันที:

```python
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
)
```

| Permission Class | `has_permission()` คืน True เมื่อ | ใช้กับ endpoint แบบไหน |
|---|---|---|
| **`AllowAny`** | เสมอ (ไม่เช็คอะไรเลย) | Endpoint สาธารณะ เช่น `/api/ping/` (Part 039), หน้า register, health check |
| **`IsAuthenticated`** | `request.user` ผ่านการ authenticate แล้วเท่านั้น (`request.user.is_authenticated is True`) | Endpoint ที่ต้อง login ก่อนเข้าถึงได้ทุก method (เช่น `/api/me/`, `/api/orders/`) |
| **`IsAdminUser`** | `request.user.is_staff is True` เท่านั้น | Endpoint สำหรับทีมงานภายใน เช่น dashboard สถิติ, จัดการรายงาน |
| **`IsAuthenticatedOrReadOnly`** | `True` เสมอสำหรับ **safe method** (`GET`/`HEAD`/`OPTIONS`); สำหรับ method อื่น ต้อง authenticated | Endpoint สาธารณะที่ "อ่านได้ทุกคน แต่แก้ไขต้อง login" เช่น `/api/posts/` |

ซอร์สโค้ดจริงของแต่ละตัวสั้นมาก คุ้มค่าที่จะอ่านให้เข้าใจกลไก:

```python
# rest_framework/permissions.py (ย่อจากซอร์สโค้ดจริง)

class AllowAny(BasePermission):
    def has_permission(self, request, view):
        return True


class IsAuthenticated(BasePermission):
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)


class IsAdminUser(BasePermission):
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_staff)


class IsAuthenticatedOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        return bool(
            request.method in SAFE_METHODS
            or (request.user and request.user.is_authenticated)
        )
```

`SAFE_METHODS` คือ tuple คงที่ `('GET', 'HEAD', 'OPTIONS')` — method ที่ตามหลัก REST
**ไม่ควรเปลี่ยนแปลงสถานะของระบบ** (idempotent และไม่มีผลข้างเคียง) นี่คือ constant
ที่คุณจะเห็นซ้ำในเกือบทุก custom permission class ที่เขียนเองตลอด Part นี้

### 441.4 ใช้งานจริงกับ `PostViewSet`

เชื่อมกับ `PostViewSet` ที่สร้างด้วย `ModelViewSet` และลงทะเบียนผ่าน `DefaultRouter`
ตามที่วางไว้ใน Part 044:

```python
# blog/api_views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly

from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
```

```python
# blog/api_urls.py
from rest_framework.routers import DefaultRouter

from . import api_views

router = DefaultRouter()
router.register("posts", api_views.PostViewSet, basename="post")

urlpatterns = router.urls
```

ทดสอบผ่าน `curl` เพื่อยืนยันพฤติกรรม:

```bash
# GET ไม่ต้อง login เลย (safe method) → 200 OK
curl -i http://127.0.0.1:8000/api/posts/

# POST ไม่ได้แนบ credential ใด ๆ → 403 Forbidden (permission ปฏิเสธ)
curl -i -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -d '{"title": "ทดสอบ", "content": "เนื้อหา", "is_published": false}'
```

```
HTTP/1.1 403 Forbidden
Content-Type: application/json

{"detail":"Authentication credentials were not provided."}
```

### 441.5 permission_classes เป็น "รายการ" ที่ทำงานแบบ AND ทั้งหมด

จุดที่สำคัญอีกจุด: `permission_classes` เป็น **list** ไม่ใช่ค่าเดียว และ DRF จะรัน
**ทุกตัวในลิสต์** — ต้องผ่าน**ทุกตัว**ถึงจะเข้า handler ได้ (AND logic ไม่ใช่ OR):

```python
class PostViewSet(viewsets.ModelViewSet):
    # ต้อง "authenticated (หรืออ่านอย่างเดียว)" AND "เป็น staff" พร้อมกันทั้งสองเงื่อนไข
    permission_classes = [IsAuthenticatedOrReadOnly, IsAdminUser]
```

โค้ดข้างบนนี้ทำให้ **ไม่มีใครสร้าง/แก้ไขโพสต์ได้เลยนอกจาก staff** เพราะแม้จะ
`IsAuthenticatedOrReadOnly` ผ่าน (เป็น user ทั่วไปที่ login แล้ว) แต่ `IsAdminUser` จะ
ปฏิเสธถ้าไม่ใช่ staff — นี่คือกับดักที่มือใหม่เจอบ่อย: อยากได้ "staff หรือเจ้าของ" (OR)
แต่ list ให้แค่ AND เท่านั้น ทางแก้สำหรับ logic แบบ OR/NOT จะอยู่ในขั้นตอนที่ 443.6
(operator แบบ `|`, `&`, `~`) และการเขียน Custom Permission เองในขั้นตอนที่ 442 และ 450

### 441.6 ตารางสรุปพฤติกรรมเมื่อไม่ผ่าน Permission

| สถานการณ์ | `request.user` | ผลลัพธ์ |
|---|---|---|
| ไม่แนบ credential ใด ๆ + endpoint ต้องการ authenticator ที่รองรับ challenge (เช่น `BasicAuthentication`) | `AnonymousUser` | `401 Unauthorized` พร้อม header `WWW-Authenticate` |
| ไม่แนบ credential ใด ๆ + authenticator ที่ตั้งไว้ไม่รองรับ challenge (เช่น `SessionAuthentication` เท่านั้น) | `AnonymousUser` | `403 Forbidden` |
| แนบ credential แต่ผิด (username/password ผิด, token ไม่ถูกต้อง) | ไม่ authenticate สำเร็จ → `AuthenticationFailed` | `401 Unauthorized` เสมอ ไม่ว่าจะตั้ง authenticator ตัวไหน |
| Authenticate สำเร็จ แต่ permission ปฏิเสธ (เช่น ไม่ใช่ staff) | เป็น user จริง | `403 Forbidden` |

ความแตกต่างระหว่างแถวที่ 1 กับ 2 คือหัวใจของขั้นตอนที่ 446.3 ที่จะอธิบายว่า DRF ตัดสิน
401 กับ 403 จาก `authenticate_header()` ของ authenticator อย่างไร

---

## ขั้นตอนที่ 442: เขียน Custom Permission Class เอง — `IsOwnerOrReadOnly`

### 442.1 ทำไม Built-in Permission ไม่พอ

Permission 4 ตัวจากขั้นตอนที่ 441 ตอบคำถามระดับ "ใครคือใคร" (authenticated/staff/anon)
เท่านั้น แต่กฎที่พบบ่อยที่สุดในระบบบล็อกจริงคือ **"เจ้าของโพสต์เท่านั้นที่แก้ไข/ลบได้
คนอื่นอ่านได้อย่างเดียว"** ซึ่งเป็นกฎที่ผูกกับ **ข้อมูลของ object แต่ละตัว** ไม่ใช่กฎ
ตายตัวของทั้ง view — นี่คือจุดที่ต้องเขียน Custom Permission Class เอง โดยใช้
`has_object_permission()` จากขั้นตอนที่ 441.2 ที่ยังไม่ได้แตะ

สังเกตว่านี่คือปัญหาเดียวกับที่ Part 038 แก้ด้วย `AuthorRequiredMixin` + django-guardian
ฝั่ง Django ธรรมดา (ไม่ใช่ DRF) — ขั้นตอนนี้คือเวอร์ชัน DRF ของแนวคิดเดียวกัน

### 442.2 เขียน `IsOwnerOrReadOnly` ทีละบรรทัด

```python
# blog/permissions.py
from rest_framework.permissions import SAFE_METHODS, BasePermission


class IsOwnerOrReadOnly(BasePermission):
    """
    อนุญาตให้ทุกคนอ่านได้ (safe methods) แต่แก้ไข/ลบได้เฉพาะเจ้าของ (obj.author)
    เท่านั้น ต้องใช้คู่กับ permission อื่นที่บังคับ authentication มาก่อนเสมอ
    (เช่น IsAuthenticatedOrReadOnly) ไม่เช่นนั้น AnonymousUser ที่ยังไม่ authenticate
    ก็จะผ่าน has_object_permission() นี้ไปได้ถ้า obj.author เผอิญเป็น None
    """

    message = "คุณไม่ใช่เจ้าของข้อมูลนี้ จึงไม่มีสิทธิ์แก้ไขหรือลบ"

    def has_object_permission(self, request, view, obj):
        # (1) Safe method (GET/HEAD/OPTIONS) → อนุญาตเสมอ ไม่ว่าใครก็ตาม
        if request.method in SAFE_METHODS:
            return True

        # (2) Unsafe method (POST/PUT/PATCH/DELETE) → ต้องเป็นเจ้าของเท่านั้น
        return obj.author_id == request.user.id
```

จุดสำคัญที่ต้องอธิบายทีละส่วน:

1. **`message` attribute**: ถ้ากำหนดไว้ DRF จะใช้ข้อความนี้แทนข้อความ default
   ("You do not have permission to perform this action.") ใน response body ตอนคืน
   403 — ช่วยให้ client เข้าใจสาเหตุที่แท้จริงได้ทันทีโดยไม่ต้องเดา
2. **`obj.author_id` แทน `obj.author`**: ใช้ `_id` เพื่อเทียบ primary key ตรง ๆ
   โดยไม่ต้อง query ตาราง user เพิ่มอีกรอบ (ทบทวนแนวคิดนี้จาก Part 012 เรื่อง
   `_id` suffix ของ ForeignKey) ถ้า `obj.author` เป็น `None` ได้ (field `null=True`)
   ต้องเช็คก่อนเสมอ ไม่งั้น `AnonymousUser.id` ที่เป็น `None` จะเทียบเท่ากับ
   `obj.author_id = None` แล้ว "ผ่าน" อย่างผิดพลาด — นี่คือเหตุผลที่ข้อ (3) ด้านล่าง
   สำคัญมาก
3. **ต้องคู่กับ permission ที่บังคับ authenticated มาก่อนเสมอ**: `has_object_permission()`
   ตัวเดียวไม่พอสำหรับความปลอดภัย เพราะไม่มีการเช็คว่า `request.user` เป็นใครเลยใน
   safe method — ต้องประกาศคู่กันเสมอดังขั้นตอนที่ 442.4

### 442.3 `has_permission()` vs `has_object_permission()` — เมื่อไหร่ต้อง override อันไหน

| Method | เรียกเมื่อไหร่ | ใช้เช็คอะไร | ตัวอย่าง |
|---|---|---|---|
| `has_permission(request, view)` | **ทุก** request ก่อนเข้า handler เสมอ (ทั้ง list, create, detail) | เงื่อนไขที่ไม่ต้องพึ่งข้อมูล object เฉพาะเจาะจง | "ต้อง login ก่อนสร้างโพสต์ใหม่ได้" |
| `has_object_permission(request, view, obj)` | **เฉพาะ** เมื่อ view เรียก `self.get_object()` (retrieve/update/destroy) | เงื่อนไขที่ต้องเทียบกับข้อมูลจริงของ object นั้น | "แก้ไขได้เฉพาะโพสต์ของตัวเอง" |

สังเกตว่า `POST /api/posts/` (create) **ไม่มี object ให้เช็ค** เพราะยังไม่ถูกสร้าง —
`has_object_permission()` จึงไม่ถูกเรียกเลยตอน create แม้แต่ครั้งเดียว การควบคุมว่า
"ใครสร้างโพสต์ได้บ้าง" ต้องทำผ่าน `has_permission()` แทน (ซึ่งเป็นสิ่งที่
`IsAuthenticatedOrReadOnly` ทำอยู่แล้ว)

### 442.4 ผสาน Permission หลายตัวเข้าด้วยกันใน `PostViewSet`

```python
# blog/api_views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly

from .models import Post
from .permissions import IsOwnerOrReadOnly
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    def perform_create(self, serializer):
        # ตั้ง author เป็น user ปัจจุบันเสมอ ไม่ยอมให้ client ระบุ author เอง
        # (ทบทวนแนวคิดเดียวกับ form_valid() ของ PostCreateView ใน Part 024/038)
        serializer.save(author=self.request.user)
```

ลำดับการทำงานเมื่อ `PUT /api/posts/7/` เข้ามาจาก user คนหนึ่ง:

1. `IsAuthenticatedOrReadOnly.has_permission()` → `PUT` ไม่ใช่ safe method → ต้อง
   authenticated ก่อน → ถ้าไม่ authenticated หยุดที่นี่ (403/401 ตามขั้นตอนที่ 441.6)
2. `IsOwnerOrReadOnly.has_permission()` → **ไม่ได้ override** จึงใช้ default ของ
   `BasePermission` ที่คืน `True` เสมอ → ผ่านไปก่อน
3. `self.get_object()` ดึง `Post` id 7 มา แล้วเรียก `check_object_permissions(request, obj)`
   อัตโนมัติ (กลไกนี้อยู่ใน `GenericAPIView.get_object()` มาตั้งแต่ Part 043)
4. `IsAuthenticatedOrReadOnly.has_object_permission()` → ไม่ได้ override → `True` เสมอ
5. `IsOwnerOrReadOnly.has_object_permission()` → เช็คจริงว่า `obj.author_id == request.user.id`
   → ถ้าไม่ตรง คืน `False` → DRF ยิง `403 Forbidden` พร้อมข้อความจาก `message` attribute

### 442.5 ทดสอบผ่าน Browsable API และ curl

```bash
# user "narin" login แล้ว พยายามแก้โพสต์ของ "kai" (id=7)
curl -i -X PATCH http://127.0.0.1:8000/api/posts/7/ \
  -H "Authorization: Token <narin-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "แก้ไขโดยไม่ได้รับอนุญาต"}'
```

```
HTTP/1.1 403 Forbidden
Content-Type: application/json

{"detail":"คุณไม่ใช่เจ้าของข้อมูลนี้ จึงไม่มีสิทธิ์แก้ไขหรือลบ"}
```

ข้อความ error ตรงกับ `message` attribute ที่กำหนดไว้ทุกตัวอักษร — นี่คือประโยชน์ของการ
กำหนด `message` เอง

### 442.6 ข้อควรระวัง: `get_object()` ของ `ListAPIView`/action `list` ไม่เรียก `has_object_permission()`

ย้ำอีกครั้งจากขั้นตอนที่ 441.2: `GET /api/posts/` (list) จะคืนโพสต์ **ทุกตัว** ในระบบ
โดยไม่มีการกรองใด ๆ จาก `IsOwnerOrReadOnly` เลย เพราะไม่มีการเรียก `get_object()` ในหน้า
list — นี่ไม่ใช่บั๊ก แต่เป็นพฤติกรรมที่ถูกต้องตามการออกแบบ เพราะ "อ่านได้ทุกคน" คือกฎ
ของแอปนี้อยู่แล้ว (safe method ผ่านเสมอ) แต่ถ้าต้องการ endpoint แบบ "เห็นเฉพาะโพสต์ที่
ตัวเองแก้ไขได้" ต้องกรองที่ `get_queryset()` แทน ซึ่งจะกลับมาเจาะลึกเรื่องนี้อีกครั้งพร้อม
`get_objects_for_user()` ของ guardian ในขั้นตอนที่ 447.5

---

## ขั้นตอนที่ 443: `DEFAULT_PERMISSION_CLASSES` และการ Override เฉพาะ View

### 443.1 ตั้งค่า Default ทั้งโปรเจกต์ใน `REST_FRAMEWORK`

กลับมาที่ `REST_FRAMEWORK` dict ที่ Part 039 (ขั้นตอนที่ 385.2) วางแผนที่ไว้ให้ ตอนนี้
ตั้งค่าจริงตาม policy ของโปรเจกต์ `secureblog`:

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",
    ],
}
```

ค่านี้กลายเป็น **ค่าเริ่มต้นของทุก View/ViewSet ในโปรเจกต์** ที่ไม่ได้กำหนด
`permission_classes` ของตัวเองไว้ — นี่คือหลักการ **"ปลอดภัยโดยค่าเริ่มต้น"
(Secure by default)** ที่ Part 001 (ขั้นตอนที่ 2.3) พูดถึงไว้ตั้งแต่ Django core:
ถ้านักพัฒนาลืมกำหนด `permission_classes` ใน View ใหม่ที่สร้างขึ้นมาสักตัว ระบบจะ
**fallback ไปที่ค่านี้เสมอ** แทนที่จะเปิดโล่งแบบ `AllowAny` ซึ่งเป็นค่า default ดิบของ
DRF เอง (ทบทวนจาก Part 039 ขั้นตอนที่ 385.5)

### 443.2 กฎการ Override: `permission_classes` ระดับ View **แทนที่ทั้งหมด** ไม่ใช่ผสาน

```python
class PublicAnnouncementViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Announcement.objects.filter(is_published=True)
    serializer_class = AnnouncementSerializer
    permission_classes = [AllowAny]   # override ทั้งหมด ไม่สนใจ DEFAULT_PERMISSION_CLASSES เลย
```

ข้อควรระวังที่สำคัญมาก: การกำหนด `permission_classes` ใน View **ไม่ได้เพิ่มเข้าไปกับ**
`DEFAULT_PERMISSION_CLASSES` แต่ **แทนที่ทั้งลิสต์** เมื่อ View ไหนกำหนด
`permission_classes = [AllowAny]` แปลว่า View นั้นเปิดสาธารณะ 100% แม้ว่า global
default จะเป็น `IsAuthenticatedOrReadOnly` ก็ตาม — DRF อ่านค่าผ่าน
`self.get_permissions()` ที่เช็ค `permission_classes` ของ instance ก่อนเสมอ (attribute
lookup แบบ Python ปกติ ถ้า View กำหนดเองจะบัง class attribute ของ parent/settings)

### 443.3 `get_permissions()` — Override แบบ Dynamic ต่อ Action

ปัญหาที่พบบ่อยใน `ModelViewSet`: อยาก "list/retrieve เปิดให้ทุกคน แต่ create/update/
destroy ต้อง login" ซึ่งทำได้ด้วย `IsAuthenticatedOrReadOnly` อยู่แล้ว แต่ถ้าต้องการกฎ
ซับซ้อนกว่านั้น เช่น "destroy ต้องเป็น staff เท่านั้น ส่วน update แค่เป็นเจ้าของก็พอ"
ต้อง override method `get_permissions()` แทนการกำหนด `permission_classes` แบบ static:

```python
# blog/api_views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAdminUser, IsAuthenticatedOrReadOnly

from .models import Post
from .permissions import IsOwnerOrReadOnly
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer

    def get_permissions(self):
        """
        กำหนด permission ต่างกันตาม self.action:
        - destroy: เฉพาะ staff เท่านั้นที่ลบโพสต์ได้ (นโยบายเข้มกว่าการแก้ไข)
        - update/partial_update: เจ้าของเท่านั้น (หรือ staff ก็ได้ผ่าน IsOwnerOrStaff ใน 450)
        - action อื่นทั้งหมด: อ่านได้ทุกคน แก้ไขต้อง login
        """
        if self.action == "destroy":
            permission_classes = [IsAdminUser]
        elif self.action in ("update", "partial_update"):
            permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
        else:
            permission_classes = [IsAuthenticatedOrReadOnly]
        return [permission() for permission in permission_classes]
```

**จุดสำคัญที่มือใหม่พลาดบ่อย**: `get_permissions()` ต้องคืน **list ของ instance**
(`permission()` เรียก class ให้เป็น object) ไม่ใช่ list ของ class เฉย ๆ — ต่างจาก
`permission_classes` attribute ที่เป็น list ของ class ตรง ๆ (DRF instantiate ให้เอง
เบื้องหลังผ่าน `get_permissions()` เวอร์ชัน default ที่ `GenericAPIView` มีอยู่แล้ว)

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.generics.GenericAPIView
class GenericAPIView(views.APIView):
    def get_permissions(self):
        return [permission() for permission in self.permission_classes]
```

การ override `get_permissions()` เองก็คือการเขียนทับ method นี้ทั้งหมด โดยเปลี่ยน
`self.permission_classes` (ค่าคงที่) ให้กลายเป็น logic แบบ dynamic ตาม `self.action`

### 443.4 `self.action` มาจากไหน — ทบทวนจาก Part 044

`self.action` เป็น attribute ที่ `ViewSet` เซ็ตให้อัตโนมัติตอน `dispatch()` โดยดูจาก
mapping ที่ Router ผูกไว้ (`list`, `create`, `retrieve`, `update`, `partial_update`,
`destroy`, หรือชื่อ custom action ที่ประกาศด้วย `@action` decorator) — นี่คือเหตุผลที่
`get_permissions()` แบบ dynamic ทำได้เฉพาะกับ `ViewSet`/`GenericAPIView` เท่านั้น
`APIView` ธรรมดาที่เขียนแบบ Part 042 ไม่มี `self.action` ให้ใช้ (ต้องเช็ค
`request.method` แทน)

### 443.5 ลำดับความสำคัญ (Precedence) แบบสรุป

| ระดับ | กำหนดที่ไหน | Priority |
|---|---|---|
| 1. (สูงสุด) | `get_permissions()` override ใน View/ViewSet | ชนะทุกระดับ เพราะเป็น method ที่ถูกเรียกจริง |
| 2. | `permission_classes` attribute ใน View/ViewSet | ใช้เมื่อไม่ได้ override `get_permissions()` |
| 3. (ต่ำสุด) | `DEFAULT_PERMISSION_CLASSES` ใน `REST_FRAMEWORK` settings | ใช้เมื่อ View ไม่กำหนดอะไรเลยทั้งสองแบบข้างบน |

### 443.6 Composable Permissions — ผสม Logic แบบ OR/NOT ด้วย `|`, `&`, `~`

ทบทวนจากขั้นตอนที่ 441.5 ว่า list ธรรมดาทำได้แค่ AND ตั้งแต่ DRF 3.9 เป็นต้นมา
`BasePermission` รองรับ operator `&` (and), `|` (or), `~` (not) ทำให้เขียน logic ซับซ้อน
ได้โดยไม่ต้องเขียน class ใหม่ทุกครั้ง:

```python
from rest_framework.permissions import IsAdminUser

from .permissions import IsOwnerOrReadOnly

class PostViewSet(viewsets.ModelViewSet):
    # "เป็น staff" OR "เป็นเจ้าของ" — แก้ไขได้ถ้าเป็นอย่างใดอย่างหนึ่ง
    permission_classes = [IsAdminUser | IsOwnerOrReadOnly]
```

**ข้อควรระวัง**: `IsOwnerOrReadOnly` เดิมจากขั้นตอนที่ 442 ถูกออกแบบให้ปล่อยผ่าน safe
method เสมออยู่แล้ว การผสมกับ `|` แบบนี้ต้องเข้าใจว่า operator ทำงานทั้งใน
`has_permission()` และ `has_object_permission()` แยกกัน (DRF สร้าง class ลูกผสมที่
override ทั้งสอง method ให้ทำ OR/AND/NOT ของผลลัพธ์แต่ละฝั่ง) วิธีที่ปลอดภัยและอ่านง่าย
กว่าในระบบจริงคือเขียน Custom Permission Class ที่รวม logic ไว้ชัดเจนในที่เดียว (แบบที่
จะทำใน `IsOwnerOrStaff` ของขั้นตอนที่ 450) แทนการ chain operator หลายชั้นจนอ่านยาก

---

## ขั้นตอนที่ 444: Authentication Classes — `SessionAuthentication` เทียบกับ `BasicAuthentication`

### 444.1 Authentication ตอบคำถาม "คุณเป็นใคร" เท่านั้น

ทบทวนจากขั้นตอนที่ 441.1: Authentication class มีหน้าที่เดียวคือตรวจสอบ credential
ใน request แล้วคืนค่า `(user, auth)` tuple หรือ `None` (ถ้าไม่มี credential เลยและปล่อย
ให้ authenticator ตัวถัดไปลอง) หรือ raise `AuthenticationFailed` (ถ้ามี credential แต่
ผิด) — **ไม่มี Authentication class ตัวไหนตัดสินใจ "อนุญาต" หรือ "ปฏิเสธ" การเข้าถึง
โดยตรง** หน้าที่นั้นเป็นของ Permission เสมอ

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.authentication.BaseAuthentication
class BaseAuthentication:
    def authenticate(self, request):
        """
        คืน None  → ไม่มี credential นี้ ให้ authenticator ตัวถัดไปลอง
        คืน (user, auth) → authenticate สำเร็จ
        raise AuthenticationFailed → มี credential แต่ผิด หยุดทันที ไม่ลองตัวถัดไป
        """
        raise NotImplementedError

    def authenticate_header(self, request):
        """คืนค่า header WWW-Authenticate สำหรับตอบ 401 (ถ้ามี)"""
        pass
```

### 444.2 `SessionAuthentication` — ใช้ Session เดิมจาก Django ธรรมดา

`SessionAuthentication` ใช้กลไก **เดียวกันทุกประการ** กับระบบ session/cookie ที่เรียน
เต็มรูปแบบใน Part 034 — ไม่มีแนวคิดใหม่เลย เพียงแค่เชื่อม `request.session` เดิมของ
Django เข้ากับ DRF request object:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.authentication.SessionAuthentication
class SessionAuthentication(BaseAuthentication):
    def authenticate(self, request):
        user = getattr(request._request, "user", None)

        if not user or not user.is_active:
            return None

        self.enforce_csrf(request)   # สำคัญมาก — ดูขั้นตอนที่ 449
        return (user, None)
```

`request._request` คือ `HttpRequest` ดิบของ Django ที่ `SessionMiddleware` +
`AuthenticationMiddleware` (Part 031) ประมวลผล `sessionid` cookie ไปแล้วก่อนถึง DRF
— `SessionAuthentication` แค่ "อ่านค่า" `request.user` ที่ Django เตรียมไว้ให้แล้วมา
ใช้ต่อ ไม่ได้ทำอะไรใหม่ นี่คือเหตุผลที่ **Browsable API** (Part 039 ขั้นตอนที่ 384) ใช้
งานได้ทันทีหลัง login ผ่านหน้า Django ธรรมดา โดยไม่ต้องแนบ header อะไรเพิ่มเลย —
เบราว์เซอร์ส่ง cookie `sessionid` แนบไปกับทุก request อัตโนมัติอยู่แล้ว

### 444.3 `BasicAuthentication` — HTTP Basic Auth มาตรฐาน RFC 7617

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.authentication.BasicAuthentication
import base64

class BasicAuthentication(BaseAuthentication):
    def authenticate(self, request):
        auth_header = request.META.get("HTTP_AUTHORIZATION", "")
        if not auth_header.startswith("Basic "):
            return None

        encoded = auth_header.split(" ", 1)[1]
        decoded = base64.b64decode(encoded).decode("utf-8")
        username, password = decoded.split(":", 1)

        user = authenticate(username=username, password=password)
        if user is None or not user.is_active:
            raise exceptions.AuthenticationFailed("Invalid username/password.")

        return (user, None)

    def authenticate_header(self, request):
        return 'Basic realm="api"'
```

`BasicAuthentication` ส่ง **username:password แบบ base64-encode** (ไม่ใช่เข้ารหัส —
base64 ถอดกลับเป็น plaintext ได้ทันทีด้วยใครก็ตามที่ดักจับ header ได้) แนบมาใน header
`Authorization: Basic <base64>` **ทุก request** ไม่มี session หรือ cookie เกี่ยวข้อง
เลย ทำให้เป็น **stateless** อย่างแท้จริง แต่ก็หมายความว่า password ของ user เดินทาง
(แม้จะ encode) แนบไปกับทุก request เสมอ

```bash
# curl -u ทำ Basic Authentication ให้อัตโนมัติ
curl -i -u narin:supersecret123 http://127.0.0.1:8000/api/me/

# เทียบเท่ากับการแนบ header เองแบบนี้ (base64 ของ "narin:supersecret123")
curl -i http://127.0.0.1:8000/api/me/ \
  -H "Authorization: Basic bmFyaW46c3VwZXJzZWNyZXQxMjM="
```

**คำเตือนระดับ production ที่ต้องรู้**: `BasicAuthentication` **ต้องใช้ผ่าน HTTPS
เท่านั้นเด็ดขาด** เพราะถ้าเป็น HTTP ธรรมดา password จะเดินทางแบบเปิดเผยแทบทั้งหมด
(base64 ไม่ใช่การเข้ารหัส) เอกสารทางการของ DRF เตือนไว้ตรง ๆ ว่าเหมาะกับ "simple
testing" มากกว่าการใช้งานจริงกับ client ทั่วไป

### 444.4 ตารางเปรียบเทียบ `SessionAuthentication` กับ `BasicAuthentication`

| คุณสมบัติ | `SessionAuthentication` | `BasicAuthentication` |
|---|---|---|
| Credential ที่ส่งมา | Cookie `sessionid` | Header `Authorization: Basic <base64>` |
| State | Stateful (ต้องมี session เก็บฝั่ง server) | Stateless (ไม่มี session เก็บเลย) |
| ต้อง login ผ่านหน้าเว็บก่อนไหม | ต้อง (ผ่าน `LoginView`/`django.contrib.auth`) | ไม่ต้อง — ส่ง username/password ทุกครั้ง |
| CSRF เกี่ยวข้องหรือไม่ | **เกี่ยวข้องโดยตรง** (ดูขั้นตอนที่ 449) | ไม่เกี่ยวข้องเลย |
| เหมาะกับ client ประเภทไหน | Browsable API, SPA/JS ที่ same-origin กับ Django (ใช้ cookie เดียวกัน) | สคริปต์ทดสอบ, internal tool, curl/httpie ระหว่าง dev |
| ความเสี่ยงหลัก | CSRF ถ้าตั้งค่าไม่ถูกต้อง | Password เดินทางทุก request ถ้าไม่มี HTTPS |
| เหมาะกับ production API สาธารณะไหม | ไม่เหมาะ (mobile app ไม่มี cookie jar แบบเบราว์เซอร์) | ไม่เหมาะ (ไม่มีกลไก revoke โดยไม่เปลี่ยน password) |

จุดสรุปสำคัญ: **ทั้งสองตัวไม่เหมาะกับ API สาธารณะสำหรับ mobile app/SPA แยก origin**
— นี่คือเหตุผลที่ขั้นตอนที่ 445 จะแนะนำ `TokenAuthentication` เป็นทางเลือกที่สมดุล
กว่า และ Part 046 จะพา JWT เข้ามาสำหรับกรณีที่ต้องการ scale และ security ขั้นสูงกว่านี้
อีกขั้น

### 444.5 ตรวจสอบว่า Authenticator ไหนถูกใช้จริงผ่าน `request.successful_authenticator`

```python
# blog/api_views.py (ตัวอย่างเพื่อ debug — ไม่ได้ใช้งานจริงใน production)
from rest_framework.views import APIView
from rest_framework.response import Response


class WhoAmIAPIView(APIView):
    def get(self, request):
        authenticator = request.successful_authenticator
        return Response({
            "username": request.user.username if request.user.is_authenticated else None,
            "authenticated_via": type(authenticator).__name__ if authenticator else None,
        })
```

```bash
curl -s -u narin:supersecret123 http://127.0.0.1:8000/api/whoami/
# {"username": "narin", "authenticated_via": "BasicAuthentication"}
```

`request.successful_authenticator` เป็น attribute ที่ DRF เซ็ตให้อัตโนมัติหลัง
`perform_authentication()` สำเร็จ — มีประโยชน์มากตอน debug ระบบที่มีหลาย
Authentication class พร้อมกัน (ขั้นตอนที่ 446)

---

## ขั้นตอนที่ 445: `TokenAuthentication` — `obtain_auth_token` และสร้าง Token ให้ User

### 445.1 ทำไมต้องมี Token Authentication

`TokenAuthentication` แก้ปัญหาของทั้งสองตัวก่อนหน้า: **stateless เหมือน
`BasicAuthentication`** (ไม่ต้องเก็บ session ฝั่ง server) แต่ **ไม่ต้องส่ง password
ซ้ำทุก request** (ส่งแค่ครั้งเดียวตอน login เพื่อแลก token แล้วใช้ token นั้นตลอด) และ
**revoke ได้โดยไม่กระทบ password** (ลบ token ทิ้งก็พอ ไม่ต้องบังคับเปลี่ยนรหัสผ่าน) —
นี่คือรูปแบบที่ mobile app และ SPA (Single Page Application) นิยมใช้มากที่สุดก่อนที่
JWT จะเข้ามา (Part 046)

### 445.2 ติดตั้งและตั้งค่า `rest_framework.authtoken`

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
    "rest_framework.authtoken",   # <-- เพิ่มแอปนี้ (มากับ DRF อยู่แล้ว ไม่ต้อง pip install เพิ่ม)
    "guardian",
    "accounts",
    "blog",
]
```

```bash
python manage.py migrate
```

```
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, guardian, sessions
Running migrations:
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  ...
```

`authtoken` สร้างตาราง `authtoken_token` ที่มีโครงสร้างเรียบง่ายมาก:

```
authtoken_token
├── key          --> string 40 ตัวอักษร (random hex) คือตัว token เอง เป็น primary key
├── user_id      --> OneToOneField ไปยัง AUTH_USER_MODEL (1 user มีได้แค่ 1 token เท่านั้น)
└── created      --> วันเวลาที่สร้าง token
```

**ข้อจำกัดที่ต้องรู้**: ความสัมพันธ์เป็น `OneToOneField` ไม่ใช่ `ForeignKey` — แปลว่า
**1 user มีได้ token เดียวเท่านั้นในเวลาเดียวกัน** (ต่างจาก JWT ที่ออก token ใหม่ได้
ไม่จำกัดโดยไม่ต้อง invalidate ตัวเก่า) ถ้า user login จากหลายอุปกรณ์พร้อมกันจะใช้
token ตัวเดียวกันทั้งหมด — ข้อจำกัดนี้เป็นเหตุผลหลักที่ Part 046 จะแนะนำ JWT สำหรับ
ระบบที่ต้องรองรับ multi-device อย่างจริงจัง

### 445.3 `obtain_auth_token` — View สำเร็จรูปสำหรับแลก Token

DRF มี View สำเร็จรูปให้ใช้ทันทีโดยไม่ต้องเขียนเอง:

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path
from rest_framework.authtoken.views import obtain_auth_token

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("blog.api_urls")),
    path("api/auth-token/", obtain_auth_token, name="api-token-auth"),
]
```

```bash
curl -s -X POST http://127.0.0.1:8000/api/auth-token/ \
  -d "username=narin&password=supersecret123"
```

```json
{"token": "9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a"}
```

เบื้องหลัง `obtain_auth_token` คือ `APIView` ธรรมดาที่รับ username/password ผ่าน
Serializer (`AuthTokenSerializer`) เรียก `django.contrib.auth.authenticate()`
(ทบทวนจาก Part 031) แล้ว `get_or_create()` แถวใน `authtoken_token` ให้อัตโนมัติ:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.authtoken.views.ObtainAuthToken
class ObtainAuthToken(APIView):
    serializer_class = AuthTokenSerializer

    def post(self, request):
        serializer = self.serializer_class(data=request.data, context={"request": request})
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data["user"]
        token, created = Token.objects.get_or_create(user=user)
        return Response({"token": token.key})
```

### 445.4 ใช้ Token เรียก API จริง

```bash
curl -s http://127.0.0.1:8000/api/posts/7/ \
  -H "Authorization: Token 9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a"
```

รูปแบบ header คือ `Authorization: Token <key>` (สังเกตคำว่า `Token` ไม่ใช่ `Bearer`
ที่ JWT ใช้ — ต้องพิมพ์ตรงตามนี้เป๊ะ ถ้าพิมพ์ผิดจะได้ 401 ทันที)

เปิดใช้งานใน `REST_FRAMEWORK` settings:

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
}
```

### 445.5 สร้าง Token ให้ User อัตโนมัติทันทีที่สมัครสมาชิก

แนวทางที่ปลอดภัยที่สุดคือใช้ signal `post_save` เชื่อมกับ `AUTH_USER_MODEL` (ทบทวน
รูปแบบเดียวกับ `assign_author_object_permissions` ใน Part 038 ขั้นตอนที่ 373.5):

```python
# accounts/signals.py
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token


@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token_for_new_user(sender, instance, created, **kwargs):
    """สร้าง Token ให้ user ทันทีที่สมัครสมาชิกสำเร็จ ไม่ต้องรอเรียก /api/auth-token/ ครั้งแรก"""
    if created:
        Token.objects.create(user=instance)
```

```python
# accounts/apps.py
from django.apps import AppConfig


class AccountsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "accounts"

    def ready(self):
        import accounts.signals  # noqa: F401
```

### 445.6 Management Command: Backfill Token ให้ User เก่าที่มีอยู่แล้ว

เหมือนกับ `backfill_object_permissions` ใน Part 038 (ขั้นตอนที่ 373.6) สำหรับระบบที่
เพิ่งเปิดใช้ `authtoken` ทีหลัง:

```python
# accounts/management/commands/backfill_auth_tokens.py
from django.contrib.auth import get_user_model
from django.core.management.base import BaseCommand
from rest_framework.authtoken.models import Token

User = get_user_model()


class Command(BaseCommand):
    help = "สร้าง Token ให้ user ทุกคนที่ยังไม่มี Token"

    def handle(self, *args, **options):
        users_without_token = User.objects.filter(auth_token__isnull=True)
        created_count = 0
        for user in users_without_token.iterator(chunk_size=500):
            Token.objects.create(user=user)
            created_count += 1

        self.stdout.write(self.style.SUCCESS(f"สร้าง Token ให้ {created_count} user แล้ว"))
```

`auth_token` คือ **related_name** ของ `OneToOneField` จาก `Token` ไปยัง user (กำหนด
ไว้ใน source ของ authtoken app เอง) จึงกรองด้วย `auth_token__isnull=True` ได้ทันที

### 445.7 ดู/จัดการ Token ผ่าน Django Admin

`rest_framework.authtoken` ลงทะเบียน `TokenAdmin` ให้อัตโนมัติเมื่อเพิ่มเข้า
`INSTALLED_APPS` — เข้า `/admin/authtoken/tokenproxy/` จะเห็นรายการ Token ทั้งหมด
พร้อมปุ่มลบ (revoke) ได้ทันทีโดยไม่ต้องเขียนโค้ดเพิ่ม เหมาะสำหรับกรณีฉุกเฉิน เช่น
บัญชี user รั่วไหล ต้องการ revoke token ทันทีโดยไม่บังคับเปลี่ยน password

### 445.8 ตารางสรุปเปรียบเทียบ Authentication ทั้งสามตัวที่เรียนมา

| คุณสมบัติ | `SessionAuthentication` | `BasicAuthentication` | `TokenAuthentication` |
|---|---|---|---|
| Stateless | ❌ | ✅ | ✅ |
| ต้องส่ง password ทุก request | ❌ | ✅ | ❌ (ส่งครั้งเดียวตอนแลก token) |
| Revoke ได้โดยไม่เปลี่ยน password | ✅ (logout ทำลาย session) | ❌ | ✅ (ลบแถว Token) |
| รองรับ multi-device พร้อมกัน | ✅ (คนละ session) | ✅ (ส่ง password ทุกครั้งอยู่แล้ว) | ❌ (1 user = 1 token) |
| CSRF เกี่ยวข้อง | ✅ | ❌ | ❌ |
| เหมาะกับ | Browsable API, SPA same-origin | Testing/Internal tool | Mobile app, SPA แยก origin แบบง่าย |

---

## ขั้นตอนที่ 446: ผสมหลาย Authentication Class พร้อมกัน (ลำดับความสำคัญ)

### 446.1 `DEFAULT_AUTHENTICATION_CLASSES` เป็นลิสต์ที่ไล่ลองทีละตัว

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
        "rest_framework.authentication.BasicAuthentication",
    ],
}
```

ต่างจาก `permission_classes` ที่ต้อง "ผ่านทุกตัว" (AND) — `authentication_classes`
ทำงานแบบ **"ลองไปเรื่อย ๆ จนกว่าจะเจอตัวที่ authenticate สำเร็จ"** (คล้าย OR แต่มี
ลำดับความสำคัญชัดเจน):

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView.perform_authentication
def _authenticate(self):
    for authenticator in self.authenticators:
        try:
            user_auth_tuple = authenticator.authenticate(self.request)
        except exceptions.APIException:
            self._not_authenticated()
            raise

        if user_auth_tuple is not None:
            self._authenticator = authenticator
            self.user, self.auth = user_auth_tuple
            return

    self._not_authenticated()   # ไม่มีตัวไหนสำเร็จเลย → AnonymousUser
```

**กฎการไล่ลำดับ**:

1. ไล่ authenticator ตามลำดับที่ประกาศใน list **จากบนลงล่าง**
2. ตัวไหนคืน `(user, auth)` (ไม่ใช่ `None`) → **หยุดทันที** ใช้ผลลัพธ์นั้นเลย ไม่ลอง
   ตัวถัดไปอีก
3. ตัวไหน `raise AuthenticationFailed` (มี credential แต่ผิด) → **หยุดทันทีเช่นกัน**
   และปฏิเสธ request ไปเลย (**ไม่ใช่** ลองตัวถัดไปต่อ!) — นี่คือจุดที่มือใหม่เข้าใจผิด
   บ่อยที่สุด
4. ถ้าทุกตัวคืน `None` หมด (ไม่มี credential รูปแบบไหนเลย) → `request.user` กลายเป็น
   `AnonymousUser`

### 446.2 ตัวอย่างที่แสดงผลต่างจากลำดับ

สมมติตั้งค่าตามขั้นตอนที่ 446.1 (`Session` → `Token` → `Basic`) แล้ว client ส่ง
request ที่แนบ**ทั้ง** cookie `sessionid` ที่หมดอายุแล้ว **และ** header
`Authorization: Token <valid-key>` มาพร้อมกัน:

1. `SessionAuthentication.authenticate()` → session หมดอายุ → `request._request.user`
   เป็น `AnonymousUser` (Django กำหนดไว้แบบนี้เอง ไม่ raise exception) → เช็ค
   `if not user or not user.is_active` → คืน `None` (ไม่ raise เพราะ session ที่ไม่มี/
   หมดอายุไม่ถือเป็น "credential ที่ผิด" แต่เป็น "ไม่มี credential")
2. `TokenAuthentication.authenticate()` → เจอ header `Authorization: Token ...` →
   token ถูกต้อง → คืน `(user, token)` → **หยุดที่นี่** ใช้ผลลัพธ์จาก Token

ผลลัพธ์: request ผ่านด้วย `TokenAuthentication` แม้ session จะหมดอายุไปแล้ว เพราะ
`SessionAuthentication` เลือก "ยอมแพ้เงียบ ๆ" แทนที่จะ raise error เมื่อไม่มี session

### 446.3 `authenticate_header()` — ตัวตัดสินว่าจะได้ 401 หรือ 403

ทบทวนจากขั้นตอนที่ 441.6 ที่ทิ้งคำถามไว้ว่า DRF ตัดสิน 401 กับ 403 อย่างไร คำตอบอยู่ที่
method `authenticate_header()`:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.views.APIView.permission_denied
def permission_denied(self, request, message=None, code=None):
    if request.authenticators and not request.successful_authenticator:
        authenticate_header = request.authenticators[0].authenticate_header(request)
        if authenticate_header:
            raise exceptions.NotAuthenticated()   # → 401
    raise exceptions.PermissionDenied(detail=message, code=code)   # → 403
```

DRF จะดู **authenticator ตัวแรกในลิสต์** (ไม่ใช่ตัวที่ authenticate สำเร็จ เพราะไม่มี
ตัวไหนสำเร็จเลยในกรณีนี้) ว่ามี `authenticate_header()` คืนค่าที่ไม่ใช่ `None` หรือไม่:

| Authenticator ตัวแรกในลิสต์ | `authenticate_header()` คืนอะไร | ผลลัพธ์เมื่อไม่มี credential |
|---|---|---|
| `BasicAuthentication` | `'Basic realm="api"'` | `401 Unauthorized` + header `WWW-Authenticate: Basic realm="api"` |
| `TokenAuthentication` | `'Token'` | `401 Unauthorized` + header `WWW-Authenticate: Token` |
| `SessionAuthentication` | `None` (ไม่ override เลย) | `403 Forbidden` (ไม่มี header `WWW-Authenticate`) |

**ผลกระทบเชิงปฏิบัติ**: ถ้าตั้ง `DEFAULT_AUTHENTICATION_CLASSES` โดยมี
`SessionAuthentication` เป็นตัวแรก endpoint ที่ต้อง login ทั้งหมดจะตอบ `403` แทนที่จะ
เป็น `401` เมื่อไม่ได้ login เลย ซึ่งอาจทำให้ client บางตัว (โดยเฉพาะ browser ที่คาดหวัง
`401` เพื่อ pop-up prompt ใส่ username/password ของ `Basic Auth`) ทำงานผิดจากที่ตั้งใจ
— **คำแนะนำ**: ถ้าต้องการ `401` ที่ถูกต้องตามความหมาย ให้จัดลำดับให้ authenticator ที่มี
`authenticate_header()` (เช่น `TokenAuthentication`/`BasicAuthentication`) อยู่**ก่อน**
`SessionAuthentication` ในลิสต์ หรือยอมรับ `403` ถ้า API เป้าหมายเป็น browser-only

### 446.4 ตารางสรุปกฎการผสม Authentication Class

| กฎ | รายละเอียด |
|---|---|
| ลำดับมีผลจริง | ไล่จากบนลงล่าง ตัวแรกที่สำเร็จ "ชนะ" ทันที |
| `None` = เงียบ ลองต่อ | ไม่มี credential รูปแบบนั้น ไม่ถือเป็น error |
| `AuthenticationFailed` = หยุดทันที | มี credential แต่ผิด ไม่ลองตัวอื่นต่อแม้จะมีตัวถัดไปที่อาจสำเร็จ |
| `request.successful_authenticator` | เก็บ authenticator ตัวที่ใช้จริง (ขั้นตอนที่ 444.5) |
| `authenticate_header()` ของตัวแรก | กำหนดว่า "ไม่ได้ login เลย" จะได้ 401 หรือ 403 |

---

## ขั้นตอนที่ 447: เชื่อม `DjangoObjectPermissions`/django-guardian เข้ากับ DRF ViewSet

### 447.1 ทบทวนปัญหาจาก Part 038 ในบริบทของ DRF

Part 038 พิสูจน์ว่า Django permission มาตรฐานเป็นระดับ **model** ไม่ใช่ระดับ **object**
แล้วแก้ด้วย django-guardian ผ่าน `ObjectPermissionBackend` DRF มี Permission class
ชื่อ `DjangoObjectPermissions` ที่ออกแบบมา**เพื่อเชื่อมกับ backend แบบ guardian
โดยเฉพาะ** — เป็นสะพานเชื่อมที่สมบูรณ์ที่สุดระหว่าง Part 038 กับ Phase 5

### 447.2 `DjangoModelPermissions` ก่อน — พื้นฐานที่ `DjangoObjectPermissions` ต่อยอดมา

ก่อนเข้า Object-level ต้องเข้าใจ `DjangoModelPermissions` (permission ระดับ model ปกติ)
ที่แมป HTTP method กับ permission codename ของ Django auth (Part 033) อัตโนมัติ:

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.permissions.DjangoModelPermissions
class DjangoModelPermissions(BasePermission):
    perms_map = {
        "GET": [],
        "OPTIONS": [],
        "HEAD": [],
        "POST": ["%(app_label)s.add_%(model_name)s"],
        "PUT": ["%(app_label)s.change_%(model_name)s"],
        "PATCH": ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }

    def has_permission(self, request, view):
        queryset = view.get_queryset()   # ต้องมี queryset attribute เสมอ ไม่งั้น error
        perms = self.get_required_permissions(request.method, queryset.model)
        return request.user and request.user.has_perms(perms)
```

สังเกตว่า `GET`/`HEAD`/`OPTIONS` ไม่ต้องการ permission ใด ๆ เลย (list ว่าง) — หมายความ
ว่า `DjangoModelPermissions` **ไม่ป้องกันการอ่านข้อมูล** โดยตรง (ต้องผสมกับ
`IsAuthenticated` เองถ้าต้องการบล็อกการอ่านด้วย) และมันเรียก `has_perm()` แบบไม่ส่ง
`obj` (ทบทวนจาก Part 033/038) จึงยังเป็นแค่ระดับ model เท่านั้น

### 447.3 `DjangoObjectPermissions` — เพิ่ม Object-level เข้ามา

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.permissions.DjangoObjectPermissions
class DjangoObjectPermissions(DjangoModelPermissions):
    perms_map = {
        "GET": [],
        "OPTIONS": [],
        "HEAD": [],
        "POST": ["%(app_label)s.add_%(model_name)s"],
        "PUT": ["%(app_label)s.change_%(model_name)s"],
        "PATCH": ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }

    def has_object_permission(self, request, view, obj):
        queryset = view.get_queryset()
        model_cls = queryset.model
        user = request.user

        perms = self.get_required_object_permissions(request.method, model_cls)

        if not user.has_perms(perms, obj):
            # ถ้าไม่มีสิทธิ์เลยแม้แต่ view (GET) → ยิง 404 แทน 403 เพื่อไม่เปิดเผยว่า
            # object นี้มีอยู่จริง (ทบทวนแนวคิดนี้จาก Part 038 ขั้นตอนที่ 376.2)
            if request.method in SAFE_METHODS:
                raise Http404
            read_perms = self.get_required_object_permissions("GET", model_cls)
            if not user.has_perms(read_perms, obj):
                raise Http404
            return False   # เห็นได้ แต่แก้ไข/ลบไม่ได้ → 403 ตามปกติ
        return True
```

จุดที่ทรงพลังที่สุด: `user.has_perms(perms, obj)` เรียก `has_perm()` **พร้อมส่ง `obj`**
เข้าไปตรง ๆ — และนี่คือจุดที่ `AUTHENTICATION_BACKENDS` จาก Part 038 (ที่มี
`guardian.backends.ObjectPermissionBackend` อยู่) เข้ามาทำงานทันทีโดย DRF **ไม่ต้อง
รู้จัก guardian เลยแม้แต่น้อย** — `DjangoObjectPermissions` แค่เรียก Django permission
API มาตรฐาน (`user.has_perms()`) แล้ว Django เองที่ไล่เช็คทุก backend ตามลำดับให้
(กลไกเดียวกับขั้นตอนที่ 374.1 ของ Part 038 ทุกประการ)

### 447.4 ประกอบร่างทั้งหมดเข้ากับ `PostViewSet`

```python
# blog/api_views.py
from rest_framework import viewsets
from rest_framework.permissions import DjangoObjectPermissions
from guardian.shortcuts import assign_perm

from .models import Post
from .serializers import PostSerializer


class PostObjectPermissions(DjangoObjectPermissions):
    """
    ขยาย DjangoObjectPermissions ให้บังคับ permission กับ GET ด้วย (ค่า default ของ
    DRF ปล่อย GET ผ่านฟรีเสมอ) เพื่อให้ "เห็นเฉพาะโพสต์ที่มีสิทธิ์ view_post" จริง ๆ
    """
    perms_map = {
        "GET": ["%(app_label)s.view_%(model_name)s"],
        "OPTIONS": [],
        "HEAD": ["%(app_label)s.view_%(model_name)s"],
        "POST": ["%(app_label)s.add_%(model_name)s"],
        "PUT": ["%(app_label)s.change_%(model_name)s"],
        "PATCH": ["%(app_label)s.change_%(model_name)s"],
        "DELETE": ["%(app_label)s.delete_%(model_name)s"],
    }


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer
    permission_classes = [PostObjectPermissions]

    def perform_create(self, serializer):
        post = serializer.save(author=self.request.user)
        # มอบ object-level permission ให้เจ้าของทันที (แนวคิดเดียวกับ Part 038 ขั้นตอนที่ 373.4)
        assign_perm("view_post", self.request.user, post)
        assign_perm("change_post", self.request.user, post)
        assign_perm("delete_post", self.request.user, post)
```

**สำคัญ**: ต้องมี `queryset` attribute เสมอ (ไม่ใช่แค่ `get_queryset()` override เฉย ๆ)
เพราะ `DjangoModelPermissions.has_permission()` เรียก `view.get_queryset()` เพื่อหา
`model_name`/`app_label` — ถ้า View ไม่มี `queryset` เลย DRF จะ raise
`AssertionError` ทันทีตอน request เข้ามา (fail-fast โดยตั้งใจ เพื่อไม่ให้ permission
class ทำงานผิดพลาดแบบเงียบ ๆ)

### 447.5 ข้อจำกัดสำคัญ: Object-level Permission ไม่กรอง List Endpoint ให้อัตโนมัติ

ทบทวนจากขั้นตอนที่ 442.6 อีกครั้ง: `DjangoObjectPermissions.has_object_permission()`
ถูกเรียก**เฉพาะตอน retrieve/update/destroy object เดี่ยว** เท่านั้น — `GET
/api/posts/` (list) **ไม่ผ่านการเช็คนี้เลย** แม้จะตั้ง `perms_map["GET"]` ไว้แล้วก็ตาม
(นั่นมีผลแค่กับ `has_permission()` ระดับ view ว่า "user มี view_post permission ระดับ
model บ้างไหม" ไม่ได้กรองทีละแถว)

ทางแก้ที่ถูกต้องคือกรอง `get_queryset()` ด้วย `get_objects_for_user()` จาก guardian
โดยตรง (ทบทวนจาก Part 038 ขั้นตอนที่ 374.4):

```python
# blog/api_views.py
from guardian.shortcuts import get_objects_for_user


class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    permission_classes = [PostObjectPermissions]

    def get_queryset(self):
        # แสดงเฉพาะโพสต์ที่ user ปัจจุบันมีสิทธิ์ view_post จริง (ทั้งจาก ownership
        # และจากสิทธิ์ที่ถูกมอบเพิ่มเติมภายหลัง)
        user = self.request.user
        if not user.is_authenticated:
            return Post.objects.none()
        return get_objects_for_user(
            user, "blog.view_post", klass=Post
        ).select_related("author", "category")
```

### 447.6 ตารางเปรียบเทียบ 2 แนวทางสำหรับ DRF: Manual `IsOwnerOrReadOnly` vs `DjangoObjectPermissions` + guardian

| แนวทาง | Field ที่ต้องมี | รองรับ Delegated Permission | ต้อง query permission table เพิ่ม | ความซับซ้อนในการตั้งค่า |
|---|---|---|---|---|
| `IsOwnerOrReadOnly` (ขั้นตอนที่ 442) — เทียบ `obj.author_id` ตรง ๆ | ต้องมี field เจ้าของชัดเจน (`author`) | ❌ ไม่รองรับ | ❌ ไม่ต้อง | ต่ำมาก ไม่ต้องติดตั้งอะไรเพิ่ม |
| `DjangoObjectPermissions` + django-guardian (ขั้นตอนนี้) | ไม่จำเป็นต้องมี field เจ้าของเลย | ✅ รองรับเต็มรูปแบบ (มอบสิทธิ์ให้ใครก็ได้ทีหลัง) | ✅ ต้อง query ตาราง `guardian_userobjectpermission` | ปานกลาง ต้องมอบสิทธิ์ตอนสร้าง object เสมอ |

**คำแนะนำของหลักสูตรนี้**: ใช้ `IsOwnerOrReadOnly` เป็นค่าเริ่มต้นสำหรับกรณีง่าย ๆ ที่
กฎคือ "เจ้าของเท่านั้น" ตรงไปตรงมา และสลับมาใช้ `DjangoObjectPermissions` +
django-guardian เมื่อระบบต้องการมอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของ (เช่น บรรณาธิการ
ช่วยแก้ไขชั่วคราว) แบบเดียวกับคำแนะนำที่ Part 038 ให้ไว้ในบริบทของ Django View ธรรมดา
— หลักการเดียวกันทุกประการ เพียงแค่ย้ายมาอยู่ในโลกของ DRF

---

## ขั้นตอนที่ 448: การเขียน Test สำหรับ Permission Logic ของ API

### 448.1 เครื่องมือหลัก: `APIClient` และ `force_authenticate()`

DRF มี `rest_framework.test.APIClient` ที่สืบทอดจาก `django.test.Client` (Part 025-026)
แต่เพิ่มความสามารถเฉพาะสำหรับ API เช่น `force_authenticate()` ที่ข้ามขั้นตอน
authentication จริงไปเลย (เร็วกว่าและไม่ต้องแลก token ทุกครั้งตอนเทส):

```python
# blog/tests/test_permissions.py
from django.contrib.auth import get_user_model
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Category, Post

User = get_user_model()


class PostPermissionTests(APITestCase):
    def setUp(self):
        self.owner = User.objects.create_user(username="narin", password="pass12345")
        self.other_user = User.objects.create_user(username="kai", password="pass12345")
        self.staff_user = User.objects.create_user(
            username="admin_editor", password="pass12345", is_staff=True
        )
        self.category = Category.objects.create(name="Django", slug="django")
        self.post = Post.objects.create(
            title="โพสต์ของ narin",
            content="เนื้อหาทดสอบ" * 10,
            author=self.owner,
            category=self.category,
            is_published=True,
        )
        self.detail_url = f"/api/posts/{self.post.pk}/"
        self.list_url = "/api/posts/"
```

### 448.2 Test Matrix: Anonymous / Other User / Owner / Staff

```python
    def test_anonymous_can_read_but_not_write(self):
        response = self.client.get(self.detail_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

        response = self.client.patch(self.detail_url, {"title": "แก้ไข"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_other_authenticated_user_cannot_edit(self):
        self.client.force_authenticate(user=self.other_user)
        response = self.client.patch(self.detail_url, {"title": "แก้ไขโดยคนอื่น"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

        # ตรวจให้แน่ใจว่าข้อมูลจริงไม่ได้ถูกแก้ (ป้องกัน false positive จาก assertion ผิด endpoint)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, "โพสต์ของ narin")

    def test_owner_can_edit_own_post(self):
        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(self.detail_url, {"title": "แก้ไขโดยเจ้าของ"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, "แก้ไขโดยเจ้าของ")

    def test_authenticated_user_can_create_post(self):
        self.client.force_authenticate(user=self.other_user)
        payload = {
            "title": "โพสต์ใหม่ของ kai",
            "content": "เนื้อหา" * 10,
            "category": self.category.pk,
            "is_published": False,
        }
        response = self.client.post(self.list_url, payload, format="json")
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        # author ต้องถูกตั้งค่าเป็น request.user เสมอ ไม่ว่า client จะส่ง author มาหรือไม่
        created_post = Post.objects.get(pk=response.data["id"])
        self.assertEqual(created_post.author, self.other_user)
```

### 448.3 ทดสอบว่า Client ไม่สามารถปลอมตัวเป็น Author คนอื่นได้

จุดที่ทีมความปลอดภัยตรวจสอบเป็นอันดับต้น ๆ: ต้องยืนยันว่าต่อให้ client แนบ `author`
มาผิดคนใน payload เอง ระบบก็ต้อง**เพิกเฉยและใช้ `request.user` เสมอ**
(ทบทวนจาก `perform_create()` ในขั้นตอนที่ 442.4):

```python
    def test_client_cannot_spoof_author_field(self):
        self.client.force_authenticate(user=self.other_user)
        payload = {
            "title": "พยายามปลอมเป็นคนอื่น",
            "content": "เนื้อหา" * 10,
            "category": self.category.pk,
            "author": self.owner.pk,   # พยายามระบุ author เป็น narin ทั้งที่ login เป็น kai
        }
        response = self.client.post(self.list_url, payload, format="json")
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

        created_post = Post.objects.get(pk=response.data["id"])
        # ต้องเป็น kai (request.user จริง) ไม่ใช่ narin ที่พยายามปลอมมาใน payload
        self.assertEqual(created_post.author, self.other_user)
```

ถ้า `PostSerializer` ประกาศ `author` เป็น writable field แบบไม่ระวัง (เช่น
`fields = "__all__"` โดยไม่กำหนด `read_only_fields`) test นี้จะ **fail ทันที** เพราะ
`serializer.save(author=self.request.user)` ใน `perform_create()` จะถูก validated_data
ที่มี `author` จาก client override ทับ — นี่คือเหตุผลที่ Part 041 (ModelSerializer) ต้อง
กำหนด `read_only_fields = ["author"]` เสมอสำหรับ field ที่ server ต้องเป็นคนตั้งค่าเอง

### 448.4 ทดสอบ `IsOwnerOrReadOnly` แบบแยกหน่วย (Unit Test) โดยไม่ต้องผ่าน HTTP

บางครั้งอยากทดสอบ Permission class เดี่ยว ๆ โดยตรงโดยไม่ต้องยิง request ผ่าน HTTP
เต็มรูปแบบ (เร็วกว่าและ isolate ปัญหาได้ชัดกว่า) ใช้ `APIRequestFactory` ร่วมกับการ
เรียก method ตรง ๆ:

```python
# blog/tests/test_permissions.py (ต่อจากเดิม)
from unittest.mock import Mock

from rest_framework.test import APIRequestFactory

from blog.permissions import IsOwnerOrReadOnly


class IsOwnerOrReadOnlyUnitTests(APITestCase):
    def setUp(self):
        self.factory = APIRequestFactory()
        self.permission = IsOwnerOrReadOnly()
        self.owner = User.objects.create_user(username="narin", password="pass12345")
        self.other_user = User.objects.create_user(username="kai", password="pass12345")
        self.post = Post.objects.create(
            title="ทดสอบ", content="x" * 60, author=self.owner,
        )

    def test_safe_method_always_allowed(self):
        request = self.factory.get("/api/posts/1/")
        request.user = self.other_user
        self.assertTrue(
            self.permission.has_object_permission(request, Mock(), self.post)
        )

    def test_unsafe_method_denied_for_non_owner(self):
        request = self.factory.patch("/api/posts/1/")
        request.user = self.other_user
        self.assertFalse(
            self.permission.has_object_permission(request, Mock(), self.post)
        )

    def test_unsafe_method_allowed_for_owner(self):
        request = self.factory.patch("/api/posts/1/")
        request.user = self.owner
        self.assertTrue(
            self.permission.has_object_permission(request, Mock(), self.post)
        )
```

`Mock()` ใช้แทน `view` argument ที่สองของ `has_object_permission()` เพราะ
`IsOwnerOrReadOnly` เวอร์ชันนี้ไม่ได้ใช้ `view` เลยในการตัดสินใจ — เทคนิคนี้ทำให้ทดสอบ
Permission logic ได้เร็วมาก (ไม่ต้อง setup URL routing/serializer ใด ๆ) เหมาะกับการ
ทดสอบกฎที่ซับซ้อนหลาย edge case โดยไม่ต้องยิง HTTP request จริง

### 448.5 ทดสอบ `DjangoObjectPermissions` + guardian ร่วมกัน

```python
# blog/tests/test_object_permissions_api.py
from django.contrib.auth import get_user_model
from guardian.shortcuts import assign_perm
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Post

User = get_user_model()


class DjangoObjectPermissionsAPITests(APITestCase):
    def setUp(self):
        self.owner = User.objects.create_user(username="narin", password="pass12345")
        self.editor = User.objects.create_user(username="editor_b", password="pass12345")
        self.post = Post.objects.create(
            title="โพสต์ของ narin", content="x" * 60, author=self.owner,
        )
        assign_perm("view_post", self.owner, self.post)
        assign_perm("change_post", self.owner, self.post)
        self.detail_url = f"/api/posts/{self.post.pk}/"

    def test_user_without_object_permission_gets_404_not_403(self):
        """
        ทบทวนจาก Part 038 (376.2): DjangoObjectPermissions คืน 404 แทน 403 เมื่อ
        user ไม่มีสิทธิ์ view เลย เพื่อไม่เปิดเผยการมีอยู่ของ object
        """
        self.client.force_authenticate(user=self.editor)
        response = self.client.get(self.detail_url)
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)

    def test_delegated_permission_grants_access(self):
        """มอบสิทธิ์เพิ่มเติมให้ editor_b ที่ไม่ใช่เจ้าของ แล้วต้องแก้ไขได้"""
        assign_perm("view_post", self.editor, self.post)
        assign_perm("change_post", self.editor, self.post)

        self.client.force_authenticate(user=self.editor)
        response = self.client.patch(
            self.detail_url, {"title": "แก้ไขโดย editor ที่ได้รับมอบหมาย"}, format="json"
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

### 448.6 ตารางสรุปกลยุทธ์การทดสอบ Permission

| ระดับ | เครื่องมือ | ทดสอบอะไร | ความเร็ว |
|---|---|---|---|
| Unit test permission class ล้วน ๆ | `APIRequestFactory` + เรียก method ตรง | Logic ภายใน `has_permission()`/`has_object_permission()` แยกจาก HTTP stack | เร็วที่สุด |
| Integration test ผ่าน `APIClient` | `APITestCase` + `force_authenticate()` | พฤติกรรมจริงของ endpoint ทั้ง URL routing, serializer, permission รวมกัน | ปานกลาง |
| End-to-end ผ่าน token จริง | `APIClient` + header `Authorization: Token ...` | ทั้ง authentication + permission แบบเดียวกับ client จริงเป๊ะ | ช้าที่สุด (ต้องสร้าง token ก่อน) |

---

## ขั้นตอนที่ 449: ข้อควรระวังเรื่อง CSRF เมื่อใช้ `SessionAuthentication` กับ API

### 449.1 ย้อนไปที่ความขัดแย้งที่ Part 042 ทิ้งไว้

Part 042 (ขั้นตอนที่ 412.3) บอกไว้ว่า `APIView.as_view()` ครอบผลลัพธ์ด้วย
`csrf_exempt()` **เสมอ** — แปลว่าทุก `APIView`/`ViewSet` ใน DRF **ปิด CSRF ป้องกันไว้ที่
ระดับ Django middleware ตั้งแต่ต้น** เหตุผลคือ DRF ต้องรองรับ client ที่ไม่มี CSRF
token เลย (mobile app, token-based client จากขั้นตอนที่ 445) แต่คำถามคือ: **แล้ว
Browsable API ที่ login ผ่าน session (ขั้นตอนที่ 444.2) ปลอดภัยจาก CSRF จริงหรือ?**

คำตอบ: **ปลอดภัย** เพราะ `SessionAuthentication` **เพิ่ม CSRF check กลับเข้ามาเอง**
โดยไม่พึ่ง Django middleware เลย — ทบทวนจากซอร์สโค้ดในขั้นตอนที่ 444.2 อีกครั้ง:

```python
class SessionAuthentication(BaseAuthentication):
    def authenticate(self, request):
        user = getattr(request._request, "user", None)
        if not user or not user.is_active:
            return None
        self.enforce_csrf(request)   # <-- บรรทัดนี้
        return (user, None)

    def enforce_csrf(self, request):
        check = CSRFCheck(lambda r: None)
        check.process_request(request)
        reason = check.process_view(request, None, (), {})
        if reason:
            raise exceptions.PermissionDenied(f"CSRF Failed: {reason}")
```

### 449.2 สรุป Layer การป้องกัน CSRF สองชั้นที่แยกกันทำงาน

```
                     ┌─────────────────────────────────────────┐
                     │  Django CsrfViewMiddleware (Part 007/008) │
                     │  ทำงานกับ FBV/CBV ธรรมดาที่ไม่ผ่าน DRF     │
                     └─────────────────────────────────────────┘
                                       │
                    APIView.as_view() │ ครอบด้วย @csrf_exempt เสมอ
                                       │ (ปิด middleware ข้างบนสำหรับ DRF view ทุกตัว)
                                       ▼
                     ┌─────────────────────────────────────────┐
                     │   SessionAuthentication.enforce_csrf()    │
                     │   เช็ค CSRF token ใหม่เอง "เฉพาะ" ตอนที่   │
                     │   authenticate ผ่าน session คนเดียวเท่านั้น │
                     └─────────────────────────────────────────┘
```

**ผลลัพธ์ที่ต้องจำให้แม่น**: CSRF check ของ DRF **ผูกกับ Authentication class ที่ใช้
จริง ไม่ใช่ผูกกับ URL หรือ View** — request เดียวกันไปยัง endpoint เดียวกัน อาจโดนหรือ
ไม่โดน CSRF check เลยก็ได้ ขึ้นอยู่กับว่า authenticate ผ่านทางไหน:

| Authenticate ผ่านทางไหน | ต้องแนบ CSRF token ไหม |
|---|---|
| `SessionAuthentication` (cookie `sessionid`) | **ต้องแนบเสมอ** สำหรับ unsafe method |
| `TokenAuthentication` (header `Authorization: Token ...`) | ไม่ต้องเลย |
| `BasicAuthentication` | ไม่ต้องเลย |

### 449.3 ทำไม Session ถึงต้องมี CSRF แต่ Token ไม่ต้อง

หัวใจของ CSRF attack คือเบราว์เซอร์ **แนบ cookie ไปกับทุก request โดยอัตโนมัติ**
แม้ request นั้นจะถูกยิงมาจากเว็บไซต์อื่นที่ผู้ใช้ไม่ได้ตั้งใจ (เช่น เว็บร้ายฝัง
`<form>` ที่ submit ไปยัง `secureblog.com/api/posts/7/` โดยที่เหยื่อ login
`secureblog.com` ค้างอยู่พอดี) เบราว์เซอร์จะแนบ `sessionid` cookie ไปด้วยโดยไม่รู้ตัว
ทำให้ request ดูเหมือน "ถูกต้องตามกฎหมาย" ทั้งที่เจ้าของบัญชีไม่ได้สั่งเอง — CSRF token
แก้ปัญหานี้เพราะเว็บร้ายไม่มีทางรู้ค่า CSRF token ที่ถูกต้องได้ (มันไม่ได้อยู่ใน cookie
ที่แนบอัตโนมัติ แต่ต้องอ่านจาก DOM/cookie อีกตัวที่ same-origin policy ปิดกั้นเว็บอื่น
ไม่ให้อ่านได้)

`TokenAuthentication`/`BasicAuthentication` **ไม่มีความเสี่ยงนี้เลย** เพราะ credential
(token/password) **ไม่ได้ถูกแนบอัตโนมัติโดยเบราว์เซอร์** — client ต้อง set header
`Authorization` ด้วยโค้ด JavaScript ของตัวเองอย่างจงใจเท่านั้น เว็บร้ายไม่มีทางรู้ค่า
token ของเหยื่อได้เลย จึงไม่จำเป็นต้องมี CSRF token มาป้องกันซ้ำ

### 449.4 ตัวอย่างจริง: 403 CSRF Failed เมื่อทดสอบผ่าน Browsable API

```bash
# login ผ่านหน้า Browsable API ก่อน (ได้ cookie sessionid + csrftoken)
# แล้วพยายาม POST โดยไม่แนบ X-CSRFToken header

curl -i -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -b "sessionid=abc123...; csrftoken=xyz789..." \
  -d '{"title": "ทดสอบ CSRF", "content": "..."}'
```

```
HTTP/1.1 403 Forbidden
Content-Type: application/json

{"detail":"CSRF Failed: CSRF token missing."}
```

ต้องแนบ header `X-CSRFToken` ที่มีค่าตรงกับ cookie `csrftoken` เสมอ (กลไกเดียวกับที่
เรียนใน Part 008 สำหรับฟอร์ม HTML ธรรมดา เพียงแต่ส่งผ่าน header แทน hidden input):

```bash
curl -i -X POST http://127.0.0.1:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -H "X-CSRFToken: xyz789..." \
  -b "sessionid=abc123...; csrftoken=xyz789..." \
  -d '{"title": "ทดสอบ CSRF", "content": "..."}'
```

```
HTTP/1.1 201 Created
```

ฝั่ง JavaScript (SPA ที่ same-origin) ต้องอ่านค่า `csrftoken` จาก cookie แล้วแนบเข้า
header เองเสมอ:

```javascript
// static/js/api-client.js — ตัวอย่างการอ่าน CSRF token จาก cookie ฝั่ง client
function getCookie(name) {
    const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`));
    return match ? match[2] : null;
}

fetch("/api/posts/", {
    method: "POST",
    headers: {
        "Content-Type": "application/json",
        "X-CSRFToken": getCookie("csrftoken"),
    },
    credentials: "same-origin",   // ต้องมี เพื่อให้แนบ cookie sessionid ไปด้วย
    body: JSON.stringify({title: "โพสต์ใหม่", content: "..."}),
});
```

### 449.5 ความเสี่ยงที่เหลืออยู่แม้มี CSRF Check แล้ว — ทำไมมือใหม่ยังพลาดได้

1. **ปิด `SessionAuthentication` "เผลอ"**: ถ้าใครลบ `SessionAuthentication` ออกจาก
   `DEFAULT_AUTHENTICATION_CLASSES` เพื่อความสะดวก (เช่น ตอน debug) แล้วลืมใส่กลับ
   endpoint นั้นจะไม่มี CSRF check เลย แต่ยังคง authenticate ผ่าน session ได้ถ้ามีตัวอื่น
   ที่ fallback ไปอ่าน `request.user` แบบไม่เช็ค CSRF (พบได้ถ้าเขียน custom authenticator
   เองแบบไม่ระวัง) — บทเรียน: **อย่าเขียน custom Authentication class ที่อ่าน session
   โดยไม่เรียก `enforce_csrf()` เอง**
2. **Cookie `SameSite` ไม่ได้ตั้งไว้**: Django 5.x ตั้ง `CSRF_COOKIE_SAMESITE = "Lax"`
   เป็นค่า default อยู่แล้ว (ป้องกัน CSRF ชั้นเพิ่มเติมจาก browser เอง) แต่ถ้ามีคนตั้ง
   `SameSite=None` เพื่อแก้ปัญหา cross-origin โดยไม่เข้าใจความเสี่ยง จะเปิดช่องโหว่กลับมา
3. **Token ที่รั่วไหลไม่ต้องใช้ CSRF เลย**: `TokenAuthentication` ไม่มี CSRF risk แต่มี
   ความเสี่ยงอื่นแทน (token ที่ขโมยได้ใช้งานได้ตลอดอายุจนกว่าจะ revoke) — CSRF กับ
   token leakage เป็นภัยคุกคามคนละแบบ ป้องกันด้วยวิธีต่างกันโดยสิ้นเชิง อย่าเข้าใจผิดว่า
   "ใช้ Token แล้วปลอดภัยกว่า Session เสมอ" — มันแค่ปลอดภัยจาก**คนละภัยคุกคาม**

### 449.6 ตารางสรุปคำแนะนำเรื่อง CSRF

| สถานการณ์ | คำแนะนำ |
|---|---|
| SPA/JS ที่ same-origin กับ Django (เช่น React ที่ build แล้ว serve จาก Django เอง) | ใช้ `SessionAuthentication` + ส่ง `X-CSRFToken` เสมอสำหรับ unsafe method |
| Mobile app (iOS/Android) | ใช้ `TokenAuthentication`/JWT (Part 046) เท่านั้น — ไม่มี cookie jar ให้ CSRF โจมตี |
| SPA ที่ cross-origin (frontend คนละ domain กับ backend) | ใช้ `TokenAuthentication`/JWT แทน session เพราะ cookie cross-origin มีข้อจำกัดเรื่อง `SameSite`/CORS อยู่แล้ว |
| Internal tool/script สำหรับทีม dev | `BasicAuthentication` หรือ `TokenAuthentication` ผ่าน HTTPS เท่านั้น |
| Browsable API สำหรับ debug ระหว่างพัฒนา | ปล่อย `SessionAuthentication` ไว้ตามค่า default พร้อม CSRF ป้องกันตามปกติ |

---

## ขั้นตอนที่ 450: สรุปและแบบฝึกหัด — ระบบ Permission เต็มรูปแบบสำหรับ Blog API

### 450.1 ประกอบร่างทุกอย่างที่เรียนมาเข้าด้วยกัน

ตอนนี้มาสร้างระบบ permission ที่สมบูรณ์ตามโจทย์จริง: **"เจ้าของแก้ไข/ลบได้เฉพาะของ
ตัวเอง staff แก้ไข/ลบได้ทั้งหมด คนทั่วไปอ่านได้อย่างเดียว"**

```python
# blog/permissions.py
from rest_framework.permissions import SAFE_METHODS, BasePermission


class IsOwnerOrStaff(BasePermission):
    """
    - Safe method (GET/HEAD/OPTIONS): ทุกคนอ่านได้เสมอ
    - Unsafe method: staff แก้ไข/ลบได้ทุกโพสต์ / เจ้าของแก้ไข/ลบได้เฉพาะของตัวเอง
    - ต้องใช้คู่กับ permission ที่บังคับ authenticated มาก่อนเสมอ (ดูเหตุผลในขั้นตอนที่ 442.2)
    """

    message = "คุณต้องเป็นเจ้าของโพสต์นี้ หรือเป็นทีมงาน (staff) เท่านั้นจึงจะแก้ไข/ลบได้"

    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True

        if request.user.is_staff:
            return True

        return obj.author_id == request.user.id
```

```python
# blog/api_views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly

from .models import Post
from .permissions import IsOwnerOrStaff
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    """
    GET (list/retrieve): เปิดสาธารณะ ทุกคนอ่านได้
    POST (create): ต้อง login เท่านั้น author ถูกตั้งเป็น request.user เสมอ
    PUT/PATCH/DELETE: เจ้าของ หรือ staff เท่านั้น
    """

    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrStaff]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

```python
# blog/serializers.py
from rest_framework import serializers

from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source="author.username")

    class Meta:
        model = Post
        fields = [
            "id", "title", "slug", "content", "is_published",
            "category", "author", "created_at",
        ]
        read_only_fields = ["slug", "author", "created_at"]
```

```python
# config/settings.py — ค่าสุดท้ายของ REST_FRAMEWORK ที่ใช้ตลอด Phase 5 จากนี้ไป
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",
    ],
}
```

สังเกตว่าจัดลำดับ `TokenAuthentication` ไว้**ก่อน** `SessionAuthentication` (ทบทวนจาก
ขั้นตอนที่ 446.3) เพื่อให้ endpoint ที่เรียกโดย mobile app/token client ได้ `401` ที่
ถูกต้องตามความหมายเมื่อไม่ได้แนบ token เลย ในขณะที่ Browsable API ยังคง login ผ่าน
session ได้ตามปกติเพราะ DRF ไล่ลองทุก authenticator อยู่ดี (ทบทวนขั้นตอนที่ 446.1)

### 450.2 Test ครบชุดสำหรับระบบนี้

```python
# blog/tests/test_final_permission_system.py
from django.contrib.auth import get_user_model
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Post

User = get_user_model()


class FinalBlogPermissionSystemTests(APITestCase):
    def setUp(self):
        self.owner = User.objects.create_user(username="narin", password="pass12345")
        self.other_user = User.objects.create_user(username="kai", password="pass12345")
        self.staff = User.objects.create_user(
            username="editor", password="pass12345", is_staff=True
        )
        self.post = Post.objects.create(
            title="โพสต์ทดสอบ", content="x" * 60, author=self.owner, is_published=True,
        )
        self.url = f"/api/posts/{self.post.pk}/"

    def test_anyone_can_read(self):
        response = self.client.get(self.url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_anonymous_cannot_write(self):
        response = self.client.delete(self.url)
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_other_user_cannot_edit_or_delete(self):
        self.client.force_authenticate(user=self.other_user)
        self.assertEqual(
            self.client.patch(self.url, {"title": "x"}, format="json").status_code,
            status.HTTP_403_FORBIDDEN,
        )
        self.assertEqual(self.client.delete(self.url).status_code, status.HTTP_403_FORBIDDEN)

    def test_owner_can_edit_and_delete(self):
        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(self.url, {"title": "แก้ไขแล้ว"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_staff_can_edit_and_delete_others_post(self):
        self.client.force_authenticate(user=self.staff)
        response = self.client.patch(self.url, {"title": "แก้ไขโดย staff"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)

        response = self.client.delete(self.url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
```

```bash
python manage.py test blog.tests.test_final_permission_system
```

```
Ran 5 tests in 0.412s

OK
```

### 450.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างระหว่าง Authentication ("คุณเป็นใคร") กับ Permission
  ("คุณทำสิ่งนี้ได้ไหม") และวงจรที่ DRF รันทั้งสองอย่างใน `dispatch()`
- ✅ ใช้งาน Permission class มาตรฐาน 4 ตัว: `AllowAny`, `IsAuthenticated`,
  `IsAdminUser`, `IsAuthenticatedOrReadOnly`
- ✅ เขียน Custom Permission Class เอง (`IsOwnerOrReadOnly`, `IsOwnerOrStaff`) ด้วย
  `has_permission()` และ `has_object_permission()`
- ✅ ตั้งค่า `DEFAULT_PERMISSION_CLASSES` ทั้งโปรเจกต์ และ override เฉพาะ view ด้วย
  `permission_classes` หรือ `get_permissions()` แบบ dynamic ตาม `self.action`
- ✅ เปรียบเทียบ `SessionAuthentication` กับ `BasicAuthentication` ทั้งกลไกภายในและ
  ความเสี่ยง
- ✅ ติดตั้งและใช้งาน `TokenAuthentication` ผ่าน `obtain_auth_token` พร้อมสร้าง Token
  อัตโนมัติผ่าน signal
- ✅ เข้าใจลำดับความสำคัญของ `DEFAULT_AUTHENTICATION_CLASSES` และผลต่อ 401 กับ 403
- ✅ เชื่อม django-guardian จาก Part 038 เข้ากับ DRF ผ่าน `DjangoObjectPermissions`
  และ `get_objects_for_user()`
- ✅ เขียน Test ครอบคลุม Permission Logic ทั้งระดับ unit และ integration
- ✅ เข้าใจกลไก CSRF สองชั้นของ DRF และรู้ว่าเมื่อไหร่ต้องแนบ CSRF token เมื่อไหร่ไม่ต้อง
- ✅ สร้างระบบ permission เต็มรูปแบบสำหรับ blog API: owner/staff/anonymous

### 450.4 Checklist ก่อนไป Part ถัดไป

- [ ] ตั้งค่า `DEFAULT_PERMISSION_CLASSES = ["rest_framework.permissions.IsAuthenticatedOrReadOnly"]` ใน `REST_FRAMEWORK`
- [ ] เขียน `IsOwnerOrReadOnly`/`IsOwnerOrStaff` ใน `blog/permissions.py` และรันผ่านทุก test
- [ ] ติดตั้ง `rest_framework.authtoken`, รัน `migrate`, และสร้าง Token ให้ user ทดสอบสำเร็จ
- [ ] เรียก `/api/auth-token/` แลก token ได้จริงผ่าน `curl`
- [ ] ยิง request ที่แนบ `Authorization: Token <key>` เข้า endpoint ที่ต้อง login สำเร็จ
- [ ] ทดสอบว่า `PATCH`/`DELETE` จาก user ที่ไม่ใช่เจ้าของถูกปฏิเสธด้วย `403`
- [ ] ทดสอบว่า staff แก้ไข/ลบโพสต์ของคนอื่นได้
- [ ] ทดสอบว่า client ไม่สามารถปลอมค่า `author` ผ่าน payload ได้
- [ ] ยืนยันว่า `SessionAuthentication` ยิง `403 CSRF Failed` เมื่อไม่แนบ `X-CSRFToken`
- [ ] เชื่อม `DjangoObjectPermissions` เข้ากับ `PostViewSet` อย่างน้อยหนึ่งเวอร์ชัน และ
      ทดสอบว่า user ที่ไม่มีสิทธิ์ `view_post` ได้ `404` ไม่ใช่ `403`

### 450.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง `Comment` model ใหม่ (FK ไปยัง `Post` และ `author`) พร้อม
`CommentViewSet` ที่มีกฎ permission ดังนี้: ทุกคนอ่านคอมเมนต์ได้ เฉพาะ user ที่ login
แล้วเท่านั้นที่คอมเมนต์ได้ และเจ้าของคอมเมนต์**หรือ**เจ้าของโพสต์ต้นเรื่อง (ไม่ใช่แค่
เจ้าของคอมเมนต์อย่างเดียว) ลบคอมเมนต์นั้นได้ เขียน Custom Permission Class ที่รองรับ
กฎ "OR" แบบนี้ พร้อม test ครอบคลุมทั้ง 2 เงื่อนไข

**แบบฝึกหัดที่ 2**: เขียน endpoint `GET /api/me/permissions/` ที่คืนรายการ permission
ทั้งหมด (ทั้งระดับ model จาก `get_all_permissions()` และระดับ object จาก
`get_objects_for_user()` ของ guardian) ที่ user ปัจจุบันมีต่อโพสต์ทั้งหมดในระบบ ในรูป
แบบ JSON ที่ frontend นำไปแสดงปุ่ม "แก้ไข"/"ลบ" ได้โดยไม่ต้องยิง request ซ้ำทีละโพสต์

**แบบฝึกหัดที่ 3**: ทดลองสลับลำดับ `DEFAULT_AUTHENTICATION_CLASSES` ระหว่าง
`[SessionAuthentication, TokenAuthentication]` กับ
`[TokenAuthentication, SessionAuthentication]` แล้วสังเกต response header
`WWW-Authenticate` และ status code (401 หรือ 403) ที่ได้เมื่อยิง request แบบไม่แนบ
credential ใด ๆ เลยไปยัง endpoint ที่ต้อง `IsAuthenticated` บันทึกผลต่างที่สังเกตได้
ลงในไฟล์ `notes.md` พร้อมอธิบายว่าทำไมถึงต่างกัน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน Custom Authentication Class ของตัวเองชื่อ
`ExpiringTokenAuthentication` ที่สืบทอดจาก `TokenAuthentication` แต่เพิ่มการเช็คว่า
`token.created` เกิน 24 ชั่วโมงแล้วหรือยัง ถ้าเกิน ให้ raise `AuthenticationFailed`
พร้อมข้อความ "Token has expired" และลบ token เก่าทิ้งอัตโนมัติ (บังคับให้ client ต้อง
แลก token ใหม่) เขียน test ยืนยันทั้งกรณี token ที่ยังไม่หมดอายุและหมดอายุแล้ว

### 450.6 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่ใช้ `IsOwnerOrReadOnly` (ขั้นตอนที่ 442) ไปเลยทุกที่ ไม่ต้องพึ่ง
django-guardian เลย?**
A: `IsOwnerOrReadOnly` เพียงพอสำหรับกฎที่ตายตัวว่า "เจ้าของเท่านั้น" แต่ถ้าระบบต้องการ
มอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของในภายหลัง (เช่น บรรณาธิการช่วยแก้ไขชั่วคราว โดยไม่
เปลี่ยน field `author`) จะทำไม่ได้เลยด้วยแนวทางนี้ ต้องใช้ `DjangoObjectPermissions`
+ guardian แทน (ขั้นตอนที่ 447) หลักการเลือกเหมือนกับที่ Part 038 สรุปไว้ทุกประการ

**Q: ต้องใช้ `TokenAuthentication` หรือรอไปใช้ JWT ใน Part 046 เลยดี?**
A: `TokenAuthentication` เหมาะกับโปรเจกต์ขนาดเล็ก-กลางที่ไม่ต้องการ scale สูงมาก
เพราะติดตั้งง่ายและเข้าใจง่าย ส่วน JWT (Part 046) เหมาะกับระบบที่ต้องการ stateless
เต็มรูปแบบ (ไม่ query database ทุก request เพื่อตรวจ token), รองรับ multi-device
จริงจัง, และต้องทำงานร่วมกับ microservices หลายตัว หลักสูตรนี้แนะนำให้เข้าใจ
`TokenAuthentication` ให้แม่นก่อน เพราะแนวคิดพื้นฐาน (header `Authorization`,
stateless, revoke ได้) เหมือนกันทุกประการ เพียงแค่ JWT ซับซ้อนกว่าในเรื่อง
signature verification และ expiration

**Q: `has_object_permission()` เช็คช้าไหมถ้ามีโพสต์เป็นล้านแถว?**
A: `has_object_permission()` เช็คแค่ **object เดียว** ที่ `get_object()` ดึงมาแล้ว
เท่านั้น (ไม่ query ทั้งตาราง) จึงไม่มีปัญหา performance จากจำนวนแถวทั้งหมด แต่ถ้าใช้
`get_objects_for_user()` กรอง list endpoint (ขั้นตอนที่ 447.5) ร่วมกับ guardian
ต้องระวังเรื่อง query performance ที่ Part 038 (ขั้นตอนที่ 378) เจาะลึกไว้แล้ว
โดยเฉพาะเมื่อข้อมูลมีจำนวนมากและใช้ Generic Relation แบบ default

**Q: ถ้าอยากปิด Browsable API ทั้งหมดเพื่อความปลอดภัยใน production ทำอย่างไร?**
A: ลบ `rest_framework.renderers.BrowsableAPIRenderer` ออกจาก
`DEFAULT_RENDERER_CLASSES` เหลือแค่ `JSONRenderer` (ทบทวนจาก Part 039 ขั้นตอนที่
384.6) ซึ่งจะทำให้ทุก endpoint คืน JSON ดิบเสมอไม่ว่า client จะเป็นเบราว์เซอร์หรือไม่
เป็นแนวทางที่ทีม production จำนวนมากทำเพื่อลด attack surface และซ่อนรายละเอียด
implementation จาก client ที่ไม่ควรรู้

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 046: Authentication ขั้นสูง — JWT, Token, OAuth2** จะพา `TokenAuthentication`
ที่เรียนใน Part นี้ไปอีกขั้น ด้วย **JSON Web Token (JWT)** ผ่าน package
`djangorestframework-simplejwt` คุณจะเข้าใจโครงสร้าง JWT (`header.payload.signature`),
`access token` เทียบกับ `refresh token`, การ rotate/blacklist token, และปิดท้ายด้วย
**OAuth2** สำหรับ "Login ด้วย Google/GitHub" ผ่าน API (ต่อยอดจาก Social Authentication
ที่เรียนแบบ session-based ไปแล้วใน Part 036) เตรียมทบทวนกลไก `TokenAuthentication`
และ `AUTHENTICATION_BACKENDS` จาก Part นี้ให้แม่น เพราะ JWT จะเข้ามาแทนที่
`TokenAuthentication` ในหลายจุด แต่ยังใช้แนวคิด Permission class ชุดเดิมที่เพิ่งเรียน
ไปทั้งหมดโดยไม่ต้องเปลี่ยนแปลงอะไรเลย
