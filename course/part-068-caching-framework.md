# Part 068: Django Caching Framework เบื้องต้น

> **ขั้นตอนที่ 671-680 ของหลักสูตร** | Phase 8: Performance & Caching
>
> เป้าหมายของ Part นี้: เจาะลึก **Django Caching Framework** ตั้งแต่ระดับภาพรวม —
> Cache Backend ที่ Django รองรับทั้งหมดและควรเลือกใช้แบบไหนเมื่อไหร่ — ไปจนถึงการใช้งาน
> จริงครบทั้ง 3 ระดับ: **Per-view caching** ด้วย `@cache_page`, **Template Fragment
> Caching** ด้วย `{% cache %}` (ตามที่ Part 030 ค้างไว้), และ **Low-level Cache API**
> ที่ให้คุณควบคุมทุกรายละเอียดเอง จากนั้นเจาะลึกสิ่งที่มือใหม่มักพลาดที่สุด — **Cache
> Invalidation** ซึ่งได้ชื่อว่าเป็นหนึ่งในปัญหาที่ยากที่สุดใน Computer Science — พร้อม
> ลงมือ invalidate cache ผ่าน **Django Signals** จริงเมื่อมีการแก้ไขโพสต์ เรียนรู้เรื่อง
> `Vary` header สำหรับหน้าที่มีเนื้อหาต่างกันตามผู้ใช้ และปิดท้ายด้วยข้อควรระวังเรื่องการ
> cache QuerySet ที่ผิดพลาดบ่อยที่สุด ทุกตัวอย่างใน Part นี้จะใช้ **`LocMemCache`** เป็น
> backend หลักเพื่อโฟกัสที่หลักการของ caching framework เอง — ส่วนการตั้งค่า **Redis**
> แบบเต็มรูปแบบระดับ production (persistence, cluster, monitoring, cache stampede
> protection ด้วย distributed lock) จะเจาะลึกเต็ม ๆ ใน **Part 069: Redis Caching
> ขั้นสูง** ทันทีที่จบ Part นี้ คุณจะสามารถออกแบบระบบ caching ที่ทั้งเร็วและ **ไม่แสดง
> ข้อมูลเก่าผิดเวลา** ให้กับระบบบล็อกที่สร้างมาตลอดหลักสูตรได้อย่างมืออาชีพ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 671: ภาพรวม Django Cache Framework — Cache Backend ที่รองรับ (LocMemCache, FileBasedCache, DatabaseCache, Memcached, Redis) พร้อมตารางเปรียบเทียบ
- ขั้นตอนที่ 672: Per-view Caching ด้วย `@cache_page` Decorator
- ขั้นตอนที่ 673: Template Fragment Caching ด้วย `{% cache %}` Tag (ทำตามที่ Part 030 ค้างไว้)
- ขั้นตอนที่ 674: Low-level Cache API — `cache.get()`, `cache.set()`, `cache.delete()`, `cache.get_or_set()`
- ขั้นตอนที่ 675: การออกแบบ Cache Key ที่ดี และ Cache Versioning
- ขั้นตอนที่ 676: Cache Invalidation — "ปัญหาที่ยากที่สุดใน Computer Science" และกลยุทธ์ต่าง ๆ
- ขั้นตอนที่ 677: Invalidate Cache ผ่าน Django Signals (เมื่อ Post ถูกแก้ไข ให้ลบ Cache ที่เกี่ยวข้องอัตโนมัติ)
- ขั้นตอนที่ 678: `Vary` Headers และการ Cache หน้าที่มีเนื้อหาต่างกันตาม User (`Vary: Cookie`)
- ขั้นตอนที่ 679: ข้อควรระวังการ Cache QuerySet (QuerySet Lazy, การ Cache เฉพาะผลลัพธ์ที่ Evaluate แล้วเท่านั้น)
- ขั้นตอนที่ 680: สรุปและแบบฝึกหัด — Implement Caching เต็มรูปแบบสำหรับหน้า Blog List/Detail พร้อม Invalidation ที่ถูกต้อง

---

## ขั้นตอนที่ 671: ภาพรวม Django Cache Framework — Cache Backend ที่รองรับ พร้อมตารางเปรียบเทียบ

### 671.1 ทวนความจำ: ทำไมต้อง Caching

ตลอด Part 066-067 (Performance Profiling และ Query Optimization) คุณได้เรียนรู้วิธีทำให้
แต่ละ query เร็วขึ้นด้วย `select_related`/`prefetch_related`/`annotate`/index ที่เหมาะสม
แต่มี **ขีดจำกัดตามธรรมชาติ** อยู่เสมอ: ต่อให้ query ถูก optimize จนดีที่สุดแล้ว การไป
แตะฐานข้อมูล (round-trip เครือข่าย + disk I/O + query planning) ก็ยังช้ากว่าการอ่านค่า
จากหน่วยความจำเสมอเป็นหลักสิบถึงหลักพันเท่า

**Caching** คือการเก็บ "ผลลัพธ์ที่คำนวณเสร็จแล้ว" ไว้ในที่ที่อ่านเร็วกว่า (โดยทั่วไปคือ
memory) แล้วนำมาใช้ซ้ำแทนการคำนวณใหม่ทุกครั้ง หลักการที่ทวนมาตั้งแต่ Part 030 ข้อ 292.4
คือ:

> **"ข้อมูลที่เปลี่ยนไม่บ่อยแต่ถูกอ่านบ่อยมาก คือผู้สมัครอันดับหนึ่งสำหรับ caching"**

Part นี้จะสอนวิธี implement หลักการนี้อย่างถูกต้องและปลอดภัยครบทุกมิติ

### 671.2 สถาปัตยกรรม 3 ชั้นของ Django Caching Framework

Django Caching Framework แบ่งเป็น 3 ชั้นที่ทำงานซ้อนกัน จากระดับกว้างที่สุด (ควบคุมน้อย
ที่สุด) ไปจนถึงระดับละเอียดที่สุด (ควบคุมมากที่สุด):

```
┌─────────────────────────────────────────────────────────────────┐
│  ชั้นที่ 1: Per-site Cache (Middleware)                          │
│  cache ทั้งเว็บไซต์อัตโนมัติ — ควบคุมน้อยที่สุด แทบไม่ใช้ในทางปฏิบัติ  │
├─────────────────────────────────────────────────────────────────┤
│  ชั้นที่ 2: Per-view Cache (@cache_page)                         │
│  cache ทั้ง HTTP response ของ view เดียว — ขั้นตอนที่ 672          │
├─────────────────────────────────────────────────────────────────┤
│  ชั้นที่ 3: Template Fragment Cache ({% cache %})                │
│  cache เฉพาะบางส่วนของ template — ขั้นตอนที่ 673                 │
├─────────────────────────────────────────────────────────────────┤
│  ชั้นที่ 4: Low-level Cache API (cache.get/set/...)              │
│  cache อะไรก็ได้ที่คุณต้องการ ควบคุมทุกรายละเอียดเอง — ขั้นตอนที่ 674  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   Cache Backend    │  ← กำหนดใน settings.CACHES
                  │ (LocMem/Redis/...) │
                  └───────────────────┘
```

ทุกชั้นทั้ง 4 ชั้นด้านบน **ใช้ backend เดียวกันที่กำหนดไว้ใน `settings.CACHES`** — ความ
แตกต่างมีแค่ "อะไรถูก cache" และ "ใครเป็นคนตัดสินใจ" เท่านั้น หลักสูตรนี้จะข้ามชั้นที่ 1
(per-site cache) เพราะแทบไม่มีใครใช้ในโปรเจกต์จริงระดับมืออาชีพ (ควบคุมได้น้อยเกินไป
และมักจะ cache สิ่งที่ไม่ควร cache ปนกันไปด้วย เช่นหน้า admin หรือหน้าที่มี CSRF form)

### 671.3 ทวนการตั้งค่า `CACHES` จาก Part 010

Part 010 ข้อ 91.2 เกริ่นไว้ว่า `CACHES` คือ setting ที่บอก Django ว่าจะเก็บ cache ไว้ที่ไหน
ค่า default ถ้าไม่ตั้งอะไรเลยคือ:

```python
# ค่า default ของ Django ถ้าไม่ตั้ง CACHES เอง (ไม่ต้องเขียนบรรทัดนี้จริง)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
```

Part นี้จะตั้งค่าให้ชัดเจนเพื่อกำหนดพฤติกรรมที่คาดเดาได้:

```python
# config/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'django-mastery-cache',
        'TIMEOUT': 300,          # timeout เริ่มต้น (วินาที) ถ้า .set() ไม่ระบุ
        'OPTIONS': {
            'MAX_ENTRIES': 1000,     # จำนวน key สูงสุดก่อนเริ่มไล่ key เก่าออก (LRU-like)
            'CULL_FREQUENCY': 3,     # เมื่อเต็ม ไล่ออก 1/3 ของจำนวน entry ทั้งหมด
        },
    }
}
```

### 671.4 Cache Backend ทั้ง 5 แบบที่ Django รองรับในตัว

Django มี backend สำเร็จรูปให้ 5 แบบ (ไม่รวม backend จาก 3rd-party package เช่น
`django-redis` ที่ Part 069 จะแนะนำเพิ่ม) มาดูวิธีตั้งค่าแต่ละแบบทีละตัว

#### แบบที่ 1: `LocMemCache` — In-memory ต่อ Process (ค่า default)

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',   # ตั้งชื่อ (ป้องกันชนกันถ้ามีหลาย process รัน locmem ในเครื่องเดียวกันแบบแยก namespace)
    }
}
```

เก็บข้อมูลไว้ใน dictionary ของ Python **ในหน่วยความจำของ process ปัจจุบันเท่านั้น**
เร็วที่สุดในบรรดาทุก backend เพราะไม่มี network round-trip หรือ serialization ข้าม
process เลย แต่มีข้อจำกัดร้ายแรงที่ Part 048 ข้อ 478.1 อธิบายไว้แล้ว: **แต่ละ worker
process มี memory แยกกัน** ทำให้ cache ไม่ sync กันข้าม process/server

#### แบบที่ 2: `FileBasedCache` — เก็บเป็นไฟล์บน Disk

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```

เก็บแต่ละ cache entry เป็น**ไฟล์แยก**ในโฟลเดอร์ที่ระบุ (ต้องแน่ใจว่า process ของ Django
มีสิทธิ์เขียนโฟลเดอร์นั้น) ข้อดีคือ**ทุก process บนเครื่องเดียวกัน**เห็นไฟล์เดียวกันได้
(ต่างจาก `LocMemCache`) และข้อมูล**ไม่หายเมื่อ restart** process แต่ช้ากว่า memory มาก
เพราะต้องผ่าน disk I/O ทุกครั้ง และยังใช้ข้าม**หลายเครื่อง server** ไม่ได้ (เว้นแต่จะใช้
shared network filesystem ซึ่งมีปัญหา race condition และช้ากว่าเดิมอีก)

#### แบบที่ 3: `DatabaseCache` — เก็บในตารางฐานข้อมูล

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'django_cache_table',   # ชื่อตารางที่จะสร้าง
    }
}
```

ต้องสร้างตารางก่อนใช้งานจริงด้วย management command เฉพาะ:

```bash
python manage.py createcachetable
```

ข้อดีคือใช้ฐานข้อมูลที่มีอยู่แล้วเป็น shared storage ข้ามทุก process/server ได้ทันที
โดยไม่ต้องติดตั้ง service เพิ่ม แต่มีข้อเสียเชิงตรรกะที่ร้ายแรง: **มันเพิ่มภาระให้
ฐานข้อมูลตัวเดียวกับที่ caching ควรจะช่วยลดภาระ** ถ้าฐานข้อมูลกำลังเป็นคอขวดอยู่แล้ว
การ cache ลงฐานข้อมูลเดิมยิ่งซ้ำเติมปัญหา จึงเหมาะกับกรณีที่ **ไม่มี infrastructure
พิเศษเลย** และต้องการ shared cache ข้าม server แบบง่ายที่สุดเท่านั้น

#### แบบที่ 4: Memcached

```bash
pip install pymemcache
```

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```

**Memcached** คือ in-memory key-value store แยก process ที่ออกแบบมาเพื่อ caching
โดยเฉพาะเท่านั้น (ไม่มีจุดประสงค์อื่น) เร็วมากและใช้เป็น shared cache ข้ามหลาย
process/server ได้จริง แต่**ไม่มี persistence** (ข้อมูลหายหมดเมื่อ service restart)
และไม่มี data structure ซับซ้อนแบบที่ Redis มี (list, set, sorted set, hash)

#### แบบที่ 5: Redis (Backend ในตัวของ Django เอง)

```bash
pip install redis
```

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

ตั้งแต่ Django 4.0 เป็นต้นมา Django มี Redis backend **ในตัว** ไม่ต้องพึ่ง 3rd-party
package แล้ว (แม้ `django-redis` ที่ Part 048 ใช้ยังคงเป็นตัวเลือกยอดนิยมเพราะมีฟีเจอร์
เพิ่มเติม เช่น connection pooling ที่ปรับแต่งได้ละเอียดกว่า) Redis รวมข้อดีของ Memcached
(เร็วมาก, ใช้ข้าม process/server ได้) เข้ากับ **persistence** (เก็บข้อมูลลง disk ได้ ไม่
หายเมื่อ restart ถ้าตั้งค่า RDB/AOF) และ **data structure ขั้นสูง** ที่ทำให้ใช้เป็นทั้ง
cache, message queue (Part 073 Celery), และ session store ได้ในตัวเดียว

> Part นี้จะ**ยังไม่ตั้งค่า Redis จริง** ทุกตัวอย่างจะใช้ `LocMemCache` เพื่อให้คุณรัน
> โค้ดตัวอย่างได้ทันทีโดยไม่ต้องติดตั้ง service เพิ่ม — หลักการของ caching framework
> (cache key, invalidation, versioning) เหมือนกันทุกประการไม่ว่าจะใช้ backend ไหน
> การตั้งค่า Redis แบบเต็มรูปแบบ (persistence, cluster, Sentinel สำหรับ high
> availability, cache stampede protection ด้วย distributed lock) รอเรียนใน **Part 069**

### 671.5 ตารางเปรียบเทียบ Cache Backend ทั้ง 5 แบบ

| ประเด็น | `LocMemCache` | `FileBasedCache` | `DatabaseCache` | Memcached | Redis |
|---|---|---|---|---|---|
| ความเร็ว | เร็วที่สุด (in-process) | ปานกลาง (disk I/O) | ช้าที่สุด (ผ่าน DB) | เร็วมาก | เร็วมาก |
| Persistent ข้าม Restart | ❌ ไม่ | ✅ ใช่ | ✅ ใช่ | ❌ ไม่ | ✅ ใช่ (ตั้งค่าได้) |
| แชร์ข้ามหลาย Process ในเครื่องเดียวกัน | ❌ ไม่ | ✅ ใช่ | ✅ ใช่ | ✅ ใช่ | ✅ ใช่ |
| แชร์ข้ามหลายเครื่อง Server | ❌ ไม่ | ❌ ไม่ (ปกติ) | ✅ ใช่ | ✅ ใช่ | ✅ ใช่ |
| ต้องติดตั้ง Service เพิ่ม | ❌ ไม่ต้อง | ❌ ไม่ต้อง | ❌ ไม่ต้อง | ✅ ต้อง | ✅ ต้อง |
| Data Structure ขั้นสูง (list/set/hash) | ❌ ไม่มี | ❌ ไม่มี | ❌ ไม่มี | ❌ ไม่มี | ✅ มี |
| เหมาะกับ | Development, Test, Single-process เล็ก ๆ | Staging เล็ก ๆ ไม่มี infra พิเศษ | ไม่มี infra เลยแต่อยากได้ shared cache | Pure caching ล้วน ๆ ที่ทีมคุ้นเคยอยู่แล้ว | **Production ทั่วไป** (มาตรฐานอุตสาหกรรม) |
| ความนิยมใน Production จริง | ⭐ (เฉพาะ dev) | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 671.6 ทำไม Part นี้เลือกสอนด้วย `LocMemCache` เป็นหลัก

| เหตุผล | รายละเอียด |
|---|---|
| รันได้ทันทีไม่ต้องติดตั้งอะไรเพิ่ม | ผู้เรียนทุกคนเปิดโปรเจกต์ Django ตามหลักสูตรมาแล้วมี `LocMemCache` ให้ใช้ทันทีโดยไม่ต้องรอ Part 069 |
| API เหมือนกันทุก backend | `cache.get()`, `cache.set()`, `@cache_page`, `{% cache %}` เขียนโค้ดเหมือนกันทุกตัวอักษรไม่ว่าจะสลับไปใช้ backend ไหนก็ตาม — เปลี่ยนแค่ `settings.CACHES` บรรทัดเดียว โค้ดแอปพลิเคชันไม่ต้องแก้เลย |
| เข้าใจ "หลักการ" caching ได้บริสุทธิ์ | ไม่มีสัญญาณรบกวนจากการตั้งค่า connection pool, timeout เครือข่าย, หรือปัญหา infra ของ Redis ปนเข้ามาตอนเรียนหลักการพื้นฐาน |
| ข้อจำกัดของ `LocMemCache` (Part 048 ข้อ 478.1) จะสอนเองใน Part 069 | เมื่อเข้าใจหลักการแน่นแล้ว การเห็นปัญหาจริงของ `LocMemCache` ในระบบ multi-worker จะทำให้เข้าใจว่าทำไม Redis ถึงจำเป็นสำหรับ production ได้ลึกซึ้งกว่า |

### 671.7 ตารางสรุปขั้นตอนที่ 671

| ประเด็น | สรุป |
|---|---|
| Caching framework มีกี่ชั้น | 4 ชั้น: per-site (ไม่สอน), per-view, template fragment, low-level API |
| ทุกชั้นใช้อะไรร่วมกัน | Backend เดียวกันที่กำหนดใน `settings.CACHES` |
| Backend ที่รองรับในตัว | `LocMemCache`, `FileBasedCache`, `DatabaseCache`, Memcached, Redis |
| Backend มาตรฐาน Production | Redis (เจาะลึกเต็มรูปแบบใน Part 069) |
| Backend ที่ Part นี้ใช้สอน | `LocMemCache` เพื่อโฟกัสหลักการล้วน ๆ |

---

## ขั้นตอนที่ 672: Per-view Caching ด้วย `@cache_page` Decorator

### 672.1 โจทย์: หน้ารายการบทความที่ Query หนักแต่เนื้อหาเปลี่ยนไม่บ่อย

ทวนโครงสร้าง Model จาก Part 030 ที่จะใช้ตลอดทั้ง Part นี้:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.urls import reverse
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


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name='posts',
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='posts',
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author_name = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f"ความคิดเห็นโดย {self.author_name} บน {self.post.title}"
```

หน้ารายการบทความ (`PostListView`) ต้อง query `Post`, `Category`, นับจำนวนคอมเมนต์ของ
แต่ละโพสต์ — งานหนักที่**ผลลัพธ์เหมือนเดิมสำหรับทุกคนที่เข้ามาดู** (ไม่ผูกกับ user
เฉพาะราย) จึงเป็นตัวเลือกที่ดีมากสำหรับ **per-view caching**: cache **ทั้ง HTTP
response** ไว้ แล้วส่งอันเดิมกลับไปให้ผู้ใช้คนถัดไปโดยไม่ต้อง render ใหม่เลย

### 672.2 ใช้ `@cache_page` กับ Function-Based View

```python
# blog/views.py
from django.core.paginator import Paginator
from django.shortcuts import render
from django.views.decorators.cache import cache_page

from .models import Post


@cache_page(60 * 5)   # cache ผลลัพธ์ไว้ 5 นาที (300 วินาที)
def post_list(request):
    post_qs = (
        Post.objects
        .filter(is_published=True)
        .select_related('category', 'author')
    )
    paginator = Paginator(post_qs, 10)
    page_obj = paginator.get_page(request.GET.get('page'))
    return render(request, 'blog/post_list.html', {'page_obj': page_obj})
```

`cache_page(timeout)` รับ **timeout เป็นวินาที** เป็น argument บังคับ — เมื่อมี request
เข้ามาที่ URL นี้ Django จะเช็คก่อนว่ามี response ที่ cache ไว้แล้วสำหรับ URL (+
query string) นี้หรือไม่ ถ้ามีและยังไม่หมดอายุ **จะคืนค่าจาก cache ทันทีโดยไม่เรียก
`post_list()` เลยด้วยซ้ำ** — view function ทั้งฟังก์ชันถูกข้ามไปเลย

### 672.3 Cache Key ของ `@cache_page` คำนวณจากอะไร

ประเด็นสำคัญที่ต้องเข้าใจ: `@cache_page` สร้าง cache key จาก **full URL path รวม query
string** (ผ่านฟังก์ชันภายใน `django.utils.cache.get_cache_key` ที่จะกล่าวถึงอีกครั้งใน
ขั้นตอนที่ 677) ซึ่งหมายความว่า:

| URL ที่เข้า | Cache Key ที่ได้ |
|---|---|
| `/blog/` | Key A |
| `/blog/?page=2` | Key B (**คนละ key กับ `/blog/`**) |
| `/blog/?page=2&category=django` | Key C (**คนละ key อีกแล้ว**) |

แต่ละหน้า/query string ที่ต่างกันจะถูก cache แยกกันเป็นอิสระ — นี่คือทั้งข้อดี (แม่นยำ
ตาม URL จริง) และข้อควรระวัง (ถ้ามี query string หลากหลายมาก เช่น filter/sort หลายแบบ
จะเปลือง memory เก็บ cache หลายชุดโดยไม่จำเป็น)

### 672.4 ใช้กับ Class-Based View ผ่าน `method_decorator`

`@cache_page` เป็น decorator สำหรับฟังก์ชัน แต่ CBV มี `dispatch()` เป็นเมธอด จึงต้องใช้
`method_decorator` ห่ออีกชั้น (ทวนเทคนิคเดียวกับที่ใช้ตอนใส่ `@login_required` ให้ CBV
ใน Phase 4):

```python
# blog/views.py
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.generic import ListView

from .models import Post


@method_decorator(cache_page(60 * 5), name='dispatch')
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'page_obj'
    paginate_by = 10

    def get_queryset(self):
        return (
            Post.objects
            .filter(is_published=True)
            .select_related('category', 'author')
        )
```

`name='dispatch'` บอกให้ decorator ห่อรอบเมธอด `dispatch()` ซึ่งเป็นจุดแรกสุดที่ CBV
ทุกตัวเรียกเมื่อรับ request เข้ามา — การห่อที่จุดนี้เท่ากับห่อทั้ง view (ครอบคลุมทั้ง
`get()`, `post()`, ฯลฯ)

### 672.5 กำหนด Cache Backend และ Key Prefix เฉพาะ

`cache_page` รับ keyword argument เพิ่มเติมอีก 2 ตัว:

```python
# blog/views.py
@cache_page(60 * 5, cache='default', key_prefix='blog_post_list')
def post_list(request):
    ...
```

| Parameter | หน้าที่ |
|---|---|
| `timeout` (บังคับ, ตำแหน่งแรก) | อายุ cache เป็นวินาที |
| `cache` | ชื่อ alias ของ `CACHES` ที่จะใช้ (ค่า default คือ `'default'`) — มีประโยชน์เมื่อแยก cache alias สำหรับ page cache ออกจาก cache อื่น ๆ |
| `key_prefix` | string นำหน้า cache key เสมอ — ใช้แยก namespace เมื่อหลาย view อาจสร้าง key ชนกัน หรือใช้ตอน deploy ใหม่เพื่อ "ทิ้ง cache เก่าทั้งหมด" โดยเปลี่ยนค่านี้ (จะกล่าวถึงเพิ่มในขั้อ 675) |

### 672.6 ทางเลือกระดับ URLconf: Cache ที่ `urls.py` แทนที่ `views.py`

ในบางกรณี (เช่น view มาจาก 3rd-party package ที่แก้โค้ดไม่ได้) สามารถใส่ `cache_page`
ที่ `urls.py` แทนได้ ผลลัพธ์เหมือนกันทุกประการ:

```python
# blog/urls.py
from django.urls import path
from django.views.decorators.cache import cache_page

from . import views

app_name = 'blog'

urlpatterns = [
    path('', cache_page(60 * 5)(views.post_list), name='list'),
    path('<slug:slug>/', views.post_detail, name='detail'),
]
```

### 672.7 ข้อควรระวังสำคัญ: อย่า Cache หน้าที่มี CSRF Token หรือเนื้อหาเฉพาะ User

`@cache_page` cache **ทั้ง HTTP response แบบดิบ ๆ** รวมถึง HTML ทุกไบต์ที่ render ออกมา
ถ้าหน้านั้นมี `{% csrf_token %}` ฝังอยู่ (เช่น หน้าที่มีฟอร์มคอมเมนต์) **ผู้ใช้คนที่ 2
เป็นต้นไปจะได้รับ CSRF token ของผู้ใช้คนแรกที่ทำให้เกิด cache** ซึ่งจะทำให้ฟอร์ม submit
ไม่ผ่าน (`403 Forbidden: CSRF verification failed`) เพราะ token ไม่ตรงกับ session ของ
ตัวเอง

| สถานการณ์ | Cache ทั้งหน้าด้วย `@cache_page` ได้ไหม |
|---|---|
| หน้ารายการบทความ (ไม่มีฟอร์ม, เนื้อหาเหมือนกันทุกคน) | ✅ ได้ปลอดภัย |
| หน้ารายละเอียดบทความที่**ไม่มี**ฟอร์มคอมเมนต์แบบ inline | ✅ ได้ปลอดภัย |
| หน้ารายละเอียดบทความที่**มี**ฟอร์มคอมเมนต์แบบ inline (`{% csrf_token %}`) | ❌ อันตราย — ต้องแยกฟอร์มออกมาโหลดผ่าน AJAX/HTMX แยกจากส่วนที่ cache หรือใช้ template fragment caching (ขั้อ 673) เฉพาะส่วนเนื้อหาแทน |
| หน้า Dashboard ส่วนตัวที่แสดงชื่อผู้ใช้ที่ login | ❌ อันตรายมาก — ผู้ใช้คนถัดไปจะเห็นชื่อของคนแรก (ต้องใช้ `Vary: Cookie` ตามขั้อ 678 หรือไม่ cache ทั้งหน้าเลย) |

### 672.8 ตารางสรุปขั้นตอนที่ 672

| ประเด็น | สรุป |
|---|---|
| `@cache_page(timeout)` | Cache ทั้ง HTTP response ของ view |
| Cache Key คำนวณจาก | Full URL path + query string |
| ใช้กับ CBV อย่างไร | `@method_decorator(cache_page(...), name='dispatch')` |
| Parameter เพิ่มเติม | `cache=` (เลือก backend alias), `key_prefix=` (namespace) |
| อันตรายที่ต้องระวังที่สุด | ห้าม cache หน้าที่มี CSRF token หรือเนื้อหาเฉพาะ user โดยไม่ใส่ `Vary` header ให้ถูกต้อง |

---

## ขั้นตอนที่ 673: Template Fragment Caching ด้วย `{% cache %}` Tag (ทำตามที่ Part 030 ค้างไว้)

### 673.1 ทวนสัญญาที่ Part 030 ค้างไว้

Part 030 ข้อ 292.4 เขียน `categories_for_sidebar` context processor ที่ query ฐานข้อมูล
ทุก request แล้วเกริ่นไว้ว่าจะ cache ผลลัพธ์ด้วย low-level API แบบง่าย ๆ ก่อน และสัญญาไว้
ว่า **"การเลือก cache backend, cache invalidation, per-view caching, template fragment
caching, และ cache versioning จะถูกเจาะลึกเต็มรูปแบบใน Part 068"** — ขั้นตอนนี้คือการ
ทำตามสัญญานั้นในส่วนของ **Template Fragment Caching**

Part 028 ข้อ 278.4 ก็เกริ่นเช่นกันว่า `{% cache %}` tag คือทางแก้ปัญหา performance ของ
template ที่ทรงพลังที่สุด และบอกไว้ว่า **"เราจะยังไม่ใช้ `{% cache %}` ในหลักสูตรตอนนั้น
เพราะต้องเข้าใจ Django Caching Framework ทั้งระบบก่อน"** — ตอนนี้ถึงเวลาแล้ว

### 673.2 ปัญหาเดิม: Sidebar หมวดหมู่ Query ฐานข้อมูลทุก Request

```python
# blog/context_processors.py (ทบทวนจาก Part 030 ข้อ 292.2 — ยังไม่ cache)
from django.db.models import Count, Q

from .models import Category


def categories_for_sidebar(request):
    categories = (
        Category.objects
        .annotate(post_count=Count('posts', filter=Q(posts__is_published=True)))
        .filter(post_count__gt=0)
        .order_by('name')
    )
    return {'sidebar_categories': categories}
```

ปัญหาคือ context processor นี้ query ฐานข้อมูล**ทุกครั้งที่ render template ใด ๆ**
ทั้งเว็บไซต์ (ทวนจาก Part 030 ข้อ 292.3) — วิธีแก้ที่ถูกต้องกว่า `cache.get_or_set` ใน
context processor เอง (ซึ่งยัง query database ผ่าน Python code ที่ไม่ได้ผูกกับ template
โดยตรง) คือใช้ `{% cache %}` tag ห่อรอบ**เฉพาะส่วน HTML ของ sidebar** ในระดับ template
เลย ซึ่งให้ประโยชน์เพิ่มคือ **cache ทั้ง HTML ที่ render แล้ว** ไม่ใช่แค่ queryset ดิบ ๆ
(ประหยัดทั้งเวลา query และเวลา render ในครั้งถัดไป)

### 673.3 ใช้ `{% cache %}` Tag ครั้งแรก

```html
<!-- templates/partials/sidebar.html -->
{% load cache %}

<aside class="sidebar">
    <h3>หมวดหมู่</h3>
    {% cache 300 sidebar_categories %}
        <ul class="sidebar__categories">
            {% for category in sidebar_categories %}
                <li>
                    <a href="{% url 'blog:category-detail' slug=category.slug %}">
                        {{ category.name }} ({{ category.post_count }})
                    </a>
                </li>
            {% empty %}
                <li>ยังไม่มีหมวดหมู่ที่มีบทความ</li>
            {% endfor %}
        </ul>
    {% endcache %}
</aside>
```

ไวยากรณ์คือ `{% cache TIMEOUT FRAGMENT_NAME %}...{% endcache %}`:

| ส่วนประกอบ | ความหมาย |
|---|---|
| `{% load cache %}` | โหลด template tag library (บังคับเสมอ เหมือน `{% load static %}`) |
| `300` | Timeout เป็นวินาที (เหมือน `@cache_page`) |
| `sidebar_categories` | **ชื่อ fragment** — ตัวระบุที่ไม่ซ้ำกันสำหรับ fragment นี้ (ไม่ต้องมี quote เพราะเป็นแค่ label ไม่ใช่ตัวแปร) |

ครั้งแรกที่ template นี้ถูก render Django จะ render HTML ของส่วนที่อยู่ระหว่าง
`{% cache %}` กับ `{% endcache %}` ตามปกติ แล้ว**เก็บ HTML ที่ได้ (เป็น string ล้วน ๆ)**
ไว้ใน cache ครั้งถัดไปที่ template นี้ถูก render ภายใน 300 วินาที Django จะดึง HTML
string ที่ cache ไว้มาแปะตรงนั้นทันที **โดยไม่ประมวลผล `{% for %}` loop หรือแตะ
`sidebar_categories` (ตัวแปร) เลย**

### 673.4 Cache Key ของ Fragment คำนวณจากอะไร: `make_template_fragment_key`

Django คำนวณ cache key ของแต่ละ fragment จากฟังก์ชัน
`django.core.cache.utils.make_template_fragment_key(fragment_name, vary_on=None)`
ซึ่งจะมีประโยชน์มากตอนต้องการลบ cache นี้ด้วยมือ (ขั้อ 673.7 และขั้อ 677):

```python
>>> from django.core.cache.utils import make_template_fragment_key
>>> make_template_fragment_key('sidebar_categories')
'template.cache.sidebar_categories.d41d8cd98f00b204e9800998ecf8427e'
```

รูปแบบคือ `template.cache.<fragment_name>.<hash>` — ส่วน `<hash>` คำนวณจาก
`vary_on` (จะอธิบายต่อในขั้อถัดไป) ถ้าไม่ระบุ `vary_on` เลย hash จะเป็นค่าคงที่เสมอ

### 673.5 Cache แยกตาม Argument เพิ่มเติม (`vary_on`)

Fragment ส่วนใหญ่ในโลกจริงไม่ได้เหมือนกันสำหรับทุกคน เช่น สมมติต้องการ cache ส่วนแสดง
"บทความล่าสุดในหมวดหมู่นี้" ที่เนื้อหาต่างกันไปตามแต่ละหมวดหมู่:

```html
<!-- templates/blog/category_detail.html -->
{% load cache %}

{% cache 300 latest_posts_in_category category.slug %}
    <ul>
        {% for post in category.posts.all|slice:':5' %}
            <li><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></li>
        {% endfor %}
    </ul>
{% endcache %}
```

การเพิ่ม `category.slug` ต่อท้ายชื่อ fragment (เรียกว่า `vary_on` argument) ทำให้
Django สร้าง **cache key แยกกันสำหรับแต่ละค่าของ `category.slug`** เทียบเท่ากับการเรียก
`make_template_fragment_key('latest_posts_in_category', [category.slug])` เบื้องหลัง
— สามารถใส่ `vary_on` ได้หลายตัวคั่นด้วยช่องว่าง:

```html
{% cache 300 post_comments_preview post.id request.LANGUAGE_CODE %}
    ...
{% endcache %}
```

ตัวอย่างนี้ cache แยกกันตามทั้ง `post.id` **และ** ภาษาปัจจุบัน — ถูกต้องเพราะเนื้อหา
ควรต่างกันสำหรับแต่ละโพสต์และแต่ละภาษา ถ้าลืมใส่ `post.id` เป็น `vary_on` **ทุกโพสต์
จะแชร์ cache fragment เดียวกันโดยไม่ตั้งใจ** — เป็นบั๊กเงียบที่ตรวจจับยากมาก (คล้ายกับ
บั๊ก context processor ชื่อชนกันใน Part 030 ข้อ 293.2)

### 673.6 ระบุ Cache Backend เฉพาะด้วย `using`

```html
{% cache 300 sidebar_categories using="template_fragments" %}
    ...
{% endcache %}
```

`using` (ต้องเป็น string ใส่ quote) ระบุ cache alias จาก `settings.CACHES` ที่ต้องการ
ใช้แทน `'default'` — มีประโยชน์เมื่อแยก cache alias สำหรับ fragment cache ออกจาก
cache อื่น (คล้ายแนวคิดเดียวกับที่ Part 048 ข้อ 478.3 แยก alias สำหรับ throttle)

### 673.7 ลบ Fragment Cache ด้วยมือเมื่อข้อมูลเปลี่ยน

เพราะรู้สูตรคำนวณ cache key แล้ว (`make_template_fragment_key`) จึงลบ fragment cache
เฉพาะเจาะจงได้ทันทีเมื่อข้อมูลเปลี่ยน โดยไม่ต้องรอ timeout หมดอายุเอง:

```python
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key

# ลบ cache ของ sidebar (ไม่มี vary_on)
cache.delete(make_template_fragment_key('sidebar_categories'))

# ลบ cache ของหมวดหมู่ที่ระบุ (มี vary_on เป็น slug)
cache.delete(make_template_fragment_key('latest_posts_in_category', ['django-tips']))
```

นี่คือเทคนิคที่จะนำไปใช้จริงในขั้นตอนที่ 677 เมื่อเขียน signal สำหรับ invalidate cache
โดยอัตโนมัติเมื่อ `Post` หรือ `Category` มีการเปลี่ยนแปลง

### 673.8 เปรียบเทียบ Template Fragment Caching กับ Per-view Caching

| ประเด็น | `@cache_page` (ขั้อ 672) | `{% cache %}` (ขั้อนี้) |
|---|---|---|
| ขอบเขตที่ cache | ทั้ง HTTP response | เฉพาะส่วนของ template ที่เลือก |
| ปลอดภัยกับหน้าที่มี CSRF token ไหม | ❌ อันตรายถ้า cache ทั้งหน้าที่มีฟอร์ม | ✅ ปลอดภัย — ห่อเฉพาะส่วนที่ไม่มีฟอร์ม แล้วปล่อยฟอร์มให้ render สดทุกครั้ง |
| ใช้กับหน้าที่มีเนื้อหาผสม (บาง block cache ได้ บาง block cache ไม่ได้) | ❌ ทำไม่ได้ (all-or-nothing) | ✅ ทำได้ดีมาก — ห่อเฉพาะ block ที่ query หนักและเหมือนกันทุกคน |
| ควบคุมความละเอียด (granularity) | หยาบ (ระดับ view/URL) | ละเอียด (ระดับ block ใน template) |
| Invalidate ด้วยมือ | ต้องคำนวณ hash คีย์เอง (ขั้อ 677) | มี `make_template_fragment_key()` ให้ใช้ตรง ๆ |

### 673.9 ตารางสรุปขั้นตอนที่ 673

| ประเด็น | สรุป |
|---|---|
| Syntax | `{% load cache %}` แล้ว `{% cache TIMEOUT NAME [vary_on...] [using="alias"] %}...{% endcache %}` |
| เก็บอะไรใน cache | HTML string ที่ render เสร็จแล้วของ block นั้น |
| Cache Key คำนวณจาก | `make_template_fragment_key(fragment_name, vary_on)` |
| ทำไมต้องใส่ `vary_on` ให้ครบ | ถ้าเนื้อหาต่างกันตาม object/ภาษา/ผู้ใช้ ต้องใส่สิ่งนั้นเป็น `vary_on` ไม่งั้นทุกคนจะแชร์ cache เดียวกันผิดพลาด |
| ลบ cache ด้วยมือ | `cache.delete(make_template_fragment_key(name, vary_on))` |

---

## ขั้นตอนที่ 674: Low-level Cache API — `cache.get()`, `cache.set()`, `cache.delete()`, `cache.get_or_set()`

### 674.1 เมื่อไหร่ควรใช้ Low-level API แทนสองชั้นก่อนหน้า

`@cache_page` และ `{% cache %}` สะดวกแต่ควบคุมได้จำกัด — เมื่อสิ่งที่ต้องการ cache
**ไม่ใช่ HTTP response หรือ HTML fragment** แต่เป็นข้อมูลอื่น (ผลลัพธ์การคำนวณ, ค่าที่
ดึงจาก API ภายนอก, ผลรวมทางสถิติ) หรือเมื่อต้องการควบคุม key/timeout/invalidation ด้วย
ตัวเองอย่างละเอียด ต้องใช้ **Low-level Cache API** โดยตรง

### 674.2 Import และ Object พื้นฐาน

```python
from django.core.cache import cache          # cache alias 'default'
from django.core.cache import caches          # dict-like object เข้าถึงทุก alias

my_cache = caches['default']       # เทียบเท่ากับ import cache ตรง ๆ
other_cache = caches['sessions']   # เข้าถึง alias อื่นที่ตั้งไว้ใน settings.CACHES
```

`cache` ที่ import จาก `django.core.cache` โดยตรงคือ shortcut ไปยัง `caches['default']`
เสมอ — ใช้ `caches['ชื่อ_alias']` เมื่อต้องการ backend ที่ไม่ใช่ default

### 674.3 `cache.set()` และ `cache.get()`: พื้นฐานที่สุด

```python
from django.core.cache import cache

# เก็บค่าไว้ 300 วินาที
cache.set('total_published_posts', 42, timeout=300)

# อ่านค่ากลับมา
count = cache.get('total_published_posts')
print(count)   # 42

# ถ้า key ไม่มีอยู่ (ไม่เคย set หรือหมดอายุแล้ว) จะได้ None
missing = cache.get('key_ที่ไม่มีอยู่จริง')
print(missing)   # None

# ระบุ default เองแทนที่จะได้ None เปล่า ๆ (ป้องกัน bug ที่สับสนระหว่าง "ไม่มี key" กับ "ค่าจริงคือ None")
count = cache.get('total_published_posts', 0)
```

**ข้อควรระวังสำคัญ**: ถ้าไม่ระบุ `timeout` ตอน `.set()` จะใช้ค่า `TIMEOUT` ที่ตั้งไว้ใน
`settings.CACHES['default']` (ค่า default ของ Django เองคือ 300 วินาทีถ้าไม่ได้ตั้ง)
ถ้าต้องการให้ **ไม่มีวันหมดอายุ** ให้ระบุ `timeout=None` อย่างชัดเจน (แต่ต้อง invalidate
ด้วยมือเองเสมอ ไม่งั้นข้อมูลจะเก่าตลอดไปจนกว่าจะมีคนลบ):

```python
cache.set('site_launch_date', '2024-01-01', timeout=None)   # ไม่มีวันหมดอายุ
```

### 674.4 `cache.add()`: Set เฉพาะเมื่อ Key ยังไม่มีอยู่

```python
# ถ้า key ยังไม่มี → set สำเร็จ คืนค่า True
success = cache.add('daily_visitor_count', 1, timeout=86400)
print(success)   # True

# เรียกซ้ำอีกครั้ง (key มีอยู่แล้ว) → ไม่ set ทับ คืนค่า False
success = cache.add('daily_visitor_count', 999, timeout=86400)
print(success)         # False
print(cache.get('daily_visitor_count'))   # ยังเป็น 1 เหมือนเดิม ไม่ถูกทับด้วย 999
```

`add()` มีประโยชน์มากสำหรับ pattern แบบ "ทำครั้งแรกเท่านั้น" เช่น ป้องกันการส่งอีเมล
ต้อนรับซ้ำ หรือทำ distributed lock แบบง่าย ๆ (จะกล่าวถึงเพิ่มเติมในขั้อ 676 เรื่อง
cache stampede)

### 674.5 `cache.get_or_set()`: Pattern ที่ใช้บ่อยที่สุดในทางปฏิบัติ

ทวนจาก Part 030 ข้อ 292.4 ที่ใช้ pattern นี้ไปแล้ว:

```python
from django.core.cache import cache
from django.db.models import Count, Q

from .models import Category


def get_sidebar_categories():
    def _fetch():
        return list(
            Category.objects
            .annotate(post_count=Count('posts', filter=Q(posts__is_published=True)))
            .filter(post_count__gt=0)
            .order_by('name')
        )

    return cache.get_or_set('blog:sidebar_categories', _fetch, timeout=300)
```

`get_or_set(key, default, timeout=DEFAULT_TIMEOUT)` ทำงานเป็นขั้นตอนเดียว (atomic
ในทางตรรกะของโค้ดฝั่งเรา แม้ backend อาจมี race condition เล็กน้อยที่จะกล่าวถึงในขั้อ
676):

1. เช็คว่ามีค่าใน cache สำหรับ `key` นี้หรือไม่
2. ถ้ามี → คืนค่านั้นทันที (ไม่เรียก `default` เลย)
3. ถ้าไม่มี → ถ้า `default` เป็น **callable** (ฟังก์ชัน) จะเรียกมันเพื่อคำนวณค่า แล้ว
   `.set()` ผลลัพธ์นั้นเข้า cache ด้วย `timeout` ที่กำหนด ก่อนคืนค่ากลับ
4. ถ้า `default` **ไม่ใช่** callable (เช่น ใส่ตัวเลขหรือ list ตรง ๆ) จะใช้ค่านั้นตรง ๆ
   และ set เข้า cache เหมือนกัน

**ข้อดีสำคัญของการส่ง `default` เป็น callable แทนค่าตรง ๆ**: ถ้า cache **มี** ค่าอยู่
แล้ว ฟังก์ชัน `_fetch()` **จะไม่ถูกเรียกเลย** — หมายความว่า query ฐานข้อมูลจะไม่เกิดขึ้น
เลยเมื่อ cache hit ถ้าเขียนแบบ `cache.get_or_set(key, _fetch(), timeout=300)` (เรียก
`_fetch()` ทันทีตอนส่ง argument) query จะเกิดขึ้น**ทุกครั้ง**ไม่ว่า cache จะ hit หรือ
miss ก็ตาม — เป็นข้อผิดพลาดที่พบบ่อยมากในทีมที่เพิ่งเริ่มใช้ caching framework

```python
# ❌ ผิด — เรียก _fetch() ทันที ไม่ว่า cache จะ hit หรือไม่
cache.get_or_set('key', _fetch(), timeout=300)

# ✅ ถูก — ส่งฟังก์ชันไปเฉย ๆ ไม่เรียก จะถูกเรียกก็ต่อเมื่อ cache miss เท่านั้น
cache.get_or_set('key', _fetch, timeout=300)
```

### 674.6 `cache.delete()` และ `cache.delete_many()`

```python
cache.delete('blog:sidebar_categories')          # ลบ key เดียว

cache.delete_many([                               # ลบหลาย key พร้อมกันในคำสั่งเดียว
    'blog:sidebar_categories',
    'blog:total_published_posts',
    'blog:latest_post_id',
])
```

`delete()` ไม่ raise exception แม้ key นั้นจะไม่มีอยู่แล้ว (ลบซ้ำได้อย่างปลอดภัย)
`delete_many()` มีประสิทธิภาพดีกว่าการเรียก `delete()` วนลูปเมื่อใช้กับ backend แบบ
network round-trip (Redis/Memcached) เพราะส่ง command เดียวแทนที่จะส่งหลายครั้ง

### 674.7 `cache.get_many()` และ `cache.set_many()`: ลด Round-trip เมื่อมีหลาย Key

```python
# set หลาย key พร้อมกันในคำสั่งเดียว
cache.set_many({
    'blog:post:1:view_count': 150,
    'blog:post:2:view_count': 89,
    'blog:post:3:view_count': 342,
}, timeout=3600)

# get หลาย key พร้อมกัน — คืนเฉพาะ key ที่มีอยู่จริงเป็น dict
result = cache.get_many(['blog:post:1:view_count', 'blog:post:2:view_count', 'blog:post:999:view_count'])
print(result)
# {'blog:post:1:view_count': 150, 'blog:post:2:view_count': 89}
# สังเกตว่า key 999 หายไปเลย (ไม่ใช่ None) เพราะไม่มีอยู่จริงใน cache
```

### 674.8 `cache.incr()` และ `cache.decr()`: นับเลขแบบ Atomic

```python
cache.set('blog:post:1:view_count', 0, timeout=None)

cache.incr('blog:post:1:view_count')          # +1 → ได้ 1 (คืนค่าใหม่กลับมาด้วย)
cache.incr('blog:post:1:view_count', delta=5)  # +5 → ได้ 6

cache.decr('blog:post:1:view_count')          # -1 → ได้ 5
```

**ข้อควรระวัง**: ถ้า key ยังไม่เคยถูก `.set()` มาก่อน การเรียก `.incr()` จะ raise
`ValueError` ทันที (ไม่ใช่สร้าง key ใหม่ให้อัตโนมัติแบบที่บางคนคาดหวัง) จึงต้อง
`cache.add(key, 0)` หรือ `cache.set(key, 0)` ก่อนเสมอถ้ายังไม่มั่นใจว่า key มีอยู่แล้ว:

```python
def increment_view_count(post_id):
    key = f'blog:post:{post_id}:view_count'
    cache.add(key, 0, timeout=None)   # สร้างเป็น 0 ถ้ายังไม่มี (ไม่ทับถ้ามีอยู่แล้ว)
    return cache.incr(key)
```

ข้อดีสำคัญของ `.incr()`/`.decr()` คือ**เป็น atomic operation ในระดับ backend** (อย่าง
น้อยสำหรับ Redis/Memcached) ต่างจากการทำ `cache.set(key, cache.get(key) + 1)` เองที่มี
ช่องว่างให้เกิด **race condition** ได้ถ้ามีหลาย request เข้ามาพร้อมกัน (read-modify-write
ไม่ atomic)

### 674.9 `cache.touch()`: ต่ออายุ Key โดยไม่ต้องอ่าน/เขียนค่าใหม่

```python
cache.touch('blog:sidebar_categories', timeout=600)   # ต่ออายุเป็น 600 วินาทีนับจากนี้
```

มีประโยชน์เมื่อรู้ว่าข้อมูลยัง valid อยู่ (เช่นเพิ่งตรวจสอบแล้วว่าไม่มีการเปลี่ยนแปลง)
แต่ไม่อยากคำนวณค่าใหม่หรือส่งข้อมูลซ้ำผ่านเครือข่ายเปล่า ๆ

### 674.10 `cache.clear()`: ล้าง Cache ทั้งหมด (ระวัง!)

```python
cache.clear()   # ลบทุก key ใน cache alias นี้ทิ้งทั้งหมด
```

**ใช้ด้วยความระมัดระวังสูงสุด** โดยเฉพาะใน production ที่แชร์ cache alias เดียวกันกับ
ระบบอื่น (เช่น throttle counter จาก Part 048 ที่ใช้ cache เดียวกัน — ถ้าเรียก
`cache.clear()` โดยไม่ตั้งใจ throttle counter ของทุก user จะถูกรีเซ็ตพร้อมกันหมด)
ในทางปฏิบัติมักใช้เฉพาะใน test setup/teardown (จะกล่าวถึงในขั้อ 677) มากกว่าในโค้ด
production จริง

### 674.11 ตารางสรุป Low-level Cache API ทั้งหมด

| Method | หน้าที่ | คืนค่า |
|---|---|---|
| `cache.set(key, value, timeout)` | เก็บค่า (ทับของเดิมถ้ามี) | `None` |
| `cache.get(key, default=None)` | อ่านค่า | ค่าที่เก็บไว้ หรือ `default` ถ้าไม่มี |
| `cache.add(key, value, timeout)` | เก็บค่าเฉพาะเมื่อยังไม่มี key นี้ | `True`/`False` |
| `cache.get_or_set(key, default, timeout)` | อ่าน หรือคำนวณ+เก็บถ้ายังไม่มี | ค่าที่ได้ |
| `cache.delete(key)` | ลบ 1 key | `True`/`False` |
| `cache.delete_many(keys)` | ลบหลาย key | `None` |
| `cache.get_many(keys)` | อ่านหลาย key | `dict` เฉพาะ key ที่มีจริง |
| `cache.set_many(dict, timeout)` | เก็บหลาย key | `list` ของ key ที่ set ไม่สำเร็จ (ถ้ามี) |
| `cache.incr(key, delta=1)` | บวกเลขแบบ atomic | ค่าใหม่หลังบวก |
| `cache.decr(key, delta=1)` | ลบเลขแบบ atomic | ค่าใหม่หลังลบ |
| `cache.touch(key, timeout)` | ต่ออายุโดยไม่แก้ค่า | `True`/`False` |
| `cache.clear()` | ลบทุก key ใน alias นี้ | `None` |

---

## ขั้นตอนที่ 675: การออกแบบ Cache Key ที่ดี และ Cache Versioning

### 675.1 หลักการตั้งชื่อ Cache Key ที่ดี

Cache key ก็คือ string ธรรมดา — Django ไม่ได้บังคับรูปแบบใด ๆ แต่ทีมมืออาชีพยึดหลัก
การตั้งชื่อที่ทำให้ debug และจัดการง่ายขึ้นมาก:

```python
# ❌ ไม่ดี — กำกวม ไม่รู้ว่ามาจากแอปไหน เก็บอะไร
cache.set('cats', categories, timeout=300)
cache.set('count', 42, timeout=300)

# ✅ ดี — namespace ชัดเจนด้วย colon-separated segments
cache.set('blog:sidebar:categories', categories, timeout=300)
cache.set('blog:post:total_published_count', 42, timeout=300)
cache.set(f'blog:post:{post_id}:view_count', 0, timeout=None)
cache.set(f'blog:post:{post_id}:comments:page:{page_number}', comments, timeout=180)
```

### 675.2 ตารางหลักการตั้งชื่อ Cache Key

| หลักการ | เหตุผล |
|---|---|
| ขึ้นต้นด้วยชื่อแอป (`blog:`, `shop:`) | ป้องกันชนกันข้ามแอปในโปรเจกต์เดียวกัน (คล้ายบทเรียนเรื่อง context processor ชื่อชนกันใน Part 030 ข้อ 293.2) |
| ใช้ `:` คั่น namespace เป็นลำดับชั้น | อ่านง่าย, ใช้ pattern matching ได้สะดวกเมื่อ debug ผ่าน Redis CLI (`KEYS blog:post:*`) ใน Part 069 |
| ใส่ **ทุกตัวแปรที่ทำให้ผลลัพธ์ต่างกัน** ลงใน key | ถ้าผลลัพธ์ต่างกันตาม `post_id`, `page_number`, หรือ `language` ต้องใส่ทั้งหมดนี้ใน key ไม่งั้นจะได้ผลลัพธ์ผิดคนผิดหน้า (เหมือนปัญหา `vary_on` ในขั้อ 673.5) |
| หลีกเลี่ยงชื่อกำกวมสั้นเกินไป | `cats`, `data`, `result` ไม่บอกอะไรเลยเมื่อกลับมาอ่านโค้ดอีก 6 เดือนข้างหน้า |
| เก็บสูตรสร้าง key ไว้ที่เดียว (helper function) | ป้องกันการพิมพ์ผิดหรือลืม segment ใดไปเมื่อต้องสร้าง key เดียวกันจากหลายที่ในโค้ด |

ตัวอย่าง helper function ที่รวมสูตรสร้าง key ไว้ที่เดียว:

```python
# blog/cache_keys.py
def sidebar_categories_key():
    return 'blog:sidebar:categories'


def post_view_count_key(post_id):
    return f'blog:post:{post_id}:view_count'


def post_list_page_key(page_number, category_slug=None):
    if category_slug:
        return f'blog:post_list:category:{category_slug}:page:{page_number}'
    return f'blog:post_list:all:page:{page_number}'
```

```python
# blog/views.py
from django.core.cache import cache

from .cache_keys import post_view_count_key


def increment_post_views(post_id):
    key = post_view_count_key(post_id)
    cache.add(key, 0, timeout=None)
    return cache.incr(key)
```

การรวมสูตรสร้าง key ไว้ในไฟล์เดียวทำให้ทั้งโค้ดที่ `.set()`/`.get()` และโค้ดที่
`.delete()` (ในขั้อ 677) เรียกใช้ฟังก์ชันเดียวกันเสมอ — ไม่มีทางพิมพ์ key ผิดจนหา
ไม่เจอตอน invalidate

### 675.3 ทำความเข้าใจ `KEY_PREFIX` และ `VERSION` ใน `settings.CACHES`

Django มีกลไก 2 อย่างในระดับ backend ที่ทำงาน "อยู่เบื้องหลัง" ทุก key โดยอัตโนมัติ:

```python
# config/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'django-mastery-cache',
        'KEY_PREFIX': 'djmastery',   # ← เติมหน้าทุก key อัตโนมัติ
        'VERSION': 1,                # ← เติมต่อท้าย prefix อัตโนมัติ
    }
}
```

เมื่อเรียก `cache.set('blog:sidebar:categories', ...)` key จริงที่ถูกเก็บใน backend
ไม่ใช่ `'blog:sidebar:categories'` ตรง ๆ แต่เป็น:

```
djmastery:1:blog:sidebar:categories
```

รูปแบบคือ `<KEY_PREFIX>:<VERSION>:<key ที่คุณเขียน>` (คำนวณโดยฟังก์ชัน
`django.core.cache.backends.base.default_key_func`)

### 675.4 ประโยชน์ของ `KEY_PREFIX`: แยก Namespace ระหว่าง Environment

ถ้า staging และ production ใช้ Redis instance เดียวกัน (บางทีมทำเพื่อประหยัดค่าใช้จ่าย
infra) การตั้ง `KEY_PREFIX` ต่างกันในแต่ละ environment ป้องกันไม่ให้ cache ของ staging
ไปปนกับ production โดยไม่ตั้งใจ:

```python
# config/settings.py
import os

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'KEY_PREFIX': os.environ.get('DJANGO_ENV', 'dev'),   # 'dev', 'staging', 'production'
    }
}
```

### 675.5 ประโยชน์ของ `VERSION`: Bulk Invalidation แบบไม่ต้องรู้จัก Key ทุกตัว

นี่คือกลยุทธ์ invalidation ที่ทรงพลังที่สุดอย่างหนึ่ง: แทนที่จะต้องไล่ลบทุก key ที่
เกี่ยวข้องทีละตัว (ซึ่งอาจมีเป็นร้อยเป็นพัน key ถ้า cache แยกตาม pagination/filter) ให้
**เพิ่มค่า `VERSION`** แทน — key เก่าทั้งหมดจะกลายเป็น "มองไม่เห็น" ทันที (เพราะ full
key ที่คำนวณได้จะไม่ตรงกับของเดิมอีกต่อไป) โดยไม่ต้องลบอะไรออกจาก backend เลยจริง ๆ
(ปล่อยให้ TTL หมดอายุไปเองในที่สุด)

```python
from django.core.cache import cache

# เพิ่ม version ของ key เดียว (เฉพาะ key นั้น)
cache.set('blog:sidebar:categories', categories, timeout=300, version=1)
cache.incr_version('blog:sidebar:categories')   # ตอนนี้ข้อมูลอยู่ที่ version=2

# อ่านค่าที่ version ปัจจุบัน (2) จะได้ None เพราะยังไม่เคย set ที่ version นี้
print(cache.get('blog:sidebar:categories'))                # None (default version = 2 แล้ว)
print(cache.get('blog:sidebar:categories', version=1))      # ยังอ่านค่าเก่าที่ version=1 ได้ถ้าต้องการ
```

ในทางปฏิบัติที่ใช้บ่อยกว่าคือ **เก็บเลข version ปัจจุบันไว้ใน cache key แยกต่างหาก**
แล้วนำมาประกอบเป็นส่วนหนึ่งของ key ตอนอ่าน/เขียนข้อมูลจริง — pattern นี้เรียกว่า
**"Generation-based caching"** และจะถูกใช้เต็มรูปแบบในขั้อ 677 เพื่อ invalidate
per-view cache ที่คำนวณ key แบบ hashed (ซึ่งลบด้วยชื่อ key ตรง ๆ ไม่ได้):

```python
# blog/cache_keys.py
from django.core.cache import cache

POSTS_GENERATION_KEY = 'blog:posts:generation'


def get_posts_generation():
    """คืนเลข generation ปัจจุบัน ถ้ายังไม่มีให้เริ่มที่ 1"""
    return cache.get_or_set(POSTS_GENERATION_KEY, 1, timeout=None)


def bump_posts_generation():
    """เรียกทุกครั้งที่ Post มีการเปลี่ยนแปลง (จะเรียกจาก signal ในขั้อ 677)"""
    cache.add(POSTS_GENERATION_KEY, 1, timeout=None)
    return cache.incr(POSTS_GENERATION_KEY)


def post_list_page_key(page_number):
    generation = get_posts_generation()
    return f'blog:post_list:gen:{generation}:page:{page_number}'
```

เมื่อ `bump_posts_generation()` ถูกเรียก (เช่นตอน Post ถูกแก้ไข) **key ทุกตัวที่คำนวณ
จาก generation เก่าจะไม่ถูกอ่านอีกเลย** เพราะ `post_list_page_key()` ครั้งถัดไปจะได้
generation ใหม่เสมอ ไม่ต้องรู้ล่วงหน้าว่า page ไหนบ้างที่เคยถูก cache ไว้ — แก้ปัญหา
"ต้องรู้จัก key ทุกตัวก่อนถึงจะลบได้" ได้อย่างสมบูรณ์

### 675.6 ตารางสรุปขั้นตอนที่ 675

| แนวคิด | ประโยชน์ |
|---|---|
| Namespace ด้วย `:` (`app:entity:id:field`) | อ่านง่าย ป้องกันชนกัน ค้นหาด้วย pattern ได้ |
| รวมสูตรสร้าง key ไว้ในฟังก์ชันเดียว | ป้องกันพิมพ์ผิด/ลืม segment เมื่อต้อง `.get()`/`.set()`/`.delete()` key เดียวกันจากหลายที่ |
| `KEY_PREFIX` | แยก namespace ข้าม environment (dev/staging/prod) ที่แชร์ backend เดียวกัน |
| `VERSION` (global) | Bump ทีเดียวเพื่อ invalidate ทุก key ใน cache alias นั้น (ใช้น้อย เพราะกระทบวงกว้างเกินไป) |
| Generation-based key (custom) | Bump เฉพาะกลุ่ม key ที่เกี่ยวข้อง โดยไม่ต้องรู้จัก key ทุกตัวล่วงหน้า — เทคนิคที่ใช้บ่อยที่สุดในทางปฏิบัติ |

---

## ขั้นตอนที่ 676: Cache Invalidation — "ปัญหาที่ยากที่สุดใน Computer Science" และกลยุทธ์ต่าง ๆ

### 676.1 คำพูดที่โด่งดังที่สุดในวงการ Caching

Phil Karlton วิศวกรซอฟต์แวร์ชื่อดังเคยกล่าวไว้ (มักถูกอ้างถึงบ่อยที่สุดในวงการ):

> "There are only two hard things in Computer Science: cache invalidation and naming
> things." (มีเรื่องยากอยู่แค่สองอย่างใน Computer Science: การทำให้ cache หมดอายุอย่าง
> ถูกต้อง กับการตั้งชื่อ)

ทำไม cache invalidation ถึงยาก? เพราะมัน**ไม่ใช่ปัญหาทางเทคนิคล้วน ๆ** แต่เป็นปัญหาที่
ต้องตอบคำถามเชิงตรรกะทางธุรกิจให้ถูกทุกจุดพร้อมกัน: "ข้อมูลนี้เปลี่ยนได้จากกี่ทาง?",
"ใครบ้างที่ต้องเห็นข้อมูลใหม่ทันที และใครยอมรับข้อมูลเก่าได้บ้าง?", "ถ้าลืม invalidate
จุดใดจุดหนึ่ง จะเกิดอะไรขึ้น?" — คำตอบผิดแม้แค่จุดเดียวก็ทำให้ผู้ใช้เห็นข้อมูลผิดโดยไม่มี
error ใด ๆ เตือนเลย (เหมือนบั๊ก context processor ชนกันใน Part 030)

### 676.2 กลยุทธ์ที่ 1: TTL-based Expiration (พึ่ง Timeout อย่างเดียว)

วิธีที่ง่ายที่สุด: ตั้ง `timeout` สั้น ๆ แล้วปล่อยให้ cache หมดอายุเอง ไม่ต้อง invalidate
ด้วยมือเลย

```python
cache.set('blog:sidebar:categories', categories, timeout=60)   # หมดอายุใน 1 นาที
```

| ข้อดี | ข้อเสีย |
|---|---|
| Implement ง่ายที่สุด ไม่ต้องเขียน signal/logic เพิ่ม | ผู้ใช้อาจเห็นข้อมูลเก่านานถึง `timeout` วินาทีหลังข้อมูลเปลี่ยนจริง |
| ทนต่อบั๊ก — ต่อให้ลืม invalidate จุดไหน ข้อมูลก็จะถูกต้องเองภายในเวลาไม่นาน | ไม่เหมาะกับข้อมูลที่ต้อง**ถูกต้องทันที** (เช่น ยอดเงินในบัญชี, สถานะการชำระเงิน) |

**เหมาะกับ**: ข้อมูลที่ "เก่าไปนิดหน่อยก็ยังยอมรับได้" เช่น sidebar หมวดหมู่, จำนวน
บทความทั้งหมด, รายการบทความยอดนิยม — ไม่ใช่ข้อมูลที่ critical ต่อความถูกต้องทันที

### 676.3 กลยุทธ์ที่ 2: Explicit Invalidation (ลบด้วยมือทันทีที่ข้อมูลเปลี่ยน)

ตั้ง `timeout` ยาว (หรือ `None` = ไม่มีวันหมดอายุเอง) แล้ว**ลบ cache ด้วยโค้ดทันทีที่
รู้ว่าข้อมูลเปลี่ยน** (มักผ่าน signal — เจาะลึกเต็มรูปแบบในขั้อ 677):

```python
def update_post(post_id, **fields):
    post = Post.objects.get(pk=post_id)
    for field, value in fields.items():
        setattr(post, field, value)
    post.save()
    cache.delete(f'blog:post:{post_id}:detail')   # ลบทันทีที่รู้ว่าเปลี่ยน
```

| ข้อดี | ข้อเสีย |
|---|---|
| ข้อมูลถูกต้องทันทีหลังเปลี่ยน (ไม่ต้องรอ timeout) | ต้องหาให้ครบ**ทุกจุด**ในโค้ดที่ทำให้ข้อมูลเปลี่ยน (ทุก view, management command, signal, admin action) — ถ้าลืมจุดใดจุดหนึ่ง จะเกิด **stale cache** ที่ debug ยากมาก |
| ควบคุมแม่นยำที่สุด | เพิ่มความซับซ้อนของโค้ด (coupling ระหว่างจุดที่แก้ข้อมูลกับจุดที่ต้องรู้ว่า cache ไหนเกี่ยวข้อง) |

**เหมาะกับ**: ข้อมูลที่ critical ต่อความถูกต้อง และมีจุดที่ทำให้ข้อมูลเปลี่ยนได้ชัดเจน
จำกัด (เช่นเปลี่ยนผ่าน Model.save() เท่านั้น) — ใช้ Django Signals ผูกไว้ที่ model level
(ขั้อ 677) เพื่อไม่ต้องไล่แก้ทุก view ด้วยมือ

### 676.4 กลยุทธ์ที่ 3: Cache Versioning / Generation (ทวนจากขั้อ 675.5)

Bump generation number แทนการลบ key ทีละตัว — เหมาะกับกรณีที่มี key จำนวนมากที่
เกี่ยวข้องกัน (เช่น หลาย page ของ pagination) ที่ไม่สามารถไล่รู้จัก key ทุกตัวได้ล่วงหน้า

### 676.5 ปัญหา Cache Stampede (Thundering Herd)

ปัญหาที่ซับซ้อนกว่าที่มักถูกมองข้าม: เมื่อ cache entry ที่มีการเข้าถึงสูงมากหมดอายุ
พอดี ถ้ามีหลาย request เข้ามา**พร้อมกันในเวลาเดียวกัน** ทุก request จะเห็นว่า cache
miss พร้อมกันหมด แล้ว**แห่กันไปคำนวณค่าใหม่ (query ฐานข้อมูล) พร้อมกันทั้งหมด** —
ในกรณีเลวร้ายที่สุด ฐานข้อมูลอาจโดนกระหน่ำด้วย query เดียวกันหลายพันครั้งพร้อมกัน
จนล่ม ทั้งที่ปกติมี cache ช่วยป้องกันอยู่แล้ว

```
เวลา T=0: cache หมดอายุพอดี
เวลา T=0.001: 500 requests เข้ามาพร้อมกัน (traffic peak)
  → ทุก request เจอ cache miss
  → ทุก request วิ่งไป query ฐานข้อมูลพร้อมกันหมด (500 queries พร้อมกัน!)
  → ฐานข้อมูลรับภาระหนักผิดปกติ อาจ timeout หรือล่มได้
```

### 676.6 ทางแก้ Cache Stampede: Jittered Expiration

เทคนิคง่ายที่สุด: **สุ่มเวลาหมดอายุเล็กน้อย (jitter)** แทนที่จะใช้ timeout คงที่เป๊ะ ๆ
เพื่อกระจายเวลาที่ cache entry ต่าง ๆ หมดอายุไม่ให้ตรงกันเป๊ะ:

```python
import random

from django.core.cache import cache


def cache_with_jitter(key, value_func, base_timeout=300, jitter_seconds=30):
    """เพิ่ม timeout แบบสุ่มเล็กน้อยเพื่อกระจายเวลาหมดอายุ ป้องกัน stampede"""
    timeout = base_timeout + random.randint(-jitter_seconds, jitter_seconds)
    value = value_func()
    cache.set(key, value, timeout=timeout)
    return value
```

### 676.7 ทางแก้ Cache Stampede: Lock ด้วย `cache.add()`

เทคนิคที่แม่นยำกว่า: ใช้ `cache.add()` (ทวนจากขั้อ 674.4) เป็น **distributed lock**
แบบง่าย ๆ เพื่อให้มีแค่ 1 request เท่านั้นที่ได้ไป query ฐานข้อมูลจริง ส่วน request
อื่น ๆ ที่มาพร้อมกันให้รอหรือใช้ค่าเก่าไปพลาง ๆ:

```python
import time

from django.core.cache import cache


def get_expensive_data(cache_key, compute_func, timeout=300, lock_timeout=10):
    value = cache.get(cache_key)
    if value is not None:
        return value

    lock_key = f'{cache_key}:lock'
    got_lock = cache.add(lock_key, '1', timeout=lock_timeout)

    if got_lock:
        try:
            value = compute_func()
            cache.set(cache_key, value, timeout=timeout)
        finally:
            cache.delete(lock_key)
        return value
    else:
        # มี request อื่นกำลังคำนวณอยู่แล้ว — รอสั้น ๆ แล้วลองอ่าน cache อีกครั้ง
        time.sleep(0.1)
        return cache.get(cache_key) or compute_func()
```

> **หมายเหตุ**: `cache.add()` เป็น atomic operation ที่เชื่อถือได้ในระดับ backend
> จริง ๆ (Redis/Memcached) แต่ **`LocMemCache` ที่ Part นี้ใช้สอนมีข้อจำกัดเรื่อง
> thread-safety ในบาง edge case** เพราะทำงานในหน่วยความจำ Python ล้วน ๆ — เทคนิค
> distributed lock ที่แม่นยำและปลอดภัยระดับ production เต็มรูปแบบ (รวมถึง
> `django-redis` cache lock helper) จะเจาะลึกใน **Part 069**

### 676.8 ตารางเปรียบเทียบกลยุทธ์ Invalidation ทั้งหมด

| กลยุทธ์ | ความแม่นยำ | ความซับซ้อนในการ Implement | เหมาะกับ |
|---|---|---|---|
| TTL-based (timeout สั้น) | ต่ำ (มี delay ก่อนข้อมูลใหม่ปรากฏ) | ต่ำที่สุด | ข้อมูลที่เก่าเล็กน้อยได้ (sidebar, สถิติ) |
| Explicit invalidation (signal) | สูง (ทันทีที่ข้อมูลเปลี่ยน) | ปานกลาง-สูง | ข้อมูลที่ต้องถูกต้องทันที และจุดเปลี่ยนแปลงชัดเจน |
| Versioning/Generation | สูง | ปานกลาง | กลุ่ม key จำนวนมากที่ไม่รู้จักล่วงหน้าทั้งหมด (pagination) |
| Jittered expiration | เท่ากับ TTL-based | ต่ำ | ป้องกัน stampede ของข้อมูลที่ traffic สูงมาก |
| Lock-based (cache.add) | สูงและป้องกัน stampede | สูง | ข้อมูลที่ traffic สูงมาก**และ**คำนวณหนักมาก (ป้องกัน DB ล่มตอน cache miss พร้อมกัน) |

### 676.9 ตารางสรุปขั้นตอนที่ 676

| ประเด็น | สรุป |
|---|---|
| ทำไม invalidation ถึงยาก | เป็นปัญหาเชิงตรรกะทางธุรกิจ ไม่ใช่แค่เทคนิค ต้องตอบให้ถูกทุกจุดที่ข้อมูลเปลี่ยนได้ |
| กลยุทธ์หลัก 3 แบบ | TTL-based, Explicit invalidation, Versioning/Generation |
| ปัญหาขั้นสูงที่ต้องรู้จัก | Cache Stampede (Thundering Herd) เมื่อ cache หมดอายุพร้อมกันภายใต้ traffic สูง |
| ทางแก้ Stampede | Jittered expiration (ง่าย) หรือ Lock ด้วย `cache.add()` (แม่นยำกว่า) |
| Lock แบบเต็มรูปแบบระดับ Production | รอเจาะลึกใน Part 069 (Redis) |

---

## ขั้นตอนที่ 677: Invalidate Cache ผ่าน Django Signals (เมื่อ Post ถูกแก้ไข ให้ลบ Cache ที่เกี่ยวข้องอัตโนมัติ)

### 677.1 ทำไมต้องใช้ Signals แทนการลบ Cache ในทุก View

ทวนจาก Part 019 (Django Signals) — `post_save` และ `post_delete` ทำงาน **ทุกครั้งที่
`Model.save()`/`Model.delete()` ถูกเรียก ไม่ว่าจะเรียกจากที่ไหน**: view ปกติ, Django
Admin, management command, Django shell, หรือ API endpoint จาก Part 040-048 ทั้งหมด
การผูก invalidation logic ไว้ที่ signal (ซึ่งอยู่ระดับ model) แทนที่จะเขียนซ้ำในทุก view
ทำให้**รับประกันได้ว่าจะไม่มีจุดไหนลืม invalidate cache** ตราบใดที่การเปลี่ยนแปลงเกิด
ผ่าน `.save()`/`.delete()` ปกติ (ทวนหลักการ DRY จาก Part 001 ข้อ 3.4)

### 677.2 ออกแบบ Cache ที่ต้อง Invalidate ทั้งหมดเมื่อ `Post` เปลี่ยน

รวบรวม cache ทั้งหมดที่เกี่ยวข้องกับ `Post` ที่สร้างมาตลอด Part นี้:

| Cache | สร้างจากขั้นตอนไหน | Key/วิธีคำนวณ |
|---|---|---|
| Template fragment: sidebar หมวดหมู่ | ขั้อ 673 | `make_template_fragment_key('sidebar_categories')` |
| Template fragment: บทความล่าสุดในหมวดหมู่ | ขั้อ 673.5 | `make_template_fragment_key('latest_posts_in_category', [category.slug])` |
| Low-level: จำนวนบทความที่เผยแพร่ | ขั้อ 674.3 | `'blog:post:total_published_count'` |
| Per-view: หน้ารายละเอียดโพสต์ (`@cache_page`) | ขั้อ 672 | คำนวณผ่าน `django.utils.cache.get_cache_key` (ขั้อ 677.4) |
| Per-view: หน้ารายการโพสต์ (generation-based) | ขั้อ 675.5 | `bump_posts_generation()` |

### 677.3 เขียน `signals.py` สำหรับ Invalidation

```python
# blog/signals.py
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key
from django.db.models.signals import post_delete, post_save
from django.dispatch import receiver

from .cache_keys import bump_posts_generation
from .models import Category, Post


@receiver(post_save, sender=Post)
@receiver(post_delete, sender=Post)
def invalidate_post_related_cache(sender, instance, **kwargs):
    """
    เรียกทุกครั้งที่ Post ถูกสร้าง/แก้ไข/ลบ ไม่ว่าจะผ่านช่องทางไหนก็ตาม
    (view ปกติ, Django Admin, management command, shell)
    """
    # 1. ลบ cache จำนวนบทความที่เผยแพร่ (ขั้อ 674.3)
    cache.delete('blog:post:total_published_count')

    # 2. ลบ template fragment ของ sidebar (ขั้อ 673.3) — จำนวนโพสต์ต่อหมวดหมู่เปลี่ยนได้
    cache.delete(make_template_fragment_key('sidebar_categories'))

    # 3. ลบ template fragment ของหมวดหมู่ที่โพสต์นี้สังกัดอยู่ (ขั้อ 673.5)
    if instance.category_id:
        cache.delete(
            make_template_fragment_key(
                'latest_posts_in_category', [instance.category.slug]
            )
        )

    # 4. Bump generation เพื่อ invalidate หน้ารายการทุกหน้าแบบ bulk (ขั้อ 675.5)
    bump_posts_generation()

    # 5. ลบ per-view cache ของหน้ารายละเอียดโพสต์นี้โดยเฉพาะ (ขั้อ 677.4)
    invalidate_post_detail_page_cache(instance)


@receiver(post_save, sender=Category)
@receiver(post_delete, sender=Category)
def invalidate_category_related_cache(sender, instance, **kwargs):
    """เรียกทุกครั้งที่ Category ถูกสร้าง/แก้ไข/ลบ"""
    cache.delete(make_template_fragment_key('sidebar_categories'))
    cache.delete(make_template_fragment_key('latest_posts_in_category', [instance.slug]))
```

### 677.4 เทคนิคขั้นสูง: Invalidate Per-view Cache (`@cache_page`) ด้วย `get_cache_key`

ปัญหาของ `@cache_page` คือ cache key ของมันถูก**คำนวณจาก hash ของ URL** (ทวนจากขั้อ
672.3) ทำให้ลบด้วย string key ตรง ๆ ไม่ได้ถ้าไม่รู้สูตร hash ที่แน่นอน — Django มี
ฟังก์ชันสาธารณะที่คำนวณ key แบบเดียวกับที่ `@cache_page` ใช้ภายใน ให้เรียกใช้ได้:
`django.utils.cache.get_cache_key(request)`

เพราะฟังก์ชันนี้ต้องการ `HttpRequest` object ที่มี path ตรงกับหน้าที่ต้องการลบ cache
เราจึงสร้าง fake request ด้วย `RequestFactory` (เครื่องมือเดียวกับที่ใช้เขียนเทสต์ใน
Part 030 ข้อ 291.5):

```python
# blog/signals.py (ต่อจากด้านบน)
from django.test import RequestFactory
from django.utils.cache import get_cache_key


def invalidate_post_detail_page_cache(post):
    """
    ลบ cache ของหน้ารายละเอียดโพสต์ที่ cache ไว้ด้วย @cache_page (ขั้อ 672)
    โดยคำนวณ cache key แบบเดียวกับที่ CacheMiddleware ใช้ภายใน
    """
    request = RequestFactory().get(post.get_absolute_url())
    cache_key = get_cache_key(request, cache=cache)
    if cache_key:
        cache.delete(cache_key)
```

**ข้อจำกัดที่ต้องเข้าใจ**: `get_cache_key()` จะคืนค่า `None` ถ้า **ไม่เคยมี response
ถูก cache ไว้สำหรับ URL นี้มาก่อนเลย** เพราะฟังก์ชันนี้ต้องอ่าน "header cache" ภายใน
(บันทึกไว้ตอน response แรกถูก cache ว่า response นั้น `Vary` ตาม header อะไรบ้าง — จะ
อธิบายกลไกนี้ต่อในขั้อ 678) ถ้าโพสต์เพิ่งถูกสร้างและยังไม่มีใครเข้าดูเลยสักครั้ง จะไม่มี
อะไรให้ลบอยู่แล้วตั้งแต่แรก — โค้ดด้านบนจึงตรวจสอบ `if cache_key:` ก่อนเรียก `.delete()`
เพื่อความปลอดภัย

### 677.5 เชื่อมต่อ Signal ผ่าน `apps.py`

ทวนรูปแบบมาตรฐานจาก Part 019: ต้อง import module `signals.py` ใน `ready()` ของ
`AppConfig` เพื่อให้ signal ถูกลงทะเบียนตอนแอปเริ่มทำงาน

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'

    def ready(self):
        import blog.signals  # noqa: F401
```

```python
# blog/__init__.py
default_app_config = 'blog.apps.BlogConfig'  # ไม่จำเป็นตั้งแต่ Django 3.2+ ถ้าตั้งชื่อ apps.py มาตรฐาน แต่ใส่ไว้เพื่อความชัดเจน
```

> ตรวจสอบให้แน่ใจว่า `INSTALLED_APPS` ใน `settings.py` ชี้ไปที่ `'blog.apps.BlogConfig'`
> หรือแค่ `'blog'` (Django จะหา `AppConfig` เริ่มต้นให้อัตโนมัติ) มิเช่นนั้น `ready()`
> จะไม่ถูกเรียก และ signal จะไม่ทำงานเลยแบบเงียบ ๆ (ไม่มี error ใด ๆ) — เป็นข้อผิดพลาด
> ที่พบบ่อยที่สุดเมื่อ signal "ดูเหมือนไม่ทำงาน"

### 677.6 เขียน Test ยืนยันว่า Invalidation ทำงานจริง

```python
# blog/tests/test_cache_invalidation.py
from django.contrib.auth import get_user_model
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key
from django.test import TestCase

from blog.models import Category, Post

User = get_user_model()


class PostCacheInvalidationTests(TestCase):
    def setUp(self):
        cache.clear()   # เริ่มทุกเทสต์ด้วย cache ที่ว่างเปล่า ป้องกัน state รั่วข้ามเทสต์
        self.user = User.objects.create_user(username='writer', password='x')
        self.category = Category.objects.create(name='Django')

    def tearDown(self):
        cache.clear()

    def test_saving_post_invalidates_sidebar_fragment_cache(self):
        # จำลองว่า sidebar fragment ถูก cache ไว้แล้วก่อนหน้านี้
        cache.set(make_template_fragment_key('sidebar_categories'), '<ul>...</ul>', timeout=300)
        self.assertIsNotNone(cache.get(make_template_fragment_key('sidebar_categories')))

        Post.objects.create(
            title='บทความใหม่', content='...', category=self.category,
            author=self.user, is_published=True,
        )

        # หลังบันทึกโพสต์ใหม่ fragment cache ต้องถูกลบไปแล้ว
        self.assertIsNone(cache.get(make_template_fragment_key('sidebar_categories')))

    def test_saving_post_invalidates_total_published_count_cache(self):
        cache.set('blog:post:total_published_count', 999, timeout=300)

        Post.objects.create(
            title='อีกบทความ', content='...', category=self.category,
            author=self.user, is_published=True,
        )

        self.assertIsNone(cache.get('blog:post:total_published_count'))

    def test_deleting_post_also_triggers_invalidation(self):
        post = Post.objects.create(
            title='บทความที่จะถูกลบ', content='...', category=self.category,
            author=self.user, is_published=True,
        )
        cache.set('blog:post:total_published_count', 1, timeout=300)

        post.delete()

        self.assertIsNone(cache.get('blog:post:total_published_count'))
```

### 677.7 ตารางสรุปขั้นตอนที่ 677

| ประเด็น | สรุป |
|---|---|
| ทำไมใช้ Signal แทนแก้ทุก View | รับประกันว่า invalidation เกิดขึ้นไม่ว่าการแก้ไขจะมาจากช่องทางไหน (view/admin/shell) |
| Signal ที่ใช้ | `post_save` และ `post_delete` ผูกกับทั้ง `Post` และ `Category` |
| ลบ Template Fragment Cache | ใช้ `make_template_fragment_key()` คำนวณ key เดียวกับตอน set |
| ลบ Per-view Cache (`@cache_page`) | ใช้ `django.utils.cache.get_cache_key()` + `RequestFactory` จำลอง request |
| ข้อจำกัดของ `get_cache_key()` | คืน `None` ถ้าไม่เคยมี response ถูก cache ไว้สำหรับ URL นั้นมาก่อน |
| ต้องไม่ลืม | เชื่อมต่อ `signals.py` ผ่าน `AppConfig.ready()` มิฉะนั้น signal จะไม่ทำงานเงียบ ๆ |

---

## ขั้นตอนที่ 678: `Vary` Headers และการ Cache หน้าที่มีเนื้อหาต่างกันตาม User (`Vary: Cookie`)

### 678.1 ปัญหา: หน้าเดียวกัน แต่เนื้อหาต่างกันตาม User

สมมติหน้ารายละเอียดโพสต์แสดงข้อความ "สวัสดี {{ request.user.username }}" สำหรับผู้ใช้
ที่ login แล้ว และ "เข้าสู่ระบบ" สำหรับผู้ใช้ทั่วไป — ถ้า cache หน้านี้ด้วย `@cache_page`
ตรง ๆ โดยไม่ทำอะไรเพิ่ม **URL เดียวกันจะถูก cache ครั้งเดียว** แล้วส่งให้ทุกคนเหมือนกัน
หมด รวมถึงชื่อผู้ใช้ของคนแรกที่ทำให้เกิด cache ด้วย — ปัญหาความปลอดภัยร้ายแรง (ข้อมูล
รั่วไหลข้าม user)

### 678.2 `Vary` Header คืออะไร

`Vary` เป็น **HTTP response header มาตรฐาน** ที่บอกว่า "response นี้อาจแตกต่างกันไป
ตามค่าของ request header ที่ระบุ" — ทั้ง browser cache และ Django's cache framework
ใช้ header นี้เพื่อรู้ว่าควรแยก cache key ตามอะไรเพิ่มเติมนอกเหนือจาก URL:

```
HTTP/1.1 200 OK
Vary: Cookie
Content-Type: text/html
```

ความหมายคือ: "response นี้ขึ้นอยู่กับค่าของ header `Cookie` ด้วย ไม่ใช่แค่ URL อย่าง
เดียว — ถ้า `Cookie` ต่างกัน ต้องถือว่าเป็นคนละ response กัน ห้ามใช้ cache ร่วมกัน"

### 678.3 `@vary_on_cookie`: บอก Django ให้แยก Cache ตาม Session/Cookie

```python
# blog/views.py
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie
from django.views.generic import DetailView

from .models import Post


@method_decorator(vary_on_cookie, name='dispatch')
@method_decorator(cache_page(60 * 5), name='dispatch')
class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
```

**ลำดับของ decorator สำคัญมาก**: `vary_on_cookie` ต้อง apply **หลัง** `cache_page`
ในความหมายของโค้ด (อยู่ด้านบนกว่าเมื่อเขียนแบบ stack decorator) เพราะ Python apply
decorator จากล่างขึ้นบน — `cache_page` ต้องเห็น `Vary: Cookie` header ที่
`vary_on_cookie` เติมเข้าไปแล้ว **ก่อน** ที่ `cache_page` จะคำนวณ cache key สุดท้าย
(กลไกนี้คือสิ่งที่ `get_cache_key()` ในขั้อ 677.4 อ่านมาใช้)

เมื่อใช้ `vary_on_cookie` แล้ว Django จะแยก cache สำหรับแต่ละค่าของ `Cookie` header
(ซึ่งรวม session ID) โดยอัตโนมัติ — user คนละคน (session คนละ session) จะได้ cache
แยกกันคนละชุด ไม่ปนกันอีกต่อไป

### 678.4 `@vary_on_headers`: ระบุ Header อื่นที่ไม่ใช่ Cookie

```python
from django.views.decorators.vary import vary_on_headers


@vary_on_headers('Accept-Language')
def post_detail(request, slug):
    ...
```

ใช้เมื่อเนื้อหาต่างกันตาม header อื่น เช่น ภาษาที่ browser ร้องขอ (`Accept-Language`)
หรือ API ที่ตอบสนองต่างกันตาม `Accept` header (ทวนจาก Part 048 ข้อ 472.4 เรื่อง
`AcceptHeaderVersioning` — ที่บอกไว้ว่าต้องตั้ง `Vary: Accept` ให้ถูก ไม่งั้นจะ cache
ผิดเวอร์ชันปน — นี่คือคำตอบเต็มของสิ่งที่ Part 048 เกริ่นไว้)

### 678.5 ผลกระทบสำคัญต่อขนาด Cache: ระวัง `Vary: Cookie` กับหน้าที่มี Traffic สูง

**คำเตือนสำคัญที่สุดของขั้อนี้**: `Cookie` header **แทบจะไม่ซ้ำกันเลยระหว่างผู้ใช้แต่
ละคน** (แต่ละคนมี session ID ของตัวเอง) การใส่ `Vary: Cookie` บนหน้าที่มี traffic สูง
มาก จึงหมายความว่า **แทบไม่มีการแชร์ cache ระหว่างผู้ใช้เลย** — cache ที่ควรจะประหยัด
ทรัพยากรกลับกลายเป็นการสร้าง cache entry แยกต่างหากสำหรับผู้ใช้แทบทุกคน (memory เต็ม
เร็วมาก และ hit rate ต่ำมาก เพราะแต่ละคนต้องคำนวณใหม่ครั้งแรกเสมออยู่ดี)

| สถานการณ์ | ทางเลือกที่เหมาะสม |
|---|---|
| หน้าที่เนื้อหาต่างกันแค่เล็กน้อยตาม user (เช่น "สวัสดี, ชื่อ") | ใช้ **Template Fragment Caching** (ขั้อ 673) cache เฉพาะส่วนที่**เหมือนกันทุกคน** แล้วปล่อยส่วนชื่อผู้ใช้ให้ render สดเสมอ — ดีกว่า `Vary: Cookie` มาก |
| หน้าที่เนื้อหาต่างกันทั้งหน้าจริง ๆ ตาม user (เช่น Dashboard ส่วนตัว) | ไม่ควร cache ทั้งหน้าเลย ใช้ low-level caching เฉพาะข้อมูลบางส่วนที่หนักจริง ๆ แทน |
| API ที่ต่างกันตาม `Accept-Language`/`Accept` (จำนวนค่าที่เป็นไปได้จำกัด เช่น 3-4 ภาษา) | `Vary` เหมาะสมดี เพราะจำนวน "รูปแบบ" ที่เป็นไปได้มีจำกัดและ cache แชร์กันได้จริงในกลุ่มเดียวกัน |

**หลักการสรุป**: `Vary` เหมาะกับ header ที่มี **ค่าที่เป็นไปได้จำกัด** (ภาษา, การ
บีบอัดข้อมูล, เวอร์ชัน API) ไม่เหมาะกับ header ที่ **แทบไม่ซ้ำกันเลยในทางปฏิบัติ**
อย่าง `Cookie`/session ID สำหรับกรณีหลัง ให้เลือก fragment caching หรือไม่ cache
ทั้งหน้าแทน

### 678.6 `patch_vary_headers`: เพิ่ม Vary Header ด้วยมือใน View ที่ไม่ใช้ Decorator

```python
from django.http import HttpResponse
from django.utils.cache import patch_vary_headers


def custom_view(request):
    response = HttpResponse('เนื้อหา')
    patch_vary_headers(response, ['Cookie', 'Accept-Language'])
    return response
```

ใช้ `patch_vary_headers()` เมื่อสร้าง response เองโดยตรง (ไม่ผ่าน decorator สำเร็จรูป)
— ฟังก์ชันนี้ฉลาดพอที่จะ**เพิ่มเข้าไปในรายการเดิม** (ถ้า response มี `Vary` header
อยู่แล้วจาก middleware อื่น) แทนที่จะเขียนทับ

### 678.7 ตารางสรุปขั้นตอนที่ 678

| ประเด็น | สรุป |
|---|---|
| `Vary` header คืออะไร | บอกว่า response ต่างกันไปตาม request header ที่ระบุ นอกเหนือจาก URL |
| `@vary_on_cookie` | แยก cache ตาม session/cookie — จำเป็นสำหรับหน้าที่เนื้อหาต่างกันตาม login state |
| `@vary_on_headers(...)` | แยก cache ตาม header ที่ระบุเอง (เช่น `Accept-Language`, `Accept`) |
| ลำดับ Decorator สำคัญ | `vary_on_*` ต้อง apply ก่อน `cache_page` เห็น header (เขียนไว้บนกว่าในโค้ด) |
| คำเตือนสำคัญที่สุด | อย่าใช้ `Vary: Cookie` กับหน้า traffic สูง — cache แทบไม่ถูกแชร์เลย ใช้ fragment caching แทน |

---

## ขั้นตอนที่ 679: ข้อควรระวังการ Cache QuerySet (QuerySet Lazy, การ Cache เฉพาะผลลัพธ์ที่ Evaluate แล้วเท่านั้น)

### 679.1 ทวนความจำ: QuerySet เป็น Lazy Object

ทวนหลักการพื้นฐานที่เรียนมาตั้งแต่ Phase 2: **QuerySet ไม่ได้ query ฐานข้อมูลทันทีที่
สร้างขึ้น** แต่จะ query จริงก็ต่อเมื่อถูก **evaluate** (เช่น วน `for`, เรียก `list()`,
`len()`, print, slice แบบไม่มี step) ก่อนหน้านั้น QuerySet เป็นแค่ "คำสั่งที่ยังไม่ถูก
รัน" ที่เก็บ SQL ที่จะสร้างไว้เฉย ๆ

```python
posts = Post.objects.filter(is_published=True)   # ยังไม่ query ฐานข้อมูล!
posts = posts.select_related('category')          # ยังไม่ query!
posts = posts.order_by('-created_at')              # ยังไม่ query!

for post in posts:   # ← ตอนนี้แหละที่ query จริงเกิดขึ้น (ครั้งแรกที่ iterate)
    print(post.title)
```

### 679.2 ปัญหา: `cache.set()` กับ QuerySet ที่ยังไม่ Evaluate

```python
# ❌ อันตราย — ดูเหมือนใช้ได้ แต่มีปัญหาซ่อนอยู่
from django.core.cache import cache
from .models import Post

def cache_published_posts():
    qs = Post.objects.filter(is_published=True).select_related('category')
    cache.set('blog:published_posts', qs, timeout=300)
```

โค้ดนี้**รันได้ไม่ error** เพราะ Django cache framework ใช้ `pickle` (module มาตรฐาน
ของ Python สำหรับ serialize object) ในการแปลงค่าให้เก็บลง backend ได้ และ **การ pickle
QuerySet จะบังคับให้มันถูก evaluate ทันที** (Django ออกแบบ `QuerySet.__reduce__` ไว้
เพื่อรองรับกรณีนี้โดยเฉพาะ) แต่ปัญหาที่แท้จริงคือ **ความไม่ชัดเจนของโค้ด** และผลข้าง
เคียงที่ตามมา:

| ปัญหา | รายละเอียด |
|---|---|
| Evaluate ทันทีโดยไม่ตั้งใจ | นักพัฒนาที่เห็นโค้ดนี้อาจไม่รู้ว่า `.set()` บรรทัดนี้แหละคือจุดที่ query ฐานข้อมูลเกิดขึ้นจริง (ซ่อนอยู่ภายใน pickle mechanism) — debug ยากเมื่อ query ช้าแต่หา breakpoint ไม่เจอว่า query เกิดตรงไหน |
| ค่าที่ได้จาก `cache.get()` "เสีย" ความ Lazy ไปแล้ว | หลัง `cache.get('blog:published_posts')` สิ่งที่ได้กลับมาคือ **ผลลัพธ์ที่ evaluate แล้ว** (เก็บ cached rows ไว้ใน `_result_cache` ของ QuerySet ตอน unpickle) เรียก `.filter()` เพิ่มต่อจากมันได้ แต่จะ**ไม่ query ฐานข้อมูลใหม่** ทั้งที่โค้ดดูเหมือนสร้าง query ใหม่ — สร้างความสับสนร้ายแรง |
| ขนาดข้อมูลใหญ่เกินจำเป็น | Pickle ทั้ง Model instance (ทุก field, ทุก related object ที่ select_related มา) กินพื้นที่ cache มากกว่าที่จำเป็นมาก ถ้าจริง ๆ ต้องการแค่บาง field |
| Cross-version Compatibility | Pickled model instance ผูกกับโครงสร้าง class ของ Model ณ เวลาที่ pickle ถ้า deploy โค้ดใหม่ที่เปลี่ยนโครงสร้าง Model (เพิ่ม/ลบ field) แล้วมี pickled instance เก่าค้างอยู่ใน cache อาจ unpickle ไม่สำเร็จหรือได้ object ที่ไม่สมบูรณ์ |

### 679.3 แนวทางที่ถูกต้อง: Evaluate อย่างชัดเจนก่อน Cache เสมอ

```python
# ✅ ถูก — evaluate ด้วย list() อย่างชัดเจน โค้ดสื่อความหมายตรงไปตรงมา
from django.core.cache import cache
from .models import Post


def get_published_posts():
    def _fetch():
        return list(
            Post.objects.filter(is_published=True).select_related('category')
        )   # list() บังคับ evaluate ตรงนี้อย่างชัดเจน อ่านโค้ดแล้วรู้ทันทีว่า query เกิดที่นี่

    return cache.get_or_set('blog:published_posts', _fetch, timeout=300)
```

การเรียก `list(queryset)` อย่างชัดเจนทำให้**ทุกคนที่อ่านโค้ดรู้ทันทีว่านี่คือจุดที่
query ฐานข้อมูลเกิดขึ้นจริง** ไม่ต้องเดาว่า pickle จะไปบังคับ evaluate เมื่อไหร่ —
หลักการนี้สอดคล้องกับปรัชญา "Explicit is better than implicit" ของ Python (The Zen
of Python) โดยตรง

### 679.4 แนวทางที่ดีกว่า: Cache เฉพาะ Field ที่จำเป็นด้วย `values()`/`values_list()`

ถ้าไม่จำเป็นต้องใช้ Model instance เต็มรูปแบบ (ไม่ต้องเรียก method, ไม่ต้องเข้าถึง
related object เพิ่มเติมนอกจากที่ระบุ) การ cache เป็น `dict`/`tuple` ธรรมดาผ่าน
`values()`/`values_list()` ประหยัดพื้นที่กว่ามากและ serialize/deserialize เร็วกว่า
pickled Model instance:

```python
def get_post_titles_for_sitemap():
    def _fetch():
        return list(
            Post.objects
            .filter(is_published=True)
            .values('slug', 'title', 'updated_at')   # ได้ list ของ dict ธรรมดา ไม่ใช่ Model instance
        )

    return cache.get_or_set('blog:sitemap_data', _fetch, timeout=3600)
```

| รูปแบบข้อมูล | ขนาดใน Cache | Serialize เร็วแค่ไหน | ใช้เมื่อไหร่ |
|---|---|---|---|
| `list(Model instances)` (full object) | ใหญ่ที่สุด | ช้าที่สุด | ต้องใช้ method ของ model หรือ properties เพิ่มเติมหลัง cache hit |
| `list(dict)` จาก `.values()` | ปานกลาง | เร็ว | ต้องการหลาย field พร้อม field name |
| `list(tuple)` จาก `.values_list()` | เล็กที่สุด | เร็วที่สุด | ต้องการแค่บาง field แบบเรียงลำดับตายตัว |
| ตัวเลข/string เดี่ยว (`.count()`, `.aggregate()`) | เล็กที่สุด | เร็วที่สุด | ต้องการแค่ค่าสรุปเดียว |

### 679.5 ข้อควรระวัง: Cache ค่า Aggregate ก็เสี่ยง Stale เหมือนกัน

```python
def get_total_published_count():
    return cache.get_or_set(
        'blog:post:total_published_count',
        lambda: Post.objects.filter(is_published=True).count(),
        timeout=300,
    )
```

`count()` คืนตัวเลขเดี่ยวแทน QuerySet เต็ม — ปลอดภัยจากปัญหาในขั้อ 679.2-679.3 อย่าง
สิ้นเชิง (ไม่มี Model instance ให้ pickle) แต่ยัง**ต้อง invalidate ให้ถูกต้อง**เหมือน
ข้อมูลอื่นทุกประการ (ตามที่ signal ในขั้อ 677 ทำไปแล้ว) — การเก็บเป็นตัวเลขล้วน ๆ
ไม่ได้แปลว่าไม่ต้องสนใจเรื่อง invalidation เลย มันแค่แก้ปัญหาเรื่อง "pickle ทำอะไรกับ
QuerySet" เท่านั้น ไม่ใช่แก้ปัญหา cache invalidation ทั้งหมด

### 679.6 ข้อควรระวัง: อย่า Cache QuerySet ที่มี Object เฉพาะ Request (เช่น `request.user`)

```python
# ❌ อันตราย — ผูก queryset กับ request.user object ที่เปลี่ยนทุก request
def get_my_posts(request):
    qs = list(Post.objects.filter(author=request.user))
    cache.set(f'blog:user:{request.user.pk}:posts', qs, timeout=300)
    return qs
```

โค้ดนี้ดู "ถูก" เพราะแยก key ตาม `request.user.pk` แล้ว (ทำถูกตามหลักการ cache key
ในขั้อ 675) แต่ปัญหาคือ**แต่ละ `Post` instance ที่ได้จะพก reference กลับไปยัง
`author` object เดิม** — ถ้าโค้ดส่วนอื่นแก้ไขข้อมูล `User` (เช่น เปลี่ยน username)
หลัง cache ถูก set ไปแล้ว การเรียก `post.author.username` จาก cache ที่ไม่ evaluate
`select_related('author')` ไว้ล่วงหน้าจะ**เรียก query ใหม่ทุกครั้งอยู่ดี** (เพราะ
foreign key ที่ไม่ได้ select_related ไว้จะ lazy-load ใหม่เสมอ ไม่ได้ถูก cache ไปด้วย)
ทำให้เข้าใจผิดว่า "cache แล้วทำไมยัง query อยู่" — ทางแก้คือ**ระบุ field ที่ต้องการใช้
จริงให้ชัดเจนด้วย `values()`** แทนการ cache Model instance เต็มรูปแบบเสมอเมื่อทำได้

### 679.7 ตารางสรุปขั้นตอนที่ 679

| ข้อควรระวัง | ทางแก้ |
|---|---|
| Cache QuerySet ที่ยังไม่ evaluate ทำให้ debug ยาก | เรียก `list()`/`values()`/`values_list()` อย่างชัดเจนก่อน `.set()` เสมอ |
| Cache Model instance เต็มรูปแบบกิน memory เกินจำเป็น | ใช้ `.values()`/`.values_list()` เมื่อไม่ต้องใช้ method/property ของ model |
| ค่า Aggregate ก็ Stale ได้เหมือนกัน | ยังต้อง invalidate ผ่าน signal เหมือนข้อมูลอื่นทุกประการ |
| Cache queryset ที่พึ่งพา related object ที่ไม่ได้ `select_related` | ใช้ `values()` ระบุ field ที่ต้องการจริงแทน หรือ `select_related`/`prefetch_related` ให้ครบก่อน evaluate |
| หลักการทองคำ | **Cache เฉพาะข้อมูลที่ evaluate แล้วและเป็น plain Python data type (list/dict/tuple/number/string) ให้มากที่สุดเท่าที่ทำได้เสมอ** |

---

## ขั้นตอนที่ 680: สรุปและแบบฝึกหัด — Implement Caching เต็มรูปแบบสำหรับหน้า Blog List/Detail พร้อม Invalidation ที่ถูกต้อง

### 680.1 ประกอบทุกอย่างเข้าด้วยกัน: โครงสร้างไฟล์สุดท้ายของระบบ Caching

```
blog/
├── models.py              # Category, Post, Comment (ไม่เปลี่ยนจาก Part 030)
├── cache_keys.py           # รวมสูตรสร้าง cache key ทั้งหมด (ขั้อ 675.2, 675.5)
├── signals.py               # Invalidation logic ผูกกับ post_save/post_delete (ขั้อ 677)
├── apps.py                  # เชื่อมต่อ signals.py ผ่าน ready() (ขั้อ 677.5)
├── views.py                 # PostListView, PostDetailView พร้อม caching ครบ (ด้านล่าง)
├── context_processors.py    # categories_for_sidebar (ทวนจาก Part 030)
├── templates/
│   ├── partials/sidebar.html          # {% cache %} fragment (ขั้อ 673)
│   └── blog/
│       ├── post_list.html
│       └── post_detail.html
└── tests/
    ├── test_cache_invalidation.py     # ขั้อ 677.6
    └── test_caching_views.py           # เทสต์ใหม่ในขั้อนี้
```

### 680.2 `cache_keys.py` ฉบับสมบูรณ์

```python
# blog/cache_keys.py
from django.core.cache import cache

POSTS_GENERATION_KEY = 'blog:posts:generation'
TOTAL_PUBLISHED_COUNT_KEY = 'blog:post:total_published_count'


def get_posts_generation():
    return cache.get_or_set(POSTS_GENERATION_KEY, 1, timeout=None)


def bump_posts_generation():
    cache.add(POSTS_GENERATION_KEY, 1, timeout=None)
    return cache.incr(POSTS_GENERATION_KEY)


def post_list_page_key(page_number, category_slug=None):
    generation = get_posts_generation()
    if category_slug:
        return f'blog:post_list:gen:{generation}:category:{category_slug}:page:{page_number}'
    return f'blog:post_list:gen:{generation}:all:page:{page_number}'


def post_detail_view_count_key(post_id):
    return f'blog:post:{post_id}:view_count'
```

### 680.3 `views.py` ฉบับสมบูรณ์: List View + Detail View พร้อม Caching ครบทุกชั้น

```python
# blog/views.py
from django.core.cache import cache
from django.core.paginator import Paginator
from django.shortcuts import get_object_or_404, render
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie
from django.views.generic import DetailView

from .cache_keys import (
    TOTAL_PUBLISHED_COUNT_KEY,
    post_detail_view_count_key,
    post_list_page_key,
)
from .models import Post


def post_list(request):
    """
    List view ใช้ generation-based caching แบบ low-level แทน @cache_page ตรง ๆ
    เพื่อให้ invalidate ได้แม่นยำทันทีเมื่อ Post เปลี่ยน (ผ่าน bump_posts_generation
    ในขั้อ 677) โดยไม่ต้องรอ timeout หมดอายุเอง — เหมาะกับหน้าที่ต้องถูกต้องทันที
    """
    page_number = request.GET.get('page', '1')
    category_slug = request.GET.get('category')
    cache_key = post_list_page_key(page_number, category_slug)

    cached_html_context = cache.get(cache_key)
    if cached_html_context is None:
        post_qs = (
            Post.objects
            .filter(is_published=True)
            .select_related('category', 'author')
        )
        if category_slug:
            post_qs = post_qs.filter(category__slug=category_slug)

        paginator = Paginator(post_qs, 10)
        page_obj = paginator.get_page(page_number)

        cached_html_context = {
            'posts': list(page_obj.object_list),
            'has_next': page_obj.has_next(),
            'has_previous': page_obj.has_previous(),
            'number': page_obj.number,
            'num_pages': paginator.num_pages,
        }
        cache.set(cache_key, cached_html_context, timeout=600)   # timeout ยาวได้ เพราะ invalidate ทันทีอยู่แล้ว

    total_published = cache.get_or_set(
        TOTAL_PUBLISHED_COUNT_KEY,
        lambda: Post.objects.filter(is_published=True).count(),
        timeout=600,
    )

    return render(request, 'blog/post_list.html', {
        'page_data': cached_html_context,
        'total_published': total_published,
    })


@method_decorator(vary_on_cookie, name='dispatch')
@method_decorator(cache_page(60 * 2), name='dispatch')   # timeout สั้น (2 นาที) เป็นชั้นป้องกันสำรอง
class PostDetailView(DetailView):
    """
    Detail view ใช้ @cache_page ร่วมกับ signal invalidation (ขั้อ 677.4) — timeout สั้น
    ทำหน้าที่เป็น safety net เผื่อ signal พลาดจุดใดจุดหนึ่ง (defense in depth)
    """
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'

    def get_queryset(self):
        return Post.objects.filter(is_published=True).select_related('category', 'author')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        view_count_key = post_detail_view_count_key(self.object.pk)
        cache.add(view_count_key, 0, timeout=None)
        context['view_count'] = cache.incr(view_count_key)
        return context
```

**สังเกตการออกแบบสำคัญ**: หน้า list ใช้ **low-level API + generation-based key**
เพราะต้องการ invalidate แม่นยำทันที ส่วนหน้า detail ใช้ **`@cache_page` timeout สั้น
ร่วมกับ signal** — เป็น **defense in depth** (ป้องกันซ้อนสองชั้น): ถ้า signal ทำงาน
ถูกต้อง ผู้ใช้เห็นข้อมูลใหม่ทันที ถ้า signal พลาดจุดใดจุดหนึ่งโดยไม่ได้ตั้งใจ (เช่น
มีคน bulk update ผ่าน `Post.objects.filter(...).update()` ซึ่ง**ไม่ trigger signal**
เพราะข้าม `.save()` — ทวนข้อจำกัดนี้จาก Part 019) อย่างเลวร้ายที่สุดผู้ใช้จะเห็นข้อมูล
เก่าไม่เกิน 2 นาทีเท่านั้น ไม่ใช่ตลอดไป

### 680.4 เทสต์ครบวงจร

```python
# blog/tests/test_caching_views.py
from django.contrib.auth import get_user_model
from django.core.cache import cache
from django.test import TestCase
from django.urls import reverse

from blog.models import Category, Post

User = get_user_model()


class BlogCachingIntegrationTests(TestCase):
    def setUp(self):
        cache.clear()
        self.user = User.objects.create_user(username='writer', password='x')
        self.category = Category.objects.create(name='Django')
        self.post = Post.objects.create(
            title='บทความทดสอบ Cache', content='เนื้อหา', category=self.category,
            author=self.user, is_published=True,
        )

    def tearDown(self):
        cache.clear()

    def test_post_list_reflects_new_post_immediately_after_creation(self):
        response = self.client.get(reverse('blog:list'))
        self.assertContains(response, 'บทความทดสอบ Cache')

        Post.objects.create(
            title='บทความที่สองมาใหม่', content='...', category=self.category,
            author=self.user, is_published=True,
        )

        # เรียกซ้ำทันที — ต้องเห็นบทความใหม่แล้ว เพราะ generation ถูก bump โดย signal
        response = self.client.get(reverse('blog:list'))
        self.assertContains(response, 'บทความที่สองมาใหม่')

    def test_post_detail_view_count_increments(self):
        url = self.post.get_absolute_url()
        self.client.get(url)
        self.client.get(url)

        from blog.cache_keys import post_detail_view_count_key
        count = cache.get(post_detail_view_count_key(self.post.pk))
        self.assertEqual(count, 2)
```

### 680.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจสถาปัตยกรรม 4 ชั้นของ Django Caching Framework และ Cache Backend ทั้ง 5 แบบ
  ที่ Django รองรับในตัว พร้อมเหตุผลว่าทำไม Redis คือมาตรฐานอุตสาหกรรมสำหรับ production
- ✅ ใช้ `@cache_page` cache ทั้ง HTTP response ระดับ view ได้ทั้ง FBV และ CBV พร้อมรู้
  ข้อจำกัดเรื่อง CSRF token และเนื้อหาเฉพาะ user
- ✅ ใช้ `{% cache %}` template tag ทำ fragment caching ตามที่ Part 028 และ Part 030
  ค้างสัญญาไว้ พร้อมเข้าใจ `vary_on` และวิธีคำนวณ cache key ด้วย
  `make_template_fragment_key`
- ✅ ใช้ Low-level Cache API ครบทุกเมธอด (`get`, `set`, `add`, `get_or_set`, `delete`,
  `delete_many`, `get_many`, `set_many`, `incr`, `decr`, `touch`, `clear`)
- ✅ ออกแบบ cache key ที่ดีด้วยหลัก namespace และเข้าใจ `KEY_PREFIX`/`VERSION` รวมถึง
  เทคนิค generation-based caching สำหรับ bulk invalidation
- ✅ เข้าใจว่าทำไม cache invalidation ถึงเป็นปัญหาที่ยาก และกลยุทธ์ 3 แบบหลัก (TTL,
  explicit, versioning) พร้อมปัญหา cache stampede และทางแก้เบื้องต้น
- ✅ Invalidate cache อัตโนมัติผ่าน Django Signals เมื่อ `Post`/`Category` เปลี่ยน
  รวมถึงเทคนิคขั้นสูงในการลบ cache ของ `@cache_page` ด้วย `get_cache_key()`
- ✅ ใช้ `Vary` header (`@vary_on_cookie`, `@vary_on_headers`) อย่างถูกต้อง และรู้
  ขีดจำกัดว่าเมื่อไหร่ไม่ควรใช้กับ header ที่ไม่ซ้ำกันเลยอย่าง `Cookie`
- ✅ หลีกเลี่ยงข้อผิดพลาดคลาสสิกของการ cache QuerySet ที่ยังไม่ evaluate และเลือกใช้
  `values()`/`values_list()` เมื่อเหมาะสม
- ✅ Implement ระบบ caching เต็มรูปแบบสำหรับหน้า blog list/detail ที่ผสาน generation-
  based caching, `@cache_page`, signal invalidation, และ defense-in-depth เข้าด้วยกัน

### 680.6 Checklist ก่อนไป Part ถัดไป

- [ ] ตั้งค่า `CACHES['default']` เป็น `LocMemCache` อย่างชัดเจนใน `settings.py`
- [ ] เขียน `@cache_page` ครอบทั้ง FBV และ CBV (ด้วย `method_decorator`) ได้สำเร็จ
- [ ] เขียน `{% cache %}` fragment caching ใน `sidebar.html` พร้อม `vary_on` ที่ถูกต้อง
- [ ] ใช้ `cache.get_or_set()` แทนการเรียกฟังก์ชันคำนวณค่าตรง ๆ เสมอ (ไม่เรียก callable
      ก่อนส่งเข้า `get_or_set`)
- [ ] สร้างไฟล์ `cache_keys.py` รวมสูตรสร้าง cache key ทั้งหมดของแอป
- [ ] เขียน `signals.py` ที่ invalidate cache ทุกจุดเมื่อ `Post`/`Category` เปลี่ยน และ
      เชื่อมต่อผ่าน `AppConfig.ready()` แล้ว
- [ ] ทดสอบด้วยตนเองว่าแก้ไขโพสต์แล้วหน้ารายการ/รายละเอียดแสดงข้อมูลใหม่ทันทีโดยไม่ต้อง
      รอ timeout
- [ ] เขียนเทสต์ที่เรียก `cache.clear()` ใน `setUp()`/`tearDown()` ป้องกัน state รั่ว
      ระหว่างเทสต์
- [ ] เข้าใจว่าทำไมยังไม่ใช้ Redis ใน Part นี้ และรู้ว่า Part 069 จะเจาะลึกอะไรต่อ

### 680.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม caching ให้กับหน้า `CategoryDetailView` (แสดงบทความทั้งหมด
ในหมวดหมู่ที่ระบุ) โดยใช้ **template fragment caching** ครอบส่วนแสดงรายการบทความ
พร้อมใส่ `category.slug` เป็น `vary_on` ให้ถูกต้อง แล้วเขียน signal เพิ่มเติมเพื่อ
invalidate fragment นี้เมื่อมีการเพิ่ม/ลบ/แก้ไขบทความในหมวดหมู่นั้น

**แบบฝึกหัดที่ 2**: เขียนฟังก์ชัน `get_popular_posts(limit=5)` ที่ cache รายการบทความ
ที่มี `view_count` (จาก `post_detail_view_count_key`) สูงสุด 5 อันดับแรก โดยใช้เทคนิค
**jittered expiration** จากขั้อ 676.6 (timeout พื้นฐาน 10 นาที บวกลบแบบสุ่ม 60 วินาที)
เพื่อป้องกัน cache stampede เมื่อหน้านี้มี traffic สูง

**แบบฝึกหัดที่ 3**: จำลองสถานการณ์ **cache stampede** ด้วยตัวเอง — เขียนฟังก์ชันที่
`time.sleep(1)` จำลอง query หนัก แล้วใช้ `ThreadPoolExecutor` ยิง 20 thread เรียก
ฟังก์ชันนี้พร้อมกันตอน cache เพิ่งหมดอายุ นับจำนวนครั้งที่ query จริงถูกเรียก
(a) แบบไม่มีการป้องกัน กับ (b) แบบใช้ `cache.add()` เป็น lock ตามขั้อ 676.7 แล้วเปรียบ
เทียบผลลัพธ์ทั้งสองแบบ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน management command ชื่อ `warm_cache` ที่เรียก
`get_or_set` ของทุก cache key สำคัญในระบบ (sidebar categories, total published count,
popular posts) ทันทีหลัง deploy เสร็จ (เรียกว่า **cache warming**) เพื่อไม่ให้ผู้ใช้
คนแรกหลัง deploy ต้องรอ cache miss ที่ query หนักที่สุด แล้วทดสอบด้วยการรัน
`python manage.py warm_cache` แล้วตรวจสอบด้วย `cache.get()` ว่าทุก key มีค่าจริง

### 680.8 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่ใช้ `django-redis` ตั้งแต่ Part นี้เลย ในเมื่อ Part 048 ก็แนะนำไปแล้ว?**
A: Part 048 แนะนำ `django-redis` ในบริบทเฉพาะของ throttling ที่ต้องการ shared state
ข้าม worker process แต่ Part นี้ต้องการโฟกัสที่ **หลักการของ caching framework เอง**
โดยไม่ให้การตั้งค่า infrastructure (ติดตั้ง Redis server, connection pool, ฯลฯ) มา
บดบังหลักการ เมื่อเข้าใจหลักการแน่นแล้ว การเปลี่ยนไปใช้ Redis ใน Part 069 จะเป็นแค่
การเปลี่ยน `settings.CACHES` บรรทัดเดียว โค้ดที่เหลือทั้งหมดใน Part นี้ (cache key,
signal invalidation, fragment caching) ใช้ได้ทันทีโดยไม่ต้องแก้อะไรเลย

**Q: ควร cache ทุกอย่างที่ query ฐานข้อมูลเลยไหม เพื่อความเร็วสูงสุด?**
A: ไม่ควร — caching เพิ่มความซับซ้อนเสมอ (ต้องคิดเรื่อง invalidation ตามขั้อ 676) ควร
cache เฉพาะข้อมูลที่ **(1) query หนักจริง ๆ และ (2) ถูกอ่านบ่อยกว่าที่เปลี่ยนแปลงมาก**
ทวนหลักการจาก Part 028 ข้อ 278.4: **แก้ที่ query ให้ optimize ที่สุดก่อนเสมอ**
(select_related/prefetch_related/index จาก Part 067/070) caching คือชั้นป้องกันเพิ่ม
เติมสำหรับกรณีที่ optimize สุดแล้วแต่ยังหนักอยู่ดีเท่านั้น

**Q: `LocMemCache` ใช้ใน production ได้ไหมถ้าโปรเจกต์เล็กมากมี worker เดียว?**
A: ในทางทฤษฎีใช้ได้ถ้ารันด้วย **1 worker process เดียวจริง ๆ** ตลอดไป แต่ในทางปฏิบัติ
ไม่แนะนำเลย เพราะ (1) โปรเจกต์มักโตขึ้นและต้องเพิ่ม worker ในอนาคตอย่างแน่นอน (2) cache
หายทุกครั้งที่ deploy ใหม่ (process restart) ทำให้ผู้ใช้คนแรกหลัง deploy ทุกครั้งต้อง
เจอ cache miss พร้อมกันหมด (คล้ายปัญหา cache stampede) และ (3) ไม่มีทาง monitor หรือ
debug cache จากภายนอก process ได้เลย — ใช้ Redis ตั้งแต่ต้นเสมอสำหรับ production จริง
แม้โปรเจกต์จะเล็กก็ตาม

**Q: ถ้ามีคนใช้ `Post.objects.filter(...).update(...)` (bulk update) แทน `.save()`
จะเกิดอะไรขึ้นกับ cache?**
A: **Signal จะไม่ถูกเรียกเลย** เพราะ `QuerySet.update()` ส่ง SQL `UPDATE` ตรงไปยัง
ฐานข้อมูลโดยไม่ผ่าน `Model.save()` ของแต่ละ instance (ทวนข้อจำกัดนี้จาก Part 019) —
cache ที่เกี่ยวข้องจะไม่ถูก invalidate และกลายเป็น stale ทันที นี่คือเหตุผลที่การออกแบบ
ในขั้อ 680.3 ใส่ `@cache_page` timeout สั้นเป็น **safety net** สำรองไว้เสมอ (defense
in depth) แทนที่จะพึ่งพา signal เพียงอย่างเดียว 100% — ถ้าทีมของคุณใช้ bulk update
บ่อย ให้พิจารณาเรียก `bump_posts_generation()` ด้วยมือทันทีหลังทุกจุดที่เรียก
`.update()` เพิ่มเติม

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 069: Redis Caching ขั้นสูง** จะนำทุกหลักการที่เรียนใน Part นี้ไปใช้กับ **Redis
จริง** ตั้งแต่การติดตั้งและตั้งค่า `django-redis` แบบเต็มรูปแบบ, connection pooling,
persistence (RDB vs AOF), การใช้ data structure ขั้นสูงของ Redis (sorted set สำหรับ
"บทความยอดนิยม" แบบ real-time, hash สำหรับเก็บ object แบบ field-level), การทำ
**distributed lock** ที่ปลอดภัยจริงสำหรับป้องกัน cache stampede (ทวนจากขั้อ 676.7 ที่
เกริ่นไว้ด้วย `LocMemCache`), และการ monitor Redis ด้วยคำสั่ง `redis-cli` พร้อมทั้ง
เชื่อมโยงกลับไปแก้ปัญหาที่ Part 048 ข้อ 478 ทิ้งไว้เรื่อง throttling ที่ต้องการ shared
state ข้าม worker process อย่างแท้จริง — เตรียมติดตั้ง Redis server ไว้ในเครื่องของคุณ
ให้พร้อมก่อนเริ่ม Part ถัดไป (ผ่าน Docker จะสะดวกที่สุด ซึ่ง Part 069 จะแนะนำขั้นตอนให้)
