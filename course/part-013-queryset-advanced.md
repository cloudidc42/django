# Part 013: Django ORM QuerySet ขั้นสูง

> **ขั้นตอนที่ 121-130 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เจาะลึกกลไกภายในของ QuerySet ที่ Django ORM ใช้สื่อสารกับ
> ฐานข้อมูล ตั้งแต่แนวคิด Lazy Evaluation, field lookups ครบทุกตัว, การ chain query,
> การเรียงลำดับผลลัพธ์ขั้นสูง, การดึงข้อมูลแบบ optimize ด้วย `values()`/`only()`/`defer()`,
> ไปจนถึงการจัดการข้อมูลจำนวนมากด้วย `bulk_create()`/`bulk_update()` และคำเตือนสำคัญ
> เรื่อง `update()`/`delete()` ระดับ QuerySet เมื่อจบ Part นี้ คุณจะเขียน query ที่ทั้ง
> ถูกต้อง เร็ว และไม่ยิง SQL เกินความจำเป็น ซึ่งเป็นทักษะที่แยกนักพัฒนา Django มือใหม่
> ออกจากมืออาชีพอย่างชัดเจน

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 121: QuerySet เป็น Lazy — ทำไม query ไม่ยิงจริงจนกว่าจะถูก evaluate
- ขั้นตอนที่ 122: `filter()` vs `exclude()` vs `get()` และ Field Lookups ทั้งหมด
- ขั้นตอนที่ 123: Chaining QuerySets และ QuerySet Caching
- ขั้นตอนที่ 124: `order_by()` เจาะลึก
- ขั้นตอนที่ 125: `values()` และ `values_list()`
- ขั้นตอนที่ 126: `only()` และ `defer()` สำหรับ optimize การดึง field
- ขั้นตอนที่ 127: `exists()`, `count()`, `first()`, `last()`
- ขั้นตอนที่ 128: `bulk_create()`, `bulk_update()`, `in_bulk()`
- ขั้นตอนที่ 129: `update()` และ `delete()` แบบ QuerySet-level
- ขั้นตอนที่ 130: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 121: QuerySet เป็น Lazy — ทำไม query ไม่ยิงจริงจนกว่าจะถูก evaluate

### 121.1 โมเดลอ้างอิงที่ใช้ตลอด Part นี้

ก่อนเริ่มเจาะลึก QuerySet เรามาทบทวนโครงสร้างของแอป `blog` ที่เราสร้างไว้ใน Part 011-012
กันก่อน เพราะทุกตัวอย่างใน Part นี้จะอ้างอิงจากโมเดลชุดนี้ทั้งหมด:

```python
# blog/models.py
from django.conf import settings
from django.db import models
from django.utils import timezone


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name_plural = "categories"

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="posts",
    )

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    parent = models.ForeignKey(
        "self", on_delete=models.CASCADE, null=True, blank=True,
        related_name="replies",
    )

    def __str__(self):
        return f"Comment by {self.author} on {self.post}"


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile",
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)
    birth_date = models.DateField(null=True, blank=True)

    def __str__(self):
        return f"Profile of {self.user}"
```

### 121.2 QuerySet คืออะไร

เมื่อคุณเขียน `Post.objects.all()` หรือ `Post.objects.filter(is_published=True)` สิ่งที่
คุณได้กลับมา**ไม่ใช่**รายการข้อมูลจากฐานข้อมูลทันที แต่เป็นออบเจกต์ชนิด **QuerySet** ซึ่ง
เป็นเหมือน "คำสั่ง SQL ที่ยังไม่ได้ยิงจริง" (a representation of a database query)

```python
>>> qs = Post.objects.filter(is_published=True)
>>> type(qs)
<class 'django.db.models.query.QuerySet'>
```

ตราบใดที่คุณยังไม่ได้ "แตะ" ข้อมูลจริงในนั้น Django จะยังไม่ยิง SQL ไปที่ฐานข้อมูลเลย
คุณสมบัตินี้เรียกว่า **Lazy Evaluation** (การประเมินผลแบบขี้เกียจ — รอจนจำเป็นจริง ๆ ค่อยทำ)

### 121.3 อะไรบ้างที่ทำให้ QuerySet ถูก Evaluate (ยิง SQL จริง)

มีเหตุการณ์หลัก ๆ ที่ทำให้ QuerySet ถูกแปลงเป็น SQL แล้วยิงไปยังฐานข้อมูลจริง:

| การกระทำ | ตัวอย่างโค้ด | อธิบาย |
|---|---|---|
| Iteration (วนลูป) | `for post in qs:` | วนลูปครั้งแรกจะยิง query แล้ว cache ผลลัพธ์ไว้ |
| `list()` | `list(qs)` | บังคับแปลงเป็น list ทันที |
| `bool()` / ใช้ใน `if` | `if qs:` | ตรวจว่ามีข้อมูลหรือไม่ |
| `len()` | `len(qs)` | นับจำนวนผลลัพธ์ (ดึงข้อมูลทั้งหมดมาก่อน) |
| `repr()` | พิมพ์ `qs` ใน shell | Django แสดงตัวอย่าง 21 แถวแรกเพื่อ debug |
| Slicing แบบมี step | `qs[0:10:2]` | slicing ปกติ (ไม่มี step) ยังคง lazy แต่ถ้าใส่ step จะ evaluate ทันที |
| Pickling | `pickle.dumps(qs)` | ใช้เก็บ cache เช่น Django cache framework |

ตัวอย่างที่แสดงให้เห็นว่า filter ไม่ยิง SQL จนกว่าจะ evaluate:

```python
>>> from blog.models import Post
>>> qs = Post.objects.filter(is_published=True)   # ยังไม่ยิง SQL
>>> qs = qs.filter(category__slug="django")        # ยังไม่ยิง SQL (แค่ต่อเงื่อนไข)
>>> qs = qs.exclude(author__username="admin")      # ยังไม่ยิง SQL
>>> print("กำลังจะ evaluate...")
>>> posts = list(qs)   # <-- ตอนนี้แหละที่ Django สร้าง SQL แล้วยิงจริงครั้งเดียว
```

สิ่งนี้สำคัญมาก เพราะมันหมายความว่าคุณสามารถ **ต่อเงื่อนไข (chain)** ได้เรื่อย ๆ โดยไม่มี
ค่าใช้จ่ายด้าน performance จนกว่าจะถึงจุดที่ต้องการผลลัพธ์จริง ๆ

### 121.4 การพิสูจน์ด้วย Django Debug Toolbar

ในงานจริงเราไม่ต้องเดาว่า query ยิงกี่ครั้ง เราใช้เครื่องมือดู SQL panel ได้ตรง ๆ ติดตั้ง
**Django Debug Toolbar** (จะสอนละเอียดเรื่อง performance profiling ใน Phase 8 แต่ขอแนะนำ
เบื้องต้นที่นี่เพราะเหมาะกับหัวข้อนี้มาก):

```bash
pip install django-debug-toolbar
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "debug_toolbar",
]

MIDDLEWARE = [
    "debug_toolbar.middleware.DebugToolbarMiddleware",
    # ...
]

INTERNAL_IPS = [
    "127.0.0.1",
]
```

```python
# urls.py (root)
from django.urls import include, path

urlpatterns = [
    # ... url patterns อื่น ๆ
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path("__debug__/", include(debug_toolbar.urls)),
    ] + urlpatterns
```

เมื่อเปิดหน้าเว็บที่มีการ query ผ่าน Debug Toolbar คุณจะเห็น panel "SQL" ที่บอกจำนวนครั้ง
ที่ query ถูกยิง เวลาที่ใช้ และ SQL statement แบบเต็ม นี่คือวิธีที่มืออาชีพใช้ตรวจสอบว่า
โค้ดของตัวเองมีปัญหา **N+1 query** หรือยิง query ซ้ำโดยไม่จำเป็นหรือไม่ (เราจะเจาะลึก N+1
และ `select_related`/`prefetch_related` ใน Part 014-015)

### 121.5 ทดลองดู SQL จริงด้วย `.query` และ `connection.queries`

อีกวิธีหนึ่งที่ทำได้ทันทีใน shell โดยไม่ต้องรันเว็บเซิร์ฟเวอร์ คือดู SQL ที่ Django
**จะ**สร้าง (ผ่าน `.query`) และ SQL ที่ **ถูกยิงไปแล้วจริง** (ผ่าน `connection.queries`):

```bash
python manage.py shell
```

```python
>>> from blog.models import Post
>>> qs = Post.objects.filter(is_published=True)
>>> print(qs.query)   # ดู SQL ที่ "จะ" ถูกสร้าง (ยังไม่ได้ยิงจริง)
SELECT "blog_post"."id", "blog_post"."title", ... FROM "blog_post"
WHERE "blog_post"."is_published" = 1

>>> from django.db import connection, reset_queries
>>> reset_queries()
>>> list(qs)             # ตอนนี้ evaluate จริง
>>> len(connection.queries)   # ดูว่ายิงไปกี่ครั้งแล้ว (ต้องเปิด settings.DEBUG = True)
1
>>> print(connection.queries[-1]["sql"])
```

**ข้อควรระวัง**: `connection.queries` จะบันทึกข้อมูลก็ต่อเมื่อ `DEBUG = True` เท่านั้น
ใน production ที่ `DEBUG = False` list นี้จะว่างเปล่าเสมอ (ซึ่งถูกต้องแล้ว เพราะการเก็บ
log ทุก query จะกิน memory มากใน production)

---

## ขั้นตอนที่ 122: `filter()` vs `exclude()` vs `get()` และ Field Lookups ทั้งหมด

### 122.1 ความแตกต่างพื้นฐานของสามเมธอดนี้

| เมธอด | คืนค่าอะไร | ใช้เมื่อไหร่ | พฤติกรรมถ้าไม่เจอ/เจอหลายรายการ |
|---|---|---|---|
| `filter()` | QuerySet (0 ถึงหลายรายการ) | ต้องการรายการที่ **ตรงเงื่อนไข** | คืน QuerySet ว่างเปล่า ถ้าไม่มีที่ตรงเงื่อนไข |
| `exclude()` | QuerySet (0 ถึงหลายรายการ) | ต้องการรายการที่ **ไม่ตรง**เงื่อนไข | คืน QuerySet ว่างเปล่า ถ้าทุกแถวตรงเงื่อนไข (ตัดหมด) |
| `get()` | **object เดียว** (ไม่ใช่ QuerySet) | ต้องการรายการเดียวที่ไม่ซ้ำ เช่น ค้นด้วย primary key | ยก `DoesNotExist` ถ้าไม่เจอ, ยก `MultipleObjectsReturned` ถ้าเจอมากกว่า 1 |

```python
# filter() - ปลอดภัยเสมอ ไม่ error แม้ไม่เจอข้อมูล
published_posts = Post.objects.filter(is_published=True)

# exclude() - ตรงข้ามกับ filter()
draft_posts = Post.objects.exclude(is_published=True)

# get() - ใช้เมื่อมั่นใจว่ามีแค่ 1 รายการ (เช่น primary key, slug ที่ unique)
try:
    post = Post.objects.get(slug="my-first-post")
except Post.DoesNotExist:
    post = None
```

**คำแนะนำระดับมืออาชีพ**: ใน view จริง เรามักใช้ `get_object_or_404()` แทนการ try/except
`get()` เอง (เราสอนไปแล้วใน Part 007) เพราะกระชับกว่าและคืน HTTP 404 ให้อัตโนมัติ:

```python
from django.shortcuts import get_object_or_404

def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, "blog/post_detail.html", {"post": post})
```

### 122.2 Field Lookups คืออะไร

**Field Lookup** คือ syntax พิเศษที่ Django ใช้แปลเงื่อนไข Python ให้กลายเป็นเงื่อนไข SQL
โดยเขียนในรูปแบบ `field__lookuptype=value` (ใช้ underscore คู่ `__` คั่นระหว่างชื่อ field
กับชื่อ lookup)

```python
Post.objects.filter(title__icontains="django")
#                    ^field  ^lookup    ^value
```

### 122.3 ตารางสรุป Field Lookups ทั้งหมดที่ใช้บ่อยที่สุด

| Lookup | ความหมาย | ตัวอย่าง | SQL ที่ได้ (ประมาณ, SQLite) |
|---|---|---|---|
| `__exact` | เท่ากับพอดี (ค่าเริ่มต้นถ้าไม่ระบุ lookup) | `Post.objects.filter(title__exact="Hello")` | `WHERE title = 'Hello'` |
| `__iexact` | เท่ากับ แบบไม่สนตัวพิมพ์เล็ก-ใหญ่ | `Post.objects.filter(title__iexact="hello")` | `WHERE title ILIKE 'hello'` (Postgres) |
| `__contains` | มีข้อความนี้อยู่ภายใน (case-sensitive) | `Post.objects.filter(content__contains="Django")` | `WHERE content LIKE '%Django%'` |
| `__icontains` | เหมือน contains แต่ไม่สนตัวพิมพ์ | `Post.objects.filter(content__icontains="django")` | `WHERE content ILIKE '%django%'` |
| `__in` | อยู่ในลิสต์ที่กำหนด | `Post.objects.filter(id__in=[1, 2, 3])` | `WHERE id IN (1, 2, 3)` |
| `__gt` | มากกว่า (greater than) | `Post.objects.filter(id__gt=10)` | `WHERE id > 10` |
| `__gte` | มากกว่าหรือเท่ากับ | `Post.objects.filter(id__gte=10)` | `WHERE id >= 10` |
| `__lt` | น้อยกว่า | `Post.objects.filter(id__lt=10)` | `WHERE id < 10` |
| `__lte` | น้อยกว่าหรือเท่ากับ | `Post.objects.filter(id__lte=10)` | `WHERE id <= 10` |
| `__startswith` | ขึ้นต้นด้วยข้อความนี้ (case-sensitive) | `Post.objects.filter(title__startswith="Django")` | `WHERE title LIKE 'Django%'` |
| `__istartswith` | เหมือน startswith แต่ไม่สนตัวพิมพ์ | `Post.objects.filter(title__istartswith="django")` | `WHERE title ILIKE 'django%'` |
| `__endswith` | ลงท้ายด้วยข้อความนี้ | `Post.objects.filter(title__endswith="Guide")` | `WHERE title LIKE '%Guide'` |
| `__range` | อยู่ระหว่างค่าสองค่า (inclusive ทั้งสองด้าน) | `Post.objects.filter(created_at__range=(start, end))` | `WHERE created_at BETWEEN start AND end` |
| `__isnull` | เป็น NULL หรือไม่ (รับ `True`/`False`) | `Post.objects.filter(category__isnull=True)` | `WHERE category_id IS NULL` |
| `__date` | ดึงเฉพาะส่วนวันที่จาก DateTimeField | `Post.objects.filter(created_at__date=date(2026, 1, 1))` | `WHERE DATE(created_at) = '2026-01-01'` |
| `__year` | ปีของ DateField/DateTimeField | `Post.objects.filter(created_at__year=2026)` | `WHERE strftime('%Y', created_at) = '2026'` |
| `__month` | เดือน | `Post.objects.filter(created_at__month=9)` | - |
| `__day` | วันที่ (1-31) | `Post.objects.filter(created_at__day=26)` | - |
| `__week_day` | วันในสัปดาห์ (1=อาทิตย์ ... 7=เสาร์) | `Post.objects.filter(created_at__week_day=1)` | - |
| `__regex` | ตรงกับ regular expression | `Post.objects.filter(title__regex=r"^Django \d+")` | - |
| `__iregex` | เหมือน regex แต่ไม่สนตัวพิมพ์ | `Post.objects.filter(title__iregex=r"^django")` | - |

### 122.4 ตัวอย่างการใช้งานจริงกับโมเดล blog

```python
from datetime import date, timedelta
from django.utils import timezone
from blog.models import Post

# หาโพสต์ที่ชื่อขึ้นต้นด้วย "Django" ไม่สนตัวพิมพ์
Post.objects.filter(title__istartswith="django")

# หาโพสต์ที่เผยแพร่แล้วและอยู่ใน category ที่กำหนด (ผ่าน FK)
Post.objects.filter(is_published=True, category__slug="tutorials")

# หาโพสต์ที่ id อยู่ในกลุ่มที่กำหนด (มีประโยชน์มากตอนทำ bulk action ใน admin)
Post.objects.filter(id__in=[3, 7, 15, 22])

# หาโพสต์ที่ยังไม่มีหมวดหมู่ (category เป็น NULL เพราะ on_delete=SET_NULL)
Post.objects.filter(category__isnull=True)

# หาโพสต์ที่สร้างในช่วง 7 วันที่ผ่านมา
last_week = timezone.now() - timedelta(days=7)
Post.objects.filter(created_at__gte=last_week)

# หาโพสต์ที่สร้างในปี 2026 เดือน 9 เท่านั้น
Post.objects.filter(created_at__year=2026, created_at__month=9)

# หาโพสต์ที่สร้างระหว่างสองช่วงเวลา
Post.objects.filter(
    created_at__range=(date(2026, 1, 1), date(2026, 12, 31))
)

# ใช้ exclude() ร่วมกับ lookup เพื่อกลับตรรกะ
# หาโพสต์ที่ "ไม่ได้" อยู่ในหมวด "draft-only"
Post.objects.exclude(category__slug="draft-only")

# Chain filter + exclude เข้าด้วยกัน
Post.objects.filter(is_published=True).exclude(author__username="ghostwriter")
```

### 122.5 การข้าม relationship ด้วย `__` (double underscore)

จุดที่ทรงพลังมากของ field lookup คือการใช้ `__` ข้ามความสัมพันธ์ (relationship) ได้ด้วย
ไม่ใช่แค่ใช้กับ lookup type เท่านั้น:

```python
# ข้าม ForeignKey ไปยัง field ของโมเดลที่เชื่อมอยู่
Post.objects.filter(category__name="Tutorials")
Post.objects.filter(author__username="admin")
Post.objects.filter(author__profile__bio__icontains="developer")  # ข้ามได้หลายชั้น

# ข้ามไปยัง Many-to-Many
Post.objects.filter(tags__name="python")

# ข้ามแบบย้อนกลับ (reverse relation) ผ่าน related_name
Category.objects.filter(posts__is_published=True)   # หมวดหมู่ที่มีโพสต์เผยแพร่แล้ว
Post.objects.filter(comments__author="สมชาย")         # โพสต์ที่มีคอมเมนต์จากสมชาย
```

**ข้อควรระวังสำคัญ**: การ filter ผ่าน Many-to-Many หรือ reverse ForeignKey อาจทำให้เกิด
แถวซ้ำ (duplicate rows) เนื่องจาก SQL JOIN ถ้าโพสต์หนึ่งมีหลาย tag ที่ตรงเงื่อนไข ต้องใช้
`.distinct()` ช่วย:

```python
# ถ้าไม่ใส่ distinct() โพสต์ที่มีทั้ง tag "python" และ "django" อาจถูกนับซ้ำ
Post.objects.filter(tags__name__in=["python", "django"]).distinct()
```

---

## ขั้นตอนที่ 123: Chaining QuerySets และ QuerySet Caching

### 123.1 การ Chain คืออะไร

เพราะ `filter()`, `exclude()`, `order_by()` ฯลฯ ทุกตัวคืนค่าเป็น **QuerySet ใหม่**
(ไม่ได้แก้ไข QuerySet เดิม) เราจึงสามารถ **ต่อเมธอดกันเป็นทอด ๆ** ได้ (method chaining):

```python
posts = (
    Post.objects
    .filter(is_published=True)
    .exclude(category__isnull=True)
    .filter(tags__name="django")
    .order_by("-created_at")
    .distinct()
)
```

โค้ดข้างบนนี้ **ยังไม่ยิง SQL แม้แต่ครั้งเดียว** จนกว่าจะถูก evaluate (เช่น วนลูป หรือ
`list(posts)`) และเมื่อ evaluate Django จะรวมเงื่อนไขทั้งหมดเป็น **SQL statement เดียว**
ไม่ใช่ยิงทีละคำสั่งตามจำนวน `.filter()` ที่เขียน

### 123.2 QuerySet เป็น Immutable

สิ่งสำคัญที่ต้องเข้าใจ: QuerySet แต่ละก้อน**ไม่เปลี่ยนแปลงตัวเอง** (immutable) ทุกครั้งที่
เรียกเมธอดกรองข้อมูล Django จะ**สร้าง QuerySet ใหม่**ขึ้นมาเสมอ:

```python
qs1 = Post.objects.filter(is_published=True)
qs2 = qs1.filter(category__slug="tutorials")

# qs1 และ qs2 เป็นคนละ QuerySet กัน! qs1 ไม่ถูกแก้ไข
print(qs1 is qs2)   # False
```

ข้อผิดพลาดที่มือใหม่ทำบ่อยมาก คือคิดว่าการเรียก `.filter()` จะแก้ไข QuerySet เดิม:

```python
# ผิด! เพราะ .filter() ไม่แก้ไข qs แต่คืนค่าใหม่ ซึ่งถูกทิ้งไปเฉย ๆ
qs = Post.objects.all()
qs.filter(is_published=True)   # ผลลัพธ์นี้หายไป ไม่ได้เก็บไว้ที่ไหน!
print(qs.count())   # ยังคงนับโพสต์ทั้งหมด ไม่ใช่แค่ที่เผยแพร่แล้ว

# ถูกต้อง! ต้อง assign กลับ
qs = Post.objects.all()
qs = qs.filter(is_published=True)   # หรือเขียนรวดเดียว qs = Post.objects.filter(...)
```

### 123.3 QuerySet Caching คืออะไร

เมื่อ QuerySet ถูก evaluate ครั้งแรก Django จะเก็บผลลัพธ์ไว้ใน **internal cache** ของ
QuerySet นั้น (attribute `_result_cache`) การเรียกใช้ซ้ำใน**ก้อนโค้ดเดียวกัน**ที่อ้างถึง
**ตัวแปร QuerySet ตัวเดิม** จะไม่ยิง SQL ซ้ำ:

```python
qs = Post.objects.filter(is_published=True)

# ครั้งแรก: evaluate จริง ยิง SQL 1 ครั้ง แล้วเก็บผลลัพธ์ใน cache
for post in qs:
    print(post.title)

# ครั้งที่สอง: ใช้ค่าจาก cache! ไม่ยิง SQL ซ้ำ เพราะเป็น qs ตัวเดิม
for post in qs:
    print(post.title)
```

แต่ถ้าคุณสร้าง QuerySet ใหม่ (แม้เงื่อนไขจะเหมือนเดิมทุกประการ) cache จะไม่ถูกใช้ร่วมกัน
เพราะเป็นคนละออบเจกต์:

```python
# นี่คือ 2 QuerySet คนละตัว แม้เงื่อนไขจะเหมือนกัน! จะยิง SQL 2 ครั้ง
for post in Post.objects.filter(is_published=True):
    print(post.title)

for post in Post.objects.filter(is_published=True):
    print(post.title)
```

### 123.4 กับดักของ Caching: Slicing และ Indexing ไม่ใช้ cache เดียวกันเสมอไป

```python
qs = Post.objects.filter(is_published=True)

print(qs[0])    # ยิง SQL แบบ LIMIT 1 OFFSET 0 (ไม่ได้ evaluate ทั้ง QuerySet)
print(qs[1])    # ยิง SQL แบบ LIMIT 1 OFFSET 1 อีกครั้ง! เพราะ cache ยังไม่ถูกเติมเต็ม

# แต่ถ้า evaluate เต็ม QuerySet ก่อน (เช่นด้วย list() หรือวนลูป)
# การ index ครั้งต่อ ๆ ไปจะดึงจาก cache แทน ไม่ยิง SQL ใหม่
posts = list(qs)
print(posts[0])   # ไม่ยิง SQL แล้ว เพราะเป็น Python list ธรรมดา
```

**คำแนะนำระดับมืออาชีพ**: ถ้าต้องใช้ผลลัพธ์เดียวกันหลายครั้งในโค้ดเดียวกัน (เช่นใน
template หรือ loop ที่ซับซ้อน) ให้แปลงเป็น `list()` ไว้ล่วงหน้าครั้งเดียว เพื่อบังคับให้
เกิด cache แน่นอน และหลีกเลี่ยงการยิง query ซ้ำโดยไม่ตั้งใจ:

```python
def post_list(request):
    # แปลงเป็น list ครั้งเดียว ป้องกันการยิง query ซ้ำถ้า template วนลูปหลายรอบ
    posts = list(Post.objects.filter(is_published=True).select_related("category"))
    return render(request, "blog/post_list.html", {
        "posts": posts,
        "total": len(posts),   # ใช้ len() ของ list แทน .count() เพื่อไม่ยิง query ซ้ำ
    })
```

---

## ขั้นตอนที่ 124: `order_by()` เจาะลึก

### 124.1 การเรียงลำดับพื้นฐาน

```python
# เรียงจากน้อยไปมาก (ascending) ตาม created_at
Post.objects.order_by("created_at")

# เรียงจากมากไปน้อย (descending) ใช้เครื่องหมาย - นำหน้าชื่อ field
Post.objects.order_by("-created_at")
```

### 124.2 เรียงหลาย Field พร้อมกัน

Django จะเรียงตาม field แรกก่อน ถ้าค่าเท่ากันจึงค่อยเรียงตาม field ถัดไป (เหมือน
`ORDER BY` หลายคอลัมน์ใน SQL):

```python
# เรียงตาม category ก่อน (ตามตัวอักษร) ถ้า category เดียวกัน ให้เรียงตามวันที่ล่าสุดก่อน
Post.objects.order_by("category__name", "-created_at")
```

### 124.3 เรียงแบบสุ่มด้วย `?`

```python
# สุ่มลำดับ - ใช้สำหรับ "บทความแนะนำ" หรือ Quiz ที่ต้องการสุ่มคำถาม
random_posts = Post.objects.filter(is_published=True).order_by("?")[:5]
```

**คำเตือนสำคัญด้าน Performance**: `order_by("?")` แปลเป็น `ORDER BY RANDOM()` (หรือ
`RAND()` ใน MySQL) ซึ่งเป็นหนึ่งใน query ที่ **ช้าที่สุด** เมื่อตารางมีข้อมูลจำนวนมาก
เพราะฐานข้อมูลต้องสุ่มค่าให้ **ทุกแถว** ก่อนจะเรียงและเลือกออกมา ในงาน production จริงที่มี
ข้อมูลหลักหมื่น-แสนแถว ควรใช้เทคนิคอื่นแทน เช่น สุ่ม id ด้วย Python ก่อนแล้วค่อย query
ด้วย `id__in`:

```python
import random
from blog.models import Post

published_ids = list(
    Post.objects.filter(is_published=True).values_list("id", flat=True)
)
random_ids = random.sample(published_ids, min(5, len(published_ids)))
random_posts = Post.objects.filter(id__in=random_ids)
```

### 124.4 เรียงลำดับด้วย `F()` Expression

`F()` ใช้อ้างอิงถึงค่าของ field อื่นในฐานข้อมูลโดยตรง โดยไม่ต้องดึงข้อมูลออกมาที่ฝั่ง
Python ก่อน (เราจะเจาะลึก `F()` เต็มรูปแบบใน Part 014 แต่ในบริบทของ `order_by()`
มีประโยชน์มากสำหรับการเรียงลำดับที่ซับซ้อน):

```python
from django.db.models import F

# เรียงตาม updated_at แต่ถ้า updated_at เป็น NULL ให้ถือว่ามีค่าน้อยที่สุด (อยู่ท้ายสุด)
Post.objects.order_by(F("updated_at").desc(nulls_last=True))

# เรียงตาม category_id น้อยไปมาก แต่ให้ค่า NULL ขึ้นก่อน
Post.objects.order_by(F("category").asc(nulls_first=True))
```

`F()` ยังมีประโยชน์เวลาต้องเรียงตามผลลัพธ์ของการคำนวณระหว่าง field (เช่น field คำนวณ
"คะแนนความนิยม" จากผลรวมของหลาย column) ซึ่งเราจะเห็นตัวอย่างเต็มรูปแบบเมื่อรวมกับ
`annotate()` ใน Part 014

### 124.5 ยกเลิกการเรียงลำดับที่สืบทอดมาจาก `Meta.ordering`

ถ้าโมเดลกำหนด `ordering` ไว้ใน `class Meta` เช่น:

```python
class Post(models.Model):
    # ... fields
    class Meta:
        ordering = ["-created_at"]
```

QuerySet ทุกตัวของโมเดลนี้จะเรียงตาม `-created_at` โดยอัตโนมัติ ถ้าต้องการยกเลิกการเรียง
(เช่นต้องการลำดับตามที่ฐานข้อมูลคืนมาตามธรรมชาติ ซึ่งเร็วกว่าเล็กน้อย) ให้เรียก
`order_by()` แบบไม่มี argument:

```python
Post.objects.order_by()   # ยกเลิก ordering ที่ Meta กำหนดไว้
```

---

## ขั้นตอนที่ 125: `values()` และ `values_list()`

### 125.1 ปัญหาที่ `values()`/`values_list()` แก้ไข

โดยปกติเมื่อคุณ query ด้วย `Post.objects.all()` Django จะคืนค่าเป็น **model instance**
เต็มรูปแบบ ซึ่งมีทุก field พร้อม method ต่าง ๆ ของโมเดล แต่บางครั้งเราต้องการแค่ "ข้อมูล
ดิบ" บาง field เท่านั้น (เช่น ทำ dropdown, export CSV, ส่ง JSON ให้ API) การสร้าง model
instance เต็มรูปแบบจึงเป็นการสิ้นเปลือง memory และเวลาโดยไม่จำเป็น

### 125.2 `values()`: คืนค่าเป็น dict

```python
>>> Post.objects.values("id", "title")
<QuerySet [{'id': 1, 'title': 'Hello Django'}, {'id': 2, 'title': 'ORM Basics'}]>
```

`values()` คืนค่าเป็น QuerySet ที่แต่ละแถวเป็น **dictionary** แทนที่จะเป็น model instance
ถ้าไม่ระบุ field ใด ๆ เลย จะคืนทุก field ในรูปแบบ dict:

```python
Post.objects.values()   # ทุก field แต่เป็น dict ไม่ใช่ instance
```

### 125.3 `values_list()`: คืนค่าเป็น tuple

```python
>>> Post.objects.values_list("id", "title")
<QuerySet [(1, 'Hello Django'), (2, 'ORM Basics')]>
```

### 125.4 `values_list(flat=True)`: ใช้เมื่อมี field เดียว

เมื่อดึงแค่ field เดียว การได้ tuple ที่มีสมาชิกตัวเดียวในแต่ละแถวจะดูรุงรัง (`(1,)`,
`(2,)`) `flat=True` ทำให้ได้ list ของค่าดิบตรง ๆ:

```python
>>> Post.objects.values_list("title", flat=True)
<QuerySet ['Hello Django', 'ORM Basics']>

# ใช้ประโยชน์บ่อยมากในการดึง id ทั้งหมดไปใช้ต่อ
published_ids = Post.objects.filter(is_published=True).values_list("id", flat=True)
Comment.objects.filter(post_id__in=published_ids)
```

**หมายเหตุ**: `flat=True` ใช้ได้เฉพาะกรณีที่ระบุ field เดียวเท่านั้น ถ้าระบุหลาย field
พร้อม `flat=True` จะเกิด `TypeError`

### 125.5 `values_list(named=True)`: ได้ named tuple

```python
>>> posts = Post.objects.values_list("id", "title", named=True)
>>> for post in posts:
...     print(post.id, post.title)   # เข้าถึงด้วยชื่อ field ได้เหมือน object!
1 Hello Django
2 ORM Basics
```

`named=True` คืนค่าเป็น `namedtuple` ทำให้เข้าถึงด้วย attribute name ได้ (`post.title`
แทนที่จะเป็น `post[1]`) ซึ่งอ่านง่ายกว่า tuple ธรรมดา แต่ยัง**เร็วกว่า** model instance
เต็มรูปแบบ เพราะไม่มี overhead ของ ORM methods

### 125.6 ตารางเปรียบเทียบ QuerySet ปกติ vs values() vs values_list()

| รูปแบบ | ชนิดผลลัพธ์แต่ละแถว | เข้าถึง method ของโมเดลได้ไหม | Memory | เหมาะกับ |
|---|---|---|---|---|
| QuerySet ปกติ (`.all()`) | Model instance | ✅ ได้เต็มรูปแบบ | สูงสุด | ต้องแก้ไข/บันทึกข้อมูล, ต้องใช้ method ของโมเดล |
| `.values()` | `dict` | ❌ ไม่ได้ | ปานกลาง | ส่งเป็น JSON, ทำ template ที่วนลูป key-value |
| `.values_list()` | `tuple` | ❌ ไม่ได้ | ต่ำกว่า `.values()` เล็กน้อย | export CSV, ทำ pandas DataFrame |
| `.values_list(flat=True)` | ค่าดิบ (ไม่ห่อ tuple) | ❌ ไม่ได้ | ต่ำที่สุด | ดึง list ของ id หรือค่าเดียว |
| `.values_list(named=True)` | `namedtuple` | ❌ ไม่ได้ (แต่เข้าถึงด้วยชื่อได้) | ปานกลาง | อ่านง่ายกว่า tuple แต่ยังเบากว่า instance |

### 125.7 ตัวอย่างการใช้งานจริง

```python
# ทำ dropdown ตัวเลือก category ใน form โดยไม่ต้องโหลด object เต็มรูปแบบ
category_choices = Category.objects.values_list("id", "name")

# Export ข้อมูลโพสต์เป็น list of dict เพื่อส่งเป็น JSON response
from django.http import JsonResponse

def post_api_list(request):
    posts = list(
        Post.objects.filter(is_published=True)
        .values("id", "title", "slug", "created_at")
    )
    return JsonResponse({"posts": posts}, safe=False if False else True)

# annotate ร่วมกับ values() เพื่อทำ group-by แบบมือโปร (เจาะลึกใน Part 014)
from django.db.models import Count

Post.objects.values("category__name").annotate(total=Count("id"))
```

---

## ขั้นตอนที่ 126: `only()` และ `defer()` สำหรับ optimize การดึง field

### 126.1 ปัญหา: ตารางที่มี field ขนาดใหญ่

สมมติโมเดล `Post` มี field `content` ที่เก็บเนื้อหาบทความยาวมาก ถ้าคุณแค่ต้องการแสดง
รายชื่อบทความ (title เท่านั้น) แต่ query ด้วย `Post.objects.all()` Django จะดึง **ทุก
field รวมถึง `content`** มาด้วยเสมอ ซึ่งอาจสิ้นเปลือง bandwidth และ memory มากถ้ามี
บทความหลายพันบทความ

### 126.2 `only()`: ระบุเฉพาะ field ที่ต้องการ

```python
# ดึงเฉพาะ id, title, slug (field อื่นจะถูก "defer" หรือดึงแบบ lazy เมื่อถูกเรียกใช้จริง)
posts = Post.objects.only("id", "title", "slug")

for post in posts:
    print(post.title)   # ไม่ยิง query เพิ่ม เพราะ title ถูกดึงมาแล้ว
    print(post.content) # ยิง query เพิ่ม 1 ครั้งต่อ post! เพราะ content ไม่ได้ถูกดึงมา
```

**ผลลัพธ์ยังคงเป็น model instance เต็มรูปแบบ** (ต่างจาก `values()`) เพียงแต่ field ที่ไม่
ได้ระบุใน `only()` จะถูกดึงแบบ **lazy** — คือถ้าไม่เคยเข้าถึง ก็ไม่มี query เพิ่ม แต่ถ้า
เข้าถึง (`post.content`) Django จะยิง query แยกไปดึง field นั้นทันที (เรียกว่า
**deferred loading**)

**ข้อควรระวัง**: ถ้าใช้ `only()` แล้วเข้าถึง field ที่ไม่ได้ระบุใน loop จะกลายเป็นปัญหา
N+1 query ทันที (query เพิ่ม 1 ครั้งต่อ 1 instance) ดังนั้น `only()` เหมาะกับกรณีที่
มั่นใจว่าจะไม่เข้าถึง field อื่นเลย

### 126.3 `defer()`: ตรงข้ามกับ `only()`

`defer()` ระบุ field ที่ **ไม่ต้องการ** ดึงมาก่อน (ดึงทุก field ยกเว้นที่ระบุ):

```python
# ดึงทุก field ยกเว้น content (field ขนาดใหญ่)
posts = Post.objects.defer("content")

for post in posts:
    print(post.title)     # ใช้ได้ปกติ ไม่มี query เพิ่ม
    print(post.content)   # ยิง query เพิ่มเมื่อเข้าถึง (deferred)
```

### 126.4 `only()` vs `defer()` เทียบกัน

| เมธอด | หลักการ | เหมาะกับ |
|---|---|---|
| `only("a", "b")` | ดึงเฉพาะ `a`, `b` (field อื่น defer ทั้งหมด) | เมื่อรู้ชัดว่าต้องการ field น้อยตัว |
| `defer("x", "y")` | ดึงทุก field ยกเว้น `x`, `y` | เมื่อต้องการเกือบทุก field ยกเว้นตัวใหญ่ไม่กี่ตัว (เช่น `content`, `raw_html`) |

### 126.5 การใช้ `only()`/`defer()` ร่วมกับ `select_related()`

ระวังจุดที่มือใหม่พลาดบ่อย: ถ้าใช้ `only()` ร่วมกับ `select_related()` ต้องระบุ field ของ
โมเดลที่เชื่อมด้วย ไม่งั้น Django จะดึง field ของโมเดลที่เชื่อมมาทั้งหมดอยู่ดี:

```python
# ดึง Post พร้อม category (join) แต่จำกัด field ทั้งสองฝั่ง
posts = Post.objects.select_related("category").only(
    "title", "slug", "category__name"
)
```

### 126.6 เมื่อไหร่ควรใช้ `only()`/`defer()` จริง ๆ

- ใช้เมื่อโมเดลมี field ขนาดใหญ่มาก (TextField ยาว, JSONField ขนาดใหญ่) และหน้าที่แสดงผล
  ไม่ต้องใช้ field นั้น
- **ไม่ควรใช้พร่ำเพรื่อ** กับทุก query เพราะเพิ่มความซับซ้อนของโค้ดโดยไม่จำเป็น ถ้าตาราง
  มี field ไม่กี่ตัวและขนาดเล็ก ประโยชน์ที่ได้จะน้อยมากเมื่อเทียบกับความเสี่ยงเรื่อง N+1
  query จากการลืมว่า field ไหนถูก defer ไว้
- ในงานจริง เรามักใช้ `values()`/`values_list()` แทนถ้าไม่ต้องการ method ของโมเดลเลย
  และใช้ `only()`/`defer()` เฉพาะกรณีที่ต้องการ **ทั้ง instance และ field บางส่วน**
  พร้อมกัน

---

## ขั้นตอนที่ 127: `exists()`, `count()`, `first()`, `last()`

### 127.1 ปัญหาของ `len(queryset) > 0`

มือใหม่มักเขียนโค้ดตรวจสอบว่ามีข้อมูลหรือไม่แบบนี้:

```python
# ไม่ควรทำ! ต้องดึงข้อมูลทั้งหมดมาก่อนถึงจะนับได้
posts = Post.objects.filter(is_published=True)
if len(posts) > 0:
    print("มีโพสต์ที่เผยแพร่แล้ว")
```

`len(posts)` บังคับให้ Django ดึง **ทุกแถว ทุก field** ของผลลัพธ์ที่ตรงเงื่อนไขออกมาสร้าง
เป็น model instance ทั้งหมดก่อน แล้วค่อยนับจำนวน ทั้งที่จริง ๆ เราแค่ต้องการรู้ว่า "มี
หรือไม่มี" เท่านั้น

### 127.2 `exists()`: ตรวจสอบว่ามีข้อมูลหรือไม่ แบบเร็วที่สุด

```python
if Post.objects.filter(is_published=True).exists():
    print("มีโพสต์ที่เผยแพร่แล้ว")
```

`exists()` แปลเป็น SQL ประมาณ `SELECT 1 FROM ... WHERE ... LIMIT 1` — ฐานข้อมูลหยุด
ค้นหาทันทีที่เจอแถวแรก ไม่ต้องนับหรือดึงข้อมูลทั้งหมด **เร็วกว่า `len(qs) > 0` มาก**
โดยเฉพาะเมื่อตารางมีข้อมูลจำนวนมาก

### 127.3 `count()`: นับจำนวนแถว แบบเร็วที่สุด

```python
total_published = Post.objects.filter(is_published=True).count()
```

`count()` แปลเป็น `SELECT COUNT(*) FROM ...` ให้ฐานข้อมูลนับให้โดยตรง **ไม่ต้องดึง
ข้อมูลจริงมาที่ฝั่ง Python เลย** เร็วกว่า `len(list(qs))` มากเมื่อมีข้อมูลจำนวนมาก

**ข้อยกเว้นสำคัญ**: ถ้า QuerySet ถูก evaluate (มี cache) ไปแล้วก่อนหน้านี้ Django จะฉลาด
พอที่จะนับจาก cache แทนที่จะยิง query `COUNT(*)` ซ้ำ:

```python
qs = Post.objects.filter(is_published=True)
list(qs)          # evaluate แล้ว มี cache
qs.count()        # ใช้ cache นับ ไม่ยิง SQL ใหม่! (ต่างจาก .exists() ที่ยิงใหม่เสมอ)
```

### 127.4 `first()` และ `last()`

```python
# ดึงแถวแรกตามลำดับปัจจุบัน (หรือ default ordering ของ Meta)
newest_post = Post.objects.order_by("-created_at").first()

# ดึงแถวสุดท้าย
oldest_post = Post.objects.order_by("-created_at").last()
```

`first()`/`last()` คืนค่าเป็น **object เดียว หรือ `None`** ถ้าไม่มีข้อมูล (ไม่ยก
Exception เหมือน `get()`) เบื้องหลังจะแปลงเป็น SQL แบบ `LIMIT 1` เสมอ (Django จัดการ
กลับด้าน ordering ให้อัตโนมัติสำหรับ `last()`) ทำให้เร็วกว่าการเขียน
`qs.order_by(...)[0]` ที่อาจพลาด `IndexError` ถ้า QuerySet ว่างเปล่า

### 127.5 ตารางสรุปเปรียบเทียบ Performance

| เมธอด | SQL ที่ได้ | ความเร็วเมื่อข้อมูลเยอะ | คืนค่าอะไร |
|---|---|---|---|
| `exists()` | `SELECT 1 ... LIMIT 1` | เร็วที่สุด สำหรับเช็คว่ามีหรือไม่ | `True`/`False` |
| `count()` | `SELECT COUNT(*) ...` | เร็วมาก สำหรับนับจำนวน | `int` |
| `first()`/`last()` | `SELECT ... LIMIT 1` | เร็วมาก สำหรับดึง 1 แถว | object หรือ `None` |
| `len(qs)` | ดึงทุกแถวก่อนแล้วนับใน Python | **ช้าที่สุด** ถ้าข้อมูลเยอะ | `int` |
| `qs[0]` (ไม่มี evaluate มาก่อน) | `SELECT ... LIMIT 1 OFFSET 0` | เร็ว แต่ยก `IndexError` ถ้าว่าง | object |

**กฎทองที่ควรจำ**: ใช้ `exists()` แทน `len() > 0` หรือ `if qs:` เสมอเมื่อแค่ต้องการเช็คว่า
"มีข้อมูลหรือไม่" และใช้ `count()` แทนการดึงข้อมูลมานับเองเสมอเมื่อแค่ต้องการ "จำนวน"

---

## ขั้นตอนที่ 128: `bulk_create()`, `bulk_update()`, `in_bulk()`

### 128.1 ปัญหาของการสร้างหลาย record ด้วยลูป

```python
# ไม่ควรทำถ้าต้องสร้างหลายร้อย/พันรายการ! ยิง query 1 ครั้งต่อ 1 การ .save()
tags_data = ["python", "django", "orm", "backend", "web"]
for name in tags_data:
    Tag.objects.create(name=name, slug=name)
# ถ้ามี 1,000 รายการ = ยิง SQL 1,000 ครั้ง! ช้ามาก
```

### 128.2 `bulk_create()`: สร้างหลาย record ในคำสั่งเดียว

```python
from blog.models import Tag

tags = [
    Tag(name="Python", slug="python"),
    Tag(name="Django", slug="django"),
    Tag(name="ORM", slug="orm"),
    Tag(name="Backend", slug="backend"),
    Tag(name="Web", slug="web"),
]
Tag.objects.bulk_create(tags)
# ยิง SQL แค่ 1 (หรือไม่กี่ครั้งถ้าข้อมูลเยอะมากจนต้องแบ่ง batch) คำสั่ง INSERT
```

**ตัวเลือกสำคัญของ `bulk_create()`**:

```python
Tag.objects.bulk_create(
    tags,
    batch_size=500,          # แบ่งการ insert เป็นชุด ๆ ละ 500 (สำคัญมากถ้าข้อมูลหลักหมื่น)
    ignore_conflicts=True,   # ข้ามรายการที่ชนกับ unique constraint แทนที่จะ error
)
```

**ข้อจำกัดสำคัญที่ต้องรู้ (คำเตือน)**:

- `bulk_create()` **ไม่เรียก** `save()` ของแต่ละ instance ดังนั้น logic ที่เขียนไว้ใน
  `def save(self, *args, **kwargs):` ที่ override จะ**ไม่ทำงาน**
- **ไม่ส่ง signal** `pre_save`/`post_save` (เราจะเรียนเรื่อง signals เต็มรูปแบบใน
  Part 019) ถ้าโค้ดส่วนอื่นพึ่งพา signal เหล่านี้ (เช่น สร้าง log อัตโนมัติ) จะไม่ทำงาน
- ถ้าโมเดลใช้ `AutoField` เป็น primary key และฐานข้อมูลรองรับ (PostgreSQL, SQLite รุ่นใหม่)
  `pk` ของแต่ละ object จะถูกเซ็ตกลับมาให้หลังสร้างเสร็จ แต่กับ MySQL รุ่นเก่าอาจไม่ได้
- ใช้กับ Many-to-Many field ไม่ได้โดยตรง (ต้อง `bulk_create()` ตัว object หลักก่อน แล้ว
  ค่อยจัดการ M2M ทีหลัง)

### 128.3 `bulk_update()`: อัปเดตหลาย record ในคำสั่งเดียว

```python
# สมมติต้องการเพิ่ม prefix "[Archived] " ให้ทุกโพสต์ที่เก่ากว่า 2 ปี
posts = list(Post.objects.filter(created_at__year__lt=2024))
for post in posts:
    post.title = f"[Archived] {post.title}"

# อัปเดตทั้งหมดในคำสั่งเดียว โดยระบุว่า field ไหนที่เปลี่ยน
Post.objects.bulk_update(posts, ["title"], batch_size=500)
```

`bulk_update()` ต้องการ argument 2 ตัวหลัก: list ของ instance ที่แก้ไขค่าไว้แล้ว (ใน
หน่วยความจำ ยังไม่ได้ save) และ list ชื่อ field ที่ต้องการอัปเดต เช่นเดียวกับ
`bulk_create()` เมธอดนี้ **ไม่เรียก `save()`** และ **ไม่ส่ง signal** เช่นกัน

### 128.4 `in_bulk()`: ดึงหลาย record มาเป็น dict โดย key คือ primary key

```python
# ต้องการ lookup post หลายอันด้วย id อย่างรวดเร็ว
posts_by_id = Post.objects.in_bulk([1, 5, 9, 12])
# ผลลัพธ์: {1: <Post: ...>, 5: <Post: ...>, 9: <Post: ...>, 12: <Post: ...>}

post_5 = posts_by_id.get(5)
```

`in_bulk()` เหมาะมากเมื่อคุณมี list ของ id (เช่นจาก M2M field หรือ external API) แล้ว
ต้องการดึง object ที่ตรงกันมาเป็น dict เพื่อ lookup แบบ O(1) แทนที่จะวนลูป query ทีละตัว
ถ้าไม่ระบุ argument ใด ๆ จะดึงทุกแถวในตาราง (ระวังกับตารางขนาดใหญ่):

```python
# ระบุ field อื่นที่ไม่ใช่ primary key เป็น key ของ dict ก็ได้
tags_by_slug = Tag.objects.in_bulk(field_name="slug")
django_tag = tags_by_slug.get("django")
```

### 128.5 ตารางสรุปเมธอดจัดการหลาย record

| เมธอด | ใช้ทำอะไร | เรียก `save()`/signal ไหม | หมายเหตุสำคัญ |
|---|---|---|---|
| `bulk_create()` | สร้างหลาย record | ❌ ไม่เรียก | ใช้ `batch_size` เมื่อข้อมูลเยอะ |
| `bulk_update()` | อัปเดตหลาย record ที่มี pk อยู่แล้ว | ❌ ไม่เรียก | ต้องระบุ list field ที่จะอัปเดต |
| `in_bulk()` | ดึงหลาย record เป็น dict (key=pk) | - (เป็นการอ่าน) | เร็วกว่าวนลูป `.get()` ทีละตัวมาก |

---

## ขั้นตอนที่ 129: `update()` และ `delete()` แบบ QuerySet-level

### 129.1 ความแตกต่างระหว่าง instance-level กับ QuerySet-level

เราคุ้นเคยกับการอัปเดต/ลบข้อมูลทีละ instance:

```python
# Instance-level: เรียก .save() / .delete() บน object เดียว
post = Post.objects.get(pk=1)
post.is_published = True
post.save()          # เรียก save() → ส่ง signal pre_save/post_save
post.delete()        # เรียก delete() → ส่ง signal pre_delete/post_delete
```

แต่ Django ยังมี **QuerySet-level** `update()` และ `delete()` ที่ทำงานกับ**หลายแถวพร้อม
กัน** ในคำสั่งเดียว โดยไม่ต้องโหลด instance ขึ้นมาก่อน:

```python
# QuerySet-level: อัปเดตทุกแถวที่ตรงเงื่อนไข ในคำสั่ง SQL เดียว
Post.objects.filter(category__isnull=True).update(is_published=False)

# QuerySet-level: ลบทุกแถวที่ตรงเงื่อนไข ในคำสั่ง SQL เดียว
Comment.objects.filter(text__icontains="spam").delete()
```

### 129.2 `update()`: อัปเดตหลายแถวในคำสั่งเดียว

```python
# เผยแพร่ทุกโพสต์ในหมวด "tutorials" พร้อมกัน
updated_count = Post.objects.filter(category__slug="tutorials").update(
    is_published=True
)
print(f"อัปเดตไปทั้งหมด {updated_count} แถว")
```

`update()` คืนค่าเป็น**จำนวนแถวที่ถูกอัปเดต** (ไม่ใช่ QuerySet) และแปลเป็น SQL แบบ
`UPDATE ... SET ... WHERE ...` เพียงคำสั่งเดียว ไม่ว่าจะมีกี่แถวก็ตาม

### 129.3 คำเตือนสำคัญ: `update()` bypass `.save()` และ Signals

**นี่คือจุดที่อันตรายที่สุดของ Part นี้ ต้องเข้าใจให้ชัดเจน:**

```python
class Post(models.Model):
    # ... fields
    updated_at = models.DateTimeField(auto_now=True)

    def save(self, *args, **kwargs):
        # สมมติมี logic พิเศษตอน save เช่น สร้าง slug อัตโนมัติ
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

```python
# วิธีนี้: logic ใน save() ทำงานปกติ, auto_now อัปเดตอัตโนมัติ, signal ถูกส่ง
post = Post.objects.get(pk=1)
post.title = "หัวข้อใหม่"
post.save()   # ✅ slug ถูกสร้างใหม่ให้, updated_at ถูกอัปเดต, post_save signal ทำงาน

# วิธีนี้: อันตราย! .update() ไม่เรียก save() เลย
Post.objects.filter(pk=1).update(title="หัวข้อใหม่")
# ❌ logic สร้าง slug ใน save() ไม่ทำงาน
# ❌ auto_now=True ของ updated_at "ไม่ถูกอัปเดตอัตโนมัติ" (ต้องระบุเองใน update())
# ❌ signal pre_save / post_save ไม่ถูกส่งเลย
```

สรุปสิ่งที่ `update()` (และ `delete()`) **ข้าม (bypass)** ไปทั้งหมด:

| สิ่งที่ถูกข้าม | ผลกระทบ |
|---|---|
| `Model.save()` ที่ override ไว้ | Custom logic เช่น auto-generate slug จะไม่ทำงาน |
| `pre_save` / `post_save` signal | Signal receiver ที่ผูกกับโมเดลนี้จะไม่ถูกเรียก |
| `auto_now=True` | ต้องระบุค่าฟิลด์นั้นเองใน `update()` ถ้าต้องการให้อัปเดต |
| Validation (`full_clean()`) | `update()` ไม่รัน validation ใด ๆ เลย ข้อมูลผิดพลาดอาจเข้าฐานข้อมูลได้ |
| `pre_delete` / `post_delete` signal (สำหรับ `.delete()`) | Cascade logic ที่พึ่ง signal จะไม่ทำงาน (แต่ `on_delete` ระดับ DB constraint ยังทำงานตามปกติ) |

**วิธีแก้ถ้าต้องการอัปเดต `updated_at` พร้อมกับ `update()`**:

```python
from django.utils import timezone

Post.objects.filter(category__slug="tutorials").update(
    is_published=True,
    updated_at=timezone.now(),   # ต้องระบุเอง เพราะ auto_now ไม่ทำงานกับ update()
)
```

### 129.4 `delete()` แบบ QuerySet-level และผลกระทบต่อ CASCADE

```python
# ลบทุกคอมเมนต์ของโพสต์ที่ถูกลบไปแล้ว (สมมติสถานการณ์)
deleted_count, details = Comment.objects.filter(post__isnull=True).delete()
print(deleted_count)   # จำนวนแถวทั้งหมดที่ถูกลบ (รวมทุกตารางที่ cascade)
print(details)         # dict แยกตามตาราง เช่น {'blog.Comment': 5}
```

`delete()` คืนค่าเป็น **tuple** `(จำนวนแถวทั้งหมดที่ลบ, dict แยกตามโมเดล)` เพราะการลบ
อาจ **cascade** ไปยังตารางอื่นด้วย (ตาม `on_delete=models.CASCADE` ที่กำหนดไว้ใน FK)

```python
# ตัวอย่าง: ลบ Post จะ cascade ไปลบ Comment ที่ผูกอยู่ด้วย (เพราะ Comment.post ใช้ CASCADE)
result = Post.objects.filter(is_published=False).delete()
print(result)
# (12, {'blog.Comment': 8, 'blog.Post_tags': 3, 'blog.Post': 1})
```

**ข้อควรระวัง**: `on_delete=models.CASCADE` ที่ระดับฐานข้อมูล/Django ORM ยังคงทำงานปกติ
กับ `.delete()` แบบ QuerySet-level (เพราะเป็นกลไกของ foreign key ไม่ใช่ signal) แต่
**signal `pre_delete`/`post_delete` จะไม่ถูกส่งให้กับแต่ละ instance ที่ถูกลบ** ถ้ามีโค้ด
ที่พึ่งพา signal เหล่านี้ (เช่น ลบไฟล์รูปภาพที่แนบไว้เมื่อ record ถูกลบ) จะไม่ทำงานเมื่อลบ
ผ่าน QuerySet-level

### 129.5 เมื่อไหร่ควรใช้ QuerySet-level และเมื่อไหร่ควรใช้ instance-level

| สถานการณ์ | ควรใช้ |
|---|---|
| อัปเดต/ลบข้อมูลจำนวนมาก (หลักร้อย-พัน-หมื่นแถว) และไม่มี custom logic ที่ต้องพึ่งพา | QuerySet-level (`update()`/`delete()`) — เร็วกว่ามาก |
| ต้องการให้ signal ทำงาน (เช่น ส่งอีเมลแจ้งเตือนตอนโพสต์ถูกลบ) | Instance-level (`save()`/`delete()` ทีละตัว) |
| ต้องการให้ validation (`full_clean()`) ทำงานก่อนบันทึก | Instance-level เท่านั้น |
| มี custom `save()` ที่ generate ค่าอัตโนมัติ (เช่น slug) | Instance-level หรือคำนวณค่าที่ต้องการเองก่อนเรียก `update()` |
| ต้องการความเร็วสูงสุดสำหรับ batch job / management command | QuerySet-level หรือ `bulk_update()`/`bulk_create()` |

**คำแนะนำระดับมืออาชีพ**: ทีมงานมืออาชีพมักตั้งกฎในโค้ดว่า ถ้าโมเดลมี signal หรือ custom
`save()` logic ที่สำคัญต่อความถูกต้องของระบบ (เช่น audit log, การส่งอีเมล) จะ**ห้ามใช้**
`update()`/`delete()`/`bulk_create()`/`bulk_update()` กับโมเดลนั้นโดยเด็ดขาด หรือถ้าจำเป็น
ต้องใช้เพื่อความเร็ว ต้องเขียนโค้ดจัดการ side-effect เหล่านั้นเอง (เช่น เรียก signal
handler function ตรง ๆ ในลูปแยกต่างหาก) เราจะกลับมาพูดถึงเรื่องนี้อีกครั้งอย่างละเอียด
ใน Part 019 (Signals)

---

## ขั้นตอนที่ 130: สรุปและแบบฝึกหัด

### 130.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า QuerySet เป็น **Lazy** — ไม่ยิง SQL จนกว่าจะถูก evaluate ด้วยการ iterate,
  `list()`, `bool()`, `len()`, slicing แบบมี step, หรือ `repr()`
- ✅ รู้จักวิธีตรวจสอบ SQL จริงด้วย Django Debug Toolbar, `qs.query`, และ
  `connection.queries`
- ✅ เข้าใจความแตกต่างของ `filter()`, `exclude()`, `get()` และใช้ field lookups ได้ครบ
  ทุกตัว (`__exact`, `__icontains`, `__in`, `__gte`, `__range`, `__isnull`, `__year` ฯลฯ)
- ✅ เข้าใจการ chain QuerySet และกลไก caching ภายใน รวมถึงกับดักเรื่อง immutability
- ✅ ใช้ `order_by()` เรียงหลาย field, สุ่มด้วย `?`, และเรียงด้วย `F()` expression ได้
- ✅ เลือกใช้ `values()`/`values_list()` (พร้อม `flat=True`/`named=True`) แทน QuerySet
  ปกติเมื่อไม่ต้องการ model instance เต็มรูปแบบ
- ✅ ใช้ `only()`/`defer()` เพื่อ optimize การดึง field บางส่วน และเข้าใจความเสี่ยงเรื่อง
  N+1 query ที่ตามมา
- ✅ เลือกใช้ `exists()`, `count()`, `first()`, `last()` แทน `len(qs) > 0` เพื่อ
  performance ที่ดีกว่า
- ✅ จัดการข้อมูลจำนวนมากด้วย `bulk_create()`, `bulk_update()`, `in_bulk()`
- ✅ เข้าใจอย่างถ่องแท้ว่า `update()`/`delete()` ระดับ QuerySet **ข้าม** `save()`,
  validation, และ signals — และรู้ว่าเมื่อไหร่ควร/ไม่ควรใช้

### 130.2 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายได้ว่าทำไม `Post.objects.filter(...)` ไม่ยิง SQL ทันที และอะไรบ้างที่ทำให้มัน
      ถูก evaluate
- [ ] เขียน query ด้วย field lookup อย่างน้อย 8 ตัวจากตารางในขั้นตอนที่ 122 ได้โดยไม่ดู
      เอกสารอ้างอิง
- [ ] อธิบายความแตกต่างระหว่าง `values()`, `values_list()`, และ QuerySet ปกติได้
- [ ] ใช้ `exists()` และ `count()` แทน `len(qs) > 0` ได้อย่างถูกต้อง
- [ ] เขียน `bulk_create()` พร้อม `batch_size` และเข้าใจว่าทำไมมันไม่เรียก `save()`
- [ ] อธิบายคำเตือนเรื่อง `update()`/`delete()` ที่ bypass signal ให้เพื่อนร่วมทีมฟังได้
      อย่างชัดเจน
- [ ] รันคำสั่งทั้งหมดในบทนี้ผ่าน `python manage.py shell` กับข้อมูลตัวอย่างจริงในโปรเจกต์
      ของคุณอย่างน้อยครึ่งหนึ่ง

### 130.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (Field Lookups)**: เขียน query ต่อไปนี้โดยใช้โมเดล `Post`, `Comment`,
`Category`, `Tag` จาก Part นี้:

1. หาโพสต์ทั้งหมดที่ `title` มีคำว่า "django" อยู่ (ไม่สนตัวพิมพ์เล็ก-ใหญ่)
2. หาโพสต์ที่สร้างในเดือนปัจจุบัน (ใช้ `timezone.now()` ร่วมกับ `__year` และ `__month`)
3. หาคอมเมนต์ที่เป็น "reply" (คือมี `parent` ไม่เป็น NULL)
4. หาโพสต์ที่ไม่มี tag ใด ๆ เลย (ใช้ `tags__isnull=True`)
5. หาโพสต์ที่ id อยู่ในช่วง 10 ถึง 50 (ใช้ `__range`)

**แบบฝึกหัดที่ 2 (Optimize Query)**: เขียนฟังก์ชัน `get_post_titles_only()` ที่คืนค่าเป็น
list ของชื่อโพสต์ทั้งหมดที่เผยแพร่แล้ว โดยใช้ `values_list(flat=True)` แล้วเปรียบเทียบ
เวลาที่ใช้ (ด้วย `time.perf_counter()`) กับการเขียนแบบดึง QuerySet เต็มรูปแบบแล้ววนลูป
`.title` เอง เมื่อมีข้อมูลอย่างน้อย 1,000 โพสต์ในฐานข้อมูลทดสอบ

**แบบฝึกหัดที่ 3 (Bulk Operations)**: เขียนสคริปต์ (ใช้ Django shell หรือ management
command) ที่:
- สร้าง `Tag` จำนวน 100 รายการด้วย `bulk_create()` (ชื่อ `tag-1` ถึง `tag-100`)
- จากนั้นใช้ `bulk_update()` เปลี่ยนชื่อ tag ทุกตัวที่มีเลขคู่ให้มี suffix `" (even)"`
- สุดท้ายใช้ `in_bulk()` ดึง tag ที่มี id เท่ากับ `[1, 25, 50, 75, 100]` มาพิมพ์ชื่อออกมา

**แบบฝึกหัดที่ 4 (คำเตือนเรื่อง update()/delete() — ขั้นสูง)**: สมมติโมเดล `Post` มี
custom `save()` ที่ auto-generate `slug` จาก `title` ถ้ายังไม่มี slug จงเขียนโค้ดทดสอบ
เปรียบเทียบ 2 แบบ:
1. สร้าง `Post` โดยไม่ระบุ `slug` แล้วเรียก `.save()` ปกติ — สังเกตว่า `slug` ถูกสร้างให้
2. สร้าง `Post` หลายรายการด้วย `bulk_create()` โดยไม่ระบุ `slug` เลย — สังเกตว่าเกิดอะไรขึ้น
   (คำใบ้: จะเกิด error เพราะ `slug` เป็น unique field ที่ไม่ได้ถูกเติมค่าให้ และ
   `bulk_create()` ไม่เรียก `save()`) แล้วเขียนวิธีแก้ไขที่ถูกต้อง (generate slug เองก่อน
   ใส่ใน list ที่ส่งให้ `bulk_create()`)

### 130.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม `Post.objects.filter(is_published=True)` เขียนแบบนี้ถึงไม่ error แม้ไม่มีข้อมูล
ที่ตรงเงื่อนไขเลย แต่ `get()` กลับ error?**
A: เพราะ `filter()` ออกแบบมาให้คืนค่าเป็น **collection** (QuerySet) เสมอ ซึ่ง collection
ที่ไม่มีสมาชิกเลยก็ยังคงเป็น collection ที่ถูกต้อง (แค่ว่างเปล่า) แต่ `get()` ถูกออกแบบมา
ให้คืนค่าเป็น **object เดียวที่ชัดเจน** การไม่เจอเลยหรือเจอมากกว่า 1 จึงถือเป็นสถานการณ์
ที่ผิดปกติ (exceptional) และควรแจ้งด้วย Exception เพื่อให้นักพัฒนาจัดการอย่างชัดเจน

**Q: `only()` กับ `values()` ต่างกันตรงไหนกันแน่ ในเมื่อทั้งคู่ดึงแค่บาง field เหมือนกัน?**
A: `values()` คืนค่าเป็น **dict ธรรมดา** ไม่ใช่ model instance คุณจะเรียก method ของ
โมเดล (เช่น `post.get_absolute_url()`) หรือแก้ไขแล้ว `.save()` ไม่ได้เลย ส่วน `only()`
ยังคงคืนค่าเป็น **model instance เต็มรูปแบบ** ที่ใช้ method ได้ปกติ เพียงแต่ field ที่ไม่
ได้ระบุจะถูกดึงแบบ deferred (query เพิ่มถ้าถูกเข้าถึง) เลือกใช้ตามว่าต้องการ behavior
ของ model instance หรือแค่ข้อมูลดิบ

**Q: ถ้าอยากให้ `update()` ยังส่ง signal ได้ ต้องทำอย่างไร?**
A: Django ไม่มีตัวเลือกให้ `update()` ส่ง signal ได้โดยตรง (เพราะเป็นการออกแบบเพื่อความ
เร็วโดยเฉพาะ) ถ้าจำเป็นต้องมี logic ทำงานทุกครั้งที่มีการอัปเดต ต้องเลือกอย่างใดอย่างหนึ่ง:
(1) ใช้ instance-level `.save()` ในลูปแทน (ช้ากว่าแต่ signal ทำงาน) หรือ (2) เขียนโค้ด
ที่จำลอง side-effect ของ signal นั้นแยกต่างหาก แล้วเรียกหลังจาก `update()` เสร็จ เราจะ
เจาะลึกแนวทางออกแบบที่ดีกว่าเรื่องนี้ในการใช้ **custom Manager/QuerySet** ที่ Part 015

**Q: `bulk_create()` เร็วกว่าลูป `.create()` จริงหรือ เร็วกว่าเท่าไหร่?**
A: เร็วกว่ามาก โดยเฉพาะเมื่อข้อมูลมีจำนวนมาก เพราะลูป `.create()` ยิง SQL แยกกัน 1 ครั้ง
ต่อ 1 แถว (รวมทั้ง round-trip ไปกลับระหว่างแอปกับฐานข้อมูลทุกครั้ง) ในขณะที่
`bulk_create()` รวมทุกแถวเป็นคำสั่ง `INSERT` เดียว (หรือไม่กี่คำสั่งถ้าใช้ `batch_size`)
ในโปรเจกต์จริงที่วัดผล การสร้างข้อมูล 10,000 แถวด้วยลูปอาจใช้เวลาหลักสิบวินาที ในขณะที่
`bulk_create()` ใช้เวลาไม่ถึง 1 วินาทีในฐานข้อมูลเดียวกัน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 014: Aggregation, Annotation และ Q/F Expressions** จะพาไปเจาะลึกการคำนวณเชิงสถิติ
ในระดับฐานข้อมูลโดยตรง เช่น การนับจำนวนคอมเมนต์ต่อโพสต์ (`Count`), การหาค่าเฉลี่ย/ผลรวม
(`Avg`, `Sum`), การสร้าง field คำนวณชั่วคราวด้วย `annotate()`, การเขียนเงื่อนไข OR/AND
ที่ซับซ้อนด้วย `Q()` objects และการเปรียบเทียบค่าระหว่าง field ด้วย `F()` expression
แบบเต็มรูปแบบ (ต่อยอดจากที่เกริ่นไว้ใน `order_by()` ของ Part นี้) ทักษะเหล่านี้คือหัวใจของ
การทำ dashboard, รายงานสถิติ, และ feature อย่าง "โพสต์ยอดนิยม" ที่ต้องคำนวณจากข้อมูลจริง
ในฐานข้อมูล ไม่ใช่ดึงมาคำนวณเองใน Python

เตรียม `python manage.py shell` ให้พร้อม และลองสร้างข้อมูลตัวอย่างในแอป `blog` ให้มี
โพสต์ คอมเมนต์ และ tag หลากหลายไว้ล่วงหน้า เพราะ Part ถัดไปจะใช้ query ที่ซับซ้อนขึ้นมาก!
