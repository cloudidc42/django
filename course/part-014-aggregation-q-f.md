# Part 014: Django ORM Aggregation, Annotation และ Q/F Expressions

> **ขั้นตอนที่ 131-140 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เจาะลึกเครื่องมือ ORM ระดับสูงที่แยกนักพัฒนา Django มือใหม่
> ออกจากมืออาชีพ ได้แก่ `aggregate()`, `annotate()`, `Q()`, `F()`, `Case`/`When`,
> Subquery/`OuterRef`, Window functions และการ group by ด้วย `values().annotate()`
> เมื่อจบ Part นี้ คุณจะเขียนหน้า "Dashboard สถิติบล็อก" ที่คำนวณตัวเลขซับซ้อนได้ทั้งหมด
> ด้วย query เดียว โดยไม่ต้องดึงข้อมูลมาวนลูปนับเองใน Python

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 131: `aggregate()` เบื้องต้น — Count, Sum, Avg, Max, Min
- ขั้นตอนที่ 132: `annotate()` เบื้องต้น — เพิ่มค่าที่คำนวณแล้วให้แต่ละ object
- ขั้นตอนที่ 133: `aggregate()` vs `annotate()` — ความแตกต่างที่ต้องเข้าใจให้ทะลุ
- ขั้นตอนที่ 134: `Q()` objects — เงื่อนไข OR/AND/NOT ที่ซับซ้อน
- ขั้นตอนที่ 135: `F()` expressions — เปรียบเทียบ field และอัปเดตแบบ atomic
- ขั้นตอนที่ 136: `Case`/`When` — Conditional Annotation
- ขั้นตอนที่ 137: Subqueries ด้วย `OuterRef`/`Subquery` และ `Exists`
- ขั้นตอนที่ 138: Window Functions เบื้องต้น — Ranking ด้วย `Rank`
- ขั้นตอนที่ 139: Group By ด้วย `values().annotate()`
- ขั้นตอนที่ 140: สรุปและแบบฝึกหัด — สร้างหน้า Dashboard สถิติบล็อก

---

## ขั้นตอนที่ 131: `aggregate()` เบื้องต้น — Count, Sum, Avg, Max, Min

### 131.1 โมเดลอ้างอิงที่ใช้ตลอด Part นี้

ก่อนเริ่ม เรามาทบทวนโครงสร้างโมเดลของแอป `blog` ที่เราสร้างไว้ใน Part 011-013
(หากคุณยังไม่มีโค้ดนี้ในโปรเจกต์ ให้คัดลอกไปวางใน `blog/models.py` ก่อน เพราะทุกตัวอย่าง
ใน Part นี้จะอ้างอิงโมเดลชุดนี้ทั้งหมด):

```python
# blog/models.py
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
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name="posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    view_count = models.IntegerField(default=0)

    class Meta:
        ordering = ["-created_at"]

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

    class Meta:
        ordering = ["created_at"]

    def __str__(self):
        return f"Comment by {self.author} on {self.post_id}"


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile",
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)

    def __str__(self):
        return f"Profile of {self.user.username}"
```

จำโครงสร้างนี้ให้ขึ้นใจ: **1 Post มีได้หลาย Comment** (ForeignKey), **1 Post มีได้หลาย Tag
และ 1 Tag อยู่ในหลาย Post ได้** (ManyToMany), **Comment ตอบกลับ Comment อื่นได้**
(self-referential ForeignKey ชื่อ `parent`)

### 131.2 `aggregate()` คืออะไร

`aggregate()` คือเมธอดของ QuerySet ที่ **สรุปข้อมูลทั้งก้อนให้เหลือ "ค่าเดียว"**
(หรือหลายค่าที่เป็น dict) แทนที่จะคืนเป็น QuerySet ของ object หลายตัว

พูดง่าย ๆ: ถ้า `filter()` ตอบคำถามว่า "อยากได้ *object ไหนบ้าง*"
`aggregate()` ตอบคำถามว่า "อยากได้ *ตัวเลขสรุป* อะไรจากข้อมูลทั้งหมด"

ตัวอย่างคำถามที่ `aggregate()` ตอบได้:

- โพสต์ทั้งหมดในระบบมีกี่โพสต์?
- ค่าเฉลี่ยจำนวนคอมเมนต์ต่อโพสต์คือเท่าไร?
- โพสต์ที่มี view สูงสุดมี view เท่าไร?
- ผลรวมยอดวิวของทุกโพสต์คือเท่าไร?

### 131.3 Aggregate Functions ที่ Django มีให้ใช้

Django import มาจาก `django.db.models`:

| ฟังก์ชัน | ความหมาย | ตัวอย่างการใช้ |
|---|---|---|
| `Count` | นับจำนวนแถว/ความสัมพันธ์ | `Count('comments')` |
| `Sum` | ผลรวมของค่าตัวเลข | `Sum('view_count')` |
| `Avg` | ค่าเฉลี่ย | `Avg('view_count')` |
| `Max` | ค่าสูงสุด | `Max('view_count')` |
| `Min` | ค่าต่ำสุด | `Min('view_count')` |
| `StdDev` | ส่วนเบี่ยงเบนมาตรฐาน (ต้องรองรับโดย DB) | `StdDev('view_count')` |
| `Variance` | ความแปรปรวน (ต้องรองรับโดย DB) | `Variance('view_count')` |

### 131.4 ตัวอย่างจริง: นับจำนวนโพสต์ทั้งหมด

```python
# python manage.py shell
from django.db.models import Count, Sum, Avg, Max, Min
from blog.models import Post

Post.objects.aggregate(Count('id'))
# {'id__count': 25}

# หรือใช้ตัวย่อ Count('*') แบบไม่ต้องระบุ field
Post.objects.count()
# 25   <- เร็วกว่าและง่ายกว่าถ้าต้องการแค่จำนวนแถวเฉย ๆ
```

**ข้อสังเกตสำคัญ**: ถ้าแค่ต้องการนับจำนวนแถวทั้งหมดของ QuerySet เดียว ให้ใช้
`.count()` ตรง ๆ เพราะมันเร็วกว่าและอ่านง่ายกว่า `aggregate(Count(...))`
เราจะใช้ `aggregate(Count(...))` เมื่อต้องการค่าอื่นควบคู่ไปด้วย หรือ Count
ความสัมพันธ์ที่ซับซ้อนกว่านั้น

### 131.5 ตัวอย่างที่ใช้งานจริง: สถิติของบล็อกทั้งระบบ

```python
from django.db.models import Count, Sum, Avg, Max, Min
from blog.models import Post

stats = Post.objects.aggregate(
    total_posts=Count('id'),
    published_posts=Count('id', filter=models.Q(is_published=True)),
    total_views=Sum('view_count'),
    avg_views=Avg('view_count'),
    max_views=Max('view_count'),
    min_views=Min('view_count'),
)
print(stats)
# {
#     'total_posts': 25,
#     'published_posts': 18,
#     'total_views': 15420,
#     'avg_views': 616.8,
#     'max_views': 3200,
#     'min_views': 0,
# }
```

สังเกตว่า:

1. เราตั้งชื่อ key เองได้ทุกตัว (`total_posts=Count('id')`) — ถ้าไม่ตั้งชื่อ Django จะ
   สร้างชื่อ default ให้ เช่น `id__count`, `view_count__sum`
2. เราใส่ `filter=Q(...)` เข้าไปใน aggregate function ได้โดยตรง (Django 2.0+)
   เพื่อนับเฉพาะแถวที่ตรงเงื่อนไข โดยไม่ต้องเรียก `.filter()` แยก QuerySet
3. `aggregate()` คืนค่าเป็น **`dict` เดียว** เสมอ ไม่ใช่ QuerySet

### 131.6 ตัวอย่างสำคัญที่สุดของ Part นี้: จำนวนคอมเมนต์เฉลี่ยต่อโพสต์

นี่คือคำถามที่ตอบยากถ้าไม่มี ORM ที่ดี: **"เฉลี่ยแล้วแต่ละโพสต์มีคอมเมนต์กี่อัน?"**

```python
from django.db.models import Count, Avg
from blog.models import Post

# วิธีที่ 1: annotate() ก่อน แล้วค่อย aggregate() ทับ annotate นั้นอีกที
result = Post.objects.annotate(
    comment_count=Count('comments')
).aggregate(
    avg_comments_per_post=Avg('comment_count')
)
print(result)
# {'avg_comments_per_post': 3.4}
```

ทำไมต้องมี 2 ขั้นตอน (annotate ก่อน แล้ว aggregate)? เพราะเราต้อง **นับคอมเมนต์ต่อโพสต์
แต่ละอันก่อน** (นั่นคือหน้าที่ของ `annotate()`) แล้วจึง **หาค่าเฉลี่ยของตัวเลขที่นับได้นั้น**
(นั่นคือหน้าที่ของ `aggregate()`) — นี่คือกุญแจสำคัญที่จะอธิบายละเอียดในขั้นตอนที่ 133

### 131.7 aggregate() ที่ทำงานผ่านความสัมพันธ์ (Traversing Relationships)

`aggregate()` เดินทางผ่าน ForeignKey/ManyToMany ได้ด้วยเครื่องหมาย `__` เหมือน `filter()`:

```python
from django.db.models import Avg, Count
from blog.models import Category

# ค่าเฉลี่ย view_count ของโพสต์ทุกอันในหมวดหมู่ "Django" หมวดเดียว
category = Category.objects.get(slug='django')
category.posts.aggregate(Avg('view_count'))
# {'view_count__avg': 842.5}

# นับจำนวนแท็กที่แตกต่างกันทั้งหมดที่ถูกใช้ในระบบ
from blog.models import Post
Post.objects.aggregate(distinct_tags=Count('tags', distinct=True))
# {'distinct_tags': 12}
```

**สำคัญมาก**: เมื่อ `aggregate()`/`annotate()` เดินทางผ่านความสัมพันธ์แบบ many-to-many
หรือ reverse foreign key ที่ทำให้เกิด JOIN แบบ "หนึ่งแถวกลายเป็นหลายแถว" (เช่น
1 โพสต์ที่มี 3 แท็ก จะกลายเป็น 3 แถวชั่วคราวตอน JOIN) ต้องใส่ `distinct=True`
มิฉะนั้นตัวเลขจะนับซ้ำผิดพลาด เราจะเจอปัญหานี้อีกครั้งในขั้นตอนที่ 132.6

---

## ขั้นตอนที่ 132: `annotate()` เบื้องต้น — เพิ่มค่าที่คำนวณแล้วให้แต่ละ object

### 132.1 `annotate()` คืออะไร

ถ้า `aggregate()` สรุปทั้ง QuerySet เป็นค่าเดียว `annotate()` จะ **เพิ่ม field ที่คำนวณแล้ว
ให้กับแต่ละ object ใน QuerySet** โดยที่ QuerySet ยังคงเป็น QuerySet ของหลาย object เหมือนเดิม
เพียงแต่แต่ละตัวมี attribute ใหม่เพิ่มเข้ามา

### 132.2 ตัวอย่างพื้นฐานที่สุด: นับคอมเมนต์ต่อโพสต์

```python
from django.db.models import Count
from blog.models import Post

posts = Post.objects.annotate(comment_count=Count('comments'))

for post in posts:
    print(f"{post.title}: {post.comment_count} comments")
# "แนะนำ Django 5" : 12 comments
# "ทำความรู้จัก ORM" : 5 comments
# "Deploy สู่ Production" : 0 comments
```

สิ่งที่เกิดขึ้นเบื้องหลัง: Django สร้าง SQL แบบ `LEFT OUTER JOIN` กับตาราง `blog_comment`
แล้ว `GROUP BY blog_post.id` พร้อม `COUNT(blog_comment.id)` ให้อัตโนมัติ — เราไม่ต้อง
เขียน SQL เองเลย

ตรวจสอบ SQL จริงที่ Django สร้างได้ด้วย:

```python
print(posts.query)
# SELECT "blog_post"."id", ..., COUNT("blog_comment"."id") AS "comment_count"
# FROM "blog_post"
# LEFT OUTER JOIN "blog_comment" ON ("blog_post"."id" = "blog_comment"."post_id")
# GROUP BY "blog_post"."id"
```

### 132.3 `comment_count` ใช้ต่อได้เหมือน field ปกติ

จุดที่ทรงพลังของ `annotate()` คือค่าที่ได้กลายเป็น "field เสมือน" ที่ใช้ `filter()`
และ `order_by()` ต่อได้ทันที:

```python
from django.db.models import Count
from blog.models import Post

# หาโพสต์ยอดนิยม เรียงตามจำนวนคอมเมนต์มากไปน้อย
popular_posts = Post.objects.annotate(
    comment_count=Count('comments')
).order_by('-comment_count')[:5]

# หาเฉพาะโพสต์ที่มีคอมเมนต์มากกว่า 10 อัน
active_posts = Post.objects.annotate(
    comment_count=Count('comments')
).filter(comment_count__gt=10)
```

### 132.4 `annotate()` หลายค่าพร้อมกันในคำสั่งเดียว

```python
from django.db.models import Count
from blog.models import Post

posts = Post.objects.annotate(
    comment_count=Count('comments', distinct=True),
    tag_count=Count('tags', distinct=True),
)

for post in posts:
    print(f"{post.title}: {post.comment_count} comments, {post.tag_count} tags")
```

### 132.5 ทำไมต้องใส่ `distinct=True` เมื่อ annotate หลาย relation พร้อมกัน

นี่คือหลุมพรางที่มือใหม่เจอบ่อยที่สุดของ `annotate()` ลองดูตัวอย่างที่ **ผิด**:

```python
# ผิด! ตัวเลขจะพองขึ้นแบบผิดปกติ
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    tag_count=Count('tags'),
)
```

**ทำไมถึงผิด**: เมื่อ annotate ทั้ง `comments` (reverse FK) และ `tags` (M2M) พร้อมกัน
Django จะสร้าง JOIN สองตารางพร้อมกัน ทำให้เกิด **Cartesian Product** ชั่วคราว —
ถ้าโพสต์หนึ่งมี 3 คอมเมนต์และ 4 แท็ก จะเกิดแถวชั่วคราว 3×4 = 12 แถวก่อน GROUP BY
ทำให้ `comment_count` กลายเป็น 12 (ผิด ควรเป็น 3) และ `tag_count` กลายเป็น 12
(ผิด ควรเป็น 4)

**วิธีแก้**: ใส่ `distinct=True` ในทุก `Count()` ที่ annotate หลาย relation พร้อมกัน:

```python
# ถูกต้อง
posts = Post.objects.annotate(
    comment_count=Count('comments', distinct=True),
    tag_count=Count('tags', distinct=True),
)
```

| สถานการณ์ | ต้องใส่ `distinct=True` หรือไม่ |
|---|---|
| `annotate()` relation เดียว | ไม่จำเป็น (แต่ใส่ไว้ก็ไม่ผิด ปลอดภัยไว้ก่อน) |
| `annotate()` มากกว่า 1 relation พร้อมกัน | **จำเป็นมาก** ไม่งั้นตัวเลขผิด |
| ใช้ `Sum`/`Avg` ร่วมกับ relation ที่ join ซ้อนกัน | ควรตรวจสอบด้วย `.query` เสมอ |

**คำแนะนำระดับมืออาชีพ**: เมื่อ annotate ค่าที่มาจากหลาย relation ให้ตรวจสอบผลลัพธ์ด้วยมือ
(เทียบกับ `post.comments.count()` ตรง ๆ) อย่างน้อยหนึ่งครั้งก่อนขึ้น production
หรือดีที่สุดคือเขียน test เทียบค่าเสมอ

### 132.6 annotate() ผ่านความสัมพันธ์ระดับลึก (Nested Relations)

```python
from django.db.models import Count
from blog.models import Category

# นับจำนวนโพสต์ในแต่ละหมวดหมู่ พร้อมนับจำนวนคอมเมนต์รวมของหมวดหมู่นั้น
categories = Category.objects.annotate(
    post_count=Count('posts', distinct=True),
    total_comments=Count('posts__comments', distinct=True),
)

for cat in categories:
    print(f"{cat.name}: {cat.post_count} posts, {cat.total_comments} comments")
```

สังเกตว่า `posts__comments` เดินทางจาก `Category` → `Post` (reverse FK) → `Comment`
(reverse FK อีกชั้น) ได้เลย

---

## ขั้นตอนที่ 133: `aggregate()` vs `annotate()` — ความแตกต่างที่ต้องเข้าใจให้ทะลุ

### 133.1 ตารางเปรียบเทียบหลัก

| ประเด็น | `aggregate()` | `annotate()` |
|---|---|---|
| ผลลัพธ์ที่ได้ | `dict` เดียว | `QuerySet` (ยังมีหลาย object เหมือนเดิม) |
| ความหมาย | สรุปข้อมูล**ทั้งก้อน**เป็นค่าเดียว | เพิ่ม field คำนวณให้**แต่ละ object** |
| ใช้ `filter()`/`order_by()` ต่อได้ไหม | ไม่ได้ (เพราะไม่ใช่ QuerySet แล้ว) | ได้ (ยังเป็น QuerySet) |
| SQL ที่เกิด | `SELECT AGG(...) FROM ...` (ไม่มี GROUP BY เว้นแต่จะ join) | `SELECT ..., AGG(...) FROM ... GROUP BY <pk>` |
| ตัวอย่างคำถาม | "เว็บนี้มีโพสต์รวมกี่โพสต์?" | "แต่ละโพสต์มีคอมเมนต์กี่อัน?" |
| ตำแหน่งในเชน (chain) | มักอยู่ท้ายสุดของ query | อยู่กลางเชนได้ ต่อ `.filter()`/`.order_by()` ได้อีก |

### 133.2 ตัวอย่างเปรียบเทียบข้างเคียงกัน

```python
from django.db.models import Count
from blog.models import Post

# ===== aggregate(): ได้ค่าเดียวสรุปทั้งระบบ =====
result = Post.objects.aggregate(total_comments=Count('comments'))
print(result)
# {'total_comments': 87}          <- ตัวเลขเดียว รวมคอมเมนต์ของ "ทุกโพสต์"

# ===== annotate(): ได้ QuerySet ที่แต่ละตัวมีตัวเลขของตัวเอง =====
posts = Post.objects.annotate(comment_count=Count('comments'))
for post in posts:
    print(post.title, post.comment_count)
# "แนะนำ Django 5" 12
# "ทำความรู้จัก ORM" 5
# ...
# (ผลรวมของตัวเลขทั้งหมดนี้ = 87 ซึ่งตรงกับผลลัพธ์ของ aggregate() ด้านบน)
```

### 133.3 การใช้สองตัวร่วมกัน: annotate() ก่อน แล้วค่อย aggregate() ทับ

เราเห็นแพทเทิร์นนี้แล้วในขั้นตอนที่ 131.6 นี่คือแพทเทิร์นที่พบบ่อยที่สุดในงานจริง:
**"หาค่าต่อแถวก่อน (annotate) แล้วสรุปค่าที่ได้อีกที (aggregate)"**

```python
from django.db.models import Count, Avg, Max
from blog.models import Post

# ค่าเฉลี่ย และค่าสูงสุด ของจำนวนคอมเมนต์ต่อโพสต์
stats = Post.objects.annotate(
    comment_count=Count('comments')
).aggregate(
    avg_comments=Avg('comment_count'),
    max_comments=Max('comment_count'),
)
print(stats)
# {'avg_comments': 3.48, 'max_comments': 12}
```

### 133.4 กฎจำง่าย ๆ

> **aggregate() ตอบคำถามระดับ "ทั้งตาราง" ส่วน annotate() ตอบคำถามระดับ "แต่ละแถว"**

ถ้าประโยคคำถามของคุณมีคำว่า "ทั้งหมด", "รวม", "เฉลี่ยของทั้งระบบ" → `aggregate()`
ถ้าประโยคคำถามของคุณมีคำว่า "แต่ละ...", "ของโพสต์นี้", "เรียงตาม..." → `annotate()`

---

## ขั้นตอนที่ 134: `Q()` objects — เงื่อนไข OR/AND/NOT ที่ซับซ้อน

### 134.1 ปัญหาของ `filter()` แบบธรรมดา

`filter(a=1, b=2)` ที่เราเรียนมาตลอด Part 013 มีข้อจำกัด: **มันรวมเงื่อนไขด้วย AND
เท่านั้นเสมอ** ไม่มีทางเขียน OR ได้ตรง ๆ

```python
# นี่คือ AND เท่านั้น: is_published=True AND category__slug='django'
Post.objects.filter(is_published=True, category__slug='django')
```

แล้วถ้าเราต้องการ **"โพสต์ที่เผยแพร่แล้ว หรือ โพสต์ที่มี view_count มากกว่า 1000"**
(สังเกตคำว่า "หรือ") จะเขียนอย่างไร? `filter()` ธรรมดาทำไม่ได้ ต้องใช้ `Q()`

### 134.2 `Q()` คืออะไร

`Q()` object คือการห่อเงื่อนไข filter ให้กลายเป็น "object" ที่นำมารวมกันด้วย
`|` (OR), `&` (AND), และ `~` (NOT) ได้เหมือนตรรกะบูลีน

```python
from django.db.models import Q
from blog.models import Post

# OR: เผยแพร่แล้ว หรือ ยอดวิวเกิน 1000
Post.objects.filter(
    Q(is_published=True) | Q(view_count__gt=1000)
)
```

### 134.3 `Q()` แบบ AND (เขียนได้ 2 แบบ ผลเหมือนกัน)

```python
from django.db.models import Q
from blog.models import Post

# แบบที่ 1: keyword arguments ปกติ (Django ใส่ AND ให้อัตโนมัติ)
Post.objects.filter(is_published=True, category__slug='django')

# แบบที่ 2: ใช้ Q() แล้ว & กันเอง (ผลลัพธ์เหมือนกันทุกประการ)
Post.objects.filter(
    Q(is_published=True) & Q(category__slug='django')
)
```

ปกติเราจะใช้แบบที่ 1 เมื่อเป็น AND ล้วน ๆ และหันมาใช้ `Q()` เมื่อต้องมี OR หรือ
เงื่อนไขซับซ้อนเข้ามาเกี่ยวข้อง

### 134.4 `~Q()` สำหรับ NOT

```python
from django.db.models import Q
from blog.models import Post

# โพสต์ที่ "ไม่ใช่" หมวดหมู่ Django
Post.objects.filter(~Q(category__slug='django'))

# เทียบเท่ากับ .exclude() ในกรณีง่าย ๆ
Post.objects.exclude(category__slug='django')
```

`~Q()` มีประโยชน์มากเมื่อ NOT ต้องอยู่ในสมการที่ซับซ้อนกว่านี้ ซึ่ง `.exclude()`
เพียงอย่างเดียวทำไม่ได้ (ดูตัวอย่างถัดไป)

### 134.5 การรวม Q() แบบซับซ้อน (Nested Q)

```python
from django.db.models import Q
from blog.models import Post

# (เผยแพร่แล้ว AND หมวดหมู่ Django) OR (ยอดวิวเกิน 5000 AND ไม่ถูกลบหมวดหมู่)
Post.objects.filter(
    (Q(is_published=True) & Q(category__slug='django')) |
    (Q(view_count__gt=5000) & ~Q(category__isnull=True))
)
```

วงเล็บใน Python จะกำหนดลำดับความสำคัญเหมือนตรรกะบูลีนทั่วไป อ่านจากในวงเล็บออกมา

### 134.6 ตัวอย่างจริง: ระบบค้นหา (Search) ในบล็อก

นี่คือ pattern ที่ใช้บ่อยที่สุดในงานจริง — ค้นหาคำเดียวกันในหลาย field พร้อมกัน:

```python
from django.db.models import Q
from blog.models import Post

def search_posts(keyword):
    return Post.objects.filter(
        Q(title__icontains=keyword) |
        Q(content__icontains=keyword) |
        Q(tags__name__icontains=keyword)
    ).distinct()   # ต้องใส่ distinct() เพราะ join กับ tags (M2M) อาจทำให้แถวซ้ำ

results = search_posts('django')
```

**ข้อควรระวัง**: เมื่อ `Q()` เดินทางผ่าน M2M หรือ reverse FK (เช่น `tags__name`)
อาจได้แถวซ้ำเนื่องจาก JOIN เหมือนที่อธิบายในขั้นตอนที่ 132.5 ต้องใส่ `.distinct()`
ที่ท้าย QuerySet เสมอเมื่อ filter ผ่านความสัมพันธ์แบบ "หนึ่งกลายเป็นหลาย"

### 134.7 การสร้าง Q() แบบไดนามิกด้วยโค้ด (สำหรับฟอร์มค้นหาขั้นสูง)

ในงานจริง เรามักต้องสร้างเงื่อนไข filter จากฟอร์มที่ผู้ใช้กรอกมาไม่ครบทุกช่อง
`Q()` รวมกันด้วยโค้ดแบบไดนามิกได้:

```python
from django.db.models import Q
from blog.models import Post


def filter_posts(category_slug=None, tag_slug=None, keyword=None, published_only=True):
    """
    สร้าง QuerySet แบบไดนามิกตามเงื่อนไขที่ผู้ใช้ระบุมา (บางช่องอาจไม่มีค่า)
    """
    query = Q()

    if published_only:
        query &= Q(is_published=True)

    if category_slug:
        query &= Q(category__slug=category_slug)

    if tag_slug:
        query &= Q(tags__slug=tag_slug)

    if keyword:
        query &= (Q(title__icontains=keyword) | Q(content__icontains=keyword))

    return Post.objects.filter(query).distinct()


# ใช้งาน
posts = filter_posts(category_slug='django', keyword='orm')
```

pattern นี้คือหัวใจของการสร้างระบบค้นหา/กรองข้อมูลที่ยืดหยุ่นในงานจริงระดับมืออาชีพ
เราจะกลับมาขยายความรื่องนี้อีกครั้งเมื่อสร้าง Django Forms ใน Phase 3

---

## ขั้นตอนที่ 135: `F()` expressions — เปรียบเทียบ field และอัปเดตแบบ atomic

### 135.1 ปัญหา Race Condition ของการอ่าน-แก้-เขียนแบบธรรมดา

ลองดูโค้ดที่ **ดูเหมือนถูกต้อง** แต่จริง ๆ แล้วมีบั๊กร้ายแรงซ่อนอยู่ — ฟังก์ชันเพิ่ม
ยอดวิวของโพสต์เมื่อมีคนเข้าชม:

```python
# views.py — โค้ดนี้มีบั๊ก! (race condition)
def post_detail(request, slug):
    post = Post.objects.get(slug=slug)
    post.view_count = post.view_count + 1   # 1. อ่านค่าปัจจุบันมาเก็บใน Python
    post.save()                              # 2. เขียนค่าใหม่กลับไป
    ...
```

**ปัญหาคืออะไร**: สมมติโพสต์นี้มี `view_count = 100` และมีผู้ใช้ 2 คนเข้าชม
"พร้อมกันเป๊ะ" (เกิดขึ้นได้จริงในเว็บที่มีทราฟฟิกสูง):

```
เวลา    Request A                       Request B
------  ------------------------------  ------------------------------
t1      อ่าน view_count = 100
t2                                       อ่าน view_count = 100
t3      คำนวณ 100 + 1 = 101
t4                                       คำนวณ 100 + 1 = 101
t5      save() -> view_count = 101
t6                                       save() -> view_count = 101

ผลลัพธ์: view_count = 101  (ที่ถูกต้องควรเป็น 102 เพราะมี 2 คนเข้าชม!)
```

นี่คือ **Race Condition** ที่เกิดขึ้นจริงในระบบ production ที่มีคนเข้าใช้พร้อมกันจำนวนมาก
และเป็นบั๊กที่ debug ยากมากเพราะไม่เกิดขึ้นทุกครั้ง (เกิดเฉพาะตอนที่ timing ชนกันพอดี)

### 135.2 `F()` แก้ปัญหานี้ได้อย่างไร

`F()` expression บอกให้ **ฐานข้อมูลเป็นคนคำนวณเอง** โดยที่ Python ไม่ต้องอ่านค่าปัจจุบัน
เข้ามาก่อนเลย:

```python
from django.db.models import F
from blog.models import Post

def post_detail(request, slug):
    Post.objects.filter(slug=slug).update(view_count=F('view_count') + 1)
    post = Post.objects.get(slug=slug)
    ...
```

โค้ดนี้จะแปลงเป็น SQL ประมาณ:

```sql
UPDATE blog_post SET view_count = view_count + 1 WHERE slug = 'my-post';
```

**นี่คือคำสั่งเดียวที่ฐานข้อมูลรันแบบ atomic** (อ่าน+บวก+เขียน เกิดขึ้นในขั้นตอนเดียว
ที่ระดับฐานข้อมูล) ไม่มีช่วงเวลาให้ request อื่นมาแทรกกลางคันได้ ไม่ว่าจะมีกี่ request
เข้ามาพร้อมกัน ทุก `+1` จะถูกนับครบเสมอ

### 135.3 ตารางเปรียบเทียบ: อ่าน-แก้-เขียน vs `F()`

| ประเด็น | `post.view_count += 1; post.save()` | `F('view_count') + 1` |
|---|---|---|
| จำนวน query ที่ใช้ | 2 ครั้ง (SELECT + UPDATE) | 1 ครั้ง (UPDATE เดียว) |
| ปลอดภัยจาก Race Condition | ❌ ไม่ปลอดภัย | ✅ ปลอดภัย (atomic ที่ระดับ DB) |
| ค่าที่คำนวณ | คำนวณใน Python (ค่าอาจเก่าไปแล้ว) | คำนวณใน SQL โดยตรง (ค่าล่าสุดเสมอ) |
| Performance | ช้ากว่า (2 round-trip ไป DB) | เร็วกว่า (1 round-trip) |

### 135.4 ข้อควรระวัง: object ใน Python ไม่รู้ค่าล่าสุดหลังใช้ F()

```python
from django.db.models import F
from blog.models import Post

post = Post.objects.get(slug='my-post')
post.view_count = F('view_count') + 1
post.save()

print(post.view_count)
# <CombinedExpression: F(view_count) + Value(1)>   <- ไม่ใช่ตัวเลข! เป็น expression object

# ต้อง refresh_from_db() เพื่อดึงค่าจริงจากฐานข้อมูลกลับมา
post.refresh_from_db()
print(post.view_count)
# 101   <- ค่าที่ถูกต้องแล้ว
```

นี่คือเหตุผลที่ pattern ที่แนะนำในข้อ 135.2 ใช้ `.filter().update()` (ที่ทำงานกับหลายแถว
โดยไม่ต้องโหลด object เข้า Python เลย) แล้วค่อย `.get()` ใหม่อีกครั้งถ้าต้องใช้ค่าล่าสุด
แทนที่จะ set ค่าใส่ instance โดยตรง

### 135.5 `F()` เปรียบเทียบ field กับ field อื่นในตารางเดียวกัน

`F()` ไม่ได้ใช้แค่กับตัวเลข แต่ใช้เปรียบเทียบ field สองอันในแถวเดียวกันได้ด้วย —
สิ่งนี้ทำไม่ได้เลยด้วย `filter()` แบบธรรมดา:

```python
from django.db.models import F
from blog.models import Post

# หาโพสต์ที่ "ถูกแก้ไขหลังจากสร้างครั้งแรก" (updated_at ต่างจาก created_at)
edited_posts = Post.objects.filter(updated_at__gt=F('created_at'))

# หาโพสต์ที่ยังไม่เคยถูกแก้ไขเลย (updated_at เท่ากับ created_at เป๊ะ ๆ)
never_edited = Post.objects.filter(updated_at=F('created_at'))
```

### 135.6 `F()` ในการคำนวณเลขคณิต (Arithmetic) ร่วมกับ `annotate()`

`F()` รวมกับ `annotate()` เพื่อสร้าง field คำนวณใหม่จาก field ที่มีอยู่แล้วได้:

```python
from django.db.models import F, ExpressionWrapper, FloatField
from blog.models import Post

# สมมติเรามี field เพิ่ม: like_count (จำลองไว้เพื่อตัวอย่าง arithmetic)
# คำนวณ "engagement score" = view_count หาร 100 บวกด้วย comment count
from django.db.models import Count

posts = Post.objects.annotate(
    comment_count=Count('comments', distinct=True)
).annotate(
    engagement_score=ExpressionWrapper(
        F('view_count') / 100.0 + F('comment_count'),
        output_field=FloatField(),
    )
).order_by('-engagement_score')

for post in posts[:5]:
    print(post.title, post.engagement_score)
```

**หมายเหตุเรื่อง `ExpressionWrapper`**: เมื่อผสม field คนละชนิดกัน (เช่น `IntegerField`
หาร float) หรือผลลัพธ์อาจกำกวมว่าเป็นชนิดข้อมูลอะไร Django กำหนดให้เราต้องระบุ
`output_field` อย่างชัดเจนผ่าน `ExpressionWrapper` เพื่อไม่ให้เกิด error หรือผลลัพธ์
ที่ไม่คาดคิด (เช่นหาร integer แล้วปัดเศษทิ้งโดยไม่ตั้งใจ)

### 135.7 `F()` ในการอัปเดตหลายแถวพร้อมกัน (Bulk Update)

```python
from django.db.models import F
from blog.models import Post

# เพิ่มยอดวิวให้ทุกโพสต์ในหมวดหมู่ "ประกาศ" อีก 10 (เช่น เหตุการณ์พิเศษ)
Post.objects.filter(category__slug='announcement').update(
    view_count=F('view_count') + 10
)

# ลดราคาสินค้าทุกชิ้นลง 10% (ตัวอย่างสมมติถ้ามีโมเดล Product ที่มี field price)
# Product.objects.update(price=F('price') * 0.9)
```

คำสั่ง `.update()` ที่ใช้ `F()` เป็น **query เดียว** ที่อัปเดตได้เป็นพัน ๆ แถวพร้อมกัน
โดยไม่ต้องโหลดแต่ละ object เข้ามาใน Python memory เลย เร็วกว่าการวนลูป `for` แล้ว
`.save()` ทีละตัวมหาศาล

---

## ขั้นตอนที่ 136: `Case`/`When` — Conditional Annotation

### 136.1 `Case`/`When` คืออะไร

`Case`/`When` คือการเขียน `if-elif-else` ในระดับ SQL — ใช้เมื่อค่าที่ต้อง annotate
ขึ้นอยู่กับเงื่อนไข ไม่ใช่แค่การนับหรือรวมธรรมดา

Syntax พื้นฐาน:

```python
from django.db.models import Case, When, Value, CharField

Case(
    When(เงื่อนไข_1, then=ค่า_1),
    When(เงื่อนไข_2, then=ค่า_2),
    default=ค่า_default,
    output_field=CharField(),
)
```

### 136.2 ตัวอย่าง: ติดป้ายกำกับความนิยมของโพสต์

```python
from django.db.models import Case, When, Value, CharField
from blog.models import Post

posts = Post.objects.annotate(
    popularity=Case(
        When(view_count__gte=1000, then=Value('ยอดนิยม')),
        When(view_count__gte=100, then=Value('ปานกลาง')),
        default=Value('ยังไม่ค่อยมีคนอ่าน'),
        output_field=CharField(),
    )
)

for post in posts:
    print(f"{post.title}: {post.popularity}")
# "แนะนำ Django 5": ยอดนิยม
# "บทความทดสอบ": ยังไม่ค่อยมีคนอ่าน
```

Django ตรวจสอบ `When()` ตามลำดับที่เขียนจากบนลงล่าง (เหมือน `if-elif-elif-else`)
เมื่อเงื่อนไขไหน match ก่อนก็ใช้ค่าของ `When()` นั้นทันที

### 136.3 Conditional Aggregation: นับแยกประเภทในคำสั่งเดียว

นี่คือประโยชน์ที่ทรงพลังที่สุดของ `Case`/`When` — ใช้ **นับแบบมีเงื่อนไข** รวมกับ
`Sum()` เพื่อได้หลายตัวเลขจาก query เดียว แทนที่จะต้อง query แยกหลายรอบ:

```python
from django.db.models import Case, When, Value, IntegerField, Sum
from blog.models import Post

stats = Post.objects.aggregate(
    published_count=Sum(
        Case(When(is_published=True, then=Value(1)), default=Value(0),
             output_field=IntegerField())
    ),
    draft_count=Sum(
        Case(When(is_published=False, then=Value(1)), default=Value(0),
             output_field=IntegerField())
    ),
)
print(stats)
# {'published_count': 18, 'draft_count': 7}
```

เทียบกับวิธีเดิมที่ต้อง query 2 รอบแยกกัน:

```python
# วิธีเดิม (2 query แยกกัน — ทำงานได้เหมือนกัน แต่ hit ฐานข้อมูล 2 ครั้ง)
published_count = Post.objects.filter(is_published=True).count()
draft_count = Post.objects.filter(is_published=False).count()
```

ในกรณีนี้การใช้ 2 query แยกก็อ่านง่ายกว่าและเร็วพอ ๆ กัน แต่เมื่อมีหมวดหมู่ให้แยกนับ
เยอะขึ้น (เช่น 10 หมวดหมู่) การรวมเป็น query เดียวด้วย `Case`/`When` จะเริ่มได้เปรียบ
ชัดเจนเรื่อง performance (1 round-trip แทนที่จะเป็น 10)

### 136.4 `Case`/`When` ร่วมกับ `annotate()` แบบมีเงื่อนไขซับซ้อน

```python
from django.db.models import Case, When, Value, F, CharField
from blog.models import Post

posts = Post.objects.annotate(
    status_label=Case(
        When(is_published=False, then=Value('ฉบับร่าง')),
        When(view_count=0, then=Value('เผยแพร่แล้วแต่ยังไม่มีคนอ่าน')),
        When(updated_at__gt=F('created_at'), then=Value('เผยแพร่แล้ว (แก้ไขล่าสุด)')),
        default=Value('เผยแพร่แล้ว'),
        output_field=CharField(),
    )
)
```

สังเกตว่าเราผสม `Case`/`When` เข้ากับ `F()` ในเงื่อนไขได้ด้วย (เปรียบเทียบ
`updated_at` กับ `created_at`) — นี่แสดงให้เห็นว่าเครื่องมือ ORM เหล่านี้ประกอบกัน
(compose) ได้อย่างอิสระ

---

## ขั้นตอนที่ 137: Subqueries ด้วย `OuterRef`/`Subquery` และ `Exists`

### 137.1 Subquery คืออะไร และทำไมต้องมี `OuterRef`

บางครั้งเราต้องการ annotate ค่าที่มาจากการ query ตารางอื่นแบบ "correlated"
(ค่าของแต่ละแถวขึ้นอยู่กับแถวนั้น ๆ เอง) — สิ่งนี้ทำด้วย `annotate()` + `Count`/`Sum`
ธรรมดาไม่ได้เสมอไป โดยเฉพาะเมื่อต้องการ **ค่าล่าสุด** หรือ **ค่าที่ซับซ้อนกว่าการรวม**

`OuterRef` คือการ "อ้างอิงกลับไปยัง field ของ query หลัก (outer query)" จากภายใน
subquery — เหมือนตัวแปรที่เชื่อมสอง query เข้าด้วยกัน

### 137.2 ตัวอย่าง: annotate โพสต์ด้วย "ข้อความคอมเมนต์ล่าสุด"

```python
from django.db.models import OuterRef, Subquery
from blog.models import Post, Comment

latest_comment = Comment.objects.filter(
    post=OuterRef('pk')
).order_by('-created_at')

posts = Post.objects.annotate(
    latest_comment_text=Subquery(latest_comment.values('text')[:1]),
    latest_comment_author=Subquery(latest_comment.values('author')[:1]),
)

for post in posts:
    print(f"{post.title} -> ล่าสุด: {post.latest_comment_author}: {post.latest_comment_text}")
```

อธิบายทีละบรรทัด:

1. `Comment.objects.filter(post=OuterRef('pk'))` — สร้าง QuerySet ของคอมเมนต์
   "เฉพาะของโพสต์ที่กำลังพิจารณาอยู่ใน query หลัก" (`OuterRef('pk')` หมายถึง
   pk ของแถว Post ที่ query ภายนอกกำลังวนอยู่)
2. `.order_by('-created_at')` — เรียงเอาคอมเมนต์ล่าสุดขึ้นก่อน
3. `Subquery(latest_comment.values('text')[:1])` — ตัด subquery ให้เหลือแค่
   1 แถว 1 column (`text`) เพราะ `Subquery` ที่ใช้ใน `annotate()` ต้องคืนค่าเดียว

### 137.3 ทำไมใช้ `Subquery` แทน `annotate(Max('comments__created_at'))`?

คำถามที่สมเหตุสมผล: ทำไมไม่ annotate หา `created_at` ล่าสุดตรง ๆ? เพราะ `Max()`
คืนแค่ "ค่าของ field เดียว" (เช่น เวลาล่าสุด) แต่ **ไม่คืน field อื่นของแถวนั้น**
(เช่น ข้อความและผู้เขียนของคอมเมนต์ล่าสุด) ถ้าต้องการหลาย field จากแถวเดียวกัน
(แถวที่ "ล่าสุด") ต้องใช้ `Subquery` เพื่อดึงทั้งแถวที่ต้องการมา ไม่ใช่แค่ค่าที่รวมกันได้

### 137.4 `Exists()` — เร็วกว่า `Subquery` เมื่อแค่ต้องเช็คว่า "มีหรือไม่มี"

ถ้าคำถามคือ "โพสต์นี้มีคอมเมนต์ไหม" (ไม่สนใจว่ามีกี่อัน แค่ต้องการ True/False)
`Exists()` เร็วกว่า `Count()` มาก เพราะฐานข้อมูลหยุดค้นทันทีที่เจอแถวแรกที่ match
(ไม่ต้องนับให้ครบทุกแถว):

```python
from django.db.models import OuterRef, Exists
from blog.models import Post, Comment

has_comments = Comment.objects.filter(post=OuterRef('pk'))

posts = Post.objects.annotate(
    has_comment=Exists(has_comments)
)

# ใช้ filter ต่อได้ทันที
posts_without_comments = Post.objects.annotate(
    has_comment=Exists(has_comments)
).filter(has_comment=False)
```

| เครื่องมือ | ใช้เมื่อ | Performance |
|---|---|---|
| `Count('comments')` | ต้องการ**จำนวน**คอมเมนต์จริง ๆ | ต้องนับทุกแถวที่ join ได้ |
| `Exists(subquery)` | ต้องการแค่ True/False ว่ามีอยู่หรือไม่ | เร็วกว่า หยุดทันทีที่เจอแถวแรก |
| `Subquery(subquery)` | ต้องการค่า field เฉพาะจากแถวที่เกี่ยวข้อง (เช่น ค่าล่าสุด) | ขึ้นกับ index ของตารางย่อย |

### 137.5 Subquery กับ `annotate()` แบบ aggregate ภายใน (นับจำนวนภายใน subquery)

```python
from django.db.models import OuterRef, Subquery, Count
from django.db.models.functions import Coalesce
from blog.models import Post, Comment

reply_counts = Comment.objects.filter(
    parent=OuterRef('pk')
).values('parent').annotate(
    total=Count('id')
).values('total')

comments_with_reply_count = Comment.objects.annotate(
    reply_count=Coalesce(Subquery(reply_counts), 0)
)
```

ตัวอย่างนี้นับจำนวน "การตอบกลับ" (`replies`, self-FK ชื่อ `parent`) ของแต่ละคอมเมนต์
โดยใช้ subquery ที่มี aggregate อยู่ข้างในอีกที และใช้ `Coalesce` เพื่อแปลงค่า `None`
(กรณีไม่มีการตอบกลับเลย) ให้เป็น `0` แทน

---

## ขั้นตอนที่ 138: Window Functions เบื้องต้น — Ranking ด้วย `Rank`

### 138.1 Window Function คืออะไร

**Window Function** คำนวณค่าโดยอิงจาก "กลุ่มแถวที่เกี่ยวข้องกัน" (เรียกว่า
partition/window) โดยที่ **ไม่ยุบรวมแถวเป็นแถวเดียวแบบ GROUP BY** — นี่คือข้อแตกต่าง
สำคัญจาก `annotate()` + aggregate function ทั่วไป: annotate ปกติทำให้แต่ละแถว
ยังเห็นข้อมูลของตัวเองครบ แต่ยัง "รู้" ค่าที่คำนวณจากกลุ่มของมันด้วย (เช่น อันดับ
ของตัวเองในกลุ่ม) โดยไม่สูญเสียแถวใดไปเลย

Django รองรับ Window Functions ตั้งแต่เวอร์ชัน 2.0 ผ่าน `django.db.models.Window`

### 138.2 ตัวอย่าง: จัดอันดับโพสต์ตามยอดวิว "ภายในแต่ละหมวดหมู่"

```python
from django.db.models import Window, F
from django.db.models.functions import Rank
from blog.models import Post

posts = Post.objects.annotate(
    rank_in_category=Window(
        expression=Rank(),
        partition_by=[F('category')],
        order_by=F('view_count').desc(),
    )
).order_by('category', 'rank_in_category')

for post in posts:
    print(f"[{post.category}] อันดับ {post.rank_in_category}: {post.title} ({post.view_count} views)")

# [Django] อันดับ 1: แนะนำ Django 5 (3200 views)
# [Django] อันดับ 2: ทำความรู้จัก ORM (1800 views)
# [Django] อันดับ 3: Deploy สู่ Production (450 views)
# [Python] อันดับ 1: เขียน Python ให้ Clean (2100 views)
# [Python] อันดับ 2: Type Hints คืออะไร (900 views)
```

อธิบาย parameter ของ `Window`:

- `expression=Rank()` — ฟังก์ชันที่จะคำนวณ (ในที่นี้คือการจัดอันดับ)
- `partition_by=[F('category')]` — แบ่งกลุ่ม (window) ตามหมวดหมู่ ทำให้อันดับ
  "เริ่มนับใหม่" ในแต่ละหมวดหมู่ (เทียบเท่า `PARTITION BY` ใน SQL)
- `order_by=F('view_count').desc()` — เรียงลำดับภายในแต่ละ partition เพื่อกำหนดอันดับ

### 138.3 ฟังก์ชันอื่น ๆ ที่ใช้กับ `Window` ได้

| ฟังก์ชัน | ความหมาย |
|---|---|
| `Rank()` | อันดับ (ถ้าค่าเท่ากันจะได้อันดับเดียวกัน แล้วข้ามอันดับถัดไป เช่น 1, 1, 3) |
| `DenseRank()` | เหมือน `Rank()` แต่ไม่ข้ามอันดับ (เช่น 1, 1, 2) |
| `RowNumber()` | ลำดับแถวธรรมดา ไม่สนใจค่าเท่ากัน (เช่น 1, 2, 3 เสมอ) |
| `Lag()` | ดึงค่าจากแถว "ก่อนหน้า" ใน partition เดียวกัน |
| `Lead()` | ดึงค่าจากแถว "ถัดไป" ใน partition เดียวกัน |
| `Sum()`, `Avg()`, `Count()` เป็นต้น | ใช้เป็น "running total" / "moving average" ได้เมื่ออยู่ใน `Window` |

ตัวอย่าง Running Total ของยอดวิวสะสม เรียงตามวันที่สร้าง:

```python
from django.db.models import Window, Sum
from blog.models import Post

posts = Post.objects.annotate(
    running_total_views=Window(
        expression=Sum('view_count'),
        order_by=F('created_at').asc(),
    )
).order_by('created_at')
```

### 138.4 ข้อควรระวังเรื่องฐานข้อมูล

Window Functions ต้องการฐานข้อมูลที่รองรับ SQL Window Functions มาตรฐาน:

| ฐานข้อมูล | รองรับ Window Functions |
|---|---|
| PostgreSQL | ✅ รองรับเต็มรูปแบบ (แนะนำสำหรับ production) |
| MySQL 8.0+ | ✅ รองรับ (MySQL < 8.0 ไม่รองรับ) |
| SQLite 3.25+ | ✅ รองรับ (Python 3.12 มากับ SQLite ที่ใหม่พอ) |
| Oracle | ✅ รองรับ |

เนื่องจากหลักสูตรนี้แนะนำให้ใช้ PostgreSQL ตั้งแต่ Phase 2 เป็นต้นไป (ตามที่กล่าวไว้ใน
Part 001) คุณจะไม่เจอปัญหาเรื่องเวอร์ชันฐานข้อมูลไม่รองรับ แต่ถ้ายังทดลองบน SQLite
เวอร์ชันเก่าอยู่ ให้อัปเกรด Python เป็น 3.12+ (ซึ่งพ่วง SQLite ใหม่มาให้)

**ข้อจำกัดสำคัญอีกข้อ**: Window function ใช้ร่วมกับ `.filter()` บนผลลัพธ์ของ window
โดยตรงไม่ได้ (SQL ไม่อนุญาตให้ `WHERE` อ้างอิงถึง window function ในคำสั่งเดียวกัน)
ถ้าต้องการกรองผลลัพธ์ที่ผ่าน window มาแล้ว ต้อง wrap เป็น subquery อีกชั้น หรือกรอง
ที่ฝั่ง Python หลังดึงข้อมูลมาแล้ว

---

## ขั้นตอนที่ 139: Group By ด้วย `values().annotate()`

### 139.1 `annotate()` เฉย ๆ vs `values().annotate()` — ต่างกันอย่างไร

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุดอันดับสองรองจาก aggregate vs annotate:

| รูปแบบ | ผลลัพธ์ | เทียบเท่า SQL |
|---|---|---|
| `Post.objects.annotate(comment_count=Count('comments'))` | 1 แถวต่อ 1 **Post** (group by `Post.id` โดยอัตโนมัติ) | `GROUP BY post.id` |
| `Post.objects.values('category').annotate(post_count=Count('id'))` | 1 แถวต่อ 1 **ค่าของ category** (ไม่ใช่ต่อโพสต์อีกต่อไป) | `GROUP BY post.category_id` |

พูดให้ชัดคือ: **`annotate()` เพียว ๆ จะ group by primary key ของโมเดลหลักเสมอ**
(ดังนั้นแต่ละ object ที่ได้ยังเป็น 1 โพสต์ 1 ตัวเหมือนเดิม) ส่วน **`values(...)`
ที่วางไว้ *ก่อน* `annotate()` จะเปลี่ยนสิ่งที่ใช้ group by** ให้เป็น field ที่ระบุใน
`values()` แทน

### 139.2 ตัวอย่าง: นับจำนวนโพสต์ในแต่ละหมวดหมู่ (Group By Category)

```python
from django.db.models import Count
from blog.models import Post

category_stats = Post.objects.values('category__name').annotate(
    post_count=Count('id')
).order_by('-post_count')

for row in category_stats:
    print(row)
# {'category__name': 'Django', 'post_count': 12}
# {'category__name': 'Python', 'post_count': 8}
# {'category__name': None, 'post_count': 5}   <- โพสต์ที่ยังไม่ได้กำหนดหมวดหมู่
```

สังเกตว่าผลลัพธ์แต่ละแถวเป็น **`dict`** (ไม่ใช่ instance ของ `Post` อีกต่อไป)
เพราะ `values()` เปลี่ยนรูปแบบผลลัพธ์ของ QuerySet ให้เป็น dict ตั้งแต่ต้น

### 139.3 ลำดับก่อน-หลังสำคัญมาก: `values()` ต้องมาก่อน `annotate()` เสมอ

```python
from django.db.models import Count
from blog.models import Post

# ถูกต้อง: values() ก่อน -> group by category
Post.objects.values('category__name').annotate(post_count=Count('id'))

# ผิดความตั้งใจ: annotate() ก่อน values() -> ยัง group by post.id เหมือนเดิม
# แล้ว values() แค่ "เลือกว่าจะแสดง field ไหน" เท่านั้น ไม่เปลี่ยน grouping
Post.objects.annotate(post_count=Count('id')).values('category__name', 'post_count')
# ผลลัพธ์: post_count จะเป็น 1 เสมอทุกแถว (เพราะนับแค่ตัวมันเองต่อโพสต์)
```

นี่คือกฎที่ต้องจำขึ้นใจ: **ตำแหน่งของ `values()` ก่อนหรือหลัง `annotate()` เปลี่ยน
ความหมายของ query ทั้งหมด** ไม่ใช่แค่เรื่อง "จะ select field ไหน" เท่านั้น

### 139.4 Group By ตามช่วงเวลา (Truncate Date Functions)

Pattern ที่พบบ่อยมากในหน้า dashboard คือ "กี่โพสต์ต่อเดือน" — ใช้ `TruncMonth`,
`TruncDate`, `TruncYear` จาก `django.db.models.functions`:

```python
from django.db.models import Count
from django.db.models.functions import TruncMonth
from blog.models import Post

posts_per_month = Post.objects.annotate(
    month=TruncMonth('created_at')
).values('month').annotate(
    post_count=Count('id')
).order_by('month')

for row in posts_per_month:
    print(row)
# {'month': datetime.date(2026, 1, 1), 'post_count': 4}
# {'month': datetime.date(2026, 2, 1), 'post_count': 7}
# {'month': datetime.date(2026, 3, 1), 'post_count': 3}
```

สังเกตว่าที่นี่เราใช้ `annotate()` สองครั้งติดกัน: ครั้งแรกสร้าง field `month`
(ตัดวันที่ให้เหลือแค่ระดับเดือน) แล้ว `values('month')` เพื่อเปลี่ยนตัว group by
เป็น `month` แล้ว `annotate()` ครั้งที่สองค่อยนับจำนวนโพสต์ในแต่ละเดือนนั้น

### 139.5 Group By หลาย Field พร้อมกัน

```python
from django.db.models import Count
from blog.models import Post

stats = Post.objects.values('category__name', 'is_published').annotate(
    total=Count('id')
).order_by('category__name', 'is_published')

for row in stats:
    print(row)
# {'category__name': 'Django', 'is_published': False, 'total': 2}
# {'category__name': 'Django', 'is_published': True, 'total': 10}
# {'category__name': 'Python', 'is_published': True, 'total': 8}
```

การใส่ field หลายตัวใน `values()` จะ group by ทุก field ที่ระบุร่วมกัน (เหมือน
`GROUP BY category_id, is_published` ใน SQL)

### 139.6 ตารางสรุปกฎการเลือกใช้

| ต้องการอะไร | ใช้แบบไหน |
|---|---|
| ค่าคำนวณต่อ "object หลัก" (เช่น comment_count ต่อโพสต์) | `Model.objects.annotate(...)` |
| ค่าคำนวณแบบ "จัดกลุ่มตาม field อื่น" (เช่น จำนวนโพสต์ต่อหมวดหมู่) | `Model.objects.values('field').annotate(...)` |
| ค่าสรุปทั้งระบบเป็นตัวเลขเดียว | `Model.objects.aggregate(...)` |
| ค่าคำนวณที่ต้อง "เห็นทั้งแถวเดิม" พร้อมค่าจัดอันดับ/รวมสะสมในกลุ่ม | `Model.objects.annotate(Window(...))` |

---

## ขั้นตอนที่ 140: สรุปและแบบฝึกหัด — สร้างหน้า Dashboard สถิติบล็อก

### 140.1 โจทย์: สร้าง View แสดง "Dashboard สถิติบล็อก" ที่ใช้ทุกเทคนิคของ Part นี้

มาประกอบทุกเทคนิคที่เรียนมาทั้ง Part เข้าด้วยกันเป็นฟีเจอร์จริงหนึ่งฟีเจอร์ —
หน้า Dashboard ที่ผู้ดูแลระบบใช้ดูภาพรวมของบล็อกทั้งหมด:

```python
# blog/views.py
from django.db.models import (
    Count, Sum, Avg, Max, Min, Q, F, Case, When, Value,
    IntegerField, CharField, OuterRef, Subquery, Window,
)
from django.db.models.functions import TruncMonth, Rank, Coalesce
from django.shortcuts import render

from .models import Post, Category, Comment


def blog_dashboard(request):
    # ----- 1) aggregate(): สถิติรวมทั้งระบบ (ขั้นตอนที่ 131) -----
    overview = Post.objects.aggregate(
        total_posts=Count('id'),
        published_posts=Count('id', filter=Q(is_published=True)),
        draft_posts=Count('id', filter=Q(is_published=False)),
        total_views=Coalesce(Sum('view_count'), 0),
        avg_views=Coalesce(Avg('view_count'), 0.0),
        max_views=Coalesce(Max('view_count'), 0),
    )

    # ----- 2) annotate() + aggregate(): ค่าเฉลี่ยคอมเมนต์ต่อโพสต์ (ขั้นตอนที่ 131, 133) -----
    comment_stats = Post.objects.annotate(
        comment_count=Count('comments', distinct=True)
    ).aggregate(
        avg_comments_per_post=Coalesce(Avg('comment_count'), 0.0),
        max_comments_on_post=Coalesce(Max('comment_count'), 0),
    )

    # ----- 3) Q(): ค้นหาโพสต์ที่ "เผยแพร่แล้วและมาแรง" หรือ "ยอดวิวสูงมาก" -----
    trending_or_popular = Post.objects.filter(
        Q(is_published=True, view_count__gte=500) | Q(view_count__gte=3000)
    ).distinct()

    # ----- 4) F(): โพสต์ที่ถูกแก้ไขหลังเผยแพร่ (updated_at ต่างจาก created_at) -----
    edited_posts = Post.objects.filter(
        is_published=True,
        updated_at__gt=F('created_at'),
    ).count()

    # ----- 5) Case/When: แจกแจงโพสต์ตามระดับความนิยม -----
    popularity_breakdown = Post.objects.aggregate(
        very_popular=Sum(
            Case(When(view_count__gte=1000, then=Value(1)), default=Value(0),
                 output_field=IntegerField())
        ),
        somewhat_popular=Sum(
            Case(When(view_count__range=(100, 999), then=Value(1)), default=Value(0),
                 output_field=IntegerField())
        ),
        not_popular=Sum(
            Case(When(view_count__lt=100, then=Value(1)), default=Value(0),
                 output_field=IntegerField())
        ),
    )

    # ----- 6) Subquery/OuterRef: โพสต์ 5 อันดับล่าสุด พร้อมคอมเมนต์ล่าสุดของแต่ละอัน -----
    latest_comment_qs = Comment.objects.filter(
        post=OuterRef('pk')
    ).order_by('-created_at')

    recent_posts = Post.objects.filter(is_published=True).annotate(
        latest_comment_author=Subquery(latest_comment_qs.values('author')[:1]),
        latest_comment_text=Subquery(latest_comment_qs.values('text')[:1]),
    ).order_by('-created_at')[:5]

    # ----- 7) Window: จัดอันดับโพสต์ยอดนิยมในแต่ละหมวดหมู่ -----
    ranked_posts = Post.objects.filter(is_published=True).annotate(
        rank_in_category=Window(
            expression=Rank(),
            partition_by=[F('category')],
            order_by=F('view_count').desc(),
        )
    ).order_by('category__name', 'rank_in_category')
    # แสดงเฉพาะ Top 3 ของแต่ละหมวดหมู่ (กรองฝั่ง Python เพราะ filter บน window ไม่ได้โดยตรง)
    top3_per_category = [p for p in ranked_posts if p.rank_in_category <= 3]

    # ----- 8) values().annotate(): จำนวนโพสต์ต่อหมวดหมู่ และต่อเดือน -----
    posts_by_category = Post.objects.values(
        category_name=F('category__name')
    ).annotate(
        post_count=Count('id'),
        total_views=Coalesce(Sum('view_count'), 0),
    ).order_by('-post_count')

    posts_by_month = Post.objects.annotate(
        month=TruncMonth('created_at')
    ).values('month').annotate(
        post_count=Count('id')
    ).order_by('month')

    context = {
        'overview': overview,
        'comment_stats': comment_stats,
        'trending_or_popular_count': trending_or_popular.count(),
        'edited_posts_count': edited_posts,
        'popularity_breakdown': popularity_breakdown,
        'recent_posts': recent_posts,
        'top3_per_category': top3_per_category,
        'posts_by_category': posts_by_category,
        'posts_by_month': posts_by_month,
    }
    return render(request, 'blog/dashboard.html', context)
```

และ URL ที่เชื่อมเข้ากับ view นี้:

```python
# blog/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('dashboard/', views.blog_dashboard, name='blog_dashboard'),
]
```

Template แบบง่ายที่แสดงข้อมูลทั้งหมด (เราจะเรียน Django Template Language อย่าง
เจาะลึกใน Phase 3 ตอนนี้ขอให้เห็นภาพว่าข้อมูลที่คำนวณมาถูกใช้แสดงผลอย่างไร):

```html
<!-- blog/templates/blog/dashboard.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Dashboard สถิติบล็อก</title>
</head>
<body>
    <h1>ภาพรวมบล็อก</h1>
    <ul>
        <li>โพสต์ทั้งหมด: {{ overview.total_posts }}</li>
        <li>เผยแพร่แล้ว: {{ overview.published_posts }}</li>
        <li>ฉบับร่าง: {{ overview.draft_posts }}</li>
        <li>ยอดวิวรวม: {{ overview.total_views }}</li>
        <li>ยอดวิวเฉลี่ยต่อโพสต์: {{ overview.avg_views|floatformat:1 }}</li>
        <li>คอมเมนต์เฉลี่ยต่อโพสต์: {{ comment_stats.avg_comments_per_post|floatformat:2 }}</li>
        <li>โพสต์ที่กำลังมาแรง/ยอดนิยม: {{ trending_or_popular_count }}</li>
        <li>โพสต์ที่ถูกแก้ไขหลังเผยแพร่: {{ edited_posts_count }}</li>
    </ul>

    <h2>แบ่งตามความนิยม</h2>
    <ul>
        <li>ยอดนิยมมาก (1000+ views): {{ popularity_breakdown.very_popular }}</li>
        <li>ปานกลาง (100-999 views): {{ popularity_breakdown.somewhat_popular }}</li>
        <li>ยังไม่ค่อยมีคนอ่าน (&lt;100 views): {{ popularity_breakdown.not_popular }}</li>
    </ul>

    <h2>โพสต์ล่าสุด พร้อมคอมเมนต์ล่าสุด</h2>
    <ul>
        {% for post in recent_posts %}
            <li>
                {{ post.title }} —
                {% if post.latest_comment_text %}
                    ล่าสุด: "{{ post.latest_comment_text }}" โดย {{ post.latest_comment_author }}
                {% else %}
                    ยังไม่มีคอมเมนต์
                {% endif %}
            </li>
        {% endfor %}
    </ul>

    <h2>Top 3 ในแต่ละหมวดหมู่</h2>
    <ul>
        {% for post in top3_per_category %}
            <li>[{{ post.category }}] อันดับ {{ post.rank_in_category }}: {{ post.title }} ({{ post.view_count }} views)</li>
        {% endfor %}
    </ul>

    <h2>จำนวนโพสต์ต่อหมวดหมู่</h2>
    <ul>
        {% for row in posts_by_category %}
            <li>{{ row.category_name|default:"ไม่มีหมวดหมู่" }}: {{ row.post_count }} โพสต์ ({{ row.total_views }} views)</li>
        {% endfor %}
    </ul>

    <h2>จำนวนโพสต์ต่อเดือน</h2>
    <ul>
        {% for row in posts_by_month %}
            <li>{{ row.month|date:"F Y" }}: {{ row.post_count }} โพสต์</li>
        {% endfor %}
    </ul>
</body>
</html>
```

**ข้อสังเกตสำคัญของโค้ด view ด้านบน**: หน้า Dashboard ทั้งหน้านี้ทำงานด้วย
**query ไปฐานข้อมูลเพียงประมาณ 8-9 ครั้ง** เท่านั้น (แทนที่จะดึงโพสต์ทั้งหมดมาวนลูปนับ
เองใน Python ซึ่งอาจต้องใช้หน่วยความจำมหาศาลถ้ามีโพสต์เป็นแสน ๆ อัน) นี่คือพลังที่แท้จริง
ของการเรียนรู้เครื่องมือใน Part นี้: **ให้ฐานข้อมูลทำงานหนักแทนเรา** แทนที่จะโหลด
ข้อมูลดิบทั้งหมดมาประมวลผลในฝั่ง Python

### 140.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ใช้ `aggregate()` สรุปข้อมูลทั้ง QuerySet เป็นค่าเดียวด้วย `Count`, `Sum`, `Avg`,
  `Max`, `Min`
- ✅ ใช้ `annotate()` เพิ่มค่าที่คำนวณแล้วให้กับแต่ละ object ใน QuerySet และนำไป
  `filter()`/`order_by()` ต่อได้
- ✅ เข้าใจความแตกต่างของ `aggregate()` (ค่าเดียวของทั้งตาราง) กับ `annotate()`
  (ค่าต่อแถว) และรู้ว่าจะใช้สองตัวร่วมกันเมื่อไร
- ✅ ใช้ `Q()` สร้างเงื่อนไข OR/AND/NOT ที่ซับซ้อน รวมถึงสร้างเงื่อนไขแบบไดนามิก
  สำหรับระบบค้นหา
- ✅ ใช้ `F()` เปรียบเทียบ field กับ field อื่น และอัปเดตค่าแบบ atomic เพื่อป้องกัน
  Race Condition
- ✅ ใช้ `Case`/`When` สร้าง conditional annotation และ conditional aggregation
- ✅ ใช้ `Subquery`/`OuterRef`/`Exists` ทำ correlated subquery เพื่อดึงค่าจากแถว
  ที่เกี่ยวข้องแบบเฉพาะเจาะจง
- ✅ ใช้ `Window` function กับ `Rank()` เพื่อจัดอันดับข้อมูลภายในแต่ละกลุ่มโดยไม่
  ยุบรวมแถว
- ✅ เข้าใจว่า `values()` ก่อน `annotate()` เปลี่ยนตัว group by ทั้งหมด และรู้จัก
  `TruncMonth` สำหรับ group by ตามช่วงเวลา
- ✅ ประกอบทุกเทคนิคเข้าด้วยกันสร้างหน้า Dashboard สถิติจริงที่มีประสิทธิภาพสูง

### 140.3 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน `aggregate()` หาผลรวม/ค่าเฉลี่ย/ค่าสูงสุดของ `view_count` ได้เอง
- [ ] เขียน `annotate(comment_count=Count('comments'))` และเข้าใจว่าทำไมบางครั้ง
      ต้องใส่ `distinct=True`
- [ ] อธิบายความแตกต่างระหว่าง `aggregate()` กับ `annotate()` ด้วยคำพูดตัวเองได้
- [ ] เขียน `Q()` ผสม OR และ AND ในเงื่อนไขเดียวกันได้ พร้อมใช้ `~Q()` ได้ถูกต้อง
- [ ] อธิบาย Race Condition ได้ และรู้ว่าทำไม `F('field') + 1` ปลอดภัยกว่า
      `obj.field += 1; obj.save()`
- [ ] เขียน `Case`/`When` สำหรับ conditional annotation อย่างน้อย 1 แบบ
- [ ] เขียน `Subquery`/`OuterRef` เพื่อดึงค่าจากแถวที่เกี่ยวข้อง (เช่น comment ล่าสุด)
- [ ] เขียน `Window` + `Rank()` เพื่อจัดอันดับข้อมูลภายในกลุ่มได้
- [ ] อธิบายได้ว่าทำไม `values()` ต้องมาก่อน `annotate()` ถึงจะ group by ถูกต้อง
- [ ] รันหน้า Dashboard ตัวอย่างในขั้นตอนที่ 140.1 ได้จริงในโปรเจกต์ของตัวเอง

### 140.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียนฟังก์ชัน `get_category_leaderboard()` ที่คืนรายชื่อหมวดหมู่
เรียงจาก "หมวดหมู่ที่มียอดวิวรวมสูงสุด" ไปน้อยสุด โดยแต่ละแถวต้องมีข้อมูล: ชื่อหมวดหมู่,
จำนวนโพสต์ในหมวดหมู่นั้น, ยอดวิวรวม, และยอดวิวเฉลี่ยต่อโพสต์ (ใช้ `values().annotate()`)

**แบบฝึกหัดที่ 2**: เขียนฟังก์ชัน `increment_view_count(post_id)` ที่เพิ่มยอดวิวของ
โพสต์แบบ atomic ด้วย `F()` แล้วเขียนเทสต์ (ไม่ต้องใช้ pytest เป็นทางการ แค่รันใน shell
ก็พอ) จำลองการเรียกฟังก์ชันนี้ 100 ครั้งติดกัน แล้วตรวจสอบว่า `view_count` เพิ่มขึ้นครบ
100 พอดี

**แบบฝึกหัดที่ 3**: เขียนระบบค้นหาโพสต์ด้วย `Q()` ที่รับพารามิเตอร์ได้หลายแบบพร้อมกัน
(คำค้นหา, ช่วงวันที่เริ่มต้น-สิ้นสุด, หมวดหมู่, สถานะเผยแพร่) โดยพารามิเตอร์ไหนที่ไม่ถูก
ส่งมาให้ข้ามเงื่อนไขนั้นไป (คล้ายกับตัวอย่างในขั้นตอนที่ 134.7 แต่เพิ่มช่วงวันที่เข้าไปด้วย)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ใช้ `Window` กับ `Lag()` เพื่อคำนวณ "ผลต่างของยอดวิว
ระหว่างโพสต์ปัจจุบันกับโพสต์ก่อนหน้า" เมื่อเรียงตามวันที่สร้าง (ใบ้: `Lag()` import
จาก `django.db.models.functions` และใช้ใน `Window(expression=Lag('view_count'), ...)`
แล้วนำผลลัพธ์ไปลบกับ `F('view_count')` อีกที ด้วย `ExpressionWrapper`)

### 140.5 คำถามที่พบบ่อย (FAQ)

**Q: ใช้ `annotate()` กับ `aggregate()` เยอะเกินไปจะทำให้ query ช้าไหม?**
A: ไม่เสมอไป ตราบใดที่ field ที่ใช้ join/filter มี index ที่เหมาะสม (เราจะเรียน
เรื่อง Database Indexing อย่างละเอียดใน Phase 8) การให้ฐานข้อมูลคำนวณเองยังเร็วกว่า
การดึงข้อมูลดิบมาคำนวณใน Python เกือบทุกกรณี เพราะฐานข้อมูลถูกออกแบบมาให้ทำงานกับ
ข้อมูลจำนวนมากอย่างมีประสิทธิภาพโดยเฉพาะ

**Q: ทำไมบางครั้ง `annotate()` คืนตัวเลขที่ดูมากผิดปกติ?**
A: มักเกิดจากการ annotate ผ่านความสัมพันธ์แบบ many-to-many หรือ reverse FK
หลายอันพร้อมกันโดยไม่ใส่ `distinct=True` (อธิบายละเอียดในขั้นตอนที่ 132.5) ให้ตรวจสอบ
`queryset.query` เพื่อดู SQL จริงที่เกิดขึ้นเสมอเมื่อสงสัย

**Q: `F()` ใช้กับ `DateTimeField` ได้ไหม?**
A: ได้ ใช้บวก/ลบ `timedelta` ได้ด้วย เช่น `Post.objects.filter(created_at__lt=F('updated_at') - timedelta(days=7))`
เพื่อหาโพสต์ที่ผ่านมานาน 7 วันแล้วนับจากวันที่แก้ไขล่าสุด

**Q: `Subquery` กับการทำ `JOIN` ธรรมดาต่างกันอย่างไร แล้วเมื่อไรควรใช้อะไร?**
A: `annotate()`/`filter()` ผ่านความสัมพันธ์ปกติจะสร้าง `JOIN` ซึ่งเหมาะกับการรวม/นับ
ข้อมูลจากหลายแถวที่เกี่ยวข้อง ส่วน `Subquery` เหมาะกับตอนที่ต้องการ "แถวเดียวที่เฉพาะ
เจาะจงที่สุด" จากความสัมพันธ์นั้น (เช่น แถวล่าสุด, แถวที่ราคาสูงสุด) ซึ่ง `JOIN` ตรง ๆ
ทำไม่ได้ง่าย ๆ เพราะจะได้หลายแถวกลับมาแทนที่จะเป็นแถวเดียว

**Q: Window Functions ใช้แทน `values().annotate()` ได้เลยไหม?**
A: ใช้แทนกันไม่ได้เสมอไป เพราะจุดประสงค์ต่างกัน — `values().annotate()` "ยุบรวม"
หลายแถวให้เหลือ 1 แถวต่อกลุ่ม (เหมาะกับสรุปข้อมูล) ส่วน `Window` "คงทุกแถวไว้ครบ"
แต่แนบค่าที่คำนวณจากกลุ่มนั้นเข้าไปด้วย (เหมาะกับการจัดอันดับหรือคำนวณสะสมที่ยังต้อง
เห็นรายละเอียดของแต่ละแถวอยู่)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 015: Model Meta Options, Managers และ Custom QuerySets** จะพาไปเรียนรู้วิธี
ปรับแต่งพฤติกรรมของโมเดลผ่าน `class Meta` (การเรียงลำดับ default, index, unique
constraints ระดับหลาย field) และที่สำคัญที่สุดคือการสร้าง **Custom Manager** และ
**Custom QuerySet** ของตัวเอง เพื่อ "ห่อ" query ที่ซับซ้อนที่เราเขียนใน Part นี้
(เช่น `Post.objects.published()`, `Post.objects.with_comment_count()`) ให้เรียกใช้
ซ้ำได้ง่าย อ่านง่าย และทดสอบง่ายขึ้นมาก แทนที่จะต้องเขียน `annotate()`/`Q()`/`F()`
ยาว ๆ ซ้ำ ๆ ทุกที่ที่ต้องใช้

เตรียมทบทวนโค้ดทั้งหมดใน Part นี้ให้คล่องมือ เพราะ Part ถัดไปจะนำ query เหล่านี้
มาจัดระเบียบใหม่ให้ใช้งานได้อย่างมืออาชีพยิ่งขึ้น
