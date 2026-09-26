# Part 005: Django Apps และการจัดระเบียบโค้ด

> **ขั้นตอนที่ 41-50 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เข้าใจว่า "app" ใน Django คืออะไร ต่างจาก "project" อย่างไร
> สร้างแอปแรกของคุณชื่อ `blog` เจาะลึกทุกไฟล์ที่ Django สร้างให้อัตโนมัติ เรียนรู้วิธี
> ลงทะเบียนและปรับแต่งแอปอย่างถูกต้อง จนถึงหลักการออกแบบโครงสร้างโปรเจกต์ระดับมืออาชีพ
> ที่ทีมงานจริงใช้กัน เมื่อจบ Part นี้ คุณจะมีแอป `blog` พร้อม Post model ง่าย ๆ
> รอต่อยอดใน Part 011

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 41: ใช้คำสั่ง `python manage.py startapp` สร้างแอปแรกชื่อ `blog`
- ขั้นตอนที่ 42: เจาะลึกโครงสร้างไฟล์ในแอป (models.py, views.py, apps.py, admin.py, tests.py, migrations/) ทีละไฟล์
- ขั้นตอนที่ 43: ลงทะเบียนแอปใน INSTALLED_APPS อย่างถูกต้อง
- ขั้นตอนที่ 44: ปรับแต่ง AppConfig (verbose_name, default_auto_field, ready())
- ขั้นตอนที่ 45: แนวคิดว่าเมื่อไหร่ควรแยกเป็นหลาย app
- ขั้นตอนที่ 46: รูปแบบการจัดองค์กร app: by-feature vs by-layer
- ขั้นตอนที่ 47: แนวคิด Reusable App
- ขั้นตอนที่ 48: ทำความเข้าใจ `__init__.py` และ App Registry
- ขั้นตอนที่ 49: โครงสร้างโปรเจกต์ขนาดใหญ่แบบมืออาชีพ (โฟลเดอร์ `apps/`)
- ขั้นตอนที่ 50: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 41: ใช้คำสั่ง `python manage.py startapp` สร้างแอปแรกชื่อ `blog`

### 41.1 ทบทวนสถานะโปรเจกต์จาก Part 004

ใน Part 004 คุณได้สร้างโปรเจกต์ Django แรกด้วยคำสั่ง

```bash
cd django-mastery-course
source venv/bin/activate
django-admin startproject config .
```

ผลลัพธ์คือโครงสร้างไฟล์ประมาณนี้:

```
django-mastery-course/
├── venv/
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── requirements.txt
└── .gitignore
```

สังเกตว่าเราตั้งชื่อ **project package** ว่า `config` (ไม่ใช้ชื่อเดียวกับโปรเจกต์เพื่อลดความสับสน
เวลาต้อง `import` และเพราะ `config` สื่อความหมายชัดเจนว่าเก็บการตั้งค่าส่วนกลาง) ตอนนี้เรารัน
`python manage.py runserver` แล้วเห็นหน้า "The install worked successfully" ได้แล้ว แต่ยังไม่มี
โค้ดที่เป็นแอปพลิเคชันจริงของเราเลยสักบรรทัด — นั่นคือสิ่งที่ Part นี้จะเริ่มต้น

### 41.2 Project กับ App ต่างกันอย่างไร

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุดในช่วงต้นของการเรียน Django:

| | Project | App |
|---|---|---|
| ความหมาย | "เว็บไซต์ทั้งเว็บ" ที่ประกอบด้วยการตั้งค่าและแอปย่อยหลายตัว | หน่วยงานย่อยที่ทำหน้าที่เฉพาะอย่างหนึ่ง (เช่น บล็อก, ระบบสมาชิก) |
| จำนวนต่อระบบ | มีได้ **1 project** ต่อ 1 deployment | มีได้ **หลาย app** ใน 1 project |
| สร้างด้วยคำสั่ง | `django-admin startproject` | `python manage.py startapp` |
| มีไฟล์ตั้งค่า (settings.py) ไหม | ✅ มี | ❌ ไม่มี (ใช้ settings จาก project) |
| นำไปใช้ซ้ำในโปรเจกต์อื่นได้ไหม | โดยปกติไม่ได้ (เฉพาะเจาะจงกับ deployment นั้น) | ✅ ออกแบบให้ reuse ข้ามโปรเจกต์ได้ (เราจะเรียนในขั้นตอนที่ 47) |
| ตัวอย่าง | `config` (เว็บไซต์บริษัทของคุณทั้งหมด) | `blog`, `accounts`, `shop`, `comments` |

พูดง่าย ๆ คือ **project คือกล่องที่ครอบทุกอย่างไว้ ส่วน app คือชิ้นส่วนความสามารถแต่ละอย่าง**
ที่ประกอบกันเป็นเว็บไซต์นั้น เว็บไซต์หนึ่งเว็บอาจประกอบด้วย app หลายสิบตัว เช่น เว็บอีคอมเมิร์ซ
อาจมี app `products`, `cart`, `orders`, `payments`, `reviews`, `accounts` ฯลฯ ทำงานร่วมกันภายใต้
project เดียว

### 41.3 รันคำสั่ง `startapp` สร้างแอป `blog`

ตรวจสอบก่อนว่า venv ถูก activate อยู่ และคุณอยู่ที่ root ของโปรเจกต์ (โฟลเดอร์เดียวกับ `manage.py`)

```bash
# ตรวจสอบตำแหน่งปัจจุบัน (ต้องเห็น manage.py ในนี้)
ls
# manage.py  config/  venv/  requirements.txt  .gitignore

# สร้างแอปใหม่ชื่อ blog
python manage.py startapp blog
```

คำสั่งนี้ไม่มีผลลัพธ์แสดงบนหน้าจอถ้าสำเร็จ (Django เงียบเมื่อทำงานถูกต้อง) แต่จะมีโฟลเดอร์
`blog/` ใหม่เกิดขึ้นในโปรเจกต์ทันที

> **ข้อควรระวัง**: คำสั่ง `startapp` ต้องรันจาก **root ของโปรเจกต์** (ตำแหน่งเดียวกับ `manage.py`)
> เท่านั้น ถ้ารันผิดที่ Django จะสร้างโฟลเดอร์ผิดตำแหน่ง หรือ error ว่าไม่รู้จัก `manage.py`

### 41.4 โครงสร้างไฟล์ที่ได้

ตรวจสอบด้วยคำสั่ง:

```bash
ls blog/
```

จะเห็นไฟล์เหล่านี้:

```
blog/
├── __init__.py
├── admin.py
├── apps.py
├── migrations/
│   └── __init__.py
├── models.py
├── tests.py
└── views.py
```

โครงสร้างโปรเจกต์ทั้งหมดตอนนี้เป็นดังนี้:

```
django-mastery-course/
├── venv/
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── blog/                    ← แอปใหม่ที่เพิ่งสร้าง
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── requirements.txt
└── .gitignore
```

### 41.5 ทำไมชื่อแอปควรเป็นเอกพจน์หรือพหูพจน์ และข้อกำหนดการตั้งชื่อ

ก่อนตั้งชื่อแอปควรรู้กฎและธรรมเนียมต่อไปนี้:

- ชื่อแอปต้องเป็น **valid Python identifier** (ตัวอักษร, ตัวเลข, underscore เท่านั้น ห้ามขึ้นต้นด้วยตัวเลข
  ห้ามมีขีดกลาง `-`) เพราะมันจะถูก `import` เป็น Python module
- ชุมชน Django นิยมตั้งชื่อแอปเป็น **พหูพจน์และตัวพิมพ์เล็กทั้งหมด** เมื่อแอปนั้นแทน "กลุ่มของสิ่งของ"
  เช่น `posts`, `products`, `orders` แต่ก็มีข้อยกเว้นจำนวนมาก เช่น `blog` (เอกพจน์ที่สื่อถึง
  "ระบบบล็อก" ทั้งระบบ) หรือ `accounts` (พหูพจน์)
- **ห้ามตั้งชื่อแอปซ้ำกับชื่อ package มาตรฐานของ Python หรือของ Django** เช่น `auth`, `admin`,
  `test`, `models`, `django` เพราะจะเกิด import ชนกัน (name collision) ทำให้เกิด bug ที่ debug ยากมาก
- ในหลักสูตรนี้เราเลือกชื่อ `blog` เพราะสื่อความหมายชัดเจนตรงตัวว่าเป็นแอประบบบล็อก และจะใช้เป็น
  ตัวอย่างหลักตลอด Phase 2-3 ของหลักสูตร

---

## ขั้นตอนที่ 42: เจาะลึกโครงสร้างไฟล์ในแอป

ในขั้นตอนนี้เราจะเปิดไฟล์ทุกไฟล์ที่ Django สร้างให้ทีละไฟล์ และทำความเข้าใจว่าแต่ละไฟล์มีหน้าที่
อะไร แม้บางไฟล์เราจะยังไม่ได้เขียนโค้ดจริงจนกว่าจะถึง Part หลัง ๆ แต่การเข้าใจ "แผนที่" ของแอป
ตั้งแต่ต้นจะช่วยให้คุณไม่หลงทางเมื่อโปรเจกต์ใหญ่ขึ้น

### 42.1 `__init__.py`

```python
# blog/__init__.py
```

ไฟล์นี้ว่างเปล่า แต่มีความสำคัญมาก: มันบอก Python ว่าโฟลเดอร์ `blog/` เป็น **Python package**
ที่สามารถ `import` ได้ (เช่น `from blog.models import Post`) หากไม่มีไฟล์นี้ Python รุ่นเก่าจะไม่
รู้จักโฟลเดอร์นี้เป็น package เลย (Python 3.3+ รองรับ "namespace package" ที่ไม่ต้องมี `__init__.py`
ก็ได้ แต่ Django ยังคงสร้างไฟล์นี้ให้เสมอเพื่อความชัดเจนและเข้ากันได้กับเครื่องมือทุกตัว) เราจะพูดถึง
ไฟล์นี้ในรายละเอียดเชิงลึกอีกครั้งในขั้นตอนที่ 48

### 42.2 `models.py`

```python
# blog/models.py
from django.db import models

# Create your models here.
```

นี่คือไฟล์ที่คุณจะใช้เวลาเขียนมากที่สุดตลอดทั้งหลักสูตร ไฟล์นี้กำหนด **Model** ซึ่งคือ Python
class ที่แทนตารางในฐานข้อมูล ตาม MTV pattern ที่เราเรียนใน Part 001 — `M` ใน MTV ก็คือไฟล์นี้
Django import `models` module มาให้อัตโนมัติเพื่อให้คุณเริ่มเขียน class ที่สืบทอดจาก
`models.Model` ได้ทันที เราจะเรียนรายละเอียดเต็มรูปแบบใน Part 011 แต่ในขั้นตอนที่ 50 ของ Part นี้
คุณจะได้ลองเขียน model ง่าย ๆ ตัวแรกแล้ว

### 42.3 `views.py`

```python
# blog/views.py
from django.shortcuts import render

# Create your views here.
```

ไฟล์นี้เก็บ **View** ซึ่งคือฟังก์ชัน (หรือ class) ที่รับ HTTP request แล้วคืน HTTP response —
`V` ตัวสุดท้ายใน MTV Django import `render` มาให้ล่วงหน้าเพราะเป็นฟังก์ชันที่ใช้บ่อยที่สุดในการ
render template กลับไปเป็น HTML เราจะเจาะลึกเรื่อง View แบบเต็มใน Part 007

### 42.4 `apps.py`

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
```

ไฟล์นี้สำคัญกว่าที่มือใหม่ส่วนใหญ่คิด มันคือจุดที่ Django เก็บ **metadata (ข้อมูลกำกับ)** ของแอปนี้
ไว้ในรูปแบบ Python class ที่สืบทอดจาก `AppConfig` สังเกตว่า Django สร้างให้ 2 attribute
โดยอัตโนมัติ:

- `default_auto_field`: กำหนดชนิดของ primary key อัตโนมัติที่จะใช้กับทุก model ในแอปนี้ที่ไม่ได้
  ระบุ primary key เอง (ค่าเริ่มต้นตั้งแต่ Django 3.2 เป็นต้นมาคือ `BigAutoField` ซึ่งรองรับตัวเลข
  ได้มากกว่า `AutoField` แบบเดิม)
- `name`: ระบุ dotted path ของแอปนี้ในระบบ (ในกรณีนี้คือ `'blog'` เพราะอยู่ที่ root) — ค่านี้
  **ต้องตรงกับตำแหน่งจริงของแอปเสมอ** ถ้าคุณย้ายแอปไปอยู่ในโฟลเดอร์ย่อย ต้องแก้ค่านี้ด้วย
  (จะเห็นตัวอย่างจริงในขั้นตอนที่ 49)

เราจะปรับแต่งไฟล์นี้เพิ่มเติมในขั้นตอนที่ 44

### 42.5 `admin.py`

```python
# blog/admin.py
from django.contrib.admin import register

# Register your models here.
```

*(หมายเหตุ: บางเวอร์ชันของ Django สร้าง comment `# Register your models here.` โดยไม่มีการ
import ใด ๆ มาให้ล่วงหน้า — ขึ้นอยู่กับเวอร์ชัน แต่โดยทั่วไปไฟล์นี้จะว่างเกือบสนิท)*

ไฟล์นี้ใช้สำหรับลงทะเบียน model ของแอปเข้ากับ **Django Admin** ระบบจัดการข้อมูลสำเร็จรูปที่มา
พร้อม Django (จุดขายสำคัญของ "Batteries Included" ที่เราพูดถึงใน Part 001) ตัวอย่างการใช้งานจริง
(ที่เราจะทำใน Part 017-018):

```python
# blog/admin.py (ตัวอย่างที่จะเขียนใน Part ถัดไป เมื่อมี model แล้ว)
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

เพียงบรรทัดเดียวนี้ก็จะได้หน้า Admin panel ที่ใช้ CRUD (Create, Read, Update, Delete) ข้อมูล
Post ได้ทันทีโดยไม่ต้องเขียนหน้าเว็บเอง

### 42.6 `tests.py`

```python
# blog/tests.py
from django.test import TestCase

# Create your tests here.
```

ไฟล์นี้เก็บ **unit test** ของแอป Django มาพร้อมเฟรมเวิร์กทดสอบในตัว (สร้างต่อยอดจาก Python
`unittest` มาตรฐาน) ตัวอย่างเบื้องต้นของโครงสร้างที่จะเขียนใน Part 059-060:

```python
# blog/tests.py (ตัวอย่างโครงสร้างที่จะเขียนใน Part หลัง ๆ)
from django.test import TestCase
from .models import Post


class PostModelTests(TestCase):
    def test_post_str_returns_title(self):
        post = Post.objects.create(title="สวัสดี Django", content="เนื้อหาทดสอบ")
        self.assertEqual(str(post), "สวัสดี Django")
```

การเขียน test ตั้งแต่เนิ่น ๆ เป็นนิสัยที่แยกนักพัฒนามืออาชีพออกจากมือสมัครเล่น เราจะเน้นเรื่องนี้
หนักมากใน Phase 7 ของหลักสูตร (Part 59-65)

> **หมายเหตุสำหรับโปรเจกต์ที่มีหลายไฟล์ทดสอบ**: เมื่อแอปมี test เยอะขึ้น สามารถเปลี่ยน
> `tests.py` (ไฟล์เดียว) ให้เป็นแพ็กเกจ `tests/` (โฟลเดอร์) ที่มี `__init__.py`,
> `test_models.py`, `test_views.py` แยกกันได้ — Django รองรับทั้งสองรูปแบบ

### 42.7 `migrations/`

```
blog/migrations/
└── __init__.py
```

โฟลเดอร์นี้เก็บ **migration files** ซึ่งเป็นไฟล์ Python ที่ Django สร้างอัตโนมัติเพื่อบันทึกทุกครั้งที่
โครงสร้างฐานข้อมูล (schema) มีการเปลี่ยนแปลง เช่น เพิ่ม model ใหม่ เพิ่ม field ใหม่ แก้ไข field เดิม
ตอนนี้ในโฟลเดอร์มีแค่ `__init__.py` เพราะเรายังไม่ได้สร้าง model ใด ๆ เมื่อคุณรัน
`python manage.py makemigrations` ครั้งแรกหลังสร้าง model จะมีไฟล์แบบ `0001_initial.py`
ปรากฏขึ้นในนี้ (จะสาธิตในขั้นตอนที่ 50 และเรียนละเอียดเต็มใน Part 011 และ 016)

> **กฎเหล็ก**: ไฟล์ migration **ต้อง commit เข้า Git เสมอ** ห้าม `.gitignore` โฟลเดอร์นี้
> เพราะมันคือ "ประวัติศาสตร์" ของโครงสร้างฐานข้อมูลที่ทีมและ production server ต้องใช้ตาม

### 42.8 ตารางสรุปหน้าที่ของทุกไฟล์

| ไฟล์/โฟลเดอร์ | หน้าที่ | เขียนโค้ดจริงใน Part |
|---|---|---|
| `__init__.py` | บอก Python ว่านี่คือ package | (โครงสร้าง — ขั้นตอนที่ 48) |
| `models.py` | นิยามโครงสร้างข้อมูล (ตาราง DB) | Part 011-016 |
| `views.py` | ตรรกะรับ request → ส่ง response | Part 007, 021-024 |
| `apps.py` | Metadata และการตั้งค่าของแอป | ขั้นตอนที่ 44 |
| `admin.py` | ลงทะเบียน model เข้า Admin panel | Part 017-018 |
| `tests.py` | Unit test ของแอป | Part 059-065 |
| `migrations/` | ประวัติการเปลี่ยนแปลง schema DB | Part 011, 016 |
| `urls.py` *(ไม่ได้สร้างอัตโนมัติ)* | URL routing เฉพาะแอป | Part 006 |
| `templates/<app>/` *(ไม่ได้สร้างอัตโนมัติ)* | ไฟล์ HTML ของแอป | Part 008 |
| `static/<app>/` *(ไม่ได้สร้างอัตโนมัติ)* | CSS/JS/รูปภาพของแอป | Part 009 |

สังเกตว่า Django **ไม่ได้สร้าง** `urls.py`, โฟลเดอร์ `templates/`, หรือ `static/` ให้อัตโนมัติ
เพราะไม่ใช่ทุกแอปที่ต้องมีสิ่งเหล่านี้ (เช่น แอปที่ทำหน้าที่เป็น background service ล้วน ๆ อาจไม่มี
view หรือ template เลย) คุณต้องสร้างเองเมื่อต้องใช้งาน — เราจะสร้างในขั้นตอนที่ 50 และ Part ถัดไป

---

## ขั้นตอนที่ 43: ลงทะเบียนแอปใน INSTALLED_APPS อย่างถูกต้อง

### 43.1 `INSTALLED_APPS` คืออะไร และทำไมต้องลงทะเบียน

การรัน `startapp` เพียงอย่างเดียว **ยังไม่ทำให้ Django รู้จักแอปนั้น** ลองเปิดไฟล์
`config/settings.py` ดู จะเห็นตัวแปร `INSTALLED_APPS`:

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

นี่คือรายการแอปทั้งหมดที่ Django "รู้จักและโหลดใช้งาน" ในโปรเจกต์นี้ สังเกตว่า Django เองก็เป็น
"แอป" เหมือนกัน! `django.contrib.admin` คือแอป Admin panel, `django.contrib.auth` คือแอประบบ
authentication ฯลฯ — นี่คือการพิสูจน์ว่า Django "กินอาหารของตัวเอง" (dogfooding): ฟีเจอร์หลัก
ของ framework ก็ถูกสร้างด้วยระบบ app แบบเดียวกับที่เราใช้

ถ้าแอปไม่อยู่ใน `INSTALLED_APPS` จะเกิดผลเสียดังนี้:

- Django จะ**ไม่สร้างตารางฐานข้อมูล**ให้ model ในแอปนั้น แม้จะรัน `makemigrations`/`migrate` ก็ตาม
- Template loader จะ**ไม่ค้นหา** template ในโฟลเดอร์ `templates/` ของแอปนั้น
- Django Admin จะ**ไม่รู้จัก** model ของแอปนั้น แม้จะเขียน `admin.py` ไว้แล้วก็ตาม
- คำสั่ง management command ที่กำหนดในแอปนั้น (เช่น custom command) จะใช้ไม่ได้
- App registry (ขั้นตอนที่ 48) จะไม่รวมแอปนี้ไว้เลย

ให้เพิ่ม `'blog'` เข้าไปในรายการ:

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Local apps
    'blog',
]
```

การแยกกลุ่ม "Django built-in apps" กับ "Local apps" (และในอนาคตอาจมี "Third-party apps" ด้วย
comment คั่นกลาง) เป็นธรรมเนียมที่ทีมมืออาชีพนิยมทำ เพื่อให้อ่าน settings.py ได้ง่ายเมื่อโปรเจกต์
มีแอปเยอะขึ้น

### 43.2 ความแตกต่างระหว่าง `'blog'` กับ `'blog.apps.BlogConfig'`

นี่คือคำถามที่พบบ่อยมาก: เขียน `'blog'` แบบสั้น กับเขียน `'blog.apps.BlogConfig'` แบบเต็ม
ต่างกันอย่างไร คำตอบคือ **ทั้งสองแบบทำงานเหมือนกันทุกประการในกรณีส่วนใหญ่** เพราะเมื่อ Django
เจอ string แบบสั้น (`'blog'`) มันจะ:

1. มองหาไฟล์ `blog/apps.py`
2. ถ้าเจอ `AppConfig` subclass เพียงตัวเดียวในไฟล์นั้น (เช่น `BlogConfig`) จะใช้ตัวนั้นโดยอัตโนมัติ
3. ถ้าไม่เจอไฟล์ `apps.py` เลย จะสร้าง `AppConfig` แบบ default (generic) ให้ใช้ชั่วคราว

ดังนั้น `'blog'` และ `'blog.apps.BlogConfig'` จะให้ผลลัพธ์เหมือนกันทุกประการ **ตราบใดที่**
ในไฟล์ `apps.py` มี `AppConfig` subclass แค่ตัวเดียว

| รูปแบบการเขียน | เมื่อไหร่ที่ต้องใช้ |
|---|---|
| `'blog'` (สั้น) | กรณีทั่วไป มี `AppConfig` เดียวในไฟล์ `apps.py` — **แนะนำสำหรับกรณีส่วนใหญ่** |
| `'blog.apps.BlogConfig'` (เต็ม) | เมื่อมี `AppConfig` **มากกว่า 1 class** ในไฟล์เดียวกัน (เช่น ต้องการ config คนละแบบสำหรับ dev/prod) หรือต้องการความชัดเจนแบบ explicit ในโปรเจกต์ขนาดใหญ่ที่ทีมใหญ่ทำงานร่วมกัน |
| `'blog.apps.BlogConfig'` (เต็ม, บังคับ) | เมื่อแอปถูกย้ายเข้าไปในแพ็กเกจย่อย เช่น `apps.blog` — บาง IDE และเครื่องมือ static analysis ต้องการ path เต็มเพื่อ resolve ได้ถูกต้อง (จะสาธิตในขั้นตอนที่ 49) |

ตัวอย่างกรณีที่มี `AppConfig` 2 ตัว (พบได้ในบาง reusable app บน PyPI ที่รองรับหลาย configuration):

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
    verbose_name = 'ระบบบล็อก'


class BlogConfigWithAnalytics(BlogConfig):
    """ใช้เมื่อต้องการเปิด analytics tracking เพิ่มเติม"""
    def ready(self):
        super().ready()
        import blog.analytics_signals  # noqa
```

ในกรณีนี้ **จำเป็นต้อง** ระบุ path เต็มใน `INSTALLED_APPS` เพราะ Django ไม่รู้ว่าจะเลือก class ไหน
ถ้ามีมากกว่า 1 ตัวโดยไม่ระบุ:

```python
INSTALLED_APPS = [
    # ...
    'blog.apps.BlogConfigWithAnalytics',  # ต้องระบุเต็ม
]
```

### 43.3 ทดลอง: ถ้าลืมลงทะเบียนแอปจะเกิดอะไรขึ้น

ลองคอมเมนต์บรรทัด `'blog'` ออกชั่วคราว แล้วรัน:

```bash
python manage.py makemigrations blog
```

คุณจะได้ error ประมาณนี้:

```
django.core.exceptions.ImproperlyConfigured: App 'blog' could not be found.
Is it in INSTALLED_APPS?
```

นี่คือ error ที่มือใหม่เจอบ่อยที่สุดอันดับต้น ๆ เมื่อสร้างแอปใหม่แล้วลืมลงทะเบียน อย่าลืมเปิด
comment กลับคืนหลังทดลองเสร็จ

### 43.4 ลำดับใน `INSTALLED_APPS` สำคัญหรือไม่

โดยทั่วไป **ลำดับไม่สำคัญ** สำหรับแอปส่วนใหญ่ แต่มีข้อยกเว้นที่ต้องระวัง:

- **Template override**: ถ้าสองแอปมี template ชื่อเดียวกัน (เช่น `registration/login.html`)
  Django จะใช้ template จากแอปที่อยู่ **ลำดับแรกสุด** ที่เจอก่อน (ตาม `APP_DIRS` loader)
- **Static files override**: เช่นเดียวกับ template — แอปที่มาก่อนใน `INSTALLED_APPS` จะถูกใช้
  ก่อนเมื่อไฟล์ static ชื่อซ้ำกัน
- **Signal registration order**: ถ้าแอป B ต้องพึ่งพา signal ที่ลงทะเบียนโดยแอป A ใน `ready()`
  บางครั้งลำดับอาจมีผลต่อพฤติกรรม (แม้จะพบไม่บ่อยในทางปฏิบัติ)
- **`django.contrib.admin` ต้องอยู่ก่อน static-related apps บางตัวในบางเวอร์ชัน** เพื่อให้
  admin's static files (CSS/JS ของหน้า admin) โหลดถูกต้อง

ธรรมเนียมที่แนะนำคือเรียงจาก: **Django built-in apps → Third-party apps → Local apps ของเรา**
ตามลำดับนี้เสมอ เพื่อให้แอปของเราสามารถ override พฤติกรรม default ได้หากจำเป็น

---

## ขั้นตอนที่ 44: ปรับแต่ง AppConfig

### 44.1 `AppConfig` class คืออะไร

`AppConfig` คือคลาสฐาน (base class) ที่ Django ใช้เก็บ configuration ของแต่ละแอป ทุกแอปมี
`AppConfig` ของตัวเองไม่ว่าจะระบุเองหรือไม่ก็ตาม (ถ้าไม่ระบุ Django จะสร้าง default ให้)
`AppConfig` มี attribute และ method สำคัญที่คุณสามารถ override ได้ดังนี้:

| Attribute/Method | ความหมาย | ค่า default |
|---|---|---|
| `name` | dotted path ของแอป (บังคับต้องมี) | ไม่มี ต้องระบุเสมอ |
| `label` | ชื่อสั้นที่ใช้อ้างอิงแอปภายในระบบ (เช่นใน migration) | ส่วนสุดท้ายของ `name` |
| `verbose_name` | ชื่อที่แสดงในหน้า Django Admin | ชื่อแอปแบบ title case |
| `default_auto_field` | ชนิด primary key อัตโนมัติ | ตาม `DEFAULT_AUTO_FIELD` ใน settings หรือ `AutoField` |
| `path` | path เต็มของโฟลเดอร์แอปบนดิสก์ | คำนวณอัตโนมัติ |
| `ready()` | method ที่ Django เรียกตอนแอปพร้อมใช้งานเต็มรูปแบบ | ไม่ทำอะไร (pass) |

### 44.2 ปรับแต่ง `verbose_name`

ค่าเริ่มต้นของ `verbose_name` คือชื่อแอปที่แปลงเป็น title case แบบง่าย ๆ (เช่น `blog` →
`Blog`) ซึ่งอาจไม่เหมาะกับเว็บไซต์ภาษาไทย เราสามารถกำหนดชื่อภาษาไทยที่จะแสดงในหน้า
Django Admin ได้:

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
    verbose_name = 'ระบบจัดการบล็อก'
```

เมื่อคุณเปิดหน้า `/admin/` ในอนาคต (Part 017) หัวข้อกลุ่มของแอปนี้จะแสดงเป็น "ระบบจัดการบล็อก"
แทนที่จะเป็น "Blog" แบบดิบ ๆ ทำให้ประสบการณ์ของทีมที่ดูแลข้อมูล (content editor) เป็นมิตรกับ
ผู้ใช้ภาษาไทยมากขึ้น

### 44.3 `default_auto_field`: ทำไม Django สร้างบรรทัดนี้ให้อัตโนมัติ

ตั้งแต่ Django 3.2 เป็นต้นมา ทุกแอปใหม่ที่สร้างด้วย `startapp` จะมี `default_auto_field` กำกับไว้
เสมอ เหตุผลคือ Django ต้องการแก้ปัญหาการ warning ที่เคยเกิดขึ้นบ่อยในอดีต: ถ้าไม่ระบุชนิดของ
primary key อัตโนมัติ Django รุ่นเก่าจะใช้ `AutoField` (จำนวนเต็ม 32-bit สูงสุดประมาณ 2.1 พันล้าน)
เป็นค่าเริ่มต้น ซึ่งอาจไม่พอสำหรับตารางขนาดใหญ่มากในระบบที่มีข้อมูลมหาศาล (เช่น ระบบระดับ
Instagram) จึงเปลี่ยนค่าเริ่มต้นแนะนำเป็น `BigAutoField` (จำนวนเต็ม 64-bit)

| ชนิด | ขนาด | ค่าสูงสุดโดยประมาณ | เหมาะกับ |
|---|---|---|---|
| `AutoField` | 32-bit | ~2.1 พันล้าน | ตารางเล็ก-กลาง ที่แน่ใจว่าไม่เกินขีดจำกัด |
| `BigAutoField` | 64-bit | ~9.2 ล้านล้านล้าน | ตารางที่อาจมีข้อมูลเติบโตมหาศาล (แนะนำเป็นค่าเริ่มต้นในโปรเจกต์ใหม่) |
| `SmallAutoField` | 16-bit | ~32,000 | ตาราง lookup ขนาดเล็กมากที่รู้แน่นอนว่าจำนวนแถวจำกัด |

คุณสามารถกำหนดค่านี้แบบ**รวมศูนย์ทั้งโปรเจกต์**ได้ผ่าน `config/settings.py` แทนที่จะกำหนดทีละ
แอป:

```python
# config/settings.py
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

ถ้ากำหนดไว้ใน `settings.py` แล้ว ไม่จำเป็นต้องเขียนซ้ำใน `apps.py` ของทุกแอป — Django จะใช้ค่า
จาก settings เป็นค่า fallback แต่การที่ `startapp` เขียนให้ในแต่ละแอปโดยตรงก็ไม่ใช่เรื่องผิด
เพียงแค่ซ้ำซ้อนกับ settings เท่านั้น (ในหลักสูตรนี้เราจะปล่อยให้ Django เขียนแบบ per-app ตาม
ค่าเริ่มต้นเพื่อความชัดเจนของแต่ละแอป)

### 44.4 `ready()` method: จุดที่ใช้โหลด signals

`ready()` คือ method ที่ Django เรียก**ครั้งเดียว**หลังจากแอป registry โหลดแอปทั้งหมดเสร็จสมบูรณ์
แล้ว (รายละเอียดลำดับการโหลดจะอธิบายในขั้นตอนที่ 48) นี่คือจุดที่เหมาะสมที่สุดในการ:

- Import และลงทะเบียน **Django signals** (เช่น `post_save`, `pre_delete`)
- ทำ initialization บางอย่างที่ต้องการให้แอปทุกตัวพร้อมใช้งานแล้วก่อน

ตัวอย่างการใช้งานจริง สมมติเราต้องการให้ระบบส่งการแจ้งเตือนทุกครั้งที่มีการสร้างโพสต์ใหม่
(เราจะนำแนวคิดนี้ไปใช้จริงใน Part 019 เรื่อง Signals):

```python
# blog/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Post


@receiver(post_save, sender=Post)
def notify_new_post(sender, instance, created, **kwargs):
    if created:
        print(f'มีโพสต์ใหม่ถูกสร้าง: "{instance.title}"')
```

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
    verbose_name = 'ระบบจัดการบล็อก'

    def ready(self):
        import blog.signals  # noqa: F401
```

> **ข้อควรระวังสำคัญ**: ห้าม import `models.py` แบบ top-level ที่ด้านบนของไฟล์ `apps.py`
> โดยตรง (เช่น `from .models import Post` เขียนไว้บนสุดของไฟล์) เพราะตอนที่ Django กำลังโหลด
> `apps.py` นั้น **app registry ยังโหลดไม่เสร็จ** การ import model ตรง ๆ ตอนนั้นอาจทำให้เกิด
> `AppRegistryNotReady` exception ได้ วิธีที่ปลอดภัยคือ import module ที่ต้องการ (เช่น
> `blog.signals`) **ไว้ข้างใน method `ready()`** เท่านั้น ซึ่ง `blog/signals.py` ค่อยไป import
> `models.py` เองอีกที (ตอนนั้น registry พร้อมแล้ว)

การใส่ `# noqa: F401` ต่อท้ายบรรทัด import คือธรรมเนียมบอก linter (เช่น Ruff, Flake8) ว่า
"การ import นี้ดูเหมือนไม่ได้ใช้งาน (unused import) แต่จงใจทำแบบนี้" เพราะจุดประสงค์ของบรรทัดนี้
คือให้ Python รัน module `signals.py` (ซึ่งจะลงทะเบียน signal receiver) ไม่ใช่เพื่อนำ object
ออกมาใช้งานต่อ

### 44.5 ตัวอย่าง `apps.py` แบบสมบูรณ์

รวมทุกอย่างที่เรียนมาในขั้นตอนนี้เข้าด้วยกัน:

```python
# blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    """
    Configuration สำหรับแอป blog

    - ใช้ BigAutoField เป็น primary key อัตโนมัติ เพื่อรองรับข้อมูลจำนวนมากในอนาคต
    - ตั้ง verbose_name เป็นภาษาไทยเพื่อความเป็นมิตรในหน้า Django Admin
    - โหลด signals ใน ready() เพื่อลงทะเบียนการแจ้งเตือนเมื่อมีโพสต์ใหม่
    """
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
    verbose_name = 'ระบบจัดการบล็อก'

    def ready(self):
        import blog.signals  # noqa: F401
```

---

## ขั้นตอนที่ 45: แนวคิดว่าเมื่อไหร่ควรแยกเป็นหลาย app

### 45.1 Single Responsibility Principle ในระดับ app

หลักการ **Single Responsibility Principle (SRP)** ที่คุณอาจเคยได้ยินในบริบทของ class หรือ
function ก็ใช้ได้กับการออกแบบ app เช่นกัน หลักการคือ: **แต่ละ app ควรมีเหตุผลเดียวในการ
เปลี่ยนแปลง (one reason to change)**

พูดให้เป็นรูปธรรม: ถ้าคุณกำลังเขียนเว็บอีคอมเมิร์ซ และยัด logic ของ "สินค้า", "ตะกร้าสินค้า",
"การชำระเงิน", "รีวิวสินค้า" ไว้ในแอปเดียวชื่อ `shop` — เมื่อทีมงานคนหนึ่งแก้ไข logic การชำระเงิน
อาจไปกระทบโค้ดที่เกี่ยวกับสินค้าโดยไม่ตั้งใจ (เพราะทุกอย่างอยู่ไฟล์เดียวกันหรือใกล้กันมาก) และ
เมื่อเขียน test ก็ยากที่จะแยกทดสอบเฉพาะส่วน

### 45.2 สัญญาณที่บ่งบอกว่าถึงเวลาแยก app

| สัญญาณ | คำอธิบาย |
|---|---|
| `models.py` ยาวเกิน 300-500 บรรทัด | มักหมายความว่ามีหลายโดเมนความรับผิดชอบปนกันอยู่ |
| ชื่อ model ไม่เกี่ยวข้องกันเชิงความหมาย | เช่น `Product`, `BlogPost`, `Invoice` อยู่ใน app เดียวกัน |
| ทีมงานคนละกลุ่มแก้ไขไฟล์เดียวกันบ่อย | เกิด merge conflict ถี่ผิดปกติใน Git |
| ต้องการ reuse บางส่วนในโปรเจกต์อื่น | เช่น ระบบ comment ที่อยากใช้ได้ทั้งกับ blog และ product review |
| การทดสอบ (test) ของฟีเจอร์หนึ่งพึ่งพา setup ของอีกฟีเจอร์ที่ไม่เกี่ยวข้อง | สัญญาณของ coupling ที่มากเกินไป |
| `admin.py` มี `ModelAdmin` มากกว่า 5-6 ตัวที่ไม่เกี่ยวข้องกัน | หน้า Admin panel เริ่มสับสน จัดกลุ่มยาก |

### 45.3 ตัวอย่างเปรียบเทียบ: Monolithic App vs แยก App

**แบบไม่แนะนำ (ยัดทุกอย่างไว้แอปเดียว)**:

```
myproject/
└── main/
    ├── models.py       # มี Product, Order, BlogPost, Comment, UserProfile ปนกันหมด
    ├── views.py        # มี view ของทุกฟีเจอร์ปนกัน ยาวหลายพันบรรทัด
    ├── admin.py
    └── urls.py
```

**แบบแนะนำ (แยกตามความรับผิดชอบ)**:

```
myproject/
├── products/
│   ├── models.py       # Product, Category
│   ├── views.py
│   └── admin.py
├── orders/
│   ├── models.py       # Order, OrderItem
│   ├── views.py
│   └── admin.py
├── blog/
│   ├── models.py       # Post, Tag
│   ├── views.py
│   └── admin.py
└── accounts/
    ├── models.py       # UserProfile
    ├── views.py
    └── admin.py
```

การแยกแบบนี้ทำให้:

- แต่ละทีมย่อยดูแลแอปของตัวเองได้อย่างอิสระ (ownership ชัดเจน)
- การเขียน test ทำได้ตรงจุด ไม่ปนกัน
- ถ้าต้องการปิดฟีเจอร์ใดฟีเจอร์หนึ่งชั่วคราว สามารถเอาแอปนั้นออกจาก `INSTALLED_APPS` ได้ทันที
  โดยไม่กระทบส่วนอื่น
- ง่ายต่อการย้ายไปเป็น microservice ในอนาคต หากจำเป็น (Phase 12 ของหลักสูตร)

### 45.4 กฎง่าย ๆ ในการตัดสินใจ (ไม่ใช่กฎตายตัว)

Django ไม่มีกฎบังคับว่า app หนึ่งต้องมีกี่ model หรือกี่บรรทัด การตัดสินใจแยกหรือไม่แยก
ขึ้นอยู่กับดุลยพินิจ แต่มีแนวทางที่ใช้ได้จริงดังนี้:

1. **ถามว่า "โดเมนความรับผิดชอบ (domain) นี้คืออะไร"** — ถ้าตอบได้เป็นคำเดียวชัดเจน (บล็อก,
   สินค้า, การชำระเงิน) นั่นคือผู้สมัคร (candidate) ที่ดีสำหรับการเป็น 1 app
2. **อย่าแยกแอปเร็วเกินไปในโปรเจกต์เล็ก** (over-engineering) — โปรเจกต์ที่มี model แค่ 3-4 ตัว
   ที่เกี่ยวข้องกันแน่นเฟ้น (tightly coupled) ยังไม่จำเป็นต้องแยก
3. **แต่ก็อย่าแยกช้าเกินไปในโปรเจกต์ใหญ่** — ถ้า `models.py` เริ่มยาวเกิน 300-500 บรรทัดและ
   เริ่มเห็นโดเมนที่ต่างกันชัดเจน ควรแยกก่อนที่จะสายเกินไป (การ refactor แยก app ทีหลังทำได้
   แต่ยุ่งยากกว่าออกแบบไว้แต่แรกมาก โดยเฉพาะเรื่อง migration history)
4. **ยึดหลัก "high cohesion, low coupling"**: ภายในแอปเดียวกัน โค้ดควรเกี่ยวข้องกันแน่นเฟ้น
   (cohesion สูง) ส่วนระหว่างแอป ควรพึ่งพากันให้น้อยที่สุด (coupling ต่ำ)

ในหลักสูตรนี้ เราจะเริ่มต้นด้วยแอปเดียว (`blog`) และค่อย ๆ เพิ่มแอปใหม่ตามความจำเป็นเมื่อเนื้อหา
ในแต่ละ Phase ต้องการ (เช่น `accounts` ใน Phase 4, ระบบ API ใน Phase 5) เพื่อให้เห็นวิวัฒนาการ
ของโครงสร้างโปรเจกต์อย่างเป็นธรรมชาติ

---

## ขั้นตอนที่ 46: รูปแบบการจัดองค์กร App: By-Feature vs By-Layer

เมื่อพูดถึง "การจัดองค์กรโค้ดภายในโปรเจกต์" มีสองปรัชญาหลักที่ใช้กันในวงการ Django (และ
web framework อื่น ๆ ทั่วไป)

### 46.1 By-Layer Organization (จัดตาม "ชั้น" ของสถาปัตยกรรม)

แนวทางนี้จัดกลุ่มไฟล์ตาม**ประเภทหน้าที่ทางเทคนิค** ไม่ว่าจะเป็นฟีเจอร์ไหนก็เก็บรวมกันตามชนิด
ไฟล์ — นี่คือแนวทางที่ Django `startapp` ให้มาโดยธรรมชาติในระดับหนึ่ง (ทุกแอปมี `models.py`,
`views.py` ของตัวเอง) แต่คำว่า "by-layer" แบบสุดโต่งหมายถึงการจัดทั้งโปรเจกต์แบบนี้:

```
myproject/
├── models/
│   ├── product.py
│   ├── order.py
│   └── blog_post.py
├── views/
│   ├── product_views.py
│   ├── order_views.py
│   └── blog_views.py
├── admin/
│   ├── product_admin.py
│   └── order_admin.py
└── serializers/
    ├── product_serializers.py
    └── order_serializers.py
```

| ข้อดี | ข้อเสีย |
|---|---|
| ง่ายสำหรับผู้เริ่มต้นที่คุ้นกับ MVC framework อื่น | เมื่อทำฟีเจอร์หนึ่ง ต้องเปิดไฟล์กระจายอยู่หลายโฟลเดอร์ |
| เห็นภาพรวมของ "model ทั้งหมด" หรือ "view ทั้งหมด" ได้ง่ายในที่เดียว | ไม่สอดคล้องกับการออกแบบ app ตามธรรมชาติของ Django |
| เหมาะกับโปรเจกต์เล็กมาก ๆ ที่มีฟีเจอร์น้อย | ขยายยากเมื่อโปรเจกต์โต เพราะไม่มีขอบเขต (boundary) ที่ชัดเจนระหว่างฟีเจอร์ |

### 46.2 By-Feature Organization (จัดตาม "ฟีเจอร์/โดเมน")

แนวทางนี้คือสิ่งที่ Django **ออกแบบมาให้ใช้เป็นค่าเริ่มต้น** ผ่านระบบ app: จัดกลุ่มไฟล์ตาม
ฟีเจอร์หรือโดเมนทางธุรกิจ โดยแต่ละ app มีทั้ง `models.py`, `views.py`, `admin.py` ของตัวเอง
ครบชุด:

```
myproject/
├── products/
│   ├── models.py
│   ├── views.py
│   ├── admin.py
│   └── serializers.py
├── orders/
│   ├── models.py
│   ├── views.py
│   ├── admin.py
│   └── serializers.py
└── blog/
    ├── models.py
    ├── views.py
    ├── admin.py
    └── serializers.py
```

| ข้อดี | ข้อเสีย |
|---|---|
| ทำงานกับฟีเจอร์หนึ่งได้ครบในโฟลเดอร์เดียว ไม่ต้องกระโดดข้ามหลายที่ | ผู้เริ่มต้นอาจต้องปรับตัวถ้าคุ้นกับ MVC layer-based มาก่อน |
| สอดคล้องกับปรัชญาการออกแบบของ Django โดยตรง (Django "อยากให้" คุณทำแบบนี้) | ถ้าฟีเจอร์เกี่ยวข้องกันมาก อาจต้อง import ข้าม app บ่อย (ต้องออกแบบ dependency ให้ดี) |
| ทีมงานแยกกันดูแลแต่ละ feature/app ได้อย่างอิสระ (ownership ชัดเจน) | |
| ง่ายต่อการเพิ่ม/ลบฟีเจอร์ทั้งชุด (ลบทั้งโฟลเดอร์ app ได้เลย) | |
| Reuse ข้ามโปรเจกต์ได้ง่ายกว่ามาก (ขั้นตอนที่ 47) | |

### 46.3 ตารางสรุปเปรียบเทียบโดยตรง

| ประเด็น | By-Layer | By-Feature |
|---|---|---|
| หน่วยที่ทำงานด้วยบ่อยที่สุด | "ชนิดไฟล์" (model ทั้งหมด, view ทั้งหมด) | "ฟีเจอร์" (blog ทั้งหมด, order ทั้งหมด) |
| ความสอดคล้องกับ Django | ต่ำ (ต้องฝืนโครงสร้าง Django) | สูง (Django ออกแบบมาให้ใช้แบบนี้) |
| Scalability เมื่อโปรเจกต์โต | แย่ลงเรื่อย ๆ | ดีขึ้นเรื่อย ๆ เพราะมี boundary ชัดเจน |
| เหมาะกับทีมขนาด | ทีมเล็กมาก 1-2 คน โปรเจกต์เล็ก | ทีมทุกขนาด โดยเฉพาะทีมใหญ่ที่แบ่งความรับผิดชอบ |
| การ reuse โค้ดข้ามโปรเจกต์ | ยาก | ง่าย (ทั้ง app ย้ายไปได้เลย) |

### 46.4 คำแนะนำสำหรับหลักสูตรนี้

หลักสูตรนี้จะยึดแนวทาง **By-Feature เป็นหลักตลอดทั้งหลักสูตร** ตามปรัชญาดั้งเดิมของ Django
เพราะเป็นแนวทางที่:

- Django framework เองสนับสนุนอย่างเป็นทางการ
- ทีมงานระดับโลก (Instagram, Mozilla, Disqus ที่กล่าวถึงใน Part 001) ใช้แนวทางนี้เป็นหลัก
- ขยายสเกลได้ดีที่สุดเมื่อโปรเจกต์เติบโตจากแอปเดียวไปเป็นหลายสิบแอป

ในบางกรณีที่โปรเจกต์ซับซ้อนมาก ทีมอาจผสมทั้งสองแนวทางเข้าด้วยกัน เช่น จัดแบบ by-feature
ในระดับบนสุด แต่ภายในแอปที่ใหญ่มาก ๆ ตัวหนึ่ง อาจแยกไฟล์ `models.py` ออกเป็นแพ็กเกจย่อย
`models/` ที่มีหลายไฟล์ (จัดแบบ by-layer ในระดับย่อย) — เราจะเห็นตัวอย่างแบบผสมนี้เมื่อโปรเจกต์
ในหลักสูตรใหญ่ขึ้นในภายหลัง (Part 015 เรื่อง Model Meta Options และ Custom Managers)

---

## ขั้นตอนที่ 47: แนวคิด Reusable App

### 47.1 Reusable App คืออะไร

หนึ่งในจุดแข็งที่สำคัญที่สุดของสถาปัตยกรรม Django คือแนวคิด **Reusable App**: การออกแบบแอปให้
**ไม่ผูกติดกับโปรเจกต์ใดโปรเจกต์หนึ่ง** สามารถนำไปติดตั้งในโปรเจกต์อื่นได้ทันทีผ่าน `pip install`
เหมือนกับ library ทั่วไป

ตัวอย่าง reusable app ที่โด่งดังและถูกใช้งานจริงทั่วโลก:

| Package | หน้าที่ | ติดตั้งด้วย |
|---|---|---|
| `django-allauth` | ระบบ authentication + social login (Google, Facebook, GitHub) | `pip install django-allauth` |
| `djangorestframework` | สร้าง REST API (เราจะเรียนทั้ง Phase 5) | `pip install djangorestframework` |
| `django-crispy-forms` | จัดรูปแบบฟอร์มให้สวยงามอัตโนมัติ | `pip install django-crispy-forms` |
| `django-debug-toolbar` | เครื่องมือ debug ประสิทธิภาพระหว่างพัฒนา | `pip install django-debug-toolbar` |
| `django-filter` | ระบบ filter ข้อมูลผ่าน query parameter | `pip install django-filter` |
| `django-cors-headers` | จัดการ CORS header สำหรับ API | `pip install django-cors-headers` |
| `django-environ` | อ่านค่า environment variable แบบสะดวก | `pip install django-environ` |

แอปเหล่านี้ล้วนเป็น Django app ธรรมดา ๆ ที่ทำตามโครงสร้างเดียวกับที่เราสร้างด้วย `startapp`
เพียงแต่ผู้เขียนออกแบบให้ **ไม่พึ่งพา** สิ่งที่เฉพาะเจาะจงกับโปรเจกต์ใดโปรเจกต์หนึ่ง

### 47.2 กฎการเขียน Reusable App ให้ถูกต้อง

การจะทำให้แอปหนึ่ง "reusable" ได้จริง ต้องปฏิบัติตามหลักการเหล่านี้:

1. **ห้าม hardcode ชื่อโปรเจกต์หรือ import จาก project package**: เช่น ห้ามเขียน
   `from config.settings import SOME_VALUE` ในแอปที่ต้องการ reuse เพราะแอปนั้นจะพังทันทีถ้า
   นำไปใช้ในโปรเจกต์ที่ project package ชื่ออื่น ให้ใช้ `django.conf.settings` แทนเสมอ:

   ```python
   # ผิด - ผูกติดกับโปรเจกต์เฉพาะ
   from config.settings import BLOG_POSTS_PER_PAGE

   # ถูก - ดึงผ่าน django.conf.settings พร้อมค่า default
   from django.conf import settings

   POSTS_PER_PAGE = getattr(settings, 'BLOG_POSTS_PER_PAGE', 10)
   ```

2. **ให้ค่า default ที่สมเหตุสมผลเสมอ**: reusable app ที่ดีควรทำงานได้ "ทันทีที่ติดตั้ง" โดยไม่ต้อง
   ตั้งค่าอะไรเพิ่มเติมเลย (sensible defaults) การตั้งค่าเพิ่มเติมควรเป็น "ทางเลือก" ไม่ใช่ "บังคับ"
3. **Namespace ชื่อ setting ให้ชัดเจน**: ตั้งชื่อ setting ทั้งหมดด้วย prefix ของแอป เช่น
   `BLOG_POSTS_PER_PAGE` ไม่ใช่ `POSTS_PER_PAGE` เฉย ๆ เพื่อไม่ให้ชนกับ setting ของแอปอื่น
4. **Templates และ static files ต้องอยู่ใน namespace ของแอปเอง**: เช่น
   `blog/templates/blog/post_list.html` (ซ้อนโฟลเดอร์ชื่อแอปอีกชั้น) ไม่ใช่
   `blog/templates/post_list.html` ตรง ๆ เพื่อป้องกัน template ชื่อชนกับแอปอื่นเมื่อถูกติดตั้ง
   ร่วมกับแอปอื่นในโปรเจกต์เดียวกัน (เราจะอธิบายเหตุผลเชิงลึกเรื่องนี้ใน Part 008)
5. **เขียน `AppConfig.name` แบบ dotted path ที่ยืดหยุ่น** และไม่ควรอ้างอิงโครงสร้างโฟลเดอร์ของ
   โปรเจกต์ปลายทาง
6. **แนบ migration files มาด้วยเสมอ**: reusable app ที่มี model ต้องมี migration พร้อมใช้
   ไม่ควรบังคับให้ผู้ใช้ต้องรัน `makemigrations` เอง
7. **เขียน `README`, `LICENSE`, และระบุ dependency ให้ชัดเจน** ใน `pyproject.toml`

### 47.3 โครงสร้างแพ็กเกจสำหรับเผยแพร่บน PyPI

หากต้องการนำแอปของคุณไปเผยแพร่บน PyPI จริง (เช่นตั้งชื่อสมมติว่า `django-simple-blog`)
โครงสร้างไฟล์มาตรฐานจะเป็นประมาณนี้:

```
django-simple-blog/                  ← root ของ package repository
├── src/
│   └── simple_blog/                 ← ตัวแอป Django จริง ๆ (ติดตั้งแล้วจะ import เป็น simple_blog)
│       ├── __init__.py
│       ├── apps.py
│       ├── models.py
│       ├── admin.py
│       ├── views.py
│       ├── migrations/
│       │   └── __init__.py
│       ├── templates/
│       │   └── simple_blog/
│       │       └── post_list.html
│       └── static/
│           └── simple_blog/
│               └── css/
│                   └── style.css
├── tests/
│   └── test_models.py
├── pyproject.toml
├── README.md
├── LICENSE
└── .gitignore
```

### 47.4 ตัวอย่าง `pyproject.toml` สำหรับ Reusable App

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"

[project]
name = "django-simple-blog"
version = "0.1.0"
description = "แอป Django สำเร็จรูปสำหรับระบบบล็อกอย่างง่าย"
readme = "README.md"
license = { text = "MIT" }
requires-python = ">=3.10"
dependencies = [
    "Django>=4.2",
]

[project.urls]
Homepage = "https://github.com/yourname/django-simple-blog"

[tool.setuptools.packages.find]
where = ["src"]
```

เมื่อมีโครงสร้างแบบนี้ ผู้ใช้คนอื่นสามารถติดตั้งแอปของคุณด้วย `pip install django-simple-blog`
แล้วเพิ่ม `'simple_blog'` เข้า `INSTALLED_APPS` ของโปรเจกต์ตัวเองได้ทันที เหมือนกับที่เราติดตั้ง
`djangorestframework` หรือ `django-allauth` ทุกประการ

### 47.5 ในหลักสูตรนี้เมื่อไหร่จะได้เขียน Reusable App จริง

แอป `blog` ที่เราสร้างใน Part นี้ยังไม่จำเป็นต้อง reusable 100% เพราะเป็นแอปหลักของโปรเจกต์
เรียน (project-specific app) แต่หลักการที่เรียนในขั้นตอนนี้จะถูกนำไปใช้จริงเมื่อถึง **Part 047
(แนวคิด Reusable App เชิงลึก และการ publish ขึ้น PyPI)** ซึ่งอยู่ใน Phase 5 — ตอนนั้นคุณจะได้
แปลงแอปหนึ่งในโปรเจกต์ของคุณให้กลายเป็น pip package ที่ติดตั้งได้จริง

---

## ขั้นตอนที่ 48: ทำความเข้าใจ `__init__.py` และ App Registry

### 48.1 `__init__.py` ทำอะไรได้บ้าง

เราพูดถึง `__init__.py` แบบผิวเผินในขั้นตอนที่ 42 มาแล้วว่ามันบอกให้ Python รู้ว่าโฟลเดอร์นี้เป็น
package แต่ไฟล์นี้ยังมีประโยชน์เพิ่มเติมอีกหลายอย่างที่ควรรู้:

**1. Import shortcut**: คุณสามารถ re-export บางอย่างผ่าน `__init__.py` เพื่อให้ import สั้นลง:

```python
# blog/__init__.py
from .apps import BlogConfig

default_app_config = 'blog.apps.BlogConfig'  # ⚠️ รูปแบบเก่า ไม่ต้องใช้แล้ว (ดูหมายเหตุด้านล่าง)
```

> **หมายเหตุประวัติศาสตร์**: ในอดีต (Django 1.7-3.1) หากต้องการให้ Django ใช้ `AppConfig`
> ที่ไม่ใช่ค่า default ต้องประกาศตัวแปร `default_app_config` ใน `__init__.py` แบบข้างบน
> แต่ตั้งแต่ **Django 3.2 เป็นต้นไป กลไกนี้ถูกยกเลิกแล้ว (deprecated แล้วลบออก)** เพราะ Django
> เปลี่ยนมาใช้การ "ตรวจจับอัตโนมัติ" (auto-detection) ที่เราเรียนในขั้นตอนที่ 43.2 แทน — ถ้าคุณ
> เจอโค้ดเก่าที่มี `default_app_config` ใน tutorial หรือ package เก่า ให้รู้ว่านั่นคือรูปแบบที่
> **ล้าสมัยแล้วสำหรับ Django 5.x** สามารถลบทิ้งได้อย่างปลอดภัย

**2. Package-level configuration** สำหรับ third-party library บางตัว: บาง reusable app ใช้
`__init__.py` เก็บค่าคงที่ระดับแพ็กเกจ เช่น `__version__ = '1.2.0'`

ในกรณีของแอป `blog` ของเราในหลักสูตรนี้ **ให้ปล่อย `__init__.py` ว่างเปล่าไว้แบบนั้น** ตามที่
Django สร้างให้ ไม่มีความจำเป็นต้องเขียนอะไรเพิ่มในไฟล์นี้สำหรับกรณีการใช้งานทั่วไป

### 48.2 App Registry คืออะไร

**App Registry** (`django.apps.apps`) คือ "ทะเบียนกลาง" ที่ Django ใช้เก็บข้อมูลของทุกแอปและ
ทุก model ที่โหลดเข้าระบบ มันคือ object ที่มีเมธอดให้ query ข้อมูล เช่น:

```python
# ทดลองใน Django shell: python manage.py shell
from django.apps import apps

# ดูรายชื่อแอปทั้งหมดที่โหลดอยู่
for app_config in apps.get_app_configs():
    print(app_config.name, '->', app_config.verbose_name)

# ดึง AppConfig ของแอปหนึ่งโดยเฉพาะด้วย label
blog_config = apps.get_app_config('blog')
print(blog_config.verbose_name)   # ระบบจัดการบล็อก
print(blog_config.path)           # /path/to/django-mastery-course/blog

# ดึง model class ทั้งหมดของแอปหนึ่ง (มีประโยชน์มากเมื่อเขียน generic/reusable code)
for model in blog_config.get_models():
    print(model.__name__)

# ตรวจสอบว่าแอปนี้ถูกติดตั้งหรือไม่ (มีประโยชน์เมื่อเขียนโค้ดที่ทำงานแบบ "optional dependency")
print(apps.is_installed('blog'))          # True
print(apps.is_installed('django.contrib.admin'))  # True

# ดึง model class จาก string โดยตรง (มีประโยชน์เมื่อไม่ต้องการ import แบบ hard-coded)
Post = apps.get_model('blog', 'Post')
```

App Registry คือกลไกภายในที่ทำให้ Django รู้ว่า "มีแอปอะไรบ้าง แต่ละแอปมี model อะไรบ้าง"
และเป็นสิ่งที่ทำงานอยู่เบื้องหลังทุกครั้งที่คุณรัน `makemigrations`, `migrate`, เข้าหน้า Admin
หรือแม้แต่การ `import` model ใด ๆ ในระบบ

### 48.3 ลำดับการโหลดแอปของ Django (Application Loading Order)

เมื่อ Django เริ่มทำงาน (ไม่ว่าจะผ่าน `runserver`, `manage.py` คำสั่งใด ๆ, หรือ WSGI/ASGI
server) จะมีลำดับขั้นตอนการโหลดที่ตายตัวดังนี้:

```
1. Django อ่านไฟล์ settings.py และตัวแปร INSTALLED_APPS
        │
        ▼
2. สำหรับแต่ละแอปใน INSTALLED_APPS (เรียงตามลำดับที่ประกาศ):
   - Import แอปนั้น (import โมดูล เช่น import blog)
   - หา AppConfig ที่เหมาะสม (ตามที่อธิบายในขั้นตอนที่ 43.2)
   - สร้าง instance ของ AppConfig และเก็บไว้ใน registry
        │
        ▼
3. เมื่อ AppConfig ของทุกแอปถูกสร้างครบแล้ว (populate เสร็จขั้นที่ 1)
   Django จะ import โมดูล models.py ของทุกแอป
        │
        ▼
4. เมื่อ model ของทุกแอปถูก import และลงทะเบียนครบแล้ว (populate เสร็จขั้นที่ 2)
   ระบบ "apps.ready" ถือว่าพร้อมสมบูรณ์
        │
        ▼
5. Django เรียก method ready() ของทุก AppConfig ตามลำดับใน INSTALLED_APPS
   (นี่คือจังหวะที่ปลอดภัยที่สุดในการ import model และลงทะเบียน signal)
        │
        ▼
6. ระบบพร้อมรับ request / รัน management command
```

นี่คือเหตุผลเชิงลึกว่าทำไมในขั้นตอนที่ 44.4 เราถึงย้ำว่า **ห้าม import model โดยตรงในระดับบนสุด
ของ `apps.py`** — เพราะตอนที่ `apps.py` กำลังถูกประมวลผล (ขั้นที่ 2) ยังอยู่ในช่วงที่ Django
กำลังสร้าง `AppConfig` เท่านั้น ยังไม่ถึงขั้นที่ 3 ที่จะ import `models.py` เลย การพยายาม import
model ในจังหวะนี้จะทำให้เกิด error `AppRegistryNotReady`

### 48.4 ทดลองดูลำดับการโหลดด้วยตัวเอง

เพื่อความเข้าใจที่ชัดเจน ลองเพิ่ม `print()` ชั่วคราวใน `apps.py`:

```python
# blog/apps.py (ทดลองชั่วคราว เพื่อดูลำดับการโหลด)
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'blog'
    verbose_name = 'ระบบจัดการบล็อก'

    def ready(self):
        print('>>> blog.apps.BlogConfig.ready() ถูกเรียกแล้ว')
        import blog.signals  # noqa: F401
```

รัน `python manage.py runserver` แล้วดูใน terminal คุณจะเห็นข้อความ
`>>> blog.apps.BlogConfig.ready() ถูกเรียกแล้ว` แสดงขึ้นมาก่อนที่ server จะเริ่มรอรับ request
(อย่าลืมลบ `print()` ทดลองนี้ออกหลังทดสอบเสร็จ)

### 48.5 คำสั่งตรวจสอบแอปที่มีประโยชน์

```bash
# แสดงรายชื่อแอปทั้งหมด พร้อม model ที่มีในแต่ละแอป
python manage.py showmigrations

# ตรวจสอบว่าโปรเจกต์มี configuration ผิดพลาดหรือไม่ (รวมถึงปัญหาการลงทะเบียนแอป)
python manage.py check

# เปิด Django shell เพื่อสำรวจ app registry แบบ interactive
python manage.py shell
```

คำสั่ง `python manage.py check` เป็นคำสั่งที่ควรรันเป็นนิสัยก่อน commit หรือ deploy ทุกครั้ง
เพราะมันจะตรวจจับปัญหาการตั้งค่าแอปที่ผิดพลาดได้ตั้งแต่เนิ่น ๆ ก่อนที่จะกลายเป็นปัญหาใน
production

---

## ขั้นตอนที่ 49: โครงสร้างโปรเจกต์ขนาดใหญ่แบบมืออาชีพ

### 49.1 ปัญหาของโครงสร้างแบบ Flat เมื่อโปรเจกต์โต

โครงสร้างที่เราใช้อยู่ตอนนี้ (วางทุกแอปไว้ที่ root ของโปรเจกต์เรียงกัน) เรียกว่า **flat structure**:

```
django-mastery-course/
├── manage.py
├── config/
├── blog/
├── accounts/
├── shop/
├── orders/
├── payments/
├── reviews/
├── notifications/
└── analytics/
```

เมื่อโปรเจกต์มีแอปมากขึ้นเรื่อย ๆ (โปรเจกต์ enterprise จริงอาจมี 20-50 แอปขึ้นไป) โครงสร้างแบบนี้
เริ่มมีปัญหา:

- root ของโปรเจกต์รกมาก ปนกันระหว่างไฟล์ configuration (`manage.py`, `requirements.txt`,
  `.gitignore`, `Dockerfile`) กับโฟลเดอร์แอปจำนวนมาก
- ยากที่จะแยกแยะด้วยสายตาว่าอันไหนคือแอปของเรา อันไหนคือไฟล์ตั้งค่าโปรเจกต์
- Editor/IDE ที่แสดง sidebar แบบ alphabetical จะปนกันไปหมดระหว่าง config files กับ app folders
- ยากต่อการทำ `.gitignore` หรือ deployment script ที่ต้องการแยกกลุ่ม "โค้ดแอปพลิเคชัน" ออกจาก
  "ไฟล์ระดับโปรเจกต์"

### 49.2 ย้าย Apps เข้าโฟลเดอร์ `apps/`

แนวทางที่ทีมงานระดับมืออาชีพจำนวนมากใช้คือการย้ายแอปทั้งหมดเข้าไปอยู่ในโฟลเดอร์ `apps/`
เพื่อแยกให้ชัดเจนจากไฟล์ configuration ระดับโปรเจกต์:

```
django-mastery-course/
├── venv/
├── manage.py
├── requirements.txt
├── .gitignore
├── config/                  ← การตั้งค่าโปรเจกต์ (ยังอยู่ที่เดิม)
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
└── apps/                    ← แอปทั้งหมดของเราอยู่รวมกันที่นี่
    ├── __init__.py
    ├── blog/
    │   ├── __init__.py
    │   ├── apps.py
    │   ├── models.py
    │   ├── views.py
    │   ├── admin.py
    │   └── migrations/
    ├── accounts/
    │   └── ...
    └── orders/
        └── ...
```

### 49.3 ขั้นตอนการย้ายแอป `blog` เข้าโฟลเดอร์ `apps/` จริง

ทำตามลำดับนี้ (ตัวอย่างสำหรับผู้ที่ต้องการทดลองโครงสร้างนี้ในโปรเจกต์ของตัวเอง):

```bash
# 1. สร้างโฟลเดอร์ apps/ พร้อม __init__.py เพื่อให้เป็น Python package
mkdir apps
touch apps/__init__.py

# 2. ย้ายแอป blog เข้าไป
mv blog apps/blog
```

ต่อมาต้องแก้ไข **2 จุดสำคัญ** ให้ Django หาแอปที่ย้ายไปเจอ:

**จุดที่ 1: แก้ `name` ใน `apps/blog/apps.py` ให้ตรงกับตำแหน่งใหม่**

```python
# apps/blog/apps.py
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.blog'          # ← เปลี่ยนจาก 'blog' เป็น 'apps.blog'
    verbose_name = 'ระบบจัดการบล็อก'

    def ready(self):
        import apps.blog.signals  # noqa: F401  ← path ก็ต้องเปลี่ยนตาม
```

**จุดที่ 2: แก้ `INSTALLED_APPS` ใน `config/settings.py`**

```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Local apps
    'apps.blog.apps.BlogConfig',   # ← ระบุ path เต็มเสมอเมื่อแอปอยู่ในแพ็กเกจย่อย
]
```

> **ทำไมต้องระบุ path เต็ม `'apps.blog.apps.BlogConfig'` แทนที่จะเขียนสั้น ๆ ว่า `'apps.blog'`?**
> ในทางเทคนิคแล้ว `'apps.blog'` แบบสั้นก็ยังใช้งานได้ (Django จะ auto-detect
> `AppConfig` ให้เหมือนเดิมตามหลักการในขั้นตอนที่ 43.2) แต่ทีมงานมืออาชีพจำนวนมากนิยมเขียน
> path เต็มเมื่อแอปอยู่ในแพ็กเกจย่อย เพื่อความชัดเจนแบบ **explicit over implicit** และช่วยให้
> IDE/นักพัฒนาใหม่ในทีม resolve ตำแหน่งไฟล์จริงได้เร็วขึ้นโดยไม่ต้องเดา

### 49.4 ทำไมไม่จำเป็นต้องแก้ `sys.path` (ต่างจากบาง tutorial เก่า)

Tutorial บางแหล่งแนะนำให้เพิ่ม `sys.path.insert()` ใน `settings.py` เพื่อให้ import
`blog` แบบสั้น ๆ ได้โดยไม่ต้องพิมพ์ `apps.blog` เต็ม ๆ:

```python
# config/settings.py — วิธีนี้ "ใช้ได้" แต่ไม่แนะนำในหลักสูตรนี้
import sys
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(BASE_DIR / 'apps'))

INSTALLED_APPS = [
    # ...
    'blog',   # import แบบสั้น เพราะ apps/ ถูกเพิ่มเข้า sys.path แล้ว
]
```

**หลักสูตรนี้ไม่แนะนำวิธีนี้** เพราะมีข้อเสียคือ:

- ทำให้ `import blog` ในไฟล์ Python ที่ไหนก็ได้ทำงานได้แบบ "มายากล" โดยไม่รู้ที่มา ยากต่อการ
  ไล่ debug ว่า `blog` มาจากไหนเมื่อโปรเจกต์ใหญ่ขึ้น
- เครื่องมือ static analysis และ IDE บางตัวสับสนกับการ manipulate `sys.path` แบบ manual
- ขัดกับหลักการ **explicit is better than implicit** ของ Python (The Zen of Python)

การใช้ **dotted package path เต็ม** (`apps.blog`) จึงเป็นวิธีที่ Django และชุมชนแนะนำมากกว่า
ในโปรเจกต์ที่จัดโครงสร้างแบบนี้ — เพราะ `apps/` มี `__init__.py` ทำให้เป็น regular Python
package ที่ import ได้ตามปกติโดยไม่ต้องแก้ `sys.path` เลย

### 49.5 โครงสร้างโปรเจกต์ระดับ Enterprise แบบเต็มรูปแบบ

เมื่อโปรเจกต์เติบโตเต็มที่ (แบบที่คุณจะเห็นในโปรเจกต์จริง Phase 12 ของหลักสูตร) โครงสร้าง
มักจะหน้าตาประมาณนี้:

```
django-mastery-course/
├── .github/
│   └── workflows/
│       └── ci.yml                 # GitHub Actions (Part 088)
├── config/
│   ├── __init__.py
│   ├── settings/                  # แยก settings เป็นหลายไฟล์ (Part 010)
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── apps/
│   ├── __init__.py
│   ├── blog/
│   ├── accounts/
│   ├── orders/
│   └── payments/
├── static/                        # static files ระดับโปรเจกต์ (Part 009)
├── templates/                     # templates ระดับโปรเจกต์ (base.html ฯลฯ)
├── media/                         # ไฟล์ที่ผู้ใช้อัปโหลด (ไม่ commit เข้า git)
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── tests/                         # integration test ระดับโปรเจกต์ (Part 064)
├── docs/
├── manage.py
├── .env.example
├── .gitignore
└── README.md
```

ในหลักสูตรนี้เราจะ**ค่อย ๆ วิวัฒนาการ**ไปสู่โครงสร้างแบบนี้ทีละขั้นตามความจำเป็นของแต่ละ Phase
(เช่น การแยก `settings/` เป็นหลายไฟล์จะเรียนใน Part 010, การจัดการ `requirements/` แยกตาม
environment จะเรียนใน Part 084) ไม่จำเป็นต้องรีบทำทุกอย่างตั้งแต่ตอนนี้ — สำหรับ Part 005 นี้
ขอให้คุณเข้าใจ**หลักการ**ของการจัดโครงสร้างเป็นหลัก ส่วนแอป `blog` ที่จะสร้างในขั้นตอนที่ 50
เราจะ**ยังคงเก็บไว้ที่ root** (โครงสร้าง flat) เพื่อความเรียบง่ายในช่วงต้นของหลักสูตร และจะย้าย
เข้าโฟลเดอร์ `apps/` จริงเมื่อจำนวนแอปเริ่มมากขึ้นในภายหลัง

---

## ขั้นตอนที่ 50: สรุปและแบบฝึกหัด

### 50.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างระหว่าง Project กับ App ใน Django
- ✅ สร้างแอปแรกด้วย `python manage.py startapp blog` และเข้าใจโครงสร้างไฟล์ทุกไฟล์ที่ได้
- ✅ เจาะลึกหน้าที่ของ `models.py`, `views.py`, `apps.py`, `admin.py`, `tests.py`,
  `migrations/`
- ✅ ลงทะเบียนแอปใน `INSTALLED_APPS` และเข้าใจความแตกต่างระหว่าง `'blog'` กับ
  `'blog.apps.BlogConfig'`
- ✅ ปรับแต่ง `AppConfig`: `verbose_name`, `default_auto_field`, และ `ready()` สำหรับโหลด
  signals
- ✅ เข้าใจหลักการ Single Responsibility ระดับ app และสัญญาณที่บ่งบอกว่าควรแยกแอป
- ✅ เปรียบเทียบรูปแบบการจัดองค์กรแบบ By-Feature กับ By-Layer และเข้าใจว่า Django ออกแบบมา
  เพื่อ By-Feature
- ✅ เข้าใจแนวคิด Reusable App และกฎการเขียนแอปให้นำไปใช้ซ้ำข้ามโปรเจกต์ได้
- ✅ เข้าใจ `__init__.py` และลำดับการโหลดแอปของ Django ผ่าน App Registry
- ✅ รู้จักโครงสร้างโปรเจกต์ระดับมืออาชีพที่ย้ายแอปเข้าโฟลเดอร์ `apps/`
- ✅ สร้างแอป `blog` พร้อม Post model เบื้องต้น เตรียมพร้อมสำหรับ Part 011

### 50.2 ลงมือทำจริง: สร้าง Post model เบื้องต้น

ก่อนแบบฝึกหัด เรามาสร้างพื้นฐานที่จำเป็นเพื่อเตรียมพร้อมสำหรับ Part 011 ก่อน (ยังไม่ต้องเข้าใจ
รายละเอียดของ field ทุกตัวลึกซึ้ง เพราะจะเรียนเต็มรูปแบบใน Part 011 — ตอนนี้ขอให้ลงมือทำตาม
เพื่อให้แอป `blog` มีของจริงอยู่ในนั้น)

เปิดไฟล์ `blog/models.py` แล้วแก้ไขดังนี้:

```python
# blog/models.py
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    def __str__(self):
        return self.title
```

จากนั้นสร้างและรัน migration:

```bash
python manage.py makemigrations blog
```

คุณควรเห็นผลลัพธ์ประมาณนี้:

```
Migrations for 'blog':
  blog/migrations/0001_initial.py
    - Create model Post
```

```bash
python manage.py migrate
```

```
Operations to perform:
  Apply all migrations: admin, auth, blog, contenttypes, sessions
Running migrations:
  Applying blog.0001_initial... OK
```

ลงทะเบียน model เข้า Django Admin เพื่อทดสอบว่าใช้งานได้จริง:

```python
# blog/admin.py
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

ทดสอบด้วยการรัน server แล้วสร้าง superuser (หากยังไม่เคยสร้าง):

```bash
python manage.py createsuperuser
python manage.py runserver
```

เปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/admin/` login ด้วย superuser ที่สร้างไว้ คุณควร
เห็นหัวข้อ **"ระบบจัดการบล็อก"** (จาก `verbose_name` ที่ตั้งไว้ในขั้นตอนที่ 44.2) พร้อมเมนู
`Posts` ที่กดเข้าไปเพิ่ม/แก้ไข/ลบข้อมูลได้ทันที นี่คือพลังของ Django Admin ที่เราจะเจาะลึกเต็ม
รูปแบบใน Part 017-018

โครงสร้างไฟล์สุดท้ายของแอป `blog` หลังจบ Part นี้:

```
blog/
├── __init__.py
├── admin.py          # ลงทะเบียน Post เข้า admin
├── apps.py           # BlogConfig พร้อม verbose_name และ ready()
├── migrations/
│   ├── __init__.py
│   └── 0001_initial.py
├── models.py         # Post model
├── signals.py        # notify_new_post signal receiver
├── tests.py
└── views.py
```

### 50.3 Checklist ก่อนไป Part ถัดไป

- [ ] รันคำสั่ง `python manage.py startapp blog` สำเร็จ และเห็นโครงสร้างไฟล์ครบทุกไฟล์
- [ ] เพิ่ม `'blog'` เข้า `INSTALLED_APPS` ใน `config/settings.py` แล้ว
- [ ] เข้าใจความแตกต่างระหว่าง `'blog'` กับ `'blog.apps.BlogConfig'` อธิบายให้คนอื่นฟังได้
- [ ] ปรับแต่ง `verbose_name` ใน `apps.py` เป็นภาษาไทยสำเร็จ และเห็นผลในหน้า Admin
- [ ] สร้างไฟล์ `blog/signals.py` และเชื่อมกับ `ready()` ใน `apps.py` สำเร็จ
- [ ] สร้าง `Post` model เบื้องต้น รัน `makemigrations` และ `migrate` สำเร็จโดยไม่มี error
- [ ] ลงทะเบียน `Post` เข้า Django Admin และสร้าง/แก้ไขข้อมูลผ่านหน้า Admin ได้จริง
- [ ] อธิบายความแตกต่างระหว่าง By-Feature กับ By-Layer organization ได้ด้วยคำพูดตัวเอง
- [ ] รันคำสั่ง `python manage.py check` แล้วไม่มี error หรือ warning ที่เกี่ยวกับแอป

### 50.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้างแอปใหม่อีกหนึ่งแอปชื่อ `comments` (ยังไม่ต้องมี model ใด ๆ) ลงทะเบียนเข้า
`INSTALLED_APPS` ให้ถูกต้อง แล้วปรับแต่ง `apps.py` ให้มี `verbose_name = 'ระบบความคิดเห็น'`
จากนั้นรัน `python manage.py check` เพื่อยืนยันว่าไม่มี error ก่อนลบแอปนี้ทิ้ง (เพื่อฝึกกระบวนการ
สร้าง/ลงทะเบียน/ตรวจสอบแอปให้คล่อง)

**แบบฝึกหัดที่ 2**: เพิ่ม field ใหม่ในโมเดล `Post` ชื่อ `slug` (ชนิด `SlugField`, `max_length=220`,
`unique=True`, ตั้งค่า default เป็น string ว่างชั่วคราวด้วย `default=''` เพื่อให้ migration ผ่านได้
ง่ายในตอนนี้) แล้วรัน `makemigrations` และ `migrate` ใหม่ สังเกตว่าไฟล์ migration ใหม่
(`0002_...py`) ถูกสร้างขึ้นมาอย่างไร (เราจะเรียนเรื่อง `SlugField` และการใช้งานจริงแบบเต็มใน
Part 011)

**แบบฝึกหัดที่ 3**: เขียนโค้ดทดลองใน Django shell (`python manage.py shell`) เพื่อสำรวจ App
Registry ด้วยตัวเอง: (ก) แสดงรายชื่อแอปทั้งหมดที่ติดตั้งในโปรเจกต์พร้อม `verbose_name` ของแต่ละ
แอป (ข) ใช้ `apps.get_model('blog', 'Post')` เพื่อดึง model class แล้วลองสร้าง object ทดสอบ
ด้วยเมธอดนี้แทนการ `import` ตรง ๆ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ลองทำตามขั้นตอนที่ 49.3 อย่างครบถ้วน โดยสร้างโฟลเดอร์ `apps/`
แล้วย้ายแอป `blog` เข้าไปจริง แก้ไข `name` ใน `apps.py`, แก้ไข `INSTALLED_APPS`, และแก้ import
path ใน `ready()` ให้ถูกต้องทั้งหมด จากนั้นรัน `python manage.py runserver` และ
`python manage.py check` เพื่อยืนยันว่าทุกอย่างยังทำงานได้ปกติ (คำใบ้: อย่าลืมว่า migration
history ที่มีอยู่แล้วจะยังอ้างอิงถึง app label เดิม — ลองสังเกตดูว่า `app_label` ในระบบ migration
เปลี่ยนไปหรือไม่ระหว่างการย้าย และถ้าต้องการย้อนกลับมาโครงสร้างแบบเดิม ให้ `mv apps/blog blog`
กลับ และแก้ไขค่าต่าง ๆ ให้ตรงกับเดิม)

### 50.5 คำถามที่พบบ่อย (FAQ)

**Q: ลืม `startapp` ทำให้ `apps.py` หายไป แก้ไขยังไง?**
A: ไม่จำเป็นต้องรัน `startapp` ใหม่ทั้งหมด (ซึ่งจะเขียนทับไฟล์อื่นที่คุณแก้ไปแล้ว) เพียงสร้างไฟล์
`apps.py` เองแล้ว copy โครงสร้างจากตัวอย่างในขั้นตอนที่ 42.4 มาใช้ได้เลย

**Q: ทำไมรัน `startapp` แล้วได้ error ว่า "'blog' is not a valid app name"?**
A: ชื่อแอปต้องเป็น valid Python identifier เท่านั้น (ห้ามมีขีดกลาง `-`, ห้ามขึ้นต้นด้วยตัวเลข,
ห้ามมีช่องว่าง) ตรวจสอบว่าคุณพิมพ์ `python manage.py startapp blog` ไม่ใช่ `startapp my-blog`

**Q: เพิ่มแอปเข้า `INSTALLED_APPS` แล้ว แต่ยังไม่เห็นผลลัพธ์ ต้องรีสตาร์ท server ไหม?**
A: ต้องรีสตาร์ท `runserver` ทุกครั้งที่แก้ไข `settings.py` เพราะ Django อ่านค่า settings เพียง
ครั้งเดียวตอนเริ่มโปรเซส (ต่างจากไฟล์ `.py` ของ view/model ที่ Django dev server จะ auto-reload
ให้อัตโนมัติเมื่อบันทึกไฟล์)

**Q: จำเป็นต้องมี `verbose_name` ทุกแอปไหม?**
A: ไม่จำเป็น ถ้าไม่ระบุ Django จะสร้างชื่อแบบ title case ให้อัตโนมัติจากชื่อแอป (เช่น `blog` →
`Blog`) การระบุ `verbose_name` เป็นทางเลือกเพื่อความเป็นมิตรกับผู้ใช้งานหน้า Admin โดยเฉพาะเมื่อ
ต้องการแสดงเป็นภาษาไทยหรือชื่อที่สื่อความหมายชัดเจนกว่า

**Q: แอปหนึ่งสามารถอยู่ได้หลายโปรเจกต์พร้อมกันไหม (ใช้ path เดียวกัน)?**
A: ในทางเทคนิคทำได้ถ้าแอปนั้นถูกออกแบบเป็น reusable app อย่างถูกต้อง (ขั้นตอนที่ 47) และติดตั้ง
ผ่าน `pip install -e .` (editable install) เพื่อชี้ไปที่ path เดียวกัน แต่ในทางปฏิบัติ วิธีที่
ปลอดภัยและเป็นมาตรฐานกว่าคือ publish เป็น package แล้วติดตั้งแยกในแต่ละ virtual environment
ของแต่ละโปรเจกต์

**Q: ควรใช้โครงสร้าง `apps/` ตั้งแต่ Part นี้เลยไหม หรือรอไปก่อน?**
A: สำหรับหลักสูตรนี้ เราจะคงโครงสร้าง flat (แอปอยู่ที่ root) ต่อไปจนกว่าจำนวนแอปจะเริ่มมากพอที่
จะเห็นประโยชน์ของการจัดกลุ่มชัดเจน ในโปรเจกต์ของคุณเองที่ทำคู่ขนาน หากคาดว่าจะมีมากกว่า 5-6
แอปตั้งแต่ต้น การเริ่มด้วยโครงสร้าง `apps/` ตั้งแต่แรกก็เป็นทางเลือกที่สมเหตุสมผลเช่นกัน

### 50.6 เตรียมตัวสำหรับ Part ถัดไป

**Part 006: URL Routing และ URLconf เบื้องต้น** จะพาคุณไปสร้างไฟล์ `blog/urls.py` เชื่อมโยง
URL เข้ากับ view ของแอป `blog` ที่เพิ่งสร้าง เรียนรู้การใช้ `path()`, `include()`, URL parameter
แบบ dynamic (เช่น `<int:post_id>`), การตั้งชื่อ URL pattern ด้วย `name=` เพื่อใช้กับ
`{% url %}` template tag และโครงสร้าง URLconf แบบสองชั้น (project-level `urls.py` ที่
`include()` ไปยัง app-level `urls.py`) ซึ่งเป็นรูปแบบมาตรฐานที่ทุกโปรเจกต์ Django ระดับมืออาชีพ
ใช้กัน

หลังจาก Part 006 คุณจะสามารถเข้าถึง URL อย่าง `http://127.0.0.1:8000/blog/` และเห็นข้อมูลจริง
จาก `Post` model ที่สร้างไว้ใน Part นี้ผ่าน view ที่เขียนขึ้นเอง เตรียมเปิด `blog/views.py` ทิ้งไว้
ให้พร้อม แล้วไปต่อกันเลย!
