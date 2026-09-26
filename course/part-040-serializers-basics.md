# Part 040: Serializers เบื้องต้น

> **ขั้นตอนที่ 391-400 ของหลักสูตร** | Phase 5: Django REST Framework และ API
>
> เป้าหมายของ Part นี้: เข้าใจ **`rest_framework.serializers.Serializer`** ซึ่งเป็น
> รากฐานที่แท้จริงของทุก Serializer ใน DRF (รวมถึง `ModelSerializer` ที่คุณจะได้เจาะลึก
> ใน Part 041) คุณจะได้เห็นว่า Serializer ทำหน้าที่คล้าย `forms.Form` ที่เรียนไปแล้วใน
> Part 025 มาก — รับข้อมูลดิบ แปลง ตรวจสอบ แล้วส่งออกเป็น Python object หรือ JSON —
> เพียงแต่ Serializer ทำงานได้ **สองทิศทาง** ชัดเจนกว่า: แปลง Python object เป็น
> JSON-friendly data (`to_representation`) และแปลง JSON ที่รับเข้ามาเป็น Python object
> (`to_internal_value`) เมื่อจบ Part นี้ คุณจะเขียน `PostSerializer` แบบ **plain
> `serializers.Serializer`** (ยังไม่ใช่ `ModelSerializer`) ที่ serialize ข้อมูล `Post`
> จริงได้ครบทุกฟิลด์ พร้อม custom validation, `SerializerMethodField`, nested category,
> และ custom field ของตัวเอง ก่อนที่ Part 041 จะแสดงให้เห็นว่า `ModelSerializer`
> ย่นระยะทุกอย่างที่คุณเขียนด้วยมือใน Part นี้ให้สั้นลงได้มากแค่ไหน

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 391: `serializers.Serializer` class เบื้องต้น — ประกาศ field คล้าย `forms.Form`
- ขั้นตอนที่ 392: วงจรชีวิต Serializer — `to_representation()` และ `to_internal_value()`
- ขั้นตอนที่ 393: Validation ใน Serializer — `validate_<field>()` และ `validate()`
- ขั้นตอนที่ 394: Serialize QuerySet ทั้งชุดด้วย `many=True`
- ขั้นตอนที่ 395: `SerializerMethodField` — เพิ่ม field ที่คำนวณเอง
- ขั้นตอนที่ 396: เกริ่น Nested Serializer — แสดง Category ซ้อนใน Post
- ขั้นตอนที่ 397: เขียน Custom Serializer Field เอง
- ขั้นตอนที่ 398: `is_valid()`, `serializer.data` vs `serializer.validated_data`
- ขั้นตอนที่ 399: `Serializer` vs `ModelSerializer` — เกริ่นสิ่งที่รอใน Part 041
- ขั้นตอนที่ 400: สรุปและแบบฝึกหัด — สร้าง `PostSerializer` แบบ plain Serializer ที่ใช้งานได้จริง

---

## ขั้นตอนที่ 391: `serializers.Serializer` class เบื้องต้น — ประกาศ field คล้าย `forms.Form`

### 391.1 ทบทวนสถานะ: คุณมี DRF ติดตั้งแล้วจาก Part 039

Part 039 พาคุณติดตั้ง Django REST Framework และสร้าง API แรกเป็นที่เรียบร้อยแล้ว
ก่อนไปต่อ ให้ตรวจสอบว่า `rest_framework` อยู่ใน `INSTALLED_APPS` ของโปรเจกต์:

```python
# config/settings.py
INSTALLED_APPS = [
    # ... apps มาตรฐานของ Django ...
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "rest_framework",   # เพิ่มใน Part 039

    "blog",
]
```

Part นี้จะไม่พูดถึงการติดตั้ง DRF หรือการสร้าง `APIView` ซ้ำอีก (เรื่องนั้นอยู่ใน
Part 039 และจะเจาะลึกเต็มรูปแบบใน Part 042) แต่จะโฟกัสเฉพาะ **หัวใจของการแปลงข้อมูล**
ที่ทุก API endpoint ต้องพึ่งพา นั่นคือ **Serializer**

### 391.2 ทำไมต้องมี Serializer ทั้งที่มี Model และ Form อยู่แล้ว

คำถามนี้เหมือนกับคำถามใน Part 025 ข้อ 241.1 ที่ถามว่า "ทำไมต้องมี Form ทั้งที่มี
Model" — คำตอบก็เป็นแนวคิดเดียวกัน เพียงแค่เปลี่ยนบริบทจาก **HTML form** เป็น
**JSON ผ่าน HTTP API**:

```
┌──────────────┐   JSON (string   ┌────────────────────┐   Python object   ┌──────────────┐
│ Client (JS,  │   เสมอทาง wire)  │  serializers.       │   (Post instance,  │  Database    │
│ Mobile App,  │ ────────────────>│  Serializer          │   dict, ฯลฯ)       │  ผ่าน ORM    │
│ Postman ฯลฯ) │                  │  (แปลง + ตรวจสอบ)   │ ──────────────────>│              │
└──────────────┘ <────────────────└────────────────────┘ <──────────────────└──────────────┘
                   JSON (แปลงกลับ)
```

- **Model** อธิบายว่า **ฐานข้อมูลเก็บข้อมูลอย่างไร** (เหมือนเดิม ไม่เปลี่ยนแปลง)
- **Form** อธิบายว่า **HTML form รับข้อมูลจากผู้ใช้ที่พิมพ์ในเบราว์เซอร์อย่างไร**
- **Serializer** อธิบายว่า **ข้อมูลที่ส่งผ่าน API (มักเป็น JSON) ควรถูกแปลงเป็น
  Python object และตรวจสอบความถูกต้องอย่างไร ก่อนนำไปใช้ต่อ — และในทางกลับกัน
  Python object (เช่น `Post` instance) ควรถูกแปลงเป็น JSON ที่ client เข้าใจได้
  อย่างไร**

Serializer จึงทำงาน **สองทิศทาง** อย่างชัดเจนกว่า Form มาก: Form เน้นทิศทางเดียว
(รับข้อมูลจากผู้ใช้ → validate) ส่วน Serializer ต้องทำทั้ง **output** (Python →
JSON สำหรับตอบกลับ) และ **input** (JSON → Python สำหรับรับข้อมูลเข้า) เราจะเจาะลึก
สองทิศทางนี้ในขั้นตอนที่ 392

### 391.3 สร้าง Serializer แรก: `CategorySerializer`

Django REST Framework serializer ทุกตัวสืบทอดจาก `rest_framework.serializers.Serializer`
และประกาศ field เป็น class attribute — โครงสร้างหน้าตาแทบจะเหมือน `forms.Form`
ทุกประการ:

```python
# blog/serializers.py
from rest_framework import serializers


class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    slug = serializers.SlugField(max_length=120, read_only=True)
    description = serializers.CharField(required=False, allow_blank=True)
```

เทียบกับ `forms.Form` เวอร์ชันเดียวกัน (สมมติว่ามีอยู่จริงในระบบ) จะเห็นว่า syntax
คล้ายกันมากจนแทบจะ copy-paste แล้วเปลี่ยนแค่ `import`:

```python
# blog/forms.py (สมมติเปรียบเทียบ — ไม่ใช่ของจริงในระบบ)
from django import forms


class CategoryForm(forms.Form):
    name = forms.CharField(max_length=100)
    description = forms.CharField(required=False)
```

### 391.4 ตารางเปรียบเทียบ `forms.Form` กับ `serializers.Serializer`

นี่คือตารางที่สำคัญที่สุดของขั้นตอนนี้ เพราะถ้าคุณผ่าน Part 025 มาแล้ว คุณสามารถ
"แปลความรู้เดิม" มาใช้กับ Serializer ได้เกือบทั้งหมด เพียงแค่รู้ว่าตรงไหนต่างกัน:

| ประเด็น | `django.forms.Form` | `rest_framework.serializers.Serializer` |
|---|---|---|
| แหล่งข้อมูลนำเข้าโดยทั่วไป | HTML form data (`request.POST`) — เป็น string เสมอ | JSON/parsed data (`request.data`) — อาจเป็น dict, list, primitive แล้วแต่ parser |
| ตรวจสอบข้อมูลด้วย | `is_valid()` → `cleaned_data` | `is_valid()` → `validated_data` |
| Error เก็บไว้ที่ | `form.errors` (dict-like) | `serializer.errors` (dict-like, โครงสร้างคล้ายกันมาก) |
| Custom validate ต่อ field | `clean_<fieldname>()` | `validate_<fieldname>()` |
| Custom validate ข้าม field | `clean()` | `validate()` |
| แปลงข้อมูลออก (output) | ไม่มีแนวคิดนี้โดยตรง (Form เน้นรับข้อมูลเข้าเป็นหลัก) | `to_representation()` — แปลง Python object → dict สำหรับ JSON |
| แปลงข้อมูลเข้า (input) | ทำผ่านแต่ละ field's `.clean()` ภายใน | `to_internal_value()` — แปลง dict ดิบ → Python object |
| serialize ข้อมูลหลายรายการพร้อมกัน | ต้องใช้ `Formset` (ซับซ้อนกว่า) | ส่ง `many=True` ตอนสร้าง instance (ง่ายกว่ามาก) |
| ผูกกับ Model โดยอัตโนมัติ | `ModelForm` (Part 026) | `ModelSerializer` (Part 041) |
| จุดประสงค์หลัก | รับข้อมูลจาก HTML form ในเว็บเพจ | แปลงข้อมูลเข้า-ออกสำหรับ REST API |

**ข้อสรุปสำคัญที่ต้องจำ**: เช่นเดียวกับที่ `models.CharField` กับ `forms.CharField`
เป็นคนละ class กันโดยสิ้นเชิง (Part 025 ข้อ 241.4) — `forms.Form` กับ
`serializers.Serializer` ก็เป็นคนละ class คนละ module กันเช่นกัน (`django.forms.Form`
vs `rest_framework.serializers.Serializer`) ที่ syntax คล้ายกันเป็นเพราะทีม DRF
**จงใจออกแบบ API ให้คุ้นเคยสำหรับคนที่เคยใช้ Django Forms มาก่อน** ไม่ใช่เพราะ
สืบทอดหรือใช้กลไกเดียวกัน

### 391.5 Field Types ที่ใช้บ่อยที่สุดใน DRF

DRF มีชุด field ของตัวเองใน `rest_framework.serializers` ที่คล้ายกับทั้ง Model
field และ Form field แต่ก็เป็นคนละ class อีกเช่นกัน:

| Serializer Field | ค่าที่ได้ใน `validated_data` | เทียบเคียงกับ Model Field | เทียบเคียงกับ Form Field |
|---|---|---|---|
| `CharField` | `str` | `models.CharField` / `TextField` | `forms.CharField` |
| `IntegerField` | `int` | `models.IntegerField` | `forms.IntegerField` |
| `BooleanField` | `bool` | `models.BooleanField` | `forms.BooleanField` |
| `DecimalField` | `Decimal` | `models.DecimalField` | `forms.DecimalField` |
| `DateField` | `datetime.date` | `models.DateField` | `forms.DateField` |
| `DateTimeField` | `datetime.datetime` | `models.DateTimeField` | `forms.DateTimeField` |
| `SlugField` | `str` (ตรวจรูปแบบ slug แล้ว) | `models.SlugField` | `forms.SlugField` |
| `EmailField` | `str` (ตรวจรูปแบบอีเมลแล้ว) | `models.EmailField` | `forms.EmailField` |
| `ChoiceField` | ค่าที่เลือก | `models.CharField(choices=...)` | `forms.ChoiceField` |
| `PrimaryKeyRelatedField` | instance ของ Model ที่เกี่ยวข้อง | `models.ForeignKey` / `ManyToManyField` | (ไม่มีเทียบเท่าตรง ๆ ใน Form ปกติ) |
| `SerializerMethodField` | (read-only เท่านั้น) ค่าที่คำนวณจาก method | (ไม่มีเทียบเท่าใน Model) | (ไม่มีเทียบเท่าใน Form) |

สังเกตแถวสุดท้ายสองแถว: `PrimaryKeyRelatedField` และ `SerializerMethodField`
เป็น field ที่ **ไม่มีอยู่ใน Django Forms เลย** เพราะถูกออกแบบมาเฉพาะสำหรับความ
ต้องการของ REST API โดยเฉพาะ — จะเจาะลึกทั้งสองตัวในขั้นตอนที่ 394-395

### 391.6 ทดสอบ Serializer ใน Django Shell

เหมือนกับที่ Part 025 ทดสอบ `ContactForm` ใน shell ก่อนต่อกับ View เราก็ทดสอบ
`CategorySerializer` แบบเดียวกันได้:

```python
>>> from blog.serializers import CategorySerializer
>>> serializer = CategorySerializer(data={
...     "name": "Django Tips",
...     "description": "บทความเกี่ยวกับเทคนิค Django",
... })
>>> serializer.is_valid()
True
>>> serializer.validated_data
{'name': 'Django Tips', 'description': 'บทความเกี่ยวกับเทคนิค Django'}
```

ลองส่งข้อมูลผิด (ไม่มี `name` ที่เป็น field บังคับ):

```python
>>> bad_serializer = CategorySerializer(data={"description": "ทดสอบ"})
>>> bad_serializer.is_valid()
False
>>> bad_serializer.errors
{'name': [ErrorDetail(string='This field is required.', code='required')]}
```

สังเกตว่า `serializer.errors` มีโครงสร้างคล้าย `form.errors` มาก (dict ที่ key คือ
ชื่อ field) เพียงแต่ค่าใน list เป็น `ErrorDetail` object แทนที่จะเป็น string ธรรมดา
(`ErrorDetail` คือ subclass ของ `str` ที่แถม attribute `.code` มาด้วย เพื่อให้
client เขียนโค้ดจัดการ error ตาม error code ได้แม่นยำขึ้น — จะเจาะลึกอีกครั้งใน
Part 050 เรื่อง Testing)

---

## ขั้นตอนที่ 392: วงจรชีวิต Serializer — `to_representation()` และ `to_internal_value()`

### 392.1 ภาพรวมของสองทิศทาง

ทุก Serializer มี method หลักสองตัวที่ทำหน้าที่ตรงข้ามกัน และเป็นแก่นแท้ของกลไก
ทั้งหมดที่ field แต่ละตัวใช้งานอยู่เบื้องหลัง:

```
                    to_representation()
   Python object  ─────────────────────────────>  dict (พร้อมแปลงเป็น JSON)
   (เช่น Post      <─────────────────────────────  (เช่น {"title": "...", ...})
    instance)          to_internal_value()
```

| Method | ทิศทาง | เรียกโดยอัตโนมัติเมื่อ | ผลลัพธ์ |
|---|---|---|---|
| `to_representation(instance)` | Python object → dict | เข้าถึง `serializer.data` | dict ที่พร้อมส่งเป็น JSON response |
| `to_internal_value(data)` | dict ดิบ → Python object | เรียก `is_valid()` | ค่าที่ผ่านการแปลง+ตรวจสอบแล้ว (`validated_data`) |

### 392.2 `to_representation()`: จาก Python Object สู่ JSON

เมื่อคุณสร้าง Serializer โดยส่ง `instance=` (ไม่ใช่ `data=`) แล้วเข้าถึง `.data`
Django REST Framework จะเรียก `to_representation()` ให้อัตโนมัติ:

```python
# blog/models.py (จากที่คุณสร้างไว้ตั้งแต่ Phase 2)
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True, blank=True)
    description = models.TextField(blank=True)

    class Meta:
        ordering = ["name"]
        verbose_name_plural = "categories"

    def __str__(self):
        return self.name
```

```python
>>> from blog.models import Category
>>> from blog.serializers import CategorySerializer
>>> category = Category.objects.create(name="Django Tips", slug="django-tips")
>>> serializer = CategorySerializer(instance=category)
>>> serializer.data
{'id': 1, 'name': 'Django Tips', 'slug': 'django-tips', 'description': ''}
```

สิ่งที่เกิดขึ้นเบื้องหลังคือ `Serializer.to_representation()` (ที่ DRF implement
ให้อัตโนมัติในคลาสแม่) วนลูปทุก field ที่ประกาศไว้ แล้วเรียก `field.get_attribute()`
เพื่อดึงค่าจาก `category` (เช่น `category.name`, `category.slug`) จากนั้นแปลงแต่ละ
ค่าด้วย `field.to_representation()` ของ field นั้น ๆ (เช่น `IntegerField.to_representation()`
แค่ `return int(value)`)

### 392.3 Override `to_representation()` เพื่อคุมผลลัพธ์ทั้งก้อนเอง

บางครั้งการแปลงทีละ field ไม่พอ ต้องการควบคุม "รูปร่าง" ของ output ทั้งก้อน เช่น
อยากรวม field บางตัวเข้าด้วยกัน หรือใส่ metadata เพิ่มเติมที่ไม่ผูกกับ field เดี่ยว ๆ
ทำได้โดย override method นี้ตรง ๆ:

```python
# blog/serializers.py
class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    slug = serializers.SlugField(max_length=120, read_only=True)
    description = serializers.CharField(required=False, allow_blank=True)

    def to_representation(self, instance):
        # เรียก parent ก่อนเสมอ เพื่อให้ field ปกติทำงานตามระบบเดิม
        data = super().to_representation(instance)

        # เพิ่มข้อมูลที่ไม่ได้มาจาก field ใด field หนึ่งโดยตรง
        data["post_count"] = instance.posts.count()
        data["display_name"] = f"หมวดหมู่: {instance.name}"

        return data
```

```python
>>> serializer = CategorySerializer(instance=category)
>>> serializer.data
{'id': 1, 'name': 'Django Tips', 'slug': 'django-tips', 'description': '',
 'post_count': 3, 'display_name': 'หมวดหมู่: Django Tips'}
```

**กฎเหล็ก**: เกือบทุกครั้งที่ override `to_representation()` ควรเรียก
`super().to_representation(instance)` เป็นบรรทัดแรกเสมอ แล้วค่อยแก้ไข/เพิ่มเติม
ผลลัพธ์ที่ได้ ไม่ควรสร้าง dict ขึ้นมาใหม่ทั้งหมดเอง เพราะจะทำให้เสีย behavior
มาตรฐานของแต่ละ field ไปโดยไม่จำเป็น (เช่น การจัดการ `source=`, `read_only=`
ที่ field แต่ละตัวมี logic ของตัวเองอยู่แล้ว)

### 392.4 `to_internal_value()`: จาก JSON ดิบสู่ Python Object

ทิศทางตรงข้าม เมื่อคุณสร้าง Serializer โดยส่ง `data=` แล้วเรียก `is_valid()`
DRF จะเรียก `to_internal_value()` ให้อัตโนมัติเพื่อแปลง dict ดิบ (ที่มักมาจาก
`request.data` ซึ่ง parser แปลง JSON string เป็น Python dict ให้แล้วชั้นหนึ่ง)
ให้กลายเป็นค่าที่ผ่านการ validate และแปลงชนิดข้อมูลแล้ว:

```python
>>> serializer = CategorySerializer(data={"name": "Testing", "description": ""})
>>> serializer.is_valid()
True
>>> serializer.validated_data
OrderedDict([('name', 'Testing'), ('description', '')])
```

สังเกตว่า `slug` และ `id` (ที่ตั้ง `read_only=True`) **หายไปจาก `validated_data`
โดยสิ้นเชิง** เพราะ `to_internal_value()` จะข้าม field ที่เป็น `read_only` ไปเลย
ไม่พยายามอ่านค่าจาก input — นี่คือความหมายที่แท้จริงของ `read_only`: field นั้น
**ปรากฏเฉพาะตอน output (`to_representation`) แต่ไม่ถูกรับเข้ามาตอน input
(`to_internal_value`)**

### 392.5 Override `to_internal_value()` เพื่อรองรับรูปแบบ Input พิเศษ

บางระบบ client ภายนอกอาจส่งข้อมูลมาในรูปแบบที่ไม่ตรงกับชื่อ field ของเราเป๊ะ ๆ
(เช่น API เก่าที่ใช้ `category_name` แทน `name`) เราสามารถ override
`to_internal_value()` เพื่อ "แปลงร่าง" ข้อมูลก่อนส่งต่อให้ field ปกติทำงาน:

```python
class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    slug = serializers.SlugField(max_length=120, read_only=True)
    description = serializers.CharField(required=False, allow_blank=True)

    def to_internal_value(self, data):
        # รองรับ client เก่าที่ยังส่ง key ชื่อ "category_name" มาแทน "name"
        if "category_name" in data and "name" not in data:
            data = {**data, "name": data["category_name"]}

        return super().to_internal_value(data)
```

```python
>>> serializer = CategorySerializer(data={"category_name": "Legacy Client"})
>>> serializer.is_valid()
True
>>> serializer.validated_data["name"]
'Legacy Client'
```

**ข้อควรระวัง**: การ override `to_internal_value()` ควรทำเมื่อจำเป็นจริง ๆ เท่านั้น
(เช่น รองรับ backward compatibility ของ client เก่า) เพราะทำให้ contract ของ API
ไม่ชัดเจน — ทางที่ดีกว่าคือแก้ที่ client ให้ส่งชื่อ field ที่ถูกต้อง หรือใช้
`source=` ที่ field รองรับอยู่แล้ว (จะพูดถึงใน Part 041 ตอน `ModelSerializer`)

### 392.6 สรุปวงจรชีวิตแบบเต็ม

```
สร้าง Serializer(instance=obj)          สร้าง Serializer(data=raw_dict)
         │                                          │
         ▼                                          ▼
   เข้าถึง .data                              เรียก .is_valid()
         │                                          │
         ▼                                          ▼
  to_representation(obj)                    to_internal_value(raw_dict)
         │                                          │
         ▼                                          ▼
  วนทุก field →                              วนทุก field →
  field.to_representation()                  field.run_validation()
         │                                          │
         ▼                                          ▼
   ได้ dict พร้อมส่ง JSON                validate_<field>() ต่อ field (393)
                                                     │
                                                     ▼
                                          validate() ข้าม field ทั้งก้อน (393)
                                                     │
                                                     ▼
                                         ได้ serializer.validated_data
```

---

## ขั้นตอนที่ 393: Validation ใน Serializer — `validate_<field>()` เจาะเฉพาะ field, `validate()` ข้าม field

### 393.1 `validate_<field>()`: เหมือน `clean_<fieldname>()` ใน Form เกือบทุกกระเบียดนิ้ว

รูปแบบเดียวกับที่ Part 025 ข้อ 245 สอนไว้ทุกประการ เพียงเปลี่ยนคำนำหน้าจาก `clean_`
เป็น `validate_`:

```python
def validate_<fieldname>(self, value):
    # ตรวจสอบเพิ่มเติม...
    if <เงื่อนไขที่ไม่ผ่าน>:
        raise serializers.ValidationError("ข้อความ error ที่จะแสดง")
    return value   # สำคัญมาก: ต้อง return ค่ากลับเสมอ
```

ตัวอย่างจริง: ตรวจสอบว่า `title` ของ Post ต้องมีความยาวอย่างน้อย 5 ตัวอักษร และ
ห้ามซ้ำกับ Post ที่มีอยู่แล้ว (case-insensitive):

```python
# blog/serializers.py
from rest_framework import serializers

from .models import Post


class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    is_published = serializers.BooleanField(default=False)

    def validate_title(self, value):
        if len(value.strip()) < 5:
            raise serializers.ValidationError(
                "หัวข้อบทความสั้นเกินไป กรุณาตั้งอย่างน้อย 5 ตัวอักษร"
            )

        # กันหัวข้อซ้ำ — ยกเว้นตัวเองในกรณี update (self.instance จะไม่ใช่ None)
        existing = Post.objects.filter(title__iexact=value.strip())
        if self.instance is not None:
            existing = existing.exclude(pk=self.instance.pk)
        if existing.exists():
            raise serializers.ValidationError("มีบทความชื่อนี้อยู่แล้วในระบบ")

        return value.strip()
```

ทดสอบใน shell:

```python
>>> from blog.serializers import PostSerializer
>>> serializer = PostSerializer(data={"title": "สั้น", "content": "เนื้อหา"})
>>> serializer.is_valid()
False
>>> serializer.errors
{'title': [ErrorDetail(string='หัวข้อบทความสั้นเกินไป กรุณาตั้งอย่างน้อย 5 ตัวอักษร', code='invalid')]}
```

**กฎเหล็กที่พลาดบ่อยที่สุด (เหมือนกับ Form เป๊ะ)**: **ต้อง `return value` เสมอ**
ค่าที่ return จะถูกนำไปแทนที่ค่าเดิมใน `validated_data` ทันที ถ้าลืม return
ค่าของ field นั้นจะกลายเป็น `None` โดยไม่มี error แจ้งเตือนใด ๆ

### 393.2 ความแตกต่างเล็กน้อยที่สำคัญ: `self.instance`

สังเกตโค้ดข้างบนที่เช็ค `self.instance is not None` — นี่คือสิ่งที่ Serializer มี
แต่ Form ไม่ค่อยถูกใช้แบบนี้บ่อยนัก: เมื่อสร้าง Serializer แบบ **update** (ส่งทั้ง
`instance=` และ `data=` พร้อมกัน) `self.instance` จะชี้ไปที่ object เดิมที่กำลัง
แก้ไข ทำให้ validation รู้ว่า "กำลังแก้ไขของเดิม ไม่ใช่สร้างใหม่" และสามารถยกเว้น
ตัวเองจากการเช็คค่าซ้ำได้ (pattern นี้สำคัญมากเมื่อ API รองรับทั้ง `POST` สร้างใหม่
และ `PUT`/`PATCH` แก้ไขของเดิม — จะเจาะลึกใน Part 042)

```python
# สร้างใหม่: ไม่มี instance
serializer = PostSerializer(data=incoming_data)

# แก้ไขของเดิม: มี instance อยู่แล้ว
serializer = PostSerializer(instance=existing_post, data=incoming_data)
```

### 393.3 `validate()`: ตรวจสอบข้ามหลาย Field พร้อมกัน

เหมือนกับ `clean()` ใน Form (Part 025 ข้อ 246) เมื่อ logic การตรวจสอบต้องเทียบค่า
ระหว่าง field มากกว่าหนึ่งตัว ต้อง override `validate()` ของทั้ง Serializer ซึ่งจะ
ถูกเรียก **หลังจาก** `validate_<fieldname>()` ของทุก field ทำงานเสร็จหมดแล้วเสมอ

ตัวอย่าง: ถ้า `is_published=True` แล้ว `content` ต้องมีความยาวอย่างน้อย 50 ตัวอักษร
(กันไม่ให้เผยแพร่บทความที่ยังไม่มีเนื้อหาจริงจัง):

```python
class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    is_published = serializers.BooleanField(default=False)

    def validate_title(self, value):
        # ... เหมือนเดิมจาก 393.1 ...
        return value.strip()

    def validate(self, attrs):
        # attrs คือ dict ของค่าที่ validate_<field>() แต่ละตัวคืนกลับมาแล้ว
        is_published = attrs.get("is_published", False)
        content = attrs.get("content", "")

        if is_published and len(content.strip()) < 50:
            raise serializers.ValidationError(
                "บทความที่จะเผยแพร่ต้องมีเนื้อหาอย่างน้อย 50 ตัวอักษร"
            )

        return attrs   # สำคัญมาก: ต้อง return attrs เสมอ
```

**ข้อแตกต่างจาก Form ที่ควรสังเกต**: DRF's `validate()` **ไม่จำเป็นต้องเรียก**
`super().validate(attrs)` เหมือนที่ Form's `clean()` ต้องเรียก `super().clean()`
เสมอ เพราะ `Serializer.validate()` เวอร์ชัน base ของ DRF เป็นแค่
`return attrs` เฉย ๆ ไม่มี logic อะไรให้ต้องรัน แต่การเรียก `super().validate(attrs)`
ก็ยังเป็นนิสัยที่ดีถ้าคุณมีการสืบทอด Serializer หลายชั้น (เพื่อไม่ให้ logic ของ
parent class หายไปโดยไม่ตั้งใจ)

### 393.4 ผูก Error เข้ากับ Field เฉพาะจาก `validate()`

เหมือนกับ `self.add_error()` ของ Form (Part 025 ข้อ 246.3) DRF รองรับการ raise
`ValidationError` เป็น dict เพื่อผูก error เข้ากับ field ใดฟิลด์หนึ่งโดยเฉพาะ
แทนที่จะให้ error ลอยเป็นภาพรวม:

```python
def validate(self, attrs):
    is_published = attrs.get("is_published", False)
    content = attrs.get("content", "")

    if is_published and len(content.strip()) < 50:
        raise serializers.ValidationError({
            "content": "บทความที่จะเผยแพร่ต้องมีเนื้อหาอย่างน้อย 50 ตัวอักษร"
        })

    return attrs
```

```python
>>> serializer = PostSerializer(data={
...     "title": "หัวข้อทดสอบยาวพอ",
...     "content": "สั้น",
...     "is_published": True,
... })
>>> serializer.is_valid()
False
>>> serializer.errors
{'content': [ErrorDetail(string='บทความที่จะเผยแพร่ต้องมีเนื้อหาอย่างน้อย 50 ตัวอักษร', code='invalid')]}
```

| วิธีแจ้ง Error ใน `validate()` | ปรากฏที่ไหนใน `serializer.errors` | ใช้เมื่อไร |
|---|---|---|
| `raise ValidationError("ข้อความ")` | `non_field_errors` | error ภาพรวม ไม่เจาะจง field เดียว |
| `raise ValidationError({"content": "ข้อความ"})` | ผูกกับ `content` โดยตรง | ต้องการให้ client รู้ว่า field ไหนที่มีปัญหาชัดเจน |
| `raise ValidationError(["ข้อความ 1", "ข้อความ 2"])` | `non_field_errors` เป็น list หลายข้อ | มี error ภาพรวมมากกว่า 1 ข้อพร้อมกัน |

### 393.5 ลำดับการทำงานของ Validation แบบเต็ม

```
is_valid()
   │
   ▼
1. field.run_validation(value)  ── ตรวจ required, max_length, ชนิดข้อมูลพื้นฐาน
   │                                 (ทำทีละ field ตามลำดับที่ประกาศไว้ในคลาส)
   ▼
2. validate_<fieldname>(value)  ── ถ้ามี method นี้ประกาศไว้ เรียกทันทีหลัง field
   │                                 นั้น validate ผ่านขั้นพื้นฐานแล้ว
   ▼
3. validate(attrs)              ── เรียกครั้งเดียวหลังจากทุก field ผ่านขั้นตอน 1-2
   │                                 ครบหมดแล้วเท่านั้น
   ▼
4. ไม่มี error เลย → is_valid() คืน True, เติม validated_data
   มี error อย่างน้อย 1 จุด → is_valid() คืน False, เติม errors
```

---

## ขั้นตอนที่ 394: Serialize QuerySet ทั้งชุดด้วย `many=True`

### 394.1 ปัญหาที่ `many=True` แก้ไข

`forms.Form` แบบเดี่ยวจัดการฟอร์มเดียวได้เท่านั้น ถ้าต้องการฟอร์มหลายชุดพร้อมกัน
(เช่น แก้ไขคอมเมนต์ 5 รายการในหน้าเดียว) ต้องใช้ `Formset` ที่ตั้งค่าค่อนข้างซับซ้อน
(`management_form`, `TOTAL_FORMS`, `INITIAL_FORMS` ฯลฯ)

DRF แก้ปัญหานี้ให้ง่ายกว่ามาก: **ทุก Serializer สามารถ serialize รายการหลายชิ้น
พร้อมกันได้ทันที** โดยแค่ส่ง `many=True` ตอนสร้าง instance ไม่ต้องเขียนคลาสใหม่
หรือเปลี่ยนโครงสร้างใด ๆ เลย:

```python
>>> from blog.models import Category
>>> from blog.serializers import CategorySerializer
>>> categories = Category.objects.all()
>>> serializer = CategorySerializer(categories, many=True)
>>> serializer.data
[
    {'id': 1, 'name': 'Django Tips', 'slug': 'django-tips', 'description': ''},
    {'id': 2, 'name': 'DRF', 'slug': 'drf', 'description': 'บทความเกี่ยวกับ REST Framework'},
    {'id': 3, 'name': 'Deployment', 'slug': 'deployment', 'description': ''},
]
```

### 394.2 กลไกเบื้องหลัง `many=True`

เมื่อคุณส่ง `many=True` DRF จะไม่ได้ทำให้ `CategorySerializer` เพียง instance เดียว
กลายเป็น "ฉลาดขึ้น" แต่เบื้องหลังมันสร้าง **`ListSerializer`** ขึ้นมาห่อหุ้ม
`CategorySerializer` ของคุณไว้อีกชั้นหนึ่งโดยอัตโนมัติ — `ListSerializer` ทำหน้าที่
วนลูปแต่ละ item ในลิสต์ แล้วเรียก `CategorySerializer` (child serializer) ตัวเดิม
ซ้ำ ๆ กับแต่ละ item:

```
CategorySerializer(queryset, many=True)
         │
         ▼
   ListSerializer (สร้างขึ้นอัตโนมัติ)
         │
         │  วนลูปทุก item ใน queryset
         ▼
   CategorySerializer(item_1)  →  dict 1
   CategorySerializer(item_2)  →  dict 2
   CategorySerializer(item_3)  →  dict 3
         │
         ▼
   [dict 1, dict 2, dict 3]   ← ผลลัพธ์สุดท้ายใน .data
```

นี่คือเหตุผลที่คุณ **ไม่ต้องเขียน logic การวนลูปเองเลย** — field, `validate_<field>()`,
`validate()` ทุกอย่างที่คุณเขียนไว้ใน `CategorySerializer` (สำหรับ 1 รายการ) จะถูก
นำไปใช้กับทุก item ในลิสต์โดยอัตโนมัติ

### 394.3 ใช้ `many=True` กับการรับ Input ด้วย

`many=True` ใช้ได้ทั้ง output (`instance=`) และ input (`data=`) — เหมาะกับ API
ที่รับข้อมูลหลายรายการพร้อมกันในคำขอเดียว (bulk create):

```python
>>> from blog.serializers import CategorySerializer
>>> raw_data = [
...     {"name": "Testing"},
...     {"name": "Security"},
... ]
>>> serializer = CategorySerializer(data=raw_data, many=True)
>>> serializer.is_valid()
True
>>> serializer.validated_data
[OrderedDict([('name', 'Testing')]), OrderedDict([('name', 'Security')])]
```

**ข้อควรระวัง**: `ListSerializer` ที่ห่อหุ้มอัตโนมัตินี้ **ไม่มี** method
`create()`/`update()` ให้ทำงานกับหลาย object พร้อมกันโดยอัตโนมัติ (จะพูดถึงใน
ขั้นตอนที่ 398 และ 400 ว่า plain `Serializer` ต้องเขียน `create()`/`update()` เอง
เสมอ) ถ้าต้องการ bulk create/update จริง ต้อง override `list_serializer_class`
และเขียน logic เองอย่างชัดเจน — เรื่องนี้เป็นหัวข้อขั้นสูงที่จะกล่าวถึงอีกครั้งใน
Part 043 ตอน Generic API Views

### 394.4 ตัวอย่างใช้งานจริงใน View: แสดงรายการ Post ทั้งหมด

ตัวอย่างนี้ใช้ `APIView` แบบง่าย ๆ ที่เรียนไปแล้วบางส่วนใน Part 039 (จะเจาะลึกเต็ม
รูปแบบใน Part 042) เพื่อให้เห็นภาพว่า `many=True` ถูกใช้งานจริงในโปรเจกต์อย่างไร:

```python
# blog/views.py
from rest_framework.views import APIView
from rest_framework.response import Response

from .models import Post
from .serializers import PostSerializer


class PostListAPIView(APIView):
    def get(self, request):
        posts = Post.objects.filter(is_published=True).select_related("category")
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
```

```python
# blog/urls.py
from django.urls import path

from . import views

urlpatterns = [
    # ... URL เดิมจาก Part ก่อนหน้า ...
    path("api/posts/", views.PostListAPIView.as_view(), name="api-post-list"),
]
```

เมื่อเรียก `GET /api/posts/` จะได้ JSON array กลับมา:

```json
[
    {"id": 1, "title": "แนะนำ Django ORM", "content": "...", "is_published": true},
    {"id": 2, "title": "รู้จัก DRF Serializers", "content": "...", "is_published": true}
]
```

---

## ขั้นตอนที่ 395: `SerializerMethodField` — เพิ่ม field ที่คำนวณเอง

### 395.1 ปัญหาที่ `SerializerMethodField` แก้ไข

บางครั้งข้อมูลที่ต้องการส่งออกไม่ได้เก็บอยู่ตรง ๆ ใน attribute ของ Model แต่ต้อง
**คำนวณขึ้นมาใหม่** จากข้อมูลอื่น เช่น "เวลาที่ใช้อ่านบทความโดยประมาณ" ซึ่งไม่มี
คอลัมน์ `reading_time` อยู่ในฐานข้อมูลจริง แต่คำนวณได้จากความยาวของ `content`

`SerializerMethodField` คือ field พิเศษที่ **read-only เสมอ** (ไม่มีทางใช้รับ
input ได้ เพราะไม่มีความหมายที่จะให้ client ส่งค่าที่คำนวณเองมา) และดึงค่าจาก
method ที่คุณเขียนขึ้นในคลาส Serializer เอง

### 395.2 สร้าง `reading_time` Field

```python
# blog/serializers.py
from rest_framework import serializers

from .models import Post

WORDS_PER_MINUTE = 200   # ความเร็วอ่านเฉลี่ยของคนทั่วไป


class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    is_published = serializers.BooleanField(default=False)
    reading_time = serializers.SerializerMethodField()

    def get_reading_time(self, obj):
        """คำนวณเวลาอ่านโดยประมาณจากจำนวนคำใน content"""
        word_count = len(obj.content.split())
        minutes = max(1, round(word_count / WORDS_PER_MINUTE))
        return f"{minutes} นาที"
```

**กฎการตั้งชื่อที่ต้องจำ**: DRF จะมองหา method ที่ชื่อ `get_<fieldname>` โดย
อัตโนมัติเสมอ — ถ้า field ชื่อ `reading_time` DRF จะเรียก `self.get_reading_time(obj)`
ให้เอง (ชื่อ method ต้องตรงกันเป๊ะ ไม่เช่นนั้นจะได้ `AttributeError`) หากต้องการ
ตั้งชื่อ method อื่น สามารถระบุผ่านพารามิเตอร์ `method_name` ได้:

```python
reading_time = serializers.SerializerMethodField(method_name="calculate_reading_time")

def calculate_reading_time(self, obj):
    ...
```

### 395.3 ทดสอบผลลัพธ์

```python
>>> from blog.models import Post, Category
>>> from blog.serializers import PostSerializer
>>> category = Category.objects.get(slug="django-tips")
>>> post = Post.objects.create(
...     title="เจาะลึก Django ORM ฉบับสมบูรณ์",
...     slug="deep-dive-django-orm",
...     content=" ".join(["คำ"] * 450),   # จำลองเนื้อหา 450 คำ
...     category=category,
... )
>>> serializer = PostSerializer(post)
>>> serializer.data
{'id': 1, 'title': 'เจาะลึก Django ORM ฉบับสมบูรณ์', 'content': '...',
 'is_published': False, 'reading_time': '2 นาที'}
```

### 395.4 `SerializerMethodField` เข้าถึง `context` ได้ด้วย

`obj` ที่ method รับเข้ามาคือ instance ตัวเต็ม (`Post` object) ดังนั้นสามารถเข้าถึง
attribute ใด ๆ ของมันได้ รวมถึง relation อื่น ๆ นอกจากนี้ยังเข้าถึง `self.context`
ได้เสมอ (มักใช้ส่ง `request` เข้ามาเพื่อสร้าง absolute URL หรือเช็คสิทธิ์ผู้ใช้
ปัจจุบัน — เรื่องนี้จะสำคัญมากขึ้นเมื่อถึง Part 045 เรื่อง Permissions):

```python
class PostSerializer(serializers.Serializer):
    # ... fields เดิม ...
    is_owned_by_me = serializers.SerializerMethodField()

    def get_is_owned_by_me(self, obj):
        request = self.context.get("request")
        if request and request.user.is_authenticated:
            return obj.category.created_by_id == request.user.id
        return False
```

```python
# ตัวอย่างการส่ง context เข้าไปตอนสร้าง Serializer
serializer = PostSerializer(post, context={"request": request})
```

### 395.5 ข้อจำกัดสำคัญ: `SerializerMethodField` เป็น Read-Only เสมอ

```python
>>> serializer = PostSerializer(data={
...     "title": "ทดสอบ", "content": "เนื้อหา", "reading_time": "99 นาที",
... })
>>> serializer.is_valid()
True
>>> serializer.validated_data
OrderedDict([('title', 'ทดสอบ'), ('content', 'เนื้อหา')])
```

สังเกตว่า `reading_time` ที่ส่งมาใน `data` **หายไปโดยสิ้นเชิง** จาก
`validated_data` — DRF เพิกเฉยต่อค่าที่ client พยายามส่งมาสำหรับ field ประเภทนี้
เสมอ เพราะ `SerializerMethodField` ไม่มี `to_internal_value()` ที่มีความหมายอะไร
เลย มันถูกออกแบบมาเพื่อ **output เท่านั้น**

---

## ขั้นตอนที่ 396: เกริ่น Nested Serializer — แสดง Category ซ้อนใน Post

### 396.1 ปัญหา: `PrimaryKeyRelatedField` ให้ข้อมูลน้อยเกินไป

ถ้าใช้ `PrimaryKeyRelatedField` สำหรับ `category` ผลลัพธ์ที่ client ได้จะเป็นแค่
เลข ID เปล่า ๆ:

```python
class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    category = serializers.PrimaryKeyRelatedField(read_only=True)
```

```python
>>> PostSerializer(post).data
{'id': 1, 'title': 'เจาะลึก Django ORM ฉบับสมบูรณ์', 'category': 3}
```

client ที่ได้ `"category": 3` มาต้อง**ยิง request เพิ่มอีกครั้ง** ไปที่
`/api/categories/3/` เพื่อรู้ว่าหมวดหมู่นี้ชื่ออะไร — ไม่สะดวกเลยสำหรับ use case
ที่พบบ่อยมาก เช่น หน้ารายการบทความที่ต้องแสดงชื่อหมวดหมู่ควบคู่ไปด้วย

### 396.2 แก้ปัญหาด้วย Nested Serializer

วิธีแก้คือ **ใช้ Serializer อีกตัวหนึ่งเป็น field** ของ Serializer หลัก — เรียกว่า
**Nested Serializer**:

```python
# blog/serializers.py
class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    slug = serializers.SlugField(max_length=120, read_only=True)


class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    category = CategorySerializer(read_only=True)   # ← nested serializer
```

```python
>>> PostSerializer(post).data
{
    'id': 1,
    'title': 'เจาะลึก Django ORM ฉบับสมบูรณ์',
    'category': {'id': 3, 'name': 'Django Tips', 'slug': 'django-tips'}
}
```

ตอนนี้ `category` กลายเป็น **object เต็มรูปแบบ** แทนที่จะเป็นแค่ตัวเลข ID —
เบื้องหลังการทำงานยังเป็นกลไกเดียวกับที่เรียนมาตลอด Part นี้ทุกประการ: เมื่อ
`PostSerializer.to_representation()` วนลูปถึง field `category` มันจะเรียก
`CategorySerializer.to_representation(post.category)` ซ้อนเข้าไปอีกชั้นหนึ่ง
โดยอัตโนมัติ (Serializer เรียก Serializer ซ้อนกันได้ไม่จำกัดชั้น)

### 396.3 ข้อจำกัดของ Nested Serializer แบบ `read_only`

ในตัวอย่างข้างบน `category = CategorySerializer(read_only=True)` ทำงานได้ดี
สำหรับ **output** แต่ถ้าลบ `read_only=True` ออกแล้วพยายามใช้รับ **input** ด้วย
(เช่น ให้ client ส่ง object หมวดหมู่ทั้งก้อนมาสร้าง Post ใหม่พร้อมหมวดหมู่ใหม่
ในคำขอเดียว) จะซับซ้อนขึ้นมาก เพราะ DRF ไม่รู้ว่าควรจะ **สร้าง Category ใหม่**
หรือ **ค้นหา Category ที่มีอยู่แล้วมาผูก** จาก data ที่ซ้อนเข้ามา — เรื่องนี้ต้อง
override `create()`/`update()` ของ Serializer หลักให้จัดการ nested data เองอย่าง
ชัดเจน:

```python
# ตัวอย่างแนวคิดคร่าว ๆ (ยังไม่ใช่ implementation ที่สมบูรณ์)
class PostSerializer(serializers.Serializer):
    category = CategorySerializer()   # ไม่ใส่ read_only แล้ว รับ input ได้ด้วย

    def create(self, validated_data):
        category_data = validated_data.pop("category")
        category, _ = Category.objects.get_or_create(**category_data)
        return Post.objects.create(category=category, **validated_data)
```

**นี่คือจุดที่หลักสูตรจะหยุดไว้แค่นี้ก่อน** — การเขียน nested serializer ให้รองรับ
ทั้ง read และ write อย่างสมบูรณ์ (รวมถึงการจัดการ M2M อย่าง `tags` ที่ซ้อนกันได้
หลายชั้น, `PrimaryKeyRelatedField(many=True)`, `SlugRelatedField`, และการเขียน
`create()`/`update()` ที่ปลอดภัยสำหรับ nested data ที่ซับซ้อน) เป็นหัวข้อเต็ม
รูปแบบของ **Part 041: ModelSerializer และ Nested Serializers** ตอนนี้ขอให้คุณ
เข้าใจแค่แนวคิดหลักก่อนว่า **"Serializer สามารถซ้อนกันเป็น field ได้"** ก็เพียงพอ

### 396.4 ตารางสรุป: ทางเลือกในการแสดง Relation

| วิธีแสดง `category` | Output ที่ได้ | เหมาะกับ |
|---|---|---|
| `PrimaryKeyRelatedField()` | `3` (แค่ ID) | API ที่เน้นความเบา ให้ client จัดการ join เอง |
| `StringRelatedField()` | `"Django Tips"` (เรียก `__str__()`) | แสดงชื่อสั้น ๆ โดยไม่ต้องสร้าง Serializer แยก |
| `SlugRelatedField(slug_field="slug")` | `"django-tips"` | ใช้ slug เป็น identifier ที่อ่านง่ายกว่า ID |
| `CategorySerializer(read_only=True)` (nested) | `{"id": 3, "name": "...", "slug": "..."}` | ต้องการข้อมูลเต็มของ relation โดยไม่ต้องยิง request เพิ่ม |

---

## ขั้นตอนที่ 397: เขียน Custom Serializer Field เอง

### 397.1 เมื่อไรควรเขียน Custom Field

Field มาตรฐานของ DRF ครอบคลุมเกือบทุกกรณีใช้งานทั่วไป แต่บางครั้งคุณมี **รูปแบบ
ข้อมูลเฉพาะทาง** ที่ต้องแปลงไป-กลับด้วย logic พิเศษซ้ำ ๆ หลายที่ในโปรเจกต์ เช่น
รหัสสี hex (`#RRGGBB`) ที่ต้องการเก็บเป็น tuple `(r, g, b)` ฝั่ง Python แต่รับ-ส่ง
เป็น string `"#1a2b3c"` ฝั่ง JSON เสมอ — กรณีแบบนี้ควรเขียน **Custom Field**
แยกออกมาเพื่อใช้ซ้ำได้หลาย Serializer โดยไม่ต้องเขียน logic ซ้ำ ๆ

### 397.2 โครงสร้างพื้นฐานของ Custom Field

Custom Field ทุกตัวสืบทอดจาก `serializers.Field` (ตัวฐานที่สุด ไม่มี validation
สำเร็จรูปใด ๆ ให้เลย) แล้ว implement สอง method บังคับ ซึ่งตรงกับสองทิศทางที่
เรียนไปในขั้นตอนที่ 392 พอดี:

```python
# blog/fields.py
from rest_framework import serializers


class HexColorField(serializers.Field):
    """
    Custom field สำหรับรหัสสี — เก็บเป็น tuple (r, g, b) ฝั่ง Python
    แต่รับ-ส่งเป็น string แบบ "#RRGGBB" ฝั่ง JSON
    """

    def to_representation(self, value):
        # value คือ tuple (r, g, b) จาก attribute ของ object — แปลงเป็น "#RRGGBB"
        r, g, b = value
        return f"#{r:02x}{g:02x}{b:02x}"

    def to_internal_value(self, data):
        # data คือ string ดิบจาก client เช่น "#1a2b3c" — แปลงเป็น tuple (r, g, b)
        if not isinstance(data, str) or not data.startswith("#") or len(data) != 7:
            raise serializers.ValidationError(
                'รูปแบบสีไม่ถูกต้อง ต้องเป็น "#RRGGBB" เช่น "#1a2b3c"'
            )

        try:
            r = int(data[1:3], 16)
            g = int(data[3:5], 16)
            b = int(data[5:7], 16)
        except ValueError:
            raise serializers.ValidationError("รหัสสีต้องเป็นเลขฐาน 16 เท่านั้น")

        return (r, g, b)
```

### 397.3 นำ Custom Field ไปใช้ใน Serializer

```python
# blog/serializers.py
from .fields import HexColorField


class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    badge_color = HexColorField(required=False)
```

```python
>>> from blog.serializers import CategorySerializer
>>> serializer = CategorySerializer(data={"name": "Security", "badge_color": "#ff5733"})
>>> serializer.is_valid()
True
>>> serializer.validated_data
OrderedDict([('name', 'Security'), ('badge_color', (255, 87, 51))])
```

```python
>>> class FakeCategory:
...     name = "Security"
...     badge_color = (255, 87, 51)
>>> serializer = CategorySerializer(FakeCategory())
>>> serializer.data
{'id': None, 'name': 'Security', 'badge_color': '#ff5733'}
```

### 397.4 เปรียบเทียบ: Custom Field vs `SerializerMethodField` vs Override `to_representation`

สามวิธีนี้ดูคล้ายกันเพราะทั้งหมด "แปลงข้อมูลแบบพิเศษ" แต่มีจุดประสงค์ต่างกัน:

| วิธี | รับ input ได้ไหม | ใช้ซ้ำข้าม Serializer ได้ไหม | เหมาะกับ |
|---|---|---|---|
| Custom `serializers.Field` | ได้ (implement `to_internal_value`) | ได้ง่ายมาก — import ไปใช้ที่ไหนก็ได้ | รูปแบบข้อมูลเฉพาะทางที่ใช้ซ้ำหลายจุด (เช่น สี, พิกัด GPS, ช่วงเวลา) |
| `SerializerMethodField` | ไม่ได้ (read-only เสมอ) | ต้อง copy method ไปทุกคลาส | ค่าที่คำนวณจากหลาย attribute ของ object เดียว เฉพาะ output |
| Override `to_representation()` ทั้งคลาส | ได้ (คู่กับ `to_internal_value()` ของคลาส) | ผูกกับ Serializer นั้นเท่านั้น | ปรับโครงสร้าง output ทั้งก้อน ไม่ใช่แค่ field เดียว |

### 397.5 เพิ่ม Validation เพิ่มเติมให้ Custom Field ด้วย `run_validation()`

ถ้าต้องการให้ custom field รองรับ validator เพิ่มเติมแบบเดียวกับ field มาตรฐาน
(เช่น `required`, `allow_null`) ควรสืบทอดจาก `serializers.Field` ตามปกติ เพราะ
คลาสแม่จัดการเรื่อง `required`/`allow_null`/`default` ให้อยู่แล้วผ่าน
`run_validation()` — สิ่งที่คุณต้องทำเองมีแค่ `to_representation()` และ
`to_internal_value()` สองตัวเท่านั้น ไม่ต้อง override `run_validation()` เอง
เว้นแต่มีความต้องการพิเศษจริง ๆ

---

## ขั้นตอนที่ 398: `is_valid()`, ความแตกต่างระหว่าง `serializer.data` กับ `serializer.validated_data`

### 398.1 สองคำที่มือใหม่สับสนบ่อยที่สุดใน DRF

`serializer.data` และ `serializer.validated_data` ฟังดูคล้ายกันมาก แต่ทำหน้าที่
ต่างกันอย่างสิ้นเชิง และการสลับใช้ผิดที่คือบั๊กที่พบบ่อยที่สุดของมือใหม่ DRF:

| ประเด็น | `serializer.data` | `serializer.validated_data` |
|---|---|---|
| มาจาก method ไหน | `to_representation()` | `to_internal_value()` + `validate_<field>()` + `validate()` |
| ใช้ได้เมื่อไร | หลังสร้างด้วย `instance=` (ไม่ต้องเรียก `is_valid()`) หรือหลังเรียก `is_valid()` สำเร็จ | **ต้องเรียก `is_valid()` ก่อนเสมอ** และต้องคืน `True` |
| ชนิดข้อมูล | dict ที่ **พร้อมแปลงเป็น JSON** (string, number, list, dict เท่านั้น) | dict ของ **Python object จริง** (`Decimal`, `date`, Model instance ฯลฯ) |
| ใช้ทำอะไร | ส่งกลับเป็น HTTP Response (`Response(serializer.data)`) | นำไปสร้าง/แก้ไข object จริง (`Post.objects.create(**validated_data)`) |
| เข้าถึงก่อนเรียก `is_valid()` ได้ไหม | ได้ (ถ้าสร้างด้วย `instance=`) แต่ **ไม่ได้** ถ้าสร้างด้วย `data=` อย่างเดียว | **ไม่ได้เด็ดขาด** จะได้ `AssertionError` ทันที |

### 398.2 ทดสอบให้เห็นความต่างชัด ๆ

```python
>>> from blog.serializers import PostSerializer
>>> serializer = PostSerializer(data={
...     "title": "ทดสอบ is_valid",
...     "content": "เนื้อหาทดสอบสำหรับดูความแตกต่างของ data และ validated_data",
... })

# ยังไม่เรียก is_valid() — เข้าถึง validated_data จะพัง
>>> serializer.validated_data
Traceback (most recent call last):
    ...
AssertionError: You must call `.is_valid()` before accessing `.validated_data`.

>>> serializer.is_valid()
True

>>> serializer.validated_data
OrderedDict([('title', 'ทดสอบ is_valid'), ('content', 'เนื้อหาทดสอบ...')])

>>> serializer.data
{'title': 'ทดสอบ is_valid', 'content': 'เนื้อหาทดสอบ...', 'is_published': False,
 'reading_time': '1 นาที'}
```

สังเกต 2 จุดสำคัญ:

1. `validated_data` **ไม่มี** `id`, `reading_time` (field ที่เป็น `read_only`
   หรือ `SerializerMethodField` จะไม่ปรากฏในนี้เลย เพราะไม่ได้มาจาก input)
2. `data` **มี** `reading_time` ปรากฏอยู่ (เพราะ `data` มาจาก `to_representation()`
   ซึ่งรวม field read-only ทั้งหมดด้วย) แต่ในตัวอย่างนี้ `id` เป็น `None` เพราะยัง
   ไม่มี object จริงถูกสร้างขึ้น (เป็นแค่ unbound-ish state ที่ยังไม่ผ่าน `save()`)

### 398.3 `is_valid(raise_exception=True)`: ทางลัดสำหรับ View

ใน View จริง แทนที่จะเช็ค `if serializer.is_valid():` แล้ว return error response
เองทุกครั้ง DRF มีทางลัดที่สะดวกกว่า — ส่ง `raise_exception=True` แล้วให้ DRF
จัดการ error response ให้อัตโนมัติ (คืน `400 Bad Request` พร้อม `serializer.errors`
เป็น JSON body):

```python
# blog/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .serializers import PostSerializer


class PostCreateAPIView(APIView):
    def post(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)   # ถ้าไม่ผ่าน จะ raise ValidationError
                                                       # และ DRF แปลงเป็น 400 Response ให้เอง
        # โค้ดหลังบรรทัดนี้รันเมื่อ validate ผ่านเท่านั้น
        validated = serializer.validated_data
        post = Post.objects.create(
            title=validated["title"],
            content=validated["content"],
            is_published=validated.get("is_published", False),
        )
        return Response(PostSerializer(post).data, status=status.HTTP_201_CREATED)
```

**ข้อควรระวัง**: `raise_exception=True` ใช้งานได้เฉพาะภายใน View ที่ DRF จัดการ
exception handling ให้อยู่แล้ว (เช่น `APIView`, `ViewSet`) — ถ้าเรียกใน context
อื่น (เช่น management command, Celery task) `ValidationError` จะไม่ถูกจับโดย
อัตโนมัติ และต้อง `try/except` เอง

### 398.4 `is_valid()` ถูก Cache ผลลัพธ์ไว้เหมือน Form

เช่นเดียวกับ `form.is_valid()` การเรียก `serializer.is_valid()` ซ้ำหลายครั้งจะ
ได้ผลลัพธ์เดิมเสมอ (ไม่รัน validation ซ้ำ) แต่ถ้าเรียกก่อนที่ `data` จะถูกกำหนด
(สร้าง Serializer ด้วย `instance=` อย่างเดียวโดยไม่มี `data=`) จะได้ `AssertionError`
ทันที เพราะ Serializer ที่ไม่มี `data` ไม่มีอะไรให้ validate — ต่างจาก Form
ที่ unbound form เรียก `is_valid()` แล้วได้ `False` เฉย ๆ โดยไม่ error (Part 025
ข้อ 243.1)

---

## ขั้นตอนที่ 399: `Serializer` vs `ModelSerializer` — เกริ่นสิ่งที่รอใน Part 041

### 399.1 ทบทวน: ทุกอย่างที่คุณเขียนมาตลอด Part นี้ ทำด้วยมือทั้งหมด

ลองมองย้อนกลับไปดู `PostSerializer` ที่ประกอบขึ้นมาทีละส่วนตลอด Part นี้ — คุณ
ต้อง **ประกาศ field ทุกตัวเอง** (`title`, `content`, `is_published`, ...) ทั้งที่
field เหล่านี้ **มีอยู่แล้วใน `Post` model** ตั้งแต่ Phase 2 คุณกำลังเขียนสิ่งที่
Django รู้อยู่แล้วซ้ำอีกรอบหนึ่ง — เช่นเดียวกับที่ Part 025 ข้อ 241.1 ตั้งคำถาม
เดียวกันเรื่อง `forms.Form` กับ `ModelForm`

นอกจากนี้ plain `Serializer` **ไม่มี** `.save()` ให้ใช้เลย ถ้าต้องการบันทึกข้อมูล
ต้องเขียน `create()`/`update()` เอง แล้วเรียก `Post.objects.create(**validated_data)`
ด้วยมือทุกครั้ง (จะสาธิตแบบเต็มในขั้นตอนที่ 400)

### 399.2 `ModelSerializer` คือทางลัดที่ผูกกับ Model โดยตรง

`rest_framework.serializers.ModelSerializer` (หัวข้อหลักของ **Part 041**) แก้
ปัญหานี้ด้วยหลักการเดียวกับที่ `ModelForm` แก้ปัญหาให้ `forms.Form`: **สร้าง field
จาก Model field ให้อัตโนมัติ** ผ่าน `Meta.model` และ `Meta.fields`:

```python
# ตัวอย่างเปรียบเทียบ (จะเจาะลึกเต็มรูปแบบใน Part 041)
from rest_framework import serializers

from .models import Post


class PostModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ["id", "title", "slug", "content", "is_published",
                  "created_at", "updated_at", "category", "tags"]
```

โค้ดเพียงเท่านี้จะสร้าง field ทั้งหมดให้อัตโนมัติ **พร้อม** `.save()`,
`.create()`, `.update()` ที่ทำงานได้ทันทีโดยไม่ต้องเขียนเอง — เทียบกับ
`PostSerializer` แบบ plain ที่คุณเขียนมาตลอด Part นี้ซึ่งต้องประกาศทุก field
และเขียน `create()`/`update()` เองทั้งหมด

### 399.3 ตารางเปรียบเทียบเบื้องต้น (จะขยายความเต็มใน Part 041)

| ประเด็น | `serializers.Serializer` (Part นี้) | `serializers.ModelSerializer` (Part 041) |
|---|---|---|
| ต้องประกาศ field เองทุกตัวไหม | ต้อง | ไม่ต้อง — สร้างจาก `Meta.fields` อัตโนมัติ |
| มี `.save()` ให้ใช้ไหม | ไม่มี — ต้องเขียน `create()`/`update()` เอง | มี `create()`/`update()` default ให้แล้ว (override ได้) |
| ผูกกับ Model โดยตรงไหม | ไม่ผูก — ใช้ได้กับข้อมูลอะไรก็ได้ที่ไม่ใช่ Model | ผูกกับ Model เสมอผ่าน `Meta.model` |
| เหมาะกับ | ข้อมูลที่ไม่ตรงกับ Model โดยตรง (เช่น ผลรวมสถิติ, response ของ third-party API, ฟอร์มค้นหาที่ซับซ้อน) | CRUD API ทั่วไปที่ทำงานกับ Model ตรง ๆ (กรณีส่วนใหญ่ในโปรเจกต์จริง) |
| Nested relation | ต้องเขียน nested serializer + `create()`/`update()` เอง (ขั้นตอนที่ 396) | มี `PrimaryKeyRelatedField` อัตโนมัติ และรองรับ nested writable ได้สะดวกกว่า |
| ใช้บ่อยแค่ไหนในโปรเจกต์จริง | ส่วนน้อย (เฉพาะกรณีพิเศษ) | ส่วนใหญ่ (มากกว่า 80% ของ Serializer ในโปรเจกต์จริงทั่วไป) |

### 399.4 ทำไมหลักสูตรถึงสอน `Serializer` เปล่า ๆ ก่อน ทั้งที่ใช้จริงน้อยกว่า

เหตุผลเดียวกับ Part 025 ที่สอน `forms.Form` ก่อน `ModelForm`: ถ้าเรียน
`ModelSerializer` ก่อนโดยไม่เข้าใจกลไกพื้นฐาน คุณจะไม่รู้ว่า "เบื้องหลัง" ของ
`Meta.fields` ทำอะไรบ้าง เมื่อเจอกรณีที่ `ModelSerializer` เริ่มไม่พอ (เช่น API
ที่รับ input ไม่ตรงกับ Model เป๊ะ ๆ, ต้อง validate ข้ามหลาย model, หรือ endpoint
ที่ไม่ได้ CRUD Model ตรง ๆ เลย เช่น endpoint คำนวณสถิติ) คุณจะไม่รู้จะแก้ปัญหา
อย่างไร เพราะไม่เข้าใจว่า `to_representation()`, `to_internal_value()`,
`validate()`, `create()`, `update()` ทำงานร่วมกันอย่างไรใต้ผิวของ `ModelSerializer`

Part 041 จะนำ `PostSerializer` ที่คุณสร้างไว้ในขั้นตอนที่ 400 มาแปลงเป็น
`ModelSerializer` แบบ **side-by-side** ให้เห็นชัดเจนว่าทางลัดนั้นย่นระยะอะไรไปบ้าง
— เหมือนกับที่ Part 026 ทำกับ `CommentForm` จาก Part 025

---

## ขั้นตอนที่ 400: สรุปและแบบฝึกหัด

### 400.1 โมเดลอ้างอิงฉบับเต็มที่ใช้ตลอด Part นี้

```python
# blog/models.py
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=120, unique=True, blank=True)
    description = models.TextField(blank=True)

    class Meta:
        ordering = ["name"]
        verbose_name_plural = "categories"

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(max_length=60, unique=True, blank=True)

    class Meta:
        ordering = ["name"]

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    content = models.TextField()
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    category = models.ForeignKey(
        Category, on_delete=models.PROTECT, related_name="posts",
    )
    tags = models.ManyToManyField(Tag, related_name="posts", blank=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title
```

### 400.2 `PostSerializer` แบบ Plain Serializer ฉบับสมบูรณ์

นี่คือ Serializer เต็มรูปแบบที่รวมทุกเทคนิคที่เรียนมาตลอด Part นี้เข้าด้วยกัน:
field types พื้นฐาน (391), `validate_<field>()` และ `validate()` (393),
`SerializerMethodField` (395), nested `CategorySerializer` (396), และที่สำคัญ
ที่สุดคือ `create()`/`update()` ที่ plain `Serializer` **ต้องเขียนเอง** เสมอ
(ต่างจาก `ModelSerializer` ที่จะเจาะลึกใน Part 041):

```python
# blog/serializers.py
from rest_framework import serializers

from .models import Category, Post, Tag

WORDS_PER_MINUTE = 200


class CategorySerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=100)
    slug = serializers.SlugField(max_length=120, read_only=True)
    description = serializers.CharField(required=False, allow_blank=True)


class TagSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=50)
    slug = serializers.SlugField(max_length=60, read_only=True)


class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    slug = serializers.SlugField(max_length=220, read_only=True)
    content = serializers.CharField()
    is_published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)
    updated_at = serializers.DateTimeField(read_only=True)

    # nested serializer สำหรับ output ที่อ่านง่าย (396) — เขียนได้เฉพาะรับ ID ตอน input
    category_detail = CategorySerializer(source="category", read_only=True)
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(), write_only=True,
    )
    tags_detail = TagSerializer(source="tags", many=True, read_only=True)
    tags = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(), many=True, write_only=True, required=False,
    )

    reading_time = serializers.SerializerMethodField()

    def get_reading_time(self, obj):
        word_count = len(obj.content.split())
        minutes = max(1, round(word_count / WORDS_PER_MINUTE))
        return f"{minutes} นาที"

    def validate_title(self, value):
        cleaned = value.strip()
        if len(cleaned) < 5:
            raise serializers.ValidationError(
                "หัวข้อบทความสั้นเกินไป กรุณาตั้งอย่างน้อย 5 ตัวอักษร"
            )

        existing = Post.objects.filter(title__iexact=cleaned)
        if self.instance is not None:
            existing = existing.exclude(pk=self.instance.pk)
        if existing.exists():
            raise serializers.ValidationError("มีบทความชื่อนี้อยู่แล้วในระบบ")

        return cleaned

    def validate(self, attrs):
        is_published = attrs.get("is_published", False)
        content = attrs.get("content", "")

        if is_published and len(content.strip()) < 50:
            raise serializers.ValidationError({
                "content": "บทความที่จะเผยแพร่ต้องมีเนื้อหาอย่างน้อย 50 ตัวอักษร",
            })

        return attrs

    def create(self, validated_data):
        # plain Serializer ไม่มี .save() ให้ใช้ฟรี ๆ — ต้องประกอบ object เอง
        tags = validated_data.pop("tags", [])

        from django.utils.text import slugify
        validated_data["slug"] = slugify(validated_data["title"])

        post = Post.objects.create(**validated_data)
        post.tags.set(tags)
        return post

    def update(self, instance, validated_data):
        tags = validated_data.pop("tags", None)

        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        if tags is not None:
            instance.tags.set(tags)

        return instance
```

**จุดที่ควรสังเกตเป็นพิเศษ**: การใช้ `source="category"` คู่กับชื่อ field
`category_detail` เป็นเทคนิคที่ทำให้ field เดียวกันของ Model ปรากฏได้สองรูปแบบ
พร้อมกัน — `category` (รับเฉพาะ ID ตอน input ด้วย `write_only=True`) และ
`category_detail` (แสดง object เต็มตอน output ด้วย `read_only=True`) วิธีนี้แก้
ปัญหาที่กล่าวถึงในขั้นตอนที่ 396.3 ได้โดยไม่ต้องเขียน `create()`/`update()`
ที่ซับซ้อนสำหรับ nested writable data

### 400.3 ทดสอบ `PostSerializer` แบบครบวงจรใน Django Shell

```python
>>> from blog.models import Category, Tag, Post
>>> from blog.serializers import PostSerializer

>>> category = Category.objects.create(name="Django Tips", slug="django-tips")
>>> tag_orm = Tag.objects.create(name="ORM", slug="orm")
>>> tag_api = Tag.objects.create(name="API", slug="api")

# --- สร้าง Post ใหม่ผ่าน Serializer ---
>>> serializer = PostSerializer(data={
...     "title": "เจาะลึก Django ORM แบบมืออาชีพ",
...     "content": "เนื้อหาอธิบาย QuerySet, select_related, prefetch_related "
...                 "อย่างละเอียดสำหรับนักพัฒนาที่ต้องการเข้าใจการทำงานเบื้องหลัง "
...                 "ของ Django ORM อย่างถ่องแท้ตั้งแต่ต้นจนจบ",
...     "is_published": True,
...     "category": category.id,
...     "tags": [tag_orm.id, tag_api.id],
... })
>>> serializer.is_valid()
True
>>> post = serializer.save()   # เรียก create() ที่เขียนไว้ให้อัตโนมัติ
>>> post.slug
'เจาะลึก-django-orm-แบบมืออาชีพ'

# --- Serialize กลับออกมาดู output เต็มรูปแบบ ---
>>> PostSerializer(post).data
{
    'id': 1,
    'title': 'เจาะลึก Django ORM แบบมืออาชีพ',
    'slug': 'เจาะลึก-django-orm-แบบมืออาชีพ',
    'content': 'เนื้อหาอธิบาย QuerySet...',
    'is_published': True,
    'created_at': '2026-09-26T10:00:00Z',
    'updated_at': '2026-09-26T10:00:00Z',
    'category_detail': {'id': 1, 'name': 'Django Tips', 'slug': 'django-tips', 'description': ''},
    'tags_detail': [{'id': 1, 'name': 'ORM', 'slug': 'orm'}, {'id': 2, 'name': 'API', 'slug': 'api'}],
    'reading_time': '1 นาที'
}

# --- ตรวจสอบว่า validation ป้องกันหัวข้อสั้นจริง ---
>>> bad = PostSerializer(data={"title": "สั้น", "content": "-", "category": category.id})
>>> bad.is_valid()
False
>>> bad.errors
{'title': [ErrorDetail(string='หัวข้อบทความสั้นเกินไป กรุณาตั้งอย่างน้อย 5 ตัวอักษร', code='invalid')]}

# --- แก้ไข Post เดิมผ่าน Serializer (instance= + data=) ---
>>> update_serializer = PostSerializer(
...     instance=post,
...     data={
...         "title": post.title,
...         "content": post.content,
...         "is_published": False,
...         "category": category.id,
...         "tags": [tag_orm.id],
...     },
... )
>>> update_serializer.is_valid()
True
>>> updated_post = update_serializer.save()   # เรียก update() เพราะมี instance
>>> updated_post.is_published
False
>>> list(updated_post.tags.values_list("name", flat=True))
['ORM']
```

สังเกตว่า `serializer.save()` **ฉลาดพอที่จะเลือกเรียก `create()` หรือ `update()`
ให้อัตโนมัติ** โดยดูจากว่า Serializer ถูกสร้างมาพร้อม `instance=` หรือไม่ — นี่คือ
method เดียวที่ plain `Serializer` มีให้ฟรี ๆ (สืบทอดจากคลาสแม่) ส่วน `create()`
กับ `update()` เองต้องเขียนเสมอเมื่อใช้ `Serializer` แบบ plain

### 400.4 View แบบเต็มที่ใช้ `PostSerializer` จริง

```python
# blog/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .models import Post
from .serializers import PostSerializer


class PostListCreateAPIView(APIView):
    def get(self, request):
        posts = Post.objects.select_related("category").prefetch_related("tags")
        serializer = PostSerializer(posts, many=True, context={"request": request})
        return Response(serializer.data)

    def post(self, request):
        serializer = PostSerializer(data=request.data, context={"request": request})
        serializer.is_valid(raise_exception=True)
        post = serializer.save()
        return Response(
            PostSerializer(post, context={"request": request}).data,
            status=status.HTTP_201_CREATED,
        )


class PostDetailAPIView(APIView):
    def get_object(self, pk):
        return Post.objects.select_related("category").prefetch_related("tags").get(pk=pk)

    def get(self, request, pk):
        post = self.get_object(pk)
        serializer = PostSerializer(post, context={"request": request})
        return Response(serializer.data)

    def put(self, request, pk):
        post = self.get_object(pk)
        serializer = PostSerializer(instance=post, data=request.data, context={"request": request})
        serializer.is_valid(raise_exception=True)
        updated_post = serializer.save()
        return Response(PostSerializer(updated_post, context={"request": request}).data)
```

```python
# blog/urls.py
from django.urls import path

from . import views

urlpatterns = [
    # ... URL เดิมจาก Part ก่อนหน้า ...
    path("api/posts/", views.PostListCreateAPIView.as_view(), name="api-post-list-create"),
    path("api/posts/<int:pk>/", views.PostDetailAPIView.as_view(), name="api-post-detail"),
]
```

> **หมายเหตุ**: `APIView` แบบ function-by-method ที่เห็นในตัวอย่างนี้ยังเป็นแค่
> การใช้งานเบื้องต้นเพื่อทดสอบ Serializer เท่านั้น การออกแบบ URL routing, HTTP
> method, status code, และ error handling อย่างเป็นระบบเต็มรูปแบบจะอยู่ใน
> Part 042 (API Views: Function-Based และ APIView) และ Part 044 (ViewSets และ
> Routers) — ตอนนี้ขอให้โฟกัสที่ตัว Serializer เป็นหลัก

### 400.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า `serializers.Serializer` มีโครงสร้างและปรัชญาคล้าย `forms.Form`
  มาก — ประกาศ field เป็น class attribute, มี `is_valid()`, มี error handling
  แบบเดียวกัน แต่เป็นคนละ class คนละ module กันโดยสิ้นเชิง
- ✅ เข้าใจวงจรชีวิตสองทิศทางของ Serializer: `to_representation()` (Python →
  JSON) และ `to_internal_value()` (JSON → Python) และรู้วิธี override ทั้งคู่
- ✅ เขียน Custom Validation ได้ทั้งระดับ field เดียว (`validate_<field>()`)
  และข้าม field (`validate()` พร้อมผูก error เข้ากับ field เฉพาะ)
- ✅ Serialize ข้อมูลหลายรายการพร้อมกันด้วย `many=True` และเข้าใจกลไก
  `ListSerializer` เบื้องหลัง
- ✅ เพิ่ม field ที่คำนวณเองด้วย `SerializerMethodField` พร้อมเข้าถึง `context`
- ✅ เข้าใจแนวคิด Nested Serializer เบื้องต้น (แสดง Category ซ้อนใน Post)
  และรู้ข้อจำกัดของมันเมื่อต้องรับ input ที่ซับซ้อน
- ✅ เขียน Custom Serializer Field ของตัวเองโดย subclass `serializers.Field`
- ✅ แยกความแตกต่างระหว่าง `serializer.data` (สำหรับ output) กับ
  `serializer.validated_data` (สำหรับนำไปสร้าง/แก้ไข object) ได้อย่างชัดเจน
- ✅ เข้าใจภาพรวมความแตกต่างระหว่าง `Serializer` กับ `ModelSerializer` และรู้ว่า
  ทำไมหลักสูตรถึงสอนแบบ plain ก่อน
- ✅ สร้าง `PostSerializer` แบบ plain Serializer ที่ใช้งานได้จริงครบทุกด้าน:
  field ปกติ, nested read, `write_only` field, validation หลายชั้น,
  `SerializerMethodField`, และ `create()`/`update()` ที่เขียนเอง

### 400.6 Checklist ก่อนไป Part ถัดไป

- [ ] สร้างไฟล์ `blog/serializers.py` และเขียน `CategorySerializer`,
    `TagSerializer` แบบ plain Serializer ได้สำเร็จ
- [ ] ทดสอบ `to_representation()` และ `to_internal_value()` ทั้งแบบ default
    และแบบ override เองใน Django shell
- [ ] เขียน `validate_<fieldname>()` และ `validate()` ได้ถูกต้อง (ไม่ลืม
    `return`) พร้อมทดสอบว่า error ผูกกับ field ที่ถูกต้อง
- [ ] Serialize QuerySet ด้วย `many=True` สำเร็จ และเข้าใจว่าเบื้องหลังคือ
    `ListSerializer`
- [ ] เพิ่ม `SerializerMethodField` อย่างน้อย 1 ตัว (เช่น `reading_time`)
    ให้ทำงานถูกต้อง
- [ ] เขียน Nested Serializer แสดง `Category` ซ้อนใน `Post` ได้สำเร็จ
- [ ] เขียน Custom Field ของตัวเองอย่างน้อย 1 ตัว โดย subclass
    `serializers.Field`
- [ ] อธิบายความแตกต่างระหว่าง `.data` กับ `.validated_data` ได้ด้วยคำพูด
    ของตัวเอง โดยไม่ต้องเปิดเอกสาร
- [ ] สร้าง `PostSerializer` ฉบับสมบูรณ์ที่มี `create()`/`update()` ทำงานได้
    จริงผ่าน Django shell และผ่าน `APIView`

### 400.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้าง `CommentSerializer` (plain `serializers.Serializer`)
สำหรับ Model `Comment` ที่มี field `author`, `text`, `created_at` (สมมติว่า
Comment ผูกกับ `Post` ผ่าน ForeignKey เหมือนใน Part 025) เขียน `validate_text()`
ให้บังคับความยาวอย่างน้อย 10 ตัวอักษร และห้ามมี URL ปรากฏในข้อความ (ใช้ module
`re` เหมือนที่ทำกับ `CommentForm` ใน Part 025 แบบฝึกหัดที่ 2) แล้วทดสอบใน shell

**แบบฝึกหัดที่ 2**: เพิ่ม `SerializerMethodField` ชื่อ `excerpt` ให้
`PostSerializer` ที่สร้างไว้ในขั้นตอนที่ 400 โดยให้ตัดเนื้อหา `content` มาแสดง
แค่ 100 ตัวอักษรแรก แล้วต่อท้ายด้วย `"..."` ถ้าเนื้อหายาวกว่านั้น (ถ้าสั้นกว่า
100 ตัวอักษรให้แสดงเต็มโดยไม่ต้องมี `"..."`)

**แบบฝึกหัดที่ 3**: เขียน Custom Serializer Field ชื่อ `ReadableFileSizeField`
ที่แปลง `int` (จำนวน byte เช่น `1548576`) ให้กลายเป็น string ที่อ่านง่าย เช่น
`"1.5 MB"` ตอน output (`to_representation`) และแปลงกลับจาก string เป็น `int`
byte ตอน input (`to_internal_value`) — ทดสอบทั้งสองทิศทางให้ครบใน Django shell

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: แก้ไข `PostSerializer.validate()` ที่สร้างไว้ใน
ขั้นตอนที่ 400 ให้เพิ่มกฎใหม่: ถ้า `category` ที่เลือกมามีชื่อ `"Archived"`
(หมวดหมู่พิเศษที่หมายถึงบทความที่เก็บถาวรแล้ว) จะต้องไม่สามารถตั้ง
`is_published=True` ได้ (raise error ผูกกับ field `is_published`) แล้วเขียน
เทสสั้น ๆ ใน Django shell พิสูจน์ว่ากฎนี้ทำงานถูกต้องทั้งกรณีผ่านและไม่ผ่าน

### 400.8 คำถามที่พบบ่อย (FAQ)

**Q: ทำไมไม่สอน `ModelSerializer` ไปเลยตั้งแต่ต้น ในเมื่อโปรเจกต์จริงส่วนใหญ่
ใช้ `ModelSerializer` มากกว่า `Serializer` เปล่า ๆ?**
A: เป็นความตั้งใจของหลักสูตรเช่นเดียวกับที่ Part 025 สอน `forms.Form` ก่อน
`ModelForm` — `ModelSerializer` เป็น "ทางลัด" ที่สร้าง field จาก Model field
และมี `.save()`/`create()`/`update()` ให้อัตโนมัติ ถ้าเรียนมันก่อนโดยไม่เข้าใจ
`Serializer` เปล่า ๆ มาก่อน คุณจะไม่รู้ว่าเบื้องหลังทางลัดนั้นทำอะไรบ้าง เมื่อ
เจอ endpoint ที่ไม่ตรงกับ Model ตรง ๆ (เช่น endpoint คำนวณสถิติ, endpoint ที่รวม
ข้อมูลจากหลาย Model) คุณจะไม่รู้จะออกแบบ Serializer อย่างไร Part 041 จะสอน
`ModelSerializer` แบบเจาะลึกและแสดงให้เห็นชัดเจนว่ามันลัดขั้นตอนอะไรไปจาก Part นี้

**Q: `serializer.save()` เรียกได้กี่ครั้ง เหมือน `form.is_valid()` ที่ cache
ผลลัพธ์ไว้หรือไม่?**
A: ต่างจาก `is_valid()` — `save()` **ไม่ได้ถูก cache** และสามารถสร้าง object
ซ้ำได้ทุกครั้งที่เรียก (ถ้าเป็น `create()`) ดังนั้นต้องระวังไม่เรียก `save()`
ซ้ำโดยไม่ตั้งใจ (เช่น ในโค้ดที่มี logic แตกแขนงซับซ้อน) เพราะอาจสร้างข้อมูลซ้ำ
ในฐานข้อมูลโดยไม่รู้ตัว ต่างจาก `is_valid()` ที่เรียกซ้ำได้อย่างปลอดภัยเสมอ

**Q: ถ้า field เป็นทั้ง `read_only=True` และมีอยู่ใน `data` ที่ client ส่งมา
จะเกิดอะไรขึ้น?**
A: ค่านั้นจะถูกเพิกเฉยโดยสิ้นเชิง ไม่มี error ใด ๆ เกิดขึ้น field ที่เป็น
`read_only=True` จะไม่ถูกประมวลผลใน `to_internal_value()` เลย เหมือนที่แสดงให้
เห็นในขั้นตอนที่ 392.4 (`id` และ `slug` หายไปจาก `validated_data` แม้ client
จะพยายามส่งมา) พฤติกรรมนี้ต่างจาก Django Forms ที่ไม่มีแนวคิด `read_only` แบบนี้
โดยตรง (ฟิลด์ของ Form ทุกตัวเป็น "รับ input" โดยธรรมชาติ)

**Q: ควรตั้งชื่อไฟล์ `serializers.py` เดียวสำหรับทั้งแอป หรือแยกเป็นหลายไฟล์?**
A: หลักการเดียวกับ `forms.py` ใน Part 025 ข้อ 250.8 — แอปขนาดเล็กถึงกลางใช้
ไฟล์ `serializers.py` เดียวพอ แต่เมื่อแอปมี Serializer จำนวนมาก นิยมเปลี่ยนเป็น
package `serializers/` ที่แยกไฟล์ย่อยตามหมวดหมู่ เช่น `serializers/post.py`,
`serializers/category.py` แล้ว import รวมไว้ที่ `serializers/__init__.py`

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 041: ModelSerializer และ Nested Serializers** จะนำ `PostSerializer`,
`CategorySerializer`, และ `TagSerializer` ที่คุณเขียนด้วยมือทั้งหมดในขั้นตอนที่
400 มาแปลงเป็น `ModelSerializer` แบบ **side-by-side** ให้เห็นชัดเจนว่า
`Meta.model` + `Meta.fields` ย่นระยะการประกาศ field ที่ซ้ำซ้อนไปได้มากแค่ไหน
คุณจะได้เรียนรู้ `depth`, `PrimaryKeyRelatedField` แบบอัตโนมัติ, การเขียน
Nested Serializer ที่รองรับทั้ง read และ write อย่างสมบูรณ์ (รวมถึงการจัดการ
M2M อย่าง `tags` ที่ซับซ้อนกว่าที่เห็นในขั้นตอนที่ 396), `SerializerMethodField`
ที่ใช้ร่วมกับ `ModelSerializer`, และการ override `create()`/`update()` ของ
`ModelSerializer` เมื่อ default behavior ไม่พอ เตรียม `PostSerializer` ฉบับ
สมบูรณ์จากขั้นตอนที่ 400 ให้พร้อม เพราะเราจะเปรียบเทียบทุกบรรทัดกับเวอร์ชัน
`ModelSerializer` แบบทันทีในขั้นตอนแรกของ Part ถัดไป
