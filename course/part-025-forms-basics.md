# Part 025: Django Forms เบื้องต้น

> **ขั้นตอนที่ 241-250 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: เข้าใจ **`django.forms.Form`** ตั้งแต่การประกาศ field types
> ที่ใช้บ่อยที่สุด, การ render ฟอร์มในเทมเพลตด้วยหลายวิธี (`{{ form }}`, `.as_p`,
> `.as_table`, `.as_div`), วงจรชีวิตของฟอร์ม (unbound → bound → validate),
> การเขียน Custom Validation ทั้งระดับ field เดียวและข้าม field, การใช้ Widget
> เพื่อควบคุมหน้าตา HTML ของฟอร์ม, การตั้งค่า `initial` data, การรับไฟล์อัปโหลด,
> และการเชื่อม CSRF Protection เข้ากับการส่งฟอร์มแบบ AJAX เมื่อจบ Part นี้ คุณจะสร้าง
> `CommentForm` แบบ **plain `forms.Form`** (ยังไม่ใช่ `ModelForm` — เรื่องนั้นเก็บไว้
> เจาะลึกเต็มรูปแบบใน **Part 026**) พร้อม view ที่รับข้อมูลจากฟอร์มไปสร้าง object
> `Comment` เองด้วยมือ ซึ่งจะทำให้คุณเข้าใจ "เบื้องหลัง" ของ `ModelForm` อย่างถ่องแท้
> ก่อนที่จะไปใช้ทางลัดใน Part ถัดไป

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 241: `forms.Form` class เบื้องต้น — Field Types ที่ใช้บ่อย เทียบกับ Field ใน Model
- ขั้นตอนที่ 242: Render Form ใน Template — `{{ form }}`, `.as_p`, `.as_table`, `.as_div` (Django 5.0+), การ render ทีละ field เอง
- ขั้นตอนที่ 243: วงจรชีวิตของ Form — Bound vs Unbound, `is_valid()`, `cleaned_data`, `errors`
- ขั้นตอนที่ 244: Widgets เบื้องต้น — `Textarea`, `Select`, `RadioSelect`, `CheckboxInput`, การกำหนด widget attrs
- ขั้นตอนที่ 245: Custom Validation ต่อ Field ด้วย `clean_<fieldname>()`
- ขั้นตอนที่ 246: Custom Validation ข้าม Field ด้วย `clean()`
- ขั้นตอนที่ 247: `initial` Data และการสร้างฟอร์มที่มีค่าเริ่มต้น
- ขั้นตอนที่ 248: จัดการ File Upload ในฟอร์ม — `enctype`, `request.FILES`, `FileField`/`ImageField`
- ขั้นตอนที่ 249: CSRF Protection ในฟอร์ม (เชื่อมกับ Part 008) และ AJAX Form Submission เบื้องต้น
- ขั้นตอนที่ 250: สรุปและแบบฝึกหัด — สร้าง `CommentForm` (plain Form) พร้อม View ที่สร้าง `Comment` เอง

---

## ขั้นตอนที่ 241: `forms.Form` class เบื้องต้น — Field Types ที่ใช้บ่อย เทียบกับ Field ใน Model

### 241.1 ทำไมต้องมี Django Forms ทั้งที่มี Model อยู่แล้ว

ใน Phase 2 คุณเรียนรู้ Model fields (`CharField`, `IntegerField`, `BooleanField` ฯลฯ
ใน `models.py`) ไปแล้ว หลายคนอาจสงสัยว่า "ในเมื่อ Model กำหนดชนิดข้อมูลไว้แล้ว
ทำไม Django ยังต้องมี Form fields อีกชุดหนึ่งที่ชื่อคล้ายกันมาก (`CharField`,
`IntegerField`, `BooleanField` ก็มีทั้งคู่)?"

คำตอบคือ **Model กับ Form ทำหน้าที่คนละชั้น (layer) ของระบบ**:

```
┌──────────────┐   HTML input   ┌──────────────────┐   Python value   ┌──────────────┐
│   Browser    │ ──────────────>│  forms.Form       │ ────────────────>│  Database    │
│ (ข้อความดิบ  │   (string      │  (แปลง + ตรวจสอบ  │  (int, date,     │  ผ่าน Model  │
│  ทุกอย่าง)   │   เสมอ)        │   ความถูกต้อง)    │   Decimal ฯลฯ)   │              │
└──────────────┘                └──────────────────┘                   └──────────────┘
```

- **Model field** อธิบายว่า **ฐานข้อมูลควรเก็บข้อมูลอย่างไร** เช่น คอลัมน์ชนิดไหน
  ยาวสูงสุดเท่าไร มี default หรือไม่ นี่คือ "ความจริงถาวร" ของโครงสร้างข้อมูล
- **Form field** อธิบายว่า **ข้อมูลที่รับเข้ามาจากผู้ใช้ (ซึ่งมาเป็น string ดิบเสมอ
  ไม่ว่าผู้ใช้จะพิมพ์ตัวเลขหรือวันที่ก็ตาม เพราะ HTTP ส่งทุกอย่างเป็นข้อความ)
  ควรถูกแปลงและตรวจสอบความถูกต้องอย่างไรก่อนจะนำไปใช้ต่อ**

พูดง่าย ๆ: **Model field คุยกับฐานข้อมูล ส่วน Form field คุยกับผู้ใช้** ทั้งสองระบบ
แยกอิสระจากกันโดยสิ้นเชิง คุณสามารถมี `forms.Form` ที่ไม่เกี่ยวข้องกับ Model เลยก็ได้
(เช่น ฟอร์มค้นหา, ฟอร์มติดต่อ, ฟอร์ม login) นี่คือเหตุผลที่ Part นี้จะสอน
**`forms.Form` แบบเดี่ยว ๆ ก่อน** ให้คุณเข้าใจกลไกจริง ๆ ของระบบฟอร์ม ก่อนที่ Part 026
จะแนะนำ `ModelForm` ซึ่งเป็นทางลัดที่สร้าง Form field จาก Model field ให้อัตโนมัติ

### 241.2 สร้างฟอร์มแรก: `ContactForm`

Django form ทุกตัวสืบทอดจาก `django.forms.Form` และประกาศ field เป็น class attribute
เหมือนกับที่ Model ประกาศ field:

```python
# blog/forms.py
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, label="ชื่อของคุณ")
    email = forms.EmailField(label="อีเมล")
    subject = forms.CharField(max_length=150, label="หัวข้อ")
    message = forms.CharField(widget=forms.Textarea, label="ข้อความ")
    newsletter_opt_in = forms.BooleanField(
        required=False,
        label="สมัครรับจดหมายข่าว",
    )
```

สังเกตธรรมเนียมสำคัญ: **ไฟล์ฟอร์มทั้งหมดของแอปจะอยู่ใน `<app>/forms.py`** เหมือนที่
`models.py` เก็บ Model และ `views.py` เก็บ View — Django ไม่ได้บังคับชื่อไฟล์นี้
(ไม่เหมือน `models.py` ที่ถูกอ้างอิงโดยระบบ migration) แต่เป็นธรรมเนียมที่ชุมชน
Django ยึดถือกันแทบ 100% ของโปรเจกต์จริง

### 241.3 Field Types ที่ใช้บ่อยที่สุด

| Form Field | ค่าที่ได้ใน `cleaned_data` | Widget เริ่มต้น | ตัวอย่างการใช้งาน |
|---|---|---|---|
| `CharField` | `str` | `TextInput` | ชื่อ, หัวข้อ, ข้อความสั้น |
| `EmailField` | `str` (ตรวจรูปแบบอีเมลแล้ว) | `EmailInput` | ที่อยู่อีเมล |
| `IntegerField` | `int` | `NumberInput` | อายุ, จำนวนสินค้า |
| `DecimalField` | `Decimal` | `NumberInput` | ราคา, จำนวนเงิน |
| `BooleanField` | `bool` (`True`/`False`) | `CheckboxInput` | เช็คบ็อกซ์ยอมรับเงื่อนไข |
| `ChoiceField` | ค่าที่เลือก (มักเป็น `str`) | `Select` | เลือกหมวดหมู่, เลือกสถานะ |
| `MultipleChoiceField` | `list` ของค่าที่เลือก | `SelectMultiple` | เลือกได้หลายรายการ |
| `DateField` | `datetime.date` | `DateInput` | วันเกิด, วันที่นัดหมาย |
| `DateTimeField` | `datetime.datetime` | `DateTimeInput` | วันเวลานัดหมาย |
| `URLField` | `str` (ตรวจรูปแบบ URL แล้ว) | `URLInput` | ลิงก์เว็บไซต์ |
| `FileField` | `UploadedFile` | `ClearableFileInput` | อัปโหลดไฟล์ทั่วไป |
| `ImageField` | `UploadedFile` (ตรวจว่าเป็นรูปจริง) | `ClearableFileInput` | อัปโหลดรูปโปรไฟล์ |

ตัวอย่าง `ChoiceField` ที่จะใช้บ่อยเมื่อทำฟอร์มค้นหา/กรองข้อมูล:

```python
class PostFilterForm(forms.Form):
    STATUS_CHOICES = [
        ("", "ทั้งหมด"),
        ("published", "เผยแพร่แล้ว"),
        ("draft", "ฉบับร่าง"),
    ]
    status = forms.ChoiceField(choices=STATUS_CHOICES, required=False, label="สถานะ")
    keyword = forms.CharField(max_length=100, required=False, label="คำค้นหา")
```

### 241.4 Model Field vs Form Field: คนละวัตถุประสงค์ แม้ชื่อเหมือนกัน

ตารางนี้คือหัวใจของขั้นตอนนี้ — เปรียบเทียบ `CharField` ทั้งสองแบบ (จงใจเลือกชื่อ
เดียวกันเพื่อให้เห็นความต่างชัดที่สุด):

| ประเด็น | `models.CharField` | `forms.CharField` |
|---|---|---|
| อยู่ในไฟล์ | `models.py` | `forms.py` |
| หน้าที่หลัก | นิยามคอลัมน์ในฐานข้อมูล (`VARCHAR`) | รับ + validate ข้อความจาก HTML input |
| พารามิเตอร์บังคับ | `max_length` (บังคับเสมอ ใช้สร้างคอลัมน์ DB) | ไม่บังคับพารามิเตอร์ใด ๆ (`max_length` เป็น optional และใช้เพื่อ validate เท่านั้น) |
| ทำงานกับฐานข้อมูลโดยตรงไหม | ใช่ — ผ่าน migration | ไม่ — ไม่รู้จักฐานข้อมูลเลย |
| รู้จัก HTML widget ไหม | ไม่ (ไม่มีแนวคิดเรื่อง widget) | ใช่ — ทุก field ผูกกับ widget เสมอ |
| ใช้นอกบริบทเว็บได้ไหม (เช่น script, management command) | ใช้ได้ตามปกติ (เป็นส่วนหนึ่งของ Model) | ใช้ได้ (ฟอร์มไม่ผูกกับ HTTP request โดยตรง) แต่ถูกออกแบบมาเพื่อรับข้อมูลดิบจากผู้ใช้เป็นหลัก |
| ตัวอย่างการสร้าง | `title = models.CharField(max_length=200)` | `title = forms.CharField(max_length=200)` |

**ข้อสรุปสำคัญที่ต้องจำ**: ทั้งสอง class นี้**ไม่มีความสัมพันธ์กันในโค้ดเลย**
(`django.db.models.CharField` และ `django.forms.CharField` เป็นคนละ class คนละ
module กันสิ้นเชิง) ที่ชื่อเหมือนกันเป็นเพียงการตั้งชื่อให้สื่อความหมายคล้ายกัน
เพื่อให้นักพัฒนาจำง่าย ไม่ใช่เพราะ inherit จากกันหรือใช้ mechanism เดียวกัน — Django
เพียงแค่ให้ `ModelForm` (ที่จะเรียนใน Part 026) เป็นตัวเชื่อมสร้าง Form field
จาก Model field โดยอัตโนมัติในภายหลัง แต่ตอนนี้เรากำลังเรียน `forms.Form` แบบเดี่ยว
ที่ไม่ต้องพึ่ง Model เลยแม้แต่น้อย

### 241.5 ทดสอบฟอร์มใน Django Shell

ก่อนจะเอาฟอร์มไปต่อกับ View/Template ลองทดสอบพฤติกรรมพื้นฐานผ่าน shell ก่อน
(`python manage.py shell`) เพื่อให้เห็นภาพว่า field ทำหน้าที่ "แปลง + ตรวจสอบ"
อย่างไร:

```python
>>> from blog.forms import ContactForm
>>> form = ContactForm(data={
...     "name": "สมชาย ใจดี",
...     "email": "somchai@example.com",
...     "subject": "สอบถามเรื่องคอร์ส",
...     "message": "อยากทราบว่าคอร์สนี้เหมาะกับมือใหม่ไหมครับ",
...     "newsletter_opt_in": "on",
... })
>>> form.is_valid()
True
>>> form.cleaned_data
{'name': 'สมชาย ใจดี', 'email': 'somchai@example.com', 'subject': 'สอบถามเรื่องคอร์ส',
 'message': 'อยากทราบว่าคอร์สนี้เหมาะกับมือใหม่ไหมครับ', 'newsletter_opt_in': True}
```

ลองส่งข้อมูลผิด ๆ (อีเมลไม่ถูกรูปแบบ) เพื่อดูว่า `EmailField` ทำงานอย่างไร:

```python
>>> bad_form = ContactForm(data={
...     "name": "สมหญิง",
...     "email": "not-an-email",
...     "subject": "ทดสอบ",
...     "message": "ทดสอบข้อความ",
... })
>>> bad_form.is_valid()
False
>>> bad_form.errors
{'email': ['Enter a valid email address.']}
```

สังเกตว่า **`ContactForm` ไม่รู้จักฐานข้อมูล ไม่รู้จัก HTTP request เลยด้วยซ้ำ**
มันเป็นเพียง Python object ที่รับ dict เข้ามา แล้วตรวจสอบ/แปลงข้อมูลให้ — นี่คือ
แก่นแท้ของ `forms.Form` ที่เราจะต่อยอดไปเรื่อย ๆ ตลอด Part นี้

---

## ขั้นตอนที่ 242: Render Form ใน Template — `{{ form }}`, `.as_p`, `.as_table`, `.as_div`

### 242.1 เตรียม View ง่าย ๆ เพื่อแสดงฟอร์ม

ก่อนพูดถึงวิธี render ทั้งหมด เราต้องมี View ที่ส่งฟอร์มเข้า context ก่อน (ยังไม่ต้อง
สนใจการ validate ตอนนี้ — เก็บไว้ในขั้นตอนที่ 243):

```python
# blog/views.py
from django.shortcuts import render
from .forms import ContactForm


def contact_view(request):
    form = ContactForm()
    return render(request, "blog/contact.html", {"form": form})
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"

urlpatterns = [
    # ... URL เดิมจาก Part ก่อนหน้า ...
    path("contact/", views.contact_view, name="contact"),
]
```

### 242.2 วิธีที่ง่ายที่สุด: `{{ form }}`

```html
<!-- blog/templates/blog/contact.html -->
{% extends 'base.html' %}

{% block content %}
<h1>ติดต่อเรา</h1>
<form method="post">
    {% csrf_token %}
    {{ form }}
    <button type="submit">ส่งข้อความ</button>
</form>
{% endblock %}
```

`{{ form }}` เป็นวิธีที่สั้นที่สุด แต่ **ตั้งแต่ Django 5.0 เป็นต้นไป ค่าเริ่มต้นของ
`{{ form }}` จะ render ออกมาเป็นโครงสร้างแบบ `<div>`** (เทียบเท่ากับเรียก
`form.as_div()` ตรง ๆ) ซึ่งเปลี่ยนจากพฤติกรรมเดิมของ Django เวอร์ชันก่อนหน้าที่
render แบบ "ไม่มี wrapper" (คล้าย `as_table` แต่ตัด `<table>` ออก) — ถ้าโปรเจกต์ของ
คุณอัปเกรดจาก Django เวอร์ชันเก่ามาเป็น 5.x หน้าตาฟอร์มอาจเปลี่ยนไปโดยไม่ได้แก้โค้ด
เทมเพลตเลยแม้แต่บรรทัดเดียว จุดนี้เป็นสิ่งที่มืออาชีพต้องรู้เมื่ออัปเกรดเวอร์ชัน

### 242.3 `.as_p`, `.as_table`, `.as_div`: เลือก Wrapper ที่ต้องการชัดเจน

Django มอบ method สำเร็จรูป 3 ตัวให้เลือก render แบบที่ต้องการอย่างชัดเจน (ไม่ต้อง
พึ่งพฤติกรรม default ที่อาจเปลี่ยนไปตามเวอร์ชัน):

```html
{{ form.as_p }}      <!-- แต่ละ field ห่อด้วย <p> -->
{{ form.as_table }}  <!-- แต่ละ field เป็น <tr><th>...</th><td>...</td></tr> (ต้องมี <table> ครอบเอง) -->
{{ form.as_div }}    <!-- แต่ละ field ห่อด้วย <div> (Django 5.0+) -->
```

ตัวอย่างเต็มของแต่ละแบบ:

```html
<!-- as_p: ใช้บ่อยที่สุดสำหรับฟอร์มเรียบง่าย ไม่ต้องจัด layout ซับซ้อน -->
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">ส่งข้อความ</button>
</form>
```

```html
<!-- as_table: ต้องมี <table> ครอบเองเสมอ ไม่เช่นนั้น HTML จะผิดโครงสร้าง -->
<form method="post">
    {% csrf_token %}
    <table>
        {{ form.as_table }}
    </table>
    <button type="submit">ส่งข้อความ</button>
</form>
```

```html
<!-- as_div: เหมาะกับ CSS Framework สมัยใหม่ (Flexbox/Grid/Tailwind) ที่ทำงานกับ <div> ได้คล่องกว่า <table> -->
<form method="post">
    {% csrf_token %}
    {{ form.as_div }}
    <button type="submit">ส่งข้อความ</button>
</form>
```

### 242.4 ตารางเปรียบเทียบ Output ของแต่ละวิธี

| Method | โครงสร้าง HTML ต่อ field | ต้องมี wrapper ล้อมเองไหม | เหมาะกับ |
|---|---|---|---|
| `{{ form }}` (Django 5.0+) | เหมือน `as_div` | ไม่ต้อง | ฟอร์มทั่วไปที่ไม่ต้องคุมหน้าตาละเอียด |
| `.as_p` | `<p><label>...</label> <input>... <br>(errors)</p>` | ไม่ต้อง | ฟอร์มง่าย ๆ, prototype เร็ว |
| `.as_table` | `<tr><th><label>...</label></th><td><input>...(errors)</td></tr>` | **ต้อง** ครอบด้วย `<table>` เอง | เอกสารเก่า/ฟอร์มที่ยังใช้ตาราง |
| `.as_div` | `<div><label>...</label><input>...(errors)</div>` | ไม่ต้อง | โปรเจกต์ที่ใช้ CSS Framework สมัยใหม่ (Bootstrap 5, Tailwind) |

**คำแนะนำของหลักสูตรนี้**: ใช้ `.as_p` หรือ `.as_div` สำหรับ prototype และฟอร์ม
เรียบง่าย แต่สำหรับหน้าที่ต้องคุม layout ให้สวยงามระดับ production จริง (เช่น
จัดวาง field เป็น 2 คอลัมน์, ใส่ icon ข้าง input) ให้ข้ามไปใช้วิธีถัดไปคือ
**render ทีละ field เอง**

### 242.5 Render ทีละ Field เอง: ควบคุม HTML ได้เต็มที่

เมื่อ Wrapper สำเร็จรูปไม่พอ คุณสามารถเข้าถึง field แต่ละตัวผ่านชื่อ (dot lookup
แบบเดียวกับที่เรียนใน Part 008) แล้วจัด HTML เองทั้งหมด:

```html
<form method="post">
    {% csrf_token %}

    <div class="form-group">
        {{ form.name.label_tag }}
        {{ form.name }}
        {% if form.name.errors %}
            <div class="form-error">{{ form.name.errors }}</div>
        {% endif %}
    </div>

    <div class="form-group">
        {{ form.email.label_tag }}
        {{ form.email }}
        {% if form.email.errors %}
            <div class="form-error">{{ form.email.errors }}</div>
        {% endif %}
    </div>

    <button type="submit">ส่งข้อความ</button>
</form>
```

หรือใช้ `{% for %}` วนลูปทุก field โดยไม่ต้องเขียนชื่อ field ซ้ำทีละตัว — สะดวกเมื่อ
ฟอร์มมี field จำนวนมากและทุก field ใช้โครง HTML เดียวกัน:

```html
<form method="post">
    {% csrf_token %}
    {% for field in form %}
        <div class="form-group">
            {{ field.label_tag }}
            {{ field }}
            {% if field.help_text %}
                <small class="form-help">{{ field.help_text }}</small>
            {% endif %}
            {% for error in field.errors %}
                <div class="form-error">{{ error }}</div>
            {% endfor %}
        </div>
    {% endfor %}

    {# non_field_errors จะเจาะลึกในขั้นตอนที่ 243 และ 246 #}
    {% if form.non_field_errors %}
        <div class="form-error form-error--general">{{ form.non_field_errors }}</div>
    {% endif %}

    <button type="submit">ส่งข้อความ</button>
</form>
```

| Attribute ของ `field` (BoundField) | ความหมาย |
|---|---|
| `field.label_tag` | เรนเดอร์ `<label>` พร้อมเชื่อม `for="id_xxx"` ให้อัตโนมัติ |
| `field` (เฉย ๆ) | เรนเดอร์แค่ตัว `<input>`/`<select>`/`<textarea>` |
| `field.errors` | list ของข้อความ error เฉพาะ field นี้ |
| `field.help_text` | ข้อความอธิบายเพิ่มเติม (ตั้งค่าตอนประกาศ field ด้วย `help_text="..."`) |
| `field.id_for_label` | ค่า `id` ที่ Django ตั้งให้ input (ปกติคือ `id_<field_name>`) |

การ render ทีละ field เองคือแนวทางที่โปรเจกต์ระดับ production ส่วนใหญ่ใช้จริง
เพราะให้ความยืดหยุ่นสูงสุดในการจัด CSS/Layout โดยไม่ต้อง fight กับ HTML ที่ Django
generate มาให้แบบสำเร็จรูป

---

## ขั้นตอนที่ 243: วงจรชีวิตของ Form — Bound vs Unbound, `is_valid()`, `cleaned_data`, `errors`

### 243.1 Unbound Form: ฟอร์มที่ยังไม่มีข้อมูลผูกอยู่

เมื่อสร้างฟอร์มโดยไม่ส่ง `data` เข้าไปเลย (`ContactForm()`) ฟอร์มนั้นจะอยู่ในสถานะ
**unbound** — หมายถึง "ยังไม่ถูกผูกกับข้อมูลใด ๆ จากผู้ใช้" เหมาะสำหรับแสดงฟอร์ม
เปล่า ๆ ให้ผู้ใช้กรอกครั้งแรก:

```python
>>> from blog.forms import ContactForm
>>> form = ContactForm()
>>> form.is_bound
False
```

ฟอร์ม unbound **ไม่มีทาง valid หรือ invalid ได้** เพราะยังไม่มีอะไรให้ตรวจสอบ
ถ้าเรียก `is_valid()` กับฟอร์ม unbound จะได้ `False` เสมอ (ไม่ raise exception
แต่ก็ไม่มีความหมายอะไรในทางปฏิบัติ — Django เตือนเรื่องนี้ในเอกสารว่าไม่ควรเรียก
`is_valid()` กับฟอร์ม unbound)

### 243.2 Bound Form: ฟอร์มที่มีข้อมูลผูกอยู่แล้ว

เมื่อส่ง `data=` (และ/หรือ `files=` สำหรับไฟล์ — ขั้นตอนที่ 248) เข้าไปตอนสร้าง
ฟอร์ม ฟอร์มนั้นจะกลายเป็น **bound** ทันที ไม่ว่าข้อมูลจะถูกต้องหรือไม่ก็ตาม:

```python
>>> form = ContactForm(data={"name": "", "email": "bad-email"})
>>> form.is_bound
True
```

| สถานะ | สร้างอย่างไร | ใช้เมื่อไร |
|---|---|---|
| **Unbound** | `ContactForm()` (ไม่ส่ง `data`) | แสดงฟอร์มเปล่าให้ผู้ใช้กรอกครั้งแรก (GET request) |
| **Bound** | `ContactForm(data=request.POST)` | ตรวจสอบข้อมูลที่ผู้ใช้ส่งมา (POST request) |

### 243.3 `is_valid()` และ `cleaned_data`: หัวใจของกระบวนการ Validate

`is_valid()` เป็น method ที่ทำสิ่งต่อไปนี้ตามลำดับสำหรับฟอร์ม bound (และคืนค่า
`False` ทันทีถ้าฟอร์ม unbound):

1. รัน validation ของแต่ละ field ทีละตัว (ตรวจ `required`, `max_length`,
   รูปแบบอีเมล ฯลฯ ตามชนิด field)
2. รัน `clean_<fieldname>()` ของแต่ละ field ถ้ามี (ขั้นตอนที่ 245)
3. รัน `clean()` ระดับฟอร์มทั้งก้อน ถ้ามี (ขั้นตอนที่ 246)
4. ถ้าไม่มี error เกิดขึ้นเลยตลอดกระบวนการ → คืน `True` และเติมค่าที่ผ่านการแปลง
   และตรวจสอบแล้วทั้งหมดไว้ใน `form.cleaned_data` (เป็น `dict`)
5. ถ้ามี error → คืน `False` และเก็บ error ไว้ใน `form.errors`

```python
>>> form = ContactForm(data={
...     "name": "ทดสอบ",
...     "email": "test@example.com",
...     "subject": "หัวข้อ",
...     "message": "ข้อความทดสอบ",
... })
>>> if form.is_valid():
...     print(form.cleaned_data)
... else:
...     print(form.errors)
{'name': 'ทดสอบ', 'email': 'test@example.com', 'subject': 'หัวข้อ',
 'message': 'ข้อความทดสอบ', 'newsletter_opt_in': False}
```

**กฎเหล็ก**: **ห้ามเข้าถึง `form.cleaned_data` ก่อนเรียก `is_valid()`** เพราะ
`cleaned_data` จะยังไม่ถูกสร้างขึ้นเลยจนกว่า `is_valid()` จะถูกเรียก (หรือถ้า
`is_valid()` คืน `False` บาง field อาจหายไปจาก `cleaned_data` เพราะ validate
ไม่ผ่าน) โค้ดที่เข้าถึง `cleaned_data` โดยไม่เช็ค `is_valid()` ก่อน คือบั๊กที่พบบ่อย
ที่สุดของมือใหม่ที่เขียน Django Forms

### 243.4 `form.errors`: โครงสร้างของ Error ทั้งหมด

`form.errors` เป็น dict-like object ที่ key คือชื่อ field และ value คือ list ของ
ข้อความ error ของ field นั้น รวมถึง key พิเศษ `__all__` สำหรับ error ที่ไม่ผูกกับ
field ใด field หนึ่ง (จะเจาะลึกใน 246.3):

```python
>>> bad_form = ContactForm(data={"name": "", "email": "not-an-email"})
>>> bad_form.is_valid()
False
>>> bad_form.errors
{'name': ['This field is required.'],
 'email': ['Enter a valid email address.'],
 'subject': ['This field is required.'],
 'message': ['This field is required.']}
>>> bad_form.errors.as_data()   # ได้ ValidationError object จริง เผื่อต้องเช็ค .code
{'name': [ValidationError(['This field is required.'])], ...}
```

ในเทมเพลต แสดง error รายตัวกับ error ของฟอร์มทั้งก้อนแยกกันได้:

```html
{{ form.email.errors }}       <!-- error เฉพาะ field email -->
{{ form.non_field_errors }}   <!-- error ที่ไม่ผูกกับ field ใด (มาจาก __all__) -->
```

### 243.5 Pattern มาตรฐาน: View ที่จัดการ GET และ POST ในฟังก์ชันเดียว

นี่คือรูปแบบที่คุณจะพิมพ์ซ้ำหลายร้อยครั้งตลอดอาชีพนักพัฒนา Django — ควรจำให้ขึ้นใจ:

```python
# blog/views.py
from django.shortcuts import render, redirect
from .forms import ContactForm


def contact_view(request):
    if request.method == "POST":
        form = ContactForm(request.POST)   # bound form: ผูกกับข้อมูลที่ส่งมา
        if form.is_valid():
            # ณ จุดนี้ form.cleaned_data ปลอดภัย 100% ที่จะใช้งานต่อ
            data = form.cleaned_data
            print(f"ได้รับข้อความจาก {data['name']} <{data['email']}>: {data['subject']}")
            # ในโปรเจกต์จริงอาจส่งอีเมลแจ้งเตือน หรือบันทึกลงฐานข้อมูล
            return redirect("blog:contact_success")
        # ถ้า is_valid() เป็น False: ตกลงมาถึงบรรทัดนี้ ปล่อยให้ render()
        # ด้านล่างแสดงฟอร์มเดิมพร้อม error ที่เกิดขึ้น (ไม่ redirect)
    else:
        form = ContactForm()   # unbound form: ฟอร์มเปล่าสำหรับ GET request

    return render(request, "blog/contact.html", {"form": form})


def contact_success(request):
    return render(request, "blog/contact_success.html")
```

สังเกตโครงสร้างที่สำคัญ 3 จุด:

1. **GET → unbound form** เสมอ (แสดงฟอร์มเปล่า)
2. **POST → bound form** ด้วย `ContactForm(request.POST)` (ไม่ต้องใช้ keyword
   `data=` ก็ได้ เพราะเป็น positional argument แรกของ `Form.__init__`)
3. **หลัง `is_valid()` สำเร็จ ให้ `redirect()` เสมอ ไม่ใช่ `render()` ตรง ๆ** —
   หลักการนี้เรียกว่า **Post/Redirect/Get (PRG) Pattern** ป้องกันปัญหาที่ผู้ใช้กด
   refresh หน้าเว็บแล้วฟอร์มเดิมถูกส่งซ้ำโดยไม่ตั้งใจ (browser จะถามยืนยันการส่ง
   ข้อมูลซ้ำถ้าไม่ redirect) — หลักการนี้จะเจาะลึกอีกครั้งเมื่อเรียนเรื่อง
   Messages Framework ใน Phase ถัดไป

| สถานการณ์ | `request.method` | Form สร้างแบบไหน | ผลลัพธ์ที่คาดหวัง |
|---|---|---|---|
| ผู้ใช้เพิ่งเข้าหน้าฟอร์มครั้งแรก | `GET` | Unbound (`ContactForm()`) | เห็นฟอร์มเปล่า |
| ผู้ใช้กด Submit ด้วยข้อมูลถูกต้อง | `POST` | Bound + valid | `redirect()` ไปหน้าอื่น |
| ผู้ใช้กด Submit ด้วยข้อมูลผิด | `POST` | Bound + invalid | `render()` ฟอร์มเดิมพร้อม error |

---

## ขั้นตอนที่ 244: Widgets เบื้องต้น

### 244.1 Widget คืออะไร ต่างจาก Field อย่างไร

**Field** รับผิดชอบเรื่อง "ข้อมูล" — ชนิดข้อมูล, การแปลงค่า, การตรวจสอบ (validation)
ส่วน **Widget** รับผิดชอบเรื่อง "หน้าตา HTML" ล้วน ๆ — จะ render ออกมาเป็น
`<input type="text">`, `<textarea>`, `<select>` หรืออะไรก็ตาม

field หนึ่งตัวมี widget เริ่มต้นของมันเองอยู่แล้ว (ตามตารางในขั้นตอนที่ 241.3)
แต่คุณสามารถ **เปลี่ยน widget ได้อิสระ โดยไม่กระทบ logic การ validate เลย**
เพราะสอง concept นี้แยกกันเด็ดขาด ตัวอย่างเช่น `CharField` ปกติ render เป็น
`<input type="text">` แต่ถ้าอยากให้กลายเป็นกล่องข้อความหลายบรรทัด สามารถสั่งให้
ใช้ `Textarea` widget แทนได้ทันที โดยที่ field ยังคงเป็น `CharField` เหมือนเดิม
ทุกประการ (validate เหมือนเดิม, `cleaned_data` ได้ `str` เหมือนเดิม)

```python
message = forms.CharField(widget=forms.Textarea)   # field เดิม แค่เปลี่ยนหน้าตา
```

### 244.2 Widget ที่ใช้บ่อยที่สุด

| Widget | Render เป็น HTML | ใช้คู่กับ Field ที่พบบ่อย |
|---|---|---|
| `TextInput` | `<input type="text">` | `CharField` (ค่าเริ่มต้น) |
| `Textarea` | `<textarea>` | `CharField` ที่ต้องการหลายบรรทัด |
| `EmailInput` | `<input type="email">` | `EmailField` (ค่าเริ่มต้น) |
| `NumberInput` | `<input type="number">` | `IntegerField`, `DecimalField` (ค่าเริ่มต้น) |
| `PasswordInput` | `<input type="password">` | `CharField` สำหรับรหัสผ่าน |
| `CheckboxInput` | `<input type="checkbox">` | `BooleanField` (ค่าเริ่มต้น) |
| `Select` | `<select><option>...</option></select>` | `ChoiceField` (ค่าเริ่มต้น) |
| `SelectMultiple` | `<select multiple>` | `MultipleChoiceField` (ค่าเริ่มต้น) |
| `RadioSelect` | `<input type="radio">` หลายตัว | `ChoiceField` (ทางเลือกแทน `Select`) |
| `CheckboxSelectMultiple` | `<input type="checkbox">` หลายตัว | `MultipleChoiceField` (ทางเลือกแทน `SelectMultiple`) |
| `DateInput` | `<input type="text">` (หรือ `type="date"` ถ้ากำหนด `attrs`) | `DateField` |
| `ClearableFileInput` | `<input type="file">` พร้อมปุ่มล้างไฟล์เดิม | `FileField`, `ImageField` (ค่าเริ่มต้น) |

### 244.3 กำหนด Widget Attrs: `class`, `placeholder`, `rows` ฯลฯ

ทุก widget รับพารามิเตอร์ `attrs` เป็น dict เพื่อเติม HTML attribute เพิ่มเติม
เข้าไปที่ tag โดยตรง — นี่คือวิธีมาตรฐานในการใส่ CSS class (เช่น Bootstrap's
`form-control`) หรือ `placeholder`:

```python
# blog/forms.py
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(
        max_length=100,
        label="ชื่อของคุณ",
        widget=forms.TextInput(attrs={
            "class": "form-control",
            "placeholder": "กรอกชื่อ-นามสกุล",
        }),
    )
    email = forms.EmailField(
        label="อีเมล",
        widget=forms.EmailInput(attrs={
            "class": "form-control",
            "placeholder": "you@example.com",
        }),
    )
    subject = forms.CharField(
        max_length=150,
        label="หัวข้อ",
        widget=forms.TextInput(attrs={"class": "form-control"}),
    )
    message = forms.CharField(
        label="ข้อความ",
        widget=forms.Textarea(attrs={
            "class": "form-control",
            "rows": 5,
            "placeholder": "พิมพ์ข้อความของคุณที่นี่...",
        }),
    )
    newsletter_opt_in = forms.BooleanField(
        required=False,
        label="สมัครรับจดหมายข่าว",
        widget=forms.CheckboxInput(attrs={"class": "form-check-input"}),
    )
```

เมื่อ render ผ่าน `{{ form.name }}` Django จะได้ HTML ประมาณนี้:

```html
<input type="text" name="name" class="form-control" placeholder="กรอกชื่อ-นามสกุล"
       maxlength="100" required id="id_name">
```

### 244.4 ตัวอย่างเต็ม: `SurveyForm` ใช้ `Select`, `RadioSelect`, `CheckboxSelectMultiple`

```python
# blog/forms.py
class SurveyForm(forms.Form):
    EXPERIENCE_CHOICES = [
        ("beginner", "มือใหม่ (ยังไม่เคยเขียน Django)"),
        ("intermediate", "ปานกลาง (เคยทำโปรเจกต์เล็ก ๆ)"),
        ("advanced", "ขั้นสูง (ใช้งานจริงในที่ทำงาน)"),
    ]
    RATING_CHOICES = [(str(i), str(i)) for i in range(1, 6)]
    TOPIC_CHOICES = [
        ("orm", "Django ORM"),
        ("drf", "Django REST Framework"),
        ("deploy", "Deployment / DevOps"),
        ("security", "Security"),
    ]

    # ใช้ Select (ดรอปดาวน์) — เหมาะกับตัวเลือกจำนวนมาก
    experience_level = forms.ChoiceField(
        choices=EXPERIENCE_CHOICES,
        label="ระดับประสบการณ์ของคุณ",
        widget=forms.Select(attrs={"class": "form-select"}),
    )

    # ใช้ RadioSelect (ปุ่มตัวเลือกกลม) — เหมาะกับตัวเลือกน้อย ต้องการให้เห็นครบทุกตัวพร้อมกัน
    satisfaction_rating = forms.ChoiceField(
        choices=RATING_CHOICES,
        label="ให้คะแนนความพึงพอใจ (1-5)",
        widget=forms.RadioSelect,
    )

    # ใช้ CheckboxSelectMultiple — เลือกได้มากกว่า 1 ตัวเลือก
    interested_topics = forms.MultipleChoiceField(
        choices=TOPIC_CHOICES,
        required=False,
        label="หัวข้อที่คุณสนใจ (เลือกได้หลายข้อ)",
        widget=forms.CheckboxSelectMultiple,
    )
```

```html
<!-- blog/templates/blog/survey.html -->
{% extends 'base.html' %}

{% block content %}
<h1>แบบสำรวจความคิดเห็น</h1>
<form method="post">
    {% csrf_token %}
    {% for field in form %}
        <div class="form-group">
            {{ field.label_tag }}
            {{ field }}
        </div>
    {% endfor %}
    <button type="submit">ส่งแบบสำรวจ</button>
</form>
{% endblock %}
```

| สถานการณ์เลือก widget | แนะนำใช้ |
|---|---|
| ตัวเลือกน้อย (2-5 ข้อ) ต้องการให้ผู้ใช้เห็นครบทุกตัวเลือกทันทีโดยไม่ต้องคลิกเปิด | `RadioSelect` / `CheckboxSelectMultiple` |
| ตัวเลือกจำนวนมาก (มากกว่า 5-6 ข้อ) ต้องประหยัดพื้นที่หน้าจอ | `Select` / `SelectMultiple` |
| ต้องการ custom เพิ่มเติม เช่น search-as-you-type ในดรอปดาวน์ | ใช้ JavaScript library เสริม (เช่น Select2, Choices.js) ทับ widget เดิม — เจาะลึกใน Part เรื่อง Frontend Integration |

---

## ขั้นตอนที่ 245: Custom Validation ต่อ Field ด้วย `clean_<fieldname>()`

### 245.1 หลักการทำงานของ `clean_<fieldname>()`

Django รองรับการเขียน validation เพิ่มเติมเฉพาะ field ใดฟิลด์หนึ่ง โดยตั้งชื่อ
method ในรูปแบบ `clean_<ชื่อ field>` ไว้ในคลาสฟอร์ม เมื่อ `is_valid()` ถูกเรียก
Django จะเรียก method เหล่านี้ **หลังจากที่ field validate ตามชนิดพื้นฐานผ่านแล้ว**
(เช่น `CharField` ตรวจ `required`/`max_length` ผ่านก่อน ถึงจะมาเรียก
`clean_<field>` ต่อ) รูปแบบมาตรฐานคือ:

```python
def clean_<fieldname>(self):
    value = self.cleaned_data["<fieldname>"]
    # ตรวจสอบเพิ่มเติม...
    if <เงื่อนไขที่ไม่ผ่าน>:
        raise forms.ValidationError("ข้อความ error ที่จะแสดง")
    return value   # สำคัญมาก: ต้อง return ค่ากลับเสมอ
```

**กฎเหล็กที่พลาดบ่อยที่สุด**: **ต้อง `return value` เสมอ** ไม่ว่าจะแก้ไขค่าหรือไม่
ก็ตาม เพราะค่าที่ return จาก `clean_<fieldname>()` จะถูกนำไปแทนที่ค่าเดิมใน
`cleaned_data` ทันที ถ้าลืม `return` ค่าของ field นั้นจะกลายเป็น `None` ใน
`cleaned_data` โดยไม่มี error ใด ๆ แจ้งเตือน (บั๊กเงียบที่ debug ยากมาก)

### 245.2 ตัวอย่าง: ตรวจสอบความยาวขั้นต่ำของข้อความคอมเมนต์

สมมติเราต้องการฟอร์มรับความคิดเห็น (ใช้ต่อยอดเป็น `CommentForm` เต็มรูปแบบใน
ขั้นตอนที่ 250) และต้องการบังคับว่าข้อความต้องมีความยาวอย่างน้อย 10 ตัวอักษร
และห้ามมีคำหยาบบางคำปนอยู่:

```python
# blog/forms.py
from django import forms

BANNED_WORDS = ["สแปม", "โฆษณา", "คลิกที่นี่"]


class CommentForm(forms.Form):
    author = forms.CharField(max_length=100, label="ชื่อผู้แสดงความคิดเห็น")
    text = forms.CharField(
        label="ความคิดเห็น",
        widget=forms.Textarea(attrs={"rows": 4}),
    )

    def clean_text(self):
        text = self.cleaned_data["text"]

        if len(text.strip()) < 10:
            raise forms.ValidationError(
                "ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร"
            )

        lowered = text.lower()
        for word in BANNED_WORDS:
            if word in lowered:
                raise forms.ValidationError(
                    f'ข้อความมีคำที่ไม่อนุญาต: "{word}" กรุณาแก้ไขก่อนส่ง'
                )

        return text.strip()   # trim ช่องว่างหัวท้ายก่อนบันทึกจริง
```

ทดสอบใน shell:

```python
>>> from blog.forms import CommentForm
>>> form = CommentForm(data={"author": "สมชาย", "text": "ดี"})
>>> form.is_valid()
False
>>> form.errors
{'text': ['ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร']}
```

### 245.3 การแจ้ง Error หลายข้อพร้อมกันใน Field เดียว

ถ้าต้องการรายงาน error มากกว่า 1 ข้อความพร้อมกันสำหรับ field เดียว ให้ส่ง
`list` ของข้อความเข้าไปใน `ValidationError` แทนที่จะเป็น `str` เดี่ยว:

```python
def clean_text(self):
    text = self.cleaned_data["text"]
    errors = []

    if len(text.strip()) < 10:
        errors.append("ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร")
    if text.isupper():
        errors.append("กรุณาอย่าพิมพ์ด้วยตัวพิมพ์ใหญ่ทั้งหมด (ดูเหมือนตะโกน)")

    if errors:
        raise forms.ValidationError(errors)

    return text.strip()
```

เมื่อเกิด error แบบนี้ `form.errors["text"]` จะเป็น list ที่มีข้อความ error
ทุกข้อที่พบพร้อมกัน (ไม่ใช่แค่ข้อความแรก) ทำให้ผู้ใช้เห็นปัญหาทั้งหมดในครั้งเดียว
ไม่ต้องแก้ไปทีละรอบ

---

## ขั้นตอนที่ 246: Custom Validation ข้าม Field ด้วย `clean()`

### 246.1 ทำไมต้องมี `clean()` แยกจาก `clean_<fieldname>()`

`clean_<fieldname>()` ตรวจสอบได้แค่ **field เดียวโดยลำพัง** ไม่สามารถเทียบค่ากับ
field อื่นได้ เพราะ ณ ตอนที่ Django เรียก `clean_<fieldname>()` ยังไม่รับประกันว่า
field อื่นจะ validate ผ่านแล้วหรือยัง (ถ้า field อื่น validate ไม่ผ่าน มันจะไม่ถูกใส่
ใน `cleaned_data` เลยด้วยซ้ำ)

เมื่อ **logic การตรวจสอบต้องเทียบค่าระหว่าง 2 field ขึ้นไป** (เช่น "รหัสผ่านกับ
ยืนยันรหัสผ่านต้องตรงกัน", "วันที่เริ่มต้องมาก่อนวันที่สิ้นสุด") ต้อง override
method `clean()` ของทั้งฟอร์ม ซึ่งจะถูกเรียก **หลังจาก** `clean_<fieldname>()`
ของทุก field ทำงานเสร็จหมดแล้วเสมอ

### 246.2 ตัวอย่าง: ฟอร์มตั้งรหัสผ่าน — Password กับ Confirm Password ต้องตรงกัน

```python
# blog/forms.py
from django import forms


class SetPasswordForm(forms.Form):
    password = forms.CharField(
        label="รหัสผ่านใหม่",
        widget=forms.PasswordInput(attrs={"class": "form-control"}),
        min_length=8,
    )
    confirm_password = forms.CharField(
        label="ยืนยันรหัสผ่าน",
        widget=forms.PasswordInput(attrs={"class": "form-control"}),
    )

    def clean(self):
        cleaned_data = super().clean()   # สำคัญมาก: ต้องเรียก super().clean() เสมอ
        password = cleaned_data.get("password")
        confirm_password = cleaned_data.get("confirm_password")

        if password and confirm_password and password != confirm_password:
            raise forms.ValidationError(
                "รหัสผ่านและการยืนยันรหัสผ่านไม่ตรงกัน กรุณาตรวจสอบอีกครั้ง"
            )

        return cleaned_data   # สำคัญมาก: ต้อง return cleaned_data เสมอ
```

**กฎเหล็ก 2 ข้อของ `clean()`**:

1. **ต้องเรียก `super().clean()` เป็นบรรทัดแรกเสมอ** และเก็บผลลัพธ์ไว้ในตัวแปร
   (โดยทั่วไปตั้งชื่อ `cleaned_data`) เพื่อให้แน่ใจว่า logic ของ parent class
   ทำงานครบถ้วน (ในกรณีที่มีการสืบทอดฟอร์มต่อกันหลายชั้น)
2. **ต้อง `return cleaned_data` เสมอที่ท้าย method** ไม่ว่าจะแก้ไขอะไรหรือไม่
   ถ้าลืม return ค่า `cleaned_data` ทั้งก้อนจะกลายเป็น `None`

3. **ใช้ `.get()` แทนการเข้าถึงด้วย `[]` ตรง ๆ** เพราะถ้า field ใด field หนึ่ง
   validate ไม่ผ่านตั้งแต่ขั้นตอนก่อนหน้า มันจะไม่มี key นั้นอยู่ใน
   `cleaned_data` เลย การใช้ `[]` ตรง ๆ จะทำให้เกิด `KeyError` แทนที่จะแสดง
   error ที่เข้าใจง่าย

### 246.3 `add_error()`: ผูก Error ข้าม Field เข้ากับ Field ใดฟิลด์หนึ่งโดยเฉพาะ

ค่าเริ่มต้นเมื่อ `raise ValidationError(...)` ภายใน `clean()` (ไม่ใช่ใน
`clean_<fieldname>()`) ข้อความ error จะไปอยู่ใน `form.non_field_errors()`
(key พิเศษ `__all__`) ซึ่งเหมาะกับ error ที่ไม่ผูกกับ field ใดโดยเฉพาะจริง ๆ

แต่ถ้าต้องการให้ error แสดง **ติดอยู่ข้าง field ใดฟิลด์หนึ่งโดยเฉพาะ** (เช่น
อยากให้ error "รหัสผ่านไม่ตรงกัน" ไปแสดงใต้ช่อง `confirm_password` แทนที่จะลอย
อยู่บนสุดของฟอร์ม) ให้ใช้ `self.add_error()` แทนการ `raise` ตรง ๆ:

```python
def clean(self):
    cleaned_data = super().clean()
    password = cleaned_data.get("password")
    confirm_password = cleaned_data.get("confirm_password")

    if password and confirm_password and password != confirm_password:
        self.add_error(
            "confirm_password",
            "รหัสผ่านไม่ตรงกับที่กรอกไว้ด้านบน",
        )

    return cleaned_data
```

| วิธีแจ้ง Error ใน `clean()` | ปรากฏที่ไหน | ใช้เมื่อไร |
|---|---|---|
| `raise forms.ValidationError("...")` | `form.non_field_errors()` (`__all__`) | error ที่เป็นภาพรวม ไม่เจาะจง field เดียว |
| `self.add_error("confirm_password", "...")` | `form.confirm_password.errors` (ติดกับ field นั้น) | error ที่ควรชี้เป้าให้ผู้ใช้แก้ที่ field ใดฟิลด์หนึ่งชัดเจน |
| `self.add_error(None, "...")` | `form.non_field_errors()` เหมือนกับ `raise` | เขียนแบบ explicit ว่าตั้งใจไม่ผูกกับ field ใด |

**ข้อควรระวัง**: เมื่อใช้ `self.add_error()` ภายใน `clean()` **ห้าม** `raise`
ตามหลังอีก เพราะ `add_error()` จัดการ error state ให้เสร็จสมบูรณ์ในตัวเองแล้ว
(มันลบ field นั้นออกจาก `cleaned_data` ให้อัตโนมัติด้วย) การ `raise` ซ้ำจะทำให้
error message ซ้อนกันโดยไม่จำเป็น

---

## ขั้นตอนที่ 247: `initial` Data และการสร้างฟอร์มที่มีค่าเริ่มต้น

### 247.1 `initial` ตอนประกาศ Field ใน Class

สามารถกำหนดค่าเริ่มต้นให้ field ตอนประกาศคลาสได้โดยตรงผ่านพารามิเตอร์ `initial`
ค่านี้จะถูกใช้ **ทุกครั้งที่สร้างฟอร์มแบบ unbound** โดยไม่ต้องส่งอะไรเพิ่ม:

```python
class SurveyForm(forms.Form):
    country = forms.CharField(max_length=100, initial="ประเทศไทย")
    subscribe = forms.BooleanField(required=False, initial=True)
```

```python
>>> form = SurveyForm()
>>> form.initial
{}
>>> print(form["country"])
<input type="text" name="country" value="ประเทศไทย" id="id_country">
```

### 247.2 `initial` ตอนสร้าง Instance ของฟอร์ม

บ่อยครั้งค่าเริ่มต้นต้องมาจาก**ข้อมูลจริงที่ดึงมาตอน runtime** (ไม่ใช่ค่าคงที่
ที่รู้ล่วงหน้าตอนเขียนโค้ด) เช่น ต้องการฟอร์มที่ pre-fill ด้วยข้อมูลของผู้ใช้ที่
login อยู่ ให้ส่ง `initial=` เป็น dict ตอนสร้าง instance แทน:

```python
def edit_profile_view(request):
    if request.method == "POST":
        form = ProfileForm(request.POST)
        if form.is_valid():
            # บันทึกข้อมูล...
            return redirect("blog:profile_saved")
    else:
        # ดึงข้อมูลปัจจุบันมา pre-fill ให้ฟอร์ม (ยังคง unbound เพราะไม่ได้ส่ง data=)
        form = ProfileForm(initial={
            "display_name": request.user.get_full_name() or request.user.username,
            "bio": "นักพัฒนา Django มือใหม่ที่กำลังเรียนคอร์สนี้อยู่",
        })

    return render(request, "blog/edit_profile.html", {"form": form})
```

```python
class ProfileForm(forms.Form):
    display_name = forms.CharField(max_length=150, label="ชื่อที่แสดง")
    bio = forms.CharField(
        required=False,
        label="แนะนำตัวสั้น ๆ",
        widget=forms.Textarea(attrs={"rows": 3}),
    )
```

`initial=` ที่ส่งตอนสร้าง instance จะ**มีความสำคัญเหนือกว่า** `initial=` ที่
ประกาศไว้ระดับ field เสมอ ถ้ามีทั้งคู่พร้อมกัน

### 247.3 `initial` vs `data`: ความแตกต่างที่ต้องเข้าใจให้ชัด

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุด — ทั้งคู่ดูเหมือน "ใส่ค่าให้ฟอร์ม" แต่ความหมาย
ต่างกันโดยสิ้นเชิง:

| ประเด็น | `initial=` | `data=` |
|---|---|---|
| ทำให้ฟอร์มเป็น bound หรือไม่ | **ไม่** — ฟอร์มยังเป็น unbound เสมอ | **ใช่** — ทำให้ฟอร์มเป็น bound ทันที |
| ผ่าน validation หรือไม่ | ไม่ผ่านการ validate ใด ๆ เลย | ผ่านการ validate เต็มรูปแบบเมื่อเรียก `is_valid()` |
| ใช้เพื่ออะไร | **แสดงผล**ค่าเริ่มต้นให้ผู้ใช้เห็น/แก้ไขต่อ | **ตรวจสอบ**ข้อมูลที่ผู้ใช้ส่งมาจริง |
| เรียก `is_valid()` มีความหมายไหม | ไม่มีความหมาย (คืน `False` เสมอเพราะ unbound) | มีความหมายเต็มที่ |
| ตัวอย่างแหล่งข้อมูล | ค่าคงที่, ข้อมูลเดิมจากฐานข้อมูลที่ดึงมาแสดง | `request.POST`, `request.GET` |

พูดสั้น ๆ: **`initial` คือ "ข้อเสนอ" ให้ผู้ใช้เห็นตอนเปิดฟอร์มครั้งแรก
ส่วน `data` คือ "คำตอบจริง" ที่ผู้ใช้กด submit ส่งกลับมา** ทั้งสองไม่เกี่ยวข้องกัน
เลยในเชิง logic — ฟอร์มหนึ่งอาจมีทั้งคู่พร้อมกันได้ (เช่นตอนแสดงฟอร์มที่กรอกผิด
ซ้ำ Django จะใช้ `data` ที่ผู้ใช้กรอกมาแสดงแทน ไม่ใช่ `initial` เดิม เพื่อไม่ให้
ผู้ใช้ต้องพิมพ์ใหม่ทั้งหมด)

### 247.4 ตัวอย่างเต็ม: ฟอร์มแก้ไขความคิดเห็นแบบ Plain Form (ยังไม่ใช้ ModelForm)

สาธิตการนำ `initial` มาใช้กับข้อมูลจริงจากฐานข้อมูล โดยยังคงใช้ plain `forms.Form`
(ไม่พึ่ง `ModelForm` เพราะยังไม่ได้เรียน — Part 026 จะทำให้ pattern นี้สั้นลงมาก):

```python
# blog/views.py
from django.shortcuts import render, redirect, get_object_or_404
from .models import Comment
from .forms import CommentForm


def edit_comment_view(request, comment_id):
    comment = get_object_or_404(Comment, pk=comment_id)

    if request.method == "POST":
        form = CommentForm(request.POST)
        if form.is_valid():
            comment.author = form.cleaned_data["author"]
            comment.text = form.cleaned_data["text"]
            comment.save()
            return redirect("blog:detail", slug=comment.post.slug)
    else:
        # pre-fill ฟอร์มด้วยข้อมูลปัจจุบันของ comment ที่มีอยู่แล้วในฐานข้อมูล
        form = CommentForm(initial={
            "author": comment.author,
            "text": comment.text,
        })

    return render(request, "blog/edit_comment.html", {"form": form, "comment": comment})
```

สังเกตว่าโค้ดนี้ **ยาวกว่าที่ `ModelForm` จะทำให้ได้มาก** (ต้อง copy ค่าจาก
`cleaned_data` ไปใส่ instance ทีละ field ด้วยมือ) — นี่คือสิ่งที่ Part 026 จะช่วย
ย่นระยะให้สั้นลง แต่การเข้าใจ pattern แบบเต็มรูปแบบนี้ก่อน จะทำให้คุณรู้ว่า
`ModelForm` "ลัด" ขั้นตอนอะไรไปบ้างเมื่อเรียนถึง

---

## ขั้นตอนที่ 248: จัดการ File Upload ในฟอร์ม

### 248.1 `enctype="multipart/form-data"`: ทำไมฟอร์มปกติอัปโหลดไฟล์ไม่ได้

ฟอร์ม HTML ทั่วไปที่ไม่ระบุ `enctype` จะใช้ค่าเริ่มต้นคือ
`application/x-www-form-urlencoded` ซึ่ง**เข้ารหัสได้แค่ข้อความเท่านั้น**
ไม่สามารถส่งไฟล์ binary (รูปภาพ, PDF, ฯลฯ) ไปพร้อมกับ request ได้เลย

เมื่อฟอร์มมี field ประเภทไฟล์ (`<input type="file">`) **ต้อง**ระบุ
`enctype="multipart/form-data"` ที่ tag `<form>` เสมอ ไม่เช่นนั้นไฟล์ที่ผู้ใช้
เลือกจะไม่ถูกส่งไปกับ request เลยแม้แต่ไบต์เดียว (ไม่มี error ใด ๆ เตือนด้วย —
เป็นบั๊กเงียบที่พบบ่อยมากสำหรับมือใหม่):

```html
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">อัปโหลด</button>
</form>
```

**กฎเหล็ก**: **ฟอร์มใดก็ตามที่มี `FileField` หรือ `ImageField` ต้องมี
`enctype="multipart/form-data"` เสมอ ไม่มีข้อยกเว้น**

### 248.2 `request.FILES`: ที่เก็บไฟล์ที่อัปโหลดมา (แยกจาก `request.POST`)

Django แยกข้อมูลไฟล์ออกจากข้อมูลข้อความโดยเด็ดขาด — ข้อความทั่วไปอยู่ใน
`request.POST` แต่**ไฟล์ทั้งหมดอยู่ใน `request.FILES`** (เป็น dict-like object
เหมือนกัน) เมื่อสร้างฟอร์มที่มี field ประเภทไฟล์ ต้องส่งทั้งสองอย่างเข้าไปพร้อมกัน:

```python
form = UploadForm(request.POST, request.FILES)
#                  ↑ data=            ↑ files=
```

ถ้าลืมส่ง `request.FILES` เข้าไป ฟอร์มจะมองว่าไม่มีไฟล์ถูกอัปโหลดมาเลย และ
`FileField` ที่เป็น `required=True` จะ validate ไม่ผ่านทันที

### 248.3 `forms.FileField` และ `forms.ImageField`

```python
# blog/forms.py
from django import forms


class DocumentUploadForm(forms.Form):
    title = forms.CharField(max_length=200, label="ชื่อเอกสาร")
    document = forms.FileField(label="ไฟล์เอกสาร (PDF, DOCX)")


class AvatarUploadForm(forms.Form):
    avatar = forms.ImageField(label="รูปโปรไฟล์")
```

| Field | ตรวจสอบอะไรเพิ่มเติมจาก `FileField` พื้นฐาน | ต้องติดตั้งอะไรเพิ่ม |
|---|---|---|
| `FileField` | ตรวจว่ามีไฟล์ถูกส่งมาจริง (ถ้า `required=True`) และไม่เกิน `max_upload_size` ที่กำหนดเองถ้ามี | ไม่ต้อง |
| `ImageField` | ตรวจเพิ่มเติมว่าไฟล์ที่ส่งมา**เปิดเป็นรูปภาพได้จริง** (ไม่ใช่แค่เช็คนามสกุลไฟล์) | **ต้องติดตั้ง Pillow**: `pip install Pillow` |

ถ้าลืมติดตั้ง Pillow แล้วใช้ `ImageField` Django จะ raise error ทันทีตอน
`makemigrations`/`check` ว่า "Cannot use ImageField because Pillow is not
installed" — เป็นข้อความ error ที่ชัดเจนและช่วยเตือนได้ตรงจุด

### 248.4 ตัวอย่างเต็ม: ฟอร์มอัปโหลด Avatar พร้อม View ที่บันทึกไฟล์

```python
# blog/views.py
import os
from django.conf import settings
from django.shortcuts import render, redirect
from .forms import AvatarUploadForm


def upload_avatar_view(request):
    if request.method == "POST":
        form = AvatarUploadForm(request.POST, request.FILES)
        if form.is_valid():
            avatar_file = form.cleaned_data["avatar"]

            # บันทึกไฟล์ลง MEDIA_ROOT ด้วยมือ (ทบทวน MEDIA_ROOT จาก Part 009)
            save_path = os.path.join(settings.MEDIA_ROOT, "avatars", avatar_file.name)
            os.makedirs(os.path.dirname(save_path), exist_ok=True)
            with open(save_path, "wb+") as destination:
                for chunk in avatar_file.chunks():
                    destination.write(chunk)

            return redirect("blog:upload_success")
    else:
        form = AvatarUploadForm()

    return render(request, "blog/upload_avatar.html", {"form": form})
```

```html
<!-- blog/templates/blog/upload_avatar.html -->
{% extends 'base.html' %}

{% block content %}
<h1>อัปโหลดรูปโปรไฟล์</h1>
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">อัปโหลด</button>
</form>
{% endblock %}
```

**ทำไมต้องวนลูปด้วย `.chunks()` แทนการอ่านไฟล์ทั้งก้อนทีเดียว**: ไฟล์ขนาดใหญ่
(เช่นวิดีโอหลายร้อย MB) ถ้าโหลดทั้งไฟล์เข้าหน่วยความจำทีเดียวอาจทำให้ server
ใช้ RAM สูงเกินจำเป็นหรือถึงขั้น crash `.chunks()` อ่านและเขียนไฟล์ทีละส่วนเล็ก ๆ
(ค่าเริ่มต้นประมาณ 64KB ต่อ chunk) ทำให้ใช้หน่วยความจำคงที่ไม่ว่าไฟล์จะใหญ่แค่ไหน
— นี่คือแนวทางที่ปลอดภัยสำหรับ production เสมอ ส่วนการจัดการไฟล์แบบเต็มรูปแบบ
ด้วย `Storage` API และการอัปโหลดไปยัง cloud storage (S3 ฯลฯ) จะเจาะลึกในภายหลัง

> **หมายเหตุ**: เมื่อคุณเรียน `ModelForm` ใน Part 026 Django จะจัดการบันทึกไฟล์
> ให้อัตโนมัติทั้งหมดผ่าน `FileField`/`ImageField` ของ Model (ไม่ต้องเขียน
> `open()`/`.chunks()` เองแบบนี้) แต่การเข้าใจกลไกเบื้องหลังตรงนี้ก่อน จะช่วยให้
> คุณ debug ปัญหาการอัปโหลดไฟล์ได้อย่างเข้าใจจริง ไม่ใช่แค่ท่องจำ

---

## ขั้นตอนที่ 249: CSRF Protection ในฟอร์ม และ AJAX Form Submission เบื้องต้น

### 249.1 ทบทวน CSRF จาก Part 008

ใน Part 008 ขั้นตอนที่ 78.5 คุณได้เรียนแล้วว่า **ทุกฟอร์ม `method="post"` ต้องมี
`{% csrf_token %}` อยู่ข้างในเสมอ** ไม่มีข้อยกเว้น เพราะ `CsrfViewMiddleware`
จะปฏิเสธ POST request ทุกตัวที่ไม่มี token ที่ถูกต้องด้วย HTTP 403 Forbidden

หลักการนี้ใช้ได้กับทุกฟอร์มที่เราสร้างมาตลอด Part นี้เช่นกัน — ทุกตัวอย่าง
`<form method="post">` ที่ผ่านมาจึงมี `{% csrf_token %}` กำกับไว้เสมอ สิ่งที่
Part นี้จะเพิ่มเติมคือ: **แล้วถ้าฟอร์มไม่ได้ submit แบบปกติ แต่ submit ผ่าน
JavaScript (AJAX) ล่ะ จะแนบ CSRF token อย่างไร เพราะไม่มี `<form>` ธรรมดาให้
`{% csrf_token %}` แทรก hidden input ให้อัตโนมัติ**

### 249.2 ทำไม AJAX (`fetch`) ต้องแนบ CSRF Token เอง

เมื่อ submit ฟอร์มแบบ AJAX ด้วย JavaScript's `fetch()` หรือ `XMLHttpRequest`
เบราว์เซอร์**ไม่ได้ submit `<form>` จริง ๆ** จึงไม่มีกลไกอัตโนมัติที่แนบ
hidden input `csrfmiddlewaretoken` ไปด้วย เราต้องดึงค่า CSRF token มาแนบเป็น
**HTTP Header** ด้วยตัวเอง — Django กำหนดชื่อ header มาตรฐานไว้แล้วคือ
`X-CSRFToken`

### 249.3 ดึงค่า CSRF Token จาก Cookie ด้วย JavaScript

Django เก็บ CSRF token ไว้ใน cookie ชื่อ `csrftoken` โดยอัตโนมัติ (ตราบใดที่
หน้าเว็บนั้นเคย render `{% csrf_token %}` หรือมี view ที่ใช้
`@ensure_csrf_cookie` มาก่อน) ฟังก์ชัน JavaScript มาตรฐานที่ทีม Django แนะนำ
ในเอกสารทางการสำหรับดึงค่าจาก cookie มีดังนี้:

```javascript
// blog/static/blog/js/csrf.js
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== "") {
        const cookies = document.cookie.split(";");
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + "=")) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie("csrftoken");
```

### 249.4 ตัวอย่างเต็ม: ส่งฟอร์มด้วย `fetch()` พร้อมแนบ CSRF Header

```html
<!-- blog/templates/blog/comment_ajax.html -->
{% extends 'base.html' %}

{% block content %}
<h1>แสดงความคิดเห็น (แบบ AJAX ไม่รีเฟรชหน้า)</h1>

<form id="comment-form">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">ส่งความคิดเห็น</button>
</form>

<div id="comment-result"></div>

{% endblock %}

{% block extra_js %}
<script src="{% static 'blog/js/csrf.js' %}"></script>
<script>
document.getElementById("comment-form").addEventListener("submit", function (event) {
    event.preventDefault();   // ป้องกันไม่ให้ฟอร์ม submit แบบปกติ (ที่ทำให้หน้ารีเฟรช)

    const form = event.target;
    const formData = new FormData(form);

    fetch("{% url 'blog:add_comment_ajax' %}", {
        method: "POST",
        headers: {
            "X-CSRFToken": csrftoken,   // แนบ CSRF token ผ่าน header แทนที่จะพึ่ง hidden input
        },
        body: formData,
    })
        .then((response) => response.json())
        .then((data) => {
            const resultBox = document.getElementById("comment-result");
            if (data.success) {
                resultBox.textContent = "ส่งความคิดเห็นสำเร็จ!";
                form.reset();
            } else {
                resultBox.textContent = "เกิดข้อผิดพลาด: " + JSON.stringify(data.errors);
            }
        });
});
</script>
{% endblock %}
```

สังเกตว่า `{% csrf_token %}` ยังคงอยู่ใน `<form>` เหมือนเดิม (เผื่อ JavaScript
ปิดอยู่หรือโหลดไม่สำเร็จ ฟอร์มก็ยัง fallback เป็นการ submit ปกติได้) แต่โค้ด
JavaScript **ไม่ได้ใช้ hidden input นั้นเลย** — มันดึง token จาก cookie มาแนบ
เป็น header เอง (`FormData(form)` จะรวม hidden input `csrfmiddlewaretoken`
เข้าไปใน body ด้วยอยู่แล้วก็จริง แต่การแนบซ้ำผ่าน header เป็นวิธีที่ปลอดภัยกว่า
และเป็นมาตรฐานที่เอกสาร Django แนะนำสำหรับ AJAX request โดยเฉพาะ)

### 249.5 View ที่รองรับทั้ง Normal POST และ AJAX

```python
# blog/views.py
from django.http import JsonResponse
from .forms import CommentForm


def add_comment_ajax_view(request):
    if request.method != "POST":
        return JsonResponse({"success": False, "errors": "ต้องใช้ POST เท่านั้น"}, status=405)

    form = CommentForm(request.POST)
    if form.is_valid():
        # ในตัวอย่างนี้ยังไม่บันทึกลงฐานข้อมูลจริง (เต็มรูปแบบในขั้นตอนที่ 250)
        return JsonResponse({
            "success": True,
            "author": form.cleaned_data["author"],
        })

    return JsonResponse({"success": False, "errors": form.errors}, status=400)
```

```python
# blog/urls.py
urlpatterns += [
    path("comment/ajax/", views.add_comment_ajax_view, name="add_comment_ajax"),
]
```

| | Normal Form Submission | AJAX Submission |
|---|---|---|
| ต้องมี `{% csrf_token %}` ใน `<form>` ไหม | ต้องมี | ควรมีไว้เป็น fallback แต่ JavaScript จะไม่ใช้ hidden input นี้โดยตรง |
| ต้องแนบ CSRF token เพิ่มใน JavaScript ไหม | ไม่ต้อง (เบราว์เซอร์ส่ง form data ทั้งหมดให้อัตโนมัติ) | **ต้อง** แนบผ่าน header `X-CSRFToken` เอง |
| View คืนอะไรกลับมา | `HttpResponse`/`redirect()` (โหลดหน้าใหม่ทั้งหน้า) | `JsonResponse` (หน้าเว็บไม่รีเฟรช) |
| เหมาะกับ | ฟอร์มทั่วไป, ฟอร์มที่ redirect ไปหน้าอื่นหลัง submit | ฟอร์มที่ต้องการ UX ลื่นไหลไม่รีเฟรชหน้า (เช่น ส่งคอมเมนต์แล้วโผล่ทันที) |

---

## ขั้นตอนที่ 250: สรุปและแบบฝึกหัด — สร้าง `CommentForm` พร้อม View ที่สร้าง `Comment` เอง

### 250.1 ทบทวนโครงสร้าง Model `Comment`

ก่อนประกอบร่างทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน ทบทวน Model `Comment`
ของแอป `blog` ที่มีอยู่แล้ว:

```python
# blog/models.py (ทบทวนจาก Phase 2)
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title


class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.CharField(max_length=100)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return f"ความคิดเห็นโดย {self.author} บน {self.post.title}"
```

### 250.2 สร้าง `CommentForm` แบบ Plain Form ฉบับสมบูรณ์

รวมทุกเทคนิคที่เรียนมาตลอด Part นี้ — field types (241), widget attrs (244),
`clean_<fieldname>()` (245) และ `clean()` (246) — เข้าไปใน `CommentForm`
ตัวเดียว:

```python
# blog/forms.py
from django import forms

BANNED_WORDS = ["สแปม", "โฆษณา", "คลิกที่นี่"]


class CommentForm(forms.Form):
    author = forms.CharField(
        max_length=100,
        label="ชื่อของคุณ",
        widget=forms.TextInput(attrs={
            "class": "form-control",
            "placeholder": "ชื่อที่จะแสดงบนความคิดเห็น",
        }),
    )
    text = forms.CharField(
        label="ความคิดเห็น",
        widget=forms.Textarea(attrs={
            "class": "form-control",
            "rows": 4,
            "placeholder": "แสดงความคิดเห็นของคุณ...",
        }),
    )
    honeypot = forms.CharField(required=False, widget=forms.HiddenInput)

    def clean_author(self):
        author = self.cleaned_data["author"]
        if len(author.strip()) < 2:
            raise forms.ValidationError("ชื่อสั้นเกินไป กรุณาระบุอย่างน้อย 2 ตัวอักษร")
        return author.strip()

    def clean_text(self):
        text = self.cleaned_data["text"]
        if len(text.strip()) < 10:
            raise forms.ValidationError(
                "ความคิดเห็นสั้นเกินไป กรุณาเขียนอย่างน้อย 10 ตัวอักษร"
            )
        lowered = text.lower()
        for word in BANNED_WORDS:
            if word in lowered:
                raise forms.ValidationError(
                    f'ข้อความมีคำที่ไม่อนุญาต: "{word}"'
                )
        return text.strip()

    def clean(self):
        cleaned_data = super().clean()
        # honeypot field: field ที่คนจริงมองไม่เห็น (ซ่อนด้วย CSS/HiddenInput)
        # ถ้ามีค่าถูกกรอกมา แปลว่าน่าจะเป็นบอทสแปมที่กรอกทุก field อัตโนมัติ
        if cleaned_data.get("honeypot"):
            raise forms.ValidationError("ตรวจพบความผิดปกติ กรุณาลองใหม่อีกครั้ง")
        return cleaned_data
```

> **หมายเหตุเรื่อง `honeypot`**: นี่คือเทคนิคป้องกันสแปมเบื้องต้นที่ไม่ต้องพึ่ง
> CAPTCHA — เพิ่ม field ที่ซ่อนไว้ด้วย `HiddenInput` (หรือซ่อนด้วย CSS
> `display: none`) ที่มนุษย์จริงมองไม่เห็นและจะไม่กรอก แต่บอทสแปมทั่วไปที่
> กรอกทุก input ในฟอร์มอัตโนมัติจะกรอกฟิลด์นี้ด้วย ทำให้ตรวจจับได้ง่าย
> เป็นเทคนิคพื้นฐานที่ใช้ประกอบ (ไม่ใช่ทดแทน) ระบบป้องกันสแปมที่เข้มงวดกว่า
> เช่น `django-recaptcha` ซึ่งจะแนะนำในภายหลัง

### 250.3 View `add_comment`: รับข้อมูลจากฟอร์มไปสร้าง `Comment` ด้วยมือ

จุดสำคัญที่สุดของขั้นตอนนี้คือ **View นี้สร้าง `Comment.objects.create()` เอง
ทีละ field จาก `form.cleaned_data`** ไม่ได้ใช้ทางลัดของ `ModelForm.save()`
เพราะเรายังไม่ได้เรียนเรื่องนั้น (Part 026 จะทำให้โค้ดส่วนนี้สั้นลงเหลือไม่กี่
บรรทัด):

```python
# blog/views.py
from django.shortcuts import render, redirect, get_object_or_404
from .models import Post, Comment
from .forms import CommentForm


def post_detail_view(request, slug):
    post = get_object_or_404(Post, slug=slug)
    comments = post.comments.all()   # related_name="comments" จาก Comment.post

    if request.method == "POST":
        form = CommentForm(request.POST)
        if form.is_valid():
            # ประกอบ Comment object ขึ้นมาเองจากข้อมูลที่ผ่านการ validate แล้ว
            # (นี่คือสิ่งที่ ModelForm.save() จะทำให้อัตโนมัติใน Part 026)
            Comment.objects.create(
                post=post,
                author=form.cleaned_data["author"],
                text=form.cleaned_data["text"],
            )
            return redirect("blog:detail", slug=post.slug)
        # ถ้า is_valid() False: ปล่อยผ่านไป render() ด้านล่างพร้อม error เดิม
    else:
        form = CommentForm()

    return render(request, "blog/post_detail.html", {
        "post": post,
        "comments": comments,
        "form": form,
    })
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"

urlpatterns = [
    path("", views.post_list_view, name="list"),
    path("post/<slug:slug>/", views.post_detail_view, name="detail"),
    path("contact/", views.contact_view, name="contact"),
    path("contact/success/", views.contact_success, name="contact_success"),
    path("comment/ajax/", views.add_comment_ajax_view, name="add_comment_ajax"),
]
```

### 250.4 Template: แสดงรายการคอมเมนต์ พร้อมฟอร์มส่งคอมเมนต์ใหม่

```html
<!-- blog/templates/blog/post_detail.html -->
{% extends 'base.html' %}

{% block title %}{{ post.title }} | Django Mastery Blog{% endblock %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>
    <time datetime="{{ post.created_at|date:'c' }}">
        เผยแพร่เมื่อ {{ post.created_at|date:"d F Y" }}
    </time>
    {{ post.content|linebreaks }}
</article>

<section class="comments">
    <h2>ความคิดเห็น ({{ comments|length }})</h2>

    {% for comment in comments %}
        <div class="comment">
            <strong>{{ comment.author }}</strong>
            <span class="comment__date">{{ comment.created_at|date:"d/m/Y H:i" }}</span>
            <p>{{ comment.text|linebreaksbr }}</p>
        </div>
    {% empty %}
        <p>ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็นสิ!</p>
    {% endfor %}

    <h3>แสดงความคิดเห็นของคุณ</h3>
    <form method="post">
        {% csrf_token %}
        {% for field in form %}
            {% if field.is_hidden %}
                {{ field }}
            {% else %}
                <div class="form-group">
                    {{ field.label_tag }}
                    {{ field }}
                    {% for error in field.errors %}
                        <div class="form-error">{{ error }}</div>
                    {% endfor %}
                </div>
            {% endif %}
        {% endfor %}
        {% if form.non_field_errors %}
            <div class="form-error form-error--general">{{ form.non_field_errors }}</div>
        {% endif %}
        <button type="submit">ส่งความคิดเห็น</button>
    </form>
</section>
{% endblock %}
```

สังเกตการใช้ `field.is_hidden` เพื่อแยก `honeypot` (ที่เป็น `HiddenInput`)
ออกจาก field ที่ต้องแสดง `<label>` ตามปกติ — ป้องกันไม่ให้ label ของ hidden
field โผล่มาให้ผู้ใช้เห็นโดยไม่จำเป็น

### 250.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `forms.Form` กับ Model field เป็นคนละระบบที่ไม่เกี่ยวข้องกันในโค้ด
  แม้ชื่อ field จะคล้ายกัน — Model อธิบายฐานข้อมูล ส่วน Form อธิบายการรับ/
  ตรวจสอบข้อมูลจากผู้ใช้
- ✅ รู้จัก field types ที่ใช้บ่อยที่สุด: `CharField`, `EmailField`,
  `IntegerField`, `ChoiceField`, `BooleanField`, `FileField`, `ImageField`
- ✅ Render ฟอร์มได้หลายวิธี: `{{ form }}` (default เปลี่ยนเป็น div ตั้งแต่
  Django 5.0), `.as_p`, `.as_table`, `.as_div`, และการ render ทีละ field เอง
- ✅ เข้าใจวงจรชีวิตฟอร์มอย่างถ่องแท้: unbound → bound → `is_valid()` →
  `cleaned_data`/`errors`
- ✅ ใช้ Widget ควบคุมหน้าตา HTML แยกอิสระจาก Field ที่ควบคุมข้อมูล
- ✅ เขียน Custom Validation ได้ทั้งระดับ field เดียว (`clean_<fieldname>()`)
  และข้าม field (`clean()` พร้อม `add_error()`)
- ✅ แยกความแตกต่างระหว่าง `initial` (ค่าเริ่มต้นสำหรับแสดงผล) กับ `data`
  (ข้อมูลจริงที่ต้อง validate) ได้อย่างชัดเจน
- ✅ จัดการ File Upload ได้ครบวงจร: `enctype`, `request.FILES`, `FileField`/
  `ImageField`, การบันทึกไฟล์ด้วย `.chunks()`
- ✅ เชื่อม CSRF Protection เข้ากับการ submit ฟอร์มแบบ AJAX ผ่าน header
  `X-CSRFToken`
- ✅ สร้าง `CommentForm` แบบ plain Form ที่ใช้งานได้จริง พร้อม View ที่ประกอบ
  `Comment` object เองทีละ field

### 250.6 Checklist ก่อนไป Part ถัดไป

- [ ] สร้างไฟล์ `blog/forms.py` และเขียน `ContactForm` ได้สำเร็จ
- [ ] Render ฟอร์มในเทมเพลตได้ทั้ง 3 วิธี (`as_p`, `as_table`, `as_div`)
    และแบบ render ทีละ field เอง
- [ ] เขียน View ที่จัดการ GET/POST ตาม pattern มาตรฐาน (unbound สำหรับ GET,
    bound + `is_valid()` สำหรับ POST) ได้เอง
- [ ] เขียน `clean_<fieldname>()` และ `clean()` ได้ถูกต้อง (ไม่ลืม `return`)
- [ ] เข้าใจความแตกต่างระหว่าง `initial=` กับ `data=` อย่างชัดเจน
- [ ] สร้างฟอร์มอัปโหลดไฟล์ที่ทำงานได้จริง (มี `enctype`, ใช้ `request.FILES`)
- [ ] ทดสอบส่งฟอร์มผ่าน AJAX ด้วย `fetch()` พร้อมแนบ `X-CSRFToken` สำเร็จ
- [ ] สร้าง `CommentForm` และ View `post_detail_view` ที่บันทึก `Comment`
    ลงฐานข้อมูลได้จริงผ่านเบราว์เซอร์

### 250.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง `SearchForm` (plain `forms.Form`) ที่มี field
`keyword` (`CharField`, `required=False`) และ `category` (`ChoiceField` ที่มี
ตัวเลือกอย่างน้อย 3 หมวด) แล้วสร้าง View `search_posts_view` ที่รับค่าจาก
`request.GET` (ไม่ใช่ `request.POST` — ลองคิดว่าทำไมฟอร์มค้นหาควรใช้ GET)
มา validate ด้วยฟอร์มนี้ แล้วกรอง `Post.objects.filter(...)` ตามคำค้นหา

**แบบฝึกหัดที่ 2**: เพิ่ม `clean_text()` ให้ `CommentForm` ที่สร้างไว้ในขั้นตอนที่
250.2 ให้ตรวจสอบเพิ่มเติมว่าข้อความห้ามยาวเกิน 1000 ตัวอักษร และห้ามมี URL
ปรากฏอยู่ในข้อความ (ใช้ module `re` ตรวจ pattern `http://` หรือ `https://`)
ถ้าพบ ให้ raise `ValidationError` พร้อมข้อความที่อธิบายชัดเจนว่าทำไมถูกปฏิเสธ

**แบบฝึกหัดที่ 3**: สร้าง `RegistrationForm` ที่มี field `username`,
`email`, `password`, `confirm_password` เขียน `clean()` ให้ตรวจสอบว่า
`password` กับ `confirm_password` ตรงกัน โดยใช้ `self.add_error()` ผูก error
เข้ากับ field `confirm_password` โดยเฉพาะ (ไม่ใช่ non-field error) แล้วทดสอบ
ใน Django shell ว่า error ปรากฏที่ field ที่ถูกต้องจริง

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ทำให้หน้า `post_detail.html` ในขั้นตอนที่ 250.4
ส่งคอมเมนต์แบบ AJAX แทนการ submit แบบปกติ (อ้างอิงเทคนิคจากขั้นตอนที่ 249)
โดยเมื่อส่งสำเร็จ ให้เพิ่ม `<div class="comment">` ใหม่เข้าไปใน DOM ทันทีด้วย
JavaScript โดยไม่ต้องรีเฟรชหน้าเว็บทั้งหน้า และแสดง error ใต้ field ที่เกี่ยวข้อง
ถ้า validate ไม่ผ่าน (ต้องปรับ View ให้คืน `JsonResponse` ที่มีทั้งข้อมูล
คอมเมนต์ที่สร้างสำเร็จ และ `form.errors` กรณีล้มเหลว)

### 250.8 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่สอน `ModelForm` ไปเลยตั้งแต่ต้น ในเมื่อโปรเจกต์จริงส่วนใหญ่ใช้
`ModelForm` มากกว่า `forms.Form` เปล่า ๆ?**
A: เป็นความตั้งใจของหลักสูตร `ModelForm` เป็น "ทางลัด" ที่สร้าง Form field
จาก Model field และมี `.save()` ให้อัตโนมัติ — ถ้าเรียน `ModelForm` ก่อนโดย
ไม่เข้าใจ `forms.Form` เปล่า ๆ มาก่อน คุณจะไม่เข้าใจว่า "เบื้องหลัง" ของทางลัด
นั้นทำอะไรบ้าง เมื่อเจอบั๊กหรือกรณีพิเศษที่ `ModelForm` เริ่มไม่พอ (เช่นฟอร์ม
ที่ต้อง custom logic เยอะ หรือฟอร์มที่ไม่ตรงกับ Model ตรง ๆ) คุณจะแก้ปัญหาไม่ได้
เพราะไม่เข้าใจกลไกพื้นฐาน Part 026 จะสอน `ModelForm` แบบเจาะลึก และคุณจะเห็น
ชัดเจนว่ามันลัดขั้นตอนอะไรจาก Part นี้ไปบ้าง

**Q: `form.is_valid()` เรียกได้กี่ครั้ง ผลลัพธ์จะเปลี่ยนไปไหมถ้าเรียกซ้ำ?**
A: เรียกซ้ำได้และผลลัพธ์จะเหมือนเดิมเสมอ (Django cache ผลลัพธ์ไว้ภายในไม่ต้อง
รัน validation ซ้ำทุกครั้งที่เรียก) แต่ธรรมเนียมที่ดีคือเรียกครั้งเดียวแล้วเก็บ
ผลลัพธ์ไว้ในตัวแปร หรือใช้ในเงื่อนไข `if form.is_valid():` แบบที่สอนมาตลอด
Part นี้ ไม่ควรเรียกกระจัดกระจายหลายจุดในโค้ดเดียวกัน

**Q: ถ้าฟอร์มมี field ที่ไม่ได้ประกาศ `required=False` ผู้ใช้เว้นว่างได้ไหม?**
A: ไม่ได้ — ทุก Form field มีค่า `required=True` เป็นค่าเริ่มต้น (ต่างจาก
Model field ที่ค่าเริ่มต้นของ `blank` คือ `False` เช่นกันแต่ทำงานคนละกลไก)
ถ้าต้องการให้ field เป็นทางเลือก (optional) ต้องระบุ `required=False` อย่าง
ชัดเจนเสมอ เหมือนที่ `newsletter_opt_in` และ `honeypot` ทำในตัวอย่างของ Part นี้

**Q: ควรเก็บ Form ทั้งหมดไว้ในไฟล์ `forms.py` เดียว หรือแยกเป็นหลายไฟล์เมื่อ
โปรเจกต์ใหญ่ขึ้น?**
A: สำหรับแอปขนาดเล็กถึงกลาง ไฟล์ `forms.py` เดียวต่อแอปเพียงพอ (เหมือนที่ทำ
ตลอด Part นี้) แต่เมื่อแอปมีฟอร์มจำนวนมาก (สิบตัวขึ้นไป) หลายทีมนิยมเปลี่ยนเป็น
package `forms/` ที่มีไฟล์ย่อยแยกตามหมวดหมู่ เช่น `forms/comment.py`,
`forms/profile.py` แล้ว import รวมไว้ที่ `forms/__init__.py` — หลักการเดียวกับ
ที่ Django แอปขนาดใหญ่มักแยก `views.py` เป็น package `views/` เช่นกัน

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 026: ModelForms และ Formsets** จะพาคุณกลับมาดูทุก pattern ที่เขียนด้วยมือ
ใน Part นี้ (การ copy ค่าจาก `cleaned_data` ไปสร้าง object, การ pre-fill ค่าจาก
instance เดิม) แล้วแสดงให้เห็นว่า `ModelForm` ย่นระยะขั้นตอนเหล่านั้นให้สั้นลง
มากแค่ไหนด้วย `Meta.model`, `Meta.fields` และ method `.save()` ที่สร้างหรือ
อัปเดต object ให้อัตโนมัติ รวมถึง `Formset` สำหรับจัดการฟอร์มหลายชุดพร้อมกันในหน้า
เดียว (เช่น เพิ่มความคิดเห็นหลายรายการพร้อมกัน) เตรียม `CommentForm` และ
`post_detail_view` ที่สร้างไว้ในขั้นตอนที่ 250 ให้พร้อม เพราะเราจะแปลงมันเป็น
`ModelForm` ให้ดูเปรียบเทียบกันแบบ side-by-side ทันทีในขั้นตอนแรกของ Part ถัดไป
