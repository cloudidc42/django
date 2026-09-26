# Part 027: Form Validation ขั้นสูงและ Custom Widgets

> **ขั้นตอนที่ 261-270 ของหลักสูตร** | Phase 3: Views, Templates, Forms และ CBV
>
> เป้าหมายของ Part นี้: เจาะลึกระบบ **Validator** ของ Django ทั้งแบบ built-in
> (`MinValueValidator`, `RegexValidator`, `EmailValidator` ฯลฯ) และการเขียน
> Custom Validator function เอง, เรียนรู้การสร้าง **Custom Widget** ตั้งแต่
> widget ง่าย ๆ อย่าง `RatingWidget` ไปจนถึง widget ที่ต้องแนบ CSS/JS ของตัวเอง
> ผ่าน `Media` class, เปรียบเทียบสองแนวทางการจัดสไตล์ฟอร์มที่นิยมที่สุดใน
> ระบบนิเวศ Django คือ `django-widget-tweaks` กับ `django-crispy-forms`,
> ปรับแต่ง `ClearableFileInput` ให้แสดง preview รูปภาพ, ทำความเข้าใจว่าทำไม
> Django แยก Field กับ Widget ออกเป็นสองเลเยอร์อย่างเจาะลึก, สร้าง Dynamic
> Form ที่เพิ่ม/ลบ field ตามเงื่อนไข และสุดท้ายผสาน Custom Widget เข้ากับ
> Formset ที่เรียนใน Part 026 เมื่อจบ Part นี้ คุณจะสามารถสร้างฟอร์มระดับ
> production ที่มี validation รัดกุม หน้าตาสวยงาม และ interactive ได้ด้วยตัวเอง
> โดยไม่ต้องพึ่งพา JavaScript framework หนัก ๆ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 261: Built-in Validators — `MinValueValidator`, `MaxValueValidator`, `RegexValidator`, `EmailValidator`, `URLValidator`, `MinLengthValidator` ใน Model field และ Form field
- ขั้นตอนที่ 262: เขียน Custom Validator function เอง — validate คำหยาบ, validate เบอร์โทรไทย
- ขั้นตอนที่ 263: เขียน Custom Widget โดย subclass widget ที่มีอยู่ — `RatingWidget` render เป็นดาว 1-5
- ขั้นตอนที่ 264: Widget `Media` class — แนบ CSS/JS ให้ widget เอง (date picker ที่ต้องใช้ JS library)
- ขั้นตอนที่ 265: จัดสไตล์ฟอร์มด้วย django-widget-tweaks และ django-crispy-forms (เปรียบเทียบสองแนวทาง)
- ขั้นตอนที่ 266: Customize `ClearableFileInput` — แสดง preview รูปภาพปัจจุบันก่อนอัปโหลดใหม่
- ขั้นตอนที่ 267: ความแตกต่างระหว่าง Field กับ Widget แบบเจาะลึก — ทำไม Django แยกสองเลเยอร์นี้ออกจากกัน
- ขั้นตอนที่ 268: Dynamic Forms — เพิ่ม/ลบ field แบบมีเงื่อนไขตามข้อมูลที่ส่งมา
- ขั้นตอนที่ 269: ผสาน Custom Widget เข้ากับ Formset จาก Part 026
- ขั้นตอนที่ 270: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 261: Built-in Validators — ใช้ทั้งใน Model field และ Form field

### 261.1 Validator คืออะไร และอยู่ตรงไหนในวงจรชีวิตของ Field

ใน Part 025 คุณเขียน custom validation ด้วย `clean_<fieldname>()` ไปแล้ว แต่ Django
ยังมีกลไกอีกชั้นหนึ่งที่ **เบากว่าและนำกลับมาใช้ซ้ำได้ง่ายกว่า** เรียกว่า
**Validator** — เป็นเพียง **callable** (ฟังก์ชันหรือ class ที่มี `__call__`) ที่รับค่า
เข้ามา 1 ตัว แล้ว `raise django.core.exceptions.ValidationError` ถ้าค่านั้นไม่ผ่าน
เงื่อนไข หรือไม่ทำอะไรเลยถ้าผ่าน (ไม่ต้อง `return` ค่าใด ๆ)

จุดที่ทำให้ validator ต่างจาก `clean_<fieldname>()` คือ **validator ผูกกับ field
โดยตรง ณ ตอนประกาศ** ผ่านพารามิเตอร์ `validators=[...]` และที่สำคัญที่สุด:
**validator ตัวเดียวกันใช้ได้ทั้งใน Model field และ Form field** เพราะทั้งสองฝั่ง
เรียก validator ชุดเดียวกันจาก `django.core.validators` — นี่คือหัวใจของหลักการ
DRY ที่ Django ยึดถือ: เขียนกฎ validation ครั้งเดียว ใช้ได้ทั้งตอนบันทึกลง
ฐานข้อมูล (ผ่าน `full_clean()` ของ Model) และตอนรับข้อมูลจากฟอร์ม

```
┌─────────────────────┐
│  Validator function │   ← กฎเดียว เขียนครั้งเดียว
│  (callable)          │
└──────────┬───────────┘
           │ ใช้ร่วมกันได้
     ┌─────┴─────┐
     ▼           ▼
Model field   Form field
(validators=[...]) (validators=[...])
```

### 261.2 Validator ที่มากับ Django (`django.core.validators`)

| Validator | ตรวจสอบอะไร | ใช้กับ Field ประเภทไหน |
|---|---|---|
| `MinValueValidator(limit_value)` | ค่าตัวเลขต้อง >= limit_value | `IntegerField`, `DecimalField`, `FloatField` |
| `MaxValueValidator(limit_value)` | ค่าตัวเลขต้อง <= limit_value | `IntegerField`, `DecimalField`, `FloatField` |
| `MinLengthValidator(limit_value)` | ความยาว string/list ต้อง >= limit_value | `CharField`, `TextField` |
| `MaxLengthValidator(limit_value)` | ความยาว string/list ต้อง <= limit_value | `CharField`, `TextField` |
| `RegexValidator(regex, message=...)` | ค่าต้องตรงกับ regular expression | `CharField` |
| `EmailValidator(message=...)` | รูปแบบอีเมลถูกต้อง | `CharField` (ปกติใช้ `EmailField` ซึ่งใส่ให้อัตโนมัติแล้ว) |
| `URLValidator(schemes=[...])` | รูปแบบ URL ถูกต้อง | `CharField` (ปกติใช้ `URLField` ซึ่งใส่ให้อัตโนมัติแล้ว) |
| `validate_slug` | เป็น slug ที่ถูกต้อง (a-z, 0-9, -, _) | `SlugField` (ใส่ให้อัตโนมัติแล้ว) |
| `validate_ipv46_address` | เป็น IP address (v4 หรือ v6) ที่ถูกต้อง | `GenericIPAddressField` (ใส่ให้อัตโนมัติแล้ว) |
| `FileExtensionValidator(allowed_extensions=[...])` | นามสกุลไฟล์อยู่ในรายการที่อนุญาต | `FileField`, `ImageField` |
| `DecimalValidator(max_digits, decimal_places)` | จำนวนหลักของ Decimal ถูกต้อง | `DecimalField` (ใส่ให้อัตโนมัติแล้ว) |

### 261.3 ใช้ Validator ใน Model Field

```python
# products/models.py
from django.core.validators import (
    MinValueValidator,
    MaxValueValidator,
    MinLengthValidator,
    RegexValidator,
)
from django.db import models


class Product(models.Model):
    name = models.CharField(
        max_length=200,
        validators=[MinLengthValidator(3, message="ชื่อสินค้าต้องมีอย่างน้อย 3 ตัวอักษร")],
    )
    sku = models.CharField(
        max_length=20,
        validators=[
            RegexValidator(
                regex=r"^[A-Z]{3}-\d{4}$",
                message="รหัสสินค้า (SKU) ต้องอยู่ในรูปแบบ AAA-0000 เช่น TSH-0001",
            )
        ],
    )
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0.01, message="ราคาต้องมากกว่า 0 บาท")],
    )
    discount_percent = models.PositiveIntegerField(
        default=0,
        validators=[MaxValueValidator(90, message="ส่วนลดสูงสุดไม่เกิน 90%")],
    )
```

**ข้อควรระวังสำคัญ**: validator บน Model field **ไม่ได้ทำงานอัตโนมัติตอนบันทึก
ผ่าน `.save()`** เพราะ `.save()` ไม่ได้เรียก `full_clean()` ให้เอง (ต่างจาก
`ModelForm` ที่เรียก `full_clean()` ให้อัตโนมัติเสมอ — เจาะลึกใน Part 026)
ถ้าต้องการบังคับ validate ตอนบันทึกตรง ๆ ผ่าน shell หรือ script ต้องเรียกเอง:

```python
>>> product = Product(name="AB", sku="bad-sku", price=-5)
>>> product.save()   # ผ่านฉลุย! ไม่มีการ validate ใด ๆ เกิดขึ้น (นี่คือพฤติกรรมปกติของ Django)
>>> product.full_clean()   # ต้องเรียกเองถึงจะ validate
django.core.exceptions.ValidationError: {
    'name': ['ชื่อสินค้าต้องมีอย่างน้อย 3 ตัวอักษร'],
    'sku': ['รหัสสินค้า (SKU) ต้องอยู่ในรูปแบบ AAA-0000 เช่น TSH-0001'],
    'price': ['ราคาต้องมากกว่า 0 บาท'],
}
```

### 261.4 ใช้ Validator เดียวกันซ้ำใน Form Field

จุดแข็งที่แท้จริงของ validator คือการนำกลับมาใช้ซ้ำ — สมมติมีฟอร์มค้นหาสินค้าด้วย
SKU ที่ไม่ได้ผูกกับ Model โดยตรง (`forms.Form` ธรรมดา) ก็ยังใช้ validator ตัวเดียว
กับที่ประกาศไว้ใน Model ได้ทันที โดยไม่ต้องเขียนกฎซ้ำ:

```python
# products/forms.py
from django import forms
from django.core.validators import RegexValidator, MinValueValidator, MaxValueValidator

SKU_VALIDATOR = RegexValidator(
    regex=r"^[A-Z]{3}-\d{4}$",
    message="รหัสสินค้า (SKU) ต้องอยู่ในรูปแบบ AAA-0000 เช่น TSH-0001",
)


class ProductSearchForm(forms.Form):
    sku = forms.CharField(
        max_length=20,
        required=False,
        validators=[SKU_VALIDATOR],   # validator ตัวเดียวกับที่ใช้ใน Model
        label="ค้นหาด้วยรหัสสินค้า",
    )
    min_price = forms.DecimalField(
        required=False,
        validators=[MinValueValidator(0)],
        label="ราคาต่ำสุด",
    )
    max_discount = forms.IntegerField(
        required=False,
        validators=[MaxValueValidator(90)],
        label="ส่วนลดสูงสุด (%)",
    )
```

**คำแนะนำระดับมืออาชีพ**: แยกตัวแปร validator ที่ใช้ร่วมกันบ่อย ๆ ไว้ในโมดูล
กลาง เช่น `products/validators.py` แล้ว import ไปใช้ทั้งใน `models.py` และ
`forms.py` เพื่อไม่ให้ต้องแก้ regex หรือ error message ซ้ำ 2 ที่เวลาข้อกำหนด
ทางธุรกิจเปลี่ยน

```python
# products/validators.py
from django.core.validators import RegexValidator

sku_validator = RegexValidator(
    regex=r"^[A-Z]{3}-\d{4}$",
    message="รหัสสินค้า (SKU) ต้องอยู่ในรูปแบบ AAA-0000 เช่น TSH-0001",
)
```

```python
# products/models.py
from .validators import sku_validator

class Product(models.Model):
    sku = models.CharField(max_length=20, validators=[sku_validator])
```

```python
# products/forms.py
from .validators import sku_validator

class ProductSearchForm(forms.Form):
    sku = forms.CharField(max_length=20, required=False, validators=[sku_validator])
```

### 261.5 ผสม Validator หลายตัวใน Field เดียว

`validators` รับเป็น `list` เสมอ ดังนั้นใส่ได้หลายตัวพร้อมกัน โดย Django จะรัน
ทุกตัวและ**เก็บ error จากทุก validator ที่ไม่ผ่าน** ไม่ใช่แค่ตัวแรกที่เจอ:

```python
class ReviewForm(forms.Form):
    rating = forms.IntegerField(
        validators=[
            MinValueValidator(1, message="คะแนนต้องอย่างน้อย 1 ดาว"),
            MaxValueValidator(5, message="คะแนนสูงสุดคือ 5 ดาว"),
        ],
        label="ให้คะแนน (1-5)",
    )
```

```python
>>> form = ReviewForm(data={"rating": 10})
>>> form.is_valid()
False
>>> form.errors
{'rating': ['คะแนนสูงสุดคือ 5 ดาว']}
```

| แนวทาง | ใช้เมื่อไร | ข้อดี |
|---|---|---|
| `validators=[...]` (validator function) | กฎที่ตรวจสอบ field เดียวโดยลำพัง และอยากใช้ซ้ำได้ในหลาย field/หลายฟอร์ม | นำกลับมาใช้ซ้ำง่าย, ทดสอบแยกหน่วยได้, ใช้ร่วมกับ Model ได้ |
| `clean_<fieldname>()` (จาก Part 025) | กฎเฉพาะฟอร์มนี้ฟอร์มเดียว หรือ logic ซับซ้อนที่ต้องเข้าถึง `self` | เขียนง่าย เข้าถึง instance ของฟอร์มได้เต็มที่ |

---

## ขั้นตอนที่ 262: เขียน Custom Validator Function เอง

### 262.1 โครงสร้างพื้นฐานของ Custom Validator

Validator function ต้องมีคุณสมบัติ 2 อย่าง: **(1)** รับ argument เดียว (ค่าที่
จะตรวจสอบ) และ **(2)** `raise ValidationError` เมื่อค่าไม่ผ่านเงื่อนไข (ไม่ต้อง
`return` อะไรเมื่อผ่าน):

```python
from django.core.exceptions import ValidationError


def validate_เงื่อนไข(value):
    if <ไม่ผ่านเงื่อนไข>:
        raise ValidationError(
            "ข้อความ error",
            code="รหัส error (optional แต่แนะนำให้ใส่)",
            params={"value": value},   # (optional) ใช้แทรกใน %(value)s ของข้อความ
        )
```

### 262.2 ตัวอย่างที่ 1: Validator ตรวจคำหยาบ/คำต้องห้าม

```python
# blog/validators.py
from django.core.exceptions import ValidationError
from django.utils.deconstruct import deconstructible

BANNED_WORDS = ["สแปม", "โฆษณาแฝง", "คลิกที่นี่ด่วน", "การพนัน"]


@deconstructible
class BannedWordsValidator:
    """
    Validator แบบ class-based — ใช้ @deconstructible เพื่อให้ Django สามารถ
    serialize validator นี้ลงไฟล์ migration ได้อย่างถูกต้อง (จำเป็นเมื่อ
    validator ถูกใช้ใน Model field ที่ต้องผ่านระบบ migration)
    """

    def __init__(self, banned_words=None):
        self.banned_words = banned_words or BANNED_WORDS

    def __call__(self, value):
        lowered = value.lower()
        found = [word for word in self.banned_words if word in lowered]
        if found:
            raise ValidationError(
                "ข้อความมีคำที่ไม่อนุญาต: %(words)s",
                code="banned_words",
                params={"words": ", ".join(found)},
            )

    def __eq__(self, other):
        # จำเป็นสำหรับ @deconstructible เพื่อให้ Django เทียบ validator
        # สองตัวว่า "เหมือนกันไหม" ตอนสร้างไฟล์ migration ใหม่
        return (
            isinstance(other, BannedWordsValidator)
            and self.banned_words == other.banned_words
        )


validate_no_banned_words = BannedWordsValidator()
```

**ทำไมต้อง `@deconstructible` และ `__eq__`?** เมื่อ validator ถูกใช้ใน Model
field (`validators=[validate_no_banned_words]`) Django ต้องเขียนโค้ด Python
ที่สร้าง validator ตัวนี้ขึ้นมาใหม่ลงในไฟล์ migration (เพื่อให้ migration ไฟล์นั้น
รันได้อิสระโดยไม่ต้อง import จากแอปจริง) `@deconstructible` บอก Django ว่าจะ
"แยกส่วนประกอบ" object นี้กลับเป็น constructor call ได้อย่างไร ส่วน `__eq__`
ใช้ตอนรัน `makemigrations` เพื่อเช็คว่า validator เปลี่ยนไปจากเดิมหรือไม่
(ถ้าไม่มี `__eq__` Django อาจสร้าง migration ใหม่ที่ไม่จำเป็นทุกครั้งที่รันคำสั่ง)

### 262.3 ตัวอย่างที่ 2: Validator ตรวจเบอร์โทรศัพท์ไทย

เบอร์โทรไทยมีกฎเฉพาะที่ `RegexValidator` ตัวเดียวไม่พอ (ต้องรองรับทั้งเบอร์บ้าน
เบอร์มือถือ และเบอร์ที่มี/ไม่มีขีดคั่น) จึงเหมาะกับการเขียนเป็น function แยก:

```python
# accounts/validators.py
import re
from django.core.exceptions import ValidationError


def validate_thai_phone_number(value):
    """
    ตรวจสอบเบอร์โทรศัพท์ไทย รองรับรูปแบบ:
    - 0812345678 (มือถือ 10 หลัก ขึ้นต้นด้วย 06, 08, 09)
    - 02-123-4567 (เบอร์บ้านกรุงเทพฯ มีขีดคั่น)
    - 021234567 (เบอร์บ้านกรุงเทพฯ ไม่มีขีดคั่น)
    - 053-123456 (เบอร์บ้านต่างจังหวัด)
    """
    cleaned = re.sub(r"[\s\-]", "", value)   # ตัดช่องว่างและขีดคั่นออกก่อนตรวจ

    if not cleaned.isdigit():
        raise ValidationError(
            "เบอร์โทรศัพท์ต้องเป็นตัวเลขเท่านั้น (สามารถมีขีด - คั่นได้)",
            code="invalid_characters",
        )

    mobile_pattern = re.compile(r"^0[689]\d{8}$")       # มือถือ: 0 + 6/8/9 + 8 หลัก = 10 หลัก
    bangkok_pattern = re.compile(r"^02\d{7}$")            # บ้านกรุงเทพฯ: 02 + 7 หลัก = 9 หลัก
    provincial_pattern = re.compile(r"^0[3-7]\d{7}$")     # บ้านต่างจังหวัด: 0 + 3-7 + 7 หลัก = 9 หลัก

    if not (
        mobile_pattern.match(cleaned)
        or bangkok_pattern.match(cleaned)
        or provincial_pattern.match(cleaned)
    ):
        raise ValidationError(
            "รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง กรุณากรอกเบอร์มือถือ (10 หลัก) "
            "หรือเบอร์บ้าน (9 หลัก) ที่ถูกต้อง",
            code="invalid_format",
        )
```

ใช้งานใน Model และ Form พร้อมกัน:

```python
# accounts/models.py
from django.db import models
from .validators import validate_thai_phone_number


class Customer(models.Model):
    name = models.CharField(max_length=100)
    phone_number = models.CharField(
        max_length=20,
        validators=[validate_thai_phone_number],
    )
```

```python
# accounts/forms.py
from django import forms
from .validators import validate_thai_phone_number


class CustomerContactForm(forms.Form):
    phone_number = forms.CharField(
        max_length=20,
        validators=[validate_thai_phone_number],
        label="เบอร์โทรศัพท์ติดต่อกลับ",
        widget=forms.TextInput(attrs={"placeholder": "08X-XXX-XXXX"}),
    )
```

ทดสอบใน shell:

```python
>>> from accounts.forms import CustomerContactForm
>>> CustomerContactForm(data={"phone_number": "081-234-5678"}).is_valid()
True
>>> bad = CustomerContactForm(data={"phone_number": "123"})
>>> bad.is_valid()
False
>>> bad.errors
{'phone_number': ['รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง กรุณากรอกเบอร์มือถือ (10 หลัก) หรือเบอร์บ้าน (9 หลัก) ที่ถูกต้อง']}
```

### 262.4 การเขียน Unit Test ให้กับ Validator

Validator ที่แยกเป็น function อิสระทดสอบได้ง่ายมากโดยไม่ต้องยุ่งกับ Model หรือ
Form เลย — นี่คือข้อดีสำคัญของการแยก validator ออกมาต่างหาก:

```python
# accounts/tests/test_validators.py
from django.core.exceptions import ValidationError
from django.test import SimpleTestCase
from accounts.validators import validate_thai_phone_number


class ThaiPhoneNumberValidatorTests(SimpleTestCase):
    def test_valid_mobile_number(self):
        validate_thai_phone_number("0812345678")   # ไม่ raise = ผ่าน

    def test_valid_mobile_with_dashes(self):
        validate_thai_phone_number("081-234-5678")

    def test_valid_bangkok_landline(self):
        validate_thai_phone_number("02-123-4567")

    def test_invalid_too_short(self):
        with self.assertRaises(ValidationError):
            validate_thai_phone_number("123")

    def test_invalid_contains_letters(self):
        with self.assertRaises(ValidationError):
            validate_thai_phone_number("08XXXXXXXX")
```

รันด้วย `python manage.py test accounts.tests.test_validators` — ใช้
`SimpleTestCase` แทน `TestCase` เพราะ validator ไม่แตะฐานข้อมูลเลย ทำให้เทสต์
รันเร็วกว่ามาก (เรื่อง Testing framework เต็มรูปแบบจะเจาะลึกใน Phase 9)

---

## ขั้นตอนที่ 263: เขียน Custom Widget โดย Subclass Widget ที่มีอยู่

### 263.1 ทบทวน: Widget ทุกตัวสืบทอดจาก `django.forms.Widget`

ใน Part 025 คุณใช้ widget สำเร็จรูป (`TextInput`, `Textarea`, `RadioSelect` ฯลฯ)
มาแล้ว ทุก widget เหล่านี้เป็น subclass ของ `django.forms.widgets.Widget`
(หรือ `django.forms.widgets.Input` ซึ่งเป็น subclass ของ `Widget` อีกที) และมี
method หลักที่กำหนดพฤติกรรมการ render คือ `render(self, name, value, attrs=None,
renderer=None)` ซึ่งคืนค่าเป็น HTML string (หรือ `SafeString`)

การเขียน custom widget ในระดับเริ่มต้นที่สุดคือการ **subclass widget ที่ใกล้เคียง
ที่สุด** แล้ว override เท่าที่จำเป็น แทนที่จะเขียน `Widget` จากศูนย์ทั้งหมด

### 263.2 ตัวอย่าง: `RatingWidget` — Render เป็นดาว 1-5 ที่คลิกได้

เราจะสร้าง widget ที่แสดงดาว 5 ดวงให้ผู้ใช้คลิกให้คะแนน โดย subclass จาก
`forms.RadioSelect` (เพราะโดยพื้นฐานแล้วมันคือการเลือก 1 ค่าจาก 5 ตัวเลือก
เหมือน radio button ทุกประการ เพียงแค่หน้าตาต่างออกไป):

```python
# reviews/widgets.py
from django import forms
from django.utils.safestring import mark_safe


class RatingWidget(forms.RadioSelect):
    """
    Widget สำหรับให้คะแนนแบบดาว 1-5 ดวง
    ใช้กลไกเดิมของ RadioSelect (radio input ที่มองไม่เห็น) แต่เปลี่ยน label
    ให้กลายเป็นไอคอนดาว ควบคุมด้วย CSS (::before / sibling selector)
    """

    template_name = "reviews/widgets/rating_widget.html"
    option_template_name = "reviews/widgets/rating_option.html"

    def __init__(self, attrs=None, max_stars=5):
        choices = [(str(i), str(i)) for i in range(1, max_stars + 1)]
        default_attrs = {"class": "rating-widget"}
        if attrs:
            default_attrs.update(attrs)
        super().__init__(attrs=default_attrs, choices=choices)
```

Django ใช้ **Widget Rendering API** ที่แยก template ของทั้ง widget
(`template_name`) ออกจาก template ของแต่ละ option (`option_template_name`)
ทำให้เขียน HTML ที่ต้องการได้ตรงเป้าโดยไม่ต้อง override `render()` เอง — นี่คือ
วิธีที่แนะนำตั้งแต่ Django 1.11 เป็นต้นมา (แทนการต่อ string HTML เองแบบเดิม):

```html
<!-- reviews/templates/reviews/widgets/rating_widget.html -->
<div class="rating-widget-wrapper">
    {% for group, options, index in widget.optgroups %}
        {% for option in options %}
            {% include option.template_name with widget=option %}
        {% endfor %}
    {% endfor %}
</div>
```

```html
<!-- reviews/templates/reviews/widgets/rating_option.html -->
<input type="{{ widget.type }}"
       name="{{ widget.name }}"
       value="{{ widget.value }}"
       id="{{ widget.attrs.id }}"
       class="rating-widget-input"
       {% if widget.attrs.checked %}checked{% endif %}
       {% if widget.attrs.required %}required{% endif %}>
<label for="{{ widget.attrs.id }}" class="rating-widget-star" title="{{ widget.value }} ดาว">★</label>
```

CSS ที่ทำให้ดาวคลิกได้และเปลี่ยนสีเมื่อเลือก (ใช้เทคนิค "checked sibling
selector" ที่ไม่ต้องพึ่ง JavaScript เลย):

```css
/* static/css/rating-widget.css */
.rating-widget-wrapper {
    display: flex;
    flex-direction: row-reverse;   /* กลับลำดับเพื่อให้ CSS ~ selector ทำงานแบบ "เลือกดาวนี้และก่อนหน้า" ได้ */
    justify-content: flex-end;
    gap: 4px;
}
.rating-widget-input {
    display: none;   /* ซ่อน radio input จริง ใช้ label แทน */
}
.rating-widget-star {
    font-size: 28px;
    color: #d0d0d0;
    cursor: pointer;
}
.rating-widget-input:checked ~ .rating-widget-star,
.rating-widget-input:checked ~ .rating-widget-star ~ .rating-widget-star {
    color: #f5b301;
}
.rating-widget-wrapper:hover .rating-widget-star:hover,
.rating-widget-wrapper:hover .rating-widget-star:hover ~ .rating-widget-star {
    color: #f5c93a;
}
```

### 263.3 ใช้งาน `RatingWidget` ใน Form จริง

```python
# reviews/forms.py
from django import forms
from .widgets import RatingWidget


class ReviewForm(forms.Form):
    RATING_CHOICES = [(str(i), str(i)) for i in range(1, 6)]

    rating = forms.ChoiceField(
        choices=RATING_CHOICES,
        widget=RatingWidget,
        label="ให้คะแนนสินค้านี้",
    )
    comment = forms.CharField(
        widget=forms.Textarea(attrs={"rows": 4}),
        label="ความคิดเห็นเพิ่มเติม",
        required=False,
    )
```

สังเกตว่า `cleaned_data["rating"]` ยังคงได้ค่าเป็น `str` เหมือน `ChoiceField`
ปกติทุกประการ (`"1"` ถึง `"5"`) เพราะเราแค่เปลี่ยน **หน้าตา (widget)** ไม่ได้
แตะ **ตรรกะ (field)** เลย — นี่คือตัวอย่างที่ชัดเจนที่สุดของหลักการแยก Field/Widget
ที่จะเจาะลึกเต็มรูปแบบในขั้นตอนที่ 267

| ส่วนประกอบ | เปลี่ยนหรือไม่ | เหตุผล |
|---|---|---|
| `ChoiceField` (field logic) | ไม่เปลี่ยน | ยังคง validate ว่าค่าต้องอยู่ใน `choices` เหมือนเดิม |
| `RadioSelect` → `RatingWidget` (widget) | เปลี่ยน | เปลี่ยนแค่วิธี render เป็น HTML |
| `cleaned_data` ที่ได้ | ไม่เปลี่ยน | ยังคงเป็น `str` ค่าเดียวจาก choices เหมือน `ChoiceField` ทุกตัว |

---

## ขั้นตอนที่ 264: Widget `Media` Class — แนบ CSS/JS ให้ Widget เอง

### 264.1 ปัญหา: Widget ที่ต้องพึ่ง JavaScript Library ภายนอก

widget บางตัวไม่สามารถทำงานได้ด้วย HTML/CSS ล้วน ๆ เช่น **Date Picker** ที่
สวยงามกว่า `<input type="date">` มาตรฐานของเบราว์เซอร์ (ซึ่งหน้าตาต่างกันในแต่ละ
เบราว์เซอร์/OS) จำเป็นต้องพึ่ง JS library เช่น Flatpickr

ปัญหาคือ: **ถ้า widget ต้องการ CSS/JS ของตัวเอง แล้วนักพัฒนาที่เอา widget นี้ไป
ใช้ในฟอร์มลืม `<link>`/`<script>` ที่ถูกต้องใน template ล่ะ?** Django แก้ปัญหานี้
ด้วย **Widget `Media` class** ที่ให้ widget "ประกาศ" ว่าตัวเองต้องการไฟล์อะไรบ้าง
แล้วให้ Django รวบรวมไฟล์เหล่านั้นจากทุก widget ในฟอร์มให้อัตโนมัติ (พร้อมตัด
ไฟล์ที่ซ้ำกันออกโดยอัตโนมัติด้วย ถ้าหลาย field ใช้ widget เดียวกัน)

### 264.2 ประกาศ `Media` ผ่าน Inner Class

```python
# events/widgets.py
from django import forms


class FlatpickrDateWidget(forms.DateInput):
    """
    Date picker ที่ใช้ Flatpickr (https://flatpickr.js.org/) แทน
    input type="date" มาตรฐานของเบราว์เซอร์
    """

    template_name = "events/widgets/flatpickr_date.html"

    def __init__(self, attrs=None):
        default_attrs = {"class": "flatpickr-input", "autocomplete": "off"}
        if attrs:
            default_attrs.update(attrs)
        super().__init__(attrs=default_attrs, format="%Y-%m-%d")

    class Media:
        css = {
            "all": (
                "https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css",
                "css/flatpickr-custom.css",   # ไฟล์ static ของโปรเจกต์เราเอง — ปรับแต่งเพิ่มเติม
            )
        }
        js = (
            "https://cdn.jsdelivr.net/npm/flatpickr",
            "js/flatpickr-init.js",   # สคริปต์ที่เรียก flatpickr() ให้กับทุก .flatpickr-input
        )
```

```html
<!-- events/templates/events/widgets/flatpickr_date.html -->
<input type="text"
       name="{{ widget.name }}"
       {% if widget.value != None %}value="{{ widget.value }}"{% endif %}
       {% include "django/forms/widgets/attrs.html" %}>
```

```javascript
// static/js/flatpickr-init.js
document.addEventListener("DOMContentLoaded", function () {
    document.querySelectorAll(".flatpickr-input").forEach(function (el) {
        flatpickr(el, {
            dateFormat: "Y-m-d",
            altInput: true,
            altFormat: "j F Y",   // แสดงผลแบบอ่านง่าย แต่ส่งค่าจริงเป็น Y-m-d ไปให้ Django
            locale: { firstDayOfWeek: 1 },
        });
    });
});
```

### 264.3 กฎของ `css` Dict — ทำไมต้องเป็น Dict ไม่ใช่ List

สังเกตว่า `css` เป็น **dict** (ไม่ใช่ list/tuple แบบ `js`) เพราะ Django ให้ระบุ
**media type ของ CSS** ตามมาตรฐาน HTML (`all`, `screen`, `print` ฯลฯ) เพื่อ
รองรับกรณีที่ต้องการโหลด stylesheet ต่างกันสำหรับหน้าจอกับตอนพิมพ์:

```python
class Media:
    css = {
        "all": ("css/base-widget.css",),
        "print": ("css/print-widget.css",),
    }
```

### 264.4 การ Render `Media` ในเทมเพลต: `{{ form.media }}`

จุดที่ทรงพลังที่สุดของระบบนี้คือ Django **รวบรวม Media จากทุก field ในฟอร์ม
อัตโนมัติ** ผ่าน `form.media` — ไม่ว่าฟอร์มจะมีกี่ field ที่ใช้ `Media` (ซ้ำกันไหม
ก็ตาม) คุณเรียกครั้งเดียวในเทมเพลตพอ:

```python
# events/forms.py
from django import forms
from .widgets import FlatpickrDateWidget


class EventForm(forms.Form):
    title = forms.CharField(max_length=200, label="ชื่องาน")
    start_date = forms.DateField(widget=FlatpickrDateWidget, label="วันที่เริ่มงาน")
    end_date = forms.DateField(widget=FlatpickrDateWidget, label="วันที่สิ้นสุด")
```

```html
<!-- events/templates/events/event_form.html -->
{% extends 'base.html' %}

{% block extra_head %}
    {{ form.media }}   {# render <link> + <script> ของ FlatpickrDateWidget ให้อัตโนมัติ #}
{% endblock %}

{% block content %}
<form method="post">
    {% csrf_token %}
    {{ form.as_div }}
    <button type="submit">บันทึกกิจกรรม</button>
</form>
{% endblock %}
```

แม้ `EventForm` จะมี field ที่ใช้ `FlatpickrDateWidget` ถึง 2 ตัว (`start_date`,
`end_date`) แต่ `{{ form.media }}` จะ render `<link>`/`<script>` ของ Flatpickr
**เพียงครั้งเดียว** ไม่ซ้ำ — Django จัดการ deduplication ให้อัตโนมัติโดยสมบูรณ์

### 264.5 `Media` แบบกำหนดเองด้วย Property (Dynamic Media)

บางกรณีไฟล์ที่ต้องโหลดขึ้นอยู่กับเงื่อนไข runtime (เช่น locale ของผู้ใช้) แทนที่
จะใช้ inner class ธรรมดา สามารถ override `media` เป็น property ได้:

```python
from django.forms.widgets import Media


class LocaleAwareDateWidget(forms.DateInput):
    def __init__(self, attrs=None, locale="th"):
        self.locale = locale
        super().__init__(attrs=attrs)

    @property
    def media(self):
        return Media(
            css={"all": ("css/flatpickr-custom.css",)},
            js=(
                "https://cdn.jsdelivr.net/npm/flatpickr",
                f"https://cdn.jsdelivr.net/npm/flatpickr/dist/l10n/{self.locale}.js",
            ),
        )
```

| แนวทางประกาศ `Media` | ใช้เมื่อไร |
|---|---|
| `class Media:` (inner class คงที่) | ไฟล์ CSS/JS เดิมเสมอ ไม่เปลี่ยนตามเงื่อนไข (กรณีส่วนใหญ่) |
| `@property def media(self):` | ไฟล์ต้องเปลี่ยนตาม instance attribute เช่น locale, theme |

---

## ขั้นตอนที่ 265: จัดสไตล์ฟอร์มด้วย django-widget-tweaks และ django-crispy-forms

### 265.1 ปัญหาที่ทั้งสอง Package แก้: HTML ที่ Django Generate มาให้ "ดิบเกินไป"

ฟอร์มที่ render ด้วย `{{ form.as_div }}` หรือ `{{ field }}` ตรง ๆ จะไม่มี CSS
class ของ framework อย่าง Bootstrap หรือ Tailwind ติดมาให้เลย (เช่น
`class="form-control"`) ทางเลือกที่คุณเรียนใน Part 025 คือ ใส่ `attrs={"class":
"form-control"}` เองทุก field ใน `forms.py` — ใช้ได้ผลแต่ **ผูก HTML/CSS
framework ไว้ในโค้ด Python** ซึ่งขัดกับหลักการแยกส่วนที่ดี (separation of
concerns) สอง package นี้แก้ปัญหาด้วยแนวทางที่ต่างกันโดยสิ้นเชิง

### 265.2 แนวทางที่ 1: `django-widget-tweaks` — เติม Attribute ในเทมเพลต

**ปรัชญา**: ให้ควบคุม HTML attribute ของแต่ละ field **จากฝั่งเทมเพลต** โดยไม่
ต้องแตะ `forms.py` เลย เหมาะกับทีมที่อยากให้ designer/frontend แก้ style เองได้
โดยไม่ต้องรอ backend developer

```bash
pip install django-widget-tweaks
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "widget_tweaks",
]
```

```html
<!-- ใช้ template tag {% render_field %} หรือ filter |add_class -->
{% load widget_tweaks %}

<form method="post">
    {% csrf_token %}

    <div class="form-group">
        {{ form.name.label_tag }}
        {% render_field form.name class="form-control" placeholder="กรอกชื่อ" %}
        {% if form.name.errors %}
            <div class="invalid-feedback d-block">{{ form.name.errors }}</div>
        {% endif %}
    </div>

    <div class="form-group">
        {{ form.email.label_tag }}
        {{ form.email|add_class:"form-control"|attr:"placeholder:you@example.com" }}
    </div>

    <button type="submit" class="btn btn-primary">ส่ง</button>
</form>
```

| Filter/Tag ของ widget-tweaks | หน้าที่ |
|---|---|
| `{{ field\|add_class:"form-control" }}` | เติม CSS class เข้าไปใน widget |
| `{{ field\|attr:"placeholder:กรอกชื่อ" }}` | เติม HTML attribute ใด ๆ |
| `{{ field\|append_attr:"class:extra-class" }}` | เติม attribute แบบต่อท้ายค่าที่มีอยู่แล้ว (ไม่ทับของเดิม) |
| `{% render_field form.name class="form-control" %}` | เทียบเท่า filter แต่เขียนแบบ tag รับ keyword argument ได้หลายตัว |

**ข้อดี**: ควบคุมได้ละเอียดระดับ field เดียว ไม่ต้องเรียนรู้ syntax ใหม่มาก
(ยังคงเขียนเทมเพลตแบบ field-by-field เหมือน Part 025) เหมาะกับฟอร์มที่ต้องการ
custom layout เฉพาะจุด

**ข้อเสีย**: ต้องเขียนซ้ำทุก field ทุกฟอร์ม (boilerplate เยอะถ้ามีหลายฟอร์ม
ที่ต้องการหน้าตาเหมือนกัน)

### 265.3 แนวทางที่ 2: `django-crispy-forms` — Render ทั้งฟอร์มด้วย Layout Object

**ปรัชญา**: กำหนด **Layout** ของฟอร์มทั้งก้อนแบบ declarative ใน Python
(ผ่าน `FormHelper`) แล้ว render ทั้งฟอร์มด้วย tag เดียว เหมาะกับทีมที่ต้องการ
ความสม่ำเสมอสูง (ฟอร์มทุกอันในระบบหน้าตาเหมือนกันหมดโดยอัตโนมัติ)

```bash
pip install django-crispy-forms crispy-bootstrap5
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "crispy_forms",
    "crispy_bootstrap5",
]

CRISPY_ALLOWED_TEMPLATE_PACKS = "bootstrap5"
CRISPY_TEMPLATE_PACK = "bootstrap5"
```

```python
# blog/forms.py
from django import forms
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Layout, Row, Column, Submit, Field, Fieldset, HTML


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, label="ชื่อของคุณ")
    email = forms.EmailField(label="อีเมล")
    subject = forms.CharField(max_length=150, label="หัวข้อ")
    message = forms.CharField(widget=forms.Textarea, label="ข้อความ")
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
                "subject",
                "message",
            ),
            HTML("<hr>"),
            "newsletter_opt_in",
            Submit("submit", "ส่งข้อความ", css_class="btn btn-primary"),
        )
```

```html
<!-- blog/templates/blog/contact.html -->
{% load crispy_forms_tags %}

<h1>ติดต่อเรา</h1>
{% crispy form %}   {# render ฟอร์มทั้งหมด รวม <form> tag, CSRF, layout, ปุ่ม submit ในบรรทัดเดียว #}
```

`{% crispy form %}` จะ render HTML ที่มี `<form>` tag, `{% csrf_token %}`, และ
โครงสร้าง Bootstrap 5 (grid columns, `form-control`, `invalid-feedback` เมื่อมี
error) ให้ครบถ้วนอัตโนมัติ **โดยไม่ต้องเขียน HTML เทมเพลตของฟอร์มเองเลย**

### 265.4 ตารางเปรียบเทียบสองแนวทาง

| ประเด็น | django-widget-tweaks | django-crispy-forms |
|---|---|---|
| ปรัชญา | เติม attribute ให้ field ทีละตัวในเทมเพลต | กำหนด Layout ทั้งฟอร์มใน Python แล้ว render ด้วย tag เดียว |
| ต้องเขียนเทมเพลตของฟอร์มเองไหม | ต้อง (field-by-field) | ไม่ต้อง (ถ้าใช้ `{% crispy form %}` เต็มรูปแบบ) |
| ความยืดหยุ่นด้าน HTML/Layout | สูงมาก (คุมทุกจุดในเทมเพลต) | สูง แต่ผ่าน Layout API ของ crispy เอง (เรียนรู้ syntax เพิ่ม) |
| Boilerplate เมื่อมีหลายฟอร์ม | เยอะ (ต้องเขียนซ้ำทุกฟอร์ม) | น้อย (`{% crispy form %}` ใช้ซ้ำได้ทันที) |
| เหมาะกับทีมที่มี Designer แยก | ดีมาก (Designer แก้เทมเพลตได้อิสระ) | ปานกลาง (ต้องแก้ Layout ใน Python) |
| รองรับ CSS Framework | ไม่ผูกกับ framework ใด (ใส่ class อะไรก็ได้เอง) | มี "template pack" สำเร็จรูปสำหรับ Bootstrap, Tailwind, Bulma ฯลฯ |
| Learning curve | ต่ำมาก (เป็นแค่ template filter) | ปานกลาง (ต้องเรียน Layout object: `Row`, `Column`, `Fieldset` ฯลฯ) |
| เหมาะกับโปรเจกต์ | ฟอร์มจำนวนน้อย ต้องการคุม UI ละเอียดเฉพาะจุด | ระบบที่มีฟอร์มจำนวนมากและต้องการความสม่ำเสมอของ UI ทั้งระบบ |

**คำแนะนำของหลักสูตรนี้**: เริ่มด้วย `django-widget-tweaks` เมื่อโปรเจกต์มีฟอร์ม
ไม่กี่แบบ แล้วพิจารณาย้ายไป `django-crispy-forms` เมื่อจำนวนฟอร์มในระบบเริ่มมาก
(10+ ฟอร์ม) และต้องการบังคับความสม่ำเสมอของ UI ทั้งระบบโดยไม่ต้องคอย copy-paste
โครงสร้าง HTML เดิมซ้ำ ๆ ทั้งสอง package **ใช้ร่วมกันในโปรเจกต์เดียวกันได้**
(เช่น ใช้ crispy สำหรับฟอร์มหลัก และ widget-tweaks สำหรับฟอร์มเล็ก ๆ ที่ฝังใน
modal หรือ partial template)

---

## ขั้นตอนที่ 266: Customize `ClearableFileInput` — แสดง Preview รูปภาพ

### 266.1 ปัญหาของ `ClearableFileInput` เริ่มต้น

`ClearableFileInput` (widget เริ่มต้นของ `ImageField`/`FileField` ที่เรียนใน
Part 025) เมื่อใช้แก้ไข object ที่มีรูปอยู่แล้ว จะแสดงแค่ **ลิงก์ข้อความ** ไปยัง
ไฟล์เดิม (เช่น "Currently: profile_pics/somchai.jpg") ไม่มีการแสดง**รูปตัวอย่าง
(preview)** ให้ผู้ใช้เห็นจริง ๆ ว่ารูปที่กำลังจะถูกแทนที่หน้าตาเป็นอย่างไร ซึ่ง
ไม่เป็นมิตรกับผู้ใช้เอาเสียเลยในฟอร์มแก้ไขโปรไฟล์หรือสินค้า

### 266.2 Subclass `ClearableFileInput` เพื่อแสดง Preview

```python
# accounts/widgets.py
from django import forms


class ImagePreviewWidget(forms.ClearableFileInput):
    """
    ต่อยอดจาก ClearableFileInput เดิม แต่เพิ่ม <img> preview ของไฟล์ปัจจุบัน
    และ preview แบบ real-time ของไฟล์ใหม่ที่ผู้ใช้เพิ่งเลือก (ผ่าน JS เล็กน้อย)
    """

    template_name = "accounts/widgets/image_preview_widget.html"

    def get_context(self, name, value, attrs):
        context = super().get_context(name, value, attrs)
        # ส่งค่า URL ของรูปปัจจุบันเข้าไปใน context ของ template โดยตรง
        # value คือ FieldFile object (หรือ None ถ้ายังไม่เคยมีรูป)
        context["current_image_url"] = value.url if value and hasattr(value, "url") else None
        return context
```

```html
<!-- accounts/templates/accounts/widgets/image_preview_widget.html -->
<div class="image-preview-widget">
    {% if widget.current_image_url %}
        <div class="image-preview-widget__current">
            <img src="{{ widget.current_image_url }}" alt="รูปปัจจุบัน" class="image-preview-widget__img">
            <p class="image-preview-widget__caption">รูปปัจจุบัน</p>
        </div>
    {% endif %}

    <img id="{{ widget.attrs.id }}-preview"
         class="image-preview-widget__img image-preview-widget__new-preview"
         style="display: none;" alt="ตัวอย่างรูปใหม่ที่เลือก">

    {% include "django/forms/widgets/clearable_file_input.html" %}

    <script>
        (function () {
            const input = document.getElementById("{{ widget.attrs.id }}");
            const preview = document.getElementById("{{ widget.attrs.id }}-preview");
            input.addEventListener("change", function (event) {
                const file = event.target.files[0];
                if (!file) {
                    preview.style.display = "none";
                    return;
                }
                const reader = new FileReader();
                reader.onload = function (e) {
                    preview.src = e.target.result;
                    preview.style.display = "block";
                };
                reader.readAsDataURL(file);
            });
        })();
    </script>
</div>
```

สังเกตว่าเรา `{% include "django/forms/widgets/clearable_file_input.html" %}`
เพื่อนำ HTML ของ input จริง (ปุ่มเลือกไฟล์ ลิงก์ "Clear" checkbox) มาใช้ต่อ
แทนที่จะเขียน `<input type="file">` ใหม่เอง — นี่คือวิธี **ต่อยอด** (extend)
template ของ built-in widget แทนการเขียนซ้ำทั้งหมด ซึ่งปลอดภัยกว่าเพราะยัง
ได้พฤติกรรมเดิมของ Django (เช่น checkbox "Clear" ที่ล้างรูปเดิมได้) มาโดยไม่ต้อง
เขียนเอง

### 266.3 ใช้งานใน `ModelForm`

```python
# accounts/forms.py
from django import forms
from .models import Profile
from .widgets import ImagePreviewWidget


class ProfileForm(forms.ModelForm):
    class Meta:
        model = Profile
        fields = ["display_name", "bio", "avatar"]
        widgets = {
            "avatar": ImagePreviewWidget(),
        }
```

```css
/* static/css/image-preview-widget.css */
.image-preview-widget__img {
    max-width: 200px;
    max-height: 200px;
    border-radius: 8px;
    display: block;
    margin-bottom: 8px;
    object-fit: cover;
}
.image-preview-widget__caption {
    font-size: 0.85rem;
    color: #666;
    margin-bottom: 12px;
}
```

### 266.4 แนบ CSS ผ่าน `Media` (เชื่อมกับขั้นตอนที่ 264)

```python
class ImagePreviewWidget(forms.ClearableFileInput):
    template_name = "accounts/widgets/image_preview_widget.html"

    class Media:
        css = {"all": ("css/image-preview-widget.css",)}

    def get_context(self, name, value, attrs):
        context = super().get_context(name, value, attrs)
        context["current_image_url"] = value.url if value and hasattr(value, "url") else None
        return context
```

เมื่อผสาน `Media` เข้ากับ widget แล้ว ในเทมเพลตของฟอร์มก็แค่เรียก
`{{ form.media }}` เหมือนเดิมตามที่เรียนในขั้นตอนที่ 264 โดยไม่ต้องจำ import
ไฟล์ CSS นี้ด้วยตัวเองในทุกหน้าที่ใช้ `ProfileForm`

| ขั้นตอนของการปรับแต่ง `ClearableFileInput` | สิ่งที่ทำ |
|---|---|
| 1. Subclass `ClearableFileInput` | สร้าง widget class ใหม่ |
| 2. Override `get_context()` | เติมข้อมูล URL ของรูปปัจจุบันเข้า context |
| 3. เขียน template ใหม่ แต่ `{% include %}` ของเดิม | ได้ preview ใหม่ + พฤติกรรมเดิม (checkbox Clear) ครบ |
| 4. แนบ `Media` (CSS/JS) | ผู้ใช้ widget ไม่ต้องจำ import ไฟล์เอง |
| 5. ระบุใน `Meta.widgets` ของ `ModelForm` | เปิดใช้งานจริงโดยไม่แตะ field logic |

---

## ขั้นตอนที่ 267: ความแตกต่างระหว่าง Field กับ Widget แบบเจาะลึก

### 267.1 ทบทวนสั้น ๆ จาก Part 025 ก่อนเจาะลึก

Part 025 บอกไว้แล้วว่า "Field รับผิดชอบข้อมูล, Widget รับผิดชอบหน้าตา HTML"
ขั้นตอนนี้จะพาไปดู **โค้ดจริงภายใน Django** เพื่อให้เห็นภาพว่าการแยกสองเลเยอร์
นี้ถูก implement อย่างไร และทำไม Django ถึงออกแบบมาแบบนี้

### 267.2 หน้าที่ของ Field: `clean()`, `to_python()`, `validate()`, `run_validators()`

`django.forms.Field` (parent class ของ `CharField`, `IntegerField` ทุกตัว)
มี pipeline การประมวลผลค่าที่ชัดเจนเป็นขั้นตอน ซึ่งทั้งหมดนี้**ไม่เกี่ยวข้องกับ
HTML เลยแม้แต่น้อย**:

```python
# ภายใน django/forms/fields.py (แบบย่อเพื่อการอธิบาย)
class Field:
    def clean(self, value):
        """
        method หลักที่ถูกเรียกตอน is_valid() — เป็นตัวควบคุม pipeline ทั้งหมด
        """
        value = self.to_python(value)      # แปลง string ดิบ → Python type (str -> int, str -> date ฯลฯ)
        self.validate(value)               # ตรวจสอบเงื่อนไขพื้นฐาน (required ฯลฯ)
        self.run_validators(value)         # รัน validators=[...] ทั้งหมดที่ประกาศไว้ (ขั้นตอนที่ 261)
        return value
```

- **`to_python(value)`**: รับ **string ดิบเสมอ** (เพราะ HTTP ส่งทุกอย่างเป็น
  text) แล้วแปลงเป็น Python type ที่ต้องการ เช่น `IntegerField.to_python("42")`
  คืนค่า `42` (int) ส่วน `DateField.to_python("2026-01-15")` คืนค่า
  `datetime.date(2026, 1, 15)`
- **`validate(value)`**: ตรวจเงื่อนไขพื้นฐานที่ผูกกับชนิด field โดยตรง เช่น
  `required=True` ต้องไม่เป็นค่าว่าง
- **`run_validators(value)`**: วนลูปเรียก validator ทุกตัวใน `self.validators`
  (รวม validator ที่ Django ใส่ให้อัตโนมัติ เช่น `MaxLengthValidator` จาก
  `max_length=100`, กับที่คุณใส่เองในขั้นตอนที่ 261-262)

**สิ่งสำคัญที่สุดที่ต้องเข้าใจ**: **Field ไม่รู้จัก HTML เลย** มันแค่รับ string
เข้ามาแล้วคืน Python object ที่ validate แล้วออกไป มันทำงานได้แม้ไม่มี HTTP
request เกี่ยวข้องเลย เช่น เรียกตรง ๆ ใน shell:

```python
>>> from django import forms
>>> field = forms.IntegerField(min_value=1, max_value=100)
>>> field.clean("42")
42
>>> field.clean("999")
django.core.exceptions.ValidationError: ['Ensure this value is less than or equal to 100.']
```

ไม่มี `<input>` ไม่มี `render()` ไม่มี HTML เกี่ยวข้องแม้แต่นิดเดียวในตัวอย่างนี้

### 267.3 หน้าที่ของ Widget: `render()`, `value_from_datadict()`, `format_value()`

`django.forms.Widget` มีหน้าที่ตรงข้ามกันโดยสิ้นเชิง — **รู้จัก HTML แต่ไม่รู้จัก
ความหมายของข้อมูลเลย**:

```python
# ภายใน django/forms/widgets.py (แบบย่อเพื่อการอธิบาย)
class Widget:
    def render(self, name, value, attrs=None, renderer=None):
        """แปลง value (Python) → HTML string ที่จะแสดงในเบราว์เซอร์"""
        ...

    def value_from_datadict(self, data, files, name):
        """
        ดึงค่าดิบออกจาก QueryDict ของ request.POST (หรือ request.FILES)
        ตามชื่อ field — สำคัญเพราะ widget บางตัว (เช่น SelectMultiple,
        SplitDateTimeWidget) ต้องดึงค่าจากหลาย key พร้อมกัน ไม่ใช่แค่ key เดียว
        """
        return data.get(name)

    def format_value(self, value):
        """แปลง Python value เป็น string ที่จะใส่ใน attribute value= ของ HTML"""
        ...
```

`value_from_datadict()` คือจุดที่แสดงให้เห็นชัดที่สุดว่า **Widget เป็นตัวกลาง
ระหว่าง HTTP data กับ Field** — มันรู้วิธี "แกะ" ค่าจาก `request.POST` ตามรูปแบบ
HTML ที่ตัวเองสร้างขึ้น ก่อนที่ค่านั้นจะถูกส่งต่อไปให้ Field ประมวลผลด้วย
`clean()` ตัวอย่างเช่น `SplitDateTimeWidget` ที่แยก date กับ time เป็นคนละ
`<input>` จะต้อง override `value_from_datadict()` เพื่อรวมค่าจาก 2 key
(`fieldname_0`, `fieldname_1`) กลับเป็นค่าเดียวก่อนส่งให้ Field:

```python
class SplitDateTimeWidget(MultiWidget):
    def decompress(self, value):
        # แยก datetime เดียว → (date, time) สำหรับตอน render 2 input แยกกัน
        if value:
            return [value.date(), value.time()]
        return [None, None]
```

### 267.4 แผนภาพเต็มของ Pipeline: HTTP → Widget → Field → Python

```
                    request.POST["price"] = "1,500.50"   (string ดิบเสมอ)
                              │
                              ▼
              ┌───────────────────────────────┐
              │  Widget.value_from_datadict()  │  ← ดึงค่าดิบจาก POST data ตามชื่อ
              └───────────────┬────────────────┘
                              │ "1,500.50" (ยังเป็น string)
                              ▼
              ┌───────────────────────────────┐
              │      Field.clean(value)        │
              │  ├─ to_python()   → Decimal    │  ← แปลงชนิดข้อมูล
              │  ├─ validate()    → ตรวจ required│
              │  └─ run_validators() → MinValue │  ← รัน validators ทั้งหมด
              └───────────────┬────────────────┘
                              │
                              ▼
                cleaned_data["price"] = Decimal("1500.50")
```

และเมื่อจะแสดงฟอร์มกลับ (เช่น GET request ครั้งแรก หรือหลัง POST ไม่ผ่าน):

```
              form.initial["price"] = Decimal("1500.50")
                              │
                              ▼
              ┌───────────────────────────────┐
              │   Widget.format_value(value)   │  ← แปลง Python value → string สำหรับ value=""
              └───────────────┬────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │      Widget.render(...)        │  ← สร้าง HTML <input value="1500.50">
              └───────────────┬────────────────┘
                              │
                              ▼
                     ส่งไปแสดงในเบราว์เซอร์
```

### 267.5 ทำไม Django ถึงแยกสองเลเยอร์นี้ออกจากกัน — เหตุผลเชิงสถาปัตยกรรม

| เหตุผล | อธิบาย |
|---|---|
| **1. Single Responsibility Principle** | Field รับผิดชอบแค่ "ข้อมูลถูกต้องไหม" Widget รับผิดชอบแค่ "หน้าตาเป็นอย่างไร" — แต่ละ class มีเหตุผลเดียวที่จะเปลี่ยนแปลง |
| **2. Reusability สูงสุด** | Widget เดียว (เช่น `Select`) ใช้ได้กับหลาย Field (`ChoiceField`, `TypedChoiceField`, `ModelChoiceField`) และ Field เดียวใช้ได้กับหลาย Widget (`CharField` ใช้ได้ทั้ง `TextInput`, `Textarea`, `PasswordInput`) |
| **3. ทดสอบง่ายแยกส่วน** | ทดสอบ validation logic ของ Field ได้โดยไม่ต้อง render HTML เลย (ตามตัวอย่าง 267.2) และทดสอบหน้าตา widget ได้โดยไม่ต้องพึ่ง validation logic |
| **4. รองรับ Data Source ที่ไม่ใช่ HTML ในอนาคต** | เพราะ Field ไม่ผูกกับ HTML จึงมีโอกาสนำไปใช้กับ data source อื่นได้ในทางทฤษฎี (เช่น API ที่รับ JSON โดยตรง) แม้ในทางปฏิบัติ Django ยังใช้คู่กับ HTML เป็นหลัก |
| **5. เปลี่ยนหน้าตาได้โดยไม่กระทบข้อมูล** | ตัวอย่าง `RatingWidget` ในขั้นตอนที่ 263 คือหลักฐานชัดเจน: เปลี่ยนจาก radio ธรรมดาเป็นดาว โดยที่ validation และ `cleaned_data` ไม่เปลี่ยนแม้แต่บรรทัดเดียว |

### 267.6 ตารางสรุปเปรียบเทียบ Field vs Widget

| ประเด็น | Field | Widget |
|---|---|---|
| อยู่ใน module | `django.forms.fields` | `django.forms.widgets` |
| รู้จัก HTML ไหม | ไม่รู้จักเลย | รู้จักเต็มรูปแบบ (สร้าง HTML tag) |
| รู้จักความหมายของข้อมูลไหม | รู้จักเต็มรูปแบบ (int, date, email ฯลฯ) | ไม่รู้จัก (แค่ string/list ของ string) |
| Method สำคัญ | `clean()`, `to_python()`, `validate()`, `run_validators()` | `render()`, `value_from_datadict()`, `format_value()` |
| ใช้ validator (ขั้นตอนที่ 261-262) ไหม | ใช่ — เก็บใน `self.validators` | ไม่ใช้เลย |
| เปลี่ยนได้อิสระจากอีกฝั่งไหม | เปลี่ยน widget โดยไม่กระทบ field ได้ | เปลี่ยน field (คนละชนิด) แต่ยังใช้ widget เดิมได้ในบางกรณี |
| ตัวอย่างการทดสอบแยกหน่วย | `field.clean("42")` ไม่ต้องมี request | `widget.render("name", "value")` ไม่ต้อง validate |

---

## ขั้นตอนที่ 268: Dynamic Forms — เพิ่ม/ลบ Field แบบมีเงื่อนไข

### 268.1 สถานการณ์ใช้งานจริง: แบบฟอร์มที่ field ขึ้นอยู่กับคำตอบก่อนหน้า

สมมติระบบลงทะเบียนงานสัมมนาที่มี checkbox "ต้องการที่พักไหม" — ถ้าติ๊กเลือก
ต้องแสดง field เพิ่มเติม (จำนวนคืน, ประเภทห้อง) ที่ไม่ควรปรากฏถ้าไม่ได้เลือก
ทางเลือกในการ implement มี 2 แนวทางหลัก:

1. **Dynamic ฝั่ง Python** — เพิ่ม/ลบ field ใน `__init__()` ตามค่าที่ส่งมา
   (เหมาะกับ progressive enhancement, ทำงานได้แม้ JS ปิด)
2. **Dynamic ฝั่ง Client (JS)** — ซ่อน/แสดง field ด้วย CSS/JS แล้วค่อย validate
   ฝั่ง server ตามเงื่อนไขเดียวกัน (ให้ประสบการณ์ผู้ใช้ลื่นไหลกว่า)

โปรเจกต์ระดับ production ที่ดีมักทำ**ทั้งสองอย่างร่วมกัน**: ใช้ JS ซ่อน/แสดง
field เพื่อ UX ที่ดี แต่ต้อง validate ฝั่ง Python เสมอเพื่อความปลอดภัย (เพราะ
ผู้ใช้สามารถปิด JS หรือส่ง request ตรงไปที่ view ได้เสมอ)

### 268.2 แนวทางที่ 1: เพิ่ม/ลบ Field ใน `__init__()` ตามค่าที่ส่งมา

```python
# events/forms.py
from django import forms


class SeminarRegistrationForm(forms.Form):
    full_name = forms.CharField(max_length=150, label="ชื่อ-นามสกุล")
    email = forms.EmailField(label="อีเมล")
    needs_accommodation = forms.BooleanField(
        required=False,
        label="ต้องการที่พักระหว่างงานสัมมนา",
    )

    ROOM_TYPE_CHOICES = [
        ("single", "ห้องเดี่ยว"),
        ("twin", "ห้องคู่ (แชร์กับผู้เข้าร่วมท่านอื่น)"),
    ]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # เช็คว่าฟอร์มนี้ bound หรือไม่ และผู้ใช้ติ๊ก needs_accommodation ไหม
        # ใช้ self.data (ไม่ใช่ cleaned_data เพราะยังไม่ validate ตอนนี้)
        wants_accommodation = self._wants_accommodation()

        if wants_accommodation:
            self.fields["nights"] = forms.IntegerField(
                min_value=1,
                max_value=30,
                label="จำนวนคืนที่พัก",
            )
            self.fields["room_type"] = forms.ChoiceField(
                choices=self.ROOM_TYPE_CHOICES,
                label="ประเภทห้องพัก",
            )

    def _wants_accommodation(self):
        if not self.is_bound:
            return False
        # self.data คือ QueryDict ดิบ (ยังไม่ validate) — ใช้เช็คได้ปลอดภัย
        # เพราะ checkbox ที่ไม่ติ๊กจะไม่ส่งค่ามาเลยใน POST data
        return self.data.get("needs_accommodation") in ("on", "true", "True", "1")

    def clean(self):
        cleaned_data = super().clean()

        if cleaned_data.get("needs_accommodation"):
            # ตรวจซ้ำอีกชั้นว่า field ที่ควรถูกเพิ่มเข้ามามีค่าจริง
            # (ป้องกันกรณีมีคน submit ข้อมูลตรงไปที่ view โดยข้าม JS)
            if not cleaned_data.get("nights"):
                self.add_error("nights", "กรุณาระบุจำนวนคืนที่ต้องการพัก")
            if not cleaned_data.get("room_type"):
                self.add_error("room_type", "กรุณาเลือกประเภทห้องพัก")

        return cleaned_data
```

**จุดสำคัญที่ต้องเข้าใจ**: field ที่เพิ่มใน `__init__()` ด้วยเงื่อนไขแบบนี้
**ต้องอาศัย `self.data` ไม่ใช่ `self.cleaned_data`** เพราะ ณ ตอนที่
`__init__()` ทำงาน `is_valid()` ยังไม่เคยถูกเรียกเลย ไม่มี `cleaned_data`
อยู่จริง — นี่คือความแตกต่างสำคัญจาก `clean()` ที่เรียนใน Part 025 ซึ่งทำงาน
**หลัง** field ทั้งหมดถูกกำหนดแล้ว

### 268.3 View ที่รองรับ Dynamic Form

```python
# events/views.py
from django.shortcuts import render, redirect
from .forms import SeminarRegistrationForm


def seminar_registration_view(request):
    if request.method == "POST":
        form = SeminarRegistrationForm(request.POST)
        if form.is_valid():
            data = form.cleaned_data
            # field "nights"/"room_type" จะอยู่ใน cleaned_data ก็ต่อเมื่อ
            # needs_accommodation ถูกติ๊ก (เพราะถูกเพิ่มเข้ามาใน __init__ เท่านั้น)
            print(data)
            return redirect("events:registration_success")
    else:
        form = SeminarRegistrationForm()

    return render(request, "events/registration_form.html", {"form": form})
```

### 268.4 แนวทางที่ 2: ใช้ JavaScript ซ่อน/แสดง Field ที่มีอยู่แล้วในฟอร์ม (ทางเลือกที่ UX ดีกว่า)

อีกวิธีที่ใช้กันมากในโปรเจกต์จริงคือ **ประกาศ field ไว้ล่วงหน้าทุกตัวเสมอ**
(ไม่ต้องเพิ่ม/ลบใน `__init__()`) แล้วให้ JS เป็นตัวซ่อน/แสดง ส่วนฝั่ง Python
validate แบบมีเงื่อนไขใน `clean()` เท่านั้น — ข้อดีคือ **ไม่ reload หน้าเว็บ
ระหว่างกรอกฟอร์ม** ทำให้ผู้ใช้เห็น field ใหม่ปรากฏขึ้นทันทีที่ติ๊ก checkbox:

```python
class SeminarRegistrationForm(forms.Form):
    full_name = forms.CharField(max_length=150, label="ชื่อ-นามสกุล")
    email = forms.EmailField(label="อีเมล")
    needs_accommodation = forms.BooleanField(required=False, label="ต้องการที่พัก")
    nights = forms.IntegerField(min_value=1, max_value=30, required=False, label="จำนวนคืนที่พัก")
    room_type = forms.ChoiceField(
        choices=[("single", "ห้องเดี่ยว"), ("twin", "ห้องคู่")],
        required=False,
        label="ประเภทห้องพัก",
    )

    def clean(self):
        cleaned_data = super().clean()
        if cleaned_data.get("needs_accommodation"):
            if not cleaned_data.get("nights"):
                self.add_error("nights", "กรุณาระบุจำนวนคืนที่ต้องการพัก")
            if not cleaned_data.get("room_type"):
                self.add_error("room_type", "กรุณาเลือกประเภทห้องพัก")
        return cleaned_data
```

```html
<!-- events/templates/events/registration_form.html -->
<form method="post" id="registration-form">
    {% csrf_token %}
    {{ form.full_name.as_field_group }}
    {{ form.email.as_field_group }}
    {{ form.needs_accommodation.as_field_group }}

    <div id="accommodation-fields" style="display: none;">
        {{ form.nights.as_field_group }}
        {{ form.room_type.as_field_group }}
    </div>

    <button type="submit">ลงทะเบียน</button>
</form>

<script>
    const checkbox = document.querySelector("#id_needs_accommodation");
    const extraFields = document.querySelector("#accommodation-fields");

    function toggleAccommodationFields() {
        extraFields.style.display = checkbox.checked ? "block" : "none";
    }

    checkbox.addEventListener("change", toggleAccommodationFields);
    toggleAccommodationFields();   // เรียกครั้งแรกตอนโหลดหน้า เผื่อฟอร์มเดิมมีค่าติ๊กไว้แล้ว (เช่นหลัง validate ไม่ผ่าน)
</script>
```

`{{ field.as_field_group }}` เป็น method ใหม่ที่เพิ่มเข้ามาใน Django 5.0
render field เดียวพร้อม label, help text, และ error ในโครงสร้าง `<div>`
เดียวกัน (คล้าย `as_div` แต่ทำระดับ field เดียว) สะดวกเมื่อต้องผสมการ render
อัตโนมัติกับ layout กำหนดเองแบบในตัวอย่างนี้

### 268.5 ตารางเปรียบเทียบสองแนวทาง Dynamic Form

| ประเด็น | แนวทาง 1: เพิ่ม/ลบ field ใน `__init__()` | แนวทาง 2: ประกาศ field ไว้ล่วงหน้า + JS ซ่อน/แสดง |
|---|---|---|
| ทำงานได้แม้ JS ปิดไหม | ได้ (ต้อง submit form ใหม่เพื่อเห็น field เพิ่ม ถ้าไม่มี JS ช่วย) | field มีอยู่แล้วใน HTML เสมอ (แค่ถูกซ่อนด้วย CSS) — validate ฝั่ง server ทำงานได้ปกติ |
| UX (ไม่ reload หน้า) | แย่กว่า ถ้าไม่มี JS ช่วยเสริม | ดีกว่ามาก — เห็น field ปรากฏทันที |
| ความซับซ้อนของโค้ด Python | สูงกว่า (ต้องจัดการ `self.data` ใน `__init__`) | ต่ำกว่า (field ประกาศตรงไปตรงมา, logic อยู่ใน `clean()` เท่านั้น) |
| เหมาะกับ | ฟอร์มที่มีโครงสร้างเปลี่ยนแปลงมาก (field ใหม่ทั้งชุดตามประเภทที่เลือก) | ฟอร์มที่ field เพิ่มเติมมีจำนวนคงที่ ไม่เปลี่ยนโครงสร้างมาก |

**กฎเหล็กที่ใช้ได้กับทั้งสองแนวทาง**: **ต้อง validate เงื่อนไขซ้ำเสมอฝั่ง
Python ใน `clean()`** ไม่ว่าจะซ่อน/แสดง field ด้วย JS หรือไม่ก็ตาม เพราะ
ผู้ใช้ที่ประสงค์ร้ายสามารถส่ง POST request ตรงไปที่ view ได้โดยไม่ผ่านหน้าเว็บ
หรือ JS เลย (เช่นผ่าน `curl` หรือเครื่องมืออย่าง Postman) — Client-side
validation มีไว้เพื่อ **UX** เท่านั้น ไม่ใช่เพื่อ **ความปลอดภัย**

---

## ขั้นตอนที่ 269: ผสาน Custom Widget เข้ากับ Formset

### 269.1 ทบทวนแนวคิด Formset จาก Part 026 โดยย่อ

Part 026 แนะนำ `formset_factory()` และ `modelformset_factory()` สำหรับจัดการ
ฟอร์มหลายชุดพร้อมกันในหน้าเดียว (เช่น เพิ่มสินค้าหลายรายการในใบสั่งซื้อเดียว)
ขั้นตอนนี้จะแสดงว่า custom widget ที่สร้างมาตลอด Part นี้ — `RatingWidget`,
`FlatpickrDateWidget`, `ImagePreviewWidget` — **ทำงานร่วมกับ Formset ได้โดยไม่
ต้องแก้อะไรเพิ่มเลย** เพราะ Formset เป็นเพียง "ตัวจัดการฟอร์มหลายชุด" ที่แต่ละ
ฟอร์มภายในยังคงเป็น `Form`/`ModelForm` ปกติทุกประการ

### 269.2 ตัวอย่าง: Formset ของรีวิวสินค้าหลายรายการพร้อม `RatingWidget`

สมมติหน้าเพิ่มรีวิวสินค้าแบบ batch (เช่น ทีมงานนำเข้ารีวิวเก่าจากระบบอื่น
ทีละหลายรายการ) โดยแต่ละแถวต้องมีดาวให้คะแนนของตัวเอง:

```python
# reviews/forms.py
from django import forms
from django.forms import formset_factory
from .widgets import RatingWidget


class ReviewForm(forms.Form):
    product_sku = forms.CharField(max_length=20, label="รหัสสินค้า")
    rating = forms.ChoiceField(
        choices=[(str(i), str(i)) for i in range(1, 6)],
        widget=RatingWidget,
        label="คะแนน",
    )
    comment = forms.CharField(
        widget=forms.Textarea(attrs={"rows": 2}),
        required=False,
        label="ความคิดเห็น",
    )


ReviewFormSet = formset_factory(ReviewForm, extra=3, max_num=20)
```

```python
# reviews/views.py
from django.shortcuts import render, redirect
from .forms import ReviewFormSet


def bulk_review_import_view(request):
    if request.method == "POST":
        formset = ReviewFormSet(request.POST)
        if formset.is_valid():
            for form in formset:
                if form.cleaned_data:   # ข้ามแถวว่างที่ผู้ใช้ไม่ได้กรอก
                    print(form.cleaned_data)
            return redirect("reviews:import_success")
    else:
        formset = ReviewFormSet()

    return render(request, "reviews/bulk_review_import.html", {"formset": formset})
```

### 269.3 Render Formset พร้อม Media ของ Custom Widget — จุดที่ต้องระวัง

เมื่อฟอร์มภายใน formset ใช้ widget ที่มี `Media` (เช่น ถ้าจะผสม
`FlatpickrDateWidget` เข้าไปด้วยสำหรับ field วันที่รีวิว) **ต้อง render
`{{ formset.media }}` ไม่ใช่ `{{ form.media }}`** เพราะ `formset.media` จะ
รวบรวม Media จาก **empty_form** (แม่แบบของทุกฟอร์มใน formset) ให้ครบถ้วนเพียง
ครั้งเดียว แม้ formset จะมีหลายแถว:

```html
<!-- reviews/templates/reviews/bulk_review_import.html -->
{% extends 'base.html' %}

{% block extra_head %}
    {{ formset.media }}   {# รวม CSS/JS ของ RatingWidget (ถ้ามี Media) จากทุกฟอร์มในชุดนี้ ครั้งเดียว #}
{% endblock %}

{% block content %}
<h1>นำเข้ารีวิวสินค้า (Batch Import)</h1>

<form method="post">
    {% csrf_token %}
    {{ formset.management_form }}

    {% for form in formset %}
        <fieldset class="review-row">
            <legend>รีวิวรายการที่ {{ forloop.counter }}</legend>
            {{ form.as_div }}
        </fieldset>
    {% endfor %}

    <button type="submit">บันทึกรีวิวทั้งหมด</button>
</form>
{% endblock %}
```

**อย่าลืม `{{ formset.management_form }}`** (สอนไว้ใน Part 026) — ถ้าลืมใส่
Django จะไม่รู้ว่ามีกี่ฟอร์มในชุดนี้ทั้งหมด (`TOTAL_FORMS`, `INITIAL_FORMS`)
ทำให้ formset ทั้งชุด raise `ValidationError` ทันทีตอนเรียก `is_valid()`

### 269.4 ข้อควรระวัง: `id` ของ Widget ต้องไม่ชนกันข้ามฟอร์มใน Formset

Custom widget ที่เขียน JavaScript อ้างอิง `id` ตรง ๆ (อย่าง
`ImagePreviewWidget` และ `RatingWidget` ในตัวอย่างก่อนหน้า) ต้องระวังเรื่อง
**id ซ้ำกันข้ามฟอร์มในหน้าเดียว** เพราะ Formset ตั้งชื่อ field ให้แต่ละ
ฟอร์มด้วย prefix ตัวเลขอัตโนมัติ (เช่น `form-0-rating`, `form-1-rating`)
ทำให้ `id` ของแต่ละ field ที่ Django gen ให้ (`id_form-0-rating`,
`id_form-1-rating`) **ไม่ชนกันอยู่แล้วโดยอัตโนมัติ** ถ้า template ของ widget
ใช้ `{{ widget.attrs.id }}` อย่างถูกต้อง (ตามที่เขียนไว้ในขั้นตอนที่ 263 และ
266) โค้ด JS ที่ผูกกับ `id` เฉพาะเจาะจงแบบนี้จะทำงานถูกต้องในทุกแถวของ
formset โดยอัตโนมัติ **โดยไม่ต้องแก้โค้ด widget เพิ่มเลย**

แต่ถ้า widget เขียน JS แบบ hardcode id (เช่น
`document.getElementById("rating-star-1")` ตรง ๆ โดยไม่อ้างอิงจาก
`widget.attrs.id`) จะเกิดปัญหาทันทีเมื่อใช้ใน formset เพราะ JS จะผูกกับ
element แรกที่เจอเท่านั้น ทุกแถวที่เหลือจะไม่ทำงาน — นี่คือเหตุผลสำคัญที่
ทุกตัวอย่าง widget ใน Part นี้ **ใช้ `{{ widget.attrs.id }}` เสมอแทนการ
hardcode ชื่อ** เพื่อให้พร้อมใช้งานร่วมกับ Formset ได้ทันทีโดยไม่ต้องแก้ไข

| หลักการออกแบบ Custom Widget ให้ใช้กับ Formset ได้ | เหตุผล |
|---|---|
| ใช้ `{{ widget.attrs.id }}` ใน template เสมอ ไม่ hardcode | Formset ตั้ง id ให้ไม่ซ้ำกันอัตโนมัติ (prefix ด้วยเลขแถว) |
| Render `{{ formset.media }}` แทน `{{ form.media }}` | ป้องกัน CSS/JS ถูกโหลดซ้ำหลายรอบโดยไม่จำเป็น |
| อย่าลืม `{{ formset.management_form }}` | จำเป็นเสมอไม่ว่าฟอร์มภายในจะใช้ widget แบบไหนก็ตาม |
| ทดสอบ widget กับ `extra=2` ขึ้นไปเสมอ ไม่ทดสอบแค่ 1 แถว | เพื่อจับบั๊กเรื่อง id ซ้ำกันตั้งแต่ตอนพัฒนา |

### 269.5 ตัวอย่างสมบูรณ์: ผสาน `ImagePreviewWidget` เข้ากับ Formset ของ `ModelForm`

```python
# products/forms.py
from django.forms import modelformset_factory
from .models import ProductImage
from accounts.widgets import ImagePreviewWidget


ProductImageFormSet = modelformset_factory(
    ProductImage,
    fields=["image", "caption"],
    extra=3,
    widgets={"image": ImagePreviewWidget()},
)
```

```python
# products/views.py
from django.shortcuts import render, redirect
from .models import ProductImage
from .forms import ProductImageFormSet


def manage_product_images_view(request, product_id):
    queryset = ProductImage.objects.filter(product_id=product_id)

    if request.method == "POST":
        formset = ProductImageFormSet(
            request.POST, request.FILES, queryset=queryset
        )
        if formset.is_valid():
            instances = formset.save(commit=False)
            for instance in instances:
                instance.product_id = product_id
                instance.save()
            for obj in formset.deleted_objects:
                obj.delete()
            return redirect("products:manage_images", product_id=product_id)
    else:
        formset = ProductImageFormSet(queryset=queryset)

    return render(
        request,
        "products/manage_images.html",
        {"formset": formset, "product_id": product_id},
    )
```

```html
<!-- products/templates/products/manage_images.html -->
{% extends 'base.html' %}
{% block extra_head %}{{ formset.media }}{% endblock %}

{% block content %}
<h1>จัดการรูปภาพสินค้า</h1>

<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ formset.management_form }}
    {% for form in formset %}
        <div class="product-image-row">
            {{ form.id }}
            {{ form.as_div }}
        </div>
    {% endfor %}
    <button type="submit">บันทึกรูปภาพทั้งหมด</button>
</form>
{% endblock %}
```

ทุกฟอร์มในหน้านี้แสดง preview รูปเดิม (ถ้ามี) และ preview รูปใหม่แบบ
real-time ผ่าน `ImagePreviewWidget` ที่สร้างในขั้นตอนที่ 266 โดยไม่ต้องแก้
widget class แม้แต่บรรทัดเดียว — เป็นเครื่องพิสูจน์ที่ชัดเจนว่า **การออกแบบ
Widget ให้ถูกต้องตั้งแต่แรก (ไม่ hardcode id, ประกาศ Media อย่างเหมาะสม)
ทำให้นำไปใช้ซ้ำได้ในทุกบริบท** ไม่ว่าจะเป็นฟอร์มเดี่ยวหรือ Formset

---

## ขั้นตอนที่ 270: สรุปและแบบฝึกหัด

### 270.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ Built-in Validators (`MinValueValidator`, `RegexValidator`,
  `EmailValidator` ฯลฯ) และรู้วิธีใช้ validator ตัวเดียวกันทั้งใน Model field
  และ Form field เพื่อรักษาหลักการ DRY
- ✅ เขียน Custom Validator function/class เอง (`BannedWordsValidator`,
  `validate_thai_phone_number`) พร้อมเข้าใจ `@deconstructible` และเหตุผลที่
  ต้องมี `__eq__`
- ✅ สร้าง Custom Widget โดย subclass widget ที่มีอยู่ (`RatingWidget` จาก
  `RadioSelect`) โดยใช้ Widget Rendering API (`template_name`,
  `option_template_name`) แทนการต่อ HTML string เอง
- ✅ ใช้ Widget `Media` class แนบ CSS/JS ให้ widget พกพาตัวเอง และเข้าใจว่า
  `{{ form.media }}`/`{{ formset.media }}` รวบรวมไฟล์ให้อัตโนมัติแบบไม่ซ้ำ
- ✅ เปรียบเทียบ django-widget-tweaks (เติม attribute ในเทมเพลต) กับ
  django-crispy-forms (Layout object ใน Python) และรู้ว่าจะเลือกใช้แบบไหน
  เมื่อไร
- ✅ ปรับแต่ง `ClearableFileInput` ให้แสดง preview รูปภาพทั้งรูปเดิมและรูปใหม่
  ที่เพิ่งเลือก
- ✅ เข้าใจความแตกต่างระหว่าง Field (`clean`, `to_python`, `validate`,
  `run_validators`) กับ Widget (`render`, `value_from_datadict`,
  `format_value`) แบบเจาะลึกถึงเหตุผลเชิงสถาปัตยกรรม
- ✅ สร้าง Dynamic Form ทั้งแบบเพิ่ม/ลบ field ใน `__init__()` และแบบ
  ประกาศไว้ล่วงหน้า + JS ซ่อน/แสดง พร้อมกฎเหล็กเรื่อง server-side validation
- ✅ ผสาน Custom Widget เข้ากับ Formset ได้อย่างปลอดภัย โดยไม่เกิดปัญหา `id`
  ชนกันข้ามแถว

### 270.2 Checklist ก่อนไป Part ถัดไป

- [ ] เขียน `validate_thai_phone_number` และเทสต์ด้วย `SimpleTestCase` สำเร็จ
- [ ] สร้าง `RatingWidget` และเห็นดาวคลิกได้จริงในเบราว์เซอร์
- [ ] ประกาศ `Media` ใน custom widget อย่างน้อย 1 ตัว และเห็น `{{ form.media }}`
      render CSS/JS ให้อัตโนมัติ
- [ ] ลองใช้ทั้ง `django-widget-tweaks` และ `django-crispy-forms` ในโปรเจกต์
      ทดลอง แล้วเปรียบเทียบด้วยตัวเอง
- [ ] สร้าง `ImagePreviewWidget` และเห็น preview รูปเดิม/รูปใหม่ทำงานถูกต้อง
- [ ] อธิบายความแตกต่าง Field vs Widget ด้วยคำพูดตัวเองได้ โดยไม่ต้องเปิด
      เอกสารดู
- [ ] สร้าง Dynamic Form ที่ field เปลี่ยนตามเงื่อนไข พร้อม validate ฝั่ง
      server ครบถ้วน
- [ ] ผสาน custom widget เข้ากับ Formset และทดสอบกับ `extra=3` ขึ้นไปโดยไม่มี
      ปัญหา id ชนกัน

### 270.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน Custom Validator ชื่อ `validate_thai_national_id`
ที่ตรวจสอบเลขบัตรประชาชนไทย 13 หลัก รวมถึงการตรวจ **checksum digit** ตาม
สูตรที่ใช้จริง (หลักที่ 13 คำนวณจากผลรวมถ่วงน้ำหนักของ 12 หลักแรก) แล้วนำไปใช้
ทั้งใน Model field ของ `Customer` และใน `forms.Form` แยกต่างหาก พร้อมเขียน
unit test อย่างน้อย 4 กรณี (เลขถูกต้อง, checksum ผิด, ความยาวผิด, มีตัวอักษร
ปน)

**แบบฝึกหัดที่ 2**: สร้าง Custom Widget ชื่อ `ColorSwatchWidget` ที่ subclass
จาก `RadioSelect` เพื่อแสดงตัวเลือกสีเป็นสี่เหลี่ยมสีจริง (ไม่ใช่ข้อความ)
สำหรับฟอร์มเลือกสีสินค้า (เช่น แดง, น้ำเงิน, เขียว, ดำ) โดยใช้เทคนิค Widget
Rendering API เหมือน `RatingWidget` ในขั้นตอนที่ 263 (กำหนด `template_name`
และ `option_template_name` เอง) พร้อมแนบ CSS ผ่าน `Media`

**แบบฝึกหัดที่ 3**: นำ `django-crispy-forms` มาใช้กับฟอร์ม `SeminarRegistrationForm`
จากขั้นตอนที่ 268 (แนวทางที่ 2) โดยจัด Layout ให้ `nights` กับ `room_type`
อยู่ในกลุ่ม (`Fieldset` หรือ `Div`) ที่มี CSS class พิเศษ เพื่อให้ต่อยอดด้วย
JavaScript ซ่อน/แสดงได้ง่ายขึ้น เปรียบเทียบปริมาณโค้ดเทมเพลตที่ต้องเขียนกับ
วิธีเดิมที่ใช้ `{{ field.as_field_group }}` ตรง ๆ

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้างหน้า "นำเข้าคำถามแบบทดสอบ (Quiz Import)"
ที่ใช้ `formset_factory` ร่วมกับฟอร์มที่มี field `difficulty` ใช้
`RatingWidget` (1-5 ดาว แทนระดับความยาก) และ field `due_date` ใช้
`FlatpickrDateWidget` จากขั้นตอนที่ 264 ในฟอร์มเดียวกัน ทดสอบว่า
`{{ formset.media }}` รวบรวม CSS/JS ของทั้งสอง widget โดยไม่ซ้ำกันเมื่อมี
`extra=5` แถว และทดสอบว่าดาวคลิกได้ถูกต้องในทุกแถวโดยไม่มีแถวใดเกาะติดกัน
(ใช้ Developer Tools ตรวจสอบว่า `id` ของแต่ละแถวไม่ซ้ำกันจริง)

### 270.4 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเขียน `@deconstructible` ทุกครั้งที่สร้าง custom validator ไหม?**
A: ไม่จำเป็นเสมอไป — จำเป็นเฉพาะเมื่อ validator นั้นเป็น **class-based** (มี
`__init__` ที่รับ parameter) **และ** ถูกใช้ใน **Model field** เพราะต้องผ่าน
ระบบ migration ถ้า validator เป็นแค่ **function ธรรมดา** (ไม่มี state ใด ๆ
เก็บไว้ใน instance) อย่าง `validate_thai_phone_number` ไม่จำเป็นต้องใช้
`@deconstructible` เลย เพราะ Django serialize function โดยอ้างอิง import
path ตรง ๆ ได้อยู่แล้ว และถ้า validator ใช้แค่ใน `forms.Form` (ไม่แตะ Model
เลย) ก็ไม่ต้องกังวลเรื่อง migration เช่นกัน

**Q: ควรเขียน Custom Widget เมื่อไร กับควรใช้ JavaScript library สำเร็จรูปทับ
widget เดิมเมื่อไร?**
A: ถ้าสิ่งที่ต้องการเปลี่ยนแค่ **หน้าตา HTML แบบง่าย** ที่ยัง render ฝั่ง
server ได้ครบ (เช่น `RatingWidget`, `ColorSwatchWidget`) เขียน Custom Widget
ตรงไปตรงมาและควบคุมได้เต็มที่ แต่ถ้าต้องการ **behavior ที่ซับซ้อนฝั่ง client**
จริง ๆ (เช่น autocomplete แบบเรียก API, drag-and-drop) ให้ใช้ widget เดิม
(หรือ widget ง่าย ๆ) เป็นฐาน แล้วแนบ JS library ผ่าน `Media` เหมือน
`FlatpickrDateWidget` — อย่าพยายาม "เขียน JS ทั้งหมดเองใหม่" เมื่อมี library
ที่ได้รับการทดสอบมาอย่างดีอยู่แล้ว

**Q: `django-crispy-forms` ทำให้ฟอร์มช้าลงไหม เพราะต้อง render ผ่าน Layout
object เพิ่ม?**
A: ผลกระทบด้าน performance น้อยมากจนไม่มีนัยสำคัญในทางปฏิบัติ เพราะ crispy
forms แค่เลือก template ที่เหมาะสมให้ตาม Layout ที่กำหนด ไม่ได้มี logic หนัก
ระดับที่ส่งผลต่อ response time ในระบบจริง (เว้นแต่ฟอร์มนั้นมี field มาก
ระดับหลักร้อยใน request เดียว ซึ่งเป็นปัญหาการออกแบบฟอร์มมากกว่าปัญหาของ
crispy forms เอง)

**Q: ทำไมไม่ใช้ HTML5 validation attributes (`required`, `pattern`,
`minlength`) แทน Django Validator ไปเลย จะได้ไม่ต้อง roundtrip ไปเซิร์ฟเวอร์?**
A: Django ใส่ HTML5 attributes เหล่านี้ให้อัตโนมัติอยู่แล้วบางส่วน (เช่น
`required`, `maxlength` จาก `max_length`) เพื่อให้ผู้ใช้ได้ฟีดแบ็กทันทีโดยไม่
ต้อง submit ก่อน แต่ **ต้อง validate ฝั่ง server เสมอเป็นด่านสุดท้าย** เพราะ
HTML5 validation เป็นเพียงการช่วยด้าน UX ที่ผู้ใช้สามารถ bypass ได้ง่ายมาก
(ปิด JS, แก้ HTML ผ่าน DevTools, หรือส่ง request ตรงด้วยเครื่องมืออย่าง
`curl`/Postman) — หลักการนี้ตรงกับที่กล่าวไว้ในขั้นตอนที่ 268.5: **Client-side
validation มีไว้เพื่อ UX เท่านั้น ไม่ใช่เพื่อความปลอดภัย**

### 270.5 ตารางสรุปเปรียบเทียบเทคนิคทั้งหมดใน Part นี้

| เทคนิค | แก้ปัญหาอะไร | ใช้ตอนไหน |
|---|---|---|
| Built-in Validators | ตรวจสอบเงื่อนไขมาตรฐาน (ช่วงค่า, รูปแบบ regex) แบบใช้ซ้ำได้ | เมื่อกฎ validation เหมือนกับที่ Django มีให้อยู่แล้ว |
| Custom Validator function | ตรวจสอบ business logic เฉพาะโดเมน | เมื่อกฎซับซ้อนกว่า built-in และต้องใช้ซ้ำหลายที่/หลายฟอร์ม |
| Custom Widget (subclass) | เปลี่ยนหน้าตา HTML โดยไม่กระทบ validation | เมื่อ UI มาตรฐานไม่ตอบโจทย์ (เช่น ดาว, สี, ปฏิทิน) |
| Widget `Media` | แนบ CSS/JS ให้ widget พกพาตัวเอง | เมื่อ widget ต้องพึ่ง library ภายนอก |
| django-widget-tweaks | จัด CSS class จากเทมเพลตโดยไม่แตะ `forms.py` | ฟอร์มจำนวนน้อย ต้องการคุม UI ละเอียด |
| django-crispy-forms | Layout ทั้งฟอร์มแบบ declarative ใน Python | ระบบที่มีฟอร์มเยอะ ต้องการ UI สม่ำเสมอ |
| Custom `ClearableFileInput` | แสดง preview ไฟล์/รูปภาพ | ฟอร์มอัปโหลดไฟล์ที่ต้องการ UX ที่ดีกว่า |
| Dynamic Form | ฟอร์มที่โครงสร้าง field เปลี่ยนตามเงื่อนไข | ฟอร์มแบบมีเงื่อนไข (conditional fields) |
| Custom Widget + Formset | ใช้ widget ที่ซับซ้อนกับฟอร์มหลายชุดพร้อมกัน | หน้าจัดการข้อมูลแบบ batch/bulk |

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 028: Template Inheritance และ Template Tags** จะพาย้อนกลับไปเจาะลึก
ฝั่ง Template ให้ครบเครื่องยิ่งขึ้น หลังจากที่ Part 025-027 เน้นหนักไปทาง
Forms/Widgets เราจะเรียนรู้ `{% extends %}`/`{% block %}` แบบหลายชั้น
(multi-level inheritance), `{% include %}` พร้อมส่ง context, built-in
template tags ที่ยังไม่ได้ใช้ (`{% with %}`, `{% cycle %}`, `{% regroup %}`)
และเทคนิคจัดโครงสร้างเทมเพลตขนาดใหญ่ให้ดูแลง่ายในระยะยาว ซึ่งจะปูทางไปสู่
การเขียน **Custom Template Tags และ Filters ของตัวเอง** ใน Part 029 — ทักษะ
ที่จะทำให้คุณลดการเขียน logic ซ้ำ ๆ ในหลายเทมเพลตได้อย่างมาก

เตรียมทบทวนโครงสร้างเทมเพลตของโปรเจกต์ที่คุณสร้างมาตั้งแต่ Part 008 ไว้ให้
พร้อม แล้วไปต่อกันเลย!
