# Part 067: Query Optimization: select_related, prefetch_related

> **ขั้นตอนที่ 661-670 ของหลักสูตร** | Phase 8: Performance & Caching
>
> เป้าหมายของ Part นี้: เจาะลึกปัญหา N+1 Query ด้วยการดู SQL จริงที่เกิดขึ้นเบื้องหลัง
> แล้วแก้ไขมันอย่างเป็นระบบด้วยเครื่องมือหลักสามตัวของ Django ORM คือ
> `select_related()`, `prefetch_related()` และ `Prefetch()` object รวมถึงผสาน
> `only()`/`defer()`/`annotate()` เข้าไปเพื่อบีบจำนวน query และปริมาณข้อมูลให้เหลือ
> น้อยที่สุด คุณจะใช้ django-debug-toolbar (ที่ติดตั้งไว้ใน Part 066) ยืนยันผลลัพธ์จริง
> และปิดท้ายด้วยการเขียน `assertNumQueries()` เพื่อล็อกจำนวน query ไว้ใน test suite
> ป้องกันไม่ให้ N+1 แอบกลับมาในอนาคตโดยไม่มีใครรู้ตัว เมื่อจบ Part นี้ หน้า Homepage
> ของบล็อกที่เคยยิง 40+ query จะเหลือเพียงไม่กี่ query คงที่ ไม่ว่าจะมีกี่บทความก็ตาม

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 661: ทบทวน N+1 Query Problem แบบเจาะลึก พร้อม SQL จริงที่เกิดขึ้น
- ขั้นตอนที่ 662: `select_related()` เจาะลึก — ทำงานอย่างไร (SQL JOIN) ใช้กับความสัมพันธ์แบบไหนได้บ้าง
- ขั้นตอนที่ 663: `prefetch_related()` เจาะลึก — ทำงานอย่างไร (query แยก + join ฝั่ง Python)
- ขั้นตอนที่ 664: `Prefetch()` object — ควบคุม QuerySet ที่ใช้ prefetch แบบละเอียด
- ขั้นตอนที่ 665: ผสาน `only()`/`defer()` กับ `select_related()` เพื่อลดข้อมูลที่ดึงยิ่งขึ้น
- ขั้นตอนที่ 666: ใช้ `annotate()` แทนการ query แยกเพื่อลดจำนวน query
- ขั้นตอนที่ 667: ใช้ django-debug-toolbar ยืนยันผลลัพธ์การลด query count จริง (before/after)
- ขั้นตอนที่ 668: Anti-pattern ที่พบบ่อยในการ optimize query
- ขั้นตอนที่ 669: `assertNumQueries()` ใน test เพื่อป้องกัน N+1 กลับมาใหม่ในอนาคต
- ขั้นตอนที่ 670: สรุปและแบบฝึกหัด — optimize หน้า blog homepage ให้เหลือ query น้อยที่สุด

---

## ขั้นตอนที่ 661: ทบทวน N+1 Query Problem แบบเจาะลึก พร้อม SQL จริงที่เกิดขึ้น

### 661.1 ทบทวนและขยายโมเดลบล็อกที่ใช้ตลอด Part นี้

ก่อนเริ่ม เรามาทบทวนโครงสร้างโมเดล `blog` ที่สร้างไว้ตั้งแต่ Part 011-014 อีกครั้ง
คราวนี้เราจะเพิ่ม field `author` (ผู้เขียนบทความ) และ `is_approved` บน `Comment`
เข้าไปด้วย เพราะหน้า "Homepage บล็อก" ที่เป็นกรณีศึกษาหลักของ Part นี้ต้องแสดงทั้งชื่อ
ผู้เขียน หมวดหมู่ แท็ก และจำนวนคอมเมนต์ที่อนุมัติแล้วพร้อมกันในหน้าเดียว — สถานการณ์
ที่พบบ่อยที่สุดในระบบจริงและเป็นจุดที่ N+1 Query ชอบซ่อนตัวอยู่:

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


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile",
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)

    def __str__(self):
        return f"Profile of {self.user.username}"
```

รัน migration ตามปกติ:

```bash
python manage.py makemigrations blog
python manage.py migrate
```

### 661.2 สร้างข้อมูลตัวอย่างเพื่อสาธิตปัญหา

ปัญหา N+1 จะไม่เห็นชัดเลยถ้ามีข้อมูลแค่ 2-3 แถว เราจึงต้องสร้างข้อมูลจำลองที่มีปริมาณ
สมจริงพอสมควรก่อน สร้าง management command ใหม่:

```python
# blog/management/commands/seed_demo_data.py
from django.contrib.auth import get_user_model
from django.core.management.base import BaseCommand
from django.db import transaction

from blog.models import Category, Comment, Post, Tag

User = get_user_model()


class Command(BaseCommand):
    help = "สร้างข้อมูลตัวอย่างสำหรับสาธิต N+1 Query Problem"

    @transaction.atomic
    def handle(self, *args, **options):
        Comment.objects.all().delete()
        Post.objects.all().delete()
        Tag.objects.all().delete()
        Category.objects.all().delete()
        User.objects.filter(is_superuser=False).delete()

        categories = [
            Category.objects.create(name=name, slug=name.lower())
            for name in ["Django", "Python", "DevOps", "Frontend", "Database"]
        ]
        tags = [
            Tag.objects.create(name=name, slug=name.lower())
            for name in ["orm", "testing", "docker", "api", "security", "cache"]
        ]
        authors = [
            User.objects.create_user(username=f"author{i}", password="pass12345")
            for i in range(1, 6)
        ]

        for i in range(1, 51):
            post = Post.objects.create(
                title=f"บทความที่ {i}",
                slug=f"post-{i}",
                content="เนื้อหาบทความตัวอย่าง " * 20,
                is_published=True,
                category=categories[i % len(categories)],
                author=authors[i % len(authors)],
            )
            post.tags.set(tags[i % 3: i % 3 + 3])
            for j in range(3):
                Comment.objects.create(
                    post=post,
                    author=f"ผู้อ่าน{j}",
                    text=f"ความเห็นที่ {j} ของบทความ {i}",
                    is_approved=(j != 2),
                )

        self.stdout.write(self.style.SUCCESS("สร้างข้อมูลตัวอย่างสำเร็จ: 50 บทความ"))
```

```bash
python manage.py seed_demo_data
```

### 661.3 หน้า Homepage แบบ "ยังไม่ optimize" (สถานการณ์ตั้งต้น)

นี่คือ view และ template ทั่วไปที่นักพัฒนามือใหม่ (และมือกลางจำนวนมาก) เขียนกัน
โดยไม่รู้ตัวว่ากำลังสร้างปัญหา N+1 ไว้:

```python
# blog/views.py
from django.shortcuts import render

from .models import Post


def homepage(request):
    posts = Post.objects.filter(is_published=True)[:10]
    return render(request, "blog/homepage.html", {"posts": posts})
```

```html
<!-- blog/templates/blog/homepage.html -->
<!DOCTYPE html>
<html lang="th">
<head><meta charset="UTF-8"><title>บล็อกของเรา</title></head>
<body>
    <h1>บทความล่าสุด</h1>
    {% for post in posts %}
        <article>
            <h2>{{ post.title }}</h2>
            <p>โดย {{ post.author.username }} | หมวดหมู่: {{ post.category.name }}</p>
            <p>แท็ก:
                {% for tag in post.tags.all %}{{ tag.name }}{% if not forloop.last %}, {% endif %}{% endfor %}
            </p>
            <p>{{ post.comments.count }} ความคิดเห็น</p>
        </article>
    {% endfor %}
</body>
</html>
```

โค้ดนี้ **รันได้จริงและแสดงผลถูกต้อง 100%** แต่มันทำงานช้าลงเรื่อย ๆ เมื่อจำนวน
บทความเพิ่มขึ้น เพราะทุก `{{ post.author.username }}`, `{{ post.category.name }}`,
`{% for tag in post.tags.all %}` และ `{{ post.comments.count }}` แต่ละบรรทัดจะยิง
query ใหม่แยกต่างหากทุกครั้งที่ template loop ไปแตะ attribute นั้น — นี่คือหัวใจของ
ปัญหาที่เราจะพิสูจน์ด้วยตาตัวเองในขั้นตอนถัดไป

### 661.4 จับ SQL จริงด้วย `django.db.connection.queries`

Django เก็บ log ของทุก query ที่รันไว้ใน `connection.queries` **แต่มีเงื่อนไขสำคัญ**:
ต้องอยู่ในสภาวะที่ `settings.DEBUG = True` เท่านั้น (ไม่ว่าจะรันผ่าน shell หรือ view จริง)
มิฉะนั้น list นี้จะว่างเปล่าเสมอเพื่อประหยัด memory ใน production

```bash
python manage.py shell
```

```python
from django.conf import settings
settings.DEBUG = True  # บังคับเปิดไว้ใน shell (ปกติ dev server ตั้งเป็น True อยู่แล้ว)

from django.db import connection, reset_queries
from blog.models import Post

reset_queries()  # เคลียร์ log เก่าทิ้งก่อนเริ่มนับใหม่

posts = list(Post.objects.filter(is_published=True)[:10])
for post in posts:
    _ = post.author.username     # query แยกทุกครั้งที่เข้าถึง author ของ post คนละตัว
    _ = post.category.name       # query แยกทุกครั้งที่เข้าถึง category ของ post คนละตัว
    _ = list(post.tags.all())    # query แยกทุกครั้งเช่นกัน
    _ = post.comments.count()    # query แยกทุกครั้งเช่นกัน

print(f"จำนวน query ทั้งหมด: {len(connection.queries)}")
for i, q in enumerate(connection.queries, start=1):
    print(f"{i}. [{q['time']}s] {q['sql'][:100]}...")
```

ผลลัพธ์ที่ได้ (ตัวเลขจริงจากข้อมูล 10 โพสต์):

```
จำนวน query ทั้งหมด: 41
1. [0.001s] SELECT "blog_post"."id", ... FROM "blog_post" WHERE "blog_post"."is_published" = 1 LIMIT 10...
2. [0.000s] SELECT "auth_user"."id", "auth_user"."username" FROM "auth_user" WHERE "auth_user"."id" = 2...
3. [0.000s] SELECT "blog_category"."id", "blog_category"."name" FROM "blog_category" WHERE "blog_category"."id" = 1...
4. [0.000s] SELECT "blog_tag"."id", "blog_tag"."name" FROM "blog_tag" INNER JOIN "blog_post_tags" ...
5. [0.000s] SELECT COUNT(*) FROM "blog_comment" WHERE "blog_comment"."post_id" = 1...
... (ซ้ำรูปแบบเดิมอีก 9 รอบ สำหรับโพสต์ที่เหลือ)
```

### 661.5 สูตรคำนวณจำนวน Query ของ N+1 Problem

```
จำนวน query ทั้งหมด = 1 (ดึง posts) + N × (จำนวน relation ที่ access ต่อ 1 post)

กรณีนี้: N = 10 โพสต์, relation ที่ access = 4 (author, category, tags, comments.count)
จำนวน query = 1 + 10 × 4 = 41 query
```

| จำนวนโพสต์ (N) | จำนวน Query (ก่อน optimize) | สังเกต |
|---|---|---|
| 10 | 41 | 1 + 10×4 |
| 50 | 201 | 1 + 50×4 |
| 200 | 801 | 1 + 200×4 |
| 1,000 | 4,001 | 1 + 1000×4 — ระบบจะล่มก่อนถึงจุดนี้แน่นอน |

นี่คือเหตุผลที่ปัญหานี้ชื่อว่า "**N+1**": จำนวน query เติบโตเป็นเส้นตรงตามจำนวนแถว
ของ query แรก (N) บวกด้วย query แรกนั้นเอง (+1) — ยิ่งข้อมูลเยอะ ยิ่งช้าเป็นเชิงเส้น
ในขณะที่ query ที่ optimize ดีแล้วควรใช้จำนวน query **คงที่ (constant)** ไม่ว่า N
จะเป็นเท่าไรก็ตาม ซึ่งคือเป้าหมายที่เราจะไปถึงในตอนท้าย Part นี้

### 661.6 ทำไม `len(queryset)` หรือ `list(queryset)` ก่อน loop จึงสำคัญ

สังเกตว่าในโค้ด 661.4 เราเขียน `posts = list(Post.objects.filter(...)[:10])` ก่อน
เข้า loop เสมอ เพราะ QuerySet ของ Django เป็น **lazy** — มันยังไม่ยิง query จริงจนกว่า
จะถูก evaluate (เช่นด้วย `list()`, `for`, `len()`, หรือ slicing ที่มี step) การบังคับ
evaluate ก่อนด้วย `list()` ทำให้เรานับ query "ของ query แรก" แยกออกจาก query ที่เกิด
จากการ loop เข้าถึง attribute แต่ละตัวได้ชัดเจน ซึ่งเป็นวินัยที่มีประโยชน์มากเวลา debug
ปัญหาแบบนี้ในโค้ดจริง

---

## ขั้นตอนที่ 662: `select_related()` เจาะลึก — ทำงานอย่างไร (SQL JOIN) ใช้ได้กับความสัมพันธ์แบบไหน

### 662.1 กลไกเบื้องหลัง: SQL JOIN เดียวจบ

`select_related()` แก้ปัญหาการ query แยกสำหรับ `author` และ `category` ด้วยการรวม
ทุกอย่างเข้าเป็น **query เดียว** ผ่าน SQL `JOIN` — ฐานข้อมูลจะดึงข้อมูลของตารางแม่และ
ตารางที่เชื่อมด้วย FK มาพร้อมกันในแถวเดียวกันเลย ไม่ต้องกลับไปถามฐานข้อมูลซ้ำอีก:

```python
from blog.models import Post

posts = Post.objects.filter(is_published=True).select_related("category", "author")[:10]
print(posts.query)
```

```sql
SELECT
    "blog_post"."id", "blog_post"."title", ..., "blog_post"."category_id", "blog_post"."author_id",
    "blog_category"."id", "blog_category"."name", "blog_category"."slug",
    "auth_user"."id", "auth_user"."username", "auth_user"."password", ...
FROM "blog_post"
LEFT OUTER JOIN "blog_category" ON ("blog_post"."category_id" = "blog_category"."id")
LEFT OUTER JOIN "auth_user" ON ("blog_post"."author_id" = "auth_user"."id")
WHERE "blog_post"."is_published" = 1
LIMIT 10
```

ทดสอบ query count จริง:

```python
from django.db import connection, reset_queries

reset_queries()
posts = list(Post.objects.filter(is_published=True).select_related("category", "author")[:10])
for post in posts:
    _ = post.author.username     # ไม่ query เพิ่ม! ข้อมูลอยู่ในหน่วยความจำแล้วจาก JOIN
    _ = post.category.name       # ไม่ query เพิ่มเช่นกัน

print(len(connection.queries))
# 1   <- เหลือแค่ query เดียว ไม่ว่าจะมีกี่โพสต์ก็ตาม
```

### 662.2 ใช้ได้กับความสัมพันธ์แบบไหนบ้าง

`select_related()` ใช้ได้เฉพาะความสัมพันธ์ที่ **แต่ละแถวมีคู่เดียวแน่นอน (single-valued
relation)** เท่านั้น เพราะกลไก JOIN แบบนี้จะรวมข้อมูลของอีกฝั่งเข้ามาเป็น "คอลัมน์เพิ่ม"
ในแถวเดียวกัน ถ้าอีกฝั่งมีได้หลายค่า การรวมแบบนี้จะทำให้แถวหลักถูกคูณซ้ำ (Cartesian
join) ซึ่งผิดความหมายที่ต้องการ:

| ความสัมพันธ์ | ใช้ `select_related()` ได้ไหม | เหตุผล |
|---|---|---|
| `ForeignKey` (ฝั่ง "many" มองไปยัง "one") | ✅ ได้ | 1 โพสต์มี category เดียวแน่นอน |
| `OneToOneField` (ทั้งสองทิศทาง) | ✅ ได้ | ทั้งสองฝั่งมีคู่เดียวเสมอ |
| `ManyToManyField` | ❌ ไม่ได้ | 1 โพสต์มีได้หลายแท็ก — ไม่ใช่ single-valued |
| Reverse `ForeignKey` (ฝั่ง "one" มองไปยัง "many") | ❌ ไม่ได้ | 1 หมวดหมู่มีได้หลายโพสต์ |

ลองใช้ผิดประเภทดูจะเห็น error ที่ Django บอกตรงตัวมาก:

```python
Post.objects.select_related("comments")
```

```
django.core.exceptions.FieldError: Invalid field name(s) given in select_related:
'comments'. Choices are: category, author
```

สังเกตว่า Django บอกตัวเลือกที่ใช้ได้จริงมาให้เลย (`category`, `author`) ซึ่งคือทุก
`ForeignKey`/`OneToOneField` ที่ประกาศบน `Post`

### 662.3 Multi-level select_related ด้วย Double Underscore

`select_related()` เดินทางลึกได้หลายชั้นด้วย `__` เหมือน `filter()` ทุกประการ — เช่น
ถ้าอยากได้ทั้ง `author` และ `author.profile` มาในคำสั่งเดียว:

```python
posts = Post.objects.select_related("category", "author__profile")[:10]

for post in posts:
    print(post.author.username, post.author.profile.bio)  # ไม่ query เพิ่มเลยทั้งคู่
```

Django จะสร้าง SQL ที่ JOIN สามตารางพร้อมกันในคำสั่งเดียว (`blog_post` → `auth_user`
→ `accounts_profile`):

```sql
SELECT ...
FROM "blog_post"
LEFT OUTER JOIN "blog_category" ON (...)
LEFT OUTER JOIN "auth_user" ON (...)
LEFT OUTER JOIN "accounts_profile" ON ("auth_user"."id" = "accounts_profile"."user_id")
WHERE "blog_post"."is_published" = 1
```

### 662.4 `INNER JOIN` vs `LEFT OUTER JOIN` — Django เลือกให้อัตโนมัติ

สังเกตจาก SQL ในข้อ 662.1 ว่า Django ใช้ `LEFT OUTER JOIN` ทั้งสำหรับ `category`
และ `author` เพราะทั้งสอง field ถูกประกาศด้วย `null=True` (`on_delete=models.SET_NULL`)
— แถวที่ `category_id` หรือ `author_id` เป็น `NULL` ก็ยังต้องถูกดึงมาด้วย (ไม่งั้น
โพสต์ที่ไม่มีหมวดหมู่จะหายไปจากผลลัพธ์ทั้งที่มันควรอยู่)

ถ้า field เป็น **non-nullable** (เช่น `Comment.post` ที่ `on_delete=models.CASCADE`
ไม่มี `null=True`) Django จะฉลาดพอที่จะใช้ `INNER JOIN` แทน ซึ่งเร็วกว่าเล็กน้อยเพราะ
ฐานข้อมูลรู้ล่วงหน้าว่าทุกแถวต้องมีคู่จับคู่ได้แน่นอน ไม่ต้องเผื่อกรณี NULL:

```python
comments = Comment.objects.select_related("post")
print(comments.query)
```

```sql
SELECT ...
FROM "blog_comment"
INNER JOIN "blog_post" ON ("blog_comment"."post_id" = "blog_post"."id")
```

| ลักษณะ field | JOIN ที่ Django สร้างให้ |
|---|---|
| `ForeignKey(null=True)` | `LEFT OUTER JOIN` |
| `ForeignKey` (ไม่มี `null=True`) | `INNER JOIN` |
| `OneToOneField(null=True)` | `LEFT OUTER JOIN` |
| `OneToOneField` (ไม่มี `null=True`) | `INNER JOIN` |

### 662.5 ตารางสรุป select_related()

| ประเด็น | รายละเอียด |
|---|---|
| กลไก | SQL `JOIN` เดียว รวมข้อมูลของทุกความสัมพันธ์เป็นแถวเดียว |
| ใช้ได้กับ | `ForeignKey`, `OneToOneField` (forward และ reverse) เท่านั้น |
| ใช้ไม่ได้กับ | `ManyToManyField`, reverse `ForeignKey` |
| จำนวน query ที่ใช้ | 1 query เสมอ ไม่ว่าจะ select_related กี่ field |
| Multi-level | ใช้ `__` เดินทางลึกได้ เช่น `select_related('author__profile')` |
| ผลต่อขนาด query | คอลัมน์กว้างขึ้น (ดึงทุกคอลัมน์ของตารางที่ join มาด้วย เว้นแต่ใช้ `only()`) |

---

## ขั้นตอนที่ 663: `prefetch_related()` เจาะลึก — ทำงานอย่างไร (query แยก + join ฝั่ง Python)

### 663.1 ทำไม `tags` และ `comments` ใช้ select_related() ไม่ได้

`Post.tags` เป็น `ManyToManyField` และ `Post.comments` เป็น reverse `ForeignKey`
— ทั้งคู่เป็นความสัมพันธ์แบบ **multi-valued** (1 โพสต์มีได้หลายแท็ก/หลายคอมเมนต์)
ถ้า Django พยายาม JOIN ตารางเหล่านี้เข้ากับ `blog_post` ตรง ๆ ในคำสั่งเดียว แถวของ
`Post` แต่ละแถวจะถูก **คูณซ้ำ** ตามจำนวนแท็ก/คอมเมนต์ที่มี (เช่น โพสต์ที่มี 3 แท็ก
จะกลายเป็น 3 แถวซ้ำของโพสต์เดียวกันใน result set) ทำให้ Django ต้องมาแยกแถวซ้ำออกอีก
ที ซึ่งซับซ้อนและสิ้นเปลืองกว่าวิธีที่สอง คือ **`prefetch_related()`**

### 663.2 กลไกเบื้องหลัง: Query แยก + จับคู่ (join) ฝั่ง Python

`prefetch_related()` ทำงานด้วยแนวคิดคนละแบบกับ `select_related()` โดยสิ้นเชิง:

1. รัน query แรกตามปกติเพื่อดึง object หลัก (เช่น `Post`) ทั้งหมด
2. เก็บ primary key ของทุก object ที่ได้ไว้
3. รัน query ที่สอง (แยกต่างหาก) เพื่อดึง object ที่เกี่ยวข้องทั้งหมด โดยใช้
   `WHERE ... IN (pk1, pk2, pk3, ...)` — ดึงมาทีเดียวครบทุกโพสต์ ไม่ใช่ทีละโพสต์
4. Django **จับคู่ผลลัพธ์ทั้งสอง query เข้าด้วยกันในหน่วยความจำ Python** (ไม่ใช่ SQL
   JOIN) แล้วผูกเข้ากับ object หลักแต่ละตัวให้พร้อมใช้งานราวกับถูก join มาแล้ว

```python
from django.db import connection, reset_queries
from blog.models import Post

reset_queries()
posts = list(Post.objects.filter(is_published=True).prefetch_related("tags")[:10])
for post in posts:
    _ = list(post.tags.all())  # ไม่ query เพิ่ม! ข้อมูลถูกจับคู่ไว้ในหน่วยความจำแล้ว

print(len(connection.queries))
# 2   <- query แรกดึง posts, query สองดึง tags ของโพสต์ทั้ง 10 อันพร้อมกัน
for q in connection.queries:
    print(q["sql"])
```

```sql
-- Query 1: ดึงโพสต์ 10 อันตามปกติ
SELECT "blog_post".* FROM "blog_post" WHERE "blog_post"."is_published" = 1 LIMIT 10;

-- Query 2: ดึง tags ของโพสต์ทั้ง 10 อันในคำสั่งเดียว ด้วย IN (...)
SELECT "blog_tag"."id", "blog_tag"."name", "blog_post_tags"."post_id" AS "_prefetch_related_val_post_id"
FROM "blog_tag"
INNER JOIN "blog_post_tags" ON ("blog_tag"."id" = "blog_post_tags"."tag_id")
WHERE "blog_post_tags"."post_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
```

สังเกตคอลัมน์พิเศษ `_prefetch_related_val_post_id` — นี่คือ "กุญแจ" ที่ Django ใช้
จับคู่แท็กแต่ละแท็กกลับไปยังโพสต์ที่ถูกต้องในหน่วยความจำ Python

### 663.3 ใช้กับ Reverse ForeignKey: `comments`

```python
reset_queries()
posts = list(Post.objects.filter(is_published=True).prefetch_related("comments")[:10])
for post in posts:
    for comment in post.comments.all():  # ไม่ query เพิ่ม
        pass

print(len(connection.queries))
# 2   <- 1 สำหรับ posts, 1 สำหรับ comments ทั้งหมดของโพสต์ 10 อัน (WHERE post_id IN (...))
```

### 663.4 ใช้ได้กับความสัมพันธ์แบบไหนบ้าง

| ความสัมพันธ์ | ใช้ `prefetch_related()` ได้ไหม | หมายเหตุ |
|---|---|---|
| `ManyToManyField` | ✅ ได้ (กรณีหลักที่ต้องใช้) | `Post.tags`, `Tag.posts` (ทั้งสองทิศทาง) |
| Reverse `ForeignKey` | ✅ ได้ (กรณีหลักที่ต้องใช้) | `Post.comments`, `Category.posts` |
| `ForeignKey` (forward) | ✅ ใช้ได้ แต่ไม่ค่อยจำเป็น | ใช้เมื่อมีเหตุผลเฉพาะ ดู 663.5 |
| `OneToOneField` | ✅ ใช้ได้ แต่ไม่ค่อยจำเป็น | เช่นเดียวกับข้างบน |

ต่างจาก `select_related()` ตรงที่ **`prefetch_related()` ใช้ได้กับความสัมพันธ์ทุก
ประเภท** ไม่มีข้อจำกัด เพียงแต่สำหรับ `ForeignKey`/`OneToOneField` การใช้
`select_related()` มักจะดีกว่าเสมอ (1 query แทนที่จะเป็น 2 query)

### 663.5 เมื่อไหร่ควรใช้ prefetch_related() กับ ForeignKey แทน select_related()

มีบางสถานการณ์ที่จงใจใช้ `prefetch_related()` กับความสัมพันธ์แบบ FK แทน:

```python
from django.db.models import Prefetch
from blog.models import Category

# ต้องการ Category พร้อมด้วย Post ทั้งหมดในหมวดหมู่นั้น ที่ควบคุม field ให้แคบมาก
categories = Category.objects.prefetch_related(
    Prefetch("posts", queryset=Post.objects.only("id", "title", "author_id"))
)
```

เหตุผลคือเมื่อโมเดลปลายทางมีคอลัมน์จำนวนมาก (เช่น `TextField` ขนาดใหญ่) การ
`select_related()` จะทำให้แถวของ query หลักกว้างขึ้นทุกแถว (แม้จะไม่ได้ใช้ข้อมูลนั้น
ทุกแถวก็ตาม) ในขณะที่ `prefetch_related()` แยก query ออกมาต่างหาก ทำให้ควบคุมขนาด
ข้อมูลของแต่ละ query ได้อิสระกว่า — เราจะเจาะลึกเรื่องนี้ต่อในขั้นตอนที่ 664-665

### 663.6 ผสาน select_related() และ prefetch_related() เข้าด้วยกัน

ทั้งสองเมธอดใช้ร่วมกันในคำสั่งเดียวได้เสมอ (และมักต้องใช้ร่วมกันในงานจริง):

```python
posts = (
    Post.objects.filter(is_published=True)
    .select_related("category", "author")
    .prefetch_related("tags", "comments")
    [:10]
)
```

ผลลัพธ์: **1 query สำหรับ posts+category+author (JOIN) + 1 query สำหรับ tags +
1 query สำหรับ comments = 3 query รวม** ไม่ว่าจะมีกี่โพสต์ก็ตาม (คงที่)

### 663.7 ตารางเปรียบเทียบ select_related() vs prefetch_related()

| ประเด็น | `select_related()` | `prefetch_related()` |
|---|---|---|
| กลไก | SQL `JOIN` เดียว | Query แยก + จับคู่ในหน่วยความจำ Python |
| จำนวน query ที่เพิ่ม | ไม่เพิ่ม (รวมเข้า query เดิม) | +1 query ต่อ 1 relation ที่ prefetch |
| ใช้ได้กับ | `ForeignKey`, `OneToOneField` เท่านั้น | ทุกความสัมพันธ์ (M2M, reverse FK, FK, O2O) |
| ขนาดข้อมูลต่อแถว | แถวกว้างขึ้น (คอลัมน์ของตาราง join มาด้วย) | แถวของ query หลักไม่เปลี่ยน |
| ควบคุม queryset ของฝั่งที่ดึง | ทำไม่ได้ (ดึงทั้งหมดของ related object เสมอ) | ทำได้เต็มที่ผ่าน `Prefetch()` object |
| เหมาะกับข้อมูลปริมาณ | น้อย-ปานกลาง ต่อ 1 แถว (แค่ 1 ค่า) | มาก (list ของหลาย object ต่อ 1 แถว) |

---

## ขั้นตอนที่ 664: `Prefetch()` object — ควบคุม QuerySet ที่ใช้ prefetch แบบละเอียด

### 664.1 ทำไมต้องมี `Prefetch()` object

`prefetch_related("comments")` แบบธรรมดาจะดึง **คอมเมนต์ทั้งหมดของทุกโพสต์** มาโดย
ไม่มีเงื่อนไข แต่ในหน้า Homepage จริง เราอยากได้แค่ "คอมเมนต์ที่อนุมัติแล้ว" และอยาก
ให้เรียงจากใหม่ไปเก่า — นี่คือหน้าที่ของ `Prefetch()` object ซึ่งให้เราส่ง **QuerySet
ที่ปรับแต่งเองได้เต็มรูปแบบ** เข้าไปแทนที่จะใช้ manager เริ่มต้นของ Django

```python
from django.db.models import Prefetch
from blog.models import Comment, Post

posts = Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).order_by("-created_at"),
    )
)

for post in posts:
    for comment in post.comments.all():   # ได้เฉพาะคอมเมนต์ที่ is_approved=True เรียงใหม่สุดก่อน
        print(comment.text)
```

พารามิเตอร์หลักของ `Prefetch()`:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `lookup` (ตัวแรก, ไม่มีชื่อ) | ชื่อ relation เดียวกับที่ใช้ใน `prefetch_related()` ปกติ เช่น `"comments"` |
| `queryset` | QuerySet ที่กำหนดเองแทน manager เริ่มต้น (filter/order_by/select_related ได้อิสระ) |
| `to_attr` | ชื่อ attribute ใหม่ที่จะเก็บผลลัพธ์ (ดูข้อ 664.2) |

### 664.2 `to_attr` — เก็บผลลัพธ์ไว้ใน attribute แยกต่างหาก

ถ้าไม่ระบุ `to_attr` ผลลัพธ์ของ `Prefetch()` จะไปแทนที่ cache ของ `post.comments.all()`
ตามปกติ แต่ถ้าต้องการเก็บไว้เป็น attribute ใหม่ (เพื่อไม่ไปรบกวนการใช้งาน
`post.comments.all()` แบบเดิมที่อื่นในโค้ด หรือเพื่อเก็บผลลัพธ์ที่ **slice** แล้ว) ใช้
`to_attr`:

```python
from django.db.models import Prefetch
from blog.models import Comment, Post

posts = Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).order_by("-created_at"),
        to_attr="recent_comments",
    )
)

for post in posts:
    # post.comments.all() ยังคง query ปกติ (unfiltered) เหมือนเดิมถ้าถูกเรียก
    # post.recent_comments เป็น list ธรรมดา (ไม่ใช่ QuerySet) ที่ผ่านการ filter ไว้แล้ว
    print(len(post.recent_comments))
```

> **ข้อสำคัญ**: `post.recent_comments` เป็น **Python `list` ธรรมดา** ไม่ใช่ QuerySet
> จึงเรียกเมธอดของ QuerySet เพิ่ม (เช่น `.filter()` ต่ออีกที) ไม่ได้ — ต้องกำหนดเงื่อนไข
> ทั้งหมดไว้ใน `queryset=` ของ `Prefetch()` ให้ครบตั้งแต่ต้น

**เฉพาะเมื่อใช้ `to_attr` เท่านั้นที่ slice QuerySet ของ `Prefetch()` ได้** เช่น
ดึงมาแค่ 3 คอมเมนต์ล่าสุดต่อโพสต์:

```python
posts = Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).order_by("-created_at"),
        to_attr="recent_comments",
    )
)
# ต้อง slice ใน Python เพราะ slicing ต่อโพสต์แต่ละอันทำใน SQL เดียวไม่ได้
for post in posts:
    post.recent_comments = post.recent_comments[:3]
```

> **หมายเหตุ**: การ slice ระดับ "3 รายการล่าสุด**ต่อโพสต์**" ในคำสั่ง SQL เดียวจริง ๆ
> ต้องใช้ Window Function (`Rank()` ที่เรียนไปแล้วใน Part 014 ขั้นตอนที่ 138) ผสมกับ
> `Subquery` ซึ่งซับซ้อนกว่ามาก สำหรับข้อมูลระดับหลักร้อย-พันแถว การ slice ใน Python
> หลัง prefetch (ตามตัวอย่างข้างต้น) เร็วพอและอ่านง่ายกว่ามาก แนะนำให้ใช้ Window
> Function เฉพาะเมื่อ N ของ comment ต่อโพสต์มีขนาดใหญ่จริง ๆ (หลักพันขึ้นไป)

### 664.3 หลาย `Prefetch()` บน relation เดียวกัน ด้วย `to_attr` คนละชื่อ

จุดเด่นอีกอย่างของ `to_attr` คือทำให้ prefetch **relation เดียวกันได้หลายแบบพร้อมกัน**
ในคำสั่งเดียว (เช่น ต้องการทั้ง "คอมเมนต์ที่อนุมัติแล้ว" และ "คอมเมนต์ที่รอตรวจ" แยกกัน):

```python
posts = Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).order_by("-created_at"),
        to_attr="approved_comments",
    ),
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=False),
        to_attr="pending_comments",
    ),
)

for post in posts:
    print(f"อนุมัติแล้ว {len(post.approved_comments)} | รอตรวจ {len(post.pending_comments)}")
```

สิ่งนี้ทำไม่ได้เลยถ้าไม่มี `to_attr` เพราะถ้าไม่ตั้งชื่อแยก ทั้งสอง `Prefetch()` จะ
แย่งกันเขียนทับ cache ของ `post.comments` ตัวเดียวกัน

### 664.4 ผสาน `select_related()` เข้าไปใน `queryset` ของ `Prefetch()`

QuerySet ที่ส่งเข้า `Prefetch()` เป็น QuerySet ปกติทุกประการ จึงเรียก `select_related()`
ต่อได้ตามใจ เพื่อ optimize ความสัมพันธ์ **ภายใน** ผลลัพธ์ที่ prefetch มาอีกชั้นหนึ่ง:

```python
from django.db.models import Prefetch
from blog.models import Category, Post

categories = Category.objects.prefetch_related(
    Prefetch(
        "posts",
        queryset=Post.objects.filter(is_published=True).select_related("author"),
    )
)

for category in categories:
    for post in category.posts.all():
        print(post.title, post.author.username)  # author ไม่ query เพิ่มเช่นกัน
```

รวมทั้งหมดใช้ query เพียง **2 ครั้ง**: 1 สำหรับ `Category`, 1 สำหรับ `Post` ที่ JOIN
กับ `author` ไปพร้อมกันในตัว (แม้จะอยู่ "ข้างใน" ของ `prefetch_related()` ระดับบนสุด)

### 664.5 ตารางสรุปการใช้งาน `Prefetch()`

| ต้องการ | วิธีเขียน |
|---|---|
| Prefetch ธรรมดา (ทั้งหมด ไม่กรอง) | `prefetch_related("comments")` |
| กรอง/เรียงลำดับผลลัพธ์ที่ prefetch | `Prefetch("comments", queryset=Comment.objects.filter(...).order_by(...))` |
| เก็บผลลัพธ์แยกจาก manager เดิม | เพิ่ม `to_attr="ชื่อ_attribute"` |
| Slice ผลลัพธ์ (เช่น 3 รายการล่าสุด) | ต้องใช้ `to_attr` แล้ว slice list ที่ได้ใน Python |
| Prefetch relation เดียวกันหลายแบบ | ใช้ `Prefetch()` หลายตัว คนละ `to_attr` |
| Optimize ความสัมพันธ์ซ้อนภายใน prefetch | เรียก `.select_related()`/`.prefetch_related()` ต่อบน `queryset=` ได้เลย |

---

## ขั้นตอนที่ 665: ผสาน `only()`/`defer()` กับ `select_related()` เพื่อลดข้อมูลที่ดึงยิ่งขึ้น

### 665.1 ทบทวน `only()` และ `defer()`

- **`only(*fields)`**: ดึง **เฉพาะ field ที่ระบุ** เท่านั้น (บวก primary key เสมอ)
  field อื่นที่ไม่ได้ระบุจะถูก "defer" (ไม่ดึงมาตอนแรก)
- **`defer(*fields)`**: ตรงข้ามกับ `only()` — ดึง **ทุก field ยกเว้น** ที่ระบุ

```python
from blog.models import Post

# ดึงเฉพาะ title, slug (และ id ที่ติดมาเสมอ) — เร็วขึ้นเมื่อ content เป็น TextField ขนาดใหญ่
posts = Post.objects.only("title", "slug")

# ดึงทุกอย่างยกเว้น content — เหมาะกับหน้า list ที่ไม่ต้องแสดงเนื้อหาเต็ม
posts = Post.objects.defer("content")
```

### 665.2 ผสาน `only()` กับ `select_related()`: ระบุ field ของตารางที่ join ด้วย `__`

จุดที่ทรงพลังที่สุดคือการใช้ `only()` ร่วมกับ `select_related()` เพื่อควบคุมว่าจาก
ตารางที่ JOIN มา อยากได้ **เฉพาะคอลัมน์ไหน** ไม่ใช่ทั้งตาราง:

```python
posts = (
    Post.objects.filter(is_published=True)
    .select_related("category", "author")
    .only(
        "id", "title", "slug", "created_at", "view_count",
        "category__id", "category__name",
        "author__id", "author__username",
    )
)
print(posts.query)
```

```sql
SELECT
    "blog_post"."id", "blog_post"."title", "blog_post"."slug",
    "blog_post"."created_at", "blog_post"."view_count",
    "blog_post"."category_id", "blog_post"."author_id",
    "blog_category"."id", "blog_category"."name",
    "auth_user"."id", "auth_user"."username"
FROM "blog_post"
LEFT OUTER JOIN "blog_category" ON (...)
LEFT OUTER JOIN "auth_user" ON (...)
WHERE "blog_post"."is_published" = 1
```

เทียบกับ `select_related()` เฉย ๆ ที่จะดึง **ทุกคอลัมน์** ของ `auth_user` มาด้วย
(รวมถึง `password`, `email`, `is_staff`, `date_joined` ฯลฯ ที่ไม่ได้ใช้ในหน้านี้เลย)
— การใส่ `only()` ช่วยลดปริมาณข้อมูลที่ส่งผ่านเครือข่ายระหว่าง Django กับฐานข้อมูล
ได้จริงในระดับที่วัดผลได้เมื่อตารางมีคอลัมน์จำนวนมากหรือมีข้อมูลปริมาณสูง

### 665.3 กับดัก: เข้าถึง field ที่ deferred ทำให้เกิด query เพิ่มโดยไม่รู้ตัว

```python
from django.db import connection, reset_queries
from blog.models import Post

reset_queries()
post = Post.objects.only("title", "slug").first()
print(post.title)      # ไม่ query เพิ่ม (title อยู่ใน only() แล้ว)
print(post.content)    # query เพิ่มทันที! เพราะ content ถูก defer ไว้

print(len(connection.queries))
# 2   <- 1 สำหรับ .first(), 1 สำหรับดึง content ตอนเข้าถึง (lazy load)
```

Django ไม่ raise error เมื่อเข้าถึง field ที่ deferred ไว้ แต่จะยิง query เพิ่มแบบ
เงียบ ๆ เพื่อไปดึงค่านั้นมาให้ (เรียกว่า **deferred loading**) — นี่คือ N+1 อีกรูปแบบ
หนึ่งที่มองข้ามได้ง่ายมาก ถ้า template หรือ serializer ไปเข้าถึง field ที่ไม่ได้อยู่ใน
`only()` โดยไม่ตั้งใจ ทุกแถวจะเกิด query เพิ่มขึ้นมาอีกหนึ่งครั้ง

> **กฎของหลักสูตรนี้**: ใช้ `only()` เฉพาะเมื่อ **แน่ใจ 100%** ว่า template/serializer/
> logic ที่ตามมาจะไม่แตะ field อื่นนอกเหนือจากที่ระบุไว้ ถ้าไม่แน่ใจ ให้ใช้ `defer()`
> กับ field ขนาดใหญ่ที่รู้แน่ชัดว่าไม่ต้องใช้ (เช่น `content`) แทน ซึ่งปลอดภัยกว่าเพราะ
> field ใหม่ที่เพิ่มเข้ามาทีหลังจะไม่ถูก defer ไปด้วยโดยไม่ตั้งใจ

### 665.4 กับดักที่สอง: `only()` กับ related object ที่ไม่ครบ field ที่จำเป็น

```python
posts = Post.objects.select_related("author").only("title", "author__username")

for post in posts:
    print(post.author.username)   # ไม่ query เพิ่ม (มีใน only())
    print(post.author.email)      # query เพิ่มทันที! email ไม่ได้อยู่ใน only()
```

หลักการเดียวกับข้อ 665.3 แต่เกิดกับ related object แทน — ทุกครั้งที่ผสาน `only()`
กับ `select_related()` ต้องตรวจสอบให้ครบว่า field ทุกตัวที่จะถูกใช้งานจริง (ทั้งฝั่ง
model หลักและ related model) รวมอยู่ใน `only()` แล้ว

### 665.5 ตัวอย่างจริง: Query ของหน้า Homepage แบบ optimize บางส่วน (ยังไม่รวม tags/comments)

```python
posts = (
    Post.objects.filter(is_published=True)
    .select_related("category", "author")
    .only(
        "id", "title", "slug", "created_at", "view_count",
        "category__id", "category__name",
        "author__id", "author__username",
    )
    .order_by("-created_at")[:10]
)
```

query นี้ให้ทุกอย่างที่ template ต้องใช้สำหรับ `author.username` และ `category.name`
มาในคำสั่งเดียว พร้อมจำกัดคอลัมน์ให้แคบที่สุดเท่าที่จำเป็น — ในขั้นตอนที่ 666 เราจะ
เติม `tags` และ `comment_count` เข้าไปให้ครบสมบูรณ์

### 665.6 ตารางสรุป only()/defer() ร่วมกับ select_related()

| สถานการณ์ | วิธีเขียน |
|---|---|
| ดึงเฉพาะบาง field ของ model หลัก | `.only("title", "slug")` |
| ดึงทุก field ยกเว้นบาง field ขนาดใหญ่ | `.defer("content")` |
| ดึงเฉพาะบาง field ของ related object | `.select_related("category").only("title", "category__name")` |
| ต้องระวังเสมอ | field ที่ไม่ได้ระบุใน `only()` แต่ถูกเข้าถึง → เกิด query เพิ่มแบบเงียบ |
| ปลอดภัยกว่าเมื่อไม่แน่ใจ field ที่จะใช้ครบ | ใช้ `defer()` กับ field ที่รู้แน่ว่าไม่ใช้ แทน `only()` |

---

## ขั้นตอนที่ 666: ใช้ `annotate()` แทนการ query แยกเพื่อลดจำนวน query

### 666.1 `post.comments.count()` ใน Template คือ N+1 ที่ซ่อนอยู่

กลับไปดู template ในขั้นตอนที่ 661.3 บรรทัด `{{ post.comments.count }}` — แม้จะดู
ไม่เหมือน query เลยเพราะเขียนในภาษา template แต่ Django Template Language จะแปล
`post.comments.count` เป็นการเรียก `post.comments.count()` จริง ๆ ซึ่งยิง SQL
`SELECT COUNT(*)` แยกทุกครั้งที่ template loop ไปแตะโพสต์คนละตัว — เป็น N query
เพิ่มเข้ามาอย่างเงียบ ๆ ที่ `prefetch_related("comments")` แก้ปัญหาได้ก็จริง แต่จะ
ดึง **คอมเมนต์ทั้งหมด** มาไว้ในหน่วยความจำทั้งที่ต้องการแค่ "ตัวเลขจำนวน" เท่านั้น

### 666.2 แก้ด้วย `annotate(Count(...))` แทน

ทบทวนจาก Part 014 ขั้นตอนที่ 132: `annotate()` เพิ่ม field ที่คำนวณแล้วให้ทุก object
ในคำสั่งเดียว — เหมาะที่สุดสำหรับสถานการณ์นี้ เพราะเราต้องการแค่ "จำนวน" ไม่ใช่ตัว
object คอมเมนต์เอง:

```python
from django.db.models import Count
from blog.models import Post

posts = Post.objects.filter(is_published=True).annotate(
    comment_count=Count("comments", distinct=True)
)

for post in posts:
    print(post.title, post.comment_count)  # ไม่ query เพิ่มเลย เป็นตัวเลขที่ติดมากับแถวอยู่แล้ว
```

```sql
SELECT "blog_post".*, COUNT(DISTINCT "blog_comment"."id") AS "comment_count"
FROM "blog_post"
LEFT OUTER JOIN "blog_comment" ON ("blog_post"."id" = "blog_comment"."post_id")
WHERE "blog_post"."is_published" = 1
GROUP BY "blog_post"."id"
```

ปรับ template จาก `{{ post.comments.count }}` เป็น `{{ post.comment_count }}`:

```html
<p>{{ post.comment_count }} ความคิดเห็น</p>
```

### 666.3 ทำไม annotate() ดีกว่า prefetch_related() เมื่อต้องการแค่ "จำนวน"

| วิธี | จำนวน query เพิ่ม | ข้อมูลที่โหลดเข้าหน่วยความจำ | เหมาะกับ |
|---|---|---|---|
| `post.comments.count()` (ไม่ optimize) | +N (1 ต่อโพสต์) | ไม่มี (นับที่ DB) | ❌ ไม่เหมาะเลย |
| `prefetch_related("comments")` แล้วใช้ `len(post.comments.all())` | +1 (คงที่) | **ทุก field ของทุกคอมเมนต์** | เมื่อต้อง**แสดง**คอมเมนต์ด้วย ไม่ใช่แค่นับ |
| `annotate(comment_count=Count("comments"))` | +0 (รวมใน query เดิม) | ไม่มี (นับที่ DB เหมือนเดิม แต่ทำครั้งเดียวทุกแถว) | ✅ เมื่อต้องการแค่ตัวเลข |

กฎง่าย ๆ: **ถ้าต้องการแค่ "ตัวเลขสรุป" ให้ annotate() ถ้าต้องการ "ตัว object จริง" ให้
prefetch_related()** — ใช้ทั้งสองแบบพร้อมกันได้ในกรณีที่ต้องการทั้งสองอย่าง เช่น
Homepage ที่แสดงทั้ง "แท็กจริง ๆ" (ต้อง prefetch) และ "จำนวนคอมเมนต์" (annotate พอ):

```python
from django.db.models import Count
from blog.models import Post

posts = (
    Post.objects.filter(is_published=True)
    .select_related("category", "author")
    .prefetch_related("tags")
    .annotate(comment_count=Count("comments", distinct=True))
    .order_by("-created_at")[:10]
)
```

### 666.4 อย่าลืม `distinct=True` เมื่อผสม prefetch_related() (M2M) กับ annotate() (Count ผ่าน relation อื่น) ในคำสั่งเดียว

ทบทวนจาก Part 014 ขั้นตอนที่ 132.5: เมื่อ query มี JOIN มากกว่าหนึ่งเส้นทางพร้อมกัน
(ในที่นี้คือ `tags` ที่ query แยกจาก `prefetch_related` ก็จริง แต่ `annotate(Count
("comments"))` ยังคง JOIN กับ `blog_comment` ตรง ๆ ในคำสั่งหลัก) ตัวเลขอาจพองผิดปกติ
ถ้ามี JOIN อื่นแฝงอยู่ในคำสั่งเดียวกัน (เช่น ถ้า `filter()` มีเงื่อนไขที่ join ตารางอื่น
เพิ่มด้วย) การใส่ `distinct=True` ใน `Count()` เสมอเป็นนิสัยที่ปลอดภัยที่สุด:

```python
posts = Post.objects.annotate(comment_count=Count("comments", distinct=True))
```

### 666.5 ตารางสรุปการเลือกใช้เครื่องมือลด query

| ต้องการ | เครื่องมือที่เหมาะสม |
|---|---|
| ค่า field เดี่ยวจากตารางที่เชื่อมด้วย FK/O2O (เช่น `author.username`) | `select_related()` |
| List ของ object จากความสัมพันธ์แบบหลายค่า (เช่น `post.tags.all()`) | `prefetch_related()` |
| List ที่กรอง/เรียงลำดับ/จำกัดจำนวนแบบกำหนดเอง | `Prefetch()` object |
| ตัวเลขสรุป (count, sum, avg) ต่อแถว | `annotate()` |
| ลดคอลัมน์ที่ดึงมาจากตารางที่ join | `only()`/`defer()` |

---

## ขั้นตอนที่ 667: ใช้ django-debug-toolbar ยืนยันผลลัพธ์การลด query count จริง (before/after)

### 667.1 ทบทวนการติดตั้ง (จาก Part 066)

Part 066 ได้ติดตั้ง **django-debug-toolbar** ไว้แล้วสำหรับงาน profiling โดยทั่วไป
ทบทวนการตั้งค่าอย่างย่อ (ข้ามได้ถ้าโปรเจกต์ของคุณติดตั้งไว้แล้ว):

```bash
pip install django-debug-toolbar
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ...
    "debug_toolbar",
]

MIDDLEWARE = [
    "debug_toolbar.middleware.DebugToolbarMiddleware",
    # ... middleware อื่น ๆ (ควรอยู่บนสุด ๆ ก่อน middleware ที่ compress response)
]

INTERNAL_IPS = [
    "127.0.0.1",
]
```

```python
# config/urls.py
from django.conf import settings

urlpatterns = [
    # ... urlpatterns เดิม ...
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path("__debug__/", include(debug_toolbar.urls)),
    ] + urlpatterns
```

### 667.2 อ่านค่า SQL Panel

เปิดหน้า Homepage ผ่านเบราว์เซอร์ (`http://127.0.0.1:8000/`) จะเห็นแถบเครื่องมือ
Debug Toolbar ทางด้านขวา คลิกที่แถบ **"SQL"** จะเห็น:

- **จำนวน query ทั้งหมด** ที่ใช้ในการ render หน้านี้ (ตัวเลขใหญ่เด่นชัด)
- **เวลารวม** ที่ใช้ไปกับฐานข้อมูลทั้งหมด (หน่วย ms)
- รายการ SQL statement แต่ละอันพร้อม **stack trace** บอกว่าเรียกมาจากบรรทัดไหนของ
  view/template
- ไอคอนเตือน **"Similar" / "Duplicated"** เมื่อ Django Debug Toolbar ตรวจพบว่ามี
  query ที่ **โครงสร้างเดียวกัน** ถูกรันซ้ำหลายครั้ง (สัญญาณชัดเจนที่สุดของ N+1)

### 667.3 เปรียบเทียบก่อน/หลัง optimize บน view เดียวกัน

**ก่อน optimize** (view จากขั้นตอนที่ 661.3):

```
SQL Panel
─────────────────────────────────────
Queries: 41
Time: 187 ms
⚠️ 30 similar queries (author lookups)
⚠️ 10 duplicated queries (comment count lookups)
```

**หลัง optimize** (view จากขั้นตอนที่ 666.3):

```python
# blog/views.py
from django.db.models import Count
from django.shortcuts import render

from .models import Post


def homepage(request):
    posts = (
        Post.objects.filter(is_published=True)
        .select_related("category", "author")
        .prefetch_related("tags")
        .annotate(comment_count=Count("comments", distinct=True))
        .only(
            "id", "title", "slug", "created_at", "view_count",
            "category__id", "category__name",
            "author__id", "author__username",
        )
        .order_by("-created_at")[:10]
    )
    return render(request, "blog/homepage.html", {"posts": posts})
```

```
SQL Panel
─────────────────────────────────────
Queries: 3
Time: 9 ms
✅ ไม่มี similar/duplicated queries
```

จาก **41 query / 187 ms** เหลือ **3 query / 9 ms** — ลดลงประมาณ **93% ของจำนวน
query** และเร็วขึ้นกว่า **20 เท่า** โดยที่ผลลัพธ์บนหน้าเว็บ **เหมือนเดิมทุกประการ**
ผู้ใช้ไม่เห็นความแตกต่างใด ๆ เลยนอกจากความเร็ว

### 667.4 ทำไมต้องเหลือ 3 query ไม่ใช่ 1 query

หลายคนสงสัยว่าทำไมไม่เหลือแค่ 1 query ไปเลย — คำตอบคือ `tags` เป็น `ManyToMany`
จึง**ต้อง**ใช้ query แยกเสมอ (ตามกลไกของ `prefetch_related()` ในขั้นตอนที่ 663)
ในขณะที่ `category`, `author`, และ `comment_count` ถูกรวมเข้า query หลักผ่าน
`select_related()` และ `annotate()` แล้ว ดังนั้น:

```
Query 1: SELECT posts JOIN category JOIN author, COUNT(comments) GROUP BY post.id
Query 2: SELECT tags WHERE post_id IN (...)
Query 3: (เกิดจาก Django Debug Toolbar เอง ไม่ใช่จาก view — เช่น session/auth query)
```

**3 query คงที่** ไม่ว่าจะมี 10 หรือ 10,000 โพสต์ก็ตาม (ยกเว้น query ที่ 3 ซึ่งเป็น
overhead ของ framework ที่ไม่เกี่ยวกับ N — เช่น session middleware) นี่คือเป้าหมาย
ที่แท้จริงของการ optimize: **เปลี่ยนจากการเติบโตแบบเชิงเส้น (O(N)) ให้กลายเป็น
ค่าคงที่ (O(1))**

### 667.5 ใช้ SQL Panel ตรวจ query ที่ซ้ำซ้อนโดยไม่รู้ตัว

ฟีเจอร์ "Duplicated queries" ของ Debug Toolbar มีประโยชน์มากกว่าการนับ N+1 อย่างเดียว
— มันยังจับกรณีที่โค้ดเรียก QuerySet เดิมซ้ำโดยไม่จำเป็น เช่น เรียก
`Post.objects.filter(is_published=True).count()` สองครั้งในสอง context processor
คนละที่โดยไม่รู้ตัว ทั้งที่ query ให้ผลเหมือนกันทุกประการ — นี่คือปัญหาคนละแบบกับ N+1
แต่ตรวจจับได้ด้วยเครื่องมือเดียวกัน

---

## ขั้นตอนที่ 668: Anti-pattern ที่พบบ่อยในการ optimize query

### 668.1 Over-prefetching: ดึงมาเผื่อทั้งที่ไม่ได้ใช้

```python
# ผิด: prefetch ทุกอย่างที่นึกออก "เผื่อไว้" ทั้งที่ template ใช้แค่ tags
posts = Post.objects.prefetch_related(
    "tags", "comments", "comments__post", "category__posts",
)
```

ทุก relation ที่ prefetch คือ **query เพิ่ม 1 ครั้งเสมอ** ไม่ว่าจะถูกใช้จริงหรือไม่
— การ prefetch แบบ "เผื่อไว้ก่อน" ทำให้เสีย query และหน่วยความจำไปฟรี ๆ โดยไม่ได้
ประโยชน์อะไร ยิ่ง prefetch หลายชั้นซ้อนกัน (เช่น `comments__post` ที่วนกลับไปยัง
`Post` อีกครั้ง) ยิ่งทำให้ query ซับซ้อนและดึงข้อมูลซ้ำซ้อนมากเกินจำเป็น

> **กฎ**: prefetch/select_related เฉพาะ relation ที่ **template หรือ serializer
> เข้าถึงจริง** เท่านั้น ตรวจสอบด้วย Debug Toolbar (ขั้นตอนที่ 667) ทุกครั้งหลังแก้ไข
> ว่าจำนวน query ที่ได้ตรงกับที่คาดไว้ ไม่มากไม่น้อยกว่านั้น

### 668.2 Filter หลัง Prefetch: การเรียก `.filter()` ต่อบน related manager ทำลาย cache

นี่คือ anti-pattern ที่พบบ่อยและเข้าใจผิดง่ายที่สุด:

```python
posts = Post.objects.prefetch_related("comments")

for post in posts:
    # ผิด! การเรียก .filter() ต่อบน related manager จะยิง query ใหม่เสมอ
    # ไม่สนใจว่า .prefetch_related("comments") ทำ cache ไว้แล้วหรือไม่
    approved = post.comments.filter(is_approved=True)
```

```python
from django.db import connection, reset_queries

reset_queries()
posts = list(Post.objects.prefetch_related("comments")[:10])
for post in posts:
    _ = list(post.comments.filter(is_approved=True))  # query ใหม่ทุกครั้ง!

print(len(connection.queries))
# 11   <- 1 (posts) + 1 (prefetch comments ทั้งหมด ที่ไม่ได้ใช้ผลเลย) + 10 (filter แยกทุกโพสต์)
```

**เหตุผล**: เมธอด `.filter()`, `.exclude()`, `.order_by()` ที่เรียกต่อบน related
manager (`post.comments`) จะสร้าง QuerySet ใหม่เสมอ ซึ่งไม่รู้จักและไม่ใช้ cache ที่
`prefetch_related()` เตรียมไว้เลย ผลคือ **prefetch ที่ทำไปก่อนหน้ากลายเป็นการเสีย
query ฟรี ๆ ซ้ำซ้อนกับ query ที่เกิดจาก `.filter()` อีกที**

**วิธีแก้ที่ถูกต้อง — มีสองทาง:**

```python
# ทางที่ 1: กรองด้วย Prefetch() object ตั้งแต่ต้น (แนะนำที่สุด — ดูขั้นตอนที่ 664)
from django.db.models import Prefetch
from blog.models import Comment, Post

posts = Post.objects.prefetch_related(
    Prefetch("comments", queryset=Comment.objects.filter(is_approved=True))
)
for post in posts:
    approved = post.comments.all()   # ใช้ cache ได้จริง ไม่ query เพิ่ม

# ทางที่ 2: กรองด้วย Python list comprehension จาก cache ที่มีอยู่แล้ว
posts = Post.objects.prefetch_related("comments")
for post in posts:
    approved = [c for c in post.comments.all() if c.is_approved]  # ใช้ cache ไม่ query เพิ่ม
```

| วิธี | ใช้ cache จาก prefetch หรือไม่ | เหมาะกับ |
|---|---|---|
| `post.comments.filter(...)` | ❌ ไม่ใช้ ยิง query ใหม่เสมอ | ห้ามใช้เมื่อ prefetch ไว้แล้ว |
| `Prefetch(..., queryset=...filter(...))` | ✅ ใช้ | เมื่อรู้เงื่อนไข filter ล่วงหน้าตั้งแต่ต้น |
| Python list comprehension บน `.all()` | ✅ ใช้ | เมื่อ logic การกรองซับซ้อนหรือเปลี่ยนตามรอบ request |

### 668.3 เข้าถึง Related Object ที่เป็น `None` โดยไม่ระวัง (Nullable FK)

```python
posts = Post.objects.select_related("category")

for post in posts:
    print(post.category.name)  # AttributeError ถ้า category เป็น None!
```

เพราะ `Post.category` ประกาศด้วย `null=True` โพสต์บางอันอาจไม่มีหมวดหมู่จริง ๆ
(`category_id IS NULL`) — `select_related()` ไม่ได้แก้ปัญหานี้ ทำให้ `post.category`
เป็น `None` ตามปกติ และการเข้าถึง `.name` ต่อจาก `None` จะ raise `AttributeError`
วิธีแก้คือเช็คก่อนเสมอ (ทั้งใน view/template):

```python
print(post.category.name if post.category else "ไม่มีหมวดหมู่")
```

```html
{{ post.category.name|default:"ไม่มีหมวดหมู่" }}
```

### 668.4 Slice ต่อโพสต์บน Related Manager ยังคงเป็น N+1

```python
# ผิด! ดูเหมือนจะจำกัดจำนวน แต่ยังคง query แยกทุกโพสต์อยู่ดี
posts = Post.objects.prefetch_related("comments")
for post in posts:
    latest_three = post.comments.all()[:3]   # ยัง query ใหม่ทุกครั้ง เหมือนข้อ 668.2!
```

การ slice (`[:3]`) บน related manager ก็นับเป็นการสร้าง QuerySet ใหม่เช่นเดียวกับ
`.filter()` — ยังคงยิง query แยกต่อโพสต์ ไม่ต่างจากไม่ prefetch เลย วิธีแก้คือใช้
`Prefetch()` พร้อม `to_attr` แล้ว slice ผลลัพธ์ที่เป็น Python list ตามที่แสดงในขั้นตอน
ที่ 664.2

### 668.5 พยายาม select_related() บนความสัมพันธ์แบบหลายค่า

```python
Post.objects.select_related("tags")
```

```
django.core.exceptions.FieldError: Invalid field name(s) given in select_related: 'tags'.
Choices are: category, author
```

Error นี้เกิดขึ้นทันทีตอนสร้าง QuerySet (ไม่ต้อง evaluate ก่อน) เพราะ Django ตรวจสอบ
field ได้จาก schema ล่วงหน้า — ถือเป็นข้อดีที่ error ประเภทนี้จับได้เร็วตั้งแต่ตอน
เขียนโค้ด ไม่ต้องรอไปเจอตอน production

### 668.6 ลืม `.distinct()` เมื่อ `filter()` ผ่านความสัมพันธ์ M2M ทำให้แถวซ้ำ

```python
# ผิด: อาจได้โพสต์ซ้ำถ้าโพสต์นั้นมีหลายแท็กที่ตรงเงื่อนไข
posts = Post.objects.filter(tags__name__in=["python", "django"])
print(posts.count())  # อาจมากกว่าจำนวนโพสต์จริงที่ตรงเงื่อนไข เพราะนับซ้ำ
```

ทบทวนจาก Part 014 ขั้นตอนที่ 134.6: เมื่อ `filter()` เดินทางผ่าน M2M ที่ทำให้เกิด
JOIN แบบ "หนึ่งกลายเป็นหลาย" ต้องปิดท้ายด้วย `.distinct()` เสมอ ไม่เช่นนั้นแม้แต่การ
นับจำนวน (`.count()`) ก็จะผิดพลาด ไม่ใช่แค่ปัญหาเรื่อง query count แต่เป็นปัญหาเรื่อง
**ความถูกต้องของข้อมูล**:

```python
posts = Post.objects.filter(tags__name__in=["python", "django"]).distinct()
```

### 668.7 ตารางสรุป Anti-pattern ทั้งหมด

| Anti-pattern | อาการ | วิธีแก้ |
|---|---|---|
| Prefetch เผื่อไว้ทั้งที่ไม่ใช้ | Query/หน่วยความจำเสียฟรี | prefetch เฉพาะที่ใช้จริง ตรวจด้วย Debug Toolbar |
| `.filter()`/`.exclude()`/`.order_by()` ต่อบน related manager ที่ prefetch ไว้ | N+1 กลับมาโดยไม่รู้ตัว | ใช้ `Prefetch(queryset=...)` หรือ Python list comprehension |
| เข้าถึง nullable FK โดยไม่เช็ค `None` | `AttributeError` | เช็ค `if post.category else ...` หรือใช้ `\|default:` ใน template |
| Slice (`[:n]`) บน related manager | ยังเป็น N+1 เหมือนเดิม | ใช้ `Prefetch(to_attr=...)` แล้ว slice list ใน Python |
| `select_related()` บนความสัมพันธ์หลายค่า | `FieldError` ทันที | ใช้ `prefetch_related()` แทน |
| ลืม `.distinct()` เมื่อ filter ผ่าน M2M | ข้อมูลนับซ้ำ/แถวซ้ำ | เติม `.distinct()` ท้าย QuerySet |

---

## ขั้นตอนที่ 669: `assertNumQueries()` ใน test เพื่อป้องกัน N+1 กลับมาใหม่ในอนาคต

### 669.1 ทำไมแค่ optimize ให้ผ่านครั้งเดียวไม่พอ

สมมติวันนี้คุณ optimize หน้า Homepage เหลือ 3 query สำเร็จแล้ว แต่อีก 3 เดือนข้างหน้า
เพื่อนร่วมทีมมาเพิ่มบรรทัด `{{ post.author.profile.bio }}` เข้าไปใน template โดยไม่รู้
ว่า `select_related()` ใน view ไม่ได้ join ไปถึง `profile` — N+1 จะกลับมาทันทีโดยไม่มี
ใครสังเกตเห็น จนกว่าจะมีคนไปเจอในหน้า production ที่ช้าลง นี่คือเหตุผลที่ **การล็อก
จำนวน query ไว้ใน automated test** สำคัญพอ ๆ กับการ optimize ครั้งแรก — เชื่อมโยงกับ
วินัยการเขียน test ที่เรียนมาตลอด Phase 7

### 669.2 `self.assertNumQueries(n)` พื้นฐาน

Django มี assertion พิเศษสำหรับเรื่องนี้โดยเฉพาะ ใช้เป็น context manager ภายใน
`TestCase`:

```python
# blog/tests/test_query_optimization.py
from django.contrib.auth import get_user_model
from django.test import TestCase

from blog.models import Category, Comment, Post, Tag

User = get_user_model()


class HomepageQueryCountTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        category = Category.objects.create(name="Django", slug="django")
        tag = Tag.objects.create(name="orm", slug="orm")
        author = User.objects.create_user(username="sireeporn", password="pass12345")

        for i in range(15):
            post = Post.objects.create(
                title=f"บทความที่ {i}",
                slug=f"post-{i}",
                content="เนื้อหาบทความ",
                is_published=True,
                category=category,
                author=author,
            )
            post.tags.add(tag)
            Comment.objects.create(
                post=post, author="ผู้อ่าน", text="ความเห็น", is_approved=True,
            )

    def test_homepage_uses_fixed_number_of_queries(self):
        # เลข 3 มาจากการวัดจริงด้วย Debug Toolbar ในขั้นตอนที่ 667
        # (1 สำหรับ posts+category+author+comment_count, 1 สำหรับ tags, 1 overhead ของ session)
        with self.assertNumQueries(3):
            response = self.client.get("/")
            # ต้อง evaluate QuerySet ให้ครบภายใน context นี้ (render ทำให้ template loop แล้ว)
            self.assertEqual(response.status_code, 200)
```

ถ้าจำนวน query จริงไม่ตรงกับ `3` ที่ระบุไว้ test จะ **fail ทันที** พร้อมข้อความบอก
จำนวนที่แท้จริง:

```
AssertionError: 12 != 3 : 12 queries executed, 3 expected
Captured queries were:
1. SELECT ... FROM "blog_post" ...
2. SELECT ... FROM "auth_user" WHERE "auth_user"."id" = 1 ...
3. SELECT ... FROM "auth_user" WHERE "auth_user"."id" = 1 ...
... (แสดง SQL ของทุก query ที่รันจริงในรอบนั้น)
```

### 669.3 Debug ด้วย `CaptureQueriesContext` เมื่อ assertNumQueries fail

เมื่อ test fail ข้อความ error ของ `assertNumQueries` จะแสดง SQL ทุกคำสั่งมาให้อยู่แล้ว
(ตามตัวอย่างข้างบน) แต่ถ้าต้องการตรวจสอบแบบละเอียดกว่านั้นนอก assertion (เช่น ขณะ
พัฒนาและยังไม่รู้ว่าตัวเลขที่ถูกต้องคือเท่าไร) ใช้ `CaptureQueriesContext` ตรง ๆ ได้:

```python
from django.db import connection
from django.test.utils import CaptureQueriesContext
from django.test import TestCase


class HomepageQueryInspectionTests(TestCase):
    def test_inspect_homepage_queries(self):
        with CaptureQueriesContext(connection) as ctx:
            self.client.get("/")

        print(f"จำนวน query: {len(ctx.captured_queries)}")
        for i, query in enumerate(ctx.captured_queries, start=1):
            print(f"{i}. [{query['time']}s] {query['sql']}")
```

`CaptureQueriesContext` ให้ข้อมูลเหมือนกับ `connection.queries` (ขั้นตอนที่ 661.4)
แต่ปลอดภัยกว่าเพราะ **ไม่ต้อง** ตั้ง `settings.DEBUG = True` เอง — มันจัดการเปิด/ปิด
การ log ให้อัตโนมัติเฉพาะช่วงที่อยู่ใน `with` block เท่านั้น เหมาะสำหรับใช้ใน test
โดยเฉพาะ

### 669.4 เขียน Test ครอบคลุมทั้งกรณี "ไม่มีข้อมูล" และ "มีข้อมูลเยอะ"

ข้อสำคัญมากคือต้องทดสอบด้วย **จำนวนแถวมากกว่า 1** เสมอ (เช่น 15 โพสต์ในตัวอย่าง
ข้างต้น) เพราะถ้าทดสอบด้วยแค่ 1 โพสต์ N+1 ที่มีค่า N=1 จะให้จำนวน query เท่ากับ
เวอร์ชันที่ optimize ดีแล้วพอดี (บังเอิญผ่าน test ทั้งที่มีบั๊กซ่อนอยู่):

```python
class HomepageQueryScalingTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        category = Category.objects.create(name="Django", slug="django")
        author = User.objects.create_user(username="writer", password="pass12345")
        cls.posts = [
            Post.objects.create(
                title=f"Post {i}", slug=f"p{i}", content="...",
                is_published=True, category=category, author=author,
            )
            for i in range(20)
        ]

    def test_query_count_does_not_scale_with_post_count(self):
        # ทดสอบด้วย 20 โพสต์ ถ้าจำนวน query ยังคงที่ (ไม่ใช่ 1 + 20*k) แปลว่า optimize ถูกต้อง
        with self.assertNumQueries(3):
            self.client.get("/")
```

การตั้งค่าทดสอบด้วยจำนวนแถวที่มากพอ (เช่น 15-20 แถวขึ้นไป) ทำให้ถ้า N+1 กลับมาจริง
จำนวน query ที่ผิดจะสูงขึ้นอย่างเห็นได้ชัด (เช่น 61 แทนที่จะเป็น 3) ทำให้ test fail
ชัดเจนและ debug ง่ายกว่าตัวเลขที่ใกล้เคียงกันมาก

### 669.5 อัปเดตค่า assertNumQueries เมื่อ feature เปลี่ยนแปลงจริง

เมื่อทีมเพิ่ม feature ใหม่ที่ต้องใช้ query เพิ่มขึ้นจริง ๆ (เช่น เพิ่มการแสดง avatar
ของผู้เขียนที่ต้อง join ไปถึง `Profile`) ตัวเลขใน `assertNumQueries()` ต้องถูก
**ปรับปรุงอย่างมีสติ** พร้อมคอมเมนต์อธิบายเหตุผล ไม่ใช่แก้ตัวเลขให้ผ่านแบบไม่ตรวจสอบ:

```python
def test_homepage_uses_fixed_number_of_queries(self):
    # อัปเดตจาก 3 เป็น 4 เมื่อเพิ่ม select_related("author__profile")
    # เพื่อแสดง avatar ของผู้เขียนในหน้า Homepage (PR #142)
    with self.assertNumQueries(4):
        self.client.get("/")
```

นี่คือจุดประสงค์ที่แท้จริงของ `assertNumQueries()`: **ไม่ใช่การห้ามเพิ่ม query
โดยเด็ดขาด** แต่คือการบังคับให้ทุกครั้งที่จำนวน query เปลี่ยนไป ต้องมีคนตัดสินใจ
อย่างมีสติว่ามันคุ้มค่าหรือเป็นบั๊ก ไม่ใช่การเปลี่ยนแปลงที่หลุดรอดไปโดยไม่มีใครรู้ตัว

### 669.6 ตารางสรุปเครื่องมือทดสอบ Query Count

| เครื่องมือ | ใช้เมื่อ |
|---|---|
| `self.assertNumQueries(n)` | ล็อกจำนวน query ของ endpoint/ฟังก์ชันไว้ใน test แบบถาวร |
| `django.test.utils.CaptureQueriesContext` | ตรวจสอบ SQL แบบละเอียดขณะพัฒนา หรือ debug test ที่ fail |
| `django.db.connection.queries` + `reset_queries()` | สำรวจ query ใน shell แบบ interactive (ขั้นตอนที่ 661) |
| django-debug-toolbar | ตรวจสอบด้วยตาผ่านเบราว์เซอร์ พร้อม stack trace และ duplicate detection |

---

## ขั้นตอนที่ 670: สรุปและแบบฝึกหัด

### 670.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจกลไกของ N+1 Query Problem อย่างละเอียด และวิธีวัดจำนวน query จริงด้วย
  `django.db.connection.queries`
- ✅ เข้าใจว่า `select_related()` ทำงานด้วย SQL `JOIN` เดียว ใช้ได้กับ `ForeignKey`
  และ `OneToOneField` เท่านั้น
- ✅ เข้าใจว่า `prefetch_related()` ทำงานด้วย query แยก + จับคู่ในหน่วยความจำ Python
  ใช้ได้กับทุกความสัมพันธ์ โดยเฉพาะ `ManyToManyField` และ reverse `ForeignKey`
- ✅ ใช้ `Prefetch()` object ควบคุม QuerySet ที่ prefetch ได้ละเอียด (filter, order,
  to_attr, nested select_related)
- ✅ ผสาน `only()`/`defer()` กับ `select_related()` เพื่อลดคอลัมน์ที่ดึงมาให้แคบที่สุด
- ✅ ใช้ `annotate()` แทนการ query นับแยกต่อแถว เพื่อลดทั้งจำนวน query และหน่วยความจำ
- ✅ ใช้ django-debug-toolbar ยืนยันผลลัพธ์ before/after ได้จริงด้วยตาตัวเอง
- ✅ รู้จัก anti-pattern สำคัญ 6 แบบที่ทำให้การ optimize ล้มเหลวโดยไม่รู้ตัว
- ✅ เขียน `assertNumQueries()` เพื่อล็อกจำนวน query ไว้ป้องกัน regression ในอนาคต

### 670.2 Checklist ก่อนไป Part ถัดไป

- [ ] เพิ่ม field `author` และ `is_approved` เข้า model `Post`/`Comment` และ migrate สำเร็จ
- [ ] รัน seed data ได้และเห็นข้อมูลตัวอย่าง 50 บทความในระบบ
- [ ] วัด query count ของหน้า Homepage แบบ "ยังไม่ optimize" ได้ตัวเลขจริงด้วย
      `connection.queries`
- [ ] เขียน query ที่ใช้ `select_related()` และเห็น SQL `JOIN` จริงผ่าน `.query`
- [ ] เขียน query ที่ใช้ `prefetch_related()` และเห็น 2 query แยกกันจริง
- [ ] ใช้ `Prefetch()` พร้อม `to_attr` ดึงคอมเมนต์ที่กรองแล้วได้สำเร็จ
- [ ] ติดตั้งและเปิดใช้ django-debug-toolbar เห็นจำนวน query บนเบราว์เซอร์จริง
- [ ] เขียน `assertNumQueries()` อย่างน้อย 1 test และเห็นมัน fail เมื่อจงใจทำให้เกิด N+1

### 670.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน view `homepage` เวอร์ชัน "ยังไม่ optimize" ตามขั้นตอนที่
661.3 ให้ทำงานได้จริงในโปรเจกต์ของคุณ แล้วใช้ `django.db.connection.queries`
วัดจำนวน query จริงเมื่อมีโพสต์ 10, 30 และ 50 รายการ บันทึกตัวเลขทั้งสามกรณีลงใน
`notes.md` พร้อมพิสูจน์ว่าความสัมพันธ์เป็นเส้นตรง (linear) ตามสูตรในขั้นตอนที่ 661.5

**แบบฝึกหัดที่ 2**: Optimize view `homepage` ให้เหลือ query น้อยที่สุดเท่าที่เป็นไปได้
โดยต้อง:
- ใช้ `select_related()` สำหรับ `category` และ `author`
- ใช้ `prefetch_related()` สำหรับ `tags`
- ใช้ `annotate()` แทนการเรียก `post.comments.count()` ใน template
- ใช้ `only()` จำกัดคอลัมน์ที่ดึงจาก `Post` และ `auth_user` ให้แคบที่สุด

จากนั้นวัดจำนวน query อีกครั้งด้วยจำนวนโพสต์เท่าเดิม (10, 30, 50) แล้วพิสูจน์ว่า
จำนวน query **คงที่** ไม่ขึ้นกับจำนวนโพสต์อีกต่อไป

**แบบฝึกหัดที่ 3**: เขียน `assertNumQueries()` test สำหรับ view `homepage` ที่
optimize แล้วจากแบบฝึกหัดที่ 2 โดย:
- สร้างข้อมูลทดสอบอย่างน้อย 15 โพสต์ใน `setUpTestData`
- ยืนยันว่าจำนวน query คงที่ตามที่วัดได้จริง
- จงใจ comment บรรทัด `.select_related("category", "author")` ออก แล้วรัน test
  อีกครั้ง เพื่อยืนยันว่า test **fail** ทันที (พิสูจน์ว่า test นี้ตรวจจับ N+1 ได้จริง)
  แล้วนำ `.select_related()` กลับมาก่อนส่งงาน

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ใช้ `Prefetch()` object พร้อม `to_attr="recent_comments"`
เพิ่มการแสดง **3 คอมเมนต์ล่าสุดที่ `is_approved=True`** ของแต่ละโพสต์ในหน้า Homepage
(นอกเหนือจากตัวเลข `comment_count` ที่ทำไว้แล้ว) โดยต้อง:
- ไม่ทำให้จำนวน query เพิ่มขึ้นเกิน 1 query จากเดิม (รวมแล้วต้องยังคงที่ ไม่ขึ้นกับ N)
- เขียน `assertNumQueries()` test ใหม่ยืนยันตัวเลขที่เพิ่มขึ้นนี้
- ใช้ django-debug-toolbar ยืนยันว่าไม่มี query ซ้ำซ้อน (duplicated queries) เกิดขึ้น

### 670.4 คำถามที่พบบ่อย (FAQ)

**Q: ควรใช้ `select_related()`/`prefetch_related()` กับทุก QuerySet เสมอไปหรือไม่?**
A: ไม่ควร ใช้เฉพาะเมื่อรู้แน่ชัดว่าจะเข้าถึงความสัมพันธ์นั้นจริงในโค้ดถัดไป (เช่น ใน
template หรือ serializer) การเรียก `select_related()`/`prefetch_related()` โดยไม่ใช้
ผลลัพธ์เลยเป็นการเสีย query/หน่วยความจำไปฟรี ๆ ตามที่อธิบายในขั้นตอนที่ 668.1

**Q: `prefetch_related()` ใช้กับ `ForeignKey` ได้ ทำไมยังต้องมี `select_related()` อยู่?**
A: `select_related()` รวมทุกอย่างเป็น 1 query เสมอ ในขณะที่ `prefetch_related()`
กับ `ForeignKey` ยังคงเพิ่ม 1 query ต่อ relation เสมอ (แม้จะไม่ใช่ N+1 แต่ก็ยังมากกว่า
`select_related()`) สำหรับความสัมพันธ์แบบ single-valued `select_related()` จึงดีกว่า
เกือบทุกกรณี ยกเว้นสถานการณ์พิเศษตามขั้นตอนที่ 663.5

**Q: ใช้ `only()` แล้วโค้ดพังเพราะ deferred field ถูกเข้าถึงโดยไม่ตั้งใจ ควรทำอย่างไร?**
A: เปลี่ยนมาใช้ `defer()` กับ field ขนาดใหญ่ที่ **รู้แน่ชัด** ว่าไม่ใช้แทน (เช่น
`content`, `description`) ปลอดภัยกว่า `only()` เพราะ field ใหม่ที่เพิ่มเข้ามาในอนาคต
จะไม่ถูก defer ไปด้วยโดยไม่ตั้งใจ ตามคำแนะนำในขั้นตอนที่ 665.3

**Q: `assertNumQueries()` เปราะบางเกินไปหรือไม่ เพราะพัง (fail) ง่ายมากเมื่อมีการ
เปลี่ยนแปลงเล็กน้อย?**
A: ความเปราะบางนี้คือ **จุดประสงค์** ของมัน ไม่ใช่ข้อเสีย มันบังคับให้ทุกครั้งที่มีการ
เปลี่ยนแปลงจำนวน query ต้องมีคนมาตรวจสอบและตัดสินใจอย่างมีสติว่าคุ้มค่าหรือเป็นบั๊ก
(ตามขั้นตอนที่ 669.5) ดีกว่าปล่อยให้ N+1 หลุดเข้า production โดยไม่มีใครรู้ตัว

**Q: ควร optimize ทุก QuerySet ในโปรเจกต์ตั้งแต่แรกเลยหรือไม่?**
A: ไม่จำเป็น หลักการที่ดีคือเขียนโค้ดให้ถูกต้องก่อน แล้วค่อย optimize เฉพาะจุดที่วัดผล
แล้วพบว่าเป็นปัญหาจริง (เช่นผ่าน django-debug-toolbar หรือ Part 066 Performance
Profiling) — การ optimize ก่อนวัดผล (premature optimization) มักทำให้โค้ดซับซ้อน
ขึ้นโดยไม่ได้ประโยชน์ที่วัดผลได้จริง แต่สำหรับ query ที่รู้แน่ชัดล่วงหน้าว่าจะถูกเรียก
ในลูป (เช่นหน้า list ที่วนแสดงหลายรายการ) ควรใส่ `select_related()`/
`prefetch_related()` ไว้ตั้งแต่ต้นเป็นนิสัยที่ดี

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 068: Django Caching Framework เบื้องต้น** จะพาไปแก้ปัญหา performance ในอีก
มิติหนึ่ง — ครั้งนี้เราลด**จำนวน query** ให้เหลือน้อยที่สุดไปแล้ว แต่ query ที่เหลืออยู่
(แม้จะน้อยและเร็วแล้ว) ก็ยังคงถูกยิงไปยังฐานข้อมูลซ้ำ ๆ ทุกครั้งที่มีคนเข้าดูหน้าเดียวกัน
Part หน้าจะแนะนำ **Django Caching Framework** ตั้งแต่ระดับ view, template fragment,
ไปจนถึง low-level cache API เพื่อเก็บผลลัพธ์ที่คำนวณแล้วไว้ใช้ซ้ำ โดยไม่ต้องถามฐานข้อมูล
ใหม่ทุกครั้ง — เตรียมเปิดโปรเจกต์บล็อกที่ optimize query เสร็จแล้วจาก Part นี้ไว้ให้พร้อม
เพราะเราจะนำ view เดียวกันนี้มา cache กันต่อทันที
