# Part 050: Testing REST APIs

> **ขั้นตอนที่ 491-500 ของหลักสูตร** | Phase 5: Django REST Framework และ API (Part สุดท้ายของ Phase)
>
> เป้าหมายของ Part นี้: เจาะลึกการทดสอบ REST API ให้ไกลกว่าที่ Part 045
> (ขั้นตอนที่ 448) แตะไว้แค่ผิวเผิน — คุณจะเรียนรู้ทุกความสามารถของ `APIClient`/
> `APITestCase`, เปรียบเทียบวิธี authenticate ระหว่างทดสอบทั้งสามแบบ (`force_authenticate`,
> JWT header จริง, Session), แยกทดสอบ Serializer validation และ Permission class
> เป็นหน่วยเดี่ยว (unit test) ให้ชัดเจนจาก integration test ผ่าน endpoint, ทดสอบ
> พฤติกรรมของ Pagination/Filtering ที่ตั้งค่าไว้ตั้งแต่ Part 047, ทำ mock การเรียก
> external API ด้วย `unittest.mock` และไลบรารี `responses`, เข้าใจแนวคิด Contract
> Testing ที่ตรวจสอบว่า response จริงตรงกับ OpenAPI schema จาก Part 049 หรือไม่,
> เกริ่นแนวคิด Load Testing และการผสาน test เข้ากับ CI Pipeline ก่อนจะปิดท้ายด้วย
> **การสรุปภาพรวมทั้ง Phase 5** (Part 039-050) พร้อม Quiz และแบบฝึกหัดใหญ่ที่รวบยอด
> ทุกทักษะ — สร้าง Blog REST API ฉบับสมบูรณ์ที่มี JWT auth, filtering, pagination,
> versioning, Swagger docs และ test coverage สูง เพื่อเตรียมพร้อมสำหรับ Phase 6
> ที่จะพา Frontend เข้ามาเชื่อมกับ API ที่คุณสร้างมาตลอด Phase นี้

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 491: `APIClient`/`APITestCase` เจาะลึก — HTTP methods ครบชุดและรูปแบบการยืนยันผล
- ขั้นตอนที่ 492: Testing Endpoint ที่ต้อง Authenticate ด้วยหลายวิธี (`force_authenticate`, JWT header จริง, Session)
- ขั้นตอนที่ 493: Testing Serializer Validation แยกจาก View (unit test serializer โดยตรง)
- ขั้นตอนที่ 494: Testing Permission Class แยกหน่วย (unit test) เทียบกับ Integration Test ผ่าน Endpoint
- ขั้นตอนที่ 495: Testing Pagination และ Filtering Behavior
- ขั้นตอนที่ 496: Mocking การเรียก External API ระหว่าง Test (`unittest.mock`, `responses`)
- ขั้นตอนที่ 497: แนวคิด Contract Testing / Schema Validation Testing
- ขั้นตอนที่ 498: เกริ่น Load Testing สำหรับ API Endpoint
- ขั้นตอนที่ 499: ผสาน API Test เข้ากับ CI Pipeline
- ขั้นตอนที่ 500: สรุป Phase 5 ทั้งหมด (Part 039-050) + Quiz + แบบฝึกหัดใหญ่ปิดท้าย Phase

---

## ขั้นตอนที่ 491: `APIClient`/`APITestCase` เจาะลึก — HTTP Methods ครบชุดและรูปแบบการยืนยันผล

### 491.1 ทบทวนสิ่งที่ Part 045 ใช้ไปแล้ว

Part 045 (ขั้นตอนที่ 448) แนะนำ `APITestCase` และ `APIClient` แบบพอให้ใช้งานได้
— สร้าง object ด้วย ORM ตรง ๆ ใน `setUp()`, ยิง `self.client.get()`/`.patch()`/
`.post()` แล้วเช็คแค่ `response.status_code` เป็นหลัก ขั้นตอนนี้จะพาไปดู
**กายวิภาคทั้งหมด** ของเครื่องมือทั้งสองตัวนี้ ซึ่งเป็นรากฐานของทุกเทคนิคที่เหลือใน
Part นี้

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.test
class APIClient(APIRequestFactory, DjangoClient):
    """
    สืบทอดจากทั้ง APIRequestFactory (รู้จักการสร้าง DRF Request) และ
    django.test.Client (มี session, cookie, middleware stack ครบเหมือนเบราว์เซอร์จริง)
    """


class APITestCase(SimpleTestCase, DjangoTestCase):
    client_class = APIClient   # ทำให้ self.client เป็น APIClient แทน django.test.Client
```

จุดสำคัญที่ต้องเข้าใจ: `APITestCase` แค่เปลี่ยน `client_class` ให้เป็น `APIClient`
เท่านั้น ส่วนพฤติกรรมพื้นฐาน (การครอบทุก test ด้วย transaction แล้ว rollback,
`setUpTestData()`, `assertNumQueries()` ที่เรียนจาก Part 025-026) **เหมือนกับ
`django.test.TestCase` ทุกประการ** เพราะสืบทอดมาโดยตรง

### 491.2 HTTP Methods ครบทั้ง 5 ตัวของ `APIClient`

```python
# ทุก method มี signature คล้ายกัน: (path, data=None, format=None, **extra)
self.client.get(path, data=None, **extra)
self.client.post(path, data=None, format=None, **extra)
self.client.put(path, data=None, format=None, **extra)
self.client.patch(path, data=None, format=None, **extra)
self.client.delete(path, data=None, format=None, **extra)
```

| Method | ใช้ทดสอบ Action ไหนของ `ModelViewSet` | Body ที่ส่งไปด้วยปกติ |
|---|---|---|
| `GET` | `list()` (ไม่มี pk) / `retrieve()` (มี pk) | ไม่มี (ใช้ query string ผ่าน `data=` แทน) |
| `POST` | `create()` | ข้อมูล object ใหม่ทั้งหมด |
| `PUT` | `update()` | ข้อมูล object **ครบทุก field** (full update) |
| `PATCH` | `partial_update()` | ข้อมูลเฉพาะ field ที่ต้องการแก้ (partial update) |
| `DELETE` | `destroy()` | ไม่มี |

**กับดักที่มือใหม่เจอบ่อยที่สุด**: ใช้ `PUT` แล้วส่ง field ไปไม่ครบ — เพราะ `PUT`
หมายถึง "แทนที่ทั้ง object" ตาม semantic ของ REST การส่ง field ไปไม่ครบจะทำให้
`serializer.is_valid()` fail ด้วย `400 Bad Request` (field ที่ขาดไปเป็น required)
ในขณะที่ `PATCH` ยอมรับ field บางส่วนได้เสมอ — นี่คือความแตกต่างที่ต้องทดสอบแยกกัน
อย่างชัดเจน ไม่ใช่ใช้แทนกันได้ตามใจ

### 491.3 พารามิเตอร์ `format` — ตัวกำหนดว่า Body จะถูกเข้ารหัสแบบไหน

```python
# format="json" (แนะนำเป็นค่าเริ่มต้นสำหรับ REST API เกือบทุกกรณี)
response = self.client.post(
    "/api/posts/",
    {"title": "โพสต์ใหม่", "content": "เนื้อหา" * 10, "category": 1},
    format="json",
)

# format="multipart" (จำเป็นเมื่อมีการอัปโหลดไฟล์ เช่น รูปปกโพสต์ — เจาะลึกเต็มรูปแบบ
# ใน Part 058) — ถ้าไม่ระบุ format ตอนส่งไฟล์ APIClient จะ fallback ไป multipart
# โดยอัตโนมัติเมื่อเจอ File object ใน data อยู่ดี
from django.core.files.uploadedfile import SimpleUploadedFile

cover_image = SimpleUploadedFile(
    "cover.jpg", b"fake-image-bytes", content_type="image/jpeg",
)
response = self.client.post(
    "/api/posts/",
    {"title": "โพสต์มีรูป", "content": "x" * 60, "cover": cover_image},
    format="multipart",
)
```

**ข้อควรระวังสำคัญที่สุด**: ถ้า**ไม่ระบุ** `format=` เลย `APIClient` จะใช้
`TEST_REQUEST_DEFAULT_FORMAT` (ค่า default ของ DRF settings คือ `'multipart'`
ไม่ใช่ `'json'`!) ซึ่งหมายความว่าถ้าลืมใส่ `format="json"` แล้วส่ง nested dict/list
เข้าไปใน body multipart form-data จะแปลงข้อมูลซับซ้อนให้ผิดรูปแบบแบบเงียบ ๆ
(เช่น list กลายเป็น string) จนทำให้ test fail ด้วยสาเหตุที่งงมาก — วิธีป้องกันแบบ
ถาวรคือตั้งค่านี้ไว้ใน settings ของโปรเจกต์เลย:

```python
# config/settings.py (หรือ config/settings/test.py)
REST_FRAMEWORK = {
    # ... ค่าอื่น ๆ ตามที่ตั้งไว้ตลอด Phase 5 ...
    "TEST_REQUEST_DEFAULT_FORMAT": "json",
}
```

ตั้งค่านี้ครั้งเดียวแล้วไม่ต้องพิมพ์ `format="json"` ซ้ำทุกบรรทัดในทุก test อีกต่อไป
(หลักสูตรนี้แนะนำให้ตั้งค่านี้ไว้เสมอตั้งแต่ Part 039 เป็นต้นไป)

### 491.4 กายวิภาคของ Response Object

```python
response = self.client.get("/api/posts/1/")

response.status_code    # int เช่น 200, 404 — ใช้คู่กับ rest_framework.status เสมอ
response.data           # Python dict/list ที่ยังไม่ผ่าน render (ใช้ตรวจสอบค่าได้ทันที)
response.content        # bytes ดิบหลัง render แล้ว (JSON string ในรูป bytes)
response.json()         # ของ django.test.Client เดิม — parse response.content เป็น dict
response.headers        # dict-like ของ response headers ทั้งหมด
response["Content-Type"]  # เข้าถึง header เฉพาะตัวได้โดยตรง
```

**ความแตกต่างที่สำคัญระหว่าง `response.data` กับ `response.json()`**: `response.data`
เป็นข้อมูลระดับ **Python object ก่อน render** (ยังไม่ผ่าน `JSONRenderer`) เข้าถึงได้
เร็วกว่าและไม่ต้อง parse ซ้ำ ในขณะที่ `response.json()` แปลงจาก `response.content`
(bytes ที่ render เป็น JSON string แล้ว) กลับมาเป็น dict อีกที — สำหรับ `APITestCase`
ควรใช้ `response.data` เป็นหลักเสมอเพราะตรงและเร็วกว่า `response.json()` ที่มีไว้
สำหรับ `django.test.Client` ทั่วไปที่ไม่รู้จัก DRF

### 491.5 รูปแบบ Assertion ที่ควรใช้กับ DRF Response

```python
from rest_framework import status


# 1. เช็ค status code — ใช้ constant จาก rest_framework.status เสมอ ไม่ hardcode ตัวเลข
self.assertEqual(response.status_code, status.HTTP_200_OK)

# 2. เช็คว่ามี key ที่ต้องการอยู่ใน response
self.assertIn("id", response.data)

# 3. เช็คค่าของ field เฉพาะเจาะจง
self.assertEqual(response.data["title"], "โพสต์ใหม่")

# 4. เช็คจำนวนรายการใน list response (ก่อนมี pagination ในขั้นตอนที่ 495)
self.assertEqual(len(response.data), 3)

# 5. เช็คว่า field ที่ควรเป็น read-only ไม่ถูกเปลี่ยนแม้ client จะพยายามส่งมา
self.assertNotEqual(response.data["author"], "attacker")

# 6. เช็ค error message ของ validation error (โครงสร้างเป็น dict ของ list เสมอ)
self.assertIn("title", response.data)                  # key ของ field ที่ error
self.assertEqual(
    response.data["title"][0].code, "required",         # DRF error มี .code เสมอ
)

# 7. เช็คจำนวน query ที่ยิงจริง (ป้องกัน N+1 — ทวนจาก Part 067 ล่วงหน้า)
with self.assertNumQueries(2):
    self.client.get("/api/posts/")
```

**จุดที่มือใหม่พลาดบ่อยที่สุดในข้อ 6**: DRF `ValidationError` แต่ละตัวเป็น
`ErrorDetail` (subclass ของ `str`) ที่มี attribute `.code` ติดมาด้วยเสมอ
(`"required"`, `"blank"`, `"invalid"`, หรือ code ที่ custom validator กำหนดเอง)
การเช็คแค่ `assertIn("title", response.data)` พิสูจน์ได้แค่ว่า "field นี้ error"
แต่การเช็ค `.code` เพิ่มพิสูจน์ได้ว่า **error ด้วยเหตุผลที่ถูกต้องจริง** — สำคัญมาก
เวลามี validator หลายตัวซ้อนกันในฟิลด์เดียว

### 491.6 `setUp()` vs `setUpTestData()` — ผลกระทบต่อความเร็วของชุด Test

```python
# blog/tests/test_post_crud_api.py
from django.contrib.auth import get_user_model
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Category, Post

User = get_user_model()


class PostCrudAPITests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        """
        รันแค่ 'ครั้งเดียว' ต่อทั้ง TestCase class (ไม่ใช่ทุก test method) แล้วห่อด้วย
        transaction ให้ทุก test method ใช้ข้อมูลชุดเดียวกันแบบ isolated (แก้ไขใน test
        หนึ่งจะไม่กระทบ test อื่น เพราะ Django rollback ให้อัตโนมัติหลังแต่ละ method)
        เร็วกว่า setUp() มากเมื่อมี test method จำนวนมากในคลาสเดียว
        """
        cls.category = Category.objects.create(name="Django", slug="django")
        cls.owner = User.objects.create_user(username="narin", password="pass12345")

    def setUp(self):
        """
        รันทุกครั้งก่อนแต่ละ test method — ใช้สำหรับสิ่งที่ต้อง 'สด' ทุกครั้ง เช่น
        object ที่ test บางตัวตั้งใจแก้ไขแล้วอาจกระทบ state ถ้าใช้ร่วมกับ test อื่น
        """
        self.post = Post.objects.create(
            title="โพสต์ทดสอบ",
            content="เนื้อหา" * 20,
            author=self.owner,
            category=self.category,
            is_published=True,
        )
        self.list_url = "/api/posts/"
        self.detail_url = f"/api/posts/{self.post.pk}/"
```

| เทคนิค | รันตอนไหน | เหมาะกับข้อมูลแบบไหน | ผลต่อความเร็ว |
|---|---|---|---|
| `setUpTestData()` (classmethod) | ครั้งเดียวต่อทั้ง class ห่อด้วย `TestCase`-level transaction | ข้อมูลอ้างอิงที่ทุก test ใช้ร่วมกันแบบ read-only เป็นหลัก (เช่น `Category`, `User`) | เร็วที่สุด — ลด query ซ้ำซ้อน |
| `setUp()` | ทุกครั้งก่อนแต่ละ test method | ข้อมูลที่ test คาดว่าจะ "สด" เสมอ หรือจะถูกแก้ไข/ลบใน test นั้น | ช้ากว่า แต่ปลอดภัยกว่าเมื่อ test แก้ไขข้อมูล |

### 491.7 ตัวอย่างเต็ม: ทดสอบ CRUD ครบวงจรของ `PostViewSet`

```python
# blog/tests/test_post_crud_api.py (ต่อจากเดิม)
    def test_list_returns_all_published_and_draft_posts_for_owner(self):
        response = self.client.get(self.list_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_retrieve_single_post_by_pk(self):
        response = self.client.get(self.detail_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["title"], "โพสต์ทดสอบ")

    def test_create_requires_authentication(self):
        payload = {"title": "ยังไม่ login", "content": "x" * 60, "category": self.category.pk}
        response = self.client.post(self.list_url, payload, format="json")
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_create_succeeds_when_authenticated(self):
        self.client.force_authenticate(user=self.owner)
        payload = {"title": "โพสต์ใหม่", "content": "x" * 60, "category": self.category.pk}
        response = self.client.post(self.list_url, payload, format="json")
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Post.objects.count(), 2)

    def test_put_requires_every_field(self):
        self.client.force_authenticate(user=self.owner)
        # ส่งแค่ title โดยไม่ส่ง content/category ที่จำเป็น → ต้อง fail ด้วย PUT
        response = self.client.put(self.detail_url, {"title": "แก้ทั้งก้อนแต่ไม่ครบ"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn("content", response.data)

    def test_patch_allows_partial_update(self):
        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(self.detail_url, {"title": "แก้แค่ title"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, "แก้แค่ title")

    def test_delete_removes_post(self):
        self.client.force_authenticate(user=self.owner)
        response = self.client.delete(self.detail_url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertFalse(Post.objects.filter(pk=self.post.pk).exists())
```

### 491.8 ตารางสรุป Status Code ที่ควรคาดหวังต่อ Action

| Action | Status Code สำเร็จ | Status Code ที่พบบ่อยเมื่อ fail |
|---|---|---|
| `list` (`GET` ไม่มี pk) | `200 OK` | — |
| `retrieve` (`GET` มี pk) | `200 OK` | `404 Not Found` (ไม่พบ object) |
| `create` (`POST`) | `201 Created` | `400 Bad Request` (validation), `403 Forbidden` (permission) |
| `update` (`PUT`) | `200 OK` | `400 Bad Request` (ส่ง field ไม่ครบ) |
| `partial_update` (`PATCH`) | `200 OK` | `400 Bad Request` (field ที่ส่งมาไม่ผ่าน validate) |
| `destroy` (`DELETE`) | `204 No Content` | `403 Forbidden`, `404 Not Found` |

---

## ขั้นตอนที่ 492: Testing Endpoint ที่ต้อง Authenticate ด้วยหลายวิธี

### 492.1 สามวิธีหลักในการจำลอง Authentication ระหว่างทดสอบ

Part 045 ใช้ `force_authenticate()` เพียงวิธีเดียวตลอดทั้ง Part เพราะเร็วและง่าย
แต่ในระบบจริงคุณต้องมั่นใจว่า **กลไก authentication ที่แท้จริง** (ไม่ใช่แค่ permission
logic) ก็ทำงานถูกต้องด้วย — ขั้นตอนนี้เปรียบเทียบวิธีทดสอบ authentication ทั้ง 3 แบบ
ที่ใช้ในระบบจริง

| วิธี | ทดสอบอะไรจริง | ความเร็ว |
|---|---|---|
| `force_authenticate(user=...)` | **เฉพาะ** permission/business logic — ข้าม authentication class ทั้งหมด | เร็วที่สุด (ไม่มี DB query สำหรับ auth, ไม่ต้องสร้าง token) |
| JWT ผ่าน header จริง (`Authorization: Bearer ...`) | ทั้ง `JWTAuthentication` และ permission — เหมือน client จริงเป๊ะ | ปานกลาง (มีการเรียก `/api/token/` ก่อน 1 ครั้ง) |
| Session ผ่าน `client.login()` | ทั้ง `SessionAuthentication`, CSRF policy, และ permission | ช้าที่สุด (ต้อง query password hash + สร้าง session) |

### 492.2 `force_authenticate()` — กลไกเบื้องหลังที่ต้องเข้าใจก่อนใช้

```python
# แนวคิดจากซอร์สโค้ดจริงของ rest_framework.test.APIClient
def force_authenticate(self, user=None, token=None):
    self.handler._force_user = user
    self.handler._force_token = token
    if user is None:
        self.logout()   # ล้างค่า _force_user เดิม (ใช้เคลียร์ state ระหว่าง test methods)
```

`force_authenticate()` แทรกค่า `request.user`/`request.auth` เข้าไป**ก่อน**ที่
`perform_authentication()` (ทบทวนจาก Part 045 ขั้นตอนที่ 441.1) จะทำงานด้วยซ้ำ —
พูดง่าย ๆ คือมัน **ไม่ได้เรียก authentication class ตัวไหนเลยแม้แต่ตัวเดียว**
ข้อดีคือเร็วมากและ isolate ปัญหาได้ชัดเจน (ถ้า test fail รู้ทันทีว่าไม่ใช่ปัญหาจาก
authentication) แต่ข้อเสียคือ **ไม่มีทางจับบั๊กที่อยู่ใน authentication class เองได้เลย**
เช่น ถ้า `JWTAuthentication` settings ผิดพลาด (secret key ไม่ตรง, algorithm ผิด)
test ที่ใช้ `force_authenticate()` จะยังผ่านเขียวปกติ ทั้งที่ client จริงจะเจอ `401`
ทันที — นี่คือเหตุผลที่ควรมี test อย่างน้อยบางส่วนที่ใช้ JWT จริงคู่กันเสมอ ไม่ใช่ใช้
`force_authenticate()` 100% ของทุก test

### 492.3 ทดสอบด้วย JWT ผ่าน Header จริง

```python
# blog/tests/test_auth_methods.py
from django.contrib.auth import get_user_model
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Post

User = get_user_model()


class JWTRealAuthTests(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="somchai", password="SecurePass123!")
        self.post = Post.objects.create(title="โพสต์ทดสอบ", content="x" * 60, author=self.user)
        self.detail_url = f"/api/posts/{self.post.pk}/"

    def _obtain_access_token(self, username, password):
        """Helper: เรียก endpoint จริงของ Part 046 เพื่อแลก access token"""
        response = self.client.post(
            "/api/token/", {"username": username, "password": password}, format="json",
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        return response.data["access"]

    def test_valid_jwt_grants_access(self):
        access = self._obtain_access_token("somchai", "SecurePass123!")
        self.client.credentials(HTTP_AUTHORIZATION=f"Bearer {access}")

        response = self.client.patch(self.detail_url, {"title": "แก้ไขด้วย JWT จริง"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_malformed_jwt_is_rejected(self):
        self.client.credentials(HTTP_AUTHORIZATION="Bearer this-is-not-a-real-jwt")
        response = self.client.get(self.detail_url)
        # request.user จะกลาย AnonymousUser เพราะ authenticate() ล้มเหลว
        # แต่ endpoint นี้เป็น safe method จึงยังอ่านได้ (IsOwnerOrStaff อนุญาต GET เสมอ)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_missing_bearer_prefix_is_rejected(self):
        access = self._obtain_access_token("somchai", "SecurePass123!")
        # ลืมใส่คำว่า "Bearer" นำหน้า — รูปแบบผิดตาม AUTH_HEADER_TYPES (ทวนจาก Part 046 452.4)
        self.client.credentials(HTTP_AUTHORIZATION=access)
        response = self.client.patch(self.detail_url, {"title": "ไม่ควรผ่าน"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

`self.client.credentials(**kwargs)` ตั้งค่า header ที่จะแนบไปกับ **ทุก request
ถัดไป** ของ client instance นั้นโดยอัตโนมัติ (คล้าย `defaults` ของ `requests.Session`)
เรียก `self.client.credentials()` เปล่า ๆ (ไม่มี argument) เพื่อล้าง header ทั้งหมด
เมื่อไม่ต้องการให้ค้างไปยัง test method อื่น

### 492.4 ทดสอบด้วย Session ผ่าน `client.login()`

```python
# blog/tests/test_auth_methods.py (ต่อจากเดิม)
class SessionAuthTests(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="narin", password="pass12345")
        self.post = Post.objects.create(title="โพสต์ทดสอบ", content="x" * 60, author=self.user)
        self.detail_url = f"/api/posts/{self.post.pk}/"

    def test_login_via_session_grants_access(self):
        logged_in = self.client.login(username="narin", password="pass12345")
        self.assertTrue(logged_in)   # login() คืน False เงียบ ๆ ถ้า credential ผิด — ต้องเช็คเสมอ

        response = self.client.patch(self.detail_url, {"title": "แก้ไขผ่าน session"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_wrong_password_login_fails_silently(self):
        logged_in = self.client.login(username="narin", password="ผิดรหัส")
        self.assertFalse(logged_in)

        response = self.client.patch(self.detail_url, {"title": "ไม่ควรผ่าน"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

**เรื่อง CSRF ที่ต้องรู้ตอนทดสอบ**: `APIClient` (และ `django.test.Client` ที่มันสืบทอด)
มี `enforce_csrf_checks=False` เป็นค่า default เสมอ — แปลว่า `client.login()` แล้ว
ยิง `POST`/`PATCH` ตรง ๆ จะ **ไม่โดน CSRF บล็อก** ทั้งที่ browser จริงจะโดน (ทวนจาก
Part 045 ขั้นตอนที่ 449) เพราะ test client "แกล้งทำเป็น" ผ่าน CSRF middleware ให้เสมอ
เพื่อความสะดวก ถ้าต้องการทดสอบ CSRF behavior จริง ๆ ต้องสร้าง client แยกที่เปิด
enforcement:

```python
from rest_framework.test import APIClient

def test_session_post_without_csrf_token_is_rejected(self):
    strict_client = APIClient(enforce_csrf_checks=True)
    strict_client.login(username="narin", password="pass12345")

    response = strict_client.post("/api/posts/", {"title": "x", "content": "x" * 60}, format="json")
    self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    self.assertIn("CSRF", str(response.data.get("detail", "")))
```

### 492.5 ตารางเปรียบเทียบ 3 วิธีแบบละเอียด

| ประเด็น | `force_authenticate()` | JWT Header จริง | Session `login()` |
|---|---|---|---|
| ทดสอบ authentication class จริงไหม | ❌ ไม่เลย | ✅ ใช่ | ✅ ใช่ |
| ต้อง query DB เพิ่มก่อน test ไหม | ไม่ต้อง | ต้อง (เรียก `/api/token/` ก่อน) | ต้อง (ตรวจ password hash) |
| ทดสอบ CSRF ได้ไหม | ไม่เกี่ยวข้อง | ไม่เกี่ยวข้อง (JWT ไม่ผ่าน CSRF) | ได้ ถ้าตั้ง `enforce_csrf_checks=True` |
| เหมาะกับการทดสอบอะไร | Permission/business logic ล้วน ๆ (ไม่สนใจ auth) | End-to-end flow ของ mobile app/SPA client จริง | End-to-end flow ของ Browsable API/เว็บที่ใช้ session |
| ความเร็วเมื่อมี test นับร้อยตัว | เร็วที่สุด | ช้ากว่าเล็กน้อย | ช้าที่สุด |
| คำแนะนำการใช้งาน | ใช้เป็น**ค่าเริ่มต้น**สำหรับ test ส่วนใหญ่ที่เน้นทดสอบ business logic | ใช้กับ test กลุ่มเล็กที่เจาะจงทดสอบ auth flow เอง (obtain/refresh/expire) | ใช้กับ test ของ Browsable API หรือฟีเจอร์ที่ผูกกับ CSRF โดยตรง |

**คำแนะนำเชิงปฏิบัติของหลักสูตรนี้**: ใช้ `force_authenticate()` เป็นค่าเริ่มต้น
สำหรับ test ส่วนใหญ่ (เร็วและ isolate ปัญหาได้ดี) แล้วเขียน test แยกกลุ่มเล็ก ๆ
ที่ใช้ JWT/Session จริงเฉพาะสำหรับไฟล์ที่ทดสอบ **authentication flow เอง**
(`test_auth_methods.py`) — ไม่ต้องเปลี่ยนทุก test ในโปรเจกต์ไปใช้ JWT จริงหมด
เพราะจะทำให้ test suite ช้าลงมากโดยไม่ได้ประโยชน์เพิ่มในการทดสอบ permission

### 492.6 ทดสอบ Token หมดอายุด้วยการปลอมเวลาหมดอายุแบบ Deterministic

การรอให้ token หมดอายุจริง (15 นาทีตาม `ACCESS_TOKEN_LIFETIME`) ในระหว่าง test
เป็นไปไม่ได้ในทางปฏิบัติ วิธีที่ถูกต้องคือสร้าง token แล้วบังคับตั้งเวลาหมดอายุให้
เป็นอดีตโดยตรงผ่าน API ของ `simplejwt` เอง (ไม่ต้อง mock เวลาทั้งระบบให้ซับซ้อน):

```python
# blog/tests/test_auth_methods.py (ต่อจากเดิม)
from datetime import timedelta

from rest_framework_simplejwt.tokens import AccessToken


class JWTExpiryTests(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(username="somchai", password="SecurePass123!")

    def test_expired_access_token_is_rejected(self):
        token = AccessToken.for_user(self.user)
        token.set_exp(lifetime=timedelta(seconds=-1))   # บังคับให้หมดอายุไปแล้ว 1 วินาที

        self.client.credentials(HTTP_AUTHORIZATION=f"Bearer {token}")
        response = self.client.patch(
            "/api/posts/1/", {"title": "ไม่ควรผ่าน"}, format="json",
        )
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
        self.assertEqual(response.data["code"], "token_not_valid")
```

`token.set_exp(lifetime=timedelta(seconds=-1))` เขียนทับ claim `exp` ตรง ๆ ให้เป็น
เวลาในอดีต — วิธีนี้ **แม่นยำ 100% และรันได้ทันที** ต่างจากการใช้ library อย่าง
`freezegun` ที่ต้อง mock เวลาทั้งระบบ (ซึ่งเสี่ยงกระทบส่วนอื่นของ test ที่ไม่เกี่ยวข้อง)
— ใช้เทคนิคนี้เมื่อต้องการทดสอบเฉพาะจุดที่เกี่ยวกับ `exp` claim เท่านั้น

---

## ขั้นตอนที่ 493: Testing Serializer Validation แยกจาก View

### 493.1 ทำไมต้อง Unit Test Serializer แยกออกจาก View

Test ทั้งหมดในขั้นตอนที่ 491-492 เป็น **integration test** — ยิง HTTP request
เต็มรูปแบบผ่าน URL routing → permission → serializer → model ทุกชั้น ข้อดีคือ
สมจริง แต่ข้อเสียคือ **ช้ากว่า** (ต้องผ่านทุกชั้น) และเมื่อ test fail บอกได้แค่ว่า
"endpoint นี้มีปัญหา" ไม่ได้บอกตรง ๆ ว่าปัญหาอยู่ที่ layer ไหน

การเรียก Serializer โดยตรง (ไม่ผ่าน HTTP เลย) แก้ปัญหานี้: เร็วกว่ามาก (ไม่มี
URL routing/permission/middleware) และเมื่อ fail จะรู้ทันทีว่าปัญหาอยู่ที่ validation
logic ของ serializer เท่านั้น จริง ๆ

### 493.2 Serializer ที่จะใช้ทดสอบตลอดขั้นตอนนี้

ต่อยอดจาก `PostSerializer` ที่ Part 045 (ขั้นตอนที่ 450.1) สรุปไว้ เพิ่ม validation
logic ที่ซับซ้อนขึ้นเพื่อให้มีอะไรให้ทดสอบจริงจัง:

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

    def validate_title(self, value):
        """Field-level validation — ตรวจแค่ field เดียวโดด ๆ"""
        forbidden_words = ["สแปม", "spam"]
        if any(word in value.lower() for word in forbidden_words):
            raise serializers.ValidationError(
                "หัวข้อห้ามมีคำต้องห้าม", code="forbidden_word",
            )
        return value

    def validate(self, attrs):
        """
        Object-level validation — ตรวจความสัมพันธ์ระหว่างหลาย field พร้อมกัน
        กฎธุรกิจ: โพสต์ที่ตั้ง is_published=True ต้องมี category เสมอ (ห้ามเผยแพร่
        โพสต์ที่ยังไม่จัดหมวดหมู่)
        """
        is_published = attrs.get(
            "is_published", getattr(self.instance, "is_published", False),
        )
        category = attrs.get(
            "category", getattr(self.instance, "category", None),
        )
        if is_published and category is None:
            raise serializers.ValidationError(
                {"category": "โพสต์ที่เผยแพร่แล้วต้องมีหมวดหมู่เสมอ"},
                code="published_without_category",
            )
        return attrs
```

สังเกตว่า `validate()` ต้องเช็ค `getattr(self.instance, ..., default)` เสมอเพื่อ
รองรับทั้งกรณี **create** (ไม่มี `self.instance` เป็น `None`) และ **partial update**
(client อาจส่งมาแค่ `is_published` โดยไม่ส่ง `category` มาด้วย ต้องดึงค่าเดิมจาก
`self.instance` มาเทียบ) — นี่คือกับดักคลาสสิกที่ทำให้ validation ผ่านตอน create
แต่พังตอน `PATCH` (หรือกลับกัน) ถ้าเขียนไม่ระวัง

### 493.3 ทดสอบ Field-level Validation (`validate_title`) โดยตรง

```python
# blog/tests/test_serializers.py
from django.test import TestCase

from blog.models import Category
from blog.serializers import PostSerializer


class PostSerializerValidationTests(TestCase):
    def setUp(self):
        self.category = Category.objects.create(name="Django", slug="django")

    def test_forbidden_word_in_title_is_rejected(self):
        serializer = PostSerializer(data={
            "title": "นี่คือ SPAM แน่นอน",
            "content": "x" * 60,
            "category": self.category.pk,
            "is_published": False,
        })

        self.assertFalse(serializer.is_valid())
        self.assertIn("title", serializer.errors)
        self.assertEqual(serializer.errors["title"][0].code, "forbidden_word")

    def test_clean_title_passes_validation(self):
        serializer = PostSerializer(data={
            "title": "หัวข้อปกติดี",
            "content": "x" * 60,
            "category": self.category.pk,
            "is_published": False,
        })
        self.assertTrue(serializer.is_valid())
```

สังเกตว่าไม่ได้สืบทอดจาก `APITestCase` เลย — ใช้แค่ `django.test.TestCase` ธรรมดา
เพราะ**ไม่มีการยิง HTTP request ใด ๆ เกิดขึ้น** การเรียก `PostSerializer(data=...)`
แล้ว `.is_valid()` คือการเรียก Python class ตรง ๆ ไม่ต่างจากการทดสอบฟังก์ชันทั่วไป

### 493.4 ทดสอบ Object-level Validation (`validate()`)

```python
# blog/tests/test_serializers.py (ต่อจากเดิม)
    def test_publish_without_category_is_rejected(self):
        serializer = PostSerializer(data={
            "title": "พร้อมเผยแพร่แต่ลืมเลือกหมวดหมู่",
            "content": "x" * 60,
            "is_published": True,
            # ไม่ส่ง category มาเลย
        })

        self.assertFalse(serializer.is_valid())
        self.assertIn("category", serializer.errors)
        self.assertEqual(
            serializer.errors["category"][0].code, "published_without_category",
        )

    def test_draft_without_category_is_allowed(self):
        """กฎนี้บังคับเฉพาะตอนเผยแพร่ ฉบับร่าง (is_published=False) ไม่บังคับหมวดหมู่"""
        serializer = PostSerializer(data={
            "title": "ฉบับร่างยังไม่มีหมวดหมู่",
            "content": "x" * 60,
            "is_published": False,
        })
        self.assertTrue(serializer.is_valid())

    def test_partial_update_keeps_existing_category_for_validation(self):
        """
        ทดสอบกับดักคลาสสิก: partial_update ไม่ส่ง category มา ต้องดึงค่าเดิมจาก
        instance มาเช็ค ไม่ใช่ถือว่าเป็น None แล้ว fail อย่างผิด ๆ
        """
        from django.contrib.auth import get_user_model
        from blog.models import Post

        User = get_user_model()
        owner = User.objects.create_user(username="narin", password="pass12345")
        existing_post = Post.objects.create(
            title="โพสต์เดิม", content="x" * 60,
            author=owner, category=self.category, is_published=True,
        )

        serializer = PostSerializer(
            instance=existing_post,
            data={"is_published": True},   # ไม่ส่ง category มา แต่ instance มีอยู่แล้ว
            partial=True,
        )
        self.assertTrue(serializer.is_valid(), serializer.errors)
```

### 493.5 ทดสอบ Serializer ที่ต้องการ `context` (เช่น `HyperlinkedIdentityField`)

Serializer บางตัวต้องการ `request` ใน context เพื่อสร้าง absolute URL (ทวนจาก
`HyperlinkedModelSerializer` ของ Part 041) — การทดสอบต้องส่ง context ปลอมเข้าไปเอง
โดยใช้ `APIRequestFactory` สร้าง request เปล่า ๆ:

```python
# blog/tests/test_serializers.py (ต่อจากเดิม)
from rest_framework.test import APIRequestFactory

from blog.serializers import PostHyperlinkedSerializer   # สมมติมี HyperlinkedModelSerializer


class PostHyperlinkedSerializerTests(TestCase):
    def setUp(self):
        self.factory = APIRequestFactory()
        self.category = Category.objects.create(name="Django", slug="django")

    def test_serializer_without_context_raises_clear_error(self):
        """
        ถ้าลืมส่ง context={'request': ...} ให้ HyperlinkedModelSerializer จะ raise
        AssertionError ทันทีตอน .data ถูกเข้าถึง (fail-fast ตามที่ DRF ออกแบบไว้)
        """
        serializer = PostHyperlinkedSerializer(data={
            "title": "ทดสอบ", "content": "x" * 60, "category": self.category.pk,
        })
        serializer.is_valid()

        with self.assertRaises(AssertionError):
            _ = PostHyperlinkedSerializer(serializer.save()).data

    def test_serializer_with_request_context_builds_absolute_url(self):
        request = self.factory.get("/api/posts/")
        serializer = PostHyperlinkedSerializer(
            data={"title": "ทดสอบ", "content": "x" * 60, "category": self.category.pk},
            context={"request": request},
        )
        serializer.is_valid(raise_exception=True)
        post = serializer.save()

        output = PostHyperlinkedSerializer(post, context={"request": request}).data
        self.assertTrue(output["url"].startswith("http://testserver/api/posts/"))
```

`APIRequestFactory().get(...)` สร้าง `HttpRequest` เปล่า ๆ ที่มี host เป็น
`testserver` เสมอ (ค่า default ของ Django test runner) — เพียงพอสำหรับ
`build_absolute_uri()` ที่ `HyperlinkedIdentityField` ใช้ภายใน โดยไม่ต้องเปิด
web server จริงเลย

### 493.6 ตารางสรุป: Unit Test Serializer vs Integration Test ผ่าน View

| ประเด็น | Unit Test Serializer โดยตรง | Integration Test ผ่าน `APIClient` |
|---|---|---|
| ความเร็ว | เร็วมาก (ไม่มี URL routing/permission/middleware) | ช้ากว่า (ผ่านทุก layer) |
| ทดสอบอะไร | Validation logic ล้วน ๆ (`validate_<field>`, `validate()`) | ทั้ง pipeline: URL → permission → serializer → response |
| เมื่อ fail บอกอะไรได้ชัดกว่า | ปัญหาอยู่ที่ validation logic แน่นอน | ต้องสืบต่อว่าปัญหาอยู่ที่ layer ไหน |
| จับบั๊กที่เกิดจาก permission/URL ได้ไหม | ❌ ไม่ได้ | ✅ ได้ |
| จำนวน test ที่ควรมีในโปรเจกต์จริง | เยอะ (ครอบคลุมทุก edge case ของ validation) | น้อยกว่า (พอครอบคลุม happy path + permission หลัก ๆ) |

**หลักการ Testing Pyramid ที่ควรยึดตลอดหลักสูตรนี้** (จะเจาะลึกอีกครั้งใน Part 061-062):
ยิ่งลงไปใกล้ unit test มากเท่าไหร่ ควรมี test จำนวนมากเท่านั้น (เร็ว, isolate ปัญหา
ชัด) ส่วน integration/end-to-end test ควรมีจำนวนพอประมาณ (สมจริงแต่ช้าและ debug ยาก
กว่า) ขั้นตอนที่ 494 จะนำหลักการเดียวกันนี้ไปใช้กับ Permission class ต่อ

---

## ขั้นตอนที่ 494: Testing Permission Class แยกหน่วย (Unit Test) เทียบกับ Integration Test ผ่าน Endpoint

### 494.1 ทบทวนสิ่งที่ Part 045 ทำไปแล้ว

Part 045 (ขั้นตอนที่ 448.4) unit test `IsOwnerOrReadOnly` ด้วย `APIRequestFactory`
ไปแล้วหนึ่งตัว ขั้นตอนนี้จะเจาะลึกเทคนิคเดียวกันกับ `IsOwnerOrStaff` (permission
ตัวสุดท้ายที่ Part 045 สรุปไว้ในขั้นตอนที่ 450.1) ซึ่งมี logic ซับซ้อนกว่าเพราะมี
3 เงื่อนไขซ้อนกัน (safe method / staff / owner) — เหมาะกับการโชว์ edge case ให้ครบ

```python
# blog/permissions.py (ทบทวนจาก Part 045 ขั้นตอนที่ 450.1)
from rest_framework.permissions import SAFE_METHODS, BasePermission


class IsOwnerOrStaff(BasePermission):
    message = "คุณต้องเป็นเจ้าของโพสต์นี้ หรือเป็นทีมงาน (staff) เท่านั้นจึงจะแก้ไข/ลบได้"

    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        if request.user.is_staff:
            return True
        return obj.author_id == request.user.id
```

### 494.2 `APIRequestFactory` เจาะลึก — สร้าง Request ปลอมโดยไม่ต้องมี URL Routing

```python
from rest_framework.test import APIRequestFactory

factory = APIRequestFactory()

# สร้าง request แต่ละ method ได้ครบเหมือน APIClient แต่ "ไม่ผ่าน" URL routing/
# middleware/permission ใด ๆ เลย — ได้แค่ HttpRequest object เปล่า ๆ กลับมา
request = factory.get("/api/posts/1/")
request = factory.post("/api/posts/", {"title": "x"}, format="json")
request = factory.patch("/api/posts/1/", {"title": "x"}, format="json")

# ต้องตั้ง request.user เอง เพราะไม่มี AuthenticationMiddleware มาทำให้อัตโนมัติ
request.user = some_user
```

ข้อแตกต่างสำคัญจาก `APIClient`: `APIRequestFactory` **ไม่รัน middleware ใด ๆ เลย**
(ไม่มี `AuthenticationMiddleware`, ไม่มี `CsrfViewMiddleware`) คุณต้อง set
`request.user` เองด้วยมือเสมอ — นี่คือสิ่งที่ทำให้มันเร็วมาก (ข้ามทุกอย่างที่ไม่
เกี่ยวข้องกับสิ่งที่กำลังทดสอบ) แต่ก็หมายความว่า**ทดสอบได้แค่ logic ภายใน method
ที่เรียกตรง ๆ เท่านั้น** ไม่ได้ทดสอบว่า Django/DRF ประกอบทุกอย่างเข้าด้วยกันถูกต้อง
จริงหรือไม่

### 494.3 Unit Test `IsOwnerOrStaff` ครบทุก Edge Case

```python
# blog/tests/test_permissions_unit.py
from unittest.mock import Mock

from django.contrib.auth import get_user_model
from django.test import TestCase
from rest_framework.test import APIRequestFactory

from blog.models import Post
from blog.permissions import IsOwnerOrStaff

User = get_user_model()


class IsOwnerOrStaffUnitTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.owner = User.objects.create_user(username="narin", password="pass12345")
        cls.other_user = User.objects.create_user(username="kai", password="pass12345")
        cls.staff = User.objects.create_user(
            username="editor", password="pass12345", is_staff=True,
        )
        cls.post = Post.objects.create(title="โพสต์ทดสอบ", content="x" * 60, author=cls.owner)

    def setUp(self):
        self.factory = APIRequestFactory()
        self.permission = IsOwnerOrStaff()
        self.fake_view = Mock()   # ไม่ได้ใช้ view เลยใน has_object_permission ของ class นี้

    def test_safe_method_allowed_for_anyone(self):
        request = self.factory.get("/api/posts/1/")
        request.user = self.other_user
        self.assertTrue(
            self.permission.has_object_permission(request, self.fake_view, self.post)
        )

    def test_unsafe_method_denied_for_random_user(self):
        request = self.factory.patch("/api/posts/1/")
        request.user = self.other_user
        self.assertFalse(
            self.permission.has_object_permission(request, self.fake_view, self.post)
        )

    def test_unsafe_method_allowed_for_owner(self):
        request = self.factory.delete("/api/posts/1/")
        request.user = self.owner
        self.assertTrue(
            self.permission.has_object_permission(request, self.fake_view, self.post)
        )

    def test_unsafe_method_allowed_for_staff_even_if_not_owner(self):
        request = self.factory.put("/api/posts/1/")
        request.user = self.staff
        self.assertTrue(
            self.permission.has_object_permission(request, self.fake_view, self.post)
        )

    def test_message_attribute_is_set_correctly(self):
        self.assertEqual(
            self.permission.message,
            "คุณต้องเป็นเจ้าของโพสต์นี้ หรือเป็นทีมงาน (staff) เท่านั้นจึงจะแก้ไข/ลบได้",
        )
```

สังเกตว่า test 5 ตัวนี้ครอบคลุม **ทุกช่องของตาราง truth table** ของเงื่อนไข 3 ตัว
(safe/unsafe × owner/non-owner × staff/non-staff) — สิ่งที่ยากจะทำให้ครบถ้วนด้วย
integration test เพราะต้องสร้าง user 3 แบบ + endpoint จริงสำหรับทุก combination
ในขณะที่ unit test ทำได้เร็วและอ่านง่ายกว่ามาก

### 494.4 Integration Test ตัวเดียวกันผ่าน Endpoint จริง (สำหรับเทียบ)

```python
# blog/tests/test_permissions_integration.py
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Post


class IsOwnerOrStaffIntegrationTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        from django.contrib.auth import get_user_model
        User = get_user_model()
        cls.owner = User.objects.create_user(username="narin", password="pass12345")
        cls.staff = User.objects.create_user(username="editor", password="pass12345", is_staff=True)
        cls.post = Post.objects.create(title="โพสต์ทดสอบ", content="x" * 60, author=cls.owner)
        cls.detail_url = f"/api/posts/{cls.post.pk}/"

    def test_staff_can_delete_any_post_via_real_endpoint(self):
        self.client.force_authenticate(user=self.staff)
        response = self.client.delete(self.detail_url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertFalse(Post.objects.filter(pk=self.post.pk).exists())
```

Test นี้ทดสอบตัวอย่างเดียว (`staff` ลบโพสต์คนอื่นได้) แต่ผ่าน **ทุก layer จริง**
(URL routing → `IsAuthenticatedOrReadOnly` → `IsOwnerOrStaff` → `perform_destroy()`
→ ORM `.delete()` จริง) — เป็นการยืนยันว่าทุกชิ้นส่วนถูกประกอบเข้าด้วยกันถูกต้อง
แต่ **ไม่เหมาะจะเขียนซ้ำสำหรับทุก combination** เพราะแต่ละ test ต้องสร้าง user +
post ใหม่และยิง HTTP เต็มรูปแบบ ช้ากว่า unit test หลายเท่า

### 494.5 ตารางสรุป Trade-off

| ประเด็น | Unit Test Permission (`APIRequestFactory`) | Integration Test (`APIClient`) |
|---|---|---|
| ความเร็วต่อ test | เร็วมาก (ไม่มี DB query สำหรับ auth, ไม่มี URL routing) | ช้ากว่า |
| ครอบคลุม edge case ได้ง่ายแค่ไหน | ง่ายมาก — เปลี่ยนแค่ `request.user`/`request.method` | ยากกว่า — ต้องสร้าง user/URL จริงทุก case |
| จับบั๊กจาก `get_permissions()`/`permission_classes` ผิดที่ view ได้ไหม | ❌ ไม่ได้ (ทดสอบแค่ class เดี่ยว ๆ) | ✅ ได้ |
| จับบั๊กจาก URL routing ผิดได้ไหม | ❌ ไม่ได้ | ✅ ได้ |
| จำนวนที่ควรมีในโปรเจกต์จริง | เยอะ ครอบคลุมทุก combination ของ permission logic | น้อยกว่า พอยืนยันว่าประกอบกันถูกต้องในภาพรวม |

### 494.6 กลยุทธ์แนะนำ: Testing Pyramid สำหรับ Permission โดยเฉพาะ

```
              ▲
             ╱ ╲        E2E ผ่าน JWT จริง (ขั้นตอนที่ 492)
            ╱   ╲       — น้อยที่สุด, ยืนยัน flow สมบูรณ์แบบ client จริง
           ╱─────╲
          ╱       ╲     Integration ผ่าน APIClient + force_authenticate()
         ╱         ╲    (ขั้นตอนที่ 494.4, Part 045 ขั้นตอนที่ 448.1-448.3)
        ╱───────────╲   — พอประมาณ, ยืนยันว่าทุก layer ประกอบกันถูกต้อง
       ╱             ╲
      ╱               ╲  Unit Test Serializer + Permission แยกหน่วย
     ╱                 ╲ (ขั้นตอนที่ 493, 494.3)
    ╱───────────────────╲ — เยอะที่สุด, ครอบคลุมทุก edge case เร็วและชัดเจน
```

รูปสามเหลี่ยมนี้คือหลักการเดียวกับที่ Part 061-062 (Phase 7) จะเจาะลึกอย่างเป็น
ระบบ — Part นี้แค่ปูพื้นฐานให้เห็นภาพจริงในบริบทของ DRF ก่อน

---

## ขั้นตอนที่ 495: Testing Pagination และ Filtering Behavior

### 495.1 ทบทวนการตั้งค่าจาก Part 047

Part 047 (ขั้นตอนที่ 461-470) ตั้งค่า `PostViewSet` ให้รองรับ filtering, searching,
และ pagination ตามนี้ (ทบทวนก่อนเริ่มเขียน test):

```python
# blog/api_views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import viewsets
from rest_framework.filters import OrderingFilter, SearchFilter

from .models import Post
from .permissions import IsAuthenticatedOrReadOnly, IsOwnerOrStaff
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author", "category").all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrStaff]

    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ["is_published", "category", "author"]
    search_fields = ["title", "content"]
    ordering_fields = ["created_at", "title"]
    ordering = ["-created_at"]   # ค่าเริ่มต้นเมื่อ client ไม่ระบุ ?ordering=
```

```python
# blog/pagination.py
from rest_framework.pagination import PageNumberPagination


class StandardResultsPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = "page_size"
    max_page_size = 100
```

```python
# config/settings.py
REST_FRAMEWORK = {
    # ... ค่าอื่น ๆ ตามที่สะสมมาตั้งแต่ Part 039-049 ...
    "DEFAULT_PAGINATION_CLASS": "blog.pagination.StandardResultsPagination",
}
```

### 495.2 ทดสอบ Pagination เริ่มต้น: โครงสร้าง Response และ `next`/`previous`

```python
# blog/tests/test_filtering_pagination.py
from django.contrib.auth import get_user_model
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Category, Post

User = get_user_model()


class PaginationTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="narin", password="pass12345")
        cls.category = Category.objects.create(name="Django", slug="django")
        # สร้างโพสต์ 15 รายการ — มากกว่า page_size (10) เพื่อบังคับให้มี 2 หน้า
        for i in range(15):
            Post.objects.create(
                title=f"โพสต์ที่ {i}", content="x" * 60,
                author=cls.author, category=cls.category, is_published=True,
            )

    def test_default_page_size_is_ten(self):
        response = self.client.get("/api/posts/")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data["results"]), 10)
        self.assertEqual(response.data["count"], 15)

    def test_response_structure_has_pagination_keys(self):
        response = self.client.get("/api/posts/")
        # PageNumberPagination ห่อ response ด้วย 4 key เสมอ: count, next, previous, results
        for key in ("count", "next", "previous", "results"):
            self.assertIn(key, response.data)

    def test_next_link_present_on_first_page(self):
        response = self.client.get("/api/posts/")
        self.assertIsNotNone(response.data["next"])
        self.assertIsNone(response.data["previous"])

    def test_second_page_has_remaining_items_and_no_next(self):
        response = self.client.get("/api/posts/?page=2")
        self.assertEqual(len(response.data["results"]), 5)   # เหลือ 15 - 10 = 5
        self.assertIsNone(response.data["next"])
        self.assertIsNotNone(response.data["previous"])

    def test_custom_page_size_query_param(self):
        response = self.client.get("/api/posts/?page_size=5")
        self.assertEqual(len(response.data["results"]), 5)

    def test_page_size_capped_at_max_page_size(self):
        response = self.client.get("/api/posts/?page_size=999")
        # max_page_size=100 แต่มีข้อมูลแค่ 15 → ได้ทั้งหมด 15 (ไม่เกิน max ที่ตั้งไว้)
        self.assertEqual(len(response.data["results"]), 15)

    def test_out_of_range_page_returns_404(self):
        response = self.client.get("/api/posts/?page=999")
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)
```

**จุดที่มือใหม่พลาดบ่อยที่สุด**: ลืมว่าเมื่อเปิด pagination แล้ว `response.data`
**ไม่ใช่ list ของ object อีกต่อไป** แต่กลายเป็น dict ที่มี key `results` ห่อ list
ไว้ข้างใน — test เก่าที่เขียนไว้ก่อนเปิด pagination (เช่นในขั้นตอนที่ 491.7 ที่เช็ค
`len(response.data)` ตรง ๆ) จะ **fail ทันที** เพราะ `len(dict)` นับจำนวน key
(เท่ากับ 4 เสมอ) ไม่ใช่จำนวนรายการ ต้องแก้เป็น `len(response.data["results"])`
เสมอหลังเปิด pagination — นี่คือเหตุผลที่ควรเขียน test ให้ "พัง" (fail) ทันทีเมื่อ
มีการเปลี่ยนแปลงโครงสร้าง response แบบนี้ เพื่อจับได้ตั้งแต่เนิ่น ๆ

### 495.3 ทดสอบ `DjangoFilterBackend`

```python
# blog/tests/test_filtering_pagination.py (ต่อจากเดิม)
class FilteringTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="narin", password="pass12345")
        cls.django_category = Category.objects.create(name="Django", slug="django")
        cls.python_category = Category.objects.create(name="Python", slug="python")

        Post.objects.create(
            title="เรียน Django", content="x" * 60,
            author=cls.author, category=cls.django_category, is_published=True,
        )
        Post.objects.create(
            title="เรียน Python พื้นฐาน", content="x" * 60,
            author=cls.author, category=cls.python_category, is_published=False,
        )

    def test_filter_by_is_published(self):
        response = self.client.get("/api/posts/?is_published=true")
        self.assertEqual(response.data["count"], 1)
        self.assertEqual(response.data["results"][0]["title"], "เรียน Django")

    def test_filter_by_category(self):
        response = self.client.get(f"/api/posts/?category={self.python_category.pk}")
        self.assertEqual(response.data["count"], 1)
        self.assertEqual(response.data["results"][0]["title"], "เรียน Python พื้นฐาน")

    def test_filter_by_invalid_category_returns_empty_not_error(self):
        """
        django-filter ไม่ raise error เมื่อค่าที่กรองไม่ match กับอะไรเลย — คืน
        queryset ว่างเปล่าแทน (200 OK พร้อม count=0 ไม่ใช่ 400/404)
        """
        response = self.client.get("/api/posts/?category=99999")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["count"], 0)
```

### 495.4 ทดสอบ `SearchFilter`

```python
# blog/tests/test_filtering_pagination.py (ต่อจากเดิม)
class SearchFilterTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="narin", password="pass12345")
        Post.objects.create(title="สอน Django ตั้งแต่พื้นฐาน", content="x" * 60, author=cls.author)
        Post.objects.create(title="สอน Flask เบื้องต้น", content="เนื้อหาเกี่ยวกับ django ด้วย", author=cls.author)
        Post.objects.create(title="สูตรอาหารไทย", content="ไม่เกี่ยวกับเว็บเลย", author=cls.author)

    def test_search_matches_title_or_content_case_insensitive(self):
        response = self.client.get("/api/posts/?search=django")
        # SearchFilter ค้นแบบ case-insensitive และค้นทั้ง title และ content (search_fields)
        self.assertEqual(response.data["count"], 2)

    def test_search_with_no_match_returns_empty(self):
        response = self.client.get("/api/posts/?search=วิ่งมาราธอน")
        self.assertEqual(response.data["count"], 0)
```

### 495.5 ทดสอบ `OrderingFilter`

```python
# blog/tests/test_filtering_pagination.py (ต่อจากเดิม)
class OrderingFilterTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="narin", password="pass12345")
        cls.post_a = Post.objects.create(title="เอ", content="x" * 60, author=cls.author)
        cls.post_b = Post.objects.create(title="บี", content="x" * 60, author=cls.author)
        cls.post_c = Post.objects.create(title="ซี", content="x" * 60, author=cls.author)

    def test_default_ordering_is_newest_first(self):
        response = self.client.get("/api/posts/")
        titles = [item["title"] for item in response.data["results"]]
        self.assertEqual(titles, ["ซี", "บี", "เอ"])   # สร้างล่าสุด = ซี → มาก่อนตาม -created_at

    def test_explicit_ordering_by_title_ascending(self):
        response = self.client.get("/api/posts/?ordering=title")
        titles = [item["title"] for item in response.data["results"]]
        self.assertEqual(titles, ["เอ", "บี", "ซี"])

    def test_ordering_by_field_not_in_ordering_fields_is_ignored(self):
        """
        ทดสอบว่า client พยายาม ordering ด้วย field ที่ไม่ได้อยู่ใน ordering_fields
        (เช่น 'id') จะถูก 'เพิกเฉย' อย่างเงียบ ๆ (ใช้ ordering default แทน) ไม่ error
        — เป็นพฤติกรรม default ของ OrderingFilter เพื่อความปลอดภัย (ป้องกัน client
        สั่ง ordering ด้วย field ที่ query แพงเกินไปหรือ field ที่ไม่ควรเปิดเผย)
        """
        response = self.client.get("/api/posts/?ordering=nonexistent_field")
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        titles = [item["title"] for item in response.data["results"]]
        self.assertEqual(titles, ["ซี", "บี", "เอ"])   # กลับไปใช้ default ordering
```

### 495.6 ตารางสรุปพฤติกรรมที่ต้องทดสอบเสมอสำหรับ Filtering/Pagination

| ฟีเจอร์ | Test ที่ต้องมีอย่างน้อย |
|---|---|
| Pagination | ขนาดหน้า default, โครงสร้าง response (`count`/`next`/`previous`/`results`), หน้าสุดท้าย, `page_size` query param, หน้าที่ไม่มีอยู่จริง |
| `DjangoFilterBackend` | กรองด้วยแต่ละ field ใน `filterset_fields`, ค่าที่ไม่ match (ต้องได้ผลว่างไม่ใช่ error) |
| `SearchFilter` | ค้นแบบ case-insensitive, ค้นได้ทุก field ใน `search_fields`, ไม่มีผลลัพธ์ |
| `OrderingFilter` | ordering default, ordering ที่ระบุเอง (ทั้งเพิ่มขึ้น/ลดลงด้วย `-`), field ที่ไม่ได้รับอนุญาต |

---

## ขั้นตอนที่ 496: Mocking การเรียก External API ระหว่าง Test

### 496.1 สถานการณ์จำลอง: แจ้งเตือน Slack เมื่อมีการเผยแพร่โพสต์

สมมติว่าทีมของคุณเพิ่มฟีเจอร์: ทุกครั้งที่โพสต์ถูกเผยแพร่ (`is_published` เปลี่ยน
เป็น `True`) ระบบจะยิง webhook ไปแจ้งเตือนใน Slack channel ของทีมเนื้อหาโดยอัตโนมัติ

```python
# blog/services.py
import requests
from django.conf import settings


def notify_slack_on_publish(post):
    """เรียก Slack Incoming Webhook เพื่อแจ้งเตือนทีมเนื้อหาเมื่อมีโพสต์ใหม่เผยแพร่"""
    response = requests.post(
        settings.SLACK_WEBHOOK_URL,
        json={"text": f"📢 โพสต์ใหม่เผยแพร่แล้ว: {post.title} โดย {post.author.username}"},
        timeout=5,
    )
    response.raise_for_status()
    return response
```

```python
# blog/api_views.py
from .services import notify_slack_on_publish


class PostViewSet(viewsets.ModelViewSet):
    # ... (เหมือนขั้นตอนที่ 495.1) ...

    def perform_update(self, serializer):
        was_published = serializer.instance.is_published
        post = serializer.save()
        if not was_published and post.is_published:
            notify_slack_on_publish(post)
```

**ปัญหาที่ต้องแก้ก่อนเขียน test**: ถ้าปล่อยให้ `notify_slack_on_publish()` ทำงาน
จริงระหว่าง test จะเกิดปัญหาอย่างน้อย 3 อย่าง: (1) test ยิง network request จริง
ทำให้ **ช้าและไม่เสถียร** (ขึ้นกับ Slack server จริง), (2) ถ้า `SLACK_WEBHOOK_URL`
ไม่ได้ตั้งค่าใน test environment จะ error ทันที, (3) ทุกครั้งที่รัน test suite
จะส่งข้อความ Slack จริงไปกวนทีมโดยไม่ตั้งใจ — ทางแก้คือ **mock** ฟังก์ชันที่เรียก
network ออกไปทั้งหมด

### 496.2 `unittest.mock`: `patch`, `Mock`, `MagicMock`

```python
from unittest.mock import MagicMock, Mock, patch

# Mock() / MagicMock() — สร้าง object ปลอมที่ "รับได้ทุก attribute/method call"
fake_response = Mock()
fake_response.status_code = 200
fake_response.raise_for_status = Mock(return_value=None)

# patch() — แทนที่ object จริงด้วย Mock ชั่วคราว เฉพาะในขอบเขตที่กำหนด (context
# manager หรือ decorator) แล้วคืนค่าเดิมกลับอัตโนมัติเมื่อจบ
with patch("blog.services.requests.post") as mock_post:
    mock_post.return_value = fake_response
    # โค้ดที่เรียก requests.post() ภายใน blog/services.py จะได้ fake_response แทน
```

**กฎสำคัญที่สุดของ `patch()`**: ต้อง patch **ที่ที่มันถูกเรียกใช้** ไม่ใช่ที่ที่มัน
ถูกนิยาม — เพราะ `from .services import notify_slack_on_publish` ทำให้
`blog/api_views.py` มี reference ของตัวเองไปยังฟังก์ชันนั้นแล้ว การ patch
`blog.services.notify_slack_on_publish` จะไม่มีผลกับ reference ที่ `api_views.py`
ถืออยู่แล้ว ต้อง patch `blog.api_views.notify_slack_on_publish` แทน (หรือ patch
`requests.post` ที่ระดับต่ำกว่าตามตัวอย่างข้างบนซึ่งปลอดภัยกว่าเสมอเพราะ patch
ที่จุดต้นตอจริง ๆ)

### 496.3 ทดสอบด้วย `unittest.mock.patch` โดยตรง

```python
# blog/tests/test_mocking.py
from unittest.mock import patch

from django.contrib.auth import get_user_model
from django.test import override_settings
from rest_framework import status
from rest_framework.test import APITestCase

from blog.models import Post

User = get_user_model()


@override_settings(SLACK_WEBHOOK_URL="https://hooks.slack.com/services/fake/webhook/url")
class SlackNotificationMockTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.owner = User.objects.create_user(username="narin", password="pass12345")
        cls.post = Post.objects.create(
            title="ฉบับร่าง", content="x" * 60, author=cls.owner, is_published=False,
        )

    @patch("blog.services.requests.post")
    def test_publishing_a_draft_triggers_slack_notification(self, mock_post):
        mock_post.return_value.raise_for_status.return_value = None

        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f"/api/posts/{self.post.pk}/", {"is_published": True}, format="json",
        )

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        mock_post.assert_called_once()   # ยืนยันว่าเรียก requests.post() จริง 1 ครั้ง

        # ตรวจสอบว่า argument ที่ส่งไปถูกต้อง (ไม่ได้แค่เช็คว่าถูกเรียก แต่เช็คเนื้อหาด้วย)
        _, kwargs = mock_post.call_args
        self.assertIn("โพสต์ใหม่เผยแพร่แล้ว", kwargs["json"]["text"])
        self.assertIn("ฉบับร่าง", kwargs["json"]["text"])

    @patch("blog.services.requests.post")
    def test_updating_already_published_post_does_not_notify_again(self, mock_post):
        """ป้องกัน spam แจ้งเตือนซ้ำ — ต้องแจ้งแค่ตอน 'เปลี่ยนสถานะ' เป็นเผยแพร่เท่านั้น"""
        self.post.is_published = True
        self.post.save()

        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f"/api/posts/{self.post.pk}/", {"title": "แก้แค่ชื่อ"}, format="json",
        )

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        mock_post.assert_not_called()   # ยืนยันว่าไม่ได้เรียก Slack ซ้ำ

    @patch("blog.services.requests.post")
    def test_slack_failure_does_not_break_the_api_response(self, mock_post):
        """
        กรณี Slack ล่มชั่วคราว (network error) ไม่ควรทำให้ API หลักพังไปด้วย —
        ทดสอบว่าระบบจัดการ exception จาก external service อย่างเหมาะสม (สมมติว่า
        perform_update() ครอบด้วย try/except ในโค้ดจริง)
        """
        import requests as requests_module
        mock_post.side_effect = requests_module.exceptions.ConnectionError("Slack unreachable")

        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f"/api/posts/{self.post.pk}/", {"is_published": True}, format="json",
        )
        # แม้ Slack ล่ม โพสต์ก็ยังต้องถูกอัปเดตสำเร็จ (ไม่ปล่อยให้ external service
        # ที่ไม่สำคัญเท่ามาทำให้ core feature ใช้งานไม่ได้)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

`mock_post.side_effect = SomeException(...)` คือวิธีจำลอง "การเรียกแล้ว raise
exception" แทนที่จะคืนค่าปกติ — มีประโยชน์มากสำหรับทดสอบ error handling โดยไม่ต้อง
ตัดสาย network จริงหรือปิด service จริงเพื่อจำลองความล้มเหลว

### 496.4 ทดสอบด้วยไลบรารี `responses` — Mock ที่ระดับ HTTP แทนที่ระดับ Python Function

```bash
pip install responses
pip freeze | grep responses >> requirements.txt
```

`responses` ทำงานต่างจาก `unittest.mock` ตรงที่มัน **ดักที่ระดับ HTTP layer**
(intercept ที่ `requests` library เอง) แทนที่จะ patch function เฉพาะจุด — ข้อดีคือ
ทดสอบได้สมจริงกว่า (ตรวจ URL, method, headers ที่ส่งจริงได้ครบ) และไม่ต้องกังวล
เรื่อง "patch ผิดที่" ตามกฎในขั้นตอนที่ 496.2:

```python
# blog/tests/test_mocking.py (ต่อจากเดิม)
import responses
from django.test import override_settings
from rest_framework.test import APITestCase


@override_settings(SLACK_WEBHOOK_URL="https://hooks.slack.com/services/fake/webhook/url")
class SlackNotificationResponsesLibraryTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.owner = User.objects.create_user(username="narin", password="pass12345")
        cls.post = Post.objects.create(
            title="ฉบับร่าง", content="x" * 60, author=cls.owner, is_published=False,
        )

    @responses.activate
    def test_publish_calls_correct_slack_webhook_url(self):
        # ลงทะเบียนว่า URL นี้ควรตอบกลับอะไรเมื่อถูกเรียกด้วย POST
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/fake/webhook/url",
            json={"ok": True},
            status=200,
        )

        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f"/api/posts/{self.post.pk}/", {"is_published": True}, format="json",
        )

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(responses.calls), 1)   # เรียกจริง 1 ครั้ง
        sent_request = responses.calls[0].request
        self.assertEqual(sent_request.url, "https://hooks.slack.com/services/fake/webhook/url")
        self.assertIn(b"\xe0\xb8\x89\xe0\xb8\x9a\xe0\xb8\xb1\xe0\xb8\x9a\xe0\xb8\xa3\xe0\xb9\x88\xe0\xb8\xb2\xe0\xb8\x87", sent_request.body)

    @responses.activate
    def test_slack_5xx_error_is_handled_gracefully(self):
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/fake/webhook/url",
            json={"error": "internal_error"},
            status=500,
        )

        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f"/api/posts/{self.post.pk}/", {"is_published": True}, format="json",
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    @responses.activate
    def test_unregistered_url_raises_connection_error(self):
        """
        ไม่ลงทะเบียน URL ไว้เลย — responses จะ raise ConnectionError ทันทีถ้าโค้ด
        พยายามเรียก URL ที่ไม่รู้จัก ป้องกัน test 'หลุด' ไปยิง network จริงโดยไม่ตั้งใจ
        (ต่างจาก unittest.mock.patch ที่ไม่มีการป้องกันข้อนี้ให้อัตโนมัติ)
        """
        with self.assertRaises(Exception):
            import requests
            requests.post("https://this-was-never-registered.example.com/")
```

### 496.5 ตารางเปรียบเทียบ `unittest.mock` กับ `responses`

| ประเด็น | `unittest.mock.patch` | ไลบรารี `responses` |
|---|---|---|
| ระดับที่ดักจับ | ระดับ Python function/object (ต้อง patch ให้ถูกที่) | ระดับ HTTP request จริง (ดักที่ `requests` library) |
| เสี่ยง "patch ผิดที่" ไหม | เสี่ยง (กฎในขั้นตอนที่ 496.2) | ไม่เสี่ยง — ดักทุกจุดที่เรียก `requests` โดยอัตโนมัติ |
| ตรวจสอบ URL/method/headers ที่ส่งจริงได้ละเอียดแค่ไหน | ได้ผ่าน `call_args` แต่ต้องเขียนเจาะจงเอง | ได้ละเอียดมากผ่าน `responses.calls[i].request` |
| ป้องกัน network call หลุดออกไปจริงโดยไม่ตั้งใจ | ❌ ไม่ป้องกัน (ถ้า patch ผิดที่ จะยิงจริงแบบเงียบ ๆ) | ✅ ป้องกันอัตโนมัติ (URL ที่ไม่ลงทะเบียนจะ error ทันที) |
| เหมาะกับ | Mock function/method ทั่วไปที่ไม่ใช่ HTTP call (เช่น mock `django.utils.timezone.now`) | Mock เฉพาะ HTTP call ไปยัง external API/webhook โดยเฉพาะ |
| ติดตั้งเพิ่มไหม | ไม่ต้อง (built-in ตั้งแต่ Python 3.3) | ต้อง `pip install responses` |

**คำแนะนำของหลักสูตรนี้**: ใช้ `responses` เป็นค่าเริ่มต้นเมื่อ mock การเรียก HTTP
ไปยัง external service โดยเฉพาะ (ปลอดภัยกว่าเรื่อง patch ผิดที่ และตรวจสอบ request
ที่ส่งจริงได้ละเอียดกว่า) ส่วน `unittest.mock` เหมาะกับการ mock สิ่งอื่นที่ไม่ใช่
HTTP call เช่น เวลาปัจจุบัน (`timezone.now`), การส่งอีเมล, หรือฟังก์ชันภายใน
โปรเจกต์เองที่ต้องการตัดออกชั่วคราว

---

## ขั้นตอนที่ 497: แนวคิด Contract Testing / Schema Validation Testing

### 497.1 Contract Testing คืออะไร และทำไมต้องมี

ทุก test ที่ผ่านมาใน Part นี้ตรวจสอบว่า **API ทำงานถูกต้องตาม business logic**
แต่ยังไม่มี test ตัวไหนตรวจสอบว่า **รูปร่าง (shape) ของ response ตรงกับสัญญา
(contract) ที่ประกาศไว้ใน OpenAPI schema จริงหรือไม่** — ปัญหาที่เกิดขึ้นบ่อยใน
ทีมจริง: นักพัฒนา backend แก้ `PostSerializer` เพิ่ม field ใหม่หรือเปลี่ยนชื่อ field
โดยลืมอัปเดตเอกสาร หรือกลับกัน ทำให้ **frontend/mobile team ที่พัฒนาแยกกันอ้างอิง
เอกสารที่ไม่ตรงกับของจริง** จนเกิดบั๊กตอน integrate

**Contract Testing** คือแนวคิดการเขียน test ที่ตรวจสอบว่า **สัญญา (schema/contract)
ระหว่างสองระบบยังคงตรงกันอยู่เสมอ** โดยไม่ต้องรันทั้งสองระบบพร้อมกันจริง สำหรับ
REST API ที่มี OpenAPI schema (จาก Part 049) หมายถึงการเขียน test ที่:

1. ตรวจสอบว่า schema ที่ generate ออกมายัง**ถูกต้องตามมาตรฐาน OpenAPI** เสมอ
2. ตรวจสอบว่า **response จริงจาก endpoint ตรงกับ schema ที่ประกาศไว้**

### 497.2 ทบทวนการตั้งค่า OpenAPI Schema จาก Part 049

```python
# config/settings.py (ทบทวนจาก Part 049)
INSTALLED_APPS = [
    # ...
    "drf_spectacular",
]

REST_FRAMEWORK = {
    # ...
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

SPECTACULAR_SETTINGS = {
    "TITLE": "SecureBlog API",
    "DESCRIPTION": "REST API สำหรับระบบบล็อกที่สร้างตลอด Phase 5",
    "VERSION": "1.0.0",
}
```

```python
# config/urls.py (ทบทวนจาก Part 049)
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    # ...
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
]
```

### 497.3 ทดสอบว่า Schema ที่ Generate ออกมาถูกต้องตามมาตรฐาน OpenAPI

```bash
pip install openapi-spec-validator
pip freeze | grep openapi-spec-validator >> requirements.txt
```

```python
# blog/tests/test_contract.py
import json

from openapi_spec_validator import validate
from rest_framework.test import APITestCase


class OpenAPISchemaValidityTests(APITestCase):
    def test_generated_schema_is_valid_openapi_document(self):
        response = self.client.get("/api/schema/?format=json")
        self.assertEqual(response.status_code, 200)

        schema = json.loads(response.content)
        # raise เมื่อ schema ผิดโครงสร้างตามมาตรฐาน OpenAPI 3.x — ไม่ raise = ผ่าน
        validate(schema)

    def test_schema_declares_all_expected_post_endpoints(self):
        response = self.client.get("/api/schema/?format=json")
        schema = json.loads(response.content)

        paths = schema["paths"]
        self.assertIn("/api/posts/", paths)
        self.assertIn("/api/posts/{id}/", paths)
        self.assertIn("get", paths["/api/posts/"])
        self.assertIn("post", paths["/api/posts/"])
        self.assertIn("patch", paths["/api/posts/{id}/"])
        self.assertIn("delete", paths["/api/posts/{id}/"])
```

Test แรกสำคัญมากในทีมที่มีคนหลายคนแก้ view/serializer พร้อมกัน — ถ้าใครเผลอเขียน
`@extend_schema` (decorator ของ `drf_spectacular` ที่ปรับแต่ง schema เอง จาก
Part 049) ผิดรูปแบบจนทำให้ schema ที่ generate ออกมาเสียหาย test นี้จะ fail ทันที
ในทันทีที่ CI รัน โดยไม่ต้องรอให้ frontend team มาแจ้งว่า Swagger UI ใช้งานไม่ได้

### 497.4 ทดสอบว่า Response จริงตรงกับ Schema — Contract Testing แบบเต็มรูปแบบ

```python
# blog/tests/test_contract.py (ต่อจากเดิม)
from django.contrib.auth import get_user_model
from jsonschema import validate as validate_against_json_schema

from blog.models import Category, Post

User = get_user_model()


class ResponseMatchesSchemaTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="narin", password="pass12345")
        cls.category = Category.objects.create(name="Django", slug="django")
        cls.post = Post.objects.create(
            title="โพสต์ทดสอบ", content="x" * 60,
            author=cls.author, category=cls.category, is_published=True,
        )

    def _get_component_schema(self, component_name):
        """ดึง schema component เฉพาะตัว (เช่น 'Post') ออกมาจาก OpenAPI document เต็ม"""
        response = self.client.get("/api/schema/?format=json")
        schema = json.loads(response.content)
        return schema["components"]["schemas"][component_name]

    def test_post_detail_response_matches_openapi_schema(self):
        post_schema = self._get_component_schema("Post")

        response = self.client.get(f"/api/posts/{self.post.pk}/")
        self.assertEqual(response.status_code, 200)

        # jsonschema.validate raise เมื่อโครงสร้างไม่ตรง (field หาย, type ผิด, ฯลฯ)
        validate_against_json_schema(instance=response.data, schema=post_schema)

    def test_post_list_item_matches_schema_for_every_item(self):
        post_schema = self._get_component_schema("Post")

        response = self.client.get("/api/posts/")
        for item in response.data["results"]:
            validate_against_json_schema(instance=item, schema=post_schema)
```

**สิ่งที่ test นี้จับได้ที่ test อื่นในหลักสูตรจับไม่ได้เลย**: สมมติว่านักพัฒนาคนหนึ่ง
เพิ่ม field `view_count` ลงใน `Post` model แล้วอัปเดต `PostSerializer` ให้ส่ง field
นี้กลับไปด้วย แต่ **ลืม** เพิ่ม `@extend_schema_field` หรือลืมอัปเดต schema ให้ตรง
— test แบบ `assertEqual(response.data["title"], ...)` ทั่วไปจะยังผ่านสบาย ๆ
(เพราะไม่ได้เช็ค field ที่ "เกินมา" หรือ "ขาดไป" เทียบกับ schema) แต่
`test_post_detail_response_matches_openapi_schema` จะจับความไม่ตรงกันนี้ได้ทันที
ถ้า schema ประกาศ `additionalProperties: false` ไว้อย่างเข้มงวด

### 497.5 ผสาน Contract Test เข้ากับ Workflow การพัฒนา

```
Developer แก้ไข PostSerializer (เพิ่ม/ลบ/เปลี่ยนชื่อ field)
         │
         ▼
รัน test suite (รวม test_contract.py) ก่อน commit/push
         │
         ├─ Schema validation ไม่ผ่าน (โครงสร้าง OpenAPI เสียหาย)
         │  → แก้ @extend_schema/serializer annotation ก่อน commit
         │
         └─ Response ไม่ตรง schema (field หาย/เกิน/type ผิด)
            → ต้องตัดสินใจ: อัปเดต schema ให้ตรงกับ response ใหม่ (ถ้าตั้งใจเปลี่ยน)
              หรือแก้ serializer ให้ตรงกับ schema เดิม (ถ้าเปลี่ยนโดยไม่ตั้งใจ)
```

นี่คือเหตุผลที่ contract test ควรอยู่ใน CI pipeline เสมอ (จะเจาะลึกเต็มรูปแบบใน
ขั้นตอนที่ 499 และ Part 088) เพื่อจับความไม่สอดคล้องกันนี้**ก่อน**ที่จะ merge เข้า
main branch ไม่ใช่ปล่อยให้ frontend/mobile team มาเจอปัญหาเองตอน integrate จริง

### 497.6 ตารางสรุประดับของ Schema Testing

| ระดับ | ตรวจสอบอะไร | เครื่องมือ |
|---|---|---|
| Schema Validity | Schema ที่ generate ถูกต้องตามมาตรฐาน OpenAPI 3.x หรือไม่ | `openapi-spec-validator` |
| Schema Completeness | Schema ประกาศ endpoint/method ครบตามที่ view จริงมีหรือไม่ | ตรวจ `schema["paths"]` ตรง ๆ |
| Response-Schema Conformance | Response จริงจาก endpoint ตรงกับ schema component หรือไม่ | `jsonschema.validate()` |
| Breaking Change Detection | Schema เปลี่ยนไปจากเวอร์ชันก่อนหน้าแบบที่ทำลาย backward compatibility หรือไม่ | เปรียบเทียบ schema เก่ากับใหม่ (diff) ใน CI — เจาะลึกใน Part 088 |

---

## ขั้นตอนที่ 498: เกริ่น Load Testing สำหรับ API Endpoint

### 498.1 Load Testing คืออะไร ต่างจาก Unit/Integration Testing อย่างไร

ทุก test ที่เรียนมาตั้งแต่ขั้นตอนที่ 491 ตอบคำถามว่า **"API ทำงานถูกต้องหรือไม่"**
(correctness) — ยิง request ทีละตัว ตรวจสอบผลลัพธ์ แล้วจบ ไม่ว่าจะรันกี่รอบก็ตาม
คำถามที่ test เหล่านี้ **ตอบไม่ได้เลย** คือ **"API ทำงานถูกต้องและเร็วพอ เมื่อมีคน
เรียกพร้อมกันหลายพันคนหรือไม่"** (performance ภายใต้ load) — นี่คือขอบเขตของ
**Load Testing**

| ประเด็น | Unit/Integration Testing (ขั้นตอนที่ 491-497) | Load Testing |
|---|---|---|
| ตอบคำถามว่า | ผลลัพธ์ถูกต้องไหม (correctness) | เร็วพอและเสถียรไหมภายใต้ traffic สูง (performance) |
| จำนวน request ที่ยิงต่อครั้ง | 1 request ต่อ 1 assertion | หลายร้อย-หลายพัน request พร้อมกัน (concurrent) |
| Database ที่ใช้ | Test database แยกต่างหาก (สร้าง/ทำลายทุกรอบ) | Environment ที่ใกล้เคียง production มากที่สุด |
| รันบ่อยแค่ไหน | ทุกครั้งที่ commit/push (ส่วนหนึ่งของ CI) | เป็นระยะ (ก่อน release ใหญ่, ก่อนเทศกาลที่คาดว่า traffic จะพุ่ง) |

### 498.2 Metric สำคัญที่ Load Testing วัด

| Metric | ความหมาย |
|---|---|
| **Throughput** | จำนวน request ที่ระบบรับมือได้ต่อวินาที (requests/sec หรือ RPS) |
| **Latency (p50/p95/p99)** | เวลาตอบสนอง — p95 หมายถึง "95% ของ request ตอบเร็วกว่าค่านี้" (สำคัญกว่าค่าเฉลี่ยมาก เพราะค่าเฉลี่ยซ่อน outlier ที่ผู้ใช้จริงเจอได้) |
| **Error Rate** | สัดส่วนของ request ที่ล้มเหลว (5xx, timeout) เมื่อ load เพิ่มขึ้น |
| **Concurrent Users** | จำนวนผู้ใช้ที่จำลองว่าใช้งานพร้อมกันในแต่ละช่วงเวลา |

### 498.3 ตัวอย่างเบื้องต้นด้วย `locust` (เจาะลึกเต็มรูปแบบใน Part 072)

```python
# locustfile.py (ตัวอย่างเบื้องต้นเท่านั้น — Part 072 จะเจาะลึกเต็มรูปแบบ)
from locust import HttpUser, task, between


class BlogAPIUser(HttpUser):
    wait_time = between(1, 3)   # จำลอง user แต่ละคนหน่วงเวลา 1-3 วินาทีระหว่าง request

    def on_start(self):
        """จำลองการ login ครั้งเดียวตอนเริ่ม simulate user แต่ละคน"""
        response = self.client.post("/api/token/", json={
            "username": "loadtest_user", "password": "pass12345",
        })
        self.access_token = response.json()["access"]

    @task(3)   # weight=3 → ถูกเรียกบ่อยกว่า task อื่น 3 เท่า (จำลองว่า user อ่านบ่อยกว่าเขียน)
    def list_posts(self):
        self.client.get(
            "/api/posts/",
            headers={"Authorization": f"Bearer {self.access_token}"},
        )

    @task(1)
    def create_post(self):
        self.client.post(
            "/api/posts/",
            json={"title": "Load test post", "content": "x" * 100},
            headers={"Authorization": f"Bearer {self.access_token}"},
        )
```

```bash
pip install locust
locust -f locustfile.py --host=http://127.0.0.1:8000
# เปิด http://localhost:8089 เพื่อกำหนดจำนวน concurrent users แล้วเริ่มยิง load
```

### 498.4 ตารางเปรียบเทียบเครื่องมือ Load Testing ยอดนิยม

| เครื่องมือ | ภาษาที่เขียน scenario | จุดเด่น |
|---|---|---|
| **Locust** | Python (เหมาะกับทีม Django ที่คุ้นเคย Python อยู่แล้ว) | เขียน scenario แบบ code ได้ยืดหยุ่นมาก, มี Web UI ในตัว |
| **k6** | JavaScript | เร็วมาก (เขียนด้วย Go), ผสาน CI/CD ได้ดี, ผลลัพธ์ export เป็น Grafana ได้ทันที |
| **Apache Bench (`ab`)** | Command-line flags ล้วน ๆ (ไม่มี scripting) | ง่ายและเร็วสำหรับการทดสอบพื้นฐานแบบ quick-check |
| **JMeter** | GUI-based (XML เบื้องหลัง) | ครบเครื่องมาก แต่ setup ซับซ้อนกว่าตัวอื่น |

### 498.5 ทำไมเนื้อหาส่วนนี้จึงมีแค่เท่านี้ใน Part 050

Load Testing เป็นหัวข้อที่ลึกและกว้างมากพอที่จะมี Part เต็มเป็นของตัวเอง
(**Part 072: Load Testing และ Scalability Testing** ใน Phase 8: Performance &
Caching) ซึ่งจะเจาะลึกทั้งการออกแบบ test scenario ที่สมจริง, การอ่านผลลัพธ์
เชิงสถิติ, การหา bottleneck จากผลการทดสอบ, และการทดสอบ scalability แบบ horizontal
scaling ขั้นตอนนี้ตั้งใจให้แค่ **เห็นภาพรวมว่ามันคืออะไรและต่างจาก unit test
อย่างไร** เพื่อไม่ให้คุณสับสนว่า "ทำไม test ที่เขียนมาตลอด Part นี้ถึงไม่ครอบคลุม
เรื่องความเร็วภายใต้ load เลย" — คำตอบคือมันเป็นคนละหมวดของการทดสอบโดยสิ้นเชิง

---

## ขั้นตอนที่ 499: ผสาน API Test เข้ากับ CI Pipeline

### 499.1 ทำไมต้องรัน Test อัตโนมัติทุกครั้งที่มีการเปลี่ยนโค้ด

Test ทั้งหมดที่เขียนมาตลอด Part นี้มีค่าน้อยมากถ้า**ไม่มีใครรันมันสม่ำเสมอ** —
ปัญหาคลาสสิกของทีมที่ไม่มี CI: นักพัฒนา A แก้โค้ดแล้วลืมรัน test ก่อน push,
นักพัฒนา B pull โค้ดมาแล้วเจอ test พังโดยไม่รู้สาเหตุ, ท้ายที่สุดทีมเริ่ม "ไม่เชื่อถือ"
test suite เพราะไม่รู้ว่า fail เพราะโค้ดจริงพังหรือ environment ของใครบางคนเพี้ยน
**Continuous Integration (CI)** แก้ปัญหานี้ด้วยการรัน test ชุดเดียวกันบน environment
ที่สะอาดและเหมือนกันทุกครั้ง โดยอัตโนมัติทุกครั้งที่มีการ push หรือเปิด Pull Request

### 499.2 เตรียม Test Settings แยกสำหรับ CI

```python
# config/settings/test.py
from .base import *  # noqa: F401, F403

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",   # ใช้ SQLite in-memory แทน PostgreSQL จริงเพื่อความเร็ว
    }
}

PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.MD5PasswordHasher",   # เร็วกว่า PBKDF2 มาก ใช้เฉพาะตอนทดสอบ
]

SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/test/webhook/url"

SIMPLE_JWT = {
    **SIMPLE_JWT,
    "SIGNING_KEY": "test-secret-key-not-for-production",
}
```

**คำเตือนสำคัญที่สุด**: `MD5PasswordHasher` เร็วกว่า `PBKDF2PasswordHasher`
(ค่า default ที่ปลอดภัยกว่ามากสำหรับ production) หลายสิบเท่า — ใช้เฉพาะใน
`test.py` เท่านั้น**ห้ามใช้ใน production เด็ดขาด** เพราะ MD5 ถอดรหัสได้ง่ายกว่า
มาก การสลับ hasher นี้ช่วยลดเวลารัน test suite ทั้งหมดได้อย่างมีนัยสำคัญเมื่อมี
test ที่สร้าง user จำนวนมาก (ทุกครั้งที่ `create_user()` ถูกเรียก Django ต้อง hash
password ทันที)

### 499.3 ตัวอย่าง GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Run Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: ตั้งค่า Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: ติดตั้ง dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install coverage

      - name: รัน Django system check
        run: python manage.py check --settings=config.settings.test

      - name: รัน test suite พร้อมวัด coverage
        run: |
          coverage run --source='.' manage.py test --settings=config.settings.test
          coverage report --fail-under=80
        env:
          SECRET_KEY: "ci-only-secret-key-not-for-production"

      - name: อัปโหลด coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: htmlcov/
```

### 499.4 Coverage Gate — บังคับมาตรฐานคุณภาพขั้นต่ำ

`coverage report --fail-under=80` คือ **coverage gate** — ทำให้ CI job **fail ทันที**
(exit code ไม่ใช่ 0) ถ้า test coverage โดยรวมของโปรเจกต์ต่ำกว่า 80% แม้ test ทุกตัว
ที่มีอยู่จะผ่านหมดก็ตาม กลไกนี้ป้องกันสถานการณ์ที่นักพัฒนาเขียนฟีเจอร์ใหม่โดยไม่เขียน
test คู่กันเลย แล้ว merge เข้า `main` ได้อย่างเงียบ ๆ

```bash
# รันในเครื่องตัวเองก่อน push เพื่อดู coverage แบบละเอียดเป็นราย-ไฟล์
coverage run --source='.' manage.py test --settings=config.settings.test
coverage report -m   # -m แสดง line number ที่ยังไม่ถูก test ครอบคลุม
coverage html        # สร้างรายงาน HTML แบบดูรายไฟล์ได้ที่ htmlcov/index.html
```

### 499.5 ทำไมเนื้อหาส่วนนี้จึงมีแค่เท่านี้ใน Part 050

เช่นเดียวกับ Load Testing ในขั้นตอนที่ 498 การผสาน test เข้ากับ CI/CD เป็นหัวข้อ
ที่ลึกพอจะมี Part เต็มเป็นของตัวเอง — **Part 065: Continuous Testing และ Code
Quality Tools** (Phase 7) จะเจาะลึกเรื่อง coverage เชิงลึก, linting/formatting
อัตโนมัติ (`ruff`, `black`), pre-commit hooks และ **Part 088: CI/CD ด้วย GitHub
Actions** (Phase 11) จะเจาะลึกเต็มรูปแบบเรื่อง multi-stage pipeline, matrix
testing (รัน test กับหลายเวอร์ชัน Python/Django พร้อมกัน), caching dependencies
ให้ CI เร็วขึ้น, และการเชื่อม CI เข้ากับขั้นตอน deploy อัตโนมัติ ขั้นตอนนี้ตั้งใจ
ให้แค่เห็นภาพรวมว่า **test ทุกตัวที่เขียนมาตลอด Part 050 ควรถูกใช้งานอย่างไรใน
เวิร์กโฟลว์การพัฒนาจริงของทีม** ก่อนที่จะไปเจาะลึกกลไกเบื้องหลังในภายหลัง

---

## ขั้นตอนที่ 500: สรุป Phase 5 ทั้งหมด (Part 039-050)

ยินดีด้วย! คุณเดินทางมาถึงจุดสิ้นสุดของ **Phase 5: Django REST Framework และ API**
แล้ว — นี่คือขั้นตอนที่ 500 จาก 1000 ขั้นตอนของหลักสูตรทั้งหมด (**ครึ่งทางพอดี!**)
ก่อนจะก้าวเข้าสู่ Phase 6 เราจะทบทวนภาพรวมทุกอย่างที่ผ่านมาตลอด 12 Part, ทดสอบ
ความเข้าใจด้วย Quiz และปิดท้ายด้วยแบบฝึกหัดใหญ่ที่รวบยอดทุกทักษะของทั้ง Phase

### 500.1 ทบทวนภาพรวม Part 039-050

| Part | หัวข้อหลัก | สิ่งที่ได้เรียนรู้สำคัญที่สุด |
|---|---|---|
| **039** | บทนำสู่ Django REST Framework | ทำไมต้องมี DRF, ติดตั้งและตั้งค่าเบื้องต้น, `Request`/`Response` object, `@api_view` |
| **040** | Serializers เบื้องต้น | `Serializer`, field types, `is_valid()`, `.data` vs `.validated_data`, การ serialize/deserialize |
| **041** | ModelSerializer และ Nested Serializers | `ModelSerializer` ลดโค้ดซ้ำซ้อน, nested serializer, `SerializerMethodField`, `HyperlinkedModelSerializer` |
| **042** | API Views: FBV และ `APIView` | `@api_view` decorator, `APIView` class, `dispatch()` ของ DRF, content negotiation |
| **043** | Generic API Views และ Mixins | `GenericAPIView`, mixin classes, `get_object()`, `check_object_permissions()` |
| **044** | ViewSets และ Routers | `ViewSet`/`ModelViewSet`, `DefaultRouter`, การลดโค้ดซ้ำซ้อนของ CRUD endpoint ทั้งชุด |
| **045** | DRF Permissions และ Authentication | Permission class มาตรฐาน, Custom Permission (`IsOwnerOrReadOnly`/`IsOwnerOrStaff`), `Session`/`Basic`/`Token` Authentication, CSRF |
| **046** | Authentication ขั้นสูง: JWT, OAuth2 | โครงสร้าง JWT, `simplejwt` เต็มรูปแบบ, Token Rotation/Blacklisting, Custom Claims, OAuth2 Provider, API Key |
| **047** | Filtering, Searching, Pagination | `DjangoFilterBackend`, `SearchFilter`, `OrderingFilter`, `PageNumberPagination` แบบ custom |
| **048** | API Versioning และ Throttling | กลยุทธ์ versioning ต่าง ๆ, `AcceptHeaderVersioning`, `UserRateThrottle`/`AnonRateThrottle` |
| **049** | API Documentation | `drf-spectacular`, OpenAPI schema generation, Swagger UI, `@extend_schema` |
| **050** | Testing REST APIs | `APIClient` เจาะลึก, ทดสอบ auth หลายวิธี, unit test serializer/permission, mocking, contract testing |

### 500.2 แผนภาพสถาปัตยกรรมรวมของ Phase 5

```
                    ┌───────────────────────────────────────────┐
                    │         Client (Mobile App / SPA)           │
                    └───────────────────────┬───────────────────┘
                                             │ HTTP + JSON
                    ┌────────────────────────▼───────────────────┐
                    │   Authentication Layer (Part 045-046)        │
                    │   JWTAuthentication → request.user           │
                    └────────────────────────┬───────────────────┘
                                             │
                    ┌────────────────────────▼───────────────────┐
                    │   Versioning + Throttling (Part 048)         │
                    │   AcceptHeaderVersioning, Rate Limiting       │
                    └────────────────────────┬───────────────────┘
                                             │
                    ┌────────────────────────▼───────────────────┐
                    │   Permission Layer (Part 045)                │
                    │   IsAuthenticatedOrReadOnly + IsOwnerOrStaff  │
                    └────────────────────────┬───────────────────┘
                                             │
              ┌──────────────────────────────▼──────────────────────────────┐
              │           PostViewSet (Part 044) — ModelViewSet               │
              │  filter_backends (Part 047): DjangoFilterBackend,             │
              │  SearchFilter, OrderingFilter → pagination_class              │
              └──────────────────────────────┬──────────────────────────────┘
                                             │
                    ┌────────────────────────▼───────────────────┐
                    │   PostSerializer (Part 040-041)              │
                    │   validate_title() / validate() (Part 050)   │
                    └────────────────────────┬───────────────────┘
                                             │
                    ┌────────────────────────▼───────────────────┐
                    │   Post / Category Model (Phase 2)            │
                    └───────────────────────────────────────────┘

              ▲ ทุกชั้นข้างบนถูกทดสอบครบด้วยเทคนิคจาก Part 050:
              │ unit test (493-494), integration test (491-492, 495),
              │ mocking (496), contract test กับ OpenAPI schema (Part 049 + 497)
```

### 500.3 Checklist ทักษะที่ควรมีก่อนไป Phase 6

- [ ] สร้าง Serializer/ModelSerializer พร้อม custom validation ได้เองตั้งแต่ต้น
- [ ] สร้าง `ModelViewSet` ที่ผสาน permission, filtering, pagination, versioning ครบ
- [ ] ตั้งค่า JWT authentication เต็มรูปแบบ (obtain/refresh/verify/blacklist)
- [ ] เขียน Custom Permission Class ทั้งระดับ view (`has_permission`) และระดับ object (`has_object_permission`)
- [ ] สร้างเอกสาร API อัตโนมัติด้วย `drf-spectacular` และอ่าน/แก้ schema ได้
- [ ] เขียน test ครบทั้ง unit test (serializer, permission) และ integration test (endpoint) ได้เอง
- [ ] ทดสอบ authentication ได้ทั้ง 3 วิธี (`force_authenticate`, JWT header จริง, session)
- [ ] Mock การเรียก external API ระหว่าง test ได้ทั้งด้วย `unittest.mock` และ `responses`
- [ ] เข้าใจความแตกต่างระหว่าง unit test, integration test, contract test, และ load test
- [ ] อธิบายได้ว่าทำไมต้องมี test coverage gate ใน CI pipeline

### 500.4 แบบทดสอบความเข้าใจรวม (Quiz)

ลองตอบคำถามต่อไปนี้ด้วยตัวเองก่อนเปิดดูเฉลย เพื่อประเมินว่าคุณพร้อมสำหรับ Phase 6
หรือยัง:

**1.** อธิบายความแตกต่างระหว่าง Authentication และ Permission ใน DRF พร้อมบอกว่า
แต่ละอย่างล้มเหลวแล้วจะได้ status code อะไร

**2.** ทำไม `permission_classes` ถึงเป็นการ "AND" (ต้องผ่านทุกตัว) แต่
`authentication_classes` เป็นการ "ไล่ลองจนกว่าจะสำเร็จ" ทั้งที่ทั้งคู่เป็น list
เหมือนกัน

**3.** อธิบายความแตกต่างระหว่าง Access Token กับ Refresh Token ของ JWT ว่าทำไม
ต้องแยกเป็นสองตัว ไม่ใช้ token เดียวที่อายุยาวไปเลย

**4.** `force_authenticate()` กับการ login ด้วย JWT header จริงระหว่างทดสอบ
ต่างกันอย่างไร และทำไมไม่ควรใช้ `force_authenticate()` กับทุก test 100%

**5.** อธิบายความแตกต่างระหว่าง unit test serializer โดยตรง กับ integration test
ผ่าน `APIClient` — แต่ละแบบจับบั๊กประเภทไหนได้ดีกว่ากัน

**6.** ทำไมการเปิด pagination ถึงทำให้ test เดิมที่เช็ค `len(response.data)`
ตรง ๆ พังทันที ต้องแก้เป็นอะไรแทน

**7.** อธิบายกฎ "patch ที่ที่มันถูกเรียกใช้ ไม่ใช่ที่ที่มันถูกนิยาม" ของ
`unittest.mock.patch()` พร้อมยกตัวอย่าง

**8.** Contract Testing แก้ปัญหาอะไรที่ unit test/integration test ทั่วไปแก้ไม่ได้

**9.** Load Testing ต่างจาก Integration Testing อย่างไร และทำไมถึงต้องรันคนละ
ความถี่กัน

**10.** `coverage report --fail-under=80` ทำหน้าที่อะไรใน CI pipeline และป้องกัน
ปัญหาอะไร

**11. (ขั้นสูง)** อธิบายว่าทำไม `DjangoObjectPermissions` (Part 045) ถึงคืน
`404 Not Found` แทน `403 Forbidden` เมื่อ user ไม่มีสิทธิ์ view object เลย
และเขียน test ยืนยันพฤติกรรมนี้แบบคร่าว ๆ

**12. (ขั้นสูง)** ออกแบบ test scenario สั้น ๆ (ไม่ต้องเขียนโค้ดเต็ม) ที่ยืนยันว่า
เมื่อ `ROTATE_REFRESH_TOKENS=True` และ `BLACKLIST_AFTER_ROTATION=True` refresh
token เดิมที่ถูกใช้ไปแล้วครั้งหนึ่งจะไม่สามารถใช้ซ้ำได้อีก

<details>
<summary>คลิกเพื่อดูเฉลยแบบย่อ</summary>

1. Authentication ตอบคำถาม "คุณเป็นใคร" ล้มเหลว → `401 Unauthorized` (ถ้ามี
   authenticator ที่รองรับ challenge) Permission ตอบคำถาม "คุณทำสิ่งนี้ได้ไหม"
   ล้มเหลว → `403 Forbidden`
2. เพราะความหมายต่างกันโดยธรรมชาติ: Permission ต้อง "ผ่านทุกกฎที่ตั้งไว้" (AND)
   ถึงจะอนุญาต ส่วน Authentication แค่ต้องการ "หาตัวตนให้เจอสักวิธีหนึ่งก็พอ"
   (ใครก็ได้ที่ authenticate สำเร็จก่อน ชนะ)
3. Access token อายุสั้นเพราะส่งไปกับทุก request (เสี่ยงรั่วไหลสูงกว่า) Refresh
   token อายุยาวแต่ส่งไปแค่ endpoint เดียว (เสี่ยงน้อยกว่า) การแยกจำกัดความเสียหาย
   เมื่อ token รั่วไหลให้อยู่ในกรอบเวลาสั้นที่สุดเท่าที่ทำได้
4. `force_authenticate()` ข้าม authentication class ทั้งหมด ทดสอบได้แค่ permission/
   business logic ส่วน JWT header จริงทดสอบทั้ง authentication class และ permission
   ครบ ไม่ควรใช้ `force_authenticate()` 100% เพราะจะไม่มีทาง จับบั๊กที่เกิดจาก
   authentication settings ผิดพลาดได้เลย
5. Unit test serializer เร็วกว่าและจับบั๊กด้าน validation logic ได้ชัดเจนกว่า
   Integration test ผ่าน `APIClient` ช้ากว่าแต่จับบั๊กจาก URL routing/permission/
   การประกอบทุก layer เข้าด้วยกันได้ ซึ่ง unit test จับไม่ได้
6. เพราะเมื่อเปิด pagination `response.data` เปลี่ยนจาก list กลายเป็น dict ที่มี
   key `count`/`next`/`previous`/`results` — `len(response.data)` จะนับจำนวน key
   (ได้ 4 เสมอ) ไม่ใช่จำนวนรายการ ต้องแก้เป็น `len(response.data["results"])`
7. เพราะ `from module import function` สร้าง reference ใหม่ในโมดูลที่ import ไป
   การ patch ที่โมดูลต้นทางจะไม่มีผลกับ reference ที่ถูก import ไปแล้ว ต้อง patch
   ที่ path ของโมดูลที่ **เรียกใช้งานจริง** เช่น patch
   `blog.api_views.notify_slack_on_publish` แทน `blog.services.notify_slack_on_publish`
   ถ้า `api_views.py` import ฟังก์ชันนั้นมาใช้ตรง ๆ
8. แก้ปัญหาที่ backend เปลี่ยนรูปร่าง response โดยไม่ได้ตั้งใจ (เพิ่ม/ลบ/เปลี่ยนชื่อ
   field) แล้วไม่มีใครรู้จนกว่า frontend/mobile team จะมาเจอปัญหาตอน integrate
   จริง — unit/integration test ทั่วไปตรวจแค่ค่าที่คาดหวัง ไม่ได้ตรวจ "รูปร่าง
   ทั้งหมด" เทียบกับสัญญาที่ประกาศไว้
9. Integration Testing ตอบคำถามเรื่องความถูกต้อง (correctness) ทีละ request
   Load Testing ตอบคำถามเรื่องประสิทธิภาพภายใต้ผู้ใช้จำนวนมากพร้อมกัน
   (performance/scalability) รันคนละความถี่กันเพราะ load test ใช้เวลานานกว่ามาก
   และต้องการ environment ที่ใกล้เคียง production มากกว่า integration test ที่
   ควรรันได้ทุกครั้งที่ commit
10. เป็น "coverage gate" ที่บังคับให้ CI job fail ถ้า test coverage รวมต่ำกว่า
    80% แม้ test ที่มีอยู่จะผ่านหมด ป้องกันไม่ให้ทีมเขียนฟีเจอร์ใหม่โดยไม่เขียน
    test คู่กันแล้ว merge เข้า main ได้อย่างเงียบ ๆ
11. เพื่อไม่เปิดเผยว่า object นั้น "มีอยู่จริง" ให้กับ user ที่ไม่มีสิทธิ์เห็นแม้แต่
    น้อย — ถ้าตอบ `403` แปลว่ายอมรับว่า object มีอยู่แค่ไม่ให้สิทธิ์ ในขณะที่ `404`
    ทำให้ผลลัพธ์เหมือนกับ "ไม่มี object นี้อยู่เลยตั้งแต่แรก" ปลอดภัยกว่าในแง่ information
    disclosure ทดสอบได้ด้วยการสร้าง user ที่ไม่มี object permission ใด ๆ เลยแล้วยิง
    `GET` ไปยัง object ที่มีอยู่จริง คาดหวัง `404` ไม่ใช่ `403`
12. เรียก `/api/token/refresh/` ด้วย refresh token ตัวหนึ่งครั้งแรก (ต้องสำเร็จและ
    ได้ refresh token ใหม่กลับมา) จากนั้นเรียก `/api/token/refresh/` **อีกครั้ง**
    ด้วย refresh token **ตัวเดิม** (ตัวที่เพิ่งใช้ไปแล้ว) ต้องคาดหวังว่าครั้งที่สอง
    ได้ `401` พร้อม error code ที่บ่งบอกว่า token ถูก blacklist แล้ว

</details>

### 500.5 แบบฝึกหัดใหญ่ปิดท้าย Phase 5: Blog REST API ฉบับสมบูรณ์

นี่คือแบบฝึกหัดที่รวบยอดทุกทักษะจาก Part 039-050 เข้าด้วยกัน โจทย์คือทำให้ `blog`
API ที่คุณสร้างมาตลอด Phase 5 มีคุณสมบัติครบทุกข้อต่อไปนี้ และมี test ครอบคลุมสูง
พอที่จะมั่นใจได้ว่าทุกฟีเจอร์ทำงานถูกต้องจริง

**ข้อกำหนดด้าน Model และ Serializer:**

1. มี model `Category` (name, slug) และ `Post` (title, slug, content, is_published,
   category FK, author FK, created_at, updated_at) พร้อม `PostSerializer` ที่มี
   custom validation อย่างน้อย 2 กฎ (field-level 1 กฎ, object-level 1 กฎ)
2. เขียน unit test สำหรับทุกกฎ validation ทั้งกรณีผ่านและไม่ผ่าน (`blog/tests/test_serializers.py`)

**ข้อกำหนดด้าน Authentication และ Permission:**

3. ตั้งค่า JWT authentication เต็มรูปแบบ: `/api/token/`, `/api/token/refresh/`,
   `/api/token/verify/`, และ endpoint logout ที่ blacklist refresh token
4. เขียน Custom Permission Class อย่างน้อย 1 ตัวที่รวม logic "เจ้าของหรือ staff"
   พร้อม unit test ครบทุก combination (safe/unsafe method × owner/non-owner × staff/non-staff)
5. เขียน integration test ที่ทดสอบ endpoint เดียวกันด้วยทั้ง 3 วิธี authenticate
   (`force_authenticate`, JWT header จริง, session login)

**ข้อกำหนดด้าน Filtering, Pagination, Versioning:**

6. เปิดใช้งาน `DjangoFilterBackend`, `SearchFilter`, `OrderingFilter` บน `PostViewSet`
   พร้อม custom pagination class (`page_size_query_param`, `max_page_size`)
7. เขียน test ครอบคลุมทุกฟีเจอร์ตามตารางในขั้นตอนที่ 495.6
8. ตั้งค่า API versioning (`AcceptHeaderVersioning` หรือแบบอื่นที่เลือก) อย่างน้อย
   2 version พร้อมอธิบายว่า response ต่างกันอย่างไรระหว่าง version

**ข้อกำหนดด้าน Documentation:**

9. ติดตั้ง `drf-spectacular` และเปิดใช้งาน Swagger UI ที่ `/api/docs/`
10. เขียน contract test อย่างน้อย 2 ตัว: ตรวจสอบว่า schema ถูกต้องตามมาตรฐาน
    OpenAPI และตรวจสอบว่า response ของ `/api/posts/{id}/` ตรงกับ schema component

**ข้อกำหนดด้าน Mocking และ CI:**

11. เพิ่มฟีเจอร์ที่เรียก external service อย่างน้อย 1 จุด (เช่น แจ้งเตือนเมื่อ
    เผยแพร่โพสต์) พร้อม mock test ทั้งด้วย `unittest.mock` และ `responses`
12. ตั้งค่า test settings แยก (`config/settings/test.py`) และเขียน GitHub Actions
    workflow ที่รัน test พร้อม coverage gate อย่างน้อย 80%

**เกณฑ์ตรวจสอบความสำเร็จ (Checklist ปิดท้าย Phase 5):**

- [ ] `python manage.py test --settings=config.settings.test` ผ่านทุกตัวไม่มี fail
- [ ] `coverage report --fail-under=80` ผ่าน (coverage รวมของแอป `blog` อย่างน้อย 80%)
- [ ] เข้า `/api/docs/` เห็น Swagger UI ที่แสดง endpoint ทั้งหมดถูกต้องครบ
- [ ] ทดสอบ obtain/refresh/verify/logout JWT ผ่าน `curl` ได้ครบวงจรจริง (ไม่ใช่แค่ผ่าน test)
- [ ] `?search=`, `?ordering=`, `?page=`, `?page_size=`, และ filter fields ทำงานถูกต้องเมื่อทดสอบผ่าน `curl` จริง
- [ ] Mock test ไม่มีตัวไหนยิง network request จริงออกไปข้างนอกเลย (ตรวจด้วยการปิด
      อินเทอร์เน็ตชั่วคราวแล้วรัน test suite — ต้องผ่านเหมือนเดิมทุกตัว)
- [ ] GitHub Actions workflow รันผ่านสีเขียวเมื่อ push ขึ้น repository จริง
- [ ] Contract test จับความไม่ตรงกันได้จริง (ทดลองเพิ่ม field ใหม่ใน serializer
      โดยตั้งใจไม่อัปเดต schema แล้วดูว่า test ตัวไหน fail)

**ระดับขั้นสูง (โบนัส)**: เพิ่ม endpoint ใหม่ `/api/posts/{id}/publish/` (custom
action ผ่าน `@action` decorator ที่เรียนจาก Part 044) ที่เปลี่ยน `is_published`
เป็น `True` โดยเฉพาะ (แยกจาก `PATCH` ทั่วไป) พร้อม permission ที่บังคับว่าต้องเป็น
เจ้าของหรือ staff เท่านั้น แล้วเขียน test ครบทั้ง unit test permission, integration
test ผ่าน endpoint, และ contract test ยืนยันว่า endpoint ใหม่นี้ถูกประกาศใน OpenAPI
schema ถูกต้องด้วย

### 500.6 คำนำสู่ Phase 6: Frontend Integration

ตลอด Phase 5 คุณสร้าง `blog` API ที่แข็งแกร่งระดับ production จริง — มี
authentication ที่ปลอดภัย, permission ที่ละเอียดถึงระดับ object, filtering/
pagination ที่พร้อมรับข้อมูลจำนวนมาก, เอกสารที่ generate อัตโนมัติ, และ test
coverage ที่ทำให้มั่นใจได้ว่าทุกอย่างทำงานถูกต้อง — แต่ตลอดทาง คุณทดสอบ API
ผ่าน `curl` และ Browsable API ของ DRF เท่านั้น ยังไม่มี **หน้าเว็บจริงที่ผู้ใช้
ทั่วไปจะเข้ามาใช้งาน** เชื่อมกับ API ตัวนี้เลยสักหน้า

**Phase 6: Frontend Integration (Part 051-058 | ขั้นตอนที่ 501-580)** จะพาคุณ:

- **Part 051**: เชื่อม Django กับ Bootstrap และ CSS Framework เพื่อทำให้หน้าเว็บ
  ที่ render จาก Django Template (Phase 1) ดูเป็นมืออาชีพโดยไม่ต้องเขียน CSS เอง
  ทั้งหมด
- **Part 052**: ใช้ JavaScript และ Fetch API เรียก `blog` API ที่สร้างมาตลอด
  Phase 5 โดยตรงจากฝั่ง client — นี่คือจุดที่ JWT authentication, filtering,
  pagination ที่เรียนมาทั้งหมดจะถูกใช้งานจริงจากฝั่ง frontend เป็นครั้งแรก
- **Part 053-054**: HTMX และ Alpine.js สำหรับสร้าง Interactive UI โดยเขียน
  JavaScript ให้น้อยที่สุดเท่าที่จะทำได้ (แนวทางที่ได้รับความนิยมมากขึ้นเรื่อย ๆ
  ในชุมชน Django ตั้งแต่ปี 2023 เป็นต้นมา)
- **Part 055-056**: เชื่อม Django (เป็น API Backend ล้วน ๆ ผ่าน DRF) เข้ากับ
  React และ Vue.js — สถาปัตยกรรมแบบ **decoupled frontend/backend** ที่ระบบใหญ่
  ระดับโลกจำนวนมากใช้จริง โดย Django ทำหน้าที่แค่ serve JSON ผ่าน endpoint ที่
  คุณ test ไว้อย่างละเอียดแล้วใน Part 050 นี้เอง
- **Part 057-058**: WebSockets ด้วย Django Channels สำหรับ real-time feature และ
  File Upload/Image Processing สำหรับระบบที่ต้องจัดการรูปภาพ/ไฟล์จากผู้ใช้

จุดที่สำคัญที่สุดที่ต้องจำไว้เมื่อเข้าสู่ Phase 6: **API ที่ทดสอบไว้ดีตลอด Phase 5
คือรากฐานที่ทำให้ Phase 6 ทำงานได้อย่างมั่นใจ** — เมื่อ frontend team (หรือตัวคุณ
เองที่สวมหมวก frontend) เรียก `/api/posts/` แล้วได้ผลลัพธ์ไม่ตรงกับที่คาด จะรู้
ทันทีว่าต้องกลับมาดู test suite ของ Part 050 ก่อนเสมอว่า contract ระหว่างสองฝั่ง
ยังตรงกันอยู่หรือไม่ นี่คือคุณค่าที่แท้จริงของการลงทุนเวลาเขียน test อย่างละเอียด
ตลอดทั้ง Part นี้

เตรียม `blog` API เวอร์ชันสมบูรณ์ที่ผ่านการทดสอบครบถ้วนของคุณไว้ให้พร้อม แล้วไปต่อ
กันที่ Phase 6 — ถึงเวลาทำให้ผู้ใช้จริงได้เห็นหน้าตาของสิ่งที่คุณสร้างมาตลอดครึ่งทาง
แรกของหลักสูตรนี้แล้ว!
