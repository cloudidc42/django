# Part 017: Django Admin เบื้องต้น: ModelAdmin

> **ขั้นตอนที่ 161-170 ของหลักสูตร** | Phase 2: Models, ORM และ Admin
>
> เป้าหมายของ Part นี้: เปลี่ยนโมเดลทั้ง 6 ตัวที่คุณสร้างและปรับแต่งมาตลอด Part 011-016
> (`Category`, `Tag`, `Post`, `PostTag`, `Comment`, `Profile`) ให้กลายเป็น **แผงควบคุม
> จัดการเนื้อหา (Content Management Dashboard)** ที่ใช้งานได้จริงระดับมืออาชีพ โดยไม่ต้อง
> เขียน HTML หรือ view สักบรรทัดเดียว คุณจะได้เรียนรู้ `admin.site.register()` แบบพื้นฐาน
> ไปจนถึง `ModelAdmin` แบบเจาะลึก: การจัดคอลัมน์และตัวกรองในหน้ารายการ
> (`list_display`, `list_filter`, `search_fields`, `list_editable`), การจัดกลุ่มฟอร์มให้
> อ่านง่ายด้วย `fieldsets`/`readonly_fields`, ทางลัดสำหรับข้อมูลเยอะด้วย
> `prepopulated_fields`/`raw_id_fields`/`autocomplete_fields`, การแสดงข้อมูลลูกแบบ Inline
> (`TabularInline`/`StackedInline`), การกำหนดฟอร์ม Admin เองพร้อม custom validation,
> ไปจนถึง widget พิเศษสำหรับ ManyToMany อย่าง `filter_horizontal` — พร้อมเจอ **ข้อจำกัดจริง
> ของ Django Admin กับ M2M ที่มี `through` model กำหนดเอง** ซึ่งเป็นกับดักที่มือใหม่เกือบ
> ทุกคนเจอ เมื่อจบ Part นี้ คุณจะมี Admin Site ที่สมบูรณ์ ใช้งานง่าย และปลอดภัยพอจะส่งมอบ
> ให้ทีม content จัดการบล็อกได้จริงโดยไม่ต้องพึ่งโปรแกรมเมอร์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 161: ลงทะเบียน Model เข้า Django Admin และสร้าง Superuser (Recap)
- ขั้นตอนที่ 162: `ModelAdmin` เจาะลึก: `list_display`, `list_filter`, `search_fields`
- ขั้นตอนที่ 163: ปรับแต่งหน้ารายการ: `list_editable`, `list_display_links`, `ordering`, `list_per_page`
- ขั้นตอนที่ 164: จัดฟอร์มให้อ่านง่ายด้วย `fieldsets`, `fields`, `readonly_fields`
- ขั้นตอนที่ 165: `prepopulated_fields` และ `raw_id_fields` สำหรับ Foreign Key ข้อมูลเยอะ
- ขั้นตอนที่ 166: Inline Admin — แสดง `Comment` และ `PostTag` เป็น Inline ด้วย `TabularInline`/`StackedInline`
- ขั้นตอนที่ 167: กำหนดฟอร์ม Admin เอง (`form = CustomModelForm`) และ Custom Validation
- ขั้นตอนที่ 168: `filter_horizontal`/`filter_vertical` สำหรับ `ManyToManyField`
- ขั้นตอนที่ 169: `date_hierarchy` และ `autocomplete_fields`
- ขั้นตอนที่ 170: สรุปและแบบฝึกหัด — Admin Site ที่สมบูรณ์สำหรับบล็อกทั้งระบบ

---

## ขั้นตอนที่ 161: ลงทะเบียน Model เข้า Django Admin และสร้าง Superuser (Recap)

### 161.1 ทบทวนสถานะโปรเจกต์ก่อนเริ่ม Part นี้

หลังจาก Part 011-016 โปรเจกต์บล็อกของคุณควรมีโมเดลครบ 6 ตัวใน 2 แอป ดังนี้:

| แอป | โมเดล | ความสัมพันธ์สำคัญ |
|---|---|---|
| `blog` | `Category` | มี `Post` หลายตัวอ้างอิงกลับมา (`related_name='posts'`) |
| `blog` | `Tag` | เชื่อมกับ `Post` แบบ M2M ผ่าน `PostTag` |
| `blog` | `Post` | `category` (FK, `SET_NULL`), `tags` (M2M ผ่าน `through='PostTag'`) |
| `blog` | `PostTag` | Through model เก็บ `added_at`, `added_by` |
| `blog` | `Comment` | `post` (FK, `CASCADE`), `parent` (self-FK สำหรับตอบกลับ) |
| `accounts` | `Profile` | `user` (`OneToOneField` ไปยัง `settings.AUTH_USER_MODEL`) |

Schema ทั้งหมดนี้ผ่านการ migrate เรียบร้อยแล้ว และ Part 016 ได้พาคุณไปเจาะลึกเรื่องการ
จัดการ migration ขั้นสูง (squash, data migration, การแก้ conflict) จนมั่นใจได้ว่าฐานข้อมูล
ของคุณสอดคล้องกับโค้ด `models.py` 100% เป๊ะ — ถึงเวลาที่จะเปิดให้ทีมงานที่ไม่ใช่โปรแกรมเมอร์
(บรรณาธิการ, ผู้ดูแลระบบ, การตลาด) สามารถเข้ามา**เพิ่ม/แก้/ลบ**ข้อมูลเหล่านี้ได้เองผ่านหน้าเว็บ
โดยไม่ต้องเปิด `python manage.py shell` ทุกครั้ง

### 161.2 Django Admin คืออะไร และทำไมถึงพิเศษ

ย้อนกลับไปที่ Part 001 ขั้นตอนที่ 2 เราพูดถึงปรัชญา **"Batteries Included"** ของ Django และ
ระบุว่า "ระบบ Admin Panel อัตโนมัติ" คือหนึ่งในฟีเจอร์ที่มีมาให้ในตัว — นี่คือจุดที่เราจะเจาะลึก
ฟีเจอร์นั้นอย่างเต็มรูปแบบ

**`django.contrib.admin`** เป็นแอปที่ Django สร้างมาให้ตั้งแต่ `startproject` ครั้งแรก มันจะ
**อ่านโครงสร้างของ Model** (field, type, ความสัมพันธ์) แล้ว **สร้างหน้าเว็บ CRUD
(Create-Read-Update-Delete) ให้อัตโนมัติ** โดยที่คุณไม่ต้องเขียน view, template หรือ form
เองเลยสักบรรทัด ต่างจาก framework อื่นอย่าง Flask หรือ Express.js ที่ถ้าอยากได้หน้า admin
ต้องไปติดตั้ง library เพิ่มเอง (เช่น Flask-Admin) หรือเขียนเองทั้งหมด

ตรวจสอบว่าโปรเจกต์ของคุณมี admin เปิดใช้งานอยู่แล้วหรือไม่ (ควรมีมาตั้งแต่สร้างโปรเจกต์):

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',      # ← แอป admin
    'django.contrib.auth',       # ← ระบบผู้ใช้ (admin ต้องพึ่งพาสิ่งนี้)
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
    'accounts',
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),   # ← เส้นทางเข้าสู่ admin
    # ... urlpatterns อื่น ๆ ...
]
```

ทั้งสองส่วนนี้ Django สร้างให้อัตโนมัติตั้งแต่รัน `django-admin startproject` และเราไม่เคยต้อง
แก้ไขมันเลยตลอด 16 Part ที่ผ่านมา

### 161.3 ทบทวนการสร้าง Superuser

หน้า admin ต้อง **login ด้วยบัญชีที่มีสิทธิ์ `is_staff=True`** เท่านั้น บัญชีที่มีสิทธิ์สูงสุด
(เข้าถึงทุกโมเดล ทุกสิทธิ์) เรียกว่า **superuser** สร้างได้ด้วยคำสั่ง:

```bash
python manage.py createsuperuser
```

```
Username: admin
Email address: admin@example.com
Password:
Password (again):
Superuser created successfully.
```

> **หมายเหตุ**: `createsuperuser` จะตรวจสอบรหัสผ่านผ่าน `AUTH_PASSWORD_VALIDATORS`
> เช่นเดียวกับการสมัครสมาชิกทั่วไป ถ้ารหัสผ่านสั้นเกินไปหรือเดาง่ายเกินไป (เช่น
> `password123`) Django จะเตือนและถามยืนยันอีกครั้งว่าต้องการใช้รหัสผ่านที่ไม่ปลอดภัยจริง ๆ
> หรือไม่ — ระบบ Authentication เต็มรูปแบบจะเรียนใน **Part 031**

รันเซิร์ฟเวอร์และเปิดหน้า admin:

```bash
python manage.py runserver
```

เปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/admin/` ล็อกอินด้วยบัญชีที่เพิ่งสร้าง คุณจะเห็น
หน้า Django Admin เริ่มต้นที่มีเพียง 2 ส่วน: **Authentication and Authorization** (จัดการ
`Users` และ `Groups`) เพราะ `django.contrib.auth` ลงทะเบียน model ของตัวเองไว้ให้อัตโนมัติ
แล้ว — แต่ `Category`, `Tag`, `Post`, `Comment`, `Profile` ของเรายัง**ไม่ปรากฏเลย** เพราะ
ยังไม่ได้ลงทะเบียน

### 161.4 ลงทะเบียน Model แบบพื้นฐานด้วย `admin.site.register()`

ทุกแอปที่ต้องการให้ model ปรากฏใน admin ต้องมีไฟล์ `admin.py` (Django สร้างไฟล์เปล่านี้ให้
อัตโนมัติทุกครั้งที่ `startapp`) แล้วเรียก `admin.site.register(ModelClass)`:

```python
# blog/admin.py
from django.contrib import admin

from .models import Category, Tag, Post, Comment

admin.site.register(Category)
admin.site.register(Tag)
admin.site.register(Post)
admin.site.register(Comment)
```

```python
# accounts/admin.py
from django.contrib import admin

from .models import Profile

admin.site.register(Profile)
```

> **สังเกต**: เรา**ยังไม่ลงทะเบียน `PostTag`** ในขั้นตอนนี้โดยตั้งใจ เพราะ `PostTag` เป็น
> through model ของความสัมพันธ์ `Post.tags` ซึ่งเราจะจัดการมันผ่าน **Inline Admin** แทนใน
> ขั้นตอนที่ 166 (ฝัง `PostTag` ไว้ในหน้าแก้ไข `Post` โดยตรง จะสะดวกกว่าเปิดหน้าแยกต่างหาก
> มาก)

รีเฟรชหน้า `/admin/` — ตอนนี้คุณจะเห็นส่วน **Blog** และ **Accounts** ปรากฏขึ้นมา พร้อมลิงก์
ไปยัง `Categorys`, `Tags`, `Posts`, `Comments`, `Profiles` (สังเกตคำว่า "Categorys" สะกดผิด
หลักไวยกรณ์ — ปัญหานี้เราแก้ไปแล้วด้วย `verbose_name_plural = 'categories'` ใน Part 012
ขั้นตอนที่ 111.3 ถ้ายังไม่ได้ใส่ ให้กลับไปตรวจสอบ `Category.Meta` ของคุณอีกครั้ง)

คลิกเข้าไปที่ `Posts` — คุณจะเห็นรายการบทความทั้งหมดที่แสดงผลด้วย `__str__()` ของแต่ละ
object (เช่น ชื่อบทความ) เพราะเรากำหนด `__str__()` ไว้ให้ทุกโมเดลตั้งแต่ Part 011-012 แล้ว
— นี่คือเหตุผลที่กฎ **"ทุก Model ต้องมี `__str__()` เสมอ"** ที่เราเน้นย้ำมาตลอดสำคัญมาก
ถ้าไม่มี `__str__()` แถวในรายการจะแสดงเป็น `Post object (1)`, `Post object (2)` ซึ่งไม่มี
ประโยชน์อะไรเลยกับผู้ใช้งานจริง

### 161.5 ทางเลือกที่นิยมกว่า: Decorator `@admin.register()`

`admin.site.register(Model)` แบบข้างบนใช้งานได้ แต่เมื่อไหร่ก็ตามที่เราต้องการปรับแต่ง
พฤติกรรมของ admin (ซึ่งเราจะทำตลอดทั้ง Part นี้) เราต้องสร้างคลาสที่สืบทอดจาก
**`admin.ModelAdmin`** แล้วส่งเข้าไปเป็นพารามิเตอร์ที่สอง:

```python
from django.contrib import admin
from .models import Tag


class TagAdmin(admin.ModelAdmin):
    pass


admin.site.register(Tag, TagAdmin)
```

Django มีทางลัดที่สั้นและอ่านง่ายกว่าโดยใช้ **decorator `@admin.register()`** ซึ่งทำสิ่ง
เดียวกันทุกประการ:

```python
from django.contrib import admin
from .models import Tag


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    pass
```

`@admin.register()` รับ model ได้มากกว่า 1 ตัวพร้อมกันด้วย ถ้าหลาย model ใช้ `ModelAdmin`
class เดียวกัน:

```python
@admin.register(Category, Tag)
class SimpleNameAdmin(admin.ModelAdmin):
    """ใช้ร่วมกันสำหรับโมเดลที่มีแค่ field name/slug ธรรมดา"""
    list_display = ('name',)
```

| ประเด็น | `admin.site.register(Model, Admin)` | `@admin.register(Model)` |
|---|---|---|
| รูปแบบ | เรียก function แยกจากการประกาศคลาส | Decorator ผูกกับคลาสโดยตรง |
| ความอ่านง่าย | ต้องเลื่อนตาไปดูว่า class ไหนคู่กับ model ไหน | เห็นชัดเจนในบรรทัดเดียวกัน |
| ใช้กับหลาย Model พร้อมกัน | เขียนเรียก `register()` หลายครั้ง | ใส่หลาย model ใน decorator เดียวได้ |
| มาตรฐานที่ใช้ในหลักสูตรนี้ต่อจากนี้ | - | ✅ ใช้ตลอดตั้งแต่ขั้นตอนที่ 162 เป็นต้นไป |

ตั้งแต่ขั้นตอนถัดไป เราจะเปลี่ยนมาใช้ `@admin.register()` ทั้งหมด เพราะทุก model ของเราจะมี
`ModelAdmin` ที่ปรับแต่งเฉพาะตัว

### 161.6 ข้อผิดพลาดที่พบบ่อยตอนเริ่มต้น

| ข้อผิดพลาด | สาเหตุ | วิธีแก้ |
|---|---|---|
| `django.contrib.admin.sites.AlreadyRegistered: The model Post is already registered` | เรียก `register(Post)` ซ้ำสองครั้ง (เช่น เขียนไว้ทั้งแบบ function และแบบ decorator) | ลบการเรียกที่ซ้ำออก เหลือแค่ทางเดียว |
| Model ไม่ปรากฏใน `/admin/` เลย | ลืม import model ใน `admin.py` หรือลืมเพิ่มแอปใน `INSTALLED_APPS` | ตรวจสอบทั้งสองจุด แล้ว restart dev server |
| หน้า `/admin/login/` ขึ้น "Please enter the correct username and password" ตลอด | ใช้บัญชีที่ `is_staff=False` (เช่น user ทั่วไปที่สร้างผ่าน `create_user()` ไม่ใช่ `create_superuser()`) | เข้า shell แล้วตั้ง `user.is_staff = True; user.save()` หรือสร้าง superuser ใหม่ |
| แถวในรายการแสดง `Post object (1)` | ลืมเขียน `__str__()` ใน model | เพิ่ม `def __str__(self): return self.title` |

---

## ขั้นตอนที่ 162: `ModelAdmin` เจาะลึก: `list_display`, `list_filter`, `search_fields`

### 162.1 ปัญหาของหน้ารายการ (Changelist) แบบ Default

เปิดหน้า `Posts` ใน admin ตอนนี้ — คุณจะเห็นแค่**คอลัมน์เดียว** ที่เป็นผลลัพธ์ของ
`__str__()` เท่านั้น ถ้ามีบทความ 500 บทความ คุณจะไม่มีทางรู้เลยว่าบทความไหนเผยแพร่แล้ว
บทความไหนอยู่หมวดไหน หรือถูกสร้างเมื่อไหร่ โดยไม่ต้องคลิกเข้าไปดูทีละอัน — นี่คือปัญหาที่
`ModelAdmin` ถูกออกแบบมาแก้โดยเฉพาะ

### 162.2 `list_display`: เพิ่มคอลัมน์ในหน้ารายการ

```python
# blog/admin.py
from django.contrib import admin

from .models import Category, Tag, Post, Comment


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
```

`list_display` รับ **tuple ของชื่อ field หรือชื่อ method** ก็ได้ ไม่จำกัดแค่ field ตรง ๆ
ตัวอย่างการเพิ่มคอลัมน์ที่คำนวณจาก method ของ `ModelAdmin` เอง เช่น จำนวนคอมเมนต์:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')

    @admin.display(description='จำนวนคอมเมนต์')
    def comment_count(self, obj):
        return obj.comments.count()
```

`@admin.display(description=...)` คือ decorator ที่ Django เพิ่มเข้ามาตั้งแต่เวอร์ชัน 3.2
เพื่อกำหนดหัวคอลัมน์ให้อ่านง่าย (ก่อนหน้านั้นต้องเขียน
`comment_count.short_description = 'จำนวนคอมเมนต์'` แยกต่างหาก ซึ่งยังใช้ได้อยู่แต่ไม่ใช่
วิธีที่แนะนำอีกต่อไป) นอกจาก `description` ยังตั้ง `boolean=True` (แสดงเป็นไอคอน ✅/❌
แทนข้อความ True/False) และ `ordering=` (ให้คลิกหัวคอลัมน์เพื่อ sort ได้) ได้ด้วย:

```python
@admin.display(description='เผยแพร่แล้ว', boolean=True)
def published_status(self, obj):
    return obj.is_published
```

> **คำเตือนเรื่อง Performance**: `comment_count` ข้างบนเรียก `obj.comments.count()` แยก
> query ทุกแถวในหน้ารายการ ถ้ามี 25 บทความต่อหน้า (ค่า default) จะเกิดปัญหา **N+1 Query**
> ทันที (ทบทวนจาก Part 012 ขั้นตอนที่ 117) วิธีบรรเทาปัญหานี้ในระดับ admin คือกำหนด
> **`list_select_related`** สำหรับ FK (เทียบเท่า `select_related()` ใน queryset) และเรียก
> `prefetch_related()` เองผ่านการ override `get_queryset()`:
>
> ```python
> @admin.register(Post)
> class PostAdmin(admin.ModelAdmin):
>     list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')
>     list_select_related = ('category',)
>
>     def get_queryset(self, request):
>         qs = super().get_queryset(request)
>         return qs.prefetch_related('comments')
>
>     @admin.display(description='จำนวนคอมเมนต์')
>     def comment_count(self, obj):
>         return obj.comments.count()
> ```
>
> การเจาะลึกเรื่อง query optimization แบบเต็มรูปแบบ (รวมถึง `annotate(Count(...))` ที่ทำงาน
> ได้เร็วกว่าการวน `.count()` ต่อแถวมาก) จะอยู่ใน **Part 067**

### 162.3 `list_filter`: กรองข้อมูลด้วยแถบด้านข้าง

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    list_filter = ('is_published', 'category', 'created_at')
```

Django จะสร้าง **sidebar ตัวกรอง** ที่ด้านขวาของหน้ารายการให้อัตโนมัติ โดยพฤติกรรมของ
ตัวกรองขึ้นอยู่กับชนิดของ field:

| ชนิด Field | พฤติกรรมตัวกรองที่ได้ |
|---|---|
| `BooleanField` (`is_published`) | ตัวเลือก "All / Yes / No" (และ "Unknown" ถ้า `null=True`) |
| `ForeignKey` (`category`) | รายชื่อ `Category` ทุกตัวที่**มีอยู่จริง**ในข้อมูล ให้เลือกกรองทีละหมวด |
| `DateTimeField`/`DateField` (`created_at`) | ตัวเลือกสำเร็จรูป "Any date / Today / Past 7 days / This month / This year" |
| `CharField` ที่มี `choices=` | รายการตัวเลือกทั้งหมดจาก `choices` |

สำหรับตัวกรองที่ซับซ้อนกว่านี้ (เช่น "บทความที่มีคอมเมนต์อย่างน้อย 5 อัน") ต้องเขียน
**`admin.SimpleListFilter`** แบบกำหนดเอง ซึ่งเป็นหัวข้อของ **Part 018: Django Admin
ขั้นสูง** — Part นี้เราจะใช้แค่ `list_filter` แบบมาตรฐานที่ครอบคลุมงานส่วนใหญ่ในชีวิตจริง
ได้แล้ว

### 162.4 `search_fields`: ช่องค้นหาด้านบน

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    list_filter = ('is_published', 'category', 'created_at')
    search_fields = ('title', 'content', 'category__name')
```

สังเกตว่า `category__name` ใช้ **double underscore เดินข้ามความสัมพันธ์** แบบเดียวกับที่
เรียนใน Part 012 ขั้นตอนที่ 116 ทุกประการ — ค้นหา "เทคโนโลยี" ในช่องค้นหาจะเจอบทความทุก
บทความที่อยู่ในหมวดหมู่ชื่อนั้น แม้คำว่า "เทคโนโลยี" จะไม่ปรากฏใน title/content เลยก็ตาม

`search_fields` รองรับ **prefix พิเศษ** ที่เปลี่ยนวิธีเปรียบเทียบข้อความ:

| Prefix | ความหมาย | เทียบเท่า Lookup | ตัวอย่าง |
|---|---|---|---|
| (ไม่มี prefix) | ค้นหาแบบ contains ไม่สนตัวพิมพ์เล็ก-ใหญ่ | `icontains` | `'title'` |
| `^` | ต้องขึ้นต้นด้วยคำค้นหา | `istartswith` | `'^title'` |
| `=` | ต้องตรงทั้งหมดเป๊ะ (ไม่สนตัวพิมพ์) | `iexact` | `'=slug'` |
| `@` | Full-text search (**เฉพาะ PostgreSQL** เท่านั้น) | `search` | `'@content'` |

`^` มีประโยชน์มากเมื่อค้นหาในตารางที่มีข้อมูลจำนวนมาก เพราะ `istartswith` ใช้ index ของ
ฐานข้อมูลได้อย่างมีประสิทธิภาพกว่า `icontains` ซึ่งต้องสแกนทั้งข้อความ (`LIKE '%คำ%'`
ไม่สามารถใช้ B-tree index ได้เต็มประสิทธิภาพ) — เราจะกลับมาพูดเรื่อง index และ query
performance แบบเต็มใน Part 067 เช่นกัน

### 162.5 นำมารวมกันสำหรับทุก Model ที่มีในตอนนี้

```python
# blog/admin.py
from django.contrib import admin

from .models import Category, Tag, Post, Comment


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    prepopulated_fields = {'slug': ('name',)}  # เจาะลึกในขั้นตอนที่ 165


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')
    list_filter = ('is_published', 'category', 'created_at')
    search_fields = ('title', 'content', 'category__name')
    list_select_related = ('category',)

    def get_queryset(self, request):
        return super().get_queryset(request).prefetch_related('comments')

    @admin.display(description='จำนวนคอมเมนต์')
    def comment_count(self, obj):
        return obj.comments.count()


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('author', 'post', 'parent', 'created_at')
    list_filter = ('created_at',)
    search_fields = ('author', 'content', 'post__title')
```

ทดลองเปิดหน้า `Posts` และ `Comments` อีกครั้ง — ตอนนี้คุณจะเห็นตารางข้อมูลที่มีคอลัมน์
หลายคอลัมน์ ตัวกรองด้านขวา และช่องค้นหาด้านบน ใช้งานได้ใกล้เคียงระบบ CMS เชิงพาณิชย์แล้ว
ทั้งหมดนี้เขียนโค้ดไปแค่ประมาณ 30 บรรทัด

> **หมายเหตุ**: `CommentAdmin` ที่ลงทะเบียนไว้ตรงนี้จะยังใช้งานได้ควบคู่ไปกับการแสดง
> `Comment` เป็น **inline** ภายในหน้าแก้ไข `Post` ที่เราจะทำในขั้นตอนที่ 166 — Django ไม่มี
> ปัญหาอะไรถ้า model เดียวกันถูกทั้งลงทะเบียนแบบเดี่ยว (สำหรับงาน "ดูคอมเมนต์ทั้งระบบเพื่อ
> ควบคุมสแปม") และถูกฝังเป็น inline (สำหรับงาน "ดูคอมเมนต์ของบทความนี้บทความเดียว")
> พร้อมกันได้

---

## ขั้นตอนที่ 163: ปรับแต่งหน้ารายการ: `list_editable`, `list_display_links`, `ordering`, `list_per_page`

### 163.1 `list_editable`: แก้ไขค่าตรงจากหน้ารายการโดยไม่ต้องเปิดหน้าแก้ไข

งานที่พบบ่อยมากคือ "เปิด/ปิดการเผยแพร่บทความหลาย ๆ บทความพร้อมกัน" ถ้าต้องคลิกเข้าไปทีละ
บทความจะเสียเวลามาก `list_editable` ช่วยให้แก้ไขค่าได้ตรงจากหน้ารายการเลย:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    list_editable = ('is_published',)
```

ลองรันดูตรง ๆ แบบนี้ Django จะปฏิเสธทันทีตั้งแต่ `python manage.py check` ด้วย error ที่ชื่อ
**`admin.E124`**:

```
ERRORS:
<class 'blog.admin.PostAdmin'>:
    (admin.E124) The value of 'list_editable[0]' refers to the first field
    in 'list_display' ('title'), which cannot be used unless
    'list_display_links' is set.
```

**เหตุผล**: โดย default คอลัมน์แรกใน `list_display` จะเป็น**ลิงก์คลิกเพื่อเข้าหน้าแก้ไข**
เสมอ ถ้า field เดียวกันถูกตั้งให้เป็นทั้งลิงก์และทั้งช่องแก้ไขตรง ๆ พร้อมกัน (คลิกได้ + พิมพ์
ได้ในช่องเดียวกัน) จะสร้างความกำกวมในการโต้ตอบ (UX) ดังนั้น field ที่จะอยู่ใน
`list_editable` **ต้องไม่ใช่คอลัมน์แรก** — ต้องกำหนด `list_display_links` แยกออกมาก่อน

### 163.2 `list_display_links`: กำหนดคอลัมน์ที่ใช้คลิกเข้าหน้าแก้ไข

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    list_display_links = ('title',)
    list_editable = ('is_published',)
```

ตอนนี้ `title` ยังคงเป็นลิงก์คลิกเข้าหน้าแก้ไขเหมือนเดิม (ระบุไว้ชัดเจนใน
`list_display_links`) แต่คอลัมน์ `is_published` กลายเป็น **checkbox ที่กดติ๊กได้ทันที** ใน
หน้ารายการ พร้อมปุ่ม **"Save"** ที่ปรากฏขึ้นด้านล่างตารางเมื่อมีการแก้ไข — เลือกติ๊ก/ปลดติ๊ก
หลายแถวพร้อมกันแล้วกด Save ครั้งเดียวได้เลย ประหยัดเวลากว่าคลิกเข้าไปทีละบทความมาก

`list_display_links` ยังตั้งเป็น **หลายคอลัมน์พร้อมกัน** ได้ (ทุกคอลัมน์ที่ระบุจะกลายเป็น
ลิงก์) หรือตั้งเป็น `None` เพื่อ**ปิดลิงก์ทั้งหมด** (ต้องมี `list_editable` ครอบคลุมทุก field
ที่ต้องการแก้ไข ไม่งั้นจะไม่มีทางเข้าไปแก้ field อื่นได้เลย — ใช้ได้เฉพาะกรณีพิเศษจริง ๆ
เท่านั้น)

### 163.3 `ordering`: ควบคุมลำดับการแสดงผลเฉพาะใน Admin

`Post.Meta.ordering = ['-created_at']` ที่ตั้งไว้ตั้งแต่ Part 011 มีผลกับการ query ทั้งระบบ
(รวมถึงหน้าเว็บจริงที่จะสร้างใน Part 020 เป็นต้นไป) แต่บางครั้งหน้า admin อยากเรียงคนละแบบ
จากหน้าเว็บจริง (เช่น ทีม content อยากดูบทความที่ **ยังไม่เผยแพร่ก่อนเสมอ** เพื่อรีบตรวจ) —
`ModelAdmin.ordering` **override** ค่าจาก `Meta.ordering` เฉพาะตอนแสดงในหน้า admin เท่านั้น:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    ordering = ('is_published', '-created_at')
    # เรียงบทความที่ is_published=False (0) มาก่อน is_published=True (1)
    # แล้วเรียงตามวันที่สร้างล่าสุดก่อนภายในกลุ่มเดียวกัน
```

ผู้ใช้ admin ยังคลิกที่หัวคอลัมน์เพื่อ sort ชั่วคราวได้เองเสมอ (ไม่ต้องตั้งค่าอะไรเพิ่ม)
`ordering` ที่กำหนดใน `ModelAdmin` เป็นแค่**ค่าเริ่มต้น**ก่อนผู้ใช้จะคลิกเปลี่ยนเอง

### 163.4 `list_per_page` และ `list_max_show_all`

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at')
    list_per_page = 25       # ค่า default คือ 100
    list_max_show_all = 200  # ค่า default คือ 200
```

- **`list_per_page`**: จำนวนแถวต่อหน้า ค่า default ของ Django คือ 100 ซึ่งมากเกินไปสำหรับ
  ตารางที่มีคอลัมน์เยอะหรือมี query หนักต่อแถว (เช่น `comment_count` ที่เราเพิ่มไปก่อนหน้านี้)
  — ลดเหลือ 20-25 มักเหมาะกับการใช้งานจริงมากกว่า
- **`list_max_show_all`**: ถ้าจำนวนแถวทั้งหมดไม่เกินค่านี้ Django จะแสดงลิงก์ **"Show all"**
  ให้กดดูทุกแถวในหน้าเดียว ถ้าตารางมีข้อมูลมาก (เช่นหลักหมื่นแถว) ควรตั้งค่านี้ให้ต่ำ เพื่อ
  ป้องกันไม่ให้ผู้ใช้กดปุ่มนี้แล้วทำให้ query ดึงข้อมูลทั้งหมดมาแสดงพร้อมกันจนเซิร์ฟเวอร์ค้าง

### 163.5 ตัวอย่างรวมสำหรับ `PostAdmin` ณ จุดนี้

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')
    list_display_links = ('title',)
    list_editable = ('is_published',)
    list_filter = ('is_published', 'category', 'created_at')
    search_fields = ('title', 'content', 'category__name')
    list_select_related = ('category',)
    ordering = ('is_published', '-created_at')
    list_per_page = 25

    def get_queryset(self, request):
        return super().get_queryset(request).prefetch_related('comments')

    @admin.display(description='จำนวนคอมเมนต์')
    def comment_count(self, obj):
        return obj.comments.count()
```

### 163.6 ตารางสรุปตัวเลือกระดับ "หน้ารายการ" ทั้งหมดที่เรียนไปแล้ว

| ตัวเลือก | หน้าที่ | ค่า default |
|---|---|---|
| `list_display` | กำหนดคอลัมน์ที่แสดง | `('__str__',)` |
| `list_display_links` | กำหนดคอลัมน์ที่คลิกแล้วไปหน้าแก้ไข | คอลัมน์แรกใน `list_display` |
| `list_editable` | field ที่แก้ไขได้ตรงจากหน้ารายการ | ไม่มี |
| `list_filter` | ตัวกรองด้านข้าง | ไม่มี |
| `search_fields` | ช่องค้นหาด้านบน | ไม่มี (ไม่แสดงช่องค้นหาเลยถ้าไม่ตั้งค่า) |
| `ordering` | ลำดับเริ่มต้นเฉพาะใน admin | `Meta.ordering` ของ model |
| `list_per_page` | จำนวนแถวต่อหน้า | `100` |
| `list_max_show_all` | เพดานแสดงลิงก์ "Show all" | `200` |
| `list_select_related` | เพิ่ม `select_related()` ให้ query ของหน้ารายการ | `False` |

---

## ขั้นตอนที่ 164: จัดฟอร์มให้อ่านง่ายด้วย `fieldsets`, `fields`, `readonly_fields`

### 164.1 ปัญหาของฟอร์มแก้ไขแบบ Default

เปิดหน้าแก้ไขบทความ (คลิกที่ชื่อบทความในหน้ารายการ) — Django จะแสดง**ทุก field ที่แก้ไข
ได้เรียงกันเป็นแนวตั้งยาว ๆ** โดยไม่มีการจัดกลุ่มใด ๆ ถ้า model มี field 15-20 ตัว หน้าฟอร์ม
จะยาวและงงมากสำหรับผู้ใช้ที่ไม่คุ้นเคย

### 164.2 `fields`: จำกัด/จัดลำดับ Field แบบง่าย

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    fields = ('title', 'category', 'content', 'is_published')
```

`fields` รับ tuple/list ของชื่อ field ตามลำดับที่ต้องการให้แสดง (เรียงจากบนลงล่าง) —
เหมาะกับฟอร์มที่ไม่ซับซ้อนและไม่ต้องการแบ่งเป็นหมวดหมู่

> **ข้อจำกัดสำคัญที่ต้องรู้ล่วงหน้า**: `fields`/`fieldsets` **ไม่สามารถอ้างถึง `tags`
> ได้เลย** เพราะ `Post.tags` เป็น `ManyToManyField` ที่มี `through='PostTag'` กำหนดเอง —
> ถ้าลองใส่ `'tags'` เข้าไปใน `fields` ตอนนี้ Django จะฟ้อง `admin.E013` ทันทีตอน
> `python manage.py check` เราจะอธิบายสาเหตุแบบละเอียดพร้อมทดลองจริงในขั้นตอนที่ 168

### 164.3 `readonly_fields`: แสดงค่าแต่ห้ามแก้ไข

Field อย่าง `created_at`/`updated_at` ที่มี `auto_now_add=True`/`auto_now=True` **ไม่สามารถ
แก้ไขได้อยู่แล้วในระดับ Django Form** (Django ไม่สร้าง input ให้ field ที่ `editable=False`
ซึ่งเป็นค่า default โดยอัตโนมัติของ `auto_now`/`auto_now_add`) แต่บางครั้งเราต้องการ**แสดง
ค่าให้ดู**โดยไม่ให้แก้ไข — ใช้ `readonly_fields`:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    fields = ('title', 'category', 'content', 'is_published', 'created_at', 'updated_at')
    readonly_fields = ('created_at', 'updated_at')
```

> **กฎสำคัญที่ต้องจำ**: field ใน `readonly_fields` **ต้องถูกระบุไว้ใน `fields` (หรือ
> `fieldsets`) ด้วยเสมอ** ถ้าคุณกำหนด `fields`/`fieldsets` แบบเจาะจงเองแล้ว — ทดลองจริงแล้ว
> พบว่าถ้าตั้ง `readonly_fields = ('created_at',)` แต่**ไม่ใส่ `'created_at'` ไว้ใน `fields`
> เลย** field นั้นจะ**หายไปจากฟอร์มโดยสิ้นเชิง** (ไม่ error แต่ไม่แสดงผลด้วย) ต่างจากกรณีที่
> **ไม่กำหนด `fields`/`fieldsets` เลย** ซึ่ง Django จะเติม field ที่อยู่ใน
> `readonly_fields` เข้าไปในฟอร์มให้อัตโนมัติเป็นค่า default

`readonly_fields` ยังรับ**ชื่อ method** ได้เหมือน `list_display` เพื่อแสดงค่าที่คำนวณสด ๆ
เช่น รายชื่อแท็กทั้งหมดของบทความ (ซึ่งแก้ไขไม่ได้ตรงนี้เพราะข้อจำกัดเรื่อง `through`
ที่กล่าวไปข้างต้น แต่**ดูได้**):

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    fields = ('title', 'category', 'content', 'is_published', 'tag_list', 'created_at')
    readonly_fields = ('tag_list', 'created_at')

    @admin.display(description='แท็กทั้งหมด')
    def tag_list(self, obj):
        return ', '.join(tag.name for tag in obj.tags.all()) or '(ยังไม่มีแท็ก)'
```

### 164.4 `fieldsets`: จัดกลุ่มฟอร์มแบบมืออาชีพ

`fieldsets` คือทางเลือกที่ทรงพลังกว่า `fields` มาก โดยแบ่งฟอร์มเป็น **หมวดหมู่ที่มีหัวข้อ**
พร้อมตัวเลือกเสริมอย่าง `classes` เพื่อทำให้บาง section **พับเก็บได้ (collapsible)**:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    fieldsets = (
        ('ข้อมูลบทความ', {
            'fields': ('title', 'content'),
        }),
        ('การจัดหมวดหมู่', {
            'fields': ('category',),
            'description': 'เลือกหมวดหมู่ที่เหมาะสมที่สุดสำหรับบทความนี้',
        }),
        ('สถานะการเผยแพร่', {
            'fields': ('is_published',),
        }),
        ('ข้อมูลระบบ (Metadata)', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',),
        }),
    )
    readonly_fields = ('created_at', 'updated_at')
```

`fieldsets` เป็น tuple ของ `(ชื่อหมวด, options_dict)` — `ชื่อหมวด` เป็น `None` ได้ถ้าไม่
ต้องการหัวข้อ (มักใช้กับ section แรกสุด) ค่าใน `options_dict` ที่ใช้บ่อยมีดังนี้:

| Key ใน options dict | ความหมาย |
|---|---|
| `fields` | (บังคับ) field ที่อยู่ใน section นี้ — ใส่ tuple ซ้อน tuple เพื่อวาง 2 field ในแถวเดียวกันได้ เช่น `(('title', 'category'),)` |
| `classes` | CSS class พิเศษ; `'collapse'` ทำให้ section พับเก็บโดย default (คลิกขยายได้), `'wide'` ทำให้ label กว้างขึ้น |
| `description` | ข้อความอธิบายเพิ่มเติมใต้หัวข้อ section |

> **กฎเหล็ก**: `fields` และ `fieldsets` **ห้ามใช้พร้อมกันเด็ดขาด** — ทดลองจริงแล้วพบว่า
> Django จะฟ้อง error `admin.E005: Both 'fieldsets' and 'fields' are specified.`
> ทันทีตอน `check` ให้เลือกใช้อย่างใดอย่างหนึ่งเท่านั้น: `fields` สำหรับฟอร์มสั้น ๆ
> ไม่กี่ field, `fieldsets` สำหรับฟอร์มที่ซับซ้อนและต้องการจัดกลุ่ม

### 164.5 ตัวอย่างรวม: `CategoryAdmin` และ `PostAdmin` ที่จัดฟอร์มแล้ว

```python
@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    fieldsets = (
        (None, {
            'fields': ('name', 'slug', 'description'),
        }),
    )


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')
    list_display_links = ('title',)
    list_editable = ('is_published',)
    list_filter = ('is_published', 'category', 'created_at')
    search_fields = ('title', 'content', 'category__name')
    list_select_related = ('category',)
    ordering = ('is_published', '-created_at')
    list_per_page = 25
    readonly_fields = ('created_at', 'updated_at')
    fieldsets = (
        ('ข้อมูลบทความ', {
            'fields': ('title', 'content'),
        }),
        ('การจัดหมวดหมู่', {
            'fields': ('category',),
        }),
        ('สถานะการเผยแพร่', {
            'fields': ('is_published',),
        }),
        ('ข้อมูลระบบ (Metadata)', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',),
        }),
    )

    def get_queryset(self, request):
        return super().get_queryset(request).prefetch_related('comments')

    @admin.display(description='จำนวนคอมเมนต์')
    def comment_count(self, obj):
        return obj.comments.count()
```

เปิดหน้าแก้ไขบทความอีกครั้ง — ตอนนี้ฟอร์มถูกแบ่งเป็น 4 หมวดชัดเจน มี section
"ข้อมูลระบบ" ที่พับเก็บโดย default (เพราะไม่ค่อยมีใครต้องดูบ่อย ๆ) และ `created_at`/
`updated_at` แสดงเป็นข้อความล้วน (ไม่มีช่องให้แก้ไข) นี่คือความแตกต่างระหว่าง Admin ของ
มือใหม่กับ Admin ระดับมืออาชีพ

---

## ขั้นตอนที่ 165: `prepopulated_fields` และ `raw_id_fields` สำหรับ Foreign Key ข้อมูลเยอะ

### 165.1 `prepopulated_fields`: สร้าง Slug อัตโนมัติจาก Title ในหน้า Admin

ทบทวนจาก Part 011-012: `Post.save()` มี logic auto-generate slug จาก title อยู่แล้วถ้า
`self.slug` ว่างเปล่า แต่นั่นเกิดขึ้น**หลัง**กดปุ่ม Save เท่านั้น ผู้ใช้ในหน้า admin ไม่เห็น
slug ที่จะได้จนกว่าจะบันทึกจริง ซึ่งไม่สะดวกถ้าอยากตรวจสอบหรือแก้ไข slug ก่อนบันทึก

`prepopulated_fields` แก้ปัญหานี้ด้วย **JavaScript ที่ auto-fill ช่อง slug แบบเรียลไทม์**
ทันทีที่ผู้ใช้พิมพ์ในช่อง title (ก่อนกด Save ด้วยซ้ำ):

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    prepopulated_fields = {'slug': ('title',)}
    fieldsets = (
        ('ข้อมูลบทความ', {
            'fields': ('title', 'slug', 'content'),
        }),
        # ... section อื่น ๆ เหมือนเดิม ...
    )
```

Syntax คือ `{'ชื่อ_field_ปลายทาง': ('field_ต้นทาง_1', 'field_ต้นทาง_2', ...)}` — ถ้าระบุ
field ต้นทางมากกว่า 1 ตัว Django จะเอาค่ามาต่อกันด้วยขีดกลางแล้ว slugify ให้อัตโนมัติ เช่น
`{'slug': ('category', 'title')}` (แม้ในทางปฏิบัติมักไม่ทำแบบนี้เพราะ `category` เป็น FK
ไม่ใช่ข้อความตรง ๆ)

ใช้แบบเดียวกันกับ `CategoryAdmin` และ `TagAdmin`:

```python
@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    prepopulated_fields = {'slug': ('name',)}
```

> **ข้อจำกัดสำคัญ**: `prepopulated_fields` ใช้ **JavaScript ฝั่ง browser** เท่านั้น มันไม่ได้
> เรียก `slugify()` ของ Python ที่เราเขียนไว้ใน `save()` เลย (เป็นคนละกลไกกัน) ดังนั้น
> ผลลัพธ์อาจต่างกันเล็กน้อยในภาษาที่ไม่ใช่ภาษาอังกฤษ (เช่น ภาษาไทย) — โชคดีที่ logic
> `save()` ของเรายังทำงานเป็น**ตาข่ายนิรภัย (safety net)** อยู่เสมอ: ถ้าผู้ใช้ลบ slug ที่
> auto-fill มาออกจนว่างเปล่าก่อนกด Save, `save()` จะ generate ให้ใหม่จาก Python อยู่ดี —
> ผู้ใช้ admin แก้ไข slug เองได้อิสระเสมอเพราะมันเป็นแค่การเติมค่าเริ่มต้นให้ ไม่ใช่การบังคับ

### 165.2 `raw_id_fields`: ทางออกสำหรับ Foreign Key ที่มีข้อมูลนับพันนับหมื่นแถว

โดย default field ที่เป็น `ForeignKey` จะแสดงเป็น **`<select>` dropdown** ที่โหลดตัวเลือก
**ทุกแถว**ของ model ปลายทางมาแสดงพร้อมกันทั้งหมด สำหรับ `Post.category` ที่มีหมวดหมู่แค่
ไม่กี่สิบหมวด ไม่มีปัญหาอะไร แต่ลองนึกภาพ `Profile.user` ที่อ้างอิงไปยัง `User` — ถ้าระบบมี
ผู้ใช้ 50,000 คน dropdown นั้นจะ**โหลดข้อมูลผู้ใช้ทั้งหมด 50,000 แถวมาไว้ใน HTML เดียว**
ทำให้หน้าเว็บโหลดช้ามาก (บางครั้งถึงขั้น browser ค้าง)

`raw_id_fields` แก้ปัญหานี้โดยเปลี่ยน dropdown เป็น **ช่อง input ธรรมดา + ปุ่มแว่นขยาย** ที่
เปิด popup หน้าต่างค้นหาแยกต่างหาก (มี pagination และช่องค้นหาในตัว) แทนที่จะโหลดทุกแถวมา
พร้อมกัน:

```python
# accounts/admin.py
from django.contrib import admin

from .models import Profile


@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ('user', 'bio')
    raw_id_fields = ('user',)
```

เปิดหน้าเพิ่ม/แก้ไข `Profile` — ช่อง `user` จะกลายเป็นช่อง input ที่มีปุ่มไอคอนแว่นขยาย
คลิกแล้วจะเปิด popup ให้ค้นหาผู้ใช้ด้วยชื่อ/username ก่อนเลือก แทนที่จะต้องเลื่อนหา
dropdown ยาว ๆ

`raw_id_fields` ยังใช้ได้กับ `ForeignKey` ที่ชี้ไปยัง model ตัวเองด้วย (self-referential)
เช่น `Comment.parent`:

```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('author', 'post', 'parent', 'created_at')
    list_filter = ('created_at',)
    search_fields = ('author', 'content', 'post__title')
    raw_id_fields = ('post', 'parent')
```

เลือกใช้ `raw_id_fields` กับทั้ง `post` (ตารางบทความอาจมีเยอะ) และ `parent` (ตาราง
คอมเมนต์เองก็เติบโตเร็วมากในบล็อกที่มีคนเข้ามาก)

### 165.3 ตารางเปรียบเทียบ: Dropdown ปกติ เทียบกับ `raw_id_fields`

| ประเด็น | `<select>` Dropdown (default) | `raw_id_fields` |
|---|---|---|
| วิธีเลือกค่า | เลื่อนหาในรายการที่โหลดมาทั้งหมด | พิมพ์ค้นหาใน popup แยกหน้าต่าง |
| Performance เมื่อมีข้อมูลเยอะ | แย่มาก (โหลดทุกแถวมาพร้อมกัน) | ดี (โหลดทีละหน้าใน popup) |
| UX เมื่อมีข้อมูลน้อย (< 100 แถว) | ดีกว่า (เห็นตัวเลือกทั้งหมดทันที) | ยุ่งยากเกินความจำเป็น |
| แสดงค่าปัจจุบันที่เลือกไว้ | ชื่อเต็มใน dropdown | รหัส pk + ลิงก์ไปหน้าแก้ไข object นั้น |
| เหมาะกับ | FK ที่มีตัวเลือก < 100 แถว เช่น `Post.category` | FK ที่มีตัวเลือกหลักร้อยขึ้นไป เช่น `Profile.user`, `Comment.post` |

> **กฎเบื้องต้น**: ถ้าไม่แน่ใจว่าตารางปลายทางจะโตแค่ไหนในอนาคต ให้ตั้ง `raw_id_fields`
> ไว้ล่วงหน้าเลยเป็นนิสัยสำหรับ FK ที่ชี้ไปยัง `User`, `Post`, `Comment` หรือ model ใด ๆ
> ที่คาดว่าจะมีข้อมูลเติบโตเร็ว ส่วน FK ที่ชี้ไปยัง lookup table ขนาดเล็กที่ค่อนข้างคงที่
> (เช่น `Category`, `Tag`, `Status`) ปล่อยเป็น dropdown ปกติได้สบาย ๆ

### 165.4 ตัวอย่างเปรียบเทียบ: `autocomplete_fields` คือทางเลือกที่ดีกว่า

คุณอาจสังเกตว่า `raw_id_fields` แม้จะเร็วกว่า dropdown แต่ UX (popup แยกหน้าต่าง) ยังดูเก่า
อยู่บ้าง Django มีอีกทางเลือกที่ทันสมัยกว่าคือ **`autocomplete_fields`** ที่ให้พิมพ์ค้นหา
แบบ live-search ในช่องเดียวกันเลยโดยไม่ต้องเปิด popup — แต่ต้องมีการตั้งค่าเพิ่มเติมอีก
เล็กน้อย เราจะเจาะลึกเรื่องนี้แบบเต็มในขั้นตอนที่ 169 (ต้องรอให้ `ModelAdmin` ของ model
ปลายทางมี `search_fields` ก่อน ซึ่งเราเพิ่งตั้งให้ `CategoryAdmin` ไปแล้วในขั้นตอนที่ 162 —
พร้อมใช้งานได้ทันทีเมื่อถึงขั้นตอนนั้น)

---

## ขั้นตอนที่ 166: Inline Admin — แสดง `Comment` และ `PostTag` เป็น Inline

### 166.1 Inline Admin คืออะไร และทำไมสำคัญ

จนถึงตอนนี้ ถ้าอยากดูคอมเมนต์ของบทความหนึ่ง คุณต้องเปิดหน้า `Comments` แยกต่างหาก แล้ว
filter ด้วย `post__title` เอง — ไม่สะดวกเลยเมื่อเทียบกับการ**เห็นคอมเมนต์ทั้งหมดของบทความ
นั้นอยู่ในหน้าแก้ไขบทความเดียวกันเลย** นี่คือสิ่งที่ **Inline Admin** ทำได้: แสดง (และแก้ไข)
ข้อมูลของ model ลูก (ที่มี `ForeignKey` ชี้กลับมา) ฝังอยู่**ภายในหน้าแก้ไขของ model แม่**
โดยตรง

Django มี inline 2 แบบให้เลือก:

| ประเภท | หน้าตา | เหมาะกับ |
|---|---|---|
| `TabularInline` | ตาราง กระชับ แถวละ 1 รายการ | field จำนวนน้อย ต้องการดูหลายแถวพร้อมกัน |
| `StackedInline` | ฟอร์มเต็ม แยก section ต่อรายการ (เหมือน fieldsets ซ้อนกัน) | field จำนวนมาก ต้องการพื้นที่อ่านง่ายต่อรายการ |

### 166.2 สร้าง `CommentInline` ด้วย `TabularInline`

```python
# blog/admin.py
from django.contrib import admin

from .models import Category, Tag, Post, Comment


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ('author', 'content', 'parent', 'created_at')
    readonly_fields = ('created_at',)
    show_change_link = True


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # ... options เดิมจากขั้นตอนก่อนหน้า ...
    inlines = [CommentInline]
```

อธิบายทีละบรรทัด:

- **`model = Comment`**: บอกว่า inline นี้จัดการ model ไหน Django จะหา `ForeignKey`
  ที่ชี้กลับไปยัง `Post` ให้อัตโนมัติ (เจอ `Comment.post`) — ถ้า model มี FK ชี้ไปยัง `Post`
  มากกว่า 1 field ต้องระบุ `fk_name = 'post'` ให้ชัดเจนด้วย
- **`extra = 0`**: จำนวนแถวเปล่าที่แสดงไว้ให้กรอกเพิ่มโดย default (ค่า default ของ Django
  คือ `3`) — ตั้งเป็น `0` เพราะคอมเมนต์มักถูกสร้างจากฝั่งผู้ใช้เว็บจริง ไม่ใช่ทีม admin พิมพ์
  เพิ่มเอง การมีแถวเปล่าเยอะ ๆ ในบทความที่มีคอมเมนต์อยู่แล้วรกโดยไม่จำเป็น
- **`fields`**: จำกัด field ที่แสดงใน inline (ใช้ syntax เดียวกับ `fields` ของ
  `ModelAdmin` ปกติทุกประการ)
- **`readonly_fields`**: field ที่แสดงแต่แก้ไม่ได้ เหมือนกับ `ModelAdmin`
- **`show_change_link = True`**: เพิ่มลิงก์ "Change" ให้แต่ละแถว เผื่ออยากเปิดหน้าแก้ไข
  `Comment` แบบเต็ม (มีประโยชน์เพราะ inline ไม่แสดง field ที่ไม่ได้ระบุใน `fields`)

### 166.3 ปัญหา: Self-Referential FK ใน Inline แสดงข้อมูลไม่เป็นลำดับชั้น

`Comment.parent` เป็น self-referential FK (ตอบกลับกันเอง, Part 012 ขั้นตอนที่ 118) —
เมื่อแสดงเป็น inline แบบตาราง คอมเมนต์ทุกระดับ (ทั้ง root และ reply) จะถูกแสดง**เรียงตาม
`created_at` แบบแบนราบ** ไม่ได้เยื้องแสดงเป็นต้นไม้เหมือนหน้าเว็บจริง และ dropdown ของ
`parent` จะแสดง**คอมเมนต์ทุกอันของทุกบทความ**ในระบบ (ไม่ได้กรองเฉพาะบทความนี้) ซึ่งสับสน
มาก — แก้ไขได้ด้วยการ override `formfield_for_foreignkey()`:

```python
class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ('author', 'content', 'parent', 'created_at')
    readonly_fields = ('created_at',)
    show_change_link = True

    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        if db_field.name == 'parent':
            post_id = request.resolver_match.kwargs.get('object_id')
            if post_id:
                kwargs['queryset'] = Comment.objects.filter(post_id=post_id)
        return super().formfield_for_foreignkey(db_field, request, **kwargs)
```

โค้ดนี้จำกัด dropdown ของ `parent` ให้เลือกได้เฉพาะคอมเมนต์ **ของบทความเดียวกัน**เท่านั้น
(ดึง `object_id` ของ `Post` ที่กำลังแก้ไขอยู่จาก URL ผ่าน `request.resolver_match`) —
สำหรับการแสดงผลแบบต้นไม้ที่เยื้องระดับจริง ๆ (คล้ายกับที่เขียนใน Part 012 แบบฝึกหัดที่ 2)
จะต้องเขียน custom template หรือ custom widget ซึ่งอยู่นอกขอบเขตของ `ModelAdmin` พื้นฐาน
— Part นี้เน้นให้เข้าใจกลไก inline หลัก การปรับแต่ง UI ระดับลึกกว่านี้เป็นเรื่องของ
Part 018

### 166.4 เปรียบเทียบด้วยตัวเดียวกัน: ลอง `StackedInline` แทน `TabularInline`

การเปลี่ยนจาก `TabularInline` เป็น `StackedInline` ทำได้ง่ายมาก — เปลี่ยนแค่ base class
โครงสร้าง field เหมือนเดิมทุกประการ:

```python
class CommentInline(admin.StackedInline):
    model = Comment
    extra = 0
    fields = ('author', 'content', 'parent', 'created_at')
    readonly_fields = ('created_at',)
```

| ประเด็น | `TabularInline` | `StackedInline` |
|---|---|---|
| Layout | ตาราง 1 แถว = 1 record ทุก field อยู่แนวนอนเดียวกัน | ฟอร์มเต็ม ทุก field เรียงแนวตั้ง แยกกล่องต่อ record |
| เหมาะกับจำนวน field | น้อย (2-5 field) | มาก (6 field ขึ้นไป) |
| เหมาะกับจำนวนแถวที่คาดว่าจะมี | เยอะ (สแกนดูพร้อมกันหลายแถวง่าย) | น้อย-ปานกลาง (สแกนแนวตั้งยาวถ้ามีหลายแถว) |
| ตัวอย่างที่เหมาะในโปรเจกต์นี้ | `CommentInline` (4 field, มักมีหลายสิบคอมเมนต์ต่อบทความ) | เหมาะกับ inline ที่มี field เยอะกว่า |

สำหรับ `Comment` ที่มี field ไม่เยอะแต่มักมี**จำนวนแถวเยอะ**ต่อบทความ `TabularInline`
เหมาะสมกว่า `StackedInline` อย่างชัดเจน เราจึงกลับไปใช้ `TabularInline` เป็นตัวเลือกสุดท้าย

### 166.5 Inline ตัวที่สอง: `PostTagInline` สำหรับ Through Model

ย้อนกลับไปที่ขั้นตอนที่ 161.4 — เรา**จงใจไม่ลงทะเบียน `PostTag` แบบเดี่ยว** เพราะมันเป็น
through model ของ `Post.tags` ที่เหมาะกับการจัดการผ่าน inline มากกว่า สร้าง inline ให้มัน
ได้แบบเดียวกับ `CommentInline` ทุกประการ (เพราะ `PostTag` ก็เป็นแค่ model ธรรมดาที่มี FK
สองตัว):

```python
from .models import Category, Tag, Post, PostTag, Comment


class PostTagInline(admin.TabularInline):
    model = PostTag
    extra = 1
    autocomplete_fields = ('tag',)  # เจาะลึกเต็มรูปแบบในขั้นตอนที่ 169


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # ... options เดิม ...
    inlines = [PostTagInline, CommentInline]
```

ทดสอบเปิดหน้าเพิ่มบทความใหม่ — ตอนนี้คุณจะเห็น **2 ส่วน inline** ต่อท้ายฟอร์มหลัก: ส่วน
"Post tags" (จัดการแท็กพร้อมข้อมูล `added_by` ที่ผู้ใช้กรอกเองได้) และส่วน "Comments"
(แสดงคอมเมนต์ที่มีอยู่แล้ว) — นี่คือ**คำตอบที่ถูกต้องอย่างเป็นทางการของ Django** สำหรับการ
จัดการ M2M ที่มี `through` แบบกำหนดเอง เพราะฟอร์มหลักของ `Post` **จะไม่มีช่องให้เลือก tags
ตรง ๆ เลย** (ดังที่อธิบายไปในขั้นตอนที่ 164.2) — `PostTagInline` คือทางออกเดียวที่ใช้งานได้
จริงในหน้า admin ของ `Post` เพื่อผูกแท็กเข้ากับบทความ

> **ตรวจสอบด้วยตัวเอง**: ลองเปิด view-source หรือ inspect element ของฟอร์มแก้ไข `Post`
> คุณจะเห็น input ที่ชื่อ `posttag_set-0-tag`, `posttag_set-0-added_by` (มาจาก inline
> formset) แต่จะ**ไม่มี** input ชื่อ `tags` อยู่เลยในฟอร์มหลัก — นี่คือหลักฐานที่ยืนยันตรงกับ
> ที่อธิบายไว้

### 166.6 ตารางสรุปตัวเลือกของ Inline

| ตัวเลือก | ความหมาย |
|---|---|
| `model` | model ลูกที่จะแสดงเป็น inline (บังคับ) |
| `fk_name` | ระบุ FK field ที่ใช้เชื่อมกลับมา (จำเป็นเมื่อมี FK มากกว่า 1 ตัวชี้ไปยัง model แม่) |
| `extra` | จำนวนแถวเปล่าเริ่มต้นสำหรับกรอกเพิ่ม |
| `max_num` / `min_num` | จำกัดจำนวนแถวสูงสุด/ต่ำสุดที่อนุญาต |
| `fields` / `fieldsets` / `readonly_fields` | เหมือนกับ `ModelAdmin` ทุกประการ |
| `can_delete` | อนุญาตให้ลบแถวจาก inline ได้หรือไม่ (default `True`) |
| `show_change_link` | เพิ่มลิงก์ไปหน้าแก้ไขเต็มรูปแบบของแถวนั้น |
| `autocomplete_fields` | ใช้ live-search แทน dropdown สำหรับ FK ภายใน inline (เหมือน `ModelAdmin`) |

---

## ขั้นตอนที่ 167: กำหนดฟอร์ม Admin เอง (`form = CustomModelForm`) และ Custom Validation

### 167.1 เมื่อไหร่ที่ Model Validation ไม่พอ

Model ของเรามี validation อยู่บ้างแล้ว (เช่น `UniqueConstraint`, `CheckConstraint` จาก
Part 015) แต่บาง rule เป็น **business logic ที่เกี่ยวกับ "การกรอกฟอร์ม" โดยเฉพาะ** ไม่ใช่
ความถูกต้องของข้อมูลในฐานข้อมูลเสมอไป เช่น:

- "ห้ามเผยแพร่บทความ (`is_published=True`) ถ้าเนื้อหาสั้นกว่า 100 ตัวอักษร"
- "ถ้าเลือก `is_published=True` ต้องเลือก `category` ด้วยเสมอ (ห้ามเผยแพร่บทความที่ยังไม่มี
  หมวดหมู่)"

Rule แบบนี้เหมาะจะอยู่ใน **`ModelForm`** ที่ใช้เฉพาะตอนกรอกฟอร์มผ่าน admin (หรือฟอร์ม
หน้าเว็บจริงที่จะเรียนใน Part 025) มากกว่าอยู่ใน `Model.save()` เอง เพราะ `save()` ถูกเรียก
จากหลายที่ (shell, data migration, test) ที่อาจไม่ต้องการให้ validation ระดับฟอร์มมาบล็อก

### 167.2 สร้าง `PostAdminForm`

```python
# blog/forms.py
from django import forms

from .models import Post


class PostAdminForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = '__all__'

    def clean(self):
        cleaned_data = super().clean()
        is_published = cleaned_data.get('is_published')
        category = cleaned_data.get('category')
        content = cleaned_data.get('content', '')

        if is_published and not category:
            raise forms.ValidationError(
                'ไม่สามารถเผยแพร่บทความที่ยังไม่มีหมวดหมู่ได้ กรุณาเลือกหมวดหมู่ก่อน'
            )

        if is_published and len(content) < 100:
            raise forms.ValidationError(
                'เนื้อหาต้องมีความยาวอย่างน้อย 100 ตัวอักษรก่อนเผยแพร่ได้ '
                f'(ตอนนี้มี {len(content)} ตัวอักษร)'
            )

        return cleaned_data
```

`clean()` (ไม่ใช่ `clean_<field>()`) ใช้เมื่อ validation ต้อง**เปรียบเทียบระหว่างหลาย
field พร้อมกัน** (`is_published` คู่กับ `category` และ `content`) ต่างจาก
`clean_<field_name>()` ที่ตรวจสอบ field เดียวโดด ๆ ตัวอย่างเช่น การบังคับให้ title ไม่มีคำ
ต้องห้าม:

```python
    def clean_title(self):
        title = self.cleaned_data['title']
        banned_words = ['สแปม', 'โฆษณา']
        for word in banned_words:
            if word in title:
                raise forms.ValidationError(f'ชื่อบทความห้ามมีคำว่า "{word}"')
        return title
```

### 167.3 ผูก `PostAdminForm` เข้ากับ `PostAdmin`

```python
# blog/admin.py
from .forms import PostAdminForm


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    form = PostAdminForm
    # ... options เดิมทั้งหมด ...
```

ทดสอบ: ลองติ๊ก `is_published` โดยไม่เลือก `category` แล้วกด Save — Django จะแสดงข้อความ
error สีแดงที่ด้านบนฟอร์ม ("ไม่สามารถเผยแพร่บทความที่ยังไม่มีหมวดหมู่ได้...") และ**ไม่
บันทึกข้อมูล**จนกว่าจะแก้ให้ผ่าน validation

### 167.4 `form` ระดับ `ModelAdmin` เทียบกับ `form` ระดับ Inline

Inline ก็รับ `form = CustomForm` ได้เช่นเดียวกับ `ModelAdmin` ปกติ ตัวอย่างเช่น บังคับให้
`added_by` ใน `PostTagInline` ต้องกรอกเสมอ (ทั้งที่ model กำหนด `blank=True` ไว้เพื่อความ
ยืดหยุ่นตอนสร้างข้อมูลผ่าน shell):

```python
# blog/forms.py
class PostTagInlineForm(forms.ModelForm):
    class Meta:
        model = PostTag
        fields = '__all__'

    def clean_added_by(self):
        added_by = self.cleaned_data.get('added_by', '').strip()
        if not added_by:
            raise forms.ValidationError('กรุณาระบุชื่อผู้เพิ่มแท็กนี้ (สำหรับตรวจสอบย้อนหลัง)')
        return added_by
```

```python
class PostTagInline(admin.TabularInline):
    model = PostTag
    form = PostTagInlineForm
    extra = 1
    autocomplete_fields = ('tag',)
```

### 167.5 ทำไมไม่ยัด Logic ทั้งหมดไว้ใน `Model.clean()` แทน

Django `Model` ก็มี method `clean()` ของตัวเองเช่นกัน (เรียกผ่าน `full_clean()`) แล้วทำไม
เราไม่เขียน validation ไว้ตรงนั้นแทนที่จะแยกมาไว้ใน `ModelForm`?

| ประเด็น | `Model.clean()` | `ModelForm.clean()` |
|---|---|---|
| ถูกเรียกอัตโนมัติเมื่อไหร่ | **ไม่ถูกเรียกอัตโนมัติ** ตอน `.save()` เลย (ต้องเรียก `full_clean()` เอง) | ถูกเรียกอัตโนมัติทุกครั้งที่ฟอร์ม `is_valid()` (รวมถึงในหน้า admin) |
| เหมาะกับ | กฎที่ต้องเป็นจริง**เสมอ** ไม่ว่าจะสร้างข้อมูลจากที่ไหน (shell, API, admin, data migration) | กฎที่เกี่ยวกับ**การกรอกฟอร์มของมนุษย์**โดยเฉพาะ อาจไม่ต้องบังคับตอนสร้างข้อมูลผ่านโค้ด |
| ตัวอย่างในโปรเจกต์นี้ | เหมาะกับกฎอย่าง "ราคาต้องไม่ติดลบ" (ใช้ `CheckConstraint` แทนได้ในกรณีนี้) | เหมาะกับกฎอย่าง "ห้ามเผยแพร่โดยไม่มีหมวดหมู่" (เป็นกฎของ *ขั้นตอนการทำงาน* ไม่ใช่ความถูกต้องของข้อมูลเสมอไป) |

> **กฎของหลักสูตรนี้**: ถ้ากฎต้องเป็นจริง**ทุกครั้งไม่มีข้อยกเว้น** ไม่ว่าข้อมูลจะเข้ามาทาง
> ไหน ให้ใช้ database constraint (`CheckConstraint`/`UniqueConstraint` จาก Part 015)
> เพราะมันบังคับที่ระดับฐานข้อมูลเอง แข็งแกร่งที่สุด ถ้ากฎเกี่ยวกับ**ขั้นตอนการทำงานของคน
> กรอกฟอร์ม** (draft ยังไม่ครบไม่เป็นไร แต่เผยแพร่แล้วต้องครบ) ให้ใช้ `ModelForm.clean()`
> ที่ผูกกับ admin หรือฟอร์มหน้าเว็บโดยเฉพาะ

---

## ขั้นตอนที่ 168: `filter_horizontal`/`filter_vertical` สำหรับ `ManyToManyField`

### 168.1 Widget พิเศษสำหรับ ManyToMany: แนวคิดเบื้องต้น

โดย default field ที่เป็น `ManyToManyField` (ที่ไม่มี custom `through`) จะแสดงเป็น
`<select multiple>` — กล่องเดียวที่ต้องกด `Ctrl`/`Cmd` ค้างไว้แล้วคลิกเลือกหลายรายการ ซึ่ง
ใช้งานยากและมองไม่เห็นภาพรวมว่าเลือกอะไรไปแล้วบ้าง Django มี widget ที่ดีกว่ามากคือ
**"filter interface"**: แสดงเป็น 2 กล่องข้าง ๆ กัน (ซ้าย = ตัวเลือกที่ยังไม่ได้เลือก,
ขวา = ตัวเลือกที่เลือกแล้ว) พร้อมช่องค้นหาในตัว และปุ่มลูกศรย้ายไปมา:

```python
filter_horizontal = ('field_name',)   # จัดวางกล่อง 2 อันแนวนอน (ซ้าย-ขวา)
filter_vertical = ('field_name',)     # จัดวางกล่อง 2 อันแนวตั้ง (บน-ล่าง)
```

ความแตกต่างมีแค่การจัดวาง UI เท่านั้น กลไกการทำงานเหมือนกันทุกประการ — `filter_horizontal`
เหมาะกับหน้าจอกว้าง ส่วน `filter_vertical` เหมาะกับ field ที่อยู่ใน sidebar แคบ ๆ ของ
fieldset

### 168.2 ลองใช้กับ `Post.tags` — ทำไมมันไม่ทำงาน

เนื่องจากในโปรเจกต์นี้ `Post.tags` คือฟิลด์ M2M ที่ใช้บ่อยที่สุด เราลองเพิ่ม
`filter_horizontal` เข้าไปดู:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    filter_horizontal = ('tags',)
    # ...
```

ทดสอบจริงด้วย `python manage.py check` พบว่า Django **ปฏิเสธทันที** ด้วย error:

```
ERRORS:
<class 'blog.admin.PostAdmin'>:
    (admin.E013) The value of 'filter_horizontal[0]' cannot include the
    ManyToManyField 'tags', because that field manually specifies a
    relationship model.
```

**สาเหตุที่แท้จริง**: เมื่อ `ManyToManyField` มี `through=` เป็น model ที่**คุณกำหนดเอง**
(ไม่ใช่ตารางกลางอัตโนมัติที่ Django สร้างให้) Django Admin **ไม่มีทางรู้ได้เลย**ว่าจะเติมค่า
ใน field พิเศษของ through model (`PostTag.added_at`, `PostTag.added_by`) ให้อัตโนมัติได้
อย่างไรเมื่อผู้ใช้กดปุ่ม "เพิ่ม" ใน widget แบบ filter interface — เพราะ widget นั้นถูก
ออกแบบมาสำหรับกรณีที่ตารางกลางมีแค่ 2 คอลัมน์ (FK ไปตาราง A + FK ไปตาราง B) เท่านั้น
ด้วยเหตุนี้ ในระดับที่ลึกกว่านั้น **Django ไม่สร้าง form field ให้ `tags` เลยตั้งแต่แรก**
เมื่อ `through` เป็นแบบกำหนดเอง (ยืนยันแล้วจากการทดสอบจริง: ฟอร์มเพิ่มบทความที่ไม่ได้ตั้ง
`fields`/`filter_horizontal` ใด ๆ เป็นพิเศษ จะไม่มี input ชื่อ `tags` ปรากฏอยู่เลย) —
`filter_horizontal`/`filter_vertical`/`raw_id_fields`/`autocomplete_fields` และการระบุใน
`fields`/`fieldsets` ตรง ๆ **ล้วนใช้ไม่ได้กับ `tags` ทั้งหมดด้วยเหตุผลเดียวกัน**

นี่ไม่ใช่ bug แต่เป็น**พฤติกรรมที่ตั้งใจ**ของ Django — และเป็นเหตุผลที่แท้จริงว่าทำไมเรา
สร้าง `PostTagInline` ไว้ตั้งแต่ขั้นตอนที่ 166.5 เพื่อเป็น**ทางออกที่ถูกต้องอย่างเป็นทางการ**
สำหรับสถานการณ์นี้

### 168.3 ตัวอย่างที่ `filter_horizontal` ใช้งานได้จริง: ระบบ "บทความโปรด" ของผู้ใช้

เพื่อให้เห็นภาพว่า `filter_horizontal` ทำงานได้ดีแค่ไหนเมื่อใช้กับ M2M **ที่ไม่มี custom
through** เราจะเพิ่มฟีเจอร์เล็ก ๆ ที่มีประโยชน์จริงเข้าไปในระบบ: ให้ผู้ใช้บันทึก "บทความ
โปรด" ไว้ใน `Profile` ได้

```python
# accounts/models.py
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    bio = models.TextField(max_length=500, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    favorite_posts = models.ManyToManyField(
        'blog.Post',
        related_name='favorited_by',
        blank=True,
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f'โปรไฟล์ของ {self.user.username}'
```

สังเกตว่าเราอ้างอิง `Post` ด้วย **string `'blog.Post'`** (รูปแบบ `'ชื่อแอป.ชื่อโมเดล'`)
แทนที่จะ `from blog.models import Post` มา import ตรง ๆ — เพราะ `accounts` กับ `blog` เป็น
คนละแอป การ import ข้ามแอปตรง ๆ ในบาง configuration อาจทำให้เกิด **circular import**
(ถ้าวันหนึ่ง `blog/models.py` ต้อง import อะไรบางอย่างจาก `accounts` กลับมาด้วย) การอ้างอิง
ด้วย string ทำให้ Django resolve ความสัมพันธ์นี้แบบ lazy หลังโหลด app registry ครบทุกแอป
แล้ว จึงปลอดภัยกว่าเสมอเมื่ออ้างอิงข้ามแอป

`favorite_posts` เป็น `ManyToManyField` **ธรรมดา** (ไม่มี `through=`) ดังนั้น
`filter_horizontal` จะใช้งานได้เต็มรูปแบบ:

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

```python
# accounts/admin.py
@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ('user', 'bio')
    raw_id_fields = ('user',)
    filter_horizontal = ('favorite_posts',)
```

เปิดหน้าแก้ไข `Profile` — ตอนนี้คุณจะเห็น widget filter interface ที่ใช้งานได้จริงสำหรับ
`favorite_posts`: กล่องซ้ายแสดงบทความทั้งหมดที่ยังไม่ถูกเลือก กล่องขวาแสดงบทความที่ถูกตั้ง
เป็นบทความโปรดแล้ว มีช่องค้นหาด้านบนแต่ละกล่อง และปุ่มลูกศรคู่ (เลือกทั้งหมด/ทั้งหมดคืน)
ให้ใช้งาน — ครบทุกฟีเจอร์ของ widget นี้

### 168.4 ตารางสรุป: ตัวเลือกสำหรับ Field ManyToMany ใน Admin

| สถานการณ์ | Widget ที่ใช้ได้ | ตัวอย่างในโปรเจกต์นี้ |
|---|---|---|
| M2M ธรรมดา ไม่มี `through` กำหนดเอง | `<select multiple>` (default), `filter_horizontal`, `filter_vertical` | `Profile.favorite_posts` |
| M2M ที่มี `through` กำหนดเอง | **ไม่มี widget ใดใช้ได้เลยบนฟิลด์ M2M เอง** ต้องจัดการผ่าน Inline ของ through model แทน | `Post.tags` → ใช้ `PostTagInline` |

### 168.5 เมื่อไหร่ควรเลือก `filter_horizontal` เมื่อไหร่ควรเลือก `filter_vertical`

| ปัจจัย | เลือก `filter_horizontal` | เลือก `filter_vertical` |
|---|---|---|
| ความกว้างหน้าจอที่ใช้งาน | จอกว้าง (desktop เป็นหลัก) | จอแคบ หรืออยู่ใน sidebar ของ fieldset |
| ตำแหน่งใน fieldset | อยู่ section เดี่ยว ๆ กว้างเต็มความกว้างฟอร์ม | อยู่ fieldset ที่มีความกว้างจำกัด |
| ความนิยมในโปรเจกต์จริง | นิยมมากกว่า (เป็นค่าที่ Django ใช้เป็นตัวอย่างในเอกสารทางการ) | ใช้เฉพาะกรณีพิเศษ |

---

## ขั้นตอนที่ 169: `date_hierarchy` และ `autocomplete_fields`

### 169.1 `date_hierarchy`: เมนูไต่ลำดับเวลาแบบ Drill-Down

สำหรับ model ที่มี `DateField`/`DateTimeField` และมีข้อมูลจำนวนมากสะสมมาเรื่อย ๆ (เช่น
`Post.created_at` ที่จะมีบทความเพิ่มขึ้นทุกวัน) `date_hierarchy` เพิ่มแถบนำทางแบบ **ปี →
เดือน → วัน** ไว้ด้านบนของหน้ารายการ:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    date_hierarchy = 'created_at'
    # ... options อื่น ๆ ...
```

รับแค่ **ชื่อ field เดียว** เป็น string (ไม่ใช่ tuple เหมือนตัวเลือกอื่น ๆ) เมื่อเปิดหน้า
`Posts` จะเห็นแถบ `2026` ด้านบนตาราง คลิกเข้าไปจะเห็นเดือนทั้ง 12 เดือนที่มีบทความอยู่
คลิกต่อไปจะเห็นวันในเดือนนั้น และคลิกวันจะกรองเฉพาะบทความที่สร้างในวันนั้น — Django จัดการ
สร้าง URL parameter (`created_at__year=2026&created_at__month=3`) และ query filter ให้
อัตโนมัติทั้งหมด ไม่ต้องเขียนโค้ดเพิ่มเลย

เพิ่มให้กับ `CommentAdmin` ด้วย เนื่องจากคอมเมนต์มักมีจำนวนมากและเกิดขึ้นทุกวันเช่นกัน:

```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    date_hierarchy = 'created_at'
    # ... options อื่น ๆ ...
```

> **ข้อควรระวัง**: `date_hierarchy` เพิ่ม query เพื่อนับจำนวนแถวต่อปี/เดือน/วันทุกครั้งที่
> โหลดหน้า ถ้าตารางมีข้อมูลระดับล้านแถว อาจกระทบ performance เล็กน้อย — สำหรับ scale
> ระดับนั้น มักต้องพิจารณา caching หรือ custom admin view ซึ่งเป็นหัวข้อขั้นสูงกว่าที่จะ
> พูดถึงใน Phase 8 (Performance & Caching)

### 169.2 `autocomplete_fields`: Live-Search แทน Dropdown

ย้อนกลับไปที่ขั้นตอนที่ 165.4 — เราเกริ่นไว้ว่า `autocomplete_fields` ให้ UX ที่ดีกว่า
`raw_id_fields` เพราะพิมพ์ค้นหาได้ในช่องเดียวกันเลยแบบ live-search (เหมือนช่องค้นหาของ
เว็บสมัยใหม่ทั่วไป) ไม่ต้องเปิด popup แยกหน้าต่าง:

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    autocomplete_fields = ('category',)
    # ... options อื่น ๆ ...
```

ทดสอบจริงด้วย `python manage.py check` ทันที (ก่อนเตรียมอะไรเพิ่มเติม) จะพบ error:

```
ERRORS:
<class 'blog.admin.PostAdmin'>:
    (admin.E040) CategoryAdmin must define "search_fields", because
    it's referenced by PostAdmin.autocomplete_fields.
```

**เหตุผล**: `autocomplete_fields` ทำงานโดยยิง AJAX request ไปที่ endpoint พิเศษ
(`/admin/autocomplete/`) ทุกครั้งที่ผู้ใช้พิมพ์ในช่องค้นหา แล้ว Django ต้องรู้ว่าจะ**ค้นหา
ด้วย field ไหนบ้าง** ของ model ปลายทาง (`Category`) — มันจึง**บังคับ**ให้
`ModelAdmin` ของ model ปลายทางต้องมี `search_fields` กำหนดไว้ก่อนเสมอ ไม่เช่นนั้นจะไม่รู้
ว่าจะค้นหาจาก field ไหน

โชคดีที่เรากำหนด `search_fields = ('name',)` ให้ `CategoryAdmin` ไปแล้วตั้งแต่ขั้นตอนที่
162.5 — เพียงแค่เพิ่ม `autocomplete_fields` ที่ `PostAdmin` ก็ใช้งานได้ทันที:

```python
@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)          # ← จำเป็นสำหรับ autocomplete_fields ที่อ้างถึงมัน
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    autocomplete_fields = ('category',)
    # ...
```

เปิดหน้าเพิ่ม/แก้ไขบทความอีกครั้ง — ช่อง `category` ตอนนี้กลายเป็นช่องพิมพ์ค้นหาแบบ
live-search (ขับเคลื่อนด้วย Select2 ที่ Django ฝังมาให้ในตัว) พิมพ์ชื่อหมวดหมู่บางส่วนแล้ว
ระบบจะกรองตัวเลือกให้แบบเรียลไทม์ ใช้งานลื่นไหลกว่า `raw_id_fields` มาก โดยไม่ต้องเปิด
popup แยกหน้าต่างเลย

`autocomplete_fields` ใช้ได้กับ **Inline** เช่นกัน อย่างที่เราใช้ไปแล้วกับ `tag` ใน
`PostTagInline` ตั้งแต่ขั้นตอนที่ 166.5 (ซึ่งใช้งานได้เพราะ `TagAdmin` ก็มี
`search_fields = ('name',)` กำหนดไว้แล้วเช่นกัน)

### 169.3 ตารางเปรียบเทียบทางเลือกทั้งหมดสำหรับ Foreign Key ที่มีข้อมูลเยอะ

| ตัวเลือก | ต้องมี `search_fields` บน model ปลายทางก่อนไหม | UX | ความเร็วเมื่อข้อมูลเยอะ |
|---|---|---|---|
| `<select>` default | ไม่ต้อง | แย่ที่สุดเมื่อข้อมูลเยอะ | แย่มาก (โหลดทุกแถว) |
| `raw_id_fields` | ไม่ต้อง | ต้องเปิด popup แยกหน้าต่าง | ดี |
| `autocomplete_fields` | **ต้องมี** (ไม่งั้น `admin.E040`) | ดีที่สุด (live-search ในช่องเดียว) | ดี |

> **คำแนะนำระดับมืออาชีพ**: ในโปรเจกต์จริง ให้เลือก `autocomplete_fields` เป็นค่าเริ่มต้น
> สำหรับ FK ที่มีข้อมูลเยอะเสมอ เพราะ UX ดีกว่า `raw_id_fields` อย่างชัดเจนโดยแทบไม่มี
> ข้อเสียเพิ่มเติม (แค่ต้องมี `search_fields` ที่เหมาะสมกับ model ปลายทางอยู่แล้ว ซึ่งเป็น
> good practice ที่ควรมีอยู่แล้วตั้งแต่ขั้นตอนที่ 162)

### 169.4 อัปเดต `PostTagInline` ให้ใช้ `autocomplete_fields` แทน Default

```python
class PostTagInline(admin.TabularInline):
    model = PostTag
    form = PostTagInlineForm
    extra = 1
    autocomplete_fields = ('tag',)
```

(นี่คือสิ่งที่เราเขียนไว้แล้วตั้งแต่ขั้นตอนที่ 166.5 — ตอนนี้คุณเข้าใจแล้วว่าทำไมมันใช้งานได้
เพราะ `TagAdmin.search_fields` ถูกกำหนดไว้ล่วงหน้าแล้ว)

---

## ขั้นตอนที่ 170: สรุปและแบบฝึกหัด

### 170.1 ไฟล์ `blog/admin.py` ฉบับสมบูรณ์

```python
# blog/admin.py
from django.contrib import admin

from .forms import PostAdminForm, PostTagInlineForm
from .models import Category, Comment, Post, PostTag, Tag


class PostTagInline(admin.TabularInline):
    model = PostTag
    form = PostTagInlineForm
    extra = 1
    autocomplete_fields = ('tag',)


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ('author', 'content', 'parent', 'created_at')
    readonly_fields = ('created_at',)
    show_change_link = True

    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        if db_field.name == 'parent':
            post_id = request.resolver_match.kwargs.get('object_id')
            if post_id:
                kwargs['queryset'] = Comment.objects.filter(post_id=post_id)
        return super().formfield_for_foreignkey(db_field, request, **kwargs)


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    prepopulated_fields = {'slug': ('name',)}
    fieldsets = (
        (None, {'fields': ('name', 'slug', 'description')}),
    )


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    search_fields = ('name',)
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    form = PostAdminForm
    list_display = ('title', 'category', 'is_published', 'created_at', 'comment_count')
    list_display_links = ('title',)
    list_editable = ('is_published',)
    list_filter = ('is_published', 'category', 'created_at')
    search_fields = ('title', 'content', 'category__name')
    list_select_related = ('category',)
    ordering = ('is_published', '-created_at')
    list_per_page = 25
    date_hierarchy = 'created_at'
    autocomplete_fields = ('category',)
    readonly_fields = ('created_at', 'updated_at')
    prepopulated_fields = {'slug': ('title',)}
    inlines = [PostTagInline, CommentInline]
    fieldsets = (
        ('ข้อมูลบทความ', {
            'fields': ('title', 'slug', 'content'),
        }),
        ('การจัดหมวดหมู่', {
            'fields': ('category',),
        }),
        ('สถานะการเผยแพร่', {
            'fields': ('is_published',),
        }),
        ('ข้อมูลระบบ (Metadata)', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',),
        }),
    )

    def get_queryset(self, request):
        return super().get_queryset(request).prefetch_related('comments')

    @admin.display(description='จำนวนคอมเมนต์')
    def comment_count(self, obj):
        return obj.comments.count()


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ('author', 'post', 'parent', 'created_at')
    list_filter = ('created_at',)
    search_fields = ('author', 'content', 'post__title')
    raw_id_fields = ('post', 'parent')
    date_hierarchy = 'created_at'
```

### 170.2 ไฟล์ `blog/forms.py` ฉบับสมบูรณ์

```python
# blog/forms.py
from django import forms

from .models import Post, PostTag


class PostAdminForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = '__all__'

    def clean_title(self):
        title = self.cleaned_data['title']
        banned_words = ['สแปม', 'โฆษณา']
        for word in banned_words:
            if word in title:
                raise forms.ValidationError(f'ชื่อบทความห้ามมีคำว่า "{word}"')
        return title

    def clean(self):
        cleaned_data = super().clean()
        is_published = cleaned_data.get('is_published')
        category = cleaned_data.get('category')
        content = cleaned_data.get('content', '')

        if is_published and not category:
            raise forms.ValidationError(
                'ไม่สามารถเผยแพร่บทความที่ยังไม่มีหมวดหมู่ได้ กรุณาเลือกหมวดหมู่ก่อน'
            )
        if is_published and len(content) < 100:
            raise forms.ValidationError(
                'เนื้อหาต้องมีความยาวอย่างน้อย 100 ตัวอักษรก่อนเผยแพร่ได้ '
                f'(ตอนนี้มี {len(content)} ตัวอักษร)'
            )
        return cleaned_data


class PostTagInlineForm(forms.ModelForm):
    class Meta:
        model = PostTag
        fields = '__all__'

    def clean_added_by(self):
        added_by = self.cleaned_data.get('added_by', '').strip()
        if not added_by:
            raise forms.ValidationError('กรุณาระบุชื่อผู้เพิ่มแท็กนี้ (สำหรับตรวจสอบย้อนหลัง)')
        return added_by
```

### 170.3 ไฟล์ `accounts/admin.py` ฉบับสมบูรณ์ (รวมการฝัง `Profile` เข้าไปในหน้าแก้ไข `User`)

นอกจากลงทะเบียน `Profile` แบบเดี่ยว เราจะโชว์เทคนิคระดับมืออาชีพอีกอย่างหนึ่ง: การ**ฝัง
`Profile` เป็น inline อยู่ในหน้าแก้ไข `User`** ของ Django เอง (แทนที่จะต้องเปิด 2 หน้าแยก
กันเพื่อแก้ user คนเดียว) ทำได้โดย **unregister** `UserAdmin` เดิม แล้ว register ใหม่ด้วย
คลาสที่สืบทอดจากของเดิมและเพิ่ม inline เข้าไป:

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as DefaultUserAdmin
from django.contrib.auth.models import User

from .models import Profile


class ProfileInline(admin.StackedInline):
    model = Profile
    can_delete = False
    verbose_name_plural = 'โปรไฟล์'


class UserAdmin(DefaultUserAdmin):
    inlines = (ProfileInline,)


admin.site.unregister(User)
admin.site.register(User, UserAdmin)


@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ('user', 'bio')
    raw_id_fields = ('user',)
    filter_horizontal = ('favorite_posts',)
    search_fields = ('user__username', 'user__email')
```

ตอนนี้เปิดหน้า **Users** ใน admin แล้วคลิกแก้ไขผู้ใช้คนไหนก็ได้ — คุณจะเห็น section
"โปรไฟล์" ต่อท้ายฟอร์มแก้ไขผู้ใช้มาตรฐานของ Django โดยอัตโนมัติ พร้อมกันนั้น `Profile`
ก็ยังเปิดเป็นหน้าเดี่ยวแยกต่างหากได้ด้วย (ผ่าน `ProfileAdmin` ที่ยังลงทะเบียนคู่ขนานกันไว้)
— ตัวอย่างนี้สรุปทุกเทคนิคที่เรียนไปตลอด Part นี้ไว้ในที่เดียว: inline, `raw_id_fields`,
`filter_horizontal`, `search_fields`

> **หมายเหตุ**: `can_delete = False` ใน `ProfileInline` ป้องกันไม่ให้ผู้ดูแลระบบลบ
> `Profile` ทิ้งโดยไม่ได้ตั้งใจจากหน้าแก้ไข `User` (เพราะ `Profile` ควรอยู่คู่กับ `User`
> เสมอตามที่ออกแบบไว้ตั้งแต่ Part 012) — ถ้าต้องการลบ `Profile` จริง ๆ ต้องลบ `User`
> ทั้งตัวไปเลย (ซึ่งจะ `CASCADE` ลบ `Profile` ตามไปโดยอัตโนมัติ)

### 170.4 ปรับแต่งหน้าตา Admin Site โดยรวมเล็กน้อย (Bonus)

ก่อนปิดท้าย Part นี้ มาปรับแต่งหัวเรื่องของ Admin Site ให้ดูเป็นมืออาชีพขึ้นอีกนิด (ไม่ต้อง
สร้าง `AdminSite` class เองก็ทำได้ ผ่านการตั้งค่า attribute ของ `admin.site` โดยตรง):

```python
# config/urls.py หรือไฟล์ apps.py ของแอปหลักก็ได้ (ให้รันครั้งเดียวตอน startup)
from django.contrib import admin

admin.site.site_header = 'ระบบจัดการบล็อก Django Mastery'
admin.site.site_title = 'Django Mastery Admin'
admin.site.index_title = 'แผงควบคุมสำหรับทีม Content'
```

การปรับแต่ง Admin Site แบบเต็มรูปแบบ (custom `AdminSite` class, custom template, custom
actions แบบ bulk operation, permission แบบละเอียด) จะอยู่ใน **Part 018** ทั้งหมด

### 170.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ลงทะเบียน model เข้า admin ได้ทั้งแบบ `admin.site.register()` และ `@admin.register()`
  พร้อมเข้าใจว่าเมื่อไหร่ควรใช้แบบไหน
- ✅ ปรับแต่งหน้ารายการด้วย `list_display` (รวมถึงคอลัมน์ที่มาจาก method),
  `list_filter`, และ `search_fields` (พร้อม prefix พิเศษ `^`, `=`, `@`)
- ✅ เพิ่มประสิทธิภาพการทำงานของทีมด้วย `list_editable`, `list_display_links`,
  `ordering`, `list_per_page`
- ✅ จัดฟอร์มแก้ไขให้อ่านง่ายด้วย `fieldsets`, `fields`, `readonly_fields` และรู้กฎว่า
  `fields`/`fieldsets` ใช้พร้อมกันไม่ได้ (`admin.E005`)
- ✅ ใช้ `prepopulated_fields` สร้าง slug อัตโนมัติ และเลือกระหว่าง `raw_id_fields`/
  `autocomplete_fields` ให้เหมาะกับขนาดข้อมูลของ FK ปลายทาง
- ✅ สร้าง Inline Admin ด้วย `TabularInline`/`StackedInline` สำหรับทั้ง `Comment` และ
  through model อย่าง `PostTag`
- ✅ กำหนดฟอร์ม admin เองด้วย `form = CustomModelForm` พร้อม custom validation ทั้งระดับ
  field เดียว (`clean_<field>`) และหลาย field พร้อมกัน (`clean`)
- ✅ เข้าใจข้อจำกัดจริงของ `filter_horizontal`/`filter_vertical` กับ M2M ที่มี `through`
  กำหนดเอง (`admin.E013`) และรู้ทางออกที่ถูกต้อง (Inline ของ through model)
- ✅ ใช้ `date_hierarchy` และ `autocomplete_fields` (พร้อมเข้าใจ `admin.E040`)
- ✅ ประกอบทุกเทคนิคเข้าด้วยกันเป็น Admin Site ที่สมบูรณ์สำหรับบล็อกทั้งระบบ รวมถึงการฝัง
  `Profile` เข้าไปในหน้าแก้ไข `User` มาตรฐานของ Django

### 170.6 Checklist ก่อนไป Part ถัดไป

- [ ] ทุก model (`Category`, `Tag`, `Post`, `Comment`, `Profile`) ปรากฏใน `/admin/` และ
  ใช้งานได้ครบ (ยกเว้น `PostTag` ที่ตั้งใจไม่ลงทะเบียนเดี่ยว)
- [ ] `PostAdmin` มี `list_display`/`list_filter`/`search_fields` ครบและทดสอบค้นหา/กรอง
  ได้จริง
- [ ] ทดลองแก้ `is_published` ผ่าน `list_editable` ในหน้ารายการได้สำเร็จ
- [ ] หน้าแก้ไข `Post` แบ่งเป็น fieldsets และ `created_at`/`updated_at` เป็น readonly
- [ ] `prepopulated_fields` ทำงานถูกต้องเมื่อพิมพ์ title/name ในหน้าเพิ่มข้อมูล
- [ ] `PostTagInline` และ `CommentInline` ปรากฏในหน้าแก้ไข `Post` และบันทึกข้อมูลได้จริง
- [ ] ลองเผยแพร่บทความที่ไม่มีหมวดหมู่ แล้วเห็น validation error จาก `PostAdminForm`
- [ ] `Profile.favorite_posts` ใช้ `filter_horizontal` ได้จริง และเข้าใจว่าทำไม
  `Post.tags` ทำแบบเดียวกันไม่ได้
- [ ] `date_hierarchy` และ `autocomplete_fields` (`category`) ทำงานถูกต้องในหน้า `Post`
- [ ] หน้าแก้ไข `User` มาตรฐานแสดง section "โปรไฟล์" ที่ฝังมาจาก `ProfileInline`

### 170.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: เพิ่ม `CategoryAdmin` ให้มี `list_display` แสดงจำนวนบทความ
ในแต่ละหมวดหมู่ (เขียน method `post_count(self, obj)` ที่คืนค่า `obj.posts.count()`
พร้อม `@admin.display(description='จำนวนบทความ')`) แล้วทดสอบว่าตัวเลขแสดงถูกต้องจริงเมื่อ
เทียบกับที่นับเองผ่าน shell

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม field `is_flagged = models.BooleanField(default=False)`
ให้ `Comment` (อย่าลืม `makemigrations`/`migrate`) แล้วปรับ `CommentAdmin` ให้มี:
`list_display` รวม `is_flagged`, `list_editable = ('is_flagged',)` (ระวังกฎเรื่องคอลัมน์
แรกจากขั้นตอนที่ 163.1), และ `list_filter` ที่กรองเฉพาะคอมเมนต์ที่ถูก flag ได้ ทดสอบว่าติ๊ก
เลือกหลายแถวแล้ว Save ครั้งเดียวได้ผลลัพธ์ตรงกับที่คาดหวัง

**แบบฝึกหัดที่ 3 (Custom Validation)**: แก้ไข `PostAdminForm.clean()` ให้เพิ่มกฎใหม่:
"ห้ามตั้ง `is_published=True` ถ้าบทความไม่มีแท็กเลยสักอัน" — สังเกตว่าใน `clean()` คุณยัง
เข้าถึง M2M ของ instance ที่ยังไม่ถูก save ไม่ได้โดยตรงในบางกรณี (ทดลองแล้วเขียนบันทึกไว้ว่า
เจอปัญหาอะไรบ้าง และแก้ด้วยการ override `save_model()` ของ `ModelAdmin` แทนถ้าจำเป็น
— นี่คือ gotcha จริงที่มักเจอกับ validation ที่เกี่ยวกับ M2M ใน admin form)

**แบบฝึกหัดที่ 4 (ขั้นสูง — ทดลองข้อจำกัดด้วยตัวเอง)**: สร้างแอปทดลองใหม่ชื่อ `sandbox`
(ไม่ต้องเพิ่มใน `INSTALLED_APPS` ของโปรเจกต์จริงถ้าไม่ต้องการ ทำในโปรเจกต์ทดลองแยกต่างหาก
ก็ได้) สร้าง model `A`, `B`, และ `AB` (through model ที่มี field พิเศษ) เชื่อมกันแบบ M2M
ผ่าน `through=` เหมือน `Post`/`Tag`/`PostTag` แล้วลองทำสิ่งต่อไปนี้ทีละอย่างและบันทึก error
message ที่ได้: (1) ใส่ field M2M ใน `fields` ตรง ๆ, (2) ใส่ใน `filter_horizontal`,
(3) ใส่ใน `raw_id_fields`, (4) ใส่ใน `autocomplete_fields` — เปรียบเทียบว่า error ที่ได้
เหมือนหรือต่างกันอย่างไร (ทั้งหมดควรเป็น `admin.E013` เหมือนกัน) เพื่อให้เข้าใจข้อจำกัดนี้
อย่างถ่องแท้จากประสบการณ์ตรง ไม่ใช่แค่จำจากตำรา

### 170.8 คำถามที่พบบ่อย (FAQ)

**Q: Django Admin ใช้ในระบบ Production จริงได้เลยไหม หรือควรใช้แค่ตอนพัฒนา?**
A: ใช้ใน production ได้จริง และบริษัทจำนวนมากก็ทำแบบนั้น (เช่นใช้เป็น internal tool ให้
ทีม content/support จัดการข้อมูล) แต่ต้องระวังเรื่อง **permission** ให้ดี (จำกัดว่าใครเข้าถึง
อะไรได้บ้าง ผ่านระบบ `Groups`/`Permissions` ซึ่งเรียนเต็มรูปแบบใน Part 033) และไม่ควรเปิด
ให้ผู้ใช้ทั่วไป (end user) เข้าถึง Django Admin เด็ดขาด — มันถูกออกแบบมาสำหรับ **ทีมงาน
ภายใน** เท่านั้น ไม่ใช่หน้าเว็บสาธารณะ

**Q: ทำไม `filter_horizontal` ถึงใช้กับ `Post.tags` ไม่ได้ ทั้งที่มันเป็น field เดียวกับที่
เราใช้ `.add()`/`.set()` ใน shell ได้ปกติใน Part 012?**
A: เพราะ `.add()`/`.set()`/`.remove()` เป็นการเรียกผ่าน **Python API ของ Manager** ซึ่ง
Django ให้เราควบคุมเองได้เต็มที่ (รวมถึงใช้ `through_defaults` เติมค่า `added_by` ได้ตามที่
เรียนไปแล้ว) แต่ widget ของ Admin Form เป็นกลไกที่ต้อง**เดา**เองว่าจะเติมค่า field พิเศษของ
through model ให้อัตโนมัติอย่างไรโดยไม่มี input จากผู้ใช้ตรง ๆ — Django เลือกที่จะ**ไม่เดา**
และตัด field ออกจากฟอร์มไปเลยเพื่อความปลอดภัย (ป้องกันการสร้างข้อมูลที่ไม่สมบูรณ์แบบเงียบ ๆ)
แล้วปล่อยให้เราจัดการผ่าน Inline ของ through model แทน ซึ่งให้ control เต็มรูปแบบกว่ามาก

**Q: `list_editable` กับการแก้ไขผ่านหน้า Change form ปกติ ต่างกันตรงไหนในแง่การ trigger
`save()`/signal?**
A: ทั้งสองทางเรียก `Model.save()` เหมือนกันทุกประการ (ไม่ใช่ bulk `QuerySet.update()`)
ดังนั้น logic ใน `save()` ที่เขียนไว้ (เช่น auto-slug) และ signal `pre_save`/`post_save`
(เรียนเต็มใน Part 019) จะถูกเรียกตามปกติทั้งคู่ — `list_editable` แค่เปลี่ยน**อินเทอร์เฟซ**
ที่ผู้ใช้กรอกข้อมูลเข้ามา ไม่ได้เปลี่ยนกลไกการบันทึกเบื้องหลังแต่อย่างใด

**Q: ควรใช้ `raw_id_fields` หรือ `autocomplete_fields` เป็นค่าเริ่มต้นสำหรับโปรเจกต์ใหม่?**
A: เลือก `autocomplete_fields` เป็นอันดับแรกเสมอในโปรเจกต์ปี 2025-2026 เป็นต้นไป เพราะ UX
ดีกว่าอย่างชัดเจนโดยแทบไม่มีต้นทุนเพิ่ม (แค่ต้องมี `search_fields` ที่เหมาะสมอยู่แล้ว ซึ่ง
เป็น good practice ที่ควรทำอยู่แล้ว) `raw_id_fields` ยังมีที่ใช้ในกรณีพิเศษ เช่น เมื่อ
model ปลายทางมี `ModelAdmin` ที่ซับซ้อนเกินกว่าจะรองรับ autocomplete endpoint ได้อย่าง
มีประสิทธิภาพ แต่กรณีแบบนั้นพบได้น้อยมากในทางปฏิบัติ

**Q: ทำไมต้องแยก `PostAdminForm` ไว้ใน `forms.py` แทนที่จะเขียน inline อยู่ใน `admin.py`
เลย?**
A: เป็นเรื่องของ **การจัดระเบียบโค้ด (code organization)** ล้วน ๆ ไม่ใช่ข้อบังคับทาง
เทคนิค — เขียนไว้ใน `admin.py` เลยก็ทำงานได้เหมือนกันทุกประการ แต่การแยกไฟล์ `forms.py`
ทำให้ `ModelForm` เดียวกันนี้**นำไปใช้ซ้ำได้**ในบริบทอื่น เช่น ฟอร์มหน้าเว็บจริงสำหรับให้
ผู้เขียนบล็อกสร้างบทความเองโดยไม่ต้องผ่าน admin เลย ซึ่งเป็นหัวข้อของ **Part 025: Django
Forms และ ModelForm**

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 018: Django Admin ขั้นสูง: Customization และ Actions** จะพาคุณไปไกลกว่า
`ModelAdmin` มาตรฐาน สู่การปรับแต่ง Admin Site ระดับที่ทีมงานมืออาชีพใช้จริง ได้แก่
**Admin Actions** (เลือกหลายแถวแล้วสั่งทำงานพร้อมกัน เช่น "เผยแพร่บทความที่เลือกทั้งหมด")
รวมถึง action ที่ export ข้อมูลเป็น CSV, **custom `SimpleListFilter`** สำหรับตัวกรองที่
ซับซ้อนเกินกว่า `list_filter` มาตรฐานจะรองรับ (เช่น "บทความที่มีคอมเมนต์เกิน 10 อัน"),
การ override `get_queryset()`/`save_model()`/`delete_model()` เพื่อ inject business logic
เข้าไปในทุกจุดของ admin, ระบบ **permission แบบละเอียด** (ให้บาง staff แก้ได้แต่ลบไม่ได้),
และการสร้าง **custom `AdminSite`** ของตัวเองแยกจาก default พร้อม custom template และ
custom index page เตรียมโปรเจกต์บล็อกที่มี Admin สมบูรณ์จาก Part นี้ให้พร้อม เพราะเราจะ
ต่อยอดจาก `PostAdmin`/`CommentAdmin` ตัวเดิมทันทีใน Part ถัดไป!
