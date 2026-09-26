# Part 015: Model Meta Options, Managers และ Custom QuerySets

> **ขั้นตอนที่ 141-150 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เจาะลึกความสามารถของ `class Meta` ในโมเดล Django ครบทุก option
> สำคัญ, เข้าใจ `UniqueConstraint`/`CheckConstraint` แบบสมัยใหม่, เรียนรู้การออกแบบ
> Abstract Base Class, Multi-table Inheritance และ Proxy Model อย่างถูกต้อง, สร้าง
> Custom Manager และ Custom QuerySet ที่ chain method ต่อกันได้แบบมืออาชีพ และเข้าใจ
> ข้อควรระวังของการ override `save()` เมื่อจบ Part นี้ คุณจะสามารถ refactor โมเดล Django
> ให้สะอาด ไม่ซ้ำซ้อน (DRY) และเขียน query ที่อ่านง่ายระดับ production ได้จริง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 141: `class Meta` เจาะลึกครบทุก option สำคัญ
- ขั้นตอนที่ 142: `UniqueConstraint` และ `CheckConstraint` แทน `unique_together` แบบเก่า
- ขั้นตอนที่ 143: Abstract Base Classes — สร้าง `TimeStampedModel` ให้โมเดลอื่น inherit
- ขั้นตอนที่ 144: Multi-table Inheritance vs Abstract Base Class vs Proxy Model
- ขั้นตอนที่ 145: Proxy Models เจาะลึก — ตัวอย่างจริงด้วย `PublishedPost`
- ขั้นตอนที่ 146: Custom Manager เบื้องต้น — สร้าง `PublishedManager`
- ขั้นตอนที่ 147: Custom QuerySet + `as_manager()` — chain method ได้แบบมืออาชีพ
- ขั้นตอนที่ 148: Manager หลายตัวในโมเดลเดียว และผลต่อ default manager
- ขั้นตอนที่ 149: Overriding `save()` และข้อควรระวังเรื่อง business logic
- ขั้นตอนที่ 150: สรุปและแบบฝึกหัด — refactor โมเดล blog ทั้งหมด

---

## ขั้นตอนที่ 141: `class Meta` เจาะลึกครบทุก option สำคัญ

### 141.1 `class Meta` คืออะไร และทำไมสำคัญ

ทุกโมเดลใน Django สามารถมี inner class ชื่อ `Meta` ซึ่งไม่ใช่ field ของตาราง แต่เป็น
**metadata** ที่บอก Django ว่าจะสร้างตารางและปฏิบัติกับโมเดลนี้อย่างไร เช่น เรียงลำดับ
ผลลัพธ์ยังไง, ชื่อตารางในฐานข้อมูลคืออะไร, มี constraint หรือ index อะไรบ้าง

จนถึงตอนนี้ (Part 011-014) คุณอาจเคยเห็น `class Meta: ordering = [...]` มาบ้างแล้ว
ใน Part นี้เราจะเจาะลึกทุก option ที่ใช้งานจริงบ่อยที่สุดในโปรเจกต์ระดับมืออาชีพ

สมมติแอป `blog` ของเรามีโมเดลดังนี้ (จะใช้ตัวอย่างนี้ตลอดทั้ง Part):

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=100, unique=True)

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.PositiveIntegerField(default=0)

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    parent = models.ForeignKey(
        "self", on_delete=models.CASCADE, null=True, blank=True, related_name="replies"
    )

    def __str__(self):
        return f"Comment by {self.author} on {self.post}"


class Profile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)

    def __str__(self):
        return f"Profile of {self.user}"
```

โค้ดข้างต้นยังไม่มี `Meta` เลย ต่อไปเราจะเพิ่มทีละ option พร้อมอธิบายว่าแต่ละตัวทำอะไร

### 141.2 `ordering` — กำหนดลำดับผลลัพธ์เริ่มต้น

ถ้าไม่ตั้ง `ordering` ลำดับของ `Post.objects.all()` จะ **ไม่แน่นอน** (ขึ้นอยู่กับฐานข้อมูล
และแผนการ query ภายใน) ซึ่งอันตรายมากในโปรเจกต์จริง (เช่น pagination เพี้ยน)

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["-created_at"]  # เรียงโพสต์ล่าสุดขึ้นก่อน
```

- `"-created_at"` หมายถึงเรียงจากมากไปน้อย (DESC) ส่วน `"created_at"` (ไม่มีเครื่องหมาย
  ลบ) หมายถึงน้อยไปมาก (ASC)
- ตั้งได้หลาย field พร้อมกัน เช่น `ordering = ["category", "-created_at"]`
  (เรียงตาม category ก่อน แล้วค่อยเรียงวันที่ล่าสุดในแต่ละ category)
- `ordering` ที่ตั้งใน `Meta` เป็นค่า **default** เท่านั้น เมธอด `.order_by()` ที่เรียกทีหลัง
  จะ override ค่านี้เสมอ เช่น `Post.objects.order_by("title")` จะไม่สนใจ `Meta.ordering`
- **ข้อควรระวังด้าน performance**: `ordering` ที่ตั้งไว้จะถูกใช้ทุกครั้งที่ query โมเดลนี้
  (ยกเว้นถูก override) ถ้า field ที่ใช้ ordering ไม่มี index จะทำให้ query ช้าลงเมื่อข้อมูล
  เยอะขึ้น (เราจะเรียนเรื่อง indexes ในหัวข้อ 141.6 และเจาะลึกใน Part 070)

### 141.3 `verbose_name` และ `verbose_name_plural` — ชื่อที่อ่านง่ายสำหรับมนุษย์

ค่าเริ่มต้น Django จะสร้างชื่อโมเดลแบบ human-readable ให้อัตโนมัติจากชื่อ class
(เช่น `Post` → "post", พหูพจน์ "posts") แต่ถ้าชื่อโมเดลเป็นตัวย่อหรือคำที่แปลงพหูพจน์
อัตโนมัติไม่ถูกต้อง ควรกำหนดเอง:

```python
class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name = "หมวดหมู่"
        verbose_name_plural = "หมวดหมู่ทั้งหมด"

    def __str__(self):
        return self.name
```

ค่าเหล่านี้จะไปปรากฏใน **Django Admin** (หัวข้อเมนู, breadcrumb, ข้อความยืนยันการลบ)
และในข้อความ error ของ form validation เป็นหลัก ตัวอย่างที่ verbose_name plural อัตโนมัติ
ผิดพลาดบ่อย: โมเดลชื่อ `Category` (พหูพจน์ที่ถูกคือ "Categories" ไม่ใช่ "Categorys" —
Django รุ่นใหม่จัดการเคสนี้ได้ดีขึ้นแล้ว แต่คำภาษาไทยหรือคำทับศัพท์ควรกำหนดเองเสมอ)

### 141.4 `db_table` — กำหนดชื่อตารางในฐานข้อมูลเอง

ค่าเริ่มต้น Django จะตั้งชื่อตารางเป็น `<app_label>_<model_name แบบตัวพิมพ์เล็ก>` เช่น
`blog_post`, `blog_category` ถ้าต้องการควบคุมชื่อตารางเอง (เช่น เชื่อมกับฐานข้อมูล legacy
ที่มีตารางอยู่แล้ว หรือทีมมี naming convention ขององค์กร) ใช้ `db_table`:

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["-created_at"]
        db_table = "cms_articles"  # แทนที่จะเป็น blog_post อัตโนมัติ
```

**คำแนะนำระดับมืออาชีพ**: อย่าตั้ง `db_table` เอง เว้นแต่มีเหตุผลจำเป็นจริง ๆ
(เช่น legacy database, ข้อกำหนดขององค์กร) เพราะการปล่อยให้ Django ตั้งชื่ออัตโนมัติ
ช่วยลดโอกาสชื่อชนกันและอ่านโค้ดเข้าใจง่ายกว่าสำหรับคนอื่นในทีม

### 141.5 `unique_together` — แบบเก่า (เกริ่นนำ)

Django รุ่นเก่ามี option `unique_together` สำหรับบังคับว่าชุด field หลายตัวรวมกันต้อง
ไม่ซ้ำกัน:

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        unique_together = [["category", "slug"]]  # slug ซ้ำได้ถ้าอยู่คนละ category
```

Option นี้ยังใช้งานได้ใน Django 5.x แต่ **ถือว่า legacy แล้ว** — Django แนะนำให้ใช้
`UniqueConstraint` แทนเสมอสำหรับโค้ดใหม่ เราจะอธิบายเหตุผลและวิธี migrate แบบละเอียด
ในขั้นตอนที่ 142

### 141.6 `indexes` — เพิ่ม database index แบบกำหนดเอง

Field ที่ตั้ง `db_index=True` หรือ `unique=True` จะได้ index อัตโนมัติ แต่ถ้าต้องการ
index ที่ครอบคลุมหลาย field พร้อมกัน (composite index) หรือ index แบบพิเศษ ให้ใช้
`Meta.indexes`:

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["is_published", "-created_at"], name="post_published_idx"),
            models.Index(fields=["category", "slug"], name="post_category_slug_idx"),
        ]
```

- `models.Index(fields=["is_published", "-created_at"], ...)` สร้าง composite index
  ที่ช่วยเร่ง query แบบ `Post.objects.filter(is_published=True).order_by("-created_at")`
  ให้เร็วขึ้นมากเมื่อข้อมูลมีหลักหมื่น-แสนแถว
- `name` ควรตั้งเองเสมอ (ไม่เกิน 30 ตัวอักษรในบางฐานข้อมูล) เพื่อให้ migration อ่านง่าย
  และแก้ไข/ลบ index ในอนาคตได้ตรงเป้า
- เราจะเจาะลึกเรื่อง index กับ query performance แบบเต็มใน Part 070
  (Database Indexing และ Query Analysis) ตอนนี้ขอให้เข้าใจแค่ว่า `Meta.indexes`
  คือจุดที่ประกาศ index เพิ่มเติมนอกเหนือจาก field-level

### 141.7 `abstract` — เกริ่นนำสู่ Abstract Base Class

```python
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

`abstract = True` บอก Django ว่าโมเดลนี้ **ไม่ต้องสร้างตารางในฐานข้อมูล** — มันเป็นแค่
"แม่แบบ" ให้โมเดลอื่น inherit field และพฤติกรรมไปใช้ เราจะเจาะลึกเรื่องนี้เต็ม ๆ
ในขั้นตอนที่ 143 เพราะเป็นเทคนิคที่ช่วยลดโค้ดซ้ำซ้อนได้มากในโปรเจกต์จริง

### 141.8 ตารางสรุป Meta Options ที่สำคัญที่สุด

| Option | ทำหน้าที่ | ตัวอย่างค่า | ใช้บ่อยแค่ไหน |
|---|---|---|---|
| `ordering` | ลำดับผลลัพธ์เริ่มต้น | `["-created_at"]` | สูงมาก (ควรตั้งแทบทุกโมเดล) |
| `verbose_name` | ชื่ออ่านง่าย (เอกพจน์) | `"หมวดหมู่"` | ปานกลาง |
| `verbose_name_plural` | ชื่ออ่านง่าย (พหูพจน์) | `"หมวดหมู่ทั้งหมด"` | ปานกลาง |
| `db_table` | ชื่อตารางจริงในฐานข้อมูล | `"cms_articles"` | ต่ำ (เฉพาะกรณีจำเป็น) |
| `unique_together` | unique หลาย field รวมกัน (legacy) | `[["category", "slug"]]` | ต่ำลงเรื่อย ๆ (ใช้ `UniqueConstraint` แทน) |
| `constraints` | รวม `UniqueConstraint`/`CheckConstraint` | ดู ขั้นตอนที่ 142 | สูง (มาตรฐานใหม่) |
| `indexes` | เพิ่ม database index แบบกำหนดเอง | `[models.Index(...)]` | สูง (สำหรับ query ที่ใช้บ่อย) |
| `abstract` | ทำให้เป็นแม่แบบ ไม่สร้างตาราง | `True` | สูง (สำหรับ base class) |
| `app_label` | ระบุแอปเจ้าของโมเดล (เมื่อจำเป็น) | `"blog"` | ต่ำมาก (กรณี model อยู่นอก app pattern ปกติ) |
| `permissions` | สิทธิ์แบบกำหนดเองเพิ่มจาก CRUD | `[("can_publish", "Can publish post")]` | ปานกลาง (เจาะลึกใน Part 033) |
| `get_latest_by` | field ที่ใช้กับ `.latest()`/`.earliest()` | `"created_at"` | ต่ำ-ปานกลาง |
| `default_related_name` | ชื่อ reverse relation เริ่มต้นของทุก FK | `"posts"` | ปานกลาง |
| `default_manager_name` / `base_manager_name` | ควบคุม manager เริ่มต้น | `"objects"` | สูง (ดู ขั้นตอนที่ 148) |

Meta option อื่น ๆ ที่มีอยู่แต่ใช้น้อยกว่านี้ (เช่น `managed`, `proxy`, `select_on_save`)
จะถูกอธิบายในบริบทที่เกี่ยวข้องโดยตรง เช่น `proxy` ในขั้นตอนที่ 145

---

## ขั้นตอนที่ 142: `UniqueConstraint` และ `CheckConstraint` แทน `unique_together` แบบเก่า

### 142.1 ทำไม `unique_together` ถึงถูกมองว่าเป็นของเก่า

`unique_together` มีข้อจำกัดสำคัญหลายอย่างที่ทำให้ Django เพิ่ม `UniqueConstraint`
เข้ามาแทนที่ (ตั้งแต่ Django 2.2) และแนะนำให้ใช้ `constraints` ในโค้ดใหม่ทั้งหมด:

| ประเด็น | `unique_together` (legacy) | `UniqueConstraint` (แนะนำ) |
|---|---|---|
| ตั้ง condition (partial unique) | ❌ ทำไม่ได้ | ✅ ทำได้ผ่าน `condition=Q(...)` |
| ใช้ expression/function ในการเช็ค unique | ❌ ทำไม่ได้ | ✅ ทำได้ (Django 4.0+ รองรับ `Func` expressions) |
| ตั้งชื่อ constraint เอง | ❌ Django ตั้งชื่อ auto (อ่านยากใน error) | ✅ ตั้งชื่อเองผ่าน `name=` |
| ข้อความ error ตอน validate form | กำกวมกว่า | ชัดเจนกว่า ผ่าน `violation_error_message` |
| ปรากฏใน `Meta` ร่วมกับ constraint อื่น | แยก attribute ต่างหาก | รวมอยู่ใน list เดียวกับ `CheckConstraint` |
| อนาคตของ Django | เก็บไว้เพื่อ backward compatibility | ทิศทางหลักที่ Django พัฒนาต่อ |

สรุปสั้น ๆ: **โปรเจกต์ใหม่ควรใช้ `constraints = [...]` เสมอ** ส่วน `unique_together`
ให้รู้จักไว้เผื่อเจอในโค้ด legacy เท่านั้น

### 142.2 `UniqueConstraint` พื้นฐาน — แทนที่ `unique_together`

ย้ายจาก:

```python
class Post(models.Model):
    class Meta:
        unique_together = [["category", "slug"]]
```

มาเป็น:

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["-created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"],
                name="unique_slug_per_category",
                violation_error_message="Slug นี้ถูกใช้ไปแล้วในหมวดหมู่นี้",
            ),
        ]
```

ผลลัพธ์เชิงพฤติกรรมเหมือนเดิมทุกประการ (slug ซ้ำกันได้ถ้าอยู่คนละ category) แต่ตอนนี้
เรามีชื่อ constraint ชัดเจน (`unique_slug_per_category`) และข้อความ error ที่กำหนดเอง

### 142.3 Conditional `UniqueConstraint` — สิ่งที่ `unique_together` ทำไม่ได้เลย

สมมติกฎธุรกิจ: "ป้องกันไม่ให้ผู้ใช้คนเดียวคอมเมนต์ข้อความซ้ำเป๊ะในโพสต์เดียวกัน
แต่กฎนี้ใช้เฉพาะคอมเมนต์ระดับบนสุด (ไม่ใช่การตอบกลับ) เพื่อกันสแปม" นี่คือกรณีที่ต้องใช้
`condition`:

```python
class Comment(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["post", "author", "text"],
                condition=models.Q(parent__isnull=True),
                name="unique_top_level_comment_text",
                violation_error_message="คุณคอมเมนต์ข้อความนี้ไปแล้วในโพสต์นี้",
            ),
        ]
```

- `condition=models.Q(parent__isnull=True)` ทำให้ constraint นี้บังคับใช้ **เฉพาะแถวที่
  `parent` เป็น `NULL`** (คอมเมนต์ระดับบนสุด) — การตอบกลับ (`parent` ไม่ใช่ NULL)
  จะไม่ถูกเช็คด้วยกฎนี้
- นี่เรียกว่า **Partial Unique Index** ในระดับฐานข้อมูล ซึ่ง `unique_together`
  ไม่มีทางทำได้เลยเพราะไม่รองรับ `condition`
- Conditional constraint รองรับใน PostgreSQL และ SQLite เต็มรูปแบบ ส่วน MySQL
  รองรับตั้งแต่เวอร์ชัน 8.0.16 ขึ้นไป (ควรตรวจสอบฐานข้อมูล production ก่อนใช้จริง)

### 142.4 `CheckConstraint` — บังคับกฎระดับแถวในฐานข้อมูล

`CheckConstraint` ใช้บังคับว่า **ค่าภายในแถวเดียวกัน** ต้องเป็นไปตามเงื่อนไขที่กำหนด
โดยฐานข้อมูลจะปฏิเสธการ insert/update ที่ผิดกฎทันที (ไม่ใช่แค่ validate ฝั่ง Python)

```python
class Post(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["-created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"],
                name="unique_slug_per_category",
            ),
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name="post_view_count_non_negative",
                violation_error_message="view_count ต้องไม่ติดลบ",
            ),
        ]
```

ตัวอย่างที่ซับซ้อนกว่า — ป้องกันไม่ให้คอมเมนต์อ้างอิงตัวเองเป็น parent (ซึ่งจะทำให้เกิด
ลูปไม่รู้จบตอนแสดงผล):

```python
class Comment(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["post", "author", "text"],
                condition=models.Q(parent__isnull=True),
                name="unique_top_level_comment_text",
            ),
            models.CheckConstraint(
                check=~models.Q(parent=models.F("id")),
                name="comment_parent_not_self",
                violation_error_message="คอมเมนต์ไม่สามารถเป็น parent ของตัวเองได้",
            ),
        ]
```

- `~models.Q(parent=models.F("id"))` แปลเป็นภาษาคนว่า "ไม่ใช่กรณีที่ `parent`
  เท่ากับ `id` ของตัวเอง" — ใช้ `F()` เพื่อเปรียบเทียบสอง field ในแถวเดียวกัน
  (ทบทวน `F()` expression ได้จาก Part 014)
- **ข้อควรระวังสำคัญ**: `CheckConstraint` แบบนี้เช็คได้ที่ระดับฐานข้อมูลเมื่อ insert/update
  ตรง ๆ ผ่าน SQL แต่ **ไม่ได้ครอบคลุมทุกกรณีเสมอไป** — เช่น SQLite รุ่นเก่าบางเวอร์ชัน
  รองรับ `CheckConstraint` ไม่สมบูรณ์ ควรทดสอบบนฐานข้อมูล production จริงเสมอ (PostgreSQL
  รองรับเต็มรูปแบบที่สุด)

### 142.5 Migration ที่ Django สร้างให้อัตโนมัติ

เมื่อรัน `python manage.py makemigrations` หลังเพิ่ม constraints ข้างต้น Django จะสร้าง
migration ประมาณนี้:

```python
# blog/migrations/0005_add_constraints.py
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0004_previous_migration"),
    ]

    operations = [
        migrations.AddConstraint(
            model_name="post",
            constraint=models.UniqueConstraint(
                fields=("category", "slug"), name="unique_slug_per_category"
            ),
        ),
        migrations.AddConstraint(
            model_name="post",
            constraint=models.CheckConstraint(
                check=models.Q(("view_count__gte", 0)),
                name="post_view_count_non_negative",
            ),
        ),
        migrations.AddConstraint(
            model_name="comment",
            constraint=models.UniqueConstraint(
                condition=models.Q(("parent__isnull", True)),
                fields=("post", "author", "text"),
                name="unique_top_level_comment_text",
            ),
        ),
        migrations.AddConstraint(
            model_name="comment",
            constraint=models.CheckConstraint(
                check=models.Q(("parent", models.F("id")), _negated=True),
                name="comment_parent_not_self",
            ),
        ),
    ]
```

สังเกตว่า Django ใช้ operation `AddConstraint` แยกต่างหาก ไม่ปนกับ `AlterField`
ทำให้เห็นได้ชัดเจนในประวัติ migration ว่ามีการเพิ่ม/แก้กฎระดับฐานข้อมูลเมื่อไหร่

### 142.6 ทดสอบ constraint ใน Django shell

```bash
python manage.py shell
```

```python
>>> from blog.models import Post, Category
>>> from django.db import IntegrityError

>>> cat = Category.objects.create(name="Django", slug="django")
>>> Post.objects.create(title="Post 1", slug="hello-world", category=cat, content="...")
>>> Post.objects.create(title="Post 2", slug="hello-world", category=cat, content="...")
Traceback (most recent call last):
    ...
django.db.utils.IntegrityError: UNIQUE constraint failed: blog_post.category_id, blog_post.slug

>>> post = Post.objects.first()
>>> post.view_count = -5
>>> post.save()
Traceback (most recent call last):
    ...
django.db.utils.IntegrityError: CHECK constraint failed: post_view_count_non_negative
```

สังเกตว่า `IntegrityError` เกิดขึ้น **ที่ระดับฐานข้อมูล** ไม่ใช่ตอนเรียก `.save()`
เฉย ๆ — ถ้าต้องการ error message ที่สวยงามสำหรับผู้ใช้ปลายทาง (เช่นในฟอร์ม) ต้องเรียก
`full_clean()` ก่อน `.save()` เสมอ (เดี๋ยวเราจะพูดถึงเรื่องนี้อีกครั้งในขั้นตอนที่ 149)

---

## ขั้นตอนที่ 143: Abstract Base Classes — สร้าง `TimeStampedModel`

### 143.1 ปัญหาโค้ดซ้ำซ้อนที่ Abstract Base Class มาแก้

สังเกตว่าทั้ง `Post` และ `Comment` มี field `created_at` เหมือนกัน และถ้าเราต้องการเพิ่ม
`updated_at` ให้ทั้งคู่ (field ที่เก็บเวลาที่แก้ไขล่าสุด) เราจะต้องเขียนซ้ำสองที่:

```python
# ❌ ซ้ำซ้อน ผิดหลัก DRY
class Post(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    # ...

class Comment(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    # ...
```

ถ้าในอนาคตอยากเปลี่ยน behavior (เช่น เพิ่ม `db_index=True` ให้ `created_at` ของทุกโมเดล)
ต้องไปแก้ทุกที่ที่ copy วางไว้ — นี่คือสัญญาณชัดเจนว่าควรใช้ **Abstract Base Class**

### 143.2 สร้าง `TimeStampedModel`

```python
# blog/models.py (หรือแยกไปไว้ใน core/models.py ถ้ามีหลายแอปใช้ร่วมกัน)
from django.db import models


class TimeStampedModel(models.Model):
    """Abstract base ที่เพิ่ม created_at / updated_at ให้โมเดลลูกทุกตัว"""

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]  # ค่า default ให้โมเดลลูก (override ได้)
```

จุดสำคัญ:

- `class Meta: abstract = True` **บังคับต้องมี** ไม่เช่นนั้น Django จะพยายามสร้างตาราง
  `TimeStampedModel` จริง ๆ ในฐานข้อมูล ซึ่งไม่ใช่สิ่งที่เราต้องการ
- `TimeStampedModel` เอง **ไม่ปรากฏใน migration เป็นตาราง** — มันมีไว้เพื่อให้ field
  และ `Meta` ถูก "คัดลอก" ไปยังโมเดลลูกตอน Django ประมวลผล class เท่านั้น
- ตั้ง `ordering` ไว้ใน abstract base ได้ด้วย โมเดลลูกจะได้ค่านี้เป็น default
  แต่สามารถ override ได้ถ้าประกาศ `Meta` ของตัวเองพร้อมระบุ `ordering` ใหม่

### 143.3 ให้ `Post` และ `Comment` inherit จาก `TimeStampedModel`

```python
class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.PositiveIntegerField(default=0)
    # ไม่ต้องประกาศ created_at / updated_at อีกต่อไป — ได้มาจาก TimeStampedModel แล้ว

    class Meta(TimeStampedModel.Meta):
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"], name="unique_slug_per_category"
            ),
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name="post_view_count_non_negative",
            ),
        ]

    def __str__(self):
        return self.title


class Comment(TimeStampedModel):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    text = models.TextField()
    parent = models.ForeignKey(
        "self", on_delete=models.CASCADE, null=True, blank=True, related_name="replies"
    )
    # created_at ได้มาจาก TimeStampedModel — updated_at ก็เช่นกัน

    class Meta(TimeStampedModel.Meta):
        ordering = ["created_at"]  # override: คอมเมนต์ควรเรียงเก่า→ใหม่ ไม่ใช่ใหม่→เก่า
        constraints = [
            models.UniqueConstraint(
                fields=["post", "author", "text"],
                condition=models.Q(parent__isnull=True),
                name="unique_top_level_comment_text",
            ),
            models.CheckConstraint(
                check=~models.Q(parent=models.F("id")),
                name="comment_parent_not_self",
            ),
        ]

    def __str__(self):
        return f"Comment by {self.author} on {self.post}"
```

สังเกตเทคนิค `class Meta(TimeStampedModel.Meta):` — การ inherit `Meta` ของ base class
แบบนี้ทำให้โมเดลลูกได้ `ordering` และ option อื่น ๆ จาก parent มาโดยอัตโนมัติ ถ้าไม่เขียน
inherit แบบนี้ (เขียนแค่ `class Meta:` เฉย ๆ) โมเดลลูกจะ **ไม่ได้รับ** `Meta` ของ parent
เลย (ต้องเขียนใหม่ทั้งหมดเอง) — นี่เป็นจุดที่มือใหม่พลาดบ่อยมาก

### 143.4 Migration ที่เกิดขึ้นหลังเปลี่ยนมาใช้ Abstract Base

รัน `python manage.py makemigrations blog` แล้วดูผลลัพธ์:

```
Migrations for 'blog':
  blog/migrations/0006_alter_post_options_and_more.py
    ~ Change Meta options on comment
    ~ Change Meta options on post
    - Remove field created_at from comment
    + (created_at ถูกสร้างใหม่ผ่าน inheritance — โครงสร้างคอลัมน์เหมือนเดิมทุกประการ)
```

**สิ่งสำคัญที่ต้องเข้าใจ**: แม้โค้ด Python จะเปลี่ยนจาก "field ประกาศตรง ๆ" เป็น
"field มาจาก abstract base" แต่ **โครงสร้างตารางในฐานข้อมูลไม่เปลี่ยนเลย** เพราะ column
`created_at`/`updated_at` ยังคงอยู่ในตาราง `blog_post` และ `blog_comment` เหมือนเดิม
(abstract inheritance ไม่ได้สร้าง JOIN หรือตารางแยก) migration ที่เกิดขึ้นส่วนใหญ่จะเป็น
แค่การเปลี่ยนแปลง `Meta` options เท่านั้น หากมีอยู่แล้วไม่ต่างจากเดิมก็อาจไม่มี migration
เลยด้วยซ้ำ

### 143.5 Multiple Inheritance ของ Abstract Base หลายตัว

Abstract base สามารถผสมกันได้มากกว่าหนึ่งตัว (คล้าย mixin ใน Python OOP ทั่วไป):

```python
class SluggedModel(models.Model):
    slug = models.SlugField(max_length=220, unique=True)

    class Meta:
        abstract = True


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Post(TimeStampedModel, SluggedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    # ได้ทั้ง created_at, updated_at (จาก TimeStampedModel)
    # และ slug (จาก SluggedModel) โดยไม่ต้องประกาศเอง

    class Meta(TimeStampedModel.Meta):
        pass
```

ลำดับการ inherit (`TimeStampedModel, SluggedModel`) มีผลต่อ **field ordering** ในตาราง
(field จาก class แรกจะมาก่อน) และมีผลต่อ **Method Resolution Order (MRO)** ถ้ามีเมธอด
ชื่อซ้ำกันในหลาย abstract base — Python จะใช้กฎ MRO มาตรฐานเหมือน multiple inheritance
ทั่วไป (ทบทวนได้จาก Part 002)

### 143.6 ข้อจำกัดสำคัญของ Abstract Base Class

- **ForeignKey จาก abstract model ไปยัง abstract model อื่นทำไม่ได้** เพราะ field
  จะถูกคัดลอกไปยังโมเดลลูกแต่ละตัว การอ้างอิงจะสับสนว่าชี้ไปที่โมเดลลูกตัวไหน
- Abstract base **ไม่มี table ของตัวเอง** ดังนั้นจะ query `TimeStampedModel.objects.all()`
  ตรง ๆ ไม่ได้ (ไม่มี manager ผูกกับ table ที่ไม่มีอยู่จริง)
- ถ้า abstract base มี `related_name` ใน ForeignKey/ManyToMany และมีโมเดลลูกหลายตัว
  inherit ไปพร้อมกัน จะเกิด clash (`related_name` ซ้ำกัน) ต้องใช้ `%(class)s` เป็น
  placeholder เช่น `related_name="%(class)s_comments"` เพื่อให้ Django แทนที่ด้วยชื่อ
  โมเดลลูกอัตโนมัติ

---

## ขั้นตอนที่ 144: Multi-table Inheritance vs Abstract Base Class vs Proxy Model

### 144.1 Multi-table Inheritance (MTI) คืออะไร

Django รองรับการ inherit โมเดลอีกแบบที่เรียกว่า **Multi-table Inheritance** ซึ่งต่างจาก
Abstract Base Class ตรงที่ **โมเดล parent เป็น concrete model จริง (มีตารางของตัวเอง)**
และโมเดลลูกจะมีตารางแยกต่างหาก เชื่อมกับ parent ด้วย `OneToOneField` โดยอัตโนมัติ

```python
# ตัวอย่างเพื่ออธิบายแนวคิด MTI (นอกบริบท blog app เพื่อความชัดเจน)
class Content(models.Model):
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title


class Article(Content):  # ไม่มี Meta.abstract = True → นี่คือ MTI ไม่ใช่ abstract
    body = models.TextField()
    editor_notes = models.TextField(blank=True)
```

เมื่อรัน migration Django จะสร้าง **สองตาราง**: `content` (มี `id`, `title`,
`created_at`) และ `article` (มี `content_ptr_id` เป็น OneToOne ไปยัง `content`,
บวกกับ `body`, `editor_notes`) การ query `Article.objects.first().title` จะทำงานได้
เหมือน field นั้นอยู่ใน `Article` เลย แต่เบื้องหลัง Django ต้อง **JOIN สองตาราง**
เข้าด้วยกันเสมอ

### 144.2 ตารางเปรียบเทียบ Abstract Base / MTI / Proxy

| ประเด็น | Abstract Base Class | Multi-table Inheritance | Proxy Model |
|---|---|---|---|
| Parent มีตารางของตัวเองไหม | ❌ ไม่มี (ไม่ใช่ concrete model) | ✅ มี (เป็น concrete model จริง) | ✅ มี (ใช้ตารางเดียวกับ parent) |
| ลูกมีตารางแยกไหม | ✅ ลูกมีตารางของตัวเอง (field ถูกคัดลอกไปรวม) | ✅ ลูกมีตารางแยก เชื่อมด้วย OneToOne | ❌ ไม่มี — ใช้ตารางเดียวกับ parent เป๊ะ |
| ต้อง JOIN ตอน query ไหม | ❌ ไม่ต้อง (field รวมอยู่ในตารางเดียว) | ✅ ต้อง JOIN เสมอ (มี overhead) | ❌ ไม่ต้อง (ไม่มีตารางใหม่) |
| เพิ่ม field ใหม่ในลูกได้ไหม | ✅ ได้เต็มที่ | ✅ ได้เต็มที่ | ❌ ไม่ได้ (schema เดียวกับ parent) |
| เปลี่ยน behavior (Meta, manager, method) ได้ไหม | ✅ ได้ | ✅ ได้ | ✅ ได้ (นี่คือจุดประสงค์หลัก) |
| Query `parent.objects.all()` เห็นลูกด้วยไหม | ไม่เกี่ยวข้อง (parent ไม่มีตาราง) | ไม่เห็น field ของลูก (ต้อง query ผ่านลูก) | เห็น (ใช้ตารางเดียวกัน แค่ Python class ต่างกัน) |
| เหมาะกับ | ลด field ซ้ำซ้อน (created_at, updated_at) | โมเดลที่เป็น "ชนิดย่อย" จริง ๆ ของ parent ที่มี field ต่างกันเยอะ | เปลี่ยนแค่ default manager/ordering/method โดยไม่แตะ schema |
| ใช้บ่อยแค่ไหนในงานจริง | สูงมาก (มาตรฐาน) | ต่ำ (มักแทนด้วย composition/ForeignKey แทน) | ปานกลาง (สำหรับ use case เฉพาะทาง) |

### 144.3 ผลกระทบด้าน Performance

- **Abstract Base**: ไม่มีต้นทุนเพิ่มเลย เพราะสุดท้ายกลายเป็นตารางเดียวแบบแบน (flat)
  เหมือนเขียน field ตรง ๆ ทุกประการ
- **MTI**: ทุกครั้งที่ query โมเดลลูก Django ต้อง JOIN กับตาราง parent เสมอ
  (`SELECT ... FROM article INNER JOIN content ON article.content_ptr_id = content.id`)
  ถ้ามีลำดับชั้นลึกหลายระดับ (parent ของ parent) จะยิ่ง JOIN เยอะขึ้นและช้าลง
- **Proxy**: ไม่มีต้นทุนเพิ่มด้าน query เลย เพราะ SQL ที่ generate ออกมาเหมือนกับใช้
  โมเดล parent ตรง ๆ ทุกประการ ต่างกันแค่ฝั่ง Python object เท่านั้น

### 144.4 คำแนะนำเชิงปฏิบัติ

ในงานจริงระดับมืออาชีพ ทีมส่วนใหญ่**เลี่ยงการใช้ MTI** เพราะ overhead ของ JOIN
และความซับซ้อนที่เพิ่มขึ้น (เช่น การ query แบบ `select_related` ต้องระวังมากขึ้น)
ทางเลือกที่นิยมกว่าเมื่อต้องการ "โมเดลที่มีบาง field ร่วมกันแต่มีชนิดย่อยต่างกัน":

1. **Abstract Base Class** — ถ้า field ที่ใช้ร่วมกันเป็นแค่ metadata ทั่วไป
   (created_at, updated_at, slug) และแต่ละลูกไม่จำเป็นต้อง query ร่วมกันเป็นชุดเดียว
2. **Composition ผ่าน ForeignKey/OneToOne** — ถ้าต้องการความสัมพันธ์แบบ "has-a"
   มากกว่า "is-a" เช่น `Profile` ที่มี OneToOne กับ `User` (ตามที่เราออกแบบไว้แล้ว)
3. **Proxy Model** — ถ้าแค่ต้องการ behavior ของ Python class ที่ต่างกัน
   (default manager, ordering, method) โดยไม่ต้องการ schema ใหม่เลย (ดูขั้นตอนที่ 145)

MTI ยังมีที่ใช้จริงอยู่บ้าง เช่น Django's own `auth` app ใช้แนวคิดคล้าย ๆ กันในบาง
ส่วน แต่สำหรับแอป `blog` ของเราจะไม่ใช้ MTI เลย

---

## ขั้นตอนที่ 145: Proxy Models เจาะลึก — ตัวอย่างจริงด้วย `PublishedPost`

### 145.1 Proxy Model คืออะไร

Proxy Model คือการสร้างโมเดลใหม่ที่ **ใช้ตารางฐานข้อมูลเดียวกับโมเดลเดิมทุกประการ**
แต่มี Python class แยกต่างหาก ทำให้สามารถกำหนด:

- default manager ที่ต่างออกไป
- `Meta.ordering` ที่ต่างออกไป
- เมธอดเพิ่มเติมที่เกี่ยวกับ "มุมมอง" เฉพาะของข้อมูลชุดเดียวกัน

โดยไม่ต้องสร้างตารางใหม่หรือ migration schema เพิ่มเลย

### 145.2 สร้าง `PublishedPost` — Proxy ของ `Post`

```python
# blog/models.py

class PublishedPostManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)


class PublishedPost(Post):
    """Proxy model: มองเห็นเฉพาะโพสต์ที่เผยแพร่แล้ว ใช้ตาราง blog_post ตัวเดียวกับ Post"""

    objects = PublishedPostManager()

    class Meta:
        proxy = True
        ordering = ["-created_at"]
        verbose_name = "โพสต์ที่เผยแพร่แล้ว"
        verbose_name_plural = "โพสต์ที่เผยแพร่แล้วทั้งหมด"

    def excerpt(self, length=150):
        """เมธอดเสริมเฉพาะของมุมมอง 'โพสต์ที่เผยแพร่แล้ว'"""
        return self.content[:length] + ("..." if len(self.content) > length else "")
```

- `class Meta: proxy = True` คือกุญแจสำคัญที่บอก Django ว่า "อย่าสร้างตารางใหม่
  ใช้ตารางของ `Post` เดิม (ซึ่งเป็น parent class) ต่อไป"
- `PublishedPost` inherit จาก `Post` โดยตรง (ไม่ใช่จาก `models.Model`) เพื่อให้ Django
  รู้ว่านี่คือ proxy ของโมเดลไหน
- ใช้งานได้ทันที: `PublishedPost.objects.all()` จะคืนเฉพาะโพสต์ที่ `is_published=True`
  และเรียงตาม `-created_at` โดยอัตโนมัติ — โดยที่ `Post.objects.all()` (ของโมเดลเดิม)
  ยังคงคืนข้อมูล**ทุกแถว**เหมือนเดิมทุกประการ ไม่ได้รับผลกระทบใด ๆ

### 145.3 ใช้ Proxy Model ใน Django Admin

จุดประโยชน์ที่ชัดเจนมากของ proxy model คือการแยกหน้า Admin ให้ผู้ดูแลระบบเห็น
"มุมมอง" ที่ต่างกันของข้อมูลชุดเดียวกัน:

```python
# blog/admin.py
from django.contrib import admin
from .models import Post, PublishedPost


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "is_published", "created_at"]


@admin.register(PublishedPost)
class PublishedPostAdmin(admin.ModelAdmin):
    list_display = ["title", "category", "created_at", "view_count"]

    def get_queryset(self, request):
        # ใช้ default manager ของ PublishedPost อยู่แล้ว แต่เขียนชัดเจนไว้เพื่อความปลอดภัย
        return PublishedPost.objects.all()
```

ผลลัพธ์: เมนู Admin จะมีทั้ง "Posts" (เห็นทุกโพสต์) และ "โพสต์ที่เผยแพร่แล้วทั้งหมด"
(เห็นเฉพาะที่เผยแพร่) แยกกันชัดเจน โดยข้อมูลจริงอยู่ในตารางเดียวกัน (`blog_post`)
เท่านั้น เราจะเจาะลึกการ customize Admin เพิ่มเติมใน Part 017-018

### 145.4 ข้อจำกัดของ Proxy Model

- **เพิ่ม field ใหม่ไม่ได้เด็ดขาด** เพราะ proxy ใช้ตารางเดียวกับ parent การเพิ่ม field
  ใน proxy class จะทำให้ Django แจ้ง error ทันทีตอนรัน `makemigrations`
- เพิ่มได้เฉพาะ: Python methods, custom manager, `Meta` options บางตัว (`ordering`,
  `verbose_name`, `permissions`, `proxy`, `get_latest_by` — **ไม่รวม** field-level
  options อย่าง `unique_together`/`constraints`/`indexes` ซึ่งเป็นเรื่องของ schema)
- Proxy model **inherit constraints/indexes ของ parent มาด้วย** (เพราะใช้ตารางเดียวกัน)
  แต่ **ไม่สามารถเพิ่ม constraint ใหม่ที่ parent ไม่มี** ได้
- ถ้าต้องการ proxy ของ proxy (ซ้อนกันหลายชั้น) ก็ทำได้ แต่ควรระวังไม่ให้ซับซ้อนเกินไป

### 145.5 ตัวอย่างเพิ่มเติม: `DraftPost`

```python
class DraftPostManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=False)


class DraftPost(Post):
    objects = DraftPostManager()

    class Meta:
        proxy = True
        ordering = ["-updated_at"]  # โชว์ draft ที่แก้ไขล่าสุดขึ้นก่อน
        verbose_name = "ฉบับร่าง"
        verbose_name_plural = "ฉบับร่างทั้งหมด"
```

ตอนนี้เรามีมุมมองสามแบบของข้อมูล `blog_post` ตารางเดียว: `Post` (ทุกแถว),
`PublishedPost` (เฉพาะเผยแพร่แล้ว), `DraftPost` (เฉพาะฉบับร่าง) — เป็นตัวอย่างที่ดีของ
การใช้ proxy model เพื่อสื่อความหมายทางธุรกิจ (business intent) ให้ชัดเจนขึ้นในโค้ด

---

## ขั้นตอนที่ 146: Custom Manager เบื้องต้น — สร้าง `PublishedManager`

### 146.1 Manager คืออะไรกันแน่

ทุกครั้งที่คุณเขียน `Post.objects.all()` หรือ `Post.objects.filter(...)` คุณกำลังใช้
**Manager** ชื่อ `objects` ซึ่งเป็น instance ของ `models.Manager` ที่ Django เพิ่มให้
ทุกโมเดลโดยอัตโนมัติถ้าคุณไม่ได้ประกาศ manager เอง

Manager คือ **ตัวกลางระหว่างโมเดลกับฐานข้อมูล** — มันสร้าง `QuerySet` เริ่มต้นให้
(ผ่านเมธอด `get_queryset()`) ทุก query ที่คุณเขียนจริง ๆ แล้วเริ่มต้นจากการเรียก
`Manager.get_queryset()` ก่อนเสมอ

### 146.2 สร้าง `PublishedManager` แรกของเรา

```python
# blog/models.py

class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)
```

- override เมธอด `get_queryset()` เพื่อ "กรอง" queryset เริ่มต้นก่อนที่มันจะถูกใช้งาน
- `super().get_queryset()` เรียก behavior เดิมของ `models.Manager` ก่อน (คืน queryset
  ที่ครอบคลุมทุกแถว) แล้วค่อย `.filter(is_published=True)` ต่อท้าย

### 146.3 นำมาใช้กับ `Post` เป็น manager เสริม

```python
class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.PositiveIntegerField(default=0)

    objects = models.Manager()      # manager มาตรฐาน — เห็นทุกแถว (ควรอยู่บนสุดเสมอ)
    published = PublishedManager()  # manager เสริม — เห็นเฉพาะที่เผยแพร่แล้ว

    class Meta(TimeStampedModel.Meta):
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"], name="unique_slug_per_category"
            ),
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name="post_view_count_non_negative",
            ),
        ]

    def __str__(self):
        return self.title
```

ตอนนี้ใช้งานได้สองแบบ:

```python
>>> Post.objects.count()      # ทุกโพสต์ ทั้งเผยแพร่และร่าง
15
>>> Post.published.count()    # เฉพาะที่ is_published=True
9
```

### 146.4 เพิ่มเมธอดพิเศษให้ Manager

Manager ไม่ได้ทำได้แค่ override `get_queryset()` แต่ยังเพิ่มเมธอดของตัวเองได้ด้วย:

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)

    def by_author(self, user):
        """หมายเหตุ: ตัวอย่างนี้สมมติว่ามี field author บน Post ในโปรเจกต์จริงของคุณ
        (โมเดล Post ในบทเรียนนี้ยังไม่มี field author — ปรับตามสคีมาจริงของคุณ)"""
        return self.get_queryset().filter(author=user)

    def most_viewed(self, limit=5):
        return self.get_queryset().order_by("-view_count")[:limit]
```

ใช้งาน: `Post.published.most_viewed(10)` — ได้โพสต์ที่เผยแพร่แล้ว 10 อันดับยอดวิวสูงสุด

### 146.5 ข้อจำกัดสำคัญของ Manager แบบธรรมดา — ทำไมต้องมี QuerySet ต่อ

ลองสมมติว่าอยากเพิ่มเมธอด `by_category(category)` ให้ manager แล้ว **chain ต่อ** กับ
`most_viewed()`:

```python
>>> Post.published.by_category(django_category).most_viewed(5)
AttributeError: 'QuerySet' object has no attribute 'most_viewed'
```

เกิด error ทันที! เหตุผลคือ `by_category()` คืนค่าเป็น `QuerySet` ธรรมดา (จาก
`self.get_queryset().filter(...)`) ซึ่ง **ไม่มี** เมธอด `most_viewed()` ติดมาด้วย
(เมธอดนั้นอยู่บน Manager ไม่ใช่บน QuerySet) — นี่คือข้อจำกัดสำคัญที่สุดของการเขียน
เมธอดไว้บน `Manager` ตรง ๆ: **เมธอดที่กำหนดเองจะเรียกได้แค่ "ระดับแรก" เท่านั้น
ไม่สามารถ chain ต่อกันเป็นทอด ๆ ได้**

วิธีแก้ปัญหานี้คือการย้ายเมธอดไปไว้ที่ **Custom QuerySet** แทน ซึ่งเราจะเรียนใน
ขั้นตอนที่ 147

---

## ขั้นตอนที่ 147: Custom QuerySet + `as_manager()` — chain method ได้แบบมืออาชีพ

### 147.1 แนวคิด: ย้ายเมธอดจาก Manager ไปไว้ที่ QuerySet

แทนที่จะเขียนเมธอดกรองข้อมูลไว้บน `Manager` เราจะเขียนไว้บน **`QuerySet` ของเราเอง**
เพราะทุกเมธอดของ `QuerySet` (เช่น `.filter()`, `.exclude()`, `.order_by()`) จะคืนค่า
เป็น `QuerySet` เสมอ ทำให้ chain ต่อกันได้ไม่จำกัด ถ้าเมธอดที่เราเขียนเองก็คืน
`QuerySet` เหมือนกัน มันก็จะ chain ต่อได้เช่นเดียวกัน

### 147.2 สร้าง `PostQuerySet`

```python
# blog/models.py
from django.db.models import Count, Q


class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(is_published=True)

    def draft(self):
        return self.filter(is_published=False)

    def by_category(self, category):
        return self.filter(category=category)

    def by_tag(self, tag):
        return self.filter(tags=tag)

    def search(self, keyword):
        return self.filter(
            Q(title__icontains=keyword) | Q(content__icontains=keyword)
        )

    def popular(self, min_views=100):
        return self.filter(view_count__gte=min_views)

    def with_comment_count(self):
        return self.annotate(comment_count=Count("comments"))
```

ทุกเมธอดคืนค่า `self.filter(...)` หรือ `self.annotate(...)` ซึ่งเป็น `QuerySet`
เสมอ — นี่คือกฎเหล็กของการเขียน custom QuerySet method: **ต้อง return QuerySet
เสมอ ห้าม return list, ห้าม evaluate queryset ก่อนเวลา (เช่น ห้ามใส่ `list(...)`
หรือ slicing ที่ไม่จำเป็นระหว่างทาง)**

### 147.3 แปลง QuerySet เป็น Manager ด้วย `as_manager()`

```python
class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.PositiveIntegerField(default=0)

    objects = PostQuerySet.as_manager()   # ✨ ตอนนี้ objects มีเมธอดของ PostQuerySet ทั้งหมด

    class Meta(TimeStampedModel.Meta):
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"], name="unique_slug_per_category"
            ),
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name="post_view_count_non_negative",
            ),
        ]

    def __str__(self):
        return self.title
```

`PostQuerySet.as_manager()` เป็น classmethod ที่ Django เตรียมไว้ให้ — มันสร้าง
`Manager` ใหม่ที่มี `get_queryset()` คืน `PostQuerySet` โดยอัตโนมัติ ทำให้ทุกเมธอดที่
เราเขียนใน `PostQuerySet` เรียกผ่าน `Post.objects.<method>()` ได้ทันที **และ chain
ต่อกันได้ไม่จำกัด**

### 147.4 ตัวอย่างการ chain แบบเต็ม

```python
>>> from blog.models import Post, Category

>>> django_cat = Category.objects.get(slug="django")

>>> Post.objects.published().by_category(django_cat).search("orm").popular(50)
<QuerySet [<Post: Django ORM ขั้นสูง>, <Post: เจาะลึก QuerySet>]>

>>> Post.objects.published().with_comment_count().order_by("-comment_count")[:5]
<QuerySet [<Post: บทความยอดนิยม>, ...]>

>>> Post.objects.draft().by_tag(python_tag)
<QuerySet [<Post: ร่างบทความ Python>]>
```

นี่คือ pattern การเขียนโค้ด ORM ที่อ่านง่ายเหมือนประโยคภาษาอังกฤษ (**fluent
interface**) ซึ่งเป็นมาตรฐานของโปรเจกต์ Django มืออาชีพระดับโลก

### 147.5 ผสมสองแบบเข้าด้วยกัน: `Manager.from_queryset()`

บางครั้งเราอยากมีทั้งเมธอดที่เรียกจาก `Manager` โดยตรง (ไม่ผ่าน queryset เช่น
สร้าง object ใหม่) **และ** เมธอดที่ chain ได้แบบ QuerySet พร้อมกัน ใช้
`Manager.from_queryset()`:

```python
class PostManager(models.Manager.from_queryset(PostQuerySet)):
    def create_draft(self, **kwargs):
        kwargs["is_published"] = False
        return self.create(**kwargs)


class Post(TimeStampedModel):
    # ... fields เดิม ...

    objects = PostManager()
```

`models.Manager.from_queryset(PostQuerySet)` สร้าง class ใหม่ที่รวมเมธอดทั้งหมดจาก
`PostQuerySet` เข้ามาเป็นเมธอดของ Manager (เรียกได้ทั้งสองแบบ) แล้วเราค่อย inherit
ต่อเพื่อเพิ่มเมธอดพิเศษที่ไม่เกี่ยวกับการ filter/query อย่าง `create_draft()`

```python
>>> Post.objects.create_draft(title="ร่างใหม่", slug="new-draft", content="...")
>>> Post.objects.published().popular()  # ยังใช้เมธอดจาก QuerySet ได้ตามปกติ
```

### 147.6 ตารางสรุป 3 แนวทางการสร้าง Manager/QuerySet

| แนวทาง | Chain ได้ไหม | เหมาะกับ | ตัวอย่าง |
|---|---|---|---|
| Manager ธรรมดา override `get_queryset()` | ❌ เมธอดเสริมเรียกได้แค่ระดับแรก | filter เริ่มต้นง่าย ๆ ที่ไม่ต้องเรียกต่อ | `PublishedManager` (ขั้นตอนที่ 146) |
| Custom QuerySet + `as_manager()` | ✅ chain ได้ไม่จำกัด | ต้องการ query ที่ยืดหยุ่น ประกอบกันได้หลายแบบ | `PostQuerySet.as_manager()` |
| `Manager.from_queryset(QuerySet)` | ✅ chain ได้ไม่จำกัด + มีเมธอดพิเศษของ Manager เอง | ต้องการทั้งสองอย่างพร้อมกัน (query + non-query methods เช่น `create_draft`) | `PostManager` |

**คำแนะนำระดับมืออาชีพ**: ให้เริ่มต้นด้วย **Custom QuerySet + `as_manager()`**
เป็นค่าเริ่มต้นสำหรับทุกโมเดลที่มี query logic ซับซ้อนกว่าพื้นฐาน เพราะยืดหยุ่นที่สุด
และ Django REST Framework, django-filter และ library อื่น ๆ ส่วนใหญ่ก็ออกแบบมาให้
ทำงานร่วมกับ QuerySet ได้ดีอยู่แล้ว

---

## ขั้นตอนที่ 148: Manager หลายตัวในโมเดลเดียว และผลต่อ default manager

### 148.1 กฎ "Manager แรกที่ประกาศ = Default Manager"

Django มีกฎสำคัญที่มือใหม่มักไม่รู้: **Manager ตัวแรกที่ถูกประกาศในโมเดล (เรียงตาม
ลำดับบรรทัดในโค้ด) จะกลายเป็น "default manager"** ของโมเดลนั้น ไม่ว่าจะตั้งชื่อว่าอะไร
ก็ตาม (ไม่จำเป็นต้องชื่อ `objects`)

Default manager ถูกใช้ในหลายที่ที่คุณอาจไม่ทันสังเกต:

- Django Admin ใช้ default manager เพื่อดึงรายการแสดงผล (ถ้าไม่ override
  `get_queryset()` เอง)
- Reverse relation จาก ForeignKey (เช่น `category.posts.all()`) จะใช้
  `_base_manager` (ค่าเริ่มต้นคือเหมือน default manager เว้นแต่ตั้ง
  `base_manager_name` แยก — ดูหัวข้อ 148.3)
- Migration serialization (เมื่อ Django ต้อง serialize ค่า default ของ field
  บางประเภทที่อ้างอิง manager)

### 148.2 ปัญหาที่เกิดขึ้นถ้าตั้งลำดับ Manager ผิด

ลองดูโค้ดที่ **มีปัญหาแอบแฝง**:

```python
class Post(TimeStampedModel):
    # ... fields เดิม ...

    published = PublishedManager()          # ⚠️ ประกาศเป็นตัวแรก!
    objects = PostQuerySet.as_manager()      # ประกาศเป็นตัวที่สอง
```

เพราะ `published` ถูกประกาศก่อน มันจะกลายเป็น **default manager** ของ `Post`
โดยไม่ได้ตั้งใจ! ผลกระทบที่ตามมา:

```python
>>> category = Category.objects.get(slug="django")
>>> category.posts.all()   # reverse relation ผ่าน related_name="posts"
<QuerySet [<Post: โพสต์ที่เผยแพร่แล้ว 1>, ...]>  # ⚠️ เห็นเฉพาะที่เผยแพร่ ทั้งที่ไม่ได้ตั้งใจ!
```

โพสต์ที่เป็นฉบับร่างในหมวดหมู่นี้จะ **หายไปจากผลลัพธ์แบบเงียบ ๆ** โดยไม่มี error
ใด ๆ เตือน — เป็นบั๊กที่หายากมากในโปรเจกต์จริง เพราะทุกอย่างดู "ทำงานได้" เพียงแค่
ข้อมูลที่ได้ไม่ครบ

### 148.3 วิธีแก้ที่ถูกต้อง: จัดลำดับ + `Meta.base_manager_name`

**วิธีที่ 1 (ควรทำเป็นนิสัยเสมอ): ประกาศ manager มาตรฐาน (`objects`) เป็นตัวแรกเสมอ**

```python
class Post(TimeStampedModel):
    # ... fields เดิม ...

    objects = PostQuerySet.as_manager()   # ✅ ตัวแรก = default manager ที่ถูกต้อง
    published = PublishedManager()        # manager เสริม ตัวที่สอง
```

**วิธีที่ 2: กำหนดชัดเจนด้วย `Meta.default_manager_name` และ `Meta.base_manager_name`**
เมื่อมีเหตุผลจำเป็นต้องประกาศ manager พิเศษไว้ก่อน (เช่น library บางตัวกำหนดให้ต้องทำ):

```python
class Post(TimeStampedModel):
    # ... fields เดิม ...

    published = PublishedManager()
    objects = PostQuerySet.as_manager()

    class Meta(TimeStampedModel.Meta):
        base_manager_name = "objects"     # ใช้ objects สำหรับ related manager/reverse FK
        default_manager_name = "objects"  # ใช้ objects เป็น default manager อย่างชัดเจน
```

- `base_manager_name` ควบคุมว่า **reverse relation** (เช่น `category.posts`) และ
  internal operations อื่น ๆ ของ Django จะใช้ manager ตัวไหนเป็นฐาน — ควรตั้งเป็น
  manager ที่ **ไม่ filter อะไรเลย** เสมอ (เห็นข้อมูลครบทุกแถว) เพื่อความปลอดภัย
- `default_manager_name` ควบคุมว่า manager ตัวไหนจะถูกใช้เป็นค่าเริ่มต้นทั่วไป

### 148.4 ผลกระทบต่อ Django Admin

`ModelAdmin` เริ่มต้นจะใช้ `self.model._default_manager.get_queryset()` เพื่อดึง
รายการมาแสดง ถ้า default manager ของคุณ filter ข้อมูลบางส่วนออกไปโดยไม่ตั้งใจ
(เหมือนตัวอย่างในหัวข้อ 148.2) **หน้า Admin จะไม่แสดงข้อมูลบางแถวเลย** โดยไม่มี
error แจ้งเตือนใด ๆ — ผู้ดูแลระบบอาจคิดว่าข้อมูลหายไปจากฐานข้อมูลจริง ๆ

วิธีป้องกันที่ดีที่สุด: **manager ชื่อ `objects` ควรเป็น "unfiltered" เสมอ** (เห็น
ทุกแถวไม่มีเงื่อนไข) ส่วน manager ที่ filter ข้อมูล (เช่น `published`, `draft`)
ให้ตั้งชื่อสื่อความหมายชัดเจนและประกาศเป็นตัวรอง — นี่คือ **convention มาตรฐาน**
ที่ทีม Django ทั่วโลกใช้กัน

### 148.5 ตารางสรุป Best Practice

| กฎ | เหตุผล |
|---|---|
| `objects` ต้องเป็น manager ตัวแรกที่ประกาศเสมอ | ป้องกันการเป็น default manager โดยไม่ตั้งใจของ manager ที่ filter ข้อมูล |
| `objects` ต้องไม่ filter อะไรเลย | Admin, reverse relation, และโค้ดส่วนอื่นคาดหวังว่าเห็นข้อมูลครบ |
| manager ที่ filter ข้อมูล ตั้งชื่อสื่อความหมาย | เช่น `published`, `draft`, `active` — อ่านแล้วรู้ทันทีว่ากรองอะไร |
| ตั้ง `base_manager_name = "objects"` อย่างชัดเจนถ้าลำดับซับซ้อน | กันบั๊กแบบในหัวข้อ 148.2 ได้แน่นอน 100% |
| หลีกเลี่ยงการมี manager มากเกินความจำเป็น | มากไปจะทำให้โมเดลอ่านยาก ควรมีแค่ที่ใช้จริงบ่อย ๆ |

---

## ขั้นตอนที่ 149: Overriding `save()` และข้อควรระวัง

### 149.1 ทำไมบางครั้งต้อง override `save()`

กรณีใช้งานที่พบบ่อยที่สุดคือการ **auto-generate slug** จาก title โดยที่ผู้ใช้ไม่ต้อง
กรอกเอง:

```python
from django.utils.text import slugify


class Post(TimeStampedModel):
    # ... fields เดิม ...

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

- **ต้องเรียก `super().save(*args, **kwargs)` เสมอ** เพื่อให้ Django ทำการบันทึก
  ข้อมูลลงฐานข้อมูลจริง ถ้าลืมเรียก object จะไม่ถูกบันทึกเลยแต่โค้ดจะไม่ error ใด ๆ
- `*args, **kwargs` ต้องส่งต่อไปด้วยเสมอ เพราะ `save()` รับ parameter อื่น ๆ ได้ เช่น
  `force_insert`, `update_fields` — ถ้าไม่ส่งต่อ ฟีเจอร์เหล่านี้จะพังไปด้วย

### 149.2 ตัวอย่างที่สมบูรณ์กว่า: ป้องกัน slug ซ้ำ

```python
from django.utils.text import slugify


class Post(TimeStampedModel):
    # ... fields เดิม ...

    def save(self, *args, **kwargs):
        if not self.slug:
            base_slug = slugify(self.title)
            slug = base_slug
            counter = 1
            # เช็คว่า slug นี้ถูกใช้ไปแล้วหรือยัง (ในหมวดหมู่เดียวกัน — สอดคล้องกับ
            # UniqueConstraint(fields=["category", "slug"]) ที่ตั้งไว้ใน Meta)
            qs = Post.objects.filter(category=self.category, slug=slug)
            if self.pk:
                qs = qs.exclude(pk=self.pk)  # ตอน update ไม่ต้องเทียบกับตัวเอง
            while qs.exists():
                slug = f"{base_slug}-{counter}"
                counter += 1
                qs = Post.objects.filter(category=self.category, slug=slug)
                if self.pk:
                    qs = qs.exclude(pk=self.pk)
            self.slug = slug
        super().save(*args, **kwargs)
```

- `if self.pk:` เช็คว่านี่คือการ **update** object ที่มีอยู่แล้ว (มี primary key)
  หรือเป็นการ **create** ใหม่ (ยังไม่มี `pk`) — ถ้าเป็น update ต้อง exclude ตัวเองออก
  จากการเช็คซ้ำ ไม่งั้นจะคิดว่า slug ของตัวเองซ้ำกับตัวเอง
- Loop นี้ยังมี **race condition** ได้ในทางทฤษฎี (ถ้ามีสอง request บันทึกพร้อมกัน
  เป๊ะ ๆ) วิธีป้องกันที่สมบูรณ์ 100% คือพึ่ง `UniqueConstraint` ที่ระดับฐานข้อมูล
  (ที่เราตั้งไว้แล้ว) เป็นด่านสุดท้าย แล้ว catch `IntegrityError` ในชั้น view/form
  อีกที — นี่คือเหตุผลที่เราต้องมีทั้งสองชั้นการป้องกัน (defense in depth)

### 149.3 การเรียก `full_clean()` ก่อนบันทึก

`save()` ของ Django **ไม่เรียก validation อัตโนมัติ** (ต่างจาก `ModelForm` ที่เรียก
`full_clean()` ให้อัตโนมัติ) ถ้าคุณสร้าง/แก้ object ผ่าน code โดยตรง (ไม่ผ่านฟอร์ม)
`CheckConstraint`/`UniqueConstraint` จะ raise `IntegrityError` เท่านั้น (error message
ไม่เป็นมิตรกับผู้ใช้) หากต้องการ validation message ที่อ่านง่ายกว่า ให้เรียก
`full_clean()` เอง:

```python
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        self.full_clean()  # เรียก validators + clean() + constraint validation
        super().save(*args, **kwargs)
```

**ข้อควรระวัง**: การเรียก `full_clean()` ใน `save()` มีข้อเสียเช่นกัน — มันจะทำให้
`bulk_create()`/`loaddata` (fixtures) ที่ไม่ได้ผ่าน `save()` ปกติไม่ได้ validate
(เพราะ `bulk_create` ไม่เรียก `save()` เลย — ดูหัวข้อ 149.4) และอาจทำให้ performance
ช้าลงถ้า `save()` ถูกเรียกจำนวนมากในลูป จึงควรพิจารณาให้ดีว่าจำเป็นจริงหรือไม่
สำหรับแต่ละโมเดล

### 149.4 ทำไมไม่ควรใส่ business logic หนักเกินไปใน `save()`

นี่คือประเด็นสำคัญที่สุดของขั้นตอนนี้ มือใหม่มักยัด logic จำนวนมากเข้าไปใน `save()`
เช่น "ส่งอีเมลแจ้งเตือนเมื่อโพสต์ถูกเผยแพร่", "อัปเดต cache", "เรียก API ภายนอก"
ซึ่งเป็นแนวทางที่ **อันตรายมาก** ด้วยเหตุผลดังนี้:

| ปัญหา | อธิบาย |
|---|---|
| **`bulk_create()`/`bulk_update()`/`QuerySet.update()` ไม่เรียก `save()`** | ถ้า logic สำคัญอยู่ใน `save()` เท่านั้น การอัปเดตข้อมูลจำนวนมากพร้อมกันจะข้าม logic นั้นไปเงียบ ๆ ทั้งหมด |
| **Signal จาก `post_save` อาจถูกยิงซ้ำ** | ถ้า `save()` ถูกเรียกซ้อนกันหลายรอบ (เช่น เรียก `.save()` ในเมธอดอื่นที่ก็เรียก `.save()` อีกที) logic ที่ผูกกับ `post_save` อาจทำงานซ้ำโดยไม่ตั้งใจ |
| **Transaction/atomicity ยากขึ้น** | ถ้า `save()` เรียก API ภายนอก (เช่น ส่งอีเมล) แล้วเกิด error ระหว่างทาง การ rollback transaction ของฐานข้อมูลจะไม่สามารถ "ยกเลิก" การส่งอีเมลที่ส่งไปแล้วได้ |
| **Testing ยากขึ้นมาก** | Unit test ที่แค่อยากสร้าง object ธรรมดา (`Post.objects.create(...)`) จะดันไปยิง side effect หนัก ๆ (เช่น เรียก external API จริง) ทุกครั้งที่รัน test โดยไม่ได้ตั้งใจ |
| **ละเมิดหลัก Single Responsibility** | `save()` ควรมีหน้าที่แค่ "บันทึกข้อมูลให้ถูกต้อง" ไม่ใช่ "ทำทุกอย่างที่เกี่ยวข้องกับ business process" |

**แนวทางที่แนะนำแทน**:

1. **Service layer function** — แยก logic ที่ซับซ้อนออกมาเป็นฟังก์ชันต่างหาก
   เรียกจาก view/API โดยตรง แทนที่จะฝังใน `save()`

```python
# blog/services.py
from django.core.mail import send_mail


def publish_post(post):
    """Service function: เผยแพร่โพสต์ + ส่งอีเมลแจ้งเตือน (แยกจาก model.save())"""
    post.is_published = True
    post.save(update_fields=["is_published", "updated_at"])
    send_mail(
        subject=f"บทความใหม่: {post.title}",
        message=post.content[:200],
        from_email="noreply@example.com",
        recipient_list=["subscribers@example.com"],
    )
```

2. **Django Signals** (`post_save`) — ใช้ได้ แต่ต้องระมัดระวังเรื่องการยิงซ้ำและ
   ทดสอบยากเช่นกัน เราจะเจาะลึกเรื่อง signals แบบเต็มใน **Part 019**
   (Signals และ Django Lifecycle Hooks) รวมถึงข้อดี-ข้อเสียเทียบกับ service layer

3. **สิ่งที่เหมาะจะอยู่ใน `save()` ได้จริง**: การคำนวณ/ปรับ field ของ object ตัวเอง
   ก่อนบันทึก (เช่น auto-slug, normalize ข้อมูล เช่น `self.email = self.email.lower()`,
   คำนวณ field ที่ derive จาก field อื่น) — สิ่งเหล่านี้ **ไม่มี side effect ภายนอก**
   จึงปลอดภัยที่จะอยู่ใน `save()`

### 149.5 Overriding `delete()` — หลักการเดียวกัน

```python
class Post(TimeStampedModel):
    # ... fields เดิม ...

    def delete(self, *args, **kwargs):
        # ตัวอย่างที่ปลอดภัย: แค่ปรับ state ของ object ตัวเองก่อนลบจริง
        # (ไม่ใช่การเรียก external service หนัก ๆ)
        return super().delete(*args, **kwargs)
```

เช่นเดียวกับ `save()` — `delete()` แบบ instance (`post.delete()`) ก็ **ไม่ถูกเรียก**
เมื่อใช้ `QuerySet.delete()` แบบ bulk (เช่น `Post.objects.filter(is_published=False)
.delete()`) ดังนั้นอย่าพึ่งพา override `delete()` สำหรับ logic ที่ต้องทำงานทุกครั้ง
100% — ถ้าต้องการความแน่นอนขนาดนั้น ต้องใช้ signal `pre_delete`/`post_delete`
(เจาะลึกใน Part 019) ซึ่งถูกยิงทั้งจาก instance delete และในบางกรณีของ
`CASCADE` แต่ **ก็ยังไม่ถูกยิงจาก `QuerySet.delete()` แบบ bulk เช่นกัน** —
นี่คือ gotcha ที่สำคัญมากที่ควรจำไว้เสมอ

---

## ขั้นตอนที่ 150: สรุปและแบบฝึกหัด

### 150.1 โค้ดสรุปฉบับเต็ม — `blog/models.py` หลัง Refactor

นี่คือไฟล์ `blog/models.py` ฉบับสมบูรณ์ที่รวมทุกเทคนิคจาก Part นี้เข้าด้วยกัน:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.db.models import Count, Q
from django.utils.text import slugify


class TimeStampedModel(models.Model):
    """Abstract base: เพิ่ม created_at / updated_at ให้โมเดลลูกทุกตัว"""

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name = "หมวดหมู่"
        verbose_name_plural = "หมวดหมู่ทั้งหมด"
        ordering = ["name"]

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    class Meta:
        ordering = ["name"]

    def __str__(self):
        return self.name


class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(is_published=True)

    def draft(self):
        return self.filter(is_published=False)

    def by_category(self, category):
        return self.filter(category=category)

    def by_tag(self, tag):
        return self.filter(tags=tag)

    def search(self, keyword):
        return self.filter(Q(title__icontains=keyword) | Q(content__icontains=keyword))

    def popular(self, min_views=100):
        return self.filter(view_count__gte=min_views)

    def with_comment_count(self):
        return self.annotate(comment_count=Count("comments"))


class PostManager(models.Manager.from_queryset(PostQuerySet)):
    def create_draft(self, **kwargs):
        kwargs["is_published"] = False
        return self.create(**kwargs)


class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.PositiveIntegerField(default=0)

    objects = PostManager()  # ✅ manager แรก, ไม่ filter อะไร, chain ได้เต็มรูปแบบ

    class Meta(TimeStampedModel.Meta):
        base_manager_name = "objects"
        default_manager_name = "objects"
        constraints = [
            models.UniqueConstraint(
                fields=["category", "slug"],
                name="unique_slug_per_category",
                violation_error_message="Slug นี้ถูกใช้ไปแล้วในหมวดหมู่นี้",
            ),
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name="post_view_count_non_negative",
            ),
        ]
        indexes = [
            models.Index(fields=["is_published", "-created_at"], name="post_published_idx"),
        ]

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            base_slug = slugify(self.title)
            slug = base_slug
            counter = 1
            qs = Post.objects.filter(category=self.category, slug=slug)
            if self.pk:
                qs = qs.exclude(pk=self.pk)
            while qs.exists():
                slug = f"{base_slug}-{counter}"
                counter += 1
                qs = Post.objects.filter(category=self.category, slug=slug)
                if self.pk:
                    qs = qs.exclude(pk=self.pk)
            self.slug = slug
        super().save(*args, **kwargs)


class PublishedPostManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)


class PublishedPost(Post):
    objects = PublishedPostManager()

    class Meta:
        proxy = True
        ordering = ["-created_at"]
        verbose_name = "โพสต์ที่เผยแพร่แล้ว"
        verbose_name_plural = "โพสต์ที่เผยแพร่แล้วทั้งหมด"

    def excerpt(self, length=150):
        return self.content[:length] + ("..." if len(self.content) > length else "")


class Comment(TimeStampedModel):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    text = models.TextField()
    parent = models.ForeignKey(
        "self", on_delete=models.CASCADE, null=True, blank=True, related_name="replies"
    )

    class Meta(TimeStampedModel.Meta):
        ordering = ["created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["post", "author", "text"],
                condition=models.Q(parent__isnull=True),
                name="unique_top_level_comment_text",
            ),
            models.CheckConstraint(
                check=~models.Q(parent=models.F("id")),
                name="comment_parent_not_self",
            ),
        ]

    def __str__(self):
        return f"Comment by {self.author} on {self.post}"


class Profile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)

    def __str__(self):
        return f"Profile of {self.user}"
```

### 150.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ `class Meta` option สำคัญทั้งหมด: `ordering`, `verbose_name`,
  `verbose_name_plural`, `db_table`, `unique_together`, `indexes`, `abstract`
- ✅ ใช้ `UniqueConstraint` และ `CheckConstraint` แทน `unique_together` แบบเก่า
  รวมถึงเทคนิค conditional constraint ด้วย `condition=Q(...)`
- ✅ สร้าง Abstract Base Class (`TimeStampedModel`) เพื่อลดโค้ดซ้ำซ้อนตามหลัก DRY
- ✅ แยกแยะความแตกต่างระหว่าง Abstract Base Class, Multi-table Inheritance และ
  Proxy Model รู้ว่าเมื่อไหร่ควรใช้แบบไหน
- ✅ สร้าง Proxy Model (`PublishedPost`) เพื่อสร้างมุมมองข้อมูลที่ต่างกันโดยไม่แตะ schema
- ✅ สร้าง Custom Manager (`PublishedManager`) และเข้าใจข้อจำกัดเรื่องการ chain
- ✅ สร้าง Custom QuerySet + `as_manager()` เพื่อให้ chain method ได้ไม่จำกัด
- ✅ เข้าใจกฎ default manager, ผลกระทบต่อ Admin/reverse relation, และวิธีควบคุมด้วย
  `base_manager_name`/`default_manager_name`
- ✅ Override `save()` อย่างถูกต้อง (auto-slug, uniqueness) และรู้ข้อควรระวังของการ
  ใส่ business logic หนักเกินไปใน `save()`/`delete()`

### 150.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน `TimeStampedModel` เป็น abstract base และให้ `Post`, `Comment` inherit ได้
- [ ] เขียน `UniqueConstraint` แบบมี `condition` ได้อย่างน้อย 1 ตัวอย่าง
- [ ] เขียน `CheckConstraint` ที่ใช้ `F()` เปรียบเทียบสอง field ในแถวเดียวกันได้
- [ ] อธิบายความแตกต่างระหว่าง Abstract Base / MTI / Proxy ได้โดยไม่ต้องเปิดตำรา
- [ ] สร้าง Proxy Model อย่างน้อย 1 ตัวที่ใช้ manager กรองข้อมูลต่างจาก parent
- [ ] เขียน Custom QuerySet ที่มีอย่างน้อย 4 เมธอด และ chain กันได้จริงใน shell
- [ ] อธิบายได้ว่าทำไม manager ตัวแรกที่ประกาศจึงสำคัญ และรู้วิธีป้องกันด้วย
      `base_manager_name`
- [ ] Override `save()` เพื่อ auto-generate slug พร้อมจัดการ slug ซ้ำได้
- [ ] อธิบายได้ว่าทำไมไม่ควรส่งอีเมล/เรียก API ภายนอกใน `save()` โดยตรง

### 150.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: Refactor โมเดล `blog` ทั้งหมดของคุณให้ตรงกับโค้ดในหัวข้อ 150.1
ทุกประการ จากนั้นรัน `python manage.py makemigrations` แล้ว **อ่าน migration ที่เกิดขึ้น
ทีละบรรทัด** เขียนอธิบายลงในไฟล์ `notes.md` ว่าแต่ละ operation (`AddConstraint`,
`AlterModelOptions`, `AddIndex` ฯลฯ) หมายถึงอะไร ก่อนจะรัน `migrate` จริง

**แบบฝึกหัดที่ 2**: เพิ่ม field `is_featured = models.BooleanField(default=False)`
ลงใน `Post` แล้วสร้าง `UniqueConstraint` แบบ conditional ที่บังคับว่า **ในแต่ละ
category จะมีโพสต์ที่ `is_featured=True` ได้สูงสุดแค่ 1 โพสต์เท่านั้น** (ใบ้: ใช้
`condition=Q(is_featured=True)` กับ `fields=["category"]`) ทดสอบใน shell ว่าเมื่อ
พยายามตั้งโพสต์ที่สองในหมวดหมู่เดียวกันเป็น featured จะเกิด `IntegrityError` จริง

**แบบฝึกหัดที่ 3**: สร้าง Proxy Model ใหม่ชื่อ `PopularPost` (proxy ของ `Post`)
ที่มี default manager คืนเฉพาะโพสต์ที่ `view_count >= 1000` และเรียงตาม `view_count`
จากมากไปน้อย พร้อมเพิ่มเมธอด `popularity_label()` ที่คืนค่าข้อความ `"🔥 Viral"` ถ้า
`view_count >= 10000` หรือ `"⭐ Popular"` ถ้าน้อยกว่านั้น

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้าง `CommentQuerySet` ที่มีเมธอด `top_level()` (เฉพาะ
คอมเมนต์ที่ `parent__isnull=True`), `by_post(post)`, และ `recent(days=7)` (คอมเมนต์ที่
สร้างภายใน N วันที่ผ่านมา — ใบ้: ใช้ `timezone.now() - timedelta(days=days)` ร่วมกับ
`created_at__gte`) แปลงเป็น manager ด้วย `as_manager()` แล้วทดสอบ chain ทั้งสามเมธอด
เข้าด้วยกันใน Django shell เช่น
`Comment.objects.top_level().by_post(post).recent(3)`

### 150.5 คำถามที่พบบ่อย (FAQ)

**Q: ถ้าเปลี่ยนจาก `unique_together` เป็น `UniqueConstraint` กับโมเดลที่มีข้อมูลอยู่
แล้วใน production จะเป็นอันตรายไหม?**
A: การเปลี่ยนแค่วิธีประกาศ (จาก `unique_together` เป็น `UniqueConstraint` ที่มีเงื่อนไข
เดียวกันทุกประการ ไม่มี `condition`) จะสร้าง migration ที่แค่ drop constraint เก่าแล้ว
สร้างใหม่ที่มีผลลัพธ์เหมือนเดิม ปลอดภัยสำหรับข้อมูลที่ผ่านการตรวจสอบ unique มาแล้ว
แต่ควรรัน migration ในช่วง maintenance window เสมอสำหรับตารางขนาดใหญ่ เพราะการสร้าง
index/constraint ใหม่อาจ lock ตารางชั่วคราว (เจาะลึกเรื่องนี้ใน Part 016)

**Q: ใช้ `Manager.from_queryset()` กับ `as_manager()` ต่างกันจริง ๆ ตรงไหน ในเมื่อ
ทั้งคู่ก็ได้ manager ที่ chain ได้เหมือนกัน?**
A: `PostQuerySet.as_manager()` สร้าง manager instance ให้ทันทีแบบสั้น ๆ เหมาะกับกรณี
ที่ไม่ต้องการเมธอดพิเศษอื่นนอกเหนือจากที่มีใน QuerySet ส่วน
`models.Manager.from_queryset(PostQuerySet)` คืน **class** ที่คุณสามารถ inherit ต่อ
เพื่อเพิ่มเมธอดที่ไม่ใช่ query logic (เช่น `create_draft()`) — เลือกใช้ตามความจำเป็น
ถ้าไม่แน่ใจให้เริ่มจาก `as_manager()` ก่อนเสมอ เพราะเขียนน้อยกว่าและอ่านง่ายกว่า

**Q: ทำไม field ที่ `unique=True` เดี่ยว ๆ (เช่น `Tag.name`) ไม่ต้องใช้
`UniqueConstraint` ด้วย?**
A: `unique=True` ที่ระดับ field เป็นวิธีสั้น ๆ ที่ Django แปลงเป็น unique index
ให้อัตโนมัติสำหรับ **field เดียว** อยู่แล้ว ใช้ `UniqueConstraint` เมื่อต้องการ
ความ unique ที่เกี่ยวข้องกับ **หลาย field รวมกัน** หรือต้องการ `condition`/ตั้งชื่อ
constraint เอง/ใส่ custom error message เท่านั้น สำหรับ field เดี่ยวธรรมดา
`unique=True` ยังคงเป็นวิธีที่กระชับและถูกต้องที่สุด

**Q: ถ้าลืมเรียก `super().save()` ใน `save()` ที่ override จะเกิดอะไรขึ้น?**
A: Object จะไม่ถูกบันทึกลงฐานข้อมูลเลย แต่โค้ดจะรันผ่านไปโดยไม่มี error ใด ๆ
(เพราะ Python ไม่รู้ว่าคุณ "ลืม" อะไร) นี่คือบั๊กที่ตรวจจับยากมาก เพราะโค้ดที่เรียก
`post.save()` จะดูเหมือนทำงานปกติ แต่พอ query กลับมาดูใหม่ (โดยเฉพาะจาก request/
connection อื่น) จะไม่พบข้อมูลที่เพิ่ง "บันทึก" ไป จึงควรเขียน test ที่ครอบคลุมเมธอด
`save()` ที่ override ทุกครั้ง (เจาะลึกการเขียน test สำหรับ model ใน Part 059-060)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 016: Database Migrations ขั้นสูงและการจัดการ Schema** จะพาคุณไปเจาะลึกเบื้องหลัง
การทำงานของระบบ Migration ที่เราใช้มาตลอดหลาย Part ได้แก่ การ squash migrations,
การจัดการ migration conflict เมื่อทำงานเป็นทีม, `RunPython`/`RunSQL` สำหรับ data
migration, การเขียน migration ที่ปลอดภัยสำหรับตารางขนาดใหญ่ใน production (zero-downtime
migration), และการย้อนกลับ (`migrate app_name 0003`) อย่างถูกวิธี — ทักษะเหล่านี้
คือสิ่งที่แยกความแตกต่างระหว่างนักพัฒนา Django มือใหม่กับมืออาชีพระดับ production จริง

เตรียม repository ของคุณให้พร้อม (commit โค้ดจาก Part นี้ให้เรียบร้อยก่อน) แล้วไปต่อกันเลย!
