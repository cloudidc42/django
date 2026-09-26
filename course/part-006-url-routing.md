# Part 006: URL Routing และ URLconf เบื้องต้น

> **ขั้นตอนที่ 51-60 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เรียนรู้วิธีออกแบบและจัดการระบบ URL Routing ของ Django
> อย่างมืออาชีพ ตั้งแต่ `path()` และ `re_path()` พื้นฐาน, path converters ที่มีมาให้,
> named URLs กับ `reverse()`/`{% url %}`, การแยก `urls.py` ระดับแอปด้วย `include()`,
> URL namespacing, การส่งค่าพิเศษเข้า view, การสร้าง custom path converter ของตัวเอง,
> การจัดการหน้า error แบบกำหนดเอง ไปจนถึงหลักการออกแบบ URL ที่ดีระดับ production
> เมื่อจบ Part นี้ คุณจะออกแบบโครงสร้าง URL ที่สะอาด ขยายง่าย และเป็นมาตรฐาน
> อุตสาหกรรมให้กับแอป `blog` ของคุณได้อย่างสมบูรณ์

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 51: `path()` และ `re_path()` พื้นฐาน ความแตกต่างและเมื่อไหร่ควรใช้อะไร
- ขั้นตอนที่ 52: URL parameters และ path converters ที่มีมาให้ (`int`, `str`, `slug`, `uuid`, `path`)
- ขั้นตอนที่ 53: Named URLs และฟังก์ชัน `reverse()` / template tag `{% url %}` (ทำไมไม่ควร hardcode URL string)
- ขั้นตอนที่ 54: ใช้ `include()` แยก urls.py ระดับ app ออกจาก project (โครงสร้าง URL แบบมืออาชีพ)
- ขั้นตอนที่ 55: URL Namespacing ด้วย `app_name` เพื่อป้องกันชื่อ URL ชนกันข้ามแอป
- ขั้นตอนที่ 56: การส่งค่าพิเศษ (extra options) เข้า view ผ่าน urls.py
- ขั้นตอนที่ 57: การสร้าง Custom Path Converter ของตัวเอง
- ขั้นตอนที่ 58: การจัดการหน้า Error (404, 500, 403, 400) แบบกำหนดเอง (handler404 ฯลฯ)
- ขั้นตอนที่ 59: Best Practice การออกแบบ URL (trailing slash, RESTful naming, SEO-friendly slugs)
- ขั้นตอนที่ 60: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 51: `path()` และ `re_path()` พื้นฐาน ความแตกต่างและเมื่อไหร่ควรใช้อะไร

### 51.1 ทบทวน: URLconf คืออะไร

ใน Part 003 คุณสร้างโปรเจกต์ Django ด้วย `django-admin startproject` และใน Part 005
คุณสร้างแอป `blog` ด้วย `python manage.py startapp blog` แล้วลงทะเบียนใน
`INSTALLED_APPS` เรียบร้อยแล้ว ตอนนี้ถึงเวลาเจาะลึกไฟล์ที่สำคัญที่สุดไฟล์หนึ่งของ
Django นั่นคือ **`urls.py`** หรือที่เรียกกันว่า **URLconf (URL configuration)**

URLconf คือ "สมุดเส้นทาง" ที่บอก Django ว่า เมื่อมี HTTP request เข้ามาที่ path ไหน
ให้ส่งต่อไปให้ view function/class ตัวไหนเป็นคนจัดการ ทุกโปรเจกต์ Django ต้องมี
ตัวแปรชื่อ `urlpatterns` ซึ่งเป็น list ของ `path()` หรือ `re_path()` ที่ประกาศไว้ใน
ไฟล์ที่ตั้งค่าไว้ที่ `ROOT_URLCONF` ใน `settings.py` (ค่าเริ่มต้นคือ `config.urls`
หรือชื่อโปรเจกต์ของคุณ ตามด้วย `.urls`)

### 51.2 ทบทวนโครงสร้างโปรเจกต์ปัจจุบันของคุณ

ณ จุดนี้ โปรเจกต์ของคุณควรมีโครงสร้างประมาณนี้ (ต่อยอดจาก Part 004-005):

```
django-mastery-course/
├── venv/
├── config/                  # โปรเจกต์หลัก (สร้างใน Part 004)
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py               # ROOT_URLCONF ชี้มาที่นี่
│   ├── asgi.py
│   └── wsgi.py
├── blog/                     # แอปแรกของเรา (สร้างใน Part 005)
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
├── requirements.txt
└── .gitignore
```

และไฟล์สำคัญ 2 ไฟล์มีเนื้อหาประมาณนี้:

```python
# config/urls.py (ก่อนแก้ไขใน Part นี้)
from django.contrib import admin
from django.urls import path
from blog import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('posts/', views.post_list),
]
```

```python
# blog/views.py (จาก Part 005 - ยังเป็น view แบบง่าย ๆ)
from django.http import HttpResponse


def post_list(request):
    return HttpResponse("<h1>รายการบทความทั้งหมด</h1>")
```

> **หมายเหตุสำคัญ**: Part นี้เราจะยังไม่ query ฐานข้อมูลจริง เพราะ Django Model
> จะเริ่มเรียนอย่างจริงจังใน **Phase 2 (Part 011)** ตอนนี้เราจะโฟกัสที่ "การจัดเส้นทาง
> URL" ล้วน ๆ โดยใช้ view แบบง่ายที่ยังไม่ผูกกับฐานข้อมูล ส่วนรายละเอียดการเขียน view
> แบบ function-based เชิงลึกจะไปเรียนต่อใน **Part 007** ทันทีหลังจาก Part นี้

### 51.3 `path()`: ฟังก์ชันมาตรฐานสำหรับกำหนด URL

ตั้งแต่ Django 2.0 เป็นต้นมา ฟังก์ชันหลักที่ใช้ประกาศ URL pattern คือ `path()`
ซึ่งมี signature ดังนี้:

```python
path(route, view, kwargs=None, name=None)
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `route` | สตริงที่ใช้จับคู่กับ path ของ URL (ไม่รวม domain และไม่รวม query string `?...`) |
| `view` | view function/class-based-view ที่จะถูกเรียกเมื่อ route ตรงกัน |
| `kwargs` | (optional) dict ของค่าพิเศษที่จะส่งเข้า view เพิ่มเติม (จะเรียนละเอียดในขั้นตอนที่ 56) |
| `name` | (optional) ชื่อ URL สำหรับอ้างอิงด้วย `reverse()`/`{% url %}` (ขั้นตอนที่ 53) |

ตัวอย่างการใช้งานพื้นฐาน:

```python
# config/urls.py
from django.contrib import admin
from django.urls import path
from blog import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', views.post_list, name='home'),
    path('about/', views.about, name='about'),
    path('contact/', views.contact, name='contact'),
]
```

จุดสังเกต:
- `route` **ไม่ต้องมี `/` นำหน้า** เพราะ Django ตัด domain และ leading slash ออกให้แล้ว
- แต่ควรมี `/` ต่อท้ายเสมอ (จะอธิบายเหตุผลละเอียดในขั้นตอนที่ 59)
- `path('', ...)` หมายถึง URL ราก (root) ของ URLconf นั้น ๆ เช่น `https://example.com/`

### 51.4 `re_path()`: เมื่อ Regular Expression จำเป็น

ก่อน Django 2.0 การประกาศ URL ทุกแบบต้องใช้ **regular expression (regex)** ผ่านฟังก์ชัน
`url()` (ปัจจุบันถูกลบออกจาก Django ตั้งแต่เวอร์ชัน 4.0 แล้ว) ปัจจุบัน Django แนะนำให้ใช้
`path()` เป็นค่าเริ่มต้น แต่ยังคงมี `re_path()` ไว้สำหรับกรณีที่ pattern มีความซับซ้อนเกินกว่า
ที่ path converters (ขั้นตอนที่ 52) จะรองรับได้

```python
from django.urls import path, re_path
from blog import views

urlpatterns = [
    # path() แบบใหม่ ใช้ converter
    path('posts/<int:pk>/', views.post_detail, name='post_detail'),

    # re_path() แบบ regex เทียบเท่ากัน
    re_path(r'^posts/(?P<pk>[0-9]+)/$', views.post_detail, name='post_detail'),
]
```

`re_path()` มี signature เหมือน `path()` ทุกประการ ต่างกันแค่ `route` เป็น regex
pattern (ใช้ raw string `r'...'` เสมอ) และตั้งชื่อกลุ่มด้วย `(?P<name>...)` แทนการใช้
`<converter:name>`

ตัวอย่างกรณีที่ **regex ทำได้แต่ converter ธรรมดาทำไม่ได้**:

```python
from django.urls import re_path
from blog import views

urlpatterns = [
    # รองรับทั้งนามสกุล .html และไม่มีนามสกุล ในรูปแบบเดียว
    re_path(r'^posts/(?P<slug>[\w-]+)(?:\.html)?/$', views.post_detail),

    # จับคู่เฉพาะ path ที่ไม่ขึ้นต้นด้วย "admin" หรือ "api" (negative lookahead)
    re_path(r'^(?!admin|api)(?P<page>[\w-]+)/$', views.generic_page),
]
```

### 51.5 ตารางเปรียบเทียบ `path()` กับ `re_path()`

| ประเด็น | `path()` | `re_path()` |
|---|---|---|
| รูปแบบ pattern | ใช้ converter อ่านง่าย เช่น `<int:pk>` | ใช้ regex เต็มรูปแบบ |
| ความอ่านง่าย | อ่านง่ายกว่ามาก | อ่านยากกว่าสำหรับ pattern ซับซ้อน |
| ความยืดหยุ่น | จำกัดตาม converter ที่มี (หรือที่สร้างเอง) | ยืดหยุ่นสูงสุด รองรับทุก regex |
| ประสิทธิภาพ | เร็วกว่าเล็กน้อย (คอมไพล์ converter ภายใน) | เร็วพอ ๆ กัน (regex ถูกคอมไพล์ล่วงหน้าเช่นกัน) |
| แนะนำให้ใช้เมื่อ | 90% ของกรณีทั่วไป | pattern ซับซ้อน, optional segments, lookahead/lookbehind, backward compatibility กับโค้ดเก่า |
| Django version | 2.0+ | ทุกเวอร์ชัน (แทนที่ `url()` เดิม) |

### 51.6 กฎการจับคู่ URL: ลำดับสำคัญมาก

Django ไล่ตรวจ `urlpatterns` **จากบนลงล่าง** และหยุดที่ pattern แรกที่ตรงกัน (match)
เป็นอันดับแรกเท่านั้น ดังนั้น **ลำดับการประกาศมีผลโดยตรง**

```python
# ผิด! path เฉพาะเจาะจงถูกวางไว้หลัง path ทั่วไป
urlpatterns = [
    path('posts/<slug:slug>/', views.post_detail),   # จะจับ "new" เป็น slug ไปก่อน!
    path('posts/new/', views.post_create),            # ไม่มีทางถูกเรียกถึง
]
```

```python
# ถูกต้อง! path เฉพาะเจาะจงต้องมาก่อน path ทั่วไปเสมอ
urlpatterns = [
    path('posts/new/', views.post_create, name='post_create'),
    path('posts/<slug:slug>/', views.post_detail, name='post_detail'),
]
```

**กฎทอง**: เรียง URL pattern จาก **เฉพาะเจาะจงที่สุดไปหาทั่วไปที่สุด** เสมอ

### 51.7 เมื่อไหร่ควรใช้ `path()` และเมื่อไหร่ควรใช้ `re_path()`

- **ใช้ `path()` เป็นค่าเริ่มต้นเสมอ** เพราะอ่านง่าย ดูแลรักษาง่าย และครอบคลุมงานส่วนใหญ่
  ของเว็บแอปพลิเคชันทั่วไป (list, detail, create, update, delete)
- **ใช้ `re_path()` เมื่อ**:
  1. ต้องการ pattern ที่ converter มาตรฐานไม่รองรับ (เช่น optional segment, alternation)
  2. กำลังดูแลโค้ดเก่าที่เขียนด้วย `url()`/regex อยู่แล้ว และยังไม่พร้อม refactor ทั้งหมด
  3. ต้องการ negative lookahead/lookbehind เพื่อกันไม่ให้ pattern อื่นชนกัน
- หากพบว่าต้องเขียน regex ซับซ้อนบ่อย ๆ ให้พิจารณาสร้าง **custom path converter** แทน
  (จะเรียนในขั้นตอนที่ 57) เพราะดูแลรักษาง่ายกว่าและนำกลับมาใช้ซ้ำได้

---

## ขั้นตอนที่ 52: URL parameters และ path converters ที่มีมาให้ (`int`, `str`, `slug`, `uuid`, `path`)

### 52.1 Path Converter คืออะไร

**Path converter** คือส่วนที่บอก Django ว่าส่วนหนึ่งของ URL เป็นค่าพารามิเตอร์แบบไหน
เขียนในรูปแบบ `<converter:name>` ฝังอยู่ใน route string เมื่อ Django จับคู่ได้ จะแปลงค่า
นั้นเป็น Python type ที่เหมาะสม แล้วส่งเข้า view function เป็น keyword argument ชื่อ `name`

```python
path('posts/<int:pk>/', views.post_detail)
#              └─────┘
#          converter:name
```

เมื่อผู้ใช้เข้า `/posts/42/` Django จะเรียก `views.post_detail(request, pk=42)`
โดย `42` ถูกแปลงเป็น **int** ให้อัตโนมัติ (ไม่ใช่ string `"42"`)

### 52.2 ตาราง Path Converter มาตรฐานที่ Django มีมาให้

| Converter | จับคู่กับ | ตัวอย่างค่าที่ match | ชนิดข้อมูลที่ส่งเข้า view | Regex เทียบเท่า |
|---|---|---|---|---|
| `str` | สตริงใด ๆ ที่ไม่มี `/` (ค่าเริ่มต้นถ้าไม่ระบุ converter) | `hello`, `abc123` | `str` | `[^/]+` |
| `int` | จำนวนเต็มบวก (0 และมากกว่า) | `0`, `42`, `1000` | `int` | `[0-9]+` |
| `slug` | ตัวอักษร, ตัวเลข, ขีดกลาง `-`, ขีดล่าง `_` (มาตรฐาน SEO slug) | `my-first-post`, `hello_world` | `str` | `[-a-zA-Z0-9_]+` |
| `uuid` | UUID มาตรฐาน (มีขีดกลางคั่น) | `075194d3-6885-417e-a8a8-6c931e272f00` | `uuid.UUID` | รูปแบบ UUID |
| `path` | สตริงใด ๆ **รวมทั้ง `/`** (ใช้จับ path ที่เหลือทั้งหมด) | `docs/2024/report.pdf` | `str` | `.+` |

### 52.3 ตัวอย่างการใช้งานจริงกับแอป `blog`

มาอัปเดต `blog/views.py` และ `config/urls.py` ให้รองรับพารามิเตอร์หลายรูปแบบ:

```python
# blog/views.py
from django.http import HttpResponse


def post_list(request):
    return HttpResponse("<h1>รายการบทความทั้งหมด</h1>")


def post_detail_by_id(request, pk):
    return HttpResponse(f"<h1>บทความหมายเลข {pk}</h1><p>ชนิดข้อมูล: {type(pk).__name__}</p>")


def post_detail_by_slug(request, slug):
    return HttpResponse(f"<h1>บทความ: {slug}</h1><p>ชนิดข้อมูล: {type(slug).__name__}</p>")


def post_detail_by_uuid(request, public_id):
    return HttpResponse(
        f"<h1>บทความ (public_id={public_id})</h1>"
        f"<p>ชนิดข้อมูล: {type(public_id).__name__}</p>"
    )


def media_file(request, file_path):
    return HttpResponse(f"<h1>เปิดไฟล์: {file_path}</h1>")
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path
from blog import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', views.post_list, name='home'),

    # int converter - เหมาะกับ primary key
    path('posts/id/<int:pk>/', views.post_detail_by_id, name='post_detail_by_id'),

    # slug converter - เหมาะกับ URL ที่อ่านง่าย เป็นมิตรกับ SEO
    path('posts/<slug:slug>/', views.post_detail_by_slug, name='post_detail_by_slug'),

    # uuid converter - เหมาะกับ public-facing ID ที่ไม่ต้องการให้เดา sequence ได้
    path('posts/ref/<uuid:public_id>/', views.post_detail_by_uuid, name='post_detail_by_uuid'),

    # path converter - จับ path ที่เหลือทั้งหมด รวม "/"
    path('media/<path:file_path>', views.media_file, name='media_file'),
]
```

ทดสอบด้วย `python manage.py runserver` แล้วลองเข้า URL เหล่านี้:

```
http://127.0.0.1:8000/posts/id/42/
http://127.0.0.1:8000/posts/my-first-django-post/
http://127.0.0.1:8000/posts/ref/075194d3-6885-417e-a8a8-6c931e272f00/
http://127.0.0.1:8000/media/uploads/2026/09/photo.jpg
```

### 52.4 การรับพารามิเตอร์หลายตัวใน 1 path เดียว

Path หนึ่งสามารถมีหลาย converter พร้อมกันได้ ชื่อพารามิเตอร์ต้องไม่ซ้ำกัน และต้องตรงกับ
ชื่อ keyword argument ใน view function ทุกตัว:

```python
# urls.py
path('posts/<int:year>/<int:month>/<slug:slug>/', views.post_detail_full, name='post_detail_full'),
```

```python
# views.py
def post_detail_full(request, year, month, slug):
    return HttpResponse(f"<h1>{slug}</h1><p>เผยแพร่เมื่อ {month}/{year}</p>")
```

เข้าถึงได้ด้วย URL เช่น `/posts/2026/09/my-first-django-post/`

### 52.5 ความแตกต่างระหว่าง Converter กับ Regex Named Group

ทั้งสองแบบทำหน้าที่เดียวกันคือ "ดึงค่าจาก URL ส่งเข้า view" แต่ต่างกันที่ไวยากรณ์และ
ระดับ abstraction:

| แบบ path converter | แบบ re_path named group |
|---|---|
| `<int:pk>` | `(?P<pk>[0-9]+)` |
| `<slug:slug>` | `(?P<slug>[-a-zA-Z0-9_]+)` |
| อ่านง่าย บอก "ความหมาย" ของข้อมูลชัดเจน | ต้องรู้ regex จึงจะเข้าใจ pattern |
| แปลงชนิดข้อมูลให้อัตโนมัติ (`int`, `uuid.UUID`) | ได้ค่าเป็น `str` เสมอ ต้องแปลงเองใน view |

**คำแนะนำ**: ใช้ converter เป็นหลักเสมอ ใช้ regex เฉพาะกรณีที่ converter ไม่รองรับ
เพราะโค้ดที่อ่านง่ายคือโค้ดที่ดูแลรักษาง่ายในระยะยาว

---

## ขั้นตอนที่ 53: Named URLs และฟังก์ชัน `reverse()` / template tag `{% url %}`

### 53.1 ปัญหาของการ Hardcode URL String

ลองจินตนาการว่าคุณมีโค้ดแบบนี้กระจายอยู่หลายสิบที่ในโปรเจกต์:

```python
# ตัวอย่างที่ไม่ดี - hardcode URL string ตรง ๆ
from django.http import HttpResponseRedirect

def create_post(request):
    # ... บันทึกโพสต์ ...
    return HttpResponseRedirect('/posts/my-first-post/')
```

```html
<!-- ตัวอย่างที่ไม่ดี - hardcode ใน template -->
<a href="/posts/my-first-post/">อ่านต่อ</a>
```

ปัญหาคือ ถ้าวันหนึ่งคุณเปลี่ยนโครงสร้าง URL จาก `/posts/<slug>/` เป็น
`/blog/articles/<slug>/` คุณต้องไปไล่แก้ทุกที่ที่ hardcode URL ไว้ ซึ่งเสี่ยงต่อการ
พลาดตกหล่น (ละเมิดหลักการ **DRY** ที่เราพูดถึงตั้งแต่ Part 001)

### 53.2 การตั้งชื่อ URL ด้วย `name=`

ทางแก้คือ**ตั้งชื่อ (name)** ให้กับทุก URL pattern แล้วอ้างอิงด้วยชื่อแทนการเขียน
path string ตรง ๆ:

```python
# urls.py
path('posts/<slug:slug>/', views.post_detail_by_slug, name='post_detail'),
```

ชื่อ `'post_detail'` นี้เป็นตัวตายตัวแทนของ path จริง ไม่ว่า path จะเปลี่ยนไปยังไง
ตราบใดที่ `name` ยังเหมือนเดิม โค้ดที่อ้างอิงถึงมันก็ยังทำงานถูกต้อง

### 53.3 ฟังก์ชัน `reverse()`: ใช้ใน Python Code

`reverse()` แปลงชื่อ URL กลับเป็น path string จริง ใช้ในไฟล์ `.py` เช่น views, forms,
tests:

```python
from django.urls import reverse
from django.http import HttpResponseRedirect


def create_post(request):
    # ... บันทึกโพสต์ ...
    url = reverse('post_detail', kwargs={'slug': 'my-first-post'})
    # url = '/posts/my-first-post/'
    return HttpResponseRedirect(url)
```

Django มี shortcut ที่สะดวกกว่าคือ `redirect()` ซึ่งเรียก `reverse()` ให้อัตโนมัติ
ถ้าอาร์กิวเมนต์แรกเป็นชื่อ URL:

```python
from django.shortcuts import redirect


def create_post(request):
    # ... บันทึกโพสต์ ...
    return redirect('post_detail', slug='my-first-post')
```

### 53.4 Template Tag `{% url %}`: ใช้ใน Template

ใน template ใช้ `{% url %}` แทนการเขียน path string ตรง ๆ:

```html
<!-- templates/blog/post_list.html -->
<a href="{% url 'post_detail' slug=post.slug %}">อ่านต่อ</a>

<!-- ไม่มีพารามิเตอร์ -->
<a href="{% url 'home' %}">หน้าแรก</a>
```

### 53.5 `reverse()` กับพารามิเตอร์แบบ `args` และ `kwargs`

```python
from django.urls import reverse

# แบบ kwargs (แนะนำ เพราะอ่านง่ายและไม่พลาดลำดับ)
reverse('post_detail', kwargs={'slug': 'hello-django'})

# แบบ args (ต้องเรียงลำดับให้ตรงกับที่ประกาศใน urls.py)
reverse('post_detail_full', args=[2026, 9, 'hello-django'])
```

**คำแนะนำ**: ใช้ `kwargs` เสมอเมื่อทำได้ เพราะชัดเจนกว่าและลดความผิดพลาดจากลำดับ
พารามิเตอร์สลับกัน

### 53.6 `reverse_lazy()`: สำหรับกรณีที่ URL ยังไม่พร้อมใช้ตอน import

บางครั้ง `reverse()` ถูกเรียกตอนที่ URLconf ยังโหลดไม่เสร็จ (เช่น ใน class attribute
ระดับโมดูลของ Class-Based View ที่เราจะเรียนใน Part 021) กรณีนี้ต้องใช้
`reverse_lazy()` แทน ซึ่งจะ**ประเมินค่า URL ตอนที่ถูกใช้งานจริง** ไม่ใช่ตอน import:

```python
from django.urls import reverse_lazy

# ตัวอย่างที่จะได้เจอบ่อยเมื่อเรียน Class-Based Views ใน Phase 3
class PostDeleteView:
    success_url = reverse_lazy('post_list')  # ต้องใช้ lazy เพราะเป็น class attribute
```

จำง่าย ๆ ว่า: **ใน view function ปกติใช้ `reverse()`, ใน class attribute หรือที่ที่
โค้ดรันตอน import module ให้ใช้ `reverse_lazy()`**

### 53.7 ทำไมการใช้ Named URL คือ Best Practice ที่ไม่มีข้อยกเว้น

| ข้อดี | อธิบาย |
|---|---|
| Maintainability | เปลี่ยนโครงสร้าง URL ได้โดยไม่กระทบโค้ดที่เรียกใช้ |
| ลด Bug | ไม่มีการพิมพ์ path ผิด (typo) เพราะ Django จะ error ทันทีถ้าชื่อ URL ไม่มีอยู่จริง |
| Testability | เขียนเทสต์โดยอ้างอิงชื่อ URL ได้ ไม่ต้องสนใจ path จริง |
| อ่านง่าย | `reverse('post_detail', ...)` สื่อความหมายชัดกว่า `/posts/xxx/` |

**กฎเหล็กของหลักสูตรนี้ตั้งแต่ Part นี้เป็นต้นไป: ห้าม hardcode URL string ใน
โค้ด Python หรือ template เด็ดขาด ต้องใช้ `reverse()`/`redirect()`/`{% url %}`
ผ่านชื่อ URL เสมอ**

---

## ขั้นตอนที่ 54: ใช้ `include()` แยก urls.py ระดับ app ออกจาก project (โครงสร้าง URL แบบมืออาชีพ)

### 54.1 ปัญหาของการยัดทุก URL ไว้ใน `config/urls.py` ไฟล์เดียว

ตอนนี้ `config/urls.py` ของเรามี URL ของแอป `blog` ปนอยู่กับของโปรเจกต์เอง เมื่อโปรเจกต์
โตขึ้นและมีหลายแอป (เช่น `blog`, `accounts`, `shop`, `api`) ไฟล์นี้จะยาวขึ้นเรื่อย ๆ
จนดูแลยาก และขัดกับหลักการ **แยกความรับผิดชอบ (Separation of Concerns)** ที่ Django
ยึดถือ

### 54.2 สร้าง `blog/urls.py` แยกออกมา

สร้างไฟล์ใหม่ในแอป `blog`:

```python
# blog/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('id/<int:pk>/', views.post_detail_by_id, name='post_detail_by_id'),
    path('<slug:slug>/', views.post_detail_by_slug, name='post_detail'),
    path('ref/<uuid:public_id>/', views.post_detail_by_uuid, name='post_detail_by_uuid'),
]
```

สังเกตว่าเราไม่เขียน prefix `posts/` ซ้ำในทุก path เพราะ prefix นี้จะถูกกำหนด
จากฝั่ง `config/urls.py` แทน (ดูขั้นตอนถัดไป)

### 54.3 แก้ `config/urls.py` ให้ใช้ `include()`

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('posts/', include('blog.urls')),
]
```

`include('blog.urls')` บอก Django ว่า: "ทุก path ที่ขึ้นต้นด้วย `posts/` ให้ไปหา
pattern ต่อในไฟล์ `blog/urls.py` โดยตัด `posts/` ออกก่อนแล้วค่อยจับคู่ที่เหลือ"

ผลลัพธ์ URL ที่ได้จริง:

| Pattern ใน `blog/urls.py` | Prefix จาก `config/urls.py` | URL จริงที่เข้าถึงได้ |
|---|---|---|
| `''` | `posts/` | `/posts/` |
| `'id/<int:pk>/'` | `posts/` | `/posts/id/42/` |
| `'<slug:slug>/'` | `posts/` | `/posts/hello-django/` |
| `'ref/<uuid:public_id>/'` | `posts/` | `/posts/ref/075194d3-.../` |

### 54.4 `include()` กับ `kwargs` ที่ใช้ร่วมกันทั้งกลุ่ม

`include()` ยังรับ dict ของ kwargs ที่จะถูกส่งเข้า **ทุก view** ภายใต้กลุ่มนั้นได้ด้วย
(ใช้น้อยในทางปฏิบัติ แต่ควรรู้จักไว้):

```python
# config/urls.py
urlpatterns = [
    path('posts/', include('blog.urls')),
]
```

```python
# blog/urls.py - รับค่าพิเศษผ่าน 3rd element ของ tuple
urlpatterns = [
    path('', views.post_list, {'is_public': True}),
]
```

(เราจะเรียนเรื่องการส่ง extra options อย่างละเอียดในขั้นตอนที่ 56)

### 54.5 โครงสร้าง URL แบบมืออาชีพสำหรับโปรเจกต์ที่มีหลายแอป

เมื่อโปรเจกต์มีหลายแอป โครงสร้างที่แนะนำคือให้ **`config/urls.py` ทำหน้าที่เป็นแค่
"สารบัญ" ที่ include ไปยังแต่ละแอป** โดยไม่มี view logic ของตัวเองปนอยู่เลย:

```python
# config/urls.py - สมมติว่ามีหลายแอปในอนาคต (accounts จะเรียนใน Phase 4)
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),      # หน้า static เช่น home, about
    path('posts/', include('blog.urls')),  # ทุกอย่างเกี่ยวกับบทความ
    path('accounts/', include('accounts.urls')),  # login, register (Phase 4)
]
```

โครงสร้างแบบนี้มีข้อดีคือ:

- **แต่ละแอปดูแล URL ของตัวเอง** ทีมงานที่รับผิดชอบแอป `blog` ไม่ต้องแตะ
  `config/urls.py` เลย
- **Reusable**: สามารถยกแอป `blog` ทั้งโฟลเดอร์ไปใช้ในโปรเจกต์อื่นได้ทันที
  เพราะ URL ของมันไม่ได้ผูกกับโปรเจกต์ใดโปรเจกต์หนึ่ง
- **Merge conflict น้อยลง**: เมื่อทำงานเป็นทีม แต่ละคนแก้ `urls.py` ของแอปตัวเอง
  โอกาสชนกันใน Git ต่ำกว่าการแก้ไฟล์เดียวร่วมกัน

---

## ขั้นตอนที่ 55: URL Namespacing ด้วย `app_name` เพื่อป้องกันชื่อ URL ชนกันข้ามแอป

### 55.1 ปัญหา: ชื่อ URL ชนกันเมื่อมีหลายแอป

สมมติในอนาคตคุณมีแอป `shop` ที่ขายสินค้า และมี URL ชื่อ `'detail'` เหมือนกับแอป
`blog`:

```python
# blog/urls.py
urlpatterns = [
    path('<slug:slug>/', views.post_detail, name='detail'),
]
```

```python
# shop/urls.py
urlpatterns = [
    path('<slug:slug>/', views.product_detail, name='detail'),
]
```

เมื่อคุณเรียก `reverse('detail')` Django จะงงว่าคุณหมายถึง `detail` ของแอปไหน
(โดยทั่วไปจะใช้ตัวที่ประกาศทีหลังสุดทับตัวก่อนหน้า ซึ่งเป็น bug ที่ตรวจจับยากมาก)

### 55.2 ทางแก้: กำหนด `app_name`

เพิ่มตัวแปร `app_name` ไว้บนสุดของไฟล์ `urls.py` ระดับแอป:

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'   # กำหนด application namespace

urlpatterns = [
    path('', views.post_list, name='list'),
    path('id/<int:pk>/', views.post_detail_by_id, name='detail_by_id'),
    path('<slug:slug>/', views.post_detail_by_slug, name='detail'),
    path('ref/<uuid:public_id>/', views.post_detail_by_uuid, name='detail_by_uuid'),
]
```

จากนั้นการอ้างอิงต้องระบุ namespace ด้วยเครื่องหมาย `:` เสมอ:

```python
# ใน views.py หรือ Python code
from django.urls import reverse

reverse('blog:detail', kwargs={'slug': 'hello-django'})
# ผลลัพธ์: '/posts/hello-django/'
```

```html
<!-- ใน template -->
<a href="{% url 'blog:detail' slug=post.slug %}">อ่านต่อ</a>
<a href="{% url 'blog:list' %}">บทความทั้งหมด</a>
```

### 55.3 การใช้ `namespace=` ตอน `include()` (ไม่บังคับถ้ามี `app_name` แล้ว)

ถ้าไฟล์ `urls.py` ของแอปมี `app_name` กำหนดไว้แล้ว Django จะใช้ค่านั้นเป็น namespace
โดยอัตโนมัติ ไม่จำเป็นต้องระบุซ้ำใน `include()`:

```python
# config/urls.py
urlpatterns = [
    path('posts/', include('blog.urls')),   # namespace='blog' ถูกดึงมาจาก app_name อัตโนมัติ
]
```

แต่ในบางกรณีคุณอาจต้องการ **override namespace** ให้ต่างจาก `app_name` เช่น
เมื่อ include แอปเดียวกันซ้ำหลายครั้งภายใต้ prefix ต่างกัน:

```python
# config/urls.py
urlpatterns = [
    path('posts/', include('blog.urls', namespace='blog')),
    path('archive/posts/', include('blog.urls', namespace='blog_archive')),
]
```

### 55.4 Instance Namespace vs Application Namespace (ระดับเข้าใจเชิงลึก)

Django แยกแนวคิด namespace เป็น 2 ระดับ:

| ประเภท | ความหมาย | ตัวอย่างการเรียก |
|---|---|---|
| **Application namespace** | ชื่อของ "แอป" โดยรวม กำหนดด้วย `app_name` | `reverse('blog:detail')` |
| **Instance namespace** | ชื่อของ "การ include ครั้งหนึ่ง ๆ" กำหนดด้วย `namespace=` ตอน include | `reverse('blog_archive:detail')` |

ถ้าไม่ระบุ `namespace=` ตอน include, instance namespace จะเท่ากับ application
namespace โดยอัตโนมัติ ซึ่งเป็นกรณีปกติ 95% ของการใช้งานจริง คุณจะเจอ instance
namespace ที่ต่างกันเฉพาะตอนที่ include แอปเดียวกันซ้ำหลายจุดเท่านั้น

### 55.5 ทำไมต้องใช้ `app_name` เสมอตั้งแต่ต้น

**กฎของหลักสูตรนี้: ทุกแอปที่มี `urls.py` ของตัวเอง ต้องกำหนด `app_name` เสมอ
ไม่มีข้อยกเว้น** แม้จะมีแอปเดียวในโปรเจกต์ตอนนี้ก็ตาม เพราะ:

1. ป้องกันปัญหาชื่อชนกันเมื่อโปรเจกต์โตขึ้นในอนาคต (และแทบทุกโปรเจกต์จริงจะโตขึ้นเสมอ)
2. ทำให้ชัดเจนว่า URL แต่ละตัวเป็นของแอปไหน อ่านโค้ดเข้าใจง่ายขึ้น
3. เป็นแนวทางที่เอกสารทางการของ Django แนะนำ (Django Coding Style)

---

## ขั้นตอนที่ 56: การส่งค่าพิเศษ (extra options) เข้า view ผ่าน urls.py

### 56.1 รูปแบบ `path(route, view, kwargs, name)`

พารามิเตอร์ตัวที่ 3 ของ `path()`/`re_path()` (ตำแหน่ง `kwargs`) คือ dict ของค่าคงที่
ที่จะถูกส่งเข้า view เป็น keyword argument เพิ่มเติม **ทุกครั้ง** ที่ URL pattern นั้น
ถูกเรียก โดยไม่ขึ้นกับค่าที่ผู้ใช้ส่งมาใน URL เลย

```python
path('route/', view_function, {'key': 'value'}, name='some_name')
```

### 56.2 ตัวอย่างการใช้งานจริง: View เดียวกัน แสดงผลต่าง template

สมมติเราต้องการหน้า "บทความทั้งหมด" และ "บทความเด่น" ที่ใช้ logic การดึงข้อมูล
คล้ายกันมาก แต่ต่างกันแค่ query กับ template สามารถใช้ view เดียวกัน แล้วแยกด้วย
extra options ได้:

```python
# blog/views.py
from django.http import HttpResponse


def post_collection(request, featured_only=False):
    if featured_only:
        return HttpResponse("<h1>บทความเด่นประจำสัปดาห์</h1>")
    return HttpResponse("<h1>บทความทั้งหมด</h1>")
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_collection, {'featured_only': False}, name='list'),
    path('featured/', views.post_collection, {'featured_only': True}, name='featured'),
]
```

ทั้งสอง URL เรียก view เดียวกัน แต่ view ได้รับค่า `featured_only` ต่างกันตามที่กำหนด
ไว้ใน `urls.py` — เป็นวิธี "reuse โค้ด view" โดยไม่ต้องเขียนฟังก์ชันซ้ำ

### 56.3 ตัวอย่างที่ใช้บ่อยในโลกจริง: การส่ง `template_name` เข้า view

```python
# blog/views.py
from django.shortcuts import render


def render_static_page(request, template_name):
    return render(request, template_name)
```

```python
# blog/urls.py
urlpatterns = [
    path('about/', views.render_static_page,
         {'template_name': 'blog/about.html'}, name='about'),
    path('privacy/', views.render_static_page,
         {'template_name': 'blog/privacy.html'}, name='privacy'),
]
```

### 56.4 ข้อควรระวัง

1. **View function ต้องรับ keyword argument นั้นได้เสมอ** ไม่เช่นนั้นจะเกิด
   `TypeError` ทันทีที่ URL ถูกเรียก (ไม่ใช่ตอน startup ทำให้ bug นี้อาจไม่ถูกเจอ
   จนกว่าจะมีคน request จริง — ควรมี test ครอบคลุมเสมอ ดังที่จะเรียนใน Phase 7)
2. **ค่าจาก URL parameter (converter) กับค่าจาก `kwargs` ห้ามใช้ชื่อซ้ำกัน**
   มิฉะนั้น Django จะ raise error ตอนจับคู่ URL
3. **อย่าใช้ extra options พร่ำเพรื่อ** ถ้าพบว่า view ต้องรับเงื่อนไขเยอะเกินไป
   จนโค้ดใน view เต็มไปด้วย `if/else` ควรพิจารณาแยกเป็นคนละ view function แทน
   จะอ่านง่ายและเทสต์ง่ายกว่า

### 56.5 ทางเลือกอื่นที่มักดีกว่าในโลกจริง (เกริ่นล่วงหน้า)

เมื่อคุณเรียน **Class-Based Views** ใน Part 021-024 คุณจะพบว่า CBV มีวิธีที่สะอาดกว่า
ในการ "reuse view logic ด้วยค่าที่ต่างกัน" ผ่านการเรียก `.as_view()` พร้อม attribute:

```python
# ตัวอย่างที่จะเจอใน Phase 3 (ยังไม่ต้องเข้าใจตอนนี้)
path('featured/', PostListView.as_view(featured_only=True), name='featured'),
```

extra options ผ่าน `urls.py` แบบ function-based ที่เรียนในขั้นตอนนี้ ยังคงเป็นเครื่องมือ
ที่มีประโยชน์และใช้งานได้จริง แต่เมื่อโปรเจกต์ซับซ้อนขึ้น CBV มักจะเป็นทางเลือกที่
maintain ง่ายกว่า

---

## ขั้นตอนที่ 57: การสร้าง Custom Path Converter ของตัวเอง

### 57.1 เมื่อไหร่ Built-in Converter ไม่พอ

Converter มาตรฐาน 5 ตัว (`str`, `int`, `slug`, `uuid`, `path`) ครอบคลุมกรณีทั่วไป
แต่บางครั้งเราต้องการ pattern ที่เฉพาะเจาะจงกว่านั้น เช่น:

- ปีต้องเป็นเลข 4 หลักเท่านั้น (ไม่ใช่ int ทั่วไปที่รับ `5` หรือ `99999`)
- เดือนต้องเป็นเลข 2 หลัก ระหว่าง 01-12
- รหัสสินค้าต้องเป็นรูปแบบ `SKU-XXXXX` เท่านั้น

การเขียน `re_path()` ทุกครั้งที่ต้องใช้ pattern เหล่านี้ทำให้โค้ดซ้ำซ้อนและอ่านยาก
Django จึงอนุญาตให้เรา **สร้าง path converter ของตัวเอง** แล้วนำกลับมาใช้ซ้ำได้เหมือน
converter มาตรฐาน

### 57.2 โครงสร้างของ Custom Converter

Custom converter คือ Python class ธรรมดาที่ต้องมี 3 ส่วน:

```python
class MyConverter:
    regex = '...'                    # (1) regex string สำหรับจับคู่ใน URL

    def to_python(self, value):      # (2) แปลงค่าจาก URL (str) เป็น Python object
        return ...

    def to_url(self, value):         # (3) แปลง Python object กลับเป็น str (ใช้ตอน reverse())
        return ...
```

| ส่วน | หน้าที่ |
|---|---|
| `regex` | บอก Django ว่าส่วนของ URL แบบไหนที่ match กับ converter นี้ |
| `to_python(value)` | รับ string ที่ match ได้ แปลงเป็นชนิดข้อมูลที่ view จะได้รับ ถ้า raise `ValueError` จะถือว่า **ไม่ match** และ Django จะลองหา pattern อื่นต่อ (หรือคืน 404 ถ้าไม่มี pattern ไหน match เลย) |
| `to_url(value)` | ทำงานย้อนกลับ ใช้ตอนเรียก `reverse()`/`{% url %}` เพื่อแปลงค่ากลับเป็น string ที่ใส่ใน URL ได้ |

### 57.3 ตัวอย่างที่ 1: `FourDigitYearConverter`

```python
# blog/converters.py
class FourDigitYearConverter:
    regex = '[0-9]{4}'

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return '%04d' % value
```

Converter นี้ต่างจาก `<int:year>` มาตรฐานตรงที่ **บังคับให้ต้องเป็นเลข 4 หลักพอดี**
ถ้าเป็น `<int:year>` ธรรมดา ค่าอย่าง `5` หรือ `202699` ก็จะ match ด้วย ซึ่งไม่ถูกต้อง
ตามความหมายของ "ปี"

### 57.4 ตัวอย่างที่ 2: `TwoDigitMonthConverter` พร้อม validation เพิ่มเติมใน `to_python`

```python
# blog/converters.py (ต่อจากด้านบน)
class TwoDigitMonthConverter:
    regex = '[0-1][0-9]'

    def to_python(self, value):
        month = int(value)
        if not (1 <= month <= 12):
            # raise ValueError เพื่อบอกว่า "ไม่ match" แม้ regex จะผ่านก็ตาม
            # Django จะถือว่า pattern นี้ไม่ตรง แล้วลอง pattern ถัดไป (หรือคืน 404)
            raise ValueError('เดือนต้องอยู่ระหว่าง 01-12 เท่านั้น')
        return month

    def to_url(self, value):
        return '%02d' % value
```

ตัวอย่างนี้แสดงให้เห็นพลังของ `to_python()`: แม้ regex `[0-1][0-9]` จะยอมให้ `19`
ผ่านมาได้ (เพราะ regex ตรวจแค่รูปแบบตัวเลข ไม่ตรวจค่าจริง) แต่ `to_python()` จะดัก
ค่าที่ไม่สมเหตุสมผล (เดือนที่ 19 ไม่มีจริง) แล้วปฏิเสธด้วย `ValueError`

### 57.5 ลงทะเบียน Converter ด้วย `register_converter()`

ก่อนใช้งานใน `urls.py` ต้อง**ลงทะเบียน** converter ก่อนเสมอ โดยทั่วไปทำที่ด้านบนของ
ไฟล์ `urls.py` ที่จะใช้งาน:

```python
# blog/urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')
register_converter(converters.TwoDigitMonthConverter, 'mm')

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='list'),
    path('<slug:slug>/', views.post_detail_by_slug, name='detail'),
    path('archive/<yyyy:year>/', views.post_archive_year, name='archive_year'),
    path('archive/<yyyy:year>/<mm:month>/', views.post_archive_month, name='archive_month'),
]
```

`register_converter(ConverterClass, 'prefix')` รับ 2 อาร์กิวเมนต์: class ของ
converter และ "คำนำหน้า" (prefix) ที่จะใช้เรียกมันใน `<prefix:name>`

### 57.6 View ที่รองรับ Custom Converter

```python
# blog/views.py
from django.http import HttpResponse


def post_archive_year(request, year):
    # year เป็น int แล้ว เพราะ to_python() แปลงให้แล้ว
    return HttpResponse(f"<h1>บทความทั้งหมดในปี {year}</h1>")


def post_archive_month(request, year, month):
    return HttpResponse(f"<h1>บทความในเดือน {month:02d}/{year}</h1>")
```

ทดลองเข้า URL:

```
http://127.0.0.1:8000/posts/archive/2026/           -> ผ่าน (year=2026)
http://127.0.0.1:8000/posts/archive/26/             -> 404 (ไม่ใช่ 4 หลัก)
http://127.0.0.1:8000/posts/archive/2026/09/        -> ผ่าน (year=2026, month=9)
http://127.0.0.1:8000/posts/archive/2026/19/        -> 404 (to_python raise ValueError)
```

### 57.7 ข้อดีของ Custom Converter เทียบกับ `re_path()`

| ประเด็น | Custom Converter | `re_path()` |
|---|---|---|
| Reusability | ลงทะเบียนครั้งเดียว ใช้ซ้ำได้หลาย `urls.py` | ต้องเขียน regex ซ้ำทุกที่ที่ใช้ |
| Type conversion | แปลงชนิดข้อมูลอัตโนมัติผ่าน `to_python()` | ได้ string เสมอ ต้องแปลงเองใน view |
| Validation เพิ่มเติม | ทำได้ใน `to_python()` (เช่น ตรวจช่วงค่า) | ทำได้เฉพาะผ่าน regex เท่านั้น |
| `reverse()` ทำงานถูกต้อง | ผ่าน `to_url()` ที่กำหนดเอง แปลงกลับได้แม่นยำ | ทำงานได้ปกติ แต่ต้องระวัง format string เอง |
| อ่านง่ายใน `urls.py` | อ่านง่ายเหมือน converter มาตรฐาน | อ่านยากขึ้นเมื่อ pattern ซับซ้อน |

Custom converter คือหนึ่งในฟีเจอร์ที่แยก Django Developer ระดับกลาง-สูงออกจากมือใหม่
เพราะแสดงถึงความเข้าใจในการออกแบบโค้ดที่ **reusable, testable, และ maintainable**

---

## ขั้นตอนที่ 58: การจัดการหน้า Error (404, 500, 403, 400) แบบกำหนดเอง (handler404 ฯลฯ)

### 58.1 ภาพรวม: DEBUG=True vs DEBUG=False

พฤติกรรมการแสดงหน้า error ของ Django ขึ้นกับค่า `DEBUG` ใน `settings.py`:

| สถานการณ์ | `DEBUG=True` (ตอนพัฒนา) | `DEBUG=False` (production) |
|---|---|---|
| เกิด 404 | หน้า "technical 404" แสดงรายการ URL pattern ทั้งหมดที่มี (มีประโยชน์ตอน debug) | เรียก `handler404` หรือ `404.html` |
| เกิด error 500 (exception ไม่ได้ดักไว้) | หน้า "technical 500" แสดง traceback เต็ม (**ห้ามให้ผู้ใช้จริงเห็นเด็ดขาด**) | เรียก `handler500` หรือ `500.html` |
| เกิด `PermissionDenied` | technical 403 | เรียก `handler403` หรือ `403.html` |
| เกิด `SuspiciousOperation`, `BadRequest` | technical 400 | เรียก `handler400` หรือ `400.html` |

**กฎเหล็กด้านความปลอดภัย**: ระบบ production ต้องตั้ง `DEBUG=False` เสมอ เพราะ
technical error page ของ Django เปิดเผยข้อมูลภายในระบบจำนวนมาก (โครงสร้างโค้ด,
environment variables บางส่วน, SQL query ฯลฯ) ซึ่งเป็นความเสี่ยงด้านความปลอดภัยร้ายแรง
(เราจะเรียนละเอียดใน Phase 10: Security)

### 58.2 `handler404`, `handler500`, `handler403`, `handler400` คืออะไร

Django อนุญาตให้กำหนด **view function ของตัวเอง** สำหรับแต่ละ error code เหล่านี้
โดยประกาศตัวแปรพิเศษไว้ใน**ไฟล์ที่ `ROOT_URLCONF` ชี้ไป** (คือ `config/urls.py`)
เท่านั้น (จะประกาศในไฟล์ `urls.py` ของแอปย่อยไม่มีผล):

```python
handler404 = 'blog.views.custom_page_not_found'
handler500 = 'blog.views.custom_server_error'
handler403 = 'blog.views.custom_permission_denied'
handler400 = 'blog.views.custom_bad_request'
```

### 58.3 Signature ของแต่ละ Handler (สำคัญมาก ต่างกันเล็กน้อย)

| Handler | Signature ของ view function | หมายเหตุ |
|---|---|---|
| `handler404` | `def view(request, exception)` | `exception` คือ instance ของ `Http404` |
| `handler500` | `def view(request)` | **ไม่มี** พารามิเตอร์ `exception` (ต่างจากตัวอื่น) |
| `handler403` | `def view(request, exception)` | `exception` คือ instance ของ `PermissionDenied` |
| `handler400` | `def view(request, exception)` | `exception` คือ instance ของ `SuspiciousOperation`/`BadRequest` |

### 58.4 เขียน Custom Error Views

```python
# blog/views.py (เพิ่มเติมจากเดิม)
from django.shortcuts import render


def custom_page_not_found(request, exception):
    return render(request, '404.html', status=404)


def custom_server_error(request):
    return render(request, '500.html', status=500)


def custom_permission_denied(request, exception):
    return render(request, '403.html', status=403)


def custom_bad_request(request, exception):
    return render(request, '400.html', status=400)
```

**ข้อควรระวังสำคัญ**: `custom_server_error` (handler500) **ต้องไม่พึ่งพาสิ่งที่อาจ
ล้มเหลวได้** เช่น query ฐานข้อมูล เพราะถ้า error 500 เกิดจากฐานข้อมูลล่มอยู่แล้ว
การพยายาม query อีกครั้งใน view นี้จะทำให้เกิด error ซ้อน error จนแสดงหน้า error
ไม่ได้เลย ควรให้ template `500.html` เป็น static HTML ล้วนที่สุด ไม่มี query ใด ๆ

### 58.5 กำหนดใน `config/urls.py`

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('posts/', include('blog.urls')),
]

handler404 = 'blog.views.custom_page_not_found'
handler500 = 'blog.views.custom_server_error'
handler403 = 'blog.views.custom_permission_denied'
handler400 = 'blog.views.custom_bad_request'
```

### 58.6 สร้าง Template สำหรับแต่ละหน้า Error

```html
<!-- templates/404.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ไม่พบหน้าที่คุณต้องการ</title>
</head>
<body>
    <h1>404 - ไม่พบหน้าที่คุณต้องการ</h1>
    <p>ขออภัย เราไม่พบหน้าที่คุณกำลังค้นหา ลิงก์อาจถูกลบหรือย้ายไปแล้ว</p>
    <a href="/">กลับสู่หน้าแรก</a>
</body>
</html>
```

```html
<!-- templates/500.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>เกิดข้อผิดพลาดของระบบ</title>
</head>
<body>
    <h1>500 - เกิดข้อผิดพลาดของระบบ</h1>
    <p>ขออภัย ระบบเกิดข้อผิดพลาดชั่วคราว ทีมงานได้รับแจ้งแล้วและกำลังแก้ไข</p>
</body>
</html>
```

```html
<!-- templates/403.html -->
<!DOCTYPE html>
<html lang="th">
<head><meta charset="UTF-8"><title>ไม่มีสิทธิ์เข้าถึง</title></head>
<body>
    <h1>403 - คุณไม่มีสิทธิ์เข้าถึงหน้านี้</h1>
    <p>กรุณาเข้าสู่ระบบด้วยบัญชีที่มีสิทธิ์ที่เหมาะสม</p>
</body>
</html>
```

```html
<!-- templates/400.html -->
<!DOCTYPE html>
<html lang="th">
<head><meta charset="UTF-8"><title>คำขอไม่ถูกต้อง</title></head>
<body>
    <h1>400 - คำขอไม่ถูกต้อง</h1>
    <p>เกิดข้อผิดพลาดกับคำขอของคุณ กรุณาลองใหม่อีกครั้ง</p>
</body>
</html>
```

ตรวจสอบว่า `settings.py` มี `TEMPLATES[0]['DIRS']` ชี้ไปที่โฟลเดอร์ `templates/`
ระดับโปรเจกต์แล้ว (ตั้งค่านี้ไว้ตั้งแต่ Part 004; รายละเอียดเต็มของระบบ template
จะเรียนใน Part 008)

### 58.7 วิธีทดสอบหน้า Error ในเครื่องของคุณ

เนื่องจากตอน `DEBUG=True` Django จะไม่เรียก handler ที่กำหนดเอง (จะโชว์ technical
error page แทนเสมอ) การทดสอบ custom error page ทำได้ 2 วิธี:

**วิธีที่ 1: ตั้ง `DEBUG=False` ชั่วคราวเพื่อทดสอบในเครื่อง**

```python
# settings.py (ทดสอบชั่วคราวเท่านั้น อย่าลืมเปลี่ยนกลับ)
DEBUG = False
ALLOWED_HOSTS = ['127.0.0.1', 'localhost']
```

เมื่อ `DEBUG=False` ต้องรัน `python manage.py collectstatic` ก่อน ไม่เช่นนั้น
ไฟล์ static (CSS/JS) จะไม่ถูก serve จาก `runserver` (เรื่อง static files จะเรียน
ละเอียดใน Part 009) จากนั้นลองเข้า URL ที่ไม่มีอยู่จริง เช่น
`http://127.0.0.1:8000/does-not-exist/` จะเห็นหน้า `404.html` ที่เราสร้างเอง

**วิธีที่ 2: เขียน Automated Test ด้วย Django Test Client (แนะนำกว่า)**

```python
# blog/tests.py
from django.test import TestCase, override_settings


class ErrorPageTests(TestCase):
    def test_custom_404_page(self):
        response = self.client.get('/does-not-exist/')
        self.assertEqual(response.status_code, 404)
        self.assertTemplateUsed(response, '404.html')

    @override_settings(DEBUG=False)
    def test_custom_404_page_with_debug_false(self):
        response = self.client.get('/does-not-exist/')
        self.assertEqual(response.status_code, 404)
```

วิธีนี้ดีกว่าเพราะเป็นส่วนหนึ่งของ automated test suite ที่จะรันซ้ำได้ทุกครั้งที่
โค้ดเปลี่ยนแปลง (รายละเอียดการเขียนเทสต์เชิงลึกจะเรียนทั้ง Phase 7)

### 58.8 ตารางสรุป

| Error Code | Exception ที่เกี่ยวข้อง | Handler | ใช้เมื่อ |
|---|---|---|---|
| 400 | `SuspiciousOperation`, `BadRequest` | `handler400` | Request ผิดรูปแบบ หรือมีพฤติกรรมน่าสงสัย |
| 403 | `django.core.exceptions.PermissionDenied` | `handler403` | ผู้ใช้ไม่มีสิทธิ์เข้าถึง (จะใช้บ่อยใน Phase 4) |
| 404 | `django.http.Http404` | `handler404` | ไม่พบ resource ที่ร้องขอ |
| 500 | Exception ใด ๆ ที่ไม่ถูกจัดการ | `handler500` | Bug/ข้อผิดพลาดที่ไม่คาดคิดในโค้ด |

---

## ขั้นตอนที่ 59: Best Practice การออกแบบ URL (trailing slash, RESTful naming, SEO-friendly slugs)

### 59.1 Trailing Slash: ทำไม Django URL ควรลงท้ายด้วย `/` เสมอ

Django มีธรรมเนียม (convention) ให้ URL ทุกตัวลงท้ายด้วย `/` เช่น `/posts/` แทนที่จะเป็น
`/posts` เหตุผลเบื้องหลังเกี่ยวข้องกับ setting `APPEND_SLASH`:

```python
# settings.py (ค่าเริ่มต้นคือ True อยู่แล้ว)
APPEND_SLASH = True
```

เมื่อ `APPEND_SLASH = True` (ค่าเริ่มต้น) และผู้ใช้เข้า URL ที่ไม่มี `/` ต่อท้าย เช่น
`/posts` แต่ใน `urlpatterns` มีแค่ `/posts/` Django's `CommonMiddleware` จะส่ง
**HTTP 301 redirect** ไปยัง `/posts/` ให้อัตโนมัติ (เฉพาะ request แบบ `GET`/`HEAD`
เท่านั้น เพื่อป้องกันการสูญเสียข้อมูลจาก `POST` request ที่ redirect ไม่ได้)

| แนวทาง | ข้อดี |
|---|---|
| URL ลงท้ายด้วย `/` เสมอ (แนวทาง Django มาตรฐาน) | สอดคล้องกับ `APPEND_SLASH`, ชัดเจนว่าเป็น "collection/resource" ไม่ใช่ไฟล์ |
| ไม่มี `/` ท้าย (แนวทางบาง framework อื่น) | สั้นกว่าเล็กน้อย แต่ขัดกับธรรมเนียม Django |

**คำแนะนำของหลักสูตรนี้**: ให้ URL pattern ทุกตัวใน `urlpatterns` ลงท้ายด้วย `/`
เสมอ ยกเว้น root path `''` ที่ไม่ต้องมี slash เพิ่ม (เพราะมันคือ path ว่างอยู่แล้ว)
และไฟล์ที่ระบุนามสกุลชัดเจน เช่น `robots.txt`, `sitemap.xml`

### 59.2 RESTful Naming: ออกแบบ URL ให้สื่อถึง "ทรัพยากร (Resource)" ไม่ใช่ "การกระทำ (Action)"

หลักการ RESTful ที่ดีคือ **ให้ URL แทน "คำนาม" (noun) ของทรัพยากร ส่วน "การกระทำ"
(verb) ให้สื่อผ่าน HTTP method แทน** (`GET`, `POST`, `PUT`, `DELETE`) หลักการนี้จะ
สำคัญมากยิ่งขึ้นเมื่อเราสร้าง REST API ด้วย Django REST Framework ใน Phase 5
แต่ควรฝึกให้เป็นนิสัยตั้งแต่การออกแบบเว็บแอปทั่วไป

```
❌ ไม่ดี (ใช้ verb ใน URL)          ✅ ดี (ใช้ noun + HTTP method)
GET  /getAllPosts/                 GET    /posts/
GET  /getPostById/42/              GET    /posts/42/
POST /createNewPost/               POST   /posts/
POST /updatePost/42/               POST   /posts/42/edit/   (form-based)
                                    PUT    /posts/42/        (API-based)
POST /deletePost/42/               POST   /posts/42/delete/ (form-based)
                                    DELETE /posts/42/        (API-based)
```

สำหรับเว็บแอปทั่วไป (ไม่ใช่ API) ที่ยังอิงกับฟอร์ม HTML ซึ่งรองรับแค่ `GET`/`POST`
เป็นเรื่องปกติที่จะเห็น URL อย่าง `/posts/42/edit/` และ `/posts/42/delete/` เพราะ
ฟอร์ม HTML ไม่รองรับ `PUT`/`DELETE` โดยตรง — นี่คือความแตกต่างที่สำคัญระหว่าง
"RESTful URL สำหรับเว็บแอปทั่วไป" กับ "RESTful API ที่เคร่งครัดตาม REST เต็มรูปแบบ"
(รายละเอียดเรื่อง REST API จะเจาะลึกใน Phase 5)

### 59.3 SEO-Friendly Slug: ทำไมต้องใช้ Slug แทน ID ในหลายกรณี

**Slug** คือสตริงที่เป็นมิตรกับ URL: ตัวพิมพ์เล็ก, ใช้ขีดกลาง `-` คั่นคำ, ไม่มีอักขระ
พิเศษหรือช่องว่าง

| URL แบบใช้ ID | URL แบบใช้ Slug |
|---|---|
| `/posts/42/` | `/posts/how-to-learn-django-in-2026/` |
| อ่านไม่รู้เรื่องว่าเนื้อหาคืออะไร | สื่อความหมาย เป็นมิตรกับผู้ใช้และเครื่องมือค้นหา (SEO) |
| เดา ID ถัดไปได้ง่าย (security concern เล็กน้อย) | เดาโพสต์อื่นได้ยากกว่า |

หลักการสร้าง slug ที่ดี:

- ใช้ตัวพิมพ์เล็กทั้งหมด (`lowercase`)
- คั่นคำด้วยขีดกลาง `-` ไม่ใช่ขีดล่าง `_` หรือช่องว่าง
- ตัดอักขระพิเศษออกทั้งหมด (`?`, `!`, `#`, ภาษาไทยที่ไม่ใช่ ASCII หากต้องการรองรับ
  URL แบบ ASCII ล้วน)
- ความยาวพอเหมาะ ไม่ควรยาวเกิน 5-8 คำ

Django มี utility function `slugify()` ช่วยแปลงข้อความเป็น slug ให้อัตโนมัติ
(รายละเอียดการใช้งานคู่กับ `SlugField` ใน Model จะเรียนเต็มรูปแบบใน Part 011):

```python
>>> from django.utils.text import slugify
>>> slugify("วิธีเรียน Django ให้เก่งใน ปี 2026!")
'django-2026'
>>> slugify("How to Learn Django in 2026!")
'how-to-learn-django-in-2026'
```

**ข้อควรระวัง**: `slugify()` จะตัดอักขระที่ไม่ใช่ ASCII ออก (รวมถึงภาษาไทย) โดย
ค่าเริ่มต้น ถ้าต้องการรองรับ slug ภาษาไทยแท้ ๆ ต้องใช้ `allow_unicode=True`
แต่โดยทั่วไปแนะนำให้ slug เป็น ASCII เสมอ เพื่อความเข้ากันได้สูงสุดกับระบบ, browser,
และ search engine ทุกตัว

### 59.4 คำแนะนำเพิ่มเติมสำหรับการออกแบบ URL ระดับมืออาชีพ

| หลักการ | อธิบาย |
|---|---|
| **URL ต้องคงที่ (Stable/Permanent)** | เมื่อเผยแพร่ URL ออกไปแล้ว (โดยเฉพาะที่ถูก index โดย Google) ห้ามเปลี่ยนแปลงพร่ำเพรื่อ ถ้าจำเป็นต้องเปลี่ยนจริง ๆ ให้ทำ HTTP redirect (301) จาก URL เก่าไปใหม่เสมอ |
| **หลีกเลี่ยงการซ้อน URL ลึกเกินไป** | `/posts/2026/09/26/tech/django/how-to-learn/` ลึกและอ่านยากเกินไป ควรจำกัดไม่เกิน 3-4 ระดับ |
| **ใช้ตัวพิมพ์เล็กเสมอ** | `/Posts/MyArticle/` กับ `/posts/my-article/` ถือเป็นคนละ URL ในสายตาเว็บเซิร์ฟเวอร์ ควรบังคับใช้ตัวพิมพ์เล็กเพื่อลดความสับสน |
| **ไม่ใส่นามสกุลไฟล์ปลอมใน URL แบบ dynamic** | หลีกเลี่ยง `/posts/42.html` สำหรับหน้าที่ render แบบ dynamic เพราะไม่ได้สื่อความจริงว่าเป็นไฟล์ static |
| **เตรียมแนวคิดเรื่อง versioning ไว้ล่วงหน้า** | สำหรับ API ในอนาคต (Phase 5) ควรออกแบบให้รองรับ `/api/v1/...` ตั้งแต่ต้น แม้ตอนนี้ยังไม่มี v2 ก็ตาม |
| **ใช้ hyphen (`-`) ไม่ใช่ underscore (`_`) ใน URL ที่มนุษย์อ่าน** | Google และเครื่องมือค้นหาส่วนใหญ่ตีความ `-` เป็นตัวแบ่งคำ แต่ตีความ `_` เป็นส่วนหนึ่งของคำเดียวกัน |

### 59.5 ตัวอย่าง Before/After: ปรับปรุงการออกแบบ URL ของแอป `blog`

```
❌ ก่อนปรับปรุง                          ✅ หลังปรับปรุง
/blog/getPost?id=42                     /posts/how-to-learn-django/
/blog/post_detail/42                    /posts/how-to-learn-django/
/blog/Category/Tech_News                /posts/category/tech-news/
/blog/deletePost.php?id=42              POST /posts/how-to-learn-django/delete/
/API/v1/GetAllPosts                     GET /api/v1/posts/
```

---

## ขั้นตอนที่ 60: สรุปและแบบฝึกหัด

### 60.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า URLconf คือ "สมุดเส้นทาง" ของ Django และกฎการจับคู่ URL แบบบนลงล่าง
- ✅ ใช้ `path()` เป็นค่าเริ่มต้น และรู้ว่าเมื่อไหร่ควรใช้ `re_path()` แทน
- ✅ ใช้ path converter มาตรฐานทั้ง 5 ตัวได้อย่างถูกต้อง: `str`, `int`, `slug`, `uuid`, `path`
- ✅ ตั้งชื่อ URL (`name=`) และใช้ `reverse()`/`redirect()`/`{% url %}` แทนการ hardcode
  URL string เสมอ
- ✅ แยก `urls.py` ระดับแอปออกจากโปรเจกต์ด้วย `include()` ทำให้โครงสร้างขยายง่าย
- ✅ ป้องกันชื่อ URL ชนกันข้ามแอปด้วย `app_name` และ URL namespacing
- ✅ ส่งค่าพิเศษเข้า view ผ่าน `kwargs` ใน `urls.py` และรู้ข้อจำกัดของมัน
- ✅ สร้าง custom path converter ของตัวเองพร้อม validation logic ใน `to_python()`
- ✅ กำหนด `handler404`, `handler500`, `handler403`, `handler400` และเขียนหน้า error
  ที่เป็นมิตรกับผู้ใช้
- ✅ เข้าใจหลักการออกแบบ URL ที่ดี: trailing slash, RESTful naming, SEO-friendly slug

### 60.2 Checklist ก่อนไป Part ถัดไป

- [ ] `blog/urls.py` ถูกสร้างแยกจาก `config/urls.py` และเชื่อมด้วย `include()`
- [ ] ทุก URL pattern ในแอป `blog` มีการตั้งชื่อ (`name=`) ครบทุกตัว
- [ ] `blog/urls.py` มี `app_name = 'blog'` กำหนดไว้
- [ ] ทดลองใช้ทั้ง `int`, `slug`, `uuid` converter ในโปรเจกต์จริงแล้วอย่างน้อยคนละ 1 URL
- [ ] สร้าง custom path converter อย่างน้อย 1 ตัว และลงทะเบียนใช้งานสำเร็จ
- [ ] สร้างและทดสอบ `handler404` พร้อม template `404.html` ได้จริง
- [ ] ไม่มี URL string ใด hardcode อยู่ใน view หรือ template อีกต่อไป (ใช้ `reverse`/`{% url %}` ทั้งหมด)

### 60.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1 (พื้นฐาน)**: สร้าง URL pattern ครบชุดสำหรับแอป `blog` ตามสเปกนี้
โดยใช้ `app_name = 'blog'` และตั้งชื่อ URL ให้ครบทุกตัว:

| Path | View (สร้าง stub ง่าย ๆ ที่คืน `HttpResponse` พอ) | ชื่อ URL ที่แนะนำ |
|---|---|---|
| `/posts/` | รายการบทความทั้งหมด | `blog:list` |
| `/posts/<slug>/` | รายละเอียดบทความตาม slug | `blog:detail` |
| `/posts/archive/<yyyy>/` | บทความทั้งหมดในปีนั้น (ใช้ custom converter จากขั้นตอนที่ 57) | `blog:archive_year` |
| `/posts/archive/<yyyy>/<mm>/` | บทความทั้งหมดในเดือนนั้น | `blog:archive_month` |

เขียนไฟล์ `blog/urls.py` และ `blog/converters.py` ให้ครบ แล้วทดสอบด้วย
`python manage.py runserver` ว่าทุก URL ทำงานถูกต้อง (ตัวอย่างคำตอบให้ดูจากโค้ด
ในขั้นตอนที่ 54-57 ของ Part นี้เป็นแนวทาง)

**แบบฝึกหัดที่ 2 (ประยุกต์)**: เพิ่ม URL สำหรับ "บทความตามหมวดหมู่" ที่ path
`/posts/category/<slug:category_slug>/` โดยใช้ extra options (`kwargs` ใน `path()`)
ส่งค่า `sort_by='latest'` เข้า view เป็นค่าเริ่มต้น แล้วสร้างอีก URL หนึ่งที่ path
`/posts/category/<slug:category_slug>/popular/` ที่ใช้ view เดียวกันแต่ส่ง
`sort_by='popular'` แทน

**แบบฝึกหัดที่ 3 (Error Handling)**: กำหนด `handler403` และเขียน view ที่ raise
`django.core.exceptions.PermissionDenied` แบบจงใจในหนึ่ง URL ทดสอบ (เช่น
`/posts/secret/`) แล้วยืนยันว่าเมื่อ `DEBUG=False` ระบบแสดงหน้า `403.html` ที่คุณ
ออกแบบเองได้ถูกต้อง โดยเขียนเป็น automated test ด้วย `django.test.TestCase`

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้าง custom path converter ชื่อ `SkuConverter` ที่จับคู่
เฉพาะรูปแบบ `SKU-` ตามด้วยเลข 5 หลัก (เช่น `SKU-00042`) โดยให้ `to_python()` คืนค่า
เป็น `int` ล้วน (ตัด `SKU-` ออก) และ `to_url()` แปลงกลับเป็นรูปแบบเดิมพร้อมเติมเลข 0
ข้างหน้าให้ครบ 5 หลักเสมอ ทดสอบทั้งการเข้า URL ตรง ๆ และการใช้ `reverse()`
เพื่อสร้าง URL กลับจากค่า `int`

### 60.4 คำถามที่พบบ่อย (FAQ)

**Q: จำเป็นต้องใช้ `include()` ตั้งแต่แอปแรกเลยหรือไม่ ถ้ามีแอปเดียวในโปรเจกต์?**
A: แนะนำให้ทำเป็นนิสัยตั้งแต่ต้น แม้จะมีแอปเดียว เพราะแทบทุกโปรเจกต์จริงจะมีแอป
เพิ่มขึ้นเรื่อย ๆ การวางโครงสร้างที่ถูกต้องตั้งแต่แรกช่วยประหยัดเวลา refactor ในอนาคต
มากกว่าการไปแก้ทีหลังตอนโปรเจกต์ใหญ่แล้ว

**Q: ควรใช้ `pk` (int) หรือ `slug` ในการอ้างอิงบทความดี?**
A: ขึ้นกับบริบท ถ้าเป็นหน้าที่ผู้ใช้ทั่วไปต้องเข้าถึงและแชร์ลิงก์ (เช่น หน้ารายละเอียด
บทความ) ควรใช้ `slug` เพราะเป็นมิตรกับ SEO และผู้ใช้ ส่วนถ้าเป็นหน้าสำหรับ admin/
internal tool ที่เน้นความเร็วและไม่สนใจ SEO การใช้ `pk` ก็เพียงพอและง่ายกว่า
บางระบบใช้ทั้งสองแบบผสมกัน เช่น `pk` สำหรับ admin panel และ `slug` สำหรับหน้า public

**Q: ทำไม `handler500` ถึงไม่มีพารามิเตอร์ `exception` เหมือนตัวอื่น?**
A: เพราะในทางเทคนิค error 500 มักเกิดจาก exception ที่ไม่คาดคิดและอาจเกิดขึ้นได้
จากหลายจุดในระบบ Django จึงออกแบบให้ handler500 เรียบง่ายที่สุดเท่าที่จะทำได้
เพื่อลดโอกาสที่ error handler เองจะพังซ้อนกับ error เดิม (ซึ่งจะทำให้ระบบไม่แสดงหน้า
error ใด ๆ เลย)

**Q: ถ้ามี URL pattern สองตัวที่ match กับ path เดียวกันได้ทั้งคู่ Django จะเลือกตัวไหน?**
A: Django เลือกตัวที่ประกาศ**ก่อน**ใน `urlpatterns` เสมอ (จากบนลงล่าง) นี่คือเหตุผลที่
เราย้ำเรื่องลำดับการเรียง pattern ตั้งแต่ขั้นตอนที่ 51 ว่าต้องเรียงจากเฉพาะเจาะจงไปหา
ทั่วไป

**Q: `re_path()` จะถูกลบออกจาก Django ในอนาคตเหมือน `url()` หรือไม่?**
A: ปัจจุบันยังไม่มีแผนจะลบ `re_path()` ออก เพราะยังมีกรณีใช้งานจริงที่ `path()`
converter มาตรฐานทำไม่ได้ (ตามที่อธิบายในขั้นตอนที่ 51) ต่างจาก `url()` ที่ถูกลบไป
เพราะเป็นแค่ชื่อ alias เก่าของ `re_path()` เท่านั้น

### เตรียมตัวสำหรับ Part ถัดไป

**Part 007: Views แบบ Function-Based เบื้องต้น** จะพาคุณเจาะลึกฝั่ง "View" อย่างเต็ม
รูปแบบ ตอนนี้เรารู้แล้วว่า URL ไหนจะถูกส่งไปหา view function ตัวไหน ใน Part ถัดไป
เราจะเรียนรู้ว่าภายใน view function นั้นควรเขียนอย่างไรให้ถูกต้องและเป็นมืออาชีพ:
`HttpRequest` object มีอะไรอยู่ข้างในบ้าง, ความแตกต่างระหว่าง `HttpResponse`,
`JsonResponse`, `render()`, `redirect()`, การอ่านค่า query string (`request.GET`)
และข้อมูลจากฟอร์ม (`request.POST`), การจัดการ HTTP method ต่าง ๆ ในฟังก์ชันเดียว,
และ decorator ที่สำคัญอย่าง `@require_http_methods`

โครงสร้าง URL ที่คุณวางไว้อย่างดีใน Part นี้จะเป็นฐานสำคัญที่ทำให้การเขียน view
ใน Part ถัดไปเป็นระเบียบและทดสอบง่ายตั้งแต่ต้น เตรียม `blog/urls.py` และ
`blog/views.py` ของคุณให้พร้อม แล้วไปต่อกันเลย!
