# Part 009: Static Files และ Media Files เบื้องต้น

> **ขั้นตอนที่ 81-90 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เข้าใจระบบจัดการไฟล์ประเภทต่าง ๆ ของ Django อย่างครบวงจร
> ตั้งแต่ **Static Files** (CSS, JavaScript, รูปภาพของธีม, ฟอนต์) ที่มากับตัวแอปพลิเคชัน
> ไปจนถึง **Media Files** (ไฟล์ที่ผู้ใช้อัปโหลดเข้ามา) คุณจะได้ลงมือจัดโครงสร้างโฟลเดอร์
> `static/` ในแอป `blog` จริง เชื่อมกับ template ที่เขียนไว้ใน Part 008 รู้จักคำสั่ง
> `collectstatic` และเหตุผลที่ production ต้องรวมไฟล์ static ไว้ที่เดียว ตั้งค่า
> `ImageField`/`FileField` เบื้องต้น ติดตั้ง **WhiteNoise** เพื่อ serve static files ใน
> production โดยไม่ต้องพึ่ง Nginx และปิดท้ายด้วยเทคนิค cache busting ระดับมืออาชีพด้วย
> `ManifestStaticFilesStorage` เมื่อจบ Part นี้ คุณจะเข้าใจว่าทำไมเว็บไซต์ Django ที่ deploy
> จริงถึงต้องจัดการไฟล์ static/media แตกต่างจากตอนรัน `runserver` บนเครื่องตัวเองโดยสิ้นเชิง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 81: `STATIC_URL`, `STATICFILES_DIRS` และ convention ของโฟลเดอร์ `static/` ในแต่ละแอป
- ขั้นตอนที่ 82: คำสั่ง `collectstatic` และ `STATIC_ROOT` (ทำไม production ต้องรวมไฟล์ static ไว้ที่เดียว)
- ขั้นตอนที่ 83: จัดการ CSS/JS/Images จริงใน `static/` ของแอป เชื่อมกับ template จาก Part 008
- ขั้นตอนที่ 84: `MEDIA_URL`, `MEDIA_ROOT` สำหรับไฟล์ที่ผู้ใช้อัปโหลด ต่างจาก static อย่างไร
- ขั้นตอนที่ 85: เกริ่น `ImageField`/`FileField` เบื้องต้น
- ขั้นตอนที่ 86: การ serve media files ตอน development เทียบกับ production
- ขั้นตอนที่ 87: WhiteNoise — serve static files ใน production โดยไม่ง้อ Nginx
- ขั้นตอนที่ 88: เกริ่นแนวคิด CDN สำหรับ static/media (AWS S3 + CloudFront, django-storages)
- ขั้นตอนที่ 89: Best practice: cache busting ด้วย `ManifestStaticFilesStorage`
- ขั้นตอนที่ 90: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 81: STATIC_URL, STATICFILES_DIRS และ convention ของโฟลเดอร์ static/ ในแต่ละแอป

### 81.1 Static Files คืออะไรกันแน่

**Static files** คือไฟล์ที่ **ไม่เปลี่ยนแปลงเนื้อหาระหว่าง request** — เนื้อหาของมันถูกเขียน
ไว้ล่วงหน้าโดยนักพัฒนา ไม่ได้ถูกสร้างขึ้นแบบไดนามิกจากฐานข้อมูลหรือ logic ของ view เช่น:

- **CSS** — ไฟล์จัดสไตล์หน้าเว็บ (`style.css`)
- **JavaScript** — ไฟล์ที่ทำงานฝั่ง browser (`main.js`)
- **รูปภาพของธีม/UI** — โลโก้, ไอคอน, ภาพพื้นหลัง (ไม่ใช่รูปที่ผู้ใช้อัปโหลด)
- **ฟอนต์** — ไฟล์ `.woff2`, `.ttf`
- **ไฟล์ third-party library** — Bootstrap, jQuery ที่ดาวน์โหลดมาเก็บไว้เอง (vendor files)

ต่างจาก HTML ที่ Django render จาก template ทุกครั้งที่มี request (dynamic content)
static files คือไฟล์ **"นิ่ง" (static)** ที่เว็บเซิร์ฟเวอร์เพียงแค่ "ส่งไฟล์ตรง ๆ" ให้
browser โดยไม่ต้องผ่านการประมวลผลของ Django view เลย

### 81.2 ทบทวน STATIC_URL จาก Part 004

ใน Part 004 (ขั้นตอนที่ 33.12) เราเห็นค่านี้อยู่แล้วใน `config/settings.py`:

```python
# config/settings.py
STATIC_URL = "static/"
```

`STATIC_URL` คือ **URL prefix** ที่ browser จะใช้อ้างอิงไฟล์ static — ไม่ใช่ path บนดิสก์
ของเครื่องคุณ ตัวอย่างเช่น ถ้าไฟล์จริงอยู่ที่ `blog/static/blog/css/style.css` บนดิสก์
เมื่อเข้าถึงผ่าน browser จะเป็น URL `http://127.0.0.1:8000/static/blog/css/style.css`
Django มีหน้าที่ **แปลผัง** ระหว่าง URL ที่ browser ขอ กับไฟล์จริงบนดิสก์ให้อัตโนมัติ

### 81.3 Convention: โฟลเดอร์ static/ ในแต่ละแอป (App-level static files)

เช่นเดียวกับ template ที่เราเรียนใน Part 008 ว่าแต่ละแอปมีโฟลเดอร์ `templates/<app_name>/`
ของตัวเอง (namespace ป้องกันชื่อไฟล์ชนกัน) — static files ก็ใช้ convention เดียวกันทุก
ประการ:

```
blog/
├── migrations/
├── static/
│   └── blog/              ← ซ้อนโฟลเดอร์ชื่อแอปอีกชั้น (เหมือน templates/)
│       ├── css/
│       │   └── style.css
│       ├── js/
│       │   └── main.js
│       └── images/
│           └── logo.png
├── templates/
│   └── blog/
│       ├── post_list.html
│       └── post_detail.html
├── __init__.py
├── admin.py
├── apps.py
├── models.py
├── urls.py
└── views.py
```

**ทำไมต้องซ้อนโฟลเดอร์ `blog/` อีกชั้นข้างใน `static/`?** เหตุผลเดียวกับ template
เป๊ะ ๆ: เมื่อ Django รวบรวม static files จากทุกแอปเข้าด้วยกัน (เราจะเห็นตอน `collectstatic`
ในขั้นตอนที่ 82) ถ้าทั้งแอป `blog` และแอป `shop` ต่างก็มีไฟล์ชื่อ `static/css/style.css`
เหมือนกัน ไฟล์หนึ่งจะถูกทับอีกไฟล์หนึ่งทันที แต่ถ้าเป็น `blog/static/blog/css/style.css`
กับ `shop/static/shop/css/style.css` จะไม่มีทางชนกันเลย

| Path | คำอธิบาย |
|---|---|
| `blog/static/` | โฟลเดอร์ static files ของแอป (Django ค้นหาอัตโนมัติถ้าแอปอยู่ใน `INSTALLED_APPS`) |
| `blog/static/blog/` | namespace ป้องกันชื่อชนกันข้ามแอป (convention เดียวกับ templates) |
| `blog/static/blog/css/style.css` | ไฟล์จริง เข้าถึงผ่าน URL `/static/blog/css/style.css` |

### 81.4 STATICFILES_FINDERS — กลไกเบื้องหลังการค้นหาไฟล์

Django ไม่ได้ "เดา" ว่าไฟล์ static อยู่ที่ไหน แต่ใช้ระบบ **finder** ที่กำหนดไว้ใน setting
`STATICFILES_FINDERS` (ค่าเริ่มต้นไม่ต้องเขียนเองก็ได้ เพราะ Django ตั้งไว้ให้แล้ว):

```python
# ค่าเริ่มต้นของ Django (ไม่ต้องเขียนเองใน settings.py ก็ได้ แต่แสดงไว้เพื่อความเข้าใจ)
STATICFILES_FINDERS = [
    "django.contrib.staticfiles.finders.FileSystemFinder",
    "django.contrib.staticfiles.finders.AppDirectoriesFinder",
]
```

| Finder | ค้นหาที่ไหน |
|---|---|
| `AppDirectoriesFinder` | ค้นหาในโฟลเดอร์ `static/` ของทุกแอปที่อยู่ใน `INSTALLED_APPS` โดยอัตโนมัติ |
| `FileSystemFinder` | ค้นหาในโฟลเดอร์ที่ระบุไว้ใน `STATICFILES_DIRS` (โฟลเดอร์ static ระดับโปรเจกต์ ไม่ผูกกับแอปใดแอปหนึ่ง) |

### 81.5 STATICFILES_DIRS — static files ระดับโปรเจกต์

บางไฟล์ static ไม่ได้ผูกกับแอปใดแอปหนึ่งโดยเฉพาะ เช่น โลโก้บริษัทที่ใช้ร่วมกันทุกหน้า,
ไฟล์ CSS หลักของทั้งเว็บไซต์, หรือไฟล์ library ภายนอกที่ดาวน์โหลดมาเก็บเอง — กรณีนี้ควรเก็บ
ไว้ในโฟลเดอร์ `static/` ที่ระดับ **root ของโปรเจกต์** (เทียบเท่ากับโฟลเดอร์ `templates/`
ระดับโปรเจกต์ที่เราตั้งค่าไว้ใน Part 008)

สร้างโฟลเดอร์:

```bash
mkdir -p static/css static/js static/images
```

โครงสร้างที่ได้:

```
django-mastery-course/
├── config/
├── blog/
├── static/                    ← static files ระดับโปรเจกต์ (ใหม่)
│   ├── css/
│   │   └── base.css
│   ├── js/
│   │   └── app.js
│   └── images/
│       └── site-logo.png
├── templates/
│   └── base.html
├── manage.py
└── ...
```

แล้วเพิ่มการตั้งค่าใน `config/settings.py`:

```python
# config/settings.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

STATIC_URL = "static/"

STATICFILES_DIRS = [
    BASE_DIR / "static",
]
```

**ข้อควรระวังสำคัญ**: `STATICFILES_DIRS` กับโฟลเดอร์ `static/` ของแต่ละแอป (ที่ Django
ค้นหาอัตโนมัติผ่าน `AppDirectoriesFinder`) เป็นคนละกลไกกัน — คุณ **ไม่ต้อง** เพิ่มโฟลเดอร์
`blog/static/` เข้าไปใน `STATICFILES_DIRS` เอง เพราะ Django หามันเจอโดยอัตโนมัติอยู่แล้ว
`STATICFILES_DIRS` มีไว้สำหรับโฟลเดอร์ static ที่ **ไม่ได้อยู่ในแอปไหนเลย** เท่านั้น

### 81.6 ใช้งาน {% static %} template tag

ห้าม hardcode URL ของไฟล์ static ตรง ๆ ใน template (เช่น `<link href="/static/blog/css/style.css">`)
เพราะถ้าเปลี่ยน `STATIC_URL` ในอนาคต (เช่นย้ายไปใช้ CDN ตามขั้นตอนที่ 88) จะต้องไล่แก้ทุกไฟล์
วิธีที่ถูกต้องคือใช้ template tag `{% static %}`:

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My Blog{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/base.css' %}">
    <link rel="icon" href="{% static 'images/site-logo.png' %}">
</head>
<body>
    {% block content %}{% endblock %}
    <script src="{% static 'js/app.js' %}"></script>
</body>
</html>
```

- `{% load static %}` ต้องเขียนไว้บนสุดของไฟล์เสมอ (โหลด template tag library ชื่อ `static`
  ซึ่งมาจากแอป `django.contrib.staticfiles`)
- `{% static 'css/base.css' %}` จะถูกแปลงเป็น `/static/css/base.css` (รวม `STATIC_URL`
  ให้อัตโนมัติ) — สังเกตว่า **ไม่ต้อง** ใส่ `static/` นำหน้าซ้ำ เพราะ tag จะเติมให้เอง

**ข้อควรระวัง**: `django.contrib.staticfiles` ต้องอยู่ใน `INSTALLED_APPS` (ซึ่งมีอยู่แล้ว
ตั้งแต่ตอน `startproject` ตาม Part 004) ไม่เช่นนั้น `{% load static %}` จะ error ทันที

---

## ขั้นตอนที่ 82: คำสั่ง collectstatic และ STATIC_ROOT

### 82.1 ปัญหาที่ collectstatic แก้ไข

ตอนพัฒนา (development) ไฟล์ static ของเรากระจัดกระจายอยู่หลายที่:

```
blog/static/blog/css/style.css
shop/static/shop/css/style.css
accounts/static/accounts/css/style.css
static/css/base.css                    (ระดับโปรเจกต์)
venv/lib/.../admin/static/admin/css/... (มากับ django.contrib.admin)
```

ตอนรัน `runserver` โดยมี `django.contrib.staticfiles` อยู่ใน `INSTALLED_APPS` และ
`DEBUG = True` Django จะใช้ **finder** ไล่ค้นหาไฟล์จากทุกที่เหล่านี้แบบ real-time ให้เอง
สะดวกมากตอนพัฒนา แต่ระบบนี้ **ช้าเกินไปสำหรับ production** และเว็บเซิร์ฟเวอร์จริงอย่าง
Nginx หรือ WhiteNoise (ขั้นตอนที่ 87) ก็ไม่รู้จักกลไก finder ของ Django — พวกมันต้องการ
**โฟลเดอร์เดียวที่รวมไฟล์ static ทั้งหมดไว้แล้ว** เพื่อ serve ไฟล์ได้อย่างรวดเร็วที่สุด

คำสั่ง `collectstatic` ถูกสร้างมาเพื่อแก้ปัญหานี้โดยเฉพาะ: **มันจะไล่คัดลอกไฟล์ static
จากทุกแหล่ง (ทุกแอป + `STATICFILES_DIRS`) มารวมไว้ในโฟลเดอร์เดียว**

```
┌─────────────────────┐
│ blog/static/blog/    │──┐
├─────────────────────┤  │
│ shop/static/shop/     │──┤     collectstatic      ┌──────────────────┐
├─────────────────────┤  ├───────────────────────▶ │  STATIC_ROOT      │
│ static/ (โปรเจกต์)   │──┤                          │  (โฟลเดอร์เดียว)  │
├─────────────────────┤  │                          └──────────────────┘
│ django.contrib.admin │──┘                                   │
└─────────────────────┘                                        │
                                                    Nginx / WhiteNoise
                                                    serve จากที่นี่ที่เดียว
```

### 82.2 ตั้งค่า STATIC_ROOT

เพิ่มค่านี้ใน `config/settings.py`:

```python
# config/settings.py
STATIC_URL = "static/"

STATICFILES_DIRS = [
    BASE_DIR / "static",
]

# โฟลเดอร์ปลายทางที่ collectstatic จะรวมไฟล์ static ทั้งหมดไว้ (ใช้เฉพาะ production)
STATIC_ROOT = BASE_DIR / "staticfiles"
```

**ข้อควรจำที่สำคัญที่สุดของขั้นตอนนี้**: `STATICFILES_DIRS` กับ `STATIC_ROOT` **ห้ามชี้ไป
ที่โฟลเดอร์เดียวกันเด็ดขาด** เพราะ `collectstatic` จะพยายามคัดลอกไฟล์จาก `STATICFILES_DIRS`
ไปไว้ที่ `STATIC_ROOT` — ถ้าทั้งสองเป็นโฟลเดอร์เดียวกัน Django จะ error ทันที

| Setting | ใช้ตอนไหน | หน้าที่ |
|---|---|---|
| `STATICFILES_DIRS` | Development (และเป็นแหล่งข้อมูลให้ `collectstatic` อ่าน) | บอกว่า static files ระดับโปรเจกต์อยู่ที่ไหน |
| `STATIC_ROOT` | Production เท่านั้น (ปลายทางของ `collectstatic`) | โฟลเดอร์รวมไฟล์ static ทั้งหมดที่พร้อม serve จริง |

### 82.3 รันคำสั่ง collectstatic

```bash
python manage.py collectstatic
```

ผลลัพธ์ที่คาดว่าจะเห็น:

```
You have requested to collect static files at the destination
location as specified in your settings:

    /home/user/django-mastery-course/staticfiles

This will overwrite existing files!
Are you sure you want to do this?

Type 'yes' to continue, or 'no' to cancel: yes

162 static files copied to '/home/user/django-mastery-course/staticfiles'.
```

Django จะถามยืนยันก่อนเสมอ (เพราะเป็นคำสั่งที่ overwrite โฟลเดอร์ปลายทาง) ตัวเลข 162
ไฟล์ในตัวอย่างนี้ส่วนใหญ่มาจาก `django.contrib.admin` (Django Admin มีไฟล์ CSS/JS ของ
ตัวเองเยอะมาก) รวมกับไฟล์ของเราเอง

### 82.4 ตัวเลือกที่ใช้บ่อยของ collectstatic

```bash
# ข้ามการถามยืนยัน (จำเป็นมากสำหรับใช้ใน CI/CD หรือ deployment script อัตโนมัติ)
python manage.py collectstatic --noinput

# ลบไฟล์เก่าทั้งหมดใน STATIC_ROOT ก่อนคัดลอกใหม่ (ป้องกันไฟล์เก่าที่ถูกลบออกจากโปรเจกต์
# แต่ยังค้างอยู่ใน STATIC_ROOT)
python manage.py collectstatic --noinput --clear

# ดูรายการไฟล์ที่ "จะ" ถูกคัดลอก โดยไม่คัดลอกจริง (dry run)
python manage.py collectstatic --dry-run

# แสดง log แบบละเอียดทุกไฟล์ที่คัดลอก
python manage.py collectstatic --noinput -v 2
```

### 82.5 อย่าลืมเพิ่ม staticfiles/ ใน .gitignore

โฟลเดอร์ `staticfiles/` เป็น **output ที่สร้างขึ้นใหม่ได้เสมอ** จากคำสั่ง `collectstatic`
จึงไม่ควร commit เข้า Git (เรากำหนดไว้แล้วใน `.gitignore` ตั้งแต่ Part 001):

```
# .gitignore (ตรวจสอบว่ามีบรรทัดนี้)
staticfiles/
media/
```

โดยทั่วไป **`collectstatic` จะถูกรันเป็นส่วนหนึ่งของขั้นตอน deploy เสมอ** (ไม่ใช่รันมือ
ทุกครั้ง) — เราจะเห็นมันอยู่ใน deployment script จริงเมื่อถึง Phase 11 (Deployment & DevOps)

### 82.6 ทำไม runserver ตอน DEBUG=True ไม่ต้อง collectstatic

คำถามที่มือใหม่สงสัยบ่อย: "ทำไมตอนพัฒนาไม่ต้องรัน `collectstatic` ก็เห็น static files
ทำงานปกติ?" คำตอบคือ เมื่อ `DEBUG = True` **และ** `django.contrib.staticfiles` อยู่ใน
`INSTALLED_APPS` Django จะเพิ่ม URL pattern พิเศษให้ `runserver` เอง serve ไฟล์ static
โดยตรงจากทุก finder แบบ real-time (สะดวกมากตอนพัฒนา แต่ **ช้าและไม่ปลอดภัยสำหรับ
production** — จะอธิบายเหตุผลแบบเต็มในขั้นตอนที่ 86)

---

## ขั้นตอนที่ 83: จัดการ CSS/JS/Images จริงใน static/ ของแอป

### 83.1 สร้างโครงสร้างไฟล์จริงในแอป blog

ต่อยอดจาก template `blog/post_list.html` และ `blog/post_detail.html` ที่เขียนไว้ใน
Part 008 มาเพิ่มไฟล์ static จริงให้แอป `blog`:

```bash
mkdir -p blog/static/blog/css blog/static/blog/js blog/static/blog/images
```

โครงสร้างที่ได้:

```
blog/
├── static/
│   └── blog/
│       ├── css/
│       │   └── blog.css
│       ├── js/
│       │   └── blog.js
│       └── images/
│           └── placeholder.png
├── templates/
│   └── blog/
│       ├── post_list.html
│       └── post_detail.html
└── ...
```

### 83.2 เขียน CSS จริงที่ใช้งานได้

```css
/* blog/static/blog/css/blog.css */

.post-list {
    max-width: 720px;
    margin: 0 auto;
    padding: 1.5rem;
    font-family: "Sarabun", "Segoe UI", sans-serif;
}

.post-card {
    border: 1px solid #e2e2e2;
    border-radius: 8px;
    padding: 1rem 1.25rem;
    margin-bottom: 1rem;
    transition: box-shadow 0.2s ease-in-out;
}

.post-card:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.post-card__title {
    font-size: 1.25rem;
    margin: 0 0 0.5rem 0;
}

.post-card__title a {
    color: #1a1a2e;
    text-decoration: none;
}

.post-card__title a:hover {
    text-decoration: underline;
}

.post-card__meta {
    font-size: 0.85rem;
    color: #666;
}

.post-card__excerpt {
    margin-top: 0.5rem;
    color: #333;
    line-height: 1.6;
}
```

### 83.3 เขียน JavaScript จริงที่ใช้งานได้

```javascript
// blog/static/blog/js/blog.js

document.addEventListener("DOMContentLoaded", function () {
    // นับจำนวนตัวอักษรที่เหลือในฟอร์มคอมเมนต์ (ตัวอย่างการใช้งานจริง)
    const commentField = document.querySelector("#id_content");
    const counter = document.querySelector("#char-counter");
    const MAX_LENGTH = 500;

    if (commentField && counter) {
        const updateCounter = () => {
            const remaining = MAX_LENGTH - commentField.value.length;
            counter.textContent = `เหลือ ${remaining} ตัวอักษร`;
            counter.classList.toggle("text-danger", remaining < 0);
        };

        commentField.addEventListener("input", updateCounter);
        updateCounter();
    }

    // เปิด/ปิดเมนูมือถือ
    const menuToggle = document.querySelector("[data-menu-toggle]");
    const mobileMenu = document.querySelector("[data-mobile-menu]");

    if (menuToggle && mobileMenu) {
        menuToggle.addEventListener("click", function () {
            mobileMenu.classList.toggle("is-open");
        });
    }
});
```

### 83.4 เชื่อมกับ template จาก Part 008

อัปเดต `blog/templates/blog/post_list.html` ให้โหลด CSS/JS ของแอปตัวเอง (เพิ่มเติมจาก
`base.html` ที่โหลด `base.css` ระดับโปรเจกต์ไปแล้วตามขั้นตอนที่ 81.6):

```html
<!-- blog/templates/blog/post_list.html -->
{% extends "base.html" %}
{% load static %}

{% block title %}บทความทั้งหมด{% endblock %}

{% block extra_css %}
    <link rel="stylesheet" href="{% static 'blog/css/blog.css' %}">
{% endblock %}

{% block content %}
<div class="post-list">
    <h1>บทความทั้งหมด</h1>
    {% for post in posts %}
        <article class="post-card">
            <h2 class="post-card__title">
                <a href="{% url 'blog:detail' slug=post.slug %}">{{ post.title }}</a>
            </h2>
            <p class="post-card__meta">เผยแพร่เมื่อ {{ post.created_at|date:"d M Y" }}</p>
            <p class="post-card__excerpt">{{ post.excerpt }}</p>
        </article>
    {% empty %}
        <p>ยังไม่มีบทความ</p>
    {% endfor %}
</div>
{% endblock %}

{% block extra_js %}
    <script src="{% static 'blog/js/blog.js' %}"></script>
{% endblock %}
```

เพื่อให้ `{% block extra_css %}` และ `{% block extra_js %}` ทำงานได้ ต้องเพิ่ม block
เหล่านี้ไว้ใน `templates/base.html` ด้วย (ในตำแหน่งที่เหมาะสม):

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My Blog{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/base.css' %}">
    {% block extra_css %}{% endblock %}
</head>
<body>
    {% block content %}{% endblock %}
    <script src="{% static 'js/app.js' %}"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

รูปแบบนี้คือ **best practice มาตรฐาน**: CSS/JS ที่ใช้ร่วมกันทุกหน้า (เช่น navbar, footer,
reset CSS) อยู่ใน `base.css`/`app.js` ระดับโปรเจกต์ ส่วน CSS/JS ที่ใช้เฉพาะฟีเจอร์ของแอปนั้น
(เช่น สไตล์เฉพาะของหน้าบล็อก) แยกเก็บไว้ในแอปของตัวเอง แล้วโหลดเพิ่มผ่าน block

### 83.5 การจัดระเบียบโฟลเดอร์ static เมื่อโปรเจกต์ใหญ่ขึ้น

| โฟลเดอร์ย่อย | เก็บอะไร |
|---|---|
| `css/` | Stylesheet ทั้งหมด |
| `js/` | JavaScript ทั้งหมด |
| `images/` | รูปภาพของ UI/ธีม (ไม่ใช่รูปที่ผู้ใช้อัปโหลด) |
| `fonts/` | ไฟล์ฟอนต์ที่ host เอง (`.woff2`, `.woff`) |
| `vendor/` | Library ภายนอกที่ดาวน์โหลดมาเก็บเอง (เช่น Bootstrap, Chart.js) แทนการโหลดจาก CDN สาธารณะ |

เราจะเรียนรู้การจัดการ CSS Framework อย่าง Bootstrap และ JavaScript library อย่างเป็น
ระบบมากขึ้นใน **Phase 6 (Part 051-058)**

---

## ขั้นตอนที่ 84: MEDIA_URL, MEDIA_ROOT สำหรับไฟล์ที่ผู้ใช้อัปโหลด

### 84.1 Media Files ต่างจาก Static Files อย่างไร

**Media files** คือไฟล์ที่ **ผู้ใช้ของระบบ** เป็นคนอัปโหลดเข้ามาระหว่างการใช้งานเว็บไซต์
เช่น รูปโปรไฟล์, ไฟล์แนบในฟอร์ม, รูปภาพหน้าปกบทความที่ผู้เขียนอัปโหลด — ไฟล์เหล่านี้
**ไม่มีอยู่ในโค้ดตั้งแต่แรก** แต่ถูกสร้างขึ้นขณะระบบทำงานจริง (runtime)

| คุณสมบัติ | Static Files | Media Files |
|---|---|---|
| ใครเป็นคนสร้างไฟล์ | นักพัฒนา (เขียนไว้ล่วงหน้า) | ผู้ใช้ระบบ (อัปโหลดตอนใช้งานจริง) |
| อยู่ใน Git repository หรือไม่ | ✅ ควร commit เข้า Git | ❌ ไม่ควร commit เข้า Git |
| เปลี่ยนแปลงบ่อยแค่ไหน | เปลี่ยนเมื่อ deploy โค้ดใหม่เท่านั้น | เปลี่ยนตลอดเวลาตามการใช้งานจริง |
| ตัวอย่าง | CSS, JS, โลโก้บริษัท | รูปโปรไฟล์ผู้ใช้, ไฟล์แนบ, ภาพหน้าปกบทความ |
| Setting ที่เกี่ยวข้อง | `STATIC_URL`, `STATIC_ROOT`, `STATICFILES_DIRS` | `MEDIA_URL`, `MEDIA_ROOT` |
| ต้องรัน `collectstatic` หรือไม่ | ✅ ต้องรันก่อน deploy | ❌ ไม่เกี่ยวข้องกับ `collectstatic` เลย |
| ในโลก production มักถูกเก็บที่ไหน | รวมไว้ที่เดียวแล้ว serve ผ่าน Nginx/WhiteNoise/CDN | มักเก็บบน Object Storage เช่น AWS S3 (ขั้นตอนที่ 88) |

### 84.2 ตั้งค่า MEDIA_URL และ MEDIA_ROOT

เพิ่มการตั้งค่าใน `config/settings.py`:

```python
# config/settings.py

# URL prefix สำหรับเข้าถึงไฟล์ media ผ่าน browser
MEDIA_URL = "media/"

# โฟลเดอร์บนดิสก์ที่ไฟล์ media จริงถูกเก็บไว้
MEDIA_ROOT = BASE_DIR / "media"
```

โครงสร้างที่จะเกิดขึ้นเมื่อมีผู้ใช้อัปโหลดไฟล์:

```
django-mastery-course/
├── media/                          ← สร้างอัตโนมัติเมื่อมีการอัปโหลดครั้งแรก
│   └── blog/
│       └── covers/
│           └── my-first-post-cover.jpg
├── static/
├── staticfiles/
└── ...
```

**สังเกตความคล้ายกันโดยตั้งใจ**: `MEDIA_URL`/`MEDIA_ROOT` มีรูปแบบเดียวกับ
`STATIC_URL`/`STATIC_ROOT` ทุกประการ — ต่างกันแค่ชื่อ setting และความหมายของไฟล์ข้างใน
เท่านั้น การตั้งชื่อคู่กันแบบนี้เป็นไปตามหลักการ **Convention over Configuration**
ที่เราพูดถึงตั้งแต่ Part 001

### 84.3 ยืนยันว่า .gitignore ครอบคลุม media/ แล้ว

```
# .gitignore
venv/
__pycache__/
*.pyc
.env
db.sqlite3
staticfiles/
media/
```

**เหตุผลที่ media/ ห้ามอยู่ใน Git**: ไฟล์ที่ผู้ใช้อัปโหลดอาจมีขนาดใหญ่มาก (รูปภาพความ
ละเอียดสูง, วิดีโอ) และเปลี่ยนแปลงตลอดเวลา การ commit เข้า Git จะทำให้ repository บวม
ขึ้นเรื่อย ๆ แบบไม่มีที่สิ้นสุด และไม่มีประโยชน์ เพราะไฟล์เหล่านี้ควรถูก backup แยกต่างหาก
(เช่น snapshot ของฐานข้อมูลที่เก็บ path คู่กับ storage บริการภายนอกอย่าง S3)

### 84.4 ทดสอบว่า MEDIA_ROOT ถูกสร้างจริง

ลองสร้างโฟลเดอร์และวางไฟล์ทดสอบด้วยมือก่อน (ก่อนที่จะเชื่อมกับ Model ในขั้นตอนถัดไป):

```bash
mkdir -p media/blog/covers
# ลองก็อปปี้รูปภาพใด ๆ ไปวางไว้เพื่อทดสอบ
cp ~/Pictures/sample.jpg media/blog/covers/sample.jpg
```

ในขั้นตอนที่ 86 เราจะทำให้ browser เปิดดูไฟล์นี้ได้จริงผ่าน URL `/media/blog/covers/sample.jpg`

---

## ขั้นตอนที่ 85: เกริ่น ImageField/FileField เบื้องต้น

### 85.1 Model Field สำหรับรับไฟล์อัปโหลด

Django มี Model field สองตัวสำหรับจัดการไฟล์ที่ผู้ใช้อัปโหลด:

| Field | ใช้เมื่อไหร่ |
|---|---|
| `models.FileField` | รับไฟล์ทั่วไป (PDF, Excel, ไฟล์แนบใด ๆ) |
| `models.ImageField` | รับเฉพาะไฟล์รูปภาพ (ตรวจสอบว่าเป็นรูปภาพจริงเพิ่มเติมจาก `FileField`) |

**ข้อกำหนดพิเศษของ `ImageField`**: ต้องติดตั้ง library **Pillow** ก่อนใช้งาน เพราะ Django
ใช้ Pillow ตรวจสอบและประมวลผลไฟล์รูปภาพเบื้องหลัง:

```bash
pip install Pillow
pip freeze > requirements.txt
```

### 85.2 ตัวอย่างการเพิ่ม ImageField ใน Model (ตัวอย่างสั้น ๆ)

สมมติแอป `blog` มี Model `Post` อยู่แล้ว (จะเรียนการสร้าง Model แบบเต็มรูปแบบใน Part 011)
ตัวอย่างการเพิ่มฟิลด์รูปภาพหน้าปก:

```python
# blog/models.py
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    excerpt = models.CharField(max_length=300, blank=True)
    content = models.TextField()
    cover_image = models.ImageField(
        upload_to="blog/covers/",
        blank=True,
        null=True,
    )
    attachment = models.FileField(
        upload_to="blog/attachments/",
        blank=True,
        null=True,
    )
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

- `upload_to="blog/covers/"` บอก Django ว่าเมื่อมีไฟล์ถูกอัปโหลดผ่านฟิลด์นี้ ให้เก็บไว้
  ในโฟลเดอร์ย่อย `blog/covers/` ภายใต้ `MEDIA_ROOT` — path เต็มจริงจะกลายเป็น
  `media/blog/covers/<ชื่อไฟล์>`
- `blank=True, null=True` ทำให้ฟิลด์นี้ **ไม่บังคับ** (ผู้เขียนบทความจะอัปโหลดรูปหรือไม่
  ก็ได้) — เราจะเจาะลึกความแตกต่างระหว่าง `blank` กับ `null` แบบเต็มใน Part 011

### 85.3 เข้าถึงไฟล์ที่อัปโหลดใน Template

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends "base.html" %}
{% load static %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>

    {% if post.cover_image %}
        <img src="{{ post.cover_image.url }}" alt="{{ post.title }}" class="cover-image">
    {% else %}
        <img src="{% static 'blog/images/placeholder.png' %}" alt="ไม่มีรูปภาพ">
    {% endif %}

    <div class="post-content">{{ post.content|linebreaks }}</div>

    {% if post.attachment %}
        <a href="{{ post.attachment.url }}" download>ดาวน์โหลดไฟล์แนบ</a>
    {% endif %}
</article>
{% endblock %}
```

สังเกตความแตกต่างที่สำคัญ:

- ไฟล์ **static** ใช้ `{% static 'path' %}` เพราะ path คงที่ รู้ล่วงหน้าตอนเขียนโค้ด
- ไฟล์ **media** ใช้ `{{ post.cover_image.url }}` โดยตรง (ไม่ต้องใช้ template tag พิเศษ)
  เพราะ Django สร้าง URL ให้อัตโนมัติจากค่าที่เก็บในฐานข้อมูล (คอลัมน์เก็บแค่ path สัมพัทธ์
  เช่น `blog/covers/sample.jpg` ส่วน `.url` property จะประกอบ `MEDIA_URL` ให้เองเป็น
  `/media/blog/covers/sample.jpg`)

### 85.4 เจาะลึกเต็มรูปแบบอยู่ที่ไหน

Part นี้แนะนำ `ImageField`/`FileField` เพียงผิวเผินเพื่อให้เข้าใจภาพรวมของระบบ static/media
เท่านั้น รายละเอียดเชิงลึกที่แท้จริงจะอยู่ใน:

- **Part 012 (Phase 2 — ความสัมพันธ์ระหว่างโมเดล)**: การออกแบบฟิลด์ไฟล์ในฐานข้อมูลอย่าง
  ถูกต้อง, custom `upload_to` function ที่ตั้งชื่อไฟล์แบบไดนามิก, `FileExtensionValidator`
- **Part 058 (Phase 6 — File Upload, Image Processing และ Media Handling)**: การ resize
  รูปภาพอัตโนมัติ, สร้าง thumbnail, ตรวจสอบขนาดไฟล์สูงสุด, จัดการฟอร์มอัปโหลดหลายไฟล์
  พร้อมกัน, progress bar ด้วย JavaScript

---

## ขั้นตอนที่ 86: การ serve media files ตอน development เทียบกับ production

### 86.1 ปัญหา: Django ไม่ serve media files ให้อัตโนมัติ

ต่างจาก static files ที่ Django ช่วย serve ให้อัตโนมัติตอน `DEBUG = True` (ผ่านกลไก
`django.contrib.staticfiles` ที่เราเห็นในขั้นตอนที่ 82.6) **media files ไม่มีกลไกแบบนั้น
มาให้เลยแม้แต่ตอนพัฒนา** ถ้าไม่ตั้งค่าเพิ่ม การเข้า URL `/media/blog/covers/sample.jpg`
จะได้ผลลัพธ์เป็นหน้า **404 Not Found** ทันที แม้ไฟล์จะมีอยู่จริงในโฟลเดอร์ `media/` ก็ตาม

### 86.2 วิธีแก้สำหรับ Development: django.conf.urls.static.static()

Django มีฟังก์ชันช่วยเหลือชื่อ `static()` ที่ import จาก `django.conf.urls.static`
สำหรับเพิ่ม URL pattern ที่ serve media files **เฉพาะตอนพัฒนาเท่านั้น**:

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("pages.urls")),
    path("posts/", include("blog.urls")),
]

# เพิ่ม URL pattern สำหรับ serve media files เฉพาะตอน DEBUG=True เท่านั้น
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

หลังเพิ่มโค้ดนี้ ลองรัน server แล้วเปิด URL `http://127.0.0.1:8000/media/blog/covers/sample.jpg`
(จากไฟล์ทดสอบที่วางไว้ในขั้นตอนที่ 84.4) — ควรเห็นรูปภาพแสดงในเบราว์เซอร์ทันที

### 86.3 ทำไมต้องมีเงื่อนไข if settings.DEBUG

นี่คือจุดที่มือใหม่พลาดบ่อยมาก **ฟังก์ชัน `static()` นี้ถูกออกแบบมาเพื่อความสะดวกตอน
พัฒนาเท่านั้น** เอกสารทางการของ Django เตือนไว้ชัดเจนว่า **"This will only work in
debug mode"** และห้ามใช้ใน production เพราะ:

1. **ไม่มีการปรับแต่งด้าน performance**: Django view ที่ serve ไฟล์นี้ (`django.views.static.serve`)
   ไม่มี caching header, ไม่มี compression, ไม่รองรับ range request (สำหรับ resume ดาวน์โหลด
   ไฟล์ใหญ่หรือ streaming วิดีโอ) อย่างมีประสิทธิภาพ
2. **ไม่ปลอดภัยพอ**: ไม่ได้ผ่านการตรวจสอบด้าน security อย่างละเอียดสำหรับรับ traffic จริง
   จากอินเทอร์เน็ตทั่วไป
3. **Django process ควรทำหน้าที่ประมวลผล logic ไม่ใช่ serve ไฟล์ดิบ**: การให้ Python
   process (ซึ่งช้ากว่าเว็บเซิร์ฟเวอร์ที่เขียนด้วยภาษาระดับต่ำอย่าง C มาก) มานั่ง serve
   ไฟล์รูปภาพ/วิดีโอ เป็นการใช้ทรัพยากรที่สิ้นเปลืองมาก

จริง ๆ แล้วแม้จะลืมใส่เงื่อนไข `if settings.DEBUG` แต่ตั้งแต่ Django 1.7 เป็นต้นมา
`static()` ก็จะ **คืนค่าเป็น list ว่างเปล่าโดยอัตโนมัติเมื่อ `DEBUG = False`** อยู่แล้ว
เพื่อความปลอดภัย แต่การเขียนเงื่อนไขให้เห็นชัดเจนยังเป็น **best practice** ที่แนะนำ เพราะ
สื่อความหมายชัดเจนกับคนอ่านโค้ดคนอื่นว่าโค้ดส่วนนี้มีไว้เพื่ออะไร

### 86.4 แล้ว Production ต้อง serve media files อย่างไร

| แนวทาง | คำอธิบาย | เหมาะกับ |
|---|---|---|
| **Nginx serve ไฟล์ตรง** | ตั้งค่า Nginx ให้ map URL `/media/` ไปที่โฟลเดอร์ `MEDIA_ROOT` บนดิสก์โดยตรง | เซิร์ฟเวอร์เดียว, ปริมาณผู้ใช้ปานกลาง |
| **Object Storage + CDN** | เก็บไฟล์บน AWS S3 / Google Cloud Storage แล้ว serve ผ่าน CloudFront/CDN | Production จริงระดับมืออาชีพ, รองรับ scale, มีหลาย server |
| **WhiteNoise** | **ใช้ได้กับ static files เท่านั้น ไม่รองรับ media files** | (ดูรายละเอียดในขั้นตอนที่ 87) |

Nginx เป็นเว็บเซิร์ฟเวอร์ที่เขียนด้วยภาษา C ออกแบบมาเพื่อ serve ไฟล์ static/media
โดยเฉพาะ เร็วกว่า Django process มาก ตัวอย่างการตั้งค่า Nginx (แสดงเพื่อความเข้าใจ
ภาพรวมเท่านั้น รายละเอียดการติดตั้งและตั้งค่า Nginx เต็มรูปแบบจะเรียนใน **Phase 11
Deployment & DevOps**):

```nginx
# ตัวอย่าง nginx.conf (เพื่อความเข้าใจภาพรวม จะเจาะลึกใน Phase 11)
server {
    listen 80;
    server_name example.com;

    location /media/ {
        alias /var/www/django-mastery-course/media/;
    }

    location /static/ {
        alias /var/www/django-mastery-course/staticfiles/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;  # ส่งต่อไปยัง Gunicorn ที่รัน Django
    }
}
```

สังเกตว่า Nginx จะ "ดักจับ" request ที่ขึ้นต้นด้วย `/media/` และ `/static/` ไว้ก่อน
serve ไฟล์ตรงจากดิสก์เลย โดยไม่ส่งต่อไปให้ Django process เลยด้วยซ้ำ — นี่คือเหตุผลที่
performance ดีกว่าการให้ Django serve เองมาก

---

## ขั้นตอนที่ 87: WhiteNoise — serve static files ใน production โดยไม่ง้อ Nginx

### 87.1 ปัญหาที่ WhiteNoise แก้ไข

การตั้งค่า Nginx แยกต่างหากสำหรับ serve static files (ตามขั้นตอนที่ 86.4) เป็นวิธีที่ดี
ที่สุดสำหรับระบบขนาดใหญ่ แต่สำหรับโปรเจกต์ขนาดเล็ก-กลาง หรือแพลตฟอร์ม deploy สมัยใหม่
บางแห่ง (เช่น Heroku, Railway, Render) ที่ **ไม่อนุญาตให้ตั้งค่า Nginx เอง** การมีเว็บ
เซิร์ฟเวอร์แยกต่างหากเพื่อ serve แค่ static files อาจซับซ้อนเกินความจำเป็น

**WhiteNoise** คือ library Python ที่ทำให้ **Django application เอง** สามารถ serve
static files ได้อย่างมีประสิทธิภาพเทียบเคียงเว็บเซิร์ฟเวอร์เฉพาะทาง โดยไม่ต้องพึ่ง Nginx
เลย เหมาะมากสำหรับโปรเจกต์ที่ต้องการความเรียบง่ายในการ deploy

### 87.2 ติดตั้ง WhiteNoise

```bash
pip install whitenoise
pip freeze > requirements.txt
```

### 87.3 เพิ่ม WhiteNoise Middleware

จุดที่สำคัญที่สุดคือ **ตำแหน่ง** ของ middleware ใน `MIDDLEWARE` list — ต้องอยู่**ทันทีหลัง**
`SecurityMiddleware` และ**ก่อน**ตัวอื่นทั้งหมด:

```python
# config/settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",   # ← เพิ่มตรงนี้ ทันทีหลัง SecurityMiddleware
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

**ทำไมต้องอยู่ตำแหน่งนี้เป๊ะ?** เพราะเราต้องการให้ WhiteNoise ดักจับ request ที่ขอไฟล์
static **ให้เร็วที่สุดเท่าที่จะทำได้** ก่อนที่ request จะเดินทางผ่าน middleware อื่น ๆ
ที่ไม่จำเป็นสำหรับการ serve ไฟล์นิ่ง ๆ (เช่น session, CSRF check) ซึ่งจะเสียเวลาโดยใช่เหตุ

### 87.4 ตั้งค่า Storage Backend สำหรับ Compression

Django 4.2 ขึ้นไปใช้ setting แบบ dict รวมศูนย์ชื่อ `STORAGES` (แทนที่ `STATICFILES_STORAGE`
แบบเดิมที่ยังใช้ได้แต่ถูก deprecate ไปในทิศทางนี้):

```python
# config/settings.py
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

`CompressedManifestStaticFilesStorage` ของ WhiteNoise ทำสองอย่างพร้อมกัน:

1. **Compression**: สร้างไฟล์ `.gz` (gzip) และ `.br` (Brotli) คู่ไปกับไฟล์ต้นฉบับตอน
   `collectstatic` ทำให้ไฟล์ที่ส่งผ่านเครือข่ายมีขนาดเล็กลงมาก (CSS/JS มักลดขนาดได้
   70-90%)
2. **Cache busting ด้วย Manifest**: เติม hash ต่อท้ายชื่อไฟล์ (เช่น `blog.a1b2c3d4.css`)
   — เราจะอธิบายกลไกนี้แบบละเอียดในขั้นตอนที่ 89

### 87.5 ทดสอบ WhiteNoise แบบจำลอง Production

เพื่อทดสอบว่า WhiteNoise ทำงานถูกต้อง ต้องจำลองสภาพแวดล้อมคล้าย production คือ
`DEBUG = False` (เพราะตอน `DEBUG = True` Django ยังคง serve static ผ่านกลไกเดิมของ
`staticfiles` app อยู่ ทำให้ไม่เห็นผลของ WhiteNoise ชัดเจน):

```bash
# 1. รวมไฟล์ static ทั้งหมดเข้า STATIC_ROOT ก่อน (WhiteNoise อ่านจากที่นี่)
python manage.py collectstatic --noinput

# 2. ตั้ง DEBUG=False ชั่วคราวใน .env (หรือ environment variable) แล้วรัน server
python manage.py runserver
```

ตรวจสอบด้วย Developer Tools ของเบราว์เซอร์ (แท็บ Network) ว่า response header ของไฟล์
CSS/JS มี `Content-Encoding: gzip` หรือ `br` และ `Cache-Control` ที่มีอายุยาว (เช่น
`max-age=31536000` คือ 1 ปี) ปรากฏอยู่ — นี่คือสัญญาณว่า WhiteNoise กำลังทำงาน

### 87.6 ข้อจำกัดของ WhiteNoise ที่ต้องรู้

- **WhiteNoise serve ได้เฉพาะ static files เท่านั้น ไม่รองรับ media files** เพราะ media
  files มีการเปลี่ยนแปลงตลอดเวลา (ผู้ใช้อัปโหลดใหม่เรื่อย ๆ) ในขณะที่ WhiteNoise ถูก
  ออกแบบมาให้อ่านไฟล์เพียงครั้งเดียวตอน server เริ่มทำงาน แล้ว serve จาก memory/cache
  เพื่อความเร็วสูงสุด — media files ยังคงต้องใช้แนวทาง Nginx หรือ Object Storage/CDN
  ตามขั้นตอนที่ 86.4 และ 88
- แม้จะเร็วกว่าการให้ Django serve แบบปกติมาก แต่ยังคง**ช้ากว่า Nginx หรือ CDN โดยตรง**
  เล็กน้อย เพราะยังคงผ่าน Python process อยู่ดี สำหรับเว็บไซต์ทราฟฟิกสูงมาก (เช่น
  หลักล้าน request ต่อวัน) แนะนำให้ใช้ CDN ควบคู่ไปด้วยเสมอ

| ทางเลือก | ความซับซ้อนในการตั้งค่า | Performance | เหมาะกับ |
|---|---|---|---|
| Nginx serve เอง | ปานกลาง-สูง (ต้องดูแล server เอง) | ดีมาก | Self-hosted server, VPS |
| WhiteNoise | ต่ำมาก (แค่ pip install + middleware) | ดี | PaaS อย่าง Heroku/Railway, โปรเจกต์ขนาดเล็ก-กลาง |
| CDN (S3 + CloudFront) | ปานกลาง (ต้องตั้งค่า cloud account) | ดีที่สุด | Production ระดับ enterprise, ทราฟฟิกสูง, ผู้ใช้กระจายทั่วโลก |

---

## ขั้นตอนที่ 88: เกริ่นแนวคิด CDN สำหรับ static/media

### 88.1 CDN คืออะไร และแก้ปัญหาอะไร

**CDN (Content Delivery Network)** คือเครือข่ายเซิร์ฟเวอร์ที่กระจายอยู่ตามจุดต่าง ๆ ทั่วโลก
(เรียกว่า **edge location**) ทำหน้าที่เก็บสำเนาไฟล์ static/media ไว้ใกล้ผู้ใช้มากที่สุด

ลองจินตนาการเว็บไซต์ที่เซิร์ฟเวอร์หลักตั้งอยู่ที่สิงคโปร์ ถ้าผู้ใช้อยู่ที่นิวยอร์ก
ทุกครั้งที่โหลดรูปภาพ ข้อมูลต้องเดินทางข้ามมหาสมุทรแปซิฟิกไปกลับ ทำให้เว็บโหลดช้า
CDN แก้ปัญหานี้โดยเก็บสำเนาไฟล์ไว้ที่ edge location ใกล้นิวยอร์กด้วย — ผู้ใช้จะได้รับ
ไฟล์จากเซิร์ฟเวอร์ที่ใกล้ที่สุดเสมอ โดยไม่ต้องเดินทางไกลถึงสิงคโปร์

```
ไม่มี CDN:
   ผู้ใช้ (นิวยอร์ก) ──────── ระยะทางไกล ────────▶ Server (สิงคโปร์)
                                                    │
                                              โหลดช้า, latency สูง

มี CDN:
   ผู้ใช้ (นิวยอร์ก) ──▶ Edge Location (นิวยอร์ก) ──▶ (sync กับ) Origin Server (สิงคโปร์)
                          │
                    โหลดเร็วมาก, latency ต่ำ
```

### 88.2 สถาปัตยกรรมยอดนิยม: AWS S3 + CloudFront

รูปแบบที่ใช้กันแพร่หลายที่สุดในโลก Django production คือ:

1. **Amazon S3 (Simple Storage Service)**: Object storage ที่เก็บไฟล์ static/media จริง
   (แทนที่จะเก็บบนดิสก์ของเซิร์ฟเวอร์ Django เอง) — มีความทนทานสูงมาก (design สำหรับ
   99.999999999% durability) และรองรับไฟล์ปริมาณมหาศาล
2. **Amazon CloudFront**: บริการ CDN ของ AWS ที่ดึงไฟล์จาก S3 (เรียกว่า **origin**) ไป
   กระจายไว้ตาม edge location ทั่วโลก

```
Browser ──▶ CloudFront (Edge Location ใกล้ผู้ใช้) ──▶ S3 (Origin, เก็บไฟล์จริง)
                  │
            Cache ไฟล์ไว้ที่ edge
            request ถัดไปไม่ต้องวิ่งไปหา S3 ซ้ำ
```

### 88.3 django-storages — ตัวเชื่อม Django กับ Cloud Storage

Django เองไม่มีความสามารถเชื่อมต่อ S3 มาให้ในตัว (ตาม principle "batteries included
เฉพาะสิ่งที่จำเป็นทั่วไป") แต่มี package ยอดนิยมชื่อ **django-storages** ที่ทำหน้าที่นี้
โดยเฉพาะ รองรับทั้ง AWS S3, Google Cloud Storage, Azure Blob Storage และอื่น ๆ

ตัวอย่างภาพรวมการตั้งค่า (แสดงเพื่อให้เห็นภาพรวมเท่านั้น — การตั้งค่าที่ถูกต้องและ
ปลอดภัยแบบสมบูรณ์ พร้อม IAM permissions, bucket policy และ signed URL จะเรียนแบบเต็ม
รูปแบบใน **Phase 11**):

```bash
pip install django-storages boto3
```

```python
# config/settings.py (ภาพรวมคร่าว ๆ เพื่อความเข้าใจ ไม่ใช่การตั้งค่าที่สมบูรณ์)
STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3.S3Storage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}

AWS_STORAGE_BUCKET_NAME = "my-django-app-media"
AWS_S3_REGION_NAME = "ap-southeast-1"
AWS_S3_CUSTOM_DOMAIN = "d1234567890.cloudfront.net"  # CloudFront distribution domain

MEDIA_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/"
```

สังเกตแนวคิดสำคัญ: เมื่อใช้ `django-storages` เป็น `default` storage backend, ทุกครั้งที่
มีการอัปโหลดไฟล์ผ่าน `ImageField`/`FileField` (ตามขั้นตอนที่ 85) Django จะอัปโหลดไฟล์
ขึ้น S3 โดยอัตโนมัติแทนที่จะบันทึกลงดิสก์ของเซิร์ฟเวอร์ — **โค้ดใน views.py และ models.py
ไม่ต้องแก้ไขอะไรเลยแม้แต่บรรทัดเดียว** เพราะ Django ออกแบบ Storage API ให้เป็นนามธรรม
(abstraction) แยกจาก logic ของแอปพลิเคชันโดยสิ้นเชิง — นี่คือพลังของสถาปัตยกรรมที่ดี

### 88.4 ทำไมยังไม่เจาะลึกใน Part นี้

การตั้งค่า S3 + CloudFront ให้ปลอดภัยและถูกต้องจริงในระดับ production เกี่ยวข้องกับ
หลายเรื่องที่ยังไม่ได้เรียนในหลักสูตรนี้ ณ จุดนี้ เช่น IAM Role/Policy, environment
variables สำหรับ credential, bucket policy, CORS configuration, และการจัดการ cost
เราจะกลับมาเจาะลึกเรื่องนี้แบบเต็มรูปแบบพร้อมลงมือทำจริงใน:

- **Phase 11 (Deployment & DevOps)**: การตั้งค่า AWS S3, CloudFront, django-storages
  แบบสมบูรณ์สำหรับ production จริง รวมถึงการเปรียบเทียบกับผู้ให้บริการอื่น เช่น
  Cloudflare R2, DigitalOcean Spaces, Google Cloud Storage

---

## ขั้นตอนที่ 89: Best Practice — Cache Busting ด้วย ManifestStaticFilesStorage

### 89.1 ปัญหา Browser Cache เก่าค้าง

Browser มักจะ **cache** ไฟล์ static ไว้เพื่อความเร็ว (ไม่ต้องโหลดซ้ำทุกครั้ง) แต่นี่
กลายเป็นปัญหาใหญ่เมื่อคุณ deploy โค้ดใหม่: ถ้าคุณแก้ไข `style.css` แล้ว deploy ไป
production ผู้ใช้ที่เคยเข้าเว็บไซต์มาก่อนอาจยังเห็น **ไฟล์ CSS เวอร์ชันเก่า** ที่ browser
cache ไว้ เพราะชื่อไฟล์ (`style.css`) ไม่เปลี่ยน — browser จึงคิดว่าไม่จำเป็นต้องโหลดใหม่

วิธีแก้ปัญหาแบบดั้งเดิมที่หลายคนคุ้นเคยคือการต่อ query string เช่น `style.css?v=2` แต่
วิธีนี้จัดการเองด้วยมือได้ยากและลืมง่าย — Django จึงมีกลไกอัตโนมัติที่ดีกว่ามาก

### 89.2 ManifestStaticFilesStorage คืออะไร

`ManifestStaticFilesStorage` เป็น storage backend ของ Django (มากับ Django เอง ไม่ต้อง
ติดตั้งเพิ่ม) ที่เมื่อรัน `collectstatic` จะ:

1. คำนวณ **hash** จากเนื้อหาไฟล์ (content-based hash)
2. เติม hash นั้นเข้าไปในชื่อไฟล์ เช่น `blog.css` → `blog.a1b2c3d4e5f6.css`
3. สร้างไฟล์ `staticfiles.json` (เรียกว่า **manifest**) ที่เก็บ mapping ระหว่างชื่อไฟล์
   เดิมกับชื่อไฟล์ใหม่ที่มี hash
4. เมื่อ template เรียก `{% static 'blog/css/blog.css' %}` Django จะอ่าน manifest แล้ว
   คืนค่า URL ที่มี hash ให้อัตโนมัติ (`/static/blog/css/blog.a1b2c3d4e5f6.css`)

```
ก่อน deploy:
  blog.css (hash: a1b2c3d4)  →  URL: /static/blog/css/blog.a1b2c3d4.css

แก้ไขเนื้อหาไฟล์ แล้ว deploy ใหม่:
  blog.css (hash: f9e8d7c6)  →  URL: /static/blog/css/blog.f9e8d7c6.css
                                       ▲
                          URL เปลี่ยนไปเพราะเนื้อหาเปลี่ยน
                          browser คิดว่าเป็นไฟล์ใหม่ → บังคับโหลดใหม่ทันที
```

**ประโยชน์คือ**: ไฟล์ที่ **เนื้อหาไม่เปลี่ยน** จะได้ hash เดิม (URL เดิม) — browser ยังคง
ใช้ cache เดิมได้ต่อไปแบบไม่จำกัดอายุ (ตั้ง `Cache-Control: max-age=31536000` ได้อย่าง
มั่นใจ) แต่ไฟล์ที่ **เนื้อหาเปลี่ยน** จะได้ hash ใหม่ (URL ใหม่) บังคับให้ browser โหลด
เวอร์ชันล่าสุดทันทีโดยอัตโนมัติ — ได้ทั้งความเร็ว (cache ยาวนาน) และความถูกต้อง
(ไม่มีปัญหาไฟล์เก่าค้าง) พร้อมกันในคราวเดียว

### 89.3 ตั้งค่า ManifestStaticFilesStorage (แบบไม่ใช้ WhiteNoise)

ถ้าใช้ Nginx serve static files โดยตรง (ไม่ใช้ WhiteNoise) สามารถใช้ตัวนี้ของ Django เอง
ได้ตรง ๆ:

```python
# config/settings.py
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.ManifestStaticFilesStorage",
    },
}
```

### 89.4 รวม Cache Busting กับ WhiteNoise (แนะนำที่สุด)

อย่างที่เห็นในขั้นตอนที่ 87.4 WhiteNoise มีเวอร์ชันของตัวเองที่รวมทั้ง **compression**
และ **cache busting** ไว้ในตัวเดียว ซึ่งเป็นทางเลือกที่ครบเครื่องที่สุด:

```python
# config/settings.py
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

`CompressedManifestStaticFilesStorage` = `ManifestStaticFilesStorage` (ของ Django) +
compression (gzip/brotli) ของ WhiteNoise รวมกันในคลาสเดียว — นี่คือ **การตั้งค่าที่
แนะนำที่สุดสำหรับโปรเจกต์ที่ใช้ WhiteNoise**

### 89.5 ทดสอบผลลัพธ์จริง

```bash
python manage.py collectstatic --noinput
ls staticfiles/blog/css/
```

ผลลัพธ์ที่ควรเห็น:

```
blog.a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6.css
blog.a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6.css.gz
staticfiles.json          (ไฟล์ manifest — อยู่ที่ root ของ STATIC_ROOT)
```

เปิดไฟล์ `staticfiles.json` (อยู่ที่ root ของ `STATIC_ROOT` ไม่ใช่ใน `blog/`) จะเห็นโครงสร้าง
mapping ประมาณนี้:

```json
{
    "paths": {
        "blog/css/blog.css": "blog/css/blog.a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6.css",
        "css/base.css": "css/base.9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c.css"
    },
    "version": "1.1"
}
```

**ข้อควรระวังสำคัญ**: หลังจากเปิดใช้ `ManifestStaticFilesStorage` (หรือเวอร์ชันของ
WhiteNoise) **ต้องรัน `collectstatic` ทุกครั้งที่ deploy เสมอ ห้ามลืมเด็ดขาด** เพราะถ้า
ไม่รัน `{% static %}` tag จะหา entry ใน manifest ไม่เจอและทำให้เกิด error
`ValueError: Missing staticfiles manifest entry` ทันทีตอน render template — Django
จงใจออกแบบให้ error แบบนี้ชัดเจน (fail loudly) แทนที่จะปล่อยผ่านเงียบ ๆ เพื่อป้องกันไม่ให้
production มีลิงก์ static files เสียโดยไม่รู้ตัว

### 89.6 สรุปตารางเปรียบเทียบ Storage Backend ทั้งหมดที่กล่าวถึงใน Part นี้

| Storage Backend | Compression | Cache Busting | ต้องติดตั้งเพิ่ม | เหมาะกับ |
|---|---|---|---|---|
| `StaticFilesStorage` (default) | ❌ | ❌ | ไม่ต้อง | Development เท่านั้น |
| `ManifestStaticFilesStorage` | ❌ | ✅ | ไม่ต้อง (มากับ Django) | ใช้ร่วมกับ Nginx serve เอง |
| `CompressedManifestStaticFilesStorage` (WhiteNoise) | ✅ | ✅ | `pip install whitenoise` | Production ที่ไม่มี Nginx (เช่น Heroku/Railway) |
| `S3Storage` (django-storages) | ขึ้นกับ config CDN | ผ่าน hash ใน key ได้ | `pip install django-storages boto3` | Production ระดับ enterprise ร่วมกับ CDN |

---

## ขั้นตอนที่ 90: สรุปและแบบฝึกหัด

### 90.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจความแตกต่างระหว่าง Static Files (เขียนไว้ล่วงหน้าโดยนักพัฒนา) กับ Media Files
  (ผู้ใช้อัปโหลดขณะใช้งานจริง)
- ✅ ตั้งค่า `STATIC_URL`, `STATICFILES_DIRS` และเข้าใจ convention การซ้อนโฟลเดอร์ชื่อแอป
  ใน `static/<app_name>/` เพื่อป้องกันชื่อไฟล์ชนกัน
- ✅ เข้าใจว่าทำไม production ต้องรวมไฟล์ static ด้วยคำสั่ง `collectstatic` ไปไว้ที่
  `STATIC_ROOT` ที่เดียว แทนการปล่อยให้กระจัดกระจายเหมือนตอนพัฒนา
- ✅ จัดการ CSS/JS/Images จริงในแอป `blog` และเชื่อมต่อเข้ากับ template ที่เขียนไว้ใน
  Part 008 ผ่าน `{% static %}` tag
- ✅ ตั้งค่า `MEDIA_URL`, `MEDIA_ROOT` และเข้าใจว่าทำไม media files ห้าม commit เข้า Git
- ✅ เกริ่น `ImageField`/`FileField` และรู้ว่าต้องติดตั้ง Pillow ก่อนใช้ `ImageField`
- ✅ ใช้ `django.conf.urls.static.static()` เพื่อ serve media files ตอนพัฒนา และเข้าใจว่า
  production ต้องใช้ Nginx หรือ Object Storage/CDN แทน
- ✅ ติดตั้งและตั้งค่า **WhiteNoise** เพื่อ serve static files ใน production โดยไม่ต้อง
  พึ่ง Nginx พร้อมเปิดใช้ compression
- ✅ เข้าใจภาพรวมแนวคิด CDN, AWS S3 + CloudFront และบทบาทของ `django-storages`
- ✅ ใช้ `ManifestStaticFilesStorage` (หรือเวอร์ชันของ WhiteNoise) เพื่อทำ cache busting
  อัตโนมัติด้วย content hash

### 90.2 Checklist ก่อนไป Part ถัดไป

- [ ] มีโฟลเดอร์ `blog/static/blog/css/`, `js/`, `images/` พร้อมไฟล์ CSS/JS จริงที่ใช้งานได้
- [ ] มีโฟลเดอร์ `static/` ระดับโปรเจกต์ และตั้งค่า `STATICFILES_DIRS` ชี้ไปถูกต้อง
- [ ] ตั้งค่า `STATIC_ROOT` และรันคำสั่ง `python manage.py collectstatic` สำเร็จ เห็นโฟลเดอร์
      `staticfiles/` ถูกสร้างขึ้นจริง
- [ ] `base.html` โหลด CSS/JS ผ่าน `{% static %}` tag ถูกต้อง ไม่มีการ hardcode URL
- [ ] ตั้งค่า `MEDIA_URL`, `MEDIA_ROOT` และเพิ่ม `media/` ใน `.gitignore` แล้ว
- [ ] เพิ่ม `ImageField` ในตัวอย่าง Model และติดตั้ง `Pillow` สำเร็จ
- [ ] เพิ่ม `static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)` ใน
      `config/urls.py` และทดสอบเปิดไฟล์ media ผ่าน browser ได้จริง
- [ ] ติดตั้ง WhiteNoise, เพิ่ม middleware ในตำแหน่งที่ถูกต้อง และตั้งค่า `STORAGES` แล้ว
- [ ] ทดสอบรัน `collectstatic` แล้วเห็นไฟล์ที่มี hash ต่อท้ายชื่อ (cache busting ทำงาน)

### 90.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้างแอปใหม่ชื่อ `pages` (ถ้ายังไม่มี) แล้วสร้างโฟลเดอร์
`pages/static/pages/css/about.css` พร้อมเขียน CSS จริงอย่างน้อย 5 กฎ (rule) จากนั้น
สร้าง template `pages/templates/pages/about.html` ที่โหลด CSS นี้ผ่าน `{% static %}`
tag และทดสอบเปิดในเบราว์เซอร์ว่าสไตล์ถูกนำไปใช้จริง

**แบบฝึกหัดที่ 2**: เพิ่มฟิลด์ `avatar = models.ImageField(upload_to="accounts/avatars/",
blank=True, null=True)` ลงใน Model ใด ๆ ที่คุณมี (หรือสร้าง Model ทดสอบชื่อ `Profile`
ขึ้นมาใหม่) ติดตั้ง Pillow ให้เรียบร้อย แล้วใช้ `python manage.py shell` ทดลองสร้าง
instance พร้อมกำหนดค่าฟิลด์นี้ด้วยไฟล์รูปภาพจริง (ใช้ `django.core.files.File` ช่วย)
จากนั้นตรวจสอบว่าไฟล์ถูกบันทึกลงในโฟลเดอร์ `media/accounts/avatars/` จริงหรือไม่

**แบบฝึกหัดที่ 3**: ตั้งค่า `STORAGES["staticfiles"]` ให้ใช้
`whitenoise.storage.CompressedManifestStaticFilesStorage` ตามที่เรียนในขั้นตอนที่ 87-89
แล้วรัน `collectstatic` เปรียบเทียบขนาดไฟล์ก่อนและหลัง compression (เทียบขนาดไฟล์ `.css`
ต้นฉบับกับไฟล์ `.css.gz` ที่ถูกสร้างขึ้น) บันทึกเปอร์เซ็นต์ที่ลดลงได้

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ทดลองตั้ง `DEBUG = False` และ `ALLOWED_HOSTS =
["127.0.0.1", "localhost"]` ชั่วคราวในเครื่องของคุณ (จำลอง production) แล้วสังเกตว่า
ถ้า**ไม่ได้**รัน `collectstatic` ก่อน จะเกิด error อะไรขึ้นเมื่อเปิดหน้าเว็บที่มี
`{% static %}` tag พร้อม `ManifestStaticFilesStorage` เปิดใช้งานอยู่ อธิบายด้วยคำพูด
ของตัวเองว่าทำไม Django ถึงออกแบบให้ error แบบนี้เกิดขึ้นอย่างชัดเจนแทนที่จะปล่อยผ่าน
เงียบ ๆ (อย่าลืมตั้งค่ากลับเป็น `DEBUG = True` หลังทดลองเสร็จ)

### 90.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมโฟลเดอร์ static/ ต้องซ้อนชื่อแอปอีกชั้น (`blog/static/blog/...`) ทั้งที่ดูซ้ำซ้อน?**
A: เพื่อป้องกันไฟล์ชื่อเดียวกันจากคนละแอปทับกันตอนรัน `collectstatic` (เช่น `style.css`
ของแอป `blog` กับ `shop`) หลักการเดียวกับที่ template ต้องซ้อนโฟลเดอร์ชื่อแอปที่เรียนใน
Part 008 แม้จะดูซ้ำซ้อนตอนโปรเจกต์เล็ก แต่จำเป็นมากเมื่อโปรเจกต์โตขึ้นมีหลายสิบแอป

**Q: ถ้าลืมรัน collectstatic ก่อน deploy จะเกิดอะไรขึ้น?**
A: ขึ้นอยู่กับ storage backend ที่ใช้ ถ้าใช้ `ManifestStaticFilesStorage` หรือเวอร์ชันของ
WhiteNoise จะเกิด error `ValueError: Missing staticfiles manifest entry` ทันทีตอน
render template (fail loudly ตามที่อธิบายในขั้นตอนที่ 89.5) แต่ถ้าใช้ storage backend
ธรรมดา เว็บไซต์อาจโหลดได้แต่ **ไม่มี CSS/JS เลย** เพราะไฟล์ยังไม่ถูกรวมไปไว้ที่
`STATIC_ROOT` ที่ Nginx/WhiteNoise คาดหวังว่าจะเจอ

**Q: ใช้ WhiteNoise ไปแล้ว ยังจำเป็นต้องใช้ CDN อีกหรือไม่?**
A: WhiteNoise เหมาะกับโปรเจกต์ขนาดเล็ก-กลางที่ deploy ง่าย ไม่ต้องดูแล infrastructure
เยอะ แต่ถ้าเว็บไซต์มีผู้ใช้กระจายอยู่หลายทวีป หรือมีทราฟฟิกสูงมาก การเพิ่ม CDN (เช่น
Cloudflare วางไว้หน้า WhiteNoise) จะช่วยลด latency ให้ผู้ใช้ที่อยู่ไกลจากเซิร์ฟเวอร์หลัก
ได้อีกมาก — ทั้งสองใช้ร่วมกันได้ ไม่ขัดแย้งกัน

**Q: ควรเก็บ media files ไว้บนดิสก์ของเซิร์ฟเวอร์เองไปตลอดหรือไม่?**
A: สำหรับโปรเจกต์ตอนเรียนหรือ MVP ขนาดเล็กที่มีเซิร์ฟเวอร์เดียว การเก็บบนดิสก์ (ตามที่
เรียนในขั้นตอนที่ 84-86) ก็เพียงพอ แต่เมื่อระบบขยายเป็นหลายเซิร์ฟเวอร์ (horizontal
scaling ที่จะเรียนใน Phase 11-12) การเก็บไฟล์บนดิสก์ของแต่ละเซิร์ฟเวอร์จะกลายเป็นปัญหา
ทันที เพราะไฟล์ที่อัปโหลดผ่าน server A จะไม่ปรากฏบน server B — จุดนี้คือเหตุผลสำคัญที่
ต้องย้ายไปใช้ Object Storage อย่าง S3 ตามขั้นตอนที่ 88 ซึ่งทุกเซิร์ฟเวอร์เข้าถึงไฟล์
ชุดเดียวกันได้พร้อมกัน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 010: Django Settings และ Environment Configuration** จะพาคุณกลับไปเจาะลึกไฟล์
`config/settings.py` อีกครั้งแบบเต็มรูปแบบ คราวนี้ในมุมของการจัดการ **environment
หลายแบบ** (development, staging, production) อย่างมืออาชีพ เรียนรู้การแยก settings
เป็นหลายไฟล์ (`settings/base.py`, `settings/dev.py`, `settings/production.py`) การใช้
`django-environ` เจาะลึกกว่าที่เห็นใน Part 004 การจัดการ secret ด้วย environment
variables อย่างปลอดภัย และเตรียมความพร้อมของ settings ทั้งหมดที่เราตั้งค่าไว้ใน Part นี้
(`STATIC_ROOT`, `MEDIA_ROOT`, `STORAGES`) ให้พร้อมสลับค่าตาม environment ได้อย่างถูกต้อง
ก่อนที่ Part 011 จะพาไปเจาะลึกโลกของ Django Models และ Migrations อย่างจริงจังใน
Phase 2

เตรียมพร้อม `config/settings.py` ของคุณไว้ให้ดี เพราะ Part ถัดไปเราจะแปลงมันให้เป็น
ระบบที่ยืดหยุ่นและปลอดภัยสำหรับ production จริง!
