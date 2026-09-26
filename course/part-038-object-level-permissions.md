# Part 038: Row-Level Permissions และ Object-Level Permission

> **ขั้นตอนที่ 371-380 ของหลักสูตร** | Phase 4: Authentication, Users และ Permissions (Part สุดท้ายของ Phase)
>
> เป้าหมายของ Part นี้: ปิดช่องว่างสำคัญที่สุดที่ Part 033 (ขั้นตอนที่ 327) ทิ้งไว้ว่า
> "Django permission เป็นระดับ **model** ไม่ใช่ระดับ **object**" คุณจะติดตั้งและเจาะลึก
> **django-guardian** เพื่อมอบและตรวจสอบสิทธิ์แบบ "โพสต์ตัวนี้เท่านั้น" ได้จริง จากนั้นจะ
> กลับไปหา `AuthorRequiredMixin` ที่เขียนไว้ใน Part 024 (ขั้นตอนที่ 232-234) แล้วยกระดับมัน
> ให้ใช้ระบบ Object-level Permission แบบมืออาชีพแทนการเทียบ `request.user == post.author`
> ตรง ๆ คุณจะเรียนรู้ทั้งแนวทางที่พึ่ง library และแนวทาง manual แบบไม่ใช้ library, วิธีผสาน
> Group-based กับ Object-level permission เข้าด้วยกัน, ผลกระทบด้าน Performance เมื่อระบบมี
> ผู้ใช้และข้อมูลจำนวนมาก, และวิธีเขียน Test ให้ครอบคลุม เพราะ Part นี้คือ **Part สุดท้าย
> ของ Phase 4** เนื้อหาส่วนท้ายจะเป็นการสรุปทั้ง Phase (Part 031-038) แบบเต็มรูปแบบ
> พร้อม Quiz และแบบฝึกหัดใหญ่ปิดท้าย ก่อนจะพาคุณก้าวเข้าสู่ **Phase 5: Django REST
> Framework และ API** ใน Part 039 เป็นต้นไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 371: ทบทวนว่าทำไม Django default permission เป็นระดับ model เท่านั้น
- ขั้นตอนที่ 372: ติดตั้งและตั้งค่า django-guardian
- ขั้นตอนที่ 373: มอบ Object-level Permission ให้ user/group เฉพาะเจาะจง
- ขั้นตอนที่ 374: เช็ค Object-level Permission ในโค้ดจริง
- ขั้นตอนที่ 375: ใช้ Object-level Permission ใน View/CBV จริง — ยกระดับ `AuthorRequiredMixin`
- ขั้นตอนที่ 376: Row-level Permission แบบ Manual โดยไม่ใช้ Library
- ขั้นตอนที่ 377: ผสาน Group-based Permission + Object-level Permission เข้าด้วยกัน
- ขั้นตอนที่ 378: ผลกระทบด้าน Performance ของ Object-level Permission ที่ Scale ใหญ่
- ขั้นตอนที่ 379: การเขียน Test สำหรับ Permission Logic ทั้งหมด
- ขั้นตอนที่ 380: สรุป Phase 4 ทั้งหมด, Quiz, แบบฝึกหัดใหญ่ปิดท้าย Phase และก้าวสู่ Phase 5

---

## ขั้นตอนที่ 371: ทบทวนว่าทำไม Django default permission เป็นระดับ model เท่านั้น

### 371.1 ย้อนกลับไปที่ Part 033 (ขั้นตอนที่ 327)

Part 033 พาคุณไปเจาะลึกระบบ Permission มาตรฐานของ Django จนครบทุกมุม และปิดท้ายด้วยการ
พิสูจน์ปัญหาที่ชัดเจนที่สุดของระบบนี้: สมมติเว็บบล็อกมีนักเขียน narin, sofia และ kai
ทุกคนอยู่ใน Group `Authors` ที่มีสิทธิ์ `blog.change_post` เหมือนกันหมด

```python
narin.has_perm("blog.change_post")
# True

kai_post = Post.objects.get(slug="kai-post")
narin.has_perm("blog.change_post", kai_post)
# False เสมอ! ไม่ใช่เพราะเช็คแล้วไม่ผ่าน แต่เพราะ ModelBackend "ปฏิเสธที่จะตอบ"
# คำถามระดับ object ทันทีที่เห็นว่า obj is not None
```

`narin.has_perm("blog.change_post")` (ไม่ส่ง `obj`) ตอบ `True` เสมอ เพราะ permission
`blog.change_post` แปลว่า "แก้ไข `Post` ตัวไหนก็ได้ในระบบ" — นี่คือรากฐานของปัญหาที่
Part นี้จะแก้ไขให้สมบูรณ์

### 371.2 ทบทวน schema ที่เป็นต้นเหตุ

Part 033 (ขั้นตอนที่ 321.4 และ 327.2) อธิบาย schema ของตาราง `auth_permission` และ
`auth_user_user_permissions`/`auth_group_permissions` ไว้แล้วว่าไม่มีคอลัมน์ใดอ้างถึง
object เฉพาะเจาะจงเลย (ไม่มี `object_id`):

```
auth_permission
├── id
├── name
├── content_type_id   <-- ผูกกับ "Model" (เช่น blog.Post) เท่านั้น ไม่ใช่แถวข้อมูล
└── codename

auth_user_user_permissions / auth_group_permissions
├── user_id / group_id
└── permission_id     <-- ไม่มีคอลัมน์เก็บว่า "ใช้ได้กับแถวไหน"
```

ทีมพัฒนา Django ตั้งใจออกแบบให้ `has_perm(perm, obj=None)` และเมธอดที่เกี่ยวข้องทั้งหมด
**มี parameter `obj` รองรับไว้ตั้งแต่แรก** เป็น "hook" ที่เปิดให้ authentication backend
อื่นเข้ามาเสริมความสามารถนี้ได้ โดยไม่ต้องแก้ core ของ Django เลย — เอกสารทางการระบุไว้
ตรง ๆ ว่า Django "provides object permission support in the model permission backend"
แต่ "there is no implementation for it in the core"

### 371.3 คำศัพท์ที่ต้องแยกให้ชัดเจนตลอด Part นี้

| คำศัพท์ | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Model-level Permission** | สิทธิ์ที่ใช้ได้กับ Model ทั้งหมด ไม่แยกแถว | `blog.change_post` = แก้ไข Post ตัวไหนก็ได้ |
| **Object-level Permission** (หรือ **Row-level Permission** — สองคำนี้ใช้แทนกันได้ในบริบทนี้) | สิทธิ์ที่ผูกกับแถวข้อมูลเฉพาะเจาะจง | `blog.change_post` เฉพาะ `Post` ที่ `id=42` เท่านั้น |
| **Ownership-based Permission** | รูปแบบเฉพาะของ Object-level ที่กฎคือ "เจ้าของเท่านั้น" | `post.author == request.user` |
| **Delegated Permission** | Object-level ที่มอบให้คนอื่นที่ **ไม่ใช่** เจ้าของ | บรรณาธิการ B ได้รับสิทธิ์แก้ไขโพสต์ของนักเขียน A ชั่วคราว |

จุดที่ต้องเข้าใจให้แม่นคือ **Object-level Permission เป็นแนวคิดที่กว้างกว่า
Ownership-based Permission** — ownership คือกรณีพิเศษที่กฎการตัดสินใจเรียบง่ายมาก
(เทียบ FK ตัวเดียว) ในขณะที่ Object-level Permission ทั่วไปรองรับกรณีที่ซับซ้อนกว่า เช่น
มอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของ หรือมอบสิทธิ์ที่ต่างกันให้คนละคนบน object เดียวกัน

### 371.4 Roadmap ของ Part นี้

Part นี้จะพาคุณไล่เรียงจากภาพรวมไปสู่รายละเอียดเชิงลึก ตามลำดับนี้:

1. ติดตั้งและทำความเข้าใจกลไกของ `django-guardian` (ขั้นตอนที่ 372)
2. มอบและตรวจสอบสิทธิ์ระดับ object จริง (ขั้นตอนที่ 373-374)
3. นำไปใช้จริงกับ `AuthorRequiredMixin` จาก Part 024 (ขั้นตอนที่ 375)
4. เทียบกับแนวทาง manual ที่ไม่ต้องพึ่ง library (ขั้นตอนที่ 376)
5. ผสานเข้ากับระบบ Group-based permission จาก Part 033 (ขั้นตอนที่ 377)
6. เข้าใจข้อจำกัดด้าน performance และวิธี optimize (ขั้นตอนที่ 378)
7. ทดสอบทุกอย่างให้มั่นใจ (ขั้นตอนที่ 379)
8. สรุปทั้ง Phase 4 และเตรียมตัวสู่ Phase 5 (ขั้นตอนที่ 380)

---

## ขั้นตอนที่ 372: ติดตั้งและตั้งค่า django-guardian

### 372.1 `django-guardian` คืออะไร

`django-guardian` คือ third-party package ที่ได้รับความนิยมที่สุดสำหรับ implement
Object-level Permission ใน Django โดยทำงานผ่านกลไก **Authentication Backend** ที่เรียน
มาตั้งแต่ Part 031 — มันไม่ได้แทนที่ระบบ permission เดิมของ Django เลย แต่ทำงาน
**ร่วมกัน** ผ่าน `AUTHENTICATION_BACKENDS` ที่รองรับหลาย backend พร้อมกันอยู่แล้ว

ติดตั้งผ่าน pip ตามมาตรฐานที่วางไว้ตั้งแต่ Part 001:

```bash
pip install django-guardian
pip freeze > requirements.txt
```

### 372.2 ตั้งค่า `INSTALLED_APPS` และ `AUTHENTICATION_BACKENDS`

```python
# config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "guardian",           # <-- เพิ่มแอปนี้
    "accounts",
    "blog",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",   # ต้องมีไว้เสมอ สำหรับ permission ระดับ model ปกติ
    "guardian.backends.ObjectPermissionBackend",   # เพิ่มความสามารถ object-level เข้ามา
]
```

**จุดสำคัญที่สุดที่มือใหม่มักพลาด**: ต้องคง `ModelBackend` ไว้เสมอ **ห้ามลบทิ้ง** เพราะ
`ObjectPermissionBackend` ทำหน้าที่ **เฉพาะ** การเช็คตอนที่ส่ง `obj` เข้าไปใน `has_perm()`
เท่านั้น ถ้าไม่ส่ง `obj` (การเช็คแบบ model-level ปกติที่ใช้มาตลอด Part 033) มันจะคืนค่า
`False`/`set()` ว่างเสมอ — Django จะไล่เช็คทีละ backend ตามลำดับใน
`AUTHENTICATION_BACKENDS` จนกว่าจะเจอ backend ที่ตอบ `True`

### 372.3 รัน migrate และตรวจสอบตารางที่ guardian สร้าง

```bash
python manage.py migrate
```

```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, guardian, sessions
Running migrations:
  Applying guardian.0001_initial... OK
  Applying guardian.0002_generic_permissions_index... OK
  ...
```

guardian สร้างตารางใหม่ 2 ตารางหลักที่เก็บ Object-level Permission จริง (ต่างจาก
`auth_permission` ที่ไม่มีคอลัมน์อ้างถึงแถวข้อมูลเลย):

```
guardian_userobjectpermission
├── id
├── permission_id     --> FK ไปยัง auth_permission (ใช้ permission ที่มีอยู่แล้ว!)
├── content_type_id   --> FK ไปยัง django_content_type (Model อะไร)
├── object_pk         --> เก็บ primary key ของแถวข้อมูลนั้นเป็น string (เช่น "42")
└── user_id           --> FK ไปยัง auth_user

guardian_groupobjectpermission
├── id
├── permission_id
├── content_type_id
├── object_pk
└── group_id           --> FK ไปยัง auth_group
```

สังเกตความสวยงามของการออกแบบนี้: guardian **ไม่ได้สร้าง permission ใหม่ซ้ำซ้อน** แต่
**ใช้แถวใน `auth_permission` เดิมที่ Django สร้างให้อัตโนมัติอยู่แล้ว** (จาก Part 033
ขั้นตอนที่ 321) เพียงแค่เพิ่มคอลัมน์ `object_pk` เข้ามาเพื่อระบุว่า "permission นี้ใช้
กับแถวไหน" — นี่คือเหตุผลที่ permission string ยังคงเป็นรูปแบบเดิมทุกประการ
(`blog.change_post`) ไม่มีการสร้างระบบ permission คู่ขนานขึ้นมาใหม่

### 372.4 `object_pk` เก็บเป็น string เสมอ — เหตุผลเบื้องหลัง

ตาราง `guardian_userobjectpermission` ใช้ **Generic Relation** (ผ่าน `ContentType` +
`object_pk` แบบเดียวกับที่ Django ใช้ใน `django.contrib.contenttypes`) แทนที่จะเป็น
Foreign Key ตรง ๆ ไปยังแต่ละ Model เพราะ guardian ต้องรองรับ Model **ทุกตัว** ในโปรเจกต์
โดยไม่รู้ล่วงหน้าว่าจะถูกใช้กับ Model ไหนบ้าง การเก็บ `object_pk` เป็น `CharField` แทน
`IntegerField` (แม้ Primary Key ส่วนใหญ่จะเป็นเลข) ทำให้รองรับ Model ที่ใช้ `UUIDField`
เป็น Primary Key ได้ด้วย — แต่นี่คือจุดที่ทำให้เกิด **query ที่ generic เกินไปและช้ากว่า
FK ตรง ๆ** ซึ่งเราจะแก้ปัญหานี้อย่างจริงจังในขั้นตอนที่ 378

### 372.5 ตั้งค่าเสริม: `ANONYMOUS_USER_NAME` และ 403 handling

```python
# config/settings.py

# ปิดการรองรับ AnonymousUser ของ guardian ถ้าระบบไม่ต้องการมอบสิทธิ์ให้ผู้ใช้ที่ยังไม่ login
# (ค่า default ของ guardian จะสร้าง user พิเศษชื่อ "AnonymousUser" ไว้ในฐานข้อมูลเพื่อรองรับ
# กรณีที่ต้องการมอบสิทธิ์ระดับ object ให้ผู้ใช้ทั่วไปที่ไม่ล็อกอิน เช่น "guest เห็นได้เฉพาะ
# โพสต์ตัวอย่างบางตัว" — ถ้าไม่ใช้ ปิดไว้จะสะอาดกว่า)
ANONYMOUS_USER_NAME = None

# ให้ guardian ยิง Http404 แทน PermissionDenied เมื่อไม่ผ่าน object-level permission
# (ค่า default คือ False = ยิง PermissionDenied ตามปกติ)
GUARDIAN_RAISE_403 = False

# ให้ guardian ใช้ template 403.html ของตัวเองโดยอัตโนมัติแทนของ Django (ไม่จำเป็นถ้ามี
# 403.html ของโปรเจกต์อยู่แล้ว)
GUARDIAN_RENDER_403 = False
```

### 372.6 ตรวจสอบการติดตั้งด้วย shell

```bash
python manage.py shell
```

```python
from django.contrib.auth import get_user_model
from guardian.shortcuts import assign_perm

from blog.models import Post

User = get_user_model()
narin = User.objects.get(username="narin")
kai_post = Post.objects.get(slug="kai-post")

assign_perm("change_post", narin, kai_post)
narin.has_perm("blog.change_post", kai_post)
# True — สำเร็จ! guardian ทำงานถูกต้องแล้ว

sofia_post = Post.objects.get(slug="sofia-post")
narin.has_perm("blog.change_post", sofia_post)
# False — narin ไม่เคยได้รับสิทธิ์นี้สำหรับโพสต์ของ sofia
```

ถ้าเห็นผลลัพธ์ตรงกันนี้ แปลว่าการติดตั้งและตั้งค่าทั้งหมดถูกต้อง พร้อมไปต่อในขั้นตอนที่
373 ที่จะเจาะลึกวิธีมอบสิทธิ์แบบเต็มรูปแบบ

---

## ขั้นตอนที่ 373: มอบ Object-level Permission ให้ user/group เฉพาะเจาะจง

### 373.1 `assign_perm()` — ฟังก์ชันหลักที่ใช้บ่อยที่สุด

`assign_perm()` อยู่ใน `guardian.shortcuts` และเป็นทางเดียวที่แนะนำให้ใช้มอบสิทธิ์
ระดับ object (ไม่แนะนำให้สร้างแถว `UserObjectPermission` ด้วยมือโดยตรง เพราะ
`assign_perm()` จัดการเรื่อง `ContentType`/`object_pk` ให้อัตโนมัติและตรวจสอบว่า
permission codename มีอยู่จริงก่อนเสมอ):

```python
from guardian.shortcuts import assign_perm

# Signature: assign_perm(perm, user_or_group, obj=None)
assign_perm("change_post", narin, kai_post)
assign_perm("delete_post", narin, kai_post)
```

`perm` รับได้ทั้งแบบสั้น (`"change_post"` — guardian จะเดา `app_label` จาก `obj` ให้เอง)
และแบบเต็ม (`"blog.change_post"`) แต่รูปแบบสั้นเป็นที่นิยมกว่าเมื่อใช้กับ `assign_perm`/
`remove_perm`/`get_perms` เพราะสั้นกว่าและ `obj` ระบุ Model อยู่แล้วในตัวมันเอง

### 373.2 มอบสิทธิ์ให้ Group แทน user เดี่ยว — เชื่อมกับ Part 033

จุดที่ทรงพลังที่สุดของ `assign_perm()` คือรับทั้ง `User` instance และ `Group` instance
เป็น argument ที่สองได้เหมือนกันทุกประการ ทำให้ผสานกับระบบ Group จาก Part 033 ได้ทันที:

```python
from django.contrib.auth.models import Group

reviewers = Group.objects.get(name="Reviewers")

# มอบสิทธิ์แก้ไข "เฉพาะโพสต์ตัวนี้" ให้ทุกคนใน Group Reviewers พร้อมกัน
assign_perm("change_post", reviewers, kai_post)

# ทุก user ที่อยู่ใน Group Reviewers จะผ่านการเช็คนี้ทันที
for user in reviewers.user_set.all():
    print(user.username, user.has_perm("blog.change_post", kai_post))
# reviewer_a True
# reviewer_b True
```

นี่คือความแตกต่างสำคัญจากตาราง `guardian_userobjectpermission`: กรณีนี้ guardian จะ
บันทึกแถวเดียวลงตาราง `guardian_groupobjectpermission` แทน (ผูกกับ `group_id` ไม่ใช่
`user_id`) แล้ว `ObjectPermissionBackend` จะเช็คทั้งสองตารางเสมอเมื่อ `has_perm(obj=...)`
ถูกเรียก — ทั้งสิทธิ์ที่ให้ตรงกับ user และสิทธิ์ที่ได้มาจาก Group ที่ user สังกัด

### 373.3 `remove_perm()` — ถอนสิทธิ์ที่เคยให้

```python
from guardian.shortcuts import remove_perm

remove_perm("change_post", narin, kai_post)
narin.has_perm("blog.change_post", kai_post)
# False — สิทธิ์ถูกถอนแล้ว (ลบแถวออกจากตาราง guardian_userobjectpermission จริง)
```

`remove_perm()` ปลอดภัยแม้จะเรียกกับ permission ที่ user ไม่เคยมีอยู่แล้ว (ไม่ throw
exception ใด ๆ) จึงเหมาะกับการเขียนใน `try/except` น้อยกว่าที่คิด

### 373.4 มอบสิทธิ์อัตโนมัติตอนสร้าง object — จุดที่สำคัญที่สุดในทางปฏิบัติ

ในระบบจริง แนวทางที่ปลอดภัยที่สุดคือ **มอบสิทธิ์ให้เจ้าของทันทีที่สร้าง object เสร็จ**
ไม่ใช่รอให้ admin มาตั้งค่าทีหลัง เชื่อมกับ `PostCreateView` จาก Part 024
(ขั้นตอนที่ 232.4):

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import CreateView
from guardian.shortcuts import assign_perm

from .forms import PostForm
from .models import Post


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        response = super().form_valid(form)   # self.object ถูกตั้งค่าหลังบรรทัดนี้

        # มอบสิทธิ์ระดับ object ให้ผู้สร้างทันที ไม่ต้องรอ Group หรือ admin ตั้งค่าเพิ่ม
        assign_perm("change_post", self.request.user, self.object)
        assign_perm("delete_post", self.request.user, self.object)
        assign_perm("view_post", self.request.user, self.object)

        return response
```

**ทำไมต้องมอบสิทธิ์ทั้งที่ผู้ใช้เป็นเจ้าของอยู่แล้ว (`post.author == request.user`)?**
เพราะ Object-level Permission ที่มอบผ่าน guardian เป็น **แหล่งความจริงเดียว (single
source of truth)** ที่ผูกกับระบบ permission ของ Django อย่างสมบูรณ์ — ใช้ร่วมกับ
`{% if perms %}`, `PermissionRequiredMixin`, และ shortcuts อื่น ๆ ได้ทั้งหมดโดยไม่ต้อง
เขียน logic แยกสำหรับ "เช็ค field author" อีกต่อไป ซึ่งเป็นสิ่งที่เราจะเห็นประโยชน์เต็ม ๆ
ใน ขั้นตอนที่ 375

### 373.5 ใช้ Signal แทน `form_valid()` — ทางเลือกที่ครอบคลุมกว่า

ถ้าต้องการให้ทุกช่องทางที่สร้าง `Post` ได้สิทธิ์เสมอ (ไม่ใช่แค่ผ่าน `PostCreateView`
เท่านั้น แต่รวมถึงการสร้างผ่าน shell, management command, หรือ API ในอนาคต) แนวทางที่
ปลอดภัยกว่าคือใช้ signal `post_save`:

```python
# blog/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from guardian.shortcuts import assign_perm

from .models import Post


@receiver(post_save, sender=Post)
def assign_author_object_permissions(sender, instance, created, **kwargs):
    """มอบ object-level permission ให้ author ทันทีที่ Post ถูกสร้างครั้งแรก"""
    if created:
        assign_perm("change_post", instance.author, instance)
        assign_perm("delete_post", instance.author, instance)
        assign_perm("view_post", instance.author, instance)
```

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        import blog.signals  # noqa: F401
```

ข้อควรระวัง: การใช้ signal ทำให้ logic "ใครมีสิทธิ์อะไร" กระจายออกจาก view ไปอยู่ใน
`signals.py` แทน ซึ่งอาจทำให้ debug ยากขึ้นเล็กน้อย (ต้องรู้ว่ามี signal ทำงานอยู่
เบื้องหลัง) — ทีมมืออาชีพส่วนใหญ่เลือกใช้ signal เมื่อกฎการมอบสิทธิ์ "ต้องเกิดขึ้นเสมอ
ไม่ว่าทางไหน" ส่วน `form_valid()` เหมาะกับกรณีที่กฎการมอบสิทธิ์ขึ้นกับ context ของ
request นั้น ๆ (เช่น มอบสิทธิ์ต่างกันตาม role ของผู้สร้าง)

### 373.6 มอบสิทธิ์เป็นชุดผ่าน Management Command (Bulk Assign)

สำหรับการ migrate ข้อมูลเก่าที่มีอยู่แล้วในระบบให้มี object-level permission ครบถ้วน
(เช่น ตอนเพิ่งติดตั้ง guardian เข้าไปในโปรเจกต์ที่มี `Post` อยู่แล้วหลายพันแถว):

```python
# blog/management/commands/backfill_object_permissions.py
from django.core.management.base import BaseCommand
from guardian.shortcuts import assign_perm

from blog.models import Post


class Command(BaseCommand):
    help = "มอบ object-level permission (change/delete/view) ให้ author ของทุก Post ที่มีอยู่แล้ว"

    def handle(self, *args, **options):
        posts = Post.objects.select_related("author").iterator(chunk_size=500)
        count = 0
        for post in posts:
            assign_perm("change_post", post.author, post)
            assign_perm("delete_post", post.author, post)
            assign_perm("view_post", post.author, post)
            count += 1

        self.stdout.write(self.style.SUCCESS(f"มอบ object-level permission ให้ {count} โพสต์แล้ว"))
```

ใช้ `.iterator(chunk_size=500)` (ทบทวนจาก Part ที่เรียนเรื่อง QuerySet ประสิทธิภาพสูง)
เพื่อไม่โหลดข้อมูลทั้งหมดเข้า memory พร้อมกันเมื่อมีข้อมูลจำนวนมาก

### 373.7 ตารางสรุป API หลักของ `assign_perm`/`remove_perm`

| ฟังก์ชัน | Signature | หมายเหตุ |
|---|---|---|
| `assign_perm(perm, user_or_group, obj=None)` | มอบสิทธิ์ | ถ้า `obj=None` จะมอบเป็น **model-level permission ปกติ** (เหมือน `user.user_permissions.add()`) — ต้องส่ง `obj` เสมอเพื่อให้เป็น object-level จริง |
| `remove_perm(perm, user_or_group, obj=None)` | ถอนสิทธิ์ | ปลอดภัยแม้เรียกกับสิทธิ์ที่ไม่เคยมี ไม่ throw exception |
| `assign_perm` คืนค่า | `Permission` instance หรือ `UserObjectPermission`/`GroupObjectPermission` instance | ใช้ตรวจสอบผลลัพธ์หรือ debug ได้ |

---

## ขั้นตอนที่ 374: เช็ค Object-level Permission

### 374.1 `has_perm(perm, obj)` — ทบทวนและเจาะลึกกลไกเบื้องหลัง

```python
narin.has_perm("blog.change_post", kai_post)
```

เมื่อเรียกด้วย `obj` ที่ไม่ใช่ `None` Django จะไล่เช็คทุก backend ใน
`AUTHENTICATION_BACKENDS` ตามลำดับ:

1. `ModelBackend.has_perm(narin, "blog.change_post", kai_post)` → คืนค่า `False` ทันที
   (ตามที่เรียนใน Part 033 ขั้นตอนที่ 327.2 — `ModelBackend` ปฏิเสธคำถามระดับ object)
2. `ObjectPermissionBackend.has_perm(narin, "blog.change_post", kai_post)` → query ตาราง
   `guardian_userobjectpermission` และ `guardian_groupobjectpermission` (ผ่าน Group ที่
   narin สังกัด) หา row ที่ `content_type` ตรงกับ `Post`, `object_pk` ตรงกับ
   `kai_post.pk`, และ `permission` ตรงกับ `change_post` → คืนค่า `True`/`False` ตามจริง
3. ถ้า backend ใดตอบ `True` การเช็คหยุดทันทีและคืนค่า `True` (short-circuit OR logic)

### 374.2 `get_perms()` — ดูสิทธิ์ทั้งหมดที่มีต่อ object หนึ่งตัว

```python
from guardian.shortcuts import get_perms

get_perms(narin, kai_post)
# ['change_post', 'delete_post', 'view_post']

get_perms(narin, sofia_post)
# []  <-- narin ไม่มี object-level permission ใด ๆ กับโพสต์ของ sofia เลย
```

**ข้อควรระวัง**: `get_perms()` คืนเฉพาะสิทธิ์ที่มาจาก guardian (object-level) เท่านั้น
ไม่รวมสิทธิ์ระดับ model จาก `auth_permission`/Group ปกติ — ต่างจาก `get_all_permissions()`
ของ Part 033 ที่รวมทุกอย่าง ต้องแยกให้ชัดเจนว่ากำลังถามคำถามไหน

### 374.3 `get_users_with_perms()` / `get_groups_with_perms()` — มองจากมุม object

บางครั้งคำถามกลับด้าน: "ใครบ้างที่มีสิทธิ์แก้ไขโพสต์นี้" แทนที่จะถาม "user นี้มีสิทธิ์
อะไรบ้าง" ใช้ shortcut คู่นี้:

```python
from guardian.shortcuts import get_groups_with_perms, get_users_with_perms

get_users_with_perms(kai_post)
# <QuerySet [<User: narin>]>

get_users_with_perms(kai_post, attach_perms=True)
# {<User: narin>: ['change_post', 'delete_post', 'view_post']}

get_groups_with_perms(kai_post, attach_perms=True)
# {<Group: Reviewers>: ['change_post']}
```

พารามิเตอร์ `attach_perms=True` เปลี่ยนผลลัพธ์จาก `QuerySet` ธรรมดาเป็น `dict` ที่ map
user/group ไปยัง list ของสิทธิ์ที่มี — มีประโยชน์มากสำหรับหน้า admin ที่ต้องการแสดง
"ใครมีสิทธิ์อะไรบ้างกับโพสต์นี้" ในตารางเดียว

### 374.4 `get_objects_for_user()` — เครื่องมือที่สำคัญที่สุดสำหรับ QuerySet

จนถึงตอนนี้เราเช็คสิทธิ์ทีละ object แต่ในสถานการณ์จริง (เช่นหน้า "โพสต์ที่ฉันแก้ไขได้")
เราต้องการ **กรอง QuerySet ทั้งก้อน** ให้เหลือเฉพาะ object ที่ user มีสิทธิ์เท่านั้น
`get_objects_for_user()` ทำสิ่งนี้โดยตรงและมีประสิทธิภาพกว่าการวน loop เช็คทีละตัวมาก:

```python
from guardian.shortcuts import get_objects_for_user

# คืน QuerySet ของ Post ทุกตัวที่ narin มีสิทธิ์ "change_post" (ทั้งจาก user และ Group)
editable_posts = get_objects_for_user(narin, "blog.change_post")
print(editable_posts)
# <QuerySet [<Post: kai-post>]>

# ระบุ any_perm=True เพื่อกรองด้วยเงื่อนไข "มีสิทธิ์อย่างน้อยหนึ่งตัวจากลิสต์" (OR logic)
posts_with_any_access = get_objects_for_user(
    narin, ["blog.change_post", "blog.delete_post"], any_perm=True
)

# ระบุ klass เมื่อ QuerySet ว่างเปล่าจนเดา Model ไม่ได้ (เช่นตอนกรองด้วย permission เดียว)
editable_posts = get_objects_for_user(narin, "blog.change_post", klass=Post)
```

`get_objects_for_user()` ทำงานโดยสร้าง SQL query เดียวที่ join กับตาราง
`guardian_userobjectpermission`/`guardian_groupobjectpermission` แทนที่จะดึงข้อมูล
ทั้งหมดมาเช็คทีละแถวใน Python — เราจะใช้ฟังก์ชันนี้เป็นหัวใจหลักของ ขั้นตอนที่ 376-378

### 374.5 ทำไม `{% if perms %}` จาก Part 033 ยังใช้กับ Object-level ตรง ๆ ไม่ได้

Part 033 (ขั้นตอนที่ 326.5) เตือนไว้แล้วว่า `{% if perms.blog.change_post %}` ใน
template ไม่รู้จัก object ที่อยู่ใน context เลย เพราะ `PermWrapper` เรียก
`user.has_perm(perm)` แบบไม่ส่ง `obj` เสมอ ต่อให้ติดตั้ง guardian แล้วก็ยังใช้ syntax
นี้ตรง ๆ ไม่ได้ — ทางแก้ที่ถูกต้องคือส่งค่า boolean ที่คำนวณแล้วจาก view เข้ามาใน
context โดยตรง:

```python
# blog/views.py
class PostDetailView(DetailView):
    model = Post
    template_name = "blog/post_detail.html"
    context_object_name = "post"

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        user = self.request.user
        context["can_edit_this_post"] = (
            user.is_authenticated and user.has_perm("blog.change_post", self.object)
        )
        context["can_delete_this_post"] = (
            user.is_authenticated and user.has_perm("blog.delete_post", self.object)
        )
        return context
```

```html
<!-- templates/blog/post_detail.html -->
{% if can_edit_this_post %}
    <a href="{% url 'blog:post-update' post.slug %}" class="btn btn-primary">แก้ไข</a>
{% endif %}
{% if can_delete_this_post %}
    <a href="{% url 'blog:post-delete' post.slug %}" class="btn btn-danger">ลบ</a>
{% endif %}
```

### 374.6 ตารางสรุป Shortcuts ทั้งหมดใน `guardian.shortcuts`

| ฟังก์ชัน | ใช้ตอบคำถามอะไร | คืนค่า |
|---|---|---|
| `assign_perm(perm, user_or_group, obj)` | มอบสิทธิ์ | `Permission`/`*ObjectPermission` instance |
| `remove_perm(perm, user_or_group, obj)` | ถอนสิทธิ์ | `None` |
| `get_perms(user_or_group, obj)` | "มีสิทธิ์อะไรบ้างกับ object นี้" | `list[str]` |
| `get_users_with_perms(obj, attach_perms=False)` | "ใครมีสิทธิ์กับ object นี้บ้าง" | `QuerySet` หรือ `dict` |
| `get_groups_with_perms(obj, attach_perms=False)` | "Group ไหนมีสิทธิ์กับ object นี้บ้าง" | `QuerySet` หรือ `dict` |
| `get_objects_for_user(user, perms, klass=None, any_perm=False)` | "object ไหนบ้างที่ user นี้มีสิทธิ์" | `QuerySet` |
| `get_objects_for_group(group, perms, klass=None)` | "object ไหนบ้างที่ Group นี้มีสิทธิ์" | `QuerySet` |

---

## ขั้นตอนที่ 375: ใช้ Object-level Permission ใน View/CBV จริง

### 375.1 ทบทวน `AuthorRequiredMixin` เดิมจาก Part 024

Part 024 (ขั้นตอนที่ 232-234) เขียน `AuthorRequiredMixin` ไว้ 2 เวอร์ชัน จบที่เวอร์ชัน
ที่ใช้ `UserPassesTestMixin`:

```python
# เวอร์ชันเดิมจาก Part 024 (ขั้นตอนที่ 234.2)
class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    def test_func(self):
        obj = self.get_object()
        return obj.author_id == self.request.user.id

    def handle_no_permission(self):
        if self.request.user.is_authenticated:
            raise PermissionDenied("คุณไม่ใช่เจ้าของโพสต์นี้ จึงไม่มีสิทธิ์ดำเนินการ")
        return super().handle_no_permission()
```

โค้ดนี้ **ใช้งานได้ถูกต้อง** และยังคงเป็นทางเลือกที่ดีสำหรับกรณีง่าย ๆ (จะเทียบให้เห็น
ชัดใน ขั้นตอนที่ 376) แต่มีข้อจำกัด 3 อย่างที่ Object-level Permission ของ guardian
แก้ได้:

1. **เช็คได้แค่ "เจ้าของ" เท่านั้น** ไม่รองรับการมอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของ
   (เช่น บรรณาธิการที่ได้รับมอบหมายให้ช่วยแก้โพสต์ของคนอื่นชั่วคราว)
2. **ไม่โผล่ในระบบ permission มาตรฐาน** — `get_perms()`, `{% if perms %}`,
   `get_users_with_perms()` ไม่รู้จัก logic นี้เลย เพราะมันเป็นแค่การเทียบ field ธรรมดา
3. **ต้อง hardcode field name `author`** ทำให้ reuse ข้าม Model ที่มีชื่อ field เจ้าของ
   ต่างกัน (เช่น `Comment.author` vs `Order.customer`) ยากกว่าที่ควร

### 375.2 เขียน `AuthorRequiredMixin` เวอร์ชันใหม่ด้วย django-guardian

แนวทางแรก: แทนที่การเทียบ field ตรง ๆ ด้วยการเรียก `has_perm()` ของ guardian โดยตรง
ในโครงสร้าง Mixin เดิมที่ยังคุ้นเคย (ยังคงเรียกว่า `AuthorRequiredMixin` เพื่อความ
ต่อเนื่องของโค้ดเดิม แต่เปลี่ยน implementation ข้างในทั้งหมด):

```python
# blog/mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.core.exceptions import PermissionDenied


class AuthorRequiredMixin(LoginRequiredMixin, UserPassesTestMixin):
    """
    อนุญาตเฉพาะ user ที่ล็อกอินแล้ว "และ" มี object-level permission ที่กำหนด
    กับ object นั้นจริง (ตรวจผ่าน django-guardian แทนการเทียบ field ตรง ๆ)

    ต้องกำหนด attribute `permission_required_guardian` เป็น permission string
    เต็มรูปแบบ (เช่น "blog.change_post") ในทุก View ที่นำ Mixin นี้ไปใช้
    """

    permission_required_guardian = None

    def get_permission_required_guardian(self):
        if self.permission_required_guardian is None:
            raise NotImplementedError(
                f"{self.__class__.__name__} ต้องกำหนด permission_required_guardian"
            )
        return self.permission_required_guardian

    def test_func(self):
        obj = self.get_object()
        perm = self.get_permission_required_guardian()
        # has_perm() ของ guardian จะ True ทั้งกรณีเป็นเจ้าของที่ได้รับสิทธิ์ตอนสร้าง
        # (ขั้นตอนที่ 373.4) และกรณีถูกมอบสิทธิ์เพิ่มเติมภายหลัง (delegated permission)
        return self.request.user.has_perm(perm, obj)

    def handle_no_permission(self):
        if self.request.user.is_authenticated:
            raise PermissionDenied("คุณไม่มีสิทธิ์ดำเนินการกับข้อมูลนี้")
        return super().handle_no_permission()
```

นำไปใช้กับ `PostUpdateView`/`PostDeleteView` โดยกำหนด permission ที่ต้องการต่อ View:

```python
# blog/views.py
from django.urls import reverse_lazy
from django.views.generic import DeleteView, UpdateView

from .forms import PostForm
from .mixins import AuthorRequiredMixin
from .models import Post


class PostUpdateView(AuthorRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    permission_required_guardian = "blog.change_post"


class PostDeleteView(AuthorRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/post_confirm_delete.html"
    success_url = reverse_lazy("blog:post-list")
    permission_required_guardian = "blog.delete_post"
```

สังเกตว่าลำดับ Mixin ยังต้องเป็นไปตามกฎ MRO จาก Part 024 (ขั้นตอนที่ 233) ทุกประการ:
`AuthorRequiredMixin` (ซึ่งภายในมี `LoginRequiredMixin` ประกอบอยู่แล้ว) ต้องอยู่ก่อน
`UpdateView`/`DeleteView` เสมอ

**ผลลัพธ์ที่ได้เพิ่มขึ้นจากเวอร์ชันเดิม**: ตอนนี้ถ้า admin หรือบรรณาธิการต้องการมอบสิทธิ์
แก้ไขโพสต์ของ kai ให้ narin ช่วยดูแลชั่วคราว เพียงแค่รัน
`assign_perm("change_post", narin, kai_post)` ผ่าน shell หรือ admin panel เท่านั้น
**ไม่ต้องแก้โค้ด view หรือ mixin เลยแม้แต่บรรทัดเดียว** ต่างจากเวอร์ชันเดิมที่ผูกกับ
`obj.author_id == request.user.id` ตายตัว ไม่มีทางขยายให้ narin ผ่านการเช็คนี้ได้เลย
นอกจากเปลี่ยน `author` ของโพสต์ตรง ๆ (ซึ่งผิดความหมายทางธุรกิจ)

### 375.3 `guardian.mixins.PermissionRequiredMixin` — ทางเลือกสำเร็จรูปจาก guardian เอง

guardian มี Mixin ของตัวเองชื่อ `PermissionRequiredMixin` ที่ทำงานคล้ายกับ
`django.contrib.auth.mixins.PermissionRequiredMixin` จาก Part 033 (ขั้นตอนที่ 324)
ทุกประการ แต่ตรวจสอบระดับ object โดยอัตโนมัติ (ดึง object ผ่าน `self.get_object()`
ให้เองโดยไม่ต้องเขียน `test_func()`):

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from guardian.mixins import PermissionRequiredMixin as GuardianPermissionRequiredMixin


class PostUpdateView(LoginRequiredMixin, GuardianPermissionRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"
    permission_required = "blog.change_post"
    return_403 = True          # คืน HTTP 403 แทน redirect ไป login เมื่อไม่ผ่าน
    accept_global_perms = True  # ถ้า user มี blog.change_post ระดับ model (เช่น เป็น editor) ก็ผ่านได้เช่นกัน
```

พารามิเตอร์ `accept_global_perms=True` คือจุดที่น่าสนใจที่สุด: มันทำให้ Mixin นี้เช็ค
**ทั้งสองระดับ** พร้อมกัน — ถ้า user มี permission ระดับ model ปกติ (จาก Group แบบ
Part 033) **หรือ** มี object-level permission (จาก guardian) อย่างใดอย่างหนึ่งก็ผ่าน
ทันที นี่คือจุดเริ่มต้นของการผสานสองกลยุทธ์เข้าด้วยกันที่จะเจาะลึกเต็มรูปแบบใน
ขั้นตอนที่ 377

### 375.4 ตารางเปรียบเทียบ 3 แนวทางสำหรับควบคุมการเข้าถึงระดับ Object

| แนวทาง | Field ที่ต้องมี | รองรับ Delegated Permission | โผล่ใน `get_perms()`/admin | ความซับซ้อนในการตั้งค่า |
|---|---|---|---|---|
| `AuthorRequiredMixin` เดิม (Part 024) — เทียบ field ตรง ๆ | ต้องมี field เจ้าของชัดเจน (`author`) | ❌ ไม่รองรับ | ❌ ไม่โผล่ | ต่ำที่สุด ไม่ต้องติดตั้งอะไรเพิ่ม |
| `AuthorRequiredMixin` เวอร์ชันใหม่ — เรียก `has_perm()` ของ guardian | ไม่จำเป็นต้องมี field เจ้าของเลย (ใช้ตาราง guardian แทน) | ✅ รองรับเต็มรูปแบบ | ✅ โผล่ | ปานกลาง ต้องมอบสิทธิ์ตอนสร้าง object |
| `guardian.mixins.PermissionRequiredMixin` สำเร็จรูป | เหมือนข้างบน | ✅ รองรับ + ผสาน model-level ได้ด้วย `accept_global_perms` | ✅ โผล่ | ต่ำสุดในบรรดาสองแบบที่ใช้ guardian (ไม่ต้องเขียน Mixin เอง) |

**คำแนะนำของหลักสูตรนี้**: ใช้ `guardian.mixins.PermissionRequiredMixin` สำเร็จรูปเป็น
ค่าเริ่มต้นเมื่อ logic การเช็คสิทธิ์ตรงไปตรงมา (เช็ค permission เดียวกับ object เดียว)
และเขียน Mixin เองแบบ ขั้นตอนที่ 375.2 เฉพาะเมื่อต้องการ custom logic เพิ่มเติมที่
สำเร็จรูปไม่รองรับ (เช่น ข้อความ error ที่ต่างกันตามสถานการณ์, logging พิเศษ)

---

## ขั้นตอนที่ 376: Row-level Permission แบบ Manual โดยไม่ใช้ Library

### 376.1 แนวคิด: กรอง QuerySet ด้วย Owner โดยตรง

ก่อนจะรีบพึ่ง `django-guardian` เสมอ ต้องเข้าใจว่ามีอีกแนวทางหนึ่งที่ **เรียบง่ายกว่า
มาก** และเพียงพอสำหรับกรณีส่วนใหญ่ที่พบในงานจริง: กรอง QuerySet ให้เหลือเฉพาะข้อมูล
ที่ user เป็นเจ้าของตั้งแต่ต้น แทนที่จะ query ข้อมูลทั้งหมดมาก่อนแล้วค่อยเช็คสิทธิ์
ทีละแถว

```python
# blog/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView, UpdateView

from .models import Post


class MyPostListView(LoginRequiredMixin, ListView):
    """แสดงเฉพาะโพสต์ของ user ที่ล็อกอินอยู่เท่านั้น"""

    model = Post
    template_name = "blog/my_post_list.html"
    context_object_name = "posts"

    def get_queryset(self):
        return Post.objects.filter(author=self.request.user).order_by("-created_at")
```

```python
class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def get_queryset(self):
        # จำกัด QuerySet ให้เหลือเฉพาะโพสต์ของ user นี้ตั้งแต่ระดับ query
        # ถ้า user พยายามเข้าถึงโพสต์ของคนอื่น get_object() จะยิง Http404 ให้อัตโนมัติ
        # (ไม่ใช่ PermissionDenied — ความแตกต่างนี้สำคัญ ดู 376.2)
        return Post.objects.filter(author=self.request.user)
```

### 376.2 404 vs 403 — ความแตกต่างเชิงความหมายที่ต้องตัดสินใจ

จุดที่น่าสนใจของแนวทาง `get_queryset()` คือ เมื่อ user B พยายามเข้าถึงโพสต์ของ user A
ผ่าน URL ที่รู้ slug (`/blog/kai-post/edit/`) ระบบจะยิง **`Http404`** (เพราะ
`get_object()` ของ `SingleObjectMixin` หาแถวที่ตรงเงื่อนไข `filter(author=user_b)`
ไม่เจอเลย ไม่ใช่เพราะ "เจอแต่ไม่มีสิทธิ์") ต่างจาก `AuthorRequiredMixin` ใน ขั้นตอนที่
375 ที่ยิง **`PermissionDenied` (403)** เพราะมันดึง object มาเจอก่อนแล้วค่อยเช็คสิทธิ์
ทีหลัง

| พฤติกรรม | 404 (ผ่าน `get_queryset()`) | 403 (ผ่าน Mixin เช็คสิทธิ์) |
|---|---|---|
| ความหมายที่ผู้ใช้ได้รับ | "ไม่พบข้อมูลนี้" (ซ่อนการมีอยู่ของข้อมูล) | "มีข้อมูลนี้อยู่ แต่คุณไม่มีสิทธิ์" |
| ความปลอดภัยเชิง Information Disclosure | ปลอดภัยกว่า — ผู้โจมตีเดา slug ที่มีอยู่จริงไม่ได้จากการสังเกต status code | เปิดเผยว่า object มีอยู่จริงในระบบ แม้จะเข้าถึงไม่ได้ |
| ใช้เมื่อไหร่ | ข้อมูลที่ผู้ใช้ไม่ควรรู้ด้วยซ้ำว่ามีอยู่ (เช่น draft ส่วนตัว, ข้อมูลการเงิน) | ข้อมูลที่เปิดเผยการมีอยู่ได้ (เช่น โพสต์ published ที่ทุกคนเห็นได้ แต่แก้ไขได้เฉพาะเจ้าของ) |

**คำแนะนำ**: สำหรับ Content ที่เปิดเผยต่อสาธารณะอยู่แล้ว (เช่น `Post` ที่ published)
ใช้ 403 (ของ ขั้นตอนที่ 375) จะสื่อความหมายชัดเจนกว่า ส่วนข้อมูลส่วนตัวที่ไม่ควรรู้ด้วย
ซ้ำว่ามีอยู่ (เช่น draft, ข้อความส่วนตัว) ใช้แนวทาง `get_queryset()` ที่คืน 404 จะ
ปลอดภัยกว่าในเชิง security

### 376.3 Manager/QuerySet Pattern เพื่อ Reuse ข้าม View

เพื่อไม่ให้ `Post.objects.filter(author=self.request.user)` กระจายซ้ำในหลาย view
(ผิดหลัก DRY ที่เรียนมาตั้งแต่ Part 001) ให้ย้าย logic นี้ไปไว้ที่ `QuerySet`/`Manager`
ของ Model โดยตรง:

```python
# blog/models.py
from django.conf import settings
from django.db import models


class PostQuerySet(models.QuerySet):
    def owned_by(self, user):
        """คืนเฉพาะโพสต์ที่ user เป็นเจ้าของ"""
        if not user.is_authenticated:
            return self.none()
        return self.filter(author=user)

    def editable_by(self, user):
        """
        กฎรวม: superuser/staff แก้ไขได้ทุกโพสต์ ส่วนคนทั่วไปแก้ไขได้เฉพาะของตัวเอง
        (ตัวอย่างการผสาน role-based เข้ากับ ownership-based ในที่เดียว)
        """
        if not user.is_authenticated:
            return self.none()
        if user.is_staff or user.is_superuser:
            return self.all()
        return self.filter(author=user)


class Post(models.Model):
    # ... fields เดิมทั้งหมดจาก Part 023-024/033 ...
    objects = PostQuerySet.as_manager()
```

```python
# blog/views.py
class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def get_queryset(self):
        return Post.objects.editable_by(self.request.user)
```

ตอนนี้กฎ "ใครแก้ไขโพสต์อะไรได้บ้าง" ถูกเก็บไว้ **ที่เดียว** (`PostQuerySet.editable_by`)
และนำไปใช้ซ้ำได้ทั้งใน view, management command, หรือ API ในอนาคต — ตรงตามหลักการ
Fat Models, Thin Views ที่เป็นแนวทางที่ดีของ Django

### 376.4 ตารางเปรียบเทียบ: Manual QuerySet Filtering vs django-guardian

| หัวข้อเปรียบเทียบ | Manual (`get_queryset()` + Manager) | django-guardian |
|---|---|---|
| ติดตั้ง Package เพิ่ม | ❌ ไม่ต้อง | ✅ ต้องติดตั้งและ migrate |
| ตารางฐานข้อมูลเพิ่ม | ❌ ไม่มี | ✅ มี 2 ตารางใหม่ (หรือมากกว่าถ้าใช้ Direct FK — ขั้นตอนที่ 378.3) |
| Query ต่อการเช็คสิทธิ์ 1 object | เร็วมาก (WHERE clause ธรรมดา) | ช้ากว่าเล็กน้อย (ต้อง JOIN กับตาราง guardian) |
| รองรับ Delegated Permission (มอบให้คนอื่นที่ไม่ใช่เจ้าของ) | ❌ ต้องเขียน logic เพิ่มเอง | ✅ รองรับโดยธรรมชาติ |
| รองรับสิทธิ์ต่างระดับต่อคนละ user บน object เดียวกัน (เช่น A แก้ได้ B แค่ดูได้) | ❌ ซับซ้อนมากถ้าทำเอง | ✅ ทำได้ตรง ๆ ด้วย `assign_perm` คนละ permission |
| แสดงผ่าน Django Admin/`{% if perms %}` | ❌ ไม่เชื่อมกับระบบ permission มาตรฐาน | ✅ เชื่อมเต็มรูปแบบ |
| Learning Curve | ต่ำ — ใช้ QuerySet ธรรมดาที่เรียนมาแล้ว | ปานกลาง — ต้องเข้าใจ backend และตาราง guardian เพิ่ม |
| เหมาะกับ | ระบบที่กฎ "เจ้าของเท่านั้น" เพียงพอ ไม่มีแผนขยายซับซ้อนกว่านี้ | ระบบที่ต้องมอบสิทธิ์ยืดหยุ่น หลาย role ทับซ้อนกัน |

### 376.5 กรอบการตัดสินใจ: เลือกทางไหนดี

ใช้คำถาม 3 ข้อนี้เพื่อตัดสินใจ:

1. **มีความต้องการมอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของหรือไม่ (Delegation)?** ถ้าไม่มี
   และไม่มีแผนจะมีในอนาคตอันใกล้ → ใช้ Manual QuerySet Filtering พอเพียง
2. **ต้องการแสดงสิทธิ์ผ่าน Django Admin หรือใช้ `has_perm()`/`{% if perms %}` แบบ
   มาตรฐานหรือไม่?** ถ้าใช่ → ต้องใช้ django-guardian เพราะ Manual filtering ไม่เชื่อม
   กับระบบ permission มาตรฐานเลย
3. **ระบบมีขนาดใหญ่มาก (object นับล้านแถว, permission ผสมซับซ้อนหลายชั้น) หรือไม่?**
   ถ้าใหญ่มากและกฎซับซ้อน → พิจารณา django-guardian แต่ต้องอ่าน ขั้นตอนที่ 378
   (Performance) อย่างละเอียดก่อน เพราะ Object-level permission ที่ scale ไม่ดีจะกลาย
   เป็นคอขวดของระบบได้ง่ายกว่าที่คิด

**กฎเหล็กของหลักสูตรนี้**: อย่าติดตั้ง `django-guardian` "เพราะมันดูโปร" ถ้าโจทย์จริง
คือ ownership ธรรมดา — เพิ่ม dependency และความซับซ้อนโดยไม่จำเป็นคือหนึ่งใน
anti-pattern ที่พบบ่อยที่สุดในทีมที่ over-engineer ระบบ permission

---

## ขั้นตอนที่ 377: ผสาน Group-based Permission + Object-level Permission เข้าด้วยกัน

### 377.1 โจทย์จริง: Editor เห็นทุกโพสต์ Author เห็นแค่ของตัวเอง

ระบบบล็อกระดับมืออาชีพมักมีกฎที่ซับซ้อนกว่า "เจ้าของเท่านั้น" เพียงอย่างเดียว ตัวอย่าง
กฎจริงที่ทีม editorial ส่วนใหญ่ใช้:

- **Author** (นักเขียน): แก้ไข/ลบได้เฉพาะโพสต์ของตัวเอง
- **Editor** (บรรณาธิการ): แก้ไข/ลบได้ **ทุกโพสต์** ในระบบ ไม่ว่าใครเป็นเจ้าของ
- **Reviewer** (ผู้ตรวจทาน): ได้รับมอบหมายให้ตรวจ **เฉพาะบางโพสต์** ที่ระบุเจาะจง
  (delegated permission ต่อ object) แม้จะไม่ใช่ทั้ง author และ editor

นี่คือสถานการณ์ที่ต้องผสาน **Group-based Permission** (จาก Part 033 — ใช้กับ Editor
ที่ต้องการสิทธิ์ครอบคลุมทั้งระบบ) เข้ากับ **Object-level Permission** (จาก guardian —
ใช้กับ Reviewer ที่ได้รับมอบหมายเฉพาะบาง object) พร้อมกัน

### 377.2 ออกแบบ Group และ Permission

```python
# blog/management/commands/seed_editorial_groups.py
from django.contrib.auth.models import Group, Permission
from django.core.management.base import BaseCommand

GROUP_PERMISSIONS = {
    "Editors": [
        "blog.add_post",
        "blog.change_post",   # ระดับ model — แก้ไขได้ "ทุกโพสต์" ในระบบ
        "blog.delete_post",
        "blog.view_post",
        "blog.can_publish_post",
    ],
    "Authors": [
        "blog.add_post",
        "blog.view_post",
        # หมายเหตุ: ไม่ให้ blog.change_post/delete_post ระดับ model กับ Authors
        # เพราะสิทธิ์แก้ไข/ลบของ Author จะมาจาก object-level permission
        # (มอบให้อัตโนมัติตอนสร้างโพสต์ ตาม ขั้นตอนที่ 373.4-373.5) แทน
    ],
    "Reviewers": [
        "blog.view_post",
        # Reviewer ไม่มีสิทธิ์ระดับ model เลยนอกจาก view — สิทธิ์แก้ไขจะมาจาก
        # assign_perm() แบบเจาะจงต่อโพสต์เท่านั้น (delegated object-level permission)
    ],
}


class Command(BaseCommand):
    help = "สร้าง/อัปเดต Group ตามโครงสร้างทีม editorial ของระบบ blog"

    def handle(self, *args, **options):
        for group_name, perm_strings in GROUP_PERMISSIONS.items():
            group, created = Group.objects.get_or_create(name=group_name)
            perms = []
            for perm_string in perm_strings:
                app_label, codename = perm_string.split(".")
                perms.append(
                    Permission.objects.get(content_type__app_label=app_label, codename=codename)
                )
            group.permissions.set(perms)
            status = "สร้างใหม่" if created else "อัปเดต"
            self.stdout.write(self.style.SUCCESS(f"{status} Group '{group_name}'"))
```

### 377.3 QuerySet ที่ผสานทั้งสองกลยุทธ์เข้าด้วยกัน

```python
# blog/models.py
from guardian.shortcuts import get_objects_for_user


class PostQuerySet(models.QuerySet):
    def visible_to(self, user):
        """
        กฎรวม 3 ชั้น:
        1. Editor (มี blog.change_post ระดับ model) เห็น/แก้ไขได้ทุกโพสต์
        2. Author เห็น/แก้ไขได้เฉพาะโพสต์ของตัวเอง (author == user)
        3. Reviewer ที่ได้รับมอบหมาย (object-level permission) เห็น/แก้ไขได้เฉพาะ
           โพสต์ที่ถูก assign_perm ให้เท่านั้น
        """
        if not user.is_authenticated:
            return self.none()

        if user.has_perm("blog.change_post"):
            # มี permission ระดับ model (เช่น เป็น Editor) -> เห็นทุกโพสต์ ไม่ต้อง query เพิ่ม
            return self.all()

        # รวม object ที่เป็นเจ้าของ กับ object ที่ได้รับมอบหมายผ่าน guardian เข้าด้วยกัน
        owned_ids = self.filter(author=user).values_list("pk", flat=True)
        delegated_qs = get_objects_for_user(
            user, "blog.change_post", klass=self.model, accept_global_perms=False
        )
        delegated_ids = delegated_qs.values_list("pk", flat=True)

        return self.filter(pk__in=set(owned_ids) | set(delegated_ids))


class Post(models.Model):
    # ... fields เดิม ...
    objects = PostQuerySet.as_manager()
```

**หมายเหตุสำคัญเรื่อง performance**: การ `set(owned_ids) | set(delegated_ids)` ในตัวอย่าง
นี้ evaluate ทั้งสอง QuerySet เข้า memory ก่อน ซึ่งเหมาะกับข้อมูลขนาดกลาง ถ้าข้อมูลใหญ่
มากควรเปลี่ยนเป็น `self.filter(Q(author=user) | Q(pk__in=delegated_ids))` เพื่อให้เหลือ
เป็น SQL query เดียวแทน (จะเจาะลึกการ optimize เพิ่มเติมใน ขั้นตอนที่ 378)

### 377.4 View ที่ใช้ QuerySet ผสาน

```python
# blog/views.py
class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = "blog/post_form.html"

    def get_queryset(self):
        return Post.objects.visible_to(self.request.user)
```

```python
class DashboardPostListView(LoginRequiredMixin, ListView):
    """
    หน้า Dashboard ที่แสดงโพสต์ตามสิทธิ์ของแต่ละคน:
    - Editor เห็นทุกโพสต์
    - Author เห็นเฉพาะของตัวเอง
    - Reviewer เห็นเฉพาะที่ได้รับมอบหมาย
    """

    model = Post
    template_name = "blog/dashboard.html"
    context_object_name = "posts"
    paginate_by = 20

    def get_queryset(self):
        return Post.objects.visible_to(self.request.user).select_related(
            "author", "category"
        ).order_by("-created_at")
```

### 377.5 Template แสดงปุ่มตาม Permission ผสม

```html
<!-- templates/blog/dashboard.html -->
{% extends "base.html" %}
{% block content %}
<h1>แดชบอร์ดโพสต์ของคุณ</h1>

<table class="table">
    <thead>
        <tr>
            <th>ชื่อโพสต์</th>
            <th>เจ้าของ</th>
            <th>สถานะ</th>
            <th>การจัดการ</th>
        </tr>
    </thead>
    <tbody>
        {% for post in posts %}
        <tr>
            <td>{{ post.title }}</td>
            <td>{{ post.author.username }}</td>
            <td>{{ post.get_status_display }}</td>
            <td>
                {% if user.is_authenticated and user|has_object_perm:"blog.change_post,post" %}
                    <a href="{% url 'blog:post-update' post.slug %}">แก้ไข</a>
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

Template ข้างต้นใช้ template filter สมมติชื่อ `has_object_perm` เพื่อความกระชับ
(guardian ไม่มี filter สำเร็จรูปสำหรับ syntax นี้โดยตรง) เขียนเองได้ง่าย ๆ ด้วย
custom template filter:

```python
# blog/templatetags/guardian_extras.py
from django import template

register = template.Library()


@register.filter(name="has_object_perm")
def has_object_perm(user, perm_and_obj):
    """
    ใช้งานใน template: {{ user|has_object_perm:"blog.change_post,post" }}
    รับ argument เป็น string "perm_name,context_variable_name" คั่นด้วย comma
    """
    perm, obj_name = perm_and_obj.split(",")
    # หมายเหตุ: filter รับได้แค่ argument เดียว ในทางปฏิบัติจริงแนะนำให้คำนวณ
    # boolean ไว้ใน get_context_data() แบบ ขั้นตอนที่ 374.5 แทน เพราะอ่านง่ายกว่ามาก
    return False  # placeholder — ดูคำแนะนำด้านล่าง
```

**คำแนะนำที่แนะนำจริง**: แม้จะเขียน custom filter ได้ แต่หลักสูตรนี้แนะนำแนวทางจาก
ขั้นตอนที่ 374.5 เสมอ (คำนวณ boolean ไว้ใน `get_context_data()` หรือใน `for` loop ของ
View แล้วแนบเป็น attribute ให้แต่ละ object) เพราะอ่านและ debug ง่ายกว่า custom filter
ที่ซ่อน logic การเช็คสิทธิ์ไว้ใน template มาก:

```python
class DashboardPostListView(LoginRequiredMixin, ListView):
    model = Post
    template_name = "blog/dashboard.html"
    context_object_name = "posts"

    def get_queryset(self):
        qs = Post.objects.visible_to(self.request.user).select_related("author")
        return qs

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        user = self.request.user
        # แนบ boolean ไว้กับแต่ละ object โดยตรง ใช้ ObjectPermissionChecker เพื่อไม่ให้เกิด
        # N+1 query (รายละเอียดเต็มรูปแบบอยู่ใน ขั้นตอนที่ 378.2)
        from guardian.core import ObjectPermissionChecker

        checker = ObjectPermissionChecker(user)
        posts = list(context["posts"])
        checker.prefetch_perms(posts)
        for post in posts:
            post.can_edit = user.has_perm("blog.change_post") or checker.has_perm(
                "change_post", post
            )
        context["posts"] = posts
        return context
```

```html
{% for post in posts %}
    ...
    {% if post.can_edit %}
        <a href="{% url 'blog:post-update' post.slug %}">แก้ไข</a>
    {% endif %}
{% endfor %}
```

---

## ขั้นตอนที่ 378: ผลกระทบด้าน Performance ของ Object-level Permission ที่ Scale ใหญ่

### 378.1 ปัญหา N+1 เมื่อเช็ค `has_perm()` ทีละ Object ใน List

ลองจินตนาการหน้า Dashboard ที่แสดงโพสต์ 50 รายการต่อหน้า และโค้ดเดิม (ก่อนปรับปรุง)
เช็คสิทธิ์แบบไร้เดียงสาในลูป:

```python
# เวอร์ชันที่มีปัญหา N+1 — อย่าเขียนแบบนี้!
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    for post in context["posts"]:
        post.can_edit = self.request.user.has_perm("blog.change_post", post)  # query ทุกครั้ง!
    return context
```

แต่ละครั้งที่เรียก `has_perm(perm, obj)` `ObjectPermissionBackend` จะยิง SQL query ไป
ยังตาราง `guardian_userobjectpermission`/`guardian_groupobjectpermission` **ใหม่ทุกครั้ง**
เพราะ Django ไม่ cache ผลลัพธ์ระดับ object เหมือนที่ cache ผลลัพธ์ระดับ model (ทบทวนจาก
Part 033 ขั้นตอนที่ 328 เรื่อง `_perm_cache`) — cache นั้นใช้ได้แค่กับการเช็คแบบไม่ส่ง
`obj` เท่านั้น ผลคือหน้าที่แสดง 50 โพสต์จะยิง query เพิ่มขึ้น **50 ครั้ง** เฉพาะสำหรับ
การเช็ค permission เพียงอย่างเดียว (ไม่นับ query อื่นของหน้า) — นี่คือปัญหา **N+1
Query** ที่ร้ายแรงกว่าปกติ เพราะเกิดขึ้นแบบเงียบ ๆ โดยไม่มี error ใด ๆ เตือน

### 378.2 `ObjectPermissionChecker` — เครื่องมือแก้ N+1 ของ guardian เอง

guardian มีคลาส `ObjectPermissionChecker` ที่ออกแบบมาเพื่อแก้ปัญหานี้โดยเฉพาะ ผ่านเมธอด
`prefetch_perms()` ที่ query สิทธิ์ของ object **ทั้งหมดในลิสต์พร้อมกันในครั้งเดียว**
แล้ว cache ผลลัพธ์ไว้ใน instance ของ checker เอง:

```python
from guardian.core import ObjectPermissionChecker

def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    posts = list(context["posts"])

    checker = ObjectPermissionChecker(self.request.user)
    checker.prefetch_perms(posts)   # <-- 1 query สำหรับทั้ง 50 โพสต์ แทนที่จะเป็น 50 query

    for post in posts:
        post.can_edit = checker.has_perm("change_post", post)   # อ่านจาก cache ใน memory ไม่ query ซ้ำ

    context["posts"] = posts
    return context
```

**ผลลัพธ์**: จาก 50 query เหลือเพียง **1 query** สำหรับการเช็คสิทธิ์ทั้งหน้า — นี่คือ
เทคนิคที่จำเป็นที่สุดเมื่อใช้ Object-level Permission ร่วมกับ `ListView`/Dashboard ใด ๆ
ที่มีจำนวนแถวมากกว่าหลักสิบ

### 378.3 Generic Relation vs Direct Foreign Key — เทคนิค Optimize ระดับ Schema

ปัญหาที่ลึกกว่านั้นคือโครงสร้างตารางเริ่มต้นของ guardian (ที่อธิบายใน ขั้นตอนที่
372.3-372.4) ใช้ **Generic Relation** (ผ่าน `content_type_id` + `object_pk` แบบ string)
ซึ่งแม้จะยืดหยุ่นรองรับทุก Model แต่ก็ **ช้ากว่า Foreign Key ตรง ๆ อย่างมีนัยสำคัญ** เมื่อ
ข้อมูลมีขนาดใหญ่ เพราะ:

- `object_pk` เป็น `CharField` ไม่ใช่ `IntegerField` — การเทียบค่าและ index ทำงานช้ากว่า
- Query ต้อง join ผ่าน `ContentType` เพิ่มอีกชั้นหนึ่งเสมอ
- Index บนตาราง generic relation ไม่สามารถเจาะจงเฉพาะ Model ใดได้ (ต้อง index รวมทุก
  Model ที่ใช้ guardian ปนกันในตารางเดียว)

guardian แก้ปัญหานี้ให้ด้วยการรองรับ **Direct Foreign Key Models** — สร้างตาราง
`UserObjectPermission`/`GroupObjectPermission` เฉพาะของแต่ละ Model เอง โดยสืบทอดจาก
base class ที่ guardian เตรียมไว้ให้:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from guardian.models import GroupObjectPermissionBase, UserObjectPermissionBase


class Post(models.Model):
    # ... fields เดิมทั้งหมด ...
    pass


class PostUserObjectPermission(UserObjectPermissionBase):
    """
    ตาราง Object-level permission เฉพาะของ Post ที่ผูกกับ Foreign Key ตรง ๆ
    (content_object เป็น ForeignKey ธรรมดา ไม่ใช่ Generic Relation) เร็วกว่ามาก
    เมื่อข้อมูลมีจำนวนมาก
    """

    content_object = models.ForeignKey(Post, on_delete=models.CASCADE)


class PostGroupObjectPermission(GroupObjectPermissionBase):
    content_object = models.ForeignKey(Post, on_delete=models.CASCADE)
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

หลังจากนี้ guardian จะ **ตรวจจับอัตโนมัติ** ว่า `Post` มีตาราง permission เฉพาะของ
ตัวเองแล้ว (ผ่านการค้นหา Model ที่สืบทอดจาก `UserObjectPermissionBase`/
`GroupObjectPermissionBase` ที่มี FK ชื่อ `content_object` ชี้มาที่ `Post`) และเปลี่ยน
ไปใช้ตารางเฉพาะนี้แทนตาราง generic กลาง (`guardian_userobjectpermission`) โดยอัตโนมัติ
ทั้งหมด **โดยไม่ต้องแก้โค้ดที่เรียก `assign_perm()`/`has_perm()`/`get_objects_for_user()`
แม้แต่บรรทัดเดียว** — นี่คือการออกแบบ API ที่ดีมาก เพราะ optimization เกิดขึ้นที่ระดับ
schema ล้วน ๆ โดยไม่กระทบ business logic ที่เขียนไว้แล้วเลย

### 378.4 ตารางเปรียบเทียบ Generic Relation vs Direct Foreign Key

| หัวข้อ | Generic Relation (ค่า default) | Direct Foreign Key |
|---|---|---|
| ตารางที่ใช้ | `guardian_userobjectpermission` ตัวเดียวรวมทุก Model | ตารางแยกต่อ Model เช่น `blog_postuserobjectpermission` |
| ชนิดของ `object_pk`/FK | `CharField` (string) | `ForeignKey` จริง ชี้ตรงไปที่ Model |
| ความเร็ว Query เมื่อข้อมูลมาก | ช้าลงเรื่อย ๆ ตามจำนวนรวมของทุก Model ที่ใช้ guardian | เร็วและคงที่ เพราะ index เฉพาะทางของ Model นั้น |
| ต้องเขียน Migration เพิ่ม | ❌ ไม่ต้อง (ใช้ตารางกลางที่มีอยู่แล้ว) | ✅ ต้องสร้าง Model และ migration เอง |
| เหมาะกับ | โปรเจกต์เล็ก-กลาง, prototype, Model ที่มี object permission น้อย | โปรเจกต์ production ที่มีข้อมูลระดับหมื่น-ล้านแถว หรือ Model ที่ถูกเช็ค permission บ่อยมาก |

**คำแนะนำ**: เริ่มต้นด้วย Generic Relation เสมอ (ค่า default ไม่ต้องตั้งค่าอะไรเพิ่ม)
แล้วค่อยย้ายไปใช้ Direct Foreign Key เฉพาะ Model ที่วัดผลจริงแล้วพบว่าเป็นคอขวด (ตาม
หลักการ "อย่า optimize ก่อนวัดผล" ที่จะเรียนเต็มรูปแบบใน Phase 8: Performance
Optimization)

### 378.5 Database Index และการ Cache ผลการเช็คสิทธิ์

นอกจาก Direct Foreign Key แล้ว ยังมีเทคนิคเสริมอีก 2 อย่างที่ช่วยเรื่อง performance:

1. **Composite Index บนคอลัมน์ที่ query บ่อย** — guardian สร้าง index พื้นฐานให้อยู่แล้ว
   แต่ถ้ามี query pattern เฉพาะทางที่ query บ่อยมาก (เช่น query ตาม `permission_id` +
   `object_pk` คู่กันเสมอ) ควรเพิ่ม composite index เองผ่าน `Meta.indexes`:

```python
class PostUserObjectPermission(UserObjectPermissionBase):
    content_object = models.ForeignKey(Post, on_delete=models.CASCADE)

    class Meta:
        indexes = [
            models.Index(fields=["content_object", "permission", "user"]),
        ]
```

2. **Cache ผลการเช็คสิทธิ์ด้วย Django Cache Framework** — สำหรับ permission ที่ไม่
   เปลี่ยนแปลงบ่อย (เช่น สิทธิ์ของ Editor ที่คงที่เป็นเวลานาน) สามารถ cache ผลลัพธ์ของ
   `has_perm()` ไว้ชั่วคราวได้:

```python
from django.core.cache import cache


def user_can_edit_post(user, post):
    cache_key = f"perm:change_post:user:{user.pk}:post:{post.pk}"
    cached_result = cache.get(cache_key)
    if cached_result is not None:
        return cached_result

    result = user.has_perm("blog.change_post") or user.has_perm("blog.change_post", post)
    cache.set(cache_key, result, timeout=300)  # cache 5 นาที
    return result
```

**ข้อควรระวังสำคัญ**: การ cache permission มีความเสี่ยงด้าน **security ถ้า invalidate
cache ไม่ถูกต้อง** (เช่น ถอนสิทธิ์ไปแล้วแต่ cache ยังบอกว่ามีสิทธิ์อยู่จนกว่าจะหมดอายุ)
ต้องเรียก `cache.delete(cache_key)` ทุกครั้งที่มีการ `assign_perm()`/`remove_perm()`
กับ user-object คู่นั้น มิฉะนั้นจะเกิดช่องโหว่ที่ผู้ใช้ยังคงมีสิทธิ์อยู่ชั่วคราวหลังจาก
ถูกถอนสิทธิ์ไปแล้วจริง — เทคนิคนี้จึงควรใช้เฉพาะเมื่อวัดผลแล้วว่าจำเป็นจริง ๆ เท่านั้น

### 378.6 ตารางสรุปแนวทาง Optimize ตามลำดับที่ควรทำ

| ลำดับ | เทคนิค | แก้ปัญหาอะไร | ความซับซ้อนในการทำ |
|---|---|---|---|
| 1 | ใช้ `get_objects_for_user()` แทนการ loop เช็คทีละ object | ลด query จาก N เหลือ 1 สำหรับการกรอง QuerySet | ต่ำ |
| 2 | ใช้ `ObjectPermissionChecker` + `prefetch_perms()` | ลด query N+1 เมื่อต้องเช็คสิทธิ์ของหลาย object ที่ query มาแล้ว | ต่ำ |
| 3 | เปลี่ยนไปใช้ Direct Foreign Key Models | ลดเวลาต่อ query ที่ตารางใหญ่ขึ้นเรื่อย ๆ | ปานกลาง (ต้องเขียน Migration) |
| 4 | เพิ่ม Composite Index เฉพาะทาง | เร่งความเร็ว query pattern เฉพาะจุดที่วัดผลว่าช้า | ปานกลาง (ต้องรู้ query pattern จริง) |
| 5 | Cache ผลการเช็คสิทธิ์ | ลด query ซ้ำสำหรับสิทธิ์ที่ไม่ค่อยเปลี่ยน | สูง (ต้องจัดการ cache invalidation ให้ถูกต้อง) |

**กฎเหล็ก**: ทำตามลำดับ 1 → 5 เสมอ อย่ากระโดดไปทำข้อ 5 (caching) ก่อนที่จะลองข้อ 1-2
ซึ่งง่ายกว่ามากและมักแก้ปัญหา 90% ของกรณีจริงได้แล้วโดยไม่ต้องแตะ cache invalidation
ที่มีความเสี่ยงด้าน security เลย

---

## ขั้นตอนที่ 379: การเขียน Test สำหรับ Permission Logic ทั้งหมด

### 379.1 Test พื้นฐาน: `assign_perm()` และ `has_perm()`

```python
# blog/tests/test_object_permissions.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from guardian.shortcuts import assign_perm, get_perms, remove_perm

from blog.models import Post

User = get_user_model()


class ObjectPermissionBasicTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.narin = User.objects.create_user(username="narin", password="pass1234")
        cls.kai = User.objects.create_user(username="kai", password="pass1234")
        cls.post = Post.objects.create(
            title="โพสต์ของ Kai", slug="kai-post", content="...", author=cls.kai,
        )

    def test_user_has_no_permission_by_default(self):
        self.assertFalse(self.narin.has_perm("blog.change_post", self.post))

    def test_assign_perm_grants_object_level_access(self):
        assign_perm("change_post", self.narin, self.post)
        self.assertTrue(self.narin.has_perm("blog.change_post", self.post))

    def test_permission_is_scoped_to_specific_object_only(self):
        other_post = Post.objects.create(
            title="โพสต์อีกอัน", slug="other-post", content="...", author=self.kai,
        )
        assign_perm("change_post", self.narin, self.post)

        self.assertTrue(self.narin.has_perm("blog.change_post", self.post))
        self.assertFalse(self.narin.has_perm("blog.change_post", other_post))

    def test_remove_perm_revokes_access(self):
        assign_perm("change_post", self.narin, self.post)
        remove_perm("change_post", self.narin, self.post)
        self.assertFalse(self.narin.has_perm("blog.change_post", self.post))

    def test_get_perms_lists_all_granted_object_permissions(self):
        assign_perm("change_post", self.narin, self.post)
        assign_perm("delete_post", self.narin, self.post)
        self.assertCountEqual(
            get_perms(self.narin, self.post), ["change_post", "delete_post"]
        )
```

### 379.2 Test `AuthorRequiredMixin` เวอร์ชันใหม่ (จาก ขั้นตอนที่ 375.2)

```python
# blog/tests/test_mixins.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse
from guardian.shortcuts import assign_perm

from blog.models import Post

User = get_user_model()


class AuthorRequiredMixinGuardianTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.author = User.objects.create_user(username="author", password="pass1234")
        cls.stranger = User.objects.create_user(username="stranger", password="pass1234")
        cls.reviewer = User.objects.create_user(username="reviewer", password="pass1234")
        cls.post = Post.objects.create(
            title="โพสต์ทดสอบ", slug="post-tests", content="...", author=cls.author,
        )
        # จำลองการมอบสิทธิ์อัตโนมัติตอนสร้างโพสต์ (เหมือน signal ใน ขั้นตอนที่ 373.5)
        assign_perm("change_post", cls.author, cls.post)
        assign_perm("delete_post", cls.author, cls.post)

    def test_author_can_edit_own_post(self):
        self.client.login(username="author", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_stranger_without_any_permission_gets_403(self):
        self.client.login(username="stranger", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 403)

    def test_delegated_reviewer_can_edit_after_being_granted_permission(self):
        # ก่อนได้รับมอบหมาย -> ต้องถูกปฏิเสธ
        self.client.login(username="reviewer", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 403)

        # มอบสิทธิ์เฉพาะโพสต์นี้ให้ reviewer (delegated object-level permission)
        assign_perm("change_post", self.reviewer, self.post)

        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_reviewer_permission_does_not_leak_to_other_posts(self):
        other_post = Post.objects.create(
            title="โพสต์อื่น", slug="another-post", content="...", author=self.author,
        )
        assign_perm("change_post", self.reviewer, self.post)   # มอบเฉพาะ self.post เท่านั้น

        self.client.login(username="reviewer", password="pass1234")
        url = reverse("blog:post-update", kwargs={"slug": other_post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 403)
```

สังเกตว่า `test_delegated_reviewer_can_edit_after_being_granted_permission` และ
`test_reviewer_permission_does_not_leak_to_other_posts` เป็น test ที่ **พิสูจน์ข้อดี
หลักของการย้ายไปใช้ guardian** — เวอร์ชันเดิมจาก Part 024 ที่เทียบ `author` ตรง ๆ ไม่
สามารถผ่าน test เหล่านี้ได้เลย เพราะไม่มีแนวคิดเรื่อง delegated permission อยู่ในระบบ

### 379.3 Test QuerySet ที่ผสาน Group-based + Object-level (จาก ขั้นตอนที่ 377)

```python
# blog/tests/test_visible_to_queryset.py
from django.contrib.auth import get_user_model
from django.contrib.auth.models import Group, Permission
from django.test import TestCase
from guardian.shortcuts import assign_perm

from blog.models import Post

User = get_user_model()


class VisibleToQuerySetTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = User.objects.create_user(username="editor", password="pass1234")
        cls.author_a = User.objects.create_user(username="author_a", password="pass1234")
        cls.author_b = User.objects.create_user(username="author_b", password="pass1234")
        cls.reviewer = User.objects.create_user(username="reviewer", password="pass1234")

        editors_group = Group.objects.create(name="Editors")
        editors_group.permissions.add(
            Permission.objects.get(content_type__app_label="blog", codename="change_post")
        )
        cls.editor.groups.add(editors_group)

        cls.post_a = Post.objects.create(
            title="โพสต์ A", slug="post-a", content="...", author=cls.author_a,
        )
        cls.post_b = Post.objects.create(
            title="โพสต์ B", slug="post-b", content="...", author=cls.author_b,
        )
        assign_perm("change_post", cls.reviewer, cls.post_a)

    def test_editor_sees_every_post(self):
        visible = Post.objects.visible_to(self.editor)
        self.assertEqual(set(visible), {self.post_a, self.post_b})

    def test_author_sees_only_own_post(self):
        visible = Post.objects.visible_to(self.author_a)
        self.assertEqual(set(visible), {self.post_a})

    def test_reviewer_sees_only_delegated_post(self):
        visible = Post.objects.visible_to(self.reviewer)
        self.assertEqual(set(visible), {self.post_a})

    def test_anonymous_user_sees_nothing(self):
        from django.contrib.auth.models import AnonymousUser

        visible = Post.objects.visible_to(AnonymousUser())
        self.assertEqual(list(visible), [])
```

### 379.4 Test Performance Regression ด้วย `assertNumQueries`

การเช็คว่าโค้ดไม่เผลอกลับไปมีปัญหา N+1 (จาก ขั้นตอนที่ 378.1) สำคัญพอ ๆ กับการเช็ค
ความถูกต้องของ logic — เขียน test ที่ **ล้มเหลวทันทีถ้ามีคนแก้โค้ดแล้วทำให้เกิด N+1
กลับมาอีก**:

```python
# blog/tests/test_permission_performance.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from guardian.shortcuts import assign_perm

from blog.models import Post

User = get_user_model()


class ObjectPermissionCheckerPerformanceTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(username="tester", password="pass1234")
        cls.posts = []
        for i in range(30):
            post = Post.objects.create(
                title=f"โพสต์ {i}", slug=f"post-{i}", content="...", author=cls.user,
            )
            assign_perm("change_post", cls.user, post)
            cls.posts.append(post)

    def test_prefetch_perms_uses_single_query_regardless_of_post_count(self):
        from guardian.core import ObjectPermissionChecker

        posts = list(Post.objects.all())

        with self.assertNumQueries(1):
            checker = ObjectPermissionChecker(self.user)
            checker.prefetch_perms(posts)
            for post in posts:
                checker.has_perm("change_post", post)

    def test_naive_loop_without_prefetch_causes_n_plus_one(self):
        posts = list(Post.objects.all())

        # ตั้งใจพิสูจน์ว่าเวอร์ชัน "ไร้เดียงสา" ยิง query เท่ากับจำนวนโพสต์จริง
        # (test นี้มีไว้เพื่อการศึกษา ไม่ควรมีโค้ดจริงแบบนี้ในระบบ production)
        with self.assertNumQueries(len(posts)):
            for post in posts:
                self.user.has_perm("blog.change_post", post)
```

Test ตัวที่สอง (`test_naive_loop_without_prefetch_causes_n_plus_one`) เป็นตัวอย่างที่ดี
ของการเขียน test เพื่อ **ยืนยันความเข้าใจ** ว่าปัญหาที่อธิบายไว้ใน ขั้นตอนที่ 378.1 นั้น
เกิดขึ้นจริงในทางปฏิบัติ ไม่ใช่แค่ทฤษฎี

### 379.5 ตารางสรุปสิ่งที่ควรมี Test ครอบคลุมเสมอ

| สถานการณ์ที่ต้อง Test | ทำไมสำคัญ |
|---|---|
| User ไม่มีสิทธิ์ใด ๆ โดย default | ป้องกัน false positive ที่ทำให้คิดว่าระบบปลอดภัยทั้งที่ไม่ได้เช็คจริง |
| `assign_perm()` แล้วสิทธิ์ใช้ได้จริง | พิสูจน์ happy path พื้นฐาน |
| สิทธิ์ไม่ "รั่ว" ไปยัง object อื่นที่ไม่ได้ assign | ป้องกันช่องโหว่ที่ร้ายแรงที่สุดของ object-level permission |
| `remove_perm()` ถอนสิทธิ์ได้จริง | พิสูจน์ว่าการเพิกถอนสิทธิ์ทำงานถูกต้อง (สำคัญมากสำหรับ security) |
| Delegated permission (มอบให้คนที่ไม่ใช่เจ้าของ) | พิสูจน์ข้อดีหลักที่ทำให้เลือกใช้ guardian แทน ownership เดิม |
| QuerySet ที่ผสาน Group-based + Object-level ให้ผลถูกต้องตาม role | ป้องกันบั๊กจากการผสาน logic หลายชั้น |
| จำนวน Query ไม่เพิ่มขึ้นตามจำนวน object (N+1 regression) | ป้องกันปัญหา performance กลับมาโดยไม่มีใครสังเกตเห็น |

---

## ขั้นตอนที่ 380: สรุป Phase 4 ทั้งหมด, Quiz, แบบฝึกหัดใหญ่ปิดท้าย Phase และก้าวสู่ Phase 5

### 380.1 ทบทวน Phase 4 ทั้งหมด (Part 031-038) ทีละ Part

Phase 4 พาคุณเดินทางจาก "ใครคือผู้ใช้คนนี้" ไปจนถึง "ผู้ใช้คนนี้ทำอะไรกับข้อมูลแถวไหน
ได้บ้าง" ครบทุกมิติของระบบ Authentication และ Authorization ระดับมืออาชีพ:

| Part | หัวข้อ | สิ่งที่ได้เรียนรู้หลัก |
|---|---|---|
| **031** | Django Authentication System เบื้องต้น | `login()`/`logout()`, `AuthenticationForm`, `LoginRequiredMixin`, `request.user`, session-based auth flow พื้นฐาน |
| **032** | Custom User Model | เหตุผลที่ **ต้อง** สร้าง `AbstractUser`/`AbstractBaseUser` ตั้งแต่ต้นโปรเจกต์, `AUTH_USER_MODEL`, `UserManager` เอง |
| **033** | Permissions และ Groups | `has_perm()`, `Group`, Custom Permission ผ่าน `Meta.permissions`, `PermissionRequiredMixin`, Permission Caching, ข้อจำกัดเรื่อง model-level |
| **034** | Django Sessions และ Cookies | `SessionMiddleware`, Session Backend 4 แบบ, `request.session`, ความปลอดภัยของ cookie, ป้องกัน Session Hijacking |
| **035** | Password Management, Reset และ Security | Password Hashers, `PasswordResetView` flow เต็มรูปแบบ, Password Validators, การบังคับเปลี่ยนรหัสผ่าน |
| **036** | Social Authentication (OAuth, django-allauth) | OAuth2 flow, ติดตั้ง `django-allauth`, เชื่อมต่อ Google/GitHub login, การจัดการ `SocialAccount` |
| **037** | Two-Factor Authentication | TOTP, `django-otp`/`django-two-factor-auth`, QR Code enrollment, Backup codes |
| **038** (Part นี้) | Row-Level Permissions และ Object-Level Permission | `django-guardian`, `assign_perm()`/`has_perm(obj=...)`, ผสาน Group-based + Object-level, Performance optimization |

### 380.2 แผนภาพภาพรวมของระบบ Auth + Authz ที่สมบูรณ์

```
                         ┌─────────────────────────────────────────┐
                         │   1. ผู้ใช้กรอก username/password        │
                         │      (Part 031: AuthenticationForm)      │
                         └───────────────────┬───────────────────────┘
                                              │
                     ┌────────────────────────▼────────────────────────┐
                     │  2. ตรวจสอบผ่าน Custom User Model                │
                     │     (Part 032: AUTH_USER_MODEL)                  │
                     │     + Password Hasher (Part 035)                 │
                     │     + (ถ้าเปิดใช้) OAuth ผ่าน allauth (Part 036) │
                     └────────────────────────┬────────────────────────┘
                                              │
                     ┌────────────────────────▼────────────────────────┐
                     │  3. ถ้าเปิดใช้ 2FA: ยืนยัน TOTP code เพิ่ม        │
                     │     (Part 037: django-otp)                        │
                     └────────────────────────┬────────────────────────┘
                                              │
                     ┌────────────────────────▼────────────────────────┐
                     │  4. login() สำเร็จ -> สร้าง Session               │
                     │     (Part 034: SessionMiddleware, sessionid cookie)│
                     └────────────────────────┬────────────────────────┘
                                              │
              ┌───────────────────────────────┼───────────────────────────────┐
              │                               │                               │
   ┌──────────▼──────────┐        ┌───────────▼──────────┐       ┌───────────▼──────────┐
   │ 5a. Model-level      │        │ 5b. Object-level      │       │ 5c. Ownership-based   │
   │ Permission           │        │ Permission            │       │ (Manual filtering)    │
   │ (Part 033: Group,    │        │ (Part 038: guardian,  │       │ (Part 038: get_queryset│
   │ has_perm(), @perm_   │        │ assign_perm(),        │       │ + PostQuerySet)       │
   │ required)            │        │ has_perm(obj=...))    │       │                       │
   └──────────┬──────────┘        └───────────┬──────────┘       └───────────┬──────────┘
              │                               │                               │
              └───────────────────────────────┼───────────────────────────────┘
                                              │
                                   ┌───────────▼───────────┐
                                   │  6. View/CBV ตัดสินใจ  │
                                   │  อนุญาต/ปฏิเสธ request  │
                                   │  (403 / 404 / render)  │
                                   └────────────────────────┘
```

### 380.3 Quiz ทบทวน Phase 4 (12 ข้อ พร้อมเฉลย)

**คำถามที่ 1**: Authentication กับ Authorization ต่างกันอย่างไร?
> **เฉลย**: Authentication ตอบคำถาม "คุณเป็นใคร" (ยืนยันตัวตน) ส่วน Authorization
> ตอบคำถาม "คุณทำสิ่งนี้ได้หรือไม่" (อนุญาตสิทธิ์) — `request.user.is_authenticated`
> ตอบข้อแรก `request.user.has_perm(...)` ตอบข้อหลัง

**คำถามที่ 2**: ทำไมโปรเจกต์ Django ทุกโปรเจกต์ควรสร้าง Custom User Model ตั้งแต่ต้น
แม้จะยังไม่รู้ว่าจะต้องขยาย field อะไรในอนาคต?
> **เฉลย**: เพราะ `AUTH_USER_MODEL` **เปลี่ยนหลัง migrate ครั้งแรกไปแล้วไม่ได้**
> (ต้องสร้างฐานข้อมูลใหม่หรือทำ migration ที่ซับซ้อนมาก) การสร้าง Custom User Model
> ตั้งแต่ต้น (แม้จะยังเหมือน `AbstractUser` เดิมทุกประการ) เปิดทางให้ขยายในอนาคตได้
> โดยไม่ต้อง migrate ฐานข้อมูลใหม่ทั้งหมด

**คำถามที่ 3**: `user.has_perm("blog.change_post")` กับ
`user.has_perm("blog.change_post", post)` ต่างกันอย่างไรเมื่อใช้ backend มาตรฐาน
(`ModelBackend`) เพียงตัวเดียว (ไม่มี guardian)?
> **เฉลย**: แบบแรกเช็คว่า user มีสิทธิ์แก้ไข `Post` **ตัวไหนก็ได้** ในระบบหรือไม่
> (model-level) แบบที่สอง (ส่ง `obj`) จะได้ผลลัพธ์ `False` **เสมอ** เพราะ `ModelBackend`
> ปฏิเสธที่จะตอบคำถามระดับ object เลย ไม่ว่า user จะมีสิทธิ์ระดับ model หรือไม่ก็ตาม

**คำถามที่ 4**: `SESSION_COOKIE_AGE` กับ `SESSION_EXPIRE_AT_BROWSER_CLOSE` ต่างกัน
อย่างไร และถ้าตั้งทั้งสองค่าพร้อมกันจะเกิดอะไรขึ้น?
> **เฉลย**: `SESSION_COOKIE_AGE` กำหนดอายุ cookie เป็นวินาที (default 1209600 = 2 สัปดาห์)
> ส่วน `SESSION_EXPIRE_AT_BROWSER_CLOSE=True` ทำให้ cookie หมดอายุทันทีที่ปิดเบราว์เซอร์
> ถ้าตั้ง `SESSION_EXPIRE_AT_BROWSER_CLOSE=True` ค่า `SESSION_COOKIE_AGE` จะถูก**ละเลย**
> เพราะ Django จะไม่ส่งค่า `Max-Age`/`Expires` ให้ cookie เลย ทำให้เป็น session cookie
> (หมดอายุเมื่อปิด browser) โดยอัตโนมัติ

**คำถามที่ 5**: เพราะเหตุใด Password Hasher ของ Django (เช่น PBKDF2, Argon2) จึงออกแบบ
มาให้ **ช้า** โดยตั้งใจ?
> **เฉลย**: เพื่อป้องกัน Brute-force/Dictionary attack — ถ้า hash เร็วเกินไป ผู้โจมตี
> ที่ได้ database dump ไปจะสามารถลองรหัสผ่านนับพันล้านชุดต่อวินาทีได้ ความช้าที่ตั้งใจ
> (ปรับผ่าน "iterations"/"work factor") ทำให้การลองรหัสผ่านจำนวนมากใช้เวลานานเกินคุ้ม

**คำถามที่ 6**: OAuth2 Authorization Code Flow ต่างจากการ login ด้วย username/password
ตรงไหนในเชิงความปลอดภัย?
> **เฉลย**: ผู้ใช้ login ที่ผู้ให้บริการ (เช่น Google) โดยตรง แอปของเราไม่เคยเห็นรหัสผ่าน
> ของผู้ใช้เลย ได้รับเพียง "authorization code" ที่แลกเป็น access token ในภายหลัง — ลด
> ความเสี่ยงที่แอปจะเก็บ/รั่วไหลรหัสผ่านของผู้ใช้โดยตรง และให้ผู้ใช้ควบคุมสิทธิ์
> (scope) ที่มอบให้แอปได้ละเอียดกว่า

**คำถามที่ 7**: TOTP (Time-based One-Time Password) ที่ใช้ใน 2FA ทำงานโดยไม่ต้องมี
การเชื่อมต่ออินเทอร์เน็ตระหว่างแอป authenticator กับ server ได้อย่างไร?
> **เฉลย**: ทั้งฝั่ง server และแอป authenticator (เช่น Google Authenticator) มี "secret
> key" ร่วมกันตั้งแต่ตอน enroll (ผ่าน QR code) แล้วคำนวณ OTP จาก secret key + เวลาปัจจุบัน
> (ปัดเป็นช่วง 30 วินาที) ด้วยอัลกอริทึมเดียวกัน (HMAC-based) ทั้งสองฝั่ง — ตราบใดที่
> นาฬิกาของทั้งสองฝั่งตรงกัน (หรือใกล้เคียงพอ) ผลลัพธ์ code จะตรงกันโดยไม่ต้องสื่อสารกัน
> แบบ real-time เลย

**คำถามที่ 8**: ทำไม `ModelBackend` ถึงคืนค่า `False`/`set()` ว่างทันทีเมื่อมีการส่ง
`obj` เข้าไปใน `has_perm()`/`get_all_permissions()` แทนที่จะ raise exception หรือ
เพิกเฉยแล้วเช็คแบบ model-level แทน?
> **เฉลย**: เพราะทีมพัฒนา Django ออกแบบให้ parameter `obj` เป็น **hook** สำหรับ backend
> อื่นที่รองรับ object-level permission (เช่น `ObjectPermissionBackend` ของ guardian)
> `ModelBackend` จึงเลือก "ปฏิเสธที่จะตอบ" (คืนค่าว่าง) แทนที่จะแกล้งตอบแบบ model-level
> เพื่อไม่ให้เกิดผลลัพธ์ที่คลุมเครือ — การไล่เช็คทีละ backend ใน
> `AUTHENTICATION_BACKENDS` จะทำงานถูกต้องได้ก็ต่อเมื่อแต่ละ backend ตอบเฉพาะสิ่งที่
> ตัวเองรับผิดชอบจริง ๆ

**คำถามที่ 9**: `guardian.shortcuts.get_objects_for_user()` ต่างจากการวน loop เช็ค
`has_perm()` ทีละ object อย่างไรในแง่ performance?
> **เฉลย**: `get_objects_for_user()` สร้าง SQL query เดียวที่ join กับตาราง permission
> ของ guardian โดยตรง คืนค่าเป็น QuerySet ที่กรองแล้ว ในขณะที่การวน loop เช็คทีละ
> object จะยิง query แยกสำหรับแต่ละ object (ปัญหา N+1) — ยิ่งจำนวน object มากเท่าไหร่
> ความต่างของ performance ยิ่งชัดเจนขึ้นเท่านั้น

**คำถามที่ 10**: เมื่อไหร่ควรเลือก Manual QuerySet Filtering (ไม่ใช้ library) แทน
`django-guardian`?
> **เฉลย**: เมื่อกฎการเข้าถึงเป็นแบบ "เจ้าของเท่านั้น" (ownership) ล้วน ๆ ไม่มีความ
> ต้องการมอบสิทธิ์ให้คนอื่นที่ไม่ใช่เจ้าของ (delegation) และไม่ต้องการให้สิทธิ์นั้น
> โผล่ในระบบ permission มาตรฐานของ Django (เช่น Admin panel, `{% if perms %}`) —
> การเพิ่ม dependency และตารางฐานข้อมูลใหม่โดยไม่จำเป็นคือ over-engineering

**คำถามที่ 11**: ทำไมการคืน `Http404` แทน `PermissionDenied` (403) บางครั้งถึงปลอดภัย
กว่าในเชิง Information Disclosure?
> **เฉลย**: การคืน 403 ยืนยันว่า "object นี้มีอยู่จริงในระบบ แต่คุณเข้าไม่ได้" ซึ่ง
> เปิดเผยข้อมูลบางอย่างให้ผู้โจมตี (เช่น ยืนยันว่า user id หรือ slug ที่เดามาถูกต้อง)
> ส่วน 404 บอกเพียงว่า "ไม่พบข้อมูลนี้" ซึ่งไม่เปิดเผยว่ามันมีอยู่จริงหรือไม่ — สำหรับ
> ข้อมูลที่อ่อนไหว (เช่น draft ส่วนตัว, ข้อมูลบัญชีผู้อื่น) การซ่อนการมีอยู่ด้วย 404
> จึงปลอดภัยกว่า

**คำถามที่ 12**: `ObjectPermissionChecker.prefetch_perms()` แก้ปัญหาอะไร และทำงาน
อย่างไรเบื้องหลังโดยสรุป?
> **เฉลย**: แก้ปัญหา N+1 Query เมื่อต้องเช็ค object-level permission ของหลาย object
> พร้อมกัน (เช่นในหน้า List) โดย query สิทธิ์ของทุก object ในลิสต์ **พร้อมกันในครั้ง
> เดียว** แล้ว cache ผลลัพธ์ไว้ใน instance ของ `ObjectPermissionChecker` เอง การเรียก
> `.has_perm()` ครั้งต่อ ๆ ไปกับ object ที่ prefetch ไว้แล้วจะอ่านจาก cache ใน memory
> โดยไม่ query ฐานข้อมูลซ้ำอีกเลย

### 380.4 แบบฝึกหัดใหญ่ปิดท้าย Phase: ระบบ Authentication + Authorization ที่สมบูรณ์

นี่คือแบบฝึกหัดสรุปที่รวมทุกเรื่องจาก Part 031-038 เข้าด้วยกันเป็นระบบเดียว ให้สร้าง
แอป Django ชื่อ `secureblog` ที่มีคุณสมบัติครบตามข้อกำหนดต่อไปนี้ (ใช้โครงสร้าง `Post`/
`Category` เดิมจาก Part 023-024/033 เป็นฐาน):

**ส่วนที่ 1 — Authentication & User (อ้างอิง Part 031-032, 035-037)**

- [ ] สร้าง Custom User Model (`AbstractUser`) พร้อม field เพิ่มเติมอย่างน้อย 1 ตัว
      (เช่น `bio`, `avatar`)
- [ ] ระบบ Signup ที่ใช้ Django Form มาตรฐาน พร้อม Password Validator ครบ 4 ตัวจาก
      `AUTH_PASSWORD_VALIDATORS` (`MinimumLengthValidator`,
      `CommonPasswordValidator`, `NumericPasswordValidator`,
      `UserAttributeSimilarityValidator`)
- [ ] ระบบ Login/Logout มาตรฐานผ่าน `LoginView`/`LogoutView`
- [ ] ระบบ Password Reset ผ่านอีเมลแบบเต็มรูปแบบ (`PasswordResetView` →
      `PasswordResetConfirmView`)
- [ ] เปิดใช้ Two-Factor Authentication (TOTP) เป็น **ตัวเลือก** ที่ user เปิดเองได้
      จากหน้า Profile (ไม่บังคับทุกคน) พร้อม backup codes สำรอง
- [ ] (โบนัส) เชื่อมต่อ Social Login อย่างน้อย 1 ผู้ให้บริการผ่าน `django-allauth`

**ส่วนที่ 2 — Authorization ตาม Role (อ้างอิง Part 033)**

- [ ] สร้าง 3 Group: `Authors`, `Editors`, `Reviewers` พร้อม permission ตามที่ออกแบบใน
      ขั้นตอนที่ 377.2 (ใช้ Data Migration ไม่ใช่ Management Command เพื่อความปลอดภัย
      ตาม Part 033 ขั้นตอนที่ 323.6)
- [ ] สร้าง Custom Permission `can_publish_post` ที่มีเฉพาะ Editor เท่านั้น

**ส่วนที่ 3 — Object-level Permission (อ้างอิง Part 038 นี้)**

- [ ] ติดตั้งและตั้งค่า `django-guardian` ให้ถูกต้องครบทุกจุด
- [ ] มอบ object-level permission ให้ author อัตโนมัติทันทีที่สร้างโพสต์ (เลือกใช้
      `form_valid()` หรือ signal ก็ได้ ต้องอธิบายเหตุผลที่เลือก)
- [ ] เขียน `AuthorRequiredMixin` เวอร์ชันที่ใช้ guardian (ตาม ขั้นตอนที่ 375.2) แล้ว
      นำไปใช้กับ `PostUpdateView`/`PostDeleteView`
- [ ] สร้างหน้า "มอบหมายผู้ตรวจทาน" (`AssignReviewerView`) ที่ Editor เท่านั้นที่เข้าถึง
      ได้ ใช้เลือก User คนหนึ่งมาเป็น Reviewer ของโพสต์หนึ่งตัวโดยเฉพาะ (เรียก
      `assign_perm()` เบื้องหลัง)
- [ ] สร้าง `PostQuerySet.visible_to(user)` ที่ผสาน Group-based + Object-level ตาม
      ขั้นตอนที่ 377.3 และใช้ในหน้า Dashboard

**ส่วนที่ 4 — Performance & Quality (อ้างอิง ขั้นตอนที่ 378-379)**

- [ ] หน้า Dashboard ต้องใช้ `ObjectPermissionChecker.prefetch_perms()` ไม่ให้เกิด N+1
      (พิสูจน์ด้วย `assertNumQueries` ใน test)
- [ ] เขียน Test ครอบคลุมทุก role (Author/Editor/Reviewer/Stranger/Anonymous) อย่างน้อย
      role ละ 2 test case ตามหัวข้อในตาราง ขั้นตอนที่ 379.5
- [ ] เขียน Test ยืนยันว่า Password Reset flow ทำงานถูกต้องแบบ end-to-end (ใช้
      `django.core.mail.outbox` ในการตรวจสอบอีเมลที่ถูกส่ง)

**เกณฑ์ความสำเร็จ**: ระบบต้องผ่านสถานการณ์ทดสอบนี้ได้ครบ — narin (Author) สร้างโพสต์
1 ตัว ได้สิทธิ์แก้ไข/ลบอัตโนมัติ; sofia (Editor) เห็นและแก้ไขโพสต์ของ narin ได้แม้ไม่ใช่
เจ้าของ (ผ่านสิทธิ์ระดับ model); kai (Reviewer) แก้ไขไม่ได้จนกว่า sofia จะมอบหมายให้ผ่าน
`AssignReviewerView` เท่านั้น; และผู้ใช้อื่นที่ไม่เกี่ยวข้องทั้งหมดถูกปฏิเสธด้วย 403
ทุกกรณี พร้อม log แสดงว่าจำนวน query ของหน้า Dashboard ไม่เพิ่มขึ้นตามจำนวนโพสต์

### 380.5 Checklist ก่อนไป Phase 5

- [ ] อธิบายความแตกต่างระหว่าง Authentication และ Authorization ได้โดยไม่ต้องเปิดตำรา
- [ ] สร้าง Custom User Model ตั้งแต่เริ่มโปรเจกต์ใหม่ได้ทุกครั้งโดยอัตโนมัติ (เป็นนิสัย)
- [ ] อธิบายกลไก Session และความสัมพันธ์กับ cookie `sessionid` ได้
- [ ] ตั้งค่า Password Validator และเข้าใจว่า Password Hasher ทำงานอย่างไร
- [ ] เชื่อมต่อ OAuth ผ่าน django-allauth ได้อย่างน้อย 1 ผู้ให้บริการ
- [ ] ตั้งค่า Two-Factor Authentication แบบ TOTP ได้
- [ ] ติดตั้งและใช้ django-guardian มอบ/เช็ค object-level permission ได้คล่องแคล่ว
- [ ] อธิบายได้ว่าเมื่อไหร่ควรใช้ Manual QuerySet Filtering แทน library
- [ ] ผสาน Group-based permission กับ Object-level permission เข้าด้วยกันได้
- [ ] แก้ปัญหา N+1 Query ของการเช็ค permission ด้วย `ObjectPermissionChecker` ได้
- [ ] เขียน Test ครอบคลุม permission logic ทุก role ในระบบได้

### 380.6 เตรียมตัวสำหรับ Phase 5: Django REST Framework และ API

ตลอด Phase 1-4 (Part 001-038) คุณสร้างเว็บแอปพลิเคชันแบบ **Server-Rendered** เต็มรูปแบบ
— Django สร้าง HTML ทั้งหน้าส่งกลับไปให้เบราว์เซอร์โดยตรง ทุก view คืนค่า
`HttpResponse`/`render()` ที่มี HTML ฝังอยู่ ระบบ Authentication/Authorization ทั้งหมด
ที่เรียนมาใน Phase 4 ก็ผูกกับแนวคิดนี้ (session, cookie, redirect ไป login page)

แต่โลกของการพัฒนาเว็บสมัยใหม่ต้องการมากกว่านั้น: แอป Mobile ที่ต้องคุยกับ backend เดียวกัน,
Frontend แบบ Single-Page Application (React, Vue) ที่แยกจาก backend อย่างสิ้นเชิง, หรือ
ระบบ Microservices ที่ต้องให้บริการหลายทีมพร้อมกัน — ทั้งหมดนี้ต้องการ **API** ที่คุยกัน
ด้วย JSON แทน HTML

**Phase 5: Django REST Framework และ API (Part 039-048)** จะพาคุณสร้างสิ่งเหล่านี้:

- **Part 039**: ติดตั้งและทำความเข้าใจสถาปัตยกรรมของ Django REST Framework (DRF)
- **Serializer**: แปลง Model instance เป็น JSON (และย้อนกลับ) แทนที่ Template ที่ใช้มา
  ตลอด Phase 1-4
- **ViewSet และ Router**: ทางเลือกใหม่แทน Generic CBV ที่เรียนใน Part 021-024 สำหรับ
  สร้าง API endpoint แบบ RESTful
- **Authentication สำหรับ API**: Token Authentication, JWT (JSON Web Token), และ
  Session Authentication แบบที่ใช้กับ API — ต่อยอดโดยตรงจากทุกอย่างที่เรียนใน Phase 4
  แต่ปรับให้เหมาะกับ client ที่ไม่ใช่เบราว์เซอร์ (mobile app, frontend แยก)
- **Permission Classes ของ DRF**: `IsAuthenticated`, `IsAdminUser`,
  `DjangoObjectPermissions` — ใช่แล้ว DRF มี permission class ที่ผูกกับ
  **`django-guardian` โดยตรง** ทำให้ Object-level Permission ที่คุณเพิ่งเรียนจบใน
  Part นี้กลายเป็นรากฐานสำคัญของการควบคุมสิทธิ์ใน API ด้วยเช่นกัน
- **API Documentation**: สร้างเอกสาร API อัตโนมัติด้วย `drf-spectacular` (OpenAPI/Swagger)

ทุกแนวคิดเรื่อง Permission, Group, Object-level Permission ที่คุณสร้างความเข้าใจมาตลอด
Phase 4 จะไม่ถูกทิ้งไปเลย — มันจะกลายเป็นฐานรากที่ทำให้คุณเข้าใจ **`DjangoObjectPermissions`**
ของ DRF ได้ทันทีโดยแทบไม่ต้องเรียนอะไรใหม่ เพียงแค่เปลี่ยนจาก "คืน HTML" เป็น "คืน JSON"
เท่านั้น

เตรียมเปิดโปรเจกต์ `secureblog` ที่สร้างจากแบบฝึกหัดใหญ่ในขั้นตอนที่ 380.4 ไว้ให้พร้อม
เพราะ Phase 5 จะกลับมาเปิด API ให้กับระบบ blog เดียวกันนี้ทั้งหมด แล้วไปพบกันที่
**Part 039: ทำความรู้จัก Django REST Framework**!
