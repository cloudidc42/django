# Part 004: สร้างโปรเจกต์ Django แรกและทำความเข้าใจโครงสร้าง

> **ขั้นตอนที่ 31-40 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: ลงมือสร้างโปรเจกต์ Django จริงเป็นครั้งแรกด้วยคำสั่ง
> `django-admin startproject` แล้วเจาะลึกทุกไฟล์ที่ถูกสร้างขึ้นมาว่าทำหน้าที่อะไร
> โดยเฉพาะ `settings.py` ที่เป็นหัวใจของการตั้งค่าทั้งโปรเจกต์ เรียนรู้คำสั่ง
> `manage.py` ที่สำคัญที่สุด เข้าใจความแตกต่างระหว่าง WSGI กับ ASGI แนวคิด
> Project vs App จัดการ secret ด้วย `.env` อย่างมืออาชีพ และปิดท้ายด้วยการเขียน
> View แรกเพื่อพิสูจน์ว่าทุกอย่างทำงานได้จริงบนเบราว์เซอร์ของคุณ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 31: ใช้คำสั่ง `django-admin startproject` สร้างโปรเจกต์แรก
- ขั้นตอนที่ 32: เจาะลึกโครงสร้างไฟล์ที่ถูกสร้าง (manage.py, settings.py, urls.py, wsgi.py, asgi.py) ทีละไฟล์
- ขั้นตอนที่ 33: เจาะลึก settings.py (INSTALLED_APPS, MIDDLEWARE, DATABASES, TEMPLATES, ALLOWED_HOSTS)
- ขั้นตอนที่ 34: รัน development server ด้วย `python manage.py runserver` และทำความเข้าใจ output/warning
- ขั้นตอนที่ 35: ภาพรวมคำสั่ง manage.py ที่สำคัญ (migrate, createsuperuser, shell, check, dbshell)
- ขั้นตอนที่ 36: WSGI vs ASGI คืออะไร ต่างกันอย่างไร เกี่ยวข้องกับ deployment อย่างไร
- ขั้นตอนที่ 37: แนวคิด Project vs App ใน Django (ทำไม Django แยกสองสิ่งนี้)
- ขั้นตอนที่ 38: จัดการ config/secret ด้วยไฟล์ .env และ library django-environ
- ขั้นตอนที่ 39: เขียน View แรกแบบไม่ต้องสร้าง app (inline ใน urls.py) แสดงผล "Hello Django"
- ขั้นตอนที่ 40: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 31: ใช้คำสั่ง `django-admin startproject` สร้างโปรเจกต์แรก

### 31.1 เตรียมความพร้อมก่อนเริ่ม

จาก Part 001 และ Part 003 คุณควรมีสิ่งเหล่านี้พร้อมแล้ว:

- Python 3.12+ ติดตั้งบนเครื่อง
- Virtual environment ถูกสร้างและ activate อยู่ (เห็น `(venv)` หน้า prompt)
- Django ติดตั้งอยู่ใน venv นั้นแล้ว (`pip install django`)

ตรวจสอบให้แน่ใจอีกครั้งก่อนเริ่ม:

```bash
# ต้องเห็น (venv) หน้า prompt ก่อนเสมอ
which python
# ควรชี้ไปที่ .../venv/bin/python ไม่ใช่ /usr/bin/python

python -m django --version
# ควรได้ 5.1.x หรือใกล้เคียง
```

ถ้ายังไม่เห็น `(venv)` ให้ activate ก่อน:

```bash
# macOS/Linux
source venv/bin/activate

# Windows PowerShell
venv\Scripts\Activate.ps1
```

**กฎเหล็ก**: ห้ามรันคำสั่ง Django ใด ๆ โดยไม่ได้ activate venv เด็ดขาด เพราะจะทำให้
Python พยายามใช้ package จากระบบ global ซึ่งอาจไม่มี Django ติดตั้งอยู่เลย

### 31.2 คำสั่ง `django-admin startproject` คืออะไร

เมื่อคุณติดตั้ง Django ผ่าน `pip install django` แล้ว pip จะติดตั้งคำสั่ง (command-line
tool) ชื่อ `django-admin` มาให้ในตัว คำสั่งนี้มีหน้าที่หลักคือ **สร้างโครงสร้างไฟล์
เริ่มต้น** ของโปรเจกต์และแอปให้อัตโนมัติ แทนที่คุณจะต้องสร้างไฟล์เปล่า ๆ เองทีละไฟล์

รูปแบบคำสั่งพื้นฐาน:

```bash
django-admin startproject <ชื่อโปรเจกต์> [ปลายทาง]
```

- `<ชื่อโปรเจกต์>` คือชื่อของ **Python package** ที่จะเก็บไฟล์ตั้งค่าทั้งหมด (settings,
  URL หลัก, WSGI/ASGI) ชื่อนี้ต้องเป็นชื่อที่ถูกต้องตามกฎการตั้งชื่อ Python module
  (ห้ามมีขีดกลาง `-`, ห้ามขึ้นต้นด้วยตัวเลข, ห้ามตรงกับชื่อ built-in module เช่น `test`,
  `django`)
- `[ปลายทาง]` (optional) คือตำแหน่งที่จะสร้างไฟล์ลงไป ถ้าไม่ระบุ Django จะสร้างโฟลเดอร์
  ใหม่ชื่อเดียวกับโปรเจกต์ให้อัตโนมัติ

### 31.3 ลงมือสร้างโปรเจกต์จริง

เราจะใช้โฟลเดอร์ `django-mastery-course` ที่สร้างไว้ตั้งแต่ Part 001 (ที่มี venv อยู่แล้ว)
เป็นฐาน แล้วสร้างโปรเจกต์ Django ชื่อ **`config`** ลงไปตรง ๆ ในโฟลเดอร์นั้น (ไม่สร้าง
โฟลเดอร์ซ้อนโฟลเดอร์อีกชั้น) โดยใช้ `.` เป็นปลายทาง:

```bash
cd django-mastery-course
# ตรวจสอบให้แน่ใจว่า (venv) ยัง active อยู่

django-admin startproject config .
```

สังเกตจุด `.` ท้ายคำสั่ง — นี่คือการบอก Django ว่า **"สร้างไฟล์ตั้งค่าลงในโฟลเดอร์ปัจจุบัน
เลย ไม่ต้องสร้างโฟลเดอร์ครอบซ้อนอีกชั้น"** ถ้าคุณลืมใส่ `.` Django จะสร้างโฟลเดอร์ชื่อ
`config` ซ้อนเข้าไปอีกชั้นหนึ่ง (กลายเป็น `django-mastery-course/config/config/...`)
ซึ่งไม่ใช่โครงสร้างที่ต้องการและจะทำให้ `manage.py` อยู่ผิดตำแหน่ง

คำสั่งนี้ไม่มี output ใด ๆ แสดงออกมา (Django ทำงานเงียบ ๆ) แต่ถ้าสำเร็จ จะมีไฟล์และ
โฟลเดอร์ใหม่เกิดขึ้นทันที ลองตรวจสอบด้วย:

```bash
ls -la
```

ผลลัพธ์ที่ควรเห็น:

```
.
..
.git
.gitignore
config
manage.py
requirements.txt
venv
```

### 31.4 โครงสร้างโฟลเดอร์ที่ได้แบบละเอียด

ใช้คำสั่ง `tree` (หรือ `find . -not -path '*/venv/*' -not -path '*/.git/*'` ถ้าไม่มี
`tree`) เพื่อดูโครงสร้างทั้งหมด:

```bash
tree -I 'venv|.git'
```

```
.
├── .gitignore
├── config
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
└── requirements.txt
```

สังเกตว่า Django สร้างให้เราสองระดับที่ดูคล้ายกันแต่ทำหน้าที่ต่างกันโดยสิ้นเชิง:

| ระดับ | คือ | เปรียบเทียบ |
|---|---|---|
| `django-mastery-course/` (โฟลเดอร์นอกสุด) | รากของ repository ทั้งหมด ไม่ใช่ Python package | "บ้าน" ที่เก็บทุกอย่าง รวม venv, .git, requirements.txt |
| `config/` (โฟลเดอร์ใน) | Python package ที่เก็บไฟล์ตั้งค่าโปรเจกต์ (settings, URL หลัก) | "ห้องควบคุม" ของบ้าน |
| `manage.py` | สคริปต์ command-line สำหรับสั่งงาน Django | "รีโมทคอนโทรล" ของทั้งระบบ |

### 31.5 ทำไมแนะนำให้ตั้งชื่อโปรเจกต์ว่า `config` ไม่ใช่ชื่อเดียวกับโปรเจกต์

เอกสารทางการของ Django (tutorial) มักใช้ตัวอย่างเช่น `django-admin startproject mysite`
ซึ่งจะได้โฟลเดอร์ config ชื่อ `mysite` — มือใหม่จำนวนมากจึงตั้งชื่อ config package ตรงกับ
ชื่อโปรเจกต์หรือชื่อบริษัท เช่น `myshop`, `blogapp` ปัญหาคือ:

1. **สับสนกับชื่อแอป**: เมื่อคุณสร้างแอปจริงในภายหลัง (Part 005) เช่นแอปชื่อ `shop`
   ถ้า config package ก็ชื่อ `myshop` จะทำให้เกิดความสับสนว่าอันไหนคือแอป อันไหนคือ config
2. **สื่อความหมายไม่ชัดเจน**: โฟลเดอร์ `config/` สื่อชัดเจนทันทีว่า "นี่คือที่เก็บการตั้งค่า"
   ไม่ว่าใครมาเปิดโปรเจกต์นี้ครั้งแรกก็เข้าใจได้ทันที
3. **เป็นธรรมเนียมที่ทีมมืออาชีพจำนวนมากใช้**: โปรเจกต์ Django ระดับ production จำนวนมาก
   (รวมถึง cookiecutter-django ซึ่งเป็น template ยอดนิยมที่สุดของ Django) ใช้ชื่อ `config`
   เป็นมาตรฐาน

หลักสูตรนี้จะใช้ชื่อ **`config`** เป็น config package ตลอดทั้งหลักสูตร

### 31.6 ตัวเลือกอื่น ๆ ของคำสั่ง startproject

```bash
# ดู help ทั้งหมดของคำสั่ง
django-admin startproject --help

# สร้างโปรเจกต์โดยใช้ template กำหนดเอง (custom template) จาก URL หรือไฟล์ zip
django-admin startproject --template=/path/to/template myproject

# ระบุนามสกุลไฟล์ที่จะไม่ render เป็น template (ขั้นสูง ใช้น้อยมาก)
django-admin startproject --exclude=.git myproject
```

ในทางปฏิบัติ 99% ของการใช้งานจริงใช้แค่รูปแบบพื้นฐาน `django-admin startproject config .`
ก็เพียงพอแล้ว ตัวเลือกขั้นสูงเหล่านี้มีประโยชน์เมื่อองค์กรต้องการสร้าง project template
มาตรฐานของตัวเองให้ทีมใช้ร่วมกัน (เราจะกลับมาพูดถึงเรื่องนี้อีกครั้งใน Phase 12)

---

## ขั้นตอนที่ 32: เจาะลึกโครงสร้างไฟล์ที่ถูกสร้าง ทีละไฟล์

ตอนนี้เรามีไฟล์ 6 ไฟล์ที่ Django สร้างให้อัตโนมัติ มาเปิดดูทีละไฟล์ว่าแต่ละไฟล์ทำหน้าที่
อะไรกันแน่

### 32.1 ภาพรวมหน้าที่ของแต่ละไฟล์

| ไฟล์ | หน้าที่หลัก | ต้องแก้บ่อยแค่ไหน |
|---|---|---|
| `manage.py` | ประตูเข้าสู่คำสั่งจัดการโปรเจกต์ทั้งหมด | แทบไม่ต้องแก้เลย |
| `config/__init__.py` | บอก Python ว่า `config/` เป็น package | แทบไม่ต้องแก้เลย |
| `config/settings.py` | ศูนย์รวมการตั้งค่าทั้งโปรเจกต์ | แก้บ่อยมากตลอดการพัฒนา |
| `config/urls.py` | จุดเริ่มต้นของ URL routing ทั้งระบบ | แก้บ่อยเมื่อเพิ่มแอปใหม่ |
| `config/wsgi.py` | จุดเข้า (entry point) สำหรับ deploy แบบ synchronous | แทบไม่ต้องแก้เลย |
| `config/asgi.py` | จุดเข้า (entry point) สำหรับ deploy แบบ asynchronous | แก้เมื่อใช้ Channels/WebSocket |

### 32.2 `manage.py` — รีโมทคอนโทรลของโปรเจกต์

เปิดไฟล์ `manage.py` ด้วย VS Code (`code manage.py`) จะเห็นเนื้อหาประมาณนี้:

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    """Run administrative tasks."""
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == "__main__":
    main()
```

อธิบายทีละบรรทัดสำคัญ:

- `os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")` — บอก Django ว่า
  ไฟล์ตั้งค่าอยู่ที่ module `config.settings` (คือไฟล์ `config/settings.py`) นี่คือจุดที่
  เชื่อมโยง `manage.py` เข้ากับ config package ของเรา ถ้าเปลี่ยนชื่อโปรเจกต์จาก `config`
  เป็นชื่ออื่น ต้องแก้บรรทัดนี้ด้วย
- `execute_from_command_line(sys.argv)` — เป็นตัวรับ argument จาก command line (เช่น
  `runserver`, `migrate`) แล้วส่งต่อไปให้ Django ประมวลผลตามคำสั่งนั้น
- ข้อความ error ที่เตรียมไว้ (`ImportError`) แสดงให้เห็นว่า Django คาดการณ์ปัญหาที่พบบ่อย
  ที่สุดของมือใหม่ไว้แล้ว นั่นคือ **ลืม activate virtual environment**

ทุกคำสั่งที่คุณจะใช้ตลอดหลักสูตรนี้ล้วนเรียกผ่าน `manage.py` ทั้งสิ้น:

```bash
python manage.py <คำสั่ง> [options]
```

### 32.3 `config/__init__.py` — ไฟล์ที่ดูว่างเปล่าแต่สำคัญ

```python
# config/__init__.py
# (ไฟล์นี้ว่างเปล่าโดยค่าเริ่มต้น)
```

ไฟล์นี้ว่างเปล่า แต่การมีอยู่ของมันคือสิ่งที่ทำให้ Python รู้จักโฟลเดอร์ `config/` ว่าเป็น
**Python package** (ก่อน Python 3.3 ไฟล์นี้จำเป็นเสมอ ปัจจุบัน Python รองรับ "namespace
package" ที่ไม่ต้องมีไฟล์นี้ก็ได้ แต่ Django ยังคงสร้างให้เพื่อความชัดเจนและเข้ากันได้กับ
เครื่องมือเก่า) เพราะมีไฟล์นี้ เราจึงสามารถ `import config.settings` หรืออ้างอิงถึง
`config.urls`, `config.wsgi.application` ได้จากที่อื่นในโปรเจกต์

ในโปรเจกต์ขั้นสูงบางครั้งไฟล์นี้ถูกใช้เพื่อตั้งค่า Celery ตั้งแต่ตอน import แอป (เราจะ
เรียนเรื่องนี้ใน Phase 9)

### 32.4 `config/urls.py` — จุดเริ่มต้นของ URL routing

```python
"""
URL configuration for config project.

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/5.1/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path("admin/", admin.site.urls),
]
```

สิ่งที่น่าสังเกต:

- Django สร้าง **docstring ตัวอย่างการใช้งานจริง** ไว้ในไฟล์ให้เลย เพื่อช่วยมือใหม่ (ลอง
  อ่านดูให้ครบ มีประโยชน์มาก)
- `urlpatterns` คือ **list ของ URL pattern** ที่ Django จะไล่ตรวจสอบจากบนลงล่างเมื่อมี
  request เข้ามา ตัวไหนตรงกับ path ก่อน จะถูกใช้งานก่อน (ไม่ตรวจสอบต่อ)
- ค่าเริ่มต้นมีแค่ 1 บรรทัด คือ `admin/` ที่ผูกกับ Django Admin panel ที่มากับ Django
  ในตัวอัตโนมัติ (จะลองเข้าใช้งานจริงใน Phase 2)
- ไฟล์นี้เรียกว่า **root URLconf** เพราะเป็นจุดเริ่มต้นสุดของระบบ routing — ไฟล์นี้จะ
  `include()` URL ของแต่ละแอปเข้ามาเมื่อเราสร้างแอปใน Part 005-006

### 32.5 `config/wsgi.py` — จุดเข้าสำหรับ deploy แบบ WSGI

```python
"""
WSGI config for config project.

It exposes the WSGI callable as a module-level variable named ``application``.

For more information on this file, see
https://docs.djangoproject.com/en/5.1/howto/deployment/wsgi/
"""
import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")

application = get_wsgi_application()
```

ไฟล์นี้สร้างตัวแปรชื่อ `application` ที่เป็น **WSGI callable** — นี่คือ "จุดเชื่อมต่อ"
มาตรฐานที่ web server อย่าง Gunicorn ใช้เรียกเข้ามาเพื่อส่ง request เข้าสู่ Django (เรา
จะเจาะลึกเรื่อง WSGI ในขั้นตอนที่ 36)

### 32.6 `config/asgi.py` — จุดเข้าสำหรับ deploy แบบ ASGI

```python
"""
ASGI config for config project.

It exposes the ASGI callable as a module-level variable named ``application``.

For more information on this file, see
https://docs.djangoproject.com/en/5.1/howto/deployment/asgi/
"""
import os

from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")

application = get_asgi_application()
```

โครงสร้างแทบเหมือน `wsgi.py` ทุกประการ ต่างกันแค่ import `get_asgi_application` แทน
`get_wsgi_application` — Django ตั้งแต่เวอร์ชัน 3.0 เป็นต้นมารองรับทั้งสองแบบพร้อมกัน
เพื่อให้นักพัฒนาเลือกใช้ตามความต้องการของ deployment (รายละเอียดในขั้นตอนที่ 36)

### 32.7 `config/settings.py` — ไฟล์ที่สำคัญที่สุดในภาพรวมนี้

ไฟล์นี้มีเนื้อหายาวและสำคัญมากจนต้องแยกไปอธิบายเป็นขั้นตอนของตัวเองในขั้นตอนที่ 33
ตอนนี้ขอให้รู้แค่ว่า **นี่คือไฟล์ศูนย์กลางที่ควบคุมพฤติกรรมทั้งหมดของ Django project**
ตั้งแต่ฐานข้อมูล, แอปที่เปิดใช้งาน, ภาษา, timezone, ไปจนถึงเรื่อง security

---

## ขั้นตอนที่ 33: เจาะลึก settings.py

เปิดไฟล์ `config/settings.py` เต็ม ๆ ด้วย VS Code แล้วไล่ดูไปทีละส่วนตามลำดับที่ปรากฏ
ในไฟล์จริง

### 33.1 ส่วนหัวและ `BASE_DIR`

```python
"""
Django settings for config project.

Generated by 'django-admin startproject' using Django 5.1.2.

For more information on this file, see
https://docs.djangoproject.com/en/5.1/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/5.1/ref/settings/
"""

from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent
```

`BASE_DIR` คือ path แบบ absolute ที่ชี้ไปยังโฟลเดอร์รากของโปรเจกต์ (ระดับเดียวกับ
`manage.py`) คำนวณมาจาก `__file__` (path ของไฟล์ `settings.py` เอง) แล้วเดินย้อนขึ้นไป
2 ระดับ (`.parent.parent`):

```
config/settings.py
  └─ .parent      → config/
       └─ .parent → django-mastery-course/   (นี่คือ BASE_DIR)
```

ประโยชน์ของ `BASE_DIR` คือทำให้เราอ้างอิง path ต่าง ๆ ในโปรเจกต์แบบ **relative ที่
พกพาได้** ไม่ hardcode path เต็ม เช่น `BASE_DIR / "db.sqlite3"` จะทำงานถูกต้องไม่ว่า
โปรเจกต์นี้จะถูก clone ไปวางไว้ที่โฟลเดอร์ไหนของเครื่องใครก็ตาม `Path` เป็นคลาสจาก
module มาตรฐาน `pathlib` ของ Python (แนะนำให้ทบทวนใน Part 002 ถ้ายังไม่คุ้นเคย)

### 33.2 `SECRET_KEY`

```python
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = "django-insecure-a1b2c3d4e5f6g7h8i9j0-k1l2m3n4o5p6q7r8s9t0"
```

`SECRET_KEY` คือค่าสุ่มขนาดยาวที่ Django ใช้ในการเข้ารหัส (cryptographic signing) หลาย
จุดในระบบ เช่น:

- เซ็น session cookie เพื่อป้องกันการปลอมแปลง
- เซ็น CSRF token
- เซ็น password reset token
- เซ็นข้อมูลใน `django.core.signing`

**ข้อควรระวังระดับความปลอดภัยสูงสุด**: ค่านี้ถูกสร้างขึ้นแบบสุ่มตอน `startproject`
และมีคำนำหน้า `django-insecure-` เพื่อเตือนว่า **ห้ามใช้ค่านี้ใน production เด็ดขาด**
และที่สำคัญยิ่งกว่านั้นคือ **ห้าม commit ค่านี้ขึ้น Git แบบ hardcode** เราจะแก้ปัญหานี้
อย่างถูกต้องในขั้นตอนที่ 38 ด้วย `.env`

### 33.3 `DEBUG`

```python
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True
```

เมื่อ `DEBUG = True`:

- หากเกิด error Django จะแสดงหน้า **error page แบบละเอียดมาก** (traceback เต็ม,
  ตัวแปรทุกตัว, SQL query ที่รันไป) ซึ่งมีประโยชน์มากตอนพัฒนา
- ไฟล์ static บางส่วนถูก serve โดย Django เองโดยอัตโนมัติเพื่อความสะดวก
- คำเตือนต่าง ๆ (deprecation warnings) จะแสดงชัดเจนขึ้น

**อันตรายมากถ้าลืมปิดใน production**: หน้า error แบบละเอียดจะเผย path ของเซิร์ฟเวอร์,
ค่า settings, environment variables และแม้แต่บางส่วนของ SECRET_KEY ให้ผู้ไม่หวังดีเห็น
กฎเหล็กคือ **production ต้องตั้ง `DEBUG = False` เสมอ** (เราจะพูดเรื่องนี้ลึกอีกครั้งใน
Phase 10 เรื่อง Security)

### 33.4 `ALLOWED_HOSTS`

```python
ALLOWED_HOSTS = []
```

รายชื่อ hostname/domain ที่อนุญาตให้เข้าถึงแอปนี้ได้ เป็นการป้องกันการโจมตีแบบ
**HTTP Host header injection** เมื่อ `DEBUG = True` และ `ALLOWED_HOSTS = []` (ค่าว่าง)
Django จะอนุญาตให้เข้าผ่าน `localhost` และ `127.0.0.1` โดยอัตโนมัติเพื่อความสะดวกตอน
พัฒนา แต่เมื่อ `DEBUG = False` การปล่อย `ALLOWED_HOSTS` ว่างจะทำให้ **ทุก request ถูก
ปฏิเสธด้วย error 400 Bad Request ทันที**

ตัวอย่างการตั้งค่าสำหรับ production:

```python
ALLOWED_HOSTS = ["example.com", "www.example.com", "api.example.com"]
```

### 33.5 `INSTALLED_APPS`

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]
```

นี่คือ **รายชื่อแอปทั้งหมด (built-in และของเราเอง) ที่ Django "รู้จัก" และเปิดใช้งาน**
ถ้าแอปไม่อยู่ใน list นี้ Django จะไม่โหลด model, template tags, static files, หรือ
management command ของแอปนั้นเลย นี่คือจุดที่เราจะต้องเพิ่มชื่อแอปของเราเองเข้าไปทุกครั้ง
ที่สร้างแอปใหม่ (Part 005)

ความหมายของแอป built-in ที่มาให้ตั้งแต่แรก:

| แอป | หน้าที่ |
|---|---|
| `django.contrib.admin` | ระบบ Admin panel อัตโนมัติสำหรับจัดการข้อมูล |
| `django.contrib.auth` | ระบบ Authentication (User, Permission, Group) |
| `django.contrib.contenttypes` | ระบบติดตามชนิดของ Model แบบ generic (ใช้เบื้องหลัง Admin และ Permission) |
| `django.contrib.sessions` | ระบบจัดการ Session ของผู้ใช้ (เก็บสถานะข้าม request) |
| `django.contrib.messages` | ระบบข้อความแจ้งเตือนชั่วคราว (flash messages) เช่น "บันทึกสำเร็จ" |
| `django.contrib.staticfiles` | ระบบจัดการไฟล์ static (CSS, JS, รูปภาพ) |

ลำดับใน list มีผลบ้างในบางกรณี (เช่น template override, app loading order) แต่โดยทั่วไป
ไม่ต้องกังวลมากในระดับเริ่มต้น

### 33.6 `MIDDLEWARE`

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

**Middleware** คือชั้นการประมวลผลที่ request/response ทุกตัวต้องเดินผ่าน เปรียบเสมือน
"ด่านตรวจ" ที่เรียงต่อกันเป็นชั้น ๆ **ลำดับใน list นี้สำคัญมาก** เพราะ:

- ตอน request ขาเข้า: Django จะไล่ประมวลผล middleware **จากบนลงล่าง**
- ตอน response ขาออก: Django จะไล่ประมวลผลย้อนกลับ **จากล่างขึ้นบน**

```
Request  ──▶ SecurityMiddleware ──▶ SessionMiddleware ──▶ ... ──▶ View
                                                                    │
Response ◀── SecurityMiddleware ◀── SessionMiddleware ◀── ... ◀────┘
```

หน้าที่ของ middleware แต่ละตัว:

| Middleware | หน้าที่ |
|---|---|
| `SecurityMiddleware` | ตั้งค่า security header ต่าง ๆ (HSTS, XSS protection, redirect HTTPS) |
| `SessionMiddleware` | เปิดใช้งาน `request.session` ให้ view เข้าถึงได้ |
| `CommonMiddleware` | จัดการเรื่องทั่วไป เช่น URL normalization, `APPEND_SLASH` |
| `CsrfViewMiddleware` | ป้องกัน CSRF attack โดยตรวจสอบ token ในทุก POST request |
| `AuthenticationMiddleware` | เปิดใช้งาน `request.user` ให้ view เข้าถึงได้ |
| `MessageMiddleware` | เปิดใช้งานระบบ flash message (`django.contrib.messages`) |
| `XFrameOptionsMiddleware` | ป้องกัน Clickjacking โดยตั้งค่า header `X-Frame-Options` |

เราจะเขียน custom middleware ของตัวเองใน Phase 8 และเจาะลึกเรื่อง security middleware
อีกครั้งใน Phase 10

### 33.7 `ROOT_URLCONF` และ `WSGI_APPLICATION`

```python
ROOT_URLCONF = "config.urls"

# ...

WSGI_APPLICATION = "config.wsgi.application"
```

- `ROOT_URLCONF` บอก Django ว่าไฟล์ URL หลักของโปรเจกต์อยู่ที่ไหน (module `config.urls`
  คือไฟล์ `config/urls.py`) ทุก URL ที่เข้ามาจะเริ่มค้นหาจากไฟล์นี้เสมอ
- `WSGI_APPLICATION` ชี้ไปที่ตัวแปร `application` ในไฟล์ `config/wsgi.py` ใช้ตอน deploy
  ด้วย WSGI server (Gunicorn เป็นต้น)

### 33.8 `TEMPLATES`

```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

- `BACKEND` เลือกเอนจินสำหรับ render template — ค่าเริ่มต้นคือ Django Template Language
  (DTL) เอง (ทางเลือกอื่นคือ Jinja2)
- `DIRS` คือ list ของโฟลเดอร์เพิ่มเติมที่ Django จะค้นหา template นอกเหนือจากในแต่ละแอป
  (มักตั้งเป็น `[BASE_DIR / "templates"]` เพื่อเก็บ template ระดับโปรเจกต์ เราจะทำจริง
  ใน Part 008)
- `APP_DIRS = True` บอกให้ Django ค้นหา template ในโฟลเดอร์ `templates/` ของแต่ละแอป
  ที่อยู่ใน `INSTALLED_APPS` โดยอัตโนมัติ
- `context_processors` คือฟังก์ชันที่ทำงานทุกครั้งก่อน render template เพื่อฉีดตัวแปร
  ที่ใช้ร่วมกันเข้าไปใน context โดยอัตโนมัติ เช่น `request.user` (จาก `auth`) หรือ
  ข้อความแจ้งเตือน (จาก `messages`)

### 33.9 `DATABASES`

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

- คีย์ `"default"` คือชื่อของการเชื่อมต่อฐานข้อมูลหลัก (Django รองรับหลายฐานข้อมูล
  พร้อมกันได้ เราจะเรียนเรื่อง multiple databases ใน Phase 2)
- `ENGINE` บอกชนิดของฐานข้อมูล — ค่าเริ่มต้นคือ **SQLite** ซึ่งเป็นไฟล์ฐานข้อมูลเดี่ยว
  ไม่ต้องติดตั้งเซิร์ฟเวอร์แยก เหมาะสำหรับพัฒนาและเรียนรู้เบื้องต้น
- `NAME` คือ path ของไฟล์ฐานข้อมูล (สำหรับ SQLite) หรือชื่อฐานข้อมูล (สำหรับ
  PostgreSQL/MySQL)

ตัวอย่างการตั้งค่าสำหรับ PostgreSQL (ที่เราจะเปลี่ยนไปใช้ตั้งแต่ Phase 2):

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "myuser",
        "PASSWORD": "mypassword",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

### 33.10 `AUTH_PASSWORD_VALIDATORS`

```python
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]
```

รายการ validator ที่ตรวจสอบความปลอดภัยของรหัสผ่านตอนผู้ใช้ตั้งรหัสผ่าน (ผ่าน
`createsuperuser`, form สมัครสมาชิก ฯลฯ):

| Validator | ตรวจสอบอะไร |
|---|---|
| `UserAttributeSimilarityValidator` | รหัสผ่านต้องไม่คล้ายกับ username, email, ชื่อ-นามสกุลมากเกินไป |
| `MinimumLengthValidator` | ความยาวรหัสผ่านขั้นต่ำ (ค่าเริ่มต้น 8 ตัวอักษร) |
| `CommonPasswordValidator` | ห้ามใช้รหัสผ่านที่พบบ่อย (เทียบกับ list รหัสผ่านยอดฮิต 20,000 คำ) |
| `NumericPasswordValidator` | ห้ามใช้รหัสผ่านที่เป็นตัวเลขล้วน |

### 33.11 `LANGUAGE_CODE`, `TIME_ZONE`, `USE_I18N`, `USE_TZ`

```python
LANGUAGE_CODE = "en-us"

TIME_ZONE = "UTC"

USE_I18N = True

USE_TZ = True
```

- `LANGUAGE_CODE` กำหนดภาษาเริ่มต้นของระบบ (มีผลกับข้อความ error, การจัดรูปแบบวันที่
  ฯลฯ) สำหรับโปรเจกต์ที่ให้บริการผู้ใช้ไทย ควรเปลี่ยนเป็น `"th"`
- `TIME_ZONE` กำหนด timezone ของระบบ ค่าเริ่มต้นคือ `UTC` ซึ่งเป็นแนวปฏิบัติที่ดีที่สุด
  สำหรับเก็บข้อมูลในฐานข้อมูล (แล้วค่อยแปลงเป็น timezone ท้องถิ่นตอนแสดงผล) สำหรับ
  โปรเจกต์ในไทยอาจตั้งเป็น `"Asia/Bangkok"` แต่คำแนะนำระดับมืออาชีพคือ **เก็บใน UTC
  เสมอ แล้วแปลงตอนแสดงผลในฝั่ง frontend**
- `USE_I18N = True` เปิดใช้งานระบบแปลภาษา (internationalization)
- `USE_TZ = True` บอก Django ให้เก็บค่า datetime แบบ **timezone-aware** ในฐานข้อมูล
  (แนะนำให้เปิดไว้เสมอ เพื่อหลีกเลี่ยงบั๊กเรื่องเวลาที่พบบ่อยมากในการพัฒนาจริง)

### 33.12 `STATIC_URL` และ `DEFAULT_AUTO_FIELD`

```python
STATIC_URL = "static/"

# Default primary key field type
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

- `STATIC_URL` คือ URL prefix ที่ใช้อ้างอิงไฟล์ static (CSS, JS, รูปภาพ) ในบราวเซอร์
  เราจะเจาะลึกระบบ static files ทั้งหมดใน Part 009
- `DEFAULT_AUTO_FIELD` กำหนดชนิดของ field ที่ใช้เป็น primary key อัตโนมัติเมื่อสร้าง
  Model โดยไม่ได้ระบุ primary key เอง `BigAutoField` คือเลขจำนวนเต็มขนาด 64-bit
  (รองรับแถวข้อมูลได้มากถึง ~9.2 ล้านล้านแถว) เป็นค่าเริ่มต้นตั้งแต่ Django 3.2 เป็นต้นมา
  (ก่อนหน้านั้นใช้ `AutoField` แบบ 32-bit ซึ่งรองรับได้แค่ ~2.1 พันล้านแถว)

### 33.13 สรุปภาพรวม settings.py ในตารางเดียว

| Setting | ค่าเริ่มต้น | มีผลต่ออะไร |
|---|---|---|
| `SECRET_KEY` | สุ่มมาให้ | การเข้ารหัส session, CSRF, signing |
| `DEBUG` | `True` | โหมด error page แบบละเอียด |
| `ALLOWED_HOSTS` | `[]` | Hostname ที่อนุญาตให้เข้าถึง |
| `INSTALLED_APPS` | 6 แอป built-in | แอปที่ Django โหลดใช้งาน |
| `MIDDLEWARE` | 7 middleware built-in | ชั้นประมวลผล request/response |
| `ROOT_URLCONF` | `config.urls` | ไฟล์ URL หลัก |
| `TEMPLATES` | DjangoTemplates + APP_DIRS | การค้นหาและ render template |
| `DATABASES` | SQLite | การเชื่อมต่อฐานข้อมูล |
| `TIME_ZONE` | `UTC` | Timezone ของระบบ |
| `STATIC_URL` | `static/` | URL prefix ของไฟล์ static |

---

## ขั้นตอนที่ 34: รัน development server และทำความเข้าใจ output

### 34.1 คำสั่งพื้นฐาน

```bash
python manage.py runserver
```

ผลลัพธ์ที่คาดว่าจะเห็น:

```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not be fully functional
until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to see all migrations.
January 15, 2026 - 09:30:12
Django version 5.1.2, using settings 'config.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

ลองเปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/` — คุณจะเห็นหน้า **"The install worked
successfully! Congratulations!"** พร้อมจรวด (rocket) สีเขียว นี่คือหน้าต้อนรับ
เริ่มต้นของ Django ที่แสดงเมื่อยังไม่มี URL pattern อื่นนอกจาก `admin/`

### 34.2 อธิบาย output ทีละบรรทัด

- **`Watching for file changes with StatReloader`** — Django เปิดใช้งาน **auto-reload**
  โดยอัตโนมัติ หมายความว่าทุกครั้งที่คุณแก้ไขและ save ไฟล์ `.py` เซิร์ฟเวอร์จะรีสตาร์ท
  ตัวเองอัตโนมัติโดยไม่ต้องกด Ctrl+C แล้วรันใหม่เอง (สะดวกมากตอนพัฒนา)
- **`Performing system checks...` / `System check identified no issues`** — Django รัน
  ชุดตรวจสอบความถูกต้องของโปรเจกต์อัตโนมัติ (เหมือนคำสั่ง `check` ในขั้นตอนที่ 35)
  ก่อนเริ่มเซิร์ฟเวอร์ทุกครั้ง
- **`You have 18 unapplied migration(s)...`** — นี่คือ **คำเตือน ไม่ใช่ error**
  Django บอกว่ามี migration (คำสั่งสร้างตารางฐานข้อมูล) ของแอป built-in (`admin`,
  `auth`, `contenttypes`, `sessions`) ที่ยังไม่ถูกนำไปสร้างตารางจริงในฐานข้อมูล
  เซิร์ฟเวอร์ยังรันได้ปกติ แต่ฟีเจอร์ที่ต้องพึ่งฐานข้อมูล (เช่น Admin panel, login)
  จะยังใช้งานไม่ได้จนกว่าจะรัน `python manage.py migrate` (เราจะแก้ในขั้นตอนถัดไป)
- **`Django version 5.1.2, using settings 'config.settings'`** — ยืนยันเวอร์ชัน Django
  และไฟล์ settings ที่กำลังใช้งานอยู่ (มีประโยชน์มากตอน debug ว่าเผลอใช้ settings ผิดไฟล์
  หรือไม่ ในโปรเจกต์ใหญ่ที่มีหลาย settings file)
- **`Starting development server at http://127.0.0.1:8000/`** — บอก URL และพอร์ตที่
  เซิร์ฟเวอร์กำลังรันอยู่

### 34.3 กำจัดคำเตือน unapplied migrations

```bash
python manage.py migrate
```

ผลลัพธ์:

```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  ... (และอีกหลายบรรทัด)
  Applying sessions.0001_initial... OK
```

หลังจากนี้ ไฟล์ `db.sqlite3` จะถูกสร้างขึ้นในโฟลเดอร์โปรเจกต์ และคำเตือน "unapplied
migrations" จะหายไปเมื่อรัน `runserver` ครั้งถัดไป (รายละเอียดเรื่อง migration แบบเต็ม
จะเรียนใน Part 011)

### 34.4 การตั้งค่าพอร์ตและ host อื่น ๆ

```bash
# รันที่พอร์ตอื่น
python manage.py runserver 8080

# รันให้เข้าถึงได้จากเครื่องอื่นในเครือข่ายเดียวกัน (0.0.0.0 = ทุก interface)
python manage.py runserver 0.0.0.0:8000

# รันที่ IP และพอร์ตเจาะจง
python manage.py runserver 192.168.1.10:8000
```

### 34.5 ข้อควรรู้: runserver ไม่ใช่สำหรับ production

Django เตือนเรื่องนี้อย่างชัดเจนในเอกสารทางการ: **`runserver` เป็นเซิร์ฟเวอร์ที่ออกแบบ
มาเพื่อการพัฒนาเท่านั้น** มันไม่ได้ผ่านการตรวจสอบด้าน security และ performance สำหรับ
โหลดจริงระดับ production เมื่อถึงเวลา deploy จริง เราจะใช้ WSGI/ASGI server อย่าง
Gunicorn, uWSGI, หรือ Uvicorn แทน (รายละเอียดในขั้นตอนที่ 36 และเจาะลึกเต็มรูปแบบใน
Phase 11)

---

## ขั้นตอนที่ 35: ภาพรวมคำสั่ง manage.py ที่สำคัญ

`manage.py` มีคำสั่งย่อยมากมาย ลองดูรายการทั้งหมดด้วย:

```bash
python manage.py help
```

ในขั้นตอนนี้เราจะโฟกัสที่คำสั่งที่ใช้บ่อยที่สุดตลอดหลักสูตร

### 35.1 `migrate` — นำ migration ไปสร้าง/แก้ไขตารางฐานข้อมูลจริง

```bash
python manage.py migrate
```

รับ migration file ที่ถูกสร้างไว้ (จากคำสั่ง `makemigrations` ซึ่งเราจะเรียนใน Part 011)
แล้วแปลงเป็นคำสั่ง SQL จริงเพื่อสร้าง/แก้ไขตารางในฐานข้อมูล เปรียบเสมือน "ระบบควบคุม
เวอร์ชันของโครงสร้างฐานข้อมูล" (คล้าย Git แต่สำหรับ schema)

### 35.2 `createsuperuser` — สร้างบัญชีผู้ดูแลระบบ

```bash
python manage.py createsuperuser
```

ระบบจะถามข้อมูลแบบ interactive:

```
Username: admin
Email address: admin@example.com
Password:
Password (again):
Superuser created successfully.
```

บัญชีนี้คือผู้ใช้ที่มีสิทธิ์สูงสุด (`is_superuser = True`) สามารถเข้า Django Admin
panel ที่ `http://127.0.0.1:8000/admin/` และจัดการข้อมูลทุกอย่างในระบบได้ (เจาะลึกเต็ม
รูปแบบใน Phase 2)

### 35.3 `shell` — เปิด Python interactive shell พร้อมโหลด Django

```bash
python manage.py shell
```

```python
>>> from django.conf import settings
>>> settings.DEBUG
True
>>> settings.DATABASES['default']['ENGINE']
'django.db.backends.sqlite3'
>>> exit()
```

คำสั่งนี้เปิด Python shell ปกติ แต่ **โหลด Django settings และ apps ให้พร้อมใช้งาน
อัตโนมัติ** ต่างจากการเปิด `python` เฉย ๆ ที่จะยังไม่รู้จัก Django project ของเราเลย
มีประโยชน์มากสำหรับทดสอบ query ฐานข้อมูลแบบเร็ว ๆ โดยไม่ต้องเขียน view (เราจะใช้บ่อย
มากตั้งแต่ Part 011 เป็นต้นไป)

### 35.4 `check` — ตรวจสอบความถูกต้องของโปรเจกต์

```bash
python manage.py check
```

```
System check identified no issues (0 silenced).
```

ตรวจสอบข้อผิดพลาดเชิงโครงสร้างของโปรเจกต์ (เช่น Model ผิด, การตั้งค่า settings ที่
ขัดแย้งกัน) โดย **ไม่ต้องรันเซิร์ฟเวอร์จริง** เหมาะมากสำหรับใช้ใน CI/CD pipeline
(Part 088) เพื่อดักจับปัญหาก่อน deploy

มีตัวเลือก `--deploy` ที่ตรวจสอบความพร้อมด้าน security สำหรับ production โดยเฉพาะ:

```bash
python manage.py check --deploy
```

### 35.5 `dbshell` — เข้าสู่ command-line ของฐานข้อมูลโดยตรง

```bash
python manage.py dbshell
```

เปิด shell ของฐานข้อมูลที่กำหนดใน `DATABASES` โดยตรง (สำหรับ SQLite จะเปิด
`sqlite3` shell, สำหรับ PostgreSQL จะเปิด `psql`) โดยไม่ต้องพิมพ์ connection
string เอง เหมาะสำหรับ debug ด้วย SQL ดิบ ๆ เมื่อจำเป็น

```sql
sqlite> .tables
django_admin_log      auth_permission        django_session
auth_group             auth_user              django_content_type
auth_group_permissions auth_user_groups
auth_permission        auth_user_user_permissions

sqlite> .quit
```

### 35.6 คำสั่งอื่น ๆ ที่จะได้ใช้ในภายหลัง

| คำสั่ง | หน้าที่ | จะเรียนละเอียดใน |
|---|---|---|
| `startapp` | สร้างแอปใหม่ในโปรเจกต์ | Part 005 |
| `makemigrations` | สร้างไฟล์ migration จากการเปลี่ยนแปลง Model | Part 011 |
| `showmigrations` | แสดงสถานะ migration ทั้งหมด | Part 011 |
| `collectstatic` | รวมไฟล์ static ทั้งหมดไว้ที่เดียวสำหรับ production | Part 009 |
| `test` | รัน automated test ทั้งหมด | Phase 7 |
| `dumpdata` / `loaddata` | export/import ข้อมูลเป็นไฟล์ (fixture) | Part 011 |
| `changepassword` | เปลี่ยนรหัสผ่านของ user ผ่าน command line | Phase 4 |

### 35.7 ดูรายละเอียดของแต่ละคำสั่ง

```bash
python manage.py help migrate
python manage.py help runserver
```

คำสั่ง `help <ชื่อคำสั่ง>` จะแสดง options ทั้งหมดของคำสั่งนั้น ๆ อย่างละเอียด เป็นนิสัย
ที่ดีที่ควรใช้บ่อย ๆ แทนการเดา flag เอง

---

## ขั้นตอนที่ 36: WSGI vs ASGI คืออะไร ต่างกันอย่างไร

### 36.1 ทำไมต้องมีมาตรฐานเหล่านี้

Django (และ Python framework อื่น ๆ) ไม่ได้ทำหน้าที่เป็น web server ที่รับ connection
จากอินเทอร์เน็ตโดยตรงใน production เสมอไป แต่ทำงานอยู่ **หลัง** web server ตัวจริง
(เช่น Nginx) และมี **application server** เป็นตัวกลางที่คุยกับ Django ผ่านมาตรฐานที่
ตกลงกันไว้ — นี่คือที่มาของ WSGI และ ASGI

```
Internet ──▶ Nginx (reverse proxy) ──▶ Gunicorn/Uvicorn (application server) ──▶ Django
                                              │
                                    คุยกันผ่านมาตรฐาน WSGI หรือ ASGI
```

### 36.2 WSGI (Web Server Gateway Interface)

**WSGI** เป็นมาตรฐานเก่าแก่ที่สุด (PEP 3333, มีมาตั้งแต่ปี 2003) กำหนดวิธีที่ web server
กับ Python web application คุยกันแบบ **synchronous** (ประมวลผลทีละ request เรียงกัน
ไม่สามารถทำหลายอย่างพร้อมกันในระดับ I/O ได้ในตัว)

```python
# config/wsgi.py — จุดเข้ามาตรฐานของ WSGI
application = get_wsgi_application()
```

### 36.3 ASGI (Asynchronous Server Gateway Interface)

**ASGI** คือมาตรฐานรุ่นใหม่กว่า ออกแบบมาเพื่อรองรับการทำงานแบบ **asynchronous**
(async/await ของ Python) ทำให้ Django สามารถ:

- จัดการหลาย request พร้อมกันได้อย่างมีประสิทธิภาพเมื่อ request นั้นต้องรอ I/O
  (เช่น รอ network, รอฐานข้อมูล) โดยไม่บล็อกกันเอง
- รองรับ **WebSocket** สำหรับการสื่อสารแบบ real-time สองทาง (เช่น แชท, notification
  สด ๆ) ซึ่ง WSGI ทำไม่ได้เลยเพราะออกแบบมาสำหรับ HTTP request-response แบบเดียวเท่านั้น
- รองรับ **async view** (`async def view(request): ...`) ที่เขียนตั้งแต่ Django 3.1
  เป็นต้นมา (เจาะลึกใน Phase 9)

```python
# config/asgi.py — จุดเข้ามาตรฐานของ ASGI
application = get_asgi_application()
```

### 36.4 ตารางเปรียบเทียบ

| หัวข้อ | WSGI | ASGI |
|---|---|---|
| ปีที่เกิด | 2003 (PEP 3333) | 2016 (โดยทีม Django Channels) |
| รูปแบบการทำงาน | Synchronous เท่านั้น | รองรับทั้ง Synchronous และ Asynchronous |
| รองรับ WebSocket | ❌ ไม่รองรับ | ✅ รองรับ |
| Application server ยอดนิยม | Gunicorn, uWSGI | Uvicorn, Daphne, Hypercorn |
| เหมาะกับ | เว็บแอปทั่วไป, REST API แบบมาตรฐาน | แอปที่ต้องการ real-time, WebSocket, high-concurrency I/O |
| ใช้กับ Django Channels ได้หรือไม่ | ❌ ไม่ได้ | ✅ ได้ |
| ความซับซ้อนในการ deploy | ต่ำกว่า เป็นมาตรฐานมานาน | สูงกว่าเล็กน้อย เครื่องมือใหม่กว่า |

### 36.5 ตัวอย่างคำสั่ง deploy จริงทั้งสองแบบ

```bash
# Deploy แบบ WSGI ด้วย Gunicorn (มาตรฐานที่ใช้กันมากที่สุด)
gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4

# Deploy แบบ ASGI ด้วย Uvicorn (เมื่อต้องการ async หรือ WebSocket)
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 4

# Deploy แบบ ASGI ด้วย Daphne (ทีมพัฒนา Django Channels แนะนำ)
daphne -b 0.0.0.0 -p 8000 config.asgi:application
```

### 36.6 คำแนะนำสำหรับหลักสูตรนี้

ตลอดหลักสูตรนี้จนถึง Phase 9 เราจะใช้ **WSGI เป็นหลัก** เพราะโปรเจกต์ส่วนใหญ่ในช่วงแรก
เป็นเว็บแอปแบบ request-response ปกติที่ไม่ต้องการ WebSocket เมื่อถึง Phase 9 (Async,
Celery และ Channels) เราจะย้ายไปใช้ ASGI เต็มรูปแบบเพื่อรองรับ async view และ Django
Channels สำหรับฟีเจอร์ real-time เช่น แชทและระบบแจ้งเตือนสด

---

## ขั้นตอนที่ 37: แนวคิด Project vs App ใน Django

### 37.1 ทำไม Django ถึงแยกสองสิ่งนี้

หนึ่งในสิ่งที่ทำให้มือใหม่สับสนที่สุดตอนเริ่มเรียน Django คือความแตกต่างระหว่าง
**Project** กับ **App** ทั้งที่หน้าตาโครงสร้างไฟล์คล้ายกันมาก มาทำความเข้าใจให้ชัดเจน

| | Project | App |
|---|---|---|
| นิยาม | คอนเทนเนอร์ระดับบนสุดที่รวมการตั้งค่าและแอปทั้งหมดเข้าด้วยกัน | หน่วยฟีเจอร์เล็ก ๆ ที่ทำงานเฉพาะเรื่องใดเรื่องหนึ่ง |
| จำนวนต่อระบบ | มีได้ **แค่ 1 project** ต่อระบบที่รันจริง | มีได้ **หลาย app** ในหนึ่ง project |
| ตัวอย่างชื่อ | `config` | `blog`, `shop`, `accounts`, `payments` |
| นำกลับมาใช้ซ้ำได้ไหม | ❌ ทำไม่ได้ (เฉพาะเจาะจงกับระบบนั้น) | ✅ ได้ ถ้าออกแบบดี สามารถย้ายไปใช้ในโปรเจกต์อื่นได้ |
| มี `settings.py` ไหม | ✅ มี (แค่ที่เดียวในระบบ) | ❌ ไม่มี (ใช้ settings ของ project ร่วมกัน) |
| มี `models.py`, `views.py` ไหม | ❌ ไม่ควรมี (เป็นแค่ config) | ✅ มี (นี่คือที่เก็บ logic จริงของฟีเจอร์) |

### 37.2 เปรียบเทียบด้วยภาพจริง

ลองนึกภาพระบบ e-commerce ขนาดกลาง โครงสร้างจะหน้าตาประมาณนี้:

```
config/                  ← Project: ศูนย์กลางการตั้งค่า มีแค่ 1 ชุด
    settings.py
    urls.py
    wsgi.py

accounts/                ← App: จัดการผู้ใช้และการล็อกอิน
    models.py
    views.py

products/                ← App: จัดการสินค้าและหมวดหมู่
    models.py
    views.py

orders/                  ← App: จัดการตะกร้าและคำสั่งซื้อ
    models.py
    views.py

payments/                ← App: จัดการการชำระเงิน
    models.py
    views.py

reviews/                 ← App: จัดการรีวิวสินค้า
    models.py
    views.py
```

สังเกตว่า **`config/` มีแค่ชุดเดียว** แต่มีแอปหลายตัวที่แยกความรับผิดชอบตาม "โดเมนทาง
ธุรกิจ" (business domain) แต่ละแอปทำงานเรื่องเดียวให้ดีที่สุด (หลักการ **Single
Responsibility** ในระดับโครงสร้างโปรเจกต์)

### 37.3 ทำไมการแยกแบบนี้ถึงมีประโยชน์

1. **นำกลับมาใช้ซ้ำได้ (Reusability)**: แอปที่ออกแบบดีสามารถ copy ไปใช้ในโปรเจกต์อื่น
   ได้เกือบทันที เช่นแอป `reviews` ระบบรีวิวสินค้า อาจนำไปใช้ในเว็บอื่นที่ไม่เกี่ยวกับ
   e-commerce เลยก็ได้ ถ้าออกแบบให้ไม่ผูกกับแอปอื่นมากเกินไป — นี่คือที่มาของ package
   สำเร็จรูปบน PyPI มากมาย เช่น `django-allauth`, `django-crispy-forms` ที่แท้จริงแล้ว
   ก็คือ "แอป Django" ที่คนอื่นเขียนไว้แล้วแชร์ให้ใช้
2. **แยกความรับผิดชอบชัดเจน (Separation of Concerns)**: เมื่อทีมงานขยายใหญ่ขึ้น
   แต่ละคน/ทีมย่อยสามารถรับผิดชอบแอปของตัวเองได้โดยไม่ชนกับคนอื่น
3. **ทดสอบง่ายขึ้น (Testability)**: เขียน test แยกตามแอปได้ ทำให้ debug และดูแลรักษา
   ง่ายกว่าเขียนทุกอย่างไว้ในไฟล์เดียว
4. **จัดการ dependency ชัดเจน**: แอปหนึ่งอาจต้องพึ่งพาแอปอื่น (เช่น `orders` ต้องใช้
   ข้อมูลจาก `products`) แต่การแยกไฟล์ทำให้เห็นความสัมพันธ์นี้ชัดเจนกว่าเขียนปนกัน

### 37.4 กฎง่าย ๆ ในการตัดสินใจว่าจะสร้างแอปใหม่เมื่อไร

- ถ้าฟีเจอร์นั้นสามารถอธิบายได้ด้วยคำนามเดียว (เช่น "สินค้า", "คำสั่งซื้อ", "บทความ")
  และมี Model ของตัวเองชัดเจน → ควรแยกเป็นแอปใหม่
- ถ้าฟีเจอร์นั้นเล็กมากและผูกติดกับแอปอื่นแนบแน่น (เช่น field เพิ่มเติมของ Model
  ที่มีอยู่แล้ว) → อาจไม่จำเป็นต้องแยกแอปใหม่
- เราจะฝึกสร้างแอปจริงและตัดสินใจเรื่องนี้อย่างละเอียดใน **Part 005** ที่กำลังจะถึง

---

## ขั้นตอนที่ 38: จัดการ config/secret ด้วยไฟล์ .env และ django-environ

### 38.1 ปัญหาของการ hardcode ค่า config ใน settings.py

ค่าเริ่มต้นที่ `startproject` สร้างให้มีปัญหาสำคัญ 2 ข้อ:

1. **`SECRET_KEY` ถูก hardcode อยู่ในโค้ด** — ถ้า push ขึ้น Git (แม้เป็น private repo)
   ใครก็ตามที่เข้าถึง repo ได้จะเห็นค่านี้ทันที และถ้าค่านี้รั่วไหลใน production
   ผู้ไม่หวังดีสามารถปลอมแปลง session, CSRF token ได้ทันที
2. **ค่าที่ต่างกันระหว่าง environment (dev/staging/production) ถูกเขียนปนกันในไฟล์
   เดียว** — เช่น `DEBUG = True` ตอนพัฒนา แต่ต้องเป็น `False` ตอน production ถ้า
   hardcode ไว้ค่าเดียว จะต้องแก้โค้ดและ commit ใหม่ทุกครั้งที่ deploy คนละ environment
   ซึ่งเสี่ยงมากและไม่เป็นมืออาชีพ

**หลักการสำคัญที่สุดของการจัดการ config มืออาชีพ (แนวคิดจาก
[The Twelve-Factor App](https://12factor.net/config))**: **"เก็บ config ไว้ใน
environment ไม่ใช่ในโค้ด"**

### 38.2 ติดตั้ง django-environ

```bash
pip install django-environ
pip freeze > requirements.txt
```

`django-environ` เป็น library ยอดนิยมที่ช่วยอ่านค่าจากไฟล์ `.env` และแปลงชนิดข้อมูล
ให้อัตโนมัติ (string → bool, string → list เป็นต้น)

### 38.3 สร้างไฟล์ `.env`

```bash
touch .env
code .env
```

ใส่เนื้อหาต่อไปนี้ในไฟล์ `.env`:

```
DEBUG=True
SECRET_KEY=django-insecure-a1b2c3d4e5f6g7h8i9j0-k1l2m3n4o5p6q7r8s9t0
ALLOWED_HOSTS=127.0.0.1,localhost
```

### 38.4 สร้าง SECRET_KEY ใหม่ที่ปลอดภัยจริง

อย่าใช้ค่าที่ Django สุ่มให้ตอน `startproject` ต่อไปเรื่อย ๆ ควรสร้างค่าใหม่ (โดยเฉพาะ
ก่อนขึ้น production) ด้วยคำสั่ง:

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

ตัวอย่างผลลัพธ์:

```
_x7$k2p9@qw3e!r5t6y7u8i9o0p1a2s3d4f5g6h7j8k9l0z1x2c3v4b
```

คัดลอกค่านี้ไปแทนที่ค่า `SECRET_KEY` ในไฟล์ `.env`

### 38.5 แก้ไข `config/settings.py` ให้อ่านค่าจาก .env

เปิดไฟล์ `config/settings.py` แล้วแก้ไขส่วนหัวและค่าที่เกี่ยวข้องดังนี้:

```python
from pathlib import Path

import environ

# ---- ตั้งค่า BASE_DIR และ django-environ ----
BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env(
    DEBUG=(bool, False),  # ค่า default ถ้าไม่พบใน .env คือ False (ปลอดภัยไว้ก่อน)
)

# อ่านไฟล์ .env จากโฟลเดอร์รากของโปรเจกต์
environ.Env.read_env(BASE_DIR / ".env")

# ---- ใช้ค่าจาก .env แทนการ hardcode ----
SECRET_KEY = env("SECRET_KEY")

DEBUG = env("DEBUG")

ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=[])
```

อธิบายเมธอดสำคัญของ `django-environ`:

| เมธอด | หน้าที่ | ตัวอย่าง |
|---|---|---|
| `env("KEY")` | อ่านค่า string ธรรมดา | `env("SECRET_KEY")` |
| `env("KEY", default=...)` | อ่านค่าพร้อมกำหนดค่า default ถ้าไม่พบ | `env("DEBUG", default=False)` |
| `env.bool("KEY")` | แปลงค่าเป็น `True`/`False` อัตโนมัติ | `"True"` → `True` |
| `env.list("KEY")` | แยกค่าที่คั่นด้วย comma เป็น list | `"a,b,c"` → `["a", "b", "c"]` |
| `env.int("KEY")` | แปลงค่าเป็นจำนวนเต็ม | `"8000"` → `8000` |
| `env.db()` | อ่านค่าฐานข้อมูลจาก URL เดียว (เช่น `postgres://...`) | ใช้บ่อยมากใน Phase 2 |

### 38.6 ป้องกันไม่ให้ .env หลุดขึ้น Git

ตรวจสอบไฟล์ `.gitignore` ที่เราสร้างไว้ตั้งแต่ Part 001 ว่ามีบรรทัด `.env` อยู่แล้ว
(ถ้ายังไม่มี ให้เพิ่มทันที):

```bash
# .gitignore (เพิ่มถ้ายังไม่มี)
.env
```

จากนั้นสร้างไฟล์ `.env.example` ที่ **ไม่มีค่าจริง** เก็บไว้เป็นตัวอย่างให้คนอื่น
(หรือตัวเองในอนาคต) รู้ว่าต้องตั้งค่าตัวแปรอะไรบ้าง — ไฟล์นี้ **ปลอดภัยที่จะ commit
ขึ้น Git ได้**:

```bash
# .env.example
DEBUG=True
SECRET_KEY=your-secret-key-here-generate-a-new-one
ALLOWED_HOSTS=127.0.0.1,localhost
```

ตรวจสอบให้แน่ใจว่า `.env` ไม่ถูก track โดย Git:

```bash
git status
# .env ไม่ควรปรากฏใน list ไฟล์ที่รอ commit เลย (เพราะถูก .gitignore ดักไว้)

git add .env.example config/settings.py requirements.txt
git commit -m "เพิ่ม django-environ สำหรับจัดการ config อย่างปลอดภัย"
```

### 38.7 ทดสอบว่ายังทำงานถูกต้อง

```bash
python manage.py check
```

```
System check identified no issues (0 silenced).
```

ถ้าเห็นผลลัพธ์นี้ แปลว่า Django อ่านค่าจาก `.env` ผ่าน `django-environ` ได้สำเร็จ
และโปรเจกต์ยังทำงานได้ปกติทุกอย่าง เพียงแต่ตอนนี้ค่าที่ sensitive ถูกแยกออกจากโค้ด
เรียบร้อยแล้ว

---

## ขั้นตอนที่ 39: เขียน View แรกแบบ inline ใน urls.py

### 39.1 เป้าหมายของขั้นตอนนี้

ก่อนที่เราจะเรียนรู้การสร้างแอปอย่างเป็นทางการใน Part 005 เราจะพิสูจน์ให้เห็นก่อนว่า
Django project ของเราพร้อมประมวลผล request และตอบกลับ response ได้จริง ด้วยการเขียน
**View ที่ง่ายที่สุดเท่าที่จะเป็นไปได้** โดยไม่ต้องสร้างแอปแยกต่างหาก — เขียนฟังก์ชัน
view ไว้ **ในไฟล์ `config/urls.py` โดยตรง**

**ข้อควรทราบ**: วิธีนี้ใช้เพื่อการทดสอบและการเรียนรู้เท่านั้น ในโปรเจกต์จริงเราจะไม่
เขียน view ปนอยู่ใน `urls.py` ของ project แบบนี้ แต่จะแยก view ไว้ในแอปของตัวเองเสมอ
(ตามที่อธิบายในขั้นตอนที่ 37) — Part 005-007 จะสอนวิธีที่ถูกต้องแบบเต็มรูปแบบ

### 39.2 แก้ไข `config/urls.py`

เปิดไฟล์ `config/urls.py` แล้วแก้ไขให้เป็นดังนี้:

```python
"""
URL configuration for config project.
"""
from django.contrib import admin
from django.http import HttpResponse
from django.urls import path


def hello_django(request):
    """View ทดสอบแรกของเรา: รับ request แล้วตอบกลับ HTML ง่าย ๆ"""
    html_content = """
    <html>
        <head>
            <title>Hello Django</title>
        </head>
        <body>
            <h1>Hello Django!</h1>
            <p>โปรเจกต์ Django แรกของคุณทำงานได้แล้ว 🎉</p>
            <p>สร้างด้วย: <code>django-admin startproject config .</code></p>
        </body>
    </html>
    """
    return HttpResponse(html_content)


urlpatterns = [
    path("admin/", admin.site.urls),
    path("", hello_django, name="hello"),
]
```

### 39.3 อธิบายโค้ดทีละส่วน

- `from django.http import HttpResponse` — `HttpResponse` คือคลาสพื้นฐานที่สุดของ
  Django สำหรับสร้าง HTTP response ทุก view ต้อง return วัตถุที่เป็น (หรือสืบทอดจาก)
  `HttpResponse` เสมอ (เจาะลึกเต็มรูปแบบใน Part 007)
- `def hello_django(request):` — นี่คือ **Function-Based View (FBV)** รูปแบบพื้นฐาน
  ที่สุด รับ parameter แรกเสมอคือ `request` (วัตถุ `HttpRequest` ที่มีข้อมูลทุกอย่าง
  ของ request ที่เข้ามา เช่น method, headers, GET/POST data)
- `return HttpResponse(html_content)` — ส่งกลับ HTML string ธรรมดา ห่อด้วย
  `HttpResponse` เพื่อให้ Django รู้ว่านี่คือ response ที่จะส่งกลับไปหาผู้ใช้ พร้อม
  Content-Type เริ่มต้นเป็น `text/html`
- `path("", hello_django, name="hello")` — ผูก URL pattern ว่างเปล่า (คือ root path
  `/`) เข้ากับฟังก์ชัน `hello_django` และตั้งชื่อ URL นี้ว่า `"hello"` (การตั้งชื่อ URL
  มีประโยชน์มากเมื่อต้องอ้างอิงกลับใน template หรือโค้ดอื่น ๆ ผ่านฟังก์ชัน `reverse()`
  ซึ่งจะเรียนใน Part 006)

### 39.4 ทดสอบผลลัพธ์

รันเซิร์ฟเวอร์ (ถ้ายังไม่ได้รัน):

```bash
python manage.py runserver
```

เปิดเบราว์เซอร์ไปที่ `http://127.0.0.1:8000/` — คุณควรเห็นข้อความ **"Hello Django!"**
แทนที่หน้าจรวดต้อนรับเริ่มต้น นี่คือหลักฐานว่า:

1. Django project ทำงานถูกต้องตั้งแต่การ routing (`urls.py`)
2. View function ถูกเรียกและประมวลผลสำเร็จ
3. Response ถูกส่งกลับไปแสดงผลในเบราว์เซอร์ได้จริง

ทดสอบอีกทางด้วย `curl` จาก terminal อีกหน้าต่างหนึ่ง (เปิด terminal ใหม่ เพราะหน้าต่าง
เดิมกำลังรัน `runserver` ค้างอยู่):

```bash
curl http://127.0.0.1:8000/
```

ผลลัพธ์ควรเป็น HTML เดียวกับที่เห็นในเบราว์เซอร์ (แสดงเป็น raw HTML text)

ทดสอบว่า `admin/` ยังทำงานได้ปกติ:

```bash
curl -I http://127.0.0.1:8000/admin/
# ควรได้ HTTP/1.1 302 Found (redirect ไปหน้า login เพราะยังไม่ได้ login)
```

### 39.5 ทำไมวิธีนี้ถึงใช้ได้ทั้งที่ไม่มีแอป

Django ไม่ได้บังคับว่า view **ต้อง** อยู่ในแอปเสมอไป — `urls.py` ของ project เองก็เป็น
ไฟล์ Python ธรรมดาที่สามารถ import และเรียกใช้ฟังก์ชันใด ๆ ก็ได้ ตราบใดที่ฟังก์ชันนั้น
รับ `request` และ return `HttpResponse` (หรือ subclass ของมัน) ก็ถือว่าเป็น "view"
ที่ถูกต้องตามหลักการของ Django ทันที นี่คือเหตุผลที่เราทำแบบนี้ได้ในขั้นตอนทดสอบ แต่
เมื่อโปรเจกต์เริ่มมีความซับซ้อน การแยกเป็นแอปตามที่อธิบายในขั้นตอนที่ 37 จะช่วยให้จัดการ
ได้ง่ายกว่ามาก

---

## ขั้นตอนที่ 40: สรุปและแบบฝึกหัด

### 40.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ สร้างโปรเจกต์ Django แรกด้วย `django-admin startproject config .` และเข้าใจ
  ความหมายของ `.` ท้ายคำสั่ง
- ✅ เจาะลึกทุกไฟล์ที่ `startproject` สร้างให้: `manage.py`, `__init__.py`, `urls.py`,
  `wsgi.py`, `asgi.py`
- ✅ เข้าใจความหมายของทุก setting สำคัญใน `settings.py`: `SECRET_KEY`, `DEBUG`,
  `ALLOWED_HOSTS`, `INSTALLED_APPS`, `MIDDLEWARE`, `DATABASES`, `TEMPLATES`
- ✅ รัน development server ด้วย `runserver` และอ่าน output/warning ได้อย่างเข้าใจ
- ✅ รู้จักคำสั่ง `manage.py` ที่สำคัญที่สุด: `migrate`, `createsuperuser`, `shell`,
  `check`, `dbshell`
- ✅ เข้าใจความแตกต่างระหว่าง WSGI กับ ASGI และเมื่อไรควรใช้แบบไหน
- ✅ เข้าใจแนวคิด Project vs App และเหตุผลที่ Django ออกแบบให้แยกกัน
- ✅ จัดการ secret ด้วย `.env` และ `django-environ` อย่างปลอดภัย ไม่ hardcode
  `SECRET_KEY` อีกต่อไป
- ✅ เขียน View แรกแบบ inline ใน `urls.py` และเห็นผลลัพธ์ "Hello Django!" บนเบราว์เซอร์

### 40.2 Checklist ก่อนไป Part ถัดไป

- [ ] มีโฟลเดอร์ `config/` พร้อมไฟล์ `settings.py`, `urls.py`, `wsgi.py`, `asgi.py`
      อยู่ในระดับเดียวกับ `manage.py`
- [ ] รัน `python manage.py runserver` ได้โดยไม่มี error
- [ ] รัน `python manage.py migrate` สำเร็จ และไม่มีคำเตือน unapplied migrations อีก
- [ ] เข้าใจความหมายของ `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` อย่างน้อยระดับพื้นฐาน
- [ ] ติดตั้ง `django-environ` และย้าย `SECRET_KEY`/`DEBUG`/`ALLOWED_HOSTS` ไปไว้ใน
      `.env` สำเร็จ
- [ ] ไฟล์ `.env` อยู่ใน `.gitignore` และไม่ปรากฏใน `git status`
- [ ] มีไฟล์ `.env.example` ที่ commit ขึ้น Git ได้อย่างปลอดภัย
- [ ] เปิดเบราว์เซอร์ที่ `http://127.0.0.1:8000/` แล้วเห็นข้อความ "Hello Django!"
- [ ] อธิบายความแตกต่างระหว่าง Project กับ App ด้วยคำพูดของตัวเองได้

### 40.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: แก้ไขฟังก์ชัน `hello_django` ใน `config/urls.py` ให้แสดงข้อมูล
เพิ่มเติมต่อไปนี้ในหน้า HTML: เวลาปัจจุบันของเซิร์ฟเวอร์ (ใช้
`from datetime import datetime`) และเวอร์ชันของ Django ที่กำลังใช้งานอยู่ (ใช้
`import django; django.get_version()`)

**แบบฝึกหัดที่ 2**: เพิ่ม URL pattern ใหม่ที่ path `/about/` ผูกกับ view ฟังก์ชันใหม่
ชื่อ `about_view` ที่ return ข้อความแนะนำตัวสั้น ๆ เป็นภาษาไทย ทดสอบด้วยเบราว์เซอร์และ
`curl` ว่าทำงานถูกต้อง

**แบบฝึกหัดที่ 3**: เปลี่ยนค่า `LANGUAGE_CODE` ใน `settings.py` จาก `"en-us"` เป็น
`"th"` และ `TIME_ZONE` เป็น `"Asia/Bangkok"` แล้วรัน `python manage.py check` เพื่อ
ยืนยันว่าไม่มี error เกิดขึ้น จากนั้นเข้าไปดูผลกระทบที่หน้า Django Admin
(`/admin/`) ว่าภาษาของหน้าจอเปลี่ยนไปหรือไม่ (ต้อง `createsuperuser` ก่อน)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ลองรัน `python manage.py runserver` แล้วเปิดไฟล์
`config/settings.py` ตั้งใจเขียนโค้ด Python ที่ผิด syntax (เช่น ลืมปิดวงเล็บ) แล้ว
save ไฟล์ สังเกตว่า auto-reloader แสดง error อย่างไรในหน้าเทอร์มินัลและในเบราว์เซอร์
บันทึกสิ่งที่สังเกตเห็นลงในไฟล์ `notes.md` แล้วแก้ไขให้ถูกต้องกลับมา

### 40.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมต้องตั้งชื่อโปรเจกต์ว่า `config` ทำไมไม่ใช้ชื่อบริษัทหรือชื่อโปรเจกต์จริงเลย?**
A: เพื่อความชัดเจนและป้องกันความสับสนกับชื่อแอปในอนาคต (อธิบายละเอียดในขั้นตอนที่ 31.5)
แต่ในทางเทคนิคคุณสามารถตั้งชื่ออื่นได้ ขอแค่ไม่ชนกับชื่อ built-in module ของ Python
หรือ Django เอง ทีมงานหลายแห่งก็มีธรรมเนียมของตัวเอง เช่น `core`, `app`, หรือชื่อ
โปรเจกต์จริงก็มี — สิ่งสำคัญคือ **ความสม่ำเสมอ (consistency)** ภายในทีม

**Q: ลืม migrate แล้วเป็นอันตรายไหม?**
A: ไม่เป็นอันตรายต่อข้อมูลใด ๆ เพราะยังไม่มีตารางถูกสร้างเลย เพียงแต่ฟีเจอร์ที่ต้องพึ่ง
ฐานข้อมูล เช่น Django Admin หรือระบบ login จะยังใช้งานไม่ได้จนกว่าจะรัน `migrate`
สามารถรันได้ทุกเมื่อโดยไม่มีผลข้างเคียง

**Q: จำเป็นต้องใช้ django-environ เสมอไปหรือไม่ มี library อื่นทดแทนได้ไหม?**
A: ไม่จำเป็นต้องใช้ `django-environ` เจาะจง มีทางเลือกอื่นเช่น `python-decouple`,
`python-dotenv` ร่วมกับการอ่านค่าเอง หรือใน framework สมัยใหม่บางทีมใช้
`pydantic-settings` หลักการสำคัญคือ **แยก config ออกจากโค้ดเสมอ** ไม่ว่าจะใช้เครื่องมือ
ไหนก็ตาม หลักสูตรนี้เลือก `django-environ` เพราะเป็นที่นิยมที่สุดในชุมชน Django และ
ใช้งานง่ายที่สุดสำหรับผู้เริ่มต้น

**Q: ถ้าลบไฟล์ `db.sqlite3` ทิ้งจะเกิดอะไรขึ้น?**
A: ข้อมูลทั้งหมดในฐานข้อมูล (รวมถึง superuser ที่สร้างไว้) จะหายไปทันที แต่โครงสร้าง
ตาราง (schema) จะไม่หายไปจากระบบ เพราะถูกเก็บอยู่ในไฟล์ migration (`migrations/`)
ต่างหาก เพียงแค่รัน `python manage.py migrate` ใหม่อีกครั้งก็จะได้ไฟล์ฐานข้อมูลเปล่า
ที่มีโครงสร้างตารางถูกต้องกลับมา (แต่ข้อมูลเก่าจะไม่กลับมา ถ้าไม่มี backup)

**Q: WSGI กับ ASGI เลือกใช้ตัวไหนดีสำหรับมือใหม่?**
A: สำหรับผู้เริ่มต้นและโปรเจกต์ทั่วไปที่ยังไม่ต้องการ WebSocket หรือ async view แนะนำ
ให้ใช้ **WSGI** ไปก่อน เพราะเครื่องมือและเอกสารรองรับมานานกว่า เข้าใจง่ายกว่า และ
เพียงพอสำหรับเว็บแอปพลิเคชัน 90% ที่พบเจอในงานจริง ค่อยเปลี่ยนไปศึกษา ASGI เมื่อถึง
Phase 9 ของหลักสูตรนี้

### 40.5 เตรียมตัวสำหรับ Part ถัดไป

**Part 005: Django Apps และการจัดระเบียบโค้ด** จะพาคุณสร้างแอปแรกอย่างเป็นทางการด้วย
คำสั่ง `python manage.py startapp` เจาะลึกไฟล์ทุกไฟล์ที่ `startapp` สร้างให้ (`models.py`,
`views.py`, `admin.py`, `apps.py`, `tests.py`, โฟลเดอร์ `migrations/`) เรียนรู้วิธี
ลงทะเบียนแอปเข้า `INSTALLED_APPS` อย่างถูกต้อง และเริ่มวางรากฐานการจัดโครงสร้างโค้ด
ที่ดีตั้งแต่แอปแรก ก่อนที่เราจะเริ่มเขียน URL routing, View และ Template อย่างจริงจัง
ใน Part 006-008 ต่อไป

เตรียม virtual environment ของคุณให้พร้อม (ยัง activate อยู่) และเปิดโปรเจกต์
`django-mastery-course` ที่สร้างไว้ใน Part นี้ค้างไว้ เพราะเราจะสร้างต่อยอดจากโปรเจกต์
เดียวกันนี้ไปตลอดทั้งหลักสูตร!
