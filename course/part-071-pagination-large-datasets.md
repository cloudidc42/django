# Part 071: Pagination และ Large Dataset Handling

> **ขั้นตอนที่ 701-710 ของหลักสูตร** | Phase 8: Performance, Scaling และ Optimization
>
> Part 047 สอนการทำ pagination ให้ **REST API** ผ่าน DRF (`PageNumberPagination`,
> `LimitOffsetPagination`, `CursorPagination`) และ Part 053 แนะนำ Infinite Scroll
> ด้วย HTMX แบบผิวเผินโดยใช้ `Paginator` ธรรมดา — Part นี้จะพาคุณกลับมาที่ฝั่ง
> **server-rendered view** (ไม่ใช่ DRF) แล้วเจาะลึกทุกอย่างที่ Part 047 พูดถึงแค่
> ในเชิงทฤษฎี: ทบทวน `Paginator`/`Page` object ของ Django เอง, พิสูจน์ปัญหาของ
> Offset Pagination ด้วย benchmark จริงบนข้อมูลนับล้านแถว, เขียน **Keyset/Cursor
> Pagination เองตั้งแต่ต้น** โดยไม่พึ่ง DRF, ประกอบ Infinite Scroll แบบสมบูรณ์กับ
> HTMX, และรับมือกับ "ข้อมูลออกจากระบบ" ในสเกลใหญ่ — ทั้ง Streaming CSV/Excel
> export, `StreamingHttpResponse` สำหรับไฟล์ขนาดใหญ่, การประมวลผล QuerySet เป็น
> chunk ด้วย `iterator()`, พร้อมเกริ่นสองเรื่องที่จะเจาะลึกเต็มรูปแบบใน Phase
> ถัดไป — Background Job (Celery, Phase 9) และ Database Partitioning (Phase 12)
> เมื่อจบ Part นี้ คุณจะมีระบบแสดงรายการ `Post` ที่รองรับข้อมูลหลักล้านแถวได้
> อย่างมืออาชีพ ทั้งหน้าเว็บและระบบ export

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 701: ทบทวน Django `Paginator`/`Page` object สำหรับ server-rendered view
- ขั้นตอนที่ 702: ข้อจำกัดของ Offset Pagination เมื่อข้อมูลเยอะมาก — เจาะลึกพร้อม benchmark จริง
- ขั้นตอนที่ 703: Implement Keyset/Cursor Pagination เองตั้งแต่ต้น (ไม่พึ่ง DRF)
- ขั้นตอนที่ 704: รูปแบบ Infinite Scroll ที่สมบูรณ์ (เชื่อมกับ HTMX จาก Part 053)
- ขั้นตอนที่ 705: จัดการการ Export ข้อมูลจำนวนมาก — Streaming CSV/Excel
- ขั้นตอนที่ 706: `StreamingHttpResponse` สำหรับไฟล์ดาวน์โหลดขนาดใหญ่
- ขั้นตอนที่ 707: การประมวลผล QuerySet ขนาดใหญ่แบบ Chunk — `iterator()`, `chunk_size`
- ขั้นตอนที่ 708: เกริ่นการประมวลผลงานหนักแบบ Background (เจาะลึกเต็มด้วย Celery ใน Phase 9)
- ขั้นตอนที่ 709: เกริ่นแนวคิด Database Partitioning สำหรับข้อมูลระดับ Enterprise (เจาะลึกเต็มใน Phase 12)
- ขั้นตอนที่ 710: สรุปและแบบฝึกหัด — Cursor Pagination และ Streaming Export สำหรับ `Post`

---

## ขั้นตอนที่ 701: ทบทวน Django `Paginator`/`Page` object สำหรับ server-rendered view

### 701.1 ความแตกต่างระหว่าง Pagination ของ DRF กับ Pagination ของ Django เอง

Part 047 สอน `PageNumberPagination` ของ **Django REST Framework** ซึ่งทำงานเฉพาะกับ
`APIView`/`ViewSet` ที่คืนค่าเป็น JSON เท่านั้น แต่ในความเป็นจริง **Django core**
มีระบบ pagination ของตัวเองอยู่แล้วตั้งแต่ก่อนมี DRF เสียอีก คือคลาส
`django.core.paginator.Paginator` — และ **DRF ก็ใช้คลาสนี้เป็นฐานเบื้องหลัง**
`PageNumberPagination` ทั้งหมด (สังเกตได้จาก `self.page.paginator.count` ที่เจอใน
Part 047 ข้อ 468.4 นั่นคือ Django `Paginator`/`Page` object ตัวเดียวกันนี้เป๊ะ)

Part นี้จะใช้ `Paginator` ตรง ๆ กับ **function-based view / class-based view ที่คืน
HTML** (ไม่ใช่ JSON) ซึ่งเป็นรูปแบบที่พบบ่อยที่สุดในเว็บแอปพลิเคชันทั่วไปที่ไม่ใช่
Single Page Application

### 701.2 โมเดลตั้งต้นสำหรับ Part นี้: ขยาย `Post` ให้มีข้อมูลจำลองจำนวนมาก

```python
# blog/models.py (Post มีอยู่แล้วตั้งแต่ Part 012 — ตรวจสอบว่ามี index ตามนี้)
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    is_published = models.BooleanField(default=True, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    view_count = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at', 'id'], name='post_created_id_idx'),
        ]

    def __str__(self):
        return self.title
```

สังเกตว่า index ที่เพิ่มเข้ามาคือ **composite index** บน `(-created_at, id)` ไม่ใช่
แค่ `created_at` เฉย ๆ — เหตุผลจะชัดเจนขึ้นในขั้นตอนที่ 703 เมื่อเราต้องใช้ `id`
เป็น tie-breaker สำหรับ Keyset Pagination

### 701.3 สร้างข้อมูลจำลองจำนวนมากด้วย Management Command

เพื่อให้ benchmark ในขั้นตอนที่ 702 เห็นผลจริง ต้องมีข้อมูลระดับ **แสนถึงล้านแถว**
ไม่ใช่แค่หลักสิบแถวที่ทดลองด้วยมือ:

```python
# blog/management/commands/seed_posts.py
import random
from django.core.management.base import BaseCommand
from django.db import connection
from django.utils import timezone
from django.utils.text import slugify
from blog.models import Post


class Command(BaseCommand):
    help = 'สร้างข้อมูล Post จำลองจำนวนมากสำหรับทดสอบ pagination บนข้อมูลขนาดใหญ่'

    def add_arguments(self, parser):
        parser.add_argument('count', type=int, help='จำนวน Post ที่ต้องการสร้าง')
        parser.add_argument('--batch-size', type=int, default=5000)

    def handle(self, *args, **options):
        total = options['count']
        batch_size = options['batch_size']
        created = 0

        self.stdout.write(f'กำลังสร้าง {total:,} Post ...')

        while created < total:
            batch = min(batch_size, total - created)
            posts = [
                Post(
                    title=f'บทความทดสอบเลขที่ {created + i}',
                    slug=f'test-post-{created + i}',
                    content='เนื้อหาตัวอย่างสำหรับทดสอบ pagination ' * 20,
                    is_published=random.random() > 0.1,
                    view_count=random.randint(0, 10_000),
                )
                for i in range(batch)
            ]
            # bulk_create ข้าม save() ปกติ (ไม่รัน auto_now_add ผ่าน Python loop
            # ทีละแถว) ทำให้เร็วกว่า .save() วนลูปหลายสิบเท่าบนข้อมูลระดับแสน-ล้านแถว
            Post.objects.bulk_create(posts, batch_size=batch_size)
            created += batch
            self.stdout.write(f'  สร้างแล้ว {created:,}/{total:,}')

        self.stdout.write(self.style.SUCCESS(f'เสร็จสิ้น! สร้าง Post ทั้งหมด {created:,} แถว'))
```

```bash
# สร้างข้อมูลจำลอง 1 ล้านแถว (ใช้เวลาสักครู่ ขึ้นกับสเปกเครื่อง)
python manage.py seed_posts 1000000
```

**คำเตือนสำคัญ**: `bulk_create` ไม่เรียก `save()`, ไม่ยิง signal `pre_save`/
`post_save`, และไม่รัน `auto_now_add` ผ่าน Python (Django จัดการค่าฟิลด์เหล่านี้
ให้ตอนสร้าง object ก่อนส่งเข้า `bulk_create` อยู่แล้ว) — เหมาะสำหรับ seed ข้อมูล
ทดสอบเท่านั้น ไม่ควรใช้แทน `.save()` ปกติในโค้ด business logic จริงที่ต้องพึ่ง signal

### 701.4 `Paginator` พื้นฐาน: สร้างจาก QuerySet ตรง ๆ

```python
# blog/views.py
from django.core.paginator import Paginator
from django.shortcuts import render
from .models import Post


def post_list_view(request):
    post_list = Post.objects.filter(is_published=True)  # ordering มาจาก Meta.ordering แล้ว

    paginator = Paginator(post_list, 20)          # 20 รายการต่อหน้า
    page_number = request.GET.get('page', 1)
    page_obj = paginator.get_page(page_number)     # ปลอดภัยจาก error แม้ page_number ผิดพลาด

    return render(request, 'blog/post_list.html', {'page_obj': page_obj})
```

**จุดสำคัญ**: `Paginator` รับ **QuerySet** เข้ามาโดยตรง (ไม่ใช่ list ที่ evaluate
แล้ว) และฉลาดพอที่จะ**ไม่** ดึงข้อมูลทั้งหมดมาไว้ใน memory ก่อน — มันจะสร้าง SQL
`LIMIT`/`OFFSET` ที่ตรงกับหน้าที่ขอเท่านั้นตอนที่ `Page` object ถูก evaluate จริง ๆ
(เช่นตอน iterate ใน template) ยกเว้น `paginator.count` ที่ต้องรัน `SELECT COUNT(*)`
แยกต่างหากเสมอ (ประเด็นนี้สำคัญมากในขั้นตอนที่ 702)

### 701.5 `Paginator.get_page()` vs `Paginator.page()`

| Method | พฤติกรรมเมื่อค่าที่รับมาผิดพลาด |
|---|---|
| `paginator.page(number)` | โยน `PageNotAnInteger` หรือ `EmptyPage` exception ตรง ๆ — ต้อง `try/except` เอง |
| `paginator.get_page(number)` | **จัดการ exception ให้อัตโนมัติ**: ถ้าไม่ใช่ตัวเลขจะกลับไปหน้า 1 ถ้าเลขหน้าเกินหน้าสุดท้ายจะกลับไปหน้าสุดท้ายแทน ไม่ error เลย |

```python
# วิธีเดิมที่ต้องเขียน try/except เอง (ก่อนจะรู้จัก get_page)
from django.core.paginator import EmptyPage, PageNotAnInteger

def post_list_view_manual(request):
    post_list = Post.objects.filter(is_published=True)
    paginator = Paginator(post_list, 20)
    page_number = request.GET.get('page')
    try:
        page_obj = paginator.page(page_number)
    except PageNotAnInteger:
        page_obj = paginator.page(1)
    except EmptyPage:
        page_obj = paginator.page(paginator.num_pages)
    return render(request, 'blog/post_list.html', {'page_obj': page_obj})
```

**คำแนะนำระดับมืออาชีพ**: ใช้ `get_page()` เสมอในโค้ด production เพราะครอบคลุม
edge case ทั้งหมดในบรรทัดเดียว ใช้ `page()` เฉพาะเมื่อต้องการ error handling
ที่ต่างจากพฤติกรรมมาตรฐาน (เช่น อยากคืน 404 แทนที่จะ fallback ไปหน้าสุดท้าย)

### 701.6 คุณสมบัติของ `Page` object ที่ใช้บ่อยที่สุดใน Template

```html
<!-- blog/templates/blog/post_list.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด</h1>

<ul>
    {% for post in page_obj %}
        <li>{{ post.title }} ({{ post.view_count }} ครั้ง)</li>
    {% endfor %}
</ul>

<nav class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1">« หน้าแรก</a>
        <a href="?page={{ page_obj.previous_page_number }}">ก่อนหน้า</a>
    {% endif %}

    <span>หน้า {{ page_obj.number }} จาก {{ page_obj.paginator.num_pages }}
        (ทั้งหมด {{ page_obj.paginator.count }} รายการ)</span>

    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">ถัดไป</a>
        <a href="?page={{ page_obj.paginator.num_pages }}">หน้าสุดท้าย »</a>
    {% endif %}
</nav>
{% endblock %}
```

| Attribute/Method | ความหมาย |
|---|---|
| `page_obj.object_list` | รายการข้อมูลของหน้านี้ (iterate ตรง ๆ ผ่าน `page_obj` ได้เลยโดยไม่ต้องพิมพ์ `.object_list`) |
| `page_obj.number` | เลขหน้าปัจจุบัน |
| `page_obj.has_next()` / `has_previous()` | มีหน้าถัดไป/ก่อนหน้าหรือไม่ |
| `page_obj.next_page_number()` / `previous_page_number()` | เลขหน้าถัดไป/ก่อนหน้า (โยน error ถ้าไม่มี — ต้องเช็ค `has_next`/`has_previous` ก่อนเสมอ) |
| `page_obj.paginator.count` | จำนวนแถวทั้งหมดในทุกหน้ารวมกัน (มาจาก `SELECT COUNT(*)`) |
| `page_obj.paginator.num_pages` | จำนวนหน้าทั้งหมด |
| `page_obj.paginator.page_range` | `range` object ของเลขหน้าทั้งหมด (1 ถึง `num_pages`) — ใช้ทำเลขหน้าแบบ 1,2,3,...,N |

### 701.7 `Paginator.get_elided_page_range()`: เลขหน้าแบบ "1 2 3 ... 48 49 50" (Django 4.0+)

เมื่อมีหลักพันหน้า การแสดงเลขหน้าทั้งหมด (`page_range`) ในหน้าเว็บเป็นเรื่องไร้
สาระและทำให้ HTML บวมมาก Django มี method สำเร็จรูปสำหรับทำ "elided page range"
(เลขหน้าที่มี `...` คั่นตรงกลาง) ให้แล้ว:

```html
<nav class="pagination">
    {% for num in page_obj.paginator.get_elided_page_range(page_obj.number) %}
        {% if num == page_obj.paginator.ELLIPSIS %}
            <span class="ellipsis">…</span>
        {% elif num == page_obj.number %}
            <span class="current">{{ num }}</span>
        {% else %}
            <a href="?page={{ num }}">{{ num }}</a>
        {% endif %}
    {% endfor %}
</nav>
```

ผลลัพธ์ตัวอย่างเมื่ออยู่หน้า 42 จากทั้งหมด 500 หน้า: `1 … 40 41 [42] 43 44 … 500`
— ไม่ต้องเขียน logic คำนวณช่วงเลขหน้าเองเลยแม้แต่บรรทัดเดียว

---

## ขั้นตอนที่ 702: ข้อจำกัดของ Offset Pagination เมื่อข้อมูลเยอะมาก — เจาะลึกพร้อม benchmark จริง

### 702.1 ทบทวนจาก Part 047: ทำไม `OFFSET` ถึงช้าลงเมื่อหน้าลึกขึ้น

Part 047 ข้อ 469.1 อธิบายไว้แล้วว่า `LIMIT m OFFSET n` มีต้นทุนแปรผันตาม `O(n + m)`
เพราะฐานข้อมูลต้อง**นับและข้าม**ทุกแถวก่อนตำแหน่ง offset เสมอ ไม่ว่าจะมี index
ช่วยเรื่องการเรียงลำดับหรือไม่ก็ตาม `Paginator` ของ Django เองก็สร้าง SQL แบบ
เดียวกันนี้เป๊ะ (`Paginator` ไม่ได้ฉลาดไปกว่า DRF's `PageNumberPagination` ในแง่นี้
เพราะทั้งคู่เดินตาม SQL semantics เดียวกัน):

```sql
-- ?page=50000 ของ Paginator(queryset, 20)
SELECT * FROM blog_post
WHERE is_published = true
ORDER BY created_at DESC
LIMIT 20 OFFSET 999980;
```

Part นี้จะไม่หยุดแค่ทฤษฎี แต่จะรัน benchmark จริงบนข้อมูล 1,000,000 แถวที่สร้างไว้
ในขั้นตอนที่ 701.3 เพื่อพิสูจน์ตัวเลขให้เห็นชัดเจน

### 702.2 เขียน Management Command สำหรับ Benchmark

```python
# blog/management/commands/benchmark_pagination.py
import time
from django.core.management.base import BaseCommand
from django.core.paginator import Paginator
from django.db import connection, reset_queries
from blog.models import Post


class Command(BaseCommand):
    help = 'วัดเวลาจริงของ Offset Pagination ที่ตำแหน่งความลึกต่าง ๆ กัน'

    def handle(self, *args, **options):
        page_size = 20
        total = Post.objects.filter(is_published=True).count()
        self.stdout.write(f'จำนวน Post ที่ published: {total:,} แถว | page_size = {page_size}\n')

        # ทดสอบที่ตำแหน่งหน้าต่าง ๆ กัน จากตื้นสุดไปลึกสุด
        page_numbers = [1, 100, 1000, 10000, 40000]

        for page_number in page_numbers:
            queryset = Post.objects.filter(is_published=True)
            paginator = Paginator(queryset, page_size)

            if page_number > paginator.num_pages:
                continue

            reset_queries()
            start = time.perf_counter()
            page_obj = paginator.page(page_number)
            list(page_obj.object_list)  # บังคับ evaluate queryset จริง ๆ
            elapsed_ms = (time.perf_counter() - start) * 1000

            offset = (page_number - 1) * page_size
            self.stdout.write(
                f'page={page_number:>6} (offset={offset:>9,}) -> {elapsed_ms:>8.2f} ms'
            )
```

```bash
python manage.py benchmark_pagination
```

### 702.3 ผลลัพธ์ตัวอย่างจากการรันจริง (PostgreSQL, ข้อมูล ~900,000 แถวที่ published)

```
จำนวน Post ที่ published: 900,124 แถว | page_size = 20

page=     1 (offset=        0) ->     1.84 ms
page=   100 (offset=    1,980) ->     3.21 ms
page=  1000 (offset=   19,980) ->    12.47 ms
page= 10000 (offset=  199,980) ->   118.63 ms
page= 40000 (offset=  799,980) ->   487.92 ms
```

(ตัวเลขจริงแปรผันตามฮาร์ดแวร์ เวอร์ชัน PostgreSQL การตั้งค่า `shared_buffers`
และว่าแถวถูก cache ไว้ใน memory ของ OS/DB แล้วหรือยัง แต่**แนวโน้มเชิงเส้นที่ชัน
ขึ้นเรื่อย ๆ ตามความลึกของ offset เป็นจริงเสมอ** ไม่ว่าจะรันบนเครื่องแบบใด)

สังเกตว่าจาก `page=1` ไป `page=40000` เวลาที่ใช้เพิ่มขึ้นเกือบ **265 เท่า** ทั้งที่
ขอข้อมูลแค่ 20 แถวเท่ากันทุกครั้ง — นี่คือหลักฐานที่จับต้องได้ของสิ่งที่ Part 047
อธิบายไว้ในเชิงทฤษฎี

### 702.4 ปัญหาที่สองที่แอบแฝง: `SELECT COUNT(*)` ก็ไม่ฟรีเช่นกัน

```python
# blog/management/commands/benchmark_count.py
import time
from django.core.management.base import BaseCommand
from blog.models import Post


class Command(BaseCommand):
    def handle(self, *args, **options):
        start = time.perf_counter()
        count = Post.objects.filter(is_published=True).count()
        elapsed_ms = (time.perf_counter() - start) * 1000
        self.stdout.write(f'COUNT(*) = {count:,} แถว ใช้เวลา {elapsed_ms:.2f} ms')
```

```
COUNT(*) = 900,124 แถว ใช้เวลา 84.36 ms
```

`Paginator` เรียก `.count()` **ทุกครั้ง**ที่สร้าง `Page` object ใหม่ (เพื่อคำนวณ
`num_pages`) แม้ว่าผู้ใช้จะแค่ต้องการดูหน้าแรกก็ตาม — บนตาราง PostgreSQL ที่มี
`WHERE` clause ซับซ้อนและข้อมูลหลักล้านแถว `COUNT(*)` เพียงอย่างเดียวก็ใช้เวลา
หลักสิบถึงหลักร้อย millisecond ได้แล้ว โดยที่ยังไม่นับต้นทุนของ `OFFSET` เลย

### 702.5 ตารางสรุปปัญหาของ Offset Pagination บนข้อมูลขนาดใหญ่

| ปัญหา | สาเหตุ | ผลกระทบ |
|---|---|---|
| `OFFSET` ช้าลงตามความลึก | ฐานข้อมูลต้องอ่านและทิ้งทุกแถวก่อนตำแหน่ง offset | หน้าลึก ๆ (เช่น "ดูผลการค้นหาหน้า 5,000") ช้าจนผู้ใช้รู้สึกได้ หรือถึงขั้น timeout |
| `COUNT(*)` มีต้นทุนของตัวเอง | ต้องสแกน (หรืออย่างน้อยนับผ่าน index) ทุกแถวที่ตรงเงื่อนไข `WHERE` | รันซ้ำทุกครั้งที่ผู้ใช้เปลี่ยนหน้า ทั้งที่ค่าจริง ๆ ไม่ค่อยเปลี่ยนบ่อย |
| ข้อมูลซ้ำ/ตกหล่นเมื่อมีการเขียนพร้อมกัน (ทบทวน Part 047 ข้อ 467.4) | ตำแหน่ง offset อ้างอิงจาก "ลำดับที่นับได้" ไม่ใช่ค่าจริงของแถว | ผู้ใช้เห็นบทความซ้ำ หรือพลาดบทความไปเมื่อมีการเพิ่ม/ลบข้อมูลระหว่างเปลี่ยนหน้า |
| Deep pagination เปิดช่องให้ scraping/DoS ได้ง่าย | Query ที่รู้ผลลัพธ์แน่นอนว่าจะช้าลงเรื่อย ๆ เป็นเป้าหมายที่ดีสำหรับการโจมตี | ผู้ไม่หวังดีส่ง request `?page=999999` ซ้ำ ๆ ทำให้ database load สูงผิดปกติ |

### 702.6 แล้วเมื่อไหร่ยัง "ใช้ Offset Pagination ได้อยู่"?

**Offset Pagination ไม่ใช่ของที่ต้องเลิกใช้เสมอไป** — มันยังเหมาะกับหลายสถานการณ์:

- ข้อมูลมีขนาดจำกัด (หลักพันถึงหลักหมื่นแถว) ที่ `OFFSET` สูงสุดไม่มีทางลึกมากจริง ๆ
- หน้า Admin ที่ต้องการ "กระโดดไปหน้า 47" หรือเห็นจำนวนหน้าทั้งหมด (UX ที่ Cursor
  Pagination ให้ไม่ได้ ตามที่ Part 047 ข้อ 470.8 อธิบายไว้)
- จำกัดความลึกสูงสุดที่ผู้ใช้เข้าถึงได้ (เช่น "ดูได้ถึงหน้า 500 เท่านั้น กรุณาใช้
  ตัวกรองเพื่อจำกัดผลลัพธ์เพิ่มเติม") ซึ่งเป็นแนวทางที่หลายเว็บไซต์จริงใช้ (เช่น
  ผลการค้นหาของ Google ก็จำกัดจำนวนหน้าที่เข้าถึงได้เช่นกัน)

**กฎการตัดสินใจ**: ถ้าข้อมูลมีแนวโน้มเติบโตแบบไม่มีเพดาน (feed, log, ประวัติ
กิจกรรม) และผู้ใช้มักเลื่อนดูต่อเนื่องมากกว่ากระโดดไปหน้าเฉพาะเจาะจง ให้ใช้
Keyset/Cursor Pagination แทนตั้งแต่ต้น — นี่คือสิ่งที่จะสร้างเองในขั้นตอนถัดไป

---

## ขั้นตอนที่ 703: Implement Keyset/Cursor Pagination เองตั้งแต่ต้น (ไม่พึ่ง DRF)

### 703.1 หลักการของ Keyset Pagination (ทบทวนสั้น ๆ จาก Part 047 ข้อ 469.3)

แทนที่จะบอกฐานข้อมูลว่า "ข้าม N แถวแรกไป" (offset) เราบอกฐานข้อมูลว่า **"เอาแถว
ที่มีค่าน้อยกว่า/มากกว่าค่าที่เห็นล่าสุด"** ตรง ๆ:

```sql
-- Offset-based (ช้าลงตามความลึก)
SELECT * FROM blog_post ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 800000;

-- Keyset-based (เร็วคงที่ ถ้ามี index รองรับ)
SELECT * FROM blog_post
WHERE (created_at, id) < ('2026-03-01 10:00:00', 41823)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Query แบบที่สองใช้ **index seek** ตรงไปยังตำแหน่งที่ `created_at`/`id` ตรงตาม
เงื่อนไขได้เลย โดยไม่ต้องนับผ่านแถวก่อนหน้าแม้แต่แถวเดียว — นี่คือเหตุผลที่มันเร็ว
คงที่ไม่ว่าจะลึกแค่ไหน (`O(log n + m)` ตามที่ Part 047 อธิบายไว้)

### 703.2 ทำไมต้อง **สองฟิลด์** เสมอ: `created_at` อย่างเดียวไม่พอ

ถ้าสอง `Post` มี `created_at` เท่ากันเป๊ะ (เกิดขึ้นได้จริงถ้า `bulk_create` ยิง
timestamp เดียวกันให้หลายแถว หรือ resolution ของ timestamp ไม่ละเอียดพอ) การใช้
`created_at` เพียงอย่างเดียวเป็นเงื่อนไขจะทำให้แถวที่มีค่าซ้ำกันถูก**ข้ามไปเงียบ ๆ
บางแถว** หรือ**แสดงซ้ำ**ได้ — ทางแก้คือเพิ่ม field ที่ **unique เสมอ** (เช่น `id`)
เป็น **tie-breaker** ควบคู่กันเสมอ ต้องเรียงด้วยทั้งคู่และใช้เงื่อนไขแบบ **tuple
comparison**:

```
(created_at, id) < (ค่า created_at ล่าสุดที่เห็น, ค่า id ล่าสุดที่เห็น)
```

นี่คือเหตุผลที่ index ในขั้นตอนที่ 701.2 เป็น **composite index** บน
`(-created_at, id)` ไม่ใช่แค่ `created_at` เฉย ๆ

### 703.3 เขียน Helper Class `KeysetPaginator` เอง

```python
# core/pagination.py
"""
Keyset (Cursor) Pagination ที่เขียนเองสำหรับ server-rendered view โดยไม่พึ่ง DRF
ใช้แนวคิดเดียวกับ rest_framework.pagination.CursorPagination (Part 047 ข้อ 467.2)
แต่ทำงานกับ Django QuerySet ตรง ๆ และ encode/decode cursor เอง
"""
import base64
import json
from dataclasses import dataclass
from django.db.models import Q


@dataclass
class KeysetPage:
    """ผลลัพธ์ของการดึงข้อมูลหนึ่งหน้าแบบ keyset"""
    object_list: list
    has_next: bool
    has_previous: bool
    next_cursor: str | None
    previous_cursor: str | None


class KeysetPaginator:
    """
    Keyset Paginator ทั่วไปที่ใช้ได้กับ Model ใดก็ได้ ตราบใดที่มี field สำหรับ
    เรียงลำดับ (ordering_field) และ field ที่ unique เสมอสำหรับ tie-break (tie_field)

    ตัวอย่าง:
        paginator = KeysetPaginator(
            queryset=Post.objects.filter(is_published=True),
            ordering_field='created_at',
            tie_field='id',
            page_size=20,
        )
        page = paginator.get_page(cursor=request.GET.get('cursor'))
    """

    def __init__(self, queryset, ordering_field, tie_field, page_size, descending=True):
        self.queryset = queryset
        self.ordering_field = ordering_field
        self.tie_field = tie_field
        self.page_size = page_size
        self.descending = descending

        order_prefix = '-' if descending else ''
        self.queryset = self.queryset.order_by(
            f'{order_prefix}{ordering_field}', f'{order_prefix}{tie_field}'
        )

    @staticmethod
    def encode_cursor(ordering_value, tie_value) -> str:
        """แปลงค่าตำแหน่งปัจจุบันเป็น string ที่ปลอดภัยสำหรับใส่ใน URL"""
        payload = json.dumps([str(ordering_value), tie_value])
        return base64.urlsafe_b64encode(payload.encode()).decode()

    @staticmethod
    def decode_cursor(cursor: str):
        """ถอด cursor string กลับเป็นค่า (ordering_value, tie_value) — คืน None ถ้า cursor เสีย"""
        try:
            payload = base64.urlsafe_b64decode(cursor.encode()).decode()
            ordering_value, tie_value = json.loads(payload)
            return ordering_value, tie_value
        except (ValueError, TypeError, UnicodeDecodeError, json.JSONDecodeError):
            # cursor ที่ client ส่งมาผิดรูปแบบหรือถูกแก้ไข — ปฏิบัติเหมือนไม่มี cursor
            return None

    def get_page(self, cursor: str | None) -> KeysetPage:
        qs = self.queryset
        has_previous = cursor is not None

        if cursor:
            decoded = self.decode_cursor(cursor)
            if decoded is not None:
                ordering_value, tie_value = decoded
                comparator = 'lt' if self.descending else 'gt'
                qs = qs.filter(
                    Q(**{f'{self.ordering_field}__{comparator}': ordering_value}) |
                    Q(**{f'{self.ordering_field}': ordering_value}, **{f'{self.tie_field}__{comparator}': tie_value})
                )
            else:
                has_previous = False

        # ดึงมาเกิน 1 แถว เพื่อรู้ว่า "มีหน้าถัดไปอีกหรือไม่" โดยไม่ต้องรัน COUNT(*) เพิ่ม
        rows = list(qs[: self.page_size + 1])
        has_next = len(rows) > self.page_size
        rows = rows[: self.page_size]

        next_cursor = None
        if has_next and rows:
            last = rows[-1]
            next_cursor = self.encode_cursor(
                getattr(last, self.ordering_field), getattr(last, self.tie_field)
            )

        previous_cursor = None
        if has_previous and rows:
            first = rows[0]
            previous_cursor = self.encode_cursor(
                getattr(first, self.ordering_field), getattr(first, self.tie_field)
            )

        return KeysetPage(
            object_list=rows,
            has_next=has_next,
            has_previous=has_previous,
            next_cursor=next_cursor,
            previous_cursor=previous_cursor,
        )
```

### 703.4 อธิบายเทคนิคสำคัญ: "ดึงเกินมา 1 แถว" แทนการรัน `COUNT(*)`

สังเกตบรรทัด `qs[: self.page_size + 1]` — แทนที่จะรัน `SELECT COUNT(*)` แยก
ต่างหากเพื่อรู้ว่า "ยังมีข้อมูลต่ออีกไหม" (ซึ่งมีต้นทุนตามขั้นตอนที่ 702.4) เราขอ
ข้อมูลเกินมา **1 แถว** จากที่จะแสดงจริง แล้วเช็คว่าได้ครบ `page_size + 1` แถวหรือไม่:

- ถ้าได้ `page_size + 1` แถว → **มีหน้าถัดไปแน่นอน** (ตัดแถวที่ 21 ทิ้งก่อนแสดงผล)
- ถ้าได้น้อยกว่านั้น → **นี่คือหน้าสุดท้ายแล้ว**

เทคนิคนี้ (เรียกกันทั่วไปว่า "**fetch N+1**") คือมาตรฐานของ Keyset Pagination ใน
อุตสาหกรรม (DRF's `CursorPagination` เองก็ใช้เทคนิคเดียวกันนี้เบื้องหลัง) — ทำให้
`CursorPagination` ไม่ต้องรัน query `COUNT(*)` เลย จึงไม่มี field `count` ใน
response ตามที่ Part 047 ข้อ 467.2 สังเกตไว้

### 703.5 นำ `KeysetPaginator` ไปใช้ใน View จริง

```python
# blog/views.py
from django.shortcuts import render
from core.pagination import KeysetPaginator
from .models import Post


def post_list_keyset_view(request):
    post_list = Post.objects.filter(is_published=True)

    paginator = KeysetPaginator(
        queryset=post_list,
        ordering_field='created_at',
        tie_field='id',
        page_size=20,
    )
    page = paginator.get_page(cursor=request.GET.get('cursor'))

    return render(request, 'blog/post_list_keyset.html', {'page': page})
```

```html
<!-- blog/templates/blog/post_list_keyset.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด (Keyset Pagination)</h1>

<ul>
    {% for post in page.object_list %}
        <li>{{ post.title }} — {{ post.created_at|date:'d/m/Y H:i' }}</li>
    {% endfor %}
</ul>

<nav class="pagination">
    {% if page.has_previous %}
        <a href="?">« เริ่มต้นใหม่</a>
    {% endif %}
    {% if page.has_next %}
        <a href="?cursor={{ page.next_cursor }}">ถัดไป »</a>
    {% endif %}
</nav>
{% endblock %}
```

### 703.6 ข้อจำกัดที่ต้องยอมรับ: ไม่มีเลขหน้า ไม่มีปุ่ม "ย้อนกลับไปหน้าก่อน" แบบสมบูรณ์

สังเกตว่า template ข้างบนมีแค่ปุ่ม "ถัดไป" กับ "เริ่มต้นใหม่" — **ไม่มีปุ่ม
"ก่อนหน้า" ที่แท้จริง** เพราะ implementation ข้างต้นเป็นเวอร์ชันแบบง่าย (เดินหน้า
ทางเดียว) ซึ่งพอเพียงสำหรับ infinite scroll (ขั้นตอนที่ 704) แล้ว การทำ "ก่อนหน้า"
ที่สมบูรณ์ต้องเก็บ **cursor stack** ของทุกหน้าที่เคยผ่านมาไว้ฝั่ง client (เช่น
ใน URL query string history หรือ session) แล้วดึง cursor ก่อนหน้าออกมาใช้ตอนกด
ย้อนกลับ — DRF's `CursorPagination` แก้ปัญหานี้ด้วยการ encode ทิศทาง (`p=`/`r=`)
ไว้ใน cursor เอง ซึ่งซับซ้อนกว่าที่ scope ของ Part นี้ต้องการ

**นี่คือข้อแลกเปลี่ยน (trade-off) ที่ต้องยอมรับเสมอเมื่อเลือก Keyset Pagination**:
แลกความสามารถในการ "กระโดดไปหน้าใดก็ได้ตามใจ" กับ **performance ที่คงที่ไม่ว่า
จะลึกแค่ไหน** — ตามตารางสรุปใน Part 047 ข้อ 467.3

---

## ขั้นตอนที่ 704: รูปแบบ Infinite Scroll ที่สมบูรณ์ (เชื่อมกับ HTMX จาก Part 053)

### 704.1 ทบทวนข้อจำกัดของ Infinite Scroll แบบ Part 053

Part 053 ข้อ 527 สอน Infinite Scroll ด้วย `hx-trigger="revealed"` แต่ใช้
`Paginator` แบบ offset ธรรมดา (ข้อ 527.2 ใช้ `paginator.get_page(page_number)`)
ซึ่งใช้งานได้ดีถ้าข้อมูลไม่เยอะมาก แต่ตามที่ขั้นตอนที่ 702 พิสูจน์ไว้ — ถ้าผู้ใช้
เลื่อนลึกมาก ๆ (เช่น scroll ผ่านไป 5,000 หน้า) การโหลดหน้าถัดไปแต่ละครั้งจะช้าลง
เรื่อย ๆ ด้วย **ลักษณะของ infinite scroll ยิ่งทำให้ปัญหานี้เห็นชัดกว่าการแบ่งหน้า
แบบเลขหน้าเสียอีก** เพราะผู้ใช้มักเลื่อนต่อเนื่องนาน ๆ โดยไม่หยุด

Part นี้จะประกอบ `KeysetPaginator` จากขั้นตอนที่ 703 เข้ากับรูปแบบ Infinite Scroll
ของ HTMX จาก Part 053 เพื่อให้ได้ทั้งประสบการณ์ผู้ใช้ที่ลื่นไหลและ performance
ที่คงที่ไม่ว่าจะเลื่อนลึกแค่ไหน

### 704.2 View ที่รองรับทั้ง Full Page และ HTMX Partial

```python
# blog/views.py
from django.shortcuts import render
from core.pagination import KeysetPaginator
from .models import Post


def post_list_infinite_keyset_view(request):
    post_list = Post.objects.filter(is_published=True)

    paginator = KeysetPaginator(
        queryset=post_list,
        ordering_field='created_at',
        tie_field='id',
        page_size=20,
    )
    page = paginator.get_page(cursor=request.GET.get('cursor'))

    # ทบทวนจาก Part 053 ข้อ 524.2: ตรวจ header ที่ HTMX แนบมาเองทุกครั้ง
    if request.headers.get('HX-Request') == 'true':
        return render(request, 'blog/partials/post_page_keyset.html', {'page': page})
    return render(request, 'blog/post_list_infinite_keyset.html', {'page': page})
```

```python
# blog/urls.py
path('infinite-keyset/', views.post_list_infinite_keyset_view, name='list_infinite_keyset'),
```

### 704.3 Template: Sentinel ที่ใช้ Cursor แทนเลขหน้า

```html
<!-- blog/templates/blog/post_list_infinite_keyset.html -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด (Infinite Scroll + Keyset Pagination)</h1>
<div id="post-container">
    {% include 'blog/partials/post_page_keyset.html' %}
</div>
{% endblock %}
```

```html
<!-- blog/templates/blog/partials/post_page_keyset.html -->
{% for post in page.object_list %}
    <article class="post-card">
        <h2><a href="{% url 'blog:detail' post.slug %}">{{ post.title }}</a></h2>
        <p>{{ post.content|truncatewords:30 }}</p>
        <time>{{ post.created_at|date:'d/m/Y H:i' }}</time>
    </article>
{% endfor %}

{% if page.has_next %}
    <div hx-get="{% url 'blog:list_infinite_keyset' %}?cursor={{ page.next_cursor }}"
         hx-trigger="revealed"
         hx-swap="outerHTML"
         hx-indicator="#loading-spinner-{{ page.next_cursor }}">
        <div id="loading-spinner-{{ page.next_cursor }}" class="htmx-indicator">
            กำลังโหลดเพิ่มเติม...
        </div>
    </div>
{% else %}
    <p class="end-of-list">— แสดงบทความครบทุกรายการแล้ว —</p>
{% endif %}
```

**จุดที่ต่างจาก Part 053 ข้อ 527.3 ชัดเจนที่สุด**: URL ของ sentinel เปลี่ยนจาก
`?page={{ page_obj.next_page_number }}` (เลขหน้าที่นับได้) เป็น
`?cursor={{ page.next_cursor }}` (ตำแหน่งอ้างอิงที่ encode มาจากค่าจริงของแถว
สุดท้ายที่เห็น) — โครงสร้าง HTML, `hx-trigger="revealed"`, `hx-swap="outerHTML"`,
และกลไก sentinel ทั้งหมด**เหมือนเดิมทุกประการ** เพราะ HTMX ไม่สนใจว่า query
parameter ที่ backend ใช้จะชื่ออะไร มันแค่ยิง `hx-get` ไปตาม URL ที่ server บอกมา
เท่านั้น — นี่คือจุดแข็งของสถาปัตยกรรม HTML-over-the-wire: **เปลี่ยนกลไก
pagination ฝั่ง backend ทั้งหมดโดยแทบไม่ต้องแตะ frontend เลย**

### 704.4 เหตุผลที่ต้องใส่ `page.next_cursor` ต่อท้าย `id` ของ loading spinner

สังเกตว่า `id="loading-spinner-{{ page.next_cursor }}"` มีค่า cursor ต่อท้ายแทนที่
จะเป็น `id` คงที่แบบ `#loading-spinner` เหมือน Part 053 — เหตุผลคือ **แต่ละหน้า
ที่โหลดเข้ามาใหม่จะมี sentinel `<div>` ของตัวเองซ้อนกันไปเรื่อย ๆ ใน DOM**
(เพราะ `hx-swap="outerHTml"` แทนที่แค่ sentinel เดิม ไม่ได้ลบของเก่าทั้งหน้า)
ถ้าทุก spinner ใช้ `id` เดียวกัน ค่าจะซ้ำกันในหน้าเดียวกันหลายจุด ซึ่งผิดกฎ HTML
(id ต้อง unique ทั้งหน้า) และ `hx-indicator` แบบ selector อาจไปจับ element ที่
ไม่ตั้งใจ — การผูก cursor ที่ unique เข้ากับ id แก้ปัญหานี้ได้ตรงจุด

### 704.5 ตารางเปรียบเทียบ Infinite Scroll แบบ Offset (Part 053) กับแบบ Keyset (Part นี้)

| ประเด็น | Infinite Scroll แบบ Offset (Part 053) | Infinite Scroll แบบ Keyset (Part นี้) |
|---|---|---|
| Query parameter ของ sentinel | `?page=N` | `?cursor=<encoded string>` |
| Performance เมื่อเลื่อนลึกมาก | ช้าลงเรื่อย ๆ ตาม `OFFSET` | คงที่ไม่ว่าจะเลื่อนลึกแค่ไหน |
| ปลอดภัยจากข้อมูลซ้ำ/ตกหล่นเมื่อมีข้อมูลใหม่แทรกเข้ามาระหว่าง scroll | ❌ เสี่ยง | ✅ ปลอดภัยกว่ามาก |
| ความซับซ้อนของโค้ด backend | ต่ำ (`Paginator` ธรรมดา) | สูงกว่าเล็กน้อย (ต้องเขียน `KeysetPaginator` เอง) |
| เหมาะกับข้อมูลขนาด | หลักพัน-หมื่นแถว | หลักแสน-ล้านแถวขึ้นไป, feed ที่ข้อมูลเปลี่ยนตลอดเวลา |

**คำแนะนำระดับมืออาชีพ**: เริ่มเขียนฟีเจอร์ infinite scroll ด้วย Offset Pagination
ได้ตามปกติถ้ารู้ชัดว่าข้อมูลจะไม่โตเกินหลักหมื่นแถว แต่ถ้าเป็น feed สาธารณะที่ไม่มี
เพดานการเติบโต (timeline, notification, activity log) ให้ลงทุนเขียน Keyset
Pagination ตั้งแต่ต้น เพราะการเปลี่ยนกลไก pagination ทีหลังเมื่อข้อมูลโตจนช้าแล้ว
มักต้องแก้ URL scheme ที่ผู้ใช้ bookmark หรือ share ไว้ (ทบทวนแนวคิด backward
compatibility จาก Part 047 ข้อ 466.3)

---

## ขั้นตอนที่ 705: จัดการการ Export ข้อมูลจำนวนมาก — Streaming CSV/Excel แทนการโหลดทั้งหมดเข้า memory

### 705.1 ปัญหาของการ Export แบบเดิม: สร้างไฟล์ทั้งหมดใน memory ก่อน

วิธี export CSV ที่มือใหม่มักเขียน (และใช้ได้ดีกับข้อมูลน้อย ๆ):

```python
# blog/views.py — วิธีที่ "ใช้ได้" กับข้อมูลน้อย แต่อันตรายกับข้อมูลเยอะ
import csv
from django.http import HttpResponse
from .models import Post


def export_posts_csv_naive(request):
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="posts.csv"'

    writer = csv.writer(response)
    writer.writerow(['ID', 'Title', 'Created At', 'View Count'])

    # อันตราย: Post.objects.all() ดึงข้อมูลทั้งหมดเข้า memory ก่อนเริ่มเขียนแม้แต่บรรทัดเดียว
    for post in Post.objects.all():
        writer.writerow([post.id, post.title, post.created_at, post.view_count])

    return response
```

ปัญหาของโค้ดนี้เมื่อ `Post` มี 1 ล้านแถว:

1. **Memory พุ่งสูง**: แม้ `csv.writer` จะเขียนเข้า `response` ทีละบรรทัด แต่
   `Post.objects.all()` (โดยไม่ใช้ `.iterator()`) จะ**แคช QuerySet ทั้งหมดไว้ใน
   memory ของ Django ก่อน** (พฤติกรรม default ของ QuerySet — ทบทวนจาก Phase 2)
   ทำให้ต้องใช้ RAM หลายร้อย MB ถึงหลาย GB สำหรับข้อมูลระดับล้านแถว
2. **ผู้ใช้รอนานโดยไม่เห็นอะไรเลย**: `HttpResponse` แบบธรรมดารอให้ **ทุกบรรทัด**
   ถูกเขียนเสร็จก่อน ถึงจะเริ่มส่ง response กลับไปยัง browser — ถ้าประมวลผล
   1 ล้านแถวใช้เวลา 30 วินาที ผู้ใช้จะเห็นหน้าเว็บค้าง 30 วินาทีโดยไม่มี progress
   ใด ๆ เลย (browser ไม่รู้ด้วยซ้ำว่ากำลังดาวน์โหลดไฟล์)
3. **เสี่ยง Worker Timeout**: web server (Gunicorn/uWSGI) มักตั้ง timeout ต่อ
   request ไว้ (เช่น 30-60 วินาที) — ถ้า export ใช้เวลานานกว่านั้น request จะถูก
   ตัดกลางคันและผู้ใช้ได้ error 504 โดยไม่รู้สาเหตุ

### 705.2 ทางแก้: Streaming CSV ด้วย `StreamingHttpResponse`

หลักการคือ**เขียนแถวข้อมูลออกไปทีละแถวทันทีที่พร้อม** แทนที่จะรอให้ครบทุกแถวก่อน
โดยใช้ Python **generator** ร่วมกับ `django.http.StreamingHttpResponse`:

```python
# blog/views.py
import csv
from django.http import StreamingHttpResponse
from .models import Post


class Echo:
    """
    Class หลอก ๆ ที่ทำหน้าที่แทน "ไฟล์" สำหรับ csv.writer — csv.writer ต้องการ
    object ที่มี method .write() เท่านั้น เราไม่ต้องการเก็บอะไรจริง ๆ แค่ต้องการ
    ให้มันคืนค่า string ที่เขียนกลับมาเฉย ๆ (เพื่อส่งต่อเป็น chunk ของ stream)
    """
    def write(self, value):
        return value


def export_posts_csv_streaming(request):
    def row_generator():
        pseudo_buffer = Echo()
        writer = csv.writer(pseudo_buffer)

        yield writer.writerow(['ID', 'Title', 'Created At', 'View Count'])

        # .iterator() ทำให้ Django ดึงข้อมูลจากฐานข้อมูลเป็น chunk แทนที่จะแคช
        # QuerySet ทั้งหมดไว้ใน memory ก่อน (เจาะลึกกลไกนี้เต็มรูปแบบในขั้นตอนที่ 707)
        queryset = Post.objects.all().iterator(chunk_size=2000)
        for post in queryset:
            yield writer.writerow([post.id, post.title, post.created_at, post.view_count])

    response = StreamingHttpResponse(row_generator(), content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="posts_export.csv"'
    return response
```

```python
# blog/urls.py
path('export/csv/', views.export_posts_csv_streaming, name='export_csv'),
```

**ผลลัพธ์**: browser เริ่มได้รับข้อมูล (และเริ่มแสดง progress bar ของการ
download) **ทันที**ที่แถวแรกถูกเขียน ไม่ต้องรอให้ query ทั้งหมดจบก่อน และ memory
ที่ Django worker ใช้อยู่ในระดับคงที่ (แค่ chunk ปัจจุบันเท่านั้น) ไม่ว่าตารางจะมี
1,000 หรือ 10,000,000 แถวก็ตาม

### 705.3 เหตุใด "Echo class" ถึงจำเป็น: `csv.writer` ต้องการ File-like Object

`csv.writer(f)` ถูกออกแบบมาให้เขียนไปที่ **ไฟล์จริง** (`f.write(string)`) เสมอ
ไม่ได้ออกแบบมาให้ "คืนค่า string กลับมา" แบบ generator — เทคนิค `Echo` class คือ
**pattern มาตรฐานที่เอกสารทางการของ Django แนะนำ** สำหรับ "หลอก" ให้ `csv.writer`
เขียนไปที่ object ปลอมที่ทำแค่ "รับค่าที่จะเขียนแล้วคืนค่านั้นกลับออกมาทันที"
ทำให้เราควบคุม stream ของข้อมูลด้วย generator (`yield`) ได้เต็มที่

### 705.4 Export เป็น Excel (`.xlsx`) แบบ Streaming ด้วย `openpyxl`

Excel ซับซ้อนกว่า CSV เพราะเป็น **binary format ที่มีโครงสร้างไฟล์แบบ ZIP**
ข้างในไม่สามารถเขียนทีละแถวแบบ pure streaming ได้ตรง ๆ เหมือน CSV — แต่ไลบรารี
`openpyxl` มีโหมด **`write_only`** ที่ออกแบบมาสำหรับสร้างไฟล์ขนาดใหญ่โดยใช้
memory คงที่ (ไม่เก็บทุกแถวไว้ใน memory พร้อมกัน):

```bash
pip install openpyxl
pip freeze > requirements.txt
```

```python
# blog/views.py
import io
from django.http import StreamingHttpResponse
from openpyxl import Workbook
from .models import Post


def export_posts_excel_streaming(request):
    def generate_excel_chunks():
        # write_only=True บอก openpyxl ให้เขียนแบบ append-only ไม่เก็บทุกแถวไว้
        # ใน memory (ต่างจากโหมดปกติที่โหลดทั้ง workbook ไว้ใน RAM)
        workbook = Workbook(write_only=True)
        worksheet = workbook.create_sheet('Posts')
        worksheet.append(['ID', 'Title', 'Created At', 'View Count'])

        queryset = Post.objects.all().iterator(chunk_size=2000)
        for post in queryset:
            worksheet.append([
                post.id, post.title,
                post.created_at.strftime('%Y-%m-%d %H:%M'),
                post.view_count,
            ])

        # openpyxl ต้อง save() เป็นก้อนเดียวสุดท้าย (ข้อจำกัดของ .xlsx ZIP format
        # ที่ไม่สามารถ stream เขียนทีละไบต์แบบ CSV ได้จริง ๆ) — เขียนลง buffer
        # ใน memory ก่อน แล้วค่อย yield ออกมาเป็นก้อนเดียว
        buffer = io.BytesIO()
        workbook.save(buffer)
        buffer.seek(0)
        yield buffer.read()

    response = StreamingHttpResponse(
        generate_excel_chunks(),
        content_type='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    )
    response['Content-Disposition'] = 'attachment; filename="posts_export.xlsx"'
    return response
```

**ข้อจำกัดที่ต้องยอมรับตรง ๆ**: ต่างจาก CSV ที่ stream ได้แบบทีละบรรทัดจริง ๆ
`.xlsx` เป็น ZIP archive ที่ต้องปิด (finalize) ไฟล์ตอนท้ายเสมอ ทำให้**หน่วยความจำ
ระหว่างสร้างยังคงประหยัดกว่ามาก** (เพราะ `write_only=True` ไม่เก็บทุกแถวเป็น
Python object ค้างไว้) **แต่ response ยังไม่เริ่มส่งออกไปจน `workbook.save()`
เสร็จ** — สรุปคือ `write_only=True` แก้ปัญหา **memory** ได้เต็มที่ แต่ไม่แก้ปัญหา
**"ผู้ใช้ต้องรอ" ก่อนเห็น progress** เหมือน CSV แบบ streaming แท้ ๆ

### 705.5 ตารางสรุปข้อจำกัดของแต่ละรูปแบบไฟล์เมื่อ Export ข้อมูลขนาดใหญ่

| รูปแบบไฟล์ | Stream ได้แบบ real-time หรือไม่ | ประหยัด Memory หรือไม่ | เหมาะกับข้อมูลขนาด |
|---|---|---|---|
| CSV (ผ่าน `StreamingHttpResponse` + generator) | ✅ ได้เต็มที่ (ทีละบรรทัด) | ✅ คงที่ไม่ว่าข้อมูลจะเยอะแค่ไหน | ไม่จำกัด (ล้าน-สิบล้านแถวได้สบาย) |
| Excel `.xlsx` (`openpyxl` โหมด `write_only`) | ❌ ต้องรอ `save()` จบก่อนส่ง response | ✅ ประหยัด memory ระหว่างสร้าง (ไม่เก็บทุกแถวเป็น object) | หลักแสนถึงหลักล้านแถว (ขึ้นกับ disk/memory buffer) |
| PDF (report ที่ format ซับซ้อน) | ❌ แทบเป็นไปไม่ได้ที่จะ stream จริง | ❌ มักต้องประกอบ layout ทั้งหมดก่อน render | ไม่เหมาะกับข้อมูลดิบจำนวนมาก ควรใช้กับ summary/report สรุปเท่านั้น |

**คำแนะนำระดับมืออาชีพ**: เมื่อผู้ใช้ต้องการ "export ข้อมูลดิบจำนวนมาก" ให้เสนอ
CSV เป็นตัวเลือกหลักเสมอ (เร็วที่สุด ประหยัด memory ที่สุด อ่านได้ด้วยเครื่องมือ
เกือบทุกชนิด) และเสนอ Excel เป็นตัวเลือกเสริมสำหรับผู้ใช้ที่ต้องการเปิดในโปรแกรม
สเปรดชีตโดยตรงพร้อม formatting ส่วน export ที่ใหญ่มากจริง ๆ (สิบล้านแถวขึ้นไป)
ควรย้ายไปเป็น background job ตามที่จะแนะนำในขั้นตอนที่ 708 แทนที่จะทำใน
request-response cycle ตรง ๆ

---

## ขั้นตอนที่ 706: `StreamingHttpResponse` สำหรับไฟล์ดาวน์โหลดขนาดใหญ่

### 706.1 `StreamingHttpResponse` ไม่ใช่แค่สำหรับ CSV: ใช้กับไฟล์ใด ๆ ที่มีอยู่แล้วบน disk ได้

ขั้นตอนที่ 705 ใช้ `StreamingHttpResponse` กับข้อมูลที่**สร้างขึ้นแบบ dynamic**
(query จากฐานข้อมูล) แต่หลักการเดียวกันนี้ใช้กับ **ไฟล์ขนาดใหญ่ที่มีอยู่แล้วบน
disk** ได้เช่นกัน (เช่น ไฟล์ backup, video, archive) โดยไม่ต้องอ่านทั้งไฟล์เข้า
memory ก่อนส่ง:

```python
# core/views.py
from django.http import StreamingHttpResponse, Http404
from django.conf import settings
import os


def download_large_file(request, filename):
    file_path = os.path.join(settings.MEDIA_ROOT, 'exports', filename)

    if not os.path.exists(file_path):
        raise Http404('ไม่พบไฟล์ที่ต้องการ')

    def file_iterator(path, chunk_size=8192):
        with open(path, 'rb') as f:
            while chunk := f.read(chunk_size):
                yield chunk

    response = StreamingHttpResponse(
        file_iterator(file_path),
        content_type='application/octet-stream',
    )
    response['Content-Disposition'] = f'attachment; filename="{filename}"'
    response['Content-Length'] = os.path.getsize(file_path)
    return response
```

`file_iterator` อ่านไฟล์ทีละ **8 KB** (ปรับขนาดได้ตามความเหมาะสม) แทนที่จะเปิด
ไฟล์ทั้งก้อนด้วย `f.read()` เพียงครั้งเดียว — ทำให้ไม่ว่าไฟล์จะมีขนาด 10 MB หรือ
10 GB, memory ที่ view นี้ใช้ก็คงที่อยู่ที่ประมาณ 8 KB ต่อครั้งเสมอ

### 706.2 `Content-Length`: ทำไมควรใส่เมื่อรู้ขนาดไฟล์แน่นอน

การใส่ header `Content-Length` (บรรทัดสุดท้ายในตัวอย่างข้างบน) ทำให้ browser
**รู้ขนาดไฟล์ทั้งหมดล่วงหน้า** และแสดง progress bar ของการดาวน์โหลดที่ถูกต้อง
("กำลังโหลด 45%") — ถ้าไม่ใส่ (เช่นกรณี CSV แบบ streaming ในขั้นตอนที่ 705 ที่
**ไม่รู้ขนาดล่วงหน้า**เพราะข้อมูลถูกสร้างขึ้นระหว่างทาง) browser จะแสดงแค่
"กำลังโหลด... (ไม่ทราบขนาด)" แทน ซึ่งเป็นเรื่องปกติและยอมรับได้สำหรับข้อมูลที่
generate แบบ dynamic

### 706.3 `FileResponse`: ทางลัดของ Django สำหรับกรณีไฟล์ที่มีอยู่แล้ว

Django มีคลาสสำเร็จรูปชื่อ `FileResponse` (สืบทอดจาก `StreamingHttpResponse`)
ที่ทำสิ่งเดียวกับขั้นตอนที่ 706.1 ให้อัตโนมัติ **ไม่ต้องเขียน `file_iterator` เอง**:

```python
# core/views.py
from django.http import FileResponse, Http404
from django.conf import settings
import os


def download_large_file_v2(request, filename):
    file_path = os.path.join(settings.MEDIA_ROOT, 'exports', filename)

    if not os.path.exists(file_path):
        raise Http404('ไม่พบไฟล์ที่ต้องการ')

    # FileResponse เปิดไฟล์และ stream ให้อัตโนมัติ พร้อมตั้ง Content-Length
    # และ Content-Type (เดาจากนามสกุลไฟล์) ให้เองด้วย
    return FileResponse(
        open(file_path, 'rb'),
        as_attachment=True,
        filename=filename,
    )
```

**คำแนะนำระดับมืออาชีพ**: ใช้ `FileResponse` เสมอเมื่อ serve ไฟล์ที่มีอยู่แล้ว
บน disk ตรง ๆ (ไม่ต้อง generate อะไรเพิ่ม) เพราะสั้นกว่า ปลอดภัยกว่า (Django
ดูแลการปิดไฟล์ให้เองผ่าน context manager ภายใน) — เขียน `StreamingHttpResponse`
+ generator เองเฉพาะกรณีที่ต้อง**สร้างเนื้อหาระหว่างทาง** (เช่น CSV export จาก
query) แบบขั้นตอนที่ 705 เท่านั้น

### 706.4 ข้อควรระวัง: `StreamingHttpResponse` ใช้กับ Middleware บางตัวไม่ได้เต็มรูปแบบ

เอกสารทางการของ Django เตือนไว้ชัดเจนว่า `StreamingHttpResponse` **ไม่มี
attribute `content`** (เพราะเนื้อหาถูกส่งเป็น stream ไม่ใช่ก้อนเดียวที่จับต้องได้)
ทำให้ middleware ที่คาดหวังจะอ่าน/แก้ไข `response.content` โดยตรง (เช่น
middleware บีบอัด gzip แบบเก่า หรือ middleware ที่แทรก analytics script เข้า
`<body>` ตอนท้าย) **จะไม่ทำงานกับ response ประเภทนี้** — Django's
`GZipMiddleware` เวอร์ชันปัจจุบันรองรับ streaming response แล้ว แต่ third-party
middleware บางตัวอาจยังไม่รองรับ ควรทดสอบให้แน่ใจก่อนใช้งานจริงเสมอ

---

## ขั้นตอนที่ 707: การประมวลผล QuerySet ขนาดใหญ่แบบ Chunk — `iterator()`, `chunk_size` parameter

### 707.1 ทบทวน: ทำไม QuerySet ปกติถึง "แคช" ผลลัพธ์ไว้ใน Memory

ตั้งแต่ Phase 2 คุณเรียนรู้ว่า Django QuerySet เป็น **lazy** (ไม่ query จริงจนกว่า
จะถูก evaluate) แต่สิ่งที่มักถูกมองข้ามคือ: **เมื่อ evaluate แล้ว QuerySet จะแคช
ผลลัพธ์ทั้งหมดไว้ใน memory** เพื่อให้ iterate ซ้ำได้โดยไม่ query ฐานข้อมูลซ้ำ:

```python
posts = Post.objects.all()   # ยังไม่ query
for post in posts:            # evaluate ครั้งแรก -> query จริง + แคชผลลัพธ์ทั้งหมดใน memory
    print(post.title)

for post in posts:            # ครั้งที่สอง -> ใช้ค่าที่แคชไว้ ไม่ query ซ้ำ (เร็ว แต่กิน memory)
    print(post.view_count)
```

พฤติกรรมนี้มีประโยชน์มากสำหรับ QuerySet ขนาดปกติ (หลักสิบถึงหลักพันแถว) แต่กลาย
เป็นปัญหาใหญ่เมื่อ QuerySet มีข้อมูลระดับ**แสนถึงล้านแถว** — การ `for post in
Post.objects.all()` (1 ล้านแถว) จะพยายามโหลดทุก row object เข้า memory ก่อนเริ่ม
iterate ใช้ RAM หลักร้อย MB ถึงหลาย GB ขึ้นกับขนาดของแต่ละ row

### 707.2 `QuerySet.iterator()`: ประมวลผลทีละ Chunk แทนที่จะแคชทั้งหมด

```python
# ก่อน: แคชทั้งหมดใน memory (อันตรายกับข้อมูลเยอะ)
for post in Post.objects.all():
    process(post)

# หลัง: ดึงข้อมูลจากฐานข้อมูลเป็น chunk (ค่า default คือ 2000 แถวต่อ chunk)
for post in Post.objects.all().iterator():
    process(post)

# ระบุขนาด chunk เอง (Django 4.1+ รองรับ chunk_size พารามิเตอร์)
for post in Post.objects.all().iterator(chunk_size=5000):
    process(post)
```

`iterator()` สั่งให้ Django ใช้กลไกของ database driver ในการดึงข้อมูลทีละก้อน
(บน PostgreSQL คือการใช้ **server-side cursor**) แทนที่จะดึงผลลัพธ์ทั้งหมดมา
สร้าง Python object พร้อมกันทีเดียว ทำให้ memory ที่ใช้คงที่ตลอดการวนลูป
ไม่ว่า QuerySet จะมีกี่ล้านแถวก็ตาม

### 707.3 ข้อแลกเปลี่ยนที่ต้องรู้เมื่อใช้ `iterator()`

| ประเด็น | QuerySet ปกติ | `.iterator()` |
|---|---|---|
| Memory | สูง (แคชทุกแถวไว้) | ต่ำและคงที่ (ประมวลผลทีละ chunk) |
| Iterate ซ้ำได้ไหม | ✅ ได้ (ใช้ค่าที่แคชไว้) | ❌ **ไม่ได้** — iterate ได้แค่รอบเดียว ต้อง query ใหม่ถ้าต้องการวนซ้ำ |
| ใช้ `len()` กับผลลัพธ์ได้ไหม | ✅ ได้ (Django ดึงข้อมูลมาแล้วนับ) | ❌ ไม่แนะนำ (ต้องดึงข้อมูลทั้งหมดมาก่อนถึงจะนับได้ ขัดกับเจตนาของการใช้ chunk) |
| `prefetch_related()` ใช้ร่วมได้ไหม | ✅ ได้ปกติ | ⚠️ Django เวอร์ชันใหม่ (4.1+) รองรับแล้ว แต่ต้องตรวจสอบเวอร์ชันที่ใช้จริงเสมอ (เวอร์ชันเก่ากว่านั้น `prefetch_related` จะถูกเพิกเฉยเงียบ ๆ) |
| เหมาะกับงานแบบไหน | Business logic ทั่วไปที่ผลลัพธ์ไม่เยอะ | Batch job, export, data migration, การประมวลผลข้อมูลจำนวนมากแบบ one-pass |

### 707.4 ตัวอย่างจริง: คำนวณสถิติจาก Post นับล้านแถวโดยไม่ให้ Memory พุ่ง

```python
# blog/management/commands/recalculate_stats.py
from django.core.management.base import BaseCommand
from blog.models import Post


class Command(BaseCommand):
    help = 'คำนวณสถิติสรุปจาก Post ทั้งหมด โดยประมวลผลทีละ chunk เพื่อประหยัด memory'

    def handle(self, *args, **options):
        total_views = 0
        processed = 0

        # ใช้ .only() ร่วมกับ .iterator() เสมอเมื่อไม่ต้องการทุก field —
        # ยิ่งดึงข้อมูลต่อแถวน้อย ยิ่งประหยัด memory และ bandwidth ระหว่าง
        # Django กับฐานข้อมูลมากขึ้นไปอีกขั้น (ทบทวน .only()/.defer() จาก Phase 2)
        queryset = Post.objects.filter(is_published=True).only('id', 'view_count').iterator(chunk_size=5000)

        for post in queryset:
            total_views += post.view_count
            processed += 1
            if processed % 100_000 == 0:
                self.stdout.write(f'ประมวลผลแล้ว {processed:,} แถว...')

        self.stdout.write(self.style.SUCCESS(
            f'เสร็จสิ้น: ประมวลผล {processed:,} แถว รวม view_count = {total_views:,}'
        ))
```

การผสาน `.only()` (จำกัด field ที่ดึง) เข้ากับ `.iterator()` (จำกัดจำนวนแถวที่
แคชพร้อมกัน) คือรูปแบบมาตรฐานสำหรับงานประมวลผลข้อมูลขนาดใหญ่แบบ **one-pass**
(อ่านผ่านครั้งเดียวจากต้นจนจบ ไม่ต้องย้อนกลับ) ซึ่งครอบคลุมงานส่วนใหญ่ในเชิง
data processing เช่น การคำนวณสถิติ, การ export (ขั้นตอนที่ 705), หรือ data
migration ขนาดใหญ่

### 707.5 เมื่อไหร่ **ไม่ควร** ใช้ `.iterator()`

- QuerySet ที่มีผลลัพธ์ไม่มาก (หลักร้อย-พันแถว) — ต้นทุนของ server-side cursor
  อาจมากกว่าประโยชน์ที่ได้ ใช้ QuerySet ปกติง่ายกว่าและเร็วพอ ๆ กัน
- โค้ดที่ต้อง iterate ผลลัพธ์เดียวกันหลายรอบ (เช่น ใน template ที่ loop ซ้ำ
  หลายจุด) — `.iterator()` บังคับ query ใหม่ทุกครั้งที่ iterate ซึ่งแพงกว่า
  การแคชไว้ครั้งเดียว
- เมื่อจำเป็นต้องรู้ `len()`/`count()` ของผลลัพธ์ควบคู่กับการวนลูป — ให้เรียก
  `.count()` แยกต่างหาก (เป็นคนละ query) แทนที่จะพยายามนับจากผลลัพธ์ของ
  `.iterator()` โดยตรง

---

## ขั้นตอนที่ 708: เกริ่นการประมวลผลงานหนักแบบ Background (เจาะลึกเต็มด้วย Celery ใน Phase 9)

### 708.1 ข้อจำกัดสุดท้ายที่ยังเหลืออยู่: Request-Response Cycle มีเพดานเวลา

ทุกเทคนิคในขั้นตอนที่ 705-707 (Streaming CSV, `iterator()`) ช่วยแก้ปัญหา
**memory** ได้อย่างสมบูรณ์ แต่ยังมีข้อจำกัดหนึ่งที่แก้ไม่ได้ด้วยเทคนิคเหล่านี้:
**เวลาที่ผู้ใช้ต้องรอ** — ถ้า export ข้อมูล 20 ล้านแถวใช้เวลาประมวลผลจริง 5 นาที
ไม่ว่าจะ stream ให้ดีแค่ไหน ผู้ใช้ก็ยังต้องเปิด connection ค้างไว้ 5 นาที ซึ่งเสี่ยง
ต่อ:

- **Worker Timeout**: Gunicorn/uWSGI worker ที่ถูกจองไว้นานเกินไปโดย request
  เดียวทำให้ worker ตัวอื่นรับ request ใหม่ไม่ได้ (ทบทวนแนวคิด concurrency
  ของ WSGI worker จาก Phase 7)
- **Browser/Proxy Timeout**: Nginx, Cloudflare, หรือแม้แต่ browser เองมักมี
  timeout ของตัวเอง (มักอยู่ที่ 30-60 วินาทีเป็นค่าเริ่มต้น) ที่ตัด connection
  ก่อนงานจะเสร็จ
- **ประสบการณ์ผู้ใช้แย่**: ผู้ใช้ต้องเปิดแท็บค้างไว้ ไม่สามารถปิดหน้าเว็บหรือ
  ทำอย่างอื่นระหว่างรองานเสร็จได้

### 708.2 ทางออก: ย้ายงานหนักออกจาก Request-Response Cycle ไปเป็น "Background Job"

แนวคิดคือ: เมื่อผู้ใช้ขอ export ข้อมูลขนาดใหญ่ **View ไม่ทำงานหนักเอง** แต่แค่
**"จ้าง" งานนี้ให้ไปทำที่อื่น** (a separate worker process) แล้วตอบกลับผู้ใช้
**ทันที** ว่า "รับคำขอแล้ว กำลังเตรียมไฟล์ให้อยู่" จากนั้นเมื่องานเสร็จ ระบบจะแจ้ง
ผู้ใช้ (ผ่านอีเมล, notification, หรือ polling หน้าเว็บ) ว่าไฟล์พร้อมดาวน์โหลดแล้ว

```
รูปแบบเดิม (Synchronous)                    รูปแบบใหม่ (Background Job)
─────────────────────────                   ──────────────────────────
Browser → POST /export/                     Browser → POST /export/
Django  → [ประมวลผล 5 นาที...]              Django  → ส่งงานเข้าคิว (ทันที)
        → รอ ... รอ ... รอ ...                       → ตอบกลับ "รับคำขอแล้ว" (< 1 วินาที)
        → ส่งไฟล์กลับในที่สุด               Worker (แยก process) → ประมวลผล 5 นาที
                                                      → บันทึกไฟล์ + แจ้งผู้ใช้เมื่อเสร็จ
```

### 708.3 เครื่องมือมาตรฐานของอุตสาหกรรม Python/Django: Celery

**Celery** คือ **distributed task queue** ที่ได้รับความนิยมสูงสุดในระบบนิเวศ
Python สำหรับงานประเภทนี้ ทำงานร่วมกับ **message broker** (มักเป็น **Redis**
หรือ **RabbitMQ**) ที่ทำหน้าที่เป็น "คิวงาน" ระหว่าง Django (ผู้ฝากงาน) กับ
**Celery Worker** (process แยกต่างหากที่คอยดึงงานจากคิวมาทำ)

ตัวอย่างหน้าตาของโค้ดที่จะได้เรียนเต็มรูปแบบใน **Phase 9** (แสดงเพื่อให้เห็นภาพ
เท่านั้น ยังไม่ต้องเข้าใจรายละเอียดทั้งหมดตอนนี้):

```python
# blog/tasks.py (ตัวอย่างเบื้องต้น — จะเจาะลึกเต็มรูปแบบใน Phase 9)
from celery import shared_task


@shared_task
def export_all_posts_task(user_id, email):
    """งานหนักที่รันบน Celery Worker แยกจาก Django request-response cycle"""
    # ... โค้ด export ข้อมูลจำนวนมาก (ใช้เทคนิคจากขั้นตอนที่ 705-707 ได้เหมือนเดิม) ...
    # ... บันทึกไฟล์ผลลัพธ์ไว้ แล้วส่งอีเมลแจ้งผู้ใช้เมื่อเสร็จ ...
    pass


# blog/views.py
def request_export_view(request):
    export_all_posts_task.delay(user_id=request.user.id, email=request.user.email)
    return render(request, 'blog/export_requested.html')  # ตอบกลับทันที ไม่ต้องรอ
```

`export_all_posts_task.delay(...)` เพียงบรรทัดเดียวคือสิ่งที่ทำให้ View ตอบกลับ
ผู้ใช้ได้ทันที — งานจริงจะถูกส่งเข้าคิว (ผ่าน Redis/RabbitMQ) แล้ว Celery Worker
process ที่รันแยกต่างหากจะดึงงานนี้ไปทำเบื้องหลัง

### 708.4 สิ่งที่ Phase 9 จะสอนเต็มรูปแบบ (Part นี้แค่เกริ่นให้เห็นภาพ)

| หัวข้อ | รายละเอียดที่จะเจาะลึก |
|---|---|
| การติดตั้งและตั้งค่า Celery + Redis/RabbitMQ | Broker คืออะไร ติดตั้งอย่างไร เชื่อมกับ Django settings อย่างไร |
| `@shared_task` และ `.delay()`/`.apply_async()` | ความแตกต่าง วิธีส่ง argument วิธีตั้งเวลาหน่วง (`countdown`, `eta`) |
| Task Result Backend | เก็บสถานะ/ผลลัพธ์ของ task ไว้ที่ไหน ตรวจสอบสถานะงานที่ยังไม่เสร็จได้อย่างไร |
| Retry และ Error Handling | จัดการเมื่อ task ล้มเหลว การ retry อัตโนมัติ |
| Periodic Task ด้วย Celery Beat | งานที่ต้องรันตามตารางเวลา (เช่น ทุกเที่ยงคืน) |
| Monitoring ด้วย Flower | เครื่องมือ dashboard สำหรับดูสถานะ worker และ task queue แบบ real-time |
| Progress Bar สำหรับงานที่ใช้เวลานาน | แจ้ง progress ระหว่างทางกลับมาที่ frontend ผ่าน polling หรือ WebSocket |

**สิ่งที่ควรจำจาก Part นี้**: เทคนิคทั้งหมดในขั้นตอนที่ 705-707 (Streaming,
`.iterator()`) ยังคง**จำเป็นต้องใช้อยู่แม้จะย้ายไป background job แล้วก็ตาม**
เพราะ Celery Worker ก็เป็น Python process ธรรมดาที่มีข้อจำกัดเรื่อง memory
เหมือนกัน — background job แก้ปัญหาเรื่อง **"เวลารอของผู้ใช้"** แต่ไม่ได้แก้ปัญหา
เรื่อง **"memory ที่ต้องใช้ในการประมวลผล"** ทั้งสองเรื่องเป็นปัญหาคนละมิติที่ต้อง
แก้คู่กันในระบบที่ scale จริง

---

## ขั้นตอนที่ 709: เกริ่นแนวคิด Database Partitioning สำหรับข้อมูลระดับ Enterprise (เจาะลึกเต็มใน Phase 12)

### 709.1 เมื่อ Keyset Pagination และ Index ก็ยังไม่พอ: ปัญหาระดับตาราง ไม่ใช่ระดับ Query

ขั้นตอนที่ 703 แก้ปัญหา pagination ด้วย index ที่ดีและ query ที่ฉลาดขึ้น แต่มี
จุดหนึ่งที่ต้องยอมรับตรง ๆ: **เทคนิคทั้งหมดที่เรียนมาใน Part นี้ยังคงทำงานบน
"ตารางเดียว" ที่ใหญ่ขึ้นเรื่อย ๆ ไม่มีที่สิ้นสุด** เมื่อตารางมีขนาดใหญ่มาก
ระดับ **สิบล้านถึงพันล้านแถว** (ระดับที่บริษัทอย่าง Instagram หรือ Twitter
เจอจริง) ปัญหาใหม่ที่ไม่เกี่ยวกับ query pattern อีกต่อไปจะเกิดขึ้น:

- **ขนาด Index เองก็ใหญ่จนไม่พอดีกับ memory** ทำให้แม้แต่ index seek ก็ต้องอ่าน
  จาก disk บ่อยขึ้น
  ช้าลง
- **การ Backup/Restore ตารางเดียวที่ใหญ่มากใช้เวลานานเกินไป** (หลักชั่วโมงถึง
  หลักวัน)
- **การลบข้อมูลเก่า** (เช่น log ที่เก่ากว่า 1 ปี) ด้วย `DELETE` ธรรมดาบนตารางที่
  มีพันล้านแถวเป็นการทำงานที่หนักมากและอาจล็อกตารางนานจนกระทบระบบ production
- **Maintenance operation** (เช่น `VACUUM` ของ PostgreSQL, การสร้าง index ใหม่)
  ใช้เวลานานขึ้นตามขนาดตารางแบบไม่เป็นเชิงเส้น

### 709.2 แนวคิดของ Database Partitioning โดยสังเขป

**Partitioning** คือการแบ่ง**ตารางเชิงตรรกะหนึ่งตาราง**ออกเป็น**ตารางย่อยหลาย
ตารางทางกายภาพ** (เรียกว่า **partition**) โดยที่แอปพลิเคชัน (Django ORM) ยังคง
มองเห็นและ query ผ่าน **"ตารางเดียว"** เหมือนเดิมทุกประการ — ฐานข้อมูลเป็นผู้
จัดการเบื้องหลังว่า query หนึ่ง ๆ ควรไปอ่าน partition ไหนบ้าง

รูปแบบที่พบบ่อยที่สุดคือ **Range Partitioning ตามช่วงเวลา**:

```
ตาราง blog_post (ตรรกะ)
├── blog_post_2024_q1   (partition: created_at ระหว่าง 2024-01-01 ถึง 2024-03-31)
├── blog_post_2024_q2   (partition: created_at ระหว่าง 2024-04-01 ถึง 2024-06-30)
├── blog_post_2024_q3   (partition: created_at ระหว่าง 2024-07-01 ถึง 2024-09-30)
├── blog_post_2024_q4   (partition: created_at ระหว่าง 2024-10-01 ถึง 2024-12-31)
└── blog_post_2025_q1   (partition: created_at ระหว่าง 2025-01-01 ถึง 2025-03-31)
```

เมื่อ query มีเงื่อนไข `WHERE created_at >= '2025-01-01'` ฐานข้อมูลรู้ทันทีว่า
**ไม่จำเป็นต้องแตะ partition ของปี 2024 เลย** (เทคนิคนี้เรียกว่า **partition
pruning**) ทำให้ query เร็วขึ้นมากเพราะสแกนแค่ partition ที่เกี่ยวข้องเท่านั้น
แทนที่จะสแกนทั้งตารางที่มีข้อมูลหลายปีสะสมกันอยู่

### 709.3 ทำไมเรื่องนี้เกี่ยวกับ Pagination ที่เรียนใน Part นี้

Keyset Pagination ในขั้นตอนที่ 703 ใช้ `created_at` เป็น ordering field — ถ้า
ตารางถูก partition ตาม `created_at` แล้ว query ที่มีเงื่อนไข `WHERE created_at <
X` ของ Keyset Pagination จะได้ประโยชน์จาก partition pruning **โดยอัตโนมัติ**
โดยไม่ต้องแก้โค้ด Django เลยแม้แต่บรรทัดเดียว — นี่คือตัวอย่างที่ชัดเจนว่าทำไม
การออกแบบ pagination ที่ดี (เลือก field ที่มี index และมีความหมายเชิงเวลา)
ตั้งแต่ต้นจึงส่งผลดีต่อเนื่องไปถึงการปรับสเกลระดับฐานข้อมูลในอนาคต

### 709.4 สิ่งที่ Phase 12 จะสอนเต็มรูปแบบ (Part นี้แค่เกริ่นให้เห็นภาพรวม)

| หัวข้อ | รายละเอียดที่จะเจาะลึก |
|---|---|
| Range / List / Hash Partitioning | ความแตกต่างและเลือกใช้แบบไหนเมื่อไหร่ |
| PostgreSQL Declarative Partitioning | การสร้างและจัดการ partitioned table จริงด้วย SQL |
| การผสาน Partitioning เข้ากับ Django ORM | ข้อจำกัดของ Django migration กับตาราง partition, การใช้ third-party package เช่น `django-postgres-extra` |
| Partition Pruning และการอ่าน Query Plan | ตรวจสอบด้วย `EXPLAIN` ว่า query ใช้ประโยชน์จาก partition ได้จริงหรือไม่ |
| กลยุทธ์ Archive ข้อมูลเก่า | การ detach partition เก่าออกไปเก็บแยก (cold storage) โดยไม่กระทบ partition ที่ใช้งานอยู่ |
| Sharding เมื่อ Partitioning เพียงอย่างเดียวไม่พอ | การกระจายข้อมูลข้าม**เครื่องฐานข้อมูลหลายเครื่อง** สำหรับสเกลระดับที่ partitioning ตัวเดียวไม่รองรับไหว |

**สิ่งที่ควรจำจาก Part นี้**: Partitioning เป็นเรื่องของ **database
administration ระดับโครงสร้างพื้นฐาน** ไม่ใช่สิ่งที่ทำผ่าน Django ORM ตรง ๆ
ได้ทั้งหมด — แต่**การออกแบบ query pattern ที่ดีตั้งแต่ระดับแอปพลิเคชัน** (เช่น
Keyset Pagination ที่กรองด้วยเงื่อนไขช่วงเวลาเสมอ) คือสิ่งที่ทำให้ระบบพร้อมสำหรับ
การทำ partitioning ในอนาคตได้อย่างราบรื่น โดยไม่ต้องเขียน query ใหม่ทั้งหมด

---

## ขั้นตอนที่ 710: สรุปและแบบฝึกหัด — Cursor Pagination และ Streaming Export สำหรับ `Post`

### 710.1 ประกอบทุกอย่างเข้าด้วยกัน: `blog/pagination.py` ฉบับสมบูรณ์ของ Part นี้

```python
# core/pagination.py (ฉบับสมบูรณ์ — ใช้ร่วมกันได้ทุก Model ในโปรเจกต์)
import base64
import json
from dataclasses import dataclass
from django.db.models import Q


@dataclass
class KeysetPage:
    object_list: list
    has_next: bool
    has_previous: bool
    next_cursor: str | None
    previous_cursor: str | None


class KeysetPaginator:
    def __init__(self, queryset, ordering_field, tie_field, page_size, descending=True):
        self.queryset = queryset
        self.ordering_field = ordering_field
        self.tie_field = tie_field
        self.page_size = page_size
        self.descending = descending

        order_prefix = '-' if descending else ''
        self.queryset = self.queryset.order_by(
            f'{order_prefix}{ordering_field}', f'{order_prefix}{tie_field}'
        )

    @staticmethod
    def encode_cursor(ordering_value, tie_value) -> str:
        payload = json.dumps([str(ordering_value), tie_value])
        return base64.urlsafe_b64encode(payload.encode()).decode()

    @staticmethod
    def decode_cursor(cursor: str):
        try:
            payload = base64.urlsafe_b64decode(cursor.encode()).decode()
            ordering_value, tie_value = json.loads(payload)
            return ordering_value, tie_value
        except (ValueError, TypeError, UnicodeDecodeError, json.JSONDecodeError):
            return None

    def get_page(self, cursor: str | None) -> KeysetPage:
        qs = self.queryset
        has_previous = cursor is not None

        if cursor:
            decoded = self.decode_cursor(cursor)
            if decoded is not None:
                ordering_value, tie_value = decoded
                comparator = 'lt' if self.descending else 'gt'
                qs = qs.filter(
                    Q(**{f'{self.ordering_field}__{comparator}': ordering_value}) |
                    Q(**{f'{self.ordering_field}': ordering_value}, **{f'{self.tie_field}__{comparator}': tie_value})
                )
            else:
                has_previous = False

        rows = list(qs[: self.page_size + 1])
        has_next = len(rows) > self.page_size
        rows = rows[: self.page_size]

        next_cursor = None
        if has_next and rows:
            last = rows[-1]
            next_cursor = self.encode_cursor(
                getattr(last, self.ordering_field), getattr(last, self.tie_field)
            )

        previous_cursor = None
        if has_previous and rows:
            first = rows[0]
            previous_cursor = self.encode_cursor(
                getattr(first, self.ordering_field), getattr(first, self.tie_field)
            )

        return KeysetPage(
            object_list=rows,
            has_next=has_next,
            has_previous=has_previous,
            next_cursor=next_cursor,
            previous_cursor=previous_cursor,
        )
```

### 710.2 `blog/views.py` ฉบับสมบูรณ์: Infinite Scroll + Streaming Export

```python
# blog/views.py
import csv
from django.http import StreamingHttpResponse
from django.shortcuts import render
from core.pagination import KeysetPaginator
from .models import Post


class Echo:
    def write(self, value):
        return value


def post_list_infinite_keyset_view(request):
    post_list = Post.objects.filter(is_published=True)
    paginator = KeysetPaginator(
        queryset=post_list, ordering_field='created_at', tie_field='id', page_size=20,
    )
    page = paginator.get_page(cursor=request.GET.get('cursor'))

    if request.headers.get('HX-Request') == 'true':
        return render(request, 'blog/partials/post_page_keyset.html', {'page': page})
    return render(request, 'blog/post_list_infinite_keyset.html', {'page': page})


def export_posts_csv_streaming(request):
    def row_generator():
        pseudo_buffer = Echo()
        writer = csv.writer(pseudo_buffer)
        yield writer.writerow(['ID', 'Title', 'Created At', 'View Count'])

        queryset = Post.objects.filter(is_published=True).only(
            'id', 'title', 'created_at', 'view_count'
        ).iterator(chunk_size=2000)

        for post in queryset:
            yield writer.writerow([post.id, post.title, post.created_at, post.view_count])

    response = StreamingHttpResponse(row_generator(), content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="posts_export.csv"'
    return response
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('infinite-keyset/', views.post_list_infinite_keyset_view, name='list_infinite_keyset'),
    path('export/csv/', views.export_posts_csv_streaming, name='export_csv'),
]
```

### 710.3 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ทบทวนและเจาะลึก Django `Paginator`/`Page` object สำหรับ view ที่คืน HTML
  รวมถึง `get_page()`, `get_elided_page_range()`
- ✅ พิสูจน์ด้วย benchmark จริงว่า Offset Pagination ช้าลงตามความลึกของ offset
  แบบ `O(n+m)` และ `SELECT COUNT(*)` ก็มีต้นทุนของตัวเองเช่นกัน
- ✅ เขียน Keyset/Cursor Pagination เองตั้งแต่ต้นด้วย `KeysetPaginator` class
  โดยใช้เทคนิค tuple comparison, tie-breaker field, และ "fetch N+1" แทน
  `COUNT(*)`
- ✅ ประกอบ Keyset Pagination เข้ากับ Infinite Scroll ของ HTMX จาก Part 053
  ได้อย่างไร้รอยต่อ โดยแทบไม่ต้องแก้โค้ด frontend เลย
- ✅ เขียน Streaming CSV Export ด้วย `StreamingHttpResponse` + generator +
  `Echo` class เทคนิคมาตรฐานที่เอกสาร Django แนะนำ
- ✅ เขียน Streaming Excel Export ด้วย `openpyxl` โหมด `write_only` พร้อมเข้าใจ
  ข้อจำกัดที่ต่างจาก CSV
- ✅ ใช้ `StreamingHttpResponse`/`FileResponse` สำหรับไฟล์ขนาดใหญ่ที่มีอยู่แล้ว
  บน disk
- ✅ เข้าใจกลไก `QuerySet.iterator(chunk_size=...)` และเมื่อไหร่ควร/ไม่ควรใช้
  ร่วมกับ `.only()`
- ✅ เข้าใจภาพรวมว่าทำไมงานหนักมากต้องย้ายไป Background Job (Celery, Phase 9)
  และทำไมข้อมูลระดับ Enterprise ต้องพึ่ง Database Partitioning (Phase 12)

### 710.4 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายความแตกต่างระหว่าง `Paginator.page()` กับ `Paginator.get_page()` ได้
- [ ] รัน benchmark เปรียบเทียบเวลาของ `?page=1` กับ `?page=` ที่ offset สูงมาก
      บนข้อมูลจำลองอย่างน้อย 100,000 แถว และอธิบายผลลัพธ์ได้ด้วยคำพูดตัวเอง
- [ ] เขียน `KeysetPaginator` เองได้ (หรืออธิบายกลไก encode/decode cursor และ
      เทคนิค fetch N+1 ได้อย่างถูกต้อง)
- [ ] อธิบายได้ว่าทำไม Keyset Pagination ต้องใช้ tie-breaker field เสมอ
- [ ] ต่อ Keyset Pagination เข้ากับ Infinite Scroll ของ HTMX ได้สำเร็จ
- [ ] เขียน Streaming CSV Export ที่ใช้ `.iterator()` แทนการโหลด QuerySet
      ทั้งหมดเข้า memory ได้
- [ ] อธิบายความแตกต่างระหว่าง `StreamingHttpResponse` กับ `FileResponse` และ
      เลือกใช้ให้ถูกสถานการณ์ได้
- [ ] อธิบายได้ว่า `.iterator()` แลกอะไรไปเพื่อประหยัด memory (ทวนซ้ำไม่ได้,
      `prefetch_related` มีข้อจำกัด)
- [ ] อธิบายภาพรวมได้ว่า Background Job กับ Database Partitioning แก้ปัญหา
      คนละมิติกันอย่างไร

### 710.5 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: สร้างข้อมูลจำลอง `Post` จำนวน 200,000 แถวด้วย
`seed_posts` จากขั้นตอนที่ 701.3 แล้วสร้าง view ที่ใช้ `Paginator` ปกติ (ทบทวน
ขั้นตอนที่ 701) แสดงผลด้วย `get_elided_page_range()` ให้ได้ UI แบบ "1 … 40 41
[42] 43 44 … N"

**แบบฝึกหัดที่ 2 (ประยุกต์)**: ทำตาม `KeysetPaginator` ในขั้นตอนที่ 703-704 ให้
ครบ แล้วเพิ่มเงื่อนไขกรองเพิ่มเติม (เช่น `?min_views=100` ที่กรองเฉพาะบทความที่มี
`view_count >= 100`) เข้าไปใน `post_list_infinite_keyset_view` — ต้องแน่ใจว่า
cursor ที่ encode มายังคง "จำ" เงื่อนไขกรองนี้ไว้ได้ถูกต้องเมื่อกดโหลดหน้าถัดไป
(คำใบ้: ต้องส่ง `min_views` ต่อไปใน URL ของ sentinel ด้วย ไม่ใช่แค่ `cursor`)

**แบบฝึกหัดที่ 3 (Export)**: เขียน view export Excel แบบ streaming (ทบทวน
ขั้นตอนที่ 705.4) ที่มีคอลัมน์เพิ่มเติม เช่น จำนวนคอมเมนต์ของแต่ละบทความ (ใช้
`annotate(Count('comments'))` ร่วมกับ `.iterator()`) แล้ววัด memory ที่ใช้จริง
ระหว่าง export ด้วยเครื่องมืออย่าง `memory_profiler` เปรียบเทียบกับ version ที่
ไม่ใช้ `write_only=True`

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน benchmark เปรียบเทียบเวลาที่ใช้ในการวนลูป
ผ่าน `Post.objects.all()` (1,000,000 แถว) แบบปกติ เทียบกับ
`.iterator(chunk_size=2000)` โดยวัดทั้ง **เวลาที่ใช้** และ **peak memory usage**
(ใช้ `tracemalloc` ของ Python standard library) บันทึกผลเป็นตารางเปรียบเทียบ
และอธิบายว่าทำไมผลลัพธ์เรื่องเวลาอาจไม่ต่างกันมาก แต่ผลลัพธ์เรื่อง memory ต่างกัน
อย่างชัดเจน

### 710.6 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `Paginator` ของ Django เอง หรือเขียน `KeysetPaginator` เองแบบ Part
นี้ หรือใช้ DRF's `CursorPagination` ดี?**
A: ขึ้นกับบริบท — ถ้าเป็น REST API ให้ใช้ DRF's `CursorPagination` เสมอ
(Part 047) เพราะรองรับ feature ครบและผ่านการทดสอบจากคนทั้งอุตสาหกรรมมาแล้ว
ถ้าเป็น server-rendered view (ไม่มี DRF อยู่ในภาพเลย) และข้อมูลไม่เยอะมาก ใช้
`Paginator` ธรรมดาของ Django ก็เพียงพอ ส่วน `KeysetPaginator` ที่เขียนเองใน Part
นี้เหมาะกับกรณี server-rendered view ที่ข้อมูลเยอะจริง ๆ และต้องการ perfomance
คงที่แบบ Keyset โดยไม่อยากติดตั้ง DRF ทั้งชุดเพียงเพื่อ pagination ฟีเจอร์เดียว

**Q: ทำไม cursor string ถึงยาวและอ่านไม่รู้เรื่อง (เช่น
`WyIyMDI2LTAzLTE1VDA5OjEyOjAwIiwgNDE4MjNd`)?**
A: เพราะมันคือ Base64-encoded JSON ของค่าตำแหน่งจริง (`[created_at, id]`) ไม่ใช่
เลขหน้าธรรมดา — ผู้ใช้ (และ client) **ไม่ควรพยายามอ่านหรือประกอบ cursor เอง**
ต้องใช้ค่า `next_cursor`/`previous_cursor` ที่ server สร้างให้เท่านั้นเสมอ ตรงตาม
คำเตือนเดียวกับที่ Part 047 ข้อ 467.2 อธิบายไว้สำหรับ DRF's `CursorPagination`

**Q: `iterator()` ทำให้ query เร็วขึ้นด้วยหรือไม่ ไม่ใช่แค่ประหยัด memory
เท่านั้น?**
A: โดยทั่วไป**ไม่ทำให้เร็วขึ้น** (บางกรณีอาจช้าลงเล็กน้อยด้วยซ้ำเพราะ overhead
ของการเปิด server-side cursor) — จุดประสงค์หลักของ `iterator()` คือ **ควบคุม
memory ให้คงที่** ไม่ใช่ทำให้ query เร็วขึ้น ส่วนความเร็วของตัว query เองขึ้นกับ
index และโครงสร้าง `WHERE`/`ORDER BY` เหมือนที่อธิบายในขั้นตอนที่ 702-703

**Q: ถ้าโปรเจกต์เล็ก ๆ ที่มีข้อมูลไม่เกินหมื่นแถว จำเป็นต้องทำทุกเทคนิคใน Part
นี้ตั้งแต่แรกไหม?**
A: ไม่จำเป็น — นี่คือหลักการ **"อย่า optimize ก่อนที่จะรู้ว่าจำเป็น" (avoid
premature optimization)** ที่สำคัญมากในงานวิศวกรรมซอฟต์แวร์ ใช้ `Paginator`
ธรรมดาและ QuerySet ปกติไปก่อนสำหรับข้อมูลขนาดเล็ก-กลาง แล้วค่อยเปลี่ยนไปใช้
Keyset Pagination/Streaming/`.iterator()` เมื่อวัดผลจริง (ผ่าน benchmark หรือ
monitoring ใน production) แล้วพบว่าเป็นคอขวดจริง ๆ — สิ่งสำคัญที่สุดคือ**เข้าใจ
ว่าเทคนิคเหล่านี้มีอยู่และรู้ว่าจะหยิบมาใช้เมื่อไหร่** มากกว่าการใช้ทุกอย่างตั้งแต่
วันแรกโดยไม่จำเป็น

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 072: Load Testing และ Scalability Testing** ซึ่งเป็น Part สุดท้ายของ
Phase 8 จะพาไปทดสอบว่าระบบที่ optimize มาตลอดทั้ง Phase (profiling, query
optimization, caching, indexing, pagination) รองรับ load จริงได้แค่ไหน — เขียน
load test ด้วย Locust และ k6, หาจุดแตกหัก (breaking point) ของระบบ, ปรับจูน
ขนาด database connection pool และจำนวน Gunicorn worker ให้เหมาะสม, เปรียบเทียบ
ผล load test ก่อน/หลังเปิดใช้ caching จาก Part 068-069 และปิดท้ายด้วยการสรุป
Phase 8 ทั้งหมด

เตรียมข้อมูลจำลอง `Post` จำนวนมากที่สร้างไว้ในขั้นตอนที่ 701.3 ของ Part นี้ไว้ให้
พร้อม เพราะ Part 072 จะใช้ชุดข้อมูลเดียวกันนี้ต่อในการยิง load test จริง!
