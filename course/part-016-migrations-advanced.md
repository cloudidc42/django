# Part 016: Database Migrations ขั้นสูงและการจัดการ Schema

> **ขั้นตอนที่ 151-160 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> ตลอด Part 011-015 คุณใช้ `makemigrations`/`migrate` มาตลอดในฐานะ "คำสั่งสองคำสั่งที่
> ต้องรันคู่กันเสมอ" และรู้วิธีอ่านไฟล์ migration ที่ Django สร้างให้ทีละบรรทัดแล้ว
> แต่สิ่งที่ยังไม่เคยพูดถึงเลยคือ: จะทำอย่างไรเมื่อต้อง **แปลงข้อมูลเก่าที่มีอยู่จริง**
> ไม่ใช่แค่เปลี่ยนโครงสร้างตาราง (เช่น Post เก่านับพันแถวที่ยังไม่มี slug), จะทำอย่างไร
> เมื่อโฟลเดอร์ `migrations/` มีไฟล์เป็นร้อยจนเปิดตามไม่ทัน, จะทำอย่างไรเมื่อสอง
> feature branch สร้าง migration ชนกันแบบที่ `--merge` แก้ไม่ตรงจุด, และที่สำคัญที่สุด
> — จะรัน `ALTER TABLE` บนตารางที่มีข้อมูลจริงหลายล้านแถวใน production **โดยไม่ทำเว็บ
> ล่ม** ได้อย่างไร Part นี้คือคำตอบของคำถามเหล่านั้นทั้งหมด เมื่อจบ Part นี้ คุณจะเขียน
> data migration ได้เอง, squash migration ได้อย่างปลอดภัย, ออกแบบ migration แบบ
> zero-downtime ได้ตามหลัก Expand-and-Contract, และเข้าใจความแตกต่างระหว่าง "สถานะที่
> Django คิดว่าฐานข้อมูลเป็น" กับ "สถานะที่ฐานข้อมูลเป็นจริง" อย่างถ่องแท้

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 151: Data Migrations ด้วย `RunPython` — Backfill ข้อมูลเก่าที่ขาดหาย
- ขั้นตอนที่ 152: `squashmigrations` — รวมไฟล์ Migration หลายสิบไฟล์เป็นไฟล์เดียว
- ขั้นตอนที่ 153: Reversible Migrations, `RunPython.noop`, และการเขียน Reverse Function
- ขั้นตอนที่ 154: Custom Migration Operations: `AddIndex`, `RemoveIndex`, `RunSQL`
- ขั้นตอนที่ 155: การจัดการ Migration Conflict ในทีมเจาะลึก — เมื่อ `--merge` ไม่พอ
- ขั้นตอนที่ 156: Zero-Downtime Migration สำหรับ Production — Expand-and-Contract Pattern
- ขั้นตอนที่ 157: การทดสอบ Migration — Data Migration Test และการอ่าน `sqlmigrate`
- ขั้นตอนที่ 158: Migration State vs Database State — `--fake`, `--fake-initial`
- ขั้นตอนที่ 159: การจัดการ Dependency ระหว่าง Migration ของหลายแอป
- ขั้นตอนที่ 160: สรุปและแบบฝึกหัด — เขียน Data Migration จริงที่ Backfill Slug ทุกตัว

---

## ขั้นตอนที่ 151: Data Migrations ด้วย `RunPython` — Backfill ข้อมูลเก่าที่ขาดหาย

### 151.1 Schema Migration vs Data Migration

ตลอด Part 011-015 ทุก migration ที่คุณสร้างเป็น **Schema Migration** — migration ที่
เปลี่ยน**โครงสร้าง**ของตาราง (`CREATE TABLE`, `ADD COLUMN`, `ADD CONSTRAINT` ฯลฯ)
โดยไม่แตะ**ข้อมูล**ที่มีอยู่แล้วเลย (นอกจากเติมค่า default ให้อัตโนมัติเมื่อจำเป็น)

**Data Migration** คือ migration อีกประเภทที่ทำหน้าที่ตรงข้าม: มัน **ไม่เปลี่ยนโครงสร้าง
ตารางเลยแม้แต่คอลัมน์เดียว** แต่เปลี่ยน**เนื้อหาข้อมูล**ที่มีอยู่แล้วในตาราง เช่น:

| สถานการณ์ | ทำไมต้องเป็น Data Migration |
|---|---|
| `Post` เก่า 5,000 แถวที่สร้างก่อนมี field `slug` ยังมีค่าเป็น `''` อยู่ | ต้อง generate slug ให้แถวเก่าทุกแถว ไม่ใช่แค่เพิ่มคอลัมน์เฉย ๆ |
| เปลี่ยน `is_premium` (boolean) เป็น `price_tier` (choices: `free`/`basic`/`pro`) | ต้องแปลง `True`→`'pro'`, `False`→`'free'` ให้ทุกแถวตามกฎธุรกิจ |
| นำเข้าข้อมูลอ้างอิงคงที่ (เช่น รายชื่อจังหวัดของไทย 77 จังหวัด) ตอนสร้างโปรเจกต์ | ต้องมีข้อมูลตั้งต้นอยู่เสมอไม่ว่าจะ deploy ที่ไหน |
| ย้ายข้อมูลจากตารางกลาง M2M อัตโนมัติไปยัง `through` model ใหม่ (ตามที่ Part 012 เกริ่นไว้) | โครงสร้างตารางเปลี่ยน แต่ข้อมูลเดิมต้องถูกคัดลอกข้ามไปด้วย ไม่ใช่หายไปเฉย ๆ |

ทั้งสองประเภทเป็นไฟล์ Python ในโฟลเดอร์ `migrations/` เหมือนกันทุกประการ ต่างกันแค่
ชนิดของ **Operation** ที่อยู่ใน `operations = [...]`: Schema migration ใช้
`AddField`/`AlterField`/`CreateModel` ฯลฯ ส่วน Data migration ใช้ **`RunPython`**
(หรือ `RunSQL` ที่จะเรียนในขั้นตอนที่ 154) เป็นหลัก

### 151.2 สถานการณ์จำลอง: `Post` เก่าที่ยังไม่มี Slug

ทบทวนจาก Part 015 (ขั้นตอนที่ 150.1): `Post.save()` ปัจจุบันจะ auto-generate `slug`
ให้ก็ต่อเมื่อ `self.slug` ยังว่างอยู่ตอนเรียก `save()`:

```python
# blog/models.py (ทบทวนจาก Part 015)
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
```

โค้ดนี้ใช้งานได้ดีกับ **`Post` ที่สร้างใหม่ผ่าน Django ORM เท่านั้น** แต่ในโลกจริง
ข้อมูลเข้าฐานข้อมูลได้หลายทาง: อาจมีการ import ข้อมูลจากระบบเก่าด้วย `bulk_create()`
(ซึ่ง**ไม่เรียก** `save()` — จะอธิบายเหตุผลใน Part 059), อาจมีแถวที่ import ผ่าน SQL
ตรง ๆ ตอนย้ายระบบ, หรืออาจเป็นข้อมูลที่สร้างไว้ตั้งแต่ก่อนที่ทีมจะเพิ่ม logic การ
auto-generate slug เข้ามาในโค้ด สมมติว่าคุณตรวจสอบฐานข้อมูล production แล้วพบว่า:

```python
>>> from blog.models import Post
>>> Post.objects.filter(slug='').count()
1284
```

มี `Post` ถึง 1,284 แถวที่ `slug` ยังว่างอยู่ — ปัญหานี้ **แก้ด้วย schema migration
ไม่ได้เลย** เพราะคอลัมน์ `slug` มีอยู่แล้ว ไม่มีอะไรต้องเปลี่ยนโครงสร้าง สิ่งที่ต้องทำ
คือ "เขียนโปรแกรมไปรันครั้งเดียวเพื่อเติมค่าที่ขาดหายไป" — นี่คือหน้าที่ของ
Data Migration โดยเฉพาะ

### 151.3 สร้างไฟล์ Migration เปล่าด้วย `makemigrations --empty`

Django มีคำสั่งพิเศษสำหรับสร้างไฟล์ migration เปล่า (ไม่ต้องรอให้ `models.py` เปลี่ยน
ก่อนถึงจะสร้างได้ เหมือน schema migration ทั่วไป):

```bash
python manage.py makemigrations blog --empty --name backfill_post_slug
```

ผลลัพธ์ที่ได้:

```
Migrations for 'blog':
  blog/migrations/0016_backfill_post_slug.py
```

เปิดไฟล์ที่ได้ขึ้นมา จะเห็นโครงสร้างเปล่า ๆ พร้อมให้เติม:

```python
# blog/migrations/0016_backfill_post_slug.py
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0015_post_view_count_non_negative_and_more'),
    ]

    operations = [
    ]
```

### 151.4 `apps.get_model()`: หัวใจสำคัญของทุก Data Migration

ก่อนเขียนโค้ด backfill ต้องเข้าใจกฎที่สำคัญที่สุดของ data migration ก่อน: **ห้าม
`import` model จาก `models.py` ตรง ๆ ใน migration เด็ดขาด**

```python
# ❌ ห้ามทำแบบนี้ใน migration
from blog.models import Post

def backfill_slug(apps, schema_editor):
    for post in Post.objects.filter(slug=''):   # ผิด!
        ...
```

```python
# ✅ ใช้ apps.get_model() เสมอ
def backfill_slug(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')
    for post in Post.objects.filter(slug=''):
        ...
```

เหตุผลคือ `RunPython` function ทุกตัวจะได้รับ argument `apps` ที่เป็น
**`django.apps.registry.Apps`** ชนิดพิเศษที่เรียกว่า **Historical Model Registry**
— มันไม่ใช่ app registry ปัจจุบันของโปรเจกต์ แต่เป็น **ภาพจำลองของ model ณ จุดเวลานั้น
ในประวัติ migration** (สร้างขึ้นจากการอ่าน `operations` ของทุก migration ที่มาก่อนหน้า
migration นี้ตามลำดับ)

| ประเด็น | `from blog.models import Post` | `apps.get_model('blog', 'Post')` |
|---|---|---|
| ได้ field อะไรบ้าง | field ปัจจุบันทั้งหมดใน `models.py` วันนี้ | เฉพาะ field ที่มีอยู่ ณ จุดของ migration นั้นในประวัติ |
| ได้ custom method (`increment_view_count()`, `is_free` ฯลฯ) ไหม | ✅ ได้ครบ | ❌ ไม่ได้เลย — เป็น "โครงกระดูก" ที่มีแค่ field/relation เท่านั้น |
| ได้ custom manager (`PostManager`, `.published()`) ไหม | ✅ ได้ | ❌ ไม่ได้ — ได้แค่ `objects = models.Manager()` มาตรฐาน |
| override `save()` ทำงานไหม | ✅ ทำงาน | ❌ **ไม่ทำงาน** — `save()` เป็นของ `models.Model` เปล่า ๆ |
| ปลอดภัยเมื่อ `models.py` เปลี่ยนในอนาคต | ❌ เสี่ยง — ถ้าลบ field ทิ้งภายหลัง migration เก่าจะ error ทันที | ✅ ปลอดภัยเสมอ — อ้างอิงตามประวัติ ไม่ใช่ปัจจุบัน |

> **ทำไมเรื่องนี้ถึงสำคัญมาก**: ลองจินตนาการว่าอีก 6 เดือนข้างหน้า ทีมงานลบ field
> `excerpt` ทิ้งจาก `Post` (ตาม Expand-and-Contract Pattern ที่จะเรียนในขั้นตอนที่ 156)
> ถ้า migration เก่าที่เขียนไว้ตอนนี้ `import Post from blog.models` ตรง ๆ และอ้างถึง
> `post.excerpt` การรัน migration นั้นซ้ำบนฐานข้อมูลใหม่ (เช่นตอน deploy ขึ้นเซิร์ฟเวอร์
> ทดสอบใหม่ที่ต้องรัน migration ทุกไฟล์ตั้งแต่ต้น) จะ **error ทันที** เพราะ `models.py`
> ปัจจุบันไม่มี `excerpt` แล้ว แต่ถ้าใช้ `apps.get_model()` migration นั้นจะยังคงอ้างถึง
> `Post` เวอร์ชัน ณ ตอนที่มันถูกเขียน (ซึ่งยังมี `excerpt` อยู่) และรันผ่านได้เสมอ
> ไม่ว่าเวลาจะผ่านไปนานแค่ไหน — นี่คือเหตุผลที่ระบบ migration ของ Django ถูกออกแบบให้
> "ประวัติศาสตร์ไม่เปลี่ยนแปลง" (immutable history)

### 151.5 เขียน Data Migration Backfill Slug ฉบับสมบูรณ์

```python
# blog/migrations/0016_backfill_post_slug.py
from django.db import migrations
from django.utils.text import slugify


def backfill_post_slug(apps, schema_editor):
    """เติม slug ให้ Post ทุกแถวที่ยังว่างอยู่ ใช้ logic เดียวกับ Post.save() ปัจจุบัน"""
    Post = apps.get_model('blog', 'Post')
    db_alias = schema_editor.connection.alias

    posts_without_slug = Post.objects.using(db_alias).filter(slug='').order_by('pk')
    for post in posts_without_slug.iterator():
        base_slug = slugify(post.title) or f'post-{post.pk}'
        slug = base_slug
        counter = 1
        # ทำซ้ำ logic กันชนกันเองแบบเดียวกับใน Post.save() แต่เขียนใหม่ที่นี่
        # เพราะ historical model ไม่มี method save() ที่ override ไว้ให้เรียกใช้ได้
        while (
            Post.objects.using(db_alias)
            .filter(category=post.category, slug=slug)
            .exclude(pk=post.pk)
            .exists()
        ):
            slug = f'{base_slug}-{counter}'
            counter += 1
        post.slug = slug
        post.save(update_fields=['slug'])


def reverse_backfill_post_slug(apps, schema_editor):
    """ย้อนกลับ: ไม่มีทางรู้ว่า slug ไหนถูกสร้างโดย migration นี้ จึงเลือกไม่ทำอะไร"""
    pass


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0015_post_view_count_non_negative_and_more'),
    ]

    operations = [
        migrations.RunPython(backfill_post_slug, reverse_backfill_post_slug),
    ]
```

รัน migration:

```bash
python manage.py migrate blog
```

```
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0016_backfill_post_slug... OK
```

ตรวจสอบผลลัพธ์:

```python
>>> from blog.models import Post
>>> Post.objects.filter(slug='').count()
0
```

### 151.6 จุดสังเกตสำคัญในโค้ดข้างต้น

- **`schema_editor.connection.alias`**: data migration ควรรองรับกรณีที่โปรเจกต์มี
  หลายฐานข้อมูล (multi-database routing — เรียนเต็มรูปแบบใน Part 073) การดึง
  `db_alias` แล้วส่งต่อด้วย `.using(db_alias)` ทุกครั้งทำให้ migration ทำงานถูก
  ฐานข้อมูลเสมอ แม้จะมีมากกว่าหนึ่งฐานข้อมูลในโปรเจกต์
- **`.iterator()`**: เมื่อ backfill ตารางที่มีข้อมูลจำนวนมาก (หลักหมื่น-แสนแถว) การ
  ใช้ `.iterator()` แทนการ evaluate ทั้ง QuerySet เป็น list ในทีเดียว ช่วยประหยัด
  หน่วยความจำอย่างมาก เพราะ Django จะดึงข้อมูลเป็น batch ทีละส่วนจากฐานข้อมูลแทนที่จะ
  โหลดทั้งหมดเข้า memory พร้อมกัน (เจาะลึกเรื่องนี้ใน Part 067 และการแบ่ง batch สำหรับ
  ตารางขนาดใหญ่มากใน ขั้นตอนที่ 156.5 ของ Part นี้)
- **`post.save(update_fields=['slug'])`**: บันทึกเฉพาะคอลัมน์ `slug` แทนที่จะ save
  ทั้งแถว ลดโอกาสเขียนทับข้อมูล field อื่นที่อาจถูกแก้ไขพร้อมกันจากที่อื่น (เทคนิคที่
  เรียนไปแล้วใน Part 011 ขั้นตอนที่ 106.3)
- **`reverse_backfill_post_slug` ที่ทำแค่ `pass`**: เพราะการ "ย้อนกลับ" การเติม slug
  หมายความว่าต้องรู้ว่า slug ไหนถูกสร้างจาก migration นี้กับ slug ไหนมีอยู่ก่อนแล้ว
  ซึ่งเป็นข้อมูลที่ **ไม่ได้ถูกเก็บไว้เลย** — การเขียน reverse function ที่ "ทำอะไร
  บางอย่างแต่ไม่ถูกต้อง" อันตรายกว่าการเขียนให้ไม่ทำอะไรเลยอย่างตรงไปตรงมา รายละเอียด
  เรื่องนี้จะเจาะลึกในขั้นตอนที่ 153

### 151.7 โบนัส: ทำตามสัญญาจาก Part 012 — ย้ายข้อมูลข้ามไปยัง `through` Model

ใน Part 012 (ขั้นตอนที่ 114.3) เราเกริ่นไว้ว่าถ้าโปรเจกต์เปลี่ยนจาก `ManyToManyField`
แบบอัตโนมัติไปใช้ `through=` model ทีหลัง โดยที่ตารางกลางเดิมมีข้อมูลอยู่แล้ว
Django จะ**ไม่ย้ายข้อมูลให้อัตโนมัติ** เพราะโครงสร้างตารางเปลี่ยนไปสิ้นเชิง (ตารางใหม่
มีคอลัมน์เพิ่มอย่าง `added_at`/`added_by`) และสัญญาไว้ว่าจะสอนวิธีแก้ใน Part นี้ —
มาดูกันว่าทำอย่างไร

```python
# blog/migrations/0017_migrate_post_tags_to_through.py
from django.db import migrations


def copy_m2m_to_through(apps, schema_editor):
    """
    คัดลอกข้อมูลจากตารางกลางอัตโนมัติเดิม (ก่อนเพิ่ม through=) ไปยัง PostTag
    ต้องรันหลัง CreateModel(PostTag) แต่ก่อน AlterField ที่เปลี่ยน tags ให้ใช้ through=
    """
    Post = apps.get_model('blog', 'Post')
    PostTag = apps.get_model('blog', 'PostTag')
    db_alias = schema_editor.connection.alias

    # ตอนนี้ Post.tags ยังเป็น M2M อัตโนมัติอยู่ (migration นี้รันก่อนเปลี่ยน through=)
    # จึงยัง .all() ผ่านตารางกลางเดิมได้ตามปกติ
    for post in Post.objects.using(db_alias).all().iterator():
        for tag in post.tags.all():
            PostTag.objects.using(db_alias).get_or_create(post=post, tag=tag)


def reverse_copy(apps, schema_editor):
    PostTag = apps.get_model('blog', 'PostTag')
    PostTag.objects.using(schema_editor.connection.alias).all().delete()


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0016_backfill_post_slug'),
        ('blog', '0017_create_posttag_model'),  # migration ที่สร้าง PostTag ไว้แล้ว
    ]

    operations = [
        migrations.RunPython(copy_m2m_to_through, reverse_copy),
    ]
```

> **ลำดับที่ถูกต้อง**: ต้องแบ่งเป็น **3 migration แยกกัน** เสมอ (1) สร้าง `PostTag`
> model ก่อนด้วย `CreateModel` (ยังไม่แตะ `Post.tags`) (2) รัน `RunPython` คัดลอก
> ข้อมูลตามโค้ดข้างต้น (3) ค่อยเปลี่ยน `Post.tags` ให้มี `through='PostTag'` ด้วย
> `AlterField` เป็น migration สุดท้าย ถ้าทำทั้งสามอย่างในไฟล์เดียวหรือสลับลำดับ
> ตารางกลางเดิมจะถูกลบไปพร้อมกับการเปลี่ยน `through=` ก่อนที่ data migration จะทัน
> คัดลอกข้อมูลออกมาได้ทัน — หลักการแบ่งเป็นหลายขั้นแบบนี้คือแก่นของ Expand-and-Contract
> Pattern ที่จะเจาะลึกเต็มรูปแบบในขั้นตอนที่ 156

---

## ขั้นตอนที่ 152: `squashmigrations` — รวมไฟล์ Migration หลายสิบไฟล์เป็นไฟล์เดียว

### 152.1 ปัญหาที่เกิดขึ้นเมื่อโปรเจกต์โตขึ้น

ลองนึกภาพโปรเจกต์ที่พัฒนามาแล้ว 2 ปี แอป `blog` อาจมีไฟล์ migration มากถึง 60-80 ไฟล์
สะสมจากการเพิ่ม field ทีละนิด, เปลี่ยนชื่อ field, เพิ่ม constraint, แก้ index ฯลฯ
ตลอดหลาย Part ที่ผ่านมา นี่คือปัญหาที่เกิดขึ้นจริงเมื่อไฟล์เยอะเกินไป:

| ปัญหา | ผลกระทบ |
|---|---|
| `python manage.py migrate` ช้าลง | ต้อง apply/ตรวจสอบทีละไฟล์ตามลำดับ graph ทั้งหมดตอน deploy ครั้งแรกบนเซิร์ฟเวอร์ใหม่ |
| ยากต่อการทำความเข้าใจ schema ปัจจุบัน | ต้องไล่อ่าน 80 ไฟล์เพื่อรู้ว่า `Post` model ตอนนี้หน้าตาเป็นอย่างไรกันแน่ |
| Migration graph ซับซ้อนเกินจำเป็น | dependency พันกันไปมาระหว่างแอป ทำให้ debug conflict ยากขึ้น |
| โฟลเดอร์ `migrations/` รกและหาไฟล์ยาก | นักพัฒนาใหม่ที่เข้าร่วมทีมงงว่าไฟล์ไหนสำคัญ ไฟล์ไหน "ประวัติศาสตร์" ที่ไม่ต้องสนใจแล้ว |

`squashmigrations` คือคำสั่งที่ Django มีให้เพื่อแก้ปัญหานี้โดยเฉพาะ: มันจะ**อ่านไฟล์
migration หลายไฟล์ในช่วงที่กำหนด แล้วสร้างไฟล์ใหม่เพียงไฟล์เดียวที่ให้ผลลัพธ์ปลายทาง
เหมือนเดิมทุกประการ** โดยไม่ต้องรันไฟล์เก่าทีละไฟล์อีกต่อไป

### 152.2 รันคำสั่ง `squashmigrations`

```bash
python manage.py squashmigrations blog 0001 0010
```

คำสั่งนี้บอกให้ Django รวม migration ของแอป `blog` ตั้งแต่ `0001` ถึง `0010` เข้าด้วยกัน
ผลลัพธ์:

```
Will squash the following migrations:
 - 0001_initial
 - 0002_post_content_type_post_excerpt_post_is_premium_and_more
 - 0003_category_post_category
 ...
 - 0010_alter_post_options

Do you wish to proceed? [y/N] y
Optimizing...
  Optimized from 34 operations to 11 operations.
Created new squashed migration /path/to/blog/migrations/0001_squashed_0010_alter_post_options.py
  You should commit this migration but leave the old ones in place;
  the new migration will be used for new installs. Once you are sure
  all instances of the codebase have applied the migrations you squash,
  you can delete them.
```

สังเกตข้อความ **"Optimized from 34 operations to 11 operations"** — Django ไม่ได้แค่
"เอาโค้ดมาต่อกัน" แต่มี **migration optimizer** ที่ฉลาดพอจะยุบ operation ที่ไม่จำเป็น
ทิ้งได้ เช่น ถ้า migration `0003` เพิ่ม field `foo` แล้ว migration `0007` มา
`AlterField` เปลี่ยน `max_length` ของ `foo` อีกที optimizer จะรวมสองขั้นตอนนี้เป็น
`AddField` เดียวที่มีค่า `max_length` สุดท้ายเลย โดยไม่ต้องสร้างแล้วแก้ทีหลังอีก

### 152.3 เปิดไฟล์ Squashed Migration มาดู

```python
# blog/migrations/0001_squashed_0010_alter_post_options.py
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):

    replaces = [
        ('blog', '0001_initial'),
        ('blog', '0002_post_content_type_post_excerpt_post_is_premium_and_more'),
        ('blog', '0003_category_post_category'),
        # ... รายชื่อ migration เดิมทั้งหมดที่ถูกแทนที่ ...
        ('blog', '0010_alter_post_options'),
    ]

    initial = True

    dependencies = [
        ('auth', '0012_alter_user_first_name_max_length'),
    ]

    operations = [
        migrations.CreateModel(
            name='Category',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False)),
                ('name', models.CharField(max_length=100)),
                ('slug', models.SlugField(max_length=100, unique=True)),
            ],
            options={'ordering': ['name'], 'verbose_name': 'หมวดหมู่'},
        ),
        # ... operations ที่เหลือที่ optimizer รวมให้แล้ว ...
    ]
```

attribute ที่สำคัญที่สุดในไฟล์นี้คือ **`replaces`** — list ของ `(app_label, name)`
ที่บอก Django ว่า "ไฟล์นี้ทำหน้าที่แทนไฟล์เหล่านี้ทั้งหมด" นี่คือกลไกที่ทำให้
squashed migration ทำงานถูกต้องกับทั้งฐานข้อมูลใหม่และฐานข้อมูลเก่าที่เคย apply
migration ต้นฉบับไปแล้ว (อธิบายเพิ่มในขั้นตอนที่ 152.4)

### 152.4 ทดสอบ Squashed Migration ให้ครบทั้ง 2 สถานการณ์

นี่คือขั้นตอนที่มือใหม่มักข้ามไป แต่**จำเป็นมาก**เพราะ squashed migration ต้องทำงาน
ถูกต้องใน **สองสถานการณ์ที่ต่างกันโดยสิ้นเชิง**:

| สถานการณ์ | สิ่งที่ Django ทำ |
|---|---|
| **ฐานข้อมูลใหม่เอี่ยม** (เช่น เซิร์ฟเวอร์ทดสอบใหม่, เครื่องนักพัฒนาใหม่ที่เพิ่ง clone) | ใช้ squashed migration ตัวเดียวรันเลย ไม่แตะไฟล์เก่าที่ถูก `replaces` เลย |
| **ฐานข้อมูลเดิมที่เคย `migrate` ไฟล์ต้นฉบับ (0001-0010) ไปแล้ว** | Django เห็นว่า migration เดิมถูก apply ไปแล้วในตาราง `django_migrations` จึง **ข้าม** squashed migration ไปเลย (ถือว่า "เทียบเท่ากับที่ apply ไปแล้ว") |

ทดสอบสถานการณ์แรกด้วยฐานข้อมูลว่างเปล่า:

```bash
# ลบฐานข้อมูลทดสอบทิ้ง แล้วสร้างใหม่ (ตัวอย่างสำหรับ SQLite)
rm db.sqlite3
python manage.py migrate
# ควรเห็น squashed migration ถูก apply โดยไม่มี error ใด ๆ
```

ทดสอบสถานการณ์ที่สองด้วยฐานข้อมูลที่มี migration เดิม apply ไปแล้ว (เช่น สำเนา
ฐานข้อมูล staging):

```bash
python manage.py showmigrations blog
```

```
blog
 [X] 0001_initial
 [X] 0002_post_content_type_post_excerpt_post_is_premium_and_more
 ...
 [X] 0010_alter_post_options
 [X] 0001_squashed_0010_alter_post_options
```

สังเกตว่า Django แสดง squashed migration เป็น `[X]` (apply แล้ว) ทันทีโดยไม่ต้องรัน
จริง เพราะมันตรวจพบว่า migration ทุกตัวที่อยู่ใน `replaces` ถูก apply ไปแล้วครบถ้วน

> **กฎเหล็กของหลักสูตรนี้**: **ห้าม** squash migration ที่ยังไม่ได้ deploy ขึ้น
> production หรือ environment อื่นที่ทีมใช้ร่วมกันครบทุกที่ ถ้ามี environment ไหนยัง
> apply migration ต้นฉบับไม่ครบ (เช่น เพิ่งสร้าง environment ใหม่กลางทาง หรือมี branch
> ที่ยังไม่ merge) การ squash จะทำให้ migration graph ของ environment นั้นสับสน

### 152.5 ลบไฟล์เก่าที่ถูกแทนที่ (เมื่อมั่นใจว่าปลอดภัยแล้วเท่านั้น)

หลังจากยืนยันแล้วว่า **ทุก environment** (production, staging, เครื่องนักพัฒนาทุกคน,
CI/CD) apply migration ต้นฉบับที่ถูก squash ไปแล้วครบถ้วน (ไม่มีที่ไหนค้างอยู่ระหว่าง
กลางอีกต่อไป) จึงจะปลอดภัยที่จะลบไฟล์เก่าและลบ attribute `replaces` ออก:

```bash
# ลบไฟล์ 0001-0010 เดิมทั้งหมดที่อยู่ใน replaces ของไฟล์ squashed
rm blog/migrations/0001_initial.py
rm blog/migrations/0002_post_content_type_post_excerpt_post_is_premium_and_more.py
# ... ลบไฟล์ที่เหลือทั้งหมดที่อยู่ใน replaces ...
```

จากนั้นแก้ไขไฟล์ squashed migration โดยลบ attribute `replaces` ทิ้ง (Django แนะนำให้
ทำด้วยมือ ไม่มีคำสั่งอัตโนมัติสำหรับขั้นตอนนี้):

```python
class Migration(migrations.Migration):

    # ลบบรรทัด replaces = [...] ทิ้งไปเลย เพราะไม่มีไฟล์เก่าให้ "แทนที่" อีกต่อไปแล้ว

    initial = True
    dependencies = [
        ('auth', '0012_alter_user_first_name_max_length'),
    ]
    operations = [
        # ... เหมือนเดิม ...
    ]
```

> **ทำไมต้องรอ**: ถ้าลบไฟล์เก่าทิ้งเร็วเกินไป (ก่อนที่ทุก environment จะ apply
> squashed migration) environment ที่ยังไม่ทันอัปเดตจะหาไฟล์ migration เดิมที่มันรู้
> จักไม่เจอ (เพราะไฟล์ถูกลบไปแล้วใน codebase ที่มันเพิ่ง pull มา) ทำให้ Django
> `NodeNotFoundError` ทันที — ไม่มีทางแก้นอกจากต้อง revert การลบไฟล์กลับมา

### 152.6 ข้อจำกัดสำคัญของ `squashmigrations` ที่ต้องรู้ไว้

| ข้อจำกัด | รายละเอียด |
|---|---|
| `RunPython`/`RunSQL` ไม่ถูก optimize ทิ้งอัตโนมัติ | Optimizer รวม operation ประเภท schema (`AddField`, `AlterField` ฯลฯ) ได้ดีมาก แต่ **ไม่กล้าลบหรือรวม `RunPython`/`RunSQL`** เพราะไม่รู้ว่าโค้ด Python/SQL ข้างในทำอะไรบ้าง อาจมี side effect ที่สำคัญ |
| Data migration ที่ squash แล้วอาจรันซ้ำ logic เดิมโดยไม่จำเป็น | ถ้า data migration เดิม backfill ข้อมูลไปแล้ว การ squash รวมไฟล์เข้าด้วยกันไม่ได้แปลว่า logic นั้นจะไม่ถูกรันซ้ำบนฐานข้อมูลใหม่ ต้องตรวจสอบด้วยมือว่ายัง sensible อยู่ |
| ต้องรัน `makemigrations` ใหม่ได้ตามปกติหลัง squash | Squashed migration ไม่ใช่ "จุดสิ้นสุด" ของ migration graph เพิ่ม field ใหม่ต่อจากนี้ได้ตามปกติ เพียงแต่ dependency จะชี้ไปที่ squashed migration แทนไฟล์เดิม |
| ควรทำเป็นระยะ ไม่ใช่ครั้งเดียวตอนจบโปรเจกต์ | ทีมมืออาชีพมักตั้งกฎ squash ทุก ๆ 6-12 เดือน หรือทุกครั้งที่ major release เพื่อไม่ให้ไฟล์สะสมมากเกินไปตั้งแต่แรก |

---

## ขั้นตอนที่ 153: Reversible Migrations, `RunPython.noop`, และการเขียน Reverse Function

### 153.1 ทำไม Migration ต้อง Reversible

ทบทวนจาก Part 011 (ขั้นตอนที่ 105.5): `python manage.py migrate blog 0009` คือคำสั่ง
ที่ **ย้อนกลับ (rollback)** ไปยัง migration ที่ระบุ Django รองรับการย้อนกลับนี้ได้กับ
schema operation เกือบทั้งหมดโดยอัตโนมัติ เพราะ operation อย่าง `AddField` มี
"ปฏิบัติการตรงข้าม" ที่ชัดเจนอยู่แล้วในตัว (`AddField` ย้อนกลับคือ `RemoveField`,
`CreateModel` ย้อนกลับคือ `DeleteModel` ฯลฯ) Django รู้วิธีย้อนกลับ operation เหล่านี้
เองโดยอัตโนมัติทั้งหมดโดยไม่ต้องเขียนอะไรเพิ่ม

แต่ **`RunPython` แตกต่างออกไปโดยสิ้นเชิง**: มันคือโค้ด Python ที่คุณเขียนขึ้นเอง
Django **ไม่มีทางรู้ได้เลย**ว่าโค้ดของคุณทำอะไร จึงไม่มีทางสร้าง "ปฏิบัติการตรงข้าม"
ให้อัตโนมัติได้ — คุณต้อง**เขียนเอง**ทุกครั้ง

### 153.2 Syntax ของ `RunPython`: `code` และ `reverse_code`

```python
migrations.RunPython(code, reverse_code=None, atomic=None, elidable=False)
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `code` | ฟังก์ชันที่จะรันตอน `migrate` เดินหน้า (forward) — **บังคับต้องมี** |
| `reverse_code` | ฟังก์ชันที่จะรันตอน `migrate` ย้อนกลับ (backward) — ไม่บังคับ แต่ถ้าไม่ใส่ migration นี้จะ **irreversible** |
| `atomic` | บังคับให้ operation นี้อยู่ใน transaction หรือไม่ (ปกติสืบทอดจากค่า `atomic` ของทั้ง Migration class) |
| `elidable` | ถ้า `True` migration optimizer (จากขั้นตอนที่ 152) จะกล้าตัด operation นี้ทิ้งได้ตอน squash ถ้าเห็นว่าไม่จำเป็นแล้ว |

### 153.3 `RunPython.noop`: เมื่อไม่ต้องการทำอะไรตอนย้อนกลับ

Django มี shortcut พิเศษชื่อ **`migrations.RunPython.noop`** สำหรับกรณีที่ต้องการ
"ย้อนกลับแบบไม่ทำอะไรเลย" (no operation) โดยไม่ต้องเขียนฟังก์ชันเปล่า ๆ เอง:

```python
from django.db import migrations


def seed_default_categories(apps, schema_editor):
    Category = apps.get_model('blog', 'Category')
    db_alias = schema_editor.connection.alias
    default_names = ['เทคโนโลยี', 'ไลฟ์สไตล์', 'ท่องเที่ยว', 'อาหาร']
    for name in default_names:
        Category.objects.using(db_alias).get_or_create(
            name=name, defaults={'slug': name.lower().replace(' ', '-')}
        )


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0016_backfill_post_slug'),
    ]

    operations = [
        migrations.RunPython(seed_default_categories, migrations.RunPython.noop),
    ]
```

`RunPython.noop` เหมาะกับกรณีที่การย้อนกลับ **ไม่มีความหมายที่ชัดเจน** หรือ **ไม่คุ้ม
ความเสี่ยง** เช่นตัวอย่างข้างต้น: ถ้าลบ `Category` ที่ seed ไว้ตอน rollback แล้วมี
`Post` ผูกอยู่กับ `Category` เหล่านั้นแล้ว (ผ่าน `on_delete=SET_NULL` ตาม Part 012)
การลบจะทำให้ `Post.category` กลายเป็น `None` ทันที ซึ่งอาจไม่ใช่สิ่งที่ต้องการเลย
การเลือก "ไม่ทำอะไร" จึงปลอดภัยกว่าการพยายามลบแล้วพลาด

### 153.4 เขียน Reverse Function ที่ทำงานจริง

ไม่ใช่ทุก data migration ที่ควรใช้ `noop` เสมอไป บางกรณีการย้อนกลับมีความหมายชัดเจน
และควรเขียนให้ทำงานจริง เช่น migration ที่แปลงค่า field แบบหนึ่งไปอีกแบบหนึ่ง:

```python
# blog/migrations/0018_convert_is_premium_to_price_tier.py
from django.db import migrations


def forward_convert(apps, schema_editor):
    """แปลง is_premium (boolean เดิม) ให้เป็น price_tier (choices ใหม่)"""
    Post = apps.get_model('blog', 'Post')
    db_alias = schema_editor.connection.alias
    Post.objects.using(db_alias).filter(is_premium=True).update(price_tier='pro')
    Post.objects.using(db_alias).filter(is_premium=False).update(price_tier='free')


def reverse_convert(apps, schema_editor):
    """ย้อนกลับ: price_tier ใด ๆ ที่ไม่ใช่ 'free' ถือเป็น is_premium=True"""
    Post = apps.get_model('blog', 'Post')
    db_alias = schema_editor.connection.alias
    Post.objects.using(db_alias).exclude(price_tier='free').update(is_premium=True)
    Post.objects.using(db_alias).filter(price_tier='free').update(is_premium=False)


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0017_post_price_tier'),
    ]

    operations = [
        migrations.RunPython(forward_convert, reverse_convert),
    ]
```

สังเกตว่า `reverse_convert` **ไม่ได้คืนค่าให้เหมือนเดิม 100%** เสมอไป (ถ้ามี
`price_tier='basic'` การย้อนกลับจะตีความเป็น `is_premium=True` ซึ่งอาจไม่ตรงกับ
ความตั้งใจเดิมทุกกรณี) — นี่คือธรรมชาติของ data migration ย้อนกลับหลายกรณี: **มันคือ
การประมาณที่ดีที่สุดเท่าที่ทำได้ ไม่ใช่การย้อนเวลากลับไปแบบสมบูรณ์แบบเสมอไป** สิ่งสำคัญ
คือต้องเขียนคอมเมนต์อธิบายข้อจำกัดนี้ไว้ให้ชัดเจนเสมอ เพื่อให้คนที่ต้อง rollback ในอนาคต
เข้าใจว่าจะเกิดอะไรขึ้นกับข้อมูลจริง

### 153.5 `IrreversibleError`: เมื่อพยายามย้อนกลับ Migration ที่ย้อนไม่ได้

ถ้า `RunPython` ไม่ได้ระบุ `reverse_code` เลย (ปล่อยเป็นค่า default `None`) migration
นั้นจะกลาย "irreversible" ทันที — ถ้ามีใครพยายาม `migrate` ย้อนกลับผ่านมันไป จะได้
error ทันที:

```bash
python manage.py migrate blog 0015
```

```
django.db.migrations.exceptions.IrreversibleError: Migration
blog.0016_backfill_post_slug is not reversible
```

> **กฎของหลักสูตรนี้**: ทุกครั้งที่เขียน `RunPython` ให้ถามตัวเองก่อนเสมอว่า "ถ้าต้อง
> rollback migration นี้ในสถานการณ์ฉุกเฉินตอนตีสองที่ production จะเกิดอะไรขึ้น"
> ถ้าคำตอบคือ "ไม่รู้ /ไม่มีความหมาย" ให้ใช้ `RunPython.noop` อย่างตั้งใจ (ดีกว่าปล่อย
> ว่างแล้วลืมไปว่าทำไมถึงย้อนกลับไม่ได้) ถ้าคำตอบคือ "มีวิธีย้อนกลับที่สมเหตุสมผล"
> ให้เขียน `reverse_code` จริงพร้อมคอมเมนต์อธิบายข้อจำกัดตามตัวอย่างขั้นตอนที่ 153.4

### 153.6 `elidable=True`: บอก Optimizer ว่า Operation นี้ตัดทิ้งได้เมื่อ Squash

ทบทวนจากขั้นตอนที่ 152.6 ว่า optimizer ไม่กล้าตัด `RunPython` ทิ้งเองตามปกติ แต่ถ้า
คุณรู้แน่ชัดว่า data migration ตัวใดตัวหนึ่ง**ไม่มีความหมายอีกต่อไปหลัง squash**
(เช่น migration ที่ backfill ค่า default ให้ field ที่ภายหลังถูกลบทิ้งไปแล้วทั้งคอลัมน์)
สามารถระบุ `elidable=True` เพื่ออนุญาตให้ optimizer ตัดทิ้งได้อย่างปลอดภัย:

```python
migrations.RunPython(
    backfill_legacy_field,
    migrations.RunPython.noop,
    elidable=True,   # ถ้า squash ในอนาคตและ optimizer เห็นว่าตัดทิ้งได้ ก็ตัดทิ้งได้เลย
)
```

ใช้ `elidable=True` อย่างระมัดระวังเสมอ เพราะเป็นการบอก Django อย่างชัดเจนว่า "operation
นี้ไม่สำคัญพอที่ต้องเก็บประวัติไว้" — เหมาะกับ data migration ที่ backfill ค่าชั่วคราว
เท่านั้น ไม่เหมาะกับ data migration ที่ import ข้อมูลอ้างอิงสำคัญที่ต้องมีอยู่เสมอ

### 153.7 ตารางสรุป Reversibility ของแต่ละ Operation

| Operation | Reversible โดยอัตโนมัติ | หมายเหตุ |
|---|---|---|
| `AddField` / `RemoveField` | ✅ | Django สร้างปฏิบัติการตรงข้ามให้เอง |
| `CreateModel` / `DeleteModel` | ✅ | เช่นกัน |
| `AlterField` / `RenameField` | ✅ | ย้อนกลับได้เสมอ (โครงสร้าง ไม่ใช่ข้อมูล) |
| `AddIndex` / `RemoveIndex` | ✅ | เจาะลึกในขั้นตอนที่ 154 |
| `RunSQL` | ⚠️ ต้องระบุ `reverse_sql` เอง | เหมือน `RunPython` ทุกประการ |
| `RunPython` | ❌ ต้องเขียน `reverse_code` เอง | หัวข้อหลักของขั้นตอนนี้ |

---

## ขั้นตอนที่ 154: Custom Migration Operations: `AddIndex`, `RemoveIndex`, `RunSQL`

### 154.1 ทบทวน `Meta.indexes` จาก Part 015

ใน Part 015 (ขั้นตอนที่ 141.6) คุณเพิ่ม `Meta.indexes` ให้ `Post`:

```python
class Post(TimeStampedModel):
    # ... field ทั้งหมด ...
    class Meta(TimeStampedModel.Meta):
        indexes = [
            models.Index(fields=["is_published", "-created_at"], name="post_published_idx"),
        ]
```

เมื่อรัน `makemigrations` Django สร้าง operation `AddIndex` ให้อัตโนมัติ:

```python
migrations.AddIndex(
    model_name='post',
    index=models.Index(fields=['is_published', '-created_at'], name='post_published_idx'),
),
```

นี่คือวิธี "มาตรฐาน" ในการเพิ่ม index ผ่าน ORM — เขียนใน `Meta.indexes` แล้วให้
`makemigrations` สร้าง `AddIndex` ให้เอง ในกรณีส่วนใหญ่วิธีนี้เพียงพอแล้ว

### 154.2 เมื่อไหร่ที่ ORM operation แบบมาตรฐานไม่พอ

มีบางสถานการณ์ที่การเขียน `Meta.indexes` แล้วปล่อยให้ `makemigrations` จัดการเองไม่
เพียงพอ:

| สถานการณ์ | ปัญหาของ ORM operation มาตรฐาน |
|---|---|
| สร้าง index บนตารางขนาดใหญ่มากใน PostgreSQL production | `CREATE INDEX` แบบปกติจะ **lock ตารางทั้งตารางระหว่างสร้าง index** (ไม่รับ write ระหว่างนั้น) ต้องใช้ `CREATE INDEX CONCURRENTLY` แทน ซึ่ง Django ORM operation มาตรฐานไม่รองรับโดยตรง |
| ใช้ database extension เฉพาะของ PostgreSQL (เช่น `pg_trgm` สำหรับ fuzzy search) | ต้องรัน `CREATE EXTENSION` ก่อน ซึ่งไม่มี ORM operation ให้ใช้ |
| สร้าง Partial Index ที่มีเงื่อนไขซับซ้อนเกินกว่า `models.Index(condition=...)` รองรับ | บาง syntax เฉพาะฐานข้อมูลที่ Django abstraction ยังไม่ครอบคลุม |
| สร้าง Database Trigger หรือ Stored Procedure | Django ไม่มี ORM concept สำหรับสิ่งเหล่านี้เลย |

ทั้งหมดนี้คือสถานการณ์ที่ต้องใช้ **`RunSQL`** — operation ที่ให้เขียน SQL ดิบ ๆ ตรง ๆ
ลงในไฟล์ migration แทนการพึ่ง ORM abstraction

### 154.3 `RunSQL` Syntax และตัวอย่างพื้นฐาน

```python
migrations.RunSQL(sql, reverse_sql=None, state_operations=None, hints=None, elidable=False)
```

ตัวอย่างเพิ่ม index ด้วย `RunSQL` ธรรมดา (ยังไม่ concurrent):

```python
# blog/migrations/0019_add_title_index_raw_sql.py
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0018_convert_is_premium_to_price_tier'),
    ]

    operations = [
        migrations.RunSQL(
            sql='CREATE INDEX post_title_lower_idx ON blog_post (LOWER(title));',
            reverse_sql='DROP INDEX post_title_lower_idx;',
        ),
    ]
```

ตัวอย่างนี้สร้าง **functional index** (index บนผลลัพธ์ของฟังก์ชัน `LOWER(title)`
ไม่ใช่บนคอลัมน์ `title` ตรง ๆ) เพื่อให้ query ที่ค้นหาแบบไม่สนตัวพิมพ์เล็ก-ใหญ่
(`title__iexact`, `title__icontains`) เร็วขึ้นมาก — สิ่งนี้ Django ORM's `models.Index`
มาตรฐานทำไม่ได้โดยตรง (ต้องใช้ `django.contrib.postgres.indexes` เพิ่มเติมสำหรับ
PostgreSQL โดยเฉพาะ ซึ่งเรียนเต็มรูปแบบใน Part 070)

### 154.4 `CREATE INDEX CONCURRENTLY` บน PostgreSQL: กรณีใช้งานจริงที่สำคัญที่สุด

นี่คือกรณีการใช้ `RunSQL` ที่สำคัญที่สุดสำหรับทีมงานที่ทำงานกับตาราง production
ขนาดใหญ่จริง ๆ:

```python
# blog/migrations/0020_add_concurrent_index.py
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0019_add_title_index_raw_sql'),
    ]

    # สำคัญมาก: CREATE INDEX CONCURRENTLY ใช้ใน transaction ไม่ได้เลยใน PostgreSQL
    # ต้องปิด atomic ของ migration นี้ทั้งไฟล์
    atomic = False

    operations = [
        migrations.RunSQL(
            sql='CREATE INDEX CONCURRENTLY IF NOT EXISTS post_category_idx '
                'ON blog_post (category_id);',
            reverse_sql='DROP INDEX CONCURRENTLY IF EXISTS post_category_idx;',
        ),
    ]
```

จุดสำคัญที่พลาดไม่ได้คือ **`atomic = False`** ที่ระดับ `class Migration` (ไม่ใช่ที่
`RunSQL` เฉย ๆ) เพราะ PostgreSQL มีกฎว่า `CREATE INDEX CONCURRENTLY` **ห้ามรันอยู่
ภายใน transaction block เด็ดขาด** (จะได้ error `CREATE INDEX CONCURRENTLY cannot run
inside a transaction block` ทันที) แต่ปกติ Django จะห่อทุก migration ด้วย
transaction ให้อัตโนมัติเสมอ (ตามที่เรียนใน Part 011 ขั้นตอนที่ 105.4) การตั้ง
`atomic = False` คือการบอก Django ว่า "migration ไฟล์นี้ไม่ต้องการ transaction
ครอบ ปล่อยให้รันตรง ๆ"

> **ข้อควรรู้เกี่ยวกับ Django สำหรับ `models.Index` ยุคใหม่**: ตั้งแต่ Django 4.2
> เป็นต้นมา `models.Index` มีพารามิเตอร์ `db_tablespace` และรองรับการส่ง
> `django.contrib.postgres.operations.AddIndexConcurrently` สำหรับ PostgreSQL
> โดยเฉพาะ ซึ่งเป็นทางเลือกที่ปลอดภัยกว่าการเขียน `RunSQL` ดิบ ๆ เอง (จัดการ
> `atomic = False` ให้อัตโนมัติในตัวมันเองแล้ว) — เราจะกลับมาเจาะลึกการใช้งาน
> PostgreSQL-specific operations เหล่านี้อย่างเต็มรูปแบบใน **Part 070 (PostgreSQL
> ขั้นสูงสำหรับ Django)** ตอนนี้ขอให้เข้าใจหลักการเบื้องหลัง (`atomic = False`,
> `CONCURRENTLY`) ให้แน่นก่อน เพราะเป็นความรู้พื้นฐานที่ใช้ได้ไม่ว่าจะเขียนผ่าน
> `RunSQL` ตรง ๆ หรือผ่าน operation สำเร็จรูปในอนาคต

### 154.5 `RunSQL.noop` และ `state_operations`

เหมือนกับ `RunPython.noop` ใน `RunSQL` ก็มี `RunSQL.noop` ให้ใช้เมื่อไม่ต้องการทำอะไร
ตอนย้อนกลับเช่นกัน:

```python
migrations.RunSQL(
    sql='CREATE EXTENSION IF NOT EXISTS pg_trgm;',
    reverse_sql=migrations.RunSQL.noop,  # ไม่ DROP EXTENSION ตอน rollback (อาจมีคนอื่นใช้อยู่)
)
```

พารามิเตอร์ `state_operations` ใช้เมื่อ SQL ที่รันจริงกับ **"migration state"**
(ที่ Django ใช้ติดตามว่า model หน้าตาเป็นอย่างไร ณ จุดนั้น) ไม่ตรงกัน เช่น ถ้าใช้
`RunSQL` สร้างคอลัมน์ใหม่ด้วย SQL ดิบ (แทนที่จะใช้ `AddField`) จำเป็นต้องบอก Django
ด้วยว่า "state" ของ model ควรมี field ใหม่นี้ด้วย ไม่เช่นนั้น `makemigrations` ครั้ง
ถัดไปจะสับสนคิดว่ายังไม่มี field นี้:

```python
migrations.RunSQL(
    sql='ALTER TABLE blog_post ADD COLUMN internal_notes TEXT DEFAULT \'\';',
    reverse_sql='ALTER TABLE blog_post DROP COLUMN internal_notes;',
    state_operations=[
        migrations.AddField(
            model_name='post',
            name='internal_notes',
            field=models.TextField(default='', blank=True),
        ),
    ],
)
```

รูปแบบนี้พบไม่บ่อยนัก (ส่วนใหญ่ใช้ `AddField` ตรง ๆ ก็เพียงพอ) แต่สำคัญมากเมื่อต้อง
ใช้ SQL feature เฉพาะฐานข้อมูลที่ ORM ยังไม่รองรับ ร่วมกับต้องการให้ migration state
ยังคง sync กับ `models.py` อยู่เสมอ

### 154.6 ตารางสรุป: เมื่อไหร่ใช้ ORM Operation เมื่อไหร่ใช้ `RunSQL`

| สถานการณ์ | ทางเลือกที่แนะนำ |
|---|---|
| เพิ่ม/ลบ index ปกติทั่วไปบนตารางขนาดเล็ก-กลาง | `Meta.indexes` + `makemigrations` (ได้ `AddIndex` อัตโนมัติ) |
| สร้าง index บนตารางใหญ่มากใน production ที่ต้องไม่ lock | `RunSQL` + `CREATE INDEX CONCURRENTLY` + `atomic = False` |
| Composite constraint/index มาตรฐาน | `Meta.constraints`/`Meta.indexes` (Part 015) |
| Database extension, trigger, stored procedure, functional index | `RunSQL` เท่านั้น — ไม่มีทางเลือกอื่นใน ORM มาตรฐาน |
| Feature เฉพาะ PostgreSQL ที่ Django มี operation สำเร็จรูปให้แล้ว | `django.contrib.postgres.operations.*` (เจาะลึกใน Part 070) |

---

## ขั้นตอนที่ 155: การจัดการ Migration Conflict ในทีมเจาะลึก — เมื่อ `--merge` ไม่พอ

### 155.1 ทบทวนสั้น ๆ จาก Part 011

Part 011 (ขั้นตอนที่ 109.4) แนะนำสถานการณ์พื้นฐาน: นักพัฒนา A และ B ต่างรัน
`makemigrations` บน branch ของตัวเองพร้อมกัน ได้ migration เลขเดียวกันคนละไฟล์
(เช่น `0006_add_author_field.py` กับ `0006_add_tags_field.py`) แก้ด้วย:

```bash
python manage.py makemigrations --merge
```

นี่แก้ปัญหาได้ในกรณีที่ **A และ B แก้ field คนละตัวกัน** (เพิ่ม `author` กับเพิ่ม
`tags` ไม่ชนกันเลย) merge migration ที่ได้จะมี `dependencies` ชี้ไปทั้งสองสาขา และ
ไม่มีการสูญเสียข้อมูลหรือ operation ใด ๆ

### 155.2 สถานการณ์ที่ `--merge` "รันผ่านได้" แต่ผลลัพธ์ผิด

ปัญหาที่อันตรายกว่ามากคือกรณีที่ A และ B **แก้ field เดียวกัน** แต่คนละแบบ สมมติว่า:

- นักพัฒนา A แก้ `Post.title` จาก `max_length=200` เป็น `max_length=255` บน branch
  `feature/longer-titles`
- นักพัฒนา B แก้ `Post.title` จาก `max_length=200` เป็น `max_length=150` บน branch
  `feature/shorter-titles-for-seo` (เพื่อจำกัดความยาวสำหรับ SEO) **ในเวลาไล่เลี่ยกัน**

ทั้งสองต่างรัน `makemigrations` ได้ migration คนละไฟล์ที่มี `AlterField` เปลี่ยน
`max_length` คนละค่า เมื่อ merge โค้ดเข้าด้วยกันแล้วรัน `makemigrations --merge`:

```bash
python manage.py makemigrations --merge
```

```
Merging blog
  Branch 0007_alter_post_title_max_length_255
    - Alter field title on post
  Branch 0007_alter_post_title_max_length_150
    - Alter field title on post

Merging will only work if the operations printed above do not conflict
with each other (working on different fields or models)
Do you want to merge these migration branches? [y/N]
```

**สังเกตว่า Django เตือนเองตรง ๆ**: "จะ merge ได้ผลดีก็ต่อเมื่อ operation ที่พิมพ์ออกมา
ไม่ชนกัน (ทำงานกับ field/model คนละตัวกัน)" — ในกรณีนี้ **ทั้งสอง operation ทำงานกับ
field เดียวกัน (`title`) ซึ่งชนกันโดยตรง** ถ้ากด `y` ต่อไป Django จะสร้าง merge
migration ที่มี `dependencies` ชี้ไปทั้งสองสาขา แต่ **ไม่มี operation ของตัวเองเลย**
(migration แบบนี้เรียกว่า "empty merge migration") — ผลลัพธ์สุดท้ายที่ฐานข้อมูลจะได้
รับขึ้นอยู่กับ **ลำดับที่ operation ทั้งสองถูก apply จริง** ซึ่งมักจะเป็นค่าจาก
operation ที่มาทีหลังตาม timestamp ของไฟล์ ไม่ใช่ตามเจตนาของทีม — นี่คือสาเหตุที่
`max_length` สุดท้ายอาจกลายเป็น 150 หรือ 255 แบบสุ่ม ๆ ขึ้นกับว่าใคร push ก่อนหลัง
โดยที่ไม่มีใครในทีมตั้งใจแบบนั้นเลย

### 155.3 วิธีแก้ที่ถูกต้อง: ตรวจสอบเนื้อหา Merge Migration ก่อนเสมอ

**อย่ากด `y` ที่คำถาม merge โดยไม่อ่านให้เข้าใจก่อน** ขั้นตอนที่ปลอดภัยคือ:

1. เปิดไฟล์ migration ทั้งสองสาขามาอ่านเนื้อหาจริงก่อน (ไม่ใช่แค่ดูชื่อไฟล์)
2. ถ้าพบว่า operation ชนกันจริง (แก้ field เดียวกันคนละค่า) **คุยกับทีมก่อนเสมอ**
   ว่าค่าไหนคือค่าที่ถูกต้องจริง ๆ ที่ควรใช้ (255 หรือ 150? หรือค่าอื่นที่เป็น
   ข้อสรุปร่วมกันใหม่?)
3. แก้ `models.py` ให้เป็นค่าที่ทีมตกลงกันแล้วค่าเดียว แล้ว **ลบ migration ทั้งสอง
   ไฟล์ที่ชนกันทิ้ง** (ทั้งคู่ ไม่ใช่แค่ไฟล์ใดไฟล์หนึ่ง)
4. รัน `makemigrations` ใหม่อีกครั้งจาก `models.py` ที่แก้ไขแล้ว จะได้ migration
   ไฟล์เดียวที่สะอาด ไม่มีความกำกวมใด ๆ

```bash
# ลบไฟล์ที่ชนกันทั้งสองไฟล์ทิ้ง
rm blog/migrations/0007_alter_post_title_max_length_255.py
rm blog/migrations/0007_alter_post_title_max_length_150.py

# แก้ models.py ให้เป็นค่าที่ทีมตกลงร่วมกัน (สมมติว่าตกลงกันที่ 220)
# title = models.CharField(max_length=220)

python manage.py makemigrations blog
```

```
Migrations for 'blog':
  blog/migrations/0007_alter_post_title.py
    - Alter field title on post
```

วิธีนี้ได้ migration graph ที่สะอาด เป็นเส้นตรง ไม่มี merge migration เปล่า ๆ
ค้างอยู่ในประวัติ และสำคัญที่สุดคือ **ค่าสุดท้ายที่ได้มาจากการตัดสินใจของทีมจริง ๆ**
ไม่ใช่ผลบังเอิญจากลำดับการ apply

### 155.4 เมื่อไหร่ที่ต้องเก็บ Merge Migration ไว้ (ไม่ลบ)

ถ้า operation ของทั้งสองสาขา **ไม่ชนกันจริง ๆ** (ทำงานกับ field/model คนละตัว
เหมือนตัวอย่างพื้นฐานใน Part 011) การปล่อยให้ `--merge` สร้างไฟล์ merge migration
ไว้ตามปกติเป็นวิธีที่ถูกต้องแล้ว **ไม่ต้องลบทิ้ง** — เก็บไฟล์ merge migration ไว้เป็น
หลักฐานในประวัติว่า ณ จุดนี้มีสองสาขาการพัฒนาถูกรวมเข้าด้วยกัน

### 155.5 แนวทางป้องกันไม่ให้เกิด Conflict ตั้งแต่แรก

| แนวทาง | รายละเอียด |
|---|---|
| Pull/rebase จาก `main` บ่อย ๆ ระหว่างพัฒนา | ยิ่งเห็น migration ใหม่ของเพื่อนร่วมทีมเร็ว ยิ่งลด window ที่จะชนกัน |
| ใช้ `python manage.py makemigrations --check --dry-run` ใน CI (Part 011 ขั้นตอนที่ 104.3) | จับกรณีลืมสร้าง migration ก่อน merge เข้า `main` |
| กำหนดเจ้าของ (owner) ของแต่ละ model ในทีมใหญ่ | ลด "การแก้ field เดียวกันพร้อมกัน" ตั้งแต่ต้น ผ่านการสื่อสารว่าใครแตะโมเดลไหนอยู่ |
| ทำ migration ให้เล็กและ commit บ่อย ๆ | migration เล็ก ๆ ชนกันน้อยกว่าและตรวจสอบง่ายกว่า migration ก้อนใหญ่ที่รวมหลายการเปลี่ยนแปลง |
| Review migration file ในทุก Pull Request เหมือน review โค้ดปกติ | migration คือโค้ดที่กระทบข้อมูลจริง ควรถูกตรวจสอบอย่างจริงจังไม่น้อยไปกว่าโค้ด Python อื่น |

> **กฎเหล็กของหลักสูตรนี้**: ทุกครั้งที่เห็นข้อความ "Merging will only work if the
> operations printed above do not conflict with each other" ให้อ่าน operation ที่
> Django พิมพ์ออกมาให้ครบทุกบรรทัดก่อนตอบคำถามเสมอ นี่ไม่ใช่ข้อความเตือนเชิงพิธีการ
> แต่เป็นคำเตือนจริงจังว่าอาจมีข้อมูลหรือ schema ผิดพลาดเกิดขึ้นถ้าตอบผิด

---

## ขั้นตอนที่ 156: Zero-Downtime Migration สำหรับ Production — Expand-and-Contract Pattern

### 156.1 ปัญหา: ทำไม `ALTER TABLE` ธรรมดาถึงอันตรายบนตารางใหญ่

ทบทวนจาก Part 011 (ขั้นตอนที่ 105.4): `sqlmigrate` แสดง SQL จริงที่จะรัน และคุณเห็น
แล้วว่าการเพิ่มคอลัมน์ที่มี `default` จะรันเป็น `ADD COLUMN ... DEFAULT x NOT NULL`
ตามด้วย `ALTER COLUMN ... DROP DEFAULT` สิ่งที่ Part 011 ยังไม่ได้พูดถึงคือ:
**คำสั่งเหล่านี้ใช้เวลานานแค่ไหน และ lock อะไรบ้างระหว่างที่รันอยู่**

| การเปลี่ยนแปลง | พฤติกรรมบน PostgreSQL 11+ | พฤติกรรมบน MySQL 8 |
|---|---|---|
| `ADD COLUMN` ที่มี `default` เป็นค่าคงที่ | เร็วมาก (metadata-only change, ไม่ต้องเขียนทับทั้งตาราง) | ส่วนใหญ่เร็ว (online DDL) แต่ขึ้นกับ storage engine/เวอร์ชัน |
| `ADD COLUMN` ที่มี `default` เป็น callable/expression ซับซ้อน | อาจต้อง rewrite ทั้งตาราง ขึ้นกับชนิดข้อมูล | มักต้อง rewrite ทั้งตาราง |
| `ALTER COLUMN ... SET NOT NULL` (โดยไม่มี default) | **ต้อง scan ทั้งตารางเพื่อตรวจสอบว่าไม่มีค่า NULL หลงเหลือ** — lock (แม้เป็น short lock) ตลอดช่วง scan | ต้อง rewrite ทั้งตารางในหลายกรณี |
| `CREATE INDEX` แบบปกติ | **lock การเขียน (write lock)** ตลอดการสร้าง index | lock ระดับหนึ่งขึ้นกับ engine |
| เปลี่ยนชนิดข้อมูลของคอลัมน์ (เช่น `INTEGER` → `BIGINT`) | มักต้อง rewrite ทั้งตาราง ใช้เวลานานตามขนาดตาราง | เช่นกัน |

ตารางที่มีไม่กี่พันแถวจะไม่รู้สึกถึงปัญหานี้เลย migration รันเสร็จในเสี้ยววินาที
แต่ตารางที่มี**หลายสิบล้านแถว** การ scan ทั้งตารางเพื่อ `SET NOT NULL` อาจใช้เวลา
หลายนาทีถึงหลายชั่วโมง — และถ้าคำสั่งนั้น lock การเขียนตลอดเวลานั้น **เว็บของคุณจะ
ค้างหรือ error ทุก request ที่พยายามเขียนข้อมูลลงตารางนั้นตลอดช่วงเวลาดังกล่าว**
นี่คือสิ่งที่เรียกว่า **downtime จาก migration**

### 156.2 หลักการของ Expand-and-Contract Pattern

ทบทวนภาพรวมจาก Part 011 (ขั้นตอนที่ 109.3): แนวคิดคือ **แบ่งการเปลี่ยนแปลงที่เสี่ยง
ให้เป็นหลายขั้นตอนเล็ก ๆ ที่แต่ละขั้นตอนปลอดภัยด้วยตัวเอง** แทนที่จะทำทุกอย่างใน
migration เดียวพร้อม deploy เดียว

```
Phase 1: EXPAND    → เพิ่มสิ่งใหม่เข้าไป โดยไม่ลบ/เปลี่ยนสิ่งเดิม (ปลอดภัย 100%)
Phase 2: MIGRATE    → คัดลอก/แปลงข้อมูลเก่าไปเติมในโครงสร้างใหม่ (background, ไม่ lock นาน)
Phase 3: SWITCH      → เปลี่ยนโค้ดแอปพลิเคชันให้ใช้ของใหม่แทนของเก่า
Phase 4: CONTRACT   → ลบของเก่าทิ้งเมื่อมั่นใจว่าไม่มีใครใช้แล้ว
```

หัวใจสำคัญคือ **แต่ละ Phase คือ deploy แยกกัน** ไม่ใช่ migration เดียวที่ทำทุกอย่าง
รวดเดียว เพราะระหว่าง deploy หนึ่งไปอีก deploy หนึ่ง (โดยเฉพาะระบบที่ deploy แบบ
**rolling deployment** ที่มีเซิร์ฟเวอร์หลายตัวรันโค้ดคนละเวอร์ชันพร้อมกันชั่วขณะ)
โค้ดเก่าและโค้ดใหม่อาจรันพร้อมกันได้ชั่วคราว จึงต้องออกแบบให้ทั้งสองเวอร์ชันทำงานได้
โดยไม่พังทั้งคู่

### 156.3 ตัวอย่างจริง: เปลี่ยน `Post.excerpt` จาก Optional เป็น Required

สมมติว่าทีมงานตัดสินใจว่า `excerpt` (ที่เดิม `blank=True` ตาม Part 011) ควรบังคับ
กรอกเสมอสำหรับบทความใหม่ทุกบทความ (เพื่อ SEO ที่ดีขึ้น) แต่ตารางมี `Post` เก่าอยู่
แล้วหลายแสนแถวที่ `excerpt` ยังว่างอยู่ — มาดูวิธีทำแบบ zero-downtime เต็มรูปแบบ

**Deploy 1 — Phase EXPAND:** เพิ่มความสามารถ backfill แต่ยังไม่บังคับ

```python
# blog/migrations/0021_post_excerpt_nullable_prep.py
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [('blog', '0020_add_concurrent_index')]
    operations = [
        # excerpt มีอยู่แล้วเป็น blank=True ตั้งแต่ Part 011 — Phase นี้แค่ deploy
        # โค้ดแอปพลิเคชันเวอร์ชันใหม่ที่เขียน excerpt ให้ทุกครั้งที่สร้าง Post ใหม่
        # (ทั้งจากฟอร์มและจาก admin) ยังไม่ต้องแก้ schema อะไรเลยในขั้นนี้
    ]
```

> ในโค้ดแอปพลิเคชัน (ไม่ใช่ migration) ให้แก้ view/form ที่สร้าง `Post` ใหม่ให้บังคับ
> กรอก `excerpt` ผ่าน validation ระดับฟอร์ม (Part 025) ไปก่อน โดยที่ฐานข้อมูลยังไม่
> บังคับ `NOT NULL` — นี่คือความหมายของ "Expand": เพิ่มพฤติกรรมใหม่โดยไม่ตัดทางถอย

**Deploy 2 — Phase MIGRATE:** Data migration backfill แบบแบ่ง batch

```python
# blog/migrations/0022_backfill_missing_excerpt.py
from django.db import migrations


def backfill_excerpt_in_batches(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')
    db_alias = schema_editor.connection.alias
    batch_size = 1000
    queryset = Post.objects.using(db_alias).filter(excerpt='').order_by('pk')

    while True:
        # ดึงมาทีละ batch เพื่อไม่ให้ transaction คาบเกี่ยวยาวเกินไป
        # และไม่ต้องโหลดข้อมูลทั้งหมดเข้า memory พร้อมกัน
        batch_pks = list(queryset.values_list('pk', flat=True)[:batch_size])
        if not batch_pks:
            break
        for post in Post.objects.using(db_alias).filter(pk__in=batch_pks):
            content = post.content or ''
            post.excerpt = (content[:297].rsplit(' ', 1)[0] + '...') if content else 'ไม่มีคำอธิบาย'
            post.save(update_fields=['excerpt'])


class Migration(migrations.Migration):
    dependencies = [('blog', '0021_post_excerpt_nullable_prep')]
    atomic = False  # ปิด transaction ครอบทั้งไฟล์ เพื่อให้แต่ละ batch commit แยกกันได้จริง
    operations = [
        migrations.RunPython(backfill_excerpt_in_batches, migrations.RunPython.noop),
    ]
```

> **ทำไมต้อง `atomic = False` ที่นี่ด้วย**: ถ้าปล่อยให้ทั้ง migration อยู่ใน
> transaction เดียว (ค่า default) การ backfill หลายแสนแถวจะกลายเป็น **transaction
> เดียวที่ยาวมาก** ซึ่งเสี่ยงต่อการ lock แถวที่เกี่ยวข้องนานเกินไป และถ้า migration
> ล้มเหลวกลางทาง (เช่น ไฟดับ, connection หลุด) จะต้อง rollback ทั้งหมดกลับไปเริ่มใหม่
> การปิด `atomic` แล้วให้แต่ละ `save()` commit ทันที ทำให้ progress ที่ทำไปแล้วไม่
> สูญหาย ถ้า migration หยุดกลางทางสามารถรันใหม่ได้และมันจะ resume จากจุดที่ค้างไว้เอง
> (เพราะ query กรองเฉพาะ `excerpt=''` ที่ยังไม่ถูกเติมเท่านั้น)

**Deploy 3 — Phase CONTRACT:** บังคับ `NOT NULL` จริงในฐานข้อมูล

```python
# blog/migrations/0023_post_excerpt_required.py
from django.db import migrations, models


class Migration(migrations.Migration):
    dependencies = [('blog', '0022_backfill_missing_excerpt')]
    operations = [
        migrations.AlterField(
            model_name='post',
            name='excerpt',
            field=models.CharField(max_length=300, blank=False),  # เอา blank=True ออก
        ),
    ]
```

ณ จุดนี้ (Deploy 3) ทุกแถวถูก backfill ครบแล้วจาก Deploy 2 ก่อนหน้า ดังนั้น
`AlterField` ที่บังคับ `blank=False` (ระดับ validation) จึงไม่มีความเสี่ยงใด ๆ
เพราะไม่มีแถวไหนที่ `excerpt=''` เหลืออยู่แล้ว หมายเหตุ: `blank=False` เป็นการ
เปลี่ยนแปลงระดับ Python/validation เท่านั้น — ถ้าต้องการบังคับที่ระดับฐานข้อมูลจริง
ด้วย (`NOT NULL` ใน SQL) ต้องพิจารณาว่าฟิลด์นี้มี `null=True` อยู่หรือไม่ (ในกรณี
`CharField` ตามกฎของ Part 011 ขั้นตอนที่ 102.2 เราไม่ใช้ `null=True` กับ field
ข้อความอยู่แล้ว จึงไม่มีคอลัมน์ระดับฐานข้อมูลให้ต้องเปลี่ยนเพิ่มเติมในกรณีนี้)

### 156.4 ตารางสรุปทั้ง 3 Deploy

| Deploy | Phase | สิ่งที่เปลี่ยน | ความเสี่ยงต่อ Downtime |
|---|---|---|---|
| 1 | Expand | เพิ่ม validation ระดับฟอร์ม/แอปพลิเคชัน (ไม่แตะ schema) | ไม่มีเลย |
| 2 | Migrate | Data migration backfill แบบแบ่ง batch, `atomic=False` | ต่ำมาก (แต่ละ batch เร็ว ไม่ lock นาน) |
| 3 | Contract | `AlterField` บังคับ `blank=False` เมื่อข้อมูลครบแล้ว | ไม่มีเลย (แค่เปลี่ยน metadata ระดับ Python) |

เทียบกับแนวทาง "ทำทุกอย่างในไฟล์เดียว" (เพิ่ม field + บังคับ NOT NULL ทันทีในคราว
เดียว) ซึ่งจะพยายาม scan และ lock ทั้งตารางในการ deploy ครั้งเดียว — สำหรับตารางเล็ก
ทั้งสองวิธีไม่ต่างกันเลยในทางปฏิบัติ แต่สำหรับตารางขนาดใหญ่ระดับ production จริง
ความแตกต่างคือ **"เว็บไม่มีปัญหาเลย" เทียบกับ "เว็บค้างไปหลายนาทีตอนกลางดึกวันที่มี
ผู้ใช้น้อยที่สุด"**

### 156.5 เทคนิคเสริม: Batch Update เพื่อไม่ Lock ตารางใหญ่นานเกินไป

หลักการแบ่ง batch ที่เห็นในขั้นตอนที่ 156.3 (ขั้นตอนที่ 0022) เป็นเทคนิคที่ใช้ซ้ำได้
ทุกครั้งที่ backfill ตารางขนาดใหญ่ สรุปหลักการสำคัญ:

```python
def backfill_in_batches(apps, schema_editor, batch_size=1000):
    Model = apps.get_model('app_label', 'ModelName')
    db_alias = schema_editor.connection.alias
    queryset = Model.objects.using(db_alias).filter(some_condition).order_by('pk')

    while True:
        batch_pks = list(queryset.values_list('pk', flat=True)[:batch_size])
        if not batch_pks:
            break
        # ทำงานกับ batch_pks เท่านั้น ไม่ใช่ทั้งตารางในทีเดียว
        Model.objects.using(db_alias).filter(pk__in=batch_pks).update(some_field='new_value')
```

| หลักการ | เหตุผล |
|---|---|
| แบ่งเป็น batch เล็ก ๆ (500-5,000 แถวต่อครั้ง ขึ้นกับขนาดแถว) | จำกัดเวลาที่แต่ละ query/transaction ใช้ ไม่ให้ lock นานเกินไป |
| `order_by('pk')` เสมอ | ทำให้การแบ่ง batch มีลำดับที่แน่นอน ไม่ข้ามหรือซ้ำแถวโดยไม่ตั้งใจ |
| `atomic = False` ที่ระดับ Migration | ให้แต่ละ batch commit แยกกันจริง ๆ ไม่ใช่รวมเป็น transaction เดียวยักษ์ |
| ใช้ `.update()` แทนการวน `.save()` ทีละแถวเมื่อทำได้ | `.update()` เป็น SQL คำสั่งเดียว เร็วกว่าการวน loop มาก (แต่ใช้ไม่ได้ถ้า logic การคำนวณค่าต่างกันในแต่ละแถว อย่างกรณี slug ในขั้นตอนที่ 151) |
| พิจารณาเพิ่ม `time.sleep(0.1)` เล็กน้อยระหว่าง batch บนระบบที่ sensitive มาก | ลดภาระต่อเนื่องบนฐานข้อมูล production ให้ระบบอื่นมีโอกาสได้ query แทรกบ้าง |

### 156.6 คำเตือนสำคัญเรื่อง Rolling Deployment

ในระบบที่ deploy แบบ rolling (เซิร์ฟเวอร์หลายตัว ทยอยอัปเดตทีละตัว ไม่ใช่ปิดทั้งหมด
พร้อมกัน — เจาะลึกเต็มรูปแบบใน Part 089) ต้องระวังเป็นพิเศษว่า **ระหว่างที่ deploy
กำลังดำเนินอยู่ โค้ดเก่าและโค้ดใหม่จะรันพร้อมกันชั่วขณะ** ทุก Phase ของ
Expand-and-Contract ต้องออกแบบให้ **โค้ดทั้งสองเวอร์ชันทำงานร่วมกับ schema ปัจจุบัน
ได้พร้อมกัน** เช่น ใน Deploy 1 (Expand) โค้ดเก่าที่ยังไม่รู้จัก validation ใหม่ต้อง
ยังคงทำงานได้ปกติกับ schema ที่ยังไม่เปลี่ยน และใน Deploy 3 (Contract) ต้องมั่นใจ
100% ว่าไม่มีเซิร์ฟเวอร์ไหนรันโค้ดเก่าที่ยังพยายามสร้าง `Post` โดยไม่ใส่ `excerpt`
หลงเหลืออยู่แล้ว ก่อนจะปล่อยให้ migration บังคับ `blank=False` จริง

---

## ขั้นตอนที่ 157: การทดสอบ Migration — Data Migration Test และการอ่าน `sqlmigrate`

### 157.1 ทำไมต้องเทส Data Migration

Schema migration (`AddField`, `CreateModel` ฯลฯ) แทบไม่มีทางเขียนผิด logic ได้เลย
เพราะ Django สร้างให้อัตโนมัติจาก `models.py` แต่ **`RunPython` คือโค้ด Python ที่
คุณเขียนเอง 100%** และมีโอกาสมี bug ได้ไม่ต่างจากโค้ดแอปพลิเคชันทั่วไป — bug ใน
data migration อันตรายกว่าปกติเพราะ **มันแก้ไขข้อมูลจริงในฐานข้อมูลจริงโดยตรง**
ถ้าเขียน logic ผิด (เช่น backfill slug ผิดหมวดหมู่ หรือคำนวณ excerpt ตัดคำผิดจุด)
ข้อมูลที่เสียหายจะกระทบผู้ใช้จริงทันทีที่ migration รันเสร็จ

### 157.2 เขียน Test สำหรับ Data Migration ด้วย `MigrationExecutor`

Django มีเครื่องมือทดสอบ migration โดยเฉพาะที่ทำให้เราจำลอง "สถานะฐานข้อมูล ณ
migration หนึ่ง" แล้วทดสอบว่า migration ถัดไปแปลงข้อมูลถูกต้องหรือไม่ ก่อนที่จะ
ปล่อยให้รันจริง:

```python
# blog/tests/test_migrations.py
from django.db import connection
from django.db.migrations.executor import MigrationExecutor
from django.test import TransactionTestCase


class MigrationTestBase(TransactionTestCase):
    """
    Base class สำหรับเทส data migration: ย้อนฐานข้อมูลไปอยู่ที่ migration ก่อนหน้า
    ('migrate_from') สร้างข้อมูลตัวอย่างในสถานะนั้น แล้วรัน migrate ไปข้างหน้าจนถึง
    ('migrate_to') เพื่อตรวจสอบผลลัพธ์
    """
    migrate_from = None
    migrate_to = None
    app = 'blog'

    def setUp(self):
        assert self.migrate_from and self.migrate_to, (
            "ต้องระบุ migrate_from และ migrate_to ใน subclass เสมอ"
        )
        self.migrate_from = [(self.app, self.migrate_from)]
        self.migrate_to = [(self.app, self.migrate_to)]
        executor = MigrationExecutor(connection)
        old_apps = executor._create_project_state(nodes=self.migrate_from).apps

        # ย้อนฐานข้อมูลกลับไปอยู่ที่สถานะ "ก่อน" migration ที่จะทดสอบ
        executor.migrate(self.migrate_from)
        self.setUpBeforeMigration(old_apps)

        # รัน migrate ไปข้างหน้าจนถึง migration ที่ต้องการทดสอบ (Django จะ re-detect
        # graph ให้อัตโนมัติเมื่อเรียก migrate() ซ้ำด้วย target ใหม่)
        executor = MigrationExecutor(connection)
        executor.loader.build_graph()
        executor.migrate(self.migrate_to)

        self.apps = executor.loader.project_state(self.migrate_to).apps

    def setUpBeforeMigration(self, apps):
        raise NotImplementedError


class BackfillPostSlugMigrationTest(MigrationTestBase):
    migrate_from = '0015_post_view_count_non_negative_and_more'
    migrate_to = '0016_backfill_post_slug'

    def setUpBeforeMigration(self, apps):
        Category = apps.get_model('blog', 'Category')
        Post = apps.get_model('blog', 'Post')

        category = Category.objects.create(name='เทคโนโลยี', slug='technology')
        # จำลองแถวเก่าที่ slug ว่างอยู่ (สถานการณ์ก่อนมี logic auto-generate)
        self.post_without_slug = Post.objects.create(
            title='แนะนำ Django 5', slug='', content='...', category=category,
        )
        self.post_with_slug = Post.objects.create(
            title='อีกบทความหนึ่ง', slug='existing-slug', content='...', category=category,
        )

    def test_backfills_empty_slug_only(self):
        Post = self.apps.get_model('blog', 'Post')

        migrated_post = Post.objects.get(pk=self.post_without_slug.pk)
        self.assertNotEqual(migrated_post.slug, '')
        self.assertEqual(migrated_post.slug, 'แนะนำ-django-5')

    def test_does_not_touch_existing_slug(self):
        Post = self.apps.get_model('blog', 'Post')

        untouched_post = Post.objects.get(pk=self.post_with_slug.pk)
        # slug ที่มีอยู่แล้วต้องไม่ถูกแก้ไข
        self.assertEqual(untouched_post.slug, 'existing-slug')
```

รันเทสด้วยคำสั่งปกติ (เจาะลึกการเขียน test เต็มรูปแบบใน Part 059-060):

```bash
python manage.py test blog.tests.test_migrations
```

> **ทำไมใช้ `TransactionTestCase` แทน `TestCase` ปกติ**: `TestCase` มาตรฐาน (ที่จะ
> เรียนใน Part 059) ห่อทุกเทสด้วย transaction แล้ว rollback ทันทีหลังจบแต่ละเทส
> เพื่อความเร็ว แต่การรัน migration จริงผ่าน `MigrationExecutor` ต้องการ **ควบคุม
> transaction เอง** (เพราะ migration บางตัวเช่นที่มี `atomic=False` ทำงานนอก
> transaction โดยเจตนา) `TransactionTestCase` จึงเป็นทางเลือกที่ถูกต้องสำหรับกรณีนี้
> โดยเฉพาะ แม้จะช้ากว่า `TestCase` ปกติก็ตาม

### 157.3 ทบทวนการอ่าน `sqlmigrate` เพื่อประเมินผลกระทบก่อน Deploy

ทบทวนจาก Part 011 (ขั้นตอนที่ 105.4) แต่คราวนี้อ่านด้วยมุมมองที่ต่างออกไป — ไม่ใช่แค่
"อ่านเพื่อเข้าใจว่า ORM แปลงเป็น SQL อย่างไร" แต่ **"อ่านเพื่อประเมินความเสี่ยงก่อน
deploy จริงบน production"**:

```bash
python manage.py sqlmigrate blog 0020
```

```sql
BEGIN;
--
-- Raw SQL operation
--
CREATE INDEX CONCURRENTLY IF NOT EXISTS post_category_idx ON blog_post (category_id);
COMMIT;
```

> **สังเกตสิ่งผิดปกติ**: ถ้า migration ตั้ง `atomic = False` ไว้อย่างถูกต้อง (ตาม
> ขั้นตอนที่ 154.4) แต่ `sqlmigrate` ยังคงแสดง `BEGIN;`/`COMMIT;` ห่ออยู่ นั่นเป็น
> สัญญาณว่ามีบางอย่างผิดพลาด — ควรตรวจสอบ `atomic = False` ให้อยู่ที่ระดับ
> `class Migration` จริง ๆ ไม่ใช่แค่ใน operation เดียว ก่อน deploy จริงเสมอ

Checklist สำหรับอ่าน `sqlmigrate` ก่อน deploy migration ที่มีความเสี่ยงบน production:

| ตรวจสอบ | คำถามที่ต้องตอบให้ได้ |
|---|---|
| มี `ALTER TABLE ... ADD COLUMN ... NOT NULL` ที่ไม่มี default หรือไม่ | ถ้ามีและตารางมีข้อมูลอยู่แล้ว จะ error ทันที ต้องแก้ก่อน deploy |
| มี `CREATE INDEX` (ไม่ใช่ `CONCURRENTLY`) บนตารางใหญ่หรือไม่ | ถ้ามี พิจารณาเปลี่ยนเป็น `RunSQL` + `CONCURRENTLY` ตามขั้นตอนที่ 154.4 |
| มีการเปลี่ยนชนิดข้อมูลของคอลัมน์ (`ALTER COLUMN TYPE`) หรือไม่ | อาจต้อง rewrite ทั้งตาราง ประเมินเวลาที่ใช้จากขนาดตารางจริงก่อน |
| operation ทั้งหมดอยู่ใน transaction เดียวหรือไม่ (`BEGIN`/`COMMIT`) | ถ้าใช่ ถ้า operation ใดล้มเหลว ทั้งหมด rollback อัตโนมัติ (ปลอดภัยกว่า) แต่ก็หมายความว่า lock จะถูกถือไว้ตลอดทั้ง transaction |
| จำนวนแถวโดยประมาณของตารางที่ถูกกระทบคือเท่าไหร่ | ยิ่งตารางใหญ่ ยิ่งต้องพิจารณา Expand-and-Contract Pattern จากขั้นตอนที่ 156 |

### 157.4 ใช้ `--plan` ร่วมกับ `showmigrations` ก่อน Deploy

```bash
python manage.py showmigrations blog --plan
```

แสดงลำดับ migration ที่จะถูก apply จริงตาม topological order (ทบทวนจาก Part 011
ขั้นตอนที่ 105.3) — ใช้ตรวจสอบก่อน deploy ว่า migration ทั้งหมดที่ยัง pending อยู่
มีลำดับตรงตามที่คาดไว้ ไม่มี migration ที่หลงเหลือจาก branch เก่าที่ไม่ได้ตั้งใจ
ให้รวมเข้ามาด้วย

---

## ขั้นตอนที่ 158: Migration State vs Database State — `--fake`, `--fake-initial`

### 158.1 สองสถานะที่ Django แยกจากกันเสมอ

นี่คือแนวคิดที่ลึกที่สุดของระบบ migration ทั้งหมด และเป็นกุญแจไขปริศนาว่าทำไม
`--fake` ถึงมีอยู่: Django แยก **"สถานะที่ Django คิดว่าฐานข้อมูลเป็น" (Migration
State)** ออกจาก **"สถานะที่ฐานข้อมูลเป็นจริง ๆ" (Database State)** อย่างเด็ดขาด
ทั้งสองสถานะ **ไม่ได้ผูกกันโดยอัตโนมัติเสมอไป**:

```
┌─────────────────────────┐          ┌──────────────────────────┐
│   Migration State        │          │    Database State         │
│  (สิ่งที่ Django "เชื่อ")  │   ควร    │   (schema จริงที่มีอยู่)    │
│                           │  ตรงกัน  │                            │
│  มาจากตาราง               │ ◄──────► │  มาจากการรัน SQL จริง       │
│  django_migrations        │          │  (CREATE TABLE, ALTER ฯลฯ) │
│  (บันทึกว่า migration ไหน  │          │                            │
│   "ถูกทำเครื่องหมายว่า      │          │                            │
│   apply แล้ว")             │          │                            │
└─────────────────────────┘          └──────────────────────────┘
```

ในสถานการณ์ปกติที่คุณรัน `python manage.py migrate` ตามปกติ **ทั้งสองสถานะจะตรงกัน
เสมอ** เพราะ Django รัน SQL จริงพร้อมกับบันทึกลง `django_migrations` ไปพร้อมกันทุก
ครั้ง แต่มีบางสถานการณ์ที่ทั้งสองสถานะ **หลุดจากกัน** ได้ ซึ่ง `--fake` และ
`--fake-initial` มีไว้จัดการสถานการณ์เหล่านั้นโดยเฉพาะ

### 158.2 `--fake`: บอก Django ว่า "Apply แล้วนะ" โดยไม่รัน SQL จริง

```bash
python manage.py migrate blog 0020 --fake
```

คำสั่งนี้เขียนบันทึกลงตาราง `django_migrations` ว่า migration `0020` ถูก apply แล้ว
**โดยไม่รัน SQL ใด ๆ เลยแม้แต่บรรทัดเดียว** — Migration State เปลี่ยน แต่
Database State ไม่เปลี่ยนตาม

ใช้เมื่อไหร่: เมื่อคุณ **สร้างการเปลี่ยนแปลงนั้นในฐานข้อมูลไปแล้วด้วยวิธีอื่นที่ไม่ใช่
Django migration** เช่น DBA ของทีมรัน `ALTER TABLE` ด้วยมือตรง ๆ ผ่าน `psql` ตอน
กลางดึกเพื่อเลี่ยง downtime (สถานการณ์ที่พบได้จริงในทีมขนาดใหญ่ที่มี DBA เฉพาะทาง)
แล้วค่อยมาเขียน migration file ทีหลังเพื่อให้ codebase สะท้อนความเปลี่ยนแปลงนั้น —
ในกรณีนี้ต้อง `--fake` migration นั้นเพื่อไม่ให้ Django พยายามรัน `ALTER TABLE` ซ้ำ
อีกรอบ (ซึ่งจะ error เพราะคอลัมน์มีอยู่แล้ว)

### 158.3 `--fake-initial`: เชื่อม Django เข้ากับฐานข้อมูลที่มีอยู่แล้ว

นี่คือกรณีใช้งานที่พบบ่อยกว่ามาก: บริษัทมีฐานข้อมูลเดิมที่สร้างมาก่อนจะใช้ Django
(เช่น ระบบเก่าที่เขียนด้วยภาษาอื่น หรือฐานข้อมูลที่ทีม DBA ดูแลแยกต่างหาก) และตอนนี้
ต้องการเขียนแอป Django ใหม่มาต่อกับฐานข้อมูลเดิมนั้น โดยที่**ตารางมีอยู่แล้วจริง**
พร้อมข้อมูลเต็ม

**ขั้นตอนที่ 1**: ใช้ `inspectdb` ให้ Django "อ่าน" โครงสร้างตารางที่มีอยู่แล้ว
แล้วสร้าง `models.py` ให้อัตโนมัติ:

```bash
python manage.py inspectdb > legacy_app/models.py
```

ผลลัพธ์ตัวอย่าง (Django เดาโครงสร้างจาก schema จริงในฐานข้อมูล):

```python
# legacy_app/models.py (สร้างโดย inspectdb — ต้องตรวจสอบและปรับแต่งเสมอ)
from django.db import models


class LegacyCustomer(models.Model):
    customer_id = models.AutoField(primary_key=True)
    full_name = models.CharField(max_length=255)
    email = models.CharField(max_length=255, blank=True, null=True)
    registered_at = models.DateTimeField(blank=True, null=True)

    class Meta:
        managed = False   # สำคัญมาก: บอก Django ว่าไม่ต้องจัดการตารางนี้ผ่าน migration
        db_table = 'customers'   # ชื่อตารางจริงในฐานข้อมูลเดิม (ไม่ใช่ตาม convention ของ Django)
```

> **`managed = False` คือกุญแจสำคัญ**: ถ้าปล่อยค่า default (`managed = True`)
> Django จะพยายามสร้าง/แก้/ลบตารางนี้ผ่าน migration เหมือน model ปกติทุกประการ
> ซึ่งอันตรายมากกับตารางที่มีข้อมูลสำคัญของระบบเดิมอยู่แล้ว การตั้ง `managed = False`
> บอก Django ว่า "ใช้ตารางนี้ในการ query ได้ตามปกติ (ORM ทำงานได้เต็มรูปแบบ) แต่**ห้าม
> แตะโครงสร้างตารางนี้ผ่าน migration เด็ดขาด**" เหมาะกับตารางที่ทีมอื่นเป็นเจ้าของ
> โครงสร้างอยู่แล้ว

**ขั้นตอนที่ 2**: สำหรับ model ที่**ต้องการให้ Django จัดการต่อ** (`managed = True`
ตามปกติ) แต่ตารางมีอยู่แล้วในฐานข้อมูล (ตรงกับที่ `makemigrations` จะสร้างให้พอดี)
ให้สร้าง migration ตามปกติก่อน:

```bash
python manage.py makemigrations legacy_app
```

**ขั้นตอนที่ 3**: แทนที่จะรัน `migrate` ปกติ (ซึ่งจะพยายาม `CREATE TABLE` ที่มีอยู่
แล้วจริง แล้ว error `table already exists`) ให้ใช้ `--fake-initial`:

```bash
python manage.py migrate legacy_app --fake-initial
```

`--fake-initial` ฉลาดกว่า `--fake` ธรรมดาตรงที่มันจะ**ตรวจสอบ** migration
`0001_initial` ก่อนว่าตารางที่มันกำลังจะสร้างนั้น **มีอยู่แล้วในฐานข้อมูลหรือไม่**:

| ผลตรวจสอบ | พฤติกรรม |
|---|---|
| ตารางยังไม่มีอยู่จริง | รัน `CREATE TABLE` ตามปกติ (เหมือน `migrate` ธรรมดาทุกประการ) |
| ตารางมีอยู่แล้วและโครงสร้าง**ตรงกัน**พอดี | ทำเครื่องหมายว่า apply แล้ว (fake) โดยไม่รัน SQL ซ้ำ |
| ตารางมีอยู่แล้วแต่โครงสร้าง**ไม่ตรงกัน** | อาจเกิด error หรือ inconsistency — ต้องตรวจสอบ `models.py` ให้ตรงกับ schema จริงเป๊ะก่อนเสมอ |

### 158.4 ตารางสรุป: `--fake` vs `--fake-initial`

| ประเด็น | `--fake` | `--fake-initial` |
|---|---|---|
| ใช้กับ migration ไหน | migration ใดก็ได้ ระบุเจาะจงเอง | เฉพาะ migration แรก (`0001_initial`) ของแอปเท่านั้น |
| ตรวจสอบ schema จริงก่อนหรือไม่ | ❌ ไม่ตรวจสอบเลย เชื่อคำสั่งทันที | ✅ ตรวจสอบว่าตารางมีอยู่แล้วตรงกันก่อน |
| ใช้บ่อยที่สุดในสถานการณ์ | รัน DDL เองด้วยมือไปแล้ว, sync migration state ให้ตรงกับที่ทำไปแล้ว | เชื่อมโปรเจกต์ Django ใหม่เข้ากับฐานข้อมูลเดิมที่มีตารางอยู่แล้ว |
| ความเสี่ยงถ้าใช้ผิดจังหวะ | สูงมาก — Migration State กับ Database State อาจไม่ตรงกันถาวร | ต่ำกว่า เพราะมีการตรวจสอบก่อน แต่ยังต้องมั่นใจว่า `models.py` ตรงกับ schema จริง |

### 158.5 อันตรายของการใช้ `--fake` ผิดจังหวะ

```bash
# ตัวอย่างอันตราย: fake migration ที่จริง ๆ ยังไม่เคยรัน SQL จริงเลย
python manage.py migrate blog 0024 --fake
```

ถ้า migration `0024` เพิ่ม field `internal_code` จริง ๆ แต่คุณสั่ง `--fake` โดยเข้าใจ
ผิดว่ามันถูกสร้างไปแล้ว (ทั้งที่จริงยังไม่มี) ผลลัพธ์คือ:

1. `django_migrations` บันทึกว่า `0024` apply แล้ว (Migration State บอกว่า "มี field
   `internal_code` แล้ว")
2. แต่ตารางจริงในฐานข้อมูล **ไม่มีคอลัมน์ `internal_code` เลย** (Database State ไม่มี)
3. เมื่อโค้ดแอปพลิเคชันพยายาม `Post.objects.filter(internal_code='X')` จะได้
   `django.db.utils.OperationalError: no such column: blog_post.internal_code`
   ทันที — error ที่งงมากสำหรับคนที่มาแก้บั๊กทีหลัง เพราะ `showmigrations` จะบอกว่า
   migration นี้ apply แล้วเรียบร้อยดี (`[X]`) ทั้งที่จริงคอลัมน์ไม่มีอยู่จริง

> **กฎเหล็กของหลักสูตรนี้**: ใช้ `--fake` เฉพาะเมื่อคุณ**มั่นใจ 100%** ว่าการ
> เปลี่ยนแปลงนั้นเกิดขึ้นในฐานข้อมูลจริงแล้วด้วยวิธีอื่นเท่านั้น หากไม่แน่ใจ ให้ตรวจสอบ
> ด้วย `dbshell` หรือ `sqlmigrate` เปรียบเทียบกับ schema จริงก่อนเสมอ ไม่มีทาง "undo"
> ที่ปลอดภัยสำหรับการ `--fake` ผิดจังหวะ นอกจากไปแก้ตาราง `django_migrations`
> ด้วยมือ ซึ่งเสี่ยงยิ่งกว่าเดิม

---

## ขั้นตอนที่ 159: การจัดการ Dependency ระหว่าง Migration ของหลายแอป

### 159.1 ทบทวน `dependencies` และเจาะลึกกรณีข้ามแอป

ทบทวนจาก Part 011 (ขั้นตอนที่ 104.2): ทุกไฟล์ migration มี `dependencies` เป็น
list ของ `(app_label, migration_name)` — สิ่งที่ Part 011 พูดถึงสั้น ๆ แต่ยังไม่ได้
ลงรายละเอียดคือ **เมื่อไหร่ที่ dependency จะข้ามแอปโดยอัตโนมัติ**

กฎคือ: **ทุกครั้งที่ model ในแอปหนึ่งมี `ForeignKey`/`OneToOneField`/`ManyToManyField`
ชี้ไปยัง model ของอีกแอปหนึ่ง** Django จะเพิ่ม dependency ข้ามแอปให้อัตโนมัติเสมอ
ตัวอย่างจริงจากโปรเจกต์ของเรา — `accounts.Profile` มี `OneToOneField` ชี้ไปยัง
`settings.AUTH_USER_MODEL` (Part 012 ขั้นตอนที่ 112.3):

```python
# accounts/migrations/0001_initial.py
from django.conf import settings
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):

    initial = True

    dependencies = [
        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
    ]

    operations = [
        migrations.CreateModel(
            name='Profile',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False)),
                ('bio', models.TextField(blank=True)),
                ('avatar', models.ImageField(blank=True, null=True, upload_to='avatars/')),
                (
                    'user',
                    models.OneToOneField(
                        on_delete=django.db.models.deletion.CASCADE,
                        to=settings.AUTH_USER_MODEL,
                    ),
                ),
            ],
        ),
    ]
```

### 159.2 `migrations.swappable_dependency()`: ทำไมไม่เขียน `('auth', '0012_...')` ตรง ๆ

สังเกตว่า dependency ที่ Django สร้างให้ไม่ใช่ `('auth', '0012_...')` ตรง ๆ แต่เป็น
`migrations.swappable_dependency(settings.AUTH_USER_MODEL)` — เหตุผลคือ `User` model
เป็น **"swappable" model** (โมเดลที่โปรเจกต์สามารถเปลี่ยนไปใช้ Custom User Model
แทนได้ ตามที่ Part 012 ขั้นตอนที่ 112.7 เกริ่นไว้ว่าจะเรียนเต็มรูปแบบใน Part 032)

`swappable_dependency()` บอก Django ว่า "แอปนี้ขึ้นกับ **ตัวแปร** `AUTH_USER_MODEL`
ไม่ใช่ขึ้นกับแอป `auth` ตรง ๆ เสมอไป" — ถ้าโปรเจกต์เปลี่ยนไปใช้ Custom User Model
(เช่น `accounts.User` แทน `auth.User`) migration ที่เขียนด้วย `swappable_dependency`
จะปรับ dependency ให้ชี้ไปที่แอปที่ถูกต้องโดยอัตโนมัติ โดยไม่ต้องแก้ไฟล์ migration
เก่าเลยแม้แต่บรรทัดเดียว — ต่างจากถ้าเขียน `('auth', '0012_...')` ตรง ๆ ซึ่งจะยังคง
ชี้ไปที่ `auth.User` ตายตัวแม้โปรเจกต์จะเปลี่ยน User model ไปแล้วก็ตาม

### 159.3 `run_before`: บังคับลำดับโดยไม่ต้องมี Field เชื่อมกัน

บางครั้งคุณต้องการบังคับว่า migration ของแอป A ต้องรันก่อน migration ของแอป B
**โดยที่ไม่มี relation ระหว่าง model ของทั้งสองแอปเลย** (เช่น ต้องการให้ data
migration ที่ seed ข้อมูลอ้างอิงของแอป `blog` รันเสร็จก่อนที่ data migration ของแอป
`shop` จะเริ่ม อ่านค่าจากมันไปใช้) ใช้ attribute พิเศษ **`run_before`**:

```python
# blog/migrations/0016_backfill_post_slug.py
class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0015_post_view_count_non_negative_and_more'),
    ]

    run_before = [
        ('shop', '0003_import_product_categories'),
    ]

    operations = [
        migrations.RunPython(backfill_post_slug, reverse_backfill_post_slug),
    ]
```

`run_before` คือ "dependency กลับด้าน" — แทนที่จะบอกว่า "ฉันต้องรันหลัง X" (แบบ
`dependencies`) มันบอกว่า "ฉันต้องรันก่อน Y" ประโยชน์หลักคือใช้ในสถานการณ์ที่คุณ
**เขียน migration ในแอป A ทีหลัง** แต่ต้องการให้มันรันก่อน migration ที่มีอยู่แล้ว
ในแอป B (ซึ่งคุณอาจไม่อยากไปแก้ไฟล์ migration เดิมของแอป B โดยตรง)

### 159.4 ปัญหา Circular Dependency ระหว่างสองแอป

สถานการณ์ที่ซับซ้อนที่สุด: แอป `blog` มี model ที่ FK ไปยังแอป `shop` **และ** แอป
`shop` ก็มี model ที่ FK กลับมายัง `blog` เช่นกัน (เช่น `shop.Product` มี FK ไปยัง
`blog.Post` สำหรับ "บทความรีวิวสินค้า" ในขณะที่ `blog.Post` ก็มี FK ไปยัง
`shop.Product` สำหรับ "สินค้าที่ถูกพูดถึงในบทความ") ถ้าทั้งสอง `ForeignKey` อยู่ใน
`CreateModel` operation ของ migration `0001_initial` ของแต่ละแอปพร้อมกัน Django
จะหา**ลำดับที่ทำได้จริงไม่เจอเลย** เพราะ `blog.0001_initial` ต้องรันหลัง
`shop.0001_initial` (เพื่อให้ `shop.Product` มีอยู่ก่อน) แต่ `shop.0001_initial` ก็
ต้องรันหลัง `blog.0001_initial` เช่นกัน (เพื่อให้ `blog.Post` มีอยู่ก่อน) — เป็น
วงกลมที่แก้ไม่ได้ (`CircularDependencyError`)

**วิธีแก้ที่ถูกต้อง**: แยก `ForeignKey` ที่ทำให้เกิดวงกลมออกมาเป็น migration แยก
ต่างหาก โดยสร้าง model ทั้งสองตัวแบบยังไม่มี FK ข้ามแอปก่อน แล้วค่อยเพิ่ม FK
ทีหลังในอีก migration หนึ่ง:

```
Migration 1: blog.0001_initial    → สร้าง Post (ยังไม่มี FK ไปยัง Product)
Migration 2: shop.0001_initial     → สร้าง Product (มี FK ไปยัง blog.Post ได้เลย เพราะสร้างทีหลัง)
Migration 3: blog.0002_add_featured_product → เพิ่ม FK จาก Post ไปยัง shop.Product (ตอนนี้ shop.Product มีอยู่แล้ว)
```

```python
# blog/migrations/0002_add_featured_product.py
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0001_initial'),
        ('shop', '0001_initial'),   # ตอนนี้ shop.Product มีอยู่แล้วแน่นอน ไม่มีวงกลมอีกต่อไป
    ]

    operations = [
        migrations.AddField(
            model_name='post',
            name='featured_product',
            field=models.ForeignKey(
                to='shop.product',
                on_delete=django.db.models.deletion.SET_NULL,
                null=True,
                blank=True,
            ),
        ),
    ]
```

หลักการคือ **"ตัดวงกลมด้วยการเลื่อน FK ตัวใดตัวหนึ่งออกไปเป็น `AddField` แยกต่างหาก
ทีหลัง"** — ในทางปฏิบัติ Django `makemigrations` จะตรวจจับสถานการณ์นี้ให้อัตโนมัติ
เมื่อสองแอปมี FK ชี้หากันตั้งแต่ต้น (มันจะสร้างไฟล์แยกให้เองแบบข้างต้นโดยไม่ต้องทำ
ด้วยมือ) แต่การเข้าใจว่า **ทำไม** Django ต้องแยกไฟล์แบบนี้ช่วยให้ debug ปัญหา
`CircularDependencyError` ได้เมื่อมันเกิดขึ้นจริงในสถานการณ์ที่ซับซ้อนกว่านี้ (เช่น
เมื่อแก้ไข migration เก่าด้วยมือแล้วทำให้ dependency พันกันเอง)

### 159.5 ตารางสรุป Dependency Mechanism ทั้งหมด

| Mechanism | ใช้เมื่อ | ตัวอย่าง |
|---|---|---|
| `dependencies` ปกติ | บอกว่า migration นี้ต้องรันหลัง migration ที่ระบุ (กรณีทั่วไปที่สุด) | `[('blog', '0015_...')]` |
| `dependencies` ข้ามแอปอัตโนมัติ | เมื่อมี FK/O2O/M2M ชี้ไปยัง model ของแอปอื่น (Django ใส่ให้เอง) | `accounts.Profile` → `auth.User` |
| `migrations.swappable_dependency()` | เมื่อ dependency ชี้ไปยัง swappable model (โดยเฉพาะ `AUTH_USER_MODEL`) | ทุก migration ที่ FK ไปยัง `settings.AUTH_USER_MODEL` |
| `run_before` | บังคับลำดับแบบ "ฉันต้องมาก่อน" โดยไม่มี relation เชื่อมกัน | seed data ที่แอปอื่นต้องใช้ต่อ |
| แยก `AddField` ออกจาก `CreateModel` | แก้ปัญหา circular dependency ระหว่างสองแอปที่ FK ชี้หากัน | `blog.Post` ↔ `shop.Product` |

---

## ขั้นตอนที่ 160: สรุปและแบบฝึกหัด — เขียน Data Migration จริงที่ Backfill Slug ทุกตัว

### 160.1 โจทย์สรุปรวม: Data Migration ที่สมบูรณ์แบบมืออาชีพ

มารวมทุกเทคนิคที่เรียนมาทั้ง Part นี้เข้าด้วยกันเป็น migration เดียวที่ backfill
`slug` ให้ `Post` ทุกตัวที่ยังว่างอยู่ — เวอร์ชันสมบูรณ์ที่พร้อมใช้งานจริงบน
production ตารางขนาดใหญ่:

```python
# blog/migrations/0016_backfill_post_slug.py
"""
Data migration: เติม slug ให้ Post ทุกแถวที่ยังว่างอยู่

สร้างขึ้นเพื่อรองรับ Post ที่ถูก import จากระบบเก่า หรือถูกสร้างขึ้นก่อนที่ทีมจะเพิ่ม
logic auto-generate slug ใน Post.save() (ดู Part 011 ขั้นตอนที่ 103.3)

ออกแบบให้:
- ปลอดภัยสำหรับตารางขนาดใหญ่ (แบ่ง batch, atomic=False)
- resume ได้เองถ้าถูกขัดจังหวะกลางทาง (query กรองเฉพาะแถวที่ยังไม่เสร็จ)
- ไม่ชนกันเองระหว่าง slug ในหมวดหมู่เดียวกัน (ตาม UniqueConstraint จาก Part 015)
"""
from django.db import migrations
from django.utils.text import slugify


def generate_unique_slug(Post, db_alias, title, category_id, exclude_pk=None):
    """คำนวณ slug ที่ไม่ซ้ำภายในหมวดหมู่เดียวกัน ใช้ logic เดียวกับ Post.save()"""
    base_slug = slugify(title) or 'post'
    slug = base_slug
    counter = 1
    while True:
        qs = Post.objects.using(db_alias).filter(category_id=category_id, slug=slug)
        if exclude_pk is not None:
            qs = qs.exclude(pk=exclude_pk)
        if not qs.exists():
            return slug
        slug = f'{base_slug}-{counter}'
        counter += 1


def backfill_post_slug(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')
    db_alias = schema_editor.connection.alias
    batch_size = 500

    queryset = Post.objects.using(db_alias).filter(slug='').order_by('pk')
    while True:
        batch_pks = list(queryset.values_list('pk', flat=True)[:batch_size])
        if not batch_pks:
            break
        for post in Post.objects.using(db_alias).filter(pk__in=batch_pks):
            post.slug = generate_unique_slug(
                Post, db_alias, post.title, post.category_id, exclude_pk=post.pk,
            )
            post.save(update_fields=['slug'])


def reverse_backfill_post_slug(apps, schema_editor):
    # ย้อนกลับไม่ได้อย่างมีความหมาย เพราะไม่มีทางแยกว่า slug ไหนมีอยู่เดิม
    # กับ slug ไหนถูกสร้างจาก migration นี้ — เลือกไม่ทำอะไรตอน rollback
    pass


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0015_post_view_count_non_negative_and_more'),
    ]

    atomic = False  # ให้แต่ละ batch commit แยกกัน ไม่ lock ทั้งตารางในทีเดียว

    operations = [
        migrations.RunPython(
            backfill_post_slug,
            reverse_backfill_post_slug,
            elidable=True,  # ในอนาคตถ้า squash migration นี้ตัดทิ้งได้อย่างปลอดภัย
        ),
    ]
```

รันและตรวจสอบผลลัพธ์:

```bash
python manage.py sqlmigrate blog 0016      # ตรวจสอบก่อน (จะไม่แสดง SQL ใด ๆ เพราะเป็น RunPython ล้วน)
python manage.py migrate blog
python manage.py shell -c "from blog.models import Post; print(Post.objects.filter(slug='').count())"
# ควรได้ 0
```

### 160.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างระหว่าง Schema Migration กับ Data Migration และเมื่อไหร่ต้อง
  ใช้แบบไหน
- ✅ เขียน Data Migration ด้วย `RunPython` ที่ backfill ข้อมูลจริงอย่างปลอดภัย
  พร้อมเข้าใจว่าทำไมต้องใช้ `apps.get_model()` แทนการ import model ตรง ๆ เสมอ
- ✅ ใช้ `squashmigrations` รวมไฟล์ migration จำนวนมากให้เหลือไฟล์เดียว พร้อมทดสอบ
  ทั้งสถานการณ์ฐานข้อมูลใหม่และฐานข้อมูลเดิม
- ✅ เขียน `reverse_code` ที่มีความหมายจริง และรู้ว่าเมื่อไหร่ควรใช้ `RunPython.noop`
  แทนแทนการเขียน reverse function ที่ไม่ถูกต้อง
- ✅ ใช้ `RunSQL` สำหรับสถานการณ์ที่ ORM operation มาตรฐานไม่รองรับ โดยเฉพาะ
  `CREATE INDEX CONCURRENTLY` บน PostgreSQL พร้อม `atomic = False`
- ✅ แก้ Migration Conflict ในทีมอย่างปลอดภัยเมื่อ operation ชนกันจริง ไม่ใช่แค่กด
  `--merge` ผ่านไปโดยไม่อ่านคำเตือน
- ✅ ออกแบบ Zero-Downtime Migration ตามหลัก Expand-and-Contract Pattern พร้อม
  เทคนิค batch update สำหรับตารางขนาดใหญ่
- ✅ เขียน Test สำหรับ Data Migration ด้วย `MigrationExecutor` และอ่าน `sqlmigrate`
  เพื่อประเมินความเสี่ยงก่อน deploy จริง
- ✅ เข้าใจ Migration State vs Database State และใช้ `--fake`/`--fake-initial`
  อย่างถูกต้องและปลอดภัย
- ✅ จัดการ dependency ระหว่าง migration ของหลายแอป รวมถึงแก้ปัญหา Circular
  Dependency

### 160.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน data migration ด้วย `RunPython` ที่ backfill ข้อมูลจริงได้เอง พร้อม
      `apps.get_model()`
- [ ] อธิบายได้ว่าทำไม historical model จาก `apps.get_model()` ไม่มี custom method
      หรือ custom manager
- [ ] รัน `squashmigrations` กับแอปทดลองอย่างน้อยหนึ่งครั้ง และทดสอบทั้งฐานข้อมูล
      ใหม่กับฐานข้อมูลที่มี migration เดิม apply แล้ว
- [ ] เขียน `RunPython` ที่มี `reverse_code` ทำงานได้จริงอย่างน้อยหนึ่งตัวอย่าง
- [ ] เขียน `RunSQL` ที่สร้าง index ด้วย `CONCURRENTLY` พร้อมตั้ง `atomic = False`
      ถูกต้อง
- [ ] อธิบาย Expand-and-Contract Pattern ได้ครบทั้ง 4 phase พร้อมยกตัวอย่างของ
      ตัวเอง
- [ ] เขียน test สำหรับ data migration ด้วย `MigrationExecutor` ได้อย่างน้อยหนึ่งไฟล์
- [ ] อธิบายความแตกต่างระหว่าง Migration State กับ Database State ด้วยคำพูดตัวเอง
      ได้
- [ ] อธิบายได้ว่าทำไม `--fake` ที่ใช้ผิดจังหวะถึงอันตราย และวิธีตรวจสอบก่อนใช้

### 160.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน data migration ใหม่ในแอป `blog` ที่ backfill field
`excerpt` (จาก Part 011 ขั้นตอนที่ 103.2) ให้ `Post` ทุกแถวที่ `excerpt=''` โดยตัด
มาจาก `content` 150 ตัวอักษรแรก (ใช้ `.rsplit(' ', 1)[0]` ตัดคำที่ขาดครึ่งออก
เหมือนใน `Post.save()`) เขียนทั้ง forward function และ reverse function ที่ตั้งค่า
`excerpt` กลับเป็น `''` ให้เฉพาะแถวที่ migration นี้เป็นคนเติมให้เท่านั้น (ใบ้: ต้อง
เก็บรายชื่อ `pk` ที่ถูกแก้ไว้ก่อน หรือยอมรับว่าการย้อนกลับแบบสมบูรณ์ 100% ทำไม่ได้
และเขียนคอมเมนต์อธิบายข้อจำกัดนั้นแทน)

**แบบฝึกหัดที่ 2**: สร้างสถานการณ์ migration conflict ขึ้นมาเองบน branch ทดลอง
สองสาขา ให้ทั้งสองสาขาแก้ `max_length` ของ field เดียวกันเป็นคนละค่า (เหมือนตัวอย่าง
ในขั้นตอนที่ 155.2) แล้วลองรัน `makemigrations --merge` ดูข้อความเตือนที่ Django
แสดง จากนั้นแก้ปัญหาด้วยวิธีที่ถูกต้อง (ลบไฟล์ที่ชนกันทั้งคู่ ตกลงค่าสุดท้ายร่วมกัน
แล้วสร้างใหม่) แทนการปล่อยให้ merge migration เปล่าตัดสินผลลัพธ์แบบสุ่ม

**แบบฝึกหัดที่ 3**: ออกแบบและเขียน migration แบบ Expand-and-Contract เต็มรูปแบบ
(อย่างน้อย 3 ไฟล์แยกกัน) สำหรับสถานการณ์: เปลี่ยน `Comment.author` (ปัจจุบันเป็น
`CharField` ตาม Part 012) ให้กลายเป็น `ForeignKey` ไปยัง `settings.AUTH_USER_MODEL`
แทน โดยที่มีคอมเมนต์เก่าอยู่แล้วที่เก็บชื่อผู้เขียนเป็น string ธรรมดา (ใบ้: Phase
Expand ต้องเพิ่ม field ใหม่ชื่อ `author_user` แบบ `null=True` ก่อน โดยยังไม่ลบ
`author` เดิม, Phase Migrate ต้องเขียน data migration ที่พยายามหา `User` ที่มี
`username` ตรงกับค่าใน `author` เดิม ถ้าหาไม่เจอให้ข้ามแถวนั้นไปและบันทึกจำนวนที่
ข้ามไว้ใน log, Phase Contract ค่อยลบ `author` เดิมทิ้งในภายหลัง)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน test สำหรับ data migration ในแบบฝึกหัดที่ 1
ด้วย `MigrationExecutor` ตามรูปแบบในขั้นตอนที่ 157.2 ให้ครอบคลุมอย่างน้อย 3 กรณี:
(1) `Post` ที่ `excerpt=''` ต้องถูกเติมค่าใหม่ (2) `Post` ที่มี `excerpt` อยู่แล้ว
ต้องไม่ถูกแก้ไข (3) `Post` ที่ `content` สั้นกว่า 150 ตัวอักษร ต้องได้ `excerpt`
เท่ากับ `content` ทั้งหมดโดยไม่มี `...` ต่อท้าย

### 160.5 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมต้องแยก Data Migration ออกจาก Schema Migration เป็นคนละไฟล์เสมอ
รวมไว้ในไฟล์เดียวกันไม่ได้หรือ?**
A: ทำได้ในทางเทคนิค (ใส่ `AddField` กับ `RunPython` ใน `operations` เดียวกันได้)
แต่หลักสูตรนี้แนะนำให้แยกไฟล์เสมอ เพราะเหตุผลด้านความปลอดภัย 2 ข้อ: (1) ถ้า data
migration ผิดพลาดกลางทาง การ rollback เฉพาะ data migration โดยไม่กระทบ schema
เดิมทำได้ง่ายกว่ามาก (2) การ squash migration ในอนาคต (ขั้นตอนที่ 152) จะรวม schema
operation ให้อัตโนมัติได้ดี แต่ `RunPython` ต้องตรวจสอบด้วยมือเสมอ — การแยกไฟล์ทำให้
รู้ชัดเจนว่าไฟล์ไหนต้องตรวจสอบเป็นพิเศษตอน squash

**Q: `RunPython` ที่ backfill ข้อมูลจำนวนมากรันช้ามากตอน deploy จะทำอย่างไรดี?**
A: ก่อนอื่นตรวจสอบว่าใช้เทคนิค batch ตามขั้นตอนที่ 156.5 หรือยัง ถ้าใช้แล้วยังช้าอยู่
ให้พิจารณาแยก data migration ออกจากขั้นตอน deploy ปกติไปเป็น **management command**
ต่างหาก (เรียนเต็มรูปแบบใน Part 040) ที่รันแยกเป็น background job หลัง deploy เสร็จ
แทนที่จะให้ deploy ต้องรอ migration ที่ใช้เวลานานให้เสร็จก่อนถึงจะถือว่า deploy
สำเร็จ — วิธีนี้ทำให้ deploy เร็วขึ้น ในขณะที่ data migration ทำงานอยู่เบื้องหลังต่อไป

**Q: ถ้าลืมใส่ `atomic = False` ตอนใช้ `CREATE INDEX CONCURRENTLY` จะเกิดอะไรขึ้น?**
A: PostgreSQL จะปฏิเสธคำสั่งทันทีด้วย error
`CREATE INDEX CONCURRENTLY cannot run inside a transaction block` และ migration
ทั้งไฟล์จะล้มเหลว (ไม่มีอะไรถูกสร้างเลย เพราะ Django ยกเลิก transaction ทั้งหมด)
วิธีแก้คือเพิ่ม `atomic = False` ที่ระดับ `class Migration` ตามขั้นตอนที่ 154.4
แล้วรันใหม่

**Q: จำเป็นต้องเขียน test สำหรับ data migration ทุกตัวเลยหรือไม่ แม้จะเป็น
migration ง่าย ๆ แค่ seed ข้อมูลอ้างอิงคงที่?**
A: สำหรับ migration ที่ seed ข้อมูลคงที่ล้วน ๆ (ไม่มี logic เงื่อนไขซับซ้อน) การ
เขียน test เต็มรูปแบบอาจไม่คุ้มเวลา แต่สำหรับ data migration ที่มี**เงื่อนไขหรือการ
คำนวณ** (เช่น backfill slug ที่ต้องเช็คความซ้ำ, แปลงค่าตามเงื่อนไขธุรกิจ) ควรเขียน
test เสมอ เพราะ bug ใน logic เหล่านี้กระทบข้อมูลจริงโดยตรงและมักตรวจพบยากถ้าไม่มี
test ครอบคลุมไว้ก่อน

### 160.6 เตรียมตัวสำหรับ Part ถัดไป

**Part 017: Django Admin เบื้องต้น: ModelAdmin** จะพาคุณกลับมาที่ฝั่งที่ใช้งานง่าย
และเห็นผลทันทีอีกครั้ง หลังจากเจาะลึกเรื่อง migration ที่ค่อนข้างเป็นงาน "เบื้องหลัง"
มาตลอด Part 011-016 คุณจะได้เปิดใช้งาน **Django Admin** — ระบบจัดการข้อมูลผ่านหน้า
เว็บที่ Django สร้างให้อัตโนมัติจาก model ที่คุณเขียนไว้แล้วทั้งหมด (`Post`,
`Category`, `Tag`, `Comment`, `Profile`) โดยแทบไม่ต้องเขียนโค้ดเพิ่มเลย เราจะ:

- ลงทะเบียน model เข้า Django Admin ด้วย `admin.site.register()`
- ปรับแต่งหน้ารายการด้วย `list_display`, `list_filter`, `search_fields`
- ปรับแต่งฟอร์มแก้ไขด้วย `fields`, `fieldsets`, `readonly_fields`
- ใช้ `ModelAdmin` แบบเต็มรูปแบบแทนการ register model เปล่า ๆ

ทักษะการเขียน migration ที่ปลอดภัยซึ่งคุณเพิ่งฝึกมาทั้ง Part นี้จะยังคงสำคัญอยู่
เบื้องหลังทุกครั้งที่แก้ไข model ต่อจากนี้ไป — Django Admin เองก็อ่านโครงสร้าง
จาก model ที่ migrate เรียบร้อยแล้วเช่นกัน แล้วไปเจาะลึกการสร้างหน้า Admin ระดับ
มืออาชีพกันต่อใน Part ถัดไป!
