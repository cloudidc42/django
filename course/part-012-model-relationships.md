# Part 012: ความสัมพันธ์ระหว่างโมเดล: ForeignKey, OneToOne, ManyToMany

> **ขั้นตอนที่ 111-120 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เปลี่ยนโมเดล `Post` เดี่ยว ๆ ที่คุณมีอยู่ให้กลายเป็น **schema
> ฐานข้อมูลที่เชื่อมโยงกันจริง** แบบที่ระบบบล็อกระดับมืออาชีพต้องมี คุณจะสร้าง `Category`
> (หมวดหมู่บทความ) ด้วย `ForeignKey`, ขยาย Django `User` ด้วย `Profile` ผ่าน
> `OneToOneField`, เพิ่มระบบ `Tag` ด้วย `ManyToManyField` ทั้งแบบอัตโนมัติและแบบกำหนดเอง
> ผ่าน `through` model, เข้าใจ `related_name`/reverse relations, สืบค้นข้ามความสัมพันธ์
> ด้วย double underscore, รู้จักปัญหา N+1 query เบื้องต้น, และปิดท้ายด้วยการสร้าง
> `Comment` ที่ตอบกลับกันเองได้แบบ self-referential ForeignKey เมื่อจบ Part นี้ โปรเจกต์
> บล็อกของคุณจะมีโมเดลที่เชื่อมกันครบ 5 ตัว พร้อม ER diagram อธิบายภาพรวมทั้งระบบ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 111: `ForeignKey` เจาะลึก และตัวเลือก `on_delete` ทั้งหมด
- ขั้นตอนที่ 112: `OneToOneField` — ขยาย Django `User` ด้วย `Profile`
- ขั้นตอนที่ 113: `ManyToManyField` — ระบบ `Tag` ของบทความ
- ขั้นตอนที่ 114: ManyToMany ผ่าน `through` model แบบกำหนดเอง
- ขั้นตอนที่ 115: `related_name` และ Reverse Relations
- ขั้นตอนที่ 116: Query ข้ามความสัมพันธ์ด้วย Double Underscore
- ขั้นตอนที่ 117: ปัญหา N+1 Query และเกริ่น `select_related()`/`prefetch_related()`
- ขั้นตอนที่ 118: Self-Referential ForeignKey — `Comment` ที่ตอบกลับกันเองได้
- ขั้นตอนที่ 119: ออกแบบ Schema บล็อกให้สมบูรณ์ พร้อม ER Diagram
- ขั้นตอนที่ 120: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 111: `ForeignKey` เจาะลึก และตัวเลือก `on_delete` ทั้งหมด

### 111.1 ทบทวนสถานะโปรเจกต์ก่อนเริ่ม Part นี้

ก่อนเริ่ม Part นี้ โปรเจกต์ของคุณควรมีแอป `blog` ที่มี `Post` model สมบูรณ์แบบนี้ (สร้างไว้
ตั้งแต่ Part 005-007 และปรับปรุง field ต่าง ๆ อย่างเป็นระบบใน Part 011):

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

จุดสังเกตสำคัญคือ **`Post` ยังเป็นโมเดลเดี่ยว ๆ ที่ไม่เชื่อมกับตารางอื่นเลย** ในโลกจริง
บล็อกแทบทุกระบบต้องมี "หมวดหมู่" (แต่ละบทความอยู่หมวดเดียว), "แท็ก" (แต่ละบทความมีได้
หลายแท็ก), "คอมเมนต์" (แต่ละบทความมีคอมเมนต์ได้หลายอัน), และ "ผู้เขียน" ที่มีโปรไฟล์แยก
ต่างหากจากบัญชีผู้ใช้พื้นฐาน — นี่คือสิ่งที่เราจะสร้างทั้งหมดใน Part นี้ โดยเริ่มจาก
ความสัมพันธ์ที่พบบ่อยที่สุด: **ForeignKey**

### 111.2 ความสัมพันธ์แบบ One-to-Many คืออะไร

`ForeignKey` แทนความสัมพันธ์แบบ **One-to-Many (หนึ่งต่อกลาย)**: หนึ่งแถวใน "ตารางแม่"
(parent) เชื่อมโยงได้กับหลายแถวใน "ตารางลูก" (child) แต่แต่ละแถวในตารางลูกเชื่อมกับแถว
แม่ได้เพียงแถวเดียวเท่านั้น

```
Category "เทคโนโลยี"  (1 หมวดหมู่)
        │
        ├──> Post "แนะนำ Django 5"
        ├──> Post "รีวิว PostgreSQL 16"
        └──> Post "เริ่มต้น Docker"
             (หลายบทความ แต่ละบทความอยู่ในหมวดหมู่นี้ได้แค่หมวดเดียว)
```

field ที่ประกาศเป็น `ForeignKey` จะถูกวางไว้ที่ตารางลูก (`Post`) เสมอ เพราะฝั่งลูกคือฝั่งที่
"อ้างอิงกลับไปหาแม่" ในทางฐานข้อมูล `ForeignKey` จะถูกสร้างเป็นคอลัมน์ที่เก็บ **primary
key ของแถวแม่** (เรียกว่า foreign key column)

### 111.3 สร้าง `Category` model

เพิ่มโมเดล `Category` ไว้เหนือ `Post` ใน `blog/models.py`:

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True, blank=True)
    description = models.TextField(blank=True)

    class Meta:
        ordering = ['name']
        verbose_name_plural = 'categories'

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

> **หมายเหตุ**: `verbose_name_plural = 'categories'` แก้ปัญหาเล็ก ๆ ที่ Django Admin
> (Part 017) จะเจอ: โดยค่าเริ่มต้น Django เติม `s` ท้ายชื่อ model ให้เป็นพหูพจน์เสมอ
> (`Categorys` ซึ่งผิดหลักไวยากรณ์อังกฤษ) เราจึงกำหนดเองให้ถูกต้อง

### 111.4 เพิ่ม field `category` ให้ `Post`

```python
# blog/models.py
class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

พารามิเตอร์แรกของ `ForeignKey` คือ **โมเดลปลายทาง** (`Category`) ซึ่งเป็นพารามิเตอร์บังคับ
ตัวเดียวที่ Django ไม่มีค่า default ให้ ส่วนพารามิเตอร์ที่สอง **`on_delete` ก็บังคับเช่นกัน**
ตั้งแต่ Django 1.9 เป็นต้นมา (ก่อนหน้านั้นมีค่า default เป็น `CASCADE` แบบเงียบ ๆ ซึ่ง
Django มองว่าอันตรายเกินไปที่จะปล่อยให้เป็นค่าเริ่มต้นอีกต่อไป) — **Django บังคับให้คุณ
ต้องตัดสินใจอย่างชัดเจนเสมอว่า "ถ้า `Category` แถวนี้ถูกลบ จะเกิดอะไรขึ้นกับ `Post` ที่อ้างอิง
ถึงมัน"**

### 111.5 ตาราง `on_delete` ทั้งหมดใน Django

Django มีตัวเลือกให้ 7 แบบ อยู่ใน `django.db.models`:

| ตัวเลือก | พฤติกรรมเมื่อแถวแม่ถูกลบ | ต้องมี `null=True` ไหม |
|---|---|---|
| `CASCADE` | ลบแถวลูกทั้งหมดที่อ้างอิงถึงแถวแม่นั้นตามไปด้วย (ลบเป็นทอด ๆ) | ไม่ต้อง |
| `PROTECT` | **ห้ามลบ** แถวแม่ทันที โดย raise `ProtectedError` ถ้ายังมีแถวลูกอ้างอิงอยู่ | ไม่ต้อง |
| `RESTRICT` | คล้าย `PROTECT` (raise `RestrictedError`) แต่ยอมให้ลบได้ถ้าการลบนั้นเป็นส่วนหนึ่งของ CASCADE ที่มาจากจุดอื่นในห่วงโซ่เดียวกัน (Django 3.1+) | ไม่ต้อง |
| `SET_NULL` | ตั้งค่า foreign key column ของแถวลูกเป็น `NULL` | **ต้องมี** `null=True` |
| `SET_DEFAULT` | ตั้งค่า foreign key column ของแถวลูกเป็นค่าที่ระบุใน `default=` ของ field นั้น | ต้องมี `default=` |
| `SET(value)` | ตั้งค่าเป็น `value` ที่ระบุ หรือเรียก callable ที่ส่งเข้ามาแล้วใช้ผลลัพธ์นั้น (ยืดหยุ่นกว่า `SET_DEFAULT`) | แล้วแต่ค่าที่กำหนด |
| `DO_NOTHING` | ไม่ทำอะไรเลยในระดับ Django — ปล่อยให้ฐานข้อมูลตัดสินใจเอง (มักทำให้เกิด `IntegrityError` ถ้า DB มี FK constraint) | ไม่ต้อง (แต่เสี่ยงมาก) |

### 111.6 เมื่อไหร่ควรใช้ตัวเลือกไหน — พร้อมตัวอย่างโค้ด

**`CASCADE`** — ใช้เมื่อแถวลูก "ไม่มีความหมายอะไรเลย" ถ้าไม่มีแถวแม่ เช่น คอมเมนต์ที่ไม่มี
บทความให้สังกัด ไม่มีประโยชน์ที่จะเก็บไว้:

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    # ลบ Post -> คอมเมนต์ทั้งหมดของ Post นั้นถูกลบตามไปด้วยอัตโนมัติ
```

**`PROTECT`** — ใช้เมื่อการลบแถวแม่ที่ยังมีข้อมูลอ้างอิงอยู่คือ "ความผิดพลาดทางธุรกิจ" ที่ควร
บังคับให้ผู้ใช้จัดการข้อมูลลูกก่อน เช่น ระบบขายของที่ไม่ควรลบ "สินค้า" ที่ถูกใช้ในใบสั่งซื้อ
ที่จ่ายเงินแล้ว:

```python
class OrderItem(models.Model):
    product = models.ForeignKey('shop.Product', on_delete=models.PROTECT)
    # พยายามลบ Product ที่มีอยู่ใน OrderItem -> raise ProtectedError ทันที
    # ผู้ดูแลระบบต้องลบ/ย้าย OrderItem ก่อน ถึงจะลบ Product ได้
```

**`SET_NULL`** — ใช้เมื่อแถวลูกยังมีความหมายและควรถูกเก็บไว้ แม้แถวแม่จะหายไป นี่คือเหตุผล
ที่เราเลือกใช้กับ `Post.category`: **การลบหมวดหมู่ไม่ควรทำให้บทความหายไปด้วย** บทความควร
กลายเป็น "ไม่มีหมวดหมู่" แทน:

```python
category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, blank=True)
# ลบ Category -> Post.category ของบทความที่เคยอยู่หมวดนั้นกลายเป็น None
```

**`SET_DEFAULT`** — ใช้เมื่อมีค่า "ค่าเริ่มต้นที่ปลอดภัย" ให้ตกกลับไปเสมอ เช่น หมวดหมู่
"ทั่วไป" ที่มีอยู่ถาวรในระบบ:

```python
class Post(models.Model):
    category = models.ForeignKey(
        Category, on_delete=models.SET_DEFAULT, default=1,  # pk=1 คือ Category "ทั่วไป"
    )
    # ข้อเสีย: ต้องรู้ pk ของแถว default ล่วงหน้าและตายตัว — เปราะบางถ้า pk เปลี่ยน
```

**`SET(value)`** — แก้ข้อเสียของ `SET_DEFAULT` ด้วยการรับ **callable** แทนค่าตายตัว ทำให้
หาค่า default แบบไดนามิกได้ (เช่น "หาหรือสร้าง" หมวดหมู่ทั่วไปทุกครั้ง):

```python
from django.db.models import SET


def get_default_category():
    category, _created = Category.objects.get_or_create(name='ทั่วไป')
    return category.pk


class Post(models.Model):
    category = models.ForeignKey(Category, on_delete=SET(get_default_category), null=True)
```

**`RESTRICT`** — เพิ่มเข้ามาใน Django 3.1 เพื่อให้ตรงกับมาตรฐาน SQL `ON DELETE RESTRICT`
พฤติกรรมพื้นฐานเหมือน `PROTECT` (ห้ามลบถ้ายังมีลูกอ้างอิง) แต่ต่างกันตรงกรณีที่ซับซ้อน:
ถ้าการลบแถวแม่เกิดจาก CASCADE ที่ไล่มาจากแถวอื่นในสายเดียวกัน `RESTRICT` จะยอมให้ผ่านได้
ในขณะที่ `PROTECT` จะบล็อกเสมอไม่ว่ากรณีใด ในงานส่วนใหญ่เลือกใช้ `PROTECT` ก็เพียงพอแล้ว
`RESTRICT` มีประโยชน์เฉพาะ schema ที่มีลำดับชั้นการ cascade ซับซ้อนหลายระดับ

**`DO_NOTHING`** — **ไม่แนะนำให้ใช้ในโปรเจกต์ทั่วไป** เพราะ Django จะไม่จัดการอะไรให้เลย
ถ้าฐานข้อมูลมี FK constraint จริง (ซึ่ง Django สร้างให้โดยอัตโนมัติ) การลบแถวแม่ที่ยังมีลูก
อ้างอิงจะทำให้เกิด `IntegrityError` ระดับฐานข้อมูลทันที ตัวเลือกนี้มีไว้สำหรับกรณีพิเศษที่คุณ
จัดการ constraint เองด้วย database trigger หรือ raw SQL เท่านั้น

> **กฎของหลักสูตรนี้**: เริ่มต้นด้วย `PROTECT` สำหรับข้อมูลสำคัญที่ไม่ควรหายไปเงียบ ๆ,
> ใช้ `SET_NULL` เมื่อความสัมพันธ์เป็น "แบบเสริม" (optional) อย่าง `Post.category`, และใช้
> `CASCADE` เฉพาะเมื่อแถวลูก **ไม่มีความหมายอะไรเลย** โดยตัวมันเองถ้าไม่มีแถวแม่ หลีกเลี่ยง
> `DO_NOTHING` เว้นแต่จะมีเหตุผลระดับฐานข้อมูลที่ชัดเจนมาก

### 111.7 รัน Migration และตรวจสอบคอลัมน์ที่ Django สร้างให้

```bash
python manage.py makemigrations blog
python manage.py migrate
```

ผลลัพธ์ที่ควรเห็น:

```
Migrations for 'blog':
  blog/migrations/000X_category_post_category.py
    - Create model Category
    - Add field category to post
```

สังเกตว่าถึงแม้เราตั้งชื่อ field ว่า `category` แต่ Django จะสร้างคอลัมน์จริงในฐานข้อมูลชื่อ
**`category_id`** เสมอ (เติม `_id` ต่อท้ายชื่อ field อัตโนมัติ) ตรวจสอบได้ด้วย:

```bash
python manage.py dbshell
```

```sql
.schema blog_post
-- จะเห็นคอลัมน์ "category_id" integer REFERENCES "blog_category" ("id")
```

หรือดูผ่าน SQL ที่ Django จะรันจริงโดยไม่ต้อง migrate:

```bash
python manage.py sqlmigrate blog 000X
```

### 111.8 พารามิเตอร์อื่น ๆ ของ `ForeignKey` ที่ควรรู้จักไว้ตั้งแต่ตอนนี้

| พารามิเตอร์ | ความหมาย | ค่า default |
|---|---|---|
| `related_name` | ชื่อที่ใช้ query ย้อนกลับจากโมเดลปลายทาง (เจาะลึกในขั้นตอนที่ 115) | `<model>_set` |
| `related_query_name` | ชื่อที่ใช้ตอน `filter()` ข้าม relation (ต่างจาก `related_name`) | ใช้ค่าเดียวกับ `related_name` |
| `limit_choices_to` | จำกัดตัวเลือกที่แสดงใน form/admin เช่น `{'is_active': True}` | ไม่จำกัด |
| `db_index` | สร้าง index ให้คอลัมน์นี้หรือไม่ | `True` (Django สร้างให้อัตโนมัติเสมอสำหรับ FK) |
| `to_field` | ใช้ field อื่นที่ไม่ใช่ primary key ของโมเดลปลายทางเป็นตัวอ้างอิง | primary key |
| `verbose_name` | ชื่อที่แสดงใน Django Admin/forms | ชื่อ field ที่แปลงจาก snake_case |

เราจะกลับมาใช้ `related_name` อย่างจริงจังในขั้นตอนที่ 115 ตอนนี้ขอให้จำไว้ก่อนว่ามันมีอยู่

---

## ขั้นตอนที่ 112: `OneToOneField` — ขยาย Django `User` ด้วย `Profile`

### 112.1 ทำไมไม่ควรแก้ไข `User` model ของ Django โดยตรง

Django มาพร้อม model ผู้ใช้สำเร็จรูปคือ `django.contrib.auth.models.User` ซึ่งมี field
พื้นฐานอย่าง `username`, `email`, `password`, `first_name`, `last_name`, `is_staff`,
`is_superuser` ฯลฯ ให้ใช้งานได้ทันที (เราจะเรียนระบบ Authentication เต็มรูปแบบใน Part 031)

หลายคนอยากเพิ่ม field อย่าง "รูปโปรไฟล์" หรือ "ประวัติย่อ" เข้าไปใน `User` แต่ **ไม่สามารถ
แก้ไข source code ของ Django โดยตรงได้** (เป็น third-party package ที่ทีมงาน Django
ดูแล ไม่ใช่ไฟล์ในโปรเจกต์ของคุณ) วิธีที่ถูกต้องในการ "ขยาย" ผู้ใช้โดยไม่แตะต้อง `User` เดิม
คือการสร้างโมเดลใหม่ที่เชื่อมกับ `User` แบบ **หนึ่งต่อหนึ่ง (One-to-One)** ผ่าน
`OneToOneField`

> **มองไปข้างหน้า**: วิธีนี้เหมาะกับการ "เพิ่มข้อมูล" ให้ผู้ใช้ แต่ถ้าคุณต้องการเปลี่ยน
> field หลักของระบบ auth เอง (เช่น ใช้ email แทน username เป็น field login) วิธีที่ถูกต้อง
> คือการสร้าง **Custom User Model** ตั้งแต่ต้นโปรเจกต์ ซึ่งเป็นหัวข้อเต็มรูปแบบใน
> **Part 032** — สิ่งสำคัญที่ต้องรู้ตอนนี้คือ Custom User Model ต้องตั้งค่าไว้**ก่อน**รัน
> `migrate` ครั้งแรกของโปรเจกต์เท่านั้น เปลี่ยนทีหลังยากมาก ในขณะที่ `Profile` แบบ
> `OneToOneField` ทำได้ทุกเมื่อโดยไม่กระทบระบบเดิม

### 112.2 `OneToOneField` คืออะไร ต่างจาก `ForeignKey` อย่างไร

`OneToOneField` คือ `ForeignKey` ที่มี **`unique=True` บังคับติดตัวมาโดยอัตโนมัติ** ทำให้
แถวแม่หนึ่งแถวจับคู่กับแถวลูกได้ **แค่หนึ่งแถวเท่านั้น** (ไม่ใช่หลายแถวเหมือน `ForeignKey`
ปกติ):

```
ForeignKey:      Category (1) ────< Post (many)   -- หนึ่งหมวดหมู่มีได้หลายบทความ
OneToOneField:   User (1) ─────── Profile (1)      -- หนึ่งผู้ใช้มีโปรไฟล์ได้แค่อันเดียว
```

### 112.3 สร้างแอป `accounts` และ `Profile` model

เนื่องจาก `Profile` เกี่ยวข้องกับระบบผู้ใช้ทั้งโปรเจกต์ ไม่ใช่แค่แอป `blog` เราจึงสร้างแอป
ใหม่แยกต่างหากตามหลักการจัดระเบียบโค้ดที่เรียนไว้ใน Part 005:

```bash
python manage.py startapp accounts
```

เพิ่มเข้า `INSTALLED_APPS`:

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'blog',
    'accounts',
]
```

สร้าง `Profile` model:

```python
# accounts/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    bio = models.TextField(max_length=500, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f'โปรไฟล์ของ {self.user.username}'
```

> **จุดสำคัญมาก**: สังเกตว่าเราอ้างอิงผู้ใช้ด้วย **`settings.AUTH_USER_MODEL`** (string
> `'auth.User'` โดย default) แทนที่จะ `from django.contrib.auth.models import User` แล้ว
> ใช้ `User` ตรง ๆ นี่คือ **best practice ที่ Django แนะนำอย่างเป็นทางการ** เพราะถ้าวันหนึ่ง
> โปรเจกต์นี้เปลี่ยนไปใช้ Custom User Model (Part 032) โค้ดส่วนนี้จะยังทำงานถูกต้องทันที
> โดยไม่ต้องแก้ไขอะไรเลย ในทางกลับกัน ถ้า import `User` ตรง ๆ ทุกจุดที่อ้างอิงจะต้องถูกไล่
> แก้ทีละที่เมื่อเปลี่ยน User model — **กฎของหลักสูตรนี้: ห้าม import `User` ตรง ๆ ในโค้ด
> ของแอปที่คุณเขียนเอง ให้ใช้ `settings.AUTH_USER_MODEL` เสมอเมื่ออยู่ใน `models.py`**
> (ส่วนในโค้ดที่ไม่ใช่ `models.py` เช่น views จะใช้ `get_user_model()` แทน ซึ่งเรียนใน
> Part 031)

`avatar = models.ImageField(...)` ต้องการ library **Pillow** และการตั้งค่า `MEDIA_URL`/
`MEDIA_ROOT` ที่เรียนไปแล้วใน Part 009:

```bash
pip install Pillow
```

รัน migration:

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

### 112.4 เข้าถึง `Profile` จาก instance ของ `User`: `user.profile`

เพราะเราตั้ง `related_name='profile'` ไว้ เราจึงเข้าถึงโปรไฟล์จาก user object ได้โดยตรง
ราวกับมันเป็น attribute ปกติ (ไม่ใช่ manager แบบ `ForeignKey` ที่ต้องเรียก `.all()`):

```bash
python manage.py shell
```

```python
>>> from django.contrib.auth.models import User
>>> from accounts.models import Profile
>>> user = User.objects.create_user(username='sireeporn', password='securepass123')
>>> profile = Profile.objects.create(user=user, bio='นักเขียนบล็อกสาย Django')
>>> user.profile
<Profile: โปรไฟล์ของ sireeporn>
>>> user.profile.bio
'นักเขียนบล็อกสาย Django'
>>> profile.user.username
'sireeporn'
```

สังเกตว่าทั้งสองทิศทางใช้งานได้: `user.profile` (reverse, จาก `related_name`) และ
`profile.user` (forward, จาก field `user` ตรง ๆ) — และทั้งคู่คืน **object เดียว** ไม่ใช่
`QuerySet` เหมือน `ForeignKey` แบบปกติ นี่คือความแตกต่างสำคัญที่สุดในทางปฏิบัติของ
`OneToOneField`

### 112.5 ปัญหาที่ต้องระวัง: `User` ที่ยังไม่มี `Profile`

```python
>>> user2 = User.objects.create_user(username='anon', password='pass123456')
>>> user2.profile
Traceback (most recent call last):
    ...
accounts.models.RelatedObjectDoesNotExist: User has no profile.
```

เพราะเราสร้าง `Profile` แยกต่างหากด้วยมือ ถ้าลืมสร้างให้ user ใหม่ การเข้าถึง
`user.profile` จะ raise exception ทันที ในงานจริงเราไม่อยากให้เกิดเหตุการณ์นี้เลย — วิธี
แก้ที่เป็นมาตรฐานอุตสาหกรรมคือใช้ **Django Signals** (`post_save` บน `User`) เพื่อสร้าง
`Profile` ให้อัตโนมัติทุกครั้งที่มี user ใหม่ถูกสร้าง ซึ่งเป็นหัวข้อเต็มรูปแบบของ
**Part 019: Signals และ Django Lifecycle Hooks** ตอนนี้ให้จำไว้ก่อนว่าปัญหานี้มีอยู่จริง
และจะมีทางแก้ที่สวยงามรออยู่ข้างหน้า

### 112.6 ตารางเปรียบเทียบ `ForeignKey` กับ `OneToOneField`

| ประเด็น | `ForeignKey` | `OneToOneField` |
|---|---|---|
| Cardinality | หนึ่งต่อกลาย (1:N) | หนึ่งต่อหนึ่ง (1:1) |
| Unique constraint บนคอลัมน์ | ❌ ไม่มีโดยอัตโนมัติ | ✅ มีเสมอ (บังคับ) |
| ผลลัพธ์จาก reverse accessor | `RelatedManager` (ต้อง `.all()`) | Object เดียวโดยตรง |
| ตัวอย่างการใช้งาน | `Post.category`, `Comment.post` | `Profile.user`, `Order.invoice` |
| เทียบเท่ากับ | - | `ForeignKey(..., unique=True)` (แบบเก่าก่อนมี `OneToOneField`) |

### 112.7 เมื่อไหร่ควรใช้ `Profile` (OneToOne) เมื่อไหร่ควรทำ Custom User Model

| สถานการณ์ | ทางเลือกที่แนะนำ |
|---|---|
| ต้องการเพิ่มข้อมูล เช่น bio, avatar, เบอร์โทร | `Profile` ผ่าน `OneToOneField` (ทำได้ทุกเมื่อ) |
| ต้องการเปลี่ยน field ที่ใช้ login (เช่น email แทน username) | Custom User Model (ต้องตั้งก่อน migrate ครั้งแรก — Part 032) |
| โปรเจกต์เริ่มต้นใหม่ทั้งหมด | แนะนำตั้ง Custom User Model ตั้งแต่ day 1 เผื่ออนาคต แม้ยังไม่ต้องใช้ทันที |
| โปรเจกต์มีอยู่แล้ว migrate ไปมากแล้ว | ใช้ `Profile` เท่านั้น (เปลี่ยน User model ตอนนี้เสี่ยงข้อมูลเสียหายสูงมาก) |

---

## ขั้นตอนที่ 113: `ManyToManyField` — ระบบ `Tag` ของบทความ

### 113.1 ความสัมพันธ์แบบ Many-to-Many คืออะไร

`ManyToManyField` แทนความสัมพันธ์ **หลายต่อหลาย (Many-to-Many)**: หนึ่งบทความมีได้
หลายแท็ก และหนึ่งแท็กก็ถูกใช้ในหลายบทความได้เช่นกัน — ไม่มีฝั่งไหนถูกจำกัดเป็น "หนึ่ง" เลย:

```
Post "แนะนำ Django 5" ──┬──> Tag "python"
                         ├──> Tag "django"
                         └──> Tag "web-development"

Tag "python" ──┬──> Post "แนะนำ Django 5"
               ├──> Post "รีวิว FastAPI"
               └──> Post "เริ่มต้น Machine Learning"
```

ความสัมพันธ์แบบนี้ **ไม่สามารถเก็บด้วยคอลัมน์เดียวในตารางใดตารางหนึ่งได้** (ไม่เหมือน
`ForeignKey`) เพราะแต่ละฝั่งมีได้หลายค่า จึงต้องใช้ **ตารางกลาง (junction table /
through table)** ที่เก็บคู่ `(post_id, tag_id)` ทุกคู่ที่เกิดขึ้นจริง

### 113.2 สร้าง `Tag` model

```python
# blog/models.py
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=60, unique=True, blank=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

### 113.3 เพิ่ม field `tags` ให้ `Post`

```python
# blog/models.py
class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
    )
    tags = models.ManyToManyField(Tag, related_name='posts', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

สังเกตว่า `ManyToManyField` **ไม่มี `on_delete`** เลย เพราะ `on_delete` ควบคุมสิ่งที่
เกิดขึ้นกับ "แถวลูก" เมื่อ "แถวแม่" หาย แต่ในความสัมพันธ์ M2M ไม่มีใครเป็นแม่หรือลูกจริง ๆ
— เมื่อลบ `Tag` หรือ `Post` ตัวใดตัวหนึ่ง Django จะลบแค่ **แถวในตารางกลาง** ที่เกี่ยวข้อง
ออกไปเท่านั้น (พฤติกรรมนี้เทียบเท่ากับ `CASCADE` แต่เกิดที่ตารางกลาง ไม่ใช่ที่ `Post` หรือ
`Tag` โดยตรง) `blank=True` หมายถึงฟอร์ม (Part 025) ไม่บังคับให้เลือกอย่างน้อย 1 แท็ก

### 113.4 Junction Table ที่ Django สร้างให้อัตโนมัติ

```bash
python manage.py makemigrations blog
python manage.py migrate
```

Django จะสร้างตารางกลางชื่อ **`blog_post_tags`** ให้อัตโนมัติ ตรวจสอบ SQL จริงได้ด้วย:

```bash
python manage.py sqlmigrate blog 000X
```

```sql
CREATE TABLE "blog_post_tags" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "post_id" integer NOT NULL REFERENCES "blog_post" ("id"),
    "tag_id" integer NOT NULL REFERENCES "blog_tag" ("id")
);
CREATE UNIQUE INDEX ... ON "blog_post_tags" ("post_id", "tag_id");
```

ตารางนี้มีแค่ 3 คอลัมน์: `id` (primary key ของตัวมันเอง), `post_id`, และ `tag_id` พร้อม
unique constraint คู่ `(post_id, tag_id)` เพื่อป้องกันไม่ให้บทความเดียวกันถูกผูกกับแท็ก
เดียวกันซ้ำสองครั้ง **นี่คือสิ่งที่ Django สร้างให้ "ฟรี" โดยไม่ต้องเขียน model เพิ่มเอง** —
เราจะเห็นในขั้นตอนที่ 114 ว่าจะเกิดอะไรขึ้นถ้าอยากเพิ่มข้อมูลเข้าไปในตารางกลางนี้เอง

### 113.5 จัดการความสัมพันธ์ M2M ผ่าน Manager: `add()`, `remove()`, `set()`, `clear()`

```bash
python manage.py shell
```

```python
>>> from blog.models import Post, Tag
>>> post = Post.objects.create(title='แนะนำ Django 5', content='...', is_published=True)
>>> python_tag = Tag.objects.create(name='python')
>>> django_tag = Tag.objects.create(name='django')
>>> web_tag = Tag.objects.create(name='web-development')

# เพิ่มแท็กเข้า post (รับได้หลายตัวในครั้งเดียว)
>>> post.tags.add(python_tag, django_tag)
>>> post.tags.all()
<QuerySet [<Tag: django>, <Tag: python>]>

# เพิ่มอีกตัว
>>> post.tags.add(web_tag)
>>> post.tags.count()
3

# ลบออกเฉพาะตัว
>>> post.tags.remove(web_tag)
>>> post.tags.count()
2

# แทนที่ทั้งชุดด้วย set() (ลบของเก่าทั้งหมด แล้วใส่ชุดใหม่)
>>> post.tags.set([python_tag])
>>> post.tags.all()
<QuerySet [<Tag: python>]>

# ลบทั้งหมด
>>> post.tags.clear()
>>> post.tags.count()
0

# มองจากฝั่ง Tag กลับมาที่ Post (reverse, จาก related_name='posts')
>>> post.tags.add(python_tag, django_tag)
>>> python_tag.posts.all()
<QuerySet [<Post: แนะนำ Django 5>]>
```

### 113.6 ข้อจำกัดสำคัญ: ต้อง `save()` ก่อนถึงจะใช้ M2M ได้

```python
>>> new_post = Post(title='บทความใหม่', content='...')
>>> new_post.tags.add(python_tag)
Traceback (most recent call last):
    ...
ValueError: "<Post: บทความใหม่>" needs to have a value for field "id" before this
many-to-many relationship can be used.
```

เพราะ M2M ต้องเขียนแถวลงตารางกลางที่อ้างอิง `post_id` — ถ้า `Post` ยังไม่มี `id` (ยังไม่
`save()`) ก็ไม่มีค่าให้เขียน ต้อง `.save()` ตัว `Post` ก่อนเสมอ ก่อนเรียก `.tags.add(...)`

### 113.7 ตารางสรุปเปรียบเทียบทั้งสามความสัมพันธ์

| ประเด็น | `ForeignKey` | `OneToOneField` | `ManyToManyField` |
|---|---|---|---|
| Cardinality | 1 : N | 1 : 1 | N : M |
| ต้องมี `on_delete` | ✅ บังคับ | ✅ บังคับ | ❌ ไม่มีพารามิเตอร์นี้ |
| คอลัมน์ที่เกิดขึ้นจริง | คอลัมน์เดียวใน child table | คอลัมน์เดียว + unique constraint | สร้างตารางกลางแยกต่างหาก |
| Reverse accessor | `RelatedManager` (`.all()`) | Object เดี่ยว | `RelatedManager` (`.all()`) |
| ตัวอย่างในโปรเจกต์นี้ | `Post.category` | `Profile.user` | `Post.tags` |

---

## ขั้นตอนที่ 114: ManyToMany ผ่าน `through` Model แบบกำหนดเอง

### 114.1 ทำไมบางครั้งตารางกลางอัตโนมัติไม่พอ

ตารางกลางที่ Django สร้างให้ใน `blog_post_tags` มีแค่ `post_id` กับ `tag_id` เท่านั้น
ถ้าธุรกิจต้องการรู้ **ข้อมูลเพิ่มเติมเกี่ยวกับความสัมพันธ์นั้นเอง** เช่น "แท็กนี้ถูกเพิ่มเข้า
บทความนี้เมื่อไหร่" หรือ "ใครเป็นคนเพิ่ม" ตารางกลางอัตโนมัติจะไม่มีที่เก็บข้อมูลเหล่านี้เลย
— นี่คือจุดที่ต้องใช้ **custom `through` model**

### 114.2 สร้าง Through Model `PostTag`

```python
# blog/models.py
class PostTag(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    tag = models.ForeignKey(Tag, on_delete=models.CASCADE)
    added_at = models.DateTimeField(auto_now_add=True)
    added_by = models.CharField(max_length=100, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=['post', 'tag'], name='unique_post_tag'),
        ]
        ordering = ['-added_at']

    def __str__(self):
        return f'{self.post.title} — {self.tag.name}'
```

สังเกตว่า `PostTag` คือ **model ธรรมดา** ที่มี `ForeignKey` สองตัวชี้ไปที่ `Post` และ
`Tag` เท่านั้นเอง ไม่มีอะไรพิเศษ — ความพิเศษอยู่ที่ขั้นตอนถัดไป

> **หมายเหตุ**: `models.UniqueConstraint` เป็นวิธีสมัยใหม่ (Django 2.2+) ในการกำหนด
> unique constraint หลายคอลัมน์พร้อมกัน แทนที่ `unique_together` แบบเก่าที่ยังใช้งานได้
> แต่ Django แนะนำให้ค่อย ๆ เปลี่ยนมาใช้ `UniqueConstraint` เพราะยืดหยุ่นกว่า (ตั้งชื่อเอง,
> ใส่เงื่อนไข `condition=` ได้) — เราจะเจาะลึก `Meta` options ทั้งหมดใน Part 015

### 114.3 ปรับ `Post.tags` ให้ใช้ `through='PostTag'`

```python
# blog/models.py
class Post(models.Model):
    # ... field อื่น ๆ เหมือนเดิม ...
    tags = models.ManyToManyField(Tag, through='PostTag', related_name='posts', blank=True)
```

เราอ้างอิง `PostTag` เป็น **string** (`'PostTag'`) แทนที่จะ import class ตรง ๆ เพราะ
`PostTag` ต้องถูกประกาศ**หลัง** `Post` และ `Tag` (เนื่องจาก `PostTag` มี `ForeignKey`
ชี้ไปหาทั้งสองตัว) การอ้างอิงด้วย string ทำให้ Django รอ resolve ชื่อคลาสตอนโหลด app
registry เสร็จสมบูรณ์ก่อน จึงไม่ติดปัญหาลำดับการประกาศ

รัน migration:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

> **ข้อควรระวังสำหรับโปรเจกต์ที่มีข้อมูลอยู่แล้ว**: ถ้า `blog_post_tags` (ตารางกลาง
> อัตโนมัติเดิม) มีข้อมูลอยู่แล้วก่อนเปลี่ยนมาใช้ `through=`, Django **ไม่ได้ย้ายข้อมูลเก่า
> ให้อัตโนมัติ** เพราะโครงสร้างตารางเปลี่ยนไป (ตารางใหม่มีคอลัมน์ `added_at`/`added_by`
> เพิ่มเข้ามา) ในสถานการณ์นั้นต้องเขียน **data migration** เพื่อคัดลอกข้อมูลข้ามตารางเอง
> ซึ่งเป็นเทคนิคที่เราจะเรียนเต็มรูปแบบใน **Part 016: Database Migrations ขั้นสูง**
> สำหรับโปรเจกต์เรียนรู้ในหลักสูตรนี้ที่ยังไม่มีข้อมูลจริงในระบบ ปัญหานี้จะไม่เกิดขึ้น

### 114.4 ข้อจำกัด: `add()`/`remove()`/`set()` ใช้ตรง ๆ ไม่ได้เต็มรูปแบบอีกต่อไป

เมื่อกำหนด `through` เอง Django ไม่รู้ว่าจะเติมค่า `added_by` ให้อย่างไรถ้าคุณเรียก
`post.tags.add(tag)` ตรง ๆ (Django รุ่นเก่าจะ raise error ทันที) ตั้งแต่ **Django 2.2**
เป็นต้นมา มีทางออกให้ผ่าน **`through_defaults`**:

```python
>>> from blog.models import Post, Tag, PostTag
>>> post = Post.objects.get(slug='แนะนำ-django-5')
>>> python_tag = Tag.objects.get(name='python')

# วิธีที่ 1: ใช้ through_defaults กับ add()
>>> post.tags.add(python_tag, through_defaults={'added_by': 'admin'})

# วิธีที่ 2: สร้างแถวใน through model โดยตรง (ควบคุมได้ละเอียดที่สุด)
>>> django_tag = Tag.objects.get(name='django')
>>> PostTag.objects.create(post=post, tag=django_tag, added_by='sireeporn')
```

`remove()` และ `clear()` ยังใช้ได้ปกติ (เพราะแค่ต้องรู้ว่าจะลบแถวไหน ไม่ต้องเติมข้อมูล
เพิ่ม) แต่ `through_defaults` ใช้ได้เฉพาะกับ `add()` และ `set()` เท่านั้น

### 114.5 Query ผ่าน Through Model โดยตรง

เพราะ `PostTag` เป็น model ปกติ เราจึง query มันได้เหมือน model ทั่วไปทุกประการ ซึ่งทำ
ไม่ได้เลยกับตารางกลางอัตโนมัติ:

```python
>>> PostTag.objects.filter(added_by='admin')
<QuerySet [<PostTag: แนะนำ Django 5 — python>]>

>>> PostTag.objects.filter(post=post).order_by('-added_at')
<QuerySet [<PostTag: แนะนำ Django 5 — django>, <PostTag: แนะนำ Django 5 — python>]>

# หาว่าแท็กไหนถูกเพิ่มเข้าระบบเร็วที่สุดในบทความนี้
>>> PostTag.objects.filter(post=post).earliest('added_at')
<PostTag: แนะนำ Django 5 — python>
```

### 114.6 ตารางสรุป: M2M อัตโนมัติ vs M2M ผ่าน `through` กำหนดเอง

| ประเด็น | M2M อัตโนมัติ | M2M ผ่าน `through` เอง |
|---|---|---|
| เก็บข้อมูลเพิ่มเติมของความสัมพันธ์ | ❌ ทำไม่ได้ | ✅ ทำได้เต็มที่ |
| `add()`/`set()` ใช้ตรง ๆ ได้ | ✅ ใช้ได้เต็มรูปแบบ | ⚠️ ต้องใช้ `through_defaults` หรือสร้างแถวเอง |
| Query ตารางกลางโดยตรง | ❌ ไม่มี model ให้ query | ✅ query เหมือน model ทั่วไป |
| ความซับซ้อนในการดูแล | ต่ำ | สูงกว่าเล็กน้อย |
| ใช้เมื่อ | ความสัมพันธ์ล้วน ๆ ไม่มี metadata | ต้องเก็บ metadata ของความสัมพันธ์ (วันที่, ผู้เพิ่ม, ลำดับ ฯลฯ) |

---

## ขั้นตอนที่ 115: `related_name` และ Reverse Relations

### 115.1 สร้าง `Comment` model เชื่อมกับ `Post`

```python
# blog/models.py
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.CharField(max_length=100)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']

    def __str__(self):
        return f'{self.author} แสดงความเห็นใน "{self.post.title}"'
```

เราเลือก `on_delete=models.CASCADE` สำหรับ `Comment.post` เพราะคอมเมนต์ที่ไม่มีบทความ
ให้สังกัด **ไม่มีความหมายอะไรเลย** ตามหลักการที่วางไว้ในขั้นตอนที่ 111.6 — ลบบทความ
คอมเมนต์ทั้งหมดของบทความนั้นควรถูกลบตามไปด้วย

รัน migration:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

### 115.2 `related_name` คืออะไร และทำไมสำคัญ

ทุกครั้งที่ประกาศ `ForeignKey`/`OneToOneField`/`ManyToManyField` Django จะสร้าง
**reverse accessor** ให้อัตโนมัติที่โมเดลปลายทาง เพื่อให้ query ย้อนกลับได้ ถ้า**ไม่ระบุ**
`related_name` เอง Django จะตั้งชื่อให้ตามรูปแบบ **`<ชื่อโมเดลตัวพิมพ์เล็ก>_set`**

ลองดูตัวอย่างที่เรายังไม่ได้ตั้ง `related_name` ให้ `Post.category` (จากขั้นตอนที่ 111):

```python
>>> from blog.models import Category
>>> tech = Category.objects.get(name='เทคโนโลยี')
>>> tech.post_set.all()          # ชื่อ default: <model>_set
<QuerySet [<Post: แนะนำ Django 5>, <Post: รีวิว PostgreSQL 16>]>
```

`post_set` ใช้งานได้ แต่ไม่ค่อยสื่อความหมายเท่าไหร่ เราจึงกลับไปเพิ่ม `related_name='posts'`
ให้ `Category`:

```python
# blog/models.py
class Post(models.Model):
    # ...
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='posts',
    )
```

```bash
python manage.py makemigrations blog
```

```
Migrations for 'blog':
  blog/migrations/000X_alter_post_category.py
    - Alter field category on post
```

> **หมายเหตุสำคัญ**: การเปลี่ยน `related_name` **ไม่ทำให้เกิดการเปลี่ยนแปลงใด ๆ ใน
> ฐานข้อมูลจริง** (ไม่มีคอลัมน์หรือตารางเปลี่ยน) เพราะ `related_name` เป็นแนวคิดระดับ
> Python/ORM เท่านั้น แต่ Django ยังคง **ต้องสร้าง migration ไฟล์** เพื่อบันทึกการ
> เปลี่ยนแปลงนี้ไว้ใน "migration state" (ดูได้จาก `sqlmigrate` ว่าไม่มี SQL คำสั่งใดถูกรัน
> เลย) เหตุผลคือ Django ต้องรู้ตลอดเวลาว่า field แต่ละตัวมีค่า kwargs อะไรบ้าง ณ จุดใด
> ของประวัติ migration เพื่อให้ `makemigrations` ครั้งถัดไปเปรียบเทียบได้ถูกต้อง

หลังรัน `migrate` แล้ว ตอนนี้ใช้ `tech.posts.all()` แทนได้:

```python
>>> tech.posts.all()
<QuerySet [<Post: แนะนำ Django 5>, <Post: รีวิว PostgreSQL 16>]>
```

### 115.3 ตารางสรุป Reverse Relation Naming Convention

| สถานการณ์ | ชื่อที่ใช้ query ย้อนกลับ |
|---|---|
| ไม่ตั้ง `related_name` | `<ชื่อโมเดลตัวพิมพ์เล็ก>_set` เช่น `post_set`, `comment_set` |
| ตั้ง `related_name='posts'` | `.posts` (เข้าถึงตรง ๆ โดยไม่มี `_set`) |
| `OneToOneField` (ไม่ตั้ง) | `<ชื่อโมเดลตัวพิมพ์เล็ก>` (object เดียว ไม่มี `_set` เพราะมีได้แค่ 1) |
| `OneToOneField` (ตั้ง `related_name='profile'`) | `.profile` (object เดียว) |

### 115.4 สรุป `related_name` ทั้งหมดในโปรเจกต์ ณ จุดนี้

| Field | อยู่ในโมเดล | `related_name` | ใช้ query ย้อนกลับด้วย |
|---|---|---|---|
| `Post.category` | `Post` | `'posts'` | `category.posts.all()` |
| `Post.tags` | `Post` | `'posts'` | `tag.posts.all()` |
| `Comment.post` | `Comment` | `'comments'` | `post.comments.all()` |
| `Profile.user` | `Profile` | `'profile'` | `user.profile` |

### 115.5 ใช้งาน Reverse Relations จาก `Post`

```python
>>> post = Post.objects.get(slug='แนะนำ-django-5')

# นับจำนวนคอมเมนต์
>>> post.comments.count()
3

# ดึงคอมเมนต์ทั้งหมด (เรียงตาม Meta.ordering ของ Comment คือ created_at)
>>> post.comments.all()
<QuerySet [<Comment: สมชาย แสดงความเห็นใน "แนะนำ Django 5">, ...]>

# สร้างคอมเมนต์ใหม่ผ่าน reverse manager โดยตรง (ไม่ต้องระบุ post= ซ้ำ)
>>> post.comments.create(author='วิภา', content='บทความดีมากครับ')
<Comment: วิภา แสดงความเห็นใน "แนะนำ Django 5">
```

`post.comments.create(...)` เป็นวิธีลัดที่สะดวกมาก — เทียบเท่ากับการเขียน
`Comment.objects.create(post=post, author='วิภา', content='...')` แต่สั้นกว่าและลด
โอกาสลืมใส่ `post=post`

### 115.6 `related_query_name`: ชื่อที่ใช้เฉพาะตอน `filter()` ข้าม Relation

`related_name` ควบคุมทั้งการเข้าถึง attribute (`post.comments`) และการ filter ข้าม
relation (`Post.objects.filter(comments__author=...)`) พร้อมกัน แต่ถ้าต้องการชื่อที่
**ต่างกัน** สำหรับสองกรณีนี้ ใช้ `related_query_name` แยกออกมาได้:

```python
class Comment(models.Model):
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name='comments',           # ใช้กับ post.comments.all()
        related_query_name='comment',      # ใช้กับ Post.objects.filter(comment__author=...)
    )
```

ในทางปฏิบัติส่วนใหญ่ไม่จำเป็นต้องแยก และปล่อยให้ `related_query_name` ใช้ค่าเดียวกับ
`related_name` (ค่า default) ก็เพียงพอ เราจะใช้ทั้งสองแบบร่วมกันในขั้นตอนถัดไป

---

## ขั้นตอนที่ 116: Query ข้ามความสัมพันธ์ด้วย Double Underscore

### 116.1 หลักการของ Double Underscore Lookup ข้ามความสัมพันธ์

Django ORM ให้เรา "เดินข้าม" ความสัมพันธ์ตอน `filter()`/`exclude()` ได้โดยใช้เครื่องหมาย
**`__`** (double underscore) คั่นระหว่างชื่อ field ที่เป็นความสัมพันธ์กับชื่อ field ของ
โมเดลปลายทาง:

```python
>>> from blog.models import Post

# กรองบทความที่อยู่ในหมวดหมู่ชื่อ "เทคโนโลยี"
>>> Post.objects.filter(category__name='เทคโนโลยี')

# เจาะลึกเข้าไปอีกชั้น: กรองผ่าน slug ของหมวดหมู่แทนชื่อ
>>> Post.objects.filter(category__slug='technology')
```

`category__name` แปลว่า "เดินตาม field `category` (ที่เป็น `ForeignKey`) ไปที่โมเดล
`Category` แล้วดูค่า field `name`" ในระดับ SQL Django จะแปลงสิ่งนี้เป็น `JOIN` ระหว่าง
`blog_post` กับ `blog_category` ให้อัตโนมัติทั้งหมด

### 116.2 กรองผ่านความสัมพันธ์แบบย้อนกลับ (Reverse) ด้วย `related_name`

double underscore ใช้ได้ทั้งสองทิศทาง ไม่ใช่แค่ forward (`ForeignKey` → โมเดลปลายทาง)
แต่ใช้ reverse (จากโมเดลปลายทางกลับมาหาโมเดลต้นทาง) ได้เช่นกัน โดยใช้ชื่อจาก
`related_name`:

```python
# กรองบทความที่มีคอมเมนต์จากผู้เขียนชื่อ "สมชาย"
>>> Post.objects.filter(comments__author='สมชาย')

# กรองบทความที่มีแท็กชื่อ "python" (M2M reverse ก็ใช้แบบเดียวกัน)
>>> Post.objects.filter(tags__name='python')

# กรองในทิศทางกลับ: หมวดหมู่ที่มีบทความที่เผยแพร่แล้วอย่างน้อย 1 บทความ
>>> from blog.models import Category
>>> Category.objects.filter(posts__is_published=True)

# แท็กที่ถูกใช้ในบทความที่เผยแพร่แล้วเท่านั้น
>>> from blog.models import Tag
>>> Tag.objects.filter(posts__is_published=True)
```

### 116.3 เชื่อมหลายเงื่อนไขข้ามหลายความสัมพันธ์พร้อมกัน

```python
# บทความในหมวด "เทคโนโลยี" ที่มีแท็ก "django" ด้วย
>>> Post.objects.filter(category__name='เทคโนโลยี', tags__name='django')

# บทความที่มีคอมเมนต์จาก "สมชาย" และถูกเผยแพร่แล้ว
>>> Post.objects.filter(comments__author='สมชาย', is_published=True)

# เจาะลึกได้หลายชั้น: หาแท็กที่ถูกเพิ่มโดย 'admin' (ผ่าน through model PostTag)
>>> Tag.objects.filter(posttag__added_by='admin')
```

บรรทัดสุดท้ายใช้ชื่อ `posttag` (ตัวพิมพ์เล็กของชื่อ model `PostTag`) เพราะเราไม่ได้ตั้ง
`related_name` ให้ field `tag` ใน `PostTag` model ไว้ — ถ้าต้องการชื่อที่อ่านง่ายกว่านี้
ก็เพิ่ม `related_name='tag_links'` ให้ field นั้นได้เช่นเดียวกับที่ทำไปแล้วในขั้นตอนที่ 115

### 116.4 ปัญหาแถวซ้ำ (Duplicate Rows) เมื่อ Join กับ M2M/Reverse FK

เมื่อ query ข้ามความสัมพันธ์แบบ "หนึ่งต่อกลาย" ในทิศทางที่ปลายทางมีได้หลายแถว (M2M หรือ
reverse FK) และมีเงื่อนไขที่ match ได้มากกว่าหนึ่งแถวย่อย ผลลัพธ์อาจมี **object ซ้ำกัน**
เพราะ SQL `JOIN` สร้างแถวใหม่ทุกครั้งที่ match:

```python
# บทความที่มีแท็ก "python" หรือ "django" — อาจได้ Post ซ้ำถ้าบทความนั้นมีทั้งสองแท็ก!
>>> qs = Post.objects.filter(tags__name__in=['python', 'django'])
>>> qs.count()
5   # แต่จริง ๆ อาจมีบทความไม่ซ้ำแค่ 3 บทความ
```

วิธีแก้คือเรียก **`.distinct()`** ต่อท้าย:

```python
>>> qs = Post.objects.filter(tags__name__in=['python', 'django']).distinct()
>>> qs.count()
3   # ตอนนี้ไม่มีบทความซ้ำแล้ว
```

> **กฎของหลักสูตรนี้**: เมื่อไหร่ก็ตามที่ `filter()` ข้าม relation แบบ M2M หรือ reverse FK
> ที่มีเงื่อนไขซับซ้อนกว่าตัวเดียว ให้ตรวจสอบผลลัพธ์ด้วย `.count()` เทียบกับที่คาดไว้เสมอ
> และเติม `.distinct()` เมื่อจำเป็น — นี่คือหนึ่งใน bug ที่พบบ่อยที่สุดของมือใหม่ Django ORM

### 116.5 `exclude()` ข้ามความสัมพันธ์

`exclude()` ใช้ syntax เดียวกับ `filter()` ทุกประการ แต่กลับตรรกะ:

```python
# บทความที่ไม่มีคอมเมนต์จากผู้เขียนชื่อ "ก่อกวน"
>>> Post.objects.exclude(comments__author='ก่อกวน')

# หมวดหมู่ที่ยังไม่มีบทความเลย (ไม่มี Post ใดอ้างอิงถึง)
>>> Category.objects.exclude(posts__isnull=False)
# เทียบเท่ากับ
>>> Category.objects.filter(posts__isnull=True)
```

> **ข้อควรระวัง**: `exclude()` กับความสัมพันธ์แบบหลายแถวมีพฤติกรรมที่ **อาจไม่ตรงกับที่
> คาดไว้** เช่น `Post.objects.exclude(tags__name='python')` ไม่ได้แปลว่า "บทความที่ไม่มี
> แท็ก python เลย" เสมอไปถ้ามีเงื่อนไขอื่นร่วมด้วยในหลาย `filter()`/`exclude()` ที่ต่อกัน
> เพราะแต่ละ `JOIN` อาจถูกสร้างแยกกันคนละรอบ — รายละเอียดเชิงลึกเรื่องนี้ (multi-valued
> relationship pitfalls) จะอธิบายอย่างละเอียดพร้อมตัวอย่างจริงใน **Part 013**

### 116.6 `values()` และ `values_list()` ข้ามความสัมพันธ์

double underscore ใช้ได้กับ `values()`/`values_list()` เช่นกัน สะดวกมากเมื่อต้องการ
เฉพาะบางคอลัมน์จากตารางที่ join กัน:

```python
>>> Post.objects.filter(is_published=True).values('title', 'category__name')
<QuerySet [{'title': 'แนะนำ Django 5', 'category__name': 'เทคโนโลยี'}, ...]>

>>> Post.objects.values_list('title', 'tags__name')
<QuerySet [('แนะนำ Django 5', 'python'), ('แนะนำ Django 5', 'django'), ...]>
```

---

## ขั้นตอนที่ 117: ปัญหา N+1 Query และเกริ่น `select_related()`/`prefetch_related()`

### 117.1 ปัญหา N+1 Query คืออะไร

ลองเขียน view ที่ดูเหมือนไม่มีอะไรผิดปกติ:

```python
def post_list(request):
    posts = Post.objects.filter(is_published=True)
    for post in posts:
        print(post.category.name if post.category else 'ไม่มีหมวดหมู่')
    # ...
```

โค้ดนี้ **ทำงานถูกต้อง** แต่มีปัญหาด้าน performance ที่ซ่อนอยู่: การเรียก
`Post.objects.filter(...)` คือ **1 query** เพื่อดึงบทความทั้งหมด แต่ทุกครั้งที่เข้าถึง
`post.category` ภายใน loop — เนื่องจาก `category` เป็น `ForeignKey` ที่ **lazy load**
(โหลดข้อมูลก็ต่อเมื่อถูกเรียกใช้จริง) Django จะยิง **query เพิ่มอีก 1 ครั้งต่อบทความ 1
บทความ** เพื่อไปดึงข้อมูล `Category` ที่เกี่ยวข้อง

ถ้ามีบทความ 100 บทความ นั่นหมายถึง **1 query (ดึง Post) + 100 query (ดึง Category
ทีละบทความ) = 101 queries** ทั้งที่ในทางทฤษฎีควรทำได้ด้วย query เดียว — นี่คือปัญหาที่
เรียกว่า **N+1 Query Problem** ซึ่งเป็นสาเหตุอันดับต้น ๆ ของเว็บที่ช้าในโลกจริง

ตรวจสอบจำนวน query จริงที่เกิดขึ้นได้ด้วย:

```python
from django.db import connection, reset_queries

reset_queries()
posts = list(Post.objects.filter(is_published=True))
for post in posts:
    _ = post.category.name if post.category else None
print(len(connection.queries))   # จะเห็นตัวเลขที่มากเกินคาด
```

### 117.2 `select_related()`: แก้ปัญหาสำหรับ `ForeignKey`/`OneToOneField`

`select_related()` บอกให้ Django ใช้ **SQL `JOIN`** ดึงข้อมูลของความสัมพันธ์ที่ระบุมา
พร้อมกันใน **query เดียว** ตั้งแต่ต้น:

```python
posts = Post.objects.select_related('category').filter(is_published=True)
for post in posts:
    print(post.category.name if post.category else 'ไม่มีหมวดหมู่')
    # ไม่มี query เพิ่มเติมเกิดขึ้นเลย เพราะ category ถูกดึงมาพร้อมกับ post แล้ว
```

`select_related()` ใช้ได้เฉพาะกับความสัมพันธ์ที่ **ฝั่งปัจจุบันมีแค่แถวเดียว** ต่อผลลัพธ์
(`ForeignKey` และ `OneToOneField`) เพราะ `JOIN` แบบนี้ยังคงได้ผลลัพธ์เป็น "หนึ่งแถวต่อ
หนึ่ง `Post`" เหมือนเดิม

### 117.3 `prefetch_related()`: แก้ปัญหาสำหรับ `ManyToManyField`/Reverse `ForeignKey`

สำหรับความสัมพันธ์ที่ปลายทางมีได้ **หลายแถว** (M2M หรือ reverse FK อย่าง `comments`,
`tags`) การใช้ `JOIN` แบบเดียวกับ `select_related()` จะทำให้เกิดปัญหาแถวซ้ำแบบที่เห็นใน
ขั้นตอนที่ 116.4 Django จึงใช้กลยุทธ์ต่างออกไปคือ **`prefetch_related()`**: ยิง query
แยกต่างหากอีกรอบสำหรับความสัมพันธ์นั้น แล้วนำผลลัพธ์มา "จับคู่" กันในฝั่ง Python แทน:

```python
posts = Post.objects.prefetch_related('tags', 'comments').filter(is_published=True)
for post in posts:
    print([tag.name for tag in post.tags.all()])       # ไม่ยิง query เพิ่ม
    print(post.comments.count())                        # ไม่ยิง query เพิ่ม
```

ผลลัพธ์คือ query ทั้งหมดเหลือแค่ **3 queries คงที่** (1 สำหรับ `Post`, 1 สำหรับ `tags`
ทั้งหมดที่เกี่ยวข้อง, 1 สำหรับ `comments` ทั้งหมดที่เกี่ยวข้อง) ไม่ว่าจะมีบทความกี่บทความ
ก็ตาม — ต่างจาก N+1 ที่จำนวน query โตตามจำนวนแถว

รวมทั้งสองอย่างเข้าด้วยกันได้ในคำสั่งเดียว:

```python
posts = (
    Post.objects
    .select_related('category')
    .prefetch_related('tags', 'comments')
    .filter(is_published=True)
)
```

### 117.4 ตารางเปรียบเทียบสั้น ๆ

| ประเด็น | `select_related()` | `prefetch_related()` |
|---|---|---|
| ใช้กับความสัมพันธ์ประเภท | `ForeignKey`, `OneToOneField` | `ManyToManyField`, reverse `ForeignKey` |
| กลไกเบื้องหลัง | SQL `JOIN` เดียว | Query แยก + จับคู่ใน Python |
| จำนวน query ที่ได้ | 1 query รวม | 1 query ต่อความสัมพันธ์ที่ระบุ |
| เจาะลึกเต็มรูปแบบใน | **Part 067** | **Part 067** |

### 117.5 กฎง่าย ๆ ที่จำไว้ก่อน

> **กฎเบื้องต้น (rule of thumb)**: field ที่เป็น `ForeignKey`/`OneToOneField` (เข้าถึงแล้ว
> ได้ object เดียว) → ใช้ `select_related()`; field ที่เป็น `ManyToManyField` หรือ reverse
> `ForeignKey` (เข้าถึงแล้วได้ manager ที่ต้อง `.all()`) → ใช้ `prefetch_related()`

นี่เป็นเพียงการเกริ่นให้รู้จักและเข้าใจปัญหาที่มันแก้ เพื่อให้คุณเริ่มมีสัญชาตญาณเลือกใช้ได้
ถูกต้องตั้งแต่ตอนนี้ รายละเอียดเชิงลึกทั้งหมด — `Prefetch` object แบบกำหนดเอง, การรวมกับ
`only()`/`defer()`, การวัด performance ด้วย `django-debug-toolbar`, และเทคนิคขั้นสูงอื่น ๆ
— จะอยู่ใน **Part 067: Query Optimization: select_related, prefetch_related** ซึ่งเป็นส่วน
หนึ่งของ Phase 8 (Performance & Caching)

---

## ขั้นตอนที่ 118: Self-Referential ForeignKey — `Comment` ที่ตอบกลับกันเองได้

### 118.1 แนวคิด Self-Referential Relationship

ระบบคอมเมนต์ในเว็บยุคใหม่มักรองรับ "การตอบกลับความเห็น" (nested replies/threaded
comments) ซึ่งหมายความว่า **`Comment` ต้องเชื่อมโยงกับ `Comment` ตัวมันเอง**:

```
Comment #1: "บทความดีมากครับ" (โดย สมชาย)
    └── Comment #2: "เห็นด้วยเลย" (โดย วิภา, ตอบกลับ #1)
        └── Comment #3: "ขอบคุณครับ" (โดย สมชาย, ตอบกลับ #2)
    └── Comment #4: "มีตัวอย่างเพิ่มไหมครับ" (โดย กิตติ, ตอบกลับ #1)
Comment #5: "รอตอนต่อไปครับ" (โดย นภา, ไม่ได้ตอบกลับใคร)
```

นี่คือ **ForeignKey ที่ชี้กลับไปหาโมเดลของตัวเอง** เรียกว่า **self-referential
ForeignKey**

### 118.2 เพิ่ม field `parent` ให้ `Comment`

```python
# blog/models.py
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    parent = models.ForeignKey(
        'self',
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name='replies',
    )
    author = models.CharField(max_length=100)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']

    def __str__(self):
        return f'{self.author} แสดงความเห็นใน "{self.post.title}"'
```

พารามิเตอร์แรกของ `ForeignKey` ปกติต้องเป็นชื่อโมเดล แต่ตอนนี้เราต้องการชี้กลับไปหา
`Comment` ซึ่งเป็น**คลาสที่กำลังถูกประกาศอยู่นี้เอง** — ยังเขียนโค้ดไม่เสร็จจึงไม่มีชื่อคลาส
ให้ import ใช้ตรง ๆ ได้ Django จึงมี keyword พิเศษคือ **string `'self'`** ที่แปลว่า "ชี้
กลับมาที่โมเดลนี้เอง" ใช้แทนชื่อคลาสได้ทันที

### 118.3 ทำไมต้อง `null=True`, `blank=True`, และเลือก `on_delete=CASCADE`

- **`null=True`**: คอมเมนต์ระดับบนสุด (top-level) ไม่มีคอมเมนต์แม่ จึงต้องอนุญาตให้
  `parent` เป็น `NULL` ได้ในฐานข้อมูล
- **`blank=True`**: เมื่อสร้างฟอร์มคอมเมนต์ (Part 025) ผู้ใช้ไม่จำเป็นต้องเลือก parent
  ทุกครั้ง (ค่าเริ่มต้นคือคอมเมนต์ใหม่)
- **`on_delete=models.CASCADE`**: ถ้าลบคอมเมนต์แม่ ควรลบคอมเมนต์ลูกทั้งหมดที่ตอบกลับ
  มันไปด้วย เพราะการเก็บ "คำตอบ" ไว้โดยไม่มี "คำถาม" ให้บริบทมักไม่มีประโยชน์

พิจารณาทางเลือกอื่นเทียบกัน:

| `on_delete` ที่เลือก | ผลลัพธ์เมื่อลบ Comment แม่ | เหมาะกับ |
|---|---|---|
| `CASCADE` (เลือกใช้จริง) | ลบทั้ง "ต้นไม้" ของการตอบกลับทั้งหมดที่ต่อจากมัน | บอร์ดคอมเมนต์ทั่วไปที่มองว่าคำตอบไม่มีความหมายถ้าไม่มีคำถามต้นเรื่อง |
| `SET_NULL` | คอมเมนต์ลูกกลายเป็นคอมเมนต์ระดับบนสุดแทน (parent เป็น `NULL`) | ฟอรัมที่อยากรักษาเนื้อหาคำตอบไว้ แม้คำถามต้นทางถูกลบ |
| `PROTECT` | ห้ามลบคอมเมนต์ที่มีคนตอบกลับอยู่ | ระบบที่ต้องการเก็บบทสนทนาไว้ครบถ้วนเสมอ (archival) |

### 118.4 ทดสอบผ่าน Shell: สร้างโครงสร้างคอมเมนต์แบบตอบกลับ

```bash
python manage.py makemigrations blog
python manage.py migrate
python manage.py shell
```

```python
>>> from blog.models import Post, Comment
>>> post = Post.objects.get(slug='แนะนำ-django-5')

>>> root = Comment.objects.create(post=post, author='สมชาย', content='บทความดีมากครับ')
>>> reply1 = Comment.objects.create(
...     post=post, parent=root, author='วิภา', content='เห็นด้วยเลย',
... )
>>> reply2 = Comment.objects.create(
...     post=post, parent=reply1, author='สมชาย', content='ขอบคุณครับ',
... )
>>> reply3 = Comment.objects.create(
...     post=post, parent=root, author='กิตติ', content='มีตัวอย่างเพิ่มไหมครับ',
... )

# ดูคอมเมนต์ระดับบนสุดเท่านั้น (ที่ไม่มี parent)
>>> Comment.objects.filter(post=post, parent__isnull=True)
<QuerySet [<Comment: สมชาย แสดงความเห็นใน "แนะนำ Django 5">]>

# ดูคำตอบทั้งหมดของ root (ใช้ related_name='replies')
>>> root.replies.all()
<QuerySet [<Comment: วิภา แสดงความเห็นใน "แนะนำ Django 5">, <Comment: กิตติ แสดงความเห็นใน "แนะนำ Django 5">]>

# ไล่ลึกลงไปอีกชั้น: คำตอบของคำตอบ
>>> reply1.replies.all()
<QuerySet [<Comment: สมชาย แสดงความเห็นใน "แนะนำ Django 5">]>

# เดินย้อนขึ้นไปหา parent
>>> reply2.parent.parent
<Comment: สมชาย แสดงความเห็นใน "แนะนำ Django 5">
```

### 118.5 ฟังก์ชันช่วยคำนวณความลึกของ Reply Thread

เขียน method เล็ก ๆ ใน `Comment` เพื่อไล่ย้อนกลับไปนับว่าคอมเมนต์นี้อยู่ลึกกี่ชั้น (มี
ประโยชน์เมื่อทำ UI แบบ "เยื้องข้อความตามระดับการตอบกลับ" ในภายหลัง):

```python
class Comment(models.Model):
    # ... field เดิมทั้งหมด ...

    def get_depth(self):
        """นับความลึกของคอมเมนต์นี้ในสายการตอบกลับ (root = 0)"""
        depth = 0
        current = self.parent
        while current is not None:
            depth += 1
            current = current.parent
        return depth
```

```python
>>> root.get_depth()
0
>>> reply1.get_depth()
1
>>> reply2.get_depth()
2
```

> **ข้อควรระวังเรื่อง performance**: `get_depth()` ที่เขียนไว้นี้ยิง query เพิ่มทุกครั้งที่
> เดินไปหา `.parent` หนึ่งชั้น (ปัญหาแบบเดียวกับ N+1 ในขั้นตอนที่ 117) สำหรับ thread ที่ลึก
> มาก ๆ ควรพิจารณาโครงสร้างข้อมูลแบบอื่น เช่น **Adjacency List** (แบบที่เราทำอยู่นี้ เหมาะ
> กับความลึกไม่มาก), **Materialized Path**, หรือใช้ library สำเร็จรูปอย่าง
> `django-treebeard`/`django-mptt` สำหรับ hierarchy ที่ลึกและซับซ้อนมาก ๆ ในระดับ
> production เราจะไม่เจาะลึกเรื่องนี้ในหลักสูตรนี้ แต่ควรรู้ไว้ว่ามีทางเลือกเหล่านี้อยู่

### 118.6 ตารางสรุป Self-Referential ForeignKey

| ประเด็น | รายละเอียด |
|---|---|
| ใช้เมื่อ | ข้อมูลมีโครงสร้างเป็นลำดับชั้น/ต้นไม้ (comment replies, org chart, category ที่ซ้อนกันได้) |
| Syntax พิเศษ | ใช้ string `'self'` แทนชื่อโมเดล |
| ต้องมี `null=True` | ✅ เกือบทุกกรณี (เพื่อให้มี "ราก" ของต้นไม้ที่ไม่มี parent) |
| Reverse accessor ที่แนะนำ | ตั้ง `related_name` ให้สื่อความหมาย เช่น `'replies'`, `'children'`, `'subcategories'` |
| ความเสี่ยงด้าน performance | การไล่ทั้งต้นไม้ทำให้เกิด query จำนวนมากถ้าไม่ระวัง (ดูขั้นตอนที่ 117) |

---

## ขั้นตอนที่ 119: ออกแบบ Schema บล็อกให้สมบูรณ์ พร้อม ER Diagram

### 119.1 ไฟล์ `blog/models.py` ฉบับสมบูรณ์

รวบรวมทุกอย่างที่สร้างมาตลอด Part นี้เข้าด้วยกันเป็นไฟล์เดียว:

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True, blank=True)
    description = models.TextField(blank=True)

    class Meta:
        ordering = ['name']
        verbose_name_plural = 'categories'

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=60, unique=True, blank=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='posts',
    )
    tags = models.ManyToManyField(Tag, through='PostTag', related_name='posts', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)


class PostTag(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    tag = models.ForeignKey(Tag, on_delete=models.CASCADE)
    added_at = models.DateTimeField(auto_now_add=True)
    added_by = models.CharField(max_length=100, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=['post', 'tag'], name='unique_post_tag'),
        ]
        ordering = ['-added_at']

    def __str__(self):
        return f'{self.post.title} — {self.tag.name}'


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    parent = models.ForeignKey(
        'self',
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name='replies',
    )
    author = models.CharField(max_length=100)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['created_at']

    def __str__(self):
        return f'{self.author} แสดงความเห็นใน "{self.post.title}"'

    def get_depth(self):
        depth = 0
        current = self.parent
        while current is not None:
            depth += 1
            current = current.parent
        return depth
```

### 119.2 ไฟล์ `accounts/models.py` ฉบับสมบูรณ์

```python
# accounts/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    bio = models.TextField(max_length=500, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f'โปรไฟล์ของ {self.user.username}'
```

### 119.3 ER Diagram ของทั้งระบบ (แบบ Text/ASCII)

```
┌────────────────────┐
│   auth.User         │
│   (Django built-in) │
│──────────────────────│
│ id (PK)              │
│ username             │
│ email                │
│ password             │
└──────────┬───────────┘
           │ 1
           │
           │ OneToOneField (CASCADE)
           │
           │ 1
┌──────────▼───────────┐
│  accounts.Profile     │
│───────────────────────│
│ id (PK)                │
│ user_id (FK, unique)   │
│ bio                    │
│ avatar                 │
│ created_at / updated_at│
└────────────────────────┘

┌──────────────────────┐            ┌──────────────────────┐
│   blog.Category       │            │      blog.Tag          │
│────────────────────────│            │─────────────────────────│
│ id (PK)                 │            │ id (PK)                  │
│ name                     │            │ name                       │
│ slug                     │            │ slug                        │
│ description              │            └────────────┬────────────────┘
└──────────┬───────────────┘                         │ M
           │ 1                                        │
           │ ForeignKey (SET_NULL)                     │ ManyToMany
           │ related_name='posts'                       │ through PostTag
           │ N                                           │ related_name='posts'
┌──────────▼───────────────────────────────────────────▼───────┐
│                          blog.Post                              │
│───────────────────────────────────────────────────────────────│
│ id (PK)                                                          │
│ title / slug / content                                           │
│ category_id (FK -> Category, nullable)                           │
│ created_at / updated_at / is_published                           │
└───────────┬───────────────────────────────────┬─────────────────┘
            │ 1                                  │ M (through)
            │ ForeignKey (CASCADE)                │
            │ related_name='comments'             │
            │ N                                    │
┌───────────▼────────────┐              ┌─────────▼──────────────┐
│      blog.Comment        │              │     blog.PostTag         │
│───────────────────────────│              │────────────────────────────│
│ id (PK)                     │              │ id (PK)                     │
│ post_id (FK -> Post)         │              │ post_id (FK -> Post)         │
│ parent_id (FK -> self, null) │◄─┐           │ tag_id (FK -> Tag)            │
│ author / content / created_at│  │ self-FK    │ added_at / added_by            │
└───────────────────────────────┘  │ CASCADE   └────────────────────────────────┘
              │ related_name='replies'
              └─────────────────────┘
```

### 119.4 ตารางสรุปความสัมพันธ์ทั้งหมดในระบบ

| จาก | ถึง | ชนิดความสัมพันธ์ | `on_delete` | `related_name` |
|---|---|---|---|---|
| `Profile.user` | `User` | OneToOne | `CASCADE` | `'profile'` |
| `Post.category` | `Category` | ForeignKey | `SET_NULL` | `'posts'` |
| `Post.tags` | `Tag` (ผ่าน `PostTag`) | ManyToMany | (ตารางกลาง) | `'posts'` |
| `PostTag.post` | `Post` | ForeignKey | `CASCADE` | ค่า default (`postag_set`) |
| `PostTag.tag` | `Tag` | ForeignKey | `CASCADE` | ค่า default (`postag_set`) |
| `Comment.post` | `Post` | ForeignKey | `CASCADE` | `'comments'` |
| `Comment.parent` | `Comment` (self) | ForeignKey | `CASCADE` | `'replies'` |

### 119.5 ลำดับคำสั่ง Migration ทั้งหมดที่ต้องรันเรียงตาม Part นี้

ถ้าคุณเริ่มทำตาม Part นี้ตั้งแต่ต้นแบบเรียงลำดับ นี่คือลำดับคำสั่งที่ควรได้รันไปแล้วทั้งหมด:

```bash
# หลังขั้นตอนที่ 111 (Category + Post.category)
python manage.py makemigrations blog
python manage.py migrate

# หลังขั้นตอนที่ 112 (accounts app + Profile)
python manage.py startapp accounts
pip install Pillow
python manage.py makemigrations accounts
python manage.py migrate

# หลังขั้นตอนที่ 113 (Tag + Post.tags)
python manage.py makemigrations blog
python manage.py migrate

# หลังขั้นตอนที่ 114 (PostTag through model)
python manage.py makemigrations blog
python manage.py migrate

# หลังขั้นตอนที่ 115 (Comment + related_name='posts' บน Category)
python manage.py makemigrations blog
python manage.py migrate

# หลังขั้นตอนที่ 118 (Comment.parent self-referential)
python manage.py makemigrations blog
python manage.py migrate
```

ถ้าคุณเพิ่งเริ่มเขียนโมเดลทั้งหมดพร้อมกันในคราวเดียว (เช่น copy โค้ดจากขั้นตอนที่ 119.1
และ 119.2 ไปวางตรง ๆ) ก็รันแค่ครั้งเดียวต่อแอปได้เช่นกัน:

```bash
python manage.py makemigrations blog accounts
python manage.py migrate
```

---

## ขั้นตอนที่ 120: สรุปและแบบฝึกหัด

### 120.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ `ForeignKey` และตัวเลือก `on_delete` ทั้ง 7 แบบ (`CASCADE`, `PROTECT`,
  `RESTRICT`, `SET_NULL`, `SET_DEFAULT`, `SET()`, `DO_NOTHING`) พร้อมรู้ว่าควรเลือกใช้
  แบบไหนในสถานการณ์ใด
- ✅ สร้าง `Category` และเชื่อมกับ `Post` ด้วย `ForeignKey(on_delete=SET_NULL)`
- ✅ เข้าใจ `OneToOneField` และสร้าง `Profile` ขยาย Django `User` โดยไม่แตะต้อง
  auth model โดยตรง พร้อมรู้จักการใช้ `settings.AUTH_USER_MODEL`
- ✅ สร้างระบบ `Tag` ด้วย `ManyToManyField` และเข้าใจ junction table ที่ Django สร้างให้
  อัตโนมัติ
- ✅ ยกระดับ M2M ให้ใช้ `through` model กำหนดเอง (`PostTag`) เพื่อเก็บ metadata อย่าง
  `added_at`/`added_by`
- ✅ เข้าใจ `related_name`/`related_query_name` และ reverse relations ทั้งแบบ default
  (`_set`) และแบบตั้งชื่อเอง
- ✅ Query ข้ามความสัมพันธ์ด้วย double underscore ทั้งทิศทาง forward และ reverse
  พร้อมรู้จักปัญหาแถวซ้ำและวิธีแก้ด้วย `.distinct()`
- ✅ เข้าใจปัญหา N+1 Query และรู้จัก `select_related()`/`prefetch_related()` เบื้องต้น
- ✅ สร้าง `Comment` ที่ตอบกลับกันเองได้ด้วย self-referential `ForeignKey('self')`
- ✅ ออกแบบ schema บล็อกที่สมบูรณ์ 5 โมเดล พร้อมอ่าน ER diagram ของทั้งระบบได้

### 120.2 Checklist ก่อนไป Part ถัดไป

- [ ] `Category`, `Tag`, `PostTag`, `Comment` model ถูกสร้างและ migrate สำเร็จในแอป `blog`
- [ ] แอป `accounts` ถูกสร้างพร้อม `Profile` model และ migrate สำเร็จ
- [ ] อธิบายความแตกต่างของ `on_delete` ทั้ง 7 แบบได้โดยไม่ต้องเปิดเอกสาร
- [ ] สร้าง `Post` พร้อมกำหนด `category` และ `tags` ผ่าน shell ได้จริง
- [ ] เพิ่มแถวใน `PostTag` ผ่าน `through_defaults` หรือสร้างตรง ๆ ได้สำเร็จ
- [ ] เขียน query ที่ใช้ `related_name` ทั้งฝั่ง forward และ reverse ได้ถูกต้อง
- [ ] เขียน query ข้ามความสัมพันธ์ด้วย `__` ได้อย่างน้อย 3 แบบที่แตกต่างกัน และรู้ว่าเมื่อไหร่
  ต้องใช้ `.distinct()`
- [ ] สร้างคอมเมนต์ที่ตอบกลับกันเองได้อย่างน้อย 3 ระดับความลึก และเรียก `get_depth()` ได้ถูกต้อง
- [ ] วาด/อธิบาย ER diagram ของ schema ทั้งหมดในโปรเจกต์ตัวเองได้ด้วยคำพูดตัวเอง

### 120.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เปิด `python manage.py shell` แล้วสร้างข้อมูลตัวอย่างต่อไปนี้
ให้ครบ: หมวดหมู่ 3 หมวด (เช่น "เทคโนโลยี", "ไลฟ์สไตล์", "ท่องเที่ยว"), แท็ก 5 แท็ก,
บทความ 5 บทความที่กระจายอยู่ในแต่ละหมวดและมีแท็กอย่างน้อยบทความละ 2 แท็ก, และคอมเมนต์
อย่างน้อย 2 คอมเมนต์ต่อบทความ จากนั้นเขียนคำสั่ง query แสดงจำนวนบทความในแต่ละหมวดหมู่
โดยไม่ใช้ Django Admin (ใช้ shell/`.count()` เท่านั้น เพราะ Admin จะเรียนใน Part 017)

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เขียนฟังก์ชัน `render_comment_tree(post)` (ไม่ต้องเป็น
Django view ก็ได้ แค่ฟังก์ชัน Python ธรรมดาที่รันใน shell) ที่รับ `Post` object แล้ว
คืนค่าเป็น string ข้อความที่แสดงคอมเมนต์ทั้งหมดของบทความนั้นเป็นรูปแบบต้นไม้ โดยเยื้อง
(indent) ตามค่า `get_depth()` เช่น:

```
สมชาย: บทความดีมากครับ
  วิภา: เห็นด้วยเลย
    สมชาย: ขอบคุณครับ
  กิตติ: มีตัวอย่างเพิ่มไหมครับ
```

**แบบฝึกหัดที่ 3 (Through Model + Query)**: เพิ่ม field `note = models.CharField(
max_length=200, blank=True)` ให้ `PostTag` แล้วรัน migration ให้เรียบร้อย จากนั้นเขียน
query หา "5 แท็กที่ถูกใช้ในบทความมากที่สุด" โดยใช้ `Tag.objects.filter(...)` ร่วมกับ
`.count()` ของแต่ละแท็ก (ยังไม่ต้องใช้ `annotate()`/`Count()` เพราะเป็นหัวข้อของ
**Part 014** — ให้วนลูปนับด้วย Python ธรรมดาไปก่อนในแบบฝึกหัดนี้)

**แบบฝึกหัดที่ 4 (ขั้นสูง — Automated Test)**: เขียน test ด้วย `django.test.TestCase`
(ดูรูปแบบพื้นฐานจาก Part 006 ขั้นตอนที่ 58.7) เพื่อยืนยันพฤติกรรมของ `on_delete` ที่เลือกไว้
ในระบบจริง อย่างน้อย 2 กรณี:
(1) เมื่อลบ `Category` ที่มี `Post` อ้างอิงอยู่ ให้ตรวจสอบว่า `post.refresh_from_db()`
แล้ว `post.category` กลายเป็น `None` (ยืนยันพฤติกรรม `SET_NULL`)
(2) เมื่อลบ `Post` ที่มี `Comment` อยู่ ให้ตรวจสอบว่าจำนวน `Comment.objects.filter(post_id=...)`
ที่เหลือเป็น 0 (ยืนยันพฤติกรรม `CASCADE`)

### 120.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่ใช้ `CASCADE` กับ `Post.category` ไปเลย จะได้ไม่ต้องเขียน `null=True` ให้ยุ่งยาก?**
A: เพราะความหมายทางธุรกิจต่างกันโดยสิ้นเชิง `CASCADE` จะทำให้ **การลบหมวดหมู่หนึ่งหมวด
ลบบทความทั้งหมดในหมวดนั้นไปด้วย** ซึ่งแทบไม่มีระบบบล็อกไหนต้องการพฤติกรรมแบบนี้จริง ๆ
(ผู้ดูแลระบบอาจแค่อยากรวมหมวดหมู่เข้าด้วยกัน ไม่ได้อยากลบเนื้อหา) `SET_NULL` ปลอดภัยกว่ามาก
เพราะบทความยังอยู่ครบ แค่ไม่มีหมวดหมู่ชั่วคราวเท่านั้น กฎทั่วไปคือให้ถามตัวเองเสมอว่า "ถ้า
แถวแม่หาย แถวลูกควรหายตามไหม" ถ้าคำตอบคือไม่ ให้เลือก `SET_NULL`/`PROTECT` แทน `CASCADE`

**Q: `OneToOneField` กับ `ForeignKey(unique=True)` ต่างกันจริง ๆ ไหม ทำไมไม่ใช้แบบหลัง
ไปเลย?**
A: ในทางเทคนิค `OneToOneField` เทียบเท่ากับ `ForeignKey(unique=True)` เกือบทั้งหมดในระดับ
ฐานข้อมูล (คอลัมน์และ constraint เหมือนกัน) แต่ต่างกันที่ **reverse accessor**:
`OneToOneField` คืน object เดียวโดยตรง (`user.profile`) ในขณะที่ `ForeignKey(unique=True)`
ยังคงคืน `RelatedManager` ที่ต้องเรียก `.get()` เอง (`user.profile_set.get()`) ซึ่งดูแปลก
และไม่สื่อความหมาย Django จึงสร้าง `OneToOneField` ขึ้นมาเป็นทางลัดที่อ่านง่ายกว่าสำหรับ
กรณีที่ตั้งใจให้เป็น 1:1 ตั้งแต่ต้น ควรใช้ `OneToOneField` เสมอเมื่อความสัมพันธ์เป็น 1:1
โดยเจตนา

**Q: ทำไม `Post` instance ต้อง `.save()` ก่อนถึงจะเรียก `.tags.add()` ได้ แต่ `category`
(ForeignKey ธรรมดา) กำหนดค่าตรง ๆ ตอนสร้าง object ได้เลย?**
A: เพราะ `ForeignKey` เก็บค่าเป็นแค่คอลัมน์หนึ่งในตาราง `Post` เอง (`category_id`) การ
กำหนดค่าจึงเป็นแค่การตั้งค่า attribute ใน Python ธรรมดา ยังไม่ต้องเขียนอะไรลงฐานข้อมูล
ทันที แต่ `ManyToManyField` ต้องเขียนแถวใหม่ลง**ตารางกลางที่แยกต่างหาก** ซึ่งต้องใช้
`post_id` เป็น foreign key — ถ้า `Post` ยังไม่เคย `.save()` มันจะยังไม่มีค่า `id` (เป็น
`None`) ให้ใช้อ้างอิงในตารางกลางได้เลย

**Q: ควรใช้ self-referential ForeignKey กับข้อมูลแบบลำดับชั้นเสมอไปหรือไม่ เช่น
หมวดหมู่ที่มีหมวดหมู่ย่อยซ้อนกันได้หลายชั้น?**
A: สำหรับลำดับชั้นที่ไม่ลึกมาก (เช่น คอมเมนต์ตอบกลับ 2-4 ชั้น) self-referential
ForeignKey แบบ Adjacency List ที่เรียนใน Part นี้เพียงพอและเข้าใจง่ายที่สุด แต่ถ้าต้อง
รองรับลำดับชั้นที่ลึกมาก ๆ และต้องการ query แบบ "ดึงทั้งต้นไม้ย่อยในครั้งเดียว" บ่อย ๆ
(เช่น เมนูสินค้าที่ซ้อนกัน 10 ชั้น) ควรพิจารณา library สำเร็จรูปอย่าง `django-mptt` หรือ
`django-treebeard` ที่ใช้เทคนิคขั้นสูงกว่า (Nested Set, Materialized Path) เพื่อให้ query
ต้นไม้ทั้งหมดทำได้ในครั้งเดียวโดยไม่ต้องวน loop เอง — เป็นหัวข้อที่อยู่นอกขอบเขตของ
หลักสูตรพื้นฐานนี้ แต่ควรรู้จักชื่อไว้สำหรับงานจริงในอนาคต

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 013: Django ORM QuerySet ขั้นสูง** จะพาคุณกลับไปสำรวจ `QuerySet` ที่ใช้มาตลอด
Part นี้อย่างละเอียดยิ่งขึ้น โดยใช้โมเดลทั้ง 5 ตัวที่เพิ่งสร้าง (`Category`, `Tag`,
`Post`, `PostTag`, `Comment`) เป็นสนามฝึกจริง คุณจะได้เรียนรู้ **lazy evaluation**
ของ QuerySet (ทำไม query ถึงยังไม่ยิงจริงจนกว่าจะถูก evaluate), field lookups ทั้งหมด
(`__gt`, `__lt`, `__contains`, `__icontains`, `__in`, `__range`, `__isnull` ฯลฯ) แบบ
ครบถ้วน, การ chain หลาย `filter()` ต่อกันเทียบกับการรวมเงื่อนไขในครั้งเดียว, `values()`/
`values_list()`/`only()`/`defer()` สำหรับควบคุมว่าจะดึงข้อมูลคอลัมน์ไหนบ้าง, และ
`exists()`/`count()` ที่ประหยัด query กว่าการดึงข้อมูลทั้งหมดมาเช็ค เตรียม shell และ
ข้อมูลตัวอย่างที่คุณสร้างไว้ในแบบฝึกหัดที่ 1 ของ Part นี้ให้พร้อม เพราะเราจะใช้มันต่อทันที
ใน Part ถัดไป!
