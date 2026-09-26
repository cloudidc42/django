# Part 001: บทนำสู่ Django, Web Development และการเตรียมเครื่องมือ

> **ขั้นตอนที่ 1-10 ของหลักสูตร** | Phase 1: รากฐาน Python & Django
>
> เป้าหมายของ Part นี้: เข้าใจว่า Django คืออะไร ทำงานอย่างไรในภาพรวม
> และเตรียมเครื่องมือทุกอย่างให้พร้อมสำหรับการเขียนโค้ดใน Part ถัดไป
> เมื่อจบ Part นี้ เครื่องของคุณจะมี Python, Git, VS Code และ Django
> พร้อมใช้งาน และคุณจะเข้าใจสถาปัตยกรรม MTV ของ Django อย่างถ่องแท้

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 1: Web Development คืออะไร และ Django อยู่ตรงไหนในภาพรวม
- ขั้นตอนที่ 2: ทำไมต้องเลือก Django? เปรียบเทียบกับ Framework อื่น
- ขั้นตอนที่ 3: สถาปัตยกรรม MTV (Model-Template-View) ของ Django
- ขั้นตอนที่ 4: ติดตั้ง Python บนเครื่องของคุณ
- ขั้นตอนที่ 5: ติดตั้งและตั้งค่า Visual Studio Code
- ขั้นตอนที่ 6: พื้นฐาน Command Line / Terminal ที่ต้องรู้
- ขั้นตอนที่ 7: ติดตั้ง Git และเชื่อมต่อ GitHub
- ขั้นตอนที่ 8: สร้าง Virtual Environment ครั้งแรก
- ขั้นตอนที่ 9: ติดตั้ง Django และตรวจสอบเวอร์ชัน
- ขั้นตอนที่ 10: สรุปและแบบฝึกหัด

---

## ขั้นตอนที่ 1: Web Development คืออะไร และ Django อยู่ตรงไหนในภาพรวม

### 1.1 เว็บแอปพลิเคชันทำงานอย่างไร

ก่อนจะเรียน Django เราต้องเข้าใจภาพรวมของการทำงานของเว็บก่อน เว็บแอปพลิเคชันสมัยใหม่
ทำงานตามรูปแบบ **Client-Server Architecture**:

```
┌─────────────┐         HTTP Request          ┌─────────────────┐
│   Browser   │ ─────────────────────────────> │   Web Server    │
│  (Client)   │                                 │  (Django App)   │
│             │ <───────────────────────────── │                 │
└─────────────┘         HTTP Response           └─────────────────┘
                                                          │
                                                          │ Query
                                                          ▼
                                                  ┌─────────────────┐
                                                  │    Database     │
                                                  │  (PostgreSQL)   │
                                                  └─────────────────┘
```

เมื่อผู้ใช้พิมพ์ URL เช่น `https://example.com/products/` ในเบราว์เซอร์ สิ่งที่เกิดขึ้นคือ:

1. เบราว์เซอร์ส่ง **HTTP Request** ไปยังเซิร์ฟเวอร์ พร้อมข้อมูล method (GET, POST, ...),
   headers, และ path (`/products/`)
2. เซิร์ฟเวอร์ (ในกรณีนี้คือแอป Django) รับ request มาประมวลผล
3. Django จะหาว่า URL นี้ควรถูกจัดการโดยฟังก์ชันไหน (เรียกว่า **routing**)
4. ฟังก์ชันนั้นอาจไปดึงข้อมูลจากฐานข้อมูล ประมวลผล แล้วสร้างหน้า HTML กลับมา
5. เซิร์ฟเวอร์ส่ง **HTTP Response** กลับไปยังเบราว์เซอร์ (โดยทั่วไปเป็น HTML, JSON หรือไฟล์อื่น ๆ)
6. เบราว์เซอร์แสดงผลลัพธ์ให้ผู้ใช้เห็น

Django คือ **framework** ที่ช่วยให้เราเขียนฝั่งเซิร์ฟเวอร์ (ข้อ 2-5) ได้ง่ายและรวดเร็วขึ้นมาก
โดยไม่ต้องเขียนทุกอย่างตั้งแต่ศูนย์

### 1.2 Django คืออะไรกันแน่

**Django** คือ **web framework ระดับสูง (high-level)** ที่เขียนด้วยภาษา Python
ออกแบบมาเพื่อให้นักพัฒนาสร้างเว็บแอปพลิเคชันได้อย่างรวดเร็ว สะอาด และปลอดภัย

คำขวัญอย่างเป็นทางการของ Django คือ:

> "The web framework for perfectionists with deadlines"
> (เฟรมเวิร์กเว็บสำหรับคนที่ต้องการความสมบูรณ์แบบ แต่ก็มี deadline ต้องส่งงาน)

Django ถูกสร้างขึ้นครั้งแรกในปี 2003 โดยทีมงานหนังสือพิมพ์ Lawrence Journal-World
(รัฐ Kansas สหรัฐอเมริกา) โดย Adrian Holovaty และ Simon Willison เพื่อรองรับความต้องการ
สร้างเว็บไซต์ข่าวที่ต้องอัปเดตเนื้อหาอย่างรวดเร็ว และเปิดเป็น Open Source ในปี 2005
ตั้งชื่อตาม Django Reinhardt นักกีตาร์แจ๊สชื่อดังชาวเบลเยียม-ฝรั่งเศส

### 1.3 บริษัทและเว็บไซต์ที่ใช้ Django จริงในระดับโลก

Django ไม่ใช่แค่เฟรมเวิร์กสำหรับมือใหม่ แต่ถูกใช้งานจริงในระบบระดับโลกที่มีผู้ใช้หลักร้อยล้านคน:

| บริษัท | การใช้งาน Django |
|---|---|
| **Instagram** | Backend หลักของแอป (หนึ่งใน deployment Django ที่ใหญ่ที่สุดในโลก) |
| **Spotify** | บางส่วนของระบบ backend และเครื่องมือภายใน |
| **Pinterest** | ระบบ backend ยุคแรกเริ่มของแพลตฟอร์ม |
| **Mozilla** | เว็บไซต์และเครื่องมือสนับสนุน Firefox |
| **The Washington Post** | ระบบจัดการเนื้อหาข่าว (Django ถือกำเนิดจากงานสายข่าว) |
| **Disqus** | ระบบคอมเมนต์ที่ใช้กันทั่วโลก |
| **National Geographic** | เว็บไซต์หลักขององค์กร |
| **YouTube (บางส่วน)** | เครื่องมือภายในบางระบบ |
| **Robinhood** | แพลตฟอร์มซื้อขายหุ้น (บางส่วนของ backend) |

ตัวอย่างเหล่านี้แสดงให้เห็นว่า Django สามารถ **สเกล (scale)** ได้จริงตั้งแต่โปรเจกต์เล็ก ๆ
ไปจนถึงระบบที่รองรับผู้ใช้หลักร้อยล้านคนต่อวัน

### 1.4 Django เหมาะกับงานประเภทไหนบ้าง

- **เว็บแอปพลิเคชันทั่วไป**: บล็อก, เว็บข่าว, เว็บบริษัท, ระบบจัดการเนื้อหา (CMS)
- **RESTful API / Backend สำหรับ Mobile App**: ด้วย Django REST Framework
- **E-Commerce**: ระบบร้านค้าออนไลน์ (มี package สำเร็จรูปอย่าง Saleor, Oscar)
- **SaaS Platform**: ระบบ Multi-tenant สำหรับให้บริการลูกค้าหลายราย
- **Social Network**: ระบบโซเชียลมีเดียขนาดเล็กถึงกลาง
- **Data Dashboard / Admin Panel**: ด้วย Django Admin ที่มีมาให้ในตัว
- **Internal Tools**: เครื่องมือภายในองค์กรที่ต้องการความรวดเร็วในการพัฒนา
- **Fintech / ระบบที่ต้องการความปลอดภัยสูง**: เพราะ Django มี security features ครบครัน

---

## ขั้นตอนที่ 2: ทำไมต้องเลือก Django? เปรียบเทียบกับ Framework อื่น

### 2.1 ปรัชญา "Batteries Included"

จุดเด่นที่สุดของ Django คือปรัชญา **"Batteries Included"** หมายความว่า Django มาพร้อม
ฟีเจอร์ที่จำเป็นสำหรับสร้างเว็บแอปพลิเคชันเกือบทั้งหมด โดยไม่ต้องไปหา library เพิ่มเติมเอง:

| ฟีเจอร์ | มีมาให้ใน Django หรือไม่ |
|---|---|
| ORM (Object-Relational Mapping) | ✅ มีในตัว |
| ระบบ Admin Panel อัตโนมัติ | ✅ มีในตัว |
| ระบบ Authentication & Authorization | ✅ มีในตัว |
| ระบบ Form handling & validation | ✅ มีในตัว |
| ระบบ Template Engine | ✅ มีในตัว |
| ระบบป้องกัน CSRF, XSS, SQL Injection | ✅ มีในตัว (เปิดใช้งานโดยค่าเริ่มต้น) |
| ระบบ Caching Framework | ✅ มีในตัว |
| ระบบ Internationalization (i18n) | ✅ มีในตัว |
| ระบบ Migration จัดการฐานข้อมูล | ✅ มีในตัว |
| ระบบส่งอีเมล | ✅ มีในตัว |

เปรียบเทียบกับ Flask หรือ Express.js (Node.js) ที่เป็น **micro-framework** ซึ่งให้เพียง
core เล็ก ๆ แล้วให้นักพัฒนาเลือก library เองทั้งหมด (routing, ORM, auth, admin ฯลฯ)

### 2.2 ตารางเปรียบเทียบ Django กับ Framework ยอดนิยมอื่น

| คุณสมบัติ | Django (Python) | Flask (Python) | Express.js (Node.js) | Ruby on Rails (Ruby) | Laravel (PHP) |
|---|---|---|---|---|---|
| ประเภท | Full-stack (Batteries included) | Micro-framework | Micro-framework | Full-stack | Full-stack |
| ORM ในตัว | ✅ | ❌ (ต้องใช้ SQLAlchemy) | ❌ (ต้องใช้ Prisma/Sequelize) | ✅ (ActiveRecord) | ✅ (Eloquent) |
| Admin Panel อัตโนมัติ | ✅ | ❌ | ❌ | ❌ (ต้องใช้ ActiveAdmin) | ❌ (ต้องใช้ Nova) |
| ความเร็วในการพัฒนา (Development Speed) | เร็วมาก | ปานกลาง | ปานกลาง | เร็วมาก | เร็วมาก |
| Learning Curve | ปานกลาง | ต่ำ | ต่ำ-ปานกลาง | ปานกลาง | ปานกลาง |
| Performance ดิบ (Raw Performance) | ดี | ดี | ดีมาก | ปานกลาง | ปานกลาง |
| Ecosystem/Package | ใหญ่มาก (PyPI) | ใหญ่ (PyPI) | ใหญ่ที่สุด (npm) | ใหญ่ (RubyGems) | ใหญ่ (Composer) |
| เหมาะกับ Data Science/AI integration | ดีมาก (Python ecosystem) | ดีมาก | ปานกลาง | น้อย | น้อย |

### 2.3 ข้อดีของ Django ในเชิงลึก

1. **ปลอดภัยโดยค่าเริ่มต้น (Secure by default)**: Django ป้องกัน CSRF, XSS, SQL Injection,
   Clickjacking ให้อัตโนมัติโดยไม่ต้องตั้งค่าเพิ่ม นักพัฒนามือใหม่จึงไม่พลาดพลั้งเรื่อง
   security ง่าย ๆ
2. **DRY Principle (Don't Repeat Yourself)**: Django ออกแบบให้เขียนโค้ดครั้งเดียวใช้ได้หลายที่
   เช่น กำหนด Model ครั้งเดียว ก็ได้ทั้งฐานข้อมูล, ฟอร์ม, และหน้า Admin
3. **ORM ที่ทรงพลัง**: เขียน Python แทน SQL ได้ และรองรับหลายฐานข้อมูล (PostgreSQL, MySQL,
   SQLite, Oracle) โดยไม่ต้องแก้โค้ด
4. **Community และเอกสารที่ยอดเยี่ยม**: Django มีเอกสารทางการที่ละเอียดที่สุดในบรรดา
   web framework ทั้งหมด (https://docs.djangoproject.com)
5. **Scalable**: พิสูจน์แล้วจาก Instagram ที่รองรับผู้ใช้หลายร้อยล้านคน
6. **เหมาะกับ Python ecosystem**: ถ้าต้องทำงานร่วมกับ Data Science, Machine Learning,
   AI (เช่น เชื่อมกับ pandas, NumPy, TensorFlow, PyTorch) การใช้ Python ทั้ง stack
   ช่วยลดความซับซ้อนได้มาก

### 2.4 ข้อจำกัดที่ควรรู้ (เพื่อความเป็นมืออาชีพ ไม่ใช่แค่ขายของ)

- Django ค่อนข้าง **"opinionated"** คือมีวิธีทำสิ่งต่าง ๆ ที่ถูกกำหนดไว้ชัดเจน
  ถ้าอยากทำนอกกรอบ (unconventional) อาจจะยากกว่า framework ที่ยืดหยุ่นกว่า
- สำหรับ real-time application แบบหนัก ๆ (เช่น เกมออนไลน์ที่ต้องการ latency ต่ำมาก)
  อาจไม่ใช่ตัวเลือกอันดับแรก (แม้ Django Channels จะรองรับ WebSocket ได้ดีในระดับหนึ่ง)
- Monolithic โดยธรรมชาติ แม้จะแยกเป็น microservices ได้ แต่ต้องออกแบบเพิ่มเติม
  (เราจะเรียนเรื่องนี้ใน Phase 12)

ในหลักสูตรนี้เราจะเรียนรู้ทั้งจุดแข็งและวิธีจัดการกับข้อจำกัดเหล่านี้อย่างมืออาชีพ

---

## ขั้นตอนที่ 3: สถาปัตยกรรม MTV (Model-Template-View) ของ Django

### 3.1 MTV vs MVC

หลายคนคุ้นเคยกับรูปแบบสถาปัตยกรรม **MVC (Model-View-Controller)** ซึ่งเป็นที่นิยมใน
framework อื่น ๆ เช่น Ruby on Rails, Laravel

Django ใช้รูปแบบที่เรียกว่า **MTV (Model-Template-View)** ซึ่งเป็นแนวคิดเดียวกันแต่ใช้
คำเรียกต่างกัน:

| MVC (แบบทั่วไป) | MTV (แบบ Django) | หน้าที่ |
|---|---|---|
| Model | **Model** | จัดการข้อมูลและตรรกะทางธุรกิจ เชื่อมกับฐานข้อมูล |
| View | **Template** | จัดการการแสดงผล (สิ่งที่ผู้ใช้เห็น) |
| Controller | **View** | จัดการตรรกะ รับ request ประมวลผล และเลือกว่าจะแสดงอะไร |
| (ไม่มี) | **URL dispatcher** | จับคู่ URL กับ View ที่เหมาะสม |

**ข้อควรระวัง**: คำว่า "View" ใน Django หมายถึง "Controller" ในความหมายของ MVC ทั่วไป
ไม่ใช่ "สิ่งที่ผู้ใช้เห็น" แบบที่คนทั่วไปเข้าใจ! นี่คือจุดที่มือใหม่มักสับสน

### 3.2 แผนภาพการทำงานของ Django แบบละเอียด

```
                     1. HTTP Request
    Browser  ─────────────────────────────>  urls.py (URL Dispatcher)
                                                      │
                                                      │ 2. จับคู่ URL Pattern
                                                      ▼
                                                  views.py (View)
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    │ 3a. ขอข้อมูล                    │ 3b. เตรียมข้อมูล
                                    ▼                                    ▼
                              models.py (Model)                 templates/*.html (Template)
                                    │                                    │
                                    │ ดึง/บันทึกข้อมูล                  │ 4. Render HTML
                                    ▼                                    │
                              Database (PostgreSQL/                     │
                              MySQL/SQLite)                              │
                                    │                                    │
                                    └────────────> views.py <────────────┘
                                                      │
                                                      │ 5. HttpResponse
                                                      ▼
    Browser  <─────────────────────────────  6. HTTP Response (HTML/JSON)
```

### 3.3 อธิบายแต่ละองค์ประกอบ

#### Model (models.py)

Model คือ Python class ที่แทนโครงสร้างข้อมูลในฐานข้อมูล แต่ละ class = 1 ตาราง (table)
แต่ละ attribute = 1 คอลัมน์ (column) ตัวอย่างเช่น:

```python
# models.py
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
```

โค้ดข้างต้นเมื่อรัน migration จะสร้างตาราง `Product` ในฐานข้อมูลให้อัตโนมัติ โดยไม่ต้อง
เขียน SQL เอง (เราจะเรียนละเอียดใน Phase 2)

#### View (views.py)

View คือฟังก์ชัน (หรือ class) ที่รับ HTTP Request เข้ามา ประมวลผล แล้วส่ง HTTP Response
กลับไป ตัวอย่างเบื้องต้น:

```python
# views.py
from django.shortcuts import render
from .models import Product

def product_list(request):
    products = Product.objects.all()  # ดึงข้อมูลทั้งหมดจากตาราง Product
    return render(request, 'products/list.html', {'products': products})
```

#### Template (templates/products/list.html)

Template คือไฟล์ HTML ที่มีการฝัง **Django Template Language (DTL)** เข้าไป เพื่อแสดง
ข้อมูลแบบไดนามิก:

```html
<!-- templates/products/list.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>รายการสินค้า</title>
</head>
<body>
    <h1>รายการสินค้าทั้งหมด</h1>
    <ul>
        {% for product in products %}
            <li>{{ product.name }} - {{ product.price }} บาท</li>
        {% empty %}
            <li>ยังไม่มีสินค้า</li>
        {% endfor %}
    </ul>
</body>
</html>
```

#### URL Dispatcher (urls.py)

ตัวเชื่อมระหว่าง URL ที่ผู้ใช้เข้ามา กับ View ที่จะจัดการ request นั้น:

```python
# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('products/', views.product_list, name='product_list'),
]
```

เมื่อผู้ใช้เข้า `https://example.com/products/` Django จะ:
1. ตรวจสอบ `urls.py` เจอ pattern `products/` ตรงกับ `views.product_list`
2. เรียกฟังก์ชัน `product_list(request)`
3. ฟังก์ชันดึงข้อมูลจาก Model `Product`
4. ส่งข้อมูลไปให้ Template `list.html` render เป็น HTML
5. ส่ง HTML กลับไปยังเบราว์เซอร์

นี่คือวงจรการทำงานพื้นฐานที่สุดของ Django ซึ่งเราจะฝึกเขียนจริงใน Part 004 เป็นต้นไป
ตอนนี้ขอให้เข้าใจ **ภาพรวม** ก่อน เพราะทุก Part ถัดไปจะอ้างอิงกลับมาที่แผนภาพนี้เสมอ

### 3.4 หลักการ DRY และ Convention over Configuration

Django ยึดหลักการสำคัญ 2 ข้อ:

- **DRY (Don't Repeat Yourself)**: ทุกความรู้ (knowledge) ในระบบควรมีแหล่งอ้างอิง
  เดียวที่ชัดเจน ไม่ซ้ำซ้อน เช่น กำหนด field ในไฟล์ `models.py` ที่เดียว ก็นำไปใช้ได้ทั้ง
  ฐานข้อมูล, ฟอร์ม, validation, และ Admin panel
- **Convention over Configuration**: Django มีข้อตกลง (convention) มาตรฐานที่ทำให้
  ไม่ต้องตั้งค่าซ้ำซ้อน เช่น ถ้าตั้งชื่อไฟล์และโฟลเดอร์ตามที่ Django คาดหวัง
  ระบบจะทำงานได้ทันทีโดยไม่ต้องเขียน config เพิ่ม

---

## ขั้นตอนที่ 4: ติดตั้ง Python บนเครื่องของคุณ

### 4.1 ตรวจสอบว่ามี Python ติดตั้งอยู่แล้วหรือไม่

เปิด Terminal (macOS/Linux) หรือ Command Prompt/PowerShell (Windows) แล้วรันคำสั่ง:

```bash
python3 --version
# หรือบน Windows
python --version
```

หากเห็นผลลัพธ์เช่น `Python 3.12.4` แสดงว่ามี Python ติดตั้งแล้ว หลักสูตรนี้แนะนำให้ใช้
**Python 3.12 หรือใหม่กว่า** (ขั้นต่ำที่ Django 5.x รองรับคือ Python 3.10)

### 4.2 ติดตั้ง Python บน Windows

1. ไปที่เว็บไซต์ทางการ https://www.python.org/downloads/
2. ดาวน์โหลดเวอร์ชันล่าสุด (แนะนำ 3.12.x)
3. รันตัวติดตั้ง (installer) **สำคัญมาก**: ต้องติ๊กช่อง
   ✅ **"Add python.exe to PATH"** ก่อนกด Install ไม่เช่นนั้นจะเรียกใช้ `python`
   จาก command line ไม่ได้
4. เลือก "Install Now"
5. เมื่อติดตั้งเสร็จ เปิด Command Prompt ใหม่ แล้วตรวจสอบด้วย:

```powershell
python --version
pip --version
```

### 4.3 ติดตั้ง Python บน macOS

macOS มักมี Python เวอร์ชันเก่าติดมาในระบบ (system Python) **ห้ามใช้ system Python
ในการพัฒนา** เพราะอาจกระทบระบบปฏิบัติการ แนะนำให้ติดตั้งผ่าน **Homebrew**:

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Python เวอร์ชันล่าสุด
brew install python@3.12

# ตรวจสอบ
python3.12 --version
```

### 4.4 ติดตั้ง Python บน Linux (Ubuntu/Debian)

Linux ส่วนใหญ่มี Python 3 ติดมาแล้ว แต่ควรอัปเดตให้เป็นเวอร์ชันล่าสุด:

```bash
sudo apt update
sudo apt install python3.12 python3.12-venv python3-pip -y

# ตรวจสอบ
python3.12 --version
```

หากใช้ distro อื่น (Fedora, Arch) ให้ใช้ package manager ของตัวเอง:

```bash
# Fedora
sudo dnf install python3.12 python3-pip

# Arch Linux
sudo pacman -S python python-pip
```

### 4.5 ทางเลือกขั้นสูง: ใช้ pyenv จัดการหลายเวอร์ชัน Python

ในงานจริงระดับมืออาชีพ คุณมักต้องทำงานกับหลายโปรเจกต์ที่ใช้ Python คนละเวอร์ชัน
เครื่องมือ **pyenv** ช่วยให้สลับเวอร์ชัน Python ได้ง่าย:

```bash
# macOS/Linux
curl https://pyenv.run | bash

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

# ติดตั้ง Python เวอร์ชันที่ต้องการ
pyenv install 3.12.4
pyenv global 3.12.4   # ตั้งเป็นเวอร์ชัน default ทั้งระบบ
pyenv local 3.12.4    # ตั้งเฉพาะโฟลเดอร์ปัจจุบัน (สร้างไฟล์ .python-version)
```

เราจะใช้ `pyenv` เมื่อถึง Phase DevOps เพื่อจำลองสภาพแวดล้อม production ที่แม่นยำ

---

## ขั้นตอนที่ 5: ติดตั้งและตั้งค่า Visual Studio Code

### 5.1 ทำไมต้องใช้ VS Code

แม้ Django จะใช้ editor อะไรก็ได้ (PyCharm, Sublime Text, Vim) แต่หลักสูตรนี้แนะนำ
**Visual Studio Code (VS Code)** เพราะ:

- ฟรี และรองรับทุกระบบปฏิบัติการ
- มี Extension สำหรับ Python และ Django โดยเฉพาะ
- มี Integrated Terminal ในตัว
- รองรับ Git ในตัว
- เป็นเครื่องมือยอดนิยมที่สุดในหมู่นักพัฒนา Python ปัจจุบัน (จากผลสำรวจ Stack Overflow)

### 5.2 ขั้นตอนการติดตั้ง

1. ดาวน์โหลดจาก https://code.visualstudio.com/
2. ติดตั้งตามระบบปฏิบัติการของคุณ
3. เปิด VS Code แล้วติดตั้ง Extension ต่อไปนี้ (กด `Ctrl+Shift+X` หรือ `Cmd+Shift+X`
   เพื่อเปิด Extensions panel):

| Extension | ผู้พัฒนา | หน้าที่ |
|---|---|---|
| Python | Microsoft | IntelliSense, debugging, linting สำหรับ Python |
| Pylance | Microsoft | Type checking และ autocompletion ขั้นสูง |
| Django | Baptiste Darthenay | Syntax highlighting สำหรับ Django templates |
| GitLens | GitKraken | ดูประวัติ Git แบบละเอียดในไฟล์ |
| autoDocstring | Nils Werner | สร้าง docstring อัตโนมัติ |
| Even Better TOML | tamasfe | จัดการไฟล์ pyproject.toml |
| Ruff | Astral Software | Linter/formatter ความเร็วสูงสำหรับ Python |

### 5.3 ตั้งค่า VS Code สำหรับ Django โดยเฉพาะ

สร้างไฟล์ `.vscode/settings.json` ในโปรเจกต์ (เราจะสร้างโปรเจกต์จริงใน Part ถัดไป
แต่บันทึกการตั้งค่านี้ไว้ก่อน):

```json
{
    "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
    "python.analysis.extraPaths": ["./"],
    "files.associations": {
        "**/templates/**/*.html": "django-html",
        "**/requirements*.txt": "pip-requirements"
    },
    "emmet.includeLanguages": {
        "django-html": "html"
    },
    "editor.formatOnSave": true,
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff"
    }
}
```

การตั้งค่านี้จะช่วยให้ VS Code รู้จัก syntax ของ Django template (`.html` ที่มี
`{% %}` และ `{{ }}`), ตั้งค่า Python interpreter ให้ชี้ไปที่ virtual environment
ของโปรเจกต์เสมอ (เดี๋ยวเราจะสร้างใน ขั้นตอนที่ 8), และฟอร์แมตโค้ดอัตโนมัติทุกครั้งที่ save

---

## ขั้นตอนที่ 6: พื้นฐาน Command Line / Terminal ที่ต้องรู้

การพัฒนา Django (และงาน backend โดยทั่วไป) ต้องใช้ Terminal เป็นหลัก มาทบทวนคำสั่ง
พื้นฐานที่จำเป็น:

### 6.1 คำสั่งพื้นฐาน (ใช้ได้ทั้ง macOS/Linux, Windows ใช้ PowerShell)

```bash
# แสดง path ปัจจุบัน
pwd

# แสดงรายการไฟล์/โฟลเดอร์
ls          # macOS/Linux
dir         # Windows Command Prompt

# เปลี่ยนโฟลเดอร์
cd my-project
cd ..       # ย้อนกลับ 1 ระดับ
cd ~        # ไปที่ home directory

# สร้างโฟลเดอร์ใหม่
mkdir my-django-project

# สร้างไฟล์เปล่า
touch app.py        # macOS/Linux
New-Item app.py      # Windows PowerShell

# ลบไฟล์
rm app.py            # macOS/Linux
Remove-Item app.py   # Windows PowerShell

# ลบโฟลเดอร์ทั้งหมด (ระวัง! ลบแบบไม่มี trash)
rm -rf my-folder     # macOS/Linux
Remove-Item -Recurse -Force my-folder  # Windows

# คัดลอกไฟล์
cp source.py destination.py

# ย้าย/เปลี่ยนชื่อไฟล์
mv old_name.py new_name.py

# แสดงเนื้อหาไฟล์
cat app.py           # macOS/Linux
type app.py          # Windows

# ค้นหาข้อความในไฟล์
grep "def " app.py
```

### 6.2 คำแนะนำด้านความปลอดภัยเมื่อใช้ Terminal

- **คิดก่อนกด Enter เสมอเมื่อใช้ `rm -rf`** เพราะไม่มี Undo และไม่มีถังขยะ
- อย่ารันคำสั่งที่ก็อปมาจากอินเทอร์เน็ตโดยไม่เข้าใจว่ามันทำอะไร โดยเฉพาะที่มี `sudo`
- ใช้ `man <command>` (macOS/Linux) เพื่อดูคู่มือคำสั่งนั้น ๆ เช่น `man rm`

### 6.3 Shell ที่แนะนำ

- **macOS**: มี `zsh` เป็นค่าเริ่มต้นตั้งแต่ macOS Catalina เป็นต้นไป
- **Linux**: ส่วนใหญ่ใช้ `bash` เป็นค่าเริ่มต้น แนะนำให้ลองใช้ `zsh` ร่วมกับ
  Oh My Zsh (https://ohmyz.sh/) เพื่อ productivity ที่ดีขึ้น
- **Windows**: แนะนำใช้ **Windows Terminal** + **WSL2 (Windows Subsystem for Linux)**
  เพื่อให้ได้ประสบการณ์ Linux-like ซึ่งใกล้เคียงกับ production server จริงมากที่สุด

```powershell
# ติดตั้ง WSL2 บน Windows (รันใน PowerShell แบบ Administrator)
wsl --install

# หลังติดตั้งเสร็จ รีสตาร์ทเครื่อง แล้วเปิด Ubuntu จาก Start Menu
```

หลักสูตรนี้แนะนำอย่างยิ่งให้ผู้ใช้ Windows ติดตั้ง WSL2 เพราะคำสั่งส่วนใหญ่ในหลักสูตร
จะเขียนในรูปแบบ Unix-like (bash) ซึ่งตรงกับสภาพแวดล้อม production server จริง
(ส่วนใหญ่รันบน Linux)

---

## ขั้นตอนที่ 7: ติดตั้ง Git และเชื่อมต่อ GitHub

### 7.1 ทำไม Git ถึงจำเป็น

Git คือระบบ **Version Control** ที่ช่วยติดตามการเปลี่ยนแปลงของโค้ด ทำงานร่วมกับทีมได้
และเป็นทักษะพื้นฐานที่นักพัฒนาทุกคนต้องมี ไม่ว่าจะทำงานคนเดียวหรือทีมใหญ่

### 7.2 ติดตั้ง Git

```bash
# macOS (ผ่าน Homebrew)
brew install git

# Ubuntu/Debian
sudo apt install git -y

# Windows: ดาวน์โหลดจาก https://git-scm.com/download/win
```

ตรวจสอบการติดตั้ง:

```bash
git --version
```

### 7.3 ตั้งค่า Git ครั้งแรก (Global Configuration)

```bash
git config --global user.name "ชื่อของคุณ"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"   # ใช้ VS Code เป็น editor เริ่มต้น

# ตรวจสอบการตั้งค่าทั้งหมด
git config --list
```

### 7.4 สร้างบัญชี GitHub และตั้งค่า SSH Key

1. สมัครบัญชีที่ https://github.com (ถ้ายังไม่มี)
2. สร้าง SSH Key เพื่อเชื่อมต่อโดยไม่ต้องพิมพ์รหัสผ่านทุกครั้ง:

```bash
# สร้าง SSH Key ใหม่
ssh-keygen -t ed25519 -C "your-email@example.com"
# กด Enter ทุกครั้งที่ถาม (ใช้ค่า default)

# เริ่ม ssh-agent
eval "$(ssh-agent -s)"

# เพิ่ม key เข้า agent
ssh-add ~/.ssh/id_ed25519

# คัดลอก public key ไปวางใน GitHub
cat ~/.ssh/id_ed25519.pub
# macOS: pbcopy < ~/.ssh/id_ed25519.pub
```

3. ไปที่ GitHub → Settings → SSH and GPG keys → New SSH key → วาง public key ที่คัดลอกมา
4. ทดสอบการเชื่อมต่อ:

```bash
ssh -T git@github.com
# ควรเห็นข้อความ "Hi <username>! You've successfully authenticated..."
```

### 7.5 คำสั่ง Git พื้นฐานที่ต้องใช้บ่อยที่สุด

```bash
git init                          # เริ่มต้น repository ใหม่
git status                        # ดูสถานะไฟล์ที่เปลี่ยนแปลง
git add .                         # เพิ่มไฟล์ทั้งหมดเข้า staging area
git add filename.py               # เพิ่มไฟล์เฉพาะเจาะจง
git commit -m "ข้อความอธิบายการเปลี่ยนแปลง"
git log --oneline                 # ดูประวัติ commit แบบย่อ
git branch                        # ดูรายการ branch
git branch feature/login          # สร้าง branch ใหม่
git checkout feature/login        # สลับไป branch นั้น
git checkout -b feature/login     # สร้างและสลับในคำสั่งเดียว
git push origin main              # push โค้ดขึ้น remote repository
git pull origin main              # ดึงโค้ดล่าสุดจาก remote
git clone git@github.com:user/repo.git   # clone repository มาไว้ในเครื่อง
```

เราจะฝึกใช้ Git ควบคู่ไปกับทุก Part ของหลักสูตร โดยเฉพาะ Part 088 (CI/CD) ที่จะเจาะลึก
GitHub Actions และ Git Workflow ระดับทีม

---

## ขั้นตอนที่ 8: สร้าง Virtual Environment ครั้งแรก

### 8.1 Virtual Environment คืออะไร และทำไมสำคัญมาก

**Virtual Environment (venv)** คือสภาพแวดล้อม Python ที่แยกออกจากระบบหลัก ทำให้แต่ละ
โปรเจกต์สามารถติดตั้ง package คนละเวอร์ชันกันได้โดยไม่ชนกัน

ตัวอย่างปัญหาที่เกิดถ้าไม่ใช้ venv:

```
โปรเจกต์ A ต้องการ Django==4.2
โปรเจกต์ B ต้องการ Django==5.1

ถ้าติดตั้ง Django ลงระบบโดยตรง (global) จะมีได้แค่เวอร์ชันเดียว
เมื่อสลับไปทำโปรเจกต์ B ต้อง uninstall/install ใหม่ตลอดเวลา → เสียเวลา และเสี่ยงพัง
```

การใช้ venv แก้ปัญหานี้โดยสร้าง "กล่องแยก" สำหรับแต่ละโปรเจกต์:

```
โปรเจกต์ A → venv_A → Django 4.2
โปรเจกต์ B → venv_B → Django 5.1
(ทำงานพร้อมกันได้โดยไม่ชนกัน)
```

**กฎเหล็กของหลักสูตรนี้: ห้ามติดตั้ง package ใด ๆ แบบ global (นอก venv) เด็ดขาด**
นี่คือหนึ่งในนิสัยที่แยกมือใหม่ออกจากมืออาชีพตั้งแต่ Day 1

### 8.2 สร้าง Virtual Environment ด้วย venv (มากับ Python อยู่แล้ว)

```bash
# สร้างโฟลเดอร์โปรเจกต์
mkdir django-mastery-course
cd django-mastery-course

# สร้าง virtual environment ชื่อ "venv"
python3 -m venv venv

# เปิดใช้งาน (activate) virtual environment
# macOS/Linux:
source venv/bin/activate

# Windows Command Prompt:
venv\Scripts\activate.bat

# Windows PowerShell:
venv\Scripts\Activate.ps1
```

เมื่อ activate สำเร็จ คุณจะเห็นชื่อ `(venv)` ปรากฏหน้า prompt ของ Terminal เช่น:

```
(venv) user@computer django-mastery-course %
```

นี่คือสัญญาณว่าตอนนี้คุณกำลังทำงานอยู่ **ภายใน** virtual environment แล้ว

### 8.3 คำสั่งที่เกี่ยวข้องกับ venv

```bash
# ปิดการใช้งาน venv (กลับสู่ระบบปกติ)
deactivate

# ตรวจสอบว่ากำลังใช้ Python ตัวไหนอยู่
which python      # macOS/Linux
where python       # Windows

# ควรเห็น path ชี้ไปที่โฟลเดอร์ venv เช่น
# /Users/you/django-mastery-course/venv/bin/python
```

### 8.4 ทางเลือกอื่น: uv, pipenv, poetry

ในโลกมืออาชีพปัจจุบัน (2025-2026) มีเครื่องมือจัดการ dependency ที่ทันสมัยกว่า venv+pip
เดี่ยว ๆ เช่น:

- **uv** (จาก Astral, ผู้สร้าง Ruff): เร็วกว่า pip 10-100 เท่า กำลังเป็นที่นิยมมาก
- **Poetry**: จัดการ dependency + packaging + virtual env ในตัวเดียว
- **Pipenv**: รวม pip + venv เข้าด้วยกัน พร้อม Pipfile.lock

หลักสูตรนี้จะเริ่มต้นด้วย `venv` + `pip` มาตรฐาน เพราะเข้าใจง่ายที่สุดสำหรับผู้เริ่มต้น
แต่จะแนะนำ **uv** ในภายหลัง (Part 003) เมื่อพื้นฐานแน่นแล้ว เพราะเป็นเครื่องมือที่ทีม
งานระดับมืออาชีพจำนวนมากเปลี่ยนมาใช้ในปี 2025 เป็นต้นมา

### 8.5 ไฟล์ .gitignore ที่ต้องมีตั้งแต่ต้น

ก่อนจะ commit โค้ดขึ้น Git ต้องสร้างไฟล์ `.gitignore` เพื่อไม่ให้ venv และไฟล์ขยะอื่น ๆ
ถูก track โดย Git:

```bash
# .gitignore
venv/
__pycache__/
*.pyc
*.pyo
.env
db.sqlite3
.vscode/
.idea/
*.log
.DS_Store
media/
staticfiles/
```

สร้างไฟล์นี้ด้วยคำสั่ง:

```bash
touch .gitignore
# แล้วเปิดด้วย VS Code เพื่อวางเนื้อหาข้างต้น
code .gitignore
```

---

## ขั้นตอนที่ 9: ติดตั้ง Django และตรวจสอบเวอร์ชัน

### 9.1 ติดตั้ง Django ผ่าน pip

ตรวจสอบให้แน่ใจว่า venv ถูก activate อยู่ (เห็น `(venv)` หน้า prompt) แล้วรัน:

```bash
# อัปเดต pip ให้เป็นเวอร์ชันล่าสุดก่อนเสมอ
python -m pip install --upgrade pip

# ติดตั้ง Django เวอร์ชันล่าสุด
pip install django

# หรือระบุเวอร์ชันเจาะจง (แนะนำสำหรับหลักสูตรนี้)
pip install "django>=5.1,<5.2"
```

### 9.2 ตรวจสอบการติดตั้ง

```bash
python -m django --version
# ผลลัพธ์ที่คาดหวัง: 5.1.x

django-admin --version
```

### 9.3 บันทึก dependency ลงไฟล์ requirements.txt

นี่คือขั้นตอนสำคัญที่มือใหม่มักลืม การบันทึกรายการ package ที่ใช้ทำให้คนอื่น
(หรือตัวเองในอนาคต) สามารถติดตั้ง environment เดียวกันได้:

```bash
pip freeze > requirements.txt
cat requirements.txt
```

ผลลัพธ์ควรมีลักษณะประมาณนี้:

```
asgiref==3.8.1
Django==5.1.2
sqlparse==0.5.1
```

เมื่อมีคนอื่นมา clone โปรเจกต์ของคุณ พวกเขาจะสามารถติดตั้ง dependency ทั้งหมดในคำสั่งเดียว:

```bash
pip install -r requirements.txt
```

### 9.4 ทำความเข้าใจ Django Release Cycle และ LTS

Django มีระบบการออกเวอร์ชันที่ชัดเจนซึ่งนักพัฒนามืออาชีพต้องเข้าใจ:

- Django ออกเวอร์ชันใหม่ทุก ๆ **8 เดือน** ประมาณ (เช่น 5.0 → 5.1 → 5.2)
- ทุก ๆ 2 ปี จะมีเวอร์ชัน **LTS (Long Term Support)** ที่ได้รับการซัพพอร์ตด้าน
  security patch นานถึง **3 ปี** (เวอร์ชันปกติได้รับซัพพอร์ตแค่ ~16 เดือน)
- ตัวอย่าง LTS: Django 4.2 LTS, Django 5.2 LTS (คาดการณ์)

**คำแนะนำระดับมืออาชีพ**: สำหรับโปรเจกต์ production จริง ให้เลือกใช้เวอร์ชัน **LTS**
เสมอ เพื่อความเสถียรระยะยาวและลดความถี่ในการอัปเกรดใหญ่ ส่วนหลักสูตรนี้จะใช้เวอร์ชัน
ล่าสุด (non-LTS ก็ได้) เพื่อให้ได้เรียนรู้ฟีเจอร์ใหม่ที่สุด แต่หลักการที่สอนจะใช้ได้กับ
ทุกเวอร์ชัน 4.2 ขึ้นไป

ตรวจสอบตารางการซัพพอร์ตล่าสุดได้ที่: https://www.djangoproject.com/download/#supported-versions

### 9.5 ทดสอบว่า Django ทำงานได้จริงด้วยคำสั่งเดียว

```bash
python -c "import django; print(django.get_version())"
```

หากได้ผลลัพธ์เป็นเลขเวอร์ชัน (เช่น `5.1.2`) แสดงว่าทุกอย่างพร้อมแล้ว!

---

## ขั้นตอนที่ 10: สรุปและแบบฝึกหัด

### 10.1 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจภาพรวมการทำงานของเว็บแอปพลิเคชันแบบ Client-Server
- ✅ รู้จัก Django และจุดเด่นแบบ "Batteries Included"
- ✅ เข้าใจสถาปัตยกรรม MTV (Model-Template-View) และความแตกต่างจาก MVC ทั่วไป
- ✅ ติดตั้ง Python เวอร์ชันล่าสุดบนเครื่อง
- ✅ ติดตั้งและตั้งค่า VS Code พร้อม Extension ที่จำเป็น
- ✅ ทบทวนคำสั่ง Terminal พื้นฐาน
- ✅ ติดตั้ง Git และเชื่อมต่อ GitHub ด้วย SSH Key
- ✅ เข้าใจและสร้าง Virtual Environment ได้
- ✅ ติดตั้ง Django และตรวจสอบเวอร์ชันสำเร็จ

### 10.2 Checklist ก่อนไป Part ถัดไป

ทำเครื่องหมายในใจ (หรือจดบันทึก) ว่าคุณทำสิ่งเหล่านี้สำเร็จแล้ว:

- [ ] รันคำสั่ง `python3 --version` แล้วได้ผลลัพธ์เวอร์ชัน 3.10 ขึ้นไป
- [ ] เปิด VS Code ได้ และติดตั้ง extension Python + Django แล้ว
- [ ] รันคำสั่ง `git --version` ได้ และตั้งค่า `user.name`, `user.email` แล้ว
- [ ] ทดสอบ `ssh -T git@github.com` สำเร็จ (เห็นข้อความ authenticated)
- [ ] สร้างโฟลเดอร์โปรเจกต์ `django-mastery-course` และสร้าง venv สำเร็จ
- [ ] Activate venv และเห็น `(venv)` หน้า prompt
- [ ] รันคำสั่ง `pip install django` สำเร็จ
- [ ] รันคำสั่ง `python -m django --version` แล้วเห็นเลขเวอร์ชัน
- [ ] สร้างไฟล์ `requirements.txt` และ `.gitignore` แล้ว

### 10.3 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: อธิบายด้วยคำพูดของตัวเอง (เขียนลงในไฟล์ `notes.md`) ว่าเมื่อผู้ใช้
พิมพ์ `https://mysite.com/about/` ในเบราว์เซอร์ เกิดอะไรขึ้นบ้างตามลำดับ โดยอ้างอิงถึง
Model, View, Template, และ URL Dispatcher

**แบบฝึกหัดที่ 2**: สร้าง GitHub repository ใหม่ชื่อ `django-mastery-course`
(private หรือ public ก็ได้) แล้ว push โฟลเดอร์ที่คุณสร้างในขั้นตอนที่ 8-9 ขึ้นไป
พร้อมไฟล์ `.gitignore` และ `requirements.txt` (**ห้าม push โฟลเดอร์ venv/ ขึ้นไปเด็ดขาด**
ตรวจสอบด้วย `git status` ก่อน commit ทุกครั้ง)

**แบบฝึกหัดที่ 3**: ค้นหาข้อมูลเพิ่มเติมเกี่ยวกับเว็บไซต์หรือแอปพลิเคชันอย่างน้อย 2 แห่ง
(นอกเหนือจากที่กล่าวถึงในบทนี้) ที่ใช้ Django เป็น backend แล้วเขียนสรุปสั้น ๆ ว่าทำไม
บริษัทเหล่านั้นถึงเลือกใช้ Django

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: ลองติดตั้ง `uv` (https://github.com/astral-sh/uv) แล้วใช้
สร้าง virtual environment เปรียบเทียบความเร็วกับ `venv` มาตรฐาน บันทึกผลเวลาที่ใช้
ในการติดตั้ง Django ด้วยทั้งสองวิธี

### 10.4 คำถามที่พบบ่อย (FAQ)

**Q: จำเป็นต้องรู้ HTML/CSS/JavaScript ก่อนเรียน Django หรือไม่?**
A: ควรมีพื้นฐาน HTML เล็กน้อยเพื่อเข้าใจ Template แต่ไม่จำเป็นต้องเชี่ยวชาญ CSS/JS
เพราะ Django เน้นฝั่ง Backend เป็นหลัก เราจะแนะนำ HTML ที่จำเป็นเมื่อถึง Part 008

**Q: ต้องรู้ SQL มาก่อนหรือไม่?**
A: ไม่จำเป็น เพราะ Django ORM ช่วยให้เขียน Python แทน SQL ได้ แต่การเข้าใจ SQL
พื้นฐานจะช่วยให้ debug และ optimize query ได้ดีขึ้นมากในระดับ Phase 8

**Q: ทำไมต้องใช้ PostgreSQL แทน SQLite ทั้งหลักสูตร?**
A: SQLite เหมาะสำหรับพัฒนาเบื้องต้นเท่านั้น เพราะไม่รองรับ concurrent write ที่ดีพอ
สำหรับ production เราจะเริ่มด้วย SQLite ใน Part แรก ๆ เพื่อความง่าย แล้วเปลี่ยนไปใช้
PostgreSQL ตั้งแต่ Phase 2 เป็นต้นไป ซึ่งเป็นมาตรฐานอุตสาหกรรมจริง

**Q: ควรใช้ Windows, macOS หรือ Linux ในการเรียน?**
A: ใช้ได้ทั้งหมด แต่ถ้าใช้ Windows แนะนำอย่างยิ่งให้ติดตั้ง WSL2 ตามที่แนะนำในขั้นตอนที่ 6
เพื่อให้คำสั่งทุกอย่างในหลักสูตรตรงกับที่คุณพิมพ์ได้เป๊ะ ๆ

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 002: ทบทวน Python ที่จำเป็นสำหรับ Django Developer** จะพาไปทบทวนแนวคิด Python
ที่ Django ใช้งานหนักที่สุด ได้แก่ Classes & OOP, Decorators, Context Managers,
List/Dict Comprehension, Type Hints และ `*args`/`**kwargs` ซึ่งเป็นพื้นฐานสำคัญก่อนที่
เราจะเริ่มเขียน Django Model และ View จริงใน Part 004

เตรียมเปิด VS Code และ Terminal ของคุณไว้ให้พร้อม แล้วไปต่อกันเลย!
