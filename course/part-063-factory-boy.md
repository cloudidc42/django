# Part 063: Factory Boy และ Test Data Generation

> **ขั้นตอนที่ 621-630 ของหลักสูตร** | Phase 7: Testing & Quality Assurance
>
> เป้าหมายของ Part นี้: แก้ปัญหาที่คุณเจอมาตลอด Part 059-062 นั่นคือการเขียน
> `Post.objects.create(...)`, `CustomUser.objects.create_user(...)` ซ้ำ ๆ ในทุกไฟล์
> test จนโค้ดเยิ่นเย้อและบำรุงรักษายาก คุณจะได้เรียนรู้ **Factory Boy** ไลบรารีมาตรฐาน
> อุตสาหกรรมสำหรับสร้างข้อมูลทดสอบของ Python ตั้งแต่ `DjangoModelFactory` พื้นฐาน
> ไปจนถึง `SubFactory` สำหรับความสัมพันธ์, `Sequence` สำหรับข้อมูลไม่ซ้ำกัน, การผสาน
> `Faker` เพื่อความสมจริง, `Trait` สำหรับตัวแปรของ object, `post_generation` hook
> สำหรับความสัมพันธ์แบบ many-to-many และสุดท้ายจะผสาน Factory Boy เข้ากับระบบ
> fixture ของ pytest ที่สร้างไว้ใน Part 061 ผ่าน `pytest-factoryboy` พร้อมเข้าใจ
> ความแตกต่างด้าน performance ระหว่าง `build()` กับ `create()` อย่างลึกซึ้ง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 621: ทำไมควรใช้ Factory Boy แทนการเขียน fixtures.json แบบเก่า
- ขั้นตอนที่ 622: สร้าง `Factory` class พื้นฐานสำหรับ `Post`, `Category`
- ขั้นตอนที่ 623: `SubFactory` สำหรับสร้างความสัมพันธ์ (Post ที่มี Category อัตโนมัติ)
- ขั้นตอนที่ 624: `Sequence` สำหรับ field ที่ต้องไม่ซ้ำกัน (เช่น slug, email)
- ขั้นตอนที่ 625: ผสาน `Faker` เพื่อสร้างข้อมูลปลอมที่สมจริง
- ขั้นตอนที่ 626: `Trait` สำหรับสร้าง Factory หลายรูปแบบจากคลาสเดียว
- ขั้นตอนที่ 627: `post_generation` hook — ทำงานหลัง object ถูกสร้างแล้ว
- ขั้นตอนที่ 628: ใช้ Factory ร่วมกับ pytest fixture จาก Part 061
- ขั้นตอนที่ 629: ข้อควรพิจารณาด้าน Performance — `build()` เทียบกับ `create()`
- ขั้นตอนที่ 630: สรุปและแบบฝึกหัด — สร้าง Factory ให้ครบทุก model ของแอป blog

---

## ขั้นตอนที่ 621: ทำไมควรใช้ Factory Boy แทนการเขียน fixtures.json แบบเก่า

### 621.1 `fixtures.json` คืออะไร และทำไม Django ถึงมีมาให้แต่แรก

ก่อน pytest และ Factory Boy จะเป็นที่นิยม Django มีกลไกสร้างข้อมูลทดสอบในตัวชื่อ
**fixtures** ซึ่งเป็นไฟล์ JSON (หรือ YAML/XML) ที่ dump ข้อมูลออกมาจากฐานข้อมูลจริง
ด้วยคำสั่ง `dumpdata` แล้วโหลดกลับด้วย `loaddata`:

```bash
python manage.py dumpdata blog.Category blog.Post --indent 2 > blog/fixtures/sample_posts.json
```

ผลลัพธ์ที่ได้หน้าตาประมาณนี้:

```json
[
  {
    "model": "blog.category",
    "pk": 1,
    "fields": {
      "name": "เทคโนโลยี",
      "slug": "technology",
      "description": ""
    }
  },
  {
    "model": "blog.post",
    "pk": 1,
    "fields": {
      "title": "แนะนำ Django 5",
      "slug": "intro-django-5",
      "content": "เนื้อหาบทความ...",
      "is_published": true,
      "category": 1,
      "tags": [1, 2],
      "author": 1,
      "created_at": "2025-01-10T09:00:00Z",
      "updated_at": "2025-01-10T09:00:00Z"
    }
  }
]
```

แล้วใช้ใน test แบบนี้:

```python
from django.test import TestCase


class PostListViewTest(TestCase):
    fixtures = ["sample_posts.json"]

    def test_list_view_shows_posts(self):
        response = self.client.get("/blog/")
        self.assertContains(response, "แนะนำ Django 5")
```

### 621.2 ปัญหาที่สะสมมากขึ้นเรื่อย ๆ เมื่อโปรเจกต์โต

วิธีนี้ **ไม่ได้ผิด** และยังใช้ในหลายโปรเจกต์เก่า แต่มีปัญหาเชิงโครงสร้างหลายข้อที่
ทีมมืออาชีพเจอซ้ำแล้วซ้ำเล่า:

1. **Hardcoded primary key**: fixture อ้าง `"category": 1` ตายตัว ถ้ามี migration
   หรือ test อื่นสร้างข้อมูลก่อนแล้ว pk ไม่ตรงกับที่ fixture คาดหวัง ข้อมูลจะเพี้ยน
   หรือ integrity error ทันที
2. **Schema drift แบบเงียบ ๆ**: เมื่อเพิ่ม field ใหม่ที่ `required=True` ลงใน model
   Django **ไม่เตือนคุณ** ว่าไฟล์ JSON เก่าขาด field นั้น จนกว่าจะรัน test แล้ว
   เจอ error ตอน `loaddata` (หรือแย่กว่านั้นคือโหลดผ่านได้เฉย ๆ ด้วยค่า default
   ที่ไม่ตรงกับที่ต้องการทดสอบจริง)
3. **อ่านไม่รู้เรื่องว่าทดสอบอะไร**: เปิดไฟล์ test แล้วเห็นแค่ `fixtures = [...]`
   ต้องเปิดไฟล์ JSON แยกไปดูว่าข้อมูลจริง ๆ มีอะไรบ้าง ต่างจากการเห็น
   `Post.objects.create(title="...", is_published=True)` ที่อ่านแล้วเข้าใจทันที
4. **ไม่ยืดหยุ่นสำหรับแต่ละ test case**: ถ้า test หนึ่งต้องการ Post ที่ published
   และอีก test ต้องการ draft คุณต้องมี fixture file แยกกัน หรือยัดข้อมูลทั้งสอง
   สถานะไว้ในไฟล์เดียวแล้วกรองเอาเองใน test — ทั้งสองทางล้วนเพิ่มความซับซ้อน
5. **Merge conflict ใน JSON**: เมื่อหลายคนแก้ fixture file พร้อมกัน git diff ของ
   JSON อ่านยากกว่า diff ของโค้ด Python มาก
6. **ข้อมูลไม่สมจริง**: ส่วนใหญ่ทีมจะใช้ค่าเดิมซ้ำ ๆ เช่น "Test Post 1", "Test Post 2"
   ทำให้ไม่เจอบั๊กที่เกิดจากข้อมูลขอบเขต (edge case) เช่น ชื่อยาวมาก อีเมลรูปแบบแปลก ๆ

### 621.3 Factory Boy คืออะไร

**Factory Boy** คือไลบรารี Python สำหรับสร้างข้อมูลทดสอบแบบเป็นโปรแกรม (programmatic
test data generation) แรงบันดาลใจมาจาก `factory_girl` ของ Ruby on Rails ที่โด่งดังใน
ชุมชน Rails มาก่อน แนวคิดหลักคือแทนที่จะ "dump ข้อมูลจริงมาเก็บเป็นไฟล์" คุณเขียน
**สูตร (recipe)** สำหรับสร้าง object ขึ้นมาด้วยโค้ด Python ธรรมดา แล้วเรียกสูตรนั้น
ซ้ำได้ไม่จำกัดครั้งในทุก test ที่ต้องการ

```python
# ตัวอย่างสั้น ๆ ให้เห็นภาพก่อน (รายละเอียดเต็มอยู่ในขั้นตอนที่ 622)
import factory

from blog.models import Category


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = "เทคโนโลยี"


# ใช้งานใน test ได้ทันที ไม่ต้องมีไฟล์ JSON แยก
category = CategoryFactory()          # สร้างและบันทึกลงฐานข้อมูลจริง
another = CategoryFactory(name="กีฬา")  # override ค่าตอนเรียกใช้ได้อิสระ
```

### 621.4 ตารางเปรียบเทียบ `fixtures.json` กับ Factory Boy แบบละเอียด

| หัวข้อ | `fixtures.json` (Django built-in) | Factory Boy |
|---|---|---|
| รูปแบบไฟล์ | JSON/YAML/XML แยกจากโค้ด | Python class ธรรมดา อยู่ใน codebase |
| Primary key | ต้องระบุตายตัว เสี่ยงชนกัน | Django จัดการให้อัตโนมัติ ไม่ต้องยุ่ง |
| ปรับแต่งต่อ test case | ทำยาก ต้องมีหลายไฟล์หรือกรองเอง | `PostFactory(is_published=False)` บรรทัดเดียว |
| ความสัมพันธ์ (FK/M2M) | ผูก pk ตายตัว เปราะบางมาก | `SubFactory`/`post_generation` จัดการอัตโนมัติ |
| ข้อมูลสมจริง | ต้องพิมพ์เองทุกค่า | ผสาน `Faker` ได้ในตัว |
| ตรวจจับ schema เปลี่ยน | ไม่เตือนจนกว่าจะรันแล้วพัง | เห็น error ทันทีตอนเขียนโค้ด (import/type check) |
| Diff ใน code review | อ่านยาก (JSON syntax) | อ่านง่ายเหมือน diff โค้ด Python ทั่วไป |
| สร้างข้อมูลจำนวนมาก | ต้องเขียน JSON ยาวเอง | `PostFactory.create_batch(100)` บรรทัดเดียว |
| Reuse ข้าม test | ต้อง import fixture file เดียวกัน | import factory class ปกติ พร้อม type hint/autocomplete |
| ความนิยมในโปรเจกต์ใหม่ (2025-2026) | ลดลงเรื่อย ๆ เหลือใช้กับ "ข้อมูลเริ่มต้นระบบจริง" เช่น รายชื่อประเทศ | เป็นมาตรฐานสำหรับ test data ในทีม pytest แทบทุกทีม |

### 621.5 แล้วควรทิ้ง `fixtures.json` ไปเลยหรือไม่

ไม่จำเป็นเสมอไป — `loaddata`/`dumpdata` ยังเหมาะกับ **ข้อมูลอ้างอิงคงที่ของระบบจริง**
(reference data) ที่ไม่เปลี่ยนบ่อยและต้องมีเหมือนกันทุก environment เช่น รายชื่อ
จังหวัด, ประเภทสกุลเงิน, หรือ initial data ที่ต้อง seed หลัง deploy (จะกลับมาพูดถึง
เรื่องนี้อีกครั้งในบริบทของ data migration ที่ Phase DevOps) แต่สำหรับ **ข้อมูลที่ใช้
เฉพาะใน test** Factory Boy คือคำตอบที่ทีมมืออาชีพเลือกใช้เกือบทั้งหมดในปัจจุบัน

### 621.6 ติดตั้งไลบรารีที่จำเป็น

```bash
source venv/bin/activate

pip install "factory-boy~=3.3"
```

`factory-boy` ติดตั้งมาพร้อม **`Faker`** (ไลบรารีสร้างข้อมูลปลอมสมจริง) เป็น
dependency โดยอัตโนมัติอยู่แล้ว ไม่ต้องติดตั้งแยก — เราจะเจาะลึกการใช้ `Faker` ใน
ขั้นตอนที่ 625 บันทึกลง `requirements-dev.txt` เช่นเดียวกับ `pytest`/`pytest-django`
ที่ทำไว้ใน Part 061 เพราะเป็นเครื่องมือของฝั่ง test เท่านั้น:

```bash
# requirements-dev.txt
-r requirements.txt

pytest~=8.3
pytest-django~=4.9
factory-boy~=3.3
```

```bash
pip install -r requirements-dev.txt
python -c "import factory; print(factory.__version__)"
```

### 621.7 ภาพรวมของสิ่งที่จะสร้างตลอด Part นี้

Part นี้จะสร้าง factory ให้ครบทุก model ของแอป `blog` ที่คุณสร้างไว้ตั้งแต่ Part 012
(ความสัมพันธ์), Part 032 (custom user model), และ Part 060-061 (testing):

| Model | อยู่ในแอป | ความสัมพันธ์ที่เกี่ยวข้อง |
|---|---|---|
| `CustomUser` | `accounts` | ไม่มี (เป็นจุดเริ่มต้นของความสัมพันธ์อื่น) |
| `Category` | `blog` | ถูกอ้างโดย `Post` (ForeignKey) |
| `Tag` | `blog` | ถูกอ้างโดย `Post` (ManyToMany) |
| `Post` | `blog` | อ้าง `Category`, `Tag`, `CustomUser` |
| `Comment` | `blog` | อ้าง `Post`, `CustomUser` |
| `Profile` | `blog` | อ้าง `CustomUser` (OneToOne) |

ทุกความสัมพันธ์เหล่านี้คือสิ่งที่ Factory Boy ถูกออกแบบมาให้จัดการโดยเฉพาะ — เมื่อจบ
Part นี้ การสร้าง `Post` ที่มี category, tags, author และ comments ครบถ้วนจะเหลือแค่
บรรทัดเดียว แทนที่จะต้องเขียน 15-20 บรรทัดแบบที่ทำมาตลอด Part 059-062

---

## ขั้นตอนที่ 622: สร้าง `Factory` class พื้นฐานสำหรับ `Post`, `Category`

### 622.1 โมเดลที่จะใช้อ้างอิงตลอด Part นี้

เพื่อให้ตัวอย่างทั้งหมดใน Part นี้สอดคล้องกัน เราจะรวมโมเดลจาก Part 012/032/060 เข้า
เป็นเวอร์ชันเดียวที่ใช้ตลอดบทนี้ (ถ้าโปรเจกต์ของคุณมีรายละเอียดต่างไปเล็กน้อย เช่น
field `content_type` จาก Part 061 ก็ปรับ factory ให้ตรงกับ model จริงของคุณได้ไม่ยาก
หลักการเหมือนกันทุกประการ):

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class CustomUser(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
    phone_number = models.CharField(max_length=20, blank=True)
    is_verified_author = models.BooleanField(default=False)

    def __str__(self):
        return self.username
```

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
        ordering = ["name"]
        verbose_name_plural = "categories"

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
        ordering = ["name"]

    def __str__(self):
        return self.name

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True, allow_unicode=True)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True, related_name="posts",
    )
    tags = models.ManyToManyField(Tag, blank=True, related_name="posts")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="posts",
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title, allow_unicode=True)
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse("blog:detail", kwargs={"slug": self.slug})


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="comments",
    )
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["created_at"]

    def __str__(self):
        return f"ความเห็นของ {self.author} ใน {self.post.title}"


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile",
    )
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)
    website = models.URLField(blank=True)
    bio = models.TextField(max_length=500, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"โปรไฟล์ของ {self.user.username}"
```

### 622.2 โครงสร้างพื้นฐานของ `DjangoModelFactory`

Factory ทุกตัวที่ผูกกับ Django model ต้องสืบทอดจาก `factory.django.DjangoModelFactory`
และประกาศ `class Meta: model = ...` เพื่อบอกว่า factory นี้สร้าง object ของ model ไหน:

```python
import factory


class SomeFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "app_label.ModelName"   # หรือ import class จริงมาใส่ตรง ๆ ก็ได้

    field_name = "ค่าคงที่หรือ callable"
```

โครงสร้างไฟล์ที่แนะนำ (สอดคล้องกับโครงสร้าง `blog/tests/` ที่วางไว้ตั้งแต่ Part 061):

```
blog/
├── models.py
├── tests/
│   ├── __init__.py
│   ├── factories.py       # <-- ไฟล์ใหม่ที่จะสร้างใน Part นี้
│   ├── test_models.py
│   ├── test_views.py
│   └── test_forms.py
```

### 622.3 `CategoryFactory`: factory แรกที่ง่ายที่สุด

```python
# blog/tests/factories.py
import factory

from blog.models import Category


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = "หมวดหมู่ทดสอบ"
    description = "คำอธิบายหมวดหมู่สำหรับการทดสอบ"
```

ทดลองใช้งานผ่าน `python manage.py shell` (หรือใน test):

```python
>>> from blog.tests.factories import CategoryFactory
>>> category = CategoryFactory()
>>> category.pk
1
>>> category.name
'หมวดหมู่ทดสอบ'
>>> category.slug
'หมวดหมู่ทดสอบ'   # save() ของ model สร้าง slug ให้อัตโนมัติเพราะ allow_unicode
```

สังเกตว่า `CategoryFactory()` **เรียก `Category.objects.create(...)` ให้อัตโนมัติ**
ซึ่งหมายความว่า `save()` ของ model (รวมถึง logic การสร้าง slug ที่เขียนไว้ใน
`Category.save()`) ทำงานตามปกติทุกประการ — Factory Boy ไม่ได้ bypass business logic
ของ model แต่อย่างใด

### 622.4 override ค่าตอนเรียกใช้ (ไม่ต้องแก้ factory class)

จุดเด่นที่สุดของ Factory Boy คือการ override field ใด ๆ ก็ได้ตอนเรียกใช้งาน โดยไม่
ต้องแก้ไฟล์ factory เลย:

```python
>>> tech = CategoryFactory(name="เทคโนโลยี")
>>> tech.name
'เทคโนโลยี'

>>> sports = CategoryFactory(name="กีฬา", description="ข่าวกีฬาทุกประเภท")
>>> sports.description
'ข่าวกีฬาทุกประเภท'
```

ค่าที่ไม่ได้ override จะใช้ค่า default ที่ประกาศไว้ใน class เสมอ — นี่คือหลักการ
**"sensible defaults, override เฉพาะสิ่งที่ test สนใจ"** ที่ทำให้ test อ่านง่ายขึ้นมาก
เพราะผู้อ่านเห็นทันทีว่า test นี้ **สนใจ** field ไหนเป็นพิเศษ

### 622.5 `PostFactory` แบบเบื้องต้น (ยังไม่มีความสัมพันธ์)

`Post` มี field ที่จำเป็น (`required`) สองตัวที่ยังไม่มีค่า default ที่สมเหตุสมผลใน
ขั้นตอนนี้ คือ `author` (ForeignKey แบบ `null=False`) ในขณะที่ `category` เป็น
`null=True, blank=True` จึงไม่จำเป็นต้องใส่ค่าตอนนี้ก็ได้:

```python
# blog/tests/factories.py (ต่อจากเดิม)
import factory

from blog.models import Category, Post


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = "หมวดหมู่ทดสอบ"
    description = "คำอธิบายหมวดหมู่สำหรับการทดสอบ"


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = "บทความทดสอบมาตรฐาน"
    content = "เนื้อหาบทความทดสอบที่มีความยาวเพียงพอสำหรับการทดสอบ"
    is_published = True
    # ยังไม่ใส่ category และ author — จะเพิ่มด้วย SubFactory ในขั้นตอนที่ 623
```

เพราะ `author` เป็น field บังคับ การเรียก `PostFactory()` เฉย ๆ ตอนนี้จะยัง **fail**
ด้วย `IntegrityError` คุณต้องส่ง `author` เข้าไปตรง ๆ ก่อน:

```python
# blog/tests/test_factories_demo.py (ไฟล์ทดลองชั่วคราว เพื่อสาธิตขั้นตอนนี้)
import pytest

from accounts.models import CustomUser
from blog.tests.factories import PostFactory


@pytest.mark.django_db
def test_post_factory_requires_author_for_now():
    author = CustomUser.objects.create_user(username="temp_author", password="x")

    post = PostFactory(author=author)

    assert post.pk is not None
    assert post.author == author
    assert post.category is None   # ยังไม่ตั้งค่า เพราะ field นี้ optional
```

การต้องสร้าง `author` เองแบบนี้ทุกครั้งคือปัญหาที่แท้จริงที่ **`SubFactory`** ใน
ขั้นตอนถัดไปจะแก้ให้หมดไป

### 622.6 ตารางสรุป attribute ที่ประกาศได้ใน Factory (ภาพรวมก่อนเจาะลึก)

| รูปแบบ attribute | ตัวอย่าง | ใช้เมื่อไหร่ |
|---|---|---|
| ค่าคงที่ (literal) | `is_published = True` | ค่าที่เหมือนกันทุก instance และไม่จำเป็นต้องสุ่ม |
| `factory.SubFactory(...)` | `author = factory.SubFactory(UserFactory)` | field ที่เป็น ForeignKey ไปยัง model อื่น (ขั้นตอนที่ 623) |
| `factory.Sequence(...)` | `slug = factory.Sequence(lambda n: f"post-{n}")` | field ที่ต้อง unique (ขั้นตอนที่ 624) |
| `factory.Faker(...)` | `title = factory.Faker("sentence")` | ข้อมูลที่อยากให้สมจริงและหลากหลาย (ขั้นตอนที่ 625) |
| `factory.LazyAttribute(...)` | คำนวณจาก field อื่นใน instance เดียวกัน | เมื่อ field หนึ่งขึ้นกับค่าของอีก field |
| `factory.Trait(...)` | เปิด/ปิดกลุ่ม field พร้อมกันด้วย flag เดียว | สร้าง "รูปแบบสำเร็จรูป" ของ object (ขั้นตอนที่ 626) |
| `@factory.post_generation` | ทำงานหลัง object ถูก `save()` แล้ว | ตั้งค่า many-to-many หรือความสัมพันธ์ย้อนกลับ (ขั้นตอนที่ 627) |

---

## ขั้นตอนที่ 623: `SubFactory` สำหรับสร้างความสัมพันธ์

### 623.1 ปัญหา: ForeignKey ที่ต้องมีเสมอ

จากขั้นตอนก่อนหน้า เราเห็นแล้วว่าการสร้าง `Post` ต้องมี `author` เสมอ (และในโลกจริง
อยากให้มี `category` ติดมาด้วยเพื่อให้ทดสอบ relationship ได้ครบ) การต้องสร้าง
`CustomUser` และ `Category` เองทุกครั้งก่อนเรียก `PostFactory` ทำให้เสียประโยชน์ของ
Factory Boy ไปมาก — **`SubFactory`** คือคำตอบ: มันบอก Factory Boy ว่า "field นี้ให้
สร้าง object จาก factory อีกตัวหนึ่งให้อัตโนมัติ"

### 623.2 สร้าง `UserFactory` สำหรับ `CustomUser` ก่อน

```python
# accounts/tests/factories.py
import factory

from accounts.models import CustomUser


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = CustomUser
        django_get_or_create = ("username",)

    username = "testuser"
    email = "testuser@example.com"
    is_verified_author = False

    @factory.post_generation
    def password(self, create, extracted, **kwargs):
        # อธิบายละเอียดเรื่อง post_generation ในขั้นตอนที่ 627
        # ตอนนี้ขอให้รู้แค่ว่ามันช่วย hash password ให้ถูกต้องแทนการเก็บ plaintext
        raw_password = extracted or "testpass123"
        self.set_password(raw_password)
        if create:
            self.save()
```

**จุดสำคัญ**: การสร้าง user ด้วย `CustomUser.objects.create(password="testpass123")`
ตรง ๆ จะเก็บ **plaintext password ลงฐานข้อมูล** ซึ่งผิดตั้งแต่ต้น (`AbstractUser`
คาดหวังให้ password ถูก hash เสมอผ่าน `set_password()`) เราจึงต้องใช้
`post_generation` เพื่อเรียก `set_password()` ให้ถูกต้อง (รายละเอียดเต็มของกลไกนี้
อยู่ในขั้นตอนที่ 627 — ในขั้นตอนนี้ให้โฟกัสที่ `SubFactory` เป็นหลักก่อน)

### 623.3 ใช้ `SubFactory` ใน `PostFactory`

```python
# blog/tests/factories.py
import factory

from accounts.tests.factories import UserFactory
from blog.models import Category, Post


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = "หมวดหมู่ทดสอบ"
    description = "คำอธิบายหมวดหมู่สำหรับการทดสอบ"


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = "บทความทดสอบมาตรฐาน"
    content = "เนื้อหาบทความทดสอบที่มีความยาวเพียงพอสำหรับการทดสอบ"
    is_published = True
    category = factory.SubFactory(CategoryFactory)
    author = factory.SubFactory(UserFactory)
```

ตอนนี้การสร้าง `Post` เหลือแค่บรรทัดเดียว โดยที่ `category` และ `author` ถูกสร้างให้
อัตโนมัติผ่าน factory ของมันเอง:

```python
>>> from blog.tests.factories import PostFactory
>>> post = PostFactory()
>>> post.author.username
'testuser'
>>> post.category.name
'หมวดหมู่ทดสอบ'
```

### 623.4 override field ที่อยู่ "ลึก" เข้าไปใน `SubFactory` ได้ด้วย double underscore

Factory Boy อนุญาตให้ override field ของ object ที่สร้างผ่าน `SubFactory` โดยตรงจาก
`PostFactory` เลย ผ่าน syntax `category__<field>` (สังเกต underscore สองตัว):

```python
>>> post = PostFactory(category__name="เทคโนโลยี", author__username="somchai")
>>> post.category.name
'เทคโนโลยี'
>>> post.author.username
'somchai'
```

Syntax นี้ทำงานได้ลึกกี่ชั้นก็ได้ตราบใดที่ field นั้นเป็น `SubFactory` ต่อกันเป็นทอด ๆ
เช่น `post.author.profile.website` ก็เขียนเป็น `author__profile__website="..."` ได้
เช่นกัน (ถ้า `UserFactory` มี `profile` เป็น `SubFactory` หรือ `RelatedFactory` ของ
`ProfileFactory`)

### 623.5 ใช้ object ที่มีอยู่แล้วแทนการสร้างใหม่ทุกครั้ง

บางครั้งคุณต้องการให้หลาย `Post` ใช้ `Category` หรือ `author` **ตัวเดียวกัน** (เช่น
ทดสอบว่า "ผู้เขียนคนหนึ่งมีบทความ 3 บทความ") วิธีที่ถูกต้องคือสร้าง object นั้นก่อน
แล้วส่งเข้าไปแทน `SubFactory` ตรง ๆ:

```python
import pytest

from accounts.tests.factories import UserFactory
from blog.tests.factories import PostFactory


@pytest.mark.django_db
def test_author_has_multiple_posts():
    author = UserFactory(username="prolific_writer")

    posts = [PostFactory(author=author) for _ in range(3)]

    assert all(post.author == author for post in posts)
    assert author.posts.count() == 3
```

สิ่งสำคัญคือ **ถ้าไม่ override `author`, Factory Boy จะเรียก `UserFactory()` ใหม่
ทุกครั้ง** — นี่คือพฤติกรรม default ที่ถูกต้องแล้ว เพราะโดยทั่วไป test ควรมีข้อมูล
ที่เป็นอิสระจากกัน (independent) ไม่ใช่ผูกกับ instance เดียวกันโดยไม่ตั้งใจ

### 623.6 `factory.SelfAttribute` และ `factory.LazyAttribute`: อ้างอิง field อื่นใน instance เดียวกัน

บางครั้ง field หนึ่งต้องคำนวณจากอีก field หนึ่งภายใน object เดียวกัน (ไม่ใช่จาก
`SubFactory` ที่เป็น object แยก) ใช้ `LazyAttribute` โดยรับ parameter `obj` ที่แทน
instance กำลังถูกสร้าง:

```python
import factory

from blog.models import Comment


class CommentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Comment

    content = factory.LazyAttribute(
        lambda obj: f"ความเห็นต่อบทความเรื่อง {obj.post.title}"
    )
    post = factory.SubFactory(PostFactory)
    author = factory.SubFactory(UserFactory)
```

```python
>>> comment = CommentFactory()
>>> comment.content
'ความเห็นต่อบทความเรื่อง บทความทดสอบมาตรฐาน'
```

`factory.SelfAttribute("field_name")` คือทางลัดของ `LazyAttribute` เมื่อแค่ต้องการ
"คัดลอกค่า" จาก field อื่นตรง ๆ โดยไม่ต้องคำนวณอะไรเพิ่ม เช่น
`author = factory.SelfAttribute("post.author")` ถ้าต้องการให้ comment ผูกกับ author
คนเดียวกับ post (ในที่นี้เราไม่ได้ทำแบบนั้นเพราะต้องการให้ comment มาจากคนละคนกับ
เจ้าของบทความตามธรรมชาติของระบบคอมเมนต์จริง)

### 623.7 ตารางสรุป: `SubFactory` vs การส่ง instance ตรง ๆ vs `LazyAttribute`

| เทคนิค | สร้าง object ใหม่ทุกครั้งหรือไม่ | ใช้เมื่อไหร่ |
|---|---|---|
| `category = factory.SubFactory(CategoryFactory)` | ✅ ใหม่ทุกครั้ง (ค่า default) | ต้องการ FK ที่มีอยู่เสมอ แต่ไม่สนว่าเป็น instance ไหน |
| `PostFactory(category=my_category)` | ❌ ใช้ instance เดิม | ต้องการให้หลาย object แชร์ FK เดียวกันโดยตั้งใจ |
| `factory.LazyAttribute(lambda obj: ...)` | ไม่เกี่ยวกับการสร้าง object ใหม่ | คำนวณค่า field จาก field อื่นใน instance เดียวกัน |
| `factory.SelfAttribute("other_field")` | ไม่เกี่ยวกับการสร้าง object ใหม่ | คัดลอกค่าจาก field อื่นตรง ๆ แบบสั้น ๆ |

---

## ขั้นตอนที่ 624: `Sequence` สำหรับ field ที่ต้องไม่ซ้ำกัน

### 624.1 ปัญหา `UNIQUE constraint` เมื่อสร้าง object ซ้ำหลายตัว

ลองสร้าง `Category` ซ้ำสองครั้งด้วย factory จากขั้นตอนที่แล้วดู:

```python
>>> from blog.tests.factories import CategoryFactory
>>> CategoryFactory()
>>> CategoryFactory()
django.db.utils.IntegrityError: UNIQUE constraint failed: blog_category.name
```

เพราะ `name = "หมวดหมู่ทดสอบ"` เป็นค่าคงที่ตัวเดียวกันทุกครั้ง แต่ `Category.name`
ถูกประกาศเป็น `unique=True` ในโมเดล — นี่คือปัญหาที่เกิดกับทุก field ที่มี
`unique=True` ไม่ว่าจะเป็น `slug`, `email`, หรือ `username`

### 624.2 `factory.Sequence`: ใส่เลขนับที่เพิ่มขึ้นเรื่อย ๆ

`factory.Sequence` รับฟังก์ชันที่มี parameter หนึ่งตัวชื่อ `n` (เลขจำนวนเต็มที่
Factory Boy นับเพิ่มขึ้นทุกครั้งที่สร้าง object ใหม่จาก factory class นั้น) แล้ว
คืนค่าที่ต้องการ:

```python
import factory

from blog.models import Category


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"หมวดหมู่ทดสอบ {n}")
    description = "คำอธิบายหมวดหมู่สำหรับการทดสอบ"
```

```python
>>> CategoryFactory().name
'หมวดหมู่ทดสอบ 0'
>>> CategoryFactory().name
'หมวดหมู่ทดสอบ 1'
>>> CategoryFactory().name
'หมวดหมู่ทดสอบ 2'
```

ตัวเลขเริ่มจาก `0` และเพิ่มขึ้นทีละ 1 **ต่อ factory class** (ไม่ใช่ต่อ field) และ
**นับต่อเนื่องตลอด process เดียวกัน** ไม่รีเซ็ตระหว่าง test แต่ละตัว (สำคัญมาก จะ
อธิบายผลกระทบในขั้นตอนที่ 624.5)

### 624.3 ใช้ `Sequence` กับ `slug`, `email`, `username` ที่ unique ทั้งหมด

```python
# accounts/tests/factories.py
import factory

from accounts.models import CustomUser


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = CustomUser
        django_get_or_create = ("username",)

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    is_verified_author = False

    @factory.post_generation
    def password(self, create, extracted, **kwargs):
        raw_password = extracted or "testpass123"
        self.set_password(raw_password)
        if create:
            self.save()
```

```python
# blog/tests/factories.py
import factory

from blog.models import Category, Tag


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"หมวดหมู่ทดสอบ {n}")
    description = "คำอธิบายหมวดหมู่สำหรับการทดสอบ"


class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Tag

    name = factory.Sequence(lambda n: f"แท็ก{n}")
```

### 624.4 `factory.LazyAttributeSequence`: ผสาน `Sequence` กับ field อื่นในบรรทัดเดียว

เมื่อค่า sequence ต้องขึ้นกับ field อื่นของ instance เดียวกันด้วย (ไม่ใช่แค่ตัวเลข
ล้วน ๆ) ใช้ `LazyAttributeSequence` ที่รับทั้ง `obj` และ `n`:

```python
import factory

from blog.models import Post


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Sequence(lambda n: f"บทความทดสอบลำดับที่ {n}")
    slug = factory.LazyAttributeSequence(
        lambda obj, n: f"post-{n}-{obj.title[:10]}"
    )
    content = "เนื้อหาบทความทดสอบที่มีความยาวเพียงพอสำหรับการทดสอบ"
    is_published = True
```

ในทางปฏิบัติ เนื่องจาก `Post.save()` สร้าง slug ให้อัตโนมัติจาก `title` อยู่แล้วผ่าน
`slugify(self.title, allow_unicode=True)` เราจึง **ไม่จำเป็นต้องกำหนด `slug` ใน
factory เลย** ปล่อยให้ model จัดการเอง (ตัวอย่าง `LazyAttributeSequence` ข้างต้นมีไว้
แสดงเทคนิคให้เห็นภาพ สำหรับกรณีที่ model ไม่มี logic auto-slug ให้ในตัว)

### 624.5 ข้อควรระวัง: ตัวนับ `Sequence` ไม่รีเซ็ตข้าม test โดยอัตโนมัติ

เพราะตัวนับเป็นตัวแปรระดับ class ที่อยู่ตลอดการรัน pytest process เดียว ค่าตัวเลขจะ
**ไหลต่อเนื่องข้ามไฟล์ test** เช่น ถ้า `test_models.py` สร้าง `UserFactory` ไปแล้ว
5 ตัว (`user0`-`user4`) พอมาถึง `test_views.py` ตัวถัดไปจะเริ่มที่ `user5` ไม่ใช่
`user0` — โดยทั่วไป **นี่คือพฤติกรรมที่ถูกต้องและควรปล่อยไว้แบบนี้** เพราะ:

- Test **ไม่ควร** พึ่งพาค่า sequence ที่แน่นอน (เช่น `assert user.username == "user0"`
  ถือเป็น anti-pattern เดียวกับที่เตือนเรื่อง primary key ใน Part 061 ขั้นตอนที่ 604.5)
- การรีเซ็ตตัวนับเองจะทำให้ test ที่รันแบบขนานด้วย `pytest-xdist` (Part 061 ขั้นตอนที่
  609.2) ชนกันได้ง่ายขึ้น เพราะแต่ละ worker process มีตัวนับของตัวเองอยู่แล้วโดย
  ธรรมชาติ การบังคับรีเซ็ตจะไปสร้างปัญหาซ้ำ

หากมีเหตุผลเฉพาะที่ต้องรีเซ็ตจริง ๆ (เช่น debug และต้องการผลลัพธ์คาดเดาได้ระหว่าง
พัฒนา) ทำได้ผ่าน `reset_sequence()`:

```python
>>> from blog.tests.factories import CategoryFactory
>>> CategoryFactory.reset_sequence(0)
>>> CategoryFactory().name
'หมวดหมู่ทดสอบ 0'
```

### 624.6 ตารางสรุปว่า field แบบไหนควรใช้ `Sequence`

| ลักษณะ field ใน model | ควรใช้ `Sequence` หรือไม่ |
|---|---|
| มี `unique=True` (เช่น `email`, `username`, `slug`) | ✅ ใช้แน่นอน |
| มี `unique_together`/`UniqueConstraint` ร่วมกับ field อื่น | ✅ ใช้ร่วมกับ `LazyAttributeSequence` หรือ override เป็นคู่ตอนเรียก |
| ไม่มีข้อจำกัด unique แต่อยากให้แต่ละ object แยกแยะง่ายตอน debug | ⚠️ ใช้ได้ แต่ไม่บังคับ — `Faker` (ขั้นตอนที่ 625) มักให้ผลดีกว่าเพราะสมจริงกว่า |
| Boolean, Choice field ที่มีค่าจำกัด | ❌ ไม่เหมาะ ใช้ค่าคงที่หรือ `Trait` (ขั้นตอนที่ 626) แทน |

---

## ขั้นตอนที่ 625: ผสาน `Faker` เพื่อสร้างข้อมูลปลอมที่สมจริง

### 625.1 ทำไม `factory.Sequence("หมวดหมู่ทดสอบ 0", "หมวดหมู่ทดสอบ 1", ...)` ยังไม่พอ

ข้อมูลจาก `Sequence` ถึงจะไม่ซ้ำกัน แต่ก็ยัง **ไม่สมจริง** — ในโลกจริง ชื่อผู้ใช้ อีเมล
หรือเนื้อหาบทความมีความหลากหลายมาก การทดสอบด้วยข้อมูลซ้ำรูปแบบเดิมทำให้พลาดบั๊กที่
เกิดจากข้อมูลขอบเขต เช่น ชื่อที่มีอักขระพิเศษ, อีเมลรูปแบบแปลก, ข้อความยาวเกินขนาด
`max_length` ที่ตั้งไว้

**Faker** คือไลบรารีที่ Factory Boy ผูกไว้ให้ในตัว (ติดตั้งมาพร้อมกันตั้งแต่
ขั้นตอนที่ 621.6) มี "provider" นับร้อยชนิดสำหรับสุ่มข้อมูลสมจริงตามหมวดหมู่ต่าง ๆ
และรองรับหลาย **locale** รวมถึงภาษาไทย (`th_TH`)

### 625.2 syntax พื้นฐานของ `factory.Faker`

```python
import factory


class ExampleFactory(factory.django.DjangoModelFactory):
    name = factory.Faker("name")               # ชื่อคนแบบสุ่ม
    email = factory.Faker("email")              # อีเมลแบบสุ่ม
    bio = factory.Faker("paragraph")            # ข้อความหนึ่งย่อหน้า
    created_date = factory.Faker("date_this_year")   # วันที่แบบสุ่มในปีนี้
```

### 625.3 ตาราง Faker provider ที่ใช้บ่อยที่สุดในงาน Django

| Provider | ตัวอย่างผลลัพธ์ | ใช้กับ field ประเภทไหน |
|---|---|---|
| `name` | `"John Smith"` | `CharField` สำหรับชื่อคน |
| `first_name` / `last_name` | `"John"` / `"Smith"` | `CharField` แยกชื่อ-นามสกุล |
| `user_name` | `"jsmith92"` | `username` |
| `email` | `"jsmith@example.org"` | `EmailField` |
| `word` | `"technology"` | `CharField` สั้น ๆ เช่นชื่อ tag |
| `words(nb=3)` | `["apple", "banana", "cherry"]` | สร้าง list คำ |
| `sentence` | `"Lorem ipsum dolor sit amet."` | หัวข้อบทความสั้น ๆ |
| `paragraph` | ข้อความหนึ่งย่อหน้า | `excerpt` |
| `text(max_nb_chars=500)` | ข้อความยาวควบคุมความยาวได้ | `content` ของบทความ |
| `boolean` | `True`/`False` แบบสุ่ม | `BooleanField` |
| `date_time_this_year` | `datetime` แบบสุ่มในปีนี้ | `DateTimeField` |
| `url` | `"https://example.com/"` | `URLField` |
| `image_url` | URL รูปภาพปลอม | ทดสอบ field ที่เก็บ URL รูป |
| `slug` | `"lorem-ipsum-dolor"` | `SlugField` (แต่ระวังเรื่อง unique ยังต้องผสม `Sequence`) |
| `phone_number` | `"555-0100"` | `phone_number` (รูปแบบอาจไม่ตรง locale ไทยเสมอไป ดู 625.4) |

### 625.4 ใช้ locale ภาษาไทยเพื่อความสมจริงยิ่งขึ้น

เนื่องจากหลักสูตรนี้สอนสร้างระบบภาษาไทย การกำหนด `locale="th_TH"` ให้ `Faker` จะได้
ชื่อคนและข้อความที่เหมาะกับบริบทมากกว่า:

```python
# blog/tests/factories.py
import factory

from accounts.tests.factories import UserFactory
from blog.models import Category, Post, Tag


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"หมวดหมู่ทดสอบ {n}")
    description = factory.Faker("paragraph", locale="th_TH")


class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Tag

    name = factory.Sequence(lambda n: f"แท็ก{n}")


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence", nb_words=6, locale="th_TH")
    content = factory.Faker("text", max_nb_chars=800, locale="th_TH")
    is_published = True
    category = factory.SubFactory(CategoryFactory)
    author = factory.SubFactory(UserFactory)
```

หรือกำหนด locale ให้ **ทั้งไฟล์** ในคราวเดียวผ่าน `factory.Faker._DEFAULT_LOCALE`
(สะดวกกว่าถ้าทุก field ในโปรเจกต์ต้องการภาษาไทยเหมือนกันหมด):

```python
# conftest.py (root) หรือไฟล์ตั้งค่า test ส่วนกลาง
import factory

factory.Faker._DEFAULT_LOCALE = "th_TH"
```

หลังตั้งค่านี้แล้ว ไม่ต้องระบุ `locale="th_TH"` ซ้ำในทุก `factory.Faker(...)` อีก

### 625.5 `Faker.seed()`: ทำให้ผลลัพธ์สุ่มซ้ำได้ (deterministic) เมื่อจำเป็น

ปกติ `Faker` จะสุ่มค่าใหม่ทุกครั้งที่รัน test ซึ่งเป็นพฤติกรรมที่ดีสำหรับการจับ edge
case ที่ไม่คาดคิด แต่บางครั้ง (เช่นตอน debug ว่าทำไม test fail เฉพาะบางค่า) คุณอยาก
ให้ผลลัพธ์เดิมซ้ำทุกครั้งที่รัน สามารถ seed ได้ด้วย `factory.Faker` เอง:

```python
import factory


def test_debug_with_fixed_seed():
    factory.Faker._get_faker().seed_instance(12345)

    from blog.tests.factories import PostFactory
    post = PostFactory.build()
    print(post.title)   # ได้ค่าเดิมทุกครั้งที่รันด้วย seed เดียวกัน
```

**ข้อควรระวัง**: อย่า seed ค่าคงที่นี้ไว้ถาวรใน `conftest.py` ของทั้งโปรเจกต์ เพราะ
จะทำให้ test สูญเสียประโยชน์สำคัญที่สุดของการสุ่มข้อมูล คือการเจอ edge case ที่ไม่
คาดคิดในระยะยาว ใช้ seed เฉพาะตอน debug ปัญหาเฉพาะจุดเท่านั้น

### 625.6 สร้าง Faker provider แบบกำหนดเอง (Custom Provider)

เมื่อ Faker ไม่มี provider ที่ตรงกับความต้องการเฉพาะของโดเมนธุรกิจ (เช่น เลขบัตร
ประชาชนไทย, รหัสไปรษณีย์ไทย) สามารถเขียน provider เองและลงทะเบียนเพิ่มได้:

```python
# blog/tests/providers.py
from faker.providers import BaseProvider


class ThaiBlogProvider(BaseProvider):
    _post_topics = [
        "แนะนำเทคโนโลยีใหม่", "รีวิวหนังสือ", "สรุปข่าวไอทีประจำสัปดาห์",
        "เทคนิคการเขียนโค้ด", "บทสัมภาษณ์นักพัฒนา",
    ]

    def blog_post_topic(self):
        return self.random_element(self._post_topics)
```

```python
# blog/tests/factories.py
import factory

from blog.tests.providers import ThaiBlogProvider

factory.Faker.add_provider(ThaiBlogProvider)


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("blog_post_topic")
    # ... field อื่น ๆ เหมือนเดิม
```

```python
>>> PostFactory.build().title
'เทคนิคการเขียนโค้ด'
```

### 625.7 ตารางสรุปการเลือกใช้ `Sequence` vs `Faker` vs ค่าคงที่

| สถานการณ์ | ตัวเลือกที่เหมาะสม |
|---|---|
| field ต้อง unique และไม่สนความสมจริง (เช่น `username` ภายใน) | `factory.Sequence` |
| field ต้องสมจริงและ **ไม่จำเป็นต้อง unique** (เช่น `content`, `bio`) | `factory.Faker` |
| field ต้อง unique **และ** สมจริง (เช่น `email` จำนวนมากในระบบใหญ่) | ผสมทั้งคู่: `factory.Faker("email")` มักไม่ชนกันในทางปฏิบัติ แต่ถ้าชนให้ใช้ `factory.LazyAttributeSequence` ผสม Faker คำนวณเอง |
| field ที่ test ทุกตัวไม่สนใจค่า แค่ต้องการให้ "มีค่า" | ค่าคงที่ธรรมดา (literal) — เร็วที่สุด อ่านง่ายที่สุด |
| field ที่มีค่าจำกัด (choices) | ค่าคงที่ หรือ `factory.Iterator([...])` วนค่าตามลิสต์ที่กำหนด |

---

## ขั้นตอนที่ 626: `Trait` สำหรับสร้าง Factory หลายรูปแบบจากคลาสเดียว

### 626.1 ปัญหา: ต้องการ object หลายรูปแบบจาก factory เดียวกัน

ใน test ของแอป `blog` (Part 059-061) เรามักต้องการ `Post` สองแบบคือ **published**
และ **draft** ถ้าเขียนแยกเป็นสอง factory class จะซ้ำซ้อนมาก:

```python
# ❌ ซ้ำซ้อน ไม่ DRY
class PublishedPostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence")
    content = factory.Faker("text")
    is_published = True


class DraftPostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence")
    content = factory.Faker("text")
    is_published = False
```

### 626.2 `class Params` และ `factory.Trait`: ประกาศ "รูปแบบสำเร็จรูป" ในคลาสเดียว

```python
import factory

from accounts.tests.factories import UserFactory
from blog.models import Post


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence", nb_words=6, locale="th_TH")
    content = factory.Faker("text", max_nb_chars=800, locale="th_TH")
    is_published = False
    category = factory.SubFactory("blog.tests.factories.CategoryFactory")
    author = factory.SubFactory(UserFactory)

    class Params:
        published = factory.Trait(
            is_published=True,
        )
        featured = factory.Trait(
            is_published=True,
            title=factory.Sequence(lambda n: f"[แนะนำ] บทความเด่นอันดับ {n}"),
        )
        by_verified_author = factory.Trait(
            author=factory.SubFactory(UserFactory, is_verified_author=True),
        )
```

**หมายเหตุเรื่อง `SubFactory` แบบ string path**: `factory.SubFactory(
"blog.tests.factories.CategoryFactory")` ใช้ string แทนการ import class ตรง ๆ เพื่อ
หลีกเลี่ยงปัญหา circular import เมื่อสอง factory อยู่คนละไฟล์แต่ต้องอ้างถึงกันเอง —
เทคนิคเดียวกับที่ Django model ใช้ string `"app_label.ModelName"` ใน `ForeignKey`

### 626.3 การใช้งาน Trait

```python
>>> from blog.tests.factories import PostFactory
>>> draft = PostFactory()                       # ค่า default: is_published=False
>>> draft.is_published
False

>>> live_post = PostFactory(published=True)     # เปิด trait "published"
>>> live_post.is_published
True

>>> featured = PostFactory(featured=True)       # เปิด trait "featured"
>>> featured.is_published
True
>>> featured.title
'[แนะนำ] บทความเด่นอันดับ 0'

>>> vip_post = PostFactory(by_verified_author=True)
>>> vip_post.author.is_verified_author
True
```

สังเกตว่า `PostFactory(published=True)` **ไม่ได้แก้ model** เลยแม้แต่น้อย —
`published` ไม่ใช่ field จริงของ `Post` แต่เป็น **parameter พิเศษของ factory** ที่มี
อยู่เฉพาะตอนสร้างข้อมูลทดสอบเท่านั้น Factory Boy จะดักจับ parameter นี้และ apply
field ทั้งหมดที่ประกาศไว้ใน `factory.Trait(...)` ให้อัตโนมัติ

### 626.4 รวมหลาย Trait พร้อมกัน

```python
>>> post = PostFactory(featured=True, by_verified_author=True)
>>> post.is_published
True
>>> post.title.startswith("[แนะนำ]")
True
>>> post.author.is_verified_author
True
```

ถ้า Trait หลายตัวกำหนด field เดียวกันซ้ำกัน **Trait ที่เปิดทีหลังจะทับ Trait ก่อนหน้า**
(ตามลำดับที่ Python ประมวลผล keyword argument) ดังนั้นควรออกแบบให้แต่ละ Trait ไม่
ชนกันบน field เดียวกันเพื่อพฤติกรรมที่คาดเดาได้ง่าย

### 626.5 ตัวอย่างจริง: ใช้ Trait แทน fixture แยกจาก Part 061

เทียบกับ `blog/conftest.py` ที่เขียนไว้ใน Part 061 ขั้นตอนที่ 606.5:

```python
# แบบเดิมจาก Part 061 — ต้องมี fixture แยกสำหรับแต่ละสถานะ
@pytest.fixture
def published_post(db, post_data):
    return Post.objects.create(**post_data)


@pytest.fixture
def draft_post(db, post_data):
    data = {**post_data, "is_published": False}
    return Post.objects.create(**data)
```

```python
# แบบใหม่ด้วย Factory Boy + Trait — factory เดียวครอบคลุมทุกสถานการณ์
@pytest.mark.django_db
def test_published_post_appears_in_list(client):
    from blog.tests.factories import PostFactory

    post = PostFactory(published=True)
    response = client.get("/blog/")
    assert post.title in response.content.decode()


@pytest.mark.django_db
def test_draft_post_hidden_from_list(client):
    from blog.tests.factories import PostFactory

    post = PostFactory(published=False)
    response = client.get("/blog/")
    assert post.title not in response.content.decode()
```

### 626.6 ตารางสรุป Trait ที่สร้างไว้และผลของแต่ละตัว

| Trait | ผลกระทบต่อ field | ตัวอย่างการเรียกใช้ |
|---|---|---|
| `published` | `is_published = True` | `PostFactory(published=True)` |
| `featured` | `is_published = True` + `title` ขึ้นต้นด้วย "[แนะนำ]" | `PostFactory(featured=True)` |
| `by_verified_author` | `author` เป็น user ที่ `is_verified_author=True` | `PostFactory(by_verified_author=True)` |
| (ไม่เปิด trait ใด) | ใช้ค่า default ปกติ (`is_published=False`) | `PostFactory()` |

---

## ขั้นตอนที่ 627: `post_generation` hook — ทำงานหลัง object ถูกสร้างแล้ว

### 627.1 ปัญหา: ManyToManyField ต้องมี pk ของ instance หลักก่อนถึงจะเชื่อมได้

`Post.tags` เป็น `ManyToManyField` ซึ่งมีข้อจำกัดสำคัญของ Django คือ **ต้องบันทึก
(save) instance หลักลงฐานข้อมูลให้มี pk ก่อน** ถึงจะเรียก `.add()` บนความสัมพันธ์
many-to-many ได้ ทำให้ไม่สามารถกำหนด `tags` เป็น field ธรรมดาใน `class PostFactory`
แบบเดียวกับ `SubFactory` ได้ตรง ๆ — Factory Boy แก้ปัญหานี้ด้วย **`post_generation`
hook** ซึ่งเป็นโค้ดที่รันขึ้น **หลังจาก** object ถูกสร้าง (และบันทึกแล้วถ้าใช้
`create()`) เสมอ

### 627.2 syntax ของ `@factory.post_generation`

```python
import factory


class SomeFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = SomeModel

    @factory.post_generation
    def field_name(self, create, extracted, **kwargs):
        # self      = instance ที่เพิ่งถูกสร้าง (บันทึกแล้วถ้า create=True)
        # create    = True ถ้าเรียกผ่าน .create()/.create_batch(), False ถ้าเรียกผ่าน .build()
        # extracted = ค่าที่ผู้เรียกส่งเข้ามาผ่าน field_name=... ตอนเรียก factory
        # kwargs    = ค่าที่ส่งผ่าน field_name__key=value ตอนเรียก factory
        ...
```

### 627.3 ใช้ `post_generation` เพิ่ม tags ให้ `Post`

```python
# blog/tests/factories.py
import factory

from accounts.tests.factories import UserFactory
from blog.models import Category, Post, Tag


class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Tag

    name = factory.Sequence(lambda n: f"แท็ก{n}")


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence", nb_words=6, locale="th_TH")
    content = factory.Faker("text", max_nb_chars=800, locale="th_TH")
    is_published = False
    category = factory.SubFactory(CategoryFactory)
    author = factory.SubFactory(UserFactory)

    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            # ถูกเรียกผ่าน .build() ที่ไม่บันทึกลงฐานข้อมูล
            # ManyToMany ยังใช้ .add() ไม่ได้เพราะ instance ยังไม่มี pk จริง
            return

        if extracted:
            # กรณีผู้เรียกส่ง tags เข้ามาเอง เช่น PostFactory(tags=[tag1, tag2])
            for tag in extracted:
                self.tags.add(tag)
        else:
            # กรณีไม่ได้ระบุ tags มา — สร้างแท็กใหม่ให้ 2 แท็กเป็นค่า default
            self.tags.add(TagFactory(), TagFactory())
```

การใช้งาน:

```python
>>> from blog.tests.factories import PostFactory, TagFactory
>>> post = PostFactory()
>>> post.tags.count()
2

>>> django_tag = TagFactory(name="Django")
>>> python_tag = TagFactory(name="Python")
>>> post_with_specific_tags = PostFactory(tags=[django_tag, python_tag])
>>> list(post_with_specific_tags.tags.values_list("name", flat=True))
['Django', 'Python']

>>> post_no_tags = PostFactory(tags=[])
>>> post_no_tags.tags.count()
0
```

สังเกตความแตกต่างสำคัญ: `tags=[]` (list ว่าง) ถือว่า `extracted` เป็น falsy
เหมือนกับไม่ส่งอะไรมาเลยหรือไม่? **ไม่ใช่** — `extracted` จะเป็น `[]` ซึ่งเป็น falsy
ใน Python เหมือนกัน ทำให้โค้ดข้างบนตกไปที่ branch `else` โดยไม่ตั้งใจ! นี่คือ
edge case ที่ต้องระวัง — วิธีแก้ที่ถูกต้องคือเช็ค `is not None` แทน:

```python
    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return

        if extracted is not None:
            for tag in extracted:
                self.tags.add(tag)
        else:
            self.tags.add(TagFactory(), TagFactory())
```

ตอนนี้ `PostFactory(tags=[])` จะได้ post ที่ไม่มี tags เลยตามที่ตั้งใจจริง ๆ

### 627.4 `factory.RelatedFactory` และ `factory.RelatedFactoryList`: ทางลัดสำหรับความสัมพันธ์ย้อนกลับ

สำหรับความสัมพันธ์แบบ **reverse ForeignKey** (เช่น `Comment` ที่ชี้กลับมาที่ `Post`)
มีทางลัดที่สั้นกว่าการเขียน `post_generation` เอง คือ `factory.RelatedFactoryList`
(สร้างหลาย object ที่ชี้กลับมาที่ instance นี้โดยอัตโนมัติ):

```python
# blog/tests/factories.py
import factory

from blog.models import Comment, Post


class CommentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Comment

    content = factory.Faker("sentence", locale="th_TH")
    author = factory.SubFactory("accounts.tests.factories.UserFactory")
    post = factory.SubFactory("blog.tests.factories.PostFactory")


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence", nb_words=6, locale="th_TH")
    content = factory.Faker("text", max_nb_chars=800, locale="th_TH")
    is_published = False
    category = factory.SubFactory("blog.tests.factories.CategoryFactory")
    author = factory.SubFactory("accounts.tests.factories.UserFactory")

    comments = factory.RelatedFactoryList(
        CommentFactory, factory_related_name="post", size=0,
    )
```

การตั้ง `size=0` เป็นค่า default หมายความว่าปกติจะไม่สร้าง comment ให้เลย (เพื่อไม่ให้
ทุก test ที่เรียก `PostFactory()` ต้องแบกภาระสร้าง comment โดยไม่จำเป็น) แต่เปิดใช้
ได้เมื่อ test ต้องการจริง ๆ ผ่าน override ตอนเรียก:

```python
>>> post = PostFactory(comments__size=3)
>>> post.comments.count()
3
```

**ข้อควรระวังเรื่อง circular `SubFactory`**: สังเกตว่า `CommentFactory.post` และ
`PostFactory.comments` อ้างถึงกันไปมา จึงต้องใช้ string path (`"app.module.Class"`)
ทั้งคู่แทนการ import ตรง ๆ เพื่อหลีกเลี่ยง `ImportError` แบบ circular import

### 627.5 ตัวอย่างสมบูรณ์: `PostFactory` ที่รวมทุกเทคนิคจาก 622-627

```python
# blog/tests/factories.py — เวอร์ชันสมบูรณ์หลังจบขั้นตอนที่ 627
import factory

from accounts.models import CustomUser
from blog.models import Category, Comment, Post, Tag


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = CustomUser
        django_get_or_create = ("username",)

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    first_name = factory.Faker("first_name", locale="th_TH")
    last_name = factory.Faker("last_name", locale="th_TH")
    is_verified_author = False

    @factory.post_generation
    def password(self, create, extracted, **kwargs):
        raw_password = extracted or "testpass123"
        self.set_password(raw_password)
        if create:
            self.save()


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category

    name = factory.Sequence(lambda n: f"หมวดหมู่ทดสอบ {n}")
    description = factory.Faker("paragraph", locale="th_TH")


class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Tag

    name = factory.Sequence(lambda n: f"แท็ก{n}")


class CommentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Comment

    content = factory.Faker("sentence", locale="th_TH")
    author = factory.SubFactory(UserFactory)
    post = factory.SubFactory("blog.tests.factories.PostFactory")


class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence", nb_words=6, locale="th_TH")
    content = factory.Faker("text", max_nb_chars=800, locale="th_TH")
    is_published = False
    category = factory.SubFactory(CategoryFactory)
    author = factory.SubFactory(UserFactory)

    class Params:
        published = factory.Trait(is_published=True)
        featured = factory.Trait(
            is_published=True,
            title=factory.Sequence(lambda n: f"[แนะนำ] บทความเด่นอันดับ {n}"),
        )
        by_verified_author = factory.Trait(
            author=factory.SubFactory(UserFactory, is_verified_author=True),
        )

    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return
        if extracted is not None:
            for tag in extracted:
                self.tags.add(tag)
        else:
            self.tags.add(TagFactory(), TagFactory())

    comments = factory.RelatedFactoryList(
        CommentFactory, factory_related_name="post", size=0,
    )
```

โค้ดชุดนี้ **แทนที่โค้ด setup ที่เคยต้องเขียน 20-30 บรรทัดต่อ test ในสไตล์ Part
059-061 ทั้งหมด** ด้วย factory ไม่กี่คลาส ที่ประกอบกันได้อย่างยืดหยุ่นตามที่ test
แต่ละตัวต้องการ

---

## ขั้นตอนที่ 628: ใช้ Factory ร่วมกับ pytest fixture จาก Part 061

### 628.1 ทบทวน: fixture ธรรมดาที่ห่อ factory ไว้

วิธีที่ตรงไปตรงมาที่สุดในการผสาน Factory Boy กับระบบ fixture ของ pytest (Part 061)
คือเขียน fixture ธรรมดาที่เรียก factory ข้างในเลย:

```python
# blog/conftest.py
import pytest

from blog.tests.factories import CategoryFactory, PostFactory, TagFactory


@pytest.fixture
def category(db):
    return CategoryFactory()


@pytest.fixture
def published_post(db, category):
    return PostFactory(category=category, published=True)


@pytest.fixture
def draft_post(db, category):
    return PostFactory(category=category, published=False)
```

```python
# blog/tests/test_views.py
def test_list_view_shows_only_published_posts(client, published_post, draft_post):
    response = client.get("/blog/")
    content = response.content.decode()

    assert published_post.title in content
    assert draft_post.title not in content
```

วิธีนี้ใช้ได้ดีและยังคงหลักการ fixture ของ Part 061 ไว้ทุกประการ แต่ยังต้องเขียน
fixture wrapper เองทุกตัว — ขั้นตอนถัดไปจะแนะนำเครื่องมือที่ทำให้ขั้นตอนนี้เป็น
อัตโนมัติทั้งหมด

### 628.2 `pytest-factoryboy`: แปลง Factory เป็น fixture อัตโนมัติ

```bash
pip install "pytest-factoryboy~=2.7"
```

```bash
# requirements-dev.txt
-r requirements.txt

pytest~=8.3
pytest-django~=4.9
factory-boy~=3.3
pytest-factoryboy~=2.7
```

ฟังก์ชันหลักของไลบรารีนี้คือ `register()` ซึ่งรับ factory class แล้วสร้าง **fixture
หลายตัวให้อัตโนมัติ** โดยตั้งชื่อตาม model (แปลงเป็น snake_case):

```python
# blog/conftest.py
from pytest_factoryboy import register

from blog.tests.factories import CategoryFactory, CommentFactory, PostFactory, TagFactory

register(CategoryFactory)
register(TagFactory)
register(PostFactory)
register(CommentFactory)
```

`register(CategoryFactory)` เพียงบรรทัดเดียวสร้าง fixture ให้ทันที 2 ตัว:

| fixture ที่ได้ | ความหมาย |
|---|---|
| `category` | instance ของ `Category` ที่สร้างจาก `CategoryFactory()` (เรียกเมื่อ test ขอ parameter ชื่อนี้) |
| `category_factory` | factory class เอง (`CategoryFactory`) เผื่อ test ต้องการเรียกซ้ำหลายครั้งด้วย parameter ต่างกัน |

เช่นเดียวกัน `register(PostFactory)` ให้ fixture `post` และ `post_factory`,
`register(TagFactory)` ให้ `tag` และ `tag_factory`, `register(CommentFactory)` ให้
`comment` และ `comment_factory`

### 628.3 ทดสอบด้วย fixture ที่ได้มาอัตโนมัติ

```python
# blog/tests/test_views.py
import pytest
from django.urls import reverse


@pytest.mark.django_db
def test_detail_view_returns_200(client, post):
    url = reverse("blog:detail", kwargs={"slug": post.slug})
    response = client.get(url)

    assert response.status_code == 200
    assert response.context["post"] == post


@pytest.mark.django_db
def test_post_belongs_to_its_category(post, category):
    # หมายเหตุสำคัญ: pytest-factoryboy เชื่อม SubFactory เข้ากับ fixture โดยอัตโนมัติ
    # ทำให้ post.category กับ fixture `category` เป็น instance เดียวกันเป๊ะ ๆ
    assert post.category == category
```

ทดสอบตัวที่สองแสดงพลังของ `pytest-factoryboy` ชัดเจนที่สุด: เพราะ `PostFactory.category`
ประกาศเป็น `factory.SubFactory(CategoryFactory)` ไลบรารีจึงรู้ว่า fixture `post`
ต้อง "พึ่งพา" fixture `category` โดยอัตโนมัติ — เมื่อ test ขอทั้งสอง fixture พร้อมกัน
มันจะได้ instance เดียวกันเสมอ โดยไม่ต้องเขียนอะไรเพิ่มเลย

### 628.4 Override ค่า default ของ field ผ่านการ "ประกาศ fixture ทับ"

เพื่อปรับค่า default ของ field ใดใน factory ที่ลงทะเบียนไว้ ให้ประกาศ fixture ชื่อ
`<fixture_name>__<field_name>` ทับเข้าไปใน `conftest.py` หรือแม้แต่ในไฟล์ test เอง:

```python
# ปรับ default ของ post.title ให้เป็นค่าคงที่เฉพาะไฟล์นี้
@pytest.fixture
def post__title():
    return "หัวข้อพิเศษสำหรับชุดทดสอบนี้"


def test_post_title_override(post):
    assert post.title == "หัวข้อพิเศษสำหรับชุดทดสอบนี้"
```

หรือ override เฉพาะ test เดียวผ่าน `pytest.mark.parametrize` ร่วมกับ fixture ที่
`pytest-factoryboy` เตรียมให้ (`LazyFixture` สำหรับกรณีซับซ้อนกว่านั้น เอกสารเต็มอยู่
ที่ `pytest-factoryboy` PyPI page):

```python
@pytest.mark.parametrize("post__is_published", [True, False])
def test_post_published_states(post):
    assert post.is_published in (True, False)   # ตัวอย่างง่าย ๆ ให้เห็น syntax
```

### 628.5 ตัวอย่างเต็ม: ทดสอบ `Comment` ที่ผูกกับ `Post` และ `CustomUser` โดยอัตโนมัติ

```python
# blog/tests/test_comments.py
import pytest


@pytest.mark.django_db
def test_comment_belongs_to_post_and_author(comment, post, user):
    # เพราะ CommentFactory.post = SubFactory(PostFactory)
    # และ CommentFactory.author = SubFactory(UserFactory)
    # pytest-factoryboy เชื่อม fixture `comment`, `post`, `user` เข้าด้วยกันให้อัตโนมัติ
    assert comment.post == post
    assert comment.author == user


@pytest.mark.django_db
def test_post_has_related_comments_via_related_name(post, comment):
    assert comment in post.comments.all()
```

`user` ในที่นี้มาจาก `register(UserFactory)` ที่ต้องเพิ่มเข้าไปใน `conftest.py` ของ
`accounts` app (หรือ root `conftest.py` ถ้าต้องการให้ทุกแอปมองเห็น):

```python
# accounts/conftest.py
from pytest_factoryboy import register

from accounts.tests.factories import UserFactory

register(UserFactory)
```

### 628.6 ตารางสรุปข้อดี-ข้อจำกัดของ `pytest-factoryboy`

| ประเด็น | ข้อดี | ข้อควรระวัง |
|---|---|---|
| ปริมาณโค้ด | ลด boilerplate fixture ลงมาก (ไม่ต้องเขียน wrapper เอง) | ทีมที่ไม่คุ้นเคยอาจงงว่า fixture `post`, `category` มาจากไหน (ควรมี comment อธิบายใน `conftest.py`) |
| การเชื่อม `SubFactory` กับ fixture | อัตโนมัติ ไม่ต้องเขียน dependency เอง | ต้องเข้าใจกลไก `SubFactory` ให้ดีก่อน ไม่งั้น debug ยากเมื่อพฤติกรรมไม่ตรงคาด |
| การ override field | ทำผ่าน fixture `<name>__<field>` สั้นกระชับ | Syntax ใหม่ที่ต้องเรียนรู้เพิ่ม แยกจาก syntax `.parametrize` ปกติ |
| ความเร็วในการเริ่มใช้งานทีมใหม่ | เร็วมากถ้าคุ้นกับ Factory Boy อยู่แล้ว | ทีมที่เพิ่งเริ่มอาจสับสนกับ "fixture ที่ไม่ได้เขียนเอง" ควรอธิบายในเอกสารทีม |

---

## ขั้นตอนที่ 629: ข้อควรพิจารณาด้าน Performance — `build()` เทียบกับ `create()`

### 629.1 กลยุทธ์การสร้าง object ทั้ง 4 แบบของ Factory Boy

```python
from blog.tests.factories import PostFactory

post_built = PostFactory.build()     # สร้าง instance ใน memory เท่านั้น ไม่ยิง DB
post_created = PostFactory()          # เท่ากับ PostFactory.create() ยิง DB จริง
post_stub = PostFactory.stub()        # สร้าง object เบาที่สุด ไม่ใช่ Django model จริง
posts = PostFactory.build_batch(5)    # เหมือน build() แต่สร้างทีเดียวหลายตัวเป็น list
posts2 = PostFactory.create_batch(5)  # เหมือน create() แต่สร้างทีเดียวหลายตัว
```

### 629.2 ตารางเปรียบเทียบ

| กลยุทธ์ | ยิง Query ฐานข้อมูลหรือไม่ | `pk` หลังสร้าง | เหมาะกับ |
|---|---|---|---|
| `build()` | ❌ ไม่ยิงเลย | `None` | ทดสอบ method/property ของ model ที่ไม่ต้องพึ่งฐานข้อมูล เช่น `__str__()`, validation logic, property ที่คำนวณจาก field ตัวเอง |
| `create()` | ✅ ยิงจริง (INSERT) | มีค่า (เช่น `1`, `2`, ...) | ทดสอบ query, relationship, view ที่ query ฐานข้อมูลจริง, constraint ระดับ DB |
| `stub()` | ❌ ไม่ยิงเลย และไม่ใช่ instance ของ Django model ด้วยซ้ำ | ไม่มี | ทดสอบ logic ที่รับแค่ "object ที่มี attribute ตรงชื่อ field" โดยไม่ต้องพึ่ง Django ORM เลย (เร็วที่สุด) |
| `build_batch(n)` | ❌ ไม่ยิงเลย | `None` ทุกตัว | สร้างข้อมูลจำนวนมากสำหรับทดสอบ logic ล้วน ๆ (เช่น loop คำนวณ) |
| `create_batch(n)` | ✅ ยิงจริง n ครั้ง (หรือมากกว่า ถ้ามี `SubFactory`/`post_generation`) | มีค่าทุกตัว | seed ข้อมูลจำนวนมากสำหรับทดสอบ pagination, aggregation, performance |

### 629.3 วัดจำนวน query จริงด้วย `django_assert_num_queries`

`pytest-django` มี fixture `django_assert_num_queries` ที่ช่วยยืนยันว่าโค้ดยิง query
ตามจำนวนที่คาดไว้พอดี ใช้พิสูจน์ความแตกต่างระหว่าง `build()` กับ `create()` ได้ตรง ๆ:

```python
# blog/tests/test_factory_performance.py
import pytest

from blog.tests.factories import CategoryFactory


def test_build_hits_zero_queries(django_assert_num_queries):
    with django_assert_num_queries(0):
        category = CategoryFactory.build()

    assert category.pk is None


@pytest.mark.django_db
def test_create_hits_exactly_one_query(django_assert_num_queries):
    with django_assert_num_queries(1):
        category = CategoryFactory.create()

    assert category.pk is not None
```

สำหรับ `PostFactory` ที่มีทั้ง `SubFactory` (category, author) และ `post_generation`
(tags) การเรียก `create()` หนึ่งครั้งจะยิง **หลาย query พร้อมกัน**:

```python
@pytest.mark.django_db
def test_post_create_query_count(django_assert_max_num_queries):
    from blog.tests.factories import PostFactory

    # ใช้ max_num_queries แทน exact เพราะจำนวนอาจเปลี่ยนได้ตาม migration/index
    # แต่ต้องไม่เกินขอบเขตที่สมเหตุสมผล (INSERT category + INSERT author + INSERT post
    # + INSERT tag x2 + INSERT ความสัมพันธ์ m2m x2)
    with django_assert_max_num_queries(10):
        PostFactory()
```

### 629.4 `build()` กับ `SubFactory`: จริง ๆ แล้วไม่ยิง DB เลยหรือไม่

ข่าวดีคือ Factory Boy **ฉลาดพอที่จะส่งต่อ strategy เดียวกันลงไปใน `SubFactory` โดย
อัตโนมัติ** — เมื่อเรียก `PostFactory.build()` มันจะเรียก `CategoryFactory.build()`
และ `UserFactory.build()` ให้ด้วย (ไม่ใช่ `.create()`) ทำให้ทั้ง object หลักและ
object ที่เชื่อมโยงกันไม่มีตัวไหนแตะฐานข้อมูลเลยสักตัว:

```python
def test_build_cascades_to_subfactories(django_assert_num_queries):
    from blog.tests.factories import PostFactory

    with django_assert_num_queries(0):
        post = PostFactory.build()

    assert post.pk is None
    assert post.category.pk is None   # category ก็ build() เหมือนกัน ไม่ถูกสร้างจริง
    assert post.author.pk is None     # author ก็เช่นกัน
```

แต่ **`post_generation` (tags) เป็นข้อยกเว้นสำคัญ**: เพราะ ManyToMany ต้องมี pk ของ
`post` ก่อนเสมอ (ตามที่อธิบายในขั้นตอนที่ 627.1) โค้ดใน `tags` hook ของเราจึงมีเงื่อนไข
`if not create: return` ป้องกันไว้แล้ว — ผลคือ `PostFactory.build().tags.count()`
จะ **error** ทันทีถ้าพยายามเรียก เพราะ instance ยังไม่มี pk ใน DB (Django จะ raise
`ValueError: "<Post>" needs to have a value for field "id" before this many-to-many
relationship can be used.`) นี่คือข้อจำกัดโดยธรรมชาติของ ManyToMany ใน Django เอง
ไม่ใช่ข้อจำกัดของ Factory Boy

### 629.5 กลยุทธ์สร้างข้อมูลจำนวนมากอย่างมีประสิทธิภาพ (bulk seeding)

เมื่อต้องสร้างข้อมูลทดสอบจำนวนมาก (เช่น ทดสอบ pagination ที่ต้องมี 500 บทความ)
การเรียก `PostFactory.create_batch(500)` ตรง ๆ จะ **ช้ามาก** เพราะแต่ละตัวยิง
INSERT แยกกันหลาย query (รวม category, author ใหม่ทุกตัวถ้าไม่ override) วิธีที่ทีม
มืออาชีพใช้คือรวมกับ `bulk_create()` ของ Django ORM (Part 013):

```python
# blog/tests/test_pagination_performance.py
import pytest

from accounts.tests.factories import UserFactory
from blog.models import Post
from blog.tests.factories import CategoryFactory, PostFactory


@pytest.mark.django_db
def test_pagination_with_bulk_seeded_posts(client):
    # สร้าง category และ author เพียงครั้งเดียว ใช้ร่วมกันทุก post เพื่อลด query
    shared_category = CategoryFactory()
    shared_author = UserFactory()

    # build_batch ไม่ยิง DB เลย ได้ list ของ Post object ที่ยังไม่ถูกบันทึก
    unsaved_posts = PostFactory.build_batch(
        500,
        category=shared_category,
        author=shared_author,
    )

    # bulk_create ยิง INSERT รวดเดียว (หรือแบ่งเป็น batch ตาม batch_size) แทนที่จะ
    # ยิงทีละแถว 500 ครั้ง — เร็วกว่ามากในระดับหลายสิบเท่า
    Post.objects.bulk_create(unsaved_posts, batch_size=100)

    assert Post.objects.count() == 500
```

**ข้อจำกัดสำคัญที่ต้องรู้**: `bulk_create()` **ข้าม** `save()` ของแต่ละ instance
(จึงไม่มีการยิง signal `post_save` — ทบทวนเรื่อง signal ได้ที่ Part 019) และ **ข้าม
`post_generation` hook ของ Factory Boy ไปเลยโดยสิ้นเชิง** เพราะ hook นั้นทำงานเฉพาะ
ตอนเรียกผ่าน `.create()`/`.create_batch()` ปกติเท่านั้น ดังนั้นถ้าต้องการ tags ให้
แต่ละ post ด้วย ต้องเพิ่ม M2M เองแยกหลัง `bulk_create()`:

```python
    created_posts = Post.objects.bulk_create(unsaved_posts, batch_size=100)
    # ต้อง query กลับมาเอา pk จริง (SQLite/MySQL รุ่นเก่าไม่ return pk ให้ bulk_create
    # อัตโนมัติ — PostgreSQL คืน pk ให้ตั้งแต่ Django 4.0+ เป็นต้นไป)
```

### 629.6 ตารางสรุป: ควรเลือก strategy ไหนตามสถานการณ์

| สถานการณ์ | Strategy ที่แนะนำ |
|---|---|
| ทดสอบ `__str__()`, property, method ที่คำนวณจาก field ของ instance เดียว | `build()` |
| ทดสอบ form validation ที่ไม่ query DB โดยตรง | `build()` หรือแม้แต่ `stub()` |
| ทดสอบ view ที่ query ฐานข้อมูลจริง (list view, detail view) | `create()` |
| ทดสอบความสัมพันธ์ (`post.comments.all()`, `category.posts.count()`) | `create()` (จำเป็น เพราะต้องมี pk จริงในการ query ผ่านความสัมพันธ์) |
| ทดสอบ pagination/aggregation ที่ต้องมีข้อมูลจำนวนมาก (หลักร้อยขึ้นไป) | `build_batch()` + `bulk_create()` ผสมกัน |
| ต้องการความเร็วสูงสุดในการทดสอบ logic ล้วน ๆ ที่ไม่ใช่ Django ORM เลย | `stub()` |
| Unit test ทั่วไปที่ต้องการ pk จริงแต่ไม่สนความเร็ว (test เดี่ยว ๆ ไม่กี่ตัว) | `create()` — เขียนง่าย อ่านง่ายที่สุด ใช้เป็นค่าเริ่มต้นตลอดถ้าไม่มีปัญหาความเร็ว |

**หลักการโดยรวม**: อย่า optimize ก่อนที่จะรู้ว่ามีปัญหาจริง — เริ่มต้นด้วย `create()`
เสมอเพราะอ่านง่ายและปลอดภัยที่สุด แล้วค่อยเปลี่ยนไปใช้ `build()`/`bulk_create()`
เฉพาะจุดที่ทำ test suite ช้าจริง ๆ (วัดด้วย `pytest --durations=10` เพื่อดูว่า test
ไหนช้าที่สุด 10 อันดับแรก)

---

## ขั้นตอนที่ 630: สรุปและแบบฝึกหัด

### 630.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจข้อจำกัดของ `fixtures.json` แบบเดิม และเหตุผลที่ทีมมืออาชีพเปลี่ยนมาใช้
  Factory Boy สำหรับข้อมูลทดสอบ
- ✅ สร้าง `DjangoModelFactory` พื้นฐานสำหรับ `Category` และ `Post` พร้อม override
  ค่า field ตอนเรียกใช้งาน
- ✅ ใช้ `factory.SubFactory` จัดการความสัมพันธ์ ForeignKey ให้อัตโนมัติ รวมถึง
  override field ลึกด้วย syntax `field__subfield`
- ✅ ใช้ `factory.Sequence` และ `factory.LazyAttributeSequence` แก้ปัญหา
  `UNIQUE constraint` สำหรับ field เช่น `slug`, `username`, `email`
- ✅ ผสาน `Faker` เพื่อสร้างข้อมูลสมจริง รองรับ locale ภาษาไทย และเขียน custom
  provider ของตัวเองได้
- ✅ ใช้ `factory.Trait` สร้างรูปแบบสำเร็จรูปของ object (published/draft/featured)
  จาก factory class เดียว
- ✅ ใช้ `@factory.post_generation` และ `factory.RelatedFactoryList` จัดการ
  ManyToMany และความสัมพันธ์ย้อนกลับที่ `SubFactory` ทำเองไม่ได้
- ✅ ผสาน Factory Boy เข้ากับระบบ fixture ของ pytest ทั้งแบบเขียน wrapper เองและ
  ผ่าน `pytest-factoryboy` ที่สร้าง fixture ให้อัตโนมัติ
- ✅ เข้าใจความแตกต่างด้าน performance ระหว่าง `build()`, `create()`, `stub()`
  และเทคนิคผสม `build_batch()` กับ `bulk_create()` สำหรับ seed ข้อมูลจำนวนมาก

### 630.2 Checklist ก่อนไป Part ถัดไป

- [ ] ติดตั้ง `factory-boy` และ `pytest-factoryboy` ใน venv สำเร็จ
- [ ] มีไฟล์ `blog/tests/factories.py` ที่มี `CategoryFactory`, `TagFactory`,
      `PostFactory`, `CommentFactory` ครบ
- [ ] มีไฟล์ `accounts/tests/factories.py` ที่มี `UserFactory` พร้อม
      `post_generation` สำหรับ `set_password()` ที่ถูกต้อง
- [ ] `PostFactory` มี `SubFactory` เชื่อม `category` และ `author` ครบ
- [ ] `PostFactory` มี `Trait` อย่างน้อย 2 แบบ (`published`, `featured`)
- [ ] `PostFactory` มี `post_generation` สำหรับ `tags` ที่จัดการ `extracted is None`
      ถูกต้อง (ไม่ใช้ `if extracted:` เฉย ๆ)
- [ ] ลงทะเบียน factory หลักด้วย `pytest_factoryboy.register()` ใน `conftest.py`
      แล้วใช้ fixture ที่ได้มาเขียน test อย่างน้อย 3 ตัว
- [ ] เขียน test ที่พิสูจน์ความแตกต่างระหว่าง `build()` กับ `create()` ด้วย
      `django_assert_num_queries` อย่างน้อย 1 คู่

### 630.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง `ProfileFactory` สำหรับ model `Profile` (จากขั้นตอนที่
622.1) โดยต้องมี `user = factory.SubFactory(UserFactory)`, `bio` ใช้ `factory.Faker`
locale ไทย, และ `website` ใช้ `factory.Faker("url")` จากนั้นเขียน test ยืนยันว่า
`profile.user.profile == profile` (ทดสอบความสัมพันธ์ OneToOne แบบย้อนกลับผ่าน
`related_name="profile"`)

**แบบฝึกหัดที่ 2**: เพิ่ม Trait ใหม่ชื่อ `with_avatar` ให้ `ProfileFactory` จาก
แบบฝึกหัดที่ 1 ที่กำหนดค่า `avatar` ด้วย `factory.django.ImageField()` (ค้นหาวิธีใช้
`factory.django.ImageField` จากเอกสารทางการของ Factory Boy — เป็น field type พิเศษ
สำหรับสร้างไฟล์รูปภาพปลอมให้ `ImageField`/`FileField` โดยเฉพาะ) แล้วเขียน test
ยืนยันว่า `profile.avatar.name` ไม่ว่างเปล่าเมื่อเปิด trait นี้

**แบบฝึกหัดที่ 3**: ลงทะเบียน `ProfileFactory` และ `CommentFactory` ทั้งคู่ด้วย
`pytest_factoryboy.register()` แล้วเขียน test ที่ใช้ fixture `profile`, `comment`,
`user` พร้อมกันเพื่อยืนยันว่าถ้า override `profile__user` และ `comment__author`
ให้เป็น user คนเดียวกัน (ผ่าน fixture ที่ประกาศทับใน `conftest.py`) ทั้งสอง object
จะอ้างถึง `CustomUser` instance เดียวกันจริง

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียนฟังก์ชัน seed data สำหรับ management command
(ทบทวนวิธีเขียน custom management command จาก Part ก่อนหน้าที่เกี่ยวข้อง) ชื่อ
`seed_blog_demo_data` ที่ใช้ `PostFactory.create_batch(50, category=None)` ร่วมกับ
`CategoryFactory.create_batch(5)` วนสุ่ม assign category ให้แต่ละ post ด้วย
`random.choice()` และวัดเวลาที่ใช้ด้วย `time.perf_counter()` เปรียบเทียบระหว่างการ
เรียก `create_batch()` ตรง ๆ กับการใช้เทคนิค `build_batch()` + `Post.objects.bulk_create()`
จากขั้นตอนที่ 629.5 บันทึกผลเวลาที่ต่างกันเป็นตัวเลขจริงลงในไฟล์ `notes.md`

### 630.4 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเลิกใช้ fixture ธรรมดาจาก Part 061 ทั้งหมดแล้วเปลี่ยนมาใช้
`pytest-factoryboy` เท่านั้นหรือไม่?**
A: ไม่จำเป็น ทั้งสองแนวทางอยู่ร่วมกันได้ในโปรเจกต์เดียวกันตลอดไป — fixture ธรรมดา
ยังเหมาะกับ setup ที่ซับซ้อนเกินกว่าจะเป็นแค่ "สร้าง model instance" (เช่น เตรียม
ไฟล์ชั่วคราว, เชื่อมต่อ external service จำลอง) ส่วน `pytest-factoryboy` เหมาะกับ
"ต้องการ model instance ที่มีความสัมพันธ์ซับซ้อน" เป็นหลัก เลือกใช้ตามความเหมาะสม
ของแต่ละ test ไม่ต้องเลือกอย่างใดอย่างหนึ่งทั้งโปรเจกต์

**Q: ทำไม `django_get_or_create = ("username",)` ใน `UserFactory` ถึงสำคัญ?**
A: ถ้าไม่ใส่ `django_get_or_create` ทุกครั้งที่เรียก `UserFactory(username="somchai")`
ซ้ำสองครั้ง จะได้ `IntegrityError` เพราะ `username` เป็น unique field และ Factory Boy
โดยปกติเรียก `.create()` (คือ `objects.create()`) เสมอ ไม่ใช่ `get_or_create()` การใส่
`django_get_or_create = ("username",)` บอก Factory Boy ให้ใช้
`CustomUser.objects.get_or_create(username=..., defaults={...})` แทน ทำให้เรียกซ้ำ
ด้วย username เดิมได้โดยไม่ error (คืน instance เดิมกลับมาแทนที่จะสร้างใหม่)

**Q: ควรเก็บไฟล์ `factories.py` ไว้ที่ไหนในโครงสร้างโปรเจกต์?**
A: แนวทางที่นิยมที่สุดคือเก็บไว้ใน `<app_name>/tests/factories.py` ของแต่ละแอป
(ตามที่ทำใน Part นี้) เพราะทำให้ factory อยู่ใกล้กับ model และ test ที่เกี่ยวข้อง
บางทีมขนาดใหญ่จะแยกเป็นแพ็กเกจ `testutils/factories/` กลางที่ import ได้จากทุกแอป
เมื่อมี factory ที่ใช้ร่วมกันข้ามแอปจำนวนมาก — เลือกตามขนาดของทีมและโปรเจกต์

**Q: `factory.Faker` กับ `factory.Sequence` ใช้แทนกันได้ไหมสำหรับ field ที่ต้อง
unique อย่าง `email`?**
A: `factory.Faker("email")` ให้อีเมลสุ่มจากช่วงข้อมูลจำนวนมาก โอกาสชนกันในทางปฏิบัติ
ต่ำมากสำหรับ test suite ขนาดทั่วไป (หลักพันตัวลงมา) แต่ **ไม่ได้รับประกัน 100%** ว่า
จะไม่ซ้ำ ถ้า field นั้นมี `unique=True` ในระดับฐานข้อมูลจริง และ test suite ของคุณมี
ขนาดใหญ่มาก (สร้าง user นับหมื่นตัวในการรันเดียว) แนะนำให้ผสมทั้งคู่ด้วย
`factory.LazyAttributeSequence` เพื่อความชัวร์ 100% เช่น
`email = factory.LazyAttributeSequence(lambda o, n: f"user{n}@example.com")`

**Q: ทำไม `PostFactory().tags.count()` ในขั้นตอนที่ 627 ได้ 2 เสมอ ทั้งที่ควรจะสุ่ม?**
A: เพราะ `post_generation` hook ที่เขียนไว้กำหนดค่า **คงที่** ไว้ที่ 2 แท็ก
(`self.tags.add(TagFactory(), TagFactory())`) เมื่อไม่มีการส่ง `tags=...` เข้ามา — ถ้า
ต้องการจำนวนแท็กแบบสุ่มหรือปรับได้ ให้เพิ่ม parameter ควบคุมจำนวนเอง เช่น รับ
`kwargs.get("count", 2)` ภายใน hook แล้วเรียก `PostFactory(tags__count=5)` ผ่าน
double-underscore syntax เดียวกับที่ใช้กับ `SubFactory`

### 630.5 เตรียมตัวสำหรับ Part ถัดไป

**Part 064: Integration Testing และ Selenium** จะพาคุณก้าวข้ามการทดสอบระดับ unit/
view ไปสู่การทดสอบที่จำลองพฤติกรรมผู้ใช้จริงในเบราว์เซอร์ ตั้งแต่การตั้งค่า
`LiveServerTestCase`/`live_server` fixture (ที่ปรากฏใน Part 061 ตารางขั้นตอนที่
607.5) ไปจนถึงการควบคุมเบราว์เซอร์อัตโนมัติด้วย **Selenium WebDriver** ทดสอบการ
คลิก, กรอกฟอร์ม, และตรวจสอบ JavaScript ที่ทำงานฝั่ง client จริง ๆ

Factory ทั้งหมดที่คุณสร้างไว้ใน Part นี้จะถูกนำกลับมาใช้ทันทีใน Part 064 เพื่อ
seed ข้อมูลก่อนเปิดเบราว์เซอร์จำลองแต่ละครั้ง (เช่น `PostFactory.create_batch(20,
published=True)` ก่อนทดสอบว่าหน้า pagination ทำงานถูกต้องเมื่อคลิก "หน้าถัดไป" จริง
บนเบราว์เซอร์) เตรียมทบทวนโครงสร้าง `blog/tests/factories.py` ที่เพิ่งสร้างเสร็จให้
พร้อม เพราะจะเป็นรากฐานสำคัญของ integration test ตลอด Part ถัดไป
