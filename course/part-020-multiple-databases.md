# Part 020: Multiple Databases และ Database Routing

> **ขั้นตอนที่ 191-200 ของหลักสูตร** | Phase 2: Models, ORM และ Admin (Part นี้คือ Part
> สุดท้ายของ Phase 2)
>
> เป้าหมายของ Part นี้: เรียนรู้วิธีให้โปรเจกต์ Django หนึ่งตัวคุยกับ **มากกว่าหนึ่งฐานข้อมูล
> พร้อมกัน** ตั้งแต่การตั้งค่า `DATABASES` หลายตัว, สั่ง query ไปยังฐานข้อมูลที่ต้องการด้วย
> `.using()`, เขียน **Database Router** เองเพื่อให้ Django ตัดสินใจอัตโนมัติว่า query ไหน
> ควรไปที่ไหน, ทำความเข้าใจรูปแบบ **Read Replica** ที่ใช้จริงในระบบ production ระดับโลก,
> เชื่อมต่อฐานข้อมูล legacy ที่มีอยู่แล้วด้วย `inspectdb`, รู้จักข้อจำกัดของ cross-database
> relations, เขียนเทสต์ที่ครอบคลุมหลายฐานข้อมูล, และปิดท้ายด้วย **บทสรุปใหญ่ของ Phase 2
> ทั้งหมด** (Part 011-020) ก่อนก้าวเข้าสู่ Phase 3 ที่จะเจาะลึกเรื่อง Views, Templates,
> Forms และ Class-Based Views

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 191: ตั้งค่า `DATABASES` ให้มีมากกว่า 1 ฐานข้อมูล
- ขั้นตอนที่ 192: สั่ง query ไปยังฐานข้อมูลที่ต้องการด้วย `.using('db_alias')`
- ขั้นตอนที่ 193: เขียน Database Router class เอง
- ขั้นตอนที่ 194: รูปแบบ Read Replica (Primary/Replica) พร้อม Router ตัวอย่างจริง
- ขั้นตอนที่ 195: เชื่อมต่อฐานข้อมูล Legacy ด้วย `inspectdb` และ `managed = False`
- ขั้นตอนที่ 196: เกริ่นแนวคิด Database Sharding
- ขั้นตอนที่ 197: ข้อจำกัดของ Cross-Database Relations
- ขั้นตอนที่ 198: การเขียน Test ที่เกี่ยวข้องกับหลายฐานข้อมูล
- ขั้นตอนที่ 199: เกริ่น Connection Pooling (PgBouncer)
- ขั้นตอนที่ 200: **สรุป Phase 2 ทั้งหมด** (Part 011-020) พร้อม Quiz และแบบฝึกหัดใหญ่ปิด Phase

---

## ขั้นตอนที่ 191: ตั้งค่า `DATABASES` ให้มีมากกว่า 1 ฐานข้อมูล

### 191.1 ทบทวนสถานะโปรเจกต์ก่อนเริ่ม Part นี้

ก่อนหน้านี้ตลอด Phase 2 โปรเจกต์บล็อกของคุณมี `settings.py` ที่ตั้งค่า `DATABASES` แบบ
มาตรฐานที่สุด — มีฐานข้อมูลเดียวชื่อ `default`:

```python
# config/settings.py (ก่อนหน้า Part นี้)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'blogdb'),
        'USER': os.environ.get('DB_USER', 'blogadmin'),
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

โมเดลทุกตัวที่คุณสร้างมา (`Post`, `Category`, `Tag`, `PostTag`, `Comment`, `Profile`)
ถูกอ่าน/เขียนที่ฐานข้อมูล `default` เพียงแหล่งเดียวมาโดยตลอด นี่คือรูปแบบที่ใช้ได้กับโปรเจกต์
ส่วนใหญ่ตลอดชีวิตของมัน แต่เมื่อระบบเติบโตขึ้น มีเหตุผลทางธุรกิจหลายแบบที่ทำให้ **หนึ่ง
โปรเจกต์ Django ต้องคุยกับหลายฐานข้อมูลพร้อมกัน**

### 191.2 เหตุผลที่ระบบจริงต้องใช้หลายฐานข้อมูล

| สถานการณ์ | ตัวอย่างจริง |
|---|---|
| แยกข้อมูล Analytics/Log ออกจากข้อมูลธุรกิจหลัก | เก็บ view count, click log ไว้อีกฐานข้อมูล ไม่ให้กระทบ performance ของฐานข้อมูลหลัก |
| Read Replica เพื่อกระจายโหลดการอ่าน | เว็บทราฟฟิกสูงอ่านจาก replica หลายตัว เขียนเข้า primary ตัวเดียว |
| เชื่อมต่อฐานข้อมูล Legacy ที่มีอยู่ก่อนแล้ว | ระบบ HR เก่าที่เขียนด้วยภาษาอื่น แต่ต้องดึงข้อมูลพนักงานมาแสดงใน Django |
| แยกข้อมูลตาม Tenant/ลูกค้า (Multi-tenancy แบบ database-per-tenant) | SaaS ที่แต่ละบริษัทลูกค้ามีฐานข้อมูลแยกกันเพื่อความเป็นส่วนตัวสูงสุด |
| แยกฐานข้อมูลตามภูมิภาค (Sharding ตาม region) | ระบบระดับโลกที่เก็บข้อมูลผู้ใช้ยุโรปไว้ในสหภาพยุโรปตามกฎหมาย GDPR |

Django ไม่ได้บังคับให้เลือกอย่างใดอย่างหนึ่ง — ORM ของ Django ถูกออกแบบมาให้รองรับ
สถานการณ์เหล่านี้ทั้งหมดผ่านกลไกเดียวกัน คือ `DATABASES` dict ที่รองรับหลาย key และ
**Database Router** ที่เราจะเรียนตลอด Part นี้

### 191.3 เพิ่มฐานข้อมูล `analytics` เข้าไปใน `DATABASES`

สมมติสถานการณ์จริง: ทีมของคุณต้องการเก็บ **log การเข้าชมบทความ (page view log)** ซึ่งมี
ปริมาณข้อมูลมหาศาลและถูกเขียนบ่อยมาก (ทุกครั้งที่มีคนเปิดหน้าบทความ) ถ้าเก็บไว้ในฐานข้อมูล
เดียวกับ `Post`/`Comment` จะทำให้ตารางธุรกิจหลักถูกรบกวนจาก write load มหาศาลของ log
วิธีแก้ที่เป็นมาตรฐานอุตสาหกรรมคือ **แยกฐานข้อมูลสำหรับ analytics ออกไปต่างหาก**:

```python
# config/settings.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'blogdb'),
        'USER': os.environ.get('DB_USER', 'blogadmin'),
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 60,
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('ANALYTICS_DB_NAME', 'blog_analytics'),
        'USER': os.environ.get('ANALYTICS_DB_USER', 'analytics_user'),
        'PASSWORD': os.environ.get('ANALYTICS_DB_PASSWORD', ''),
        'HOST': os.environ.get('ANALYTICS_DB_HOST', 'localhost'),
        'PORT': os.environ.get('ANALYTICS_DB_PORT', '5432'),
        'CONN_MAX_AGE': 60,
    },
}
```

**หลักการสำคัญที่สุดที่ต้องเข้าใจตั้งแต่บรรทัดแรก**: ทุก key ใน `DATABASES` คือ
**database alias** ที่คุณตั้งชื่อเองได้ตามใจ (`'default'`, `'analytics'`, `'replica1'`,
`'legacy_hr'` ฯลฯ) — **ยกเว้น `'default'` ที่ต้องมีเสมอเป็นข้อบังคับ** เพราะ Django ใช้
`'default'` เป็นค่าตั้งต้นทุกครั้งที่คุณไม่ได้ระบุฐานข้อมูลไว้ชัดเจน (เช่น `Post.objects.all()`
ธรรมดา จะวิ่งไปที่ `default` เสมอถ้าไม่มี router เข้ามาแทรกแซง — เรื่องนี้จะชัดเจนขึ้นมากใน
ขั้นตอนที่ 193)

### 191.4 สร้างฐานข้อมูล `analytics` จริงใน PostgreSQL

```bash
# เข้า psql ในฐานะ superuser
psql -U postgres

# สร้างฐานข้อมูลและผู้ใช้แยกสำหรับ analytics
CREATE DATABASE blog_analytics;
CREATE USER analytics_user WITH PASSWORD 'your-secure-password';
GRANT ALL PRIVILEGES ON DATABASE blog_analytics TO analytics_user;
\q
```

เพิ่มตัวแปรใน `.env` (ตามแนวทางที่วางไว้ตั้งแต่ Part 010: Django Settings และ Environment
Configuration):

```bash
# .env
ANALYTICS_DB_NAME=blog_analytics
ANALYTICS_DB_USER=analytics_user
ANALYTICS_DB_PASSWORD=your-secure-password
ANALYTICS_DB_HOST=localhost
ANALYTICS_DB_PORT=5432
```

### 191.5 สร้างแอป `analytics` และโมเดล `PageView`

```bash
python manage.py startapp analytics
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'blog',
    'accounts',
    'analytics',
]
```

```python
# analytics/models.py
from django.db import models


class PageView(models.Model):
    """เก็บ log การเข้าชมบทความ 1 แถวต่อ 1 ครั้งที่มีคนเปิดหน้าบทความ

    ตั้งใจไม่ใช้ ForeignKey ไปยัง blog.Post ตรง ๆ เพราะโมเดลนี้อยู่คนละฐานข้อมูล
    (เหตุผลเต็มรูปแบบอยู่ในขั้นตอนที่ 197: ข้อจำกัดของ Cross-Database Relations)
    """
    post_id = models.PositiveIntegerField(db_index=True)
    post_title_snapshot = models.CharField(max_length=200)
    viewer_ip = models.GenericIPAddressField(null=True, blank=True)
    user_agent = models.CharField(max_length=300, blank=True)
    viewed_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-viewed_at']
        indexes = [
            models.Index(fields=['post_id', 'viewed_at']),
        ]

    def __str__(self):
        return f'View ของ post #{self.post_id} เมื่อ {self.viewed_at:%Y-%m-%d %H:%M}'
```

สังเกตว่าเราเก็บ `post_id` เป็น `PositiveIntegerField` ธรรมดา **ไม่ใช่ `ForeignKey`** —
นี่ไม่ใช่ความผิดพลาด แต่เป็นการตัดสินใจตั้งใจที่จำเป็นเมื่อสองโมเดลอยู่กันคนละฐานข้อมูล
เราจะอธิบายเหตุผลทั้งหมดอย่างละเอียดในขั้นตอนที่ 197

### 191.6 เมื่อมีหลายฐานข้อมูล คำสั่ง `migrate` ต้องระบุ `--database`

```bash
# migrate ฐานข้อมูล default (blog, accounts) ตามปกติ
python manage.py migrate

# migrate ฐานข้อมูล analytics แยกต่างหาก
python manage.py makemigrations analytics
python manage.py migrate analytics --database=analytics
```

ถ้าลืมใส่ `--database=analytics` คำสั่ง `migrate analytics` จะพยายามสร้างตาราง
`analytics_pageview` ใน `default` แทน (พฤติกรรมเริ่มต้นของ Django ถ้าไม่มี router
บอกให้ทำอย่างอื่น) ซึ่งไม่ใช่สิ่งที่เราต้องการเลย — นี่คือปัญหาที่ **Database Router**
(ขั้นตอนที่ 193) จะเข้ามาแก้ให้อัตโนมัติ ไม่ต้องจำ `--database` ทุกครั้งอีกต่อไป

### 191.7 ตรวจสอบว่าตารางถูกสร้างที่ฐานข้อมูลถูกต้อง

```bash
psql -U analytics_user -d blog_analytics -c "\dt"
```

```
              List of relations
 Schema |          Name           | Type  |     Owner
--------+--------------------------+-------+----------------
 public | analytics_pageview       | table | analytics_user
 public | django_migrations        | table | analytics_user
```

สังเกตว่า `django_migrations` ก็ถูกสร้างแยกในฐานข้อมูล `analytics` เองด้วย — Django
ติดตาม migration state **แยกต่อฐานข้อมูล** เสมอ ไม่ใช่ตารางเดียวที่ใช้ร่วมกันทุกฐานข้อมูล

---

## ขั้นตอนที่ 192: สั่ง query ไปยังฐานข้อมูลที่ต้องการด้วย `.using('db_alias')`

### 192.1 ปัญหา: ถ้าไม่บอก Django ว่าจะ query ที่ไหน

ถ้ายังไม่มี router ตั้งค่าไว้ (เราจะเขียนใน ขั้นตอนที่ 193) การเรียก ORM ทุกครั้งจะวิ่งไปที่
`default` เสมอ แม้ว่าโมเดลนั้นจะถูกออกแบบมาให้อยู่ที่ฐานข้อมูลอื่นก็ตาม:

```python
>>> from analytics.models import PageView
>>> PageView.objects.create(post_id=1, post_title_snapshot='แนะนำ Django 5')
# ถ้ายังไม่มี router: แถวนี้จะถูกเขียนไปที่ 'default' ไม่ใช่ 'analytics'!
```

`.using()` คือเครื่องมือที่ให้คุณ **สั่งอย่างชัดเจน (explicit)** ว่าต้องการให้ query นี้วิ่งไป
ที่ฐานข้อมูล alias ไหน โดยไม่ต้องพึ่ง router เลยก็ได้

### 192.2 `.using()` กับการอ่านข้อมูล (QuerySet)

```python
python manage.py shell
```

```python
>>> from analytics.models import PageView

# บังคับให้ query วิ่งไปที่ฐานข้อมูล 'analytics' อย่างชัดเจน
>>> PageView.objects.using('analytics').all()
<QuerySet []>

>>> PageView.objects.using('analytics').filter(post_id=1).count()
0

# .using() ต่อท้ายกับ QuerySet method อื่น ๆ ได้ตามปกติทุกตัว
>>> PageView.objects.using('analytics').filter(post_id=1).order_by('-viewed_at')[:5]
<QuerySet []>
```

### 192.3 `.using()` กับการเขียนข้อมูล: `save()`

```python
>>> view_log = PageView(
...     post_id=1,
...     post_title_snapshot='แนะนำ Django 5',
...     viewer_ip='203.0.113.42',
...     user_agent='Mozilla/5.0 ...',
... )
>>> view_log.save(using='analytics')
>>> PageView.objects.using('analytics').count()
1
```

`save(using='analytics')` บอก Django ว่า **แถวนี้ต้องถูกเขียนไปที่ฐานข้อมูลไหน** ทั้งตอน
`INSERT` ครั้งแรกและ `UPDATE` ครั้งต่อ ๆ ไป — ถ้าไม่ระบุ Django จะพยายามใช้ฐานข้อมูล
เดียวกับที่ instance นั้นถูกโหลดมา (ถ้าโหลดมาจาก `.using('analytics')` ก็จะจำค่านั้นไว้
อัตโนมัติ) หรือ fallback ไปที่ `default`/router

### 192.4 `.using()` กับ `create()`

`create()` เป็นวิธีลัดของ `.save()` แต่ **ไม่รองรับ `using=` โดยตรงเป็นคีย์เวิร์ด** — ต้อง
เรียกผ่าน manager ที่ผูก `.using()` ไว้ก่อน:

```python
>>> PageView.objects.using('analytics').create(
...     post_id=2,
...     post_title_snapshot='รีวิว PostgreSQL 16',
...     viewer_ip='198.51.100.7',
... )
<PageView: View ของ post #2 เมื่อ 2026-03-14 10:22>
```

### 192.5 `.using()` กับการลบข้อมูล: `delete()`

```python
>>> old_log = PageView.objects.using('analytics').get(post_id=2)
>>> old_log.delete(using='analytics')
(1, {'analytics.PageView': 1})

# ลบทั้ง QuerySet พร้อมกันก็ทำได้เช่นกัน
>>> PageView.objects.using('analytics').filter(post_id=1).delete()
(1, {'analytics.PageView': 1})
```

> **ข้อควรระวังสำคัญ**: `instance.delete(using='analytics')` ต้องระบุ `using` ตรง ๆ
> เพราะ instance object เองไม่ได้ "จำ" เสมอไปว่าตัวเองมาจากฐานข้อมูลไหน โดยเฉพาะถ้า
> instance นั้นถูกสร้างขึ้นมาเองในโค้ด (ไม่ได้โหลดผ่าน queryset) — การลืมระบุ `using`
> ตรงนี้เป็นบั๊กที่พบบ่อยที่สุดในโปรเจกต์ multi-database ที่ยังไม่มี router: มันจะพยายาม
> ลบแถวที่ pk เดียวกันใน `default` แทน ซึ่งมักจะ raise `DoesNotExist` เพราะแถวนั้นไม่มี
> อยู่จริงในฐานข้อมูลนั้น

### 192.6 `.using()` กับความสัมพันธ์ (Related Objects)

จุดที่มือใหม่มักพลาด: การเข้าถึงความสัมพันธ์ผ่าน object ที่โหลดมาด้วย `.using()` **ไม่ได้
สืบทอด alias นั้นไปอัตโนมัติเสมอไป** ในทุกกรณี ต้องระบุซ้ำเมื่อจำเป็น:

```python
>>> post = Post.objects.using('default').get(pk=1)

# .comments เป็น related manager ธรรมดา จะ query ที่ default (เพราะ Comment อยู่ default)
>>> post.comments.all()
<QuerySet [...]>

# ถ้า Comment เคยอยู่ analytics จะต้องเขียน
>>> post.comments.using('analytics').all()   # ตัวอย่างสมมติ ถ้า Comment ข้ามฐานข้อมูล
```

ในกรณีของเรา `Post`, `Category`, `Tag`, `Comment`, `Profile` ทั้งหมดอยู่ `default`
เหมือนเดิม มีแค่ `PageView` เท่านั้นที่อยู่ `analytics` แยกออกไป จึงไม่มีปัญหาความสัมพันธ์
ข้ามฐานข้อมูลเกิดขึ้นในโปรเจกต์นี้ — แต่ถ้าออกแบบผิดจนมี `ForeignKey` ข้ามฐานข้อมูลจริง
จะพบปัญหาที่ร้ายแรงกว่านี้มาก ซึ่งเราจะอธิบายเต็มรูปแบบในขั้นตอนที่ 197

### 192.7 ตารางสรุปเมธอดที่รองรับการระบุฐานข้อมูล

| เมธอด | วิธีระบุฐานข้อมูล | ตัวอย่าง |
|---|---|---|
| Query (อ่าน) | `.using('alias')` ต่อท้าย manager/queryset | `Model.objects.using('analytics').all()` |
| `save()` | คีย์เวิร์ด `using=` | `obj.save(using='analytics')` |
| `create()` | ผ่าน `.using()` ก่อนเรียก `create()` | `Model.objects.using('analytics').create(...)` |
| `delete()` (instance) | คีย์เวิร์ด `using=` | `obj.delete(using='analytics')` |
| `delete()` (queryset) | `.using()` ต่อท้าย queryset | `Model.objects.using('analytics').filter(...).delete()` |
| `bulk_create()` | ผ่าน `.using()` ก่อนเรียก | `Model.objects.using('analytics').bulk_create([...])` |

การเขียน `.using('analytics')` ทุกที่ที่ต้องแตะ `PageView` **ทำงานได้ถูกต้อง แต่ซ้ำซาก
และเสี่ยงลืม** — วิธีแก้ที่เป็นมาตรฐานอุตสาหกรรมคือให้ **Router ตัดสินใจแทนอัตโนมัติ**
ซึ่งเป็นหัวข้อหลักของขั้นตอนถัดไป

---

## ขั้นตอนที่ 193: เขียน Database Router Class เอง

### 193.1 Database Router คืออะไร

**Database Router** คือ Python class ธรรมดาที่ Django เรียกใช้ทุกครั้งที่ทำ query, write,
หรือ migrate โดยไม่ต้องให้คุณเขียน `.using()` ซ้ำ ๆ ทุกที่ในโค้ด — router ตอบคำถาม 4 ข้อ
ให้ Django ผ่าน method 4 ตัว:

| Method | Django ถามว่า | คืนค่าอะไร |
|---|---|---|
| `db_for_read(model, **hints)` | "โมเดลนี้ควรอ่านจากฐานข้อมูลไหน" | alias (string) หรือ `None` |
| `db_for_write(model, **hints)` | "โมเดลนี้ควรเขียนไปที่ฐานข้อมูลไหน" | alias (string) หรือ `None` |
| `allow_relation(obj1, obj2, **hints)` | "obj1 กับ obj2 อนุญาตให้มีความสัมพันธ์กันไหม" | `True`/`False`/`None` |
| `allow_migrate(db, app_label, model_name=None, **hints)` | "แอปนี้ควร migrate ลงฐานข้อมูล db นี้ไหม" | `True`/`False`/`None` |

**คืนค่า `None` หมายถึง "ฉันไม่มีความเห็น ให้ router ตัวถัดไปในลิสต์ (หรือค่า default ของ
Django) ตัดสินใจแทน"** — นี่คือหลักการสำคัญที่ทำให้ router หลายตัวทำงานร่วมกันได้แบบ chain

### 193.2 เขียน Router ตัวแรก: `AnalyticsRouter`

```python
# analytics/routers.py
class AnalyticsRouter:
    """กำหนดเส้นทางให้ทุกโมเดลของแอป analytics วิ่งไปที่ฐานข้อมูล 'analytics' เสมอ
    โมเดลของแอปอื่นทั้งหมดปล่อยให้ router อื่น (หรือ default) จัดการต่อไป
    """

    route_app_labels = {'analytics'}

    def db_for_read(self, model, **hints):
        if model._meta.app_label in self.route_app_labels:
            return 'analytics'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label in self.route_app_labels:
            return 'analytics'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        db_set = {'default', 'analytics'}
        if obj1._state.db in db_set and obj2._state.db in db_set:
            return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if app_label in self.route_app_labels:
            return db == 'analytics'
        return None
```

### 193.3 ลงทะเบียน Router ใน `settings.py`

```python
# config/settings.py
DATABASE_ROUTERS = ['analytics.routers.AnalyticsRouter']
```

`DATABASE_ROUTERS` เป็น **list** เสมอ (แม้จะมี router เดียว) เพราะ Django รองรับการ
วาง router หลายตัวเรียงกันเป็นลำดับ (chain) — Django จะไล่ถามทีละตัวตามลำดับในลิสต์
จนกว่าจะมีตัวใดตัวหนึ่งคืนค่าที่ไม่ใช่ `None`

### 193.4 ทดสอบผลลัพธ์หลังมี Router

```bash
python manage.py shell
```

```python
>>> from analytics.models import PageView
>>> from blog.models import Post

# ตอนนี้ไม่ต้องเขียน .using('analytics') อีกต่อไป — router จัดการให้อัตโนมัติ!
>>> PageView.objects.create(post_id=1, post_title_snapshot='แนะนำ Django 5')
<PageView: View ของ post #1 เมื่อ 2026-03-14 11:05>

>>> PageView.objects.count()   # ไม่ต้องใส่ .using() แล้ว router รู้เองว่าไปดูที่ analytics
1

# Post ยังคงวิ่งไปที่ default ตามปกติ เพราะ router คืน None ให้แอป blog
>>> Post.objects.count()
5
```

### 193.5 ทดสอบ `allow_migrate` — `migrate` ทำงานถูกที่โดยไม่ต้องระบุ `--database`

```bash
# ตอนนี้ไม่ต้องพิมพ์ --database=analytics อีกแล้ว router รู้เองว่า analytics app
# ต้องไปที่ไหน ส่วนแอปอื่นก็ยังไปที่ default ตามปกติ
python manage.py migrate
```

```
Operations to perform:
  Apply all migrations: accounts, admin, analytics, auth, blog, contenttypes, sessions
Running migrations:
  ...
```

Django จะรัน migration ของแต่ละแอปกับ **ทุกฐานข้อมูลที่ตั้งค่าไว้** เสมอ แล้วถาม
`allow_migrate()` ของ router ทุกครั้งก่อนตัดสินใจว่าจะสร้างตารางจริงหรือไม่ — สำหรับแอป
`analytics` router จะตอบ `True` เฉพาะตอนที่ `db == 'analytics'` เท่านั้น (และ `False`
เมื่อ `db == 'default'`) ส่วนแอปอื่น router คืน `None` ปล่อยให้ Django ใช้พฤติกรรม
default (ซึ่งคือ migrate ลง `default` ตามปกติ)

### 193.6 Hints คืออะไร ใช้ตอนไหน

พารามิเตอร์ `**hints` ที่ทุก method รับเข้ามา คือข้อมูลบริบทเพิ่มเติมที่ Django ส่งมาให้
ในบางสถานการณ์ ที่พบบ่อยที่สุดคือ `hints['instance']` ตอนบันทึกความสัมพันธ์:

```python
class SmartRouter:
    def db_for_write(self, model, **hints):
        instance = hints.get('instance')
        if instance is not None:
            # ตัวอย่าง: ถ้ากำลังบันทึก object ที่สัมพันธ์กับ instance ที่มาจาก analytics
            # อยู่แล้ว ให้เขียนไปที่ analytics ด้วยเพื่อความสอดคล้อง
            if instance._state.db == 'analytics':
                return 'analytics'
        return None
```

ในโปรเจกต์ของเรายังไม่จำเป็นต้องใช้ hints ระดับนี้ เพราะ router ตัดสินใจจาก
`app_label` ได้ตรงไปตรงมาอยู่แล้ว แต่ในระบบที่ซับซ้อนกว่า (เช่น multi-tenant) hints
คือกลไกสำคัญที่ทำให้ router "ฉลาด" ขึ้นได้มาก

### 193.7 ตารางสรุป Method ทั้ง 4 ของ Router

| Method | ถูกเรียกตอนไหน | ค่า default ถ้าทุก router คืน `None` |
|---|---|---|
| `db_for_read` | ทุกครั้งที่ query ข้อมูล (`.all()`, `.get()`, `.filter()` ฯลฯ) | `'default'` |
| `db_for_write` | ทุกครั้งที่ `save()`, `create()`, `delete()`, `update()` | `'default'` |
| `allow_relation` | ตอนตรวจสอบความสัมพันธ์ระหว่าง object สองตัว (เช่น ตอน validate FK) | `True` ถ้า object ทั้งคู่มาจาก `default` |
| `allow_migrate` | ทุกครั้งที่รัน `migrate` เช็คทีละแอป-ทีละฐานข้อมูล | `True` เฉพาะฐานข้อมูล `default` |

---

## ขั้นตอนที่ 194: รูปแบบ Read Replica (Primary สำหรับ Write, Replica สำหรับ Read)

### 194.1 Read Replica คืออะไร และทำไมระบบระดับโลกใช้กันแทบทุกที่

เมื่อเว็บไซต์มีทราฟฟิกสูงมาก ปัญหาที่พบบ่อยที่สุดคือ **การอ่านข้อมูล (read) มีปริมาณมากกว่า
การเขียน (write) หลายสิบถึงหลายร้อยเท่า** เช่น บล็อกหนึ่งบทความถูกเขียนครั้งเดียว แต่ถูก
อ่านหลายพันครั้งต่อวัน ถ้าให้ฐานข้อมูลตัวเดียวรับทั้ง read และ write ทั้งหมด ฐานข้อมูลนั้น
จะกลายเป็นคอขวด (bottleneck) ของทั้งระบบ

**Read Replica** คือฐานข้อมูลสำเนา (copy) ของฐานข้อมูลหลัก ที่ PostgreSQL/MySQL
คอยซิงค์ข้อมูลให้ตรงกันแบบเกือบเรียลไทม์ (streaming replication) โดยมีกฎสำคัญคือ:

```
┌─────────────┐   เขียนได้เท่านั้น    ┌──────────────────┐
│   Primary   │ ◄──────────────────  │  Django (Write)   │
│  (เขียนได้)  │                       └──────────────────┘
└──────┬──────┘
       │ streaming replication (ซิงค์ข้อมูลอัตโนมัติ)
       ▼
┌─────────────┐   อ่านได้เท่านั้น     ┌──────────────────┐
│  Replica 1  │ ─────────────────►   │  Django (Read)     │
│ (อ่านอย่างเดียว) │                    └──────────────────┘
└─────────────┘
┌─────────────┐   อ่านได้เท่านั้น     ┌──────────────────┐
│  Replica 2  │ ─────────────────►   │  Django (Read)     │
│ (อ่านอย่างเดียว) │                    └──────────────────┘
└─────────────┘
```

- **Primary (หรือเรียก Master)**: รับ **write ทั้งหมด** เท่านั้น (INSERT, UPDATE,
  DELETE) เป็นแหล่งข้อมูลจริงเพียงแหล่งเดียว (single source of truth)
- **Replica (หรือเรียก Slave/Standby)**: รับ **read ทั้งหมด** เท่านั้น มีได้หลายตัวพร้อมกัน
  เพื่อกระจายโหลดการอ่าน (horizontal scaling ฝั่ง read)
- การกระจาย read ไปยังหลาย replica เรียกว่า **Load Balancing** ระดับฐานข้อมูล

### 194.2 ตั้งค่า `DATABASES` แบบ Primary + Replica

```python
# config/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'USER': 'blogadmin',
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': os.environ.get('DB_PRIMARY_HOST', 'primary.db.internal'),
        'PORT': '5432',
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'USER': 'blogreader',
        'PASSWORD': os.environ.get('DB_REPLICA_PASSWORD', ''),
        'HOST': os.environ.get('DB_REPLICA1_HOST', 'replica1.db.internal'),
        'PORT': '5432',
    },
    'replica2': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'USER': 'blogreader',
        'PASSWORD': os.environ.get('DB_REPLICA_PASSWORD', ''),
        'HOST': os.environ.get('DB_REPLICA2_HOST', 'replica2.db.internal'),
        'PORT': '5432',
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blog_analytics',
        'USER': 'analytics_user',
        'PASSWORD': os.environ.get('ANALYTICS_DB_PASSWORD', ''),
        'HOST': 'localhost',
        'PORT': '5432',
    },
}
```

สังเกตว่า `replica1` และ `replica2` ใช้ `NAME` เดียวกับ `default` (`blogdb`) เพราะมัน
คือ**สำเนาของฐานข้อมูลเดียวกัน** เพียงแต่อยู่คนละเครื่อง (`HOST` ต่างกัน) และใช้ user
ที่มีสิทธิ์ **อ่านอย่างเดียว** (`blogreader`) เพื่อความปลอดภัย — แม้โค้ดจะพลาดเขียนไปที่
replica ฐานข้อมูลจริงก็จะปฏิเสธด้วย permission error ซ้ำอีกชั้นหนึ่ง (defense in depth)

### 194.3 เขียน `PrimaryReplicaRouter`

```python
# config/routers.py
import random


class PrimaryReplicaRouter:
    """Route การอ่านทั้งหมดไปยัง replica แบบสุ่ม (round-robin อย่างง่าย)
    และ route การเขียนทั้งหมดไปยัง primary ('default') เสมอ
    """

    replica_aliases = ['replica1', 'replica2']

    def db_for_read(self, model, **hints):
        return random.choice(self.replica_aliases)

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        db_list = ('default', *self.replica_aliases)
        if obj1._state.db in db_list and obj2._state.db in db_list:
            return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        # Migration schema ควรรันที่ primary เท่านั้น
        # replica จะได้ schema มาจากการ replicate ข้อมูลจาก primary เองอัตโนมัติ
        return db == 'default'
```

```python
# config/settings.py
DATABASE_ROUTERS = [
    'analytics.routers.AnalyticsRouter',
    'config.routers.PrimaryReplicaRouter',
]
```

สังเกตลำดับ: `AnalyticsRouter` มาก่อน เพราะต้องดักโมเดลของแอป `analytics` ให้ตรงเงื่อนไข
`app_label` ก่อน ถ้า `AnalyticsRouter` คืน `None` (โมเดลนั้นไม่ใช่ของแอป analytics)
Django จะไปถาม `PrimaryReplicaRouter` ต่อ ซึ่งจะจัดการ read/write ระหว่าง primary กับ
replica ให้กับโมเดลของแอปอื่นทั้งหมด (`blog`, `accounts`)

### 194.4 ปัญหาสำคัญที่ต้องรู้: Replication Lag

**Replication Lag** คือช่วงเวลาสั้น ๆ ที่ replica ยังไม่ได้รับข้อมูลใหม่ล่าสุดจาก primary
(ปกติเป็นหลักมิลลิวินาทีถึงหลักวินาที) ทำให้เกิดสถานการณ์ที่น่าปวดหัว:

```python
# View ตัวอย่าง: สร้าง Post ใหม่แล้ว redirect ไปหน้า detail ทันที
def post_create_view(request):
    post = Post.objects.create(title='บทความใหม่', content='...')  # เขียนไปที่ primary
    return redirect('post_detail', slug=post.slug)


def post_detail_view(request, slug):
    # อ่านจาก replica ที่ "ยังไม่ทันซิงค์" ข้อมูลที่เพิ่งเขียนไปเมื่อครู่นี้!
    post = get_object_or_404(Post, slug=slug)   # อาจได้ DoesNotExist ชั่วคราว!
    return render(request, 'blog/detail.html', {'post': post})
```

**วิธีแก้ที่ใช้จริงในระบบ production**:

1. **Read-your-writes consistency**: หลังจากเขียนข้อมูลในคำขอ (request) เดียวกัน ให้
   บังคับอ่านจาก `default` (primary) แทน replica ในคำขอนั้น มักทำผ่าน middleware ที่
   สลับ router ชั่วคราว หรือใช้ `.using('default')` ตรง ๆ ในจุดที่รู้ว่าเสี่ยง
2. **Sticky session ระดับ database**: จำไว้ว่า session นี้เพิ่งเขียนข้อมูล ให้อ่านจาก
   primary ไปจนครบ time window สั้น ๆ (เช่น 2-3 วินาที) ก่อนกลับไปอ่าน replica ตามปกติ
3. **ยอมรับ eventual consistency**: สำหรับหน้าที่ไม่ critical (เช่น หน้ารายการบทความ
   ทั่วไป) การเห็นข้อมูลช้ากว่าจริงเสี้ยววินาทีมักไม่กระทบผู้ใช้จริง

```python
# ตัวอย่างการแก้ปัญหาแบบง่ายที่สุด: บังคับอ่านจาก default ทันทีหลังเขียน
def post_create_view(request):
    post = Post.objects.create(title='บทความใหม่', content='...')
    return redirect('post_detail', slug=post.slug)


def post_detail_view(request, slug):
    # บังคับอ่านจาก primary เฉพาะจุดที่มีความเสี่ยงสูงว่าข้อมูลอาจยังไม่ sync
    post = Post.objects.using('default').get(slug=slug)
    return render(request, 'blog/detail.html', {'post': post})
```

> **มองไปข้างหน้า**: การจัดการ Replication Lag อย่างเป็นระบบเต็มรูปแบบ (เช่น ใช้
> middleware ที่ track "sticky database" ต่อ session, หรือใช้ connection pooler ที่ฉลาด
> พอจะจัดการเรื่องนี้ให้) เป็นหัวข้อระดับ **Performance & Caching (Phase 8)** และ
> **Scaling (Phase 12)** ตอนนี้ขอให้เข้าใจแค่ว่าปัญหานี้**มีอยู่จริง** และรู้จักหลักการ
> แก้เบื้องต้น

### 194.5 ตารางสรุป Primary vs Replica

| ประเด็น | Primary (`default`) | Replica (`replica1`, `replica2`, ...) |
|---|---|---|
| รับ Write ได้ไหม | ✅ ใช่ (เท่านั้นที่ทำได้) | ❌ ไม่ได้ (read-only) |
| รับ Read ได้ไหม | ✅ ได้ (แต่ไม่ควรใช้เป็นหลัก) | ✅ ใช่ (ควรใช้เป็นหลัก) |
| จำนวนตัวในระบบ | 1 ตัวเสมอ | มีได้หลายตัว (scale ตามโหลด read) |
| ข้อมูลอัปเดตล่าสุดไหม | ✅ ล่าสุดเสมอ | ⚠️ อาจมี Replication Lag เล็กน้อย |
| รัน Migration ที่ไหน | ✅ ที่นี่เท่านั้น | ❌ ไม่ต้อง (ได้ schema จากการ replicate) |

---

## ขั้นตอนที่ 195: เชื่อมต่อฐานข้อมูล Legacy ด้วย `inspectdb` และ `managed = False`

### 195.1 สถานการณ์จริง: ฐานข้อมูลเก่าที่มีอยู่ก่อน Django

บริษัทจำนวนมากมีฐานข้อมูล "legacy" ที่มีอยู่ก่อนที่จะเริ่มใช้ Django เช่น ระบบ HR เก่าที่
เขียนด้วย PHP, ระบบบัญชีที่ทีมอื่นดูแล, หรือ data warehouse ที่ทีม Data Engineer จัดการ
เอง — Django สามารถ **"อ่าน" schema ของฐานข้อมูลที่มีอยู่แล้วและสร้างโมเดลให้อัตโนมัติ**
ผ่านคำสั่ง `inspectdb` โดยไม่ต้องรัน migration ใด ๆ ไปทำลาย schema เดิม

### 195.2 ตั้งค่าฐานข้อมูล Legacy ใน `DATABASES`

```python
# config/settings.py
DATABASES = {
    'default': { ... },
    'analytics': { ... },
    'legacy_hr': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'hr_system_legacy',
        'USER': os.environ.get('LEGACY_HR_DB_USER', 'hr_readonly'),
        'PASSWORD': os.environ.get('LEGACY_HR_DB_PASSWORD', ''),
        'HOST': os.environ.get('LEGACY_HR_DB_HOST', 'hr-legacy.internal'),
        'PORT': '5432',
    },
}
```

### 195.3 รัน `inspectdb` เพื่อสร้างโมเดลจาก schema เดิม

```bash
python manage.py inspectdb --database=legacy_hr > accounts/legacy_models.py
```

สมมติฐานข้อมูล `hr_system_legacy` มีตาราง `tbl_employee` อยู่แล้ว (ตั้งชื่อแบบเก่าไม่ตรง
convention ของ Django) ผลลัพธ์ที่ `inspectdb` สร้างให้จะมีลักษณะประมาณนี้:

```python
# accounts/legacy_models.py (ผลลัพธ์ที่ inspectdb สร้างให้ ยังไม่ได้แก้ไข)
# This is an auto-generated Django model module.
# You'll have to do the following manually to clean this up:
#   * Rearrange models' order
#   * Make sure each model has one field with primary_key=True
#   * Remove `managed = False` lines if you wish to allow Django to create,
#     modify, and delete the table
from django.db import models


class TblEmployee(models.Model):
    emp_id = models.AutoField(primary_key=True)
    full_name = models.CharField(max_length=150)
    department_code = models.CharField(max_length=10, blank=True, null=True)
    hire_date = models.DateField(blank=True, null=True)
    salary = models.DecimalField(max_digits=12, decimal_places=2, blank=True, null=True)

    class Meta:
        managed = False
        db_table = 'tbl_employee'
```

### 195.4 ทำความสะอาดโมเดลที่ได้จาก `inspectdb`

โมเดลที่ได้มาตรงมักต้องปรับปรุงเล็กน้อยให้ใช้งานสะดวกขึ้น (แต่ **ห้ามลบ `managed = False`
และ `db_table` เด็ดขาด** — สองบรรทัดนี้คือหัวใจของการเชื่อมต่อฐานข้อมูล legacy):

```python
# accounts/models.py (หลังทำความสะอาด นำมาวางในแอปจริง)
from django.db import models


class LegacyEmployee(models.Model):
    """แทนตาราง tbl_employee ในฐานข้อมูล HR เก่า (อ่านอย่างเดียว)

    managed = False บอก Django ว่า "ห้ามสร้าง/แก้ไข/ลบตารางนี้เด็ดขาด"
    แม้จะรัน makemigrations/migrate ก็ตาม เพราะตารางนี้มีอยู่แล้วและถูกดูแลโดยระบบอื่น
    """
    emp_id = models.AutoField(primary_key=True)
    full_name = models.CharField(max_length=150)
    department_code = models.CharField(max_length=10, blank=True, null=True)
    hire_date = models.DateField(blank=True, null=True)
    salary = models.DecimalField(max_digits=12, decimal_places=2, blank=True, null=True)

    class Meta:
        managed = False
        db_table = 'tbl_employee'
        verbose_name = 'พนักงาน (ระบบ HR เก่า)'

    def __str__(self):
        return self.full_name
```

### 195.5 `managed = False` หมายความว่าอะไรกันแน่

| พฤติกรรม | `managed = True` (ค่า default) | `managed = False` |
|---|---|---|
| `makemigrations` สร้าง migration ให้ไหม | ✅ สร้างให้ | ❌ ไม่สร้าง (ถือว่าตารางมีอยู่แล้ว) |
| `migrate` สร้าง/แก้/ลบตารางไหม | ✅ ทำ | ❌ ไม่แตะต้องตารางเลย |
| Query ข้อมูล (`.objects.all()`) ใช้ได้ปกติไหม | ✅ ใช้ได้ | ✅ ใช้ได้เหมือนกันทุกประการ |
| เขียนข้อมูล (`save()`, `create()`) ใช้ได้ไหม | ✅ ใช้ได้ | ✅ ใช้ได้ (ถ้า DB user มีสิทธิ์เขียน) |
| Django Admin ลงทะเบียนได้ไหม | ✅ ได้ | ✅ ได้เหมือนกัน |

จุดสำคัญที่สุดคือ **`managed = False` ไม่ได้แปลว่า "อ่านอย่างเดียว"** — มันแค่บอกว่า
Django จะไม่ยุ่งกับ**โครงสร้าง (schema)** ของตารางเท่านั้น ส่วนการอ่าน/เขียน**ข้อมูล**ยังทำ
ได้ปกติทุกประการ ถ้าต้องการบังคับ read-only จริง ๆ ต้องทำที่ระดับสิทธิ์ของ database user
(`GRANT SELECT` เท่านั้น ไม่ให้ `INSERT`/`UPDATE`/`DELETE`) ควบคู่ไปด้วย

### 195.6 ใช้งานโมเดล Legacy ร่วมกับ Router

```python
# accounts/routers.py
class LegacyHRRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'accounts' and model.__name__ == 'LegacyEmployee':
            return 'legacy_hr'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'accounts' and model.__name__ == 'LegacyEmployee':
            # ในทางปฏิบัติมักไม่อนุญาตให้เขียนกลับเข้าระบบ legacy จาก Django เลย
            raise PermissionError('ห้ามเขียนข้อมูลกลับเข้าระบบ HR เก่าจาก Django')
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if model_name == 'legacyemployee':
            return False  # ไม่ต้อง migrate เพราะ managed = False อยู่แล้ว แต่กันไว้สองชั้น
        return None
```

```python
>>> from accounts.models import LegacyEmployee
>>> LegacyEmployee.objects.filter(department_code='ENG').count()
14
>>> LegacyEmployee.objects.get(emp_id=101).full_name
'สมชาย ใจดี'
```

---

## ขั้นตอนที่ 196: เกริ่นแนวคิด Database Sharding

### 196.1 Sharding คืออะไร ต่างจาก Read Replica อย่างไร

**Sharding** (บางครั้งเรียก **Horizontal Partitioning**) คือการ **แบ่งข้อมูลออกเป็นส่วน ๆ
(shard) แล้วกระจายเก็บในฐานข้อมูลหลายตัวที่แยกจากกันโดยสิ้นเชิง** — ต่างจาก Read Replica
ที่ทุกตัวมีข้อมูล**ครบชุดเดียวกันหมด** (แค่ทำสำเนา), Sharding ทำให้แต่ละฐานข้อมูลมีข้อมูล
**คนละส่วน** ของทั้งระบบ:

```
Read Replica (ข้อมูลเหมือนกันทุกตัว):
  Primary:  [User 1-1000000]
  Replica1: [User 1-1000000]  (สำเนาเหมือนกันเป๊ะ)
  Replica2: [User 1-1000000]  (สำเนาเหมือนกันเป๊ะ)

Sharding (ข้อมูลคนละส่วนกัน):
  Shard A (Tenant: บริษัท X):     [User 1-50000]
  Shard B (Tenant: บริษัท Y):     [User 50001-120000]
  Shard C (Region: Asia):         [User ในเอเชียทั้งหมด]
  Shard D (Region: Europe):       [User ในยุโรปทั้งหมด]
```

### 196.2 กลยุทธ์การแบ่ง Shard ที่พบบ่อย

| กลยุทธ์ | หลักการ | ตัวอย่าง |
|---|---|---|
| **Sharding ตาม Tenant** | ลูกค้าแต่ละราย (บริษัท) มีฐานข้อมูลแยกกันเอง | SaaS ที่แต่ละบริษัทลูกค้ามี schema/database ของตัวเอง |
| **Sharding ตาม Region** | แบ่งตามภูมิภาคทางภูมิศาสตร์ | ผู้ใช้ยุโรปเก็บที่ data center ในยุโรปตามกฎ GDPR |
| **Sharding ตาม Hash/Range ของ ID** | ใช้สูตรคำนวณ (เช่น `user_id % จำนวน shard`) เพื่อกระจายข้อมูลให้สมดุล | ระบบขนาดใหญ่มาก ๆ ที่ต้องการกระจายโหลดเท่า ๆ กันทุก shard |

### 196.3 ทำไม Django Router รองรับแนวคิดนี้ได้ในหลักการ

Django Router ที่เราเขียนมาตลอด Part นี้ (`db_for_read`, `db_for_write`) รับพารามิเตอร์
`**hints` ซึ่งเปิดทางให้เขียน router ที่ตัดสินใจ **ตาม request context** ได้ เช่น:

```python
# ตัวอย่างแนวคิด (conceptual) ของ Tenant-based Sharding Router
class TenantShardRouter:
    def db_for_read(self, model, **hints):
        tenant = hints.get('tenant')  # ต้องส่ง hint นี้เข้ามาเองตอนเรียก .using() หรือผ่าน thread-local
        if tenant:
            return f'tenant_{tenant.shard_id}'
        return None

    def db_for_write(self, model, **hints):
        return self.db_for_read(model, **hints)
```

ในทางปฏิบัติจริงมักต้องใช้ **thread-local storage** หรือ **middleware** เพื่อเก็บว่า
"request ปัจจุบันนี้เป็นของ tenant ไหน" แล้วให้ router อ่านค่านั้นมาตัดสินใจ ซึ่งซับซ้อนกว่า
router ทั้งหมดที่เราเขียนใน Part นี้มาก และมีรายละเอียดเรื่อง connection management,
migration ต่อ shard, cross-shard query ที่ต้องออกแบบอย่างระมัดระวัง

### 196.4 ทำไม Sharding ยากกว่า Read Replica มาก

| ประเด็น | Read Replica | Sharding |
|---|---|---|
| ความซับซ้อนในการ Query ข้ามฐานข้อมูล | ต่ำ (query เดียวกันทุกที่) | สูงมาก (join ข้าม shard ทำไม่ได้ตรง ๆ) |
| การย้ายข้อมูลเมื่อระบบโต | ง่าย (เพิ่ม replica ใหม่) | ยาก (ต้อง re-shard ข้อมูลทั้งหมด) |
| Transaction ข้ามฐานข้อมูล | ไม่ค่อยจำเป็น | เป็นปัญหาใหญ่ (ไม่มี ACID ข้าม shard) |
| เหมาะกับระบบขนาดไหน | ระบบที่ read เยอะกว่า write มาก | ระบบที่มีข้อมูลขนาดเทราไบต์+ หรือ multi-tenant จริงจัง |

> **มองไปข้างหน้าอย่างชัดเจน**: Part นี้เกริ่นแค่ **แนวคิด** ของ Sharding ให้คุณรู้จักคำศัพท์
> และเข้าใจภาพกว้างเท่านั้น การออกแบบ Sharding Strategy อย่างจริงจัง, การเขียน
> Tenant-aware Router แบบสมบูรณ์, การจัดการ migration ต่อ shard นับร้อยตัว, และการแก้
> ปัญหา cross-shard transaction เป็นหัวข้อระดับ **Enterprise** ที่จะเจาะลึกเต็มรูปแบบใน
> **Phase 12: Scaling, Enterprise, โปรเจกต์จริง และเส้นทางอาชีพ** ซึ่งเป็น Phase เกือบ
> สุดท้ายของหลักสูตรนี้ ตอนนี้ขอให้จำหลักการและคำศัพท์ไว้ก่อนเป็นพื้นฐาน

---

## ขั้นตอนที่ 197: ข้อจำกัดของ Cross-Database Relations

### 197.1 ทำไม `ForeignKey` ข้ามฐานข้อมูลถึงทำไม่ได้ตรง ๆ

ลองจินตนาการว่าเราพยายามเขียนแบบนี้ (ซึ่ง **ไม่ควรทำ** และจะอธิบายว่าทำไม):

```python
# analytics/models.py — ตัวอย่างที่ "ดูเหมือนจะได้" แต่มีปัญหาใหญ่ซ่อนอยู่
from blog.models import Post


class PageView(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)  # ❌ อันตราย!
    viewed_at = models.DateTimeField(auto_now_add=True)
```

โค้ดนี้ **รันผ่านไม่มี error ตอน import** และ `makemigrations` ก็สร้าง migration ให้ได้
ตามปกติ แต่ปัญหาจะระเบิดตอนใช้งานจริง เพราะเหตุผลเชิงโครงสร้างฐานข้อมูลล้วน ๆ:

### 197.2 เหตุผลที่ 1: Foreign Key Constraint เป็นกลไกระดับฐานข้อมูล ไม่ใช่ระดับ Python

`ForeignKey` ใน Django ไม่ได้เป็นแค่แนวคิดระดับ Python เท่านั้น — เมื่อรัน `migrate`
Django จะสั่งให้ฐานข้อมูลสร้าง **FK constraint จริงในระดับ SQL** เช่น:

```sql
ALTER TABLE analytics_pageview
ADD CONSTRAINT fk_pageview_post
FOREIGN KEY (post_id) REFERENCES blog_post (id);
```

คำสั่งนี้ใช้ได้ก็ต่อเมื่อ **ตาราง `blog_post` กับ `analytics_pageview` อยู่ในฐานข้อมูล
เดียวกัน** เท่านั้น เพราะ PostgreSQL/MySQL ไม่มีกลไก FK constraint ที่ข้ามฐานข้อมูล
(cross-database) ได้เลย — นี่คือข้อจำกัดของตัวฐานข้อมูลเอง ไม่ใช่ข้อจำกัดของ Django

### 197.3 เหตุผลที่ 2: `allow_relation()` จะบล็อกไว้ตั้งแต่ระดับ ORM

ถ้าคุณมี router (เหมือนที่เราเขียนในขั้นตอนที่ 193) ที่ตั้งค่า `allow_relation()` อย่าง
เข้มงวด Django จะ raise error ทันทีตอนพยายามสร้างความสัมพันธ์ข้ามฐานข้อมูลที่ไม่ได้
อนุญาตไว้:

```python
>>> from analytics.models import PageView
>>> from blog.models import Post
>>> post = Post.objects.get(pk=1)          # อยู่ที่ default
>>> PageView.objects.create(post=post)      # สมมติ PageView มี FK ไปที่ Post จริง
Traceback (most recent call last):
    ...
ValueError: Cannot assign "<Post: แนะนำ Django 5>": the current database router
prevents this relation.
```

ข้อความนี้คือ Django กำลังบอกตรง ๆ ว่า **router บอกว่าห้าม** — ซึ่งถูกต้องแล้ว เพราะถ้า
ปล่อยให้ผ่านไป จะไม่สามารถสร้าง FK constraint จริงในฐานข้อมูลได้ตั้งแต่แรก

### 197.4 วิธีแก้ที่ถูกต้อง: เก็บ ID แบบ "loose coupling" แทน `ForeignKey`

นี่คือเหตุผลที่โมเดล `PageView` ของเราตั้งแต่ขั้นตอนที่ 191 ใช้
`post_id = models.PositiveIntegerField()` แทนที่จะเป็น `ForeignKey(Post)` ตรง ๆ —
รูปแบบนี้เรียกว่า **loose coupling** (การเชื่อมโยงแบบหลวม) ซึ่งเป็นวิธีมาตรฐานที่ระบบ
multi-database ระดับโลกใช้กัน:

```python
# analytics/models.py (ทบทวนจากขั้นตอนที่ 191)
class PageView(models.Model):
    post_id = models.PositiveIntegerField(db_index=True)   # เก็บแค่ ID ตัวเลข
    post_title_snapshot = models.CharField(max_length=200)  # เก็บสำเนาชื่อไว้ ณ เวลานั้น
    viewed_at = models.DateTimeField(auto_now_add=True)
```

**ทำไมต้องมี `post_title_snapshot` ด้วย?** เพราะเมื่อไม่มี FK จริง เราไม่สามารถ
`select_related('post')` เพื่อดึงชื่อบทความมาแสดงพร้อมกันได้อีกต่อไป (จะอธิบายเหตุผล
เพิ่มเติมในข้อ 197.6) การเก็บ **snapshot ของข้อมูลสำคัญ ณ เวลาที่บันทึก** จึงเป็นทางออก
ที่ใช้กันทั่วไปในระบบ log/analytics — แม้ต่อมาบทความจะถูกแก้ไขชื่อหรือถูกลบไปแล้ว log
เก่าก็ยังอ่านค่าที่ถูกต้อง ณ ตอนนั้นได้เสมอ (ซึ่งที่จริงเป็นพฤติกรรมที่ระบบ analytics
ต้องการอยู่แล้ว — เราอยากรู้ว่า "ตอนที่มีคนดู บทความชื่ออะไร" ไม่ใช่ "ตอนนี้บทความชื่ออะไร")

### 197.5 การจำลอง "join" ข้ามฐานข้อมูลด้วยโค้ด Python

เมื่อไม่มี FK จริง การดึงข้อมูลที่ต้องใช้ทั้งสองฐานข้อมูลพร้อมกันต้องทำเป็น **2 query
แยกกัน** แล้วนำมาประกอบกันในโค้ด Python เอง (ไม่ใช่ SQL JOIN):

```python
# blog/services.py (ไฟล์ใหม่สำหรับ business logic ที่ข้ามฐานข้อมูล)
from analytics.models import PageView
from blog.models import Post


def get_post_with_view_count(post_id):
    """ตัวอย่างการ 'join' ข้ามฐานข้อมูลด้วยโค้ด Python แทน SQL JOIN"""
    post = Post.objects.get(pk=post_id)                      # query ที่ default
    view_count = PageView.objects.filter(post_id=post_id).count()  # query ที่ analytics
    return {
        'post': post,
        'view_count': view_count,
    }


def get_top_viewed_posts(limit=5):
    """ตัวอย่างที่ซับซ้อนขึ้น: หาบทความยอดนิยม ต้อง aggregate ที่ analytics ก่อน
    แล้วค่อยไป fetch รายละเอียดจาก default ทีหลัง
    """
    from django.db.models import Count

    top_ids_with_counts = (
        PageView.objects.values('post_id')
        .annotate(views=Count('id'))
        .order_by('-views')[:limit]
    )
    id_to_views = {row['post_id']: row['views'] for row in top_ids_with_counts}
    posts = Post.objects.filter(pk__in=id_to_views.keys())
    return sorted(posts, key=lambda p: id_to_views[p.pk], reverse=True)
```

### 197.6 ตารางสรุปข้อจำกัดและวิธีแก้

| ข้อจำกัด | เหตุผล | วิธีแก้ |
|---|---|---|
| สร้าง `ForeignKey` ข้ามฐานข้อมูลไม่ได้จริง | FK constraint เป็นกลไกระดับ SQL ที่ข้ามฐานข้อมูลไม่ได้ | เก็บ ID แบบ `PositiveIntegerField` ธรรมดา (loose coupling) |
| `select_related()`/`prefetch_related()` ใช้ข้ามฐานข้อมูลไม่ได้ | ทั้งสองเมธอดสร้าง SQL JOIN/query เดียว ซึ่งต้องอยู่ฐานข้อมูลเดียวกัน | ดึงข้อมูลแยก 2 query แล้ว join ด้วย Python เอง (ดู 197.5) |
| `CASCADE` delete ข้ามฐานข้อมูลทำงานอัตโนมัติไม่ได้ | ไม่มี constraint จริงให้ฐานข้อมูลจัดการเอง | ใช้ Signal (Part 019) เพื่อลบข้อมูลที่เกี่ยวข้องในอีกฐานข้อมูลด้วยตนเอง |
| Transaction ครอบคลุมสองฐานข้อมูลพร้อมกันไม่ได้ (ไม่มี ACID ข้ามฐานข้อมูล) | แต่ละฐานข้อมูลมี transaction ของตัวเองแยกกัน | ยอมรับ eventual consistency หรือใช้ pattern อย่าง Saga (หัวข้อระดับ Enterprise) |
| ข้อมูลอ้างอิงอาจ "ค้าง" ถ้าต้นทางถูกลบ | ไม่มี FK คอยป้องกัน/แจ้งเตือน | เก็บ snapshot ข้อมูลสำคัญไว้ (เช่น `post_title_snapshot`) |

---

## ขั้นตอนที่ 198: การเขียน Test ที่เกี่ยวข้องกับหลายฐานข้อมูล

### 198.1 ปัญหา: ทำไมเทสต์ปกติมองไม่เห็นฐานข้อมูลที่สอง

Django `TestCase` มาตรฐานจะสร้างฐานข้อมูลทดสอบ (test database) ให้อัตโนมัติ **เฉพาะ
`default`** เท่านั้น ถ้าเทสต์ของคุณพยายามเขียน/อ่านฐานข้อมูล `analytics` โดยไม่ได้ตั้งค่า
อะไรเพิ่ม จะเจอ error ทันที:

```python
# analytics/tests.py — เวอร์ชันที่ยังพลาด
from django.test import TestCase
from analytics.models import PageView


class PageViewTests(TestCase):
    def test_create_pageview(self):
        PageView.objects.create(post_id=1, post_title_snapshot='ทดสอบ')
        self.assertEqual(PageView.objects.count(), 1)
```

```
django.test.testcases.DatabaseOperationForbidden: Database queries to 'analytics'
are not allowed in this test. Add 'analytics' to blog_databases_used_in_this_test.databases
to ensure proper test isolation and silence this failure.
```

Django ตั้งใจให้เกิด error นี้ขึ้นมา (ไม่ใช่บั๊ก) เพื่อบังคับให้นักพัฒนา **ประกาศอย่างชัดเจน**
ว่าเทสต์ตัวไหนแตะฐานข้อมูลอะไรบ้าง ป้องกันไม่ให้เทสต์แอบไปเขียนฐานข้อมูลที่ไม่ตั้งใจ

### 198.2 แก้ด้วย attribute `databases`

```python
# analytics/tests.py
from django.test import TestCase
from analytics.models import PageView


class PageViewTests(TestCase):
    databases = {'default', 'analytics'}

    def test_create_pageview(self):
        PageView.objects.create(post_id=1, post_title_snapshot='ทดสอบ')
        self.assertEqual(PageView.objects.count(), 1)

    def test_pageview_count_per_post(self):
        PageView.objects.create(post_id=1, post_title_snapshot='บทความ A')
        PageView.objects.create(post_id=1, post_title_snapshot='บทความ A')
        PageView.objects.create(post_id=2, post_title_snapshot='บทความ B')

        self.assertEqual(PageView.objects.filter(post_id=1).count(), 2)
        self.assertEqual(PageView.objects.filter(post_id=2).count(), 1)
```

`databases = {'default', 'analytics'}` เป็น **class attribute** ที่บอก Django ว่า
เทสต์คลาสนี้ต้องการสร้างฐานข้อมูลทดสอบสำหรับทั้ง `default` และ `analytics` ก่อนรัน —
Django จะสร้างฐานข้อมูลทดสอบชั่วคราวให้ทั้งสองตัว (ปกติชื่อ `test_blogdb` และ
`test_blog_analytics`) รัน migration ทั้งคู่ แล้วลบทิ้งหลังเทสต์เสร็จ เหมือนที่ทำกับ
`default` เพียงลำพังมาโดยตลอด

### 198.3 ทดสอบ Business Logic ที่ข้ามฐานข้อมูล (จาก `services.py` ในขั้นตอนที่ 197)

```python
# blog/tests/test_services.py
from django.test import TestCase
from blog.models import Post
from blog.services import get_post_with_view_count, get_top_viewed_posts
from analytics.models import PageView


class CrossDatabaseServiceTests(TestCase):
    databases = {'default', 'analytics'}

    def setUp(self):
        self.post1 = Post.objects.create(title='บทความ A', content='...')
        self.post2 = Post.objects.create(title='บทความ B', content='...')
        PageView.objects.create(post_id=self.post1.pk, post_title_snapshot='บทความ A')
        PageView.objects.create(post_id=self.post1.pk, post_title_snapshot='บทความ A')
        PageView.objects.create(post_id=self.post2.pk, post_title_snapshot='บทความ B')

    def test_get_post_with_view_count(self):
        result = get_post_with_view_count(self.post1.pk)
        self.assertEqual(result['post'], self.post1)
        self.assertEqual(result['view_count'], 2)

    def test_get_top_viewed_posts_orders_correctly(self):
        top_posts = get_top_viewed_posts(limit=2)
        self.assertEqual(top_posts[0], self.post1)  # 2 views > 1 view
        self.assertEqual(top_posts[1], self.post2)
```

### 198.4 ใช้ `'__all__'` เมื่อไม่แน่ใจว่าเทสต์แตะฐานข้อมูลไหนบ้าง

สำหรับเทสต์ที่ครอบคลุมหลายจุดในระบบและตรวจสอบยากว่าแตะฐานข้อมูลไหนบ้าง (เช่น เทสต์
integration ระดับใหญ่) Django อนุญาตให้ใช้ค่าพิเศษ `'__all__'`:

```python
class FullIntegrationTests(TestCase):
    databases = '__all__'   # อนุญาตให้แตะทุกฐานข้อมูลที่ตั้งค่าไว้ใน settings

    def test_complex_flow(self):
        ...
```

**คำแนะนำระดับมืออาชีพ**: ใช้ `'__all__'` เท่าที่จำเป็นเท่านั้น เพราะการสร้างฐานข้อมูล
ทดสอบสำหรับทุกตัวทำให้เทสต์ **รันช้าลง** อย่างมีนัยสำคัญเมื่อมีฐานข้อมูลหลายตัว — ควร
ระบุเจาะจงเฉพาะฐานข้อมูลที่เทสต์นั้นต้องใช้จริง ๆ (`{'default', 'analytics'}`) จะช่วยให้
test suite โดยรวมเร็วกว่ามาก

### 198.5 ตรวจสอบว่าเทสต์ใช้ Router ที่ถูกต้อง

```python
# analytics/tests.py
from django.test import TestCase
from analytics.models import PageView


class RouterBehaviorTests(TestCase):
    databases = {'default', 'analytics'}

    def test_pageview_routed_to_analytics_db(self):
        pv = PageView.objects.create(post_id=1, post_title_snapshot='ทดสอบ router')
        # ตรวจสอบว่า instance นี้ถูกบันทึกไปที่ 'analytics' จริง ไม่ใช่ 'default'
        self.assertEqual(pv._state.db, 'analytics')
```

`instance._state.db` คือวิธีตรวจสอบว่า object ที่เพิ่งโหลด/บันทึกมานั้น **มาจากฐานข้อมูล
alias ไหนจริง ๆ** ซึ่งเป็นเครื่องมือดีบักที่มีประโยชน์มากเมื่อ router มีความซับซ้อนหลายตัว
ต่อกันเป็น chain

### 198.6 ตารางสรุป

| สถานการณ์ | ค่า `databases` ที่ควรตั้ง |
|---|---|
| เทสต์แตะเฉพาะโมเดลใน `default` (ปกติที่สุด) | ไม่ต้องตั้งเลย (ค่า default คือ `{'default'}`) |
| เทสต์แตะโมเดลของแอป `analytics` | `{'default', 'analytics'}` |
| เทสต์ business logic ที่ข้ามฐานข้อมูล | ระบุทุก alias ที่เกี่ยวข้องให้ครบ |
| เทสต์ integration ขนาดใหญ่ ไม่แน่ใจ scope | `'__all__'` (แลกกับความเร็วที่ลดลง) |

---

## ขั้นตอนที่ 199: เกริ่น Connection Pooling (PgBouncer)

### 199.1 ปัญหา: การเปิด-ปิด Database Connection มีต้นทุนสูง

ทุกครั้งที่ Django ต้องคุยกับ PostgreSQL จะต้องเปิด **connection** ขึ้นมาก่อน ซึ่งเป็น
กระบวนการที่มีต้นทุน (overhead) ทาง network และ CPU ไม่น้อย — ถ้าแอปพลิเคชันมีทราฟฟิก
สูงมาก (หลักพัน request ต่อวินาที) การเปิด-ปิด connection ใหม่ทุกครั้งจะกลายเป็นคอขวด
สำคัญของทั้งระบบ นอกจากนี้ PostgreSQL ยังมี **ขีดจำกัดจำนวน connection พร้อมกันสูงสุด**
(ค่า default มักอยู่ที่ประมาณ 100) ซึ่งหมดได้เร็วมากถ้าแต่ละ process ของ Django เปิด
connection ของตัวเองไม่จำกัด

### 199.2 `CONN_MAX_AGE`: การแก้ปัญหาเบื้องต้นที่ Django มีให้ในตัว

ก่อนพูดถึง PgBouncer เรามาทบทวนสิ่งที่ Django มีให้อยู่แล้วก่อน — ที่จริงเราใส่ไว้แล้วใน
`DATABASES` ตั้งแต่ขั้นตอนที่ 191 และ 194:

```python
DATABASES = {
    'default': {
        # ...
        'CONN_MAX_AGE': 60,   # วินาที: ใช้ connection เดิมซ้ำได้นานแค่ไหนก่อนปิด
    },
}
```

`CONN_MAX_AGE` บอก Django ให้ **นำ connection กลับมาใช้ซ้ำ (persistent connection)**
ภายในเวลาที่กำหนด แทนที่จะเปิด-ปิดใหม่ทุก request — ตั้งเป็น `0` (ค่า default เดิม) คือ
ปิดทันทีทุกครั้งหลัง request จบ, ตั้งเป็น `None` คือไม่ปิดเลย (persistent ตลอดไปจนกว่า
จะมีปัญหา), ส่วนตัวเลขวินาทีคือ "compromise" ที่ใช้กันบ่อยที่สุดในทางปฏิบัติ

### 199.3 ทำไม `CONN_MAX_AGE` อย่างเดียวไม่พอสำหรับระบบใหญ่

`CONN_MAX_AGE` ช่วยแค่ **ระดับ process เดียว** ของ Django (เช่น 1 worker ของ Gunicorn)
แต่ในระบบ production จริงมักรัน Gunicorn/uWSGI หลาย worker process พร้อมกัน (บางระบบ
หลักสิบ worker ต่อเครื่อง คูณด้วยหลายเครื่อง) — แต่ละ worker ก็จะมี connection ของตัวเอง
อยู่ดี รวมกันแล้วอาจทะลุขีดจำกัด connection ของ PostgreSQL ได้ง่าย ๆ

### 199.4 PgBouncer คืออะไร (ภาพรวมระดับแนวคิด)

**PgBouncer** คือ **connection pooler** สำหรับ PostgreSQL โดยเฉพาะ ทำหน้าที่เป็น
"ตัวกลาง" ระหว่าง Django (หรือแอปใด ๆ) กับ PostgreSQL จริง:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Django Worker│     │              │     │              │
│      1       │────>│              │     │              │
├──────────────┤     │              │     │              │
│ Django Worker│     │  PgBouncer   │────>│  PostgreSQL  │
│      2       │────>│(Connection   │     │   (จริง)     │
├──────────────┤     │   Pool)      │     │              │
│ Django Worker│     │              │     │              │
│     ...N     │────>│              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
   (connection หลายร้อยตัว)    (pool connection จริงแค่ 20-30 ตัว)
```

หลักการคือ PgBouncer รับ connection จาก Django worker จำนวนมาก แต่ **ใช้ connection
จริงไปยัง PostgreSQL เพียงจำนวนน้อยที่ถูกจำกัดไว้** (เช่น pool size = 20) แล้วสลับให้
worker ต่าง ๆ ยืมใช้หมุนเวียนกัน (คล้ายหลักการเดียวกับ database connection pool ใน
ภาษาโปรแกรมอื่น ๆ) ทำให้ PostgreSQL ไม่ต้องรับภาระเปิด connection จำนวนมหาศาลโดยตรง

### 199.5 ตั้งค่า Django ให้ชี้ไปที่ PgBouncer แทน PostgreSQL ตรง ๆ (แนวคิดคร่าว ๆ)

```python
# config/settings.py (แนวคิดคร่าว ๆ — รายละเอียดเต็มรูปแบบอยู่ใน Phase 8/11)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'blogdb',
        'USER': 'blogadmin',
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': 'pgbouncer.internal',   # ชี้ไปที่ PgBouncer ไม่ใช่ PostgreSQL ตรง ๆ
        'PORT': '6432',                  # PgBouncer มักฟังที่พอร์ต 6432 (ไม่ใช่ 5432)
        'DISABLE_SERVER_SIDE_CURSORS': True,  # จำเป็นเมื่อใช้ pool mode 'transaction'
    },
}
```

`DISABLE_SERVER_SIDE_CURSORS = True` เป็นการตั้งค่าที่มักต้องใช้คู่กับ PgBouncer เมื่อ
ทำงานใน pool mode แบบ `transaction` (โหมดที่ประหยัด connection ที่สุด) เพราะ server-side
cursor ต้องการให้ connection เดิมคงอยู่ตลอด transaction ซึ่งขัดกับวิธีที่ PgBouncer สลับ
connection ให้ worker อื่นใช้ต่อ

### 199.6 ทำไม Part นี้ไม่ลงรายละเอียดเต็มรูปแบบ

การติดตั้งและจูน PgBouncer จริง (pool mode ทั้ง 3 แบบ: session/transaction/statement,
การตั้งค่า `max_client_conn`, `default_pool_size`, การ monitor สถานะ pool, การจัดการ
กับ prepared statements ที่ conflict กับ transaction pooling) เป็นงานระดับ **Infrastructure
/ DevOps** ที่ต้องอาศัยความรู้เรื่อง server administration ควบคู่ไปกับ Django ซึ่งเป็น
หัวข้อของ:

- **Phase 8: Performance & Caching** — จะพูดถึง connection pooling ในบริบทของการ
  optimize performance โดยรวมของแอปพลิเคชัน ควบคู่กับ caching, query optimization
- **Phase 11: DevOps, Docker และ CI/CD** — จะพูดถึงการ deploy PgBouncer จริงเป็นส่วน
  หนึ่งของ infrastructure stack (มักรันเป็น container แยกต่างหากใน Docker Compose หรือ
  Kubernetes)

ตอนนี้ขอให้จำหลักการสำคัญไว้ก่อน: **`CONN_MAX_AGE` แก้ปัญหาระดับ process เดียว,
PgBouncer แก้ปัญหาระดับทั้งระบบ (หลาย process, หลายเครื่อง)** และทั้งสองใช้ร่วมกันได้
ไม่ขัดแย้งกัน

---

## ขั้นตอนที่ 200: สรุป Phase 2 ทั้งหมด (Part 011-020) และเตรียมตัวสู่ Phase 3

### 200.1 ภาพรวมการเดินทางตลอด Phase 2

Phase 2 พาคุณเดินทางจากโมเดล `Post` เดี่ยว ๆ ที่ไม่เชื่อมกับอะไรเลย ไปจนถึงระบบฐานข้อมูล
ระดับมืออาชีพที่มีความสัมพันธ์ซับซ้อน, ORM ขั้นสูง, Admin panel เต็มรูปแบบ, ระบบอัตโนมัติ
ผ่าน Signals, และรองรับหลายฐานข้อมูลพร้อมกัน — มาดูภาพรวมทีละ Part:

| Part | หัวข้อหลัก | สิ่งที่ได้เพิ่มเข้าโปรเจกต์ |
|---|---|---|
| 011 | Models เบื้องต้น: Fields และ Migrations | `Post` model, field types พื้นฐาน, `makemigrations`/`migrate` |
| 012 | ความสัมพันธ์ระหว่างโมเดล | `Category` (FK), `Profile` (O2O), `Tag` (M2M), `PostTag` (through), `Comment` (self-referential) |
| 013 | Django ORM QuerySet ขั้นสูง | `select_related`/`prefetch_related`, `values()`/`values_list()`, lazy evaluation |
| 014 | Aggregation, Annotation, Q/F Expressions | `Count`/`Sum`/`Avg`, `annotate()`, `Q()` สำหรับเงื่อนไขซับซ้อน, `F()` สำหรับเปรียบเทียบ field |
| 015 | Model Meta Options, Managers, Custom QuerySets | `Meta.ordering`/`constraints`, custom `Manager`, chainable `QuerySet` |
| 016 | Database Migrations ขั้นสูง | Data migration, squash migration, การจัดการ schema conflict |
| 017 | Django Admin เบื้องต้น | `ModelAdmin`, `list_display`, `search_fields`, `list_filter` |
| 018 | Django Admin ขั้นสูง | Custom actions, `inlines`, `readonly_fields`, การปรับแต่ง admin site |
| 019 | Signals และ Lifecycle Hooks | `post_save`/`pre_delete`, สร้าง `Profile` อัตโนมัติเมื่อมี `User` ใหม่ |
| 020 | Multiple Databases และ Database Routing | `DATABASES` หลายตัว, `.using()`, Router, Read Replica, `inspectdb`, Sharding, Testing, PgBouncer |

### 200.2 แผนภาพ Schema ฐานข้อมูลทั้งหมดของโปรเจกต์บล็อก ณ จุดสิ้นสุด Phase 2

```
ฐานข้อมูล 'default' (blogdb):
┌─────────────┐        ┌─────────────┐
│    User     │◄──1:1──│   Profile   │
│  (Django)   │        └─────────────┘
└──────┬──────┘
       │ 1:N (author, เพิ่มเติมได้ใน Part ถัดไปของ Phase 3)
       ▼
┌─────────────┐  N:1   ┌─────────────┐
│    Post     │───────►│  Category   │
└──────┬──────┘        └─────────────┘
       │ N:M (through PostTag)
       ├──────────────►┌─────────────┐
       │                │     Tag     │
       │                └─────────────┘
       │ 1:N
       ▼
┌─────────────┐
│   Comment   │ (self-referential: parent -> Comment)
└─────────────┘

ฐานข้อมูล 'analytics' (blog_analytics):
┌──────────────────────┐
│      PageView          │  (post_id เป็น loose reference ไป Post ที่ default
│  post_id, viewed_at    │   ไม่ใช่ ForeignKey จริง — ดูเหตุผลในขั้นตอนที่ 197)
└──────────────────────┘

ฐานข้อมูล 'legacy_hr' (hr_system_legacy, managed=False):
┌──────────────────────┐
│    LegacyEmployee      │  (db_table='tbl_employee', อ่านข้อมูลจากระบบ HR เก่า)
└──────────────────────┘
```

### 200.3 Quiz ทบทวน Phase 2 (พร้อมเฉลย)

พยายามตอบเองก่อนเลื่อนไปดูเฉลยแต่ละข้อ

**คำถามที่ 1**: ถ้าลบ `Category` ที่ `Post.category` ตั้ง `on_delete=models.SET_NULL`
ไว้ จะเกิดอะไรขึ้นกับ `Post` ที่เคยอยู่ในหมวดหมู่นั้น?

> **เฉลย**: `Post.category` ของบทความที่เคยอยู่หมวดหมู่นั้นจะถูกตั้งเป็น `NULL`
> (ต้องมี `null=True` กำกับไว้ที่ field เสมอ) ตัว `Post` เองไม่ถูกลบไปด้วย

**คำถามที่ 2**: `ForeignKey` กับ `OneToOneField` ต่างกันอย่างไรในระดับฐานข้อมูล?

> **เฉลย**: `OneToOneField` คือ `ForeignKey` ที่มี `unique=True` บังคับติดตัวมาเสมอ
> ทำให้แถวแม่หนึ่งแถวจับคู่ได้กับแถวลูกได้แค่หนึ่งแถวเท่านั้น ต่างจาก `ForeignKey` ปกติที่
> จับคู่ได้หลายแถว

**คำถามที่ 3**: ทำไม `ManyToManyField` ถึงไม่มีพารามิเตอร์ `on_delete`?

> **เฉลย**: เพราะความสัมพันธ์ M2M ไม่มีฝั่งไหนเป็น "แม่" หรือ "ลูก" อย่างชัดเจน ข้อมูล
> ความสัมพันธ์ถูกเก็บแยกไว้ที่ตารางกลาง (junction table) เมื่อลบแถวใดแถวหนึ่งใน
> ทั้งสองฝั่ง Django จะลบแค่แถวที่เกี่ยวข้องในตารางกลางออกไปเท่านั้น

**คำถามที่ 4**: เมื่อไหร่ควรใช้ custom `through` model แทน `ManyToManyField` ธรรมดา?

> **เฉลย**: เมื่อต้องการเก็บข้อมูลเพิ่มเติมเกี่ยวกับความสัมพันธ์นั้นเอง เช่น วันที่เพิ่ม
> (`added_at`), ใครเป็นคนเพิ่ม (`added_by`) ซึ่งตารางกลางอัตโนมัติของ Django ไม่มีที่เก็บ
> ข้อมูลเหล่านี้

**คำถามที่ 5**: `select_related()` กับ `prefetch_related()` ต่างกันอย่างไร และควรใช้
เมื่อไหร่?

> **เฉลย**: `select_related()` ใช้กับความสัมพันธ์ฝั่ง "หนึ่ง" (`ForeignKey`,
> `OneToOneField`) ทำ SQL JOIN ครั้งเดียวได้ข้อมูลครบ ส่วน `prefetch_related()` ใช้กับ
> ความสัมพันธ์ฝั่ง "หลาย" (`ManyToManyField`, reverse `ForeignKey`) ทำ query แยกแล้ว
> จับคู่ให้ในระดับ Python ทั้งคู่มีเป้าหมายเดียวกันคือแก้ปัญหา **N+1 query**

**คำถามที่ 6**: `Q()` object ต่างจาก `F()` object อย่างไร?

> **เฉลย**: `Q()` ใช้สร้างเงื่อนไขการค้นหาที่ซับซ้อน (`AND`, `OR`, `NOT` ระหว่างหลาย
> เงื่อนไข) ส่วน `F()` ใช้อ้างอิงค่าของ field อื่นในแถวเดียวกันเพื่อเปรียบเทียบหรือคำนวณ
> โดยไม่ต้องดึงค่าออกมาที่ฝั่ง Python ก่อน (คำนวณที่ระดับฐานข้อมูลโดยตรง)

**คำถามที่ 7**: Django Signal `post_save` ต่างจาก `pre_save` อย่างไร และควรใช้ signal
ไหนในการสร้าง `Profile` อัตโนมัติเมื่อมี `User` ใหม่?

> **เฉลย**: `pre_save` ทำงาน**ก่อน**บันทึกลงฐานข้อมูล (ยังไม่มี pk ถ้าเป็นการสร้างใหม่),
> `post_save` ทำงาน**หลัง**บันทึกเสร็จแล้ว (มี pk แน่นอน) การสร้าง `Profile` ต้องใช้
> `post_save` เพราะต้องรอให้ `User` มี pk ก่อน ถึงจะสร้าง `Profile` ที่อ้างอิงกลับไปได้
> (และต้องเช็ค `created=True` เพื่อสร้างเฉพาะตอน user ใหม่เท่านั้น ไม่ใช่ทุกครั้งที่ save)

**คำถามที่ 8**: `DATABASE_ROUTERS` ใน settings เป็น list เสมอแม้จะมี router เดียว
เพราะอะไร?

> **เฉลย**: เพราะ Django รองรับการวาง router หลายตัวเรียงกันเป็น chain โดยจะไล่ถามทีละ
> ตัวตามลำดับจนกว่าจะมีตัวใดตัวหนึ่งคืนค่าที่ไม่ใช่ `None` — การออกแบบให้เป็น list
> รองรับ use case นี้ไว้ตั้งแต่ต้น แม้โปรเจกต์ส่วนใหญ่จะมี router แค่ตัวเดียวก็ตาม

**คำถามที่ 9**: ทำไม `ForeignKey` ข้ามฐานข้อมูล (จาก `analytics.PageView` ไปยัง
`blog.Post`) จึงทำไม่ได้จริง?

> **เฉลย**: เพราะ Foreign Key constraint เป็นกลไกระดับ SQL ที่ฐานข้อมูล (PostgreSQL/
> MySQL) ต้องดูแลเอง และไม่มีฐานข้อมูลตัวไหนรองรับ constraint ที่ข้ามไปอีกฐานข้อมูลหนึ่ง
> ได้ วิธีแก้คือเก็บ ID แบบ loose coupling (`PositiveIntegerField` ธรรมดา) แทน

**คำถามที่ 10**: ทำไมเทสต์ที่แตะฐานข้อมูล `analytics` ต้องประกาศ
`databases = {'default', 'analytics'}` ใน `TestCase`?

> **เฉลย**: เพราะ Django `TestCase` มาตรฐานจะสร้างฐานข้อมูลทดสอบให้เฉพาะ `default`
> เท่านั้นโดย default เพื่อป้องกันไม่ให้เทสต์แอบเขียนฐานข้อมูลอื่นโดยไม่ตั้งใจ ต้องประกาศ
> `databases` อย่างชัดเจนเพื่อให้ Django สร้างฐานข้อมูลทดสอบของ `analytics` ให้ด้วย

**คำถามที่ 11 (โบนัส)**: `CONN_MAX_AGE` กับ PgBouncer แก้ปัญหาคนละระดับกันอย่างไร?

> **เฉลย**: `CONN_MAX_AGE` แก้ปัญหาระดับ **process เดียว** ของ Django (ให้ worker
> หนึ่งตัวใช้ connection เดิมซ้ำแทนเปิด-ปิดใหม่ทุก request) ส่วน PgBouncer แก้ปัญหา
> ระดับ **ทั้งระบบ** (รวม connection จากหลาย worker/หลายเครื่องให้เหลือ connection จริง
> ไปยัง PostgreSQL จำนวนน้อยที่ควบคุมได้)

### 200.4 แบบฝึกหัดใหญ่ปิดท้าย Phase 2

แบบฝึกหัดนี้รวบยอดทุกหัวข้อของ Phase 2 เข้าด้วยกัน ให้ทำตามลำดับขั้นตอนต่อไปนี้บนโปรเจกต์
บล็อกของคุณเอง:

**ส่วนที่ 1 — Admin ที่สมบูรณ์สำหรับทุกโมเดล**

1. ลงทะเบียนโมเดลทั้งหมดของแอป `blog` (`Category`, `Post`, `Tag`, `PostTag`, `Comment`)
   และ `accounts` (`Profile`) เข้า Django Admin โดยใช้เทคนิคจาก Part 017-018:
   - `Post`: ใช้ `list_display` แสดง title, category, is_published, created_at;
     `list_filter` ตาม category และ is_published; `search_fields` ค้นหาจาก title;
     `prepopulated_fields` ให้ slug เติมอัตโนมัติจาก title
   - `Post`: เพิ่ม `TabularInline` สำหรับ `Comment` เพื่อดูคอมเมนต์ทั้งหมดในหน้าแก้ไข
     บทความเดียวกัน
   - `Category`: แสดงจำนวนบทความในแต่ละหมวดหมู่ด้วย custom method ใน `list_display`
     (ใช้ `annotate(Count('posts'))` ใน `get_queryset` ที่ override เอง)
   - `Post`: เพิ่ม custom admin action ชื่อ `mark_as_published` ที่เปลี่ยน
     `is_published=True` ให้บทความที่เลือกพร้อมกันหลายรายการ
   - `Comment`: ใช้ `readonly_fields` สำหรับ `created_at` และแสดง `post` แบบ
     `raw_id_fields` เพื่อประสิทธิภาพเมื่อมีบทความจำนวนมาก

2. เพิ่ม Signal (`post_save` บน `User`) ที่ยังไม่ได้ทำใน Part 019 ให้สมบูรณ์: สร้าง
   `Profile` อัตโนมัติทุกครั้งที่มี `User` ใหม่ถูกสร้างผ่าน `python manage.py createsuperuser`
   หรือผ่านฟอร์มสมัครสมาชิก (ที่จะเรียนเต็มรูปแบบใน Phase 4) และเขียนเทสต์ยืนยันว่า
   Signal ทำงานถูกต้อง (ใช้ `post_save.disconnect()`/`connect()` หรือ mock ตามที่เรียน
   ใน Part 019)

**ส่วนที่ 2 — เพิ่มฐานข้อมูล Analytics แยกสำหรับเก็บ View Log**

3. สร้างแอป `analytics` (ถ้ายังไม่มี) พร้อมโมเดล `PageView` ตามที่ออกแบบไว้ในขั้นตอนที่
   191 ของ Part นี้ (`post_id`, `post_title_snapshot`, `viewer_ip`, `user_agent`,
   `viewed_at`)

4. ตั้งค่า `DATABASES['analytics']` และเขียน `AnalyticsRouter` ให้ครบทั้ง 4 method
   (`db_for_read`, `db_for_write`, `allow_relation`, `allow_migrate`) ตามที่เรียนใน
   ขั้นตอนที่ 193

5. เขียน View function (function-based view ธรรมดาตามที่เรียนใน Part 007 — เราจะเรียน
   Class-Based Views ใน Phase 3 ถัดไป) ที่บันทึก `PageView` ทุกครั้งที่มีคนเปิดหน้า
   รายละเอียดบทความ (`post_detail`) โดยดึง `viewer_ip` จาก `request.META.get('REMOTE_ADDR')`
   และ `user_agent` จาก `request.META.get('HTTP_USER_AGENT', '')`

6. เพิ่มหน้า Admin แยกสำหรับ `PageView` (ทะเบียนใน `analytics/admin.py`) ที่แสดง
   `list_display` ครบ, `list_filter` ตามวันที่ (`viewed_at`), และตั้งเป็น
   read-only ทั้งหมด (override `has_add_permission()` และ `has_change_permission()`
   ให้คืน `False` เสมอ เพราะข้อมูล log ไม่ควรถูกแก้ไขย้อนหลัง)

7. เขียนฟังก์ชัน `get_post_with_view_count()` และ `get_top_viewed_posts()` ใน
   `blog/services.py` ตามตัวอย่างในขั้นตอนที่ 197.5 แล้วเพิ่มไปแสดงในหน้าแรกของบล็อก
   เป็นบล็อก "บทความยอดนิยม 5 อันดับ"

8. เขียนเทสต์ครอบคลุมทั้งหมด (`analytics/tests.py`, `blog/tests/test_services.py`)
   โดยตั้งค่า `databases = {'default', 'analytics'}` ให้ถูกต้องตามขั้นตอนที่ 198
   เป้าหมายคือให้ `python manage.py test` รันผ่านทั้งหมดโดยไม่มี
   `DatabaseOperationForbidden` error

**เกณฑ์ความสำเร็จของแบบฝึกหัด**:

- [ ] `python manage.py migrate` และ `python manage.py migrate --database=analytics`
      รันผ่านไม่มี error (หรือรัน `migrate` ครั้งเดียวแล้ว router จัดการให้ถูกต้อง)
- [ ] เปิดหน้า `/admin/` เห็นโมเดลทั้งหมดครบ พร้อม inline comments ใน Post
- [ ] กด custom action "Mark as published" ใน Admin แล้ว `is_published` เปลี่ยนถูกต้อง
      ทุกแถวที่เลือก
- [ ] เปิดหน้ารายละเอียดบทความหลายครั้ง แล้วตรวจสอบด้วย `python manage.py shell` ว่า
      `PageView.objects.using('analytics').count()` เพิ่มขึ้นตามจำนวนครั้งที่เปิด
- [ ] หน้าแรกของบล็อกแสดง "บทความยอดนิยม 5 อันดับ" ถูกต้องตามจำนวน view จริง
- [ ] `python manage.py test` ผ่านทั้งหมดโดยไม่มี error เกี่ยวกับฐานข้อมูล

### 200.5 สิ่งที่คุณควรทำได้แล้วตอนนี้ (Checklist รวม Phase 2)

- [ ] ออกแบบและสร้างโมเดลที่มีความสัมพันธ์ครบทั้ง 3 แบบ (FK, O2O, M2M) พร้อมเลือก
      `on_delete` ได้อย่างเหมาะสมตามหลักธุรกิจ
- [ ] เขียน custom `through` model เมื่อความสัมพันธ์ M2M ต้องการ metadata เพิ่มเติม
- [ ] ใช้ `related_name` และ query ย้อนกลับได้อย่างคล่องแคล่ว
- [ ] แก้ปัญหา N+1 query ด้วย `select_related()`/`prefetch_related()` ได้ถูกจุด
- [ ] เขียน query ที่ซับซ้อนด้วย `Q()`, `F()`, `annotate()`, `aggregate()`
- [ ] ออกแบบ custom Manager และ custom QuerySet ที่ chain กันได้
- [ ] เขียน data migration และจัดการ schema conflict เบื้องต้น
- [ ] ปรับแต่ง Django Admin ให้ใช้งานได้จริงในระดับทีมงาน (inlines, actions, filters)
- [ ] ใช้ Django Signals แก้ปัญหา lifecycle ของโมเดลได้อย่างเหมาะสม
- [ ] ตั้งค่าและใช้งานหลายฐานข้อมูลพร้อมกันผ่าน Router ได้อย่างถูกต้อง
- [ ] เข้าใจข้อจำกัดของ cross-database relations และออกแบบ loose coupling ได้
- [ ] เขียนเทสต์ที่ครอบคลุมหลายฐานข้อมูลได้ถูกต้อง

หากทำเครื่องหมายได้ครบทุกข้อ แปลว่าคุณมีรากฐานด้าน **Models, ORM และ Admin ระดับมืออาชีพ**
ที่แน่นพอจะรับมือกับ Phase ถัดไปได้อย่างมั่นใจ

### 200.6 คำถามที่พบบ่อยส่งท้าย Phase 2

**Q: จำเป็นต้องใช้หลายฐานข้อมูลในทุกโปรเจกต์ไหม?**
A: ไม่จำเป็นเลย โปรเจกต์ส่วนใหญ่ในโลกความเป็นจริงใช้ฐานข้อมูลเดียว (`default`) ตลอด
อายุของระบบได้สบาย ๆ การใช้หลายฐานข้อมูลควรเกิดจาก**ความจำเป็นทางธุรกิจที่ชัดเจน**
(ปริมาณข้อมูลมหาศาล, ต้องแยก legacy, ต้องทำ read replica เพราะทราฟฟิกสูงจริง ๆ) ไม่ใช่
ทำเพราะ "ดูเท่" หรือ "เผื่อไว้" เพราะความซับซ้อนที่เพิ่มขึ้นมีต้นทุนสูงกว่าที่คิดมาก

**Q: ถ้าไม่มี Router เลย Django จะพังไหม?**
A: ไม่พัง โปรเจกต์ที่มีแค่ `default` ฐานข้อมูลเดียวไม่จำเป็นต้องมี Router หรือ
`DATABASE_ROUTERS` เลย ทุกอย่างจะทำงานเหมือนที่เคยเป็นมาตลอด Phase 1-2 (ก่อน Part นี้)
Router จำเป็นก็ต่อเมื่อมีฐานข้อมูลมากกว่าหนึ่งตัวเท่านั้น

**Q: Sharding กับ Multi-tenancy เกี่ยวข้องกันยังไง?**
A: Sharding แบบ "แบ่งตาม tenant" เป็นหนึ่งในกลยุทธ์การทำ Multi-tenancy (ระบบ SaaS ที่
รองรับลูกค้าหลายราย) แบบ "database-per-tenant" — แต่ Multi-tenancy ยังทำได้ด้วยวิธีอื่น
ที่ไม่ต้องใช้ sharding เลย เช่น "schema-per-tenant" (แยก PostgreSQL schema แต่อยู่
ฐานข้อมูลเดียวกัน) หรือ "shared schema with tenant_id column" (ทุก tenant อยู่ตาราง
เดียวกัน แยกด้วยคอลัมน์ `tenant_id`) ซึ่งแต่ละวิธีมีข้อดี-ข้อเสียต่างกัน เราจะเจาะลึกเรื่อง
Multi-tenancy เต็มรูปแบบในภายหลังของหลักสูตร

**Q: ควรใช้ PgBouncer ตั้งแต่เริ่มโปรเจกต์เลยไหม?**
A: ไม่จำเป็นสำหรับโปรเจกต์เล็ก-กลาง `CONN_MAX_AGE` ที่ตั้งค่าเหมาะสมมักเพียงพอแล้วสำหรับ
ทราฟฟิกระดับกลาง PgBouncer คุ้มค่าที่จะติดตั้งเมื่อระบบเริ่มมี worker process จำนวนมาก
(หลักสิบขึ้นไป) หรือเมื่อเห็นสัญญาณว่า PostgreSQL ใกล้ชนขีดจำกัด `max_connections` จริง ๆ

---

## เตรียมตัวสำหรับ Phase 3: Views, Templates, Forms และ Class-Based Views

Phase 2 ปิดฉากลงแล้วด้วยความเข้าใจ **Models และ ORM ระดับมืออาชีพ** อย่างครบถ้วน —
ตั้งแต่ field พื้นฐาน ไปจนถึงความสัมพันธ์ซับซ้อน, query ขั้นสูง, Admin ที่ใช้งานได้จริง,
Signals ที่ทำให้ระบบตอบสนองอัตโนมัติ, และการรองรับหลายฐานข้อมูลพร้อมกัน

แต่ตลอด Phase 2 เราใช้ **Django shell** และ **Django Admin** เป็นหลักในการโต้ตอบกับ
ข้อมูล — ผู้ใช้ทั่วไปของเว็บไซต์จริงยังไม่เคยเห็นหน้าเว็บที่สวยงามที่แสดงบทความ, ฟอร์มที่
ให้กรอกคอมเมนต์, หรือปุ่มกดที่ใช้งานได้จริงเลยสักหน้า

**Phase 3: Views, Templates, Forms และ Class-Based Views (Part 021-030)** จะพาคุณ
กลับมาที่ฝั่ง "หน้าตาที่ผู้ใช้เห็นจริง" อย่างเต็มรูปแบบ:

- **Part 021-024**: เปลี่ยนจาก Function-Based Views (ที่เรียนไปแล้วใน Part 007) มาเป็น
  **Class-Based Views (CBV)** ที่ลดโค้ดซ้ำซ้อนได้มาก ตั้งแต่ CBV เบื้องต้น, Generic CBV
  อย่าง `ListView`/`DetailView`/`CreateView`/`UpdateView`/`DeleteView`, ไปจนถึงการเขียน
  Mixin ของตัวเอง
- **Part 025-027**: ระบบ **Django Forms** เต็มรูปแบบ ตั้งแต่ฟอร์มพื้นฐาน,
  **ModelForms** ที่เชื่อมกับโมเดลที่เราสร้างมาตลอด Phase 2 โดยตรง (เช่น สร้างฟอร์ม
  เขียนบทความจาก `Post` model ทันทีโดยไม่ต้องเขียน field ซ้ำ), Formsets, และ
  Validation ขั้นสูงพร้อม Custom Widgets
- **Part 028-030**: **Template Inheritance** ที่ทำให้ไม่ต้องเขียน HTML ซ้ำทุกหน้า,
  **Custom Template Tags/Filters** ของตัวเอง, และ **Context Processors** ที่ทำให้ข้อมูล
  บางอย่าง (เช่น หมวดหมู่ทั้งหมด, บทความยอดนิยมที่เราเพิ่งสร้างใน Part นี้) ปรากฏได้ทุกหน้า
  โดยไม่ต้องส่งจาก View ทุกครั้ง

เมื่อจบ Phase 3 บล็อกของคุณจะกลายเป็นเว็บไซต์ที่ใช้งานได้จริงครบวงจร — มีหน้ารายการ
บทความ, หน้ารายละเอียด, ฟอร์มเขียน/แก้ไขบทความ, ระบบคอมเมนต์ที่ผู้ใช้กรอกได้จริงผ่านหน้าเว็บ
(ไม่ใช่แค่ผ่าน Admin หรือ shell อีกต่อไป) พร้อมดีไซน์ที่สม่ำเสมอทุกหน้าด้วย Template
Inheritance

เตรียมเปิด VS Code, Terminal, และรัน `python manage.py runserver` ทิ้งไว้ให้พร้อม
เพราะ Part 021 เราจะเริ่มเขียน Class-Based View ตัวแรกกันทันที!
