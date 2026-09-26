# Part 051: Django กับ Bootstrap และ CSS Framework

> **ขั้นตอนที่ 501-510 ของหลักสูตร** | Phase 6: Frontend Integration (Part แรกของ Phase)
>
> เป้าหมายของ Part นี้: เปลี่ยนหน้าเว็บของ `blog` ที่ยังใช้ CSS มือเขียนเองจาก
> Part 008 ให้กลายเป็นหน้าเว็บที่ดูเป็นมืออาชีพระดับ production ด้วย
> **Bootstrap 5** ตั้งแต่การเชื่อมผ่าน CDN หรือ npm, การ render ฟอร์ม Django ให้
> สวยงามอัตโนมัติด้วย `django-crispy-forms` + `crispy-bootstrap5`, ระบบ Grid,
> Component จริง (navbar, card, modal), `django-widget-tweaks` สำหรับฟอร์มที่ไม่ได้
> ใช้ crispy, หลักการ Responsive Design, การ override สีธีมด้วย Sass, ไปจนถึงการ
> รู้จัก Tailwind CSS ผ่าน `django-tailwind` เป็นทางเลือก เมื่อจบ Part นี้
> `base.html`, `blog/list.html`, `blog/detail.html` และฟอร์มทั้งหมดของ `blog`
> จะกลายเป็นหน้าเว็บ Bootstrap 5 เต็มรูปแบบที่ responsive และพร้อมใช้งานจริง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 501: เชื่อม Bootstrap 5 เข้า Django ผ่าน CDN เทียบกับติดตั้งผ่าน npm/django-compressor
- ขั้นตอนที่ 502: ติดตั้ง `django-crispy-forms` + `crispy-bootstrap5` เพื่อ render ฟอร์ม Django ให้สวยงามอัตโนมัติ
- ขั้นตอนที่ 503: ระบบ Grid ของ Bootstrap ใน Django Template (container, row, col)
- ขั้นตอนที่ 504: นำ Bootstrap Component มาใช้จริง (navbar, card, modal) ในหน้า blog
- ขั้นตอนที่ 505: `django-widget-tweaks` สำหรับเติม class Bootstrap เข้า field ฟอร์มที่ไม่ได้ใช้ crispy-forms
- ขั้นตอนที่ 506: หลักการ Responsive Design และการทดสอบบนขนาดจอต่าง ๆ
- ขั้นตอนที่ 507: Custom CSS override ตัวแปร Bootstrap ด้วย Sass (เปลี่ยนสีธีมของเว็บ)
- ขั้นตอนที่ 508: เกริ่น Tailwind CSS เป็นทางเลือก พร้อมติดตั้งผ่าน `django-tailwind`
- ขั้นตอนที่ 509: ตารางเปรียบเทียบ Bootstrap vs Tailwind vs Bulma — เมื่อไหร่ควรเลือกอะไร
- ขั้นตอนที่ 510: สรุปและแบบฝึกหัด — ทำ UI บล็อกทั้งหมดให้สวยงามด้วย Bootstrap 5

---

## ขั้นตอนที่ 501: เชื่อม Bootstrap 5 เข้า Django ผ่าน CDN เทียบกับติดตั้งผ่าน npm/django-compressor

### 501.1 ทบทวนสถานะปัจจุบันของ `blog`: CSS มือเขียนเองจาก Part 008

ย้อนกลับไปที่ Part 008 คุณสร้าง `blog/static/blog/css/style.css` ด้วยมือ กำหนด
`--color-primary`, จัดหน้า `.navbar` เอง ฯลฯ วิธีนี้ใช้ได้ดีตอนเรียนรู้พื้นฐาน
CSS แต่เมื่อโปรเจกต์โตขึ้น การเขียน CSS เองทั้งหมด (grid ที่ responsive, ปุ่มที่
มีสถานะ hover/focus/disabled ครบ, modal, dropdown, tooltip ฯลฯ) ใช้เวลามหาศาล
และเสี่ยงบั๊กเรื่อง cross-browser compatibility

**CSS Framework** คือชุด CSS (และมักมี JavaScript ประกอบ) ที่เขียน component
และ layout system สำเร็จรูปไว้ให้แล้ว **Bootstrap** คือ CSS Framework ที่ได้รับ
ความนิยมสูงสุดในโลกมาอย่างต่อเนื่องตั้งแต่ปี 2011 (พัฒนาโดยทีมงาน Twitter เดิม)
เวอร์ชันล่าสุดที่ใช้ในหลักสูตรนี้คือ **Bootstrap 5.3.x** ซึ่งตัดการพึ่งพา jQuery
ออกไปแล้วทั้งหมด (Bootstrap 4 และก่อนหน้ายังต้องใช้ jQuery) ทำให้เบาและทันสมัย
กว่าเดิมมาก

### 501.2 สองวิธีหลักในการติดตั้ง Bootstrap เข้า Django

| วิธี | อธิบาย |
|---|---|
| **CDN (Content Delivery Network)** | แปะ `<link>`/`<script>` ที่ชี้ไปยังไฟล์ CSS/JS ของ Bootstrap ที่โฮสต์อยู่บนเซิร์ฟเวอร์สาธารณะ (เช่น jsDelivr) ไม่ต้องติดตั้งอะไรในเครื่องเลย |
| **npm + django-compressor** | ติดตั้ง Bootstrap เป็น dependency ผ่าน `npm`, เก็บไฟล์ไว้ใน `node_modules/`, แล้วใช้ `django-compressor` รวม/บีบอัดไฟล์ static ตอน build เพื่อควบคุมเวอร์ชันและปรับแต่ง source code ได้เต็มที่ |

### 501.3 วิธีที่ 1: เชื่อมผ่าน CDN (เร็วที่สุด เหมาะกับการเรียนรู้และโปรเจกต์เล็ก)

แก้ไข `templates/base.html` ที่สร้างไว้ใน Part 008 เพิ่ม Bootstrap CSS ใน
`<head>` และ Bootstrap JS bundle (มี Popper.js รวมอยู่แล้ว สำหรับ dropdown/
tooltip/popover) ก่อนปิด `</body>`:

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>

    <!-- Bootstrap 5.3 CSS ผ่าน CDN (jsDelivr) -->
    <link
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
        rel="stylesheet"
        integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH"
        crossorigin="anonymous">

    <!-- CSS ของเราเอง โหลดทีหลัง Bootstrap เสมอ เพื่อให้ override ได้ -->
    <link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
    {% block extra_head %}{% endblock %}
</head>
<body>
    {% include 'partials/navbar.html' %}

    <main class="container my-4">
        {% block content %}
        <p>ยังไม่มีเนื้อหา</p>
        {% endblock %}
    </main>

    {% include 'partials/footer.html' %}

    <!-- Bootstrap Bundle (รวม Popper.js) วางท้ายสุดของ body เสมอ -->
    <script
        src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz"
        crossorigin="anonymous"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

**สังเกตประเด็นสำคัญ 3 จุด:**

1. **CSS ของเราเองต้องโหลด *หลัง* Bootstrap เสมอ** เพราะ CSS ที่โหลดทีหลังจะมี
   priority ในการ override สูงกว่าเมื่อ specificity เท่ากัน (concept นี้เรียกว่า
   **Cascade** — ตัว "C" ใน CSS ที่ย่อมาจาก Cascading Style Sheets)
2. **`integrity` และ `crossorigin`** คือกลไก **Subresource Integrity (SRI)** —
   เบราว์เซอร์จะคำนวณ hash ของไฟล์ที่โหลดมาจาก CDN แล้วเทียบกับค่าที่ระบุใน
   `integrity` ถ้าไม่ตรงกัน (เช่น CDN ถูกแฮ็กแล้วสลับไฟล์) เบราว์เซอร์จะปฏิเสธ
   ไม่รันไฟล์นั้นทันที นี่คือมาตรการความปลอดภัยที่**ควรใส่เสมอ**เมื่อโหลด
   ทรัพยากรจากบุคคลที่สาม
3. **Bootstrap JS วางท้าย `<body>`** เพื่อไม่ให้บล็อกการ render หน้าเว็บระหว่าง
   รอโหลดสคริปต์ (หลักการเดียวกับที่แนะนำให้วาง `<script>` ทั่วไปไว้ท้ายหน้า)

### 501.4 ข้อดี-ข้อเสียของ CDN

| ข้อดี | ข้อเสีย |
|---|---|
| ติดตั้งเร็วที่สุด แค่ก็อปวาง `<link>`/`<script>` | ต้องพึ่งพาอินเทอร์เน็ตและความเสถียรของ CDN ภายนอก |
| ผู้ใช้อาจมีไฟล์แคชไว้แล้วจากเว็บอื่นที่ใช้ CDN เดียวกัน (browser cache ร่วม) | ไม่สามารถแก้ไข source code ของ Bootstrap (SCSS variables) ได้เลย |
| ไม่เพิ่มขนาดโปรเจกต์ ไม่ต้องมี `node_modules/` | เสี่ยงเรื่อง privacy/compliance บางองค์กร (ข้อมูล IP ผู้ใช้ถูกส่งไปยัง third-party CDN) |
| เหมาะกับการเรียนรู้ prototype และเว็บไซต์ขนาดเล็ก-กลาง | เว็บที่ต้อง deploy แบบ offline/intranet ใช้ CDN ไม่ได้เลย |

### 501.5 วิธีที่ 2: ติดตั้งผ่าน npm + django-compressor

สำหรับโปรเจกต์ระดับ production ที่ต้องการควบคุมเวอร์ชันแบบ pin ชัดเจน ปรับแต่ง
Sass variables (จะเจาะลึกในขั้นตอนที่ 507) และรวมไฟล์ CSS/JS ทั้งหมดให้เหลือ
ไฟล์เดียว (ลดจำนวน HTTP request) นิยมติดตั้งผ่าน `npm` แล้วประมวลผลด้วย
`django-compressor`

```bash
# ติดตั้ง Node.js ก่อน (ถ้ายังไม่มี) แล้วสร้าง package.json
npm init -y
npm install bootstrap@5.3.3 @popperjs/core
```

ติดตั้ง `django-compressor` ฝั่ง Python:

```bash
pip install django-compressor
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'compressor',
]

STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    'compressor.finders.CompressorFinder',
]

COMPRESS_ENABLED = True          # เปิดใช้งานแม้ตอน DEBUG=True เพื่อทดสอบ (ปกติปิดตอน dev)
COMPRESS_ROOT = STATIC_ROOT      # ทบทวนจาก Part 009
```

ใน template ใช้ tag `{% compress %}` ครอบไฟล์ CSS/JS ที่ต้องการให้รวมและบีบอัด:

```html
{% load compress static %}

{% compress css %}
<link rel="stylesheet" href="{% static 'vendor/bootstrap/dist/css/bootstrap.css' %}">
<link rel="stylesheet" href="{% static 'blog/css/style.css' %}">
{% endcompress %}

{% compress js %}
<script src="{% static 'vendor/bootstrap/dist/js/bootstrap.bundle.js' %}"></script>
{% endcompress %}
```

`django-compressor` จะรวมไฟล์ทั้งหมดใน block เดียวกันให้เหลือไฟล์เดียว, minify
ให้อัตโนมัติ, และตั้งชื่อไฟล์ด้วย hash ของเนื้อหา (cache busting — เมื่อเนื้อหา
เปลี่ยน ชื่อไฟล์เปลี่ยนตาม บังคับให้เบราว์เซอร์โหลดไฟล์ใหม่แทนที่จะใช้แคชเก่า)

> **หมายเหตุ**: ต้อง copy หรือ symlink ไฟล์จาก `node_modules/bootstrap/dist/`
> เข้ามาในโฟลเดอร์ `static/vendor/bootstrap/` ของโปรเจกต์ก่อน เพราะ Django
> staticfiles ไม่รู้จัก `node_modules/` โดยตรง ทีมงานจริงมักเขียน npm script
> `"copy-vendor"` ให้ทำหน้าที่นี้อัตโนมัติหลังจาก `npm install`

### 501.6 ตารางเปรียบเทียบสองแนวทาง

| ประเด็น | CDN | npm + django-compressor |
|---|---|---|
| ความเร็วในการติดตั้ง | เร็วที่สุด (2 บรรทัด) | ช้ากว่า ต้องตั้งค่าหลายจุด |
| ควบคุมเวอร์ชันแบบ pin แน่นอน | ทำได้ (ระบุเลขเวอร์ชันใน URL) แต่แก้ไข source ไม่ได้ | ทำได้เต็มรูปแบบ ผ่าน `package.json` |
| ปรับแต่ง Sass variables (สีธีม ฯลฯ) | **ทำไม่ได้** | ทำได้ (ขั้นตอนที่ 507) |
| ทำงานแบบ offline/intranet ได้ | ไม่ได้ | ได้ |
| จำนวน HTTP request | มากกว่า (แยกไฟล์ Bootstrap ต่างหาก) | น้อยกว่า (รวมไฟล์เป็นก้อนเดียว) |
| ความซับซ้อนของ build pipeline | ไม่มีเลย | ต้องดูแล `npm install`, compressor cache |
| เหมาะกับ | การเรียนรู้, prototype, เว็บเล็ก-กลาง | โปรเจกต์ production ที่ต้อง custom ธีมหรือควบคุม asset เต็มรูปแบบ |

**แนวทางของหลักสูตรนี้**: ใช้ **CDN** ตลอด Part 051-052 เพื่อโฟกัสที่การเรียนรู้
Component และ Layout ของ Bootstrap ก่อน แล้วค่อยเปลี่ยนไปใช้แนวทาง npm + Sass
compilation ในขั้นตอนที่ 507 เมื่อต้องการ custom สีธีมจริง ๆ — วิธีคิดแบบนี้
สะท้อนการทำงานจริงที่ทีมส่วนใหญ่เริ่มจาก CDN ตอน prototype แล้วค่อยย้ายมาที่
build pipeline เมื่อโปรเจกต์เข้าสู่ช่วง production

---

## ขั้นตอนที่ 502: ติดตั้ง `django-crispy-forms` + `crispy-bootstrap5`

### 502.1 ปัญหาของการ render ฟอร์ม Django แบบ Bootstrap ด้วยมือ

ทบทวนจาก Part 025: `{{ form.as_p }}` หรือ `{{ form.as_div }}` ให้ HTML ที่ไม่มี
class ของ Bootstrap เลย (`<input>` จะไม่มี `class="form-control"` ติดมาให้)
ทำให้ต้องเขียน widget attrs `class` ด้วยมือทุก field ทุกฟอร์ม (แบบที่ Part 025
ข้อ 244.3 สาธิตไว้) ซึ่งซ้ำซากและลืมง่าย

**`django-crispy-forms`** คือ third-party package ที่ทำให้ Django form
render ออกมาเป็น HTML ที่ตรงตาม CSS Framework ที่เลือก (Bootstrap, Bulma,
Tailwind ฯลฯ) โดยอัตโนมัติ **โดยไม่ต้องแก้ widget attrs ในฟอร์มเลยแม้แต่บรรทัด
เดียว**

### 502.2 ติดตั้ง

```bash
pip install django-crispy-forms crispy-bootstrap5
pip freeze > requirements.txt
```

`crispy-bootstrap5` คือ **template pack** แยกต่างหากที่บอก `django-crispy-forms`
ว่าจะ render ออกมาเป็นสไตล์ Bootstrap 5 (ตั้งแต่ crispy-forms เวอร์ชัน 2.0
เป็นต้นมา template pack ของแต่ละ framework ถูกแยกออกมาเป็น package ของตัวเอง
ไม่รวมมากับ core package แล้ว)

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'crispy_forms',
    'crispy_bootstrap5',
]

CRISPY_ALLOWED_TEMPLATE_PACKS = "bootstrap5"
CRISPY_TEMPLATE_PACK = "bootstrap5"
```

### 502.3 วิธีที่ง่ายที่สุด: filter `|crispy`

```html
<!-- blog/templates/blog/contact.html -->
{% extends 'base.html' %}
{% load crispy_forms_tags %}

{% block title %}ติดต่อเรา | Django Mastery Blog{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-8 col-lg-6">
        <h1 class="mb-4">ติดต่อเรา</h1>
        <form method="post">
            {% csrf_token %}
            {{ form|crispy }}
            <button type="submit" class="btn btn-primary mt-3">ส่งข้อความ</button>
        </form>
    </div>
</div>
{% endblock %}
```

`ContactForm` จาก Part 025 (ไม่ต้องแก้โค้ดในฟอร์มเลย) จะถูก render เป็น
`<div class="mb-3">` ครอบแต่ละ field, `<label class="form-label">`,
`<input class="form-control">` และแสดง error ด้วย `<div class="invalid-feedback">`
ของ Bootstrap โดยอัตโนมัติทั้งหมด — ประหยัดเวลาได้มหาศาลเมื่อเทียบกับการเขียน
`widget=forms.TextInput(attrs={"class": "form-control"})` ทุก field ทุกฟอร์ม

### 502.4 ควบคุม Layout ด้วย `FormHelper` และ `Layout`

เมื่อต้องการจัด field เป็นหลายคอลัมน์ ใส่ปุ่ม submit ที่มี class เฉพาะ หรือจัด
กลุ่ม field เป็น section ให้ใช้ `FormHelper` ผูกเข้ากับฟอร์มโดยตรงในไฟล์
`forms.py`:

```python
# blog/forms.py
from django import forms
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Layout, Row, Column, Submit, Fieldset, HTML


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, label="ชื่อของคุณ")
    email = forms.EmailField(label="อีเมล")
    subject = forms.CharField(max_length=150, label="หัวข้อ")
    message = forms.CharField(widget=forms.Textarea(attrs={"rows": 5}), label="ข้อความ")
    newsletter_opt_in = forms.BooleanField(required=False, label="สมัครรับจดหมายข่าว")

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.form_method = "post"
        self.helper.layout = Layout(
            Fieldset(
                "ข้อมูลผู้ติดต่อ",
                Row(
                    Column("name", css_class="col-md-6"),
                    Column("email", css_class="col-md-6"),
                ),
            ),
            Fieldset(
                "รายละเอียด",
                "subject",
                "message",
                "newsletter_opt_in",
            ),
            HTML("<hr class='my-3'>"),
            Submit("submit", "ส่งข้อความ", css_class="btn btn-primary px-4"),
        )
```

จากนั้นในเทมเพลตเรียกด้วย tag `{% crispy %}` แทนการเขียน `<form>` เอง (เพราะ
`FormHelper` จัดการทั้ง `<form>` tag, `{% csrf_token %}`, และปุ่ม submit ให้
ครบแล้ว):

```html
<!-- blog/templates/blog/contact.html -->
{% extends 'base.html' %}
{% load crispy_forms_tags %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-8 col-lg-6">
        <h1 class="mb-4">ติดต่อเรา</h1>
        {% crispy form %}
    </div>
</div>
{% endblock %}
```

`{% crispy form %}` ให้ผลลัพธ์เป็นฟอร์มที่มี `name`/`email` อยู่คนละคอลัมน์บน
จอกว้าง (ใช้ Bootstrap grid ภายใน) แต่จะซ้อนกันเป็นแถวเดียวอัตโนมัติบนจอมือถือ
— นี่คือตัวอย่างแรกของแนวคิด **Responsive Design** ที่จะเจาะลึกในขั้นตอนที่ 506

### 502.5 `|crispy` filter เทียบกับ `{% crispy %}` tag

| | `{{ form|crispy }}` | `{% crispy form %}` |
|---|---|---|
| ต้องเขียน `<form>`, `{% csrf_token %}` เองไหม | ต้อง | ไม่ต้อง (`FormHelper` จัดการให้) |
| ต้องมี `self.helper` ในฟอร์มไหม | ไม่บังคับ (ถ้าไม่มีจะใช้ layout เริ่มต้น) | แนะนำให้มีเสมอ เพื่อคุม layout/ปุ่ม |
| ควบคุม layout ซับซ้อน (หลายคอลัมน์, section) | ทำไม่ได้ | ทำได้เต็มรูปแบบผ่าน `Layout()` |
| เหมาะกับ | ฟอร์มง่าย ๆ ที่ไม่ต้องจัด layout พิเศษ | ฟอร์มที่ต้องการควบคุมหน้าตาอย่างละเอียด |

### 502.6 แสดง Error และ `non_field_errors` ด้วย crispy

crispy-forms จัดการแสดง error ให้ทั้งหมดโดยอัตโนมัติตามมาตรฐาน Bootstrap
validation states (`is-invalid`, `invalid-feedback`) ทดสอบได้ทันทีโดยส่งข้อมูล
ผิดเข้าฟอร์ม:

```python
>>> from blog.forms import ContactForm
>>> form = ContactForm(data={"name": "", "email": "bad-email"})
>>> form.is_valid()
False
```

เมื่อ render ผ่าน `{% crispy form %}` field `name` และ `email` จะมีกรอบสีแดง
(`.is-invalid`) พร้อมข้อความ error ใต้ field นั้นทันที **โดยไม่ต้องเขียน
`{% if field.errors %}` เองเลย** ต่างจาก Part 025 ที่ต้องเขียนเงื่อนไขแสดง error
เองทุกจุด

---

## ขั้นตอนที่ 503: ระบบ Grid ของ Bootstrap ใน Django Template (container, row, col)

### 503.1 หลักการพื้นฐานของ Bootstrap Grid: 12-column layout

Bootstrap Grid แบ่งความกว้างของหน้าจอออกเป็น **12 ส่วนเท่า ๆ กัน (columns)**
เสมอ ไม่ว่าจอจะกว้างแค่ไหน คุณกำหนดว่าแต่ละ element จะกิน "กี่ส่วนจาก 12" ผ่าน
class เช่น `col-6` (กิน 6/12 = ครึ่งหนึ่งของแถว) หรือ `col-4` (กิน 4/12 = หนึ่งใน
สามของแถว)

โครงสร้างต้องมี 3 ชั้นเสมอ:

```
.container (หรือ .container-fluid)
    └── .row
            └── .col-* (หนึ่งตัวหรือมากกว่า รวมกันไม่เกิน 12 ต่อแถว)
```

### 503.2 `.container` vs `.container-fluid`

| Class | พฤติกรรม |
|---|---|
| `.container` | มีความกว้างสูงสุดคงที่ (max-width) ที่เปลี่ยนตาม breakpoint และมี margin ซ้าย-ขวาอัตโนมัติเพื่อจัดกึ่งกลาง — เหมาะกับเนื้อหาทั่วไปที่ไม่ต้องการเต็มจอ |
| `.container-fluid` | กว้างเต็ม 100% ของหน้าจอเสมอ ไม่มี max-width — เหมาะกับ dashboard หรือหน้าที่ต้องการใช้พื้นที่เต็มจอ |
| `.container-{breakpoint}` | เช่น `.container-lg` จะเป็น fluid (เต็มจอ) จนกว่าจะถึง breakpoint `lg` แล้วค่อยกลายเป็น fixed-width — ใช้น้อยแต่มีประโยชน์เฉพาะกรณี |

`base.html` ที่แก้ในขั้นตอนที่ 501 ใช้ `<main class="container my-4">` แล้ว
(fixed-width, จัดกึ่งกลาง, มี margin บน-ล่าง `my-4`)

### 503.3 ตาราง Breakpoint มาตรฐานของ Bootstrap 5

| Breakpoint | Class infix | ความกว้างขั้นต่ำของหน้าจอ | อุปกรณ์ตัวอย่าง |
|---|---|---|---|
| Extra small | (ไม่มี, ค่าเริ่มต้น) | `<576px` | มือถือแนวตั้ง |
| Small | `sm` | `≥576px` | มือถือแนวนอน |
| Medium | `md` | `≥768px` | แท็บเล็ต |
| Large | `lg` | `≥992px` | โน้ตบุ๊กจอเล็ก |
| Extra large | `xl` | `≥1200px` | จอเดสก์ท็อป |
| Extra extra large | `xxl` | `≥1400px` | จอกว้างมาก |

Class เขียนในรูปแบบ `col-{breakpoint}-{จำนวนคอลัมน์}` เช่น `col-md-6` แปลว่า
"ตั้งแต่ขนาดจอ medium (≥768px) ขึ้นไป ให้กิน 6/12 คอลัมน์" ถ้าจอเล็กกว่านั้น
(ไม่มี prefix ตรงกับ breakpoint ที่กำหนด) จะ fallback ไปใช้ค่าที่กำหนดไว้ของ
breakpoint เล็กกว่าที่ใกล้ที่สุด หรือเต็ม 12 คอลัมน์ (`col-12` โดยปริยาย) ถ้าไม่
ได้กำหนดอะไรไว้เลย — นี่คือหัวใจของแนวคิด **Mobile-First**: กำหนดจากจอเล็กไปจอ
ใหญ่เสมอ

### 503.4 นำ Grid มาใช้จริง: จัดหน้า `blog/list.html` เป็นการ์ด 3 คอลัมน์

```html
<!-- blog/templates/blog/list.html -->
{% extends 'base.html' %}

{% block title %}บทความทั้งหมด | Django Mastery Blog{% endblock %}

{% block content %}
<h1 class="mb-4">บทความทั้งหมด</h1>

<div class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4">
    {% for post in posts %}
        <div class="col">
            {% include 'blog/_post_card.html' with post=post only %}
        </div>
    {% empty %}
        <div class="col-12">
            <p class="text-muted">ยังไม่มีบทความในขณะนี้</p>
        </div>
    {% endfor %}
</div>
{% endblock %}
```

**อธิบาย class ที่ใช้:**

| Class | ความหมาย |
|---|---|
| `row-cols-1` | บนจอ extra-small (มือถือ) แสดง 1 คอลัมน์ต่อแถว |
| `row-cols-md-2` | ตั้งแต่จอ medium (แท็บเล็ต) ขึ้นไป แสดง 2 คอลัมน์ต่อแถว |
| `row-cols-lg-3` | ตั้งแต่จอ large (เดสก์ท็อป) ขึ้นไป แสดง 3 คอลัมน์ต่อแถว |
| `g-4` | Gutter (ระยะห่างระหว่างการ์ด) ขนาด 4 ตามสเกลของ Bootstrap spacing (0-5) |

`row-cols-*` เป็นวิธีที่สะดวกกว่าการกำหนด `col-md-4` ให้แต่ละการ์ดเอง เพราะ
ไม่ต้องคำนวณเลขคอลัมน์ (12 ÷ 3 = 4) ด้วยตัวเอง — Bootstrap คำนวณให้อัตโนมัติ
จากจำนวนที่ระบุใน `row-cols-*`

### 503.5 ปรับ `_post_card.html` ให้ใช้ Bootstrap Card component

```html
<!-- blog/templates/blog/_post_card.html -->
<article class="card h-100 shadow-sm">
    <div class="card-body d-flex flex-column">
        <h2 class="card-title h5">
            <a href="{% url 'blog:detail' slug=post.slug %}" class="text-decoration-none">
                {{ post.title }}
            </a>
        </h2>
        <p class="card-subtitle mb-2 text-muted small">
            {{ post.created_at|date:"d F Y" }}
        </p>
        <p class="card-text flex-grow-1">
            {{ post.content|truncatewords:25 }}
        </p>
        <a href="{% url 'blog:detail' slug=post.slug %}" class="btn btn-outline-primary btn-sm mt-auto">
            อ่านต่อ &rarr;
        </a>
    </div>
</article>
```

`h-100` (height: 100%) ทำให้การ์ดทุกใบในแถวเดียวกันสูงเท่ากันเสมอแม้เนื้อหา
ยาวไม่เท่ากัน ส่วน `d-flex flex-column` กับ `flex-grow-1` และ `mt-auto` ผลักปุ่ม
"อ่านต่อ" ให้อยู่ชิดขอบล่างของการ์ดเสมอ — pattern ที่ใช้บ่อยมากเมื่อทำ card grid
ที่เนื้อหายาวไม่เท่ากัน

### 503.6 Utility Classes ด้าน Spacing ที่ใช้บ่อยที่สุด

| Class pattern | ความหมาย | ตัวอย่าง |
|---|---|---|
| `m-{0-5}` | margin ทุกด้าน | `m-3` |
| `mt-`, `mb-`, `ms-`, `me-` | margin-top/bottom/start(ซ้าย)/end(ขวา) | `mt-4` |
| `mx-`, `my-` | margin แนวนอน/แนวตั้ง | `mx-auto` (จัดกึ่งกลางแนวนอน) |
| `p-{0-5}` | padding ทุกด้าน (มี `pt-`, `pb-`, `ps-`, `pe-`, `px-`, `py-` เหมือน margin) | `p-4` |
| `g-{0-5}` | gap ระหว่าง column ใน grid | `g-4` |

ตัวเลข 0-5 คือสเกลมาตรฐานของ Bootstrap (`0` = ไม่มีระยะ, `5` = ระยะห่างมากสุด
ประมาณ 3rem) ระบบสเกลนี้บังคับให้ทุกจุดในเว็บไซต์ใช้ระยะห่างที่ **สอดคล้องกัน**
แทนที่จะเขียน `margin: 17px` มั่ว ๆ ตามใจแต่ละจุด — นี่คือประโยชน์แฝงที่สำคัญของ
CSS Framework ที่มักถูกมองข้าม

---

## ขั้นตอนที่ 504: นำ Bootstrap Component มาใช้จริง (navbar, card, modal) ในหน้า blog

### 504.1 Component คืออะไร ต่างจาก Utility Class อย่างไร

**Utility class** (เช่น `mt-3`, `text-center`, `d-flex`) คือ class เดี่ยว ๆ
ที่ทำหน้าที่เดียวชัดเจน ส่วน **Component** คือชุดของ HTML + class ที่ประกอบกัน
เป็น "ชิ้นส่วน UI" ที่สมบูรณ์ในตัวเอง เช่น navbar, card (ที่ทำไปแล้วในขั้นตอนที่
503.5), modal, alert, dropdown, badge — component มักต้องใช้ HTML structure
ที่เฉพาะเจาะจง (ลำดับ div, class ที่ต้องตรงกันเป๊ะ) ต่างจาก utility class ที่
แปะเข้ากับ element ไหนก็ได้อย่างอิสระ

### 504.2 Navbar Component แบบ Responsive เต็มรูปแบบ

แทนที่ `templates/partials/navbar.html` แบบ CSS มือเขียนจาก Part 008 ด้วย
Bootstrap Navbar component ที่ยุบเป็นเมนูแฮมเบอร์เกอร์อัตโนมัติบนจอเล็ก:

```html
<!-- templates/partials/navbar.html -->
<nav class="navbar navbar-expand-lg navbar-dark bg-primary sticky-top">
    <div class="container">
        <a class="navbar-brand fw-bold" href="{% url 'blog:list' %}">
            Django Mastery Blog
        </a>
        <button
            class="navbar-toggler"
            type="button"
            data-bs-toggle="collapse"
            data-bs-target="#mainNavbar"
            aria-controls="mainNavbar"
            aria-expanded="false"
            aria-label="เปิด/ปิดเมนู"
        >
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="mainNavbar">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
                <li class="nav-item">
                    <a class="nav-link {% if request.resolver_match.url_name == 'list' %}active{% endif %}"
                       href="{% url 'blog:list' %}">
                        บทความทั้งหมด
                    </a>
                </li>
                <li class="nav-item">
                    <a class="nav-link {% if request.resolver_match.url_name == 'contact' %}active{% endif %}"
                       href="{% url 'blog:contact' %}">
                        ติดต่อเรา
                    </a>
                </li>
            </ul>

            {% if user.is_authenticated %}
                <span class="navbar-text text-white me-3">
                    สวัสดี, {{ user.username }}
                </span>
                <form method="post" action="{% url 'logout' %}" class="d-inline">
                    {% csrf_token %}
                    <button type="submit" class="btn btn-outline-light btn-sm">ออกจากระบบ</button>
                </form>
            {% else %}
                <a href="{% url 'login' %}" class="btn btn-outline-light btn-sm">เข้าสู่ระบบ</a>
            {% endif %}
        </div>
    </div>
</nav>
```

**อธิบาย attribute ที่สำคัญ:**

| Attribute/Class | หน้าที่ |
|---|---|
| `navbar-expand-lg` | ยุบเมนูเป็นแฮมเบอร์เกอร์เมื่อจอเล็กกว่า `lg`, แสดงเมนูเต็มแนวนอนเมื่อจอ `lg` ขึ้นไป |
| `data-bs-toggle="collapse"` / `data-bs-target="#mainNavbar"` | บอก Bootstrap JS ให้คลิกปุ่มนี้แล้วเปิด/ปิด element ที่มี `id="mainNavbar"` — **ต้องมี Bootstrap Bundle JS โหลดอยู่เสมอ** ไม่เช่นนั้นปุ่มจะไม่ทำงาน |
| `sticky-top` | ปักหมุด navbar ให้ติดขอบบนของหน้าจอเสมอเมื่อ scroll ลง |
| `{% if request.resolver_match.url_name == '...' %}active{% endif %}` | ใส่ class `active` ให้เมนูที่ตรงกับหน้าปัจจุบัน (ทบทวน `request` context processor จาก Part 008 ข้อ 79) |

### 504.3 Modal Component: กล่องยืนยันการลบบทความ

ทบทวนจาก Part 023: `DeleteView` แสดงหน้ายืนยันก่อนลบเสมอ แต่แทนที่จะ redirect
ไปหน้าใหม่ทั้งหน้า สามารถใช้ Bootstrap Modal เพื่อยืนยันแบบ popup โดยไม่ต้อง
ออกจากหน้ารายละเอียดบทความเลย:

```html
<!-- blog/templates/blog/detail.html (ส่วนปุ่มลบ สำหรับเจ้าของบทความ/staff) -->
{% if user.is_staff %}
<button type="button" class="btn btn-danger btn-sm" data-bs-toggle="modal" data-bs-target="#deleteConfirmModal">
    ลบบทความนี้
</button>

<div class="modal fade" id="deleteConfirmModal" tabindex="-1" aria-labelledby="deleteConfirmLabel" aria-hidden="true">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="deleteConfirmLabel">ยืนยันการลบบทความ</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="ปิด"></button>
            </div>
            <div class="modal-body">
                <p>
                    คุณแน่ใจหรือไม่ว่าต้องการลบบทความ <strong>"{{ post.title }}"</strong>?
                </p>
                <p class="text-danger mb-0">การกระทำนี้ไม่สามารถย้อนกลับได้</p>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">ยกเลิก</button>
                <form method="post" action="{% url 'blog:delete' slug=post.slug %}" class="d-inline">
                    {% csrf_token %}
                    <button type="submit" class="btn btn-danger">ยืนยันการลบ</button>
                </form>
            </div>
        </div>
    </div>
</div>
{% endif %}
```

**หลักการทำงานของ Modal ไม่ต้องเขียน JavaScript เอง:**

1. `data-bs-toggle="modal"` + `data-bs-target="#deleteConfirmModal"` บนปุ่ม
   ทำให้ Bootstrap JS เปิด modal นั้นเมื่อคลิก
2. `class="modal fade"` ทำให้ modal ซ่อนอยู่โดยปริยาย (มี `display: none` และ
   transition แบบ fade เมื่อเปิด/ปิด)
3. `data-bs-dismiss="modal"` บนปุ่มใด ๆ ภายใน modal ทำให้กดแล้วปิด modal ทันที
4. ฟอร์มยืนยันการลบข้างในยัง submit ตามปกติผ่าน `<form method="post">` — modal
   เป็นเพียง "กล่องแสดงผล" ไม่ได้เปลี่ยนวิธีการทำงานของฟอร์ม Django เลย

> **ข้อควรระวังเรื่อง Accessibility**: `aria-labelledby`, `aria-hidden`, และ
> `aria-label` ที่เห็นในตัวอย่างไม่ใช่ของตกแต่ง แต่จำเป็นสำหรับโปรแกรมอ่านหน้าจอ
> (screen reader) ที่ผู้พิการทางสายตาใช้งาน Bootstrap components ทุกตัวถูก
> ออกแบบตามมาตรฐาน WAI-ARIA มาให้แล้ว **อย่าลบ attribute เหล่านี้ทิ้งแม้จะดู
> เหมือนไม่มีผลต่อหน้าตา**

### 504.4 Alert Component: แสดงข้อความแจ้งเตือนจาก Django Messages Framework

Component ที่ใช้คู่กับ Django Messages Framework บ่อยที่สุดคือ `alert`
(เตรียมพื้นฐานไว้ก่อนสำหรับตอนเรียน Messages Framework เต็มรูปแบบในเฟสถัดไป):

```html
<!-- templates/partials/messages.html -->
{% if messages %}
    {% for message in messages %}
        <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
            {{ message }}
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="ปิด"></button>
        </div>
    {% endfor %}
{% endif %}
```

```html
<!-- templates/base.html (เพิ่มก่อน {% block content %}) -->
<main class="container my-4">
    {% include 'partials/messages.html' %}
    {% block content %}{% endblock %}
</main>
```

สังเกตว่า `message.tags` (ค่าเช่น `success`, `error`, `warning`, `info` จาก
Django Messages Framework) แมปตรงกับชื่อ Bootstrap alert variant พอดี
(`alert-success`, `alert-danger` ฯลฯ — ยกเว้น Django ใช้คำว่า `error` แต่
Bootstrap ใช้ `danger` ซึ่งต้อง map เพิ่มใน `settings.py` ด้วย
`MESSAGE_TAGS` เมื่อถึงบทที่เจาะลึก Messages Framework)

### 504.5 Dropdown และ Badge: Component ขนาดเล็กที่ใช้บ่อย

```html
<!-- ตัวอย่าง Dropdown สำหรับกรองบทความตามหมวดหมู่ -->
<div class="dropdown mb-3">
    <button class="btn btn-outline-secondary dropdown-toggle" type="button" data-bs-toggle="dropdown" aria-expanded="false">
        กรองตามสถานะ
    </button>
    <ul class="dropdown-menu">
        <li><a class="dropdown-item" href="?status=published">เผยแพร่แล้ว</a></li>
        <li><a class="dropdown-item" href="?status=draft">ฉบับร่าง</a></li>
        <li><hr class="dropdown-divider"></li>
        <li><a class="dropdown-item" href="?status=">ทั้งหมด</a></li>
    </ul>
</div>

<!-- ตัวอย่าง Badge แสดงสถานะบทความในการ์ด -->
{% if post.is_published %}
    <span class="badge text-bg-success">เผยแพร่แล้ว</span>
{% else %}
    <span class="badge text-bg-secondary">ฉบับร่าง</span>
{% endif %}
```

---

## ขั้นตอนที่ 505: `django-widget-tweaks` สำหรับเติม class Bootstrap เข้า field ฟอร์มที่ไม่ได้ใช้ crispy-forms

### 505.1 เมื่อไหร่ที่ crispy-forms "หนักเกินไป"

crispy-forms เหมาะกับฟอร์มที่ซับซ้อน ต้องจัด layout เยอะ แต่บางครั้งคุณแค่
ต้องการ **เติม class เข้า field เดียวหรือสองสามตัว** ในฟอร์มง่าย ๆ (เช่นฟอร์ม
ค้นหาบน navbar) การติดตั้ง `FormHelper` ทั้งชุดอาจดูเกินความจำเป็น
`django-widget-tweaks` คือ package ที่เบากว่า ทำหน้าที่เดียวคือ "เติม HTML
attribute เข้า field ที่ render ในเทมเพลต" โดยไม่ต้องแตะ `forms.py` เลย

### 505.2 ติดตั้ง

```bash
pip install django-widget-tweaks
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'widget_tweaks',
]
```

### 505.3 `add_class`: เติม class โดยไม่แก้ `forms.py`

ทบทวน `PostFilterForm` จาก Part 025 ที่ยังไม่มี Bootstrap class ติดมาเลย:

```python
# blog/forms.py (จาก Part 025 — ไม่ต้องแก้ไขอะไรเลย)
class PostFilterForm(forms.Form):
    STATUS_CHOICES = [
        ("", "ทั้งหมด"),
        ("published", "เผยแพร่แล้ว"),
        ("draft", "ฉบับร่าง"),
    ]
    status = forms.ChoiceField(choices=STATUS_CHOICES, required=False, label="สถานะ")
    keyword = forms.CharField(max_length=100, required=False, label="คำค้นหา")
```

```html
<!-- blog/templates/blog/_filter_form.html -->
{% load widget_tweaks %}

<form method="get" class="row g-2 mb-4">
    <div class="col-auto">
        {{ form.keyword|add_class:"form-control"|attr:"placeholder:ค้นหาบทความ..." }}
    </div>
    <div class="col-auto">
        {{ form.status|add_class:"form-select" }}
    </div>
    <div class="col-auto">
        <button type="submit" class="btn btn-primary">ค้นหา</button>
    </div>
</form>
```

`|add_class:"form-control"` เป็น template filter ที่ package นี้เพิ่มเข้ามา
ทำหน้าที่แนบ class ต่อท้าย `<input>`/`<select>` ที่มีอยู่แล้วโดยอัตโนมัติ
(ไม่ overwrite class เดิมถ้ามี) ส่วน `|attr:"placeholder:..."` ใช้เติม HTML
attribute อื่น ๆ ที่ไม่ใช่ `class` ในรูปแบบ `key:value`

### 505.4 `render_field` Template Tag: อีกวิธีที่อ่านง่ายกว่าเมื่อมีหลาย attribute

```html
{% load widget_tweaks %}

<div class="mb-3">
    {{ form.keyword.label_tag }}
    {% render_field form.keyword class="form-control" placeholder="ค้นหาบทความ..." autocomplete="off" %}
</div>
```

`render_field` อ่านง่ายกว่าเมื่อต้องเติมหลาย attribute พร้อมกัน เพราะเขียนแบบ
`key="value"` ปกติแทนที่จะต้องต่อ filter หลายตัวด้วย `|`

### 505.5 เติม class แบบมีเงื่อนไข: แสดง error state ของ Bootstrap

Bootstrap ใช้ class `is-invalid` เพื่อแสดงกรอบสีแดงเมื่อ field มี error
(เหมือนที่ crispy-forms ทำให้อัตโนมัติ) แต่เมื่อใช้ widget-tweaks ต้องเติมเอง
แบบมีเงื่อนไข:

```html
{% load widget_tweaks %}

<div class="mb-3">
    {{ form.keyword.label_tag }}
    {% if form.keyword.errors %}
        {{ form.keyword|add_class:"form-control is-invalid" }}
        <div class="invalid-feedback">{{ form.keyword.errors.0 }}</div>
    {% else %}
        {{ form.keyword|add_class:"form-control" }}
    {% endif %}
</div>
```

### 505.6 ตารางเปรียบเทียบ `django-crispy-forms` vs `django-widget-tweaks`

| ประเด็น | `django-crispy-forms` + `crispy-bootstrap5` | `django-widget-tweaks` |
|---|---|---|
| ปรัชญา | "บอกว่าจะ render ฟอร์มทั้งก้อนแบบไหน" (declarative, ผ่าน `FormHelper`) | "เติม attribute เข้า field ทีละตัวในเทมเพลต" (imperative) |
| ต้องแก้ `forms.py` ไหม | แนะนำให้เพิ่ม `self.helper` เพื่อคุม layout เต็มรูปแบบ | ไม่ต้องแตะ `forms.py` เลย |
| ควบคุมความละเอียดของ HTML | ควบคุมได้แต่ต้องเรียนรู้ syntax ของ `Layout()` เพิ่ม | ควบคุมได้อิสระเต็มที่เพราะเขียน HTML เองในเทมเพลต |
| เหมาะกับฟอร์มขนาดใหญ่ ซับซ้อน หลาย field | เหมาะมาก ลดโค้ดซ้ำได้เยอะ | เขียนซ้ำเยอะถ้า field มาก |
| เหมาะกับฟอร์มเล็ก ๆ 1-3 field | overkill เล็กน้อย | เหมาะสมพอดี รวดเร็ว |
| Learning curve | สูงกว่าเล็กน้อย (ต้องเข้าใจ `Layout`, `Row`, `Column`) | ต่ำมาก (แค่ filter เดียว) |

**คำแนะนำของหลักสูตรนี้**: ใช้ `crispy-forms` สำหรับฟอร์มหลักที่มีหลาย field
(เช่น `ContactForm`, ฟอร์มลงทะเบียน) และใช้ `widget-tweaks` สำหรับฟอร์มเล็ก ๆ
ที่ฝังอยู่ในหน้าอื่น เช่น ฟอร์มค้นหาบน navbar หรือฟอร์มกรองข้อมูลสั้น ๆ —
ทั้งสอง package ติดตั้งพร้อมกันในโปรเจกต์เดียวได้โดยไม่ชนกันเลย

---

## ขั้นตอนที่ 506: หลักการ Responsive Design และการทดสอบบนขนาดจอต่าง ๆ

### 506.1 Responsive Design คืออะไร และทำไม Mobile-First ถึงสำคัญ

**Responsive Web Design** คือแนวทางออกแบบเว็บให้ปรับ layout, ขนาดตัวอักษร, และ
การจัดวาง element โดยอัตโนมัติตามขนาดหน้าจอของอุปกรณ์ที่ใช้เข้าชม โดยไม่ต้อง
สร้างเว็บแยกเวอร์ชันสำหรับมือถือกับเดสก์ท็อป

ปี 2026 การเข้าเว็บผ่านมือถือมีสัดส่วนมากกว่าเดสก์ท็อปในเกือบทุกอุตสาหกรรม
Bootstrap ถูกออกแบบตามหลัก **Mobile-First** ตั้งแต่แกนกลาง หมายความว่า:

- Style เริ่มต้น (ไม่มี breakpoint prefix เช่น `col-6`) ใช้กับ**ทุกขนาดจอ**
  รวมถึงจอเล็กสุด
- Style ที่มี breakpoint prefix (เช่น `col-md-6`) จะ**เพิ่มเข้ามาทับ**เมื่อจอ
  กว้างถึงขนาดที่กำหนด (`min-width`, ไม่ใช่ `max-width`)
- ผลคือ: ควรออกแบบ layout สำหรับจอเล็กสุดก่อนเสมอ แล้วค่อยไล่ปรับสำหรับจอที่ใหญ่
  ขึ้น ไม่ใช่ออกแบบจอใหญ่ก่อนแล้วค่อยมา "หด" ทีหลัง

### 506.2 Meta Viewport Tag: กฎเหล็กที่ขาดไม่ได้

`base.html` ของเรามี tag นี้อยู่แล้วตั้งแต่ Part 001:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

ถ้าลืม tag นี้ **Responsive CSS ทั้งหมดของ Bootstrap จะไม่ทำงานเลย** เพราะ
เบราว์เซอร์มือถือ (โดยเฉพาะ iOS Safari) จะ render หน้าเว็บที่ความกว้างสมมติ
980px แล้วค่อยย่อทั้งหน้าให้พอดีจอ แทนที่จะ render ตามความกว้างจริงของอุปกรณ์
ทำให้ media query ที่ควร trigger บนมือถือไม่ทำงานตามที่คาด

### 506.3 Responsive Utility Classes: ซ่อน/แสดง Element ตามขนาดจอ

| Class | ความหมาย |
|---|---|
| `d-none` | ซ่อนเสมอทุกขนาดจอ |
| `d-none d-md-block` | ซ่อนบนจอเล็กกว่า `md`, แสดงแบบ `block` ตั้งแต่ `md` ขึ้นไป |
| `d-block d-md-none` | แสดงบนจอเล็กกว่า `md` เท่านั้น (ตรงข้ามกับข้างบน) — ใช้บ่อยเมื่อทำเมนูแบบต่างกันระหว่างมือถือกับเดสก์ท็อป |
| `d-flex d-lg-none` | แสดงแบบ flex บนจอเล็กกว่า `lg`, ซ่อนตั้งแต่ `lg` ขึ้นไป |

ตัวอย่างจริง: แสดงข้อความสรุปสั้น ๆ บนมือถือ แต่แสดงเนื้อหาเต็มบนเดสก์ท็อป:

```html
<p class="d-block d-md-none">{{ post.content|truncatewords:15 }}</p>
<p class="d-none d-md-block">{{ post.content|truncatewords:40 }}</p>
```

### 506.4 Responsive Text Alignment และ Flex Direction

```html
<!-- ข้อความชิดกลางบนมือถือ ชิดซ้ายบนจอใหญ่ -->
<h1 class="text-center text-lg-start">Django Mastery Blog</h1>

<!-- การ์ดเรียงเป็นแนวตั้งบนมือถือ แนวนอนบนจอใหญ่ -->
<div class="d-flex flex-column flex-md-row gap-3">
    <div class="flex-fill">คอลัมน์ 1</div>
    <div class="flex-fill">คอลัมน์ 2</div>
</div>
```

### 506.5 เครื่องมือทดสอบ Responsive Design

| เครื่องมือ | วิธีเปิด | ใช้เมื่อไร |
|---|---|---|
| **Chrome DevTools Device Toolbar** | กด `F12` แล้วกด `Ctrl+Shift+M` (Windows/Linux) หรือ `Cmd+Shift+M` (macOS) | ทดสอบเบื้องต้นระหว่างพัฒนา จำลองขนาดจออุปกรณ์จริง (iPhone, iPad, Galaxy ฯลฯ) รวมถึงปรับ custom resolution เองได้ |
| **Firefox Responsive Design Mode** | `Ctrl+Shift+M` เหมือนกัน | ทางเลือกสำหรับผู้ใช้ Firefox มีฟีเจอร์ใกล้เคียง Chrome |
| **ทดสอบบนอุปกรณ์จริง** | เชื่อมมือถือกับคอมพิวเตอร์ผ่าน USB แล้วใช้ `chrome://inspect` (Chrome) หรือทดสอบผ่าน network เดียวกันด้วย `python manage.py runserver 0.0.0.0:8000` แล้วเข้าจาก IP เครื่อง | ยืนยันผลสุดท้ายก่อน deploy จริง เพราะ emulator ไม่สามารถจำลอง touch gesture, การพิมพ์ผ่านคีย์บอร์ดมือถือ, หรือความเร็ว rendering จริงได้ 100% |
| **BrowserStack / LambdaTest** | บริการ cloud testing แบบเสียเงิน (มี free tier จำกัด) | ทดสอบข้าม browser/OS จริงจำนวนมากโดยไม่ต้องมีอุปกรณ์จริงครบทุกรุ่น (นิยมใช้ในทีม QA ระดับ enterprise) |

### 506.6 รัน Django Development Server ให้เข้าถึงได้จากมือถือในเครือข่ายเดียวกัน

```bash
python manage.py runserver 0.0.0.0:8000
```

การระบุ `0.0.0.0` แทน `127.0.0.1` (ค่าเริ่มต้น) ทำให้ development server รับ
connection จากทุก network interface ไม่ใช่แค่จากเครื่องตัวเอง จากนั้นหา IP
ของเครื่องในเครือข่าย (เช่น `192.168.1.50`) แล้วเข้าจากมือถือผ่าน
`http://192.168.1.50:8000/` (ต้องเชื่อม Wi-Fi เดียวกัน) — **อย่าลืมเพิ่ม IP
นี้ใน `ALLOWED_HOSTS`** ของ `settings.py` ไม่เช่นนั้น Django จะปฏิเสธ request
ด้วย `DisallowedHost` (ทบทวนจาก Part 010)

```python
# config/settings.py (สำหรับทดสอบใน dev เท่านั้น)
ALLOWED_HOSTS = ["127.0.0.1", "localhost", "192.168.1.50"]
```

---

## ขั้นตอนที่ 507: Custom CSS override ตัวแปร Bootstrap ด้วย Sass (เปลี่ยนสีธีมของเว็บ)

### 507.1 ทำไม override ผ่าน CDN ไม่ได้

เมื่อโหลด Bootstrap ผ่าน CDN (ขั้นตอนที่ 501) คุณได้ไฟล์ `.css` ที่ **compile
เสร็จแล้ว** สีทุกสี ระยะห่างทุกจุดถูกเขียนเป็นค่าตายตัวไว้หมด การจะเปลี่ยนสี
primary ของทั้งเว็บ (ปุ่ม, navbar, ลิงก์ ฯลฯ พร้อมกันหมด) โดยไม่แก้ source
ต้องเขียน CSS **override** ทับด้วยมือทีละจุด ซึ่งไม่ยั่งยืนเมื่อ component
เพิ่มขึ้นเรื่อย ๆ

Bootstrap เขียนด้วยภาษา **Sass (SCSS)** ซึ่งเป็น CSS preprocessor ที่รองรับ
ตัวแปร (variables) หากมี **source code เป็น `.scss`** จะสามารถเปลี่ยนค่าตัวแปร
ก่อน compile ได้ ทำให้สีเปลี่ยนทั้งเว็บโดยแก้แค่บรรทัดเดียว

### 507.2 ติดตั้ง Bootstrap ผ่าน npm พร้อม Sass compiler

```bash
npm init -y
npm install bootstrap@5.3.3
npm install -D sass
```

### 507.3 สร้างไฟล์ `custom.scss` เพื่อ override ตัวแปรก่อน import

```
blog/
└── static/
    ├── scss/
    │   └── custom.scss        ← ไฟล์ต้นฉบับที่เราแก้
    └── blog/
        └── css/
            └── custom.css     ← ไฟล์ที่ compile ออกมา (ใช้จริงใน template)
```

```scss
// blog/static/scss/custom.scss

// 1. Override ตัวแปรสีหลักของ Bootstrap ก่อน import
//    (ต้องอยู่ "ก่อน" บรรทัด @import bootstrap เสมอ ไม่เช่นนั้นจะไม่มีผล)
$primary:   #7c3aed;   // ม่วงแทนสีฟ้าเริ่มต้น
$secondary: #10b981;   // เขียวมรกต
$danger:    #dc2626;
$body-bg:   #f8fafc;
$font-family-sans-serif: "Sarabun", "Segoe UI", sans-serif;
$border-radius: 0.6rem;
$border-radius-lg: 0.9rem;

// 2. Import Bootstrap ทั้งหมดต่อจากนี้ (ลำดับสำคัญมาก!)
@import "../../../node_modules/bootstrap/scss/bootstrap";

// 3. เขียน custom style เพิ่มเติมของเราเองต่อท้ายได้ตามปกติ
.navbar-brand {
    letter-spacing: 0.02em;
}

.card {
    transition: transform 0.15s ease-in-out;
}
.card:hover {
    transform: translateY(-4px);
}
```

**กฎเหล็กของ Sass override**: ตัวแปรต้องถูก**ประกาศก่อน** `@import "bootstrap"`
เสมอ เพราะ Sass ใช้ค่าตัวแปรที่ประกาศไว้ ณ จุดที่ import เกิดขึ้น ถ้าประกาศ
ตัวแปรหลัง import ไปแล้ว Bootstrap จะ compile ด้วยค่าเริ่มต้นของมันไปแล้ว
การมาประกาศทับทีหลังจะไม่มีผลอะไรเลย

### 507.4 Compile SCSS เป็น CSS

```bash
npx sass blog/static/scss/custom.scss blog/static/blog/css/custom.css --style=compressed
```

เพิ่ม script ใน `package.json` เพื่อความสะดวก:

```json
{
  "scripts": {
    "build-css": "sass blog/static/scss/custom.scss blog/static/blog/css/custom.css --style=compressed",
    "watch-css": "sass --watch blog/static/scss/custom.scss:blog/static/blog/css/custom.css"
  }
}
```

```bash
npm run build-css     # compile ครั้งเดียว (ใช้ตอน deploy)
npm run watch-css     # compile อัตโนมัติทุกครั้งที่แก้ .scss (ใช้ตอนพัฒนา)
```

### 507.5 สลับจาก CDN มาใช้ไฟล์ที่ compile เอง ใน `base.html`

```html
<!-- templates/base.html -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>

    <!-- เปลี่ยนจาก CDN มาเป็นไฟล์ที่ compile เองจาก custom.scss -->
    <link rel="stylesheet" href="{% static 'blog/css/custom.css' %}">
    {% block extra_head %}{% endblock %}
</head>
<body>
    <!-- ... เนื้อหาเดิม ... -->

    <!-- Bootstrap JS ยังใช้ CDN ได้ตามปกติ เพราะ JS ไม่มี Sass variable ให้ override -->
    <script
        src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz"
        crossorigin="anonymous"></script>
</body>
</html>
```

**สังเกต**: `custom.css` ที่ compile ออกมามีทุกอย่างของ Bootstrap อยู่ในตัวแล้ว
(เพราะมี `@import "bootstrap"` ข้างใน) จึง**ไม่ต้อง**โหลด Bootstrap CSS จาก CDN
ซ้ำอีก — โหลดไฟล์เดียวจบ ส่วน JS bundle ยังใช้จาก CDN ได้ตามปกติเพราะ JavaScript
ของ Bootstrap ไม่มีแนวคิดเรื่อง Sass variable มาเกี่ยวข้อง

### 507.6 ตัวแปร Bootstrap Sass ที่ override บ่อยที่สุด

| ตัวแปร | ควบคุม | ค่าเริ่มต้นของ Bootstrap |
|---|---|---|
| `$primary` | สีหลัก (ปุ่ม primary, ลิงก์, navbar ถ้าใช้ `bg-primary`) | `#0d6efd` (ฟ้า) |
| `$secondary` | สีรอง | `#6c757d` (เทา) |
| `$success`, `$danger`, `$warning`, `$info` | สี alert/badge ตามความหมาย | เขียว/แดง/เหลือง/ฟ้าอ่อน ตามลำดับ |
| `$font-family-sans-serif` | ฟอนต์หลักทั้งเว็บไซต์ | `system-ui, -apple-system, ...` |
| `$border-radius` | ความมนของมุม card, button, input | `0.375rem` |
| `$spacer` | หน่วยฐานของ spacing scale (`m-*`, `p-*` คำนวณจากค่านี้) | `1rem` |
| `$grid-gutter-width` | ระยะห่างเริ่มต้นระหว่าง column ใน grid | `1.5rem` |

รายการตัวแปรทั้งหมดดูได้จาก
`node_modules/bootstrap/scss/_variables.scss` ในโปรเจกต์ของคุณเอง (ไฟล์นี้มี
ตัวแปรมากกว่า 1,500 ตัว ครอบคลุมแทบทุกรายละเอียดของ Bootstrap)

---

## ขั้นตอนที่ 508: เกริ่น Tailwind CSS เป็นทางเลือก พร้อมติดตั้งผ่าน `django-tailwind`

### 508.1 Tailwind CSS คืออะไร ต่างจาก Bootstrap อย่างไรในเชิงปรัชญา

**Tailwind CSS** คือ CSS Framework แนว **Utility-First** ที่ได้รับความนิยมสูง
มากในช่วงปี 2020-2026 แนวคิดตรงข้ามกับ Bootstrap อย่างชัดเจน:

- **Bootstrap (Component-based)**: ให้ class สำเร็จรูปสำหรับ component ทั้งชิ้น
  เช่น `.card`, `.navbar`, `.btn` — เขียน HTML น้อย ได้หน้าตาสำเร็จรูปเร็ว แต่
  ถ้าต้องการดีไซน์ที่ต่างจาก Bootstrap มาก ต้อง override เยอะ
- **Tailwind (Utility-first)**: ให้ class ระดับเล็กที่สุดเท่านั้น (เช่น
  `flex`, `p-4`, `rounded-lg`, `bg-purple-600`) ไม่มี component สำเร็จรูปให้
  เลย ต้องประกอบ utility class เองทุกครั้งเพื่อสร้างหน้าตาที่ต้องการ — ยืดหยุ่น
  สูงสุด แต่ HTML จะยาวและมี class เยอะกว่ามาก

### 508.2 ตัวอย่างเปรียบเทียบโค้ดจริง: การ์ดเดียวกัน สองแนวทาง

```html
<!-- Bootstrap: component สำเร็จรูป -->
<div class="card shadow-sm">
    <div class="card-body">
        <h5 class="card-title">หัวข้อ</h5>
        <p class="card-text">เนื้อหา...</p>
    </div>
</div>
```

```html
<!-- Tailwind: ประกอบ utility เองทั้งหมด -->
<div class="rounded-lg shadow-sm bg-white p-4">
    <h5 class="text-lg font-semibold mb-2">หัวข้อ</h5>
    <p class="text-gray-700">เนื้อหา...</p>
</div>
```

### 508.3 ติดตั้ง Tailwind ใน Django ผ่าน `django-tailwind`

`django-tailwind` เป็น package ที่จัดการเชื่อม Tailwind (ซึ่งโดยธรรมชาติเป็น
เครื่องมือฝั่ง Node.js) เข้ากับ Django ให้อัตโนมัติ สร้างแอป Django แยกต่างหาก
สำหรับเก็บไฟล์ Tailwind โดยเฉพาะ:

```bash
pip install django-tailwind
pip freeze > requirements.txt
```

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps เดิม ...
    'tailwind',
]

TAILWIND_APP_NAME = 'theme'
```

สร้างแอปธีมด้วยคำสั่งของ package (จะถามชื่อแอป — ใช้ `theme`):

```bash
python manage.py tailwind init
```

```python
# config/settings.py (เพิ่มหลัง init เสร็จ)
INSTALLED_APPS = [
    # ... apps เดิม ...
    'tailwind',
    'theme',
]
```

`django-tailwind` ต้องใช้ Node.js เบื้องหลัง (ติดตั้ง dependency ของ Tailwind
เองภายในแอป `theme`):

```bash
python manage.py tailwind install
```

ระหว่างพัฒนา ต้องรัน Tailwind ในโหมด watch คู่ขนานกับ `runserver` (เปิด
Terminal 2 หน้าต่างพร้อมกัน):

```bash
# Terminal 1
python manage.py tailwind start

# Terminal 2
python manage.py runserver
```

`tailwind start` จะจับตาดูไฟล์ template ทั้งหมด แล้ว compile เฉพาะ utility
class ที่**ถูกใช้จริง**ในโปรเจกต์เท่านั้นออกมาเป็น CSS ไฟล์เดียว (กลไกนี้เรียกว่า
**JIT — Just-In-Time compilation** ซึ่งเป็นเหตุผลหลักที่ทำให้ไฟล์ CSS สุดท้าย
ของ Tailwind มีขนาดเล็กมาก แม้จะมี utility class ให้เลือกใช้หลายพันตัว)

### 508.4 ใช้งาน Tailwind ใน Template

```html
<!-- theme/templates/base.html -->
{% load tailwind_tags %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>
    {% tailwind_css %}
</head>
<body class="bg-gray-50 text-gray-900">
    <nav class="bg-purple-700 text-white px-4 py-3 flex items-center justify-between">
        <a href="{% url 'blog:list' %}" class="font-bold text-lg">Django Mastery Blog</a>
        <div class="space-x-4 hidden md:flex">
            <a href="{% url 'blog:list' %}" class="hover:underline">บทความทั้งหมด</a>
            <a href="{% url 'blog:contact' %}" class="hover:underline">ติดต่อเรา</a>
        </div>
    </nav>

    <main class="container mx-auto px-4 py-6">
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

`{% tailwind_css %}` เป็น template tag ที่ package จัดเตรียมไว้ให้ ทำหน้าที่
แทรก `<link>` ไปยังไฟล์ CSS ที่ Tailwind compile ออกมาโดยอัตโนมัติ (คล้ายกับ
`{% tailwind_css %}` เทียบเท่ากับ `{% static %}` ที่ชี้ไปยังไฟล์ที่ตำแหน่ง
เฉพาะของแอป `theme`)

### 508.5 Responsive Prefix ใน Tailwind เทียบกับ Bootstrap

Tailwind ใช้แนวคิด breakpoint prefix คล้าย Bootstrap แต่ไวยากรณ์ต่างกัน:

| Breakpoint | Bootstrap | Tailwind |
|---|---|---|
| จอเล็กสุด (ค่าเริ่มต้น) | (ไม่มี prefix) | (ไม่มี prefix) |
| ≥640px | ไม่มี breakpoint นี้ตรง ๆ | `sm:` |
| ≥768px | `md-` | `md:` |
| ≥992px | `lg-` | `lg:` |
| ≥1280px | ไม่มี breakpoint นี้ตรง ๆ | `xl:` |
| ≥1536px | ไม่มี breakpoint นี้ตรง ๆ | `2xl:` |

ตัวอย่าง: `hidden md:flex` ใน Tailwind (ซ่อนบนจอเล็ก, แสดงเป็น flex ตั้งแต่
`md` ขึ้นไป) เทียบเท่ากับ `d-none d-md-flex` ใน Bootstrap ทุกประการ — ทั้งสอง
framework ยึดหลัก Mobile-First เหมือนกัน เพียงแค่ใช้คำ (naming convention)
ต่างกัน

> **หมายเหตุสำคัญ**: หลักสูตรนี้จะกลับมาใช้ **Bootstrap 5 เป็นหลักตลอด**ตั้งแต่
> Part 052 เป็นต้นไป Tailwind ที่แนะนำใน Part นี้เป็นเพียงการ**เกริ่นให้รู้จัก**
> ทางเลือกที่มีอยู่จริงในอุตสาหกรรม เพื่อให้คุณตัดสินใจเลือกได้เองอย่างมีข้อมูล
> เมื่อไปทำโปรเจกต์ของตัวเองในอนาคต

---

## ขั้นตอนที่ 509: ตารางเปรียบเทียบ Bootstrap vs Tailwind vs Bulma — เมื่อไหร่ควรเลือกอะไร

### 509.1 แนะนำ Bulma โดยย่อ (framework ตัวที่สามที่ควรรู้จักไว้)

**Bulma** คือ CSS Framework แบบ Component-based เหมือน Bootstrap แต่มีจุดต่าง
สำคัญคือ **เป็น CSS-only ทั้งหมด ไม่มี JavaScript ผูกมาให้เลย** (Bootstrap มี
JS สำหรับ modal, dropdown, collapse ฯลฯ) หมายความว่าถ้าต้องการ modal หรือ
dropdown ที่โต้ตอบได้ ต้องเขียน JavaScript เองหรือพึ่ง library อื่นเสริม
Bulma ใช้ syntax คล้าย Bootstrap มาก (`columns`, `column`, `box`, `button`)
ทำให้เรียนรู้ง่ายสำหรับคนที่คุ้นเคย Bootstrap อยู่แล้ว

```html
<!-- ตัวอย่าง Bulma: การ์ดเดียวกับที่เปรียบเทียบไปก่อนหน้า -->
<div class="box">
    <p class="title is-5">หัวข้อ</p>
    <p class="content">เนื้อหา...</p>
</div>
```

### 509.2 ตารางเปรียบเทียบเต็มรูปแบบ

| ประเด็น | Bootstrap 5 | Tailwind CSS | Bulma |
|---|---|---|---|
| แนวคิดหลัก | Component-based | Utility-first | Component-based |
| ต้องพึ่ง JavaScript ไหม | ต้อง (สำหรับ modal, dropdown, collapse, tooltip) | ไม่ต้องเลย (เป็น CSS ล้วน) | ไม่ต้องเลย (เป็น CSS ล้วน) |
| ความยาวของ HTML | สั้นกว่า (class ระดับ component) | ยาวกว่ามาก (ประกอบ utility เอง) | สั้น (คล้าย Bootstrap) |
| ความยืดหยุ่นในการ custom ดีไซน์ | ปานกลาง (ต้อง override หรือแก้ Sass) | สูงมาก (ออกแบบอะไรก็ได้จาก utility ล้วน) | ปานกลาง (คล้าย Bootstrap) |
| Learning curve สำหรับมือใหม่ | ต่ำ (จำ component name ไม่กี่ตัวก็ใช้ได้) | สูงกว่า (ต้องจำ utility class จำนวนมาก) | ต่ำ (คล้าย Bootstrap) |
| ขนาดไฟล์ CSS สุดท้าย (production) | คงที่ ค่อนข้างใหญ่ (เว้นแต่ purge เอง) | เล็กมาก (JIT compile เฉพาะที่ใช้จริง) | คงที่ ใหญ่กว่า Bootstrap เล็กน้อย |
| ความนิยม/community (ปี 2026) | สูงสุดในกลุ่ม component-based, ใช้กันมานาน | เติบโตเร็วที่สุดในกลุ่ม, นิยมมากในทีม React/Vue/Next.js | นิยมปานกลาง, กลุ่มผู้ใช้เฉพาะทาง |
| แพ็กเกจเชื่อม Django | ไม่มี official แต่ community package (`crispy-bootstrap5`) เยอะและอกครบ | `django-tailwind` (community, ดูแลต่อเนื่อง) | ไม่มี package เชื่อมเฉพาะ ใช้ CDN/npm ตรง ๆ |
| เหมาะกับทีมที่มี Designer เฉพาะ (custom ดีไซน์เต็มรูปแบบ) | ไม่เหมาะเท่า Tailwind (ต้อง fight กับ default styles) | เหมาะที่สุด (ไม่มี default component ให้ fight ด้วย) | ไม่เหมาะเท่า Tailwind |
| เหมาะกับทีมที่ต้องการสร้าง UI เร็ว ไม่มี Designer | เหมาะที่สุด (component สำเร็จรูปครบ) | ช้ากว่า (ต้องประกอบเองทุกจุด) | เหมาะรองลงมาจาก Bootstrap |
| เหมาะกับ Admin/Internal Tool/Prototype | เหมาะมาก | ใช้ได้แต่ overkill สำหรับงานที่ไม่ต้อง custom เยอะ | เหมาะมาก |

### 509.3 เกณฑ์ตัดสินใจแบบย่อสำหรับหลักสูตรนี้และโปรเจกต์จริง

| สถานการณ์ | แนะนำให้เลือก |
|---|---|
| เรียนรู้ Django, ทำ MVP/prototype เร็ว, ทีมเล็กไม่มี Designer | **Bootstrap 5** |
| มี Design System เฉพาะของบริษัท ต้อง custom ดีไซน์ทุกจุดให้ตรงแบรนด์ | **Tailwind CSS** |
| ทำงานร่วมกับทีม Frontend ที่ใช้ React/Vue อยู่แล้วและคุ้นเคย utility-first | **Tailwind CSS** |
| ต้องการ Component สำเร็จรูปแต่ไม่ต้องการ JavaScript dependency เพิ่ม (เช่น static site generator) | **Bulma** |
| Admin Dashboard ภายในองค์กร ที่เน้นความเร็วในการพัฒนามากกว่าความสวยงามเฉพาะตัว | **Bootstrap 5** |
| ต้องการควบคุมขนาดไฟล์ CSS สุดท้ายให้เล็กที่สุดสำหรับเว็บที่มี traffic สูงมาก | **Tailwind CSS** (ด้วย JIT purge) |

**ข้อสรุปสำคัญของหลักสูตร**: ไม่มี framework ไหน "ดีที่สุด" แบบสัมบูรณ์ —
การเลือกขึ้นอยู่กับบริบทของทีมและโปรเจกต์เสมอ หลักสูตรนี้เลือกสอน **Bootstrap 5
เป็นหลัก** เพราะเรียนรู้เร็วที่สุดและเหมาะกับ mental model ของคนที่กำลังเรียน
Django ควบคู่กันไป (เข้าใจ "component" อยู่แล้วจาก CBV, Model — Bootstrap ก็ใช้
mental model แบบ component เดียวกัน) แต่ทักษะ Bootstrap Grid, breakpoint,
utility class ที่เรียนใน Part นี้จะถ่ายทอดไปใช้กับ framework อื่นได้แทบทันที
เพราะแนวคิดพื้นฐาน (breakpoint, spacing scale, mobile-first) เหมือนกันหมด

---

## ขั้นตอนที่ 510: สรุปและแบบฝึกหัด — ทำ UI บล็อกทั้งหมดให้สวยงามด้วย Bootstrap 5

### 510.1 ประกอบร่างทั้งหมด: `base.html` เวอร์ชันสมบูรณ์ของ Part นี้

```html
<!-- templates/base.html (เวอร์ชันสมบูรณ์หลังจบ Part 051) -->
{% load static %}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Django Mastery Blog{% endblock %}</title>

    <link rel="stylesheet" href="{% static 'blog/css/custom.css' %}">
    {% block extra_head %}{% endblock %}
</head>
<body>
    {% include 'partials/navbar.html' %}

    <main class="container my-4">
        {% include 'partials/messages.html' %}
        {% block content %}
        <p>ยังไม่มีเนื้อหา</p>
        {% endblock %}
    </main>

    {% include 'partials/footer.html' %}

    <script
        src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz"
        crossorigin="anonymous"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

```html
<!-- templates/partials/footer.html -->
<footer class="bg-dark text-white-50 text-center py-4 mt-5">
    <div class="container">
        <p class="mb-0">
            &copy; {% now "Y" %} Django Mastery Course —
            สร้างด้วย Django {{ django_version|default:"5.x" }} และ Bootstrap 5.3
        </p>
    </div>
</footer>
```

### 510.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เชื่อม Bootstrap 5 เข้า Django ได้ทั้งผ่าน CDN (เร็ว เหมาะกับเรียนรู้) และ
  npm + `django-compressor` (ควบคุมเวอร์ชัน/asset เต็มรูปแบบสำหรับ production)
- ✅ ติดตั้งและใช้งาน `django-crispy-forms` + `crispy-bootstrap5` เพื่อ render
  ฟอร์มให้เป็น Bootstrap โดยอัตโนมัติ ทั้งแบบ `|crispy` filter และ
  `FormHelper` + `Layout` สำหรับควบคุม layout ละเอียด
- ✅ เข้าใจระบบ Grid 12-column, `container`/`row`/`col`, breakpoint 6 ระดับ,
  และ utility class ด้าน spacing (`m-*`, `p-*`, `g-*`)
- ✅ นำ Component จริงมาใช้: Navbar responsive พร้อมแฮมเบอร์เกอร์เมนู, Card
  สำหรับ post list, Modal สำหรับยืนยันการลบ, Alert เชื่อมกับ Messages
  Framework, Dropdown และ Badge
- ✅ ใช้ `django-widget-tweaks` เติม class เข้าฟอร์มที่ไม่ต้องการความซับซ้อนของ
  crispy-forms พร้อมเข้าใจว่าควรเลือกใช้ package ไหนเมื่อไร
- ✅ เข้าใจหลักการ Mobile-First และ Responsive Design อย่างลึกซึ้ง รวมถึงวิธี
  ทดสอบด้วย DevTools Device Toolbar และอุปกรณ์จริงในเครือข่ายเดียวกัน
- ✅ Override ตัวแปรสีธีมของ Bootstrap ผ่าน Sass (`$primary`, `$font-family-sans-serif`
  ฯลฯ) และ compile เป็น CSS ไฟล์เดียวด้วย `sass` CLI
- ✅ รู้จัก Tailwind CSS เป็นทางเลือกแบบ Utility-first พร้อมติดตั้งจริงผ่าน
  `django-tailwind`
- ✅ เปรียบเทียบ Bootstrap, Tailwind, Bulma อย่างมีหลักเกณฑ์ เพื่อเลือก
  framework ที่เหมาะกับบริบทของแต่ละโปรเจกต์ในอนาคต

### 510.3 Checklist ก่อนไป Part ถัดไป

- [ ] `base.html` โหลด Bootstrap 5 สำเร็จ (ผ่าน CDN หรือไฟล์ compile เอง) และ
      navbar ยุบเป็นแฮมเบอร์เกอร์เมนูถูกต้องเมื่อย่อหน้าจอ
- [ ] ติดตั้ง `django-crispy-forms` + `crispy-bootstrap5` และตั้งค่า
      `CRISPY_TEMPLATE_PACK` ใน `settings.py` สำเร็จ
- [ ] `ContactForm` render ผ่าน `{% crispy form %}` ได้หน้าตาเป็น Bootstrap
      ครบถ้วน พร้อมแสดง error state (`is-invalid`) เมื่อกรอกข้อมูลผิด
- [ ] `blog/list.html` แสดงการ์ดบทความเป็น grid ที่ปรับจำนวนคอลัมน์ตามขนาดจอ
      (1 คอลัมน์บนมือถือ, 2 บนแท็บเล็ต, 3 บนเดสก์ท็อป)
- [ ] มี Modal ยืนยันการลบบทความที่เปิด/ปิดได้ถูกต้องโดยไม่ต้องเขียน JavaScript
      เอง
- [ ] ติดตั้ง `django-widget-tweaks` และใช้เติม class ให้ `PostFilterForm`
      สำเร็จ
- [ ] ทดสอบเว็บไซต์ผ่าน Chrome DevTools Device Toolbar อย่างน้อย 3 ขนาดจอ
      (มือถือ, แท็บเล็ต, เดสก์ท็อป)
- [ ] Override สี `$primary` ผ่าน `custom.scss` และ compile เห็นสีธีมเปลี่ยน
      ทั้งเว็บไซต์จริง
- [ ] ลองรัน `python manage.py tailwind init` อย่างน้อยหนึ่งครั้งเพื่อดู
      โครงสร้างแอป `theme` ที่ถูกสร้างขึ้น (ไม่จำเป็นต้องใช้งานต่อ)

### 510.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: แปลง `SurveyForm` จาก Part 025 (ที่มี `ChoiceField`,
`RadioSelect`, `CheckboxSelectMultiple`) ให้ render ผ่าน `django-crispy-forms`
โดยเขียน `FormHelper` ใน `__init__` จัด layout ให้ `experience_level` กับ
`satisfaction_rating` อยู่คนละคอลัมน์บนจอ `md` ขึ้นไป (`Row`/`Column`) และ
`interested_topics` อยู่แถวเต็มด้านล่าง พร้อมปุ่ม submit สีเขียว
(`css_class="btn btn-success"`)

**แบบฝึกหัดที่ 2**: สร้างหน้า `blog/post_detail.html` เวอร์ชัน Bootstrap เต็ม
รูปแบบ — จัด layout เป็น 2 คอลัมน์บนจอ `lg` ขึ้นไป (เนื้อหาบทความ 8/12 คอลัมน์
ทางซ้าย, sidebar แสดง "บทความล่าสุด" 4/12 คอลัมน์ทางขวา) แต่ซ้อนกันเป็นคอลัมน์
เดียวเรียงตามลำดับบนมือถือ (ใช้ `col-lg-8` และ `col-lg-4` ร่วมกับ `col-12`
เป็นค่าเริ่มต้น)

**แบบฝึกหัดที่ 3**: เพิ่ม Toast component ของ Bootstrap (ค้นคว้าเอกสารทางการที่
https://getbootstrap.com/docs/5.3/components/toasts/) เพื่อแสดงข้อความ
"บันทึกความคิดเห็นสำเร็จแล้ว" แบบลอยที่มุมขวาบนของจอ หลังจากผู้ใช้ส่ง
`CommentForm` จาก Part 025 สำเร็จ (ใช้ JavaScript เพียงเล็กน้อยเพื่อสั่งเปิด
Toast ผ่าน Bootstrap's JS API: `new bootstrap.Toast(element).show()`)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้างไฟล์ `custom-dark.scss` แยกต่างหากจาก
`custom.scss` ที่ override `$body-bg`, `$body-color`, `$primary` ให้เป็นโทน
มืด (dark theme) แล้วเขียนปุ่มสลับธีมบน navbar ที่สลับการโหลดไฟล์ CSS ระหว่าง
`custom.css` กับ `custom-dark.css` ด้วย JavaScript ง่าย ๆ (เปลี่ยน `href` ของ
`<link>` element และบันทึกค่าที่เลือกไว้ใน `localStorage` เพื่อจำค่าไว้เมื่อ
ผู้ใช้กลับมาเยี่ยมชมใหม่)

### 510.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเลือกระหว่าง `django-crispy-forms` กับ `django-widget-tweaks` แค่
อย่างใดอย่างหนึ่งเท่านั้นหรือไม่ ใช้พร้อมกันในโปรเจกต์เดียวได้ไหม?**
A: ใช้พร้อมกันได้อย่างสมบูรณ์ ไม่มีทางชนกันเลย เพราะทั้งสอง package ทำงานคนละ
ระดับ (`crispy-forms` render ฟอร์มทั้งก้อน ส่วน `widget-tweaks` เติม attribute
ให้ field เดี่ยว ๆ) โปรเจกต์จริงจำนวนมากใช้ `crispy-forms` สำหรับฟอร์มหลัก
และใช้ `widget-tweaks` สำหรับฟอร์มเล็ก ๆ ที่ฝังอยู่ในหน้าอื่นพร้อมกัน ตามที่
สอนไว้ในขั้นตอนที่ 505.6

**Q: ทำไมต้องใส่ `integrity` attribute เวลาโหลด Bootstrap จาก CDN ในเมื่อ
เว็บไซต์อื่น ๆ จำนวนมากไม่ใส่กัน?**
A: เว็บไซต์จำนวนมากละเลยเรื่องนี้เพราะดูยุ่งยาก แต่ในทางเทคนิคควรใส่เสมอ
โดยเฉพาะเว็บที่รับข้อมูลอ่อนไหว (login, ข้อมูลบัตรเครดิต) เพราะ `integrity`
ป้องกันกรณี CDN ถูกแฮ็กแล้วสลับไฟล์ JavaScript เป็นโค้ดอันตราย (**Supply Chain
Attack**) หลักสูตรนี้สอนให้ใส่เป็นนิสัยที่ดีตั้งแต่ต้น เพราะเป็นสิ่งที่แยก
มืออาชีพที่ใส่ใจความปลอดภัยออกจากมือใหม่ที่ก็อปโค้ดมาแบบไม่ได้อ่านรายละเอียด

**Q: ควรใช้ Bootstrap เวอร์ชันไหนถ้าโปรเจกต์เก่ายังใช้ Bootstrap 4 อยู่ ควร
อัปเกรดเป็น 5 เลยไหม?**
A: ควรอัปเกรดถ้าเป็นไปได้ เพราะ Bootstrap 4 พึ่งพา jQuery ซึ่งเป็นเทคโนโลยีที่
ล้าสมัยและเพิ่มขนาดไฟล์โดยไม่จำเป็นในปี 2026 แต่การอัปเกรดมี breaking changes
พอสมควร (เช่น class `.ml-*`/`.mr-*` เปลี่ยนเป็น `.ms-*`/`.me-*` เพื่อรองรับภาษา
ที่เขียนจากขวาไปซ้าย, `.form-group` ถูกยกเลิกไปใช้ `.mb-3` แทน) ควรอ่าน
Migration Guide อย่างเป็นทางการที่ https://getbootstrap.com/docs/5.3/migration/
ก่อนอัปเกรดโปรเจกต์จริงเสมอ และทดสอบทุกหน้าอย่างละเอียดหลังอัปเกรด

**Q: จำเป็นต้องรู้ CSS/Sass ลึกแค่ไหนถึงจะ custom ธีม Bootstrap ได้ดี?**
A: ไม่จำเป็นต้องเชี่ยวชาญ Sass ทั้งหมด แค่เข้าใจแนวคิดตัวแปร (`$variable-name:
value;`) และลำดับการ `@import` (ตัวแปรต้องมาก่อนเสมอตามขั้นตอนที่ 507.3)
ก็เพียงพอสำหรับ override สีธีมและฟอนต์พื้นฐานแล้ว สำหรับการปรับแต่งที่ลึกกว่านั้น
(เช่นเขียน mixin เอง, ปรับ component ระดับโครงสร้าง) ค่อยศึกษาเอกสาร Sass
เพิ่มเติมเมื่อถึงจุดที่ต้องการจริง ๆ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 052: Django กับ JavaScript และ Fetch API** จะพาคุณกลับไปหา `blog` API
ที่สร้างและทดสอบไว้อย่างละเอียดตลอด Phase 5 (Part 039-050) แล้วเรียก API นั้น
โดยตรงจากฝั่ง client ด้วย JavaScript `fetch()` — คุณจะได้เห็น JWT
authentication, filtering, pagination ที่เรียนมาทั้งหมดถูกใช้งานจริงจากฝั่ง
frontend เป็นครั้งแรก พร้อมเรียนรู้การอัปเดต DOM แบบไม่ต้องรีเฟรชหน้าทั้งหน้า
(AJAX pattern สมัยใหม่), การจัดการ CSRF token ร่วมกับ `fetch()`, และการแสดงผล
loading state ด้วย Bootstrap Spinner component ที่ผูกเข้ากับ UI ที่สร้างไว้ใน
Part นี้

เตรียม `base.html`, navbar, และการ์ดบทความเวอร์ชัน Bootstrap ที่ทำไว้ใน Part
นี้ให้พร้อม เดี๋ยว Part 052 จะเพิ่มปุ่ม "โหลดเพิ่มเติม" ที่ดึงบทความหน้าถัดไป
จาก API ด้วย JavaScript ล้วน ๆ โดยไม่ต้องโหลดหน้าเว็บใหม่เลย!
