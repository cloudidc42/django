# Part 002: ทบทวน Python ที่จำเป็นสำหรับ Django Developer

> **ขั้นตอนที่ 11-20 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: ทบทวนและปูพื้นฐานภาษา Python ในระดับที่ Django ใช้งานจริง
> ไม่ใช่แค่ไวยากรณ์พื้นฐาน แต่เจาะลึกแนวคิดที่ Django framework สร้างขึ้นบนพื้นฐานนั้น
> เมื่อจบ Part นี้ คุณจะเข้าใจว่าทำไม Django ถึงหน้าตาแบบที่เป็นอยู่ เพราะทุกฟีเจอร์ของ
> Django — Models, Views, Decorators อย่าง `@login_required`, Context Managers ของ
> Database Transaction — ล้วนสร้างจากแนวคิด Python พื้นฐานที่เราจะเรียนใน Part นี้ทั้งสิ้น

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 11: Variables, Data Types และ Type Hints ใน Python สมัยใหม่
- ขั้นตอนที่ 12: Functions, Default Arguments, `*args` และ `**kwargs`
- ขั้นตอนที่ 13: Classes และ Object-Oriented Programming
- ขั้นตอนที่ 14: Decorators — วิธีทำงานและทำไม Django ใช้เยอะมาก
- ขั้นตอนที่ 15: Context Managers และ `with` statement
- ขั้นตอนที่ 16: List/Dict/Set Comprehension
- ขั้นตอนที่ 17: Iterators และ Generators
- ขั้นตอนที่ 18: Exception Handling
- ขั้นตอนที่ 19: Modules, Packages และ Import System
- ขั้นตอนที่ 20: Dataclasses, typing ขั้นสูง, สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 11: Variables, Data Types และ Type Hints ใน Python สมัยใหม่

### 11.1 ทำไมต้องทบทวนเรื่องพื้นฐานขนาดนี้

หลายคนคิดว่า "ตัวแปรกับชนิดข้อมูล" เป็นเรื่องที่รู้อยู่แล้ว แต่ Django มีพฤติกรรมเฉพาะตัว
ที่ผูกกับ dynamic typing ของ Python อย่างลึกซึ้ง เช่น field ใน Model จะแปลงชนิดข้อมูล
ให้อัตโนมัติตามที่ประกาศไว้ ถ้าคุณไม่เข้าใจว่า Python แยกแยะชนิดข้อมูลอย่างไร คุณจะเจอ
บั๊กประหลาด ๆ ตอนเขียน Django เช่น การเปรียบเทียบ `None` กับ empty string หรือการส่ง
string แทนที่จะเป็น int เข้าไปใน URL parameter

### 11.2 Dynamic Typing และ Type Hints

Python เป็นภาษา **dynamically typed** คือไม่ต้องประกาศชนิดข้อมูลล่วงหน้า ตัวแปรจะรับ
ชนิดข้อมูลตามค่าที่ assign ให้ ณ ขณะรัน (runtime):

```python
x = 10          # x เป็น int
x = "สวัสดี"      # ตอนนี้ x กลายเป็น str แล้ว ไม่มี error ใด ๆ
x = [1, 2, 3]   # ตอนนี้ x เป็น list
```

สิ่งนี้ยืดหยุ่นมาก แต่ก็เสี่ยงต่อบั๊กในโปรเจกต์ใหญ่ ตั้งแต่ Python 3.5 เป็นต้นมา Python
รองรับ **Type Hints** (คำแนะนำชนิดข้อมูล) ซึ่งไม่ได้บังคับ runtime แต่ช่วยให้ editor,
linter และเครื่องมืออย่าง `mypy` ตรวจสอบความถูกต้องของโค้ดได้ล่วงหน้า:

```python
name: str = "สมชาย"
age: int = 25
price: float = 199.50
is_active: bool = True
tags: list[str] = ["python", "django", "web"]
scores: dict[str, int] = {"math": 90, "science": 85}
```

### 11.3 ตารางชนิดข้อมูลพื้นฐานที่ใช้บ่อยที่สุด

| ชนิดข้อมูล (Type) | ตัวอย่างค่า | ใช้ที่ไหนใน Django |
|---|---|---|
| `int` | `42`, `-7` | `IntegerField`, primary key (`id`), pagination |
| `float` | `3.14`, `99.9` | `FloatField` (พบน้อย นิยม `Decimal` มากกว่า) |
| `Decimal` | `Decimal("199.50")` | `DecimalField` สำหรับราคา/เงิน (แม่นยำกว่า float) |
| `str` | `"Django"` | `CharField`, `TextField`, ทุกอย่างที่เป็นข้อความ |
| `bool` | `True`, `False` | `BooleanField` |
| `None` | `None` | ค่าว่างของ Python (คนละความหมายกับ `null` ใน DB) |
| `list` | `[1, 2, 3]` | QuerySet ที่ evaluate แล้ว, JSON array |
| `dict` | `{"key": "value"}` | `request.POST`, JSON response, `context` ใน view |
| `tuple` | `(1, 2)` | `choices` ของ field, ค่าที่ไม่ควรถูกแก้ไข |
| `set` | `{1, 2, 3}` | permission checks, การหาความแตกต่างของกลุ่มข้อมูล |
| `datetime` | `datetime(2026, 1, 1)` | `DateTimeField` |

### 11.4 ทำไม Decimal ถึงสำคัญกว่า float ในงาน Django/E-commerce

นี่คือกับดักคลาสสิกที่มือใหม่ตกบ่อยมาก:

```python
>>> 0.1 + 0.2
0.30000000000000004   # float มีความคลาดเคลื่อนจากการเก็บเลขฐาน 2
```

ถ้าคุณใช้ `float` เก็บราคาสินค้า ระบบตัดเงินอาจคลาดเคลื่อนแบบนี้สะสมจนเกิดปัญหาการเงินจริง
Django จึงมี `DecimalField` โดยเฉพาะสำหรับข้อมูลการเงิน และ Python มีโมดูล `decimal`
ให้ใช้คู่กัน:

```python
from decimal import Decimal

price = Decimal("199.50")
quantity = 3
total = price * quantity
print(total)  # Decimal('598.50') แม่นยำ 100%
```

**กฎเหล็ก**: เมื่อสร้าง `Decimal` จากตัวเลขทศนิยม ให้ส่งเป็น **string** เสมอ
(`Decimal("199.50")`) ไม่ใช่ `Decimal(199.50)` เพราะ float ที่ไม่แม่นยำจะถูกแปลงเข้ามา
ในนั้นด้วย เราจะเจอ `DecimalField` ตัวจริงใน Part 011 (Django Models)

### 11.5 None และ Falsy Values — จุดที่มือใหม่งงบ่อยที่สุด

Python มีแนวคิด **truthy/falsy** ค่าที่ถือว่า "เป็นเท็จ" เมื่อใช้ในเงื่อนไข `if` มีดังนี้:

```python
# ค่าที่เป็น Falsy ทั้งหมดใน Python
falsy_values = [False, None, 0, 0.0, "", [], {}, (), set()]

for v in falsy_values:
    assert not v  # ทุกตัวผ่าน assertion นี้
```

ใน Django เรื่องนี้สำคัญมากตอนเขียน view:

```python
def product_detail(request, product_id):
    product = get_object_or_404(Product, pk=product_id)

    # ผิด! ถ้า description เป็น "" (empty string) จะเข้าเงื่อนไขนี้ด้วย
    if not product.description:
        product.description = "ไม่มีคำอธิบาย"

    # ถูกต้องกว่า ถ้าต้องการเช็คเฉพาะ None จริง ๆ
    if product.description is None:
        product.description = "ไม่มีคำอธิบาย"
```

**คำแนะนำ**: ใช้ `is None` / `is not None` เสมอเมื่อต้องการเช็คว่าไม่มีค่าจริง ๆ
(เช่นตรวจสอบว่า query parameter ถูกส่งมาหรือไม่) และใช้ `if not x:` เมื่อต้องการเช็ค
ทั้ง "ไม่มีค่า" และ "ค่าว่าง" พร้อมกัน (เช่นตรวจสอบว่า list ว่างหรือไม่)

### 11.6 f-strings: มาตรฐานการต่อ String ในโลก Python สมัยใหม่

```python
name = "Django"
version = 5.1

# วิธีเก่า (ยังใช้ได้แต่ไม่แนะนำ)
message = "เรียน " + name + " เวอร์ชัน " + str(version)

# วิธีมาตรฐานสมัยใหม่: f-string (Python 3.6+)
message = f"เรียน {name} เวอร์ชัน {version}"

# f-string รองรับ expression และ format spec ด้วย
price = 199.5
print(f"ราคา: {price:.2f} บาท")           # ราคา: 199.50 บาท
print(f"ผลรวม: {10 + 5}")                  # ผลรวม: 15
print(f"{name.upper()=}")                  # name.upper()='DJANGO' (debug syntax)
```

f-string จะกลายเป็นเพื่อนสนิทของคุณเมื่อเขียน Django เพราะใช้ต่อ query string, debug
message, และแม้แต่ใน template tags บางแบบ

### 11.7 Type Hints ขั้นสูงที่ Django Codebase สมัยใหม่ใช้จริง

```python
from typing import Optional, Union

def get_discount_price(price: float, discount_percent: int = 0) -> float:
    """คำนวณราคาหลังหักส่วนลด"""
    return price * (1 - discount_percent / 100)

# Python 3.10+ ใช้ | แทน Union และ Optional ได้เลย (อ่านง่ายกว่า)
def find_user(user_id: int) -> "User | None":
    ...

def parse_price(value: str | float) -> float:
    return float(value)
```

Django เองเริ่มเพิ่ม type hints เข้าไปในโค้ดหลักมากขึ้นเรื่อย ๆ ตั้งแต่ Django 4
และแพ็กเกจสาย modern เช่น Django REST Framework, Pydantic, FastAPI (ที่บางทีมใช้คู่กับ
Django) ก็พึ่งพา type hints อย่างหนัก การเข้าใจเรื่องนี้ตั้งแต่ต้นจะทำให้คุณอ่านโค้ด
Django เวอร์ชันใหม่ ๆ ได้ลื่นไหลกว่ามาก

### 11.8 ตัวแปร Mutable vs Immutable — กับดักที่ทำให้ Django เกิดบั๊กประหลาด

| ชนิด | Mutable (แก้ไขได้) | Immutable (แก้ไขไม่ได้) |
|---|---|---|
| ตัวอย่าง | `list`, `dict`, `set` | `int`, `str`, `float`, `tuple`, `frozenset` |
| ผลกระทบ | แก้ไขค่าในตัวแปรเดิมได้โดยตรง | ต้องสร้างค่าใหม่เสมอเมื่อ "แก้ไข" |

กับดักคลาสสิกที่เกิดกับ Django (และ Python ทั่วไป) คือการใช้ **mutable default argument**:

```python
# อันตราย! อย่าทำแบบนี้เด็ดขาด
def add_item(item, cart=[]):
    cart.append(item)
    return cart

print(add_item("apple"))   # ['apple']
print(add_item("banana"))  # ['apple', 'banana']  <- คาดหวัง ['banana'] แต่ไม่ใช่!
```

เหตุผลคือ default argument (`cart=[]`) ถูกสร้างขึ้น **ครั้งเดียว** ตอนนิยามฟังก์ชัน
แล้วถูกใช้ซ้ำทุกครั้งที่เรียก วิธีแก้ที่ถูกต้อง:

```python
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart
```

รูปแบบนี้จะกลับมาอีกครั้งตอนเราเขียน Django Forms และ Views ที่รับ default parameter
เป็น list หรือ dict ใน Part หลัง ๆ ของหลักสูตร

---

## ขั้นตอนที่ 12: Functions, Default Arguments, `*args` และ `**kwargs`

### 12.1 ฟังก์ชันพื้นฐานและเหตุผลที่ Django ใช้ฟังก์ชันเป็นหน่วยหลักของ View

Django Function-Based View (FBV) คือฟังก์ชัน Python ธรรมดา ๆ ที่รับ `request` เป็น
argument แรกเสมอ:

```python
def home_view(request):
    return HttpResponse("สวัสดีชาวโลก")
```

ก่อนจะไปเขียน View จริงใน Part 007 เราต้องเข้าใจฟังก์ชันให้แน่นก่อน

### 12.2 องค์ประกอบของฟังก์ชัน

```python
def calculate_total(price, quantity, tax_rate=0.07):
    """
    คำนวณราคารวมสินค้ารวมภาษี

    Args:
        price (float): ราคาต่อหน่วย
        quantity (int): จำนวนที่ซื้อ
        tax_rate (float): อัตราภาษี (ค่าเริ่มต้น 7%)

    Returns:
        float: ราคารวมหลังหักภาษี
    """
    subtotal = price * quantity
    total = subtotal * (1 + tax_rate)
    return round(total, 2)


result = calculate_total(100, 2)              # ใช้ tax_rate ค่า default
result2 = calculate_total(100, 2, tax_rate=0)  # ระบุ tax_rate เอง
```

Docstring (ข้อความในสามเครื่องหมายคำพูด `"""..."""`) ไม่ใช่แค่ comment แต่เป็นเอกสาร
ที่เครื่องมืออย่าง VS Code, Sphinx และ `help()` อ่านได้ Django เองมีเอกสารในรูปแบบนี้
เต็มไปหมดใน source code

### 12.3 Positional Arguments vs Keyword Arguments

```python
def create_user(username, email, is_staff=False):
    return {"username": username, "email": email, "is_staff": is_staff}

# Positional: เรียงตามตำแหน่ง
create_user("somchai", "somchai@example.com")

# Keyword: ระบุชื่อ argument ชัดเจน อ่านง่ายกว่าและปลอดภัยกว่า
create_user(username="somchai", email="somchai@example.com", is_staff=True)

# ผสมกันได้ แต่ positional ต้องมาก่อน keyword เสมอ
create_user("somchai", email="somchai@example.com")
```

ใน Django เราจะเห็น pattern การเรียกแบบ keyword argument อยู่ตลอดเวลา เช่น
`render(request, "template.html", context={"products": products})` หรือ
`Product.objects.filter(price__gte=100, is_active=True)`

### 12.4 `*args`: รับ Positional Arguments จำนวนไม่จำกัด

```python
def sum_all(*args):
    """args จะถูกรวบรวมเป็น tuple โดยอัตโนมัติ"""
    print(type(args))  # <class 'tuple'>
    return sum(args)

print(sum_all(1, 2, 3))          # 6
print(sum_all(1, 2, 3, 4, 5))    # 15
print(sum_all())                  # 0
```

### 12.5 `**kwargs`: รับ Keyword Arguments จำนวนไม่จำกัด

```python
def create_product(**kwargs):
    """kwargs จะถูกรวบรวมเป็น dict โดยอัตโนมัติ"""
    print(type(kwargs))  # <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key}: {value}")

create_product(name="เสื้อยืด", price=299, stock=50)
# name: เสื้อยืด
# price: 299
# stock: 50
```

### 12.6 ทำไม `*args` และ `**kwargs` สำคัญมากใน Django

Django Class-Based Views, Model `save()`, และฟังก์ชันแทบทุกจุดของ framework ใช้รูปแบบนี้
เพื่อ "ส่งต่อ" argument ที่ไม่รู้ล่วงหน้าว่ามีอะไรบ้าง ตัวอย่างที่จะเจอจริงในหลักสูตรนี้
(Part 011 เป็นต้นไป):

```python
class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    def save(self, *args, **kwargs):
        # ทำอะไรบางอย่างก่อนบันทึก เช่น normalize ชื่อสินค้า
        self.name = self.name.strip().title()
        # ส่ง args, kwargs ทั้งหมดต่อให้ save() ดั้งเดิมของ Django
        super().save(*args, **kwargs)
```

โค้ดข้างบนนี้คือ pattern ที่คุณจะเห็นซ้ำ ๆ ตลอดหลักสูตร: **override เมธอดเดิม แต่ยัง
ส่งต่อ argument ทั้งหมดที่อาจถูกส่งมา** (เช่น `force_insert`, `using`) ไปให้เมธอดต้นฉบับ
โดยไม่ต้องรู้ทุกชื่อ parameter ล่วงหน้า

ตัวอย่างที่สองคือ Django REST Framework ViewSet และ Django CBV:

```python
class SignUpView(CreateView):
    def form_valid(self, form):
        response = super().form_valid(form)
        # ทำอะไรเพิ่มเติมหลังจากฟอร์มถูกต้อง เช่นส่งอีเมลต้อนรับ
        return response
```

### 12.7 Keyword-Only Arguments และ Positional-Only Arguments (Python 3.8+)

Python สมัยใหม่ให้เราบังคับได้ว่า argument ตัวไหนต้องเป็น keyword เท่านั้น หรือ
positional เท่านั้น โดยใช้ `*` และ `/`:

```python
def send_email(to, *, subject, body):
    # 'subject' และ 'body' ต้องระบุเป็น keyword เท่านั้น หลัง *
    print(f"ส่งถึง {to}: {subject}")

send_email("user@example.com", subject="ยินดีต้อนรับ", body="...")
# send_email("user@example.com", "ยินดีต้อนรับ", "...")  # Error!

def divide(a, b, /):
    # 'a' และ 'b' ต้องเป็น positional เท่านั้น ก่อน /
    return a / b

divide(10, 2)      # OK
# divide(a=10, b=2)  # Error!
```

Django API หลายจุด (เช่น `get_object_or_404`) ใช้แนวคิดนี้เพื่อบังคับให้โค้ดอ่านง่าย
และป้องกันการเรียกผิดรูปแบบ

### 12.8 Lambda Functions: ฟังก์ชันไม่ระบุชื่อ

```python
# lambda คือฟังก์ชันสั้น ๆ แบบไม่มีชื่อ ใช้ได้บรรทัดเดียว
square = lambda x: x ** 2
print(square(5))  # 25

# ใช้บ่อยที่สุดคู่กับ sorted(), map(), filter()
products = [
    {"name": "เสื้อยืด", "price": 299},
    {"name": "กางเกง", "price": 599},
    {"name": "หมวก", "price": 199},
]

sorted_by_price = sorted(products, key=lambda p: p["price"])
cheap_products = list(filter(lambda p: p["price"] < 300, products))
names = list(map(lambda p: p["name"], products))
```

ใน Django คุณจะเห็น `lambda` ใน `urls.py` บางแบบ (rare), ใน Django ORM key function
สำหรับ sorting ผลลัพธ์ที่ query มาแล้ว และในการตั้งค่า `default=` ของ Model field
บางกรณี (แม้ว่าปกติ Django จะแนะนำให้ใช้ named function มากกว่า lambda สำหรับ Model
field default เพราะ migration ต้องการ reference ที่ serialize ได้)

---

## ขั้นตอนที่ 13: Classes และ Object-Oriented Programming

### 13.1 ทำไม OOP คือหัวใจของ Django

Django คือ framework ที่ออกแบบด้วยแนวคิด **Object-Oriented Programming (OOP)** อย่าง
เข้มข้น แทบทุกอย่างใน Django คือ class:

- **Model** (`class Product(models.Model)`) คือ class ที่แทนตาราง
- **Form** (`class ProductForm(forms.ModelForm)`) คือ class ที่แทนฟอร์ม
- **Class-Based View** (`class ProductListView(ListView)`) คือ class ที่แทน view
- **Middleware**, **Admin**, **Serializer** (ใน DRF) ล้วนเป็น class ทั้งหมด

ถ้าคุณไม่แข็งแรงเรื่อง OOP คุณจะงงกับ Django ทันทีที่เริ่มเขียนโค้ดจริงจัง ดังนั้น
ขั้นตอนนี้คือหนึ่งในขั้นตอนที่สำคัญที่สุดของทั้งหลักสูตร

### 13.2 พื้นฐาน Class: `__init__` และ instance attribute

```python
class Product:
    """ตัวแทนสินค้าหนึ่งชิ้น (คลาสธรรมดา ยังไม่ใช่ Django Model)"""

    def __init__(self, name, price, stock=0):
        self.name = name          # instance attribute
        self.price = price
        self.stock = stock

    def is_in_stock(self):
        return self.stock > 0

    def apply_discount(self, percent):
        self.price = self.price * (1 - percent / 100)


shirt = Product("เสื้อยืด", 299, stock=10)
print(shirt.name)             # เสื้อยืด
print(shirt.is_in_stock())    # True
shirt.apply_discount(10)
print(shirt.price)            # 269.1
```

`__init__` คือ **constructor** ถูกเรียกอัตโนมัติทุกครั้งที่สร้าง instance ใหม่ด้วย
`Product(...)` ส่วน `self` คือตัวแปรที่อ้างถึง instance ปัจจุบันเสมอ (ชื่อ `self` เป็น
ธรรมเนียมของ Python ไม่ใช่คำสงวน แต่ทุกคนใช้ชื่อนี้)

### 13.3 Class Attribute vs Instance Attribute

```python
class Product:
    # Class attribute: ใช้ร่วมกันทุก instance เว้นแต่ถูก override
    category_default = "ทั่วไป"

    def __init__(self, name, price, category=None):
        # Instance attribute: เฉพาะของแต่ละ object
        self.name = name
        self.price = price
        self.category = category or Product.category_default


p1 = Product("เสื้อยืด", 299)
p2 = Product("โน้ตบุ๊ก", 25000, category="อิเล็กทรอนิกส์")
print(p1.category)  # ทั่วไป
print(p2.category)  # อิเล็กทรอนิกส์
```

**สิ่งนี้สำคัญมากกับ Django Model** เพราะ field ที่ประกาศใน class ของ Model
(เช่น `name = models.CharField(...)`) จริง ๆ แล้วเป็น class attribute ที่ Django
ใช้ metaclass แปลงให้กลายเป็น descriptor พิเศษที่ผูกกับฐานข้อมูล — เราจะเห็นกลไกนี้
ชัดเจนขึ้นใน Part 011

### 13.4 Inheritance (การสืบทอด) และ `super()`

Inheritance คือการสร้าง class ใหม่โดย "สืบทอด" คุณสมบัติจาก class เดิม (parent/base
class) แล้วเพิ่มหรือ override พฤติกรรมบางอย่าง:

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return f"{self.name} ส่งเสียง..."


class Dog(Animal):
    def speak(self):
        return f"{self.name} เห่า: โฮ่ง โฮ่ง!"


class Cat(Animal):
    def speak(self):
        # เรียกเมธอดของ parent class ด้วย super() แล้วเพิ่มเติม
        base_message = super().speak()
        return f"{base_message} (จริง ๆ คือ เหมียว)"


dog = Dog("โปเก้")
cat = Cat("มะลิ")
print(dog.speak())  # โปเก้ เห่า: โฮ่ง โฮ่ง!
print(cat.speak())  # มะลิ ส่งเสียง... (จริง ๆ คือ เหมียว)
```

`super()` คือฟังก์ชันที่ให้เราเข้าถึงเมธอดของ parent class ได้ นี่คือกลไกที่สำคัญที่สุด
อย่างหนึ่งใน Django เพราะ Django เกือบทุก class ที่คุณเขียนคือการ **inherit** จาก class
ของ Django เอง แล้วเรียก `super()` เพื่อ "ต่อยอด" พฤติกรรมเดิม ไม่ใช่แทนที่มันทั้งหมด:

```python
from django.db import models

class TimestampedModel(models.Model):
    """Abstract base class ที่เพิ่ม timestamp ให้ Model อื่น"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Product(TimestampedModel):
    # Product สืบทอด created_at, updated_at มาจาก TimestampedModel โดยอัตโนมัติ
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
```

และตัวอย่าง Class-Based View ที่จะเจอใน Part 021-024:

```python
from django.views.generic import ListView

class ProductListView(ListView):
    model = Product
    template_name = "products/list.html"

    def get_queryset(self):
        # ดึงพฤติกรรมเดิมของ ListView มาก่อน แล้วปรับแต่งเพิ่ม
        queryset = super().get_queryset()
        return queryset.filter(is_active=True)
```

### 13.5 Method Resolution Order (MRO) และ Multiple Inheritance

Python รองรับการสืบทอดจากหลาย class พร้อมกัน (multiple inheritance) ซึ่ง Django
Class-Based Views ใช้อย่างหนักผ่านแนวคิดที่เรียกว่า **Mixin**:

```python
class LoginRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("login")
        return super().dispatch(request, *args, **kwargs)


class StaffRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_staff:
            raise PermissionDenied
        return super().dispatch(request, *args, **kwargs)


class AdminDashboardView(LoginRequiredMixin, StaffRequiredMixin, ListView):
    model = Product
```

เมื่อ Python ต้องหาว่าเมธอดไหนควรถูกเรียกก่อน มันใช้อัลกอริทึม **C3 Linearization**
เรียงลำดับตาม **MRO (Method Resolution Order)** ตรวจสอบได้ด้วย:

```python
print(AdminDashboardView.__mro__)
# หรือ
print(AdminDashboardView.mro())
```

ลำดับการ inherit ใน `class AdminDashboardView(LoginRequiredMixin, StaffRequiredMixin, ListView)`
สำคัญมาก — Mixin ต้องเขียนไว้ **ก่อน** class หลักเสมอ เพราะ `super()` จะไล่เรียกไปตาม
ลำดับนี้ นี่คือกลไกเบื้องหลัง `LoginRequiredMixin` ของ Django จริง ๆ ที่เราจะใช้ใน
Part 024 และ Part 031

### 13.6 Magic Methods (Dunder Methods) ที่สำคัญที่สุด

**Magic methods** (หรือ dunder methods เพราะขึ้นต้น-ลงท้ายด้วย double underscore `__`)
คือเมธอดพิเศษที่ทำให้ object ของเราทำงานร่วมกับ syntax มาตรฐานของ Python ได้
(เช่น `print()`, `+`, `==`, `len()`)

```python
class Money:
    def __init__(self, amount, currency="THB"):
        self.amount = amount
        self.currency = currency

    def __str__(self):
        """เรียกเมื่อใช้ str(obj) หรือ print(obj) — ใช้แสดงผลแบบ human-readable"""
        return f"{self.amount:,.2f} {self.currency}"

    def __repr__(self):
        """เรียกเมื่อพิมพ์ obj ใน console/debugger — ใช้ debug (ควรทำให้สร้าง object ซ้ำได้)"""
        return f"Money(amount={self.amount!r}, currency={self.currency!r})"

    def __eq__(self, other):
        """เรียกเมื่อใช้ == เปรียบเทียบ"""
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount and self.currency == other.currency

    def __add__(self, other):
        """เรียกเมื่อใช้ +"""
        if self.currency != other.currency:
            raise ValueError("ไม่สามารถบวกเงินคนละสกุลได้")
        return Money(self.amount + other.amount, self.currency)

    def __lt__(self, other):
        """เรียกเมื่อใช้ < ทำให้ sorted() ใช้งานกับ Money ได้"""
        return self.amount < other.amount


m1 = Money(100)
m2 = Money(50)
print(m1)                 # 100.00 THB   (เรียก __str__)
print(m1 + m2)             # 150.00 THB   (เรียก __add__)
print(m1 == Money(100))    # True         (เรียก __eq__)
print(sorted([m1, m2]))    # เรียงจากน้อยไปมาก (เรียก __lt__)
```

### 13.7 ทำไม `__str__` สำคัญกับ Django Model มากเป็นพิเศษ

ทุก Model ใน Django **ควร** มีเมธอด `__str__` เสมอ เพราะ Django ใช้มันแสดงผลใน
Admin Panel, ใน shell, และในหลาย ๆ ที่ที่ต้องแปลง object เป็นข้อความ:

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    def __str__(self):
        return self.name   # ถ้าไม่มีเมธอดนี้ Admin จะแสดง "Product object (1)" ซึ่งไม่มีประโยชน์
```

ลองจินตนาการหน้า Django Admin ที่แสดงรายการสินค้าเป็น `Product object (1)`,
`Product object (2)` แทนที่จะเป็นชื่อสินค้าจริง — นี่คือเหตุผลที่ `__str__` เป็น
"มารยาทพื้นฐาน" ของทุก Model ที่มืออาชีพต้องใส่

### 13.8 ตารางสรุป Magic Methods ที่ใช้บ่อยและจุดเชื่อมกับ Django

| Magic Method | ถูกเรียกเมื่อ | ตัวอย่างการใช้ใน Django |
|---|---|---|
| `__init__` | สร้าง object ใหม่ | ทุก Model, Form, View class |
| `__str__` | `str(obj)`, `print(obj)`, Admin panel | แสดงชื่อ record ใน Admin |
| `__repr__` | debug console, `repr(obj)` | debugging QuerySet |
| `__eq__` | `==` | เปรียบเทียบ instance ของ Model |
| `__len__` | `len(obj)` | `len(queryset)` |
| `__iter__` | `for x in obj` | วน loop ผ่าน QuerySet |
| `__getitem__` | `obj[key]` | `queryset[0:10]` (slicing) |
| `__call__` | `obj()` | Middleware ของ Django (เป็น callable object) |

### 13.9 Property Decorator: ทำให้เมธอดทำงานเหมือน Attribute

```python
class Product:
    def __init__(self, price, discount_percent=0):
        self.price = price
        self.discount_percent = discount_percent

    @property
    def final_price(self):
        """เข้าถึงเหมือน attribute (product.final_price) แต่จริง ๆ คำนวณสด"""
        return self.price * (1 - self.discount_percent / 100)

    @final_price.setter
    def final_price(self, value):
        # ให้ตั้งค่าย้อนกลับได้ด้วย (ไม่บังคับต้องมี)
        self.price = value / (1 - self.discount_percent / 100)


p = Product(1000, discount_percent=10)
print(p.final_price)   # 900.0  (ไม่ต้องเรียก p.final_price() แบบเมธอด)
```

`@property` คือรากฐานของแนวคิด **model property** ที่จะเจอบ่อยมากใน Django Model
เพื่อสร้าง field ที่คำนวณจากค่าอื่น ๆ โดยไม่ต้องเก็บลงฐานข้อมูลจริง:

```python
class Order(models.Model):
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)

    @property
    def total_price(self):
        return self.quantity * self.unit_price
```

### 13.10 Class Method และ Static Method

```python
class Product:
    tax_rate = 0.07

    def __init__(self, name, price):
        self.name = name
        self.price = price

    @classmethod
    def from_dict(cls, data):
        """Alternative constructor — สร้าง instance จาก dict"""
        return cls(name=data["name"], price=data["price"])

    @staticmethod
    def calculate_tax(price):
        """ไม่ต้องพึ่ง self หรือ cls เลย เป็นแค่ฟังก์ชัน utility ที่จัดกลุ่มไว้ใน class"""
        return price * Product.tax_rate


data = {"name": "เสื้อยืด", "price": 299}
p = Product.from_dict(data)
print(Product.calculate_tax(100))  # 7.0
```

`classmethod` คือ pattern ที่ Django ORM ใช้หนักมาก เช่น `Product.objects.create(...)`
ภายในเป็นกลไกคล้าย classmethod ที่สร้าง instance ใหม่แล้วบันทึกลงฐานข้อมูลในคำสั่งเดียว
เราจะเจอรายละเอียดนี้ใน Part 011-013

---

## ขั้นตอนที่ 14: Decorators — วิธีทำงานและทำไม Django ใช้เยอะมาก

### 14.1 Function เป็น First-Class Citizen

ก่อนเข้าใจ decorator ต้องเข้าใจก่อนว่าใน Python **ฟังก์ชันคือ object ชนิดหนึ่ง**
สามารถส่งผ่านตัวแปร ส่งเป็น argument หรือ return จากฟังก์ชันอื่นได้:

```python
def greet(name):
    return f"สวัสดี {name}"

say_hello = greet          # เก็บฟังก์ชันไว้ในตัวแปรอื่นได้ (ไม่ต้องมี ())
print(say_hello("โลก"))     # สวัสดี โลก

def apply_function(func, value):
    return func(value)      # ส่งฟังก์ชันเป็น argument ได้

print(apply_function(greet, "Django"))  # สวัสดี Django
```

### 14.2 Closure: ฟังก์ชันที่ "จำ" ตัวแปรภายนอกได้

```python
def make_multiplier(factor):
    def multiplier(number):
        return number * factor   # 'factor' มาจาก scope ภายนอก แต่ยังจำได้
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)
print(double(5))   # 10
print(triple(5))   # 15
```

`multiplier` คือ **closure** เพราะมันจดจำค่า `factor` จาก scope ที่มันถูกสร้างขึ้น
แม้ `make_multiplier` จะทำงานเสร็จไปแล้ว นี่คือกลไกพื้นฐานที่ decorator ทำงานอยู่บนนั้น

### 14.3 สร้าง Decorator ตัวแรกด้วยตัวเอง

**Decorator** คือฟังก์ชันที่รับฟังก์ชันอื่นเป็น input แล้ว return ฟังก์ชันใหม่ที่
"ห่อ" (wrap) พฤติกรรมเพิ่มเติมรอบฟังก์ชันเดิม โดยไม่ต้องแก้โค้ดต้นฉบับ:

```python
import time
from functools import wraps

def timer(func):
    """Decorator ที่วัดเวลาการทำงานของฟังก์ชัน"""
    @wraps(func)  # เก็บชื่อ/docstring ของฟังก์ชันเดิมไว้ (สำคัญมาก อย่าลืม!)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} ใช้เวลา {end - start:.4f} วินาที")
        return result
    return wrapper


@timer
def slow_calculation(n):
    return sum(i ** 2 for i in range(n))


result = slow_calculation(1_000_000)
# slow_calculation ใช้เวลา 0.1234 วินาที
```

`@timer` ด้านบนฟังก์ชัน `slow_calculation` เป็นแค่ **syntactic sugar** ของ:

```python
slow_calculation = timer(slow_calculation)
```

การใช้ `@functools.wraps(func)` สำคัญมาก เพราะถ้าไม่ใส่ `wrapper.__name__` จะกลาย
เป็น `"wrapper"` แทนที่จะเป็นชื่อฟังก์ชันเดิม ซึ่งจะทำให้ debug และ introspect โค้ด
ยากขึ้นมาก — Django และไลบรารีคุณภาพสูงทุกตัวใช้ `@wraps` เสมอ

### 14.4 Decorator ที่รับ Argument ได้ (Decorator Factory)

```python
def repeat(times):
    """Decorator factory: รับ argument แล้วคืน decorator จริง ๆ อีกที"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator


@repeat(times=3)
def send_notification(message):
    print(f"แจ้งเตือน: {message}")


send_notification("มีคำสั่งซื้อใหม่")
# แจ้งเตือน: มีคำสั่งซื้อใหม่   (พิมพ์ 3 ครั้ง)
```

รูปแบบนี้คือโครงสร้างเดียวกับ decorator ของ Django ที่รับ argument เช่น
`@permission_required("app.can_edit_product")` และ `@cache_page(60 * 15)`

### 14.5 นี่คือของจริง: `@login_required` ทำงานอย่างไรเบื้องหลัง

ตอนนี้เราพร้อมเข้าใจกลไกที่แท้จริงของ decorator ที่โด่งดังที่สุดใน Django แล้ว
ต่อไปนี้คือเวอร์ชันแบบง่าย (simplified) ของ `login_required` เพื่อสอนแนวคิด — โค้ดจริง
ของ Django ซับซ้อนกว่านี้เล็กน้อย แต่หลักการเดียวกันทุกประการ:

```python
from functools import wraps
from django.shortcuts import redirect

def login_required_demo(view_func):
    """เวอร์ชันจำลองของ django.contrib.auth.decorators.login_required"""
    @wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("login")     # เตะกลับไปหน้า login ถ้ายังไม่ล็อกอิน
        return view_func(request, *args, **kwargs)  # ทำงานตามปกติถ้าล็อกอินแล้ว
    return wrapper


@login_required_demo
def dashboard_view(request):
    return render(request, "dashboard.html")
```

เมื่อคุณเขียน `@login_required` เหนือ view ใน Django ของจริง (Part 031) มันทำสิ่งเดียวกัน
นี้เป๊ะ ๆ: **ตรวจสอบเงื่อนไขบางอย่างก่อน แล้วค่อยตัดสินใจว่าจะรัน view จริงหรือ redirect
ไปที่อื่น** decorator ยอดนิยมอื่น ๆ ของ Django ก็ใช้หลักการเดียวกันนี้ทั้งหมด:

| Decorator ของ Django | ทำงานอย่างไร (concept เดียวกับที่เพิ่งเรียน) |
|---|---|
| `@login_required` | เช็ค `request.user.is_authenticated` ก่อนรัน view |
| `@permission_required("app.can_edit")` | เช็ค permission ก่อนรัน view |
| `@require_http_methods(["POST"])` | เช็ค `request.method` ก่อนรัน view |
| `@csrf_exempt` | ปิดการตรวจสอบ CSRF token สำหรับ view นี้ |
| `@cache_page(60 * 15)` | เก็บผลลัพธ์ของ view ไว้ cache 15 นาที |
| `@transaction.atomic` | ห่อทั้งฟังก์ชันด้วย database transaction |

### 14.6 การใช้ Decorator หลายตัวซ้อนกัน (Stacking)

```python
@login_required
@require_http_methods(["GET", "POST"])
def edit_profile(request):
    ...
```

ลำดับสำคัญมาก! decorator ที่อยู่ **ใกล้ฟังก์ชันที่สุด** จะถูกเรียกก่อน (ทำงานจากล่างขึ้นบน)
เทียบเท่ากับ:

```python
edit_profile = login_required(require_http_methods(["GET", "POST"])(edit_profile))
```

### 14.7 Class-based Decorator (คั่นความรู้)

Decorator ไม่จำเป็นต้องเป็นฟังก์ชันเสมอไป สามารถเป็น class ที่มี `__call__` ได้:

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"เรียกฟังก์ชัน {self.func.__name__} ครั้งที่ {self.count}")
        return self.func(*args, **kwargs)


@CountCalls
def process_order(order_id):
    print(f"ประมวลผล order {order_id}")


process_order(1)  # เรียกฟังก์ชัน process_order ครั้งที่ 1 / ประมวลผล order 1
process_order(2)  # เรียกฟังก์ชัน process_order ครั้งที่ 2 / ประมวลผล order 2
```

Django Middleware สมัยใหม่ (ตั้งแต่ Django 1.10+) มีโครงสร้างคล้ายกันมาก — เป็น
callable ที่รับ `get_response` ตอน `__init__` แล้วมี `__call__` เพื่อประมวลผล
request/response เราจะเรียนรายละเอียดนี้ใน Phase หลัง ๆ

---

## ขั้นตอนที่ 15: Context Managers และ `with` statement

### 15.1 ปัญหาที่ Context Manager แก้: การจัดการทรัพยากรอย่างปลอดภัย

ลองดูปัญหาคลาสสิกของการเปิดไฟล์:

```python
# วิธีอันตราย: ถ้าเกิด error ระหว่างอ่านไฟล์ ไฟล์จะไม่ถูกปิด (resource leak)
f = open("data.txt")
content = f.read()
f.close()  # ถ้าบรรทัดก่อนหน้า error บรรทัดนี้จะไม่ถูกรัน!
```

`with` statement แก้ปัญหานี้โดยรับประกันว่าทรัพยากรจะถูกปิด/คืน **เสมอ** ไม่ว่าจะเกิด
error หรือไม่:

```python
with open("data.txt") as f:
    content = f.read()
# ไฟล์ถูกปิดอัตโนมัติทันทีที่ออกจาก block นี้ ไม่ว่าจะสำเร็จหรือ error
```

### 15.2 สร้าง Context Manager ด้วย Class (`__enter__` / `__exit__`)

```python
class DatabaseConnection:
    def __enter__(self):
        print("เปิดการเชื่อมต่อฐานข้อมูล...")
        self.connection = "connection_object"
        return self.connection

    def __exit__(self, exc_type, exc_value, traceback):
        print("ปิดการเชื่อมต่อฐานข้อมูล...")
        # ถ้า return True จะกลืน exception ไม่ให้ raise ต่อ (ปกติไม่ควรทำ)
        return False


with DatabaseConnection() as conn:
    print(f"ใช้งาน {conn}")
    # ทำงานกับฐานข้อมูล

# ผลลัพธ์:
# เปิดการเชื่อมต่อฐานข้อมูล...
# ใช้งาน connection_object
# ปิดการเชื่อมต่อฐานข้อมูล...
```

`__exit__` รับ 3 argument คือ `exc_type`, `exc_value`, `traceback` ซึ่งจะไม่เป็น
`None` ถ้าเกิด exception ขึ้นภายใน `with` block — ทำให้ context manager สามารถ
"รู้" ได้ว่าเกิดข้อผิดพลาดขึ้นหรือไม่ และตัดสินใจว่าจะ commit หรือ rollback

### 15.3 สร้าง Context Manager แบบง่ายด้วย `contextlib`

```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    print("เปิดการเชื่อมต่อฐานข้อมูล...")
    connection = "connection_object"
    try:
        yield connection   # ค่าที่ได้จาก 'as' variable
    finally:
        print("ปิดการเชื่อมต่อฐานข้อมูล...")  # รันเสมอไม่ว่า error หรือไม่


with database_connection() as conn:
    print(f"ใช้งาน {conn}")
```

`yield` แบ่งฟังก์ชันเป็น 2 ส่วน: ก่อน `yield` คือโค้ดที่รันตอนเข้า `with` (เทียบเท่า
`__enter__`) และหลัง `yield` (ใน `finally`) คือโค้ดที่รันตอนออกจาก `with`
(เทียบเท่า `__exit__`)

### 15.4 นี่คือของจริง: `transaction.atomic()` ของ Django

ตอนนี้เราพร้อมเข้าใจฟีเจอร์สำคัญที่สุดอย่างหนึ่งของ Django ORM: **Database
Transaction** ซึ่งใช้ context manager pattern โดยตรง:

```python
from django.db import transaction

def transfer_money(from_account, to_account, amount):
    with transaction.atomic():
        # ทุกคำสั่งในนี้จะถูกทำเป็น "หน่วยเดียว" (all-or-nothing)
        from_account.balance -= amount
        from_account.save()

        to_account.balance += amount
        to_account.save()
        # ถ้าเกิด error ตรงไหนก็ตามในนี้ Django จะ ROLLBACK ทุกอย่างอัตโนมัติ
        # ทำให้ยอดเงินไม่มีทาง "หายไป" ครึ่งทาง
```

ถ้าไม่ใช้ `transaction.atomic()` และเกิด error หลังจากหักเงินจาก `from_account`
ไปแล้วแต่ก่อนที่จะเพิ่มเงินให้ `to_account` เงินจะ "หายไปจากระบบ" กลางทาง — นี่คือ
เหตุผลที่ context manager สำคัญมากกับงานการเงินและ Django มี built-in support
ให้เต็มรูปแบบ เราจะเรียนเรื่องนี้ลึกกว่านี้ใน Phase 2 (Models & ORM)

`transaction.atomic()` ยังใช้เป็น decorator ได้ด้วย (เพราะเบื้องหลังใช้กลไกเดียวกับ
decorator ที่เราเพิ่งเรียนในขั้นตอนที่ 14):

```python
@transaction.atomic
def transfer_money(from_account, to_account, amount):
    from_account.balance -= amount
    from_account.save()
    to_account.balance += amount
    to_account.save()
```

### 15.5 ตัวอย่าง Context Manager อื่น ๆ ที่ใช้บ่อยใน Django Ecosystem

```python
# 1. เปิด/ปิดไฟล์ (พื้นฐานที่สุด)
with open("export.csv", "w", encoding="utf-8") as f:
    f.write("name,price\n")

# 2. ล็อกไฟล์เพื่อป้องกัน race condition (ใช้ใน background jobs)
from threading import Lock
lock = Lock()

def safe_update_counter():
    with lock:
        # โค้ดในนี้จะรันได้ทีละ thread เท่านั้น
        pass

# 3. เปลี่ยนการตั้งค่าชั่วคราวแล้วคืนค่าเดิมอัตโนมัติ (พบใน Django test)
from django.test import override_settings

@override_settings(DEBUG=True)
def test_something():
    ...

# หรือใช้เป็น context manager ตรง ๆ
with override_settings(LANGUAGE_CODE="th"):
    # โค้ดในนี้ทำงานเหมือนตั้ง LANGUAGE_CODE="th" ชั่วคราว
    pass
# ออกจาก with แล้ว ค่ากลับไปเป็นเหมือนเดิมอัตโนมัติ
```

### 15.6 การจัดการหลาย Context Manager พร้อมกัน

```python
with open("input.csv") as infile, open("output.csv", "w") as outfile:
    for line in infile:
        outfile.write(line.upper())
```

รูปแบบนี้เทียบเท่ากับการซ้อน `with` สองชั้น แต่เขียนกระชับกว่า ใน Django คุณจะเห็น
รูปแบบคล้ายกันเวลาต้องเปิด transaction ซ้อนกับการล็อก cache หรือไฟล์พร้อมกัน

---

## ขั้นตอนที่ 16: List/Dict/Set Comprehension

### 16.1 ทำไมต้องใช้ Comprehension แทน Loop ธรรมดา

Comprehension คือไวยากรณ์กระชับสำหรับสร้าง collection ใหม่จาก collection เดิม
อ่านง่ายกว่าและมักจะเร็วกว่า loop ธรรมดาเล็กน้อยด้วย

```python
# วิธีเดิมด้วย for loop
squares = []
for x in range(10):
    squares.append(x ** 2)

# วิธีเดียวกันด้วย List Comprehension — บรรทัดเดียว อ่านง่ายกว่า
squares = [x ** 2 for x in range(10)]
```

### 16.2 List Comprehension แบบมีเงื่อนไข

```python
numbers = range(20)

# กรองด้วย if
even_numbers = [n for n in numbers if n % 2 == 0]

# if-else ใน expression (ต้องเขียนก่อน for)
labels = ["คู่" if n % 2 == 0 else "คี่" for n in numbers]

# ซ้อนหลายเงื่อนไข
filtered = [n for n in numbers if n % 2 == 0 if n > 10]  # เทียบเท่า n % 2 == 0 and n > 10
```

### 16.3 ตัวอย่างที่เชื่อมกับ Django จริง: จัดการ QuerySet

```python
# สมมติว่า products คือ QuerySet ที่ได้จาก Product.objects.all()
products = Product.objects.all()

# สร้าง list ของชื่อสินค้าทั้งหมด
product_names = [p.name for p in products]

# สร้าง list เฉพาะสินค้าที่ยังมีสต๊อก
in_stock = [p for p in products if p.stock > 0]

# สร้าง choices สำหรับ dropdown ใน Django Form
category_choices = [(c.id, c.name) for c in Category.objects.all()]

# สร้าง dict สรุปยอดสินค้าตามหมวดหมู่ (ตัวอย่างการประมวลผลข้อมูลก่อนส่งเป็น JSON)
price_by_name = {p.name: float(p.price) for p in products}
```

**ข้อควรระวังสำคัญ**: การวน loop ผ่าน QuerySet แบบนี้ในโค้ดจริงต้องระวังเรื่อง
**N+1 query problem** ถ้ามีการเข้าถึง relation ของแต่ละ object ในลูป (เช่น
`p.category.name`) จะยิง query ไปฐานข้อมูลทุกครั้งซ้ำ ๆ วิธีแก้ (`select_related`,
`prefetch_related`) เราจะเรียนละเอียดใน Part 067 (Query Optimization) — แต่ให้จำ
concept นี้ไว้ตั้งแต่ตอนนี้ เพราะ comprehension คือจุดที่ปัญหานี้มักเกิดขึ้นบ่อยที่สุด

### 16.4 Dict Comprehension

```python
words = ["python", "django", "web", "api"]

# สร้าง dict จาก key/value expression
word_lengths = {word: len(word) for word in words}
# {'python': 6, 'django': 6, 'web': 3, 'api': 3}

# สลับ key กับ value
original = {"a": 1, "b": 2, "c": 3}
swapped = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}

# กรองข้อมูลจาก request.POST (จำลอง) เอาเฉพาะ field ที่ขึ้นต้นด้วย "product_"
form_data = {"product_name": "เสื้อยืด", "product_price": "299", "csrf_token": "xyz"}
product_fields = {k: v for k, v in form_data.items() if k.startswith("product_")}
```

### 16.5 Set Comprehension

```python
emails = ["a@x.com", "b@x.com", "a@x.com", "c@x.com"]

# สร้าง set เพื่อกำจัดค่าซ้ำ
unique_emails = {email for email in emails}
# {'a@x.com', 'b@x.com', 'c@x.com'}

# ใช้บ่อยเพื่อหา permission ทั้งหมดของ user จากหลาย group
user_groups_permissions = [{"add_product", "view_product"}, {"delete_product"}]
all_permissions = {perm for group_perms in user_groups_permissions for perm in group_perms}
```

### 16.6 Nested Comprehension (ระวังการอ่านยาก)

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

# แปลง matrix 2 มิติให้เป็น list เดียว (flatten)
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

**คำแนะนำระดับมืออาชีพ**: comprehension ที่ซ้อนเกิน 2 ชั้น หรือมีเงื่อนไขซับซ้อนเกินไป
ควรเปลี่ยนกลับไปใช้ for loop ธรรมดา เพื่อความอ่านง่าย — โค้ดที่อ่านยากคือหนี้เทคนิค
(technical debt) แม้จะรันได้ถูกต้องก็ตาม

### 16.7 Generator Expression: Comprehension แบบประหยัดหน่วยความจำ

```python
# List comprehension: สร้างทุกค่าทันทีเก็บไว้ใน memory ทั้งหมด
squares_list = [x ** 2 for x in range(1_000_000)]   # ใช้ memory เยอะ

# Generator expression: ใช้ () แทน [] สร้างค่าทีละตัวตามต้องการ (lazy evaluation)
squares_gen = (x ** 2 for x in range(1_000_000))     # ใช้ memory น้อยมาก

print(sum(squares_gen))  # ยังใช้งานได้ปกติกับฟังก์ชันที่วนอ่านค่าได้ เช่น sum(), any(), all()
```

เราจะเจาะลึกเรื่อง generator ในขั้นตอนถัดไป เพราะมันเชื่อมโยงกับวิธีที่ Django ORM
จัดการ QuerySet ขนาดใหญ่แบบมีประสิทธิภาพ (`.iterator()`)

---

## ขั้นตอนที่ 17: Iterators และ Generators

### 17.1 Iterable vs Iterator: ความแตกต่างที่ต้องเข้าใจให้ชัด

- **Iterable**: object ที่วน loop ได้ (มีเมธอด `__iter__`) เช่น `list`, `dict`, `str`,
  `range`, และ **QuerySet ของ Django**
- **Iterator**: object ที่จำ "ตำแหน่งปัจจุบัน" ได้ และดึงค่าถัดไปได้ทีละตัว
  (มีเมธอด `__iter__` และ `__next__`)

```python
numbers = [1, 2, 3]        # numbers เป็น iterable
iterator = iter(numbers)   # แปลงเป็น iterator ด้วยฟังก์ชัน iter()

print(next(iterator))  # 1
print(next(iterator))  # 2
print(next(iterator))  # 3
print(next(iterator))  # StopIteration error! ไม่มีค่าเหลือแล้ว
```

`for` loop ใน Python จริง ๆ แล้วทำงานแบบนี้อยู่เบื้องหลัง: เรียก `iter()` แล้วเรียก
`next()` ซ้ำ ๆ จนกว่าจะเจอ `StopIteration` แล้วหยุด loop ให้อัตโนมัติ

### 17.2 สร้าง Iterator ของตัวเองด้วย Class

```python
class CountDown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self  # ตัวมันเองเป็น iterator

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1


for number in CountDown(5):
    print(number)  # 5, 4, 3, 2, 1
```

### 17.3 Generator Functions: วิธีที่ง่ายกว่ามากในการสร้าง Iterator

การเขียน class iterator เองค่อนข้างยาว Python มี **generator function** ที่ใช้
`yield` แทน `return` ทำให้เขียนได้กระชับกว่ามาก:

```python
def countdown(start):
    current = start
    while current > 0:
        yield current
        current -= 1


for number in countdown(5):
    print(number)  # 5, 4, 3, 2, 1
```

เมื่อฟังก์ชันมี `yield` แม้แต่ตัวเดียว Python จะเปลี่ยนมันเป็น **generator function**
โดยอัตโนมัติ การเรียกฟังก์ชันนี้จะไม่รันโค้ดทันที แต่จะคืน **generator object** ที่
"หยุดรอ" ตรง `yield` แต่ละครั้งที่ถูกเรียก `next()`

```python
gen = countdown(3)
print(gen)          # <generator object countdown at 0x...>
print(next(gen))     # 3  (รันจนถึง yield แรก แล้วหยุด)
print(next(gen))     # 2  (รันต่อจากจุดที่ค้างไว้)
print(next(gen))     # 1
print(next(gen))     # StopIteration
```

### 17.4 ทำไม Generator ประหยัด Memory: Lazy Evaluation

```python
def read_large_file(file_path):
    """อ่านไฟล์ทีละบรรทัด ไม่โหลดทั้งไฟล์เข้า memory พร้อมกัน"""
    with open(file_path) as f:
        for line in f:
            yield line.strip()


# ไฟล์อาจมีขนาด 10 GB แต่โปรแกรมใช้ memory แค่ทีละบรรทัด
for line in read_large_file("huge_log_file.txt"):
    if "ERROR" in line:
        print(line)
```

นี่คือหลักการเดียวกับที่ Django ORM ใช้ในเมธอด `.iterator()` เมื่อต้องประมวลผล
ข้อมูลจำนวนมหาศาลจากฐานข้อมูล:

```python
# วิธีปกติ: Django จะดึงข้อมูลทั้งหมดมาเก็บใน memory เพื่อ cache QuerySet
for product in Product.objects.all():
    process(product)

# เมื่อข้อมูลมีหลักล้าน record ใช้ .iterator() เพื่อประหยัด memory มหาศาล
for product in Product.objects.all().iterator(chunk_size=2000):
    process(product)
    # Django จะดึงข้อมูลจากฐานข้อมูลเป็น "ก้อน" (chunk) ทีละ 2000 แถว
    # แทนที่จะโหลดทั้งหมดพร้อมกัน (คล้ายกับ generator ที่ประมวลผลทีละส่วน)
```

เราจะเรียนเรื่อง `.iterator()` และการจัดการข้อมูลขนาดใหญ่แบบมืออาชีพใน Part 071
(Pagination และ Large Dataset Handling) แต่แนวคิดพื้นฐาน (lazy evaluation) ที่คุณ
เรียนตอนนี้คือรากฐานของมันทั้งหมด

### 17.5 QuerySet คือ Lazy Object เหมือน Generator (แนวคิดสำคัญที่สุดของ Django ORM)

นี่คือหนึ่งในแนวคิดที่สำคัญที่สุดที่คุณต้องเข้าใจก่อนไปเจอ Django ORM จริงใน Phase 2:

```python
# บรรทัดนี้ยัง "ไม่" ยิง query ไปฐานข้อมูล! (lazy evaluation เหมือน generator)
queryset = Product.objects.filter(price__gte=100)

# การ filter ต่อก็ยังไม่ยิง query
queryset = queryset.filter(is_active=True).order_by("-created_at")

# Query จะถูกยิงไปฐานข้อมูลจริง ๆ ก็ต่อเมื่อถูก "evaluate" เท่านั้น เช่น:
for product in queryset:      # 1. วน loop
    print(product.name)

list(queryset)                 # 2. แปลงเป็น list
len(queryset)                  # 3. เช็คความยาว
if queryset:                   # 4. ใช้ใน condition
    ...
```

การที่ QuerySet เป็น **lazy** (ไม่ทำงานทันทีที่ถูกสร้าง แต่รอจนกว่าจะถูก "บริโภค"
จริง ๆ) คือหลักการเดียวกับ generator ที่เราเพิ่งเรียน สิ่งนี้ทำให้ Django สามารถ
"เชน" (chain) เงื่อนไข filter หลายอันเข้าด้วยกันได้อย่างมีประสิทธิภาพ โดยสร้าง SQL
query สุดท้ายเพียงครั้งเดียวตอนที่ข้อมูลถูกใช้งานจริง แทนที่จะยิง query ทุกครั้งที่
เรียก `.filter()`

### 17.6 `itertools`: คลังเครื่องมือ Iterator ระดับมืออาชีพ

```python
from itertools import chain, groupby, islice

# chain: รวมหลาย iterable เป็นตัวเดียว โดยไม่ต้องสร้าง list ใหม่
active_products = [1, 2, 3]
archived_products = [4, 5]
all_ids = list(chain(active_products, archived_products))  # [1, 2, 3, 4, 5]

# islice: ตัดเอาบางส่วนของ iterator (ใช้ทำ pagination แบบ manual ได้)
first_three = list(islice(range(100), 3))  # [0, 1, 2]

# groupby: จัดกลุ่มข้อมูลที่เรียงไว้แล้วตาม key (ต้อง sort ก่อนเสมอ)
products = [
    {"category": "เสื้อผ้า", "name": "เสื้อยืด"},
    {"category": "เสื้อผ้า", "name": "กางเกง"},
    {"category": "อิเล็กทรอนิกส์", "name": "โน้ตบุ๊ก"},
]
products.sort(key=lambda p: p["category"])
for category, group in groupby(products, key=lambda p: p["category"]):
    print(category, [p["name"] for p in group])
```

---

## ขั้นตอนที่ 18: Exception Handling

### 18.1 ทำไม Exception Handling สำคัญกับ Web Application

เว็บแอปพลิเคชันต้อง **ไม่ล่มทั้งระบบ** เพียงเพราะ request หนึ่งตัวมีปัญหา เช่น
ถ้า user ส่งข้อมูลผิดรูปแบบมา หรือฐานข้อมูลหลุดการเชื่อมต่อชั่วคราว แอปพลิเคชัน
ควรจัดการอย่างสุภาพ (แสดงหน้า error ที่เหมาะสม) ไม่ใช่ crash ทั้งเซิร์ฟเวอร์

### 18.2 โครงสร้างพื้นฐาน try/except/else/finally

```python
def divide_numbers(a, b):
    try:
        result = a / b
    except ZeroDivisionError:
        print("ไม่สามารถหารด้วยศูนย์ได้")
        result = None
    except TypeError:
        print("ต้องใส่ตัวเลขเท่านั้น")
        result = None
    else:
        # รันเมื่อ try สำเร็จ ไม่มี exception เกิดขึ้น
        print("คำนวณสำเร็จ")
    finally:
        # รันเสมอไม่ว่าจะสำเร็จหรือ error (เหมาะกับ cleanup code)
        print("จบการทำงานของฟังก์ชัน")

    return result


print(divide_numbers(10, 2))   # คำนวณสำเร็จ / จบการทำงานของฟังก์ชัน / 5.0
print(divide_numbers(10, 0))   # ไม่สามารถหารด้วยศูนย์ได้ / จบการทำงานของฟังก์ชัน / None
```

### 18.3 จับ Exception หลายชนิดพร้อมกัน และเข้าถึงรายละเอียด error

```python
try:
    value = int(input_string)
except (ValueError, TypeError) as e:
    print(f"เกิดข้อผิดพลาด: {e}")
    print(f"ชนิดของ error: {type(e).__name__}")
```

**ข้อควรระวังระดับมืออาชีพ**: อย่าใช้ `except:` เปล่า ๆ (bare except) โดยไม่ระบุชนิด
เพราะมันจะดักจับทุกอย่างรวมถึง `KeyboardInterrupt` และ `SystemExit` ทำให้ debug ยากมาก
และซ่อนบั๊กที่ไม่ควรถูกซ่อน:

```python
# ไม่ควรทำ
try:
    risky_operation()
except:
    pass   # กลืน error ทุกชนิดแบบเงียบ ๆ อันตรายมาก!

# ควรทำ
try:
    risky_operation()
except (ValueError, KeyError) as e:
    logger.error(f"เกิดข้อผิดพลาดที่คาดการณ์ได้: {e}")
except Exception as e:
    logger.exception("เกิดข้อผิดพลาดที่ไม่คาดคิด")
    raise  # ส่งต่อ error ให้ระดับบนจัดการ ถ้าไม่รู้วิธีจัดการที่นี่
```

### 18.4 สร้าง Custom Exception ของตัวเอง

การสร้าง exception class เฉพาะของแอปพลิเคชันช่วยให้จัดการ error ได้ตรงจุดและสื่อ
ความหมายชัดเจนกว่า built-in exception ทั่วไป:

```python
class InsufficientStockError(Exception):
    """Exception ที่เกิดเมื่อสินค้าในสต๊อกไม่พอสำหรับคำสั่งซื้อ"""
    def __init__(self, product_name, requested, available):
        self.product_name = product_name
        self.requested = requested
        self.available = available
        message = (
            f"สินค้า '{product_name}' มีไม่พอ "
            f"(ต้องการ {requested} แต่มีในสต๊อก {available})"
        )
        super().__init__(message)


def purchase_product(product, quantity):
    if quantity > product.stock:
        raise InsufficientStockError(product.name, quantity, product.stock)
    product.stock -= quantity
    product.save()


try:
    purchase_product(shirt, 100)
except InsufficientStockError as e:
    print(f"ไม่สามารถสั่งซื้อได้: {e}")
    print(f"สินค้า: {e.product_name}, ขาดสต๊อก: {e.requested - e.available}")
```

### 18.5 Exception ของ Django เองที่คุณจะเจอบ่อยที่สุด

Django มี exception class เฉพาะทางของตัวเองมากมาย ซึ่งเป็น subclass ของ
`Exception` ตามที่เราเพิ่งเรียนไป:

```python
from django.core.exceptions import ObjectDoesNotExist, ValidationError, PermissionDenied
from django.http import Http404

def get_product_detail(request, product_id):
    try:
        product = Product.objects.get(pk=product_id)
    except Product.DoesNotExist:
        # DoesNotExist คือ subclass เฉพาะของแต่ละ Model ที่สืบทอดจาก ObjectDoesNotExist
        raise Http404("ไม่พบสินค้าที่คุณค้นหา")

    return render(request, "products/detail.html", {"product": product})
```

| Exception ของ Django | เมื่อไหร่ที่เกิดขึ้น |
|---|---|
| `Model.DoesNotExist` | เรียก `.get()` แล้วไม่เจอ record |
| `Model.MultipleObjectsReturned` | เรียก `.get()` แล้วเจอมากกว่า 1 record |
| `django.core.exceptions.ValidationError` | ข้อมูลไม่ผ่านการตรวจสอบใน Form/Model |
| `django.http.Http404` | ต้องการแสดงหน้า 404 Not Found |
| `django.core.exceptions.PermissionDenied` | ต้องการแสดงหน้า 403 Forbidden |
| `django.db.IntegrityError` | ข้อมูลขัดกับ constraint ของฐานข้อมูล (เช่น unique) |

ในความเป็นจริง Django มีชอร์ตคัตให้ใช้แทนโค้ดข้างบนคือ `get_object_or_404()`
ซึ่งภายในทำ try/except แบบเดียวกันนี้ให้เราอัตโนมัติ — เราจะเจอฟังก์ชันนี้จริง
ตั้งแต่ Part 007 เป็นต้นไป

### 18.6 Exception Chaining: การรักษาบริบทของ Error เดิม

```python
def process_payment(order):
    try:
        charge_credit_card(order)
    except PaymentGatewayError as e:
        # 'from e' จะรักษา traceback เดิมไว้ ช่วยให้ debug ง่ายขึ้นมาก
        raise OrderProcessingError(f"ไม่สามารถประมวลผลคำสั่งซื้อ #{order.id}") from e
```

### 18.7 Logging แทนการใช้ print() ในโค้ดจริง

```python
import logging

logger = logging.getLogger(__name__)

def risky_view(request):
    try:
        result = perform_complex_operation()
    except Exception:
        logger.exception("เกิดข้อผิดพลาดขณะประมวลผล complex operation")
        # logger.exception() จะบันทึก traceback แบบเต็มโดยอัตโนมัติ
        return HttpResponse("เกิดข้อผิดพลาด กรุณาลองใหม่", status=500)
    return HttpResponse(result)
```

Django มีระบบ logging ที่ตั้งค่าผ่าน `settings.py` ซึ่งเราจะเรียนละเอียดใน Part 010
(Django Settings) — แต่หลักการพื้นฐานของโมดูล `logging` คือสิ่งที่คุณเพิ่งเห็นนี่เอง

---

## ขั้นตอนที่ 19: Modules, Packages และ Import System

### 19.1 Module คืออะไร

**Module** คือไฟล์ `.py` หนึ่งไฟล์ที่รวมโค้ด (function, class, variable) ไว้ด้วยกัน
เพื่อให้ไฟล์อื่นนำไปใช้ซ้ำได้:

```python
# ไฟล์ math_utils.py
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

PI = 3.14159
```

```python
# ไฟล์ main.py (อยู่โฟลเดอร์เดียวกัน)
import math_utils

print(math_utils.add(2, 3))       # 5
print(math_utils.PI)               # 3.14159

# หรือ import เฉพาะส่วนที่ต้องการ
from math_utils import add, PI
print(add(2, 3))
```

### 19.2 Package คืออะไร

**Package** คือโฟลเดอร์ที่รวม module หลายไฟล์เข้าด้วยกัน โดยต้องมีไฟล์ `__init__.py`
(ตั้งแต่ Python 3.3 ไฟล์นี้ไม่บังคับต้องมีแล้ว แต่ยังนิยมใส่ไว้เพื่อความชัดเจนและ
ควบคุม public API ของ package)

```
myproject/
├── shop/                    <- นี่คือ package
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   └── utils.py
└── main.py
```

```python
# ไฟล์ main.py
from shop import models
from shop.utils import calculate_discount
from shop.views import product_list
```

**นี่คือโครงสร้างเดียวกับ Django App ทุกตัว!** เมื่อคุณรัน `python manage.py startapp
shop` ใน Part 005 Django จะสร้างโฟลเดอร์แบบนี้ให้อัตโนมัติ ทุก Django App คือ
Python package ธรรมดา ๆ ที่มีไฟล์ตามธรรมเนียมที่ Django คาดหวัง (`models.py`,
`views.py`, `urls.py`, `admin.py`)

### 19.3 Absolute Import vs Relative Import

Python มี 2 วิธีในการ import จากภายใน package เดียวกัน:

```python
# โครงสร้างไฟล์:
# shop/
# ├── __init__.py
# ├── models.py
# ├── views.py
# └── utils.py

# --- shop/views.py ---

# วิธีที่ 1: Absolute Import (แนะนำ และเป็นมาตรฐานของ Django)
from shop.models import Product
from shop.utils import calculate_discount

# วิธีที่ 2: Relative Import (ใช้ . แทนชื่อ package ปัจจุบัน)
from .models import Product
from .utils import calculate_discount

# .. หมายถึงขึ้นไป 1 ระดับ (ใช้เมื่อต้อง import จาก package แม่ หรือ package พี่น้อง)
from ..core.utils import format_currency
```

### 19.4 ทำไม Django แนะนำ Relative Import ภายใน App

นี่คือสิ่งที่มือใหม่มักสับสน: **Django code ที่ generate จาก `startapp` ใช้ relative
import (`.`) เสมอ** เช่นในไฟล์ `views.py` ที่ Django สร้างให้จะมี:

```python
# shop/views.py (โค้ดจริงที่ Django generate ให้)
from django.shortcuts import render
from .models import Product   # <- relative import ชี้ไปที่ shop/models.py
```

เหตุผลที่ Django เลือกใช้ relative import ภายใน app คือ:

1. **ทำให้ App ย้ายพอร์ตได้ง่าย**: ถ้าคุณ copy โฟลเดอร์ `shop/` ไปใช้ในโปรเจกต์อื่น
   ที่วางโครงสร้างชื่อ package แตกต่างกัน relative import จะยังทำงานถูกต้องโดย
   ไม่ต้องแก้โค้ด เพราะมันอ้างอิงตำแหน่งสัมพัทธ์ (relative) ไม่ใช่ชื่อ package
   แบบตายตัว
2. **ชัดเจนว่ากำลัง import จากภายใน app เดียวกัน**: เห็น `.` ปุ๊บรู้ทันทีว่านี่คือไฟล์
   ในแอปเดียวกัน ไม่ใช่ library ภายนอกหรือแอปอื่น

แต่เมื่อต้อง import ข้าม app (เช่น `shop` app ต้องใช้ Model จาก `accounts` app)
ให้ใช้ **absolute import** เสมอ เพราะชัดเจนกว่าและไม่มีปัญหาเรื่องระดับความลึกของ
โฟลเดอร์:

```python
# shop/views.py — import ข้าม app ให้ใช้ absolute import
from accounts.models import UserProfile   # absolute: ชัดเจน อ่านง่าย ไม่งง
from .models import Product                # relative: อยู่ใน app เดียวกัน
```

### 19.5 ตารางสรุปเมื่อไหร่ควรใช้ Import แบบไหน

| สถานการณ์ | แนะนำให้ใช้ | ตัวอย่าง |
|---|---|---|
| Import จากไฟล์อื่นใน app เดียวกัน | Relative | `from .models import Product` |
| Import จาก app อื่นในโปรเจกต์เดียวกัน | Absolute | `from accounts.models import User` |
| Import จาก third-party package | Absolute (ไม่มีทางเลือกอื่น) | `from django.db import models` |
| Import จาก submodule ลึกในแอปเดียวกัน | Relative | `from .services.email import send_welcome_email` |

### 19.6 `__init__.py` และการควบคุม Public API ของ Package

```python
# shop/__init__.py

# สามารถ import สิ่งที่อยากให้เรียกใช้งานง่าย ๆ ไว้ที่ระดับ package ได้เลย
from .models import Product, Category

# ทำให้คนอื่นเขียนแบบนี้ได้ แทนที่จะต้องพิมพ์ path ยาว ๆ
# from shop import Product
# แทนที่จะต้องเขียน from shop.models import Product
```

`__init__.py` ยังใช้กำหนด `__all__` เพื่อควบคุมว่า `from shop import *` จะดึงอะไร
ออกมาบ้าง (แม้ว่าการใช้ `import *` ไม่ค่อยแนะนำในโค้ด production เพราะทำให้ไม่ชัดเจน
ว่าชื่อตัวแปรมาจากไหน)

```python
# shop/__init__.py
__all__ = ["Product", "Category", "calculate_discount"]
```

### 19.7 Circular Import: ปัญหาคลาสสิกที่ Django Developer ทุกคนต้องเจอสักครั้ง

```python
# shop/models.py
from accounts.models import UserProfile  # (A) import จาก accounts

class Order(models.Model):
    user_profile = models.ForeignKey(UserProfile, on_delete=models.CASCADE)


# accounts/models.py
from shop.models import Order  # (B) import กลับมาจาก shop — เกิด Circular Import!

class UserProfile(models.Model):
    favorite_order = models.ForeignKey(Order, on_delete=models.SET_NULL, null=True)
```

โค้ดข้างบนจะทำให้เกิด `ImportError` เพราะ Python พยายามโหลด `shop.models` ซึ่งต้อง
โหลด `accounts.models` ก่อน แต่ `accounts.models` ก็ต้องโหลด `shop.models` ก่อนเช่นกัน
— วนลูปไม่มีที่สิ้นสุด วิธีแก้ที่ใช้บ่อยที่สุดในโลก Django:

```python
# วิธีที่ 1: ใช้ string reference แทน class จริง (Django รองรับโดยตรงสำหรับ ForeignKey)
class Order(models.Model):
    user_profile = models.ForeignKey("accounts.UserProfile", on_delete=models.CASCADE)

# วิธีที่ 2: import แบบ local (import ไว้ข้างในฟังก์ชัน ไม่ใช่บนสุดของไฟล์)
def get_related_order(user_profile):
    from shop.models import Order   # import เฉพาะตอนที่ฟังก์ชันถูกเรียกจริง
    return Order.objects.filter(user_profile=user_profile)
```

การอ้าง ForeignKey ด้วย string (`"accounts.UserProfile"` หรือแม้แต่ `"self"` สำหรับ
self-reference) คือ pattern มาตรฐานของ Django ที่แก้ปัญหา circular import ได้อย่าง
สวยงาม เราจะเจอเรื่องนี้อีกครั้งอย่างละเอียดใน Part 012 (ความสัมพันธ์ระหว่างโมเดล)

### 19.8 `sys.path` และวิธีที่ Python หา Module

```python
import sys
print(sys.path)  # แสดง list ของ path ที่ Python จะค้นหา module
```

เมื่อคุณเขียน `import shop`, Python จะค้นหาโฟลเดอร์ชื่อ `shop` ตามลำดับใน `sys.path`
(เริ่มจากโฟลเดอร์ปัจจุบัน ตามด้วย installed packages ใน venv, แล้วตามด้วย Python
standard library) นี่คือเหตุผลที่ virtual environment (ที่เราสร้างใน Part 001)
สำคัญมาก — มันเปลี่ยน `sys.path` ให้ชี้ไปที่ package ที่ติดตั้งเฉพาะใน venv นั้น ๆ
แยกจากระบบหลักโดยสมบูรณ์

---

## ขั้นตอนที่ 20: Dataclasses, typing ขั้นสูง, สรุปและแบบฝึกหัด

### 20.1 Dataclasses: ลดโค้ดซ้ำซ้อนของ Class ที่เก็บแค่ข้อมูล

หลายครั้งเราต้องการ class ที่แค่ "เก็บข้อมูล" ไม่มี logic ซับซ้อน การเขียน
`__init__` เองทุกครั้งซ้ำซาก Python 3.7+ มี `@dataclass` มาช่วย:

```python
from dataclasses import dataclass, field

@dataclass
class ProductDTO:
    """Data Transfer Object สำหรับส่งข้อมูลสินค้าระหว่างชั้นต่าง ๆ ของแอปพลิเคชัน"""
    name: str
    price: float
    stock: int = 0
    tags: list[str] = field(default_factory=list)  # ใช้ field() แทน mutable default ตรง ๆ

    def is_in_stock(self) -> bool:
        return self.stock > 0


product = ProductDTO(name="เสื้อยืด", price=299.0, stock=10, tags=["cotton", "summer"])
print(product)               # ProductDTO(name='เสื้อยืด', price=299.0, stock=10, tags=[...])
print(product.is_in_stock())  # True
```

สังเกตว่า `@dataclass` สร้าง `__init__`, `__repr__`, และ `__eq__` ให้อัตโนมัติ
โดยไม่ต้องเขียนเอง — ประหยัดโค้ดได้มาก และยังใช้ `field(default_factory=list)`
แก้ปัญหา mutable default argument ที่เราเรียนในขั้นตอนที่ 11 ได้อย่างสวยงามอีกด้วย

### 20.2 Dataclass เชื่อมกับ Django อย่างไร: ใช้เป็น DTO ระหว่าง Layer

แม้ Django Model จะไม่ใช่ dataclass (Django ใช้ metaclass ของตัวเองที่ซับซ้อนกว่ามาก)
แต่ dataclass มีประโยชน์มากในสถาปัตยกรรม Django ระดับมืออาชีพ เพื่อส่งข้อมูลระหว่าง
"service layer" กับ view โดยไม่ต้องพึ่ง Model โดยตรง (ลดการผูกติดกับฐานข้อมูล):

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)  # frozen=True ทำให้ instance แก้ไขค่าไม่ได้หลังสร้าง (immutable)
class OrderSummary:
    order_id: int
    customer_name: str
    total_amount: Decimal
    item_count: int


def build_order_summary(order) -> OrderSummary:
    """แปลง Django Model เป็น DTO ก่อนส่งให้ template หรือ API response"""
    return OrderSummary(
        order_id=order.id,
        customer_name=order.customer.get_full_name(),
        total_amount=order.total_amount,
        item_count=order.items.count(),
    )
```

รูปแบบนี้จะมีประโยชน์มากขึ้นเรื่อย ๆ เมื่อโปรเจกต์ใหญ่ขึ้น เพราะแยก "ข้อมูลที่ view
ต้องการแสดงผล" ออกจาก "โครงสร้างฐานข้อมูลจริง" — เราจะกลับมาใช้แนวคิดนี้อีกครั้งเมื่อ
เรียน Service Layer Pattern ในภายหลังของหลักสูตร (Phase สถาปัตยกรรมขั้นสูง)

### 20.3 typing ขั้นสูง: `TypedDict`, `Protocol`, `Literal`

```python
from typing import TypedDict, Literal, Protocol

# TypedDict: กำหนดรูปแบบของ dict ให้ตรวจสอบได้ (เหมาะกับ JSON payload)
class ProductPayload(TypedDict):
    name: str
    price: float
    is_active: bool


def create_product_from_payload(data: ProductPayload) -> None:
    print(data["name"], data["price"])


# Literal: จำกัดค่าที่รับได้ให้เป็นค่าที่ระบุไว้เท่านั้น
def set_order_status(status: Literal["pending", "shipped", "delivered", "cancelled"]) -> None:
    print(f"เปลี่ยนสถานะเป็น {status}")

set_order_status("shipped")   # OK
# set_order_status("done")    # Type checker (mypy) จะแจ้ง error ทันที


# Protocol: กำหนด "รูปร่าง" ของ object โดยไม่ต้อง inherit จริง (structural typing)
class Discountable(Protocol):
    price: float
    def apply_discount(self, percent: float) -> None: ...

def show_discounted_price(item: Discountable) -> None:
    item.apply_discount(10)
    print(item.price)
```

`Literal` มีประโยชน์มากเมื่อออกแบบฟังก์ชันที่รับค่าจำกัด เช่นเดียวกับ `choices` ของ
Django Model field ที่เราจะเรียนใน Part 011 — เป็นแนวคิดเดียวกันคือ "จำกัดค่าที่
เป็นไปได้ให้ชัดเจน" เพียงแต่ `Literal` ทำงานที่ระดับ type-checking ส่วน `choices`
ทำงานที่ระดับ validation จริงตอน runtime

### 20.4 `Enum`: ทางเลือกที่ Django ใช้จริงสำหรับค่าคงที่แบบจำกัด

```python
from enum import Enum

class OrderStatus(Enum):
    PENDING = "pending"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


order_status = OrderStatus.SHIPPED
print(order_status)          # OrderStatus.SHIPPED
print(order_status.value)    # "shipped"
print(order_status.name)     # "SHIPPED"

if order_status == OrderStatus.SHIPPED:
    print("สินค้ากำลังจัดส่ง")
```

Django เวอร์ชันสมัยใหม่ (5.x) มี `models.TextChoices` และ `models.IntegerChoices`
ซึ่งสร้างขึ้นจาก `Enum` ของ Python โดยตรง เราจะได้ใช้งานจริงใน Part 011:

```python
from django.db import models

class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = "pending", "รอดำเนินการ"
        SHIPPED = "shipped", "จัดส่งแล้ว"
        DELIVERED = "delivered", "ส่งถึงแล้ว"
        CANCELLED = "cancelled", "ยกเลิก"

    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
    )
```

การที่คุณเข้าใจ `Enum` ตอนนี้จะทำให้โค้ดข้างบนดูเป็นธรรมชาติทันทีที่เจอมัน แทนที่จะ
ต้องท่องจำ syntax แบบไม่เข้าใจที่มา

### 20.5 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ Dynamic Typing, Type Hints และเหตุผลที่ต้องใช้ `Decimal` แทน `float`
  สำหรับข้อมูลการเงิน
- ✅ เข้าใจ Function, Default Arguments, `*args`, `**kwargs` และนำไปเชื่อมกับ
  `def save(self, *args, **kwargs)` ของ Django Model
- ✅ เข้าใจ OOP อย่างลึกซึ้ง: `__init__`, Inheritance, `super()`, Magic Methods,
  `@property`, `@classmethod`, `@staticmethod` — รากฐานของ Model, Form, CBV
- ✅ เข้าใจกลไกการทำงานของ Decorator จนถึงระดับเขียน `login_required` เวอร์ชันของ
  ตัวเองได้
- ✅ เข้าใจ Context Manager และเชื่อมโยงกับ `transaction.atomic()` ของ Django
- ✅ ใช้ List/Dict/Set Comprehension ได้อย่างคล่องแคล่ว พร้อมรู้ข้อควรระวังเรื่อง
  N+1 query
- ✅ เข้าใจ Iterator, Generator และ Lazy Evaluation ซึ่งเป็นหลักการเดียวกับที่
  QuerySet ของ Django ใช้
- ✅ จัดการ Exception อย่างมืออาชีพ และรู้จัก Exception เฉพาะทางของ Django
- ✅ เข้าใจระบบ Module/Package/Import และรู้ว่าทำไม Django App ใช้ Relative Import
  ภายในแอป แต่ใช้ Absolute Import ข้ามแอป พร้อมวิธีแก้ Circular Import
- ✅ ใช้ Dataclass, `Enum`, และ typing ขั้นสูง (`TypedDict`, `Literal`, `Protocol`)
  ได้ พร้อมเห็นความเชื่อมโยงกับ `TextChoices` ของ Django

### 20.6 Checklist ก่อนไป Part ถัดไป

- [ ] เขียนฟังก์ชันที่ใช้ `*args` และ `**kwargs` แล้วอธิบายได้ว่าแต่ละตัวเก็บข้อมูล
      เป็นชนิดอะไร (tuple / dict)
- [ ] เขียน class ที่มี inheritance อย่างน้อย 2 ระดับ พร้อมเรียก `super()` ได้ถูกต้อง
- [ ] เขียน decorator ของตัวเองอย่างน้อย 1 ตัว ที่ใช้ `@wraps` และรับ argument ได้
- [ ] เขียน context manager ทั้งแบบ class (`__enter__`/`__exit__`) และแบบ
      `@contextmanager` ได้ทั้งสองวิธี
- [ ] เขียน generator function ที่ใช้ `yield` แล้วอธิบายความแตกต่างจาก function
      ปกติที่ใช้ `return` ได้
- [ ] แยกความแตกต่างระหว่าง Absolute Import กับ Relative Import ได้ และรู้ว่า
      Django ใช้แบบไหนที่ไหน
- [ ] เขียน dataclass ที่มี default value เป็น list โดยใช้ `field(default_factory=list)`
      อย่างถูกต้อง (ไม่ใช้ mutable default ตรง ๆ)

### 20.7 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: สร้างไฟล์ `inventory.py` เขียน class `InventoryItem` ที่มี
attribute `name`, `price` (เป็น `Decimal`), `quantity` เขียนเมธอด `__str__`,
`__repr__`, `__eq__` (เทียบจาก name), และ property ชื่อ `total_value` ที่คำนวณ
`price * quantity` จากนั้นสร้าง list ของ `InventoryItem` หลายชิ้น แล้วใช้
list comprehension หาผลรวมมูลค่าสินค้าทั้งหมดในคลัง

**แบบฝึกหัดที่ 2**: เขียน decorator ชื่อ `require_positive` ที่ตรวจสอบว่า argument
แรกของฟังก์ชันที่ decorate เป็นค่าบวกหรือไม่ ถ้าไม่ใช่ให้ raise `ValueError` พร้อม
ข้อความที่ชัดเจน ทดสอบกับฟังก์ชัน `calculate_shipping_cost(weight_kg)` ของคุณเอง

**แบบฝึกหัดที่ 3**: เขียน context manager ชื่อ `timed_block` (ใช้ `@contextmanager`)
ที่วัดเวลาการทำงานของโค้ดใน `with` block แล้ว print ผลลัพธ์เป็นวินาทีเมื่อออกจาก
block ทดสอบโดยห่อ loop ที่คำนวณอะไรสักอย่างหนัก ๆ (เช่น หาจำนวนเฉพาะ 100,000 ตัวแรก)

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: สร้างไฟล์ `custom_exceptions.py` ที่มี custom exception
อย่างน้อย 3 ตัวที่เกี่ยวข้องกับระบบร้านค้าออนไลน์ (เช่น `OutOfStockError`,
`InvalidCouponError`, `PaymentFailedError`) ให้แต่ละตัวเก็บข้อมูลเพิ่มเติมที่เป็น
ประโยชน์ (เช่น product_id, coupon_code) จากนั้นเขียนฟังก์ชัน `checkout(cart, coupon)`
ที่ raise exception เหล่านี้ตามเงื่อนไขต่าง ๆ แล้วเขียนโค้ดเรียกใช้พร้อม try/except
ที่จัดการแต่ละ exception แยกกันอย่างเหมาะสม

### 20.8 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเชี่ยวชาญ Python 100% ก่อนเรียน Django ต่อหรือไม่?**
A: ไม่จำเป็น เนื้อหาใน Part นี้ครอบคลุมสิ่งที่ Django ใช้งานหนักที่สุดแล้ว หัวข้อ
Python ขั้นสูงอื่น ๆ ที่ไม่ได้ใช้บ่อยในงาน Django (เช่น metaclass แบบเจาะลึก,
async/await แบบเต็มรูปแบบ) เราจะแนะนำเฉพาะจุดที่จำเป็นในภายหลัง (Part 073
Async Views) เมื่อถึงเวลาที่เกี่ยวข้องจริง ๆ

**Q: ถ้ายังงงเรื่อง Decorator หรือ Generator อยู่ ควรทำอย่างไร?**
A: กลับมาอ่าน ขั้นตอนที่ 14 และ 17 อีกครั้ง แล้วลองพิมพ์โค้ดตัวอย่างด้วยตัวเอง
(ไม่ใช่แค่อ่านผ่าน ๆ) ทั้งสองแนวคิดนี้เป็นรากฐานสำคัญที่จะกลับมาปรากฏซ้ำ ๆ ตลอด
ทั้งหลักสูตร ยิ่งแน่นตอนนี้ยิ่งเรียน Django ในอนาคตได้ราบรื่นกว่ามาก

**Q: จำเป็นต้องใช้ Type Hints ในทุกโปรเจกต์ Django จริงหรือไม่?**
A: ไม่บังคับ แต่ทีมงานมืออาชีพในปี 2025-2026 นิยมใช้มากขึ้นเรื่อย ๆ โดยเฉพาะเมื่อ
ใช้คู่กับ `mypy` หรือ `django-stubs` เพื่อจับบั๊กตั้งแต่ก่อนรันโปรแกรม หลักสูตรนี้
จะใช้ type hints เป็นระยะเมื่อช่วยให้โค้ดชัดเจนขึ้น แต่จะไม่ใช้แบบเข้มงวด 100%
ในทุกตัวอย่าง เพื่อไม่ให้โค้ดอ่านยากเกินไปสำหรับผู้เริ่มต้น

**Q: OOP กับ Functional Programming อันไหนดีกว่าสำหรับ Django?**
A: Django เป็น framework แบบ OOP โดยธรรมชาติ (Model, View, Form เป็น class ทั้งหมด)
แต่ภายในเมธอดต่าง ๆ คุณสามารถเขียนสไตล์ functional ได้เต็มที่ (comprehension,
generator, `map`/`filter`, ฟังก์ชันบริสุทธิ์ที่ไม่มี side effect) หลักสูตรนี้จะสอน
ทั้งสองแนวทางผสมกันตามความเหมาะสมของแต่ละสถานการณ์ ไม่ยึดติดกับแนวทางใดแนวทางหนึ่ง
สุดโต่ง

### เตรียมตัวสำหรับ Part ถัดไป

**Part 003: Virtual Environment, pip และการจัดการแพ็กเกจ** จะพาคุณกลับไปเจาะลึก
เรื่อง dependency management ที่เราแตะไปเบื้องต้นใน Part 001 แต่คราวนี้จะลงลึกถึง
`pip`, `requirements.txt` แบบมืออาชีพ (แยก dev/production), เครื่องมือสมัยใหม่อย่าง
**uv** และ **Poetry**, การจัดการเวอร์ชัน package แบบ pin/unpin, และการแก้ปัญหา
dependency conflict ที่พบบ่อยในทีมงานจริง ความเข้าใจเรื่อง Module/Package/Import
ที่เพิ่งเรียนใน Part นี้จะเป็นพื้นฐานสำคัญที่ทำให้ Part 003 เข้าใจง่ายขึ้นมาก
เพราะ package ที่เราติดตั้งด้วย `pip` ก็คือ Python package ตามที่เราเพิ่งเรียนไป
นั่นเอง เตรียม Terminal ให้พร้อม แล้วไปต่อกันเลย!
