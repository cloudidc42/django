# Part 070: Database Indexing และ Query Analysis

> **ขั้นตอนที่ 691-700 ของหลักสูตร** | Phase 8: Performance & Caching
>
> เป้าหมายของ Part นี้: ทำความเข้าใจว่า Database Index ทำงานอย่างไรจริง ๆ ในระดับ
> โครงสร้างข้อมูล (B-tree), สร้าง index ใน Django ผ่าน `Meta.indexes` และ `db_index`
> อย่างถูกต้องและมีเหตุผลรองรับ, อ่านผลลัพธ์จาก `EXPLAIN ANALYZE` ของ PostgreSQL ได้
> คล่องพอที่จะบอกได้ว่า query หนึ่ง ๆ "เร็วเพราะอะไร" หรือ "ช้าเพราะอะไร", เข้าใจ
> Composite Index และ Partial Index แบบ PostgreSQL-specific, รู้ทันว่าเมื่อไหร่ index
> กลับกลายเป็นภาระ, ใช้เครื่องมือขั้นสูงอย่าง `GinIndex`/`BrinIndex` และ
> `pg_stat_statements` เพื่อหา query ที่ช้าที่สุดในระบบจริง เมื่อจบ Part นี้ คุณจะ
> สามารถหยิบ query ที่ช้าจากระบบ production ขึ้นมาวิเคราะห์และแก้ไขด้วยข้อมูลจริง
> ไม่ใช่การเดา — เป็นทักษะที่ทำงานคู่กับ Query Optimization (Part 067) และ Caching
> (Part 068-069) เพื่อให้ทั้งสามชั้นของ performance ทำงานประสานกันอย่างสมบูรณ์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 691: แนวคิด Database Index — B-tree เบื้องต้น ทำไม index ถึงเร่งความเร็ว query
- ขั้นตอนที่ 692: การสร้าง Index ใน Django ผ่าน `Meta.indexes` และ `db_index=True` (ทบทวนแบบเจาะลึก)
- ขั้นตอนที่ 693: อ่านผลลัพธ์จาก `EXPLAIN ANALYZE` ของ PostgreSQL — Seq Scan vs Index Scan
- ขั้นตอนที่ 694: Composite Index (index หลาย column) และลำดับ column ที่มีผลต่อประสิทธิภาพ
- ขั้นตอนที่ 695: Partial Index (PostgreSQL-specific) — index เฉพาะ row ที่ตรงเงื่อนไข
- ขั้นตอนที่ 696: เมื่อไหร่ Index กลับเป็นผลเสีย — write overhead, over-indexing
- ขั้นตอนที่ 697: เครื่องมือของ Django สำหรับ PostgreSQL โดยเฉพาะ — GinIndex, BrinIndex
- ขั้นตอนที่ 698: การวิเคราะห์ Slow Query Log
- ขั้นตอนที่ 699: `pg_stat_statements` — extension สำหรับหา query ที่ช้าที่สุดในระบบจริง
- ขั้นตอนที่ 700: สรุปและแบบฝึกหัด — วิเคราะห์และ optimize query ของ blog project ด้วย EXPLAIN ANALYZE จริง

---

## ขั้นตอนที่ 691: แนวคิด Database Index — B-tree เบื้องต้น ทำไม index ถึงเร่งความเร็ว query

### 691.1 ปัญหาที่ index มาแก้: การค้นหาแบบ Seq Scan

ลองนึกภาพหนังสือหนา 1,000 หน้าที่ไม่มีสารบัญและไม่มีดัชนีท้ายเล่มเลย ถ้าคุณอยากหาคำว่า
"Django" ในหนังสือเล่มนี้ คุณต้องเปิดอ่านตั้งแต่หน้า 1 ไปเรื่อย ๆ จนกว่าจะเจอ — นี่คือ
สิ่งที่ฐานข้อมูลทำเมื่อตารางไม่มี index เลย เรียกว่า **Sequential Scan (Seq Scan)**:
ฐานข้อมูลต้องอ่าน **ทุกแถว** ของตารางเพื่อตรวจสอบว่าแถวไหนตรงกับเงื่อนไขที่ `WHERE`
ระบุไว้บ้าง

```python
# blog/models.py (ทบทวนจาก Part 067 — ใช้ต่อเนื่องตลอด Part นี้)
from django.conf import settings
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True)

    class Meta:
        verbose_name_plural = "categories"

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=60, unique=True)

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    view_count = models.IntegerField(default=0)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="authored_posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    is_approved = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["created_at"]

    def __str__(self):
        return f"Comment by {self.author} on post #{self.post_id}"
```

สมมติตาราง `blog_post` มี 500,000 แถว และคุณรัน:

```python
Post.objects.filter(slug="hello-world").first()
```

ถ้า `slug` ไม่มี index ฐานข้อมูลต้องอ่านทุกแถวจนครบ (หรือจนกว่าจะเจอ) — ในทางทฤษฎี
ความซับซ้อนคือ **O(n)** เทียบกับจำนวนแถวในตาราง ยิ่งตารางโตขึ้นเรื่อย ๆ query ก็จะช้าลง
เป็นเส้นตรงตามไปด้วย

### 691.2 Index แก้ปัญหานี้อย่างไร: แนวคิดสารบัญหนังสือ

Index คือโครงสร้างข้อมูลแยกต่างหากที่เก็บ **ค่าของ column ที่ index ไว้ พร้อมตำแหน่ง
ของแถวจริงในตาราง** เรียงลำดับไว้ล่วงหน้า เปรียบเหมือนสารบัญท้ายเล่มที่เรียงคำตาม
ตัวอักษรพร้อมเลขหน้า — แทนที่จะอ่านทุกหน้า คุณกระโดดไปที่สารบัญ หาคำที่ต้องการ (เร็ว
เพราะเรียงลำดับแล้ว) แล้วกระโดดตรงไปหน้านั้นเลย

PostgreSQL ใช้โครงสร้างข้อมูลที่เรียกว่า **B-tree (Balanced Tree)** เป็นค่าเริ่มต้น
สำหรับ index เกือบทุกชนิด (รวมถึง index ที่ Django สร้างให้อัตโนมัติทั้งหมด)

```
                         B-tree Index บน column "slug"
                         (แสดงแบบง่าย ความสูงจริงขึ้นกับจำนวนแถว)

                              ┌─────────────┐
                              │   "m..."    │   ระดับราก (root node)
                              └──────┬──────┘
                     ┌───────────────┼───────────────┐
              ┌──────▼──────┐              ┌──────────▼──────────┐
              │  "a".."l"   │              │      "m".."z"       │  ระดับกลาง (branch)
              └──────┬──────┘              └──────────┬──────────┘
        ┌────────────┼────────────┐        ┌──────────┼──────────┐
   ┌────▼───┐   ┌────▼───┐   ┌────▼───┐ ┌──▼───┐ ┌────▼───┐ ┌────▼───┐
   │"apple" │   │"django"│   │"go"    │ │"m..."│ │"python"│ │"zebra" │  ระดับใบ (leaf)
   │→ row#7 │   │→ row#2 │   │→row#88 │ │→...  │ │→row#12 │ │→row#41 │  (เก็บ pointer ไปแถวจริง)
   └────────┘   └────────┘   └────────┘ └──────┘ └────────┘ └────────┘
```

จุดสำคัญของ B-tree:

- **สมดุลเสมอ (balanced)**: ทุก leaf node อยู่ที่ความลึกเท่ากันเสมอ ไม่ว่าข้อมูลจะเพิ่ม
  หรือลบไปเท่าไร ฐานข้อมูลจะปรับโครงสร้างอัตโนมัติ (rebalance) เพื่อรักษาคุณสมบัตินี้
- **เรียงลำดับ**: ข้อมูลใน B-tree เรียงจากน้อยไปมากเสมอ ทำให้ค้นหาแบบ range (`>`, `<`,
  `BETWEEN`) และ `ORDER BY` ทำได้เร็วมากด้วย เพราะข้อมูลที่ติดกันในลำดับก็ติดกันจริง
  ในโครงสร้าง
- **ความซับซ้อนของการค้นหา**: **O(log n)** แทนที่จะเป็น O(n) — ตารางที่มี 1,000,000
  แถว การค้นหาแบบ Seq Scan ต้องเทียบสูงสุด 1,000,000 ครั้ง แต่ B-tree (ที่มี fan-out
  สูง เช่น หลายร้อยต่อ node) อาจใช้แค่ 3-4 ระดับในการหาแถวที่ต้องการ

### 691.3 ตารางเปรียบเทียบ Seq Scan vs Index Scan

| ประเด็น | Seq Scan (ไม่มี index) | Index Scan (มี B-tree index) |
|---|---|---|
| ความซับซ้อนเชิงทฤษฎี | O(n) — เชิงเส้นตามจำนวนแถว | O(log n) — เติบโตช้ามากเมื่อข้อมูลเพิ่ม |
| วิธีอ่านข้อมูล | อ่านทุกแถวของตารางตามลำดับที่เก็บจริง (heap) | เดินตาม B-tree ไปยัง leaf ที่ตรงเงื่อนไข แล้ว jump ไปอ่านแถวจริง |
| เหมาะกับ | ตารางเล็กมาก หรือ query ที่ต้องอ่านเกือบทุกแถวอยู่แล้ว | ตารางใหญ่ที่ query กรองข้อมูลให้เหลือเปอร์เซ็นต์น้อยของทั้งหมด |
| ต้นทุนตอน write (INSERT/UPDATE/DELETE) | ไม่มีต้นทุนเพิ่ม (ไม่มี index ให้ต้องอัปเดต) | ต้องอัปเดตโครงสร้าง index ทุกครั้งที่ column ที่ index เปลี่ยน (ดูขั้นตอนที่ 696) |
| พื้นที่ดิสก์ที่ใช้ | เฉพาะข้อมูลจริงของตาราง | ข้อมูลจริง + พื้นที่ของ index เพิ่มเติม (มักหลาย % ถึงหลายสิบ % ของขนาดตาราง) |

### 691.4 Index ที่ Django สร้างให้อัตโนมัติโดยที่คุณไม่ต้องทำอะไรเลย

Django สร้าง index ให้อัตโนมัติในสามกรณีนี้เสมอ โดยไม่ต้องระบุอะไรเพิ่ม:

| กรณี | เกิด index อัตโนมัติเพราะอะไร |
|---|---|
| `primary_key=True` (รวมถึง `id` อัตโนมัติ) | ฐานข้อมูลสร้าง unique B-tree index ให้ primary key เสมอ (มาตรฐาน SQL) |
| `unique=True` บน field ใด ๆ | Django/ฐานข้อมูลต้องมี index เพื่อตรวจสอบความซ้ำได้เร็ว (ไม่งั้นต้อง Seq Scan ทุกครั้งที่ insert) |
| `ForeignKey` / `OneToOneField` | Django สร้าง index ให้อัตโนมัติเสมอ (ค่าเริ่มต้นภายในคือ `db_index=True` มีผลจริง) เพราะ join และ `WHERE fk_id = ...` เป็น pattern ที่พบบ่อยที่สุด |

ลองตรวจสอบด้วยตัวเองใน `psql` หลัง migrate โมเดล `Post` ด้านบนแล้ว:

```sql
-- เชื่อมต่อฐานข้อมูลด้วย psql
\d blog_post
```

```
Indexes:
    "blog_post_pkey" PRIMARY KEY, btree (id)
    "blog_post_slug_key" UNIQUE CONSTRAINT, btree (slug)
    "blog_post_category_id_a8e4a2f1" btree (category_id)
    "blog_post_author_id_f3c91b02" btree (author_id)
```

สังเกตว่าแม้เราไม่เคยเขียน `Meta.indexes` หรือ `db_index=True` เลยสักครั้งในโมเดลนี้
ก็มี index เกิดขึ้นแล้ว 4 ตัว — นี่คือเหตุผลที่ query อย่าง `Post.objects.get(slug=...)`
หรือ `Post.objects.filter(category=cat)` ที่เราเขียนมาตลอดหลักสูตร (ตั้งแต่ Part 011)
เร็วอยู่แล้วโดยไม่รู้ตัว สิ่งที่ Part นี้จะเพิ่มเข้ามาคือ index สำหรับ pattern การ query
ที่ **ซับซ้อนกว่านั้น** ซึ่ง Django ไม่สามารถเดาให้อัตโนมัติได้ (เช่น กรอง `is_published`
ร่วมกับเรียง `-created_at` พร้อมกัน — ดูขั้นตอนที่ 694)

### 691.5 ชนิด Index อื่นที่ PostgreSQL รองรับ (ภาพรวมก่อนเจาะลึกในขั้นตอนที่ 697)

B-tree ไม่ใช่ index ชนิดเดียวที่มี PostgreSQL ยังมีชนิดอื่นที่เหมาะกับข้อมูลต่างรูปแบบ:

| ชนิด Index | เหมาะกับ | Django class |
|---|---|---|
| **B-tree** (ค่าเริ่มต้น) | ความเท่ากัน (`=`), ช่วง (`<`, `>`, `BETWEEN`), `ORDER BY` — ใช้ได้กับข้อมูลเกือบทุกประเภท | `models.Index` |
| **GIN** (Generalized Inverted Index) | ข้อมูลที่มีหลายค่าในแถวเดียว เช่น `ArrayField`, `JSONField`, full-text search | `django.contrib.postgres.indexes.GinIndex` |
| **BRIN** (Block Range Index) | ข้อมูลขนาดใหญ่มากที่เรียงตามธรรมชาติอยู่แล้ว เช่น timestamp ของ log ที่ insert ตามลำดับเวลา | `django.contrib.postgres.indexes.BrinIndex` |
| **GiST** | ข้อมูลเชิงพื้นที่/รูปทรง (geometric), full-text search แบบ fuzzy | `django.contrib.postgres.indexes.GistIndex` |
| **Hash** | ความเท่ากันล้วน (`=` เท่านั้น ไม่รองรับ range) — ใช้น้อยกว่า B-tree มากในทางปฏิบัติ | `django.contrib.postgres.indexes.HashIndex` |

ตลอด Part นี้เราจะเน้น B-tree เป็นหลัก (ขั้นตอนที่ 692-696) แล้วค่อยไปดู GIN/BRIN
แบบเจาะลึกในขั้นตอนที่ 697

---

## ขั้นตอนที่ 692: การสร้าง Index ใน Django ผ่าน `Meta.indexes` และ `db_index=True` (ทบทวนแบบเจาะลึก)

### 692.1 ทบทวนสั้น ๆ จาก Part 015

ใน Part 015 ขั้นตอนที่ 141.6 เราเกริ่นไว้ว่า `Meta.indexes` ใช้เพิ่ม index แบบกำหนดเอง
นอกเหนือจาก field-level ใน Part นี้เราจะเจาะลึกทุกทางเลือกที่มี พร้อมเหตุผลว่าควรเลือก
แบบไหนเมื่อไหร่

### 692.2 ทางเลือกที่ 1: `db_index=True` ที่ระดับ field

วิธีที่ง่ายที่สุดสำหรับ index แบบ column เดียว:

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)  # unique=True ได้ index มาแล้ว
    is_published = models.BooleanField(default=False, db_index=True)  # index เดี่ยว
    view_count = models.IntegerField(default=0)
    # ...
```

Migration ที่เกิดขึ้น:

```python
# blog/migrations/0002_alter_post_is_published.py
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0001_initial"),
    ]

    operations = [
        migrations.AlterField(
            model_name="post",
            name="is_published",
            field=models.BooleanField(db_index=True, default=False),
        ),
    ]
```

Django จะตั้งชื่อ index ให้อัตโนมัติในรูปแบบ `<table>_<column>_<hash>` เช่น
`blog_post_is_published_4f2a1e3c` — ชื่อแบบนี้อ่านยากเมื่อต้องไปดูใน `psql` หรือ
`EXPLAIN ANALYZE` output ว่า index ตัวไหนถูกใช้งาน นี่คือข้อจำกัดหลักของ `db_index=True`

### 692.3 ทางเลือกที่ 2: `Meta.indexes` พร้อมตั้งชื่อเอง

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    view_count = models.IntegerField(default=0)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="authored_posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["is_published"], name="post_is_published_idx"),
            models.Index(fields=["view_count"], name="post_view_count_idx"),
        ]
```

ข้อดีของการใช้ `Meta.indexes` แทน `db_index=True` เสมอสำหรับโค้ดใหม่:

| ประเด็น | `db_index=True` | `Meta.indexes` |
|---|---|---|
| ตั้งชื่อ index เอง | ❌ ตั้งชื่ออัตโนมัติ (มี hash ต่อท้าย อ่านยาก) | ✅ ตั้งชื่อเองผ่าน `name=` |
| Index หลาย column (composite) | ❌ ทำไม่ได้ | ✅ ทำได้ (ดูขั้นตอนที่ 694) |
| Partial index (มีเงื่อนไข) | ❌ ทำไม่ได้ | ✅ ทำได้ผ่าน `condition=` (ดูขั้นตอนที่ 695) |
| Expression index (index บนผลลัพธ์ของฟังก์ชัน) | ❌ ทำไม่ได้ | ✅ ทำได้ผ่าน `Func()` expressions (Django 4.0+) |
| อยู่รวมกันในที่เดียวมองเห็นภาพรวมง่าย | ❌ กระจายอยู่ตาม field ต่าง ๆ | ✅ อยู่รวมกันใน `Meta.indexes` list เดียว |
| ใช้กับ GIN/BRIN/GiST | ❌ ไม่ได้ | ✅ ได้ (import จาก `django.contrib.postgres.indexes`) |

**คำแนะนำระดับมืออาชีพของ Part นี้**: ใช้ `Meta.indexes` เป็นค่าเริ่มต้นเสมอสำหรับ
index ใหม่ทุกตัว ยกเว้นกรณีง่ายที่สุด (index เดี่ยวไม่มีเงื่อนไข) ที่ `db_index=True`
ยังอ่านง่ายพอ แต่ในทางปฏิบัติทีม production ส่วนใหญ่เลือกใช้ `Meta.indexes` ทั้งหมด
เพื่อความสม่ำเสมอของโค้ด (consistency)

### 692.4 ข้อจำกัดเรื่องความยาวชื่อ index

ฐานข้อมูลหลายตัวจำกัดความยาวชื่อ identifier (รวมถึงชื่อ index) — PostgreSQL จำกัดที่
**63 ตัวอักษร** (ยาวกว่า MySQL ที่จำกัด 64 และ Oracle รุ่นเก่าที่จำกัดแค่ 30) แต่ Django
เอง**แนะนำให้ตั้งชื่อ index ไม่เกิน 30 ตัวอักษร** เพื่อให้โค้ดพกพาข้ามฐานข้อมูลได้
ปลอดภัยที่สุด แม้โปรเจกต์ของคุณจะใช้ PostgreSQL เท่านั้นก็ตาม (ตามกฎเหล็กของหลักสูตรนี้
ตั้งแต่ Part 002 ให้ใช้ PostgreSQL ตลอด แต่ชื่อสั้นก็ยังอ่านง่ายกว่าเสมอ):

```python
# ✅ ดี — สั้น ชัดเจน อ่านง่าย
models.Index(fields=["is_published"], name="post_published_idx")

# ❌ ยาวเกินไป เสี่ยงถูก truncate หรือ error บนบางฐานข้อมูล
models.Index(
    fields=["is_published"],
    name="post_is_published_status_lookup_index_v2",
)
```

ถ้าตั้งชื่อยาวเกิน Django จะแจ้ง error ตอนรัน `makemigrations` หรือ `migrate` ทันที
(`SystemCheckError` หรือ database-level error) ไม่ปล่อยผ่านให้เกิดปัญหาทีหลัง

### 692.5 ดู index ที่มีอยู่ทั้งหมดของโมเดลผ่าน Django shell

```bash
python manage.py dbshell
```

```sql
-- ดู index ทั้งหมดของตาราง blog_post พร้อมขนาด
SELECT
    indexname,
    indexdef,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'blog_post';
```

```
        indexname          |                          indexdef                              | index_size
----------------------------+-----------------------------------------------------------------+------------
 blog_post_pkey             | CREATE UNIQUE INDEX blog_post_pkey ON blog_post USING btree (id)| 40 kB
 blog_post_slug_key         | CREATE UNIQUE INDEX ... USING btree (slug)                      | 56 kB
 blog_post_category_id_...  | CREATE INDEX ... USING btree (category_id)                      | 40 kB
 blog_post_author_id_...    | CREATE INDEX ... USING btree (author_id)                        | 40 kB
 post_published_idx         | CREATE INDEX post_published_idx ON blog_post USING btree (is_published)| 24 kB
```

คำสั่งนี้มีประโยชน์มากเวลาตรวจสอบว่ามี index ซ้ำซ้อนกันโดยไม่ตั้งใจหรือไม่ (ปัญหา
over-indexing ที่จะพูดถึงในขั้นตอนที่ 696)

### 692.6 ตรวจสอบ index จากฝั่ง Django โดยไม่ต้องเข้า `psql`

```python
# ใน Django shell (python manage.py shell)
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute(
        "SELECT indexname, indexdef FROM pg_indexes WHERE tablename = %s",
        ["blog_post"],
    )
    for name, definition in cursor.fetchall():
        print(name)
        print(f"  {definition}")
```

---

## ขั้นตอนที่ 693: อ่านผลลัพธ์จาก `EXPLAIN ANALYZE` ของ PostgreSQL — Seq Scan vs Index Scan

### 693.1 `EXPLAIN` คืออะไร

`EXPLAIN` คือคำสั่ง SQL ที่ขอให้ PostgreSQL แสดง **query plan** — แผนการที่ query
planner ตัดสินใจว่าจะดึงข้อมูลอย่างไร (ใช้ index ตัวไหน, join แบบไหน, สแกนตารางไหน
ก่อน) โดย **ไม่ต้องรัน query จริง** ส่วน `EXPLAIN ANALYZE` จะ **รัน query จริง** แล้ว
แสดงทั้งแผนที่วางไว้ **และ** เวลาที่ใช้จริงในแต่ละขั้นตอน

**ข้อควรระวัง**: `EXPLAIN ANALYZE` รัน query จริง ถ้า query นั้นเป็น `DELETE`/`UPDATE`
มันจะ**ลบ/แก้ข้อมูลจริง** ห้ามใช้กับ query ที่แก้ไขข้อมูลบนฐานข้อมูล production
โดยไม่ได้ครอบด้วย transaction ที่ rollback ได้เสมอ

### 693.2 ใช้ผ่าน Django ORM ด้วย `QuerySet.explain()`

Django มีเมธอด `.explain()` บน QuerySet มาตั้งแต่ Django 3.0 ทำให้ไม่ต้องเขียน raw SQL
เอง:

```python
from blog.models import Post

qs = Post.objects.filter(is_published=True).order_by("-created_at")[:10]

# EXPLAIN ธรรมดา (ไม่รัน query จริง แค่ดูแผน)
print(qs.explain())

# EXPLAIN ANALYZE (รัน query จริง + วัดเวลาจริง) — เฉพาะ backend ที่รองรับ (PostgreSQL รองรับเต็มรูปแบบ)
print(qs.explain(analyze=True))

# แสดงข้อมูล buffer (จำนวน page ที่อ่านจาก cache/disk) — มีประโยชน์มากในขั้นตอนที่ 698
print(qs.explain(analyze=True, buffers=True))

# แสดงผลเป็น JSON เพื่อ parse ด้วยโปรแกรมต่อ
print(qs.explain(analyze=True, format="json"))
```

### 693.3 ตัวอย่างจริง: query ที่ยังไม่มี index เหมาะสม

สมมติ `Post` มี 100,000 แถว และ `is_published` **ยังไม่มี index** เลย:

```python
print(
    Post.objects.filter(is_published=True).order_by("-created_at")[:10].explain(analyze=True)
)
```

```
Limit  (cost=3245.67..3245.69 rows=10 width=120) (actual time=42.301..42.305 rows=10 loops=1)
  ->  Sort  (cost=3245.67..3370.67 rows=50000 width=120) (actual time=42.299..42.301 rows=10 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  Seq Scan on blog_post  (cost=0.00..2345.00 rows=50000 width=120) (actual time=0.015..28.442 rows=50000 loops=1)
              Filter: is_published
              Rows Removed by Filter: 50000
Planning Time: 0.234 ms
Execution Time: 42.356 ms
```

อ่านผลลัพธ์ทีละส่วนจากล่างขึ้นบน (ลำดับการทำงานจริงเริ่มจากโหนดที่ลึกที่สุด):

1. **`Seq Scan on blog_post`**: PostgreSQL อ่าน**ทุกแถว** (100,000 แถว ทั้ง published
   และ unpublished) แล้วค่อยกรอง `is_published = true` ทีหลัง (`Filter:`) — เห็นได้จาก
   `Rows Removed by Filter: 50000` ว่ามี 50,000 แถวที่อ่านมาแล้วทิ้งไปเพราะไม่ตรงเงื่อนไข
2. **`Sort`**: หลังกรองได้ 50,000 แถวที่ published ต้องนำมาเรียงตาม `created_at DESC`
   ทั้งหมดก่อนตัดเอาแค่ 10 อันบนสุด (`top-N heapsort`)
3. **`Limit`**: ตัดเหลือ 10 แถวสุดท้าย
4. **`Execution Time: 42.356 ms`**: เวลาที่ใช้จริงทั้งหมด — นี่คือตัวเลขที่สำคัญที่สุด

### 693.4 ตัวอย่างเดียวกันหลังเพิ่ม Composite Index (ทำจริงในขั้นตอนที่ 694)

```
Limit  (cost=0.43..8.52 rows=10 width=120) (actual time=0.031..0.089 rows=10 loops=1)
  ->  Index Scan using post_published_created_idx on blog_post  (cost=0.43..40521.28 rows=50000 width=120) (actual time=0.030..0.086 rows=10 loops=1)
        Index Cond: (is_published = true)
Planning Time: 0.198 ms
Execution Time: 0.112 ms
```

เวลาลดจาก **42.356 ms → 0.112 ms** (เร็วขึ้นประมาณ 378 เท่า) เพราะตอนนี้:

- ใช้ **`Index Scan using post_published_created_idx`** แทน `Seq Scan` — เดินตาม
  B-tree ไปหาแถวที่ `is_published = true` โดยตรง
- index ถูกสร้างให้เรียงตาม `created_at DESC` อยู่แล้ว (composite index รวม order)
  จึงไม่ต้องมี node `Sort` แยกต่างหากอีกต่อไป — PostgreSQL อ่านแถวจาก index ตามลำดับ
  ที่ต้องการได้เลย

### 693.5 ตารางสรุปประเภท Scan Node ที่พบบ่อยที่สุด

| Node type | ความหมาย | เมื่อไหร่เกิด |
|---|---|---|
| `Seq Scan` | อ่านทุกแถวของตารางตามลำดับที่เก็บจริง (heap) | ไม่มี index ที่เหมาะสม หรือตารางเล็กจน planner ตัดสินใจว่า Seq Scan เร็วกว่า index |
| `Index Scan` | เดินตาม B-tree แล้ว jump ไปอ่านแถวจริงจาก heap ทีละแถว | มี index ที่ตรงกับเงื่อนไข และจำนวนแถวที่ match ค่อนข้างน้อย |
| `Index Only Scan` | ดึงข้อมูลจาก index **อย่างเดียว** โดยไม่ต้องแตะ heap เลย | ทุก column ที่ query ต้องการอยู่ใน index แล้ว (covering index) — เร็วกว่า Index Scan ธรรมดา |
| `Bitmap Heap Scan` + `Bitmap Index Scan` | สร้าง "แผนที่บิต" ของแถวที่ตรงเงื่อนไขก่อน แล้วค่อยไปอ่าน heap ตามลำดับ page (ลด random I/O) | จำนวนแถวที่ match ค่อนข้างมาก (มากกว่าจะใช้ Index Scan ตรง ๆ แต่ยังน้อยกว่าจะทำ Seq Scan คุ้มกว่า) |
| `Nested Loop` | join โดยวนลูปตารางหนึ่งแล้ว query อีกตารางซ้ำ ๆ | ตารางฝั่งหนึ่งมีแถวน้อย และอีกฝั่งมี index รองรับเงื่อนไข join |
| `Hash Join` | สร้าง hash table จากตารางเล็กก่อน แล้ว join กับตารางใหญ่ | ทั้งสองตารางมีขนาดใหญ่พอสมควร ไม่มี index ที่เหมาะกับ Nested Loop |
| `Merge Join` | join ตารางที่เรียงลำดับแล้วทั้งคู่ (มักมาจาก index) โดยเดินคู่ขนานกัน | ทั้งสองฝั่งเรียงลำดับตาม join key อยู่แล้ว (เช่นมี index ตรงกัน) |

### 693.6 ตัวเลข `cost` หมายถึงอะไร

```
Seq Scan on blog_post  (cost=0.00..2345.00 rows=50000 width=120)
```

- `cost=0.00..2345.00` คือ **ต้นทุนโดยประมาณ** ของ planner ในหน่วยที่ไม่ใช่เวลาจริง
  (arbitrary unit เทียบจากต้นทุนของการอ่าน 1 หน้าดิสก์แบบ sequential = 1.0) ตัวแรก
  (0.00) คือต้นทุนก่อนได้แถวแรก ตัวที่สอง (2345.00) คือต้นทุนรวมทั้งหมด
- `rows=50000` คือ**จำนวนแถวที่ planner ประมาณการไว้ล่วงหน้า** (จากสถิติที่เก็บไว้
  ผ่าน `ANALYZE`/autovacuum) — ถ้าตัวเลขนี้ต่างจาก `actual rows` มากในผลลัพธ์ของ
  `EXPLAIN ANALYZE` แปลว่าสถิติของตารางอาจล้าสมัย ควรรัน `ANALYZE blog_post;` ใหม่
- `width=120` คือขนาดเฉลี่ยโดยประมาณของแต่ละแถว (หน่วย byte)
- **`actual time=X..Y`** (ปรากฏเฉพาะเมื่อใช้ `ANALYZE`) คือเวลาจริงที่วัดได้ (มิลลิวินาที)
  ตัวแรกคือเวลาก่อนได้แถวแรก ตัวที่สองคือเวลารวมของ node นั้น

### 693.7 ใช้ `EXPLAIN (ANALYZE, BUFFERS)` เพื่อดู I/O จริง

```python
print(
    Post.objects.filter(slug="hello-world")
    .explain(analyze=True, buffers=True, verbose=True)
)
```

```
Index Scan using blog_post_slug_key on blog_post  (cost=0.42..8.44 rows=1 width=310) (actual time=0.045..0.047 rows=1 loops=1)
  Index Cond: (slug = 'hello-world'::text)
  Buffers: shared hit=4
Planning Time: 0.112 ms
Execution Time: 0.071 ms
```

`Buffers: shared hit=4` หมายถึงอ่านข้อมูล 4 page (แต่ละ page = 8 KB โดยปกติ) จาก
**shared buffer cache ของ PostgreSQL เอง** (ไม่ต้องไปแตะดิสก์เลย) ถ้าเห็น
`shared read=N` แทน (หรือร่วมกับ `hit`) หมายความว่าต้องอ่านจากดิสก์จริง ซึ่งช้ากว่า
มาก — ตัวเลขนี้มีประโยชน์มากเวลาวิเคราะห์ query ที่ทำงานช้าเพราะ cache miss ระดับ
ฐานข้อมูล (คนละชั้นกับ Redis cache ที่เรียนใน Part 068-069 — นี่คือ cache ภายใน
PostgreSQL เอง)

---

## ขั้นตอนที่ 694: Composite Index (index หลาย column) และลำดับ column ที่มีผลต่อประสิทธิภาพ

### 694.1 ทำไมต้องมี Composite Index

query ที่พบบ่อยที่สุดในหน้า Homepage ของบล็อก (ทบทวนจาก Part 067) คือ:

```python
Post.objects.filter(is_published=True).order_by("-created_at")[:10]
```

query นี้มีเงื่อนไขสองส่วนที่ทำงานร่วมกันเสมอ: **กรอง** ด้วย `is_published` และ
**เรียงลำดับ** ด้วย `created_at` ถ้าสร้าง index แยกกันสองตัว (`is_published` ตัวหนึ่ง,
`created_at` อีกตัวหนึ่ง) PostgreSQL จะเลือกใช้ได้แค่ตัวเดียวอย่างมีประสิทธิภาพเต็มที่
ในแต่ละครั้ง (หรือต้องทำ `BitmapAnd` รวมสอง index เข้าด้วยกัน ซึ่งมีต้นทุนเพิ่ม) —
วิธีที่ดีที่สุดคือสร้าง **Composite Index** ที่รวมทั้งสอง column ไว้ในตัวเดียว:

```python
class Post(models.Model):
    # ... fields เดิมทั้งหมด ...

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
        ]
```

Migration ที่ Django สร้างให้:

```python
# blog/migrations/0003_post_post_published_created_idx.py
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ("blog", "0002_alter_post_is_published"),
    ]

    operations = [
        migrations.AddIndex(
            model_name="post",
            index=models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
        ),
    ]
```

SQL ที่ PostgreSQL รันจริงเบื้องหลัง:

```sql
CREATE INDEX post_published_created_idx
ON blog_post USING btree (is_published, created_at DESC);
```

### 694.2 กฎ Leftmost Prefix — หัวใจสำคัญของ Composite Index

Composite index ทำงานเหมือนสมุดโทรศัพท์ที่เรียงตาม **(นามสกุล, ชื่อ)** — คุณค้นหา
คนที่นามสกุล "สมิท" ได้เร็ว และถ้าอยากได้เฉพาะ "สมิท" ที่ชื่อ "จอห์น" ก็ยังเร็วเพราะ
ทั้งคู่เรียงต่อกัน **แต่ถ้าอยากหาทุกคนที่ชื่อ "จอห์น" โดยไม่สนนามสกุล สมุดเล่มนี้ช่วย
อะไรไม่ได้เลย** เพราะคนชื่อจอห์นกระจายอยู่ทั่วทุกหน้าตามนามสกุลที่ต่างกัน

กฎนี้เรียกว่า **Leftmost Prefix Rule**: composite index `(A, B, C)` ใช้ประโยชน์เต็มที่
ได้เมื่อ query กรอง/เรียงด้วย column ที่เป็น **prefix ต่อเนื่องจากซ้ายมือ** เท่านั้น:

| Query filter/order | ใช้ index `(is_published, created_at)` ได้ไหม | เหตุผล |
|---|---|---|
| `filter(is_published=True)` | ✅ ใช้ได้เต็มที่ | column แรกตรง prefix |
| `filter(is_published=True).order_by("-created_at")` | ✅ ใช้ได้เต็มที่ (กรณีในตัวอย่าง) | ครบทั้ง prefix ตามลำดับ |
| `filter(created_at__gte=some_date)` (ไม่กรอง `is_published`) | ❌ ใช้ไม่ได้ (หรือใช้ได้แค่บางส่วนแบบไม่มีประสิทธิภาพ) | ข้าม column แรกไป ไม่ตรง leftmost prefix |
| `order_by("-created_at")` อย่างเดียว (ไม่กรอง) | ⚠️ ใช้ได้บางส่วน แต่ไม่คุ้มเท่า index ที่มีแค่ `created_at` เดี่ยว ๆ | planner อาจเลือก scan ทั้ง index แทน ซึ่งไม่ต่างจาก Seq Scan มากนัก |

**บทเรียนสำคัญ**: column ที่ถูก **filter ด้วยความเท่ากัน (`=`)** ควรอยู่ **ซ้ายสุด**
ของ composite index เสมอ ส่วน column ที่ใช้ `ORDER BY` หรือ range (`>`, `<`) ควรอยู่
**ถัดไป** — ลำดับนี้สำคัญมาก สลับกันแล้วประสิทธิภาพจะต่างกันอย่างมีนัยสำคัญ

### 694.3 ทิศทางของ column ใน index (`ASC`/`DESC`) ก็มีผลเช่นกัน

```python
class Post(models.Model):
    class Meta:
        indexes = [
            models.Index(
                fields=["category", "-created_at"],
                name="post_category_created_idx",
            ),
        ]
```

`fields=["category", "-created_at"]` สร้าง index ที่เรียง `category` จากน้อยไปมาก
(ปกติ) แล้วภายในแต่ละ category เรียง `created_at` จากมากไปน้อย (`-` หมายถึง `DESC`)
— นี่คือทิศทางที่ **ตรงกับ** query แบบ `Post.objects.filter(category=cat).order_by
("-created_at")` เป๊ะ ทำให้ PostgreSQL อ่านข้อมูลตามลำดับ index ได้เลยโดยไม่ต้อง
`Sort` เพิ่ม

ถ้าเขียนสลับทิศทางผิด (เช่น index เก็บ `created_at ASC` แต่ query ขอ `ORDER BY
created_at DESC`) PostgreSQL (ตั้งแต่เวอร์ชัน 8.3 เป็นต้นมา) ยังฉลาดพอที่จะ**เดินย้อน
กลับ (backward scan)** บน B-tree ได้โดยแทบไม่มีต้นทุนเพิ่ม เพราะ B-tree เดินได้สอง
ทิศทาง แต่สำหรับ **composite** index ที่มีหลาย column ผสมทิศทางกัน (เช่น column แรก
ASC แต่ column ที่สองต้องการ DESC) การเดินย้อนกลับทั้ง index จะไม่ตรงกับที่ query
ต้องการอีกต่อไป — จึงต้องระบุทิศทางแต่ละ column ให้ตรงกับ pattern การ query จริงเสมอ

### 694.4 ตัวอย่างการออกแบบ Composite Index ให้ครอบคลุมหลาย query pattern

สมมติหน้า Category listing ของบล็อกมี query ทั้งสองแบบนี้ที่ใช้บ่อยพอ ๆ กัน:

```python
# Query A: หน้าแรกของ homepage
Post.objects.filter(is_published=True).order_by("-created_at")[:10]

# Query B: หน้าแสดงโพสต์ตามหมวดหมู่
Post.objects.filter(is_published=True, category=cat).order_by("-created_at")[:10]
```

```python
class Post(models.Model):
    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
            models.Index(
                fields=["is_published", "category", "-created_at"],
                name="post_pub_cat_created_idx",
            ),
        ]
```

Index ตัวที่สอง (`is_published, category, created_at`) ครอบคลุม **ทั้ง Query A และ
Query B** ตามกฎ leftmost prefix (Query A ใช้แค่ `is_published` + `created_at` ซึ่ง
เป็น prefix ที่ข้าม column กลางได้บางส่วนถ้า planner ตัดสินใจว่าคุ้ม) แต่ในทางปฏิบัติ
**การมี index ทั้งสองตัวแยกกันมักจะให้ผลลัพธ์ที่ชัดเจนและคาดเดาได้ง่ายกว่า** เพราะแต่ละ
ตัวตอบโจทย์ query pattern ของตัวเองตรง ๆ — trade-off นี้ (จำนวน index ที่เพิ่ม vs
ความครอบคลุม) จะอธิบายเพิ่มเติมในขั้นตอนที่ 696 เรื่อง over-indexing

### 694.5 ตรวจสอบด้วย `EXPLAIN ANALYZE` ว่า Composite Index ถูกใช้จริง

```python
print(
    Post.objects.filter(is_published=True, category_id=3)
    .order_by("-created_at")[:10]
    .explain(analyze=True)
)
```

```
Limit  (cost=0.56..12.88 rows=10 width=120) (actual time=0.028..0.095 rows=10 loops=1)
  ->  Index Scan using post_pub_cat_created_idx on blog_post  (cost=0.56..1847.21 rows=1500 width=120) (actual time=0.027..0.092 rows=10 loops=1)
        Index Cond: ((is_published = true) AND (category_id = 3))
Planning Time: 0.256 ms
Execution Time: 0.121 ms
```

`Index Cond: ((is_published = true) AND (category_id = 3))` ยืนยันว่า PostgreSQL ใช้
ทั้งสอง column แรกของ index ในการกรองโดยตรง (ไม่ต้องมี `Filter:` เพิ่มเติมหลัง Index
Scan เลย) และไม่มี node `Sort` เพราะลำดับที่ได้จาก index ตรงกับ `ORDER BY` ที่ต้องการ
อยู่แล้ว — นี่คือสัญญาณว่า composite index ถูกออกแบบมาถูกต้อง

---

## ขั้นตอนที่ 695: Partial Index (PostgreSQL-specific) — index เฉพาะ row ที่ตรงเงื่อนไข

### 695.1 ปัญหา: Index บน column ที่มีค่าไม่สมดุลกัน (Low Cardinality)

สมมติในตาราง `blog_post` 500,000 แถว มีโพสต์ที่ `is_published=True` อยู่ 490,000 แถว
และ `is_published=False` (draft) อยู่แค่ 10,000 แถว — ถ้า query ที่ใช้บ่อยที่สุดคือหา
**เฉพาะ draft** (`is_published=False`) การสร้าง index ปกติบน `is_published` ทั้งคอลัมน์
จะมีขนาดใหญ่โดยไม่จำเป็น เพราะ index เก็บทั้ง 500,000 แถว (รวมทั้งค่าที่ query ไม่เคย
สนใจคือ `True` อีก 490,000 รายการ)

**Partial Index** แก้ปัญหานี้โดยสร้าง index ที่ครอบคลุม **เฉพาะแถวที่ตรงเงื่อนไขที่
กำหนด** เท่านั้น:

```python
class Post(models.Model):
    class Meta:
        indexes = [
            models.Index(
                fields=["-created_at"],
                name="post_draft_created_idx",
                condition=models.Q(is_published=False),
            ),
        ]
```

SQL ที่เกิดขึ้นจริง:

```sql
CREATE INDEX post_draft_created_idx
ON blog_post USING btree (created_at DESC)
WHERE (is_published = false);
```

index ตัวนี้มีขนาดเล็กกว่า index เต็มคอลัมน์มาก (ครอบคลุมแค่ 10,000 แถวจาก 500,000)
ทำให้ B-tree มีความสูง (depth) น้อยกว่า, ใช้พื้นที่ดิสก์น้อยกว่า, และ**เร็วกว่า**เมื่อ
query ตรงกับเงื่อนไขที่ระบุไว้พอดี

### 695.2 กฎสำคัญ: Query ต้องมีเงื่อนไขตรงกับ (หรือ subset ของ) condition ของ index

```python
# ✅ ใช้ post_draft_created_idx ได้ — เงื่อนไขตรงกับ condition ของ index เป๊ะ
Post.objects.filter(is_published=False).order_by("-created_at")

# ❌ ใช้ index นี้ไม่ได้ — เงื่อนไขไม่ตรง (query หา published, index ครอบคลุมแค่ draft)
Post.objects.filter(is_published=True).order_by("-created_at")

# ❌ ใช้ index นี้ไม่ได้เต็มประสิทธิภาพ — query ไม่มีเงื่อนไข is_published เลย
# planner ไม่มีทางรู้ว่าผลลัพธ์ทั้งหมดอยู่ในขอบเขตของ partial index หรือไม่
Post.objects.all().order_by("-created_at")
```

**ข้อควรระวังสำคัญที่สุดของ Partial Index**: มันเป็น**เครื่องมือเฉพาะทาง** ที่ต้องรู้
pattern การ query ล่วงหน้าอย่างชัดเจน ถ้าเงื่อนไขของ query เปลี่ยนไปแม้เพียงเล็กน้อย
(เช่น เปลี่ยนจาก `is_published=False` เป็น `is_published=False, category=cat`) planner
ก็ยังใช้ partial index นี้ได้ (เพราะเป็น superset ของเงื่อนไข) แต่ถ้าเงื่อนไขไม่ตรง
เลยแม้แต่น้อย (เช่นสลับเป็น `is_published=True`) index นี้จะถูกมองข้ามไปโดยสิ้นเชิง

### 695.3 ตัวอย่างจริง: Partial Unique Index (ผสาน UniqueConstraint จาก Part 015)

ทบทวนจาก Part 015 ขั้นตอนที่ 142.3 — `UniqueConstraint` ที่มี `condition` แท้จริงแล้ว
คือ Partial Unique Index ในระดับฐานข้อมูล:

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    is_approved = models.BooleanField(default=True)
    parent = models.ForeignKey(
        "self", on_delete=models.CASCADE, null=True, blank=True, related_name="replies",
    )
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["created_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["post", "author", "text"],
                condition=models.Q(parent__isnull=True),
                name="unique_top_level_comment_text",
            ),
        ]
```

SQL ที่ PostgreSQL เห็น:

```sql
CREATE UNIQUE INDEX unique_top_level_comment_text
ON blog_comment (post_id, author, text)
WHERE (parent_id IS NULL);
```

นี่คือหลักฐานว่า `UniqueConstraint` ที่มี `condition` และ `models.Index` ที่มี
`condition` เป็นแนวคิดเดียวกันในระดับฐานข้อมูล — ต่างกันแค่ `UniqueConstraint` บังคับ
ความไม่ซ้ำเพิ่มด้วย ในขณะที่ `models.Index(condition=...)` สร้างไว้เพื่อ**เร่งความเร็ว
การอ่าน**อย่างเดียว ไม่มีผลต่อการ validate ข้อมูล

### 695.4 กรณีใช้งานจริงของ Partial Index สำหรับ blog project

```python
class Comment(models.Model):
    # ... fields เดิม ...

    class Meta:
        ordering = ["created_at"]
        indexes = [
            # หน้า Admin queue สำหรับตรวจสอบคอมเมนต์ที่ยังไม่อนุมัติ — เข้าถึงบ่อยมาก
            # แต่จำนวนคอมเมนต์ที่รอตรวจมักเป็นสัดส่วนน้อยของทั้งหมด
            models.Index(
                fields=["-created_at"],
                name="comment_pending_review_idx",
                condition=models.Q(is_approved=False),
            ),
        ]
```

```python
# blog/views.py — หน้า admin queue ที่ query นี้จะใช้ partial index ข้างต้นโดยตรง
def pending_comments_queue(request):
    comments = Comment.objects.filter(is_approved=False).select_related("post")
    return render(request, "blog/pending_comments.html", {"comments": comments})
```

### 695.5 ตารางสรุป Partial Index

| ประเด็น | รายละเอียด |
|---|---|
| Syntax ใน Django | `models.Index(fields=[...], name=..., condition=models.Q(...))` |
| รองรับฐานข้อมูล | PostgreSQL และ SQLite เต็มรูปแบบ, **MySQL ไม่รองรับ** (ต้อง fallback ไปใช้ index เต็มคอลัมน์) |
| ข้อดี | ขนาด index เล็กลงมาก, เร็วขึ้นเมื่อ query ตรงเงื่อนไข, ลด write overhead เทียบกับ index เต็มคอลัมน์ |
| ข้อจำกัด | ใช้ได้เฉพาะ query ที่มีเงื่อนไขตรง (หรือ superset ของ) condition ของ index เท่านั้น |
| เหมาะกับ | column ที่มีค่ากระจายไม่สมดุล (low cardinality ในกลุ่มที่สนใจ) เช่น status flag, soft-delete flag |

---

## ขั้นตอนที่ 696: เมื่อไหร่ Index กลับเป็นผลเสีย — write overhead, over-indexing

### 696.1 Index ไม่ใช่ของฟรี: ต้นทุนตอน Write

ทุกครั้งที่มีการ `INSERT`, `UPDATE` (เฉพาะ column ที่ index ไว้), หรือ `DELETE` แถวใน
ตาราง ฐานข้อมูล**ต้องอัปเดตโครงสร้าง B-tree ของทุก index ที่เกี่ยวข้องด้วย** ไม่ใช่
แค่อัปเดตตัวข้อมูลจริง (heap) เท่านั้น:

```
INSERT 1 แถวลงตารางที่มี 5 index
= เขียนข้อมูลจริงลง heap  1 ครั้ง
+ อัปเดต B-tree ของ index #1  1 ครั้ง
+ อัปเดต B-tree ของ index #2  1 ครั้ง
+ อัปเดต B-tree ของ index #3  1 ครั้ง
+ อัปเดต B-tree ของ index #4  1 ครั้ง
+ อัปเดต B-tree ของ index #5  1 ครั้ง
= งานเขียนทั้งหมด 6 ครั้ง แทนที่จะเป็น 1 ครั้ง
```

ทดลองวัดผลจริงด้วย Django shell:

```python
import time
from blog.models import Post, Category

cat = Category.objects.first()

start = time.perf_counter()
Post.objects.bulk_create([
    Post(title=f"Bulk Post {i}", slug=f"bulk-post-{i}", content="...", category=cat)
    for i in range(10_000)
])
elapsed = time.perf_counter() - start
print(f"Insert 10,000 แถวใช้เวลา: {elapsed:.3f} วินาที")
```

| จำนวน index บนตาราง | เวลาที่ใช้ insert 10,000 แถว (ตัวอย่างจากการวัดจริงบนเครื่องพัฒนา) |
|---|---|
| 1 (แค่ primary key) | ~0.42 วินาที |
| 4 (pk + unique slug + 2 FK index มาตรฐาน) | ~0.68 วินาที |
| 8 (เพิ่ม composite index อีก 4 ตัวแบบไม่จำเป็น) | ~1.35 วินาที |

ตัวเลขจริงแตกต่างกันไปตามฮาร์ดแวร์และเวอร์ชัน PostgreSQL แต่แนวโน้มชัดเจนเสมอ:
**ยิ่ง index เยอะ ยิ่งเขียนช้าลง** เป็นสัดส่วนใกล้เคียงเชิงเส้นกับจำนวน index

### 696.2 Over-indexing คืออะไร และสัญญาณที่บ่งบอก

**Over-indexing** คือการสร้าง index มากเกินความจำเป็น จนต้นทุนด้าน write และพื้นที่
ดิสก์เกินกว่าประโยชน์ที่ได้จากความเร็วในการอ่าน สัญญาณที่บ่งบอกว่าอาจกำลัง over-index
อยู่:

| สัญญาณ | วิธีตรวจสอบ |
|---|---|
| มี index หลายตัวที่ column ซ้อนทับกันเกือบทั้งหมด | ดูผลจาก `pg_indexes` (ขั้นตอนที่ 692.5) เทียบ column ที่ใช้ |
| มี index ที่ไม่เคยถูกใช้เลย | ดูจาก `pg_stat_user_indexes` (ขั้นตอนที่ 696.4) |
| Insert/Update ช้าลงผิดปกติเมื่อเทียบกับ read ที่เร็วขึ้นเพียงเล็กน้อย | เปรียบเทียบ benchmark ก่อน/หลังเพิ่ม index |
| ตารางมีขนาดบนดิสก์ใหญ่กว่าที่ควรมาก | `pg_total_relation_size()` รวม index ทุกตัวเข้ากับข้อมูลจริง |

### 696.2.1 หา index ที่ไม่เคยถูกใช้เลย — `pg_stat_user_indexes`

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'blog_post'
ORDER BY idx_scan ASC;
```

```
 schemaname | table_name |            index_name             | times_used | index_size
------------+------------+------------------------------------+------------+------------
 public     | blog_post  | post_view_count_idx                |          0 | 3256 kB
 public     | blog_post  | post_pub_cat_created_idx            |         12 | 4102 kB
 public     | blog_post  | post_published_created_idx          |       8841 | 3890 kB
 public     | blog_post  | blog_post_slug_key                  |      15302 | 4560 kB
```

`idx_scan = 0` สำหรับ `post_view_count_idx` หมายความว่า **ตั้งแต่ PostgreSQL รีสตาร์ท
ครั้งล่าสุด (หรือตั้งแต่สร้าง index) ยังไม่มี query ไหนใช้ index นี้เลยแม้แต่ครั้งเดียว**
— นี่คือผู้ต้องสงสัยอันดับหนึ่งที่ควรพิจารณาลบทิ้ง เพราะมันกินพื้นที่ดิสก์ 3.2 MB และ
เพิ่มต้นทุน write ทุกครั้งที่มี insert/update โดยไม่ให้ประโยชน์อะไรตอบแทนเลย

**ข้อควรระวัง**: ตัวเลข `idx_scan` รีเซ็ตเป็น 0 ทุกครั้งที่ PostgreSQL restart หรือเมื่อ
รันคำสั่ง `SELECT pg_stat_reset();` ดังนั้นควรตรวจสอบบนฐานข้อมูล production ที่รันมา
ต่อเนื่องนานพอสมควร (เช่น อย่างน้อย 1-2 สัปดาห์ที่ครอบคลุมทุก pattern การใช้งานจริง
ทั้ง batch job รายเดือน, รายงานสิ้นเดือน ฯลฯ) ก่อนตัดสินใจลบ index ใด ๆ

### 696.3 ลบ Index ที่ไม่จำเป็นออกอย่างปลอดภัย

```python
class Post(models.Model):
    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
            # ลบ index นี้ออก เพราะ pg_stat_user_indexes ยืนยันว่าไม่เคยถูกใช้เลย
            # models.Index(fields=["view_count"], name="post_view_count_idx"),
        ]
```

```bash
python manage.py makemigrations blog
```

```
Migrations for 'blog':
  blog/migrations/0004_remove_post_post_view_count_idx.py
    - Remove index post_view_count_idx from post
```

```bash
python manage.py migrate
```

**คำแนะนำระดับมืออาชีพ**: ก่อนลบ index บนฐานข้อมูล production จริง ให้ใช้
`DROP INDEX CONCURRENTLY` (PostgreSQL เท่านั้น) แทนการรัน migration ตรง ๆ ในระบบที่มี
traffic สูงมาก เพราะ `CONCURRENTLY` จะไม่ lock ตารางระหว่างลบ index (แม้จะใช้เวลานาน
กว่าเล็กน้อย) — Django migration แบบปกติจะ lock ตารางช่วงสั้น ๆ ระหว่างลบ ซึ่งมักไม่
เป็นปัญหาสำหรับตารางขนาดเล็ก-กลาง แต่ควรระวังกับตารางที่มี traffic เขียนสูงมากและ
ใหญ่มาก (เราจะเจาะลึกเรื่อง zero-downtime migration แบบเต็มใน Phase 11)

### 696.4 ตารางสรุป: เมื่อไหร่ควรสร้าง Index เมื่อไหร่ไม่ควร

| สถานการณ์ | ควรสร้าง Index ไหม |
|---|---|
| Column ที่ใช้ `filter()`/`get()` บ่อยมาก และตารางมีข้อมูลเยอะ (หลักหมื่นขึ้นไป) | ✅ ควร |
| Column ที่ใช้ `order_by()` บ่อย ร่วมกับ filter ที่ใช้ index อยู่แล้ว | ✅ ควร (ทำเป็น composite) |
| Foreign Key ที่ query ผ่าน relation บ่อย | ✅ Django สร้างให้อัตโนมัติแล้ว ไม่ต้องทำอะไรเพิ่ม |
| Column ที่ `write` (insert/update) บ่อยมาก แต่แทบไม่เคยถูก query โดยตรง | ❌ ไม่ควร — เพิ่มแต่ write overhead |
| Column ที่มีค่าเดียวเกือบทั้งตาราง (เช่น `status='active'` 99.9% ของแถว) โดยไม่ใช้ partial index | ❌ ไม่คุ้ม — planner มักเลือก Seq Scan อยู่ดีเพราะแทบทุกแถวตรงเงื่อนไข |
| ตารางเล็กมาก (ต่ำกว่าหลักพันแถว) | ❌ มักไม่คุ้ม — Seq Scan ทั้งตารางเร็วพอแล้ว การมี index กลับเพิ่มความซับซ้อนโดยไม่จำเป็น |
| Column ที่ query ไม่เคยใช้เลยแต่ "เผื่อไว้ในอนาคต" | ❌ ไม่ควร — เพิ่ม index เมื่อพิสูจน์ได้จาก `EXPLAIN ANALYZE`/`pg_stat_statements` ว่าจำเป็นจริง ไม่ใช่เดาล่วงหน้า |

**หลักการทองคำของ Part นี้**: **อย่าเพิ่ม index เพราะ "คิดว่าน่าจะช่วย" ให้เพิ่มเพราะ
มีหลักฐานจาก `EXPLAIN ANALYZE` (ขั้นตอนที่ 693) หรือ `pg_stat_statements` (ขั้นตอนที่
699) ว่า query จริงกำลังช้าเพราะขาด index นั้นอยู่** และหลังเพิ่มแล้วต้องกลับไปวัดผล
ซ้ำด้วยเครื่องมือเดียวกันเสมอเพื่อยืนยันว่าได้ผลจริง

---

## ขั้นตอนที่ 697: เครื่องมือของ Django สำหรับ PostgreSQL โดยเฉพาะ — GinIndex, BrinIndex

### 697.1 ทบทวน: ทำไม B-tree ไม่เหมาะกับทุกกรณี

B-tree เหมาะกับข้อมูลที่เปรียบเทียบได้แบบ scalar (ตัวเลข, วันที่, ข้อความสั้น) แต่ไม่
เหมาะกับข้อมูลบางประเภท:

- **`JSONField`**: ค่าภายในเป็นโครงสร้างซับซ้อน ไม่มี "ค่าเดียว" ที่จะเรียงลำดับตรง ๆ
- **`ArrayField`**: หนึ่งแถวมีได้หลายค่าพร้อมกัน (multi-valued คล้าย M2M)
- **Full-text search**: ต้องการค้นหาคำภายในข้อความยาว ไม่ใช่เทียบทั้งข้อความตรง ๆ

`django.contrib.postgres.indexes` มี index class เพิ่มเติมที่ออกแบบมาสำหรับกรณีเหล่านี้
โดยเฉพาะ — ต้องเพิ่ม `"django.contrib.postgres"` ใน `INSTALLED_APPS` ก่อนใช้งาน:

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "django.contrib.postgres",
    "blog",
]
```

### 697.2 `GinIndex` — สำหรับ Full-text Search และข้อมูลหลายค่า

**GIN (Generalized Inverted Index)** เหมาะกับข้อมูลที่ "หนึ่งค่าในคอลัมน์ ประกอบด้วย
หลายค่าย่อยข้างใน" — เปรียบเหมือนดัชนีท้ายหนังสือที่บอกว่าคำ ๆ หนึ่งปรากฏอยู่ที่หน้า
ไหนบ้าง (กลับด้านจาก B-tree ที่บอกว่าแถวหนึ่งมีค่าอะไร)

ตัวอย่างเพิ่ม full-text search ให้ `Post.content`:

```python
# blog/models.py
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    search_vector = SearchVectorField(null=True, blank=True)
    # ... field อื่น ๆ เดิม ...

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
            GinIndex(fields=["search_vector"], name="post_search_vector_gin_idx"),
        ]
```

ต้องอัปเดตค่า `search_vector` ด้วย trigger หรือ signal ทุกครั้งที่ `title`/`content`
เปลี่ยน (รายละเอียดเต็มเรื่อง full-text search จะอยู่ใน Part ถัดไปของหลักสูตรที่
เกี่ยวกับ search — ตอนนี้ขอโฟกัสที่ตัว index):

```python
# blog/signals.py
from django.contrib.postgres.search import SearchVector
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Post


@receiver(post_save, sender=Post)
def update_search_vector(sender, instance, **kwargs):
    Post.objects.filter(pk=instance.pk).update(
        search_vector=SearchVector("title", weight="A") + SearchVector("content", weight="B")
    )
```

ใช้งานค้นหา:

```python
from django.contrib.postgres.search import SearchQuery

results = Post.objects.filter(search_vector=SearchQuery("django ORM"))
print(results.explain(analyze=True))
```

```
Bitmap Heap Scan on blog_post  (cost=12.25..145.30 rows=42 width=310) (actual time=0.089..0.201 rows=38 loops=1)
  Recheck Cond: (search_vector @@ '''django'' & ''orm'''::tsquery)
  Heap Blocks: exact=35
  ->  Bitmap Index Scan on post_search_vector_gin_idx  (cost=0.00..12.24 rows=42 width=0) (actual time=0.065..0.065 rows=38 loops=1)
        Index Cond: (search_vector @@ '''django'' & ''orm'''::tsquery)
Planning Time: 0.312 ms
Execution Time: 0.245 ms
```

ไม่มี GIN index เลย query แบบนี้จะกลายเป็น Seq Scan ที่ต้องอ่านและเทียบ full-text
ทุกแถวในตาราง ซึ่งช้ามากเมื่อมีข้อมูลหลักหมื่นแถวขึ้นไป

### 697.3 `GinIndex` กับ `ArrayField`

```python
from django.contrib.postgres.fields import ArrayField
from django.contrib.postgres.indexes import GinIndex


class Post(models.Model):
    # ... fields เดิม ...
    keywords = ArrayField(models.CharField(max_length=50), blank=True, default=list)

    class Meta:
        indexes = [
            GinIndex(fields=["keywords"], name="post_keywords_gin_idx"),
        ]
```

```python
# ค้นหาโพสต์ที่มีคำว่า "orm" อยู่ใน keywords (array แบบใดก็ได้ที่มีค่านี้)
Post.objects.filter(keywords__contains=["orm"])
```

GIN index ทำให้ `__contains` บน `ArrayField` เร็วขึ้นมาก เพราะโดยปกติ PostgreSQL ต้อง
เปิดดู array ทุกแถวเพื่อเช็คว่ามีค่านั้นอยู่ไหม (คล้าย Seq Scan ระดับ array) แต่ GIN
สร้าง "ดัชนีกลับ" ที่บอกว่าคำแต่ละคำอยู่ในแถวไหนบ้างไว้ล่วงหน้า

### 697.4 `BrinIndex` — สำหรับข้อมูลขนาดใหญ่มากที่เรียงตามธรรมชาติ

**BRIN (Block Range Index)** เหมาะกับตารางที่มีข้อมูลจำนวนมหาศาล (หลักล้านถึงพันล้าน
แถว) ที่ column นั้น **มีความสัมพันธ์ตามธรรมชาติกับลำดับการจัดเก็บทางกายภาพ** —
กรณีคลาสสิกที่สุดคือ timestamp ของข้อมูลที่ insert ตามลำดับเวลาต่อเนื่อง เช่น log,
audit trail, sensor data:

```python
# analytics/models.py
from django.contrib.postgres.indexes import BrinIndex
from django.db import models


class PageView(models.Model):
    """
    ตารางบันทึกการเข้าชมหน้าเว็บ — เหมาะเป็นตัวอย่าง BRIN เพราะ:
    1. มีข้อมูลปริมาณมากมาก (append-only, insert ต่อเนื่องตามเวลา)
    2. viewed_at เรียงตามลำดับการ insert อยู่แล้วโดยธรรมชาติ (ไม่มีการ update/สลับลำดับ)
    """
    post = models.ForeignKey("blog.Post", on_delete=models.CASCADE, related_name="page_views")
    viewed_at = models.DateTimeField(auto_now_add=True)
    ip_address = models.GenericIPAddressField()

    class Meta:
        indexes = [
            BrinIndex(fields=["viewed_at"], name="pageview_viewed_at_brin_idx"),
        ]
```

จุดต่างจาก B-tree ที่สำคัญที่สุด: **BRIN ไม่เก็บตำแหน่งของทุกแถวแบบละเอียด** แต่เก็บ
แค่ **ช่วงค่าต่ำสุด-สูงสุดของแต่ละ "กลุ่ม page ทางกายภาพ" (block range)** เช่น
"page ที่ 1-128 มีค่า viewed_at อยู่ระหว่าง 2026-01-01 ถึง 2026-01-03" ทำให้ query
แบบช่วงเวลาสามารถ**ข้าม (skip)** ทั้งกลุ่ม block ที่ไม่เกี่ยวข้องได้ทันทีโดยไม่ต้อง
ไล่ดูทีละแถว:

```python
from django.utils import timezone
from datetime import timedelta

recent = PageView.objects.filter(
    viewed_at__gte=timezone.now() - timedelta(days=1)
)
print(recent.explain(analyze=True))
```

```
Bitmap Heap Scan on analytics_pageview  (cost=45.20..8934.55 rows=42000 width=32) (actual time=1.203..15.442 rows=41850 loops=1)
  Recheck Cond: (viewed_at >= '2026-09-25 10:00:00'::timestamp)
  Rows Removed by Index Recheck: 890
  Heap Blocks: lossy=1024
  ->  Bitmap Index Scan on pageview_viewed_at_brin_idx  (cost=0.00..45.19 rows=84000 width=0) (actual time=1.180..1.180 rows=10240 loops=1)
        Index Cond: (viewed_at >= '2026-09-25 10:00:00'::timestamp)
Planning Time: 0.198 ms
Execution Time: 16.021 ms
```

### 697.5 ตารางเปรียบเทียบ BRIN vs B-tree

| ประเด็น | B-tree | BRIN |
|---|---|---|
| ขนาด index | ใหญ่กว่ามาก (เก็บทุกแถวแบบละเอียด) | เล็กมาก (เก็บแค่ min/max ต่อกลุ่ม block — มักเล็กกว่า B-tree หลายร้อยเท่า) |
| ความแม่นยำ | แม่นยำเป๊ะ ชี้ตำแหน่งแถวได้ตรง | เป็น "lossy" — ต้อง recheck แถวจริงหลัง scan เสมอ (เห็นจาก `Rows Removed by Index Recheck`) |
| เหมาะกับข้อมูล | ทุกขนาด แต่คุ้มที่สุดกับข้อมูลขนาดกลาง-ใหญ่ | ข้อมูลขนาดใหญ่มาก (หลักล้าน-พันล้านแถว) ที่เรียงตามธรรมชาติ |
| ต้นทุนตอน write | สูงกว่า (ต้องอัปเดตตำแหน่งละเอียดทุกครั้ง) | ต่ำมาก (อัปเดตแค่ช่วง min/max ของ block ที่เกี่ยวข้อง) |
| ใช้ผิดที่ (ข้อมูลไม่เรียงตามธรรมชาติ) | ไม่มีปัญหา | **ไม่มีประโยชน์เลย** — ถ้าข้อมูลถูก update/สลับตำแหน่งบ่อย ช่วง min/max ของแต่ละ block จะทับซ้อนกันหมด ทำให้ query ต้อง scan เกือบทุก block อยู่ดี |

**คำแนะนำ**: ใช้ BRIN เฉพาะกับตารางแบบ **append-only** ที่มีปริมาณข้อมูลมหาศาลจริง ๆ
(เช่น log, analytics, IoT sensor data) และ column นั้นสัมพันธ์กับลำดับการ insert
โดยตรง — สำหรับตาราง `blog_post`/`blog_comment` ของโปรเจกต์เราที่ขนาดยังไม่ถึงระดับ
ล้านแถว **B-tree ยังคงเป็นตัวเลือกที่เหมาะสมที่สุดเสมอ** BRIN จะมีประโยชน์จริงเมื่อ
ระบบโตไปถึงขนาด analytics/logging ในระดับองค์กรที่จะพูดถึงเพิ่มเติมใน Phase 11

---

## ขั้นตอนที่ 698: การวิเคราะห์ Slow Query Log

### 698.1 เปิดใช้งาน Slow Query Logging ใน PostgreSQL

`EXPLAIN ANALYZE` (ขั้นตอนที่ 693) เหมาะกับการวิเคราะห์ query **ทีละตัวที่รู้อยู่แล้ว
ว่าน่าสงสัย** แต่ในระบบ production จริง คุณมักไม่รู้ล่วงหน้าว่า query ไหนช้า —
**Slow Query Log** คือกลไกที่ให้ PostgreSQL **บันทึกอัตโนมัติ** ทุก query ที่ใช้เวลา
เกินเกณฑ์ที่กำหนดไว้ ลงไฟล์ log โดยไม่ต้องรอให้ผู้ใช้ร้องเรียนก่อน

แก้ไขไฟล์ `postgresql.conf` (ตำแหน่งไฟล์ขึ้นกับการติดตั้ง เช่น
`/etc/postgresql/16/main/postgresql.conf` บน Ubuntu):

```ini
# postgresql.conf

# บันทึก query ที่ใช้เวลานานกว่า 200 มิลลิวินาที (ปรับตามความเหมาะสมของระบบ)
log_min_duration_statement = 200

# บันทึกด้วยว่า query แต่ละอันมาจาก session ไหน (มีประโยชน์เวลา debug)
log_line_prefix = '%m [%p] user=%u,db=%d,app=%a '

# บันทึกด้วยว่าใช้เวลา checkpoint นานแค่ไหน (มีผลต่อ write performance โดยรวม)
log_checkpoints = on
```

รีสตาร์ท PostgreSQL เพื่อให้ค่ามีผล:

```bash
sudo systemctl restart postgresql
```

### 698.2 อ่านและตีความ Log ที่ได้

```
2026-09-26 14:32:10.123 UTC [8821] user=blog_app,db=blog_prod,app=blog-web LOG:
  duration: 842.301 ms  statement: SELECT "blog_post"."id", "blog_post"."title", ...
  FROM "blog_post" WHERE UPPER("blog_post"."title"::text) LIKE UPPER('%django%')
  ORDER BY "blog_post"."created_at" DESC
```

log entry นี้บอกเราหลายอย่างพร้อมกัน:

- **เวลา** ที่ query เกิดขึ้น (`2026-09-26 14:32:10.123 UTC`)
- **ใช้เวลานานแค่ไหน** (`duration: 842.301 ms` — ช้ากว่าเกณฑ์ 200 ms ที่ตั้งไว้มาก)
- **SQL เต็ม** ที่รันจริง — เห็นได้ทันทีว่าปัญหาคือ `UPPER(...) LIKE UPPER('%django%')`
  ซึ่งเป็น pattern แบบ `LIKE` ที่ขึ้นต้นด้วย wildcard (`%`) ทำให้ **B-tree index ปกติ
  ใช้งานไม่ได้เลย** (ต้องใช้ full-text search หรือ `pg_trgm` extension แทน — เกินขอบเขต
  ของ Part นี้ แต่เป็นตัวอย่างที่ดีว่า log ช่วยชี้ปัญหาที่ index ปกติแก้ไม่ได้)

### 698.3 หา query ต้นตอในโค้ด Django จาก SQL ที่เห็นใน log

เมื่อเจอ SQL ที่ช้าจาก log ขั้นตอนต่อไปคือหาว่า Django view/ORM code ไหนที่สร้าง SQL
นี้ขึ้นมา วิธีที่เร็วที่สุดคือค้นหา pattern เฉพาะในโค้ด:

```python
# blog/views.py — ตัวต้นเหตุที่พบจากการค้นหาคำว่า "title__icontains" ในโค้ด
def search_posts(request):
    query = request.GET.get("q", "")
    posts = Post.objects.filter(title__icontains=query)  # icontains แปลเป็น UPPER(...) LIKE
    return render(request, "blog/search_results.html", {"posts": posts})
```

`title__icontains` ของ Django ORM แปลเป็น `UPPER(column) LIKE UPPER('%value%')` (หรือ
`ILIKE` ขึ้นกับเวอร์ชัน) ตรงกับ SQL ที่เห็นในไฟล์ log เป๊ะ — ยืนยันได้ด้วย
`str(queryset.query)` หรือ `.explain()` เพื่อเทียบ SQL ให้แน่ใจ 100% ว่าตรงกับที่เจอ
ใน log

### 698.4 ใช้ `django-silk` หรือ Django logging middleware จับ query ที่ช้าแบบ per-request

นอกจาก log ระดับฐานข้อมูล คุณยังตั้งค่าให้ Django เองบันทึก query ที่ช้าต่อ 1 HTTP
request ได้ผ่านการตั้งค่า logging ธรรมดา (ใช้ `django.db.backends` logger ที่มีอยู่
แล้วในตัว Django ตั้งแต่ Part แรก ๆ ของหลักสูตร):

```python
# settings.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {
        "slow_queries_only": {
            "()": "django.utils.log.CallbackFilter",
            "callback": lambda record: getattr(record, "duration", 0) > 0.2,
        },
    },
    "handlers": {
        "slow_query_file": {
            "level": "DEBUG",
            "class": "logging.FileHandler",
            "filename": "logs/slow_queries.log",
            "filters": ["slow_queries_only"],
        },
    },
    "loggers": {
        "django.db.backends": {
            "handlers": ["slow_query_file"],
            "level": "DEBUG",
        },
    },
}
```

**ข้อควรระวังสำคัญ**: logger `django.db.backends` ทำงานได้เฉพาะเมื่อ `settings.DEBUG
= True` เท่านั้น (Django ไม่ log query ทุกตัวใน production เพื่อประสิทธิภาพ ตามที่
ทบทวนจาก Part 067 ขั้นตอนที่ 661.4) การใช้วิธีนี้ใน production จริงจึงเหมาะกับการ
เปิดใช้ **ชั่วคราว** เพื่อ debug ปัญหาเฉพาะจุดเท่านั้น ไม่ใช่เปิดทิ้งไว้ตลอดเวลา —
สำหรับ production จริงที่ต้องการ visibility ต่อเนื่อง ควรใช้ Slow Query Log ระดับ
PostgreSQL (ขั้นตอนที่ 698.1) หรือ `pg_stat_statements` (ขั้นตอนที่ 699) แทน เพราะ
ทำงานได้โดยไม่ขึ้นกับ `DEBUG` และมี overhead ต่ำกว่ามาก

### 698.5 เครื่องมือวิเคราะห์ Log ระดับมืออาชีพ: pgBadger

เมื่อ log สะสมมากขึ้นเรื่อย ๆ การไล่อ่านด้วยตาเปล่าไม่สะดวกอีกต่อไป **pgBadger** คือ
เครื่องมือ open-source ที่วิเคราะห์ PostgreSQL log แล้วสร้างรายงาน HTML สรุปภาพรวม:

```bash
# ติดตั้ง pgBadger (Ubuntu/Debian)
sudo apt install pgbadger

# วิเคราะห์ log file แล้วสร้างรายงาน HTML
pgbadger /var/log/postgresql/postgresql-16-main.log -o /var/www/html/pgbadger_report.html
```

รายงานที่ได้ครอบคลุม: query ที่ช้าที่สุด (top N), query ที่รันบ่อยที่สุด, กราฟ query
ต่อวินาทีตามช่วงเวลา, checkpoint activity, และ error ที่เกิดขึ้น — เหมาะกับการรัน
เป็น cron job รายวัน/รายสัปดาห์เพื่อดู trend ของระบบ production อย่างสม่ำเสมอ

---

## ขั้นตอนที่ 699: `pg_stat_statements` — extension สำหรับหา query ที่ช้าที่สุดในระบบจริง

### 699.1 ข้อจำกัดของ Slow Query Log ที่ `pg_stat_statements` มาแก้

Slow Query Log (ขั้นตอนที่ 698) มีข้อจำกัดสำคัญ: มันบันทึก **แต่ละครั้งที่ query รัน
แยกกัน** ถ้า query แบบเดียวกัน (ต่างแค่ค่า parameter) ถูกเรียก 10,000 ครั้งต่อวัน
โดยแต่ละครั้งใช้เวลา 50 ms (ต่ำกว่าเกณฑ์ log) คุณจะไม่เห็นมันใน log เลย ทั้งที่รวมแล้ว
query นี้กิน CPU/เวลารวมมากกว่า query ที่ช้าแค่ครั้งเดียว 800 ms เสียอีก

**`pg_stat_statements`** คือ extension ที่ **รวมสถิติของ query pattern เดียวกันเข้า
ด้วยกัน** (โดยมองข้ามค่า parameter ที่ต่างกัน) แล้วเก็บเป็นตัวเลขสะสม: จำนวนครั้งที่
เรียก, เวลารวมทั้งหมด, เวลาเฉลี่ยต่อครั้ง — ทำให้เห็นภาพรวมว่า query pattern ไหน
"กินทรัพยากรฐานข้อมูลมากที่สุด" จริง ๆ ในภาพรวมของทั้งระบบ

### 699.2 ติดตั้งและเปิดใช้งาน

แก้ไข `postgresql.conf`:

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
```

รีสตาร์ท PostgreSQL (จำเป็น เพราะเป็นการโหลด shared library ตอน start):

```bash
sudo systemctl restart postgresql
```

เปิดใช้งาน extension บนฐานข้อมูลของโปรเจกต์:

```sql
-- รันใน psql ที่เชื่อมต่อฐานข้อมูล blog_prod
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

หรือทำผ่าน Django migration เพื่อให้ทีมทุกคน/ทุก environment ได้ extension นี้
เหมือนกันอัตโนมัติ:

```python
# core/migrations/0001_enable_pg_stat_statements.py
from django.contrib.postgres.operations import CreateExtension
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = []

    operations = [
        CreateExtension("pg_stat_statements"),
    ]
```

### 699.3 Query หา Top 10 Query ที่ช้าที่สุด (เรียงตามเวลารวม)

```sql
SELECT
    query,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS mean_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percent_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```
                          query                           | calls | total_time_ms | mean_time_ms | percent_of_total
------------------------------------------------------------+-------+---------------+--------------+------------------
 SELECT * FROM blog_post WHERE title ILIKE $1                | 48213 |     412033.88 |         8.55 |             38.21
 SELECT * FROM blog_comment WHERE post_id = $1 ORDER BY ...  | 91042 |     287654.10 |         3.16 |             26.67
 SELECT COUNT(*) FROM blog_post WHERE is_published = $1      |  8841 |      95012.45 |        10.75 |              8.81
 UPDATE blog_post SET view_count = view_count + 1 WHERE id=$1| 152300|      68201.90 |         0.45 |              6.32
```

สังเกตว่าค่า parameter จริง (เช่น `title ILIKE $1`) ถูกแทนที่ด้วย `$1` — นี่คือหัวใจ
ของ `pg_stat_statements`: มันจัดกลุ่ม query ที่มี **โครงสร้างเดียวกัน** เข้าด้วยกัน
ไม่ว่าค่าที่ส่งเข้าไปจะต่างกันแค่ไหน

query แรกในตาราง (`title ILIKE $1`) แม้แต่ละครั้งใช้เวลาเฉลี่ยแค่ 8.55 ms (ดูเหมือน
ไม่ช้า) แต่เพราะถูกเรียกถึง 48,213 ครั้ง รวมแล้วกิน **38.21%** ของเวลาฐานข้อมูลทั้งหมด
— นี่คือประเภทของปัญหาที่ Slow Query Log (ขั้นตอนที่ 698) **มองไม่เห็นเลย** เพราะแต่ละ
ครั้งเร็วกว่าเกณฑ์ที่ตั้งไว้ (200 ms) มาก แต่ผลรวมกลับใหญ่ที่สุดในระบบ

### 699.4 หา Query ที่เรียกบ่อยที่สุด (อาจเป็นเป้าหมายของ caching แทน index)

```sql
SELECT
    query,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_time_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 5;
```

query ที่ถูกเรียกบ่อยมาก **แต่แต่ละครั้งเร็วอยู่แล้ว** (เช่น `UPDATE ... view_count`
ในตัวอย่างข้างต้น ที่ 152,300 ครั้ง แต่เฉลี่ยแค่ 0.45 ms) มักไม่ใช่เป้าหมายของการเพิ่ม
index แต่เป็นเป้าหมายของ **caching หรือ batching** แทน (ทบทวนเทคนิคจาก Part 068-069)
— นี่คือตัวอย่างที่ดีว่าทำไม Index Optimization (Part นี้) และ Caching (Part 068-069)
ต้องทำงานควบคู่กัน: `pg_stat_statements` ช่วยบอกว่า **query ไหนควรแก้ด้วยวิธีไหน**

### 699.5 เรียกใช้จาก Django management command เพื่อรายงานอัตโนมัติ

```python
# blog/management/commands/report_slow_queries.py
from django.core.management.base import BaseCommand
from django.db import connection


class Command(BaseCommand):
    help = "รายงาน 10 query ที่ใช้เวลารวมมากที่สุดจาก pg_stat_statements"

    def add_arguments(self, parser):
        parser.add_argument(
            "--limit", type=int, default=10, help="จำนวน query ที่จะแสดง"
        )
        parser.add_argument(
            "--reset", action="store_true", help="รีเซ็ตสถิติหลังแสดงผล"
        )

    def handle(self, *args, **options):
        with connection.cursor() as cursor:
            cursor.execute(
                """
                SELECT
                    query,
                    calls,
                    round(total_exec_time::numeric, 2) AS total_ms,
                    round(mean_exec_time::numeric, 2) AS mean_ms
                FROM pg_stat_statements
                WHERE query NOT ILIKE '%pg_stat_statements%'
                ORDER BY total_exec_time DESC
                LIMIT %s
                """,
                [options["limit"]],
            )
            rows = cursor.fetchall()

        self.stdout.write(self.style.SUCCESS(f"Top {len(rows)} query ที่ใช้เวลารวมมากที่สุด:"))
        for i, (query, calls, total_ms, mean_ms) in enumerate(rows, start=1):
            self.stdout.write(
                f"{i}. calls={calls} total={total_ms}ms mean={mean_ms}ms\n   {query[:120]}"
            )

        if options["reset"]:
            with connection.cursor() as cursor:
                cursor.execute("SELECT pg_stat_statements_reset()")
            self.stdout.write(self.style.WARNING("รีเซ็ตสถิติ pg_stat_statements แล้ว"))
```

```bash
python manage.py report_slow_queries --limit 5
```

### 699.6 ตารางสรุปเปรียบเทียบเครื่องมือทั้งสามของ Part นี้

| เครื่องมือ | ใช้เมื่อไหร่ | ให้ข้อมูลอะไร |
|---|---|---|
| `EXPLAIN ANALYZE` (ขั้นตอนที่ 693) | รู้อยู่แล้วว่า query ไหนน่าสงสัย ต้องการดูรายละเอียดการทำงาน | แผนการทำงานทีละ node, cost, เวลาจริง, index ที่ใช้ |
| Slow Query Log (ขั้นตอนที่ 698) | ยังไม่รู้ว่า query ไหนช้า ต้องการจับทุก query ที่ช้าเกินเกณฑ์แบบ real-time | SQL เต็มของแต่ละครั้งที่รันช้า พร้อมเวลาที่เกิดขึ้น |
| `pg_stat_statements` (ขั้นตอนที่ 699) | ต้องการภาพรวมว่า query pattern ไหน "กินทรัพยากรฐานข้อมูลรวม" มากที่สุด | สถิติสะสม: จำนวนครั้ง, เวลารวม, เวลาเฉลี่ย ต่อ query pattern |

ในงานจริงระดับมืออาชีพ ทั้งสามเครื่องมือทำงานร่วมกันเป็นขั้นตอน: ใช้
`pg_stat_statements` **หา** ว่า query pattern ไหนควรให้ความสำคัญก่อน (ขั้นตอนที่ 699)
ใช้ Slow Query Log **ยืนยัน** ว่าเกิดขึ้นจริงในช่วงเวลาไหนบ้าง (ขั้นตอนที่ 698) แล้วใช้
`EXPLAIN ANALYZE` **วิเคราะห์เชิงลึก** ว่าทำไมมันช้าและจะแก้อย่างไร (ขั้นตอนที่ 693)

---

## ขั้นตอนที่ 700: สรุปและแบบฝึกหัด — วิเคราะห์และ optimize query ของ blog project ด้วย EXPLAIN ANALYZE จริง

### 700.1 กรณีศึกษาแบบครบวงจร: optimize หน้า Category Listing ของบล็อก

มารวมทุกเทคนิคที่เรียนมาใน Part นี้เข้าด้วยกัน โดยเริ่มจาก view ที่ "ยังไม่ optimize"
ของหน้าแสดงโพสต์ตามหมวดหมู่ พร้อมค้นหาด้วยคำ:

```python
# blog/views.py — เวอร์ชันเริ่มต้น (ก่อน optimize)
from django.shortcuts import get_object_or_404, render

from .models import Category, Post


def category_detail(request, slug):
    category = get_object_or_404(Category, slug=slug)
    search = request.GET.get("q", "")

    posts = Post.objects.filter(category=category, is_published=True)
    if search:
        posts = posts.filter(title__icontains=search)
    posts = posts.order_by("-created_at")

    return render(
        request, "blog/category_detail.html",
        {"category": category, "posts": posts},
    )
```

**ขั้นที่ 1 — วัดผลก่อนแก้ไข ด้วย `EXPLAIN ANALYZE`**:

```python
from blog.models import Category, Post

cat = Category.objects.get(slug="django")
qs = Post.objects.filter(category=cat, is_published=True).order_by("-created_at")
print(qs.explain(analyze=True, buffers=True))
```

```
Sort  (cost=4821.44..4871.44 rows=20000 width=310) (actual time=38.201..38.550 rows=18402 loops=1)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 4102kB
  Buffers: shared hit=812
  ->  Seq Scan on blog_post  (cost=0.00..3245.00 rows=20000 width=310) (actual time=0.021..29.884 rows=18402 loops=1)
        Filter: ((category_id = 3) AND is_published)
        Rows Removed by Filter: 81598
        Buffers: shared hit=812
Planning Time: 0.203 ms
Execution Time: 39.012 ms
```

**การวินิจฉัย**: `Seq Scan` เต็มตาราง (อ่านทั้ง 100,000 แถวจนเจอ 18,402 แถวที่ตรง) บวก
`Sort` แยกต่างหากอีก 38 ms — สาเหตุคือยังไม่มี composite index ที่ครอบคลุมทั้ง
`category`, `is_published`, และ `created_at` พร้อมกัน

**ขั้นที่ 2 — เพิ่ม index ให้ตรงกับ pattern การ query จริง**:

```python
# blog/models.py
class Post(models.Model):
    # ... fields เดิมทั้งหมด ...

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(
                fields=["is_published", "-created_at"],
                name="post_published_created_idx",
            ),
            models.Index(
                fields=["category", "is_published", "-created_at"],
                name="post_cat_pub_created_idx",
            ),
        ]
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

**ขั้นที่ 3 — วัดผลซ้ำหลังเพิ่ม index**:

```python
print(qs.explain(analyze=True, buffers=True))
```

```
Index Scan using post_cat_pub_created_idx on blog_post  (cost=0.42..612.88 rows=18402 width=310) (actual time=0.034..2.112 rows=18402 loops=1)
  Index Cond: ((category_id = 3) AND (is_published = true))
  Buffers: shared hit=203
Planning Time: 0.187 ms
Execution Time: 2.301 ms
```

เวลาลดจาก **39.012 ms → 2.301 ms** (เร็วขึ้นประมาณ 17 เท่า) และจำนวน buffer ที่อ่าน
ลดจาก 812 เหลือ 203 — ยืนยันว่าการแก้ไขได้ผลจริงทั้งด้านเวลาและด้าน I/O

**ขั้นที่ 4 — แก้ปัญหา `title__icontains` ที่ index B-tree ธรรมดาช่วยไม่ได้**:

```python
print(qs.filter(title__icontains="django").explain(analyze=True))
```

```
Index Scan using post_cat_pub_created_idx on blog_post  (cost=0.42..612.88 rows=18402 width=310) (actual time=0.033..2.245 rows=18402 loops=1)
  Index Cond: ((category_id = 3) AND (is_published = true))
  Filter: (title ~~* '%django%'::text)
  Rows Removed by Filter: 18280
Execution Time: 2.401 ms
```

สังเกตว่า index ยังใช้กรอง `category`/`is_published` ได้เหมือนเดิม (เร็วเหมือนเดิม)
แต่ `title ~~* '%django%'` (แปลจาก `icontains`) ยังต้องทำ `Filter` แบบ Seq Scan บน
ผลลัพธ์ที่เหลือ 18,402 แถวอยู่ดี เพราะ B-tree index ใช้กับ pattern ที่มี wildcard
นำหน้า (`%django%`) ไม่ได้เลย — กรณีนี้ (ที่ผลลัพธ์หลังกรอง category/published เหลือ
ไม่มากแล้ว) การ `Filter` เพิ่มยังเร็วพอ (2.4 ms) แต่ถ้าต้องการค้นหาข้อความแบบเต็ม
ประสิทธิภาพสูงสุด ควรใช้ `GinIndex` ร่วมกับ `SearchVectorField` ตามที่แสดงในขั้นตอน
ที่ 697.2

### 700.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า Database Index คือโครงสร้าง B-tree ที่เร่งการค้นหาจาก O(n) เป็น O(log n)
  แลกกับต้นทุนด้าน write และพื้นที่ดิสก์
- ✅ รู้จักทางเลือกการสร้าง index ใน Django ทั้ง `db_index=True` และ `Meta.indexes`
  พร้อมเหตุผลว่าทำไมควรใช้ `Meta.indexes` เป็นค่าเริ่มต้นสำหรับโค้ดใหม่
- ✅ อ่านผลลัพธ์จาก `EXPLAIN ANALYZE` ได้ — แยกแยะ `Seq Scan`, `Index Scan`,
  `Index Only Scan`, `Bitmap Heap Scan` และตีความ `cost`, `actual time`, `Buffers`
- ✅ ออกแบบ Composite Index ตามกฎ Leftmost Prefix ให้ตรงกับ pattern การ query จริง
- ✅ ใช้ Partial Index เพื่อลดขนาด index สำหรับ column ที่มีค่ากระจายไม่สมดุล
- ✅ รู้ทันสัญญาณของ over-indexing และตรวจสอบ index ที่ไม่เคยถูกใช้ผ่าน
  `pg_stat_user_indexes`
- ✅ ใช้ `GinIndex` สำหรับ full-text search/JSON/Array และ `BrinIndex` สำหรับข้อมูล
  ขนาดใหญ่มากที่เรียงตามธรรมชาติ
- ✅ ตั้งค่า Slow Query Log และวิเคราะห์ด้วย pgBadger
- ✅ ใช้ `pg_stat_statements` หา query pattern ที่กินทรัพยากรฐานข้อมูลมากที่สุดใน
  ภาพรวมของทั้งระบบ
- ✅ ประยุกต์ทุกเทคนิคเข้าด้วยกันเพื่อ optimize query จริงของ blog project จาก
  39 ms เหลือ 2.3 ms

### 700.2.1 ตารางสรุปภาพรวมทั้ง Part

| ขั้นตอน | หัวข้อ | ผลลัพธ์ที่ได้ |
|---|---|---|
| 691 | แนวคิด B-tree Index | เข้าใจว่าทำไม index ถึงเร่งความเร็วได้จาก O(n) เป็น O(log n) |
| 692 | `Meta.indexes` เจาะลึก | สร้าง index ที่ตั้งชื่อเองได้ ควบคุมได้เต็มที่ |
| 693 | อ่าน `EXPLAIN ANALYZE` | วินิจฉัย query ได้ว่าช้าเพราะอะไรจากข้อมูลจริง ไม่ใช่การเดา |
| 694 | Composite Index | ลดจาก 42 ms เหลือ 0.1 ms ด้วย index ที่ตรงกับ pattern query |
| 695 | Partial Index | ลดขนาด index ลงมากสำหรับ column ที่มีค่ากระจายไม่สมดุล |
| 696 | Over-indexing | รู้จักต้นทุนด้าน write และวิธีตรวจหา index ที่ไม่จำเป็น |
| 697 | GinIndex/BrinIndex | เครื่องมือเฉพาะทางสำหรับ full-text search และข้อมูลขนาดมหาศาล |
| 698 | Slow Query Log | จับ query ที่ช้าแบบ real-time โดยไม่ต้องรู้ล่วงหน้าว่าตัวไหน |
| 699 | `pg_stat_statements` | เห็นภาพรวมว่า query pattern ไหนกินทรัพยากรฐานข้อมูลมากที่สุด |
| 700 | ประกอบร่างทั้งหมด | Optimize หน้า Category listing จริงจาก 39 ms เหลือ 2.3 ms |

### 700.3 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่าทำไม B-tree ทำให้การค้นหาเร็วขึ้นจาก O(n) เป็น O(log n)
- [ ] สร้าง composite index ผ่าน `Meta.indexes` พร้อมตั้งชื่อเองได้
- [ ] รัน `queryset.explain(analyze=True)` และอ่าน `Seq Scan`/`Index Scan`/`cost`/
      `actual time` ได้ถูกต้อง
- [ ] อธิบายกฎ Leftmost Prefix ของ composite index ได้ พร้อมยกตัวอย่าง query ที่ใช้
      index ได้/ใช้ไม่ได้
- [ ] สร้าง Partial Index ด้วย `condition=models.Q(...)` และรู้ข้อจำกัดของมัน
- [ ] ตรวจสอบ index ที่ไม่เคยถูกใช้ผ่าน `pg_stat_user_indexes`
- [ ] เข้าใจความแตกต่างระหว่าง `GinIndex` และ `BrinIndex` ว่าเหมาะกับข้อมูลแบบไหน
- [ ] เปิดใช้งาน `log_min_duration_statement` และอ่าน slow query log ได้
- [ ] เปิดใช้งาน `pg_stat_statements` และเขียน query หา top slow query ได้
- [ ] วัดผล query ของ blog project ก่อน/หลังเพิ่ม index ด้วยตัวเลขจริงจาก
      `EXPLAIN ANALYZE`

### 700.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: ใช้ management command `seed_demo_data` จาก Part 067 เพิ่มจำนวน
โพสต์เป็น 10,000 บทความ (ปรับ loop `range(1, 51)` เป็น `range(1, 10001)`) แล้วรัน
`EXPLAIN ANALYZE` กับ query `Post.objects.filter(is_published=True).order_by
("-created_at")[:10]` ทั้งก่อนและหลังเพิ่ม `models.Index(fields=["is_published",
"-created_at"], ...)` บันทึกตัวเลข `Execution Time` ทั้งสองกรณีเปรียบเทียบกัน

**แบบฝึกหัดที่ 2**: ออกแบบและเพิ่ม Partial Index สำหรับ query
`Comment.objects.filter(is_approved=False).order_by("-created_at")` (หน้า admin queue
ตรวจสอบคอมเมนต์ที่ยังไม่อนุมัติ) แล้วยืนยันด้วย `EXPLAIN ANALYZE` ว่า PostgreSQL เลือก
ใช้ partial index ที่สร้างขึ้นจริง (สังเกตชื่อ index ใน `Index Scan using ...`)

**แบบฝึกหัดที่ 3**: เปิดใช้งาน `pg_stat_statements` บนฐานข้อมูลพัฒนาของคุณ รัน
management command `seed_demo_data` ตามด้วยการเรียก view หน้า homepage และหน้า
category detail หลาย ๆ ครั้งผ่าน `python manage.py shell` (หรือ `curl` ถ้ารัน dev
server) แล้วเขียน query SQL ที่แสดง top 5 query ที่ใช้เวลารวมมากที่สุด พร้อมวิเคราะห์
ว่าแต่ละตัวควรแก้ด้วย index เพิ่ม หรือควรแก้ด้วย caching (ทบทวนจาก Part 068-069)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ใช้ `pg_stat_user_indexes` ตรวจสอบ index ทั้งหมดของ
ตาราง `blog_post` และ `blog_comment` ในฐานข้อมูลของคุณ หา index ที่ `idx_scan = 0`
อย่างน้อย 1 ตัว (ถ้าไม่มี ให้จงใจสร้าง index ที่ไม่มี query ไหนใช้ขึ้นมา 1 ตัว) แล้ว
วัดเวลาที่ใช้ `bulk_create()` ข้อมูล 5,000 แถว เปรียบเทียบระหว่างตอนที่มี index ที่ไม่
จำเป็นนี้อยู่ กับตอนที่ลบออกไปแล้ว บันทึกผลต่างของเวลาที่วัดได้จริง

### 700.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรสร้าง index ตั้งแต่ตอนออกแบบโมเดลเลย หรือรอให้มีปัญหาจริงก่อนค่อยเพิ่ม?**
A: สำหรับ index ที่ Django สร้างให้อัตโนมัติ (primary key, unique, ForeignKey) ไม่ต้อง
ทำอะไรเพิ่มอยู่แล้ว ส่วน index แบบกำหนดเอง (`Meta.indexes`) ควรตั้งไว้ตั้งแต่ต้นเฉพาะ
กรณีที่**รู้ pattern การ query ล่วงหน้าอย่างชัดเจน** (เช่น รู้แน่ว่าหน้า homepage
ต้อง filter `is_published` + order `created_at` เสมอ) ส่วน index ที่ไม่แน่ใจว่าจำเป็น
จริงหรือไม่ ควรรอวัดผลจริงด้วย `EXPLAIN ANALYZE`/`pg_stat_statements` ก่อนเสมอ
ตามหลักการทองคำในขั้นตอนที่ 696.4

**Q: ทำไม `EXPLAIN ANALYZE` บางครั้งแสดง `Seq Scan` ทั้งที่มี index อยู่แล้วบน column
นั้น?**
A: PostgreSQL query planner **เลือกแผนที่คิดว่าเร็วที่สุด** ไม่ใช่แผนที่ "มี index"
เสมอไป ถ้าตารางมีข้อมูลน้อย (Seq Scan ทั้งตารางเร็วอยู่แล้ว) หรือ query กรองได้แถว
ส่วนใหญ่ของตาราง (เช่น มากกว่า ~20-30% ของทั้งหมด) planner มักเลือก Seq Scan เพราะ
การอ่านข้อมูลต่อเนื่อง (sequential I/O) เร็วกว่าการกระโดดไปมาผ่าน index (random I/O)
เมื่อต้องอ่านสัดส่วนที่สูงอยู่ดี — นี่ไม่ใช่บั๊ก แต่เป็นการตัดสินใจที่ถูกต้องแล้ว

**Q: ต้องรัน `ANALYZE` เองหรือไม่ หลังจากเพิ่ม index หรือ insert ข้อมูลจำนวนมาก?**
A: โดยปกติ PostgreSQL มี **autovacuum** ที่รัน `ANALYZE` ให้อัตโนมัติเป็นระยะเมื่อ
ตารางมีการเปลี่ยนแปลงข้อมูลมากพอ แต่หลัง `bulk_create()`/`bulk_update()` ข้อมูลจำนวน
มากในคราวเดียว (เช่นในแบบฝึกหัดของ Part นี้) สถิติของ planner อาจยังไม่อัปเดตทัน
แนะนำให้รัน `ANALYZE blog_post;` เองใน `psql` ทันทีหลัง seed ข้อมูลจำนวนมาก เพื่อให้
`EXPLAIN ANALYZE` แสดงผลที่แม่นยำและ planner เลือกแผนที่เหมาะสมที่สุด

**Q: index ที่สร้างใน Django ทำงานเหมือนกันทุกฐานข้อมูล (PostgreSQL, MySQL, SQLite)
หรือไม่?**
A: `models.Index` แบบพื้นฐาน (B-tree ธรรมดา รวมถึง composite index) ทำงานได้บนทุก
ฐานข้อมูลหลักที่ Django รองรับ แต่ฟีเจอร์ขั้นสูงบางอย่างเป็น PostgreSQL-specific
ล้วน ๆ: Partial Index (`condition=`) ใช้ได้กับ PostgreSQL และ SQLite เท่านั้น (MySQL
ไม่รองรับ) ส่วน `GinIndex`/`BrinIndex`/`GistIndex`/`HashIndex` จาก
`django.contrib.postgres.indexes` ใช้ได้กับ PostgreSQL เท่านั้นตามชื่อ module —
นี่คือเหตุผลเพิ่มเติมที่หลักสูตรนี้ยึด PostgreSQL เป็นมาตรฐานตลอดทั้งหลักสูตรตั้งแต่
Phase 2 เพราะฟีเจอร์ระดับ production จำนวนมากพึ่งพาความสามารถเฉพาะของมัน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 071: Pagination และ Large Dataset Handling** จะพาคุณไปแก้ปัญหาที่เกิดขึ้นเมื่อ
แม้แต่ query ที่มี index ที่ดีที่สุดแล้ว ก็ยังไม่สามารถส่งข้อมูลนับล้านแถวกลับไปให้
ผู้ใช้ดูพร้อมกันในหน้าเดียวได้ คุณจะได้เรียนรู้ตั้งแต่ `Paginator` พื้นฐานของ Django,
ปัญหาของ **Offset Pagination** เมื่อข้อมูลมีปริมาณมาก (ที่จริง ๆ แล้วเกี่ยวข้องโดยตรง
กับสิ่งที่เรียนใน Part นี้ — ลอง `EXPLAIN ANALYZE` query ที่มี `OFFSET 500000` ดูว่า
เกิดอะไรขึ้น), ไปจนถึง **Cursor-based Pagination** ที่สเกลได้จริงสำหรับ dataset
ขนาดใหญ่ระดับ production เตรียมข้อมูลตัวอย่างจำนวนมาก (จากแบบฝึกหัดที่ 1 ของ Part นี้)
ไว้ให้พร้อม เพราะเราจะใช้มันสาธิตปัญหาของ pagination แบบเดิมกันต่อทันที!
