# Part 011: Django Models เบื้องต้น: Fields และ Migrations

> **ขั้นตอนที่ 101-110 ของหลักสูตร** | Phase 2: Models, ORM และ Admin (Part แรกของ Phase นี้)
>
> ใน Phase 1 คุณเห็น `Post` model ผ่านตามาแล้วหลายครั้ง (Part 005, Part 007, Part 010)
> แต่เราจงใจพูดถึงมันแบบ "ผ่าน ๆ" เพียงพอให้เขียน view และ template ได้ ยังไม่เคยอธิบาย
> อย่างจริงจังว่า field แต่ละตัวมีตัวเลือกอะไรบ้าง, migration ที่ถูกสร้างขึ้นหน้าตาเป็น
> อย่างไรข้างใน, หรือทำไม Django ถึงออกแบบระบบนี้มาแบบนี้ **Phase 2 ทั้ง Phase**
> (Part 011-020) จะเจาะลึกทุกสิ่งที่ Phase 1 แอบ "เปิดเผยไว้ล่วงหน้าแบบผิวเผิน" ให้ลึกถึง
> รากฐานจริง ๆ เริ่มจาก Part นี้ที่จะทำให้คุณเข้าใจ Field types, Field options, และระบบ
> Migration ของ Django อย่างละเอียดที่สุดเท่าที่มือใหม่คนหนึ่งควรรู้ก่อนไปเรียนเรื่อง
> ความสัมพันธ์ระหว่างโมเดลใน Part 012 เมื่อจบ Part นี้ คุณจะขยาย `Post` model ให้สมบูรณ์
> แบบมืออาชีพ อ่านไฟล์ migration ที่ Django สร้างให้เข้าใจทุกบรรทัด และรู้วิธีแก้ไข model
> ที่มีข้อมูลอยู่แล้วโดยไม่ทำข้อมูลพัง

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 101: Django Model คืออะไร และ Field Types ที่ใช้บ่อยที่สุดทั้งหมด
- ขั้นตอนที่ 102: Field Options เจาะลึก: `null`, `blank`, `default`, `unique`, `choices`, `help_text`, `verbose_name`
- ขั้นตอนที่ 103: ขยาย `Post` Model จาก Part 007 ให้สมบูรณ์แบบมืออาชีพ
- ขั้นตอนที่ 104: `makemigrations` เจาะลึก — เปิดไฟล์ Migration จริงมาอ่านทีละบรรทัด
- ขั้นตอนที่ 105: `migrate` เจาะลึก — Migration Graph, `showmigrations`, `sqlmigrate`
- ขั้นตอนที่ 106: Model Methods: `__str__`, `get_absolute_url()`, Business Logic Methods
- ขั้นตอนที่ 107: Model `Meta` Class เบื้องต้น
- ขั้นตอนที่ 108: Primary Key ทางเลือก: `AutoField` vs `BigAutoField` vs `UUIDField`
- ขั้นตอนที่ 109: แก้ไข Model ที่มีข้อมูลอยู่แล้ว และผลกระทบต่อ Migration ที่ต้องระวัง
- ขั้นตอนที่ 110: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 101: Django Model คืออะไร และ Field Types ที่ใช้บ่อยที่สุดทั้งหมด

### 101.1 ทบทวนสั้น ๆ: Model คืออะไร

จาก Part 001 (ขั้นตอนที่ 3.3) คุณรู้แล้วว่า **Model** คือ Python class ที่สืบทอดจาก
`django.db.models.Model` โดยแต่ละ class แทน 1 ตารางในฐานข้อมูล และแต่ละ attribute
(ที่เป็น Field object) แทน 1 คอลัมน์ นี่คือแก่นของสิ่งที่เรียกว่า **ORM
(Object-Relational Mapping)** — เทคนิคที่แปลง object ของภาษาโปรแกรม (Python class/instance)
ให้เป็นแถวข้อมูลในตารางฐานข้อมูลเชิงสัมพันธ์ (relational database) โดยอัตโนมัติ โดยที่คุณ
แทบไม่ต้องเขียน SQL เอง

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    # ...
```

เมื่อ Django เห็น class นี้ มันรู้ทันทีว่า:

1. ต้องสร้างตารางชื่อ `blog_post` (รูปแบบ `<app_label>_<model_name ตัวพิมพ์เล็ก>`)
2. ต้องมีคอลัมน์ `title` เป็นชนิด `VARCHAR(200)` (หรือเทียบเท่าในฐานข้อมูลที่ใช้)
3. ต้องมีคอลัมน์ `id` เป็น Primary Key แบบ auto-increment ให้อัตโนมัติ (ถ้าไม่ได้กำหนด
   primary key เอง — รายละเอียดเต็มอยู่ในขั้นตอนที่ 108)

สิ่งที่ Django **ไม่ได้ทำทันที** คือสร้างตารางจริงในฐานข้อมูล — นั่นคือหน้าที่ของระบบ
**Migration** ซึ่งเป็นหัวใจของ Part นี้ตั้งแต่ขั้นตอนที่ 104 เป็นต้นไป

### 101.2 ทำไมต้องมี Field Type หลากหลายขนาดนี้

มือใหม่มักสงสัยว่าทำไมไม่ใช้ `CharField` กับทุกอย่างไปเลย เหตุผลคือ Field type แต่ละตัว
ทำหน้าที่ **3 อย่างพร้อมกัน**:

1. **กำหนดชนิดข้อมูลในฐานข้อมูลจริง** (เช่น `INTEGER`, `VARCHAR`, `TEXT`, `BOOLEAN`)
   ซึ่งส่งผลต่อพื้นที่จัดเก็บและความเร็วในการ query
2. **กำหนดการ validate ข้อมูล** ทั้งใน Django Admin และ Django Forms โดยอัตโนมัติ
   (เช่น `EmailField` จะไม่ยอมให้บันทึกค่าที่ไม่ใช่รูปแบบอีเมล)
3. **กำหนด widget เริ่มต้น** ที่ใช้แสดงผลใน Django Admin/Forms (เช่น `DateField`
   แสดงเป็น date picker, `BooleanField` แสดงเป็น checkbox)

การเลือก Field type ให้ถูกต้องตั้งแต่แรกจึงสำคัญมาก เพราะมันกำหนดทั้งประสิทธิภาพของ
ฐานข้อมูล **และ** ความถูกต้องของข้อมูลไปพร้อมกัน

### 101.3 ตารางสรุป Field Types ที่ใช้บ่อยที่สุดทั้งหมด

| Field Type | เก็บข้อมูลชนิด | ตัวอย่างการใช้งานจริง | หมายเหตุสำคัญ |
|---|---|---|---|
| `CharField` | ข้อความสั้น ความยาวจำกัด | ชื่อสินค้า, หัวข้อบทความ, username | **บังคับ** ต้องระบุ `max_length` เสมอ |
| `TextField` | ข้อความยาวไม่จำกัด | เนื้อหาบทความ, คำบรรยาย, comment | ไม่ต้องระบุ `max_length` (ถึงจะระบุได้ตั้งแต่ Django 5.0 แต่ไม่บังคับ) |
| `IntegerField` | จำนวนเต็ม (32-bit signed) | จำนวนสต็อกสินค้า, อายุ | ช่วงค่า -2,147,483,648 ถึง 2,147,483,647 |
| `PositiveIntegerField` | จำนวนเต็มบวก (รวม 0) | จำนวนครั้งที่เข้าชม, จำนวนไลก์ | ป้องกันค่าติดลบตั้งแต่ระดับ validation |
| `DecimalField` | ทศนิยมแม่นยำสูง (fixed-point) | ราคาสินค้า, ยอดเงิน | **บังคับ** ต้องระบุ `max_digits` และ `decimal_places` |
| `FloatField` | ทศนิยมแบบ floating-point | ค่าพิกัด GPS, คะแนนเฉลี่ยโดยประมาณ | มีความคลาดเคลื่อนได้ **ห้ามใช้กับเงิน** |
| `BooleanField` | จริง/เท็จ (True/False) | สถานะเผยแพร่, เปิด/ปิดใช้งาน | แนะนำให้ระบุ `default` เสมอ |
| `DateField` | วันที่ (ไม่มีเวลา) | วันเกิด, วันครบกำหนด | เก็บเป็น `date` object ใน Python |
| `DateTimeField` | วันที่และเวลา | เวลาสร้าง/แก้ไขข้อมูล, เวลานัดหมาย | เก็บเป็น `datetime` object พร้อม timezone (ถ้าเปิด `USE_TZ`) |
| `EmailField` | ข้อความที่ต้องเป็นรูปแบบอีเมล | อีเมลผู้ใช้ | เป็น `CharField` ที่เพิ่ม `EmailValidator` ให้อัตโนมัติ (`max_length` default = 254) |
| `URLField` | ข้อความที่ต้องเป็นรูปแบบ URL | ลิงก์เว็บไซต์, ลิงก์โซเชียล | เป็น `CharField` ที่เพิ่ม `URLValidator` ให้อัตโนมัติ (`max_length` default = 200) |
| `SlugField` | ข้อความสำหรับใช้ใน URL | slug ของบทความ/สินค้า | รับเฉพาะ ตัวอักษร, ตัวเลข, `-`, `_` เท่านั้น |
| `ImageField` | ไฟล์รูปภาพ | รูปโปรไฟล์, รูปสินค้า | เป็น `FileField` ที่ validate ว่าไฟล์เป็นรูปจริง (ต้องติดตั้ง `Pillow`) |
| `FileField` | ไฟล์ทั่วไป | เอกสารแนบ, resume, PDF | เก็บ **path ของไฟล์** ในฐานข้อมูล ไม่ใช่เนื้อไฟล์ |

### 101.4 ตัวอย่างการใช้งานแต่ละ Field Type

```python
from django.db import models


class FieldShowcase(models.Model):
    """
    Model สาธิต Field types ทั้งหมดในตารางข้างต้น (ไม่ใช่ model จริงของโปรเจกต์
    เขียนไว้เพื่ออธิบายเท่านั้น — ไม่ต้องสร้างไฟล์นี้จริงในโปรเจกต์ของคุณ)
    """
    # ข้อความสั้น: ต้องระบุ max_length เสมอ (บังคับโดย Django)
    name = models.CharField(max_length=100)

    # ข้อความยาวไม่จำกัด: ไม่ต้องระบุ max_length
    description = models.TextField()

    # จำนวนเต็ม: รับค่าติดลบได้
    stock_change = models.IntegerField()

    # จำนวนเต็มบวกเท่านั้น (รวม 0): เหมาะกับตัวนับ
    view_count = models.PositiveIntegerField(default=0)

    # ทศนิยมแม่นยำสูง: ใช้กับเงินเสมอ
    # max_digits=10 → เก็บได้สูงสุด 10 หลักรวมกัน (ทั้งก่อนและหลังจุด)
    # decimal_places=2 → ทศนิยม 2 ตำแหน่ง เช่น 99999999.99
    price = models.DecimalField(max_digits=10, decimal_places=2)

    # ทศนิยมทั่วไป: ใช้กับค่าที่ไม่ใช่เงินและยอมรับความคลาดเคลื่อนได้
    average_rating = models.FloatField(default=0.0)

    # true/false
    is_active = models.BooleanField(default=True)

    # วันที่อย่างเดียว ไม่มีเวลา
    birth_date = models.DateField(null=True, blank=True)

    # วันที่ + เวลา: ใช้ auto_now_add ให้ Django เติมค่าตอนสร้างครั้งแรกอัตโนมัติ
    created_at = models.DateTimeField(auto_now_add=True)

    # วันที่ + เวลา: ใช้ auto_now ให้ Django อัปเดตค่าทุกครั้งที่ save()
    updated_at = models.DateTimeField(auto_now=True)

    # อีเมล: validate รูปแบบให้อัตโนมัติ
    contact_email = models.EmailField(max_length=254)

    # URL: validate รูปแบบให้อัตโนมัติ
    website = models.URLField(max_length=200, blank=True)

    # slug: ใช้กับ URL (ทบทวนจาก Part 006-007)
    slug = models.SlugField(max_length=220, unique=True)

    # รูปภาพ: ต้องตั้งค่า MEDIA_ROOT/MEDIA_URL ไว้แล้ว (Part 009) และติดตั้ง Pillow
    cover_image = models.ImageField(upload_to='covers/', blank=True, null=True)

    # ไฟล์ทั่วไป
    attachment = models.FileField(upload_to='attachments/', blank=True, null=True)
```

> **หมายเหตุเรื่อง `ImageField`**: Field นี้ต้องพึ่งพา library ภายนอกชื่อ **Pillow**
> (`pip install Pillow`) เพื่อตรวจสอบว่าไฟล์ที่อัปโหลดเป็นรูปภาพจริง ถ้าไม่ติดตั้ง Pillow
> Django จะ raise error ทันทีตอนรัน `makemigrations` พร้อมข้อความแจ้งให้ติดตั้ง — เรื่องนี้
> เคยพูดถึงสั้น ๆ แล้วใน Part 009 (Static/Media Files) และจะกลับมาเจาะลึกการอัปโหลด
> รูปภาพแบบเต็มรูปแบบใน Part 058

### 101.5 `DecimalField` vs `FloatField`: ทำไมห้ามใช้ `FloatField` กับเงินเด็ดขาด

นี่คือหนึ่งในความผิดพลาดที่ร้ายแรงที่สุดที่มือใหม่ (และบางครั้งแม้แต่มือโปร) ทำพลาด
ลองดูตัวอย่างปัญหาของ floating-point ใน Python เอง (ไม่เกี่ยวกับ Django โดยตรง แต่เป็น
ธรรมชาติของการเก็บเลขทศนิยมแบบ binary floating-point ในทุกภาษาโปรแกรม):

```python
>>> 0.1 + 0.2
0.30000000000000004   # ไม่ใช่ 0.3 พอดี!
```

ถ้าคุณเก็บราคาสินค้าเป็น `FloatField` แล้วบวกราคาหลาย ๆ รายการเข้าด้วยกันในระบบตะกร้าสินค้า
ความคลาดเคลื่อนระดับทศนิยมปลาย ๆ นี้จะสะสมจนยอดรวมผิดเพี้ยนไปจากความเป็นจริง ซึ่ง
**ยอมรับไม่ได้เลย** ในระบบการเงิน `DecimalField` แก้ปัญหานี้โดยเก็บตัวเลขแบบ
**fixed-point decimal** ที่แม่นยำ 100% ตามจำนวนตำแหน่งทศนิยมที่กำหนดไว้

| ประเด็น | `DecimalField` | `FloatField` |
|---|---|---|
| ความแม่นยำ | แม่นยำสัมบูรณ์ (fixed-point) | มีโอกาสคลาดเคลื่อนระดับ binary |
| ใช้กับเงิน/ราคา | ✅ ต้องใช้เสมอ | ❌ ห้ามใช้เด็ดขาด |
| ใช้กับพิกัด GPS, ค่าเฉลี่ยทางสถิติ | ใช้ได้แต่เกินความจำเป็น | ✅ เหมาะสมกว่า |
| Python type ที่ได้กลับมา | `decimal.Decimal` | `float` |
| พารามิเตอร์บังคับ | `max_digits`, `decimal_places` | ไม่มี |

```python
from decimal import Decimal

# เมื่ออ่านค่าจาก DecimalField กลับมา จะได้ type เป็น Decimal ไม่ใช่ float
product.price = Decimal('299.50')   # แนะนำให้สร้างด้วย string เสมอ ไม่ใช่ Decimal(299.50)
```

> **กฎเหล็ก**: เวลาสร้าง `Decimal` ใน Python ให้สร้างจาก **string** เสมอ
> (`Decimal('299.50')`) ห้ามสร้างจาก float (`Decimal(299.50)`) เพราะ float ที่ส่งเข้าไป
> อาจมีความคลาดเคลื่อนติดมาอยู่แล้วตั้งแต่ก่อนแปลงเป็น Decimal

---

## ขั้นตอนที่ 102: Field Options เจาะลึก: `null`, `blank`, `default`, `unique`, `choices`, `help_text`, `verbose_name`

Field Type บอกว่า "เก็บข้อมูลชนิดไหน" ส่วน **Field Options** (คำสั่งเสริมที่ใส่ในวงเล็บ)
บอกว่า "field นี้มีพฤติกรรมพิเศษอะไรบ้าง" ขั้นตอนนี้จะพาไปดู option ที่ใช้บ่อยที่สุด

### 102.1 `null=True` vs `blank=True`: ความต่างสำคัญที่มือใหม่สับสนตลอด

นี่คือคู่ option ที่มือใหม่ **เกือบทุกคน** สับสนในช่วงแรก เพราะชื่อฟังดูคล้ายกันมาก
แต่ทำงานคนละชั้นกันโดยสิ้นเชิง:

| Option | ทำงานที่ชั้นไหน | ความหมาย |
|---|---|---|
| `null=True` | **ฐานข้อมูล (Database)** | อนุญาตให้คอลัมน์นี้เก็บค่า `NULL` ในฐานข้อมูลได้ |
| `blank=True` | **Validation (Forms/Admin)** | อนุญาตให้ฟอร์ม/Admin ส่งค่าเป็นค่าว่างมาได้โดยไม่ error |

พูดง่าย ๆ คือ **`null` คุยกับฐานข้อมูล ส่วน `blank` คุยกับฟอร์ม** ทั้งสองตัวเป็นคนละเรื่อง
กันโดยสิ้นเชิง และสามารถผสมกันได้ 4 แบบ:

```python
from django.db import models


class OptionDemo(models.Model):
    # แบบที่ 1: บังคับกรอกทั้งฟอร์มและฐานข้อมูล (ค่าเริ่มต้นของทุก field ถ้าไม่ระบุอะไรเลย)
    required_field = models.CharField(max_length=100)

    # แบบที่ 2: ฟอร์มเว้นว่างได้ แต่ฐานข้อมูลยังห้าม NULL (จะถูกเก็บเป็น string ว่าง '' แทน)
    # → นี่คือรูปแบบที่แนะนำที่สุดสำหรับ CharField/TextField ที่ไม่บังคับกรอก
    optional_text = models.CharField(max_length=100, blank=True)

    # แบบที่ 3: ฐานข้อมูลอนุญาต NULL แต่ฟอร์มยังบังคับกรอก (พบน้อยมาก แทบไม่มีเหตุผลใช้)
    weird_combo = models.IntegerField(null=True)

    # แบบที่ 4: อนุญาตทั้งฟอร์มและฐานข้อมูล (เหมาะกับ field ที่ไม่ใช่ข้อความ เช่น
    # DateField, ForeignKey, IntegerField ที่ไม่บังคับกรอกจริง ๆ)
    optional_date = models.DateField(null=True, blank=True)
```

### 102.2 กฎเหล็กของ Django เอง: อย่าใช้ `null=True` กับ `CharField`/`TextField`

เอกสารทางการของ Django ระบุคำแนะนำนี้ไว้ชัดเจน และเป็นกฎที่หลักสูตรนี้ยึดถือตลอดทุก Part:

> **"Avoid using `null` on string-based fields such as `CharField` and `TextField`."**
> — Django Documentation

เหตุผลคือถ้าอนุญาตทั้ง `null=True` และ `blank=True` กับ field ประเภทข้อความ จะเกิด
**สถานะ "ไม่มีค่า" ถึง 2 แบบพร้อมกัน** ในฐานข้อมูล: ค่า `NULL` และค่า string ว่าง `''`
ซึ่งทั้งสองมีความหมายเหมือนกันในทางปฏิบัติ (ไม่มีข้อมูล) แต่ query ต้องเช็คทั้งสองแบบ
แยกกัน (`field__isnull=True` กับ `field=''`) ทำให้โค้ดซับซ้อนขึ้นโดยไม่จำเป็น

```python
# ❌ ไม่แนะนำ: เปิดช่องให้มีทั้ง NULL และ '' ปนกันในคอลัมน์เดียว
excerpt = models.CharField(max_length=300, null=True, blank=True)

# ✅ แนะนำ: มีสถานะ "ไม่มีค่า" เพียงแบบเดียวคือ string ว่าง ''
excerpt = models.CharField(max_length=300, blank=True)
```

สำหรับ field ที่ไม่ใช่ข้อความ (`DateField`, `IntegerField`, `ForeignKey`, `DecimalField`
ฯลฯ) การใช้ `null=True` ร่วมกับ `blank=True` เป็นเรื่องปกติและจำเป็น เพราะ field เหล่านี้
ไม่มี "ค่าว่าง" ในตัวเองแบบ string (จะเก็บ "ไม่มีวันที่" เป็นอะไรถ้าไม่ใช่ `NULL`)

### 102.3 `default`: ค่าเริ่มต้นเมื่อไม่ได้ระบุค่า

```python
class Post(models.Model):
    view_count = models.PositiveIntegerField(default=0)
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)   # ไม่ต้องใช้ default เพราะมี auto_now_add แล้ว
```

`default` รับได้ทั้งค่าคงที่และ **callable** (ฟังก์ชันที่ไม่มี argument) ซึ่งจะถูกเรียก
ทุกครั้งที่สร้าง instance ใหม่:

```python
import uuid
from django.utils import timezone


class OptionDemo(models.Model):
    # ค่าคงที่: ทุก instance ใหม่ได้ค่าเดียวกัน
    status = models.CharField(max_length=20, default='draft')

    # callable: เรียกฟังก์ชันใหม่ทุกครั้ง (ไม่ใส่วงเล็บ! ส่งฟังก์ชันไปตรง ๆ)
    reference_code = models.UUIDField(default=uuid.uuid4)
    published_at = models.DateTimeField(default=timezone.now)
```

> **ข้อควรระวังร้ายแรง**: `default=timezone.now` (ไม่มีวงเล็บ) ถูกต้อง เพราะส่งตัว
> ฟังก์ชันไปให้ Django เรียกเองทุกครั้งที่ต้องการค่า แต่ `default=timezone.now()`
> (มีวงเล็บ) จะ **เรียกฟังก์ชันครั้งเดียวตอนโหลดไฟล์ `models.py`** แล้วใช้ค่าคงที่
> เวลานั้นซ้ำกับทุก instance ตลอดไป — เป็นข้อผิดพลาดที่พบบ่อยและตรวจจับยากมาก
> เพราะโค้ดไม่ error แต่ logic ผิดทั้งหมด

### 102.4 `unique=True`: ห้ามมีค่าซ้ำในคอลัมน์นี้

```python
class Post(models.Model):
    slug = models.SlugField(max_length=220, unique=True)
```

Django จะสร้าง **UNIQUE constraint** ระดับฐานข้อมูลให้อัตโนมัติ หมายความว่าต่อให้มีคน
พยายามบันทึกข้อมูลซ้ำผ่านช่องทางอื่นที่ไม่ใช่ Django (เช่น query SQL ตรง ๆ) ฐานข้อมูล
จะปฏิเสธทันที นี่คือการป้องกันในชั้นที่แข็งแกร่งที่สุด (แข็งแกร่งกว่าการ validate ใน
Python code เท่านั้น ซึ่งสามารถถูกข้ามได้ถ้ามีทางเข้าถึงข้อมูลหลายช่องทาง)

เมื่อพยายามบันทึกค่าซ้ำผ่าน Django ORM จะได้ `django.db.utils.IntegrityError`:

```python
>>> Post.objects.create(title="A", slug="hello-world")
>>> Post.objects.create(title="B", slug="hello-world")
Traceback (most recent call last):
    ...
django.db.utils.IntegrityError: UNIQUE constraint failed: blog_post.slug
```

### 102.5 `choices`: จำกัดค่าที่เป็นไปได้ให้อยู่ในชุดที่กำหนด

รูปแบบเก่า (ก่อน Django 3.0) เขียน choices เป็น list ของ tuple ตรง ๆ:

```python
# รูปแบบเก่า - ยังใช้งานได้ปกติ แต่หลักสูตรนี้แนะนำรูปแบบใหม่ด้านล่างแทน
STATUS_CHOICES = [
    ('draft', 'ฉบับร่าง'),
    ('published', 'เผยแพร่แล้ว'),
    ('archived', 'เก็บถาวร'),
]
status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='draft')
```

ตั้งแต่ **Django 3.0** เป็นต้นไป มีวิธีที่สะอาดกว่าและเป็นมาตรฐานของหลักสูตรนี้ นั่นคือ
ใช้ **`models.TextChoices`** (สำหรับค่าที่เป็นข้อความ) และ **`models.IntegerChoices`**
(สำหรับค่าที่เป็นตัวเลข) ซึ่งเป็น subclass ของ Python `enum.Enum`:

```python
from django.db import models


class Post(models.Model):
    class ContentType(models.TextChoices):
        ARTICLE = 'ARTICLE', 'บทความ'
        TUTORIAL = 'TUTORIAL', 'สอนการใช้งาน'
        NEWS = 'NEWS', 'ข่าวสาร'
        ANNOUNCEMENT = 'ANNOUNCEMENT', 'ประกาศ'

    content_type = models.CharField(
        max_length=20,
        choices=ContentType.choices,
        default=ContentType.ARTICLE,
    )
```

ข้อดีของการประกาศ choices เป็น class ที่ซ้อนอยู่ใน model (nested class) แบบนี้:

1. **อ่านง่าย จัดกลุ่มชัดเจน**: choices ของ field ไหน อยู่ติดกับ model นั้นเสมอ
2. **Autocomplete ใน editor ทำงานได้**: พิมพ์ `Post.ContentType.` แล้ว VS Code จะ
   แสดงตัวเลือกทั้งหมดให้ทันที ต่างจาก string ธรรมดาที่พิมพ์ผิดแล้วไม่มีใครเตือน
3. **ใช้ค่าคงที่แทน string ตรง ๆ ได้ทุกที่ในโค้ด** ลดโอกาสพิมพ์ผิด:

```python
# แทนที่จะเขียน string เปล่า ๆ ซึ่งพิมพ์ผิดได้ง่ายและ IDE ช่วยไม่ได้
post = Post.objects.create(title="ทดสอบ", content_type='ARTICLE')   # ❌ เสี่ยงพิมพ์ผิด

# เขียนแบบนี้แทน — ปลอดภัยกว่า, autocomplete ช่วยได้, refactor ง่ายกว่า
post = Post.objects.create(title="ทดสอบ", content_type=Post.ContentType.ARTICLE)   # ✅
```

`TextChoices`/`IntegerChoices` มี attribute พิเศษที่ใช้บ่อยให้เรียกดูได้ทันที:

```python
>>> Post.ContentType.choices
[('ARTICLE', 'บทความ'), ('TUTORIAL', 'สอนการใช้งาน'), ('NEWS', 'ข่าวสาร'), ('ANNOUNCEMENT', 'ประกาศ')]
>>> Post.ContentType.values
['ARTICLE', 'TUTORIAL', 'NEWS', 'ANNOUNCEMENT']
>>> Post.ContentType.labels
['บทความ', 'สอนการใช้งาน', 'ข่าวสาร', 'ประกาศ']
>>> Post.ContentType.names
['ARTICLE', 'TUTORIAL', 'NEWS', 'ANNOUNCEMENT']
>>> Post.ContentType.ARTICLE
'ARTICLE'
>>> Post.ContentType.ARTICLE.label
'บทความ'
```

และเมื่อมี instance จริง Django จะสร้าง method `get_<field_name>_display()` ให้อัตโนมัติ
ทุก field ที่มี `choices` เสมอ (ไม่ต้องเขียนเอง) เพื่อแปลงค่าที่เก็บในฐานข้อมูลกลับเป็น
label ที่อ่านง่ายสำหรับแสดงผล:

```python
>>> post = Post.objects.create(title="ทดสอบ", content_type=Post.ContentType.TUTORIAL)
>>> post.content_type
'TUTORIAL'
>>> post.get_content_type_display()
'สอนการใช้งาน'
```

ตัวอย่าง `IntegerChoices` (ใช้เมื่อค่าที่เก็บควรเป็นตัวเลข เช่น ระดับความสำคัญ):

```python
class Post(models.Model):
    class Priority(models.IntegerChoices):
        LOW = 1, 'ต่ำ'
        NORMAL = 2, 'ปกติ'
        HIGH = 3, 'สูง'
        URGENT = 4, 'ด่วนมาก'

    priority = models.PositiveSmallIntegerField(
        choices=Priority.choices,
        default=Priority.NORMAL,
    )
```

### 102.6 `help_text` และ `verbose_name`: ทำให้ Django Admin เป็นมิตรกับผู้ใช้

ทั้งสอง option นี้**ไม่ส่งผลต่อฐานข้อมูลเลยแม้แต่น้อย** เป็นเรื่องของการแสดงผลใน
Django Admin (Part 017-018) และ Django Forms (Part 025) เท่านั้น:

```python
class Post(models.Model):
    title = models.CharField(
        max_length=200,
        verbose_name='หัวข้อบทความ',            # ป้ายกำกับที่แสดงแทนชื่อ field
        help_text='ควรมีความยาว 10-60 ตัวอักษร เพื่อผล SEO ที่ดี',   # คำอธิบายใต้ช่องกรอก
    )
```

| Option | ผลลัพธ์ใน Django Admin | ผลลัพธ์ในฐานข้อมูล |
|---|---|---|
| `verbose_name` | เปลี่ยน label ของช่องกรอกจาก `Title` เป็น `หัวข้อบทความ` | ไม่มีผลใด ๆ |
| `help_text` | แสดงข้อความอธิบายสีเทาใต้ช่องกรอก | ไม่มีผลใด ๆ |

ถ้าไม่ระบุ `verbose_name` Django จะสร้างให้อัตโนมัติจากชื่อ field โดยแทนที่ `_` ด้วย
ช่องว่างและขึ้นต้นด้วยตัวพิมพ์เล็ก เช่น field ชื่อ `created_at` จะกลายเป็น
`created at` โดยอัตโนมัติในหน้า Admin (ถ้าไม่ได้ตั้งชื่อภาษาไทยเอง)

### 102.7 ตารางสรุป Field Options ทั้งหมดในขั้นตอนนี้

| Option | ทำงานที่ชั้น | ใช้เมื่อ | ค่า default ถ้าไม่ระบุ |
|---|---|---|---|
| `null` | Database | field ที่ไม่ใช่ข้อความ ต้องการเก็บ "ไม่มีค่า" เป็น `NULL` | `False` |
| `blank` | Validation (Forms/Admin) | field ไม่บังคับกรอก | `False` |
| `default` | ทั้งสองชั้น | ต้องการค่าเริ่มต้นเมื่อไม่ได้ระบุ | ไม่มี (field เป็น required ถ้าไม่มี default และไม่ blank) |
| `unique` | Database + Validation | field ต้องไม่ซ้ำกันทั้งตาราง | `False` |
| `choices` | Validation + Admin widget | จำกัดค่าที่เป็นไปได้ | ไม่มี |
| `help_text` | Admin/Forms (UI) | ต้องการอธิบายเพิ่มเติมให้ผู้กรอกข้อมูล | string ว่าง |
| `verbose_name` | Admin/Forms (UI) | ต้องการ label ที่อ่านง่ายกว่าชื่อ field | สร้างอัตโนมัติจากชื่อ field |

---

## ขั้นตอนที่ 103: ขยาย `Post` Model จาก Part 007 ให้สมบูรณ์แบบมืออาชีพ

### 103.1 ทบทวน `Post` model เดิมจาก Part 007

```python
# blog/models.py (เวอร์ชันจาก Part 007)
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

Model นี้ใช้งานได้จริงและเพียงพอสำหรับ Phase 1 แต่ยังขาดหลายอย่างที่บล็อกระดับมืออาชีพ
ควรมี ต่อไปนี้คือ field ที่เราจะเพิ่มเข้าไป พร้อมเหตุผลของแต่ละตัว

### 103.2 Field ใหม่ที่จะเพิ่ม พร้อมเหตุผล

| Field ใหม่ | ชนิด | เหตุผลที่ต้องมี |
|---|---|---|
| `excerpt` | `CharField(max_length=300, blank=True)` | หน้ารายการบทความไม่ควรแสดงเนื้อหาเต็มทั้งหมด (ช้า, SEO ไม่ดี, UX แย่) ต้องมีสรุปย่อแยกต่างหาก |
| `content_type` | `CharField` + `TextChoices` | บล็อกจริงมักมีเนื้อหาหลายประเภท (บทความ, สอนการใช้งาน, ข่าว, ประกาศ) การกรอง/แสดงผลตามประเภทเป็นฟีเจอร์พื้นฐานที่ขาดไม่ได้ |
| `is_premium` | `BooleanField(default=False)` | รองรับโมเดลธุรกิจแบบ "บทความพรีเมียม" ที่ต้องจ่ายเงินอ่าน (เหมือน Medium Member-only stories) |
| `price` | `DecimalField(max_digits=6, decimal_places=2, default=0)` | ราคาของบทความพรีเมียม ต้องใช้ `DecimalField` เพราะเกี่ยวกับเงิน (ตามเหตุผลในขั้นตอนที่ 101.5) |
| `view_count` | `PositiveIntegerField(default=0, editable=False)` | นับจำนวนครั้งที่มีคนเข้าดูบทความ ใช้จัดอันดับ "บทความยอดนิยม" ได้ในอนาคต |
| `reading_time_minutes` | `PositiveSmallIntegerField(default=1)` | บอกผู้อ่านล่วงหน้าว่าบทความนี้ใช้เวลาอ่านกี่นาที (ฟีเจอร์ UX ที่พบในบล็อกมืออาชีพแทบทุกที่) |

Field เดิมทั้งหมด (`title`, `slug`, `content`, `created_at`, `updated_at`,
`is_published`) **ยังคงอยู่เหมือนเดิมทุกประการ** เพื่อไม่ให้ view/template ที่เขียนไว้
ตั้งแต่ Part 007-010 พัง

### 103.3 `Post` Model เวอร์ชันสมบูรณ์แบบมืออาชีพ

```python
# blog/models.py (เวอร์ชันสมบูรณ์ - Part 011)
from django.db import models
from django.urls import reverse
from django.utils.text import slugify


class Post(models.Model):
    class ContentType(models.TextChoices):
        ARTICLE = 'ARTICLE', 'บทความ'
        TUTORIAL = 'TUTORIAL', 'สอนการใช้งาน'
        NEWS = 'NEWS', 'ข่าวสาร'
        ANNOUNCEMENT = 'ANNOUNCEMENT', 'ประกาศ'

    # --- Field เดิมจาก Part 007 (ไม่เปลี่ยนแปลง) ---
    title = models.CharField(max_length=200, verbose_name='หัวข้อบทความ')
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    content = models.TextField(verbose_name='เนื้อหา')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False, verbose_name='เผยแพร่แล้ว')

    # --- Field ใหม่ที่เพิ่มใน Part 011 ---
    excerpt = models.CharField(
        max_length=300,
        blank=True,
        verbose_name='สรุปย่อ',
        help_text='สรุปสั้น ๆ สำหรับหน้ารายการบทความ ถ้าไม่กรอกจะตัดมาจากเนื้อหาอัตโนมัติ',
    )
    content_type = models.CharField(
        max_length=20,
        choices=ContentType.choices,
        default=ContentType.ARTICLE,
        verbose_name='ประเภทเนื้อหา',
    )
    is_premium = models.BooleanField(default=False, verbose_name='บทความพรีเมียม')
    price = models.DecimalField(
        max_digits=6,
        decimal_places=2,
        default=0,
        verbose_name='ราคา (บาท)',
        help_text='ใช้กับบทความพรีเมียมเท่านั้น 0 หมายถึงอ่านฟรี',
    )
    view_count = models.PositiveIntegerField(default=0, editable=False, verbose_name='จำนวนเข้าชม')
    reading_time_minutes = models.PositiveSmallIntegerField(
        default=1,
        verbose_name='เวลาอ่าน (นาที)',
    )

    class Meta:
        ordering = ['-created_at']
        verbose_name = 'บทความ'
        verbose_name_plural = 'บทความทั้งหมด'

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        if not self.excerpt and self.content:
            self.excerpt = self.content[:297].rsplit(' ', 1)[0] + '...'
        super().save(*args, **kwargs)
```

สังเกตว่าใน `save()` เราเพิ่ม logic ให้สร้าง `excerpt` อัตโนมัติจาก `content` ถ้าผู้เขียน
ไม่ได้กรอกเอง (ตัดที่ 297 ตัวอักษร แล้วตัดคำสุดท้ายที่อาจขาดครึ่งออกด้วย `.rsplit(' ', 1)[0]`
เพื่อไม่ให้คำขาดกลางคัน) นี่คือตัวอย่างของ **custom business logic ใน model** ที่จะ
เจาะลึกเพิ่มเติมในขั้นตอนที่ 106

> **ทำไมใช้ `editable=False` กับ `view_count`?** เพราะค่านี้ควรถูกอัปเดตโดยระบบเท่านั้น
> (ผ่าน method ที่จะเขียนในขั้นตอนที่ 106) ไม่ควรให้ผู้ดูแลระบบมาแก้ไขมือผ่าน Django Admin
> `editable=False` จะซ่อน field นี้ออกจากฟอร์มใน Admin โดยอัตโนมัติ (จะเห็นผลจริงเมื่อ
> เรียน Django Admin ใน Part 017)

---

## ขั้นตอนที่ 104: `makemigrations` เจาะลึก — เปิดไฟล์ Migration จริงมาอ่านทีละบรรทัด

### 104.1 รันคำสั่ง `makemigrations`

หลังแก้ไข `blog/models.py` ตามขั้นตอนที่ 103 แล้ว รันคำสั่ง:

```bash
python manage.py makemigrations blog
```

ผลลัพธ์ที่ได้ควรมีลักษณะประมาณนี้:

```
Migrations for 'blog':
  blog/migrations/0002_post_content_type_post_excerpt_post_is_premium_and_more.py
    - Add field content_type to post
    - Add field excerpt to post
    - Add field is_premium to post
    - Add field price to post
    - Add field reading_time_minutes to post
    - Add field view_count to post
    - Alter field title on post
    - Alter field is_published on post
    - Alter field content on post
```

**สิ่งที่เกิดขึ้นเบื้องหลัง**: `makemigrations` ไม่ได้แตะฐานข้อมูลเลยแม้แต่นิดเดียว
มันแค่ **เปรียบเทียบ** โครงสร้าง model ปัจจุบันในโค้ด กับ "ประวัติ" ที่บันทึกไว้ในไฟล์
migration ก่อนหน้าทั้งหมด (ที่อยู่ในโฟลเดอร์ `blog/migrations/`) แล้วสร้างไฟล์ Python
ใหม่ที่บรรยาย **ความแตกต่าง** ระหว่างสองสถานะนั้นออกมาเป็นโค้ด

### 104.2 เปิดไฟล์ Migration ที่ถูกสร้างขึ้นมาอ่านทีละส่วน

```python
# blog/migrations/0002_post_content_type_post_excerpt_post_is_premium_and_more.py
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ('blog', '0001_initial'),
    ]

    operations = [
        migrations.AddField(
            model_name='post',
            name='content_type',
            field=models.CharField(
                choices=[
                    ('ARTICLE', 'บทความ'),
                    ('TUTORIAL', 'สอนการใช้งาน'),
                    ('NEWS', 'ข่าวสาร'),
                    ('ANNOUNCEMENT', 'ประกาศ'),
                ],
                default='ARTICLE',
                max_length=20,
                verbose_name='ประเภทเนื้อหา',
            ),
        ),
        migrations.AddField(
            model_name='post',
            name='excerpt',
            field=models.CharField(
                blank=True,
                help_text='สรุปสั้น ๆ สำหรับหน้ารายการบทความ ถ้าไม่กรอกจะตัดมาจากเนื้อหาอัตโนมัติ',
                max_length=300,
                verbose_name='สรุปย่อ',
            ),
        ),
        migrations.AddField(
            model_name='post',
            name='is_premium',
            field=models.BooleanField(default=False, verbose_name='บทความพรีเมียม'),
        ),
        migrations.AddField(
            model_name='post',
            name='price',
            field=models.DecimalField(
                decimal_places=2,
                default=0,
                help_text='ใช้กับบทความพรีเมียมเท่านั้น 0 หมายถึงอ่านฟรี',
                max_digits=6,
                verbose_name='ราคา (บาท)',
            ),
        ),
        migrations.AddField(
            model_name='post',
            name='reading_time_minutes',
            field=models.PositiveSmallIntegerField(default=1, verbose_name='เวลาอ่าน (นาที)'),
        ),
        migrations.AddField(
            model_name='post',
            name='view_count',
            field=models.PositiveIntegerField(default=0, editable=False, verbose_name='จำนวนเข้าชม'),
        ),
        migrations.AlterField(
            model_name='post',
            name='title',
            field=models.CharField(max_length=200, verbose_name='หัวข้อบทความ'),
        ),
        migrations.AlterField(
            model_name='post',
            name='is_published',
            field=models.BooleanField(default=False, verbose_name='เผยแพร่แล้ว'),
        ),
        migrations.AlterField(
            model_name='post',
            name='content',
            field=models.TextField(verbose_name='เนื้อหา'),
        ),
    ]
```

มาแยกอธิบายทีละส่วน:

#### `class Migration(migrations.Migration)`

ทุกไฟล์ migration คือ Python class ที่สืบทอดจาก `django.db.migrations.Migration`
Django จะสร้าง instance ของ class นี้แล้วเรียกใช้ตอนรัน `migrate` — ตัวไฟล์เองเป็น
**คำสั่งที่บันทึกไว้เป็นโค้ด (declarative record)** ไม่ใช่ log อัตโนมัติของสิ่งที่เกิดขึ้น
มันคือแผนงานที่ Django จะนำไปทำตามเมื่อสั่ง `migrate`

#### `dependencies`

```python
dependencies = [
    ('blog', '0001_initial'),
]
```

บอกว่า migration นี้ **ต้องรันหลัง** migration `0001_initial` ของแอป `blog` เสมอ
`dependencies` คือสิ่งที่สร้าง **Migration Graph** (กราฟความสัมพันธ์ระหว่าง migration
ทั้งหมดในโปรเจกต์) ซึ่งจะเจาะลึกในขั้นตอนที่ 105 — ถ้า `Post` model มี `ForeignKey`
ไปยัง model ของแอปอื่น (เช่น `auth.User`) `dependencies` จะมีรายการของแอปนั้นเพิ่มเข้ามา
ด้วย (เราจะเห็นตัวอย่างจริงใน Part 012)

#### `operations`

คือ **list ของคำสั่ง** ที่ Django จะทำตามลำดับ แต่ละคำสั่งเป็น instance ของ Operation
class ที่ Django เตรียมไว้ให้ ตารางด้านล่างสรุป Operation ที่พบบ่อยที่สุด:

| Operation Class | ความหมาย | เกิดขึ้นเมื่อ |
|---|---|---|
| `AddField` | เพิ่มคอลัมน์ใหม่ | เพิ่ม field ใหม่ใน model |
| `RemoveField` | ลบคอลัมน์ | ลบ field ออกจาก model |
| `AlterField` | เปลี่ยนคุณสมบัติของคอลัมน์ที่มีอยู่แล้ว | เปลี่ยน `max_length`, `verbose_name`, `null`, `default` ฯลฯ |
| `RenameField` | เปลี่ยนชื่อคอลัมน์ (เก็บข้อมูลเดิมไว้) | เปลี่ยนชื่อ field แล้วตอบ "yes" ตอน `makemigrations` ถาม |
| `CreateModel` | สร้างตารางใหม่ทั้งตาราง | สร้าง model class ใหม่ |
| `DeleteModel` | ลบตารางทั้งตาราง | ลบ model class ทิ้ง |
| `AlterModelOptions` | เปลี่ยนค่าใน `Meta` เช่น `ordering` | แก้ไข `Meta` class |
| `AlterUniqueTogether` / `AddConstraint` | เปลี่ยน constraint ระดับตาราง | แก้ไข `unique_together`/`constraints` ใน `Meta` (Part 015) |

สังเกตว่านอกจาก `AddField` 6 ตัว (สำหรับ field ใหม่ทั้งหมด) ยังมี `AlterField` อีก 3 ตัว
สำหรับ `title`, `is_published`, `content` ทั้งที่เราไม่ได้เปลี่ยน `max_length` หรือ
ชนิดข้อมูลของ field เหล่านี้เลย — นี่เป็นเพราะเราเพิ่ม `verbose_name` เข้าไปในขั้นตอนที่
103.3 ซึ่งแม้จะเป็นการเปลี่ยนแปลงเล็กน้อยที่ไม่กระทบฐานข้อมูลจริง (ไม่มีการ `ALTER TABLE`
ที่แท้จริงเกิดขึ้นสำหรับ SQLite/PostgreSQL ในกรณีนี้ เพราะ `verbose_name` เป็น metadata
ระดับ Python เท่านั้น) แต่ Django ก็ยังบันทึกไว้เป็น `AlterField` เพื่อให้ **state ของ
migration ตรงกับ model 100% เสมอ** ไม่มีความคลาดเคลื่อนแม้แต่นิดเดียว

### 104.3 คำสั่งเสริมที่ควรรู้คู่กับ `makemigrations`

```bash
# ดูว่าจะเกิด migration อะไรบ้าง โดยไม่สร้างไฟล์จริง (dry run)
python manage.py makemigrations --dry-run --verbosity 3

# ตั้งชื่อไฟล์ migration เอง แทนชื่อยาว ๆ ที่ Django สร้างอัตโนมัติ
python manage.py makemigrations blog --name add_post_professional_fields

# ตรวจสอบว่ามี model ไหนที่ยังไม่ได้สร้าง migration ตกหล่นอยู่หรือไม่ (ใช้ใน CI/CD)
python manage.py makemigrations --check --dry-run
```

คำสั่งสุดท้าย (`--check --dry-run`) สำคัญมากในระดับทีมงานมืออาชีพ เพราะใช้ตรวจจับ
กรณีที่นักพัฒนาแก้ไข `models.py` แล้ว **ลืม** รัน `makemigrations` ก่อน commit โค้ด —
ถ้ามีความเปลี่ยนแปลงที่ยังไม่มี migration รองรับ คำสั่งนี้จะ exit ด้วย status code
ที่ไม่ใช่ 0 ทำให้ CI/CD pipeline (Part 088) หยุดและแจ้งเตือนได้ทันที ก่อนที่ปัญหาจะไป
ถึงขั้น deploy จริง

---

## ขั้นตอนที่ 105: `migrate` เจาะลึก — Migration Graph, `showmigrations`, `sqlmigrate`

### 105.1 `migrate` ทำอะไรกันแน่

ถ้า `makemigrations` คือการ "เขียนแผนงาน" `migrate` คือการ **"ลงมือทำตามแผนงานนั้นจริง"**
บนฐานข้อมูล:

```bash
python manage.py migrate blog
```

```
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0002_post_content_type_post_excerpt_post_is_premium_and_more... OK
```

เมื่อรันสำเร็จ Django จะบันทึกไว้ในตารางพิเศษชื่อ **`django_migrations`** ว่า migration
ไหนถูก apply ไปแล้วบ้าง ตารางนี้คือ "สมุดบันทึกประวัติ" ที่ Django ใช้ตัดสินใจว่าครั้ง
ต่อไปที่รัน `migrate` ต้อง apply migration ตัวไหนเพิ่มบ้าง (ไม่ apply ซ้ำของเดิม)

```bash
# ดูตาราง django_migrations ผ่าน shell (ตัวอย่างสำหรับ SQLite)
python manage.py dbshell
sqlite> SELECT app, name, applied FROM django_migrations ORDER BY id;
```

### 105.2 Migration Graph คืออะไร

Django มองไฟล์ migration ทั้งหมดในโปรเจกต์เป็น **directed acyclic graph (DAG)** —
แต่ละ migration คือ 1 node และ `dependencies` คือเส้นเชื่อม (edge) ที่บอกว่า node ไหน
ต้องมาก่อน node ไหน

```
blog: 0001_initial ──> 0002_post_content_type_... ──> (migration ถัดไปในอนาคต)
                              ▲
                              │ (ถ้ามี ForeignKey ไปยัง auth.User)
auth: 0001_initial ──> 0002_... ──> ... ──> 0012_alter_user_first_name_max_length
```

เมื่อรัน `python manage.py migrate` (ไม่ระบุชื่อแอป) Django จะ:

1. โหลดไฟล์ migration ทั้งหมดจากทุกแอปที่อยู่ใน `INSTALLED_APPS`
2. สร้าง graph จาก `dependencies` ของทุกไฟล์
3. หา **leaf nodes** (migration ล่าสุดของแต่ละสาย ที่ไม่มี migration อื่นพึ่งพามันอีก)
4. เรียง **topological order** (ลำดับที่เคารพทุก dependency) แล้ว apply ทีละตัวตามลำดับ
   นั้น ข้ามตัวที่ apply ไปแล้ว

การออกแบบเป็น graph (ไม่ใช่ list เรียงตัวเลขธรรมดา) ทำให้ Django รองรับสถานการณ์ที่
หลายแอปมี migration ขึ้นต่อกันแบบซับซ้อนได้อย่างถูกต้อง แม้จะมาจากคนละทีม คนละเวลา
คนละลำดับการ merge ก็ตาม

### 105.3 `showmigrations`: ดูสถานะ migration ทั้งหมดในโปรเจกต์

```bash
python manage.py showmigrations
```

```
admin
 [X] 0001_initial
 [X] 0002_logentry_remove_auto_add
 [X] 0003_logentry_add_action_flag_choices
auth
 [X] 0001_initial
 [X] 0002_alter_permission_name_max_length
 ...
blog
 [X] 0001_initial
 [ ] 0002_post_content_type_post_excerpt_post_is_premium_and_more
contenttypes
 [X] 0001_initial
sessions
 [X] 0001_initial
```

`[X]` หมายถึง apply แล้ว, `[ ]` หมายถึงยังไม่ได้ apply — ตัวอย่างข้างต้นแสดงสถานะ
**ก่อน** รัน `migrate` ของ migration `0002` ที่เพิ่งสร้าง ทำให้เห็นชัดว่าต้องรัน
`migrate` เพิ่มก่อนไปทดสอบจริง

ดูเฉพาะแอปเดียวและดู dependency แบบละเอียด:

```bash
python manage.py showmigrations blog --plan
```

```
[X]  blog.0001_initial
[ ]  blog.0002_post_content_type_post_excerpt_post_is_premium_and_more
```

`--plan` จะแสดงลำดับการ apply จริงตาม topological order ของ graph ทั้งหมด (รวมทุกแอป
ที่เกี่ยวข้อง) ไม่ใช่แค่เรียงตามชื่อไฟล์เฉย ๆ

### 105.4 `sqlmigrate`: ดู SQL จริงที่จะถูกรัน โดยไม่ต้องรันจริง

นี่คือคำสั่งที่ทรงพลังที่สุดสำหรับการเรียนรู้ว่า Django ORM แปลงเป็น SQL อย่างไร และ
สำคัญมากในงานจริงเมื่อต้องตรวจสอบก่อน deploy บน production:

```bash
python manage.py sqlmigrate blog 0002
```

ผลลัพธ์ (ตัวอย่างสำหรับ SQLite — PostgreSQL จะได้ syntax ที่ต่างออกไปเล็กน้อย):

```sql
BEGIN;
--
-- Add field content_type to post
--
ALTER TABLE "blog_post" ADD COLUMN "content_type" varchar(20) DEFAULT 'ARTICLE' NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "content_type" DROP DEFAULT;
--
-- Add field excerpt to post
--
ALTER TABLE "blog_post" ADD COLUMN "excerpt" varchar(300) DEFAULT '' NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "excerpt" DROP DEFAULT;
--
-- Add field is_premium to post
--
ALTER TABLE "blog_post" ADD COLUMN "is_premium" bool DEFAULT 0 NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "is_premium" DROP DEFAULT;
--
-- Add field price to post
--
ALTER TABLE "blog_post" ADD COLUMN "price" decimal DEFAULT 0 NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "price" DROP DEFAULT;
--
-- Add field reading_time_minutes to post
--
ALTER TABLE "blog_post" ADD COLUMN "reading_time_minutes" smallint unsigned DEFAULT 1 NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "reading_time_minutes" DROP DEFAULT;
--
-- Add field view_count to post
--
ALTER TABLE "blog_post" ADD COLUMN "view_count" integer unsigned DEFAULT 0 NOT NULL;
ALTER TABLE "blog_post" ALTER COLUMN "view_count" DROP DEFAULT;
COMMIT;
```

สังเกตรูปแบบที่เกิดซ้ำ ๆ: `ADD COLUMN ... DEFAULT <ค่า> NOT NULL` ตามด้วย
`ALTER COLUMN ... DROP DEFAULT` ทันที — Django ทำแบบนี้เพื่อให้แถวข้อมูลเดิมที่มีอยู่
ในตารางได้ค่า default ไปเติมในคอลัมน์ใหม่ก่อน (ไม่เช่นนั้นจะ error เพราะคอลัมน์เป็น
`NOT NULL`) จากนั้นจึง "ถอด" default ออกจากคำนิยามคอลัมน์ในระดับฐานข้อมูล เพื่อให้
พฤติกรรม default ที่แท้จริงถูกควบคุมจากฝั่ง Django (`models.py`) เพียงจุดเดียวเท่านั้น
ไม่ใช่ทั้งจากฐานข้อมูลและจาก Python พร้อมกัน (ตาม DRY Principle ที่เรียนใน Part 001)

ทั้งหมดถูกห่อด้วย `BEGIN;` ... `COMMIT;` ทำให้เป็น **transaction เดียว** — ถ้าคำสั่งใด
คำสั่งหนึ่งล้มเหลวกลางทาง ฐานข้อมูลจะ rollback กลับสู่สถานะก่อนเริ่ม `migrate` ทั้งหมด
โดยอัตโนมัติ ไม่ทิ้งฐานข้อมูลไว้ในสถานะครึ่ง ๆ กลาง ๆ (ข้อดีนี้ใช้ได้เฉพาะฐานข้อมูลที่
รองรับ transactional DDL เช่น PostgreSQL และ SQLite เต็มรูปแบบ ส่วน MySQL รองรับ
transactional DDL แบบจำกัดกว่า)

### 105.5 คำสั่ง Migration เสริมที่ควรรู้ไว้

| คำสั่ง | ความหมาย |
|---|---|
| `python manage.py migrate blog 0001` | ย้อนกลับ (rollback) ไปที่ migration `0001` เท่านั้น (ถอย migration ที่ apply หลังจากนั้นออก) |
| `python manage.py migrate blog zero` | ย้อนกลับทั้งหมดของแอปนั้น (ลบทุกตารางที่แอปนั้นสร้าง) |
| `python manage.py migrate --fake blog 0002` | บันทึกว่า apply แล้วโดย **ไม่รัน SQL จริง** (ใช้เมื่อสร้างตารางเองมือแล้ว หรือ sync สถานะ) |
| `python manage.py migrate --fake-initial` | ใช้ตอนนำโปรเจกต์ไปต่อกับฐานข้อมูลที่มีตารางอยู่แล้วตรงกับ migration พอดี |
| `python manage.py migrate --plan` | แสดงลำดับ migration ที่จะ apply โดยไม่รันจริง (คล้าย `--dry-run` ของ `makemigrations`) |

> **คำเตือนสำคัญ**: `migrate --fake` เป็นเครื่องมือขั้นสูงที่อันตรายถ้าใช้ผิดจังหวะ
> เพราะทำให้ Django "เชื่อ" ว่า migration ถูก apply แล้วทั้งที่โครงสร้างตารางจริงอาจไม่
> ตรงกัน ควรใช้เฉพาะกรณีที่เข้าใจสถานการณ์ชัดเจนจริง ๆ เท่านั้น เราจะกลับมาพูดถึงกรณี
> การใช้งานจริงของ `--fake` ในสถานการณ์ระดับ production ที่ Part 016

---

## ขั้นตอนที่ 106: Model Methods: `__str__`, `get_absolute_url()`, Custom Business-Logic Methods

Model ใน Django ไม่ได้เป็นแค่ "โครงสร้างข้อมูล" เฉย ๆ แต่เป็น Python class เต็มรูปแบบ
ที่สามารถมี method ได้ตามต้องการ — นี่คือหัวใจของแนวคิด **"Fat Models, Thin Views"**
ที่ทีมงานมืออาชีพยึดถือ: ตรรกะทางธุรกิจ (business logic) ที่เกี่ยวกับข้อมูลของ model
ควรอยู่ **ใน model** ไม่ใช่กระจัดกระจายอยู่ใน view หลาย ๆ ที่

### 106.1 `__str__`: ทบทวนสิ่งที่เห็นมาตั้งแต่ Part 005

```python
def __str__(self):
    return self.title
```

`__str__` เป็น **Python dunder method มาตรฐาน** (ไม่ใช่ของ Django โดยเฉพาะ) ที่กำหนดว่า
เมื่อแปลง object เป็น string (เช่นด้วย `str(obj)`, `print(obj)`, หรือ f-string) จะได้
ข้อความอะไรออกมา Django เรียกใช้ `__str__` ในหลายที่โดยอัตโนมัติ ที่สำคัญที่สุดคือ
**Django Admin** (Part 017) ซึ่งใช้ผลลัพธ์ของ `__str__` เป็นข้อความแสดงแทน object
ในหน้ารายการและ dropdown ทุกจุด — ถ้าไม่เขียน `__str__` เอง Django Admin จะแสดงเป็น
`Post object (1)` ซึ่งไม่มีประโยชน์ต่อผู้ใช้งานเลย

**กฎเหล็ก**: ทุก model ที่คุณเขียนในหลักสูตรนี้ **ต้องมี** `__str__` เสมอ ไม่มีข้อยกเว้น

### 106.2 `get_absolute_url()`: ธรรมเนียมมาตรฐานของ Django

จาก Part 007 (ขั้นตอนที่ 66.3) คุณเคยเห็น method นี้แบบสั้น ๆ มาแล้ว ตอนนี้มาเจาะลึก
ความสำคัญของมัน:

```python
from django.db import models
from django.urls import reverse


class Post(models.Model):
    # ... field ทั้งหมดตามขั้นตอนที่ 103 ...

    def get_absolute_url(self):
        return reverse('blog:detail', kwargs={'slug': self.slug})
```

`get_absolute_url()` เป็น **ธรรมเนียม (convention)** ที่ Django เอกสารแนะนำอย่างเป็น
ทางการ ไม่ใช่ method ที่ Django บังคับผ่าน error แต่มีหลายส่วนของ Django ที่จะ
**มองหา method นี้โดยอัตโนมัติ** ถ้ามันมีอยู่:

| ใช้ที่ไหน | พฤติกรรมถ้ามี `get_absolute_url()` |
|---|---|
| `redirect(post)` (ส่ง instance ตรง ๆ แทนชื่อ URL) | เรียก `get_absolute_url()` ให้อัตโนมัติ (Part 007 ขั้นตอนที่ 66) |
| Django Admin: ปุ่ม "View on site" | ลิงก์ไปหน้า `get_absolute_url()` ให้อัตโนมัติ (Part 017) |
| `{% url %}` template tag เทียบเท่า | ไม่บังคับ แต่โครงการจำนวนมากเรียก `{{ post.get_absolute_url }}` ตรง ๆ ใน template แทนการเขียน `{% url %}` ซ้ำทุกที่ |

```python
def go_to_latest_post(request):
    latest = Post.objects.filter(is_published=True).first()
    if latest:
        return redirect(latest)   # เทียบเท่ากับ redirect('blog:detail', slug=latest.slug)
    return redirect('blog:list')
```

**กฎของหลักสูตรนี้**: ทุก model ที่มีหน้า "detail" ของตัวเอง (สามารถเปิดดูทีละรายการได้)
ต้องมี `get_absolute_url()` เสมอ เพื่อให้เขียน `redirect(obj)` ได้ทุกที่แทนการพิมพ์ชื่อ
URL ซ้ำ ๆ (ยึดหลัก DRY เช่นเดียวกับที่เรียนไปตั้งแต่ Part 001)

### 106.3 Custom Business-Logic Methods

นี่คือส่วนที่ทำให้ model เป็นมากกว่า "ที่เก็บข้อมูล" — เราสามารถเขียน method ที่มี
ตรรกะทางธุรกิจเฉพาะของแอปพลิเคชันไว้ในตัว model ได้เลย:

```python
class Post(models.Model):
    # ... field ทั้งหมด ...

    def increment_view_count(self):
        """
        เพิ่มจำนวนเข้าชม 1 ครั้ง แล้วบันทึกเฉพาะ field นี้ (ไม่ save ทั้ง row)
        เพื่อประสิทธิภาพที่ดีกว่าเมื่อ view ถูกเรียกบ่อยมาก
        """
        self.view_count += 1
        self.save(update_fields=['view_count'])

    @property
    def is_free(self):
        """คืนค่า True ถ้าบทความนี้อ่านได้ฟรี (ไม่ใช่พรีเมียม หรือราคา 0)"""
        return not self.is_premium or self.price == 0

    @property
    def reading_time_label(self):
        """ข้อความพร้อมแสดงผลสำหรับเวลาอ่านโดยประมาณ"""
        if self.reading_time_minutes <= 1:
            return 'อ่านไม่ถึง 1 นาที'
        return f'อ่านประมาณ {self.reading_time_minutes} นาที'

    def publish(self):
        """เผยแพร่บทความ (business logic รวมศูนย์ไว้จุดเดียว แทนการเขียน post.is_published = True ทุกที่)"""
        self.is_published = True
        self.save(update_fields=['is_published', 'updated_at'])
```

การใช้งานในมุมมองของ view (ดูเบาลงมากเมื่อ logic ถูกย้ายเข้า model):

```python
# blog/views.py
from django.shortcuts import get_object_or_404, render
from .models import Post


def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    post.increment_view_count()
    return render(request, 'blog/post_detail.html', {'post': post})
```

สังเกตว่า view ไม่รู้เลยว่า "การเพิ่มยอดวิว" ทำงานอย่างไรข้างใน (บวกเลข, เรียก
`update_fields`) มันแค่เรียก `post.increment_view_count()` — นี่คือประโยชน์ของการ
เก็บ business logic ไว้ใน model: ถ้าวันหนึ่งต้องเปลี่ยนวิธีนับยอดวิว (เช่น ป้องกันการ
รีเฟรชหน้าซ้ำ ๆ เพื่อปั่นยอด) จะแก้แค่จุดเดียวใน `models.py` โดยไม่ต้องไล่แก้ทุก view
ที่เรียกใช้

> **หมายเหตุเรื่อง Race Condition**: โค้ด `self.view_count += 1` ข้างต้นมีความเสี่ยง
> race condition เล็กน้อยถ้ามีผู้ใช้หลายคนเข้าดูพร้อมกันในเสี้ยววินาทีเดียวกัน (ยอดวิว
> อาจนับตกหล่นได้ในทางทฤษฎี) วิธีแก้ที่ปลอดภัย 100% คือใช้ **`F()` expression**
> ของ Django ORM ซึ่งสั่งให้ฐานข้อมูลคำนวณค่าบวกเพิ่มโดยตรง ไม่ต้องอ่านค่าเข้ามาที่
> Python ก่อน — เรื่องนี้จะเรียนแบบเต็มรูปแบบใน **Part 014 (Q/F Expressions)**
> ตอนนี้ขอให้เข้าใจแนวคิดของการเขียน method ไว้ใน model ก่อนเป็นหลัก

---

## ขั้นตอนที่ 107: Model `Meta` Class เบื้องต้น

### 107.1 `Meta` คืออะไร

`Meta` คือ class ที่ซ้อนอยู่ภายใน model class (nested class) ใช้กำหนด **พฤติกรรม
ระดับตาราง** ที่ไม่ใช่ field ใด field หนึ่งโดยเฉพาะ Django จะมองหา class ชื่อ `Meta`
นี้โดยอัตโนมัติเสมอถ้ามีอยู่ในตัว model

```python
class Post(models.Model):
    # ... field ทั้งหมด ...

    class Meta:
        ordering = ['-created_at']
        verbose_name = 'บทความ'
        verbose_name_plural = 'บทความทั้งหมด'
```

### 107.2 `ordering`: กำหนดลำดับการเรียงข้อมูลเริ่มต้น

```python
class Meta:
    ordering = ['-created_at']   # เรียงจากใหม่ไปเก่า (เครื่องหมาย - หมายถึง descending)
```

เมื่อกำหนด `ordering` ไว้แล้ว การ query ใด ๆ ที่**ไม่ได้ระบุ** `.order_by()` เอง จะได้
ผลลัพธ์เรียงตามนี้โดยอัตโนมัติเสมอ:

```python
Post.objects.all()                  # ได้ผลลัพธ์เรียงจากใหม่ไปเก่าอัตโนมัติ (ตาม Meta.ordering)
Post.objects.filter(is_published=True)   # เรียงตาม Meta.ordering เช่นกัน
Post.objects.order_by('title')      # ระบุ order_by() เอง → ใช้ตามที่ระบุ ไม่ใช้ Meta.ordering
```

รองรับการเรียงหลายชั้นด้วยการใส่หลาย field ใน list (เรียงตาม field แรกก่อน ถ้าเท่ากัน
ค่อยเรียงตาม field ถัดไป):

```python
class Meta:
    ordering = ['-is_premium', '-created_at']   # บทความพรีเมียมขึ้นก่อน แล้วค่อยเรียงตามวันที่ใหม่สุด
```

> **ข้อควรระวังด้านประสิทธิภาพ**: `ordering` ที่ไม่มี **index** รองรับในฐานข้อมูล
> จะทำให้ query ช้าลงเมื่อข้อมูลมีจำนวนมาก เพราะฐานข้อมูลต้องเรียงข้อมูลทุกครั้งที่ query
> (`ORDER BY` โดยไม่มี index คือการเรียงแบบ full table scan) เรื่อง **Database Index**
> อย่างเป็นทางการจะเรียนเต็มรูปแบบใน **Part 015** พร้อมกับ `Meta.indexes`

### 107.3 `verbose_name` และ `verbose_name_plural`

```python
class Meta:
    verbose_name = 'บทความ'          # ชื่อเรียกเอกพจน์ (1 รายการ)
    verbose_name_plural = 'บทความทั้งหมด'   # ชื่อเรียกพหูพจน์ (หลายรายการ)
```

ทั้งสองค่านี้ใช้แสดงผลใน **Django Admin** เป็นหลัก (Part 017): `verbose_name` ใช้ใน
หัวข้อฟอร์มเพิ่ม/แก้ไขรายการเดียว ส่วน `verbose_name_plural` ใช้เป็นชื่อลิงก์ในเมนู
sidebar ของ Admin ถ้าไม่ระบุ Django จะสร้าง `verbose_name` อัตโนมัติจากชื่อ class
(แปลง `CamelCase` เป็นตัวพิมพ์เล็กคั่นด้วยช่องว่าง เช่น `Post` → `post`) และเติม `s`
ต่อท้ายให้เป็น `verbose_name_plural` อัตโนมัติ (ซึ่งใช้ไม่ได้กับภาษาไทยที่ไม่มี
พหูพจน์แบบเติมตัวอักษร จึงควรระบุเองเสมอเมื่อใช้ label ภาษาไทย)

### 107.4 ตัวอย่างสมบูรณ์และตัวเลือกอื่นที่จะเรียนต่อใน Part 015

```python
class Meta:
    ordering = ['-created_at']
    verbose_name = 'บทความ'
    verbose_name_plural = 'บทความทั้งหมด'
```

`Meta` class ยังรองรับตัวเลือกอีกจำนวนมากที่ **ยังไม่พูดถึงในขั้นตอนนี้โดยตั้งใจ**
เพราะต้องอาศัยความเข้าใจเรื่อง QuerySet และ Database Constraint ที่ลึกกว่านี้:

| ตัวเลือกใน `Meta` | จะเรียนเต็มรูปแบบใน |
|---|---|
| `indexes` (กำหนด database index) | Part 015 |
| `constraints` (`UniqueConstraint`, `CheckConstraint`) | Part 015 |
| `unique_together` (แบบเก่า ก่อนมี `constraints`) | Part 015 |
| `permissions` (custom permission ของ model) | Part 033 |
| `abstract` (Abstract Base Class) | Part 015 |
| `proxy` (Proxy Model) | Part 015 |
| `db_table` (ตั้งชื่อตารางเอง แทนชื่อที่ Django สร้างอัตโนมัติ) | Part 015 |

ตอนนี้ขอให้จำแค่ว่า `Meta.ordering`, `Meta.verbose_name`, และ `Meta.verbose_name_plural`
คือ 3 ตัวเลือกพื้นฐานที่สุดที่ทุก model ระดับมืออาชีพควรมีติดไว้เสมอ

---

## ขั้นตอนที่ 108: Primary Key ทางเลือก: `AutoField` vs `BigAutoField` vs `UUIDField`

### 108.1 Primary Key คืออะไร (ทบทวนสั้น ๆ)

ทุกตารางในฐานข้อมูลเชิงสัมพันธ์ควรมี **Primary Key (PK)** — คอลัมน์ที่ใช้ระบุแต่ละแถว
อย่างไม่ซ้ำกัน ถ้า model ไม่ได้กำหนด field ไหนเป็น PK เอง Django จะเพิ่มคอลัมน์ชื่อ `id`
ให้อัตโนมัติเสมอ

### 108.2 `AutoField` vs `BigAutoField`

ทั้งสองคือจำนวนเต็มที่เพิ่มค่าอัตโนมัติทีละ 1 (auto-increment) ต่างกันแค่**ขนาด**:

| | `AutoField` | `BigAutoField` |
|---|---|---|
| ขนาด | 32-bit signed integer | 64-bit signed integer |
| ค่าสูงสุดที่เก็บได้ | ประมาณ 2.1 พันล้าน (2,147,483,647) | ประมาณ 9.2 ล้านล้านล้าน (9,223,372,036,854,775,807) |
| ค่า default ของ Django ปัจจุบัน | เป็น default **ก่อน** Django 3.2 | เป็น **default ตั้งแต่ Django 3.2** เป็นต้นไป |
| เหมาะกับ | โปรเจกต์เล็กที่มั่นใจว่าจะไม่มีวันเกิน 2 พันล้านแถว | โปรเจกต์ทั่วไปในปัจจุบัน (ค่าแนะนำมาตรฐาน) |

ตั้งแต่ Django 3.2 เป็นต้นไป โปรเจกต์ใหม่ที่สร้างด้วย `startproject` จะมีบรรทัดนี้ใน
`settings.py` ให้อัตโนมัติ (คุณเคยเห็นผ่านตามาแล้วตั้งแต่ Part 004):

```python
# settings.py
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

ค่านี้กำหนดว่า **ทุก model ในโปรเจกต์** ที่ไม่ได้ระบุ PK เอง จะใช้ `BigAutoField`
โดยอัตโนมัติ เป็นเหตุผลว่าทำไม `Post` model ตลอดหลักสูตรนี้ (ที่ไม่เคยระบุ PK เอง)
จึงมี `id` เป็น `BigAutoField` อยู่แล้วตั้งแต่ Part 005 โดยไม่ต้องเขียนอะไรเพิ่ม

หากต้องการ override เฉพาะ model ใด model หนึ่งให้ใช้ `AutoField` แบบเดิม ก็ทำได้โดย
ระบุ field เองตรง ๆ:

```python
class LegacyLookupTable(models.Model):
    id = models.AutoField(primary_key=True)   # บังคับใช้ 32-bit แทนค่า default ของโปรเจกต์
```

### 108.3 `UUIDField` เป็น Primary Key: ทางเลือกที่ปลอดภัยกว่าในบางสถานการณ์

**UUID (Universally Unique Identifier)** คือค่าสุ่มขนาด 128-bit ที่รับประกันว่าจะไม่ซ้ำ
กับ UUID อื่นใดในโลก (ในทางปฏิบัติ) แสดงผลเป็น string รูปแบบ
`550e8400-e29b-41d4-a716-446655440000`

```python
import uuid
from django.db import models


class Order(models.Model):
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False,
    )
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'Order {self.id}'
```

### 108.4 ตารางเปรียบเทียบข้อดี-ข้อเสีย

| ประเด็น | `BigAutoField` (ตัวเลขเรียงลำดับ) | `UUIDField` (สุ่ม) |
|---|---|---|
| ขนาดพื้นที่จัดเก็บ | เล็ก (8 bytes) | ใหญ่กว่า (16 bytes) |
| ความเร็วของ index/join | เร็วกว่า (ตัวเลขเรียงลำดับ, index กระชับ) | ช้ากว่าเล็กน้อยในตารางขนาดใหญ่มาก |
| การเดา/นับจำนวนจากภายนอก | **เดาได้ง่าย** (เห็น `/orders/42/` รู้ทันทีว่ามีประมาณ 42 order ในระบบ, เดา URL อื่นได้ง่าย) | **เดาไม่ได้เลย** ปลอดภัยกว่ามากสำหรับ URL ที่เปิดเผยต่อสาธารณะ |
| ใช้งานง่ายในการ debug/พิมพ์คุยกัน | ง่าย (`id=42` จำง่าย พิมพ์ง่าย) | ยากกว่า (string ยาว จำไม่ได้) |
| การรวมข้อมูลจากหลายระบบ (merge, sync) | เสี่ยงชนกัน (สอง database ต่างมี `id=1`) | ไม่มีวันชนกัน เหมาะกับระบบ distributed/microservices |
| เหมาะกับ | โปรเจกต์ทั่วไป, ข้อมูลภายในที่ไม่เปิดเผย PK ต่อสาธารณะ | ระบบที่ต้องป้องกันการเดา ID (คำสั่งซื้อ, invoice, API token), ระบบ distributed |

### 108.5 แนวทางแบบผสม (Hybrid) ที่ทีมมืออาชีพนิยมใช้

ในทางปฏิบัติ ทีมงานระดับมืออาชีพจำนวนมากเลือก**ทางสายกลาง**: ใช้ `BigAutoField` เป็น
Primary Key ภายใน (เร็ว, index กระชับ, ใช้ join ภายในระบบ) แต่เพิ่ม field UUID แยก
ต่างหากสำหรับ **เปิดเผยต่อสาธารณะ** (public-facing identifier) เช่นใน URL หรือ API:

```python
import uuid
from django.db import models


class Order(models.Model):
    id = models.BigAutoField(primary_key=True)   # ใช้ภายในระบบ, join, index
    public_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True, db_index=True)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)

    def get_absolute_url(self):
        # ใช้ public_id ใน URL แทน id ตรง ๆ เพื่อไม่เปิดเผยจำนวน Order ทั้งหมดในระบบ
        from django.urls import reverse
        return reverse('shop:order_detail', kwargs={'public_id': self.public_id})
```

แนวทางนี้ได้ทั้งประสิทธิภาพของ `BigAutoField` ภายใน และความปลอดภัยของ `UUIDField`
เมื่อเปิดเผยต่อภายนอก — เราจะกลับมาใช้แนวคิดนี้จริงจังเมื่อเรียน Django REST Framework
ใน Phase 5 ของหลักสูตร

> **ข้อควรระวัง**: การเปลี่ยนชนิดของ Primary Key **หลังจากที่ table มีข้อมูลอยู่แล้ว
> และมี `ForeignKey` จากตารางอื่นมาอ้างอิงแล้ว** เป็นหนึ่งใน migration ที่อันตรายที่สุด
> ที่ทำได้ในระบบ production เพราะกระทบทุกตารางที่ join กับ PK นั้น ควรตัดสินใจเลือก
> ชนิดของ PK **ตั้งแต่ตอนออกแบบ model ครั้งแรก** ก่อนที่จะมีข้อมูลจริงเข้าระบบ

---

## ขั้นตอนที่ 109: แก้ไข Model ที่มีข้อมูลอยู่แล้ว และผลกระทบต่อ Migration ที่ต้องระวัง

ทุกขั้นตอนที่ผ่านมาในสมมติฐานว่าตารางยังไม่มีข้อมูล หรือเพิ่ม field ที่มี `default`
เสมอ (ทำให้ปลอดภัย) แต่ในงานจริง โปรเจกต์ที่ deploy ไปแล้วมักมี**ข้อมูลจริงของผู้ใช้**
อยู่ในตาราง การแก้ไข model จึงต้องระมัดระวังมากกว่านี้

### 109.1 กรณีที่ 1: เพิ่ม Field ที่ไม่มี `default` ลงในตารางที่มีข้อมูลอยู่แล้ว

ลองจำลองสถานการณ์: เพิ่ม field `author_note` เป็น `CharField` แบบไม่มี `default` และ
ไม่มี `blank=True`:

```python
class Post(models.Model):
    # ... field เดิมทั้งหมด ...
    author_note = models.CharField(max_length=100)   # ลืมใส่ default หรือ blank=True!
```

เมื่อรัน `makemigrations` บนตารางที่มีข้อมูลอยู่แล้ว Django จะ**หยุดถามทันที**
เพราะรู้ว่าแถวข้อมูลเดิมทุกแถวไม่มีค่าให้กับคอลัมน์ใหม่นี้ (คอลัมน์นี้ regulaly เป็น
`NOT NULL` เพราะไม่มี `blank`/`null`):

```
You are trying to add a non-nullable field 'author_note' to post without a default;
we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value
    for this column)
 2) Quit and manually define a default value in models.py.
Select an option:
```

**ตัวเลือกที่ 1** (one-off default) ให้ Django เติมค่าที่คุณพิมพ์ตอนนั้นให้กับ**แถวเดิม
ทั้งหมด**ครั้งเดียว แล้วจากนั้นแถวใหม่ที่สร้างในอนาคตจะยังคง**บังคับกรอก** field นี้
เสมอ (เพราะ model ไม่มี `default` ถาวร) วิธีนี้เหมาะกับกรณีที่ field ใหม่ควรบังคับกรอก
จริง ๆ สำหรับข้อมูลใหม่ทั้งหมด

**ตัวเลือกที่ 2** (quit แล้วไปแก้ `models.py` เอง) คือทางเลือกที่ปลอดภัยกว่าและเป็นที่
แนะนำในหลักสูตรนี้เสมอ: กลับไปเพิ่ม `default=''` หรือ `blank=True` ใน `models.py` ก่อน
แล้วค่อยรัน `makemigrations` ใหม่ — วิธีนี้ทำให้ทั้งแถวเก่าและแถวใหม่ในอนาคตมีพฤติกรรม
สอดคล้องกัน ไม่ใช่แค่แก้ปัญหาเฉพาะหน้าตอนรัน migration ครั้งเดียว

```python
# ✅ ปลอดภัยกว่า: กำหนด default ไว้ใน models.py ให้ชัดเจนตั้งแต่แรก
author_note = models.CharField(max_length=100, blank=True, default='')
```

### 109.2 กรณีที่ 2: การเปลี่ยนชื่อ Field (Rename)

สมมติทีมตัดสินใจเปลี่ยนชื่อ `is_premium` เป็น `is_paid_content` (ชื่อที่สื่อความหมาย
ชัดเจนกว่า) วิธีที่ **ถูกต้อง** คือแก้ชื่อใน `models.py` ก่อน แล้วรัน
`makemigrations` — Django จะ**ตรวจจับความคล้ายกัน**ระหว่าง field ที่หายไปกับ field
ใหม่ที่โผล่มา แล้วถามว่าใช่การ rename หรือไม่:

```
Did you rename post.is_premium to post.is_paid_content (a BooleanField)? [y/N]
```

ถ้าตอบ **`y`** (yes) Django จะสร้าง `RenameField` operation ซึ่ง**เก็บข้อมูลเดิมไว้ครบ
ทุกแถว** เพียงแค่เปลี่ยนชื่อคอลัมน์ในฐานข้อมูล (`ALTER TABLE ... RENAME COLUMN`)

ถ้าตอบ **`N`** (no) หรือกดปุ่มอื่นโดยไม่ตั้งใจ Django จะสร้างเป็น**`RemoveField` +
`AddField`** สองคำสั่งแยกกัน ซึ่งหมายความว่า **ข้อมูลเดิมทั้งหมดในคอลัมน์นั้นจะหายไป
ถาวร** แล้วสร้างคอลัมน์ใหม่ที่ว่างเปล่าขึ้นมาแทน — นี่คือหนึ่งในกับดักที่อันตรายที่สุด
ที่มือใหม่พลาดบ่อยที่สุด เพราะไฟล์ migration ที่ได้ยัง "รันผ่าน" ไม่มี error ใด ๆ
แต่ **ข้อมูลผู้ใช้หายไปเงียบ ๆ**

```python
# ผลลัพธ์ถ้าตอบ 'N' โดยไม่ตั้งใจ (อันตราย! ข้อมูลเดิมหายหมด)
operations = [
    migrations.RemoveField(model_name='post', name='is_premium'),
    migrations.AddField(model_name='post', name='is_paid_content', field=models.BooleanField(default=False)),
]

# ผลลัพธ์ถ้าตอบ 'y' (ถูกต้อง! ข้อมูลเดิมยังอยู่ครบ)
operations = [
    migrations.RenameField(model_name='post', old_name='is_premium', new_name='is_paid_content'),
]
```

**กฎเหล็กของหลักสูตรนี้**: ทุกครั้งที่ `makemigrations` ถามคำถามแบบ "Did you rename...?"
ให้ **หยุดอ่านให้ละเอียดก่อนตอบเสมอ** และถ้าไม่แน่ใจ ให้เปิดไฟล์ migration ที่สร้างขึ้น
มาตรวจดูก่อนรัน `migrate` จริงทุกครั้ง (ทบทวนนิสัยนี้คู่กับกฎการอ่านก่อน commit ที่เรียน
มาตลอดหลักสูตร)

### 109.3 กรณีที่ 3: การลบ Field ออกจาก Model

```python
# ลบ reading_time_minutes ออกจาก models.py แล้วรัน makemigrations
```

จะได้ migration ที่มี `RemoveField` ซึ่งเมื่อรัน `migrate` จริง **คอลัมน์และข้อมูลทั้ง
คอลัมน์จะถูกลบทิ้งถาวร ไม่มีทาง undo กลับมาได้** (นอกจากมี backup ฐานข้อมูลไว้ก่อนหน้า)
นี่คือเหตุผลที่ทีมงานมืออาชีพนิยมใช้แนวทางที่เรียกว่า **"Expand and Contract Pattern"**
เมื่อจะลบหรือเปลี่ยนแปลง field ที่มีข้อมูลสำคัญ:

| ขั้นตอน | สิ่งที่ทำ |
|---|---|
| 1. Expand | เพิ่ม field ใหม่เข้าไป (ไม่ลบของเก่า) deploy โค้ดที่เขียนข้อมูลลงทั้ง field เก่าและใหม่พร้อมกัน |
| 2. Migrate data | รัน script ย้ายข้อมูลเก่าที่มีอยู่แล้วไปเติมใน field ใหม่ให้ครบ (data migration — เรียนเต็มรูปแบบใน Part 016) |
| 3. Switch | แก้โค้ดทั้งหมดให้อ่าน/เขียนเฉพาะ field ใหม่ deploy แล้วเฝ้าดูว่าไม่มีปัญหา |
| 4. Contract | เมื่อมั่นใจว่าไม่มีส่วนไหนของระบบใช้ field เก่าแล้ว ค่อยลบ field เก่าออกด้วย migration แยกต่างหาก |

รูปแบบนี้ป้องกันการ deploy ที่ทำให้ระบบล่มหรือข้อมูลหายกะทันหัน เพราะแบ่งการเปลี่ยนแปลง
ใหญ่ ๆ ออกเป็นหลายขั้นตอนเล็ก ที่แต่ละขั้นตอน **ย้อนกลับได้ (reversible)** ถ้าเกิด
ปัญหา — เราจะฝึกเขียน **Data Migration** จริงด้วยมือใน Part 016

### 109.4 กรณีที่ 4: Migration ชนกัน (Conflicting Migrations) เมื่อทำงานเป็นทีม

สถานการณ์นี้เกิดบ่อยมากในทีมจริง: นักพัฒนา A และ B ต่างแก้ `models.py` ของแอปเดียวกัน
พร้อมกันคนละ branch แล้วต่างคนต่างรัน `makemigrations` บนเครื่องตัวเอง ทำให้ได้ไฟล์
migration ที่มี `dependencies` ชี้ไปที่ migration ตัวเดียวกัน (เช่น `0005`) แต่มีเนื้อหา
คนละแบบ — เมื่อ merge โค้ดเข้าด้วยกัน จะเกิด **migration graph ที่แตกสาขา (branching)**
ซึ่ง Django ตรวจจับได้และแจ้งเตือนทันทีตอนรัน `makemigrations`/`migrate`:

```
CommandError: Conflicting migrations detected; multiple leaf nodes in the migration
graph: (0006_add_author_field, 0006_add_tags_field in blog).
To fix them run 'python manage.py makemigrations --merge'
```

แก้ไขด้วยคำสั่งที่ Django แนะนำมาให้ตรง ๆ:

```bash
python manage.py makemigrations --merge
```

Django จะสร้างไฟล์ migration ใหม่ที่มี `dependencies` ชี้ไปยัง**ทั้งสองสาขา**พร้อมกัน
(merge migration) ทำให้ graph กลับมาเป็นเส้นเดียวที่สมบูรณ์อีกครั้ง โดยไม่กระทบเนื้อหา
ของทั้งสอง migration เดิมเลย

### 109.5 ตารางสรุปความเสี่ยงของการแก้ไข Model แต่ละแบบ

| การเปลี่ยนแปลง | ความเสี่ยงต่อข้อมูลเดิม | แนวทางที่ปลอดภัย |
|---|---|---|
| เพิ่ม field ใหม่ที่มี `default` หรือ `blank=True` | ไม่มีความเสี่ยง | ทำได้ตามปกติ |
| เพิ่ม field ใหม่ที่ไม่มี `default` บนตารางที่มีข้อมูล | ต้องเติมค่าให้แถวเดิมก่อนเสมอ | ใส่ `default`/`blank=True` ใน `models.py` ก่อนเสมอ |
| เปลี่ยนชื่อ field | ข้อมูลหายถ้าตอบคำถาม rename ผิด | อ่านคำถามให้ละเอียด ตอบ `y` เมื่อใช่การ rename จริง ตรวจไฟล์ migration ก่อน `migrate` |
| ลบ field | ข้อมูลในคอลัมน์นั้นหายถาวร | ใช้ Expand-and-Contract Pattern, สำรองฐานข้อมูลก่อนเสมอ |
| ขยาย `max_length` ของ `CharField` | ไม่มีความเสี่ยง | ทำได้ตามปกติ |
| ลด `max_length` ของ `CharField` ที่มีข้อมูลยาวกว่าค่าใหม่อยู่แล้ว | ข้อมูลอาจถูกตัดทอนหรือ migration ล้มเหลว (ขึ้นกับฐานข้อมูล) | ตรวจสอบความยาวข้อมูลจริงในตารางก่อนเสมอด้วย query |
| เปลี่ยนชนิด Primary Key บนตารางที่มี ForeignKey ชี้มา | เสี่ยงสูงมาก กระทบทุกตารางที่ join | หลีกเลี่ยง ตัดสินใจชนิด PK ตั้งแต่ตอนออกแบบแรกเริ่ม |
| Migration ชนกันจากการทำงานเป็นทีม | ไม่กระทบข้อมูล แต่บล็อกการ deploy | `python manage.py makemigrations --merge` |

---

## ขั้นตอนที่ 110: สรุปและแบบฝึกหัด

### 110.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจว่า Django Model แปลง Python class เป็นตารางฐานข้อมูลผ่าน ORM อย่างไร
- ✅ รู้จัก Field Types ที่ใช้บ่อยที่สุดทั้ง 13 ตัว พร้อมตัวอย่างการใช้งานจริงของแต่ละตัว
- ✅ เข้าใจความแตกต่างสำคัญระหว่าง `null` (ระดับฐานข้อมูล) กับ `blank` (ระดับ validation)
- ✅ ใช้ `TextChoices`/`IntegerChoices` (Django 3.0+) แทนการเขียน choices เป็น list ตรง ๆ
- ✅ ขยาย `Post` model จาก Part 007 ให้สมบูรณ์แบบมืออาชีพ พร้อมเหตุผลของทุก field ใหม่
- ✅ อ่านและเข้าใจไฟล์ migration ที่ Django สร้างให้ทุกบรรทัด ทั้ง `dependencies` และ `operations`
- ✅ เข้าใจ Migration Graph และใช้ `showmigrations`, `sqlmigrate` เพื่อตรวจสอบก่อนรันจริง
- ✅ เขียน `__str__`, `get_absolute_url()`, และ custom business-logic method บน model
- ✅ ใช้ `Meta.ordering`, `verbose_name`, `verbose_name_plural` เบื้องต้น
- ✅ เข้าใจข้อดี-ข้อเสียของ `BigAutoField` vs `UUIDField` เป็น Primary Key
- ✅ รู้วิธีแก้ไข model ที่มีข้อมูลอยู่แล้วอย่างปลอดภัย และหลีกเลี่ยงกับดักที่ทำข้อมูลหาย

### 110.2 Checklist ก่อนไป Part ถัดไป

- [ ] `blog/models.py` มี `Post` model ครบทุก field ตามขั้นตอนที่ 103.3
- [ ] รัน `python manage.py makemigrations blog` สำเร็จ ไม่มี error
- [ ] เปิดไฟล์ migration ที่สร้างขึ้นมาอ่านจริง เข้าใจทุก `AddField`/`AlterField`
- [ ] รัน `python manage.py sqlmigrate blog <เลข migration>` แล้วอ่าน SQL ที่ได้เข้าใจ
- [ ] รัน `python manage.py migrate` สำเร็จ ไม่มี error
- [ ] รัน `python manage.py showmigrations blog` แล้วเห็น `[X]` ครบทุกตัว
- [ ] เปิด `python manage.py shell` แล้วสร้าง `Post` ใหม่ ทดสอบ `get_absolute_url()`
      และ `increment_view_count()` ได้ผลลัพธ์ถูกต้อง
- [ ] อธิบายความต่างระหว่าง `null=True` กับ `blank=True` ด้วยคำพูดตัวเองได้

### 110.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เพิ่ม field `tags_text` เป็น `CharField(max_length=200, blank=True)`
ลงใน `Post` model สำหรับเก็บแท็กแบบคั่นด้วยจุลภาค (เช่น `"python,django,web"`) แล้วเขียน
`@property` ชื่อ `tags_list` ที่แปลงค่านี้เป็น Python list (เช่น `['python', 'django', 'web']`)
โดยตัดช่องว่างส่วนเกินออกและข้ามค่าว่างเปล่า จากนั้นรัน `makemigrations` และ `migrate`
ให้เรียบร้อย (คำใบ้: ความสัมพันธ์แบบ Many-to-Many ที่ถูกต้องสำหรับ tag จริง ๆ จะเรียน
ใน Part 012 — แบบฝึกหัดนี้จงใจให้ใช้วิธีง่าย ๆ ก่อนเพื่อฝึก field/property)

**แบบฝึกหัดที่ 2**: สร้าง model ใหม่ชื่อ `Category` ที่มีเพียง `name` (`CharField`,
`unique=True`) และ `slug` (`SlugField`, `unique=True`, auto-generate ด้วย `slugify()`
เหมือน `Post`) พร้อม `Meta.ordering = ['name']`, `verbose_name`, `verbose_name_plural`,
`__str__()`, และ `get_absolute_url()` ที่ชี้ไปยัง URL name `blog:category_detail`
(ยังไม่ต้องสร้าง view/URL จริงก็ได้ แค่เขียน method ให้ครบถูกไวยากรณ์) รัน
`makemigrations`/`migrate` ให้สำเร็จ (นี่คือการซ้อมมือก่อนเชื่อม `Category` เข้ากับ
`Post` ด้วย `ForeignKey` ใน Part 012)

**แบบฝึกหัดที่ 3**: จำลองสถานการณ์ "เพิ่ม field ที่ไม่มี default บนตารางที่มีข้อมูล"
ด้วยตัวเอง: สร้างข้อมูล `Post` อย่างน้อย 3 รายการผ่าน shell ก่อน จากนั้นเพิ่ม field
`internal_code` เป็น `CharField(max_length=20)` **โดยไม่ใส่** `default` หรือ `blank=True`
แล้วรัน `makemigrations` สังเกตคำถามที่ Django ถาม ลองเลือกตัวเลือกที่ 1 (one-off default)
ดูผลลัพธ์ในไฟล์ migration ที่ได้ จากนั้นลบ migration นั้นทิ้ง แก้ `models.py` ให้มี
`default=''` แทน แล้วรัน `makemigrations` ใหม่อีกครั้ง เปรียบเทียบไฟล์ migration ทั้ง
สองแบบว่าต่างกันอย่างไร

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: เปลี่ยนชื่อ field `content_type` เป็น `post_type` ใน
`Post` model แล้วรัน `makemigrations` ตอบ "yes" เมื่อ Django ถามเรื่อง rename
เปิดไฟล์ migration ที่ได้ ตรวจสอบว่าเป็น `RenameField` จริง แล้วรัน `sqlmigrate`
เพื่อดู SQL ที่แท้จริงที่เกิดขึ้น (ควรเป็นคำสั่ง `RENAME COLUMN` ไม่ใช่ `DROP`+`ADD`)
จากนั้นรัน `migrate` แล้วเปิด `python manage.py shell` ตรวจสอบว่าข้อมูลเดิมที่เคย
สร้างไว้ในแบบฝึกหัดก่อนหน้ายังคงอยู่ครบถ้วนใน field ที่เปลี่ยนชื่อใหม่

### 110.4 คำถามที่พบบ่อย (FAQ)

**Q: ทำไม Django ไม่สร้างตารางในฐานข้อมูลทันทีตอนเขียน model เสร็จ ต้องรัน
`makemigrations` แล้ว `migrate` แยกกันสองขั้นตอนทำไม?**
A: การแยกสองขั้นตอนทำให้เกิด**จุดตรวจสอบ (checkpoint)** ก่อนที่ฐานข้อมูลจริงจะถูกแก้ไข
คุณสามารถเปิดไฟล์ migration ที่ `makemigrations` สร้างขึ้นมาตรวจสอบ, แก้ไขเพิ่มเติม,
หรือแม้แต่ลบทิ้งได้ก่อนที่จะรัน `migrate` จริง ต่างจากถ้า Django แก้ฐานข้อมูลทันทีที่
เซฟไฟล์ `models.py` ซึ่งอันตรายมากในระบบ production ที่มีข้อมูลจริงอยู่

**Q: ถ้าลบไฟล์ migration ทิ้งไปแล้ว จะกู้คืนได้อย่างไร?**
A: ถ้ายังไม่ได้รัน `migrate` (migration นั้นยังไม่ถูก apply) สามารถรัน
`makemigrations` ใหม่เพื่อสร้างไฟล์ทดแทนได้ทันที แต่ถ้า migration นั้น **ถูก apply
ไปแล้ว** บนฐานข้อมูลใด ๆ (โดยเฉพาะ production) **ห้ามลบไฟล์ migration ทิ้งเด็ดขาด**
เพราะ Django จะสับสนเรื่องประวัติที่บันทึกไว้ในตาราง `django_migrations` ทันที
ควรสร้าง migration ใหม่เพื่อ "แก้ไขต่อ" แทนการลบของเดิมเสมอ

**Q: `TextChoices`/`IntegerChoices` ต่างจาก `choices` แบบเก่า (list of tuples) จริง ๆ
แค่ไหน ถ้าโปรเจกต์เก่ายังใช้แบบ list of tuples อยู่ต้องรีบเปลี่ยนไหม?**
A: ทั้งสองแบบทำงานเหมือนกันทุกประการในระดับฐานข้อมูล (`choices=` รับ list ของ tuple
เหมือนกันทั้งคู่ เพราะ `TextChoices.choices` ก็คืน list of tuples เช่นกัน) ไม่จำเป็น
ต้องรีบเปลี่ยนโค้ดเก่าที่ทำงานถูกต้องอยู่แล้ว แต่สำหรับโค้ดใหม่ หลักสูตรนี้แนะนำ
`TextChoices`/`IntegerChoices` เสมอ เพราะได้ประโยชน์เรื่อง autocomplete และการอ้างอิง
ค่าคงที่แบบปลอดภัยกว่า

**Q: จำเป็นต้องมี `get_absolute_url()` ทุก model เลยหรือไม่?**
A: ไม่บังคับในทางเทคนิค (Django ไม่ error ถ้าไม่มี) แต่แนะนำอย่างยิ่งสำหรับทุก model
ที่มีหน้า "detail" ของตัวเอง เพราะทำให้เขียน `redirect(obj)` และฟีเจอร์ "View on site"
ใน Django Admin ใช้งานได้ทันที ถือเป็นธรรมเนียมมาตรฐานที่โปรเจกต์ Django มืออาชีพ
เกือบทุกโปรเจกต์ทำตาม

### 110.5 เตรียมตัวสำหรับ Part ถัดไป

**Part 012: ความสัมพันธ์ระหว่างโมเดล: ForeignKey, OneToOne, ManyToMany** จะพา `Post`
model ของเราก้าวไปอีกขั้น จากที่ตอนนี้แต่ละบทความยังเป็น "เกาะ" ที่ไม่เชื่อมกับอะไรเลย
เราจะ:

- เชื่อม `Post` เข้ากับ `Category` ที่สร้างไว้ในแบบฝึกหัดที่ 2 ด้วย **`ForeignKey`**
  (ความสัมพันธ์แบบ many-to-one: หนึ่ง Category มีได้หลาย Post)
- เชื่อม `Post` เข้ากับผู้เขียน (`auth.User`) ด้วย `ForeignKey` เช่นกัน
- เรียนรู้ **`OneToOneField`** ผ่านตัวอย่าง `Author Profile` ที่ขยายข้อมูลของ `User`
- เรียนรู้ **`ManyToManyField`** ผ่านระบบ **Tags** ที่ถูกต้อง (แทนที่ `tags_text` แบบ
  ง่าย ๆ ในแบบฝึกหัดที่ 1 ของ Part นี้) ที่หนึ่งบทความมีได้หลายแท็ก และหนึ่งแท็กก็
  ใช้กับหลายบทความได้
- ทำความเข้าใจ `related_name`, `on_delete` ทุก option, และ query ผ่านความสัมพันธ์
  ด้วย double-underscore lookup (`Post.objects.filter(category__name='เทคโนโลยี')`)

โครงสร้างฐานข้อมูลของบล็อกจะเริ่มดูเป็นระบบจริงตั้งแต่ Part 012 เป็นต้นไป เตรียม
`Post` model ที่เขียนเสร็จใน Part นี้ให้พร้อม แล้วไปเชื่อมโยงข้อมูลเข้าด้วยกันกันเลย!
