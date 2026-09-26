# Part 047: Filtering, Searching, Pagination ใน DRF

> **ขั้นตอนที่ 461-470 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: ทำให้ endpoint `/api/posts/` ของ `blog` API ที่สร้างมาตั้งแต่
> Part 039-046 รองรับการทำงานกับข้อมูลจำนวนมากแบบมืออาชีพ — กรองข้อมูลด้วย
> `django-filter` และ `DjangoFilterBackend`, ค้นหาข้อความในหลาย field พร้อมกันด้วย
> `SearchFilter`, เรียงลำดับผลลัพธ์ตาม query param ด้วย `OrderingFilter`, เขียน
> `FilterSet` class เองสำหรับกรองแบบช่วง (ช่วงวันที่ ช่วงราคา boolean), ผสานทั้งสาม
> filter backend เข้าด้วยกันในคลาสเดียว, เจาะลึก `PageNumberPagination`,
> `LimitOffsetPagination`, `CursorPagination` ทั้งการใช้งานและผลกระทบด้าน performance
> เมื่อข้อมูลมีหลักล้าน record, เขียน Custom Pagination Class ปรับรูปแบบ response เอง
> และปิดท้ายด้วยการประกอบทุกอย่างเข้ากับ Blog API ให้สมบูรณ์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 461: ติดตั้งและใช้ `django-filter` ผ่าน `DjangoFilterBackend`
- ขั้นตอนที่ 462: `SearchFilter` — ค้นหาข้อความในหลาย field พร้อมกัน
- ขั้นตอนที่ 463: `OrderingFilter` — เรียงลำดับผ่าน query param
- ขั้นตอนที่ 464: เขียน `FilterSet` class เองแบบละเอียด (ช่วงวันที่ ช่วงราคา boolean)
- ขั้นตอนที่ 465: ผสานหลาย filter backend พร้อมกัน (`filter_backends` list)
- ขั้นตอนที่ 466: Pagination Class — `PageNumberPagination` เจาะลึก
- ขั้นตอนที่ 467: `LimitOffsetPagination` และ `CursorPagination` เปรียบเทียบ
- ขั้นตอนที่ 468: เขียน Custom Pagination Class เอง (ปรับ response format)
- ขั้นตอนที่ 469: ผลกระทบด้าน Performance ของการเลือก Pagination กับข้อมูลขนาดใหญ่
- ขั้นตอนที่ 470: สรุปและแบบฝึกหัด — Blog API พร้อม filter/search/pagination เต็มรูปแบบ

---

## ขั้นตอนที่ 461: ติดตั้งและใช้ `django-filter` ผ่าน `DjangoFilterBackend`

### 461.1 ปัญหาที่เกิดขึ้นเมื่อ API มีข้อมูลจำนวนมาก

Blog API ของเราตั้งแต่ Part 044 คืนบทความ**ทั้งหมด**ทุกครั้งที่เรียก `GET /api/posts/`
ตราบใดที่ `PostViewSet.get_queryset()` ไม่ได้กรองอะไรเพิ่ม ลองนึกภาพว่าฐานข้อมูลมี
บทความ 50,000 บทความ — client ที่ต้องการแค่บทความในหมวด "Django" ที่เผยแพร่แล้ว
จะต้องดาวน์โหลดข้อมูลทั้งหมด 50,000 รายการมาก่อน แล้วค่อยกรองเอาเองฝั่ง client ซึ่ง
สิ้นเปลือง bandwidth และ memory มหาศาลโดยไม่จำเป็น

สิ่งที่ต้องการจริง ๆ คือให้ client สั่งกรองข้อมูลผ่าน **query parameter** ได้ตรง ๆ เช่น

```
GET /api/posts/?is_published=true&category=django
```

แล้วให้ **ฐานข้อมูล** เป็นคนกรองข้อมูลก่อนส่งกลับมา (WHERE clause) แทนที่จะส่งข้อมูล
ทั้งหมดมาให้ Python กรองทีหลัง — นี่คือหน้าที่ของ **Filter Backend** ใน DRF

### 461.2 ติดตั้ง `django-filter`

`django-filter` เป็น third-party package ที่ได้รับความนิยมสูงที่สุดสำหรับงาน filtering
ใน DRF (ไม่ใช่ core ของ DRF เอง แต่ DRF ออกแบบ interface ไว้ให้ผสานกันได้ทันที)

```bash
pip install django-filter
pip freeze > requirements.txt
```

เพิ่มเข้า `INSTALLED_APPS`:

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
    'blog',
]
```

### 461.3 ตั้งค่า `DjangoFilterBackend` เป็นค่าเริ่มต้นของทั้งโปรเจกต์

```python
# config/settings.py
REST_FRAMEWORK = {
    # ... (DEFAULT_AUTHENTICATION_CLASSES, DEFAULT_PERMISSION_CLASSES จาก Part 045-046)
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
}
```

การตั้งค่านี้ทำให้**ทุก** `GenericAPIView`/`ViewSet` ในโปรเจกต์มี `DjangoFilterBackend`
พร้อมใช้งานโดยอัตโนมัติ แต่ backend นี้จะ**ไม่ทำอะไรเลย** จนกว่า View จะประกาศ
`filterset_fields` หรือ `filterset_class` ไว้ — ปลอดภัยที่จะตั้งเป็นค่า default ระดับ
โปรเจกต์เพราะไม่กระทบ View ที่ไม่ได้ใช้งาน

### 461.4 วิธีที่เร็วที่สุด: `filterset_fields`

```python
# blog/viewsets.py
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'
    filterset_fields = ['is_published', 'category', 'author']
```

แค่บรรทัดเดียว `filterset_fields = [...]` ทำให้ endpoint รองรับการกรองแบบ exact-match
บนทุก field ที่ระบุทันที:

```bash
curl "http://127.0.0.1:8000/api/posts/?is_published=true"
curl "http://127.0.0.1:8000/api/posts/?category=3"
curl "http://127.0.0.1:8000/api/posts/?is_published=true&category=3"
```

สังเกตว่าเมื่อระบุหลาย query parameter พร้อมกัน `DjangoFilterBackend` จะ**รวมเงื่อนไข
ด้วย AND เสมอ** (บทความต้อง `is_published=true` **และ** อยู่ใน `category=3`)

### 461.5 กำหนด lookup แยกตาม field ด้วย dict syntax

ถ้าต้องการมากกว่า exact-match (เช่น กรองวันที่แบบ "มากกว่าหรือเท่ากับ") โดยยังไม่อยาก
เขียน `FilterSet` class เต็มรูปแบบ (เจาะลึกในขั้นตอนที่ 464) ใช้ dict แทน list ได้:

```python
# blog/viewsets.py
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'
    filterset_fields = {
        'is_published': ['exact'],
        'category': ['exact'],
        'created_at': ['exact', 'gte', 'lte', 'year', 'month'],
    }
```

```bash
# บทความที่สร้างตั้งแต่ 2026-01-01 เป็นต้นไป
curl "http://127.0.0.1:8000/api/posts/?created_at__gte=2026-01-01"

# บทความที่สร้างในปี 2026 เดือน 3
curl "http://127.0.0.1:8000/api/posts/?created_at__year=2026&created_at__month=3"
```

DRF จะสร้าง query parameter ชื่อ `<field>__<lookup>` ให้อัตโนมัติตามที่ระบุใน dict —
เบื้องหลังคือการ map ไปยัง Django ORM lookup ตรง ๆ (`created_at__gte`,
`created_at__year` เหมือนที่เขียนใน `.filter()` ปกติที่เรียนมาตั้งแต่ Phase 2)

### 461.6 ตาราง lookup ที่ใช้บ่อยที่สุดตามชนิด field

| ชนิด Field | Lookup ที่ใช้ได้ | ตัวอย่าง query param |
|---|---|---|
| `CharField`/`SlugField` | `exact`, `iexact`, `contains`, `icontains`, `startswith` | `?title__icontains=django` |
| `BooleanField` | `exact` | `?is_published=true` |
| `DateTimeField`/`DateField` | `exact`, `gt`, `gte`, `lt`, `lte`, `year`, `month`, `day`, `date` | `?created_at__gte=2026-01-01` |
| `IntegerField`/`DecimalField` | `exact`, `gt`, `gte`, `lt`, `lte`, `range` | `?price__gte=100&price__lte=500` |
| `ForeignKey` | `exact` (ใช้ pk), หรือ `<related_field>` เจาะลึก relation | `?category=3` หรือ `?category__slug=django` |
| `ManyToManyField` | `exact` (ใช้ pk ของแถวเดียว — ต้องระวังเรื่อง duplicate ผลลัพธ์) | `?tags=5` |

**ข้อควรระวังสำคัญ**: การกรองผ่าน `ManyToManyField` โดยตรง (เช่น `?tags=5`) อาจทำให้
ผลลัพธ์มี record ซ้ำถ้า queryset join กับตารางกลางแล้วไม่ `.distinct()` — ประเด็นนี้จะ
สำคัญมากขึ้นเมื่อเขียน custom filter สำหรับ tags ในขั้นตอนที่ 464.5

---

## ขั้นตอนที่ 462: `SearchFilter` — ค้นหาข้อความในหลาย field พร้อมกัน

### 462.1 ความแตกต่างระหว่าง "filter" กับ "search"

`filterset_fields` ในขั้นตอนที่แล้วเหมาะกับการกรองแบบ**เจาะจงค่าที่รู้แน่นอน** เช่น
`is_published=true` แต่เมื่อผู้ใช้พิมพ์คำค้นหาอิสระ (เช่น กล่องค้นหาบนหน้าเว็บ) เข้ามา
เช่น `"django orm"` เราต้องการให้ระบบค้นหาคำนี้**ในหลาย field พร้อมกัน** (ทั้ง `title`
และ `content`) แบบ case-insensitive substring match — นี่คือหน้าที่ของ `SearchFilter`
ซึ่งเป็น backend ที่**มากับ DRF core อยู่แล้ว** ไม่ต้องติดตั้ง package เพิ่ม

### 462.2 เพิ่ม `SearchFilter` และประกาศ `search_fields`

```python
# blog/viewsets.py
from rest_framework import viewsets
from rest_framework.filters import SearchFilter
from django_filters.rest_framework import DjangoFilterBackend
from .models import Post
from .serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'
    filter_backends = [DjangoFilterBackend, SearchFilter]
    filterset_fields = ['is_published', 'category']
    search_fields = ['title', 'content', 'author__username']
```

**ข้อสังเกตสำคัญ**: เมื่อประกาศ `filter_backends` เอง (แทนที่จะพึ่ง
`DEFAULT_FILTER_BACKENDS` จากขั้นตอนที่ 461.3) ต้องระบุ `DjangoFilterBackend` ซ้ำในนี้
ด้วย ไม่เช่นนั้น `filterset_fields` จากขั้นตอนที่แล้วจะหยุดทำงานทันที เพราะ `filter_backends`
ระดับ View **override** ค่า default ทั้งหมด ไม่ใช่ merge เข้าด้วยกัน (เจาะลึกเรื่องนี้เต็ม ๆ
ในขั้นตอนที่ 465)

### 462.3 ทดสอบการค้นหา

```bash
curl "http://127.0.0.1:8000/api/posts/?search=django"
```

DRF จะสร้าง SQL ประมาณนี้ (ย่อ):

```sql
SELECT * FROM blog_post
WHERE UPPER(title) LIKE UPPER('%django%')
   OR UPPER(content) LIKE UPPER('%django%')
   OR UPPER(author.username) LIKE UPPER('%django%');
```

สังเกตว่า `search_fields` ที่มี `__` (เช่น `author__username`) ทำให้ `SearchFilter`
เดินทาง relation ข้าม table ได้เหมือนกับ Django ORM lookup ปกติ — ค้นคำเดียวแต่ครอบคลุม
ทั้ง field ของ `Post` เองและ field ของ model ที่เกี่ยวข้อง

### 462.4 Prefix พิเศษที่ปรับพฤติกรรมการค้นหาต่อ field

```python
search_fields = ['^title', '=slug', 'content', '@content', '$title']
```

| Prefix | ความหมาย | เทียบเท่า ORM lookup | เงื่อนไขการใช้ |
|---|---|---|---|
| (ไม่มี prefix) | ค้นหาแบบ substring (contains) | `icontains` | ค่าเริ่มต้น ใช้ได้ทุกฐานข้อมูล |
| `^` | ค้นหาแบบขึ้นต้นด้วยคำนี้เท่านั้น | `istartswith` | เร็วกว่า `icontains` เพราะใช้ index ได้ในบางฐานข้อมูล |
| `=` | ค้นหาแบบตรงทั้งคำ (ไม่สนตัวพิมพ์ใหญ่-เล็ก) | `iexact` | เหมาะกับ field ที่เป็นรหัส/slug |
| `@` | Full-text search | `search` (Postgres เท่านั้น) | ต้องใช้ PostgreSQL + `django.contrib.postgres` |
| `$` | ค้นหาแบบ regular expression | `iregex` | ยืดหยุ่นสุด แต่ช้าที่สุด และเสี่ยง ReDoS ถ้ารับ pattern จาก user ตรง ๆ |

**คำแนะนำระดับมืออาชีพ**: อย่าใช้ `$` (regex) กับ field ที่รับค่าค้นหาจากผู้ใช้ทั่วไป
โดยไม่ผ่านการตรวจสอบ เพราะ query parameter `search` มาจาก client โดยตรง หากปล่อยให้
ค่านั้นกลายเป็น regex pattern ที่ซับซ้อนเกินไป อาจทำให้ database ทำงานหนักผิดปกติ
(คล้ายกับปัญหา ReDoS ฝั่ง backend)

### 462.5 `search_fields = '__all__'`? ทำไมไม่ควรทำ

ต่างจาก `ordering_fields` ที่ DRF อนุญาตให้ตั้ง `'__all__'` ได้ (ขั้นตอนที่ 463.4)
`search_fields` **ไม่มี** shortcut แบบนี้ให้ใช้ — ต้องระบุ field ทีละตัวเสมอ ซึ่งจริง ๆ
แล้วเป็นข้อดี เพราะการค้นหาข้าม field ที่ไม่เกี่ยวข้อง (เช่น `id`, `slug` แบบ UUID)
ไม่มีประโยชน์และสิ้นเปลือง query เปล่า ๆ ควรเลือกเฉพาะ field ที่เป็นข้อความที่มนุษย์
อ่านแล้วเข้าใจ (`title`, `content`, `author__username`) เท่านั้น

---

## ขั้นตอนที่ 463: `OrderingFilter` — เรียงลำดับผ่าน query param

### 463.1 ปัญหาที่ `OrderingFilter` แก้: การเรียงลำดับที่ควบคุมโดย client

`Post.Meta.ordering = ['-created_at']` (ตั้งไว้ตั้งแต่ Part 012) กำหนดลำดับ**เริ่มต้น**
ของ queryset แต่ในงานจริง client มักต้องการควบคุมลำดับเอง เช่น หน้าเว็บมีปุ่ม
"เรียงตามชื่อ A-Z" หรือ "เรียงตามวันที่เก่าสุดก่อน" — การ hardcode `.order_by()` ไว้ใน
`get_queryset()` ทำแบบนี้ไม่ได้ จึงต้องใช้ `OrderingFilter` ที่อ่านค่าจาก query
parameter `ordering` โดยตรง

### 463.2 เพิ่ม `OrderingFilter` และประกาศ `ordering_fields`

```python
# blog/viewsets.py
from rest_framework.filters import OrderingFilter, SearchFilter
from django_filters.rest_framework import DjangoFilterBackend


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['is_published', 'category']
    search_fields = ['title', 'content', 'author__username']
    ordering_fields = ['created_at', 'updated_at', 'title']
    ordering = ['-created_at']   # ใช้เมื่อ client ไม่ส่ง query param 'ordering' มา
```

### 463.3 ทดสอบการเรียงลำดับ

```bash
# เรียงจากเก่าไปใหม่ (ascending)
curl "http://127.0.0.1:8000/api/posts/?ordering=created_at"

# เรียงจากใหม่ไปเก่า (descending) — เครื่องหมาย - นำหน้า
curl "http://127.0.0.1:8000/api/posts/?ordering=-created_at"

# เรียงตามชื่อ แล้วถ้าชื่อซ้ำกันให้เรียงตามวันที่อัปเดตล่าสุด
curl "http://127.0.0.1:8000/api/posts/?ordering=title,-updated_at"
```

`OrderingFilter` แปลง query param `ordering` เป็น `.order_by(*fields)` ตรง ๆ — comma
คั่นหลาย field ได้ และเครื่องหมาย `-` นำหน้าหมายถึง descending ตรงตาม syntax ของ
Django ORM ทุกประการที่เรียนมาตั้งแต่ Phase 2

### 463.4 ทำไม `ordering_fields` ต้องระบุ whitelist เสมอ (ห้าม expose ทุก field มั่ว ๆ)

```python
ordering_fields = '__all__'   # อนุญาตให้เรียงตาม field ใดก็ได้ของ serializer — อันตราย!
```

ถ้าตั้งเป็น `'__all__'` ผู้ใช้จะสามารถส่ง `?ordering=` เป็นชื่อ field ใดก็ได้ที่มีอยู่ใน
**serializer** (ไม่ใช่ model) รวมถึง field ที่อาจไม่มี index ในฐานข้อมูล เช่น
`content` (เป็น `TextField` ขนาดใหญ่) การเรียงลำดับ text field ขนาดใหญ่โดยไม่มี index
ทำให้ database ต้อง sort ข้อมูลทั้งหมดแบบ full table scan ซึ่งช้ามากเมื่อข้อมูลเยอะ —
**คำแนะนำระดับมืออาชีพ**: ระบุ `ordering_fields` เป็น whitelist ที่ชัดเจนเสมอ จำกัดไว้
เฉพาะ field ที่มี index (เช่น `created_at`, `id`, `slug`) หรือ field สั้น ๆ ที่ sort ได้เร็ว
(`title`) เท่านั้น

### 463.5 ตารางสรุปพฤติกรรมของ `OrderingFilter`

| สถานการณ์ | ผลลัพธ์ |
|---|---|
| ไม่ส่ง query param `ordering` มาเลย | ใช้ `view.ordering` (ถ้ามี) หรือ `Meta.ordering` ของ model |
| ส่ง `?ordering=price` แต่ `price` ไม่อยู่ใน `ordering_fields` | DRF **เพิกเฉย** ค่านี้เงียบ ๆ ไม่ error, กลับไปใช้ ordering ค่าเริ่มต้นแทน |
| ส่ง `?ordering=nonexistent_field` ที่ไม่มีจริงในฐานข้อมูล | ถ้าผ่าน whitelist ของ `ordering_fields` แล้ว จะเกิด `FieldError` เพราะ ORM หา field ไม่เจอ — จึงต้องตรวจสอบ whitelist ให้ตรงกับ field ที่มีจริงเสมอ |
| ส่ง `?ordering=title,-created_at` | เรียงหลาย field ตามลำดับที่ระบุ (comma-separated) |

---

## ขั้นตอนที่ 464: เขียน `FilterSet` class เองแบบละเอียด

### 464.1 ข้อจำกัดของ `filterset_fields` ที่ทำให้ต้องเขียน `FilterSet` เอง

`filterset_fields` (ขั้นตอนที่ 461) สร้าง query parameter ชื่อตรงกับ field + lookup
เสมอ (เช่น `created_at__gte`) ซึ่งมีข้อจำกัดสำคัญ 3 ข้อ:

1. **ตั้งชื่อ query parameter เองไม่ได้** — ต้องใช้ `created_at__gte` เท่านั้น เขียน
   `?created_after=2026-01-01` ที่อ่านง่ายกว่าไม่ได้
2. **ทำ logic ซับซ้อนไม่ได้** — เช่น กรอง tags ผ่านชื่อ (ไม่ใช่ pk) พร้อม `.distinct()`
3. **validate ค่าที่รับเข้ามาเองไม่ได้** — เช่น ถ้า `price_min > price_max` ควร error
   แต่ `filterset_fields` ไม่มีจุดให้ใส่ validation logic นี้

ทางออกคือเขียน **`FilterSet` class เอง** ซึ่งให้ควบคุมได้ทุกมิติเหมือนการเขียน
`Serializer` เองแทนที่จะพึ่ง `ModelSerializer` อัตโนมัติทั้งหมด

### 464.2 ตัวอย่างฐาน: โมเดล `Product` สำหรับสาธิตการกรองแบบช่วง (Range Filtering)

เพื่อสาธิตเทคนิคการกรองช่วงราคาและ boolean อย่างชัดเจนที่สุด (ก่อนนำไปประยุกต์กับ
`Post` ในขั้นตอนที่ 464.5) เราจะเพิ่มแอป `shop` เล็ก ๆ ที่มีโมเดล `Product`:

```python
# shop/models.py
from django.db import models


class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    in_stock = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.name
```

```python
# shop/serializers.py
from rest_framework import serializers
from .models import Product


class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'in_stock', 'created_at']
```

### 464.3 เขียน `ProductFilter` ด้วย `django_filters.FilterSet`

```python
# shop/filters.py
import django_filters
from .models import Product


class ProductFilter(django_filters.FilterSet):
    # NumberFilter + lookup_expr='gte'/'lte' -> สร้าง query param ชื่อเองได้อิสระ
    price_min = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    price_max = django_filters.NumberFilter(field_name='price', lookup_expr='lte')

    # DateFilter + lookup_expr เจาะเฉพาะส่วน "วันที่" ของ DateTimeField
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='date__gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='date__lte')

    # BooleanFilter รับค่า true/false/1/0 จาก query string แล้วแปลงเป็น Python bool ให้อัตโนมัติ
    in_stock = django_filters.BooleanFilter(field_name='in_stock')

    class Meta:
        model = Product
        fields = ['in_stock']   # field พื้นฐานที่ไม่ต้อง custom เพิ่ม ยังประกาศใน Meta ได้ตามปกติ

    def qs_validate(self):
        """ตัวอย่างการ validate ข้าม field: price_min ต้องไม่มากกว่า price_max"""
        price_min = self.form.cleaned_data.get('price_min')
        price_max = self.form.cleaned_data.get('price_max')
        if price_min is not None and price_max is not None and price_min > price_max:
            raise django_filters.exceptions.FieldLookupError(
                'price_min ต้องไม่มากกว่า price_max'
            )
```

```python
# shop/viewsets.py
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer
from .filters import ProductFilter


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filterset_class = ProductFilter   # ใช้ FilterSet class แทน filterset_fields
```

**ข้อสังเกตสำคัญ**: เมื่อระบุ `filterset_class` แล้ว **ไม่ต้อง**ระบุ `filterset_fields`
อีก — สองตัวนี้ทำหน้าที่เดียวกันแค่คนละระดับความละเอียด (`filterset_fields` คือ
shortcut ที่ DRF สร้าง `FilterSet` ให้อัตโนมัติเบื้องหลัง ส่วน `filterset_class` คือ
เขียน `FilterSet` เองตรง ๆ)

### 464.4 ทดสอบการกรองแบบช่วง

```bash
# สินค้าราคา 100-500 บาท ที่ยังมีในสต็อก
curl "http://127.0.0.1:8000/api/products/?price_min=100&price_max=500&in_stock=true"

# สินค้าที่เพิ่มเข้าระบบระหว่าง 2026-01-01 ถึง 2026-03-31
curl "http://127.0.0.1:8000/api/products/?created_after=2026-01-01&created_before=2026-03-31"

# ผสมทั้งช่วงราคาและช่วงวันที่พร้อมกัน (AND เสมอ)
curl "http://127.0.0.1:8000/api/products/?price_min=100&created_after=2026-01-01"
```

### 464.5 ประยุกต์เทคนิคเดียวกันกับ `Post`: `PostFilter` เต็มรูปแบบ

```python
# blog/filters.py
import django_filters
from .models import Post


class PostFilter(django_filters.FilterSet):
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='date__gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='date__lte')
    category = django_filters.CharFilter(field_name='category__slug', lookup_expr='iexact')
    tag = django_filters.CharFilter(method='filter_by_tag')
    author = django_filters.CharFilter(field_name='author__username', lookup_expr='iexact')

    class Meta:
        model = Post
        fields = ['is_published']

    def filter_by_tag(self, queryset, name, value):
        """
        Custom filter method: รับค่าจาก query param 'tag' (ชื่อ method ต้องตรงกับ
        method= ที่ระบุไว้ข้างบน) กรองผ่าน ManyToManyField ด้วยชื่อ tag (slug)
        แทนที่จะใช้ pk ตรง ๆ อย่างในขั้นตอนที่ 461.6 พร้อม .distinct() ป้องกัน
        record ซ้ำจากการ join ตารางกลางของ ManyToManyField
        """
        return queryset.filter(tags__slug__iexact=value).distinct()
```

```python
# blog/viewsets.py
from .filters import PostFilter


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'
    filterset_class = PostFilter
```

```bash
# บทความหมวด django ที่เผยแพร่แล้ว สร้างตั้งแต่ต้นปี 2026 มี tag ชื่อ "orm"
curl "http://127.0.0.1:8000/api/posts/?category=django&is_published=true&created_after=2026-01-01&tag=orm"
```

### 464.6 `django_filters.rest_framework.FilterSet` vs `django_filters.FilterSet`

```python
import django_filters
from django_filters import rest_framework as filters
```

ทั้งสองแบบใช้ `django_filters.FilterSet` เป็นฐานเหมือนกัน แต่
`django_filters.rest_framework.FilterSet` (import ผ่าน `from django_filters import
rest_framework as filters` แล้วใช้ `filters.FilterSet`) เพิ่มการรองรับ error ให้ตรงกับ
รูปแบบของ DRF (คืนเป็น `ValidationError` ของ DRF แทนที่จะเป็น error แบบ Django Form
ธรรมดา) **คำแนะนำระดับมืออาชีพ**: เมื่อใช้กับ DRF ให้ import จาก
`django_filters.rest_framework` เสมอ (ไม่ใช่ `django_filters` เปล่า ๆ) เพื่อให้ error
message ที่ client ได้รับมีรูปแบบ JSON ตรงกับ error อื่น ๆ ของ API ทั้งระบบ

---

## ขั้นตอนที่ 465: ผสานหลาย filter backend พร้อมกัน (`filter_backends` list)

### 465.1 `filter_backends` คือ list ที่ DRF ไล่ประมวลผลทีละตัวตามลำดับ

```python
# blog/viewsets.py
from rest_framework import viewsets
from rest_framework.filters import OrderingFilter, SearchFilter
from django_filters.rest_framework import DjangoFilterBackend
from .models import Post
from .serializers import PostSerializer
from .filters import PostFilter


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'

    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = PostFilter
    search_fields = ['title', 'content', 'author__username']
    ordering_fields = ['created_at', 'updated_at', 'title']
    ordering = ['-created_at']

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

DRF เรียก `filter_queryset()` ของ `GenericAPIView` ซึ่งวน loop ผ่านทุก backend ใน
`filter_backends` ตามลำดับที่ประกาศ แต่ละ backend รับ queryset ที่ backend ก่อนหน้า
กรองมาแล้ว แล้วกรองซ้อนต่อ (queryset ไหลผ่านทุก backend แบบ pipeline):

```
queryset เริ่มต้น (จาก get_queryset())
      │
      ▼
DjangoFilterBackend   (กรองตาม is_published, category, created_after, tag, ...)
      │
      ▼
SearchFilter          (กรองต่อด้วยคำค้นหาจาก ?search=...)
      │
      ▼
OrderingFilter        (เรียงลำดับผลลัพธ์สุดท้ายตาม ?ordering=...)
      │
      ▼
queryset สุดท้าย -> ส่งต่อให้ Pagination (ขั้นตอนที่ 466) แล้วค่อย serialize
```

### 465.2 เพราะเหตุใด `filter_backends` ระดับ View จึง "แทนที่" ไม่ใช่ "รวม"

```python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
}
```

```python
class PostViewSet(viewsets.ModelViewSet):
    filter_backends = [SearchFilter]   # ระบุเองในคลาสนี้
```

ถ้า `PostViewSet` ประกาศ `filter_backends = [SearchFilter]` เอง `DjangoFilterBackend`
จาก setting ระดับโปรเจกต์จะ**หายไปทันทีเฉพาะใน ViewSet นี้** เพราะ `filter_backends`
เป็น class attribute ธรรมดาที่ override ค่า default แบบเดียวกับ `permission_classes`
(ทบทวนจาก Part 045) — นี่คือสาเหตุที่ต้องระบุ backend ทุกตัวที่ต้องการใช้งานจริงให้ครบ
ในทุก View ที่ override attribute นี้เอง ไม่ใช่แค่ตัวที่เพิ่มใหม่

### 465.3 ทดสอบผสานทั้ง 3 backend พร้อมกันในคำขอเดียว

```bash
curl "http://127.0.0.1:8000/api/posts/?is_published=true&category=django&search=orm&ordering=-created_at"
```

Query parameter ทั้งหมดในตัวอย่างนี้ทำงานพร้อมกัน:

| Query Parameter | Backend ที่จัดการ | ผลลัพธ์ |
|---|---|---|
| `is_published=true` | `DjangoFilterBackend` (ผ่าน `PostFilter.Meta.fields`) | กรองเฉพาะบทความที่เผยแพร่แล้ว |
| `category=django` | `DjangoFilterBackend` (ผ่าน `PostFilter.category`) | กรองเฉพาะหมวด django |
| `search=orm` | `SearchFilter` | ค้นคำ "orm" ใน title/content/author |
| `ordering=-created_at` | `OrderingFilter` | เรียงใหม่สุดก่อน |

### 465.4 ตารางสรุปเมื่อไหร่ควรใช้ backend ไหน

| ต้องการทำอะไร | Backend ที่เหมาะสม |
|---|---|
| กรองด้วยค่าที่ตรงเป๊ะ (exact match) | `DjangoFilterBackend` + `filterset_fields` |
| กรองแบบช่วง (ราคา วันที่) หรือ logic ซับซ้อน | `DjangoFilterBackend` + `FilterSet` class เอง |
| ค้นคำอิสระในหลาย text field | `SearchFilter` |
| ให้ client เลือกลำดับการแสดงผลเอง | `OrderingFilter` |
| ทั้งสามอย่างพร้อมกันใน endpoint เดียว | ใส่ทั้งสาม backend ใน `filter_backends` list เดียวกัน |

### 465.5 ตั้งค่า filter backend ทั้งหมดเป็นค่า default ระดับโปรเจกต์ได้เช่นกัน

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

ถ้าตั้งค่านี้ระดับโปรเจกต์ `PostViewSet` ก็**ไม่จำเป็นต้องประกาศ `filter_backends`
เองเลย** เหลือแค่ประกาศ `filterset_class`, `search_fields`, `ordering_fields` พอ —
เหมาะกับโปรเจกต์ที่ทุก (หรือเกือบทุก) ViewSet ต้องการทั้งสาม backend นี้เหมือนกันหมด

---

## ขั้นตอนที่ 466: Pagination Class — `PageNumberPagination` เจาะลึก

### 466.1 ปัญหาที่ Pagination แก้: ป้องกันการส่งข้อมูลทั้งหมดในคำขอเดียว

แม้จะกรองด้วย filter/search แล้ว ผลลัพธ์ก็ยังอาจมีหลายพันหลายหมื่น record ได้อยู่ดี
(เช่น `?is_published=true` เพียงเงื่อนไขเดียวอาจตรงกับ 30,000 บทความ) การส่งข้อมูล
ทั้งหมดในคำขอเดียวทำให้ response ใหญ่เกินไป โหลดช้า และเสี่ยงทำให้ server ใช้ memory
สูงผิดปกติ **Pagination** คือการแบ่งผลลัพธ์ออกเป็น "หน้า" (page) ย่อย ๆ

### 466.2 ตั้งค่า `PageNumberPagination` เป็นค่า default ระดับโปรเจกต์

```python
# config/settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

เมื่อตั้งค่านี้แล้ว **ทุก** endpoint ที่คืนผลลัพธ์เป็น list (ผ่าน `ListModelMixin`/
`list()`) จะถูกแบ่งหน้าอัตโนมัติทันที โดยไม่ต้องแก้โค้ด ViewSet ใด ๆ เพิ่มเลย

### 466.3 รูปแบบ response ที่เปลี่ยนไปหลังเปิด pagination

**ก่อนเปิด pagination** (`GET /api/posts/`):

```json
[
    {"id": 1, "title": "บทความที่ 1", "...": "..."},
    {"id": 2, "title": "บทความที่ 2", "...": "..."}
]
```

**หลังเปิด pagination**:

```json
{
    "count": 247,
    "next": "http://127.0.0.1:8000/api/posts/?page=2",
    "previous": null,
    "results": [
        {"id": 1, "title": "บทความที่ 1", "...": "..."},
        {"id": 2, "title": "บทความที่ 2", "...": "..."}
    ]
}
```

**การเปลี่ยนแปลงนี้ทำลาย backward compatibility ของ API ทันที** — client เดิมที่คาด
ว่า response เป็น array ตรง ๆ (`response.data[0]`) จะพังทันทีเมื่อเปลี่ยนเป็น object ที่
มี `results` ห่ออยู่ (`response.data.results[0]`) **คำแนะนำระดับมืออาชีพ**: เปิด
pagination ตั้งแต่ endpoint ยังใหม่ ๆ (release แรก) เสมอ อย่ารอไปเปิดทีหลังเมื่อมี client
ใช้งานจริงแล้ว เพราะจะกลายเป็น breaking change ที่ต้องประกาศ API version ใหม่
(เจาะลึกเรื่อง API Versioning ใน Part 048)

### 466.4 ปรับแต่ง `PageNumberPagination` เฉพาะ View ด้วย custom class

```python
# blog/pagination.py
from rest_framework.pagination import PageNumberPagination


class PostPagination(PageNumberPagination):
    page_size = 10                      # จำนวน record ต่อหน้า (ค่าเริ่มต้นถ้า client ไม่ระบุ)
    page_size_query_param = 'page_size' # อนุญาตให้ client กำหนดขนาดหน้าเอง
    max_page_size = 100                 # ป้องกัน client ขอ page_size ใหญ่เกินไป
    page_query_param = 'page'           # ชื่อ query param สำหรับเลขหน้า (ค่าเริ่มต้นคือ 'page' อยู่แล้ว)
```

```python
# blog/viewsets.py
from .pagination import PostPagination


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    pagination_class = PostPagination   # override ค่า default เฉพาะ ViewSet นี้
```

### 466.5 ทดสอบด้วย `curl`

```bash
# หน้าที่ 2 ขนาดหน้าเริ่มต้น (10 รายการ)
curl "http://127.0.0.1:8000/api/posts/?page=2"

# ให้ client กำหนดขนาดหน้าเอง (ไม่เกิน max_page_size=100)
curl "http://127.0.0.1:8000/api/posts/?page=1&page_size=50"

# ถ้าขอ page_size เกิน max_page_size จะถูกจำกัดที่ 100 อัตโนมัติ ไม่ error
curl "http://127.0.0.1:8000/api/posts/?page_size=99999"

# ถ้าขอเลขหน้าที่เกินจำนวนหน้าที่มีจริง จะได้ 404 Not Found
curl "http://127.0.0.1:8000/api/posts/?page=99999"
```

### 466.6 ตารางสรุป attribute ของ `PageNumberPagination`

| Attribute | ความหมาย | ค่าเริ่มต้น |
|---|---|---|
| `page_size` | จำนวน record ต่อหน้า | `None` (ต้องตั้งเอง หรือใช้ `PAGE_SIZE` จาก settings) |
| `page_size_query_param` | ชื่อ query param ให้ client กำหนดขนาดหน้าเอง | `None` (ปิดไว้ ต้องเปิดเอง) |
| `max_page_size` | ขนาดหน้าสูงสุดที่ client ขอได้ | `None` (ไม่จำกัด — ควรตั้งเสมอเพื่อความปลอดภัย) |
| `page_query_param` | ชื่อ query param สำหรับเลขหน้า | `'page'` |
| `last_page_strings` | ค่าพิเศษที่ใช้แทน "หน้าสุดท้าย" เช่น `?page=last` | `('last',)` |

---

## ขั้นตอนที่ 467: `LimitOffsetPagination` และ `CursorPagination` เปรียบเทียบ

### 467.1 `LimitOffsetPagination`: ควบคุมด้วย `limit`/`offset` โดยตรง

```python
# blog/pagination.py
from rest_framework.pagination import LimitOffsetPagination


class PostLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 10      # จำนวน record ต่อคำขอถ้าไม่ระบุ limit
    max_limit = 100          # จำกัด limit สูงสุดที่ client ขอได้
```

```bash
# ดึง 10 record แรก
curl "http://127.0.0.1:8000/api/posts/?limit=10&offset=0"

# ข้าม 20 record แรก แล้วดึง 10 record ถัดไป (เทียบเท่าหน้า 3 ถ้า page_size=10)
curl "http://127.0.0.1:8000/api/posts/?limit=10&offset=20"
```

Response format คล้ายกับ `PageNumberPagination` (`count`, `next`, `previous`,
`results`) แต่ `next`/`previous` เป็น URL ที่มี `limit`/`offset` แทน `page`:

```json
{
    "count": 247,
    "next": "http://127.0.0.1:8000/api/posts/?limit=10&offset=30",
    "previous": "http://127.0.0.1:8000/api/posts/?limit=10&offset=10",
    "results": [ ... ]
}
```

**ข้อดีเหนือ `PageNumberPagination`**: client ควบคุมขนาดหน้าและตำแหน่งเริ่มต้นได้อิสระ
กว่ามาก เหมาะกับ UI แบบ "infinite scroll" ที่โหลดข้อมูลทีละก้อนไม่ตายตัวตามหน้า

### 467.2 `CursorPagination`: ไม่ใช้เลขหน้า ใช้ "ตำแหน่งอ้างอิง" (cursor) แทน

```python
# blog/pagination.py
from rest_framework.pagination import CursorPagination


class PostCursorPagination(CursorPagination):
    page_size = 10
    ordering = '-created_at'   # จำเป็นต้องระบุเสมอ ใช้เป็นฐานในการสร้าง cursor
    cursor_query_param = 'cursor'
```

```bash
curl "http://127.0.0.1:8000/api/posts/?cursor=cD0yMDI2LTAxLTAxKzAwJTNBMDAlM0EwMA%3D%3D"
```

```json
{
    "next": "http://127.0.0.1:8000/api/posts/?cursor=cj0xJnA9MjAyNS0xMi0zMSswMCUzQTAwJTNBMDA%3D",
    "previous": null,
    "results": [ ... ]
}
```

สังเกตว่า **ไม่มี `count`** ใน response ของ `CursorPagination` — นี่เป็นข้อจำกัดโดย
เจตนา (ไม่ใช่บั๊ก) จะอธิบายเหตุผลเชิง performance ในขั้นตอนที่ 469 และ `cursor` เป็น
string ที่ **encode ตำแหน่งข้อมูลไว้ข้างใน** (ไม่ใช่แค่เลขหน้า) — client ไม่ควรพยายาม
"ถอดรหัส" หรือประกอบ cursor เอง ต้องใช้ค่า `next`/`previous` ที่ API ให้มาตรง ๆ เท่านั้น

**ข้อกำหนดสำคัญที่สุดของ `CursorPagination`**: ต้องเรียงลำดับด้วย field ที่**ไม่ซ้ำกัน
และไม่เปลี่ยนแปลงบ่อย** เช่น `created_at` ที่มี timestamp ละเอียดระดับ microsecond
(ถ้าหลาย record มี `created_at` เท่ากันเป๊ะ อาจมีปัญหาลำดับสลับกันได้) หรือใช้ `id`
เป็น tie-breaker ร่วมด้วยเพื่อความแม่นยำสูงสุด

### 467.3 ตารางเปรียบเทียบ Pagination ทั้ง 3 แบบ

| คุณสมบัติ | `PageNumberPagination` | `LimitOffsetPagination` | `CursorPagination` |
|---|---|---|---|
| Query parameter | `page`, `page_size` | `limit`, `offset` | `cursor` |
| แสดงจำนวนหน้าทั้งหมด (`count`) | ✅ มี | ✅ มี | ❌ ไม่มี |
| กระโดดไปหน้าใดก็ได้ตามใจ (เช่น หน้า 50) | ✅ ได้ (`?page=50`) | ✅ ได้ (`?offset=490`) | ❌ ไม่ได้ (ไปได้แค่หน้าถัดไป/ก่อนหน้าตามลำดับ) |
| Performance เมื่อข้อมูลลึกมาก (page สูง ๆ) | ❌ ช้าลงเรื่อย ๆ | ❌ ช้าลงเรื่อย ๆ (กลไกเดียวกับ page number) | ✅ เร็วคงที่ไม่ว่าจะลึกแค่ไหน |
| ปลอดภัยจากข้อมูลซ้ำ/ตกหล่นเมื่อมีการเพิ่ม/ลบระหว่างเปิดหน้า | ❌ เสี่ยง (อธิบาย 467.4) | ❌ เสี่ยง (กลไกเดียวกัน) | ✅ ปลอดภัยกว่ามาก |
| เหมาะกับ UI แบบไหน | ตัวเลขหน้า 1, 2, 3, ... แบบ pagination bar | Infinite scroll ที่ client คุมขนาด chunk เอง | Infinite scroll / feed ที่ข้อมูลเปลี่ยนแปลงตลอดเวลา (social media, timeline) |
| ความซับซ้อนสำหรับ client ในการใช้งาน | ต่ำที่สุด (คุ้นเคยง่าย) | ต่ำ | สูงกว่า (ต้องเก็บ cursor string ไว้ใช้ต่อ ไม่คำนวณเลขหน้าเอง) |

### 467.4 ทำไม Offset-based Pagination (ทั้ง `PageNumberPagination` และ
`LimitOffsetPagination`) เสี่ยงข้อมูลซ้ำ/ตกหล่น

สมมติผู้ใช้กำลังดูหน้า 2 (`offset=10, limit=10`) แต่ระหว่างนั้นมีบทความใหม่ถูกสร้างขึ้น
1 บทความ (แทรกที่ตำแหน่งบนสุดเพราะเรียง `-created_at`) เมื่อผู้ใช้กดไปหน้า 3
(`offset=20, limit=10`) ทุก record จะเลื่อนตำแหน่งไป 1 ที่ — **บทความที่ควรจะเป็น
record แรกของหน้า 3 กลับกลายเป็น record สุดท้ายของหน้า 2 ไปแล้ว** ทำให้ผู้ใช้เห็นบทความ
นั้นซ้ำ (หรือในทางกลับกัน ถ้าลบบทความออก อาจมีบทความที่ไม่เคยเห็นเลยถูกข้ามไป) —
ปัญหานี้ไม่เกิดกับ `CursorPagination` เพราะมันอ้างอิงตำแหน่งจาก**ค่าจริงของ record**
(เช่น `created_at < ค่าที่เห็นล่าสุด`) ไม่ใช่จาก "ลำดับที่นับได้" แบบ offset

---

## ขั้นตอนที่ 468: เขียน Custom Pagination Class เอง (ปรับ response format)

### 468.1 ทำไมต้อง custom: รูปแบบ response มาตรฐานของ DRF ไม่ตรงกับ spec ของทีม

หลายทีมมี**มาตรฐาน response format** ของบริษัทที่ต้องใช้เหมือนกันทุก endpoint เช่น
ห่อทุก response ด้วย `{"success": true, "data": ..., "meta": {...}}` เสมอ ซึ่งไม่ตรงกับ
`{"count", "next", "previous", "results"}` ที่ `PageNumberPagination` ให้มาโดยตรง —
วิธีแก้คือ override method `get_paginated_response()`

### 468.2 เขียน `StandardResultsPagination` ปรับรูปแบบ response ทั้งหมด

```python
# blog/pagination.py
from collections import OrderedDict
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response


class StandardResultsPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response(OrderedDict([
            ('success', True),
            ('meta', OrderedDict([
                ('total_items', self.page.paginator.count),
                ('total_pages', self.page.paginator.num_pages),
                ('current_page', self.page.number),
                ('page_size', self.get_page_size(self.request)),
                ('has_next', self.page.has_next()),
                ('has_previous', self.page.has_previous()),
            ])),
            ('links', OrderedDict([
                ('next', self.get_next_link()),
                ('previous', self.get_previous_link()),
            ])),
            ('data', data),
        ]))

    def get_paginated_response_schema(self, schema):
        """
        override schema สำหรับเอกสาร OpenAPI/Swagger (drf-spectacular ใน Part 049)
        ให้ตรงกับ response format จริงที่ปรับเอง ไม่เช่นนั้นเอกสาร API ที่ generate
        อัตโนมัติจะยังอ้างอิง schema เดิมของ PageNumberPagination อยู่
        """
        return {
            'type': 'object',
            'properties': {
                'success': {'type': 'boolean'},
                'meta': {
                    'type': 'object',
                    'properties': {
                        'total_items': {'type': 'integer'},
                        'total_pages': {'type': 'integer'},
                        'current_page': {'type': 'integer'},
                        'page_size': {'type': 'integer'},
                        'has_next': {'type': 'boolean'},
                        'has_previous': {'type': 'boolean'},
                    },
                },
                'links': {
                    'type': 'object',
                    'properties': {
                        'next': {'type': 'string', 'nullable': True, 'format': 'uri'},
                        'previous': {'type': 'string', 'nullable': True, 'format': 'uri'},
                    },
                },
                'data': schema,
            },
        }
```

```python
# blog/viewsets.py
from .pagination import StandardResultsPagination


class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    pagination_class = StandardResultsPagination
```

### 468.3 ผลลัพธ์หลัง custom

```bash
curl "http://127.0.0.1:8000/api/posts/?page=2"
```

```json
{
    "success": true,
    "meta": {
        "total_items": 247,
        "total_pages": 25,
        "current_page": 2,
        "page_size": 10,
        "has_next": true,
        "has_previous": true
    },
    "links": {
        "next": "http://127.0.0.1:8000/api/posts/?page=3",
        "previous": "http://127.0.0.1:8000/api/posts/?page=1"
    },
    "data": [
        {"id": 11, "title": "บทความที่ 11", "...": "..."},
        {"id": 12, "title": "บทความที่ 12", "...": "..."}
    ]
}
```

### 468.4 attribute และ method ที่ใช้บ่อยที่สุดตอนเขียน custom pagination

| Attribute/Method | ใช้ทำอะไร |
|---|---|
| `self.page` | Django `Page` object (มาจาก `django.core.paginator`) |
| `self.page.paginator.count` | จำนวน record ทั้งหมด (ไม่นับ pagination) |
| `self.page.paginator.num_pages` | จำนวนหน้าทั้งหมด |
| `self.page.number` | เลขหน้าปัจจุบัน |
| `self.page.has_next()` / `self.page.has_previous()` | มีหน้าถัดไป/ก่อนหน้าหรือไม่ |
| `self.get_next_link()` / `self.get_previous_link()` | สร้าง URL หน้าถัดไป/ก่อนหน้า พร้อม query param ครบ |
| `self.get_page_size(request)` | อ่านค่า page size ที่ใช้จริงในคำขอนี้ (รวม logic `max_page_size`) |

**คำเตือนสำคัญ**: การ custom pagination class **ไม่ได้จำกัดแค่ `PageNumberPagination`
เท่านั้น** — `LimitOffsetPagination` และ `CursorPagination` ก็ override
`get_paginated_response()` แบบเดียวกันได้ทั้งหมด เพราะทุกคลาสสืบทอดจาก interface
เดียวกัน (`BasePagination`) แต่ต้องเปลี่ยน field ที่อ้างอิงใน `get_paginated_response()`
ให้ตรงกับ attribute ของแต่ละคลาส (เช่น `CursorPagination` ไม่มี `self.page.paginator`
แบบเดียวกับ `PageNumberPagination`)

---

## ขั้นตอนที่ 469: ผลกระทบด้าน Performance ของการเลือก Pagination กับข้อมูลขนาดใหญ่

### 469.1 กลไกเบื้องหลัง Offset-based Pagination ในระดับ SQL

`PageNumberPagination` และ `LimitOffsetPagination` ทั้งคู่แปลงคำขอเป็น SQL แบบ
`OFFSET ... LIMIT ...` เบื้องหลัง:

```sql
-- ?page=3&page_size=10 (เทียบเท่า offset=20, limit=10)
SELECT * FROM blog_post
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
```

`OFFSET` **ไม่ใช่การ "กระโดด" ไปตำแหน่งนั้นตรง ๆ** — ฐานข้อมูลต้อง**อ่านและทิ้ง**
record ทั้งหมดที่อยู่ก่อนตำแหน่ง offset ก่อนเสมอ (ต่อให้ index ช่วยเรื่องการเรียงลำดับได้
แต่การนับ+ข้าม N record แรกยังคงเป็นต้นทุนที่หลีกเลี่ยงไม่ได้) พูดให้ชัดคือต้นทุนของ
`OFFSET n LIMIT m` แปรผันตาม **`O(n + m)`** ไม่ใช่ `O(m)` — ยิ่ง `n` (offset) มากเท่าไหร่
ยิ่งช้าเท่านั้น ไม่ว่า `m` (limit) จะเล็กแค่ไหนก็ตาม

### 469.2 ตัวอย่าง `EXPLAIN ANALYZE` แสดงต้นทุนที่เพิ่มขึ้นตามความลึกของหน้า

```sql
-- หน้าแรก (offset=0) — เร็วมาก
EXPLAIN ANALYZE
SELECT * FROM blog_post ORDER BY created_at DESC LIMIT 10 OFFSET 0;
-- Execution Time: 0.842 ms

-- หน้ากลาง ๆ ของข้อมูล 1 ล้าน record (offset=500,000)
EXPLAIN ANALYZE
SELECT * FROM blog_post ORDER BY created_at DESC LIMIT 10 OFFSET 500000;
-- Execution Time: 312.418 ms   <- ช้าลงกว่า 300 เท่า ทั้งที่ขอข้อมูลแค่ 10 แถวเท่ากัน

-- หน้าลึกสุด (offset=990,000) จากข้อมูล 1 ล้าน record
EXPLAIN ANALYZE
SELECT * FROM blog_post ORDER BY created_at DESC LIMIT 10 OFFSET 990000;
-- Execution Time: 601.203 ms
```

(ตัวเลขข้างต้นเป็นค่าประมาณเชิงสาธิตบน PostgreSQL ทั่วไปที่ไม่มี partitioning พิเศษ
ตัวเลขจริงแปรผันตามฮาร์ดแวร์และการตั้งค่า index แต่**ทิศทางของแนวโน้มเป็นจริงเสมอ**:
ยิ่ง offset ลึก ยิ่งช้าลงแบบเป็นเส้นตรง)

### 469.3 ตารางเปรียบเทียบเวลาตอบสนองโดยประมาณ: Offset vs Cursor Pagination

| ตำแหน่งหน้าที่ขอ | จำนวน record ในตาราง | Offset Pagination (โดยประมาณ) | Cursor Pagination (โดยประมาณ) |
|---|---|---|---|
| หน้า 1 (offset=0) | 1,000,000 | ~1 ms | ~1 ms |
| หน้า 100 (offset=990) | 1,000,000 | ~5 ms | ~1-2 ms |
| หน้า 10,000 (offset=99,990) | 1,000,000 | ~60-120 ms | ~1-2 ms |
| หน้า 100,000 (offset=999,990) | 1,000,000 | ~500-900 ms | ~1-2 ms |
| หน้า 1,000,000 (offset=9,999,990) | 10,000,000 | หลายวินาที ถึง timeout | ~1-2 ms (คงที่) |

เหตุผลที่ `CursorPagination` **คงที่ไม่ว่าจะลึกแค่ไหน**: มันแปลงเป็น SQL รูปแบบ

```sql
-- Cursor-based: อ้างอิงค่าจริงของ record ล่าสุดที่เห็น ไม่ใช่นับ offset
SELECT * FROM blog_post
WHERE created_at < '2026-03-15 09:12:00'   -- ค่าจาก cursor ที่ decode มา
ORDER BY created_at DESC
LIMIT 10;
```

ถ้า `created_at` มี **index** อยู่แล้ว (ซึ่งควรมีเสมอสำหรับ field ที่ใช้ ordering
บ่อย ๆ) การค้นหาแบบ `WHERE created_at < X ORDER BY created_at LIMIT 10` ใช้
**index seek** ตรงไปยังตำแหน่งได้เลย ต้นทุนคือ `O(log n + m)` แทบไม่ขึ้นกับตำแหน่งความลึก
ของหน้าเลย ต่างจาก `OFFSET` ที่ต้องนับผ่านทุก record ก่อนหน้าเสมอ

### 469.4 ต้นทุนแอบแฝงอีกอย่าง: `SELECT COUNT(*)` สำหรับ field `count`

`PageNumberPagination` และ `LimitOffsetPagination` ต้องรัน query แยกต่างหากเพื่อนับ
จำนวน record ทั้งหมด (สำหรับ field `count` ใน response) ทุกครั้งที่เรียก:

```sql
SELECT COUNT(*) FROM blog_post WHERE is_published = true;
```

บนตารางขนาดใหญ่มาก (หลักสิบล้าน record) แม้แต่ `COUNT(*)` เพียงอย่างเดียว (ไม่นับ
`OFFSET` เลย) ก็ใช้เวลานานได้ถ้าไม่มี index ที่ครอบคลุมเงื่อนไข `WHERE` พอดี —
`CursorPagination` **ไม่ต้องรัน query นี้เลย** เพราะไม่คืนค่า `count` ตั้งแต่แรก
(ตามที่สังเกตได้จากขั้นตอนที่ 467.2) นี่คือเหตุผลอีกข้อที่ทำให้มันเร็วกว่ามากในสเกลใหญ่

### 469.5 คำแนะนำระดับมืออาชีพ: เลือก Pagination ตามลักษณะการใช้งานจริง

| สถานการณ์ | Pagination ที่แนะนำ | เหตุผล |
|---|---|---|
| หน้า Admin/Dashboard ที่ผู้ใช้ต้องการ "กระโดดไปหน้า 47" หรือเห็นจำนวนหน้าทั้งหมด | `PageNumberPagination` | ต้องมีเลขหน้าและ `count` ให้ผู้ใช้อ้างอิง ข้อมูลมักไม่ลึกมาก (หลักพัน-หมื่น record) |
| Mobile App แบบ infinite scroll ที่ต้องการควบคุมขนาด chunk เอง | `LimitOffsetPagination` | ยืดหยุ่นกว่า `PageNumberPagination` เรื่องขนาดหน้า แต่ยังยอมรับต้นทุน `OFFSET` ได้ (ข้อมูลไม่ลึกเกินไป) |
| Feed/Timeline ที่ข้อมูลเพิ่มใหม่ตลอดเวลา (social media, notification list) | `CursorPagination` | ป้องกันปัญหาข้อมูลซ้ำ/ตกหล่นจากขั้นตอนที่ 467.4 และเร็วคงที่แม้ scroll ลึกมาก |
| API สาธารณะที่มี traffic สูงและตารางข้อมูลมีหลักล้าน record ขึ้นไป | `CursorPagination` | หลีกเลี่ยงทั้งต้นทุน `OFFSET` และต้นทุน `COUNT(*)` พร้อมกัน (ขั้นตอนที่ 469.1-469.4) |
| Export ข้อมูลทั้งหมดเป็น batch job เบื้องหลัง (ไม่ใช่ user-facing) | ไม่ใช้ DRF pagination เลย ใช้ `.iterator()` หรือ `QuerySet.iterator(chunk_size=...)` ของ Django ตรง ๆ | Pagination ของ DRF ออกแบบมาสำหรับ HTTP request/response ทีละหน้า ไม่เหมาะกับงาน batch ที่ต้องอ่านข้อมูลทั้งหมดต่อเนื่อง |

### 469.6 สิ่งที่ต้องเตรียมล่วงหน้าเมื่อจะใช้ `CursorPagination` ให้ได้ประโยชน์เต็มที่

`CursorPagination` จะเร็วตามที่โฆษณาไว้**ก็ต่อเมื่อ field ที่ใช้ ordering มี database
index รองรับเท่านั้น** — ถ้า `ordering = '-created_at'` แต่ `created_at` ไม่มี index
เลย ฐานข้อมูลยังคงต้องทำ full table scan เหมือนเดิม (แค่ไม่ต้องนับ offset เพิ่มเท่านั้น)
ตรวจสอบและเพิ่ม index ให้ field ที่ใช้เป็น cursor เสมอ:

```python
# blog/models.py
class Post(models.Model):
    # ...
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at']),
        ]
```

```bash
python manage.py makemigrations blog
python manage.py migrate
```

**สรุปกฎที่จำง่าย**: ยิ่งตารางใหญ่และมี traffic สูงเท่าไหร่ ยิ่งควรเอนเอียงไปทาง
`CursorPagination` มากเท่านั้น ส่วน `PageNumberPagination` เหมาะกับกรณีที่ข้อมูลมีขนาด
จำกัดและผู้ใช้ต้องการควบคุม navigation แบบเลขหน้าจริง ๆ — ไม่มีคำตอบเดียวที่ถูกต้อง
เสมอไป ต้องเลือกตามลักษณะข้อมูลและผู้ใช้งานจริงของแต่ละ endpoint

---

## ขั้นตอนที่ 470: สรุปและแบบฝึกหัด — Blog API พร้อม filter/search/pagination เต็มรูปแบบ

### 470.1 ประกอบทุกอย่างเข้าด้วยกัน: `blog/filters.py` ฉบับสมบูรณ์

```python
# blog/filters.py
import django_filters
from .models import Post


class PostFilter(django_filters.FilterSet):
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='date__gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='date__lte')
    category = django_filters.CharFilter(field_name='category__slug', lookup_expr='iexact')
    tag = django_filters.CharFilter(method='filter_by_tag')
    author = django_filters.CharFilter(field_name='author__username', lookup_expr='iexact')

    class Meta:
        model = Post
        fields = ['is_published']

    def filter_by_tag(self, queryset, name, value):
        return queryset.filter(tags__slug__iexact=value).distinct()
```

### 470.2 `blog/pagination.py` ฉบับสมบูรณ์

```python
# blog/pagination.py
from collections import OrderedDict
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response


class StandardResultsPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response(OrderedDict([
            ('success', True),
            ('meta', OrderedDict([
                ('total_items', self.page.paginator.count),
                ('total_pages', self.page.paginator.num_pages),
                ('current_page', self.page.number),
                ('page_size', self.get_page_size(self.request)),
                ('has_next', self.page.has_next()),
                ('has_previous', self.page.has_previous()),
            ])),
            ('links', OrderedDict([
                ('next', self.get_next_link()),
                ('previous', self.get_previous_link()),
            ])),
            ('data', data),
        ]))
```

### 470.3 `blog/viewsets.py` ฉบับสมบูรณ์: `PostViewSet` พร้อมทุกฟีเจอร์ของ Part นี้

```python
# blog/viewsets.py
from rest_framework import viewsets
from rest_framework.filters import OrderingFilter, SearchFilter
from django_filters.rest_framework import DjangoFilterBackend
from .models import Post, Category, Comment
from .serializers import PostSerializer, CategorySerializer, CommentSerializer
from .filters import PostFilter
from .pagination import StandardResultsPagination


class PostViewSet(viewsets.ModelViewSet):
    """
    Blog API ของ Post — ครบทั้ง filter (django-filter), search (SearchFilter),
    ordering (OrderingFilter) และ pagination แบบ custom format (Part 047)
    ต่อยอดจาก ModelViewSet + Router (Part 044) และ JWT auth (Part 046)
    """
    queryset = Post.objects.select_related('category', 'author').prefetch_related('tags')
    serializer_class = PostSerializer
    lookup_field = 'slug'

    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = PostFilter
    search_fields = ['title', 'content', 'author__username']
    ordering_fields = ['created_at', 'updated_at', 'title']
    ordering = ['-created_at']

    pagination_class = StandardResultsPagination

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


class CategoryViewSet(viewsets.ModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    filter_backends = [SearchFilter, OrderingFilter]
    search_fields = ['name']
    ordering_fields = ['name']


class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    filter_backends = [OrderingFilter]
    ordering_fields = ['created_at']
    ordering = ['created_at']

    def get_queryset(self):
        return Comment.objects.filter(post_id=self.kwargs['post_pk'])

    def perform_create(self, serializer):
        serializer.save(post_id=self.kwargs['post_pk'], author=self.request.user)
```

### 470.4 ทดสอบภาพรวมด้วย `curl` ครบทุกฟีเจอร์ในคำขอเดียว

```bash
curl "http://127.0.0.1:8000/api/posts/?is_published=true&category=django&tag=orm&created_after=2026-01-01&search=queryset&ordering=-created_at&page=1&page_size=5"
```

```json
{
    "success": true,
    "meta": {
        "total_items": 18,
        "total_pages": 4,
        "current_page": 1,
        "page_size": 5,
        "has_next": true,
        "has_previous": false
    },
    "links": {
        "next": "http://127.0.0.1:8000/api/posts/?...&page=2",
        "previous": null
    },
    "data": [
        {"id": 42, "title": "เจาะลึก QuerySet Optimization", "...": "..."}
    ]
}
```

คำขอเดียวนี้ผ่านทุกชั้นที่เรียนมาทั้ง Part: `DjangoFilterBackend` กรอง 4 เงื่อนไข
(`is_published`, `category`, `tag`, `created_after`) → `SearchFilter` ค้นคำ
"queryset" ต่อ → `OrderingFilter` เรียงใหม่สุดก่อน → `StandardResultsPagination`
ตัดเหลือหน้าละ 5 record พร้อมห่อ response ตามรูปแบบที่ทีมกำหนดเอง

### 470.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ติดตั้งและใช้ `django-filter` ผ่าน `DjangoFilterBackend` ทั้งแบบ `filterset_fields`
  แบบเร็วและแบบ dict ที่ระบุ lookup ได้ละเอียดขึ้น
- ✅ ใช้ `SearchFilter` ค้นหาข้อความแบบ substring ในหลาย field พร้อมกัน รวมถึง prefix
  พิเศษ `^`, `=`, `@`, `$`
- ✅ ใช้ `OrderingFilter` ให้ client ควบคุมลำดับผลลัพธ์ผ่าน query param `ordering`
  พร้อมเข้าใจความเสี่ยงของการตั้ง `ordering_fields = '__all__'`
- ✅ เขียน `FilterSet` class เองเพื่อกรองแบบช่วง (วันที่ ราคา) และ custom filter method
  ที่ validate/join queryset ซับซ้อนได้
- ✅ ผสาน `DjangoFilterBackend`, `SearchFilter`, `OrderingFilter` เข้าด้วยกันใน
  `filter_backends` list เดียว และเข้าใจว่ามัน override ไม่ใช่ merge กับค่า default
- ✅ เจาะลึก `PageNumberPagination` ทั้ง `page_size`, `page_size_query_param`,
  `max_page_size`
- ✅ เปรียบเทียบ `LimitOffsetPagination` และ `CursorPagination` ทั้งการใช้งานและ
  ข้อจำกัด/ข้อดีของแต่ละแบบ
- ✅ เขียน Custom Pagination Class ปรับรูปแบบ response ให้ตรงตามมาตรฐานของทีม
- ✅ เข้าใจว่าทำไม Offset-based Pagination ช้าลงเมื่อหน้าลึกขึ้น (`O(n+m)`) ในขณะที่
  Cursor Pagination เร็วคงที่ (`O(log n + m)`) พร้อมรู้ว่าต้องมี index รองรับเสมอ
- ✅ ประกอบ Blog API เต็มรูปแบบที่มีทั้ง filter, search, ordering, pagination พร้อมกัน

### 470.6 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `django-filter` และตั้งค่า `DjangoFilterBackend` ได้ทั้งระดับโปรเจกต์และ
      ระดับ View
- [ ] เขียน `FilterSet` class เองที่มีทั้ง exact filter, range filter (`gte`/`lte`),
      และ custom filter method ได้โดยไม่ต้องเปิดเอกสาร
- [ ] อธิบายความแตกต่างระหว่าง `filterset_fields` กับ `filterset_class` ได้ว่าใช้
      เมื่อไหร่
- [ ] ใช้ `SearchFilter` พร้อม prefix พิเศษ (`^`, `=`, `@`, `$`) ได้ถูกต้องตามสถานการณ์
- [ ] ใช้ `OrderingFilter` พร้อมตั้ง whitelist ของ `ordering_fields` อย่างปลอดภัย
- [ ] อธิบายได้ว่าทำไม `filter_backends` ระดับ View จึง override ไม่ใช่ merge กับ
      `DEFAULT_FILTER_BACKENDS`
- [ ] เลือกระหว่าง `PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination`
      ได้อย่างมีเหตุผลตามลักษณะข้อมูลและผู้ใช้งาน
- [ ] เขียน Custom Pagination Class ที่ override `get_paginated_response()` ได้เอง
- [ ] อธิบายได้ว่าทำไม Offset-based Pagination ช้าลงเมื่อหน้าลึกขึ้น และทำไม
      `CursorPagination` ไม่มี field `count`

### 470.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เพิ่ม `filterset_fields` ให้ `CategoryViewSet` เพื่อกรอง
ตาม `name` แบบ `icontains` (ใช้ dict syntax จากขั้นตอนที่ 461.5) แล้วทดสอบด้วย
`GET /api/categories/?name__icontains=dja`

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เขียน `CommentFilter` เป็น `FilterSet` class ที่กรอง
comment ตามช่วงวันที่ (`created_after`/`created_before` เหมือนขั้นตอนที่ 464.5) และ
ตาม `author` (username) แล้วนำไปใช้กับ `CommentViewSet` ที่เป็น nested resource จาก
Part 044 — ต้องแน่ใจว่าการกรองยังทำงานร่วมกับ `get_queryset()` ที่กรองด้วย
`post_pk` อยู่แล้วได้ถูกต้อง (AND กันทั้งสองเงื่อนไข)

**แบบฝึกหัดที่ 3 (Pagination)**: สร้าง `PostCursorPagination` (จากขั้นตอนที่ 467.2)
แล้วสลับใช้กับ `PostViewSet` แทน `StandardResultsPagination` ชั่วคราว ทดสอบว่า
`?ordering=-created_at` (จาก `OrderingFilter`) กับ `ordering = '-created_at'` ที่ตั้งไว้
ใน `CursorPagination` ทำงานสอดคล้องกันหรือขัดแย้งกัน อธิบายว่าทำไม
(คำใบ้: `CursorPagination.ordering` เป็นค่าตายตัวที่ไม่ได้ผูกกับ `OrderingFilter`
โดยอัตโนมัติ — ต้อง override `get_ordering()` เองถ้าต้องการให้ทั้งสองทำงานร่วมกัน)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ใช้เครื่องมือสร้างข้อมูลจำลอง (เช่น เขียน management
command ด้วย `bulk_create` ที่เรียนจาก Phase 2) สร้างบทความจำลอง 100,000 รายการ
แล้ววัดเวลาตอบสนองจริงของ `?page=1` เทียบกับ `?page=5000` (ใช้
`PageNumberPagination`) ด้วย `curl -w "%{time_total}\n"` บันทึกผลเป็นตาราง แล้วเปลี่ยน
ไปใช้ `CursorPagination` วัดซ้ำที่ตำแหน่งเทียบเท่ากัน เปรียบเทียบผลลัพธ์จริงกับตัวเลข
ประมาณการในขั้นตอนที่ 469.3

### 470.8 คำถามที่พบบ่อย (FAQ)

**Q: ใช้ `filterset_fields` กับ `search_fields`/`ordering_fields` พร้อมกันได้ไหม?**
A: ได้เต็มที่ และเป็นแนวปฏิบัติปกติในโปรเจกต์จริง เพียงแค่ต้องใส่ backend ทั้งหมดที่
ต้องการไว้ใน `filter_backends` list เดียวกัน (ขั้นตอนที่ 465) — ทั้งสามระบบทำงาน
อิสระต่อกันและ query parameter ชื่อไม่ชนกัน (filter ใช้ชื่อ field ตรง ๆ, search ใช้
`search`, ordering ใช้ `ordering`)

**Q: ทำไมเปลี่ยนจาก `filterset_fields` ธรรมดาไปเป็น `filterset_class` แล้ว endpoint
กรองด้วย field เดิมไม่ได้ผลเหมือนเดิม?**
A: เพราะ `Meta.fields` ใน `FilterSet` class ต้องระบุ field ที่ต้องการกรองแบบ exact-match
ธรรมดาไว้ด้วยตัวเอง (เหมือนที่ทำในขั้นตอนที่ 464.5 ที่ใส่ `fields = ['is_published']`)
ไม่ได้สืบทอดมาจาก `filterset_fields` เดิมโดยอัตโนมัติ ต้องเช็ค `Meta.fields` ให้ครบ
ทุก field ที่เคยใช้งานอยู่ก่อนแล้ว

**Q: `CursorPagination` เหมาะกับ Django Admin หรือหน้าที่ต้องการปุ่ม "ไปหน้าสุดท้าย"
หรือไม่?**
A: ไม่เหมาะ เพราะ `CursorPagination` ออกแบบมาให้เดินทางแบบ "ถัดไป/ก่อนหน้า" ตามลำดับ
เท่านั้น ไม่รองรับการกระโดดไปตำแหน่งใดก็ได้ตามใจ (ไม่มี concept ของ "หน้า 47" หรือ
"หน้าสุดท้าย") ถ้า UI ต้องการความสามารถนี้ ให้ใช้ `PageNumberPagination` แทน แล้วยอมรับ
ต้นทุน `OFFSET` ที่มากับมัน หรือจำกัดจำนวนหน้าสูงสุดที่ผู้ใช้เข้าถึงได้เพื่อควบคุม
performance (เช่น "ดูได้ถึงหน้า 1,000 เท่านั้น กรุณาใช้ตัวกรองเพื่อจำกัดผลลัพธ์")

**Q: ต้องเลือก pagination class เดียวใช้ทั้งโปรเจกต์หรือเลือกต่างกันได้ตาม endpoint?**
A: เลือกต่างกันได้ และในงานจริงมักเลือกต่างกันจริง ๆ ตั้ง `DEFAULT_PAGINATION_CLASS`
ระดับโปรเจกต์เป็นตัวที่ใช้บ่อยที่สุด (มักเป็น `PageNumberPagination` เพราะเข้าใจง่าย)
แล้ว override `pagination_class` เป็นรายตัวสำหรับ endpoint พิเศษที่ต้องการ
`CursorPagination` (เช่น feed) หรือ `LimitOffsetPagination` (เช่น endpoint ที่ mobile
team ขอมาโดยเฉพาะ) — เหมือนกับที่ `permission_classes` และ `filter_backends`
override ได้เป็นรายตัวมาตั้งแต่ Part ก่อน ๆ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 048: API Versioning และ Throttling** จะพาไปเรียนรู้วิธีจัดการเมื่อ API ต้อง
เปลี่ยนแปลงแบบ breaking change (เช่น เปลี่ยนรูปแบบ response ของ pagination แบบที่เห็น
ในขั้นตอนที่ 466.3) โดยไม่ทำให้ client เดิมพัง ผ่านระบบ **API Versioning**
(`URLPathVersioning`, `NamespaceVersioning`, `AcceptHeaderVersioning`) และวิธีป้องกัน
ไม่ให้ endpoint ถูกเรียกถี่เกินไปจนระบบล่มด้วย **Throttling**
(`AnonRateThrottle`, `UserRateThrottle`, Custom Throttle Class) ซึ่งเป็นทักษะสำคัญ
คู่กันเมื่อ API เปิดให้ third-party หรือ public ใช้งานจริงตามที่วางรากฐานไว้ตั้งแต่
Part 046 เรื่อง OAuth2

เตรียม `blog/viewsets.py`, `blog/filters.py`, และ `blog/pagination.py` ของคุณที่ทำ
เสร็จสมบูรณ์ใน Part นี้ไว้ให้พร้อม เพราะ Part 048 จะเพิ่ม versioning และ throttling
เข้าไปในไฟล์เดียวกันนี้ต่อทันที!
