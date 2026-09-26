# Part 033: Permissions และ Groups

> **ขั้นตอนที่ 321-330 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions
>
> เป้าหมายของ Part นี้: เจาะลึกระบบ **Authorization** ของ Django หลังจากที่ Part 031-032
> พาคุณสร้างระบบ Authentication (login/logout) และ Custom User Model เรียบร้อยแล้ว
> Part นี้ตอบคำถามที่ Part 024 (ขั้นตอนที่ 234) ค้างไว้ว่า "แล้ว `PermissionRequiredMixin`
> ทำงานอย่างไรกันแน่ Permission มาจากไหน และ Group คืออะไร" คุณจะเข้าใจตั้งแต่กลไก
> `add`/`change`/`delete`/`view` permission ที่ Django สร้างอัตโนมัติ ไปจนถึงการสร้าง
> Custom Permission ของตัวเอง จัดกลุ่มสิทธิ์ผ่าน Group แบบมืออาชีพ ตรวจสอบ permission
> ทั้งใน view และ template และเข้าใจข้อจำกัดสำคัญที่สุดของระบบนี้ — Django permission
> เป็นระดับ **model** ไม่ใช่ระดับ **object** — ซึ่งเป็นเหตุผลที่ต้องมี `django-guardian`
> เข้ามาเสริม (เจาะลึกเต็มรูปแบบใน Part 038)

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 321: ภาพรวมระบบ Permission ของ Django
- ขั้นตอนที่ 322: `has_perm()`, `get_all_permissions()`, และ `get_group_permissions()`
- ขั้นตอนที่ 323: Groups — จัดกลุ่มสิทธิ์แทนการให้ทีละคน
- ขั้นตอนที่ 324: `@permission_required` และ `PermissionRequiredMixin` แบบเจาะลึกเต็มรูปแบบ
- ขั้นตอนที่ 325: Custom Permissions ผ่าน `Meta.permissions`
- ขั้นตอนที่ 326: เช็ค Permission ใน Template ด้วย `{% if perms %}`
- ขั้นตอนที่ 327: Object-Level Permission และเหตุผลที่ Django Default ไม่รองรับ
- ขั้นตอนที่ 328: Permission Caching — กับดักที่มือใหม่ (และมือเก่า) ตกบ่อยที่สุด
- ขั้นตอนที่ 329: พฤติกรรมพิเศษของ Superuser
- ขั้นตอนที่ 330: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 321: ภาพรวมระบบ Permission ของ Django

### 321.1 Authentication vs Authorization — อย่าสับสนสองคำนี้

ก่อนเข้าเนื้อหา ต้องแยกสองแนวคิดนี้ให้ชัดเจน เพราะเป็นรากฐานของทั้ง Phase 4:

| แนวคิด | คำถามที่ตอบ | ตัวอย่างใน Django |
|---|---|---|
| **Authentication** (การยืนยันตัวตน) | "คุณเป็นใคร" | ระบบ login, session, `request.user` (เรียนไปแล้วใน Part 031) |
| **Authorization** (การอนุญาตสิทธิ์) | "คุณทำสิ่งนี้ได้หรือไม่" | ระบบ Permission, Group ที่เรากำลังเรียนใน Part นี้ |

พูดง่าย ๆ: Authentication ตอบว่า "นี่คือคุณ narin แน่นอน" ส่วน Authorization ตอบว่า
"narin มีสิทธิ์ลบโพสต์นี้หรือเปล่า" ทั้งสองระบบทำงานร่วมกันเสมอ แต่เป็นคนละชั้น
(layer) กัน — `request.user.is_authenticated` ตอบคำถามแรก ส่วน
`request.user.has_perm(...)` ตอบคำถามที่สอง

### 321.2 Permission ถูกสร้างอัตโนมัติให้ทุก Model — 4 permission มาตรฐาน

จุดที่ทรงพลังที่สุดของระบบ Permission ใน Django คือ: **ทุกครั้งที่คุณสร้าง Model ใหม่
แล้วรัน `migrate` Django จะสร้าง Permission ให้อัตโนมัติ 4 รายการ** โดยไม่ต้องเขียน
โค้ดเพิ่มแม้แต่บรรทัดเดียว สมมติ Model `Post` จาก Part 023-024:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.urls import reverse


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name_plural = "categories"
        ordering = ["name"]

    def __str__(self):
        return self.name


class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "ฉบับร่าง"
        PUBLISHED = "published", "เผยแพร่แล้ว"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="posts",
    )
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True, related_name="posts"
    )
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("blog:post-detail", kwargs={"slug": self.slug})
```

เมื่อรัน `python manage.py migrate` Django จะสร้างแถวใหม่ในตาราง `auth_permission`
ให้ `Post` ทั้ง 4 รายการนี้:

| Codename | ชื่อเต็ม (permission string) | ความหมาย |
|---|---|---|
| `add_post` | `blog.add_post` | สร้าง `Post` ใหม่ได้ |
| `change_post` | `blog.change_post` | แก้ไข `Post` ที่มีอยู่ได้ |
| `delete_post` | `blog.delete_post` | ลบ `Post` ได้ |
| `view_post` | `blog.view_post` | ดู `Post` ได้ (เพิ่มเข้ามาตั้งแต่ Django 2.1) |

รูปแบบ permission string เต็มคือ **`<app_label>.<codename>`** เสมอ — `blog` คือ
`app_label` ของแอปที่ประกาศ Model นี้ (กำหนดใน `blog/apps.py` หรือชื่อโฟลเดอร์แอป)

### 321.3 กลไกเบื้องหลัง: `post_migrate` signal และ `create_permissions`

Permission เหล่านี้ไม่ได้ถูกสร้างโดยเวทมนตร์ แต่มาจากฟังก์ชัน
`django.contrib.auth.management.create_permissions` ที่ถูกเชื่อมเข้ากับ Django signal
**`post_migrate`** ทุกครั้งที่รัน `migrate` (ไม่ว่าจะมี schema เปลี่ยนแปลงจริงหรือไม่)
Django จะ:

1. สแกนทุก Model ที่ลงทะเบียนใน `INSTALLED_APPS`
2. สร้าง `ContentType` หนึ่งแถวต่อหนึ่ง Model (ถ้ายังไม่มี) ในตาราง `django_content_type`
3. สร้าง `Permission` 4 แถวต่อหนึ่ง Model โดยผูกกับ `ContentType` ของ Model นั้น
   (ผ่าน field `content_type`) — ยกเว้นจะปิดด้วย `default_permissions = []` ใน `Meta`
   (จะเรียนใน ขั้นตอนที่ 325)

ตรวจสอบผลลัพธ์จริงได้ผ่าน `python manage.py shell`:

```python
from django.contrib.auth.models import Permission
from django.contrib.contenttypes.models import ContentType
from blog.models import Post

ct = ContentType.objects.get_for_model(Post)
print(ct)
# blog | post

for perm in Permission.objects.filter(content_type=ct):
    print(perm.codename, "->", perm.name)

# add_post -> Can add post
# change_post -> Can change post
# delete_post -> Can delete post
# view_post -> Can view post
```

### 321.4 โครงสร้างตารางในฐานข้อมูลที่แท้จริง

การเข้าใจ schema จริง ๆ จะช่วยให้เข้าใจข้อจำกัดใน ขั้นตอนที่ 327 ได้ง่ายขึ้นมาก
ระบบ Permission ประกอบด้วย 4 ตารางหลัก:

```
django_content_type
├── id
├── app_label      (เช่น "blog")
└── model          (เช่น "post")

auth_permission
├── id
├── name           (เช่น "Can add post")
├── content_type_id  --> FK ไปยัง django_content_type
└── codename       (เช่น "add_post")

auth_user_user_permissions      (Many-to-Many: user <-> permission โดยตรง)
├── user_id
└── permission_id

auth_group_permissions          (Many-to-Many: group <-> permission)
├── group_id
└── permission_id

auth_user_groups                (Many-to-Many: user <-> group)
├── user_id
└── group_id
```

สังเกตให้ดี: **ไม่มีคอลัมน์ใดใน `auth_permission` หรือตาราง M2M ที่อ้างถึง object
เฉพาะเจาะจงเลย (เช่น `object_id` หรือ `post_id`)** — นี่คือรากฐานสำคัญของข้อจำกัดที่
เราจะพูดถึงเต็ม ๆ ใน ขั้นตอนที่ 327 ว่าทำไม Django permission ถึงเป็นระดับ **model**
(เช่น "แก้ไข Post ตัวไหนก็ได้ในระบบ") ไม่ใช่ระดับ **object** (เช่น "แก้ไขได้เฉพาะ
Post ที่ id=42")

### 321.5 ปิดหรือปรับแต่งการสร้าง Permission อัตโนมัติ

บาง Model อาจไม่ต้องการ permission มาตรฐานทั้ง 4 ตัว (เช่น Model ที่เป็น log
read-only ที่ไม่มีใครควรลบ) ใช้ `Meta.default_permissions` ปรับได้:

```python
class AuditLog(models.Model):
    action = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        default_permissions = ["view"]  # สร้างแค่ view_auditlog ตัวเดียว ไม่มี add/change/delete
```

หรือปิดทั้งหมดด้วย `default_permissions = []` แล้วค่อยกำหนด Custom Permission เอง
ทั้งหมดผ่าน `Meta.permissions` (จะเรียนละเอียดใน ขั้นตอนที่ 325)

### 321.6 ตรวจสอบ Permission ทั้งหมดผ่าน Django Admin

วิธีที่เร็วที่สุดในการดู Permission ทั้งหมดในระบบแบบภาพรวม (ไม่ต้องเปิด shell) คือ
เข้า Django Admin ที่ `/admin/auth/user/<id>/change/` แล้วเลื่อนไปที่ส่วน
"User permissions" — จะเห็น multi-select box ที่แสดง permission ทุกตัวในระบบในรูปแบบ
`app_label | model | Can xxx model` ครบทั้งหมด รวมถึง Custom Permission ที่คุณสร้างเอง

---

## ขั้นตอนที่ 322: `has_perm()`, `get_all_permissions()`, และ `get_group_permissions()`

### 322.1 `user.has_perm(perm, obj=None)` — เมธอดหลักที่ใช้บ่อยที่สุด

`has_perm()` คือเมธอดที่คุณจะเรียกใช้บ่อยที่สุดในทุก view, template, และ business
logic ที่ต้องเช็คสิทธิ์ รับ argument เป็น permission string เต็มรูปแบบ
(`"app_label.codename"`) และคืนค่า `True`/`False`:

```python
python manage.py shell
```

```python
from django.contrib.auth import get_user_model

User = get_user_model()
user = User.objects.get(username="narin")

user.has_perm("blog.add_post")
# False (narin ยังไม่มีสิทธิ์นี้)

user.is_authenticated
# True — คนละเรื่องกับ has_perm() นะ อย่าสับสน!
```

มอบสิทธิ์ให้โดยตรงแล้วลองใหม่:

```python
from django.contrib.auth.models import Permission

perm = Permission.objects.get(codename="add_post", content_type__app_label="blog")
user.user_permissions.add(perm)

# สำคัญมาก! ต้อง fetch user ใหม่จาก DB ก่อนเช็คซ้ำ ไม่งั้นจะเจอ cache ค้าง
# (รายละเอียดเต็มเรื่องนี้อยู่ใน ขั้นตอนที่ 328)
user = User.objects.get(username="narin")
user.has_perm("blog.add_post")
# True
```

### 322.2 `user.get_all_permissions(obj=None)` — ดึงสิทธิ์ทั้งหมดในคราวเดียว

คืนค่าเป็น `set` ของ permission string ทั้งหมดที่ user คนนั้นมี **ทั้งจากที่ให้ตรง
(`user_permissions`) และจาก Group ที่ user สังกัดอยู่รวมกัน**:

```python
user.get_all_permissions()
# {'blog.add_post'}

from django.contrib.auth.models import Group

editors = Group.objects.get(name="Editors")
change_perm = Permission.objects.get(codename="change_post", content_type__app_label="blog")
editors.permissions.add(change_perm)
user.groups.add(editors)

user = User.objects.get(username="narin")   # fetch ใหม่อีกครั้ง (กัน cache)
user.get_all_permissions()
# {'blog.add_post', 'blog.change_post'}
```

### 322.3 `user.get_group_permissions(obj=None)` — ดึงเฉพาะสิทธิ์ที่มาจาก Group

ต่างจาก `get_all_permissions()` ตรงที่ตัวนี้ **กรองเฉพาะสิทธิ์ที่มาจาก Group เท่านั้น**
ไม่รวมสิทธิ์ที่ให้ตรงกับ user:

```python
user.get_group_permissions()
# {'blog.change_post'}   <- มาจาก Group "Editors" เท่านั้น ไม่รวม add_post ที่ให้ตรง
```

### 322.4 `user.get_user_permissions(obj=None)` — ดึงเฉพาะสิทธิ์ที่ให้ตรงกับ user

เมธอดคู่กันที่มักถูกลืม คือด้านตรงข้ามของ `get_group_permissions()`:

```python
user.get_user_permissions()
# {'blog.add_post'}   <- มาจาก user.user_permissions โดยตรงเท่านั้น
```

### 322.5 `user.has_perms(perm_list, obj=None)` — เช็คหลายสิทธิ์พร้อมกันแบบ AND

```python
user.has_perms(["blog.add_post", "blog.change_post"])
# True เฉพาะเมื่อมีสิทธิ์ "ครบทุกตัว" ในลิสต์ (AND logic)

user.has_perms(["blog.add_post", "blog.delete_post"])
# False เพราะยังไม่มี blog.delete_post
```

### 322.6 ตารางสรุปเมธอดทั้งหมดของ `PermissionsMixin`

| เมธอด | คืนค่า | รวม Group หรือไม่ | ใช้เมื่อไหร่ |
|---|---|---|---|
| `has_perm(perm, obj=None)` | `bool` | ✅ | เช็คสิทธิ์เดียวใน view/template — ใช้บ่อยที่สุด |
| `has_perms(perm_list, obj=None)` | `bool` | ✅ | เช็คหลายสิทธิ์พร้อมกันแบบ AND |
| `get_all_permissions(obj=None)` | `set[str]` | ✅ | ดู/debug สิทธิ์ทั้งหมดที่มี |
| `get_user_permissions(obj=None)` | `set[str]` | ❌ (เฉพาะที่ให้ตรง) | Audit ว่าใครให้สิทธิ์ตรงกับ user คนไหนบ้าง |
| `get_group_permissions(obj=None)` | `set[str]` | ✅ (เฉพาะจาก Group) | ตรวจสอบว่า Group ให้สิทธิ์อะไรบ้าง |
| `has_module_perms(app_label)` | `bool` | ✅ | Django Admin ใช้ตัดสินว่าจะแสดงแอปนั้นในเมนู admin หรือไม่ |

**ข้อสังเกตสำคัญ**: พารามิเตอร์ `obj` ที่ปรากฏในทุกเมธอดข้างต้น มีไว้สำหรับ
Object-Level Permission ซึ่งค่า default backend ของ Django (`ModelBackend`) **ไม่รองรับ**
— ถ้าส่ง `obj` เข้าไป เมธอดเหล่านี้จะคืนค่าที่ไม่ตรงกับที่คาดหวังเสมอ (เราจะพิสูจน์และ
อธิบายเหตุผลแบบละเอียดใน ขั้นตอนที่ 327)

---

## ขั้นตอนที่ 323: Groups — จัดกลุ่มสิทธิ์แทนการให้ทีละคน

### 323.1 ปัญหาของการให้ Permission ทีละ User

ลองจินตนาการทีมงานเว็บบล็อกที่มีบรรณาธิการ (editor) 15 คน ทุกคนต้องมีสิทธิ์
`add_post`, `change_post`, `view_post`, และ `can_publish_post` (custom permission
ที่จะสร้างใน ขั้นตอนที่ 325) ถ้าให้สิทธิ์ทีละคนแบบ `user.user_permissions.add(...)`
เมื่อมีบรรณาธิการคนที่ 16 เข้ามาใหม่ ต้องเขียนโค้ด/คลิก admin เพิ่มสิทธิ์ 4 ครั้งซ้ำ
และถ้าวันหนึ่งต้องเพิ่มสิทธิ์ที่ 5 ให้ทุกคน ต้องไปแก้ทีละ user ทั้ง 16 คน — นี่คือ
ฝันร้ายด้าน maintainability

**Group** แก้ปัญหานี้โดยตรง: ผูก Permission เข้ากับ **Group** แทนที่จะผูกกับ User
โดยตรง แล้วให้ User เข้าร่วม Group นั้น การเปลี่ยนแปลงสิทธิ์ทำที่ Group แห่งเดียว
ส่งผลกับสมาชิกทุกคนทันที

### 323.2 สร้าง Group ผ่าน Django Admin

วิธีที่ง่ายที่สุดสำหรับทีมที่ไม่ใช่โปรแกรมเมอร์ (เช่น Project Manager ที่ดูแล
สิทธิ์ผู้ใช้) คือผ่าน `/admin/auth/group/add/`:

1. ตั้งชื่อ Group เช่น `Editors`
2. เลือก Permission จาก multi-select box (กด Ctrl/Cmd ค้างเพื่อเลือกหลายตัว)
3. กด Save

### 323.3 สร้าง Group ผ่าน Python Shell (สำหรับ Debug/ทดลอง)

```python
python manage.py shell
```

```python
from django.contrib.auth.models import Group, Permission

editors, created = Group.objects.get_or_create(name="Editors")
print(created)
# True (ถ้าเพิ่งสร้างครั้งแรก)

# ดึง permission ที่ต้องการทั้งหมดของ blog app
perm_codenames = ["add_post", "change_post", "view_post"]
perms = Permission.objects.filter(
    content_type__app_label="blog", codename__in=perm_codenames
)
editors.permissions.set(perms)   # .set() แทนที่ทั้งชุด ต่างจาก .add() ที่เพิ่มเข้าไป

# เพิ่ม user เข้า Group
from django.contrib.auth import get_user_model

User = get_user_model()
narin = User.objects.get(username="narin")
narin.groups.add(editors)

narin = User.objects.get(username="narin")   # fetch ใหม่กัน cache
narin.has_perm("blog.change_post")
# True — ได้สิทธิ์นี้มาจาก Group "Editors" ไม่ใช่ user_permissions ตรง ๆ
```

### 323.4 `.set()` vs `.add()` vs `.remove()` — ระวังพฤติกรรมที่ต่างกัน

| เมธอด | พฤติกรรม |
|---|---|
| `.add(perm1, perm2, ...)` | เพิ่ม permission เข้าไปในชุดเดิม (ไม่ลบของเก่า) |
| `.set([perm1, perm2, ...])` | **แทนที่ทั้งชุด** — permission เก่าที่ไม่อยู่ในลิสต์ใหม่จะถูกถอดออกทั้งหมด |
| `.remove(perm1)` | ถอด permission ตัวที่ระบุออกเพียงตัวเดียว |
| `.clear()` | ถอดสิทธิ์ทั้งหมดออกจาก Group/User |

ข้อผิดพลาดที่พบบ่อยที่สุด: ใช้ `.add()` ในสคริปต์ seed ข้อมูลที่รันซ้ำหลายรอบ (เช่น
ทุกครั้งที่ deploy) โดยตั้งใจจะ "sync" รายการสิทธิ์ให้ตรงกับโค้ด แต่ `.add()` ไม่ลบ
สิทธิ์เก่าที่ถูกเอาออกจากโค้ดแล้ว ทำให้ Group มีสิทธิ์เกินเจตนาสะสมไปเรื่อย ๆ —
สำหรับสคริปต์ seed ที่ต้องการ "sync ให้ตรงเป๊ะ" ให้ใช้ `.set()` เสมอ

### 323.5 สร้าง Group ผ่าน Management Command (นำไปใช้ซ้ำได้)

สำหรับทีมที่มีหลาย environment (dev/staging/production) การสร้าง Group ผ่าน admin
มือ ๆ ไม่ scale เขียนเป็น management command เพื่อรันซ้ำได้ทุกที่:

```python
# blog/management/commands/seed_groups.py
from django.contrib.auth.models import Group, Permission
from django.core.management.base import BaseCommand

GROUP_PERMISSIONS = {
    "Editors": [
        "blog.add_post",
        "blog.change_post",
        "blog.view_post",
        "blog.can_publish_post",
    ],
    "Authors": [
        "blog.add_post",
        "blog.change_post",
        "blog.view_post",
    ],
}


class Command(BaseCommand):
    help = "สร้าง/อัปเดต Group และ Permission มาตรฐานของระบบ blog ให้ตรงกับที่กำหนดไว้ในโค้ด"

    def handle(self, *args, **options):
        for group_name, perm_strings in GROUP_PERMISSIONS.items():
            group, created = Group.objects.get_or_create(name=group_name)

            perms = []
            for perm_string in perm_strings:
                app_label, codename = perm_string.split(".")
                try:
                    perms.append(
                        Permission.objects.get(
                            content_type__app_label=app_label, codename=codename
                        )
                    )
                except Permission.DoesNotExist:
                    self.stderr.write(
                        self.style.WARNING(f"ไม่พบ permission: {perm_string} (ข้ามไป)")
                    )

            group.permissions.set(perms)

            status = "สร้างใหม่" if created else "อัปเดต"
            self.stdout.write(
                self.style.SUCCESS(f"{status} Group '{group_name}' -> {len(perms)} permissions")
            )
```

รันได้ด้วย:

```bash
python manage.py seed_groups
```

```
สร้างใหม่ Group 'Editors' -> 4 permissions
สร้างใหม่ Group 'Authors' -> 3 permissions
```

### 323.6 ทางเลือกระดับมืออาชีพ: Data Migration แทน Management Command

Management command ต้องมีคนจำให้รันหลัง deploy ทุกครั้ง — ถ้าลืมรัน Group จะไม่ตรง
กับโค้ด ทางเลือกที่ปลอดภัยกว่าในโปรเจกต์ production คือ **Data Migration** ซึ่งรัน
อัตโนมัติเป็นส่วนหนึ่งของ `python manage.py migrate` เสมอ:

```bash
python manage.py makemigrations blog --empty --name seed_editor_group
```

```python
# blog/migrations/0004_seed_editor_group.py
from django.db import migrations


def create_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    Permission = apps.get_model("auth", "Permission")

    editors, _ = Group.objects.get_or_create(name="Editors")
    codenames = ["add_post", "change_post", "view_post", "can_publish_post"]
    perms = Permission.objects.filter(
        content_type__app_label="blog", codename__in=codenames
    )
    editors.permissions.set(perms)


def remove_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    Group.objects.filter(name="Editors").delete()


class Migration(migrations.Migration):
    dependencies = [
        ("blog", "0003_post_can_publish_post"),  # ต้องรันหลัง migration ที่สร้าง custom permission
    ]

    operations = [
        migrations.RunPython(create_groups, reverse_code=remove_groups),
    ]
```

**ข้อควรระวังสำคัญ**: data migration นี้ต้องมี `dependencies` ชี้ไปยัง migration
ที่สร้าง custom permission `can_publish_post` แล้ว (จาก ขั้นตอนที่ 325) ไม่เช่นนั้น
`Permission.objects.filter(codename="can_publish_post")` จะได้ผลลัพธ์ว่างเปล่าเพราะ
permission ยังไม่ถูกสร้างในฐานข้อมูล ณ จุดที่ migration นี้รัน

### 323.7 ตารางเปรียบเทียบ: ให้สิทธิ์ตรง User vs ผ่าน Group

| แนวทาง | ข้อดี | ข้อเสีย | เหมาะกับ |
|---|---|---|---|
| `user.user_permissions.add()` | ควบคุมละเอียดเป็นรายบุคคล | Maintenance ยากเมื่อทีมใหญ่ขึ้น, ลืมง่าย | สิทธิ์พิเศษเฉพาะบุคคลจริง ๆ ไม่กี่คน (เช่น สิทธิ์ debug ของ dev คนเดียว) |
| `Group` + `user.groups.add()` | จัดการรวมศูนย์ เปลี่ยนที่เดียวส่งผลทุกคน, สื่อความหมาย role ชัดเจน | ต้องออกแบบ Group ล่วงหน้าให้ครอบคลุม role จริง | ทีมงานที่มี role ชัดเจน (Editor, Author, Moderator, Support) — **แนะนำเป็นค่าเริ่มต้นเสมอ** |

**กฎเหล็กของหลักสูตรนี้**: ให้สิทธิ์ผ่าน **Group เสมอ** เว้นแต่มีเหตุผลเฉพาะเจาะจง
จริง ๆ ที่ต้องให้ user คนเดียว การให้สิทธิ์ตรงกับ user จำนวนมากคือสัญญาณของการออกแบบ
ระบบสิทธิ์ที่ไม่ดี

---

## ขั้นตอนที่ 324: `@permission_required` และ `PermissionRequiredMixin` แบบเจาะลึกเต็มรูปแบบ

> Part 024 (ขั้นตอนที่ 234.3-234.4) แนะนำ `PermissionRequiredMixin` แบบผิวเผินไว้แล้ว
> และบอกไว้ว่ารายละเอียดเต็มรูปแบบจะอยู่ใน Part นี้ — มาถึงเวลานั้นแล้ว

### 324.1 `@permission_required` สำหรับ Function-Based View

```python
# blog/views.py
from django.contrib.auth.decorators import login_required, permission_required
from django.shortcuts import get_object_or_404, redirect

from .models import Post


@login_required
@permission_required("blog.can_publish_post", raise_exception=True)
def publish_post(request, slug):
    post = get_object_or_404(Post, slug=slug)
    post.status = Post.Status.PUBLISHED
    post.save(update_fields=["status"])
    return redirect(post.get_absolute_url())
```

Signature เต็มของ decorator นี้คือ:

```python
permission_required(perm, login_url=None, raise_exception=False)
```

| Parameter | ความหมาย |
|---|---|
| `perm` | permission string เดียว (`"blog.add_post"`) หรือ list/tuple ของหลายสิทธิ์ (ตรวจแบบ AND ทั้งหมด) |
| `login_url` | URL ที่จะ redirect ไปถ้าไม่ผ่าน (default คือ `settings.LOGIN_URL`) — ใช้เมื่อ `raise_exception=False` |
| `raise_exception` | `False` (default) = redirect ไปหน้า login; `True` = ยิง `PermissionDenied` (แสดงหน้า 403) ทันที |

**ลำดับ decorator สำคัญมาก** เช่นเดียวกับลำดับ Mixin ใน Part 024 (ขั้นตอนที่ 233):
`@login_required` ต้องอยู่ **บนสุด** (outermost) เพราะ decorator ที่อยู่บนสุดจะ
ครอบทำงานก่อนเสมอเมื่อ request เข้ามา — ถ้าสลับตำแหน่งกัน ผู้ใช้ที่ยังไม่ล็อกอินจะ
เจอ error message ของ `permission_required` (ซึ่งพยายามเช็คสิทธิ์ของ
`AnonymousUser`) ก่อนที่จะถูกส่งไป login ให้เข้าใจง่าย ๆ

รองรับหลายสิทธิ์พร้อมกัน (AND logic):

```python
@permission_required(["blog.change_post", "blog.can_publish_post"], raise_exception=True)
def publish_post(request, slug):
    ...
```

### 324.2 `PermissionRequiredMixin` สำหรับ Class-Based View — โครงสร้างภายในทั้งหมด

`PermissionRequiredMixin` อยู่ใน `django.contrib.auth.mixins` และสืบทอดจาก
`AccessMixin` เช่นเดียวกับ `LoginRequiredMixin` และ `UserPassesTestMixin` ที่เรียนไป
ใน Part 024 โครงสร้างจริงของมัน (แบบย่อเพื่อความเข้าใจ) หน้าตาประมาณนี้:

```python
# ภายในของ django.contrib.auth.mixins (แสดงเพื่ออธิบาย ไม่ต้องเขียนเอง)
class PermissionRequiredMixin(AccessMixin):
    permission_required = None

    def get_permission_required(self):
        if self.permission_required is None:
            raise ImproperlyConfigured(
                f"{self.__class__.__name__} is missing the permission_required attribute."
            )
        if isinstance(self.permission_required, str):
            perms = (self.permission_required,)
        else:
            perms = self.permission_required
        return perms

    def has_permission(self):
        perms = self.get_permission_required()
        return self.request.user.has_perms(perms)

    def dispatch(self, request, *args, **kwargs):
        if not self.has_permission():
            return self.handle_no_permission()
        return super().dispatch(request, *args, **kwargs)
```

การรู้โครงสร้างนี้สำคัญมาก เพราะทำให้เห็นว่ามี **3 จุดที่ override ได้** ตามความ
ต้องการ: `permission_required` (attribute ง่ายสุด), `get_permission_required()`
(ถ้าต้องการ logic แบบ dynamic), และ `has_permission()` (ถ้าต้องการเปลี่ยนจาก AND
เป็น OR หรือ logic อื่นทั้งหมด)

### 324.3 ใช้งานพื้นฐาน: `permission_required` เป็น string เดียว

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import UpdateView

from .models import Post


class PostPublishView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["status"]
    template_name = "blog/post_publish.html"
    permission_required = "blog.can_publish_post"
    permission_denied_message = "คุณไม่มีสิทธิ์เผยแพร่โพสต์นี้"
    raise_exception = True
```

ลำดับ `(LoginRequiredMixin, PermissionRequiredMixin, UpdateView)` ยังคงต้องเป็นไป
ตามกฎ MRO จาก Part 024 (ขั้นตอนที่ 233): Mixin ทั้งคู่ต้องอยู่ก่อน `UpdateView` เสมอ

### 324.4 ใช้งานหลายสิทธิ์พร้อมกัน (AND logic ในตัว)

```python
class PostFeatureView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["is_featured"]
    template_name = "blog/post_feature.html"
    permission_required = ("blog.change_post", "blog.can_feature_post")
    raise_exception = True
```

`has_permission()` ค่า default จะเรียก `self.request.user.has_perms(perms)` ซึ่ง
ตรวจสอบว่า **ต้องมีครบทุกสิทธิ์ในลิสต์** (AND) ถึงจะผ่าน

### 324.5 Override `get_permission_required()` — logic แบบ dynamic ตาม request

บางกรณี permission ที่ต้องการขึ้นอยู่กับ HTTP method หรือข้อมูลใน URL:

```python
class PostManageView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["title", "content", "status"]
    template_name = "blog/post_manage.html"

    def get_permission_required(self):
        if self.request.method == "POST" and self.request.POST.get("status") == "published":
            # ถ้าพยายามเผยแพร่ผ่านฟอร์มนี้ ต้องมีสิทธิ์เพิ่ม
            return ("blog.change_post", "blog.can_publish_post")
        return ("blog.change_post",)
```

### 324.6 Override `has_permission()` — เปลี่ยนจาก AND เป็น OR

ถ้าต้องการอนุญาตเมื่อมี "สิทธิ์ใดสิทธิ์หนึ่ง" แทนที่จะต้องมีครบทุกตัว ต้อง override
`has_permission()` เองโดยตรง เพราะ default implementation เป็น AND เสมอ:

```python
class PostPublishOrFeatureView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["status", "is_featured"]
    template_name = "blog/post_publish.html"
    permission_required = ("blog.can_publish_post", "blog.can_feature_post")

    def has_permission(self):
        perms = self.get_permission_required()
        # OR logic: ผ่านถ้ามีสิทธิ์ "อย่างน้อยหนึ่งตัว" จากรายการ
        return any(self.request.user.has_perm(p) for p in perms)
```

### 324.7 Override `handle_no_permission()` — ปรับพฤติกรรมเมื่อไม่ผ่าน

เมธอดนี้สืบทอดมาจาก `AccessMixin` (เหมือนที่ `AuthorRequiredMixin` เคย override ใน
Part 024 ขั้นตอนที่ 234.2) ใช้ปรับ response เมื่อ permission ไม่ผ่าน เช่น ส่ง JSON
กลับแทน HTML สำหรับ API endpoint:

```python
from django.http import JsonResponse


class PostPublishApiView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["status"]
    permission_required = "blog.can_publish_post"

    def handle_no_permission(self):
        if self.request.headers.get("x-requested-with") == "XMLHttpRequest":
            return JsonResponse(
                {"error": "permission_denied", "detail": "คุณไม่มีสิทธิ์เผยแพร่โพสต์"},
                status=403,
            )
        return super().handle_no_permission()
```

### 324.8 ตารางเปรียบเทียบ: `@permission_required` (FBV) vs `PermissionRequiredMixin` (CBV)

| คุณสมบัติ | `@permission_required` | `PermissionRequiredMixin` |
|---|---|---|
| ใช้กับ | Function-Based View | Class-Based View |
| กำหนดสิทธิ์ | argument แรกของ decorator | attribute `permission_required` |
| หลายสิทธิ์ | ส่ง list/tuple เข้า argument | ส่ง tuple ให้ attribute (AND เสมอ ยกเว้น override) |
| Redirect เมื่อไม่ผ่าน | `login_url` parameter | attribute `login_url` (สืบทอดจาก `AccessMixin`) |
| ยิง 403 แทน redirect | `raise_exception=True` | attribute `raise_exception = True` |
| Custom logic (OR, dynamic) | ต้องเขียน decorator เอง | override `has_permission()` หรือ `get_permission_required()` |
| Custom response เมื่อไม่ผ่าน | ต้องเขียน decorator ครอบเพิ่ม | override `handle_no_permission()` โดยตรง |

### 324.9 กฎการรวมกับ Mixin ตัวอื่นที่เรียนมาใน Part 024

ทบทวนกฎทองจาก Part 024 (ขั้นตอนที่ 233): **Mixin ทุกตัวต้องอยู่ทางซ้ายของ base View
class เสมอ** และเรียงจาก "เช็คทั่วไปที่สุด" ไปหา "เฉพาะเจาะจงที่สุด" ดังนั้นเมื่อ
รวม `LoginRequiredMixin`, `AuthorRequiredMixin` (จาก Part 024), และ
`PermissionRequiredMixin` เข้าด้วยกัน ลำดับที่ถูกต้องคือ:

```python
class PostAdminEditView(
    LoginRequiredMixin,        # 1. เช็คทั่วไปที่สุด: ล็อกอินหรือยัง
    PermissionRequiredMixin,   # 2. เช็คเฉพาะทาง: มี role/permission ที่ต้องการหรือไม่
    UpdateView,                # 3. base View อยู่ขวาสุดเสมอ
):
    model = Post
    fields = ["title", "content", "status", "category"]
    permission_required = "blog.change_post"
```

---

## ขั้นตอนที่ 325: Custom Permissions ผ่าน `Meta.permissions`

### 325.1 เมื่อ 4 permission มาตรฐานไม่พอ

`add_post`, `change_post`, `delete_post`, `view_post` ครอบคลุมแค่การกระทำ CRUD
พื้นฐาน แต่ธุรกิจจริงมักมีกฎที่ซับซ้อนกว่านั้น เช่น "เผยแพร่โพสต์" ไม่ใช่แค่การ
"แก้ไข" ธรรมดา (`change_post`) แต่เป็นการกระทำพิเศษที่ควรจำกัดเฉพาะบรรณาธิการ
เท่านั้น — ทั้งที่ในทางเทคนิคมันก็คือการเรียก `.save()` แก้ field `status` เหมือนกัน

Django รองรับการประกาศ **Custom Permission** เพิ่มเติมผ่าน `Meta.permissions`:

```python
# blog/models.py
class Post(models.Model):
    # ... fields เดิมทั้งหมด ...

    class Meta:
        ordering = ["-created_at"]
        permissions = [
            ("can_publish_post", "Can publish post"),
            ("can_feature_post", "Can feature post on homepage"),
        ]
```

### 325.2 สร้าง Migration และตรวจสอบผลลัพธ์

```bash
python manage.py makemigrations blog
```

```
Migrations for 'blog':
  blog/migrations/0003_alter_post_options.py
    - Change Meta options on post
```

เนื้อหาไฟล์ migration ที่ได้:

```python
# blog/migrations/0003_alter_post_options.py
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [
        ("blog", "0002_post_category"),
    ]

    operations = [
        migrations.AlterModelOptions(
            name="post",
            options={
                "ordering": ["-created_at"],
                "permissions": [
                    ("can_publish_post", "Can publish post"),
                    ("can_feature_post", "Can feature post on homepage"),
                ],
            },
        ),
    ]
```

**ข้อสังเกตสำคัญ**: `AlterModelOptions` ไม่ได้แก้ไข schema ของตาราง `blog_post`
ในฐานข้อมูลเลย (ไม่มีคอลัมน์ใหม่ ไม่มี `ALTER TABLE`) เพราะ permission ไม่ใช่ field
ของ Model แต่ Django ยังคง generate migration ให้เพื่อเก็บ **model state** ให้ตรงกับ
โค้ดเสมอ (ตามหลักการ migration history ที่ต้อง reproducible) ตัว migration นี้เพียง
บันทึกการเปลี่ยนแปลงไว้ ส่วนการสร้างแถวจริงใน `auth_permission` เกิดขึ้นตอนรัน
`migrate` ผ่าน signal `post_migrate` ตามที่อธิบายไว้ใน ขั้นตอนที่ 321.3

รัน migrate แล้วตรวจสอบ:

```bash
python manage.py migrate
```

```python
python manage.py shell
```

```python
from django.contrib.auth.models import Permission

Permission.objects.filter(content_type__app_label="blog", codename="can_publish_post")
# <QuerySet [<Permission: blog | post | Can publish post>]>
```

### 325.3 ใช้งานเหมือน permission ปกติทุกประการ

จุดที่สวยงามที่สุดของ custom permission คือ **มันทำงานเหมือน permission มาตรฐาน
ทุกประการ** — ใช้กับ `has_perm()`, `PermissionRequiredMixin`, `{% if perms %}`,
Group ได้เหมือนกันหมด ไม่ต้องเขียนกลไกพิเศษเพิ่ม:

```python
user.has_perm("blog.can_publish_post")

class PostPublishView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    permission_required = "blog.can_publish_post"
    ...
```

### 325.4 Custom Permission ที่ไม่ผูกกับ Model จริง (Global/App-level Permission)

บางครั้งสิทธิ์ที่ต้องการไม่เกี่ยวกับ Model ใดโดยเฉพาะเลย เช่น "สิทธิ์ดู Analytics
Dashboard ของทั้งเว็บไซต์" เนื่องจาก `Permission` model บังคับต้องผูกกับ
`ContentType` เสมอ (ดู schema ใน ขั้นตอนที่ 321.4) แนวทางที่ทีมมืออาชีพนิยมใช้คือ
สร้าง Model เปล่าที่ `managed = False` (ไม่มีตารางจริงในฐานข้อมูล) ไว้เป็น "ที่แขวน"
permission ระดับแอปโดยเฉพาะ:

```python
# core/models.py
from django.db import models


class AppPermissions(models.Model):
    """
    Model เปล่าที่ไม่มีตารางจริงในฐานข้อมูล (managed=False) สร้างขึ้นเพียงเพื่อเป็น
    'ที่แขวน' Custom Permission ระดับแอปที่ไม่ผูกกับ Model ใดโดยเฉพาะ
    """

    class Meta:
        managed = False
        default_permissions = ()  # ปิด add/change/delete/view ที่ไม่มีความหมายสำหรับ model นี้
        permissions = [
            ("can_view_analytics_dashboard", "Can view analytics dashboard"),
            ("can_export_reports", "Can export reports"),
        ]
```

ใช้งานเหมือนเดิมทุกประการ: `user.has_perm("core.can_view_analytics_dashboard")`

### 325.5 ข้อควรระวัง: การตั้งชื่อ Codename ต้องไม่ชนกัน

Codename ต้อง **unique ภายใน `content_type` เดียวกัน** (Django enforce ด้วย
`unique_together = [["content_type", "codename"]]` ใน `Permission.Meta`) แต่
`codename` เดียวกันสามารถใช้ซ้ำได้ข้าม Model คนละตัว (เพราะ `content_type` ต่างกัน)
เช่น `Post` และ `Page` ต่างก็มี `can_publish` ได้พร้อมกัน โดยจะกลายเป็น
`blog.can_publish` และ `cms.can_publish` ซึ่งเป็นคนละ permission กันโดยสิ้นเชิง

### 325.6 ตารางสรุปแนวทางตั้งชื่อ Custom Permission ที่แนะนำ

| รูปแบบ | ตัวอย่าง | เหมาะกับ |
|---|---|---|
| `can_<verb>_<noun>` | `can_publish_post`, `can_approve_comment` | การกระทำเฉพาะทางที่ผูกกับ Model ที่มีอยู่แล้ว |
| `<verb>_<noun>` (ไม่มี `can_`) | `moderate_post`, `feature_post` | สั้นกว่า สอดคล้องกับ pattern มาตรฐาน `add_post` |
| `can_<action>_<resource>` บน dummy model | `can_view_analytics_dashboard` | สิทธิ์ระดับแอปที่ไม่ผูกกับ Model จริง |

หลักสูตรนี้ใช้รูปแบบ `can_<verb>_<noun>` เป็นหลัก เพราะอ่านเข้าใจง่ายและแยกจาก
permission มาตรฐาน (`add`/`change`/`delete`/`view`) ได้ชัดเจนด้วยสายตา

---

## ขั้นตอนที่ 326: เช็ค Permission ใน Template ด้วย `{% if perms %}`

### 326.1 `perms` มาจากไหน — Context Processor

ตัวแปร `perms` ที่ใช้ใน template ไม่ได้เกิดขึ้นเอง แต่มาจาก **context processor**
ชื่อ `django.contrib.auth.context_processors.auth` ซึ่งถูกเปิดใช้งานเป็นค่า default
อยู่แล้วเมื่อสร้างโปรเจกต์ด้วย `django-admin startproject` (ตรวจสอบได้ใน
`settings.py`):

```python
# settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        # ...
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",  # <-- ตัวนี้ให้ทั้ง `user` และ `perms`
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

Context processor ตัวนี้ให้ตัวแปร 2 ตัวกับทุก template ที่ render ผ่าน
`render(request, ...)` หรือ CBV มาตรฐาน: `user` (คือ `request.user`) และ `perms`
(instance ของ `django.contrib.auth.context_processors.PermWrapper`)

**ข้อควรระวัง**: `perms` ใช้ได้เฉพาะเมื่อ template render ผ่าน `RequestContext`
(ซึ่งเกิดขึ้นอัตโนมัติเมื่อใช้ `render()` shortcut หรือ CBV ของ Django) ถ้า render
ด้วย `Template(...).render(Context({...}))` แบบดิบ ๆ โดยไม่ผ่าน request จะไม่มี
`perms` ให้ใช้เลย

### 326.2 Syntax การใช้งานพื้นฐาน

```html
{% if perms.blog.add_post %}
    <a href="{% url 'blog:post-create' %}" class="btn btn-primary">เขียนโพสต์ใหม่</a>
{% endif %}

{% if perms.blog.can_publish_post %}
    <button class="btn btn-success">เผยแพร่โพสต์นี้</button>
{% endif %}
```

`perms.blog.add_post` แปลตรง ๆ ว่า `user.has_perm("blog.add_post")` — `PermWrapper`
ทำหน้าที่แปลง attribute access (`.blog.add_post`) ที่ Django Template Language
รองรับ ให้กลายเป็นการเรียก `has_perm()` เบื้องหลังแบบ **lazy** (คำนวณเฉพาะตอนที่ถูก
เข้าถึงจริงใน template เท่านั้น ไม่คำนวณสิทธิ์ทั้งหมดล่วงหน้า)

### 326.3 เช็คแบบ "มีสิทธิ์อะไรก็ได้ในแอปนี้"

```html
{% if perms.blog %}
    <li><a href="{% url 'blog:dashboard' %}">จัดการบล็อก</a></li>
{% endif %}
```

`perms.blog` (ไม่มี `.` ตามหลังอีก) จะเป็น `True` ถ้า user มี permission **อย่างน้อย
หนึ่งตัว** ในแอป `blog` ไม่ว่าจะเป็นตัวไหนก็ตาม — เบื้องหลังคือการเรียก
`user.has_module_perms("blog")`

### 326.4 ตัวอย่างเต็มรูปแบบ: Navbar ที่ปรับตาม Role

```html
<!-- templates/blog/_navbar.html -->
<nav class="navbar">
    <a href="{% url 'blog:post-list' %}">หน้าแรก</a>

    {% if user.is_authenticated %}
        {% if perms.blog.add_post %}
            <a href="{% url 'blog:post-create' %}">เขียนโพสต์ใหม่</a>
        {% endif %}

        {% if perms.blog.can_publish_post %}
            <a href="{% url 'blog:publish-queue' %}">คิวรอเผยแพร่</a>
        {% endif %}

        {% if perms.blog.delete_post %}
            <a href="{% url 'blog:trash' %}" class="text-danger">ถังขยะ</a>
        {% endif %}

        <a href="{% url 'account_logout' %}">ออกจากระบบ ({{ user.username }})</a>
    {% else %}
        <a href="{% url 'account_login' %}">เข้าสู่ระบบ</a>
    {% endif %}
</nav>
```

### 326.5 ข้อจำกัดสำคัญ: `perms` เช็คได้แค่ระดับ Model ไม่ใช่ระดับ Object

จุดที่มือใหม่มักเข้าใจผิดคือคิดว่า `{% if perms.blog.change_post %}` จะรู้ว่า
"โพสต์ตัวที่กำลังแสดงอยู่นี้" แก้ไขได้หรือไม่ แต่จริง ๆ แล้วมันตอบแค่ว่า "user คนนี้
มีสิทธิ์แก้ไข Post **ตัวไหนก็ได้** ในระบบหรือไม่" เท่านั้น ไม่รู้จัก object ที่อยู่ใน
context เลย:

```html
<!-- ผิดความคาดหวัง! ไม่ได้เช็คว่าแก้ "โพสต์นี้" ได้หรือไม่ -->
{% if perms.blog.change_post %}
    <a href="{% url 'blog:post-update' post.slug %}">แก้ไข</a>
{% endif %}
```

ถ้าต้องการเช็คว่า "user นี้เป็นเจ้าของโพสต์นี้หรือไม่" (ไม่ใช่แค่มีสิทธิ์แก้ไขทั่วไป)
ต้องเขียน logic เพิ่มเอง เช่นส่งค่า boolean จาก view เข้ามาใน context โดยตรง หรือใช้
template tag ที่เขียนเอง — เราจะเจาะลึกวิธีแก้ปัญหานี้อย่างถูกต้องด้วย
`django-guardian` ใน ขั้นตอนที่ 327 และ Part 038

---

## ขั้นตอนที่ 327: Object-Level Permission และเหตุผลที่ Django Default ไม่รองรับ

### 327.1 ทบทวนปัญหาจาก ขั้นตอนที่ 326.5

สมมติสถานการณ์จริง: เว็บบล็อกมีนักเขียน (author) 3 คน — narin, sofia, และ kai —
ทุกคนอยู่ใน Group "Authors" ที่มีสิทธิ์ `blog.change_post` เหมือนกันหมด (เพื่อให้
แก้ไขโพสต์ **ของตัวเอง** ได้) คำถามคือ: ถ้า narin พยายามยิง
`POST /blog/kai-post/edit/` (โพสต์ของ kai) จะเกิดอะไรขึ้น?

```python
narin.has_perm("blog.change_post")
# True!
```

**Django ตอบ `True` แม้ว่า narin ไม่ใช่เจ้าของโพสต์นั้นเลย** เพราะ permission
`blog.change_post` หมายถึง "แก้ไข Post ตัวไหนก็ได้ในระบบ" ไม่มีแนวคิดเรื่อง "โพสต์
ตัวไหน" อยู่ในระบบ permission เริ่มต้นเลย — นี่คือสิ่งที่เรียกว่า Django permission
เป็น **Model-Level Permission** ไม่ใช่ **Object-Level (Row-Level) Permission**

### 327.2 ทำไม Django Default ถึงไม่รองรับ Object-Level ในตัว

ย้อนกลับไปดู schema ในตาราง `auth_permission` และ `auth_user_user_permissions`
จาก ขั้นตอนที่ 321.4 อีกครั้ง:

```
auth_permission
├── id
├── name
├── content_type_id   <-- ผูกกับ "Model" (เช่น blog.Post) เท่านั้น
└── codename

auth_user_user_permissions
├── user_id
└── permission_id     <-- ไม่มีคอลัมน์ object_id / post_id เลย
```

ไม่มีที่ไหนในโครงสร้างนี้ที่เก็บว่า "permission นี้ใช้ได้กับ object แถวไหนบ้าง"
เพราะระบบนี้ถูกออกแบบมาให้เรียบง่ายและครอบคลุมกรณีทั่วไปที่สุด (role-based
authorization) การเพิ่มความสามารถ per-object จะทำให้ core ของ Django ซับซ้อนขึ้นและ
เกิด overhead กับ 99% ของโปรเจกต์ที่ไม่ต้องการมันเลย

ทีมพัฒนา Django จึงเลือกทางออกที่ฉลาด: **ออกแบบ hook ไว้ให้ แต่ไม่ implement เอง**
สังเกตว่าทุกเมธอดที่เราเรียนมา — `has_perm(perm, obj=None)`, `get_all_permissions
(obj=None)` — **มี parameter `obj` รองรับอยู่แล้วตั้งแต่แรก** เพียงแต่
`ModelBackend` (backend เริ่มต้น) เลือกที่จะไม่ทำอะไรกับมัน:

```python
# พฤติกรรมจริงของ ModelBackend (django.contrib.auth.backends) เมื่อส่ง obj เข้าไป
narin.has_perm("blog.change_post", kai_post)
# False เสมอ! เพราะ ModelBackend คืน set() ว่างทันทีที่เห็นว่า obj is not None
# (ไม่ใช่เพราะเช็คแล้วไม่ผ่าน แต่เพราะ "ปฏิเสธที่จะตอบ" คำถามระดับ object เลย)
```

นี่คือพฤติกรรมที่เอกสารทางการของ Django ระบุไว้ชัดเจน: "Although Django provides
object permission support in the model permission backend, there is no
implementation for it in the core" — ระบบมีโครงไว้รองรับ แต่ไม่มี logic จริง
ให้ในตัว ต้องพึ่ง backend เสริมจากภายนอก

### 327.3 ทางออก: `django-guardian` — Object-Level Permission Backend ยอดนิยม

`django-guardian` คือ third-party package ที่ implement Object-Level Permission
เต็มรูปแบบ โดยเพิ่มตาราง `guardian_userobjectpermission` และ
`guardian_groupobjectpermission` ที่มีคอลัมน์ `object_pk` เก็บ id ของ object
เฉพาะเจาะจงจริง ๆ (ต่างจาก `auth_permission` ที่ไม่มีคอลัมน์นี้เลย)

ติดตั้งเบื้องต้น (รายละเอียดเต็มรูปแบบอยู่ใน **Part 038**):

```bash
pip install django-guardian
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "guardian",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",  # ยังต้องมีตัวนี้ไว้ (สำหรับ permission ปกติ)
    "guardian.backends.ObjectPermissionBackend",  # เพิ่มตัวนี้เข้ามาสำหรับ object-level
]

ANONYMOUS_USER_NAME = None  # ปิด anonymous user ของ guardian ถ้าไม่ต้องการใช้
```

ตัวอย่างการใช้งานคร่าว ๆ (เพื่อให้เห็นภาพว่าแก้ปัญหาใน ขั้นตอนที่ 327.1 ได้จริง):

```python
from guardian.shortcuts import assign_perm

kai_post = Post.objects.get(slug="kai-post")

# มอบสิทธิ์แก้ไข "เฉพาะโพสต์ตัวนี้" ให้ narin (ไม่ใช่ทุกโพสต์ในระบบ!)
assign_perm("blog.change_post", narin, kai_post)

narin.has_perm("blog.change_post", kai_post)
# True — เพราะ ObjectPermissionBackend ตรวจสอบ object_pk ที่ตรงกันจริง

sofia_post = Post.objects.get(slug="sofia-post")
narin.has_perm("blog.change_post", sofia_post)
# False — narin ไม่ได้รับสิทธิ์นี้สำหรับโพสต์ของ sofia
```

### 327.4 ทางเลือกอื่นที่ไม่ต้องพึ่ง Library ภายนอก

ก่อนจะไปถึง `django-guardian` เต็มรูปแบบใน Part 038 ควรรู้ว่ามีทางเลือกที่ง่ายกว่า
สำหรับกรณี "เจ้าของ object" ซึ่งเป็นกรณีที่พบบ่อยที่สุด นั่นคือ `AuthorRequiredMixin`
ที่เราเขียนเองใน Part 024 (ขั้นตอนที่ 232) — เปรียบเทียบสองแนวทาง:

| แนวทาง | เหมาะกับ | ข้อจำกัด |
|---|---|---|
| Custom Mixin เทียบ field เจ้าของ (`obj.author == request.user`) | กรณี "เจ้าของ object เท่านั้นที่แก้ไขได้" ซึ่งเป็นกรณีส่วนใหญ่ | ต้องเขียนเองทุกจุด, ไม่ผูกกับระบบ Permission/Group มาตรฐาน, ไม่โผล่ใน `{% if perms %}` |
| `django-guardian` | กรณีซับซ้อนกว่า เช่น "มอบสิทธิ์แก้ไขโพสต์นี้ให้ user คนอื่นที่ไม่ใช่เจ้าของ" หรือ "editor คนหนึ่งดูแลเฉพาะบาง category" | ต้องติดตั้ง package เพิ่ม, มี query overhead มากกว่า, ต้องบริหารตาราง object permission เพิ่มเติม |

**คำแนะนำ**: ถ้าโจทย์คือ "เจ้าของเท่านั้นที่แก้ไขได้" (ownership) ใช้ Custom Mixin
แบบ Part 024 ก็เพียงพอและเร็วกว่า ถ้าโจทย์ซับซ้อนกว่านั้น (มอบสิทธิ์ให้คนอื่นเป็น
ราย object, หรือต้องแสดงผลผ่าน `{% if perms %}` แบบ per-object) ค่อยพิจารณา
`django-guardian` — เราจะเรียนวิธีตัดสินใจและ implementation แบบเต็มใน **Part 038:
Row-Level Permissions และ Object-Level Permission**

---

## ขั้นตอนที่ 328: Permission Caching — กับดักที่มือใหม่ (และมือเก่า) ตกบ่อยที่สุด

### 328.1 ทำไม Django ต้อง Cache Permission

การเช็ค `has_perm()` แต่ละครั้งต้อง query ฐานข้อมูลเพื่อดึงสิทธิ์ทั้งหมดของ user
(ทั้งจาก `user_permissions` และผ่านทุก Group ที่สังกัด) ถ้า view หนึ่งเรียก
`has_perm()` หลายครั้ง (เช่นใน template ที่เช็ค `perms.blog.xxx` ซ้ำหลายจุด) การ
query ซ้ำทุกครั้งจะสิ้นเปลืองมาก Django จึง**cache ผลลัพธ์ไว้ในตัว object `user`
เอง** โดยเก็บเป็น attribute พิเศษที่ขึ้นต้นด้วย underscore

### 328.2 Attribute ที่ใช้เก็บ Cache

`ModelBackend` (ใน `django.contrib.auth.backends`) เก็บ cache ไว้ 3 ตัวแปรแยกกัน
บน instance ของ user:

| Attribute | เก็บผลลัพธ์ของ |
|---|---|
| `user._user_perm_cache` | `get_user_permissions()` (สิทธิ์ที่ให้ตรง) |
| `user._group_perm_cache` | `get_group_permissions()` (สิทธิ์จาก Group) |
| `user._perm_cache` | `get_all_permissions()` (รวมทั้งสองแบบข้างต้น) |

ตรวจสอบได้จริงใน shell:

```python
user = User.objects.get(username="narin")
hasattr(user, "_perm_cache")
# False (ยังไม่เคยเช็คอะไรเลย)

user.has_perm("blog.add_post")
hasattr(user, "_perm_cache")
# True — ถูกสร้างขึ้นทันทีที่เรียก has_perm() ครั้งแรก

user._perm_cache
# {'blog.add_post', 'blog.change_post', ...}
```

### 328.3 กับดักที่ 1: มอบสิทธิ์กลางคันในฟังก์ชันเดียวกันแล้วเช็คซ้ำ

นี่คือ bug ที่พบบ่อยที่สุดในโค้ดจริง — ฟังก์ชันที่ทั้ง "มอบสิทธิ์" และ "เช็คสิทธิ์"
ให้กับ user object ตัวเดียวกันในการเรียกครั้งเดียว:

```python
def upgrade_to_editor_and_notify(request, user_id):
    user = User.objects.get(pk=user_id)

    user.has_perm("blog.can_publish_post")
    # สมมติคืน False -> Django cache "False" (คือ permission ไม่อยู่ใน set) ไว้ใน user._perm_cache แล้ว

    editors = Group.objects.get(name="Editors")
    user.groups.add(editors)   # ให้สิทธิ์เพิ่มจริงในฐานข้อมูลแล้ว!

    if user.has_perm("blog.can_publish_post"):   # <-- BUG: ยังคืน False!
        send_welcome_email(user)
    # ผลลัพธ์: อีเมลต้อนรับไม่ถูกส่ง ทั้งที่สิทธิ์ถูกให้ไปแล้วจริง ๆ ในฐานข้อมูล
```

สาเหตุ: บรรทัดที่ 3 (`user.has_perm(...)` ครั้งแรก) ทำให้ `ModelBackend` สร้าง
`user._perm_cache` ขึ้นมาโดยอิง query ฐานข้อมูล ณ เวลานั้น (ก่อนที่จะเพิ่ม Group)
เมื่อเรียก `has_perm()` ครั้งที่สอง Django **ไม่ query ฐานข้อมูลใหม่** แต่อ่านจาก
`_perm_cache` ที่ตั้งไว้แล้วตั้งแต่ก่อนหน้านี้ ผลคือได้ค่าเก่าที่ล้าสมัย (stale)

### 328.4 วิธีแก้: ลบ Cache หรือดึง Instance ใหม่

**วิธีที่ 1 — ลบ cache attribute ด้วยมือ**:

```python
def upgrade_to_editor_and_notify(request, user_id):
    user = User.objects.get(pk=user_id)
    user.has_perm("blog.can_publish_post")

    editors = Group.objects.get(name="Editors")
    user.groups.add(editors)

    # ลบ cache ทั้ง 3 ตัวก่อนเช็คซ้ำ
    for cache_attr in ("_perm_cache", "_user_perm_cache", "_group_perm_cache"):
        if hasattr(user, cache_attr):
            delattr(user, cache_attr)

    if user.has_perm("blog.can_publish_post"):   # ตอนนี้ query ใหม่ -> True ถูกต้อง
        send_welcome_email(user)
```

**วิธีที่ 2 — ดึง instance ใหม่จากฐานข้อมูล (ง่ายและปลอดภัยกว่า แนะนำเป็นค่าเริ่มต้น)**:

```python
def upgrade_to_editor_and_notify(request, user_id):
    user = User.objects.get(pk=user_id)
    user.has_perm("blog.can_publish_post")

    editors = Group.objects.get(name="Editors")
    user.groups.add(editors)

    user = User.objects.get(pk=user.pk)   # instance ใหม่ = ไม่มี cache เก่าติดมาเลย
    if user.has_perm("blog.can_publish_post"):
        send_welcome_email(user)
```

### 328.5 กับดักที่ 2: `request.user` ใน Middleware/View เดียวกัน

Pattern เดียวกันเกิดขึ้นได้ในสถานการณ์ที่พบบ่อยกว่านั้นอีก — เมื่อ middleware หรือ
view ตัวก่อนหน้าเผลอเรียก `request.user.has_perm(...)` ไปแล้ว (เช่น
`PermissionRequiredMixin` ของ view ก่อนหน้าในสาย, หรือ context processor ที่ทำงาน
ระหว่าง render template ก่อนหน้า) แล้วโค้ดส่วนหลังมามอบสิทธิ์ใหม่ให้ user คนเดียวกัน
ในคำขอเดียวกัน — เพราะ `request.user` เป็น object เดียวกันตลอดทั้ง request-response
cycle จึงแชร์ cache เดียวกันไปด้วย

```python
def bulk_action_view(request):
    if request.user.has_perm("blog.can_publish_post"):
        # ทำอะไรบางอย่าง...
        pass

    if request.POST.get("action") == "self_upgrade":
        Permission_obj = Permission.objects.get(
            codename="can_publish_post", content_type__app_label="blog"
        )
        request.user.user_permissions.add(Permission_obj)

        # ผิด! request.user ตัวนี้ถูก cache ไปแล้วตั้งแต่บรรทัดแรกของฟังก์ชัน
        request.user.has_perm("blog.can_publish_post")  # ยังคืนค่าตาม cache เดิม
```

**ข้อดีที่ควรรู้**: cache นี้จะ**ไม่**รั่วไหลข้าม request เพราะ Django สร้าง
`request.user` เป็น object ใหม่ (`SimpleLazyObject` ที่ resolve จาก session) ทุก
ครั้งที่มี request ใหม่เข้ามา — ปัญหาจึงเกิดเฉพาะ **ภายในคำขอเดียวกัน** เท่านั้น
ไม่ต้องกังวลว่า cache จะค้างข้ามผู้ใช้หรือข้ามเวลานาน ๆ

### 328.6 กับดักที่ 3: Unit Test ที่ตกหลุมนี้บ่อยที่สุด

```python
# tests.py — เวอร์ชันที่มี bug
def test_user_can_publish_after_joining_editors_group(self):
    user = User.objects.create_user(username="test_editor", password="pass1234")

    self.assertFalse(user.has_perm("blog.can_publish_post"))  # เรียก has_perm ครั้งแรก -> cache ถูกสร้าง

    editors = Group.objects.get(name="Editors")
    user.groups.add(editors)

    self.assertTrue(user.has_perm("blog.can_publish_post"))
    # FAIL! เพราะ user ตัวเดิมยังใช้ _perm_cache เก่าที่ไม่มี Group นี้อยู่
```

แก้ไขโดยดึง instance ใหม่ก่อน assert รอบสอง:

```python
def test_user_can_publish_after_joining_editors_group(self):
    user = User.objects.create_user(username="test_editor", password="pass1234")
    self.assertFalse(user.has_perm("blog.can_publish_post"))

    editors = Group.objects.get(name="Editors")
    user.groups.add(editors)

    user = User.objects.get(pk=user.pk)   # <-- บรรทัดสำคัญที่แก้ปัญหา
    self.assertTrue(user.has_perm("blog.can_publish_post"))
```

### 328.7 ตารางสรุปกฎการจัดการ Permission Cache

| สถานการณ์ | ต้องระวังเรื่อง Cache หรือไม่ | วิธีแก้ |
|---|---|---|
| เช็ค permission ครั้งเดียวต่อ request (กรณีทั่วไป) | ไม่ต้องกังวล | ไม่ต้องทำอะไรเพิ่ม |
| มอบ/ถอน permission แล้วเช็คซ้ำใน **request/ฟังก์ชันเดียวกัน** | ✅ ต้องระวัง | ดึง instance ใหม่ด้วย `User.objects.get(pk=...)` หรือ `delattr` cache attribute |
| มอบ/ถอน permission แล้วให้ user login ใหม่ (request ใหม่) | ไม่ต้องกังวล | request ใหม่ = user object ใหม่ = ไม่มี cache เก่า |
| เขียน Unit Test ที่แก้ permission กลาง test แล้วเช็คซ้ำ | ✅ ต้องระวังเสมอ | ดึง instance ใหม่ก่อน assert ครั้งถัดไป |

---

## ขั้นตอนที่ 329: พฤติกรรมพิเศษของ Superuser

### 329.1 Superuser คืออะไร และต่างจาก `is_staff` อย่างไร

`AbstractUser`/`PermissionsMixin` มี 2 flag ที่มักถูกสับสน:

| Flag | ความหมาย | ผลต่อ Permission |
|---|---|---|
| `is_staff` | อนุญาตให้ **เข้าหน้า Django Admin** ได้ (`/admin/`) | ไม่ได้ให้สิทธิ์อะไรเพิ่มเลย ยังต้องมี permission ปกติถึงจะ CRUD ข้อมูลใน admin ได้ |
| `is_superuser` | เป็นผู้ดูแลระบบระดับสูงสุด | **บายพาสการเช็ค permission ทั้งหมด** โดยอัตโนมัติ |

ข้อผิดพลาดที่พบบ่อย: ตั้ง `is_staff=True` แล้วคาดหวังว่า user จะทำทุกอย่างได้ในระบบ
แต่จริง ๆ แล้ว `is_staff` เพียงอย่างเดียว **ไม่ได้ปลดล็อกสิทธิ์อะไรเลย** ต้องมี
permission จริงประกอบด้วยเสมอ (หรือเป็น `is_superuser`)

### 329.2 กลไกจริง: `has_perm()` บายพาส Backend ทั้งหมด

จุดสำคัญที่สุดที่ต้องเข้าใจคือ **การบายพาสของ superuser ไม่ได้เกิดขึ้นใน
`ModelBackend`** แต่เกิดขึ้นที่ระดับ `PermissionsMixin` เอง (ใน
`django.contrib.auth.base_user`/`models`) ก่อนที่จะเรียก authentication backend
ใด ๆ เลยด้วยซ้ำ โครงสร้างจริง (แบบย่อ) หน้าตาประมาณนี้:

```python
# ภายในของ django.contrib.auth.models.PermissionsMixin (แสดงเพื่ออธิบาย)
class PermissionsMixin(models.Model):
    def has_perm(self, perm, obj=None):
        # Active superuser มีสิทธิ์ทุกอย่างเสมอ โดยไม่ต้องเช็คอะไรต่อ
        if self.is_active and self.is_superuser:
            return True
        return _user_has_perm(self, perm, obj)

    def has_module_perms(self, app_label):
        if self.is_active and self.is_superuser:
            return True
        return _user_has_module_perms(self, app_label)
```

สังเกตว่าเงื่อนไข `if self.is_active and self.is_superuser: return True` ทำงาน
**ก่อน** ที่จะเรียก `_user_has_perm()` (ซึ่งเป็นตัวที่ไปวนเช็คทุก authentication
backend ใน `AUTHENTICATION_BACKENDS`) เสียอีก พูดง่าย ๆ คือ: **ถ้าเป็น active
superuser, `ModelBackend` และแม้แต่ `guardian.backends.ObjectPermissionBackend`
(จาก ขั้นตอนที่ 327) จะไม่ถูกเรียกเลยด้วยซ้ำ**

```python
python manage.py shell
```

```python
admin_user = User.objects.get(username="admin", is_superuser=True)

admin_user.has_perm("blog.can_publish_post")
# True — แม้จะไม่เคยมอบ permission นี้ให้เลย ไม่มี Group ใดที่ admin_user สังกัดด้วยซ้ำ

admin_user.has_perm("some_app.some_permission_that_does_not_even_exist")
# True เช่นกัน! เพราะ superuser บายพาสก่อนที่จะเช็คว่า permission นั้นมีอยู่จริงหรือไม่

kai_post = Post.objects.get(slug="kai-post")
admin_user.has_perm("blog.change_post", kai_post)
# True แม้จะส่ง obj เข้าไปด้วย! superuser บายพาส object-level permission (guardian) ด้วยเช่นกัน
```

### 329.3 แต่ `get_all_permissions()` ทำงานต่างออกไปเล็กน้อย

จุดที่น่าสนใจ (และมักสร้างความสับสน) คือ `get_all_permissions()` **ไม่มี** shortcut
ระดับ `PermissionsMixin` แบบ `has_perm()` แต่ superuser ก็ยังได้ผลลัพธ์เป็น
permission "ทุกตัวในระบบ" เพราะ `ModelBackend._get_permissions()` (เมธอดภายใน)
เช็ค `is_superuser` เองอีกชั้นหนึ่ง แล้วคืน `Permission.objects.all()` แทนที่จะ
query จาก `user_permissions`/`groups` ตามปกติ:

```python
admin_user.get_all_permissions()
# {'blog.add_post', 'blog.change_post', 'blog.delete_post', 'blog.view_post',
#  'blog.can_publish_post', 'auth.add_user', 'auth.change_group', ... }
# คือ "ทุก permission ที่เคยถูกสร้างในระบบทั้งหมด" ไม่ใช่แค่ของแอป blog
```

แต่ถ้าส่ง `obj` เข้าไปด้วย ผลจะกลับไปเป็นค่าว่างเหมือนเดิม เพราะการเช็ค
`obj is not None` เกิดขึ้น**ก่อน**การเช็ค `is_superuser` ใน `_get_permissions()`:

```python
admin_user.get_all_permissions(kai_post)
# set() — ว่างเปล่า! (ต่างจาก has_perm(perm, obj) ที่คืน True เสมอ)
```

นี่คือความไม่สม่ำเสมอเล็ก ๆ ที่ควรรู้ไว้: **`has_perm()` บายพาสทุกกรณีรวมถึง
object-level แต่ `get_all_permissions()`/`get_user_permissions()`/
`get_group_permissions()` ไม่ได้บายพาสเมื่อส่ง `obj`** — ในทางปฏิบัติแทบไม่มีผล
กระทบเพราะโค้ดจริงมักเรียก `has_perm()` เป็นหลัก แต่ควรรู้ไว้เผื่อ debug

### 329.4 Superuser **ไม่** บายพาส Custom Logic ที่คุณเขียนเอง

ข้อควรระวังที่สำคัญที่สุดในขั้นตอนนี้: การบายพาสข้างต้นเกิดขึ้นเฉพาะกับกลไกที่พึ่ง
`has_perm()`/`has_perms()`/`has_module_perms()` เท่านั้น — **`AuthorRequiredMixin`
ที่เขียนเองใน Part 024 (ขั้นตอนที่ 232) ไม่ได้รับการบายพาสนี้โดยอัตโนมัติ** เพราะ
เป็น custom logic ที่เทียบ field ตรง ๆ ไม่ได้เรียก `has_perm()` เลย:

```python
class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    def test_func(self):
        post = self.get_object()
        return post.author_id == self.request.user.id
        # ไม่มีการเช็ค is_superuser ในนี้เลย!
```

ผลคือถ้า `admin_user` (superuser) พยายามแก้ไขโพสต์ของ `kai` ผ่าน view ที่ใช้
`AuthorRequiredMixin` นี้ **จะถูกปฏิเสธ (403)** ทั้งที่เป็น superuser! เพราะ
`test_func()` เช็คแค่ `post.author_id == self.request.user.id` ตรง ๆ ไม่เกี่ยวกับ
ระบบ permission เลย ถ้าต้องการให้ superuser ผ่านเงื่อนไขนี้ด้วย ต้องเขียนเช็คเพิ่ม
เอง:

```python
class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    def test_func(self):
        post = self.get_object()
        if self.request.user.is_superuser:
            return True   # ต้องเขียนเงื่อนไขนี้เองถ้าต้องการให้ superuser บายพาสได้
        return post.author_id == self.request.user.id
```

### 329.5 ตารางสรุปพฤติกรรมการบายพาสของ Superuser

| กลไก | Superuser บายพาสอัตโนมัติหรือไม่ | หมายเหตุ |
|---|---|---|
| `user.has_perm(perm)` | ✅ บายพาสเสมอ | เช็คที่ `PermissionsMixin` ก่อนแตะ backend ใด ๆ |
| `user.has_perms(perm_list)` | ✅ บายพาสเสมอ | เรียก `has_perm()` วนซ้ำภายใน จึงบายพาสเช่นกัน |
| `user.has_module_perms(app_label)` | ✅ บายพาสเสมอ | ใช้ตัดสินเมนู Django Admin |
| `user.has_perm(perm, obj)` (object-level ผ่าน guardian) | ✅ บายพาสเสมอ | บายพาสก่อนที่ guardian backend จะถูกเรียกด้วยซ้ำ |
| `@permission_required` / `PermissionRequiredMixin` | ✅ บายพาสเสมอ | เพราะสร้างจาก `has_perm()`/`has_perms()` ภายใน |
| `{% if perms.blog.xxx %}` ใน template | ✅ บายพาสเสมอ | เหตุผลเดียวกัน — เรียก `has_perm()` เบื้องหลัง |
| `UserPassesTestMixin` (`test_func()` ที่เขียนเอง) | ❌ **ไม่บายพาส** | เป็น custom logic ล้วน ๆ ต้องเช็ค `is_superuser` เอง |
| Custom Mixin ที่ override `dispatch()` เอง (เช่น `AuthorRequiredMixin` แบบดิบ) | ❌ **ไม่บายพาส** | เหตุผลเดียวกัน |

### 329.6 ข้อควรระวังเชิง Security: อย่าทดสอบด้วย Superuser เพียงอย่างเดียว

ผลข้างเคียงที่อันตรายที่สุดของพฤติกรรม superuser คือ **มันซ่อน bug ด้าน
permission ระหว่างพัฒนา** — ถ้านักพัฒนา login ด้วยบัญชี superuser ตลอดเวลาระหว่าง
เทส ฟีเจอร์ที่ควรจำกัดสิทธิ์ (เช่น `PermissionRequiredMixin` ที่ลืมใส่
`permission_required` หรือใส่ผิด codename) จะดูเหมือนทำงานถูกต้องเสมอ เพราะ
superuser ผ่านทุกด่านโดยไม่สนใจว่าโค้ดเขียนถูกหรือผิด

**แนวทางที่ทีมมืออาชีพใช้**: สร้าง test user ที่เป็น staff ธรรมดา (ไม่ใช่
superuser) พร้อม assign เข้า Group ที่ต้องการทดสอบจริง แล้วทดสอบ flow ทั้งหมดผ่าน
บัญชีนั้น หรือเขียน automated test (`self.client.force_login(non_superuser)`) ที่
ครอบคลุมทั้งกรณี "มีสิทธิ์" และ "ไม่มีสิทธิ์" เสมอ ไม่ใช่ทดสอบแค่ happy path ผ่าน
superuser คนเดียว

---

## ขั้นตอนที่ 330: สรุปและแบบฝึกหัด

### 330.1 ประกอบทุกอย่างเข้าด้วยกัน: ระบบ Editors/Authors แบบสมบูรณ์

มาประกอบทุกแนวคิดจาก 9 ขั้นตอนที่ผ่านมาเข้าเป็นระบบเดียวที่ทำงานได้จริง

**Model** (เพิ่ม custom permission เข้าไปใน `Post` ที่มีอยู่แล้ว):

```python
# blog/models.py
class Post(models.Model):
    # ... fields เดิมทั้งหมดจาก ขั้นตอนที่ 321.2 ...

    class Meta:
        ordering = ["-created_at"]
        permissions = [
            ("can_publish_post", "Can publish post"),
        ]
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

**Data migration สำหรับ Group** (ต่อจาก ขั้นตอนที่ 323.6 แต่ครบทั้ง 2 Group):

```python
# blog/migrations/0004_seed_groups.py
from django.db import migrations

GROUP_PERMISSIONS = {
    "Editors": ["add_post", "change_post", "delete_post", "view_post", "can_publish_post"],
    "Authors": ["add_post", "change_post", "view_post"],
}


def create_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    Permission = apps.get_model("auth", "Permission")

    for group_name, codenames in GROUP_PERMISSIONS.items():
        group, _ = Group.objects.get_or_create(name=group_name)
        perms = Permission.objects.filter(
            content_type__app_label="blog", codename__in=codenames
        )
        group.permissions.set(perms)


def remove_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    Group.objects.filter(name__in=GROUP_PERMISSIONS.keys()).delete()


class Migration(migrations.Migration):
    dependencies = [
        ("blog", "0003_alter_post_options"),
    ]

    operations = [
        migrations.RunPython(create_groups, reverse_code=remove_groups),
    ]
```

**Mixins** (รวม `AuthorRequiredMixin` จาก Part 024 เข้ากับ `PermissionRequiredMixin`):

```python
# blog/mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin


class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    """เจ้าของโพสต์เท่านั้นที่ผ่าน (หรือ superuser ที่เช็คเพิ่มเอง)"""

    def test_func(self):
        post = self.get_object()
        if self.request.user.is_superuser:
            return True
        return post.author_id == self.request.user.id
```

**Views** (ครบทั้งการแก้ไขของตัวเอง และการเผยแพร่ที่ต้องเป็น Editor เท่านั้น):

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DeleteView, DetailView, ListView, UpdateView

from .forms import PostForm
from .mixins import AuthorRequiredMixin
from .models import Post


class PostListView(ListView):
    model = Post
    template_name = "blog/post_list.html"
    context_object_name = "posts"
    paginate_by = 10


class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"


class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    permission_required = "blog.add_post"   # ทั้ง Editors และ Authors มีสิทธิ์นี้

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    # เจ้าของโพสต์แก้ไขโพสต์ของตัวเองได้เสมอ ไม่ว่าจะอยู่ Group ไหน


class PostDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    success_url = reverse_lazy("blog:post-list")
    permission_required = "blog.delete_post"   # เฉพาะ Editors เท่านั้นที่มีสิทธิ์นี้


class PostPublishView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Post
    fields = ["status"]
    template_name = "blog/post_publish.html"
    permission_required = "blog.can_publish_post"   # เฉพาะ Editors เท่านั้น
    permission_denied_message = "เฉพาะบรรณาธิการ (Editors) เท่านั้นที่เผยแพร่โพสต์ได้"
    raise_exception = True
```

**Template** ที่ปรับ UI ตาม role โดยอัตโนมัติ:

```html
<!-- templates/blog/post_detail.html -->
{% extends "base.html" %}

{% block content %}
    <article>
        <h1>{{ post.title }}</h1>
        <p>โดย {{ post.author.get_full_name|default:post.author.username }}</p>
        {{ post.content|linebreaks }}
    </article>

    <div class="post-actions">
        {% if user == post.author or user.is_superuser %}
            <a href="{% url 'blog:post-update' post.slug %}">แก้ไขโพสต์นี้</a>
        {% endif %}

        {% if perms.blog.can_publish_post and post.status == "draft" %}
            <form action="{% url 'blog:post-publish' post.slug %}" method="post">
                {% csrf_token %}
                <button type="submit">เผยแพร่โพสต์นี้</button>
            </form>
        {% endif %}

        {% if perms.blog.delete_post %}
            <a href="{% url 'blog:post-delete' post.slug %}" class="text-danger">ลบโพสต์นี้</a>
        {% endif %}
    </div>
{% endblock %}
```

**URLs**:

```python
# blog/urls.py
from django.urls import path

from . import views

app_name = "blog"

urlpatterns = [
    path("", views.PostListView.as_view(), name="post-list"),
    path("<slug:slug>/", views.PostDetailView.as_view(), name="post-detail"),
    path("create/", views.PostCreateView.as_view(), name="post-create"),
    path("<slug:slug>/edit/", views.PostUpdateView.as_view(), name="post-update"),
    path("<slug:slug>/delete/", views.PostDeleteView.as_view(), name="post-delete"),
    path("<slug:slug>/publish/", views.PostPublishView.as_view(), name="post-publish"),
]
```

ผลลัพธ์สุดท้าย: `narin` (อยู่ Group "Authors") สร้างและแก้ไขโพสต์ของตัวเองได้ แต่ลบ
หรือเผยแพร่ไม่ได้ (ปุ่มไม่แสดงในเทมเพลตด้วยซ้ำ และถ้ายิง request ตรง ๆ ก็โดน 403)
ส่วน `sofia` (อยู่ Group "Editors") ทำได้ทุกอย่างรวมถึงเผยแพร่และลบโพสต์ของคนอื่นด้วย
— ระบบสิทธิ์ทั้งหมดควบคุมผ่าน Group เพียง 2 กลุ่ม ไม่ต้องยุ่งกับ user รายบุคคลเลย

### 330.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า Django สร้าง permission `add`/`change`/`delete`/`view` ให้อัตโนมัติ
  ทุก Model ผ่าน signal `post_migrate` และรู้จักโครงสร้างตาราง `auth_permission`
- ✅ ใช้เมธอด `has_perm()`, `has_perms()`, `get_all_permissions()`,
  `get_user_permissions()`, `get_group_permissions()` ได้อย่างถูกต้อง
- ✅ สร้างและจัดการ Group ผ่าน Admin, Shell, Management Command, และ Data Migration
  พร้อมเข้าใจความแตกต่างของ `.set()`/`.add()`/`.remove()`
- ✅ เจาะลึก `@permission_required` และ `PermissionRequiredMixin` ครบทุกจุดที่
  override ได้ (`permission_required`, `get_permission_required()`,
  `has_permission()`, `handle_no_permission()`)
- ✅ สร้าง Custom Permission ผ่าน `Meta.permissions` และรู้วิธีสร้าง permission
  ระดับแอปที่ไม่ผูกกับ Model จริง
- ✅ ใช้ `{% if perms.app.codename %}` ในเทมเพลต และเข้าใจข้อจำกัดว่าเช็คได้แค่
  ระดับ Model
- ✅ เข้าใจอย่างลึกซึ้งว่าทำไม Django permission เป็นระดับ Model ไม่ใช่ระดับ Object
  และรู้จัก `django-guardian` เป็นทางออก (เจาะลึกเต็มใน Part 038)
- ✅ รู้จักกับดัก Permission Caching (`_perm_cache`, `_user_perm_cache`,
  `_group_perm_cache`) และวิธีแก้ปัญหา stale cache
- ✅ เข้าใจพฤติกรรมบายพาสของ Superuser ทั้งที่ `has_perm()` บายพาสเสมอ แต่ custom
  logic ที่เขียนเองไม่บายพาสอัตโนมัติ

### 330.3 Checklist ก่อนไป Part ถัดไป

- [ ] รัน `python manage.py shell` แล้วเรียก `Permission.objects.filter(content_type__app_label="blog")` เห็น permission มาตรฐาน 4 ตัวของ `Post`
- [ ] เพิ่ม `Meta.permissions` ให้ `Post` มี `can_publish_post` และรัน migrate สำเร็จ
- [ ] สร้าง Group "Editors" และ "Authors" พร้อม permission ที่ถูกต้องตาม ขั้นตอนที่ 330.1
- [ ] เขียน `PostPublishView` ด้วย `PermissionRequiredMixin` และทดสอบว่า user ที่ไม่มีสิทธิ์โดน 403 จริง
- [ ] เพิ่ม `{% if perms.blog.can_publish_post %}` ในเทมเพลตและเห็นปุ่มปรากฏ/หายไปตาม role ของ user ที่ login
- [ ] ทดสอบกับดัก Permission Cache ด้วยตัวเอง: เพิ่ม permission ให้ user ใน shell แล้วเช็ค `has_perm()` ซ้ำโดยไม่ fetch instance ใหม่ ดูผลลัพธ์ที่ผิดคาดด้วยตา
- [ ] ทดสอบว่า superuser บายพาส `PermissionRequiredMixin` ได้ แต่โดน `AuthorRequiredMixin` (ที่ไม่เช็ค `is_superuser`) บล็อกจริง

### 330.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง Group ที่สามใน `seed_groups` migration ชื่อ `"Moderators"`
ที่มีสิทธิ์ `view_post` และ custom permission ใหม่ `can_hide_comment` (สมมติว่ามี
Model `Comment` ที่คุณเพิ่มเข้ามาเอง) แล้วเขียน `CommentHideView` ด้วย
`PermissionRequiredMixin` ที่จำกัดให้เฉพาะ Moderators ซ่อนคอมเมนต์ได้

**แบบฝึกหัดที่ 2**: เขียน management command ชื่อ `audit_permissions` ที่ลูปผ่าน
user ทุกคนในระบบ แล้วพิมพ์รายงานว่าแต่ละคนอยู่ Group อะไรบ้าง และมี permission
อะไรบ้าง (ใช้ `get_all_permissions()`) โดยแยกแสดง permission ที่มาจาก Group กับ
ที่ให้ตรงกับ user คนนั้นแยกกันให้ชัดเจน (ใช้ `get_group_permissions()` และ
`get_user_permissions()`)

**แบบฝึกหัดที่ 3**: จำลองสถานการณ์กับดัก Permission Cache จาก ขั้นตอนที่ 328.3
ด้วยตัวเองใน `python manage.py shell` — สร้าง user ใหม่, เรียก `has_perm()`
ครั้งแรก (ควรเป็น `False`), เพิ่มเข้า Group "Editors", เรียก `has_perm()` ซ้ำโดย
**ไม่** fetch instance ใหม่ (ควรเห็นว่ายังเป็น `False` ทั้งที่ไม่ควรใช่) แล้วแก้ไข
ด้วยการ fetch instance ใหม่ให้ได้ผลลัพธ์ที่ถูกต้อง บันทึกทั้งสองผลลัพธ์เปรียบเทียบกัน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน Unit Test (`django.test.TestCase`) ที่ครอบคลุม
ระบบ Editors/Authors จาก ขั้นตอนที่ 330.1 ให้ครบ 4 เคส: (1) Author สร้างโพสต์ได้
(2) Author แก้ไขโพสต์ของตัวเองได้แต่ของคนอื่นไม่ได้ (คาดหวัง 403) (3) Author เผยแพร่
โพสต์ไม่ได้ (คาดหวัง 403) (4) Editor เผยแพร่โพสต์ของใครก็ได้สำเร็จ ใช้
`self.client.force_login()` สลับ user ระหว่างเคส และตรวจสอบ `response.status_code`
ให้ตรงกับที่คาดหวังทุกเคส

### 330.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องรัน `migrate` ทุกครั้งที่แก้ `Meta.permissions` หรือไม่ ถ้าไม่มีการเปลี่ยน field ของ Model เลย?**
A: ต้องรันทั้ง `makemigrations` และ `migrate` เสมอ แม้จะไม่มีคอลัมน์ในฐานข้อมูล
เปลี่ยนแปลงเลยก็ตาม เพราะ permission ใหม่ถูกสร้างผ่าน signal `post_migrate` ที่ทำงาน
ตอนรัน `migrate` เท่านั้น ถ้าลืมรัน permission ใหม่จะไม่ปรากฏใน `auth_permission`
เลย และทุกอย่างที่อ้างอิง permission นั้น (`has_perm`, `PermissionRequiredMixin`)
จะพังทันที

**Q: ทำไม `user.has_perm("blog.change_post")` คืน `True` ทั้งที่ user คนนั้นไม่เคย
ถูกมอบสิทธิ์นี้โดยตรงหรือผ่าน Group เลย?**
A: เช็ค 3 อย่างตามลำดับ: (1) user เป็น superuser หรือไม่ — ถ้าใช่จะ `True` เสมอตาม
ขั้นตอนที่ 329 (2) user object ที่ใช้เช็คถือ cache เก่าที่ query มาตอนที่ยังไม่มี
Group อยู่หรือไม่ (ขั้นตอนที่ 328) (3) มี custom `AUTHENTICATION_BACKENDS` อื่น
(เช่น guardian) ที่ให้สิทธิ์ผ่านทางอื่นหรือไม่

**Q: ควรใช้ `PermissionRequiredMixin` หรือ `UserPassesTestMixin` (จาก Part 024)
ดี?**
A: ใช้ `PermissionRequiredMixin` เมื่อสิทธิ์เป็นแบบ **role-based** ทั่วไปที่จัดการ
ผ่านระบบ Permission/Group ของ Django ได้ (เช่น "ต้องเป็น Editor") ใช้
`UserPassesTestMixin` เมื่อเงื่อนไขเป็น **แบบเฉพาะเจาะจงกับข้อมูล** ที่ระบบ
permission มาตรฐานตอบไม่ได้ (เช่น "ต้องเป็นเจ้าของ object นี้") ทั้งสองใช้ร่วมกันได้
และมักใช้ร่วมกันในโปรเจกต์จริงเสมอ ตามที่แสดงใน ขั้นตอนที่ 330.1

**Q: จำเป็นต้องติดตั้ง `django-guardian` ตั้งแต่ตอนนี้เลยหรือไม่?**
A: ไม่จำเป็น ถ้าโจทย์ของคุณคือ "เจ้าของ object เท่านั้นที่แก้ไขได้" (ownership)
Custom Mixin แบบ Part 024 ก็เพียงพอแล้วและเรียบง่ายกว่ามาก ให้ติดตั้ง
`django-guardian` เมื่อโจทย์ซับซ้อนกว่านั้นจริง ๆ เช่น "มอบสิทธิ์แก้ไขให้ user คนอื่น
เป็นราย object" — เราจะพาติดตั้งและใช้งานแบบเต็มรูปแบบใน Part 038

### 330.6 เตรียมตัวสำหรับ Part ถัดไป

**Part 034: Django Sessions และ Cookies** จะพาคุณลงลึกกลไกเบื้องหลังที่ทำให้
`request.user` "จำ" ได้ว่าใครล็อกอินอยู่ในทุก request — Session Framework,
Session Backend (database, cache, signed cookies), การตั้งค่า `SESSION_COOKIE_AGE`,
`SESSION_EXPIRE_AT_BROWSER_CLOSE`, ความแตกต่างระหว่าง Session กับ Cookie โดยตรง,
และวิธีจัดการ Session อย่างปลอดภัยในระบบที่มีผู้ใช้พร้อมกันจำนวนมาก ซึ่งเป็นรากฐาน
สำคัญก่อนที่ Part 035 จะพาไปเจาะลึกเรื่อง Password Management และ Security ต่อไป

เตรียมเปิด Django Admin และดูตาราง `django_session` ในฐานข้อมูลของคุณไว้ก่อน
แล้วไปต่อกันเลย!
