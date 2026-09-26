# Part 064: Integration Testing และ Selenium

> **ขั้นตอนที่ 631-640 ของหลักสูตร** | Phase 7: Testing & Quality Assurance
>
> Part 059-063 สอนให้คุณเขียน test ที่รันเร็วมาก — `unittest`/`TestCase`,
> `pytest-django`, coverage, mocking, factory_boy — แต่ทุก test เหล่านั้นมี
> ข้อจำกัดร่วมกันหนึ่งอย่าง: **`django.test.Client` ไม่ใช่เบราว์เซอร์จริง**
> มันแค่จำลอง HTTP request/response ในหน่วยความจำ ไม่มี DOM ไม่มี CSS ไม่รัน
> JavaScript แม้แต่บรรทัดเดียว ตราบใดที่แอปของคุณยังเป็น server-rendered
> ธรรมดา นั่นก็เพียงพอ แต่ตั้งแต่ Part 053-054 เป็นต้นมา บล็อกของคุณมีปุ่ม
> ถูกใจที่อัปเดตด้วย HTMX โดยไม่ reload หน้า และมี Modal ที่เปิด/ปิดด้วย
> Alpine.js ล้วน ๆ ฝั่ง client — พฤติกรรมเหล่านี้**เกิดขึ้นในเบราว์เซอร์เท่านั้น**
> `self.client.get()` มองไม่เห็นเลยว่าปุ่มนั้นทำงานถูกต้องหรือไม่หลังคลิก
> Part นี้จะพาคุณไปรู้จัก **Integration Testing และ End-to-End (E2E) Testing**
> ที่ควบคุมเบราว์เซอร์จริงด้วย **Selenium** และเครื่องมือสมัยใหม่กว่าอย่าง
> **Playwright** ตั้งแต่ทฤษฎี Testing Pyramid ที่บอกว่าควรมี test แต่ละชนิด
> ในสัดส่วนเท่าไหร่ ไปจนถึงการเขียน test ที่คลิกปุ่ม กรอกฟอร์ม ยืนยันผลลัพธ์
> บนหน้าจอจริง ถ่าย screenshot อัตโนมัติเมื่อ test ล้มเหลว รันแบบ headless
> ใน CI และรับมือกับ "flaky test" ที่เป็นฝันร้ายของทุกทีม เมื่อจบ Part นี้
> คุณจะเขียน E2E test ที่จำลองผู้ใช้จริงตั้งแต่ login → สร้างโพสต์ →
> เห็นโพสต์นั้นปรากฏในหน้า list ได้ทั้งหมดโดยอัตโนมัติ

---

## สารบัญของ Part นี้

- ขั้นตอนที่ 631: Testing Pyramid — Unit vs Integration vs End-to-End (E2E) เปรียบเทียบปริมาณ/ความเร็ว/ความมั่นใจ
- ขั้นตอนที่ 632: `LiveServerTestCase` เบื้องต้น — รัน Django server จริงสำหรับ test
- ขั้นตอนที่ 633: ติดตั้งและตั้งค่า Selenium กับ Django (webdriver-manager)
- ขั้นตอนที่ 634: เขียน Browser Automation Test จริง — คลิก, กรอกฟอร์ม, ตรวจสอบผลลัพธ์
- ขั้นตอนที่ 635: Playwright เป็นทางเลือกสมัยใหม่แทน Selenium (เร็วกว่า, auto-wait, เขียนง่ายกว่า)
- ขั้นตอนที่ 636: Testing หน้าที่มี JavaScript หนัก (HTMX/Alpine.js จาก Phase 6) ที่ unittest ธรรมดาทดสอบไม่ได้
- ขั้นตอนที่ 637: การถ่าย Screenshot อัตโนมัติเมื่อ test ล้มเหลว (debug ง่ายขึ้น)
- ขั้นตอนที่ 638: รัน Headless Browser Test ใน CI (เกริ่น เจาะลึกเต็มใน Part 088)
- ขั้นตอนที่ 639: กลยุทธ์รับมือ Flaky Test (test ที่บางครั้งผ่านบางครั้งไม่ผ่าน)
- ขั้นตอนที่ 640: สรุปและแบบฝึกหัด — เขียน E2E test สำหรับ flow login → สร้างโพสต์ → เห็นโพสต์ในหน้า list

---

## ขั้นตอนที่ 631: Testing Pyramid — Unit vs Integration vs E2E

### 631.1 ทบทวน: ทำไม Part 059-063 ยังไม่พอ

ตลอด Part 059-063 คุณเขียน test มาแล้วหลายร้อยเคสด้วย `django.test.TestCase`
และ `pytest-django` เช่น:

```python
# ตัวอย่าง unit test จาก Part 060 (ทบทวน)
from django.test import TestCase
from django.contrib.auth import get_user_model
from blog.models import Post

User = get_user_model()


class PostModelTests(TestCase):
    def test_str_returns_title(self):
        user = User.objects.create_user(username="author1", password="x")
        post = Post.objects.create(title="สวัสดี Django", slug="hello-django",
                                    content="เนื้อหา", author=user)
        self.assertEqual(str(post), "สวัสดี Django")
```

test แบบนี้เร็วมาก (หลักมิลลิวินาที) และแม่นยำมากในการบอกว่า "โค้ดชิ้นนี้ทำงาน
ถูกต้องหรือไม่" แต่มันตอบคำถามที่ **แคบมาก**: มันไม่เคยเปิดเบราว์เซอร์ ไม่เคย
เห็นว่าปุ่มบนหน้าเว็บอยู่ตรงไหน ไม่เคยรัน JavaScript สักบรรทัด และไม่เคยรู้ว่า
CSS ทำให้ปุ่มถูกซ่อนอยู่หลังปุ่มอื่นหรือเปล่า คำถามเหล่านี้ต้องการ test ที่
"มองเห็นสิ่งที่ผู้ใช้เห็นจริง ๆ" ซึ่งเป็นหัวข้อของ Part นี้ทั้งหมด

### 631.2 นิยามของ Unit / Integration / End-to-End Test

| ระดับ | นิยาม | ตัวอย่างในโปรเจกต์บล็อกของเรา |
|---|---|---|
| **Unit Test** | ทดสอบหน่วยเล็กที่สุด (ฟังก์ชัน, method, class เดียว) แบบแยกขาด (isolated) จากส่วนอื่น มัก mock dependency ทั้งหมด | ทดสอบ `Post.__str__()`, ทดสอบ validator ของ `CommentForm`, ทดสอบฟังก์ชัน `slugify_title()` |
| **Integration Test** | ทดสอบว่าหลายส่วนทำงาน**ร่วมกัน**ถูกต้อง (View + Model + Database + URL routing) ผ่าน HTTP request จำลอง แต่ยังไม่ใช่เบราว์เซอร์จริง | `self.client.post('/blog/new/', data)` แล้วตรวจว่า `Post` ถูกสร้างในฐานข้อมูลจริงและ response redirect ถูก URL |
| **End-to-End (E2E) Test** | ทดสอบทั้งระบบผ่าน**เบราว์เซอร์จริง** ตั้งแต่ UI จนถึงฐานข้อมูล จำลองผู้ใช้จริงคลิก/พิมพ์/เลื่อนหน้าจอ | เปิดเบราว์เซอร์ พิมพ์ username/password กด login คลิก "สร้างโพสต์ใหม่" กรอกฟอร์ม กด submit แล้วตรวจว่าโพสต์ปรากฏในหน้า list จริง |

Part 059-060 สอน Unit Test, Part 060 (ส่วน View testing ด้วย `self.client`)
กับ Part 061-063 ส่วนใหญ่คือ **Integration Test** ในความหมายเชิงวิชาการ
(เพราะ `self.client` ยิง request ผ่าน middleware, URL routing, View, ORM
จริงทั้งหมด เพียงแต่ไม่ผ่านเบราว์เซอร์) ส่วน Part นี้คือก้าวสุดท้ายของพีระมิด
คือ **End-to-End Test** ที่ควบคุมเบราว์เซอร์จริง

### 631.3 Testing Pyramid: แผนภาพและตารางเปรียบเทียบ

แนวคิด **Testing Pyramid** ถูกเผยแพร่ครั้งแรกโดย Mike Cohn ในหนังสือ
*Succeeding with Agile* (2009) และถูกนำมาขยายความอย่างมีอิทธิพลโดย
Martin Fowler แนวคิดหลักคือ: ยิ่งขึ้นไปสูง (E2E) จำนวน test ควรยิ่งน้อยลง
เพราะ**ต้นทุน**ในการเขียน/รัน/ดูแลรักษาสูงขึ้นเรื่อย ๆ

```
                    ▲  ความมั่นใจต่อ test 1 เคส (Confidence)
                    │  ความเร็วในการรันต่อ test 1 เคส (ช้าลง)
                    │  ต้นทุนการดูแลรักษา (สูงขึ้น)
                    │
                 ╱─────╲
                ╱  E2E   ╲          ← จำนวนน้อยที่สุด (~10%)
               ╱  (Part   ╲            รันช้าที่สุด (วินาที-นาที/เคส)
              ╱   064 นี้)  ╲          มั่นใจสูงสุด (ทดสอบทั้งระบบจริง)
             ╱───────────────╲
            ╱   Integration    ╲    ← จำนวนปานกลาง (~20%)
           ╱  (Part 060-061,     ╲      รันปานกลาง (สิบ-ร้อยมิลลิวินาที/เคส)
          ╱   self.client)        ╲     มั่นใจปานกลาง (ทดสอบ View+Model+DB)
         ╱───────────────────────────╲
        ╱          Unit Test            ╲   ← จำนวนมากที่สุด (~70%)
       ╱   (Part 059-060, 062-063)        ╲     รันเร็วที่สุด (มิลลิวินาที/เคส)
      ╱───────────────────────────────────────╲  มั่นใจต่อเคสต่ำสุด (แต่รวมกันครอบคลุมมาก)
      ▼  จำนวนเคสทั้งหมดในระบบ (Volume)
```

| มิติ | Unit Test | Integration Test | E2E Test |
|---|---|---|---|
| **ปริมาณที่แนะนำ** | มากที่สุด (~70% ของ suite) | ปานกลาง (~20%) | น้อยที่สุด (~10%) |
| **ความเร็วต่อเคส** | เร็วมาก (< 10ms) | ปานกลาง (10-200ms) | ช้า (1-30+ วินาที) |
| **ความมั่นใจต่อเคส** | ต่ำ (ทดสอบส่วนเดียว, mock เยอะ) | ปานกลาง (ทดสอบหลายชั้นจริง) | สูงสุด (ทดสอบทั้งระบบเหมือนผู้ใช้จริง) |
| **จุดที่ fail แล้วรู้ทันที** | บอกตำแหน่งบั๊กชัดเจนมาก | บอกได้กว้าง ๆ (View นี้มีปัญหา) | บอกได้แค่ "flow นี้พัง" ต้อง debug ต่อ |
| **ความเปราะบาง (flakiness)** | แทบไม่มี | น้อย | สูงสุด (ขั้นตอนที่ 639) |
| **ต้นทุนดูแลรักษาเมื่อ UI เปลี่ยน** | ไม่กระทบ | กระทบเล็กน้อย | กระทบมาก (selector เปลี่ยน, timing เปลี่ยน) |
| **ต้องใช้เครื่องมืออะไร** | `unittest`, `pytest`, `unittest.mock` | `django.test.Client`, `pytest-django` | Selenium / Playwright + เบราว์เซอร์จริง |
| **เหมาะทดสอบอะไร** | Logic ล้วน ๆ, edge case ของฟังก์ชัน | View + Form + ORM ทำงานร่วมกันถูกต้อง | Critical user journey (login, checkout, สมัครสมาชิก) |

### 631.4 Google Test Sizes: มุมมองอีกแบบที่ใช้ในทีมระดับโลก

นอกจาก Testing Pyramid แบบดั้งเดิม ทีมวิศวกรรมของ Google ใช้แนวคิด
**"Test Sizes"** (small/medium/large) ซึ่งคล้ายกันแต่เน้นที่ "resource ที่
test ต้องใช้" มากกว่าชื่อประเภท:

| Size | นิยามของ Google | เทียบกับพีระมิดของเรา |
|---|---|---|
| **Small** | รันในโปรเซสเดียว ห้ามมี network/disk/database จริง (mock ทุกอย่างภายนอก) | Unit Test |
| **Medium** | รันบนเครื่องเดียวได้ อนุญาตให้เข้าถึง localhost/database จริง แต่ห้ามมี network ข้ามเครื่อง | Integration Test |
| **Large** | อนุญาตทุกอย่าง รวมถึงเรียกบริการภายนอกจริง, ควบคุม UI จริง | E2E Test |

แนวคิดนี้สำคัญเพราะช่วยตอบคำถามที่มักถกเถียงกัน เช่น "test ที่ใช้
`LiveServerTestCase` (ขั้นตอนที่ 632) แต่ไม่ได้เปิดเบราว์เซอร์ นับเป็น Unit
หรือ Integration?" — คำตอบตามแนวคิด Google คือมันเป็นอย่างน้อย **Medium**
เพราะต้องเปิด TCP port จริง แม้จะยังไม่ใช่ Large ก็ตาม

### 631.5 กับดัก "Ice Cream Cone Anti-pattern"

ทีมจำนวนมากที่เริ่มทำ Automated Testing ทีหลัง (เช่น หลังระบบใช้งานจริงมา
หลายปีแล้ว) มักตกหลุมพรางที่เรียกว่า **"Ice Cream Cone"** — พีระมิดกลับหัว:

```
      ╲───────────────────────╲   ← E2E Test เยอะที่สุด (เพราะเขียนง่ายสุด
       ╲    E2E Test (เยอะ)    ╲     สำหรับคนที่ไม่คุ้น unit test แค่คลิก
        ╲───────────────────────╲    ผ่านหน้าเว็บแล้ว "ดูเหมือนจะครอบคลุม")
         ╲  Integration (ปานกลาง)╲
          ╲───────────────────────╲
           ╲   Unit Test (น้อย)    ╲  ← มีน้อยเพราะ "เขียนยากกว่า" ทั้งที่
            ╲───────────────────────╲    ควรเป็นฐานที่ใหญ่ที่สุด
```

ผลลัพธ์ของทีมที่ตกหลุมนี้: test suite **รันช้ามาก** (ทุกอย่างต้องเปิดเบราว์เซอร์)
**เปราะบางมาก** (UI เปลี่ยนนิดเดียว test พังทั้งกระดาน) และ **หา root cause
ยาก** เมื่อ test ล้มเหลว (รู้แค่ว่า "หน้า checkout พัง" แต่ไม่รู้ว่าพังที่ Model,
View, หรือ Template) หลักสูตรนี้ยึดพีระมิดแบบปกติเสมอ: **เขียน Unit Test
ให้ครอบคลุม logic ก่อนเสมอ (Part 059-063), ใช้ Integration Test คุมพฤติกรรม
ของ View/Form (Part 060-061), และสงวน E2E Test ไว้กับ "user journey สำคัญ
ที่สุด" เพียงไม่กี่เส้นทางเท่านั้น** (login, การสมัครสมาชิก, การชำระเงิน,
และ flow ที่ต้องพึ่ง JavaScript อย่าง HTMX/Alpine.js ที่ unit test มองไม่เห็น)

---

## ขั้นตอนที่ 632: `LiveServerTestCase` เบื้องต้น

### 632.1 ปัญหาของ `django.test.Client`: ไม่มี "เซิร์ฟเวอร์" จริงอยู่เบื้องหลัง

`django.test.Client` ที่ใช้มาตั้งแต่ Part 060 **ไม่ได้ยิง HTTP request ผ่าน
เครือข่ายจริงเลย** มันเรียกฟังก์ชัน View ของ Django โดยตรงในหน่วยความจำเดียวกัน
กับตัว test process ไม่มีการเปิด TCP socket ไม่มี host:port ให้เชื่อมต่อ —
นี่คือเหตุผลที่ `Client` เร็วมาก แต่ก็หมายความว่า**ไม่มีสิ่งใดให้เบราว์เซอร์จริง
เชื่อมต่อเข้ามาได้เลย** เพราะไม่มี server รันอยู่จริง ๆ

**`LiveServerTestCase`** แก้ปัญหานี้โดยรัน Django development server ตัวจริง
ในเธรดแยกต่างหาก (background thread) บน port ที่สุ่มเลือกได้ ตลอดช่วงเวลาที่
test class นั้นทำงาน ทำให้มี URL จริง (เช่น `http://localhost:47291/`) ให้
เบราว์เซอร์หรือ HTTP client ใด ๆ เชื่อมต่อเข้ามาได้เหมือนระบบ production จริง

### 632.2 ตัวอย่างแรก: ทดสอบด้วย `requests` library ก่อนไปใช้ Selenium

ก่อนจะเพิ่มความซับซ้อนของเบราว์เซอร์ในขั้นตอนที่ 633 มาดูก่อนว่า
`LiveServerTestCase` ทำอะไรให้เราบ้าง โดยทดสอบง่าย ๆ ด้วย library
`requests` ธรรมดา (ทบทวนจาก Part 048 ที่ใช้ `requests` เรียก external API):

```python
# blog/tests/test_live_server_basics.py
import requests
from django.test import LiveServerTestCase
from blog.models import Post
from django.contrib.auth import get_user_model

User = get_user_model()


class LiveServerBasicsTests(LiveServerTestCase):
    """สาธิตว่า LiveServerTestCase สร้างเซิร์ฟเวอร์จริงที่เข้าถึงผ่าน HTTP ได้"""

    def setUp(self):
        self.author = User.objects.create_user(username="writer", password="x")
        Post.objects.create(
            title="ทดสอบ LiveServerTestCase",
            slug="test-live-server",
            content="เนื้อหาตัวอย่าง",
            author=self.author,
            is_published=True,
        )

    def test_server_is_reachable_over_real_http(self):
        # self.live_server_url คือ URL จริง เช่น http://localhost:47291
        response = requests.get(f"{self.live_server_url}/blog/", timeout=5)

        self.assertEqual(response.status_code, 200)
        self.assertIn("ทดสอบ LiveServerTestCase", response.text)
        # ตรวจว่านี่คือ HTML จริงที่ผ่าน network stack เต็มรูปแบบ ไม่ใช่ response
        # object จำลองแบบที่ django.test.Client คืนให้
        self.assertIn("text/html", response.headers["Content-Type"])
```

รันด้วย:

```bash
python manage.py test blog.tests.test_live_server_basics
```

สังเกตความแตกต่างสำคัญจาก `self.client.get('/blog/')` ที่เคยใช้มา:
`self.live_server_url` เป็น **URL ข้อความจริง** ที่ส่งให้ `requests` (หรือ
เบราว์เซอร์ใด ๆ) เชื่อมต่อได้ตรง ๆ ผ่าน TCP/IP เต็มรูปแบบ

### 632.3 การตั้งค่าที่ต้องรู้: `ALLOWED_HOSTS` และพอร์ต

Django จัดการเรื่อง `ALLOWED_HOSTS` ให้อัตโนมัติระหว่างรัน test — เพิ่ม
`testserver` และ `localhost` เข้าไปให้ชั่วคราวโดยไม่ต้องแก้ `settings.py` เอง
แต่ถ้าต้องการกำหนดพอร์ตตายตัว (เช่น ต้องอ้างอิงจาก config อื่นภายนอก) ทำได้
ผ่าน attribute `host`/`port`:

```python
class LiveServerBasicsTests(LiveServerTestCase):
    host = "localhost"
    port = 8081  # ค่าเริ่มต้นคือ 0 = ให้ OS สุ่มพอร์ตว่างให้อัตโนมัติ (แนะนำเสมอ)
```

**คำแนะนำระดับมืออาชีพ**: อย่ากำหนด `port` ตายตัวถ้าไม่จำเป็นจริง ๆ เพราะถ้า
รัน test หลายชุดพร้อมกัน (parallel testing ที่จะเรียนใน Part 065) พอร์ตที่
ตายตัวจะชนกันทันที ปล่อยให้เป็นค่าเริ่มต้น (`0` = สุ่มพอร์ตว่าง) ปลอดภัยที่สุด

### 632.4 `LiveServerTestCase` สืบทอดจาก `TransactionTestCase` ไม่ใช่ `TestCase`

จุดที่มือใหม่มักพลาดและทำให้ test ทำงานช้าโดยไม่รู้ตัว: `LiveServerTestCase`
สืบทอดจาก **`TransactionTestCase`** (ทบทวนความแตกต่างจาก Part 060) ไม่ใช่
`TestCase` ธรรมดา เหตุผลคือเธรดของ live server ทำงานคนละเธรดกับ test method
เอง แต่ transaction ของ `TestCase` (ที่ wrap ทุก test ด้วย `SAVEPOINT` แล้ว
`ROLLBACK`) ผูกกับ**เธรดเดียว** เท่านั้น ถ้า live server (อีกเธรดหนึ่ง) พยายาม
query ข้อมูลที่ถูกสร้างในเธรดของ test แต่ยัง**ไม่ commit จริง** มันจะมองไม่เห็น
ข้อมูลนั้นเลย

| ประเด็น | TestCase (Part 060) | TransactionTestCase / LiveServerTestCase |
|---|---|---|
| กลไกล้างข้อมูลระหว่างเทส | ROLLBACK ธุรกรรม (เร็วมาก) | TRUNCATE ตารางทั้งหมดจริง (ช้ากว่า) |
| มองเห็นข้อมูลข้ามเธรดได้ไหม | ไม่ได้ (ข้อมูลอยู่ใน transaction ที่ยังไม่ commit) | ได้ (ข้อมูล commit จริงลงฐานข้อมูล) |
| ความเร็วโดยรวม | เร็ว | ช้ากว่าอย่างมีนัยสำคัญ |
| จำเป็นสำหรับ | Unit/Integration Test ทั่วไป | ทุก test ที่ต้องมี "เซิร์ฟเวอร์จริง" ให้เธรดอื่นเชื่อมต่อ |

นี่คือเหตุผลเชิงสถาปัตยกรรมว่าทำไม E2E test ถึง**ช้ากว่า** unit test เสมอ
ตามตารางในขั้นตอนที่ 631 — ไม่ใช่แค่เพราะต้องเปิดเบราว์เซอร์ แต่เพราะกลไก
จัดการฐานข้อมูลเบื้องหลังก็หนักกว่าโดยธรรมชาติด้วย

---

## ขั้นตอนที่ 633: ติดตั้งและตั้งค่า Selenium กับ Django

### 633.1 Selenium คืออะไร

**Selenium WebDriver** คือเครื่องมือควบคุมเบราว์เซอร์จริง (Chrome, Firefox,
Edge, Safari) ผ่านโค้ดโปรแกรม เริ่มพัฒนาตั้งแต่ปี 2004 เป็นมาตรฐานอุตสาหกรรม
ที่เก่าแก่และได้รับการยอมรับมากที่สุดสำหรับ browser automation โดยสื่อสารกับ
เบราว์เซอร์ผ่านโพรโทคอลมาตรฐาน **W3C WebDriver Protocol**

### 633.2 ติดตั้ง Package ที่จำเป็น

```bash
# เพิ่มไฟล์แยกสำหรับ dependency เฉพาะการเทส (แนวทางจาก Part 061)
pip install selenium webdriver-manager
pip freeze | grep -iE "selenium|webdriver-manager" >> requirements-test.txt
```

```
# requirements-test.txt (ต่อยอดจาก Part 061-063)
pytest==8.3.3
pytest-django==4.9.0
pytest-cov==5.0.0
factory-boy==3.3.1
freezegun==1.5.1
selenium==4.24.0
webdriver-manager==4.0.2
```

### 633.3 ทำไมต้อง `webdriver-manager`

ก่อนหน้านี้ การใช้ Selenium ต้องดาวน์โหลดไฟล์ driver ของแต่ละเบราว์เซอร์เอง
(เช่น `chromedriver`) ให้ตรงกับเวอร์ชันเบราว์เซอร์บนเครื่องเป๊ะ ๆ แล้ววาง path
เอง ซึ่งเป็นจุดปวดหัวคลาสสิกเวลาเบราว์เซอร์อัปเดตอัตโนมัติแต่ driver ไม่ตรง
เวอร์ชันอีกต่อไป **`webdriver-manager`** แก้ปัญหานี้โดยตรวจสอบเวอร์ชัน Chrome
ที่ติดตั้งอยู่บนเครื่องอัตโนมัติ แล้วดาวน์โหลด driver เวอร์ชันที่ตรงกันมาแคชไว้
ให้เอง โดยไม่ต้องจัดการ path ด้วยมือเลย

> **หมายเหตุสำหรับความถูกต้อง**: ตั้งแต่ Selenium 4.6 เป็นต้นมา ตัว Selenium
> เองก็มีกลไกภายในชื่อ **Selenium Manager** ที่จัดการดาวน์โหลด driver ให้
> อัตโนมัติได้เช่นกันโดยไม่ต้องติดตั้งอะไรเพิ่ม (แค่ `webdriver.Chrome()`
> เปล่า ๆ ก็ทำงานได้) หลักสูตรนี้ยังคงสอน `webdriver-manager` เพราะทีมงาน
> จำนวนมากในอุตสาหกรรมยังใช้อยู่จริง ให้การควบคุมเวอร์ชัน driver ที่ชัดเจนกว่า
> (ล็อกเวอร์ชันได้ผ่าน `ChromeDriverManager(driver_version="...")`) และมี
> cache แยกที่ตรวจสอบ/ลบง่ายกว่าเมื่อ debug ปัญหาใน CI (ขั้นตอนที่ 638)

### 633.4 สร้าง Base Test Class สำหรับ Selenium

รวมทุกอย่างที่ test ทุกตัวในหลักสูตรนี้ต้องใช้ซ้ำ ๆ ไว้ที่คลาสฐานเดียว
(หลักการ DRY ที่ยึดมาตั้งแต่ Part 001):

```python
# blog/tests/selenium_base.py
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.chrome.service import Service as ChromeService
from webdriver_manager.chrome import ChromeDriverManager


class SeleniumTestCase(StaticLiveServerTestCase):
    """
    คลาสฐานสำหรับ E2E test ทุกตัวในโปรเจกต์ — ใช้ StaticLiveServerTestCase
    (ไม่ใช่ LiveServerTestCase เฉย ๆ) เพราะหน้าเว็บของเราต้องโหลดไฟล์ static
    จริง (HTMX, Alpine.js, CSS) ผ่านเบราว์เซอร์ ซึ่ง LiveServerTestCase
    ธรรมดาไม่ serve static file ให้ในโหมด production-like
    """

    @classmethod
    def setUpClass(cls):
        super().setUpClass()

        options = ChromeOptions()
        options.add_argument("--headless=new")       # ขั้นตอนที่ 638 อธิบายเพิ่ม
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-dev-shm-usage")
        options.add_argument("--disable-gpu")
        options.add_argument("--window-size=1920,1080")

        service = ChromeService(ChromeDriverManager().install())
        cls.selenium = webdriver.Chrome(service=service, options=options)

        # ปิด implicit wait ไว้ที่ 0 เสมอ — หลักสูตรนี้ใช้ "explicit wait"
        # (WebDriverWait) เท่านั้น เพราะการผสม implicit + explicit wait
        # ทำให้เวลารอรวมคาดเดาไม่ได้ และเป็นสาเหตุคลาสสิกของ flaky test
        # (รายละเอียดเต็มในขั้นตอนที่ 639)
        cls.selenium.implicitly_wait(0)

    @classmethod
    def tearDownClass(cls):
        cls.selenium.quit()
        super().tearDownClass()
```

### 633.5 ทดสอบว่าโครงสร้างพื้นฐานทำงาน: Smoke Test แรก

```python
# blog/tests/test_selenium_smoke.py
from selenium.webdriver.common.by import By
from .selenium_base import SeleniumTestCase


class SeleniumSmokeTests(SeleniumTestCase):
    def test_homepage_loads_in_real_browser(self):
        self.selenium.get(self.live_server_url)

        self.assertIn("Django Mastery Blog", self.selenium.title)
        # ตรวจว่า Chrome จริง ๆ render <body> ออกมาได้ (ไม่ใช่หน้าเปล่า/error)
        body = self.selenium.find_element(By.TAG_NAME, "body")
        self.assertTrue(body.is_displayed())
```

```bash
python manage.py test blog.tests.test_selenium_smoke
```

ครั้งแรกที่รัน `webdriver-manager` จะดาวน์โหลด ChromeDriver มาแคชไว้ที่
`~/.wdm/` (ใช้เวลาสักครู่) รันครั้งถัดไปจะเร็วขึ้นมากเพราะใช้ของที่แคชไว้แล้ว
ถ้าเห็น error ประเภท `session not created: This version of ChromeDriver
only supports Chrome version X` แปลว่า Chrome บนเครื่องอัปเดตแต่แคชเก่าค้าง
อยู่ แก้ได้ด้วยการลบโฟลเดอร์ `~/.wdm/` แล้วรันใหม่

### 633.6 ตรวจสอบว่ามี Google Chrome ติดตั้งอยู่บนเครื่องหรือไม่

Selenium ควบคุมเบราว์เซอร์ที่ติดตั้งอยู่จริง ไม่ได้มาพร้อมเบราว์เซอร์ในตัว —
ต้องมี Chrome (หรือ Chromium) อยู่บนเครื่องก่อนเสมอ:

```bash
# macOS
brew install --cask google-chrome

# Ubuntu/Debian
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list'
sudo apt update && sudo apt install google-chrome-stable -y

# ตรวจสอบเวอร์ชัน
google-chrome --version
```

---

## ขั้นตอนที่ 634: เขียน Browser Automation Test จริง

### 634.1 เตรียม View และ Template ที่จะทดสอบตลอด Part นี้

ต่อยอดจาก Model `Post` (Part 026, 053, 054) เพิ่ม View สำหรับสร้างโพสต์ผ่าน
ฟอร์มจริง ซึ่งจะเป็นแกนหลักของ E2E test ตลอดทั้ง Part นี้:

```python
# blog/forms.py
from django import forms
from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "content"]
        widgets = {
            "content": forms.Textarea(attrs={"rows": 6}),
        }
```

```python
# blog/views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404, redirect, render
from django.utils.text import slugify
from .forms import PostForm
from .models import Post


def post_list_view(request):
    posts = Post.objects.filter(is_published=True).select_related("author")
    return render(request, "blog/post_list.html", {"posts": posts})


def post_detail_view(request, slug):
    post = get_object_or_404(Post, slug=slug, is_published=True)
    return render(request, "blog/post_detail.html", {"post": post})


@login_required
def post_create_view(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.slug = slugify(post.title)
            post.is_published = True
            post.save()
            return redirect("blog:post_list")
    else:
        form = PostForm()

    return render(request, "blog/post_form.html", {"form": form})
```

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"

urlpatterns = [
    path("", views.post_list_view, name="post_list"),
    path("new/", views.post_create_view, name="post_create"),
    path("<slug:slug>/", views.post_detail_view, name="post_detail"),
]
```

```html
<!-- blog/templates/blog/post_form.html -->
{% extends "base.html" %}
{% block content %}
<h1>เขียนโพสต์ใหม่</h1>
<form method="post" id="post-form">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">เผยแพร่โพสต์</button>
</form>
{% endblock %}
```

```html
<!-- blog/templates/blog/post_list.html -->
{% extends "base.html" %}
{% block content %}
<h1>บทความทั้งหมด</h1>
<a href="{% url 'blog:post_create' %}">เขียนโพสต์ใหม่</a>
<ul id="post-list">
    {% for post in posts %}
        <li class="post-item">
            <a href="{% url 'blog:post_detail' post.slug %}">{{ post.title }}</a>
        </li>
    {% empty %}
        <li>ยังไม่มีบทความ</li>
    {% endfor %}
</ul>
{% endblock %}
```

### 634.2 ตัวชี้วัดพื้นฐาน: `find_element`, `By`, และ `send_keys`

```python
from selenium.webdriver.common.by import By

# ตัวอย่างวิธีค้นหา element แบบต่าง ๆ — เรียงจากที่แนะนำมากไปน้อย
element = self.selenium.find_element(By.ID, "id_title")            # เร็วสุด ชัดเจนสุด
element = self.selenium.find_element(By.NAME, "title")             # ดีเมื่อไม่มี id ตายตัว
element = self.selenium.find_element(By.CSS_SELECTOR, "#post-form button")
element = self.selenium.find_element(By.LINK_TEXT, "เขียนโพสต์ใหม่")
element = self.selenium.find_element(By.XPATH, "//button[contains(text(),'เผยแพร่')]")
```

### 634.3 E2E Test แรก: Login ผ่านฟอร์มจริง

ใช้ `LoginView` มาตรฐานของ Django (Part 031) — field ที่ `AuthenticationForm`
render ออกมามี `id="id_username"` และ `id="id_password"` เสมอตามค่าเริ่มต้น:

```python
# blog/tests/test_e2e_login.py
from django.contrib.auth import get_user_model
from django.urls import reverse
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from .selenium_base import SeleniumTestCase

User = get_user_model()


class LoginE2ETests(SeleniumTestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username="namwan", password="StrongPass123!"
        )

    def test_user_can_login_via_real_browser(self):
        self.selenium.get(f"{self.live_server_url}{reverse('accounts:login')}")

        username_input = self.selenium.find_element(By.ID, "id_username")
        password_input = self.selenium.find_element(By.ID, "id_password")

        username_input.send_keys("namwan")
        password_input.send_keys("StrongPass123!")
        password_input.send_keys(Keys.RETURN)  # จำลองการกด Enter เพื่อ submit ฟอร์ม

        # ต้องรอให้เบราว์เซอร์ redirect เสร็จก่อนตรวจสอบผลลัพธ์เสมอ
        # (อธิบาย WebDriverWait อย่างละเอียดในข้อ 634.5)
        WebDriverWait(self.selenium, 10).until(
            EC.url_contains("/blog/")
        )

        self.assertIn("namwan", self.selenium.page_source)

    def test_login_with_wrong_password_shows_error(self):
        self.selenium.get(f"{self.live_server_url}{reverse('accounts:login')}")

        self.selenium.find_element(By.ID, "id_username").send_keys("namwan")
        password_input = self.selenium.find_element(By.ID, "id_password")
        password_input.send_keys("รหัสผิดแน่นอน")
        password_input.send_keys(Keys.RETURN)

        error_message = WebDriverWait(self.selenium, 10).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, ".errorlist"))
        )
        self.assertIn("Please enter a correct", error_message.text)
```

### 634.4 E2E Test ที่สอง: กรอกฟอร์มสร้างโพสต์

```python
# blog/tests/test_e2e_post_create.py
from django.contrib.auth import get_user_model
from django.urls import reverse
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from blog.models import Post
from .selenium_base import SeleniumTestCase

User = get_user_model()


class PostCreateE2ETests(SeleniumTestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username="namwan", password="StrongPass123!"
        )

    def _login_via_browser(self):
        self.selenium.get(f"{self.live_server_url}{reverse('accounts:login')}")
        self.selenium.find_element(By.ID, "id_username").send_keys("namwan")
        self.selenium.find_element(By.ID, "id_password").send_keys("StrongPass123!")
        self.selenium.find_element(By.CSS_SELECTOR, "button[type=submit]").click()
        WebDriverWait(self.selenium, 10).until(EC.url_contains("/blog/"))

    def test_authenticated_user_can_create_post(self):
        self._login_via_browser()

        self.selenium.get(f"{self.live_server_url}{reverse('blog:post_create')}")

        title_input = self.selenium.find_element(By.ID, "id_title")
        content_input = self.selenium.find_element(By.ID, "id_content")

        title_input.send_keys("โพสต์แรกที่สร้างผ่าน Selenium")
        content_input.send_keys("เนื้อหานี้ถูกพิมพ์ผ่านเบราว์เซอร์จริงทั้งหมด")

        self.selenium.find_element(By.CSS_SELECTOR, "#post-form button[type=submit]").click()

        WebDriverWait(self.selenium, 10).until(
            EC.url_matches(rf"{self.live_server_url}/blog/$")
        )

        # ตรวจทั้งสองระดับ: (1) DOM ที่เบราว์เซอร์เห็นจริง (2) ฐานข้อมูลจริง
        self.assertIn("โพสต์แรกที่สร้างผ่าน Selenium", self.selenium.page_source)
        self.assertTrue(
            Post.objects.filter(title="โพสต์แรกที่สร้างผ่าน Selenium").exists()
        )
```

สังเกตว่า test นี้ตรวจสอบผลลัพธ์ **สองชั้น**: (1) ฝั่ง UI ผ่าน
`self.selenium.page_source` ว่าผู้ใช้เห็นโพสต์ใหม่จริง และ (2) ฝั่งฐานข้อมูล
ผ่าน Django ORM ตรง ๆ ว่าข้อมูลถูกบันทึกจริง — การตรวจทั้งสองชั้นนี้คือหัวใจ
ของ E2E test ที่ดี เพราะบางครั้ง UI แสดงผลถูกแต่ข้อมูลไม่ได้ถูกบันทึกจริง
(หรือกลับกัน) และ unit test อย่างเดียวจะจับปัญหาแบบนี้ไม่ได้เลย

### 634.5 `WebDriverWait` และ `expected_conditions`: หัวใจของ Selenium ที่เชื่อถือได้

ปัญหาที่พบบ่อยที่สุดของมือใหม่ Selenium คือ **race condition**: โค้ดพยายาม
`find_element` element ที่ยังไม่ปรากฏบนหน้าจอ (เพราะเบราว์เซอร์ยัง render
ไม่เสร็จ หรือ redirect ยังไม่เกิดขึ้น) แล้วได้ `NoSuchElementException` ทั้งที่
element นั้นจะปรากฏจริงในอีกไม่กี่ร้อยมิลลิวินาทีถัดมา `WebDriverWait` แก้
ปัญหานี้โดย **poll ซ้ำ ๆ จนกว่าเงื่อนไขจะเป็นจริง หรือหมดเวลาที่กำหนด**:

| `expected_conditions` (มักย่อว่า `EC`) | ใช้เมื่อไหร่ |
|---|---|
| `presence_of_element_located((By.ID, "x"))` | element อยู่ใน DOM แล้ว (ไม่สนใจว่าเห็นบนจอหรือไม่) |
| `visibility_of_element_located((By.ID, "x"))` | element อยู่ใน DOM **และ**มองเห็นได้ (ไม่ถูก `display:none`) |
| `element_to_be_clickable((By.ID, "x"))` | element พร้อมให้คลิกจริง (มองเห็นได้และไม่ถูก disable) |
| `url_contains("/blog/")` | รอจน URL ปัจจุบันมีคำนี้อยู่ (ใช้ตรวจ redirect) |
| `url_matches(pattern)` | รอจน URL ตรงกับ regex ทั้งหมด (เข้มงวดกว่า `url_contains`) |
| `text_to_be_present_in_element((By.ID, "x"), "คำ")` | รอจนข้อความปรากฏใน element (มีประโยชน์มากกับ HTMX — ขั้นตอนที่ 636) |
| `staleness_of(element)` | รอจน element เดิมถูกลบออกจาก DOM (มีประโยชน์หลัง submit ฟอร์มที่ re-render หน้า) |

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(self.selenium, timeout=10, poll_frequency=0.2)
button = wait.until(EC.element_to_be_clickable((By.ID, "submit-btn")))
button.click()
```

**กฎเหล็กของหลักสูตรนี้: ห้ามใช้ `time.sleep()` ในการรอให้หน้าเว็บพร้อมเด็ดขาด**
`time.sleep(2)` ทั้งช้าเกินความจำเป็น (ถ้าหน้าโหลดเสร็จใน 200ms ก็ยังรอครบ 2
วินาทีอยู่ดี) และเร็วเกินไปในบางครั้ง (ถ้าเครื่อง CI ช้ากว่าปกติ 2 วินาทีอาจ
ไม่พอ) ในขณะที่ `WebDriverWait` รอ**พอดี**เท่าที่จำเป็นเสมอ และมีเพดานเวลา
สูงสุดกันไม่ให้ test ค้างตลอดไปถ้ามีบางอย่างพังจริง ๆ — รายละเอียดเพิ่มเติมใน
ขั้นตอนที่ 639 เรื่อง Flaky Test

---

## ขั้นตอนที่ 635: Playwright เป็นทางเลือกสมัยใหม่แทน Selenium

### 635.1 Playwright คืออะไร

**Playwright** เป็นเครื่องมือ browser automation ที่พัฒนาโดยทีมงาน Microsoft
(ทีมเดียวกับที่เคยสร้าง Puppeteer ที่ Google มาก่อน) เปิดตัวปี 2020 ออกแบบมา
เพื่อแก้ปัญหาคลาสสิกของ Selenium โดยตรง โดยเฉพาะเรื่อง **auto-waiting**

### 635.2 จุดต่างที่สำคัญที่สุด: Auto-waiting

Selenium ต้องเขียน `WebDriverWait` + `expected_conditions` เองทุกครั้งที่
ต้องการความเชื่อถือได้ (ตามขั้นตอนที่ 634.5) มิเช่นนั้นจะเจอ flaky test ทันที
— **Playwright ทำสิ่งนี้ให้อัตโนมัติในทุก action** ก่อนคลิกหรือพิมพ์ลงช่องใด
Playwright จะรอเองจนกว่า element นั้นจะ **visible, enabled, และ stable**
(ไม่มี animation กำลังเล่นอยู่) โดยไม่ต้องเขียนโค้ดรอเพิ่มเลย

### 635.3 ตารางเปรียบเทียบ Selenium vs Playwright

| คุณสมบัติ | Selenium | Playwright |
|---|---|---|
| ปีที่เปิดตัว | 2004 | 2020 |
| Auto-waiting ในตัว | ❌ ต้องเขียน `WebDriverWait` เอง | ✅ ทุก action รอให้อัตโนมัติ |
| ความเร็วโดยรวม | ปานกลาง (WebDriver Protocol ผ่าน HTTP) | เร็วกว่า (สื่อสารผ่าน WebSocket โดยตรง) |
| รองรับหลายเบราว์เซอร์ | Chrome, Firefox, Edge, Safari (ผ่าน driver แยกแต่ละตัว) | Chromium, Firefox, WebKit ในไลบรารีเดียว |
| ติดตั้งเบราว์เซอร์ | ต้องติดตั้ง Chrome เองบนเครื่อง | มีคำสั่ง `playwright install` ดาวน์โหลดเบราว์เซอร์ (browser binary) ให้เองครบ |
| Network interception | ทำได้แต่ setup ยุ่งยากกว่า | ทำได้ง่ายมากในตัว (`page.route()`) |
| Trace/Video/Screenshot อัตโนมัติ | ต้องเขียนเอง (ขั้นตอนที่ 637) | มีธงคำสั่งสำเร็จรูป (`--screenshot`, `--video`, `--trace`) |
| Codegen (บันทึกการคลิกแล้วสร้างโค้ดให้) | ไม่มีในตัว (มี Selenium IDE แยกต่างหาก) | มี `playwright codegen` ในตัว |
| ความนิยมในทีมเก่า/legacy | สูงมาก (มาตรฐานมานาน) | กำลังเติบโตเร็วมากในโปรเจกต์ใหม่ (2022-2026) |
| การรองรับใน Django ecosystem | ใช้ร่วมกับ `LiveServerTestCase` ได้ตรง ๆ | ใช้ผ่าน `pytest-playwright` + `pytest-django` |

**จุดยืนของหลักสูตรนี้**: ทั้งสองเครื่องมือใช้งานจริงในอุตสาหกรรม Selenium
ยังคงเป็นมาตรฐานที่พบได้ในโปรเจกต์เก่าจำนวนมากและมีเอกสาร/ชุมชนใหญ่ที่สุด
ส่วน Playwright คือตัวเลือกที่แนะนำสำหรับ**โปรเจกต์ใหม่**เพราะ auto-waiting
ลดโอกาสเกิด flaky test ได้มากในทางปฏิบัติ และ API อ่านง่ายกว่าอย่างชัดเจน
หลักสูตรนี้สอนทั้งสองตัวเพื่อให้คุณอ่านโค้ด E2E test ได้ไม่ว่าจะเจอในทีมไหน

### 635.4 ติดตั้ง Playwright สำหรับ Django

```bash
pip install pytest-playwright pytest-django
playwright install chromium --with-deps
```

`playwright install chromium` ดาวน์โหลด browser binary ของ Chromium มาเก็บ
ไว้ในเครื่อง (แยกจาก Chrome ที่ติดตั้งผ่านระบบปฏิบัติการปกติ) ธง `--with-deps`
ติดตั้ง system library ที่จำเป็นให้ครบด้วย (สำคัญมากบน CI ที่เป็น Linux image
เปล่า ๆ — รายละเอียดในขั้นตอนที่ 638)

ตั้งค่า `pytest.ini` ต่อยอดจาก Part 061:

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py *_tests.py
```

### 635.5 เขียน Test เดิมจากขั้นตอนที่ 634.4 ด้วย Playwright

```python
# blog/tests/test_playwright_post_create.py
import re
import pytest
from blog.models import Post


@pytest.mark.django_db
def test_authenticated_user_can_create_post(live_server, page, django_user_model):
    django_user_model.objects.create_user(
        username="namwan", password="StrongPass123!"
    )

    # --- ล็อกอินผ่านฟอร์มจริง ---
    page.goto(f"{live_server.url}/accounts/login/")
    page.fill("#id_username", "namwan")
    page.fill("#id_password", "StrongPass123!")
    page.click("button[type=submit]")
    page.wait_for_url(re.compile(r".*/blog/.*"))

    # --- สร้างโพสต์ผ่านฟอร์มจริง ---
    page.goto(f"{live_server.url}/blog/new/")
    page.fill("#id_title", "โพสต์แรกที่สร้างผ่าน Playwright")
    page.fill("#id_content", "เนื้อหานี้ถูกพิมพ์ผ่าน Playwright ทั้งหมด")
    page.click("#post-form button[type=submit]")

    # ไม่ต้องเขียน explicit wait เอง — click() ของ Playwright รอ element
    # พร้อมคลิกให้อัตโนมัติอยู่แล้ว ส่วน expect() ด้านล่างจะ auto-retry
    # จนกว่าเงื่อนไขเป็นจริงหรือหมดเวลาเอง (ค่าเริ่มต้น 5 วินาที)
    from playwright.sync_api import expect

    expect(page.locator("#post-list")).to_contain_text(
        "โพสต์แรกที่สร้างผ่าน Playwright"
    )
    assert Post.objects.filter(title="โพสต์แรกที่สร้างผ่าน Playwright").exists()
```

รันด้วย:

```bash
pytest blog/tests/test_playwright_post_create.py -v
# ดูเบราว์เซอร์จริงตอนรัน (มีประโยชน์มากตอน debug):
pytest blog/tests/test_playwright_post_create.py --headed --slowmo 500
```

สังเกตว่าไม่มีการเขียน `WebDriverWait`/`expected_conditions` เลยแม้แต่จุด
เดียวในโค้ด Playwright — `page.fill()`, `page.click()` และ `expect(...)`
ทุกตัวมี auto-waiting ในตัวเองอยู่แล้วตามที่อธิบายในข้อ 635.2 นี่คือเหตุผล
หลักที่โค้ด Playwright มักสั้นกว่าและอ่านง่ายกว่า Selenium อย่างชัดเจนเมื่อ
เทียบ test ที่ทำสิ่งเดียวกันทุกประการ

### 635.6 `live_server` Fixture ของ `pytest-django` เทียบกับ `LiveServerTestCase`

`pytest-django` มี fixture ชื่อ `live_server` ที่ทำหน้าที่เหมือนกับ
`LiveServerTestCase` ทุกประการแต่อยู่ในรูปแบบ fixture ของ pytest (ทบทวน
แนวคิด fixture จาก Part 061) — ข้อดีคือประกอบร่วมกับ fixture อื่นได้อิสระกว่า
เช่น `django_user_model`, `db`, หรือ fixture ที่สร้างเองจาก `factory_boy`
(Part 063) ได้ในบรรทัดพารามิเตอร์เดียว โดยไม่ต้องสืบทอด class ใด ๆ เลย

---

## ขั้นตอนที่ 636: Testing หน้าที่มี JavaScript หนัก (HTMX/Alpine.js)

### 636.1 ทำไม `django.test.Client` ทดสอบพฤติกรรมเหล่านี้ไม่ได้เลย

ทบทวนจาก Part 053 (HTMX) และ Part 054 (Alpine.js): ปุ่มถูกใจที่อัปเดตจำนวน
โดยไม่ reload หน้า (`hx-post` + `hx-swap="outerHTML"`) และ Modal ที่เปิด/ปิด
ด้วย `x-show`/`x-data` **ทำงานอยู่ในเบราว์เซอร์ทั้งหมด** — `django.test.Client`
ยิง HTTP request ไปที่ View แล้วได้ HTML กลับมาเป็น string ก้อนเดียวเท่านั้น
มันไม่มี concept ของ "DOM ที่มีการเปลี่ยนแปลงหลัง JavaScript รัน" เลย

```python
# สิ่งที่ Client ทำได้: ตรวจ HTML ก้อนแรกที่ View คืนมา (Part 060-061)
response = self.client.get("/blog/post-1/")
self.assertContains(response, 'hx-post="/blog/post-1/like/"')  # ✅ ตรวจ attribute ได้

# สิ่งที่ Client ทำไม่ได้เลย: จำลองการ "คลิก" แล้วดูว่า DOM เปลี่ยนจริงไหม
# ไม่มีวิธีใดใน django.test.Client ที่จะบอกว่า "หลังคลิกปุ่มนี้ ตัวเลขบนปุ่ม
# เปลี่ยนจาก 0 เป็น 1 โดยไม่มีการ reload หน้าเกิดขึ้น" — เพราะไม่มี JavaScript
# engine ทำงานอยู่เบื้องหลัง Client เลย
```

นี่คือช่องว่างที่ E2E test เข้ามาเติมเต็มพอดี: มันควบคุมเบราว์เซอร์**จริง**
ที่มี JavaScript engine ครบ (V8 ของ Chrome) ทำให้ HTMX และ Alpine.js ทำงาน
ได้จริงเหมือนที่ผู้ใช้เจอ

### 636.2 ทดสอบปุ่มถูกใจแบบ HTMX (ทบทวนจาก Part 053 ข้อ 522.4)

```python
# blog/tests/test_e2e_htmx_like.py
from django.contrib.auth import get_user_model
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from blog.models import Post
from .selenium_base import SeleniumTestCase

User = get_user_model()


class HtmxLikeButtonE2ETests(SeleniumTestCase):
    def setUp(self):
        self.author = User.objects.create_user(username="writer", password="x")
        self.post = Post.objects.create(
            title="โพสต์ทดสอบปุ่มถูกใจ",
            slug="test-like-button",
            content="เนื้อหา",
            author=self.author,
            is_published=True,
            like_count=0,
        )

    def test_clicking_like_button_updates_count_without_full_reload(self):
        self.selenium.get(f"{self.live_server_url}/blog/{self.post.slug}/")

        like_button = WebDriverWait(self.selenium, 10).until(
            EC.element_to_be_clickable((By.CSS_SELECTOR, ".like-button"))
        )
        self.assertIn("(0)", like_button.text)

        # เก็บ reference ของ <html> ไว้ก่อนคลิก เพื่อพิสูจน์ว่าไม่มี full
        # page reload เกิดขึ้นจริง (ถ้า reload จริง element เดิมจะ "stale")
        html_element_before_click = self.selenium.find_element(By.TAG_NAME, "html")

        like_button.click()

        # รอจนข้อความบนปุ่มเปลี่ยนเป็น "(1)" — HTMX แทนที่ปุ่มด้วย
        # hx-swap="outerHTML" ดังนั้นต้องรอ "หา" ปุ่มใหม่อีกครั้ง ไม่ใช้
        # reference เดิมที่อาจกลายเป็น stale element แล้ว
        WebDriverWait(self.selenium, 10).until(
            EC.text_to_be_present_in_element((By.CSS_SELECTOR, ".like-button"), "(1)")
        )

        # พิสูจน์ว่า <html> element เดิมยังอยู่ (ไม่มีการโหลดหน้าใหม่ทั้งหน้า)
        html_element_after_click = self.selenium.find_element(By.TAG_NAME, "html")
        self.assertEqual(
            html_element_before_click.id, html_element_after_click.id,
            "คาดว่า element <html> เดิมควรยังอยู่ ถ้ามี full page reload "
            "เกิดขึ้น Selenium จะได้ element ใหม่ที่มี internal id ต่างไป",
        )

        self.post.refresh_from_db()
        self.assertEqual(self.post.like_count, 1)
```

`EC.text_to_be_present_in_element` คือกุญแจสำคัญของการทดสอบ HTMX: มันรอ
**เนื้อหาข้อความ**เปลี่ยนแปลง โดยไม่สนใจว่า element เดิมจะถูกแทนที่ทั้งก้อน
ด้วย `outerHTML` หรือไม่ (ต่างจากการเก็บ reference element ไว้ก่อนแล้วเช็ค
`.text` ทีหลัง ซึ่งจะโยน `StaleElementReferenceException` ทันทีถ้า HTMX
swap element นั้นทิ้งไปแล้วสร้างใหม่)

### 636.3 ทดสอบ Modal แบบ Alpine.js (ทบทวนจาก Part 054 ข้อ 535.2)

Modal ของ Alpine.js ใช้ `x-show` ซึ่งควบคุมด้วย CSS `display: none` เท่านั้น
(ทบทวนข้อ 532.4) — element **ยังอยู่ใน DOM เสมอ** เพียงแต่มองไม่เห็น
Selenium มีเมธอด `is_displayed()` ที่ตรวจสอบสิ่งนี้ได้ตรง ๆ:

```python
# blog/tests/test_e2e_alpine_modal.py
from django.contrib.auth import get_user_model
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from blog.models import Post
from .selenium_base import SeleniumTestCase

User = get_user_model()


class AlpineModalE2ETests(SeleniumTestCase):
    def setUp(self):
        self.author = User.objects.create_user(
            username="writer", password="x", is_staff=True
        )
        self.post = Post.objects.create(
            title="โพสต์ที่จะถูกลบ",
            slug="post-to-delete",
            content="เนื้อหา",
            author=self.author,
            is_published=True,
        )

    def _login_as_staff(self):
        self.selenium.get(f"{self.live_server_url}/accounts/login/")
        self.selenium.find_element(By.ID, "id_username").send_keys("writer")
        self.selenium.find_element(By.ID, "id_password").send_keys("x")
        self.selenium.find_element(By.CSS_SELECTOR, "button[type=submit]").click()
        WebDriverWait(self.selenium, 10).until(EC.url_contains("/blog/"))

    def test_modal_is_hidden_by_default_then_shown_on_click(self):
        self._login_as_staff()
        self.selenium.get(f"{self.live_server_url}/blog/{self.post.slug}/")

        modal_box = self.selenium.find_element(By.CSS_SELECTOR, ".modal-box")
        # ตอนโหลดหน้าแรก x-data เริ่มต้นที่ modalOpen: false → มองไม่เห็น
        # (แต่ยังอยู่ใน DOM จริง — สำคัญมากสำหรับ SEO ตามที่อธิบายใน Part 054)
        self.assertFalse(modal_box.is_displayed())

        open_button = self.selenium.find_element(
            By.CSS_SELECTOR, "[data-testid='delete-post-trigger']"
        )
        open_button.click()

        WebDriverWait(self.selenium, 10).until(
            EC.visibility_of(modal_box)
        )
        self.assertTrue(modal_box.is_displayed())

        confirm_button = self.selenium.find_element(
            By.CSS_SELECTOR, "[data-testid='confirm-delete']"
        )
        confirm_button.click()

        WebDriverWait(self.selenium, 10).until(EC.url_contains("/blog/"))
        self.assertFalse(Post.objects.filter(pk=self.post.pk).exists())
```

**เทคนิคสำคัญ**: attribute `data-testid` ในตัวอย่างข้างต้นถูกเพิ่มเข้าไปใน
template โดยเฉพาะสำหรับ test (ไม่มีผลต่อ CSS/JavaScript ใด ๆ) นี่คือ pattern
มาตรฐานของอุตสาหกรรมที่เรียกว่า **"test selector attribute"** — แยก selector
ที่ test ใช้ออกจาก class ที่ CSS ใช้ (`.modal-box`, `.btn-danger`) เพื่อไม่ให้
test พังทุกครั้งที่นักออกแบบเปลี่ยนชื่อ CSS class (เหตุผลเดียวกับที่ตาราง
ในขั้นตอนที่ 631.3 ระบุว่า E2E test "กระทบมากเมื่อ UI เปลี่ยน" — การใช้
`data-testid` ช่วยลดผลกระทบนั้นได้มาก):

```html
<button @click="modalOpen = true" data-testid="delete-post-trigger" class="btn btn-danger">
    ลบโพสต์
</button>
```

### 636.4 ตารางสรุป: อะไรทดสอบได้ด้วยเครื่องมือไหน

| สิ่งที่ต้องการทดสอบ | `django.test.Client` (Part 060-061) | Selenium/Playwright (Part นี้) |
|---|---|---|
| View คืน HTTP status/HTML ที่ถูกต้อง | ✅ เหมาะสมที่สุด (เร็ว) | ทำได้แต่ช้าเกินความจำเป็น |
| `ModelForm` validate ข้อมูลถูกต้อง | ✅ เหมาะสมที่สุด | ไม่จำเป็น |
| ปุ่ม HTMX คลิกแล้ว DOM เปลี่ยนจริงในเบราว์เซอร์ | ❌ ทำไม่ได้เลย | ✅ จำเป็นต้องใช้ |
| Alpine.js `x-show` ซ่อน/แสดง element ถูกต้อง | ❌ ทำไม่ได้เลย | ✅ จำเป็นต้องใช้ |
| CSS ทำให้ปุ่มถูกซ่อนโดยไม่ตั้งใจ (overlap, z-index) | ❌ มองไม่เห็น CSS เลย | ✅ เห็นจริงตามที่ render |
| การนำทางข้ามหลายหน้า (login → create → list) | ทำได้ แต่ต้องเขียน assertion เชื่อมกันเอง | ✅ จำลอง user journey ได้ตรงที่สุด |

---

## ขั้นตอนที่ 637: การถ่าย Screenshot อัตโนมัติเมื่อ Test ล้มเหลว

### 637.1 ทำไมเรื่องนี้สำคัญมากสำหรับ E2E Test โดยเฉพาะ

เมื่อ unit test ล้มเหลว ข้อความ error (`AssertionError: 5 != 3`) มักบอกสาเหตุ
ชัดเจนพอสมควร แต่เมื่อ E2E test ล้มเหลวด้วยข้อความอย่าง
`NoSuchElementException: Unable to locate element: .like-button` เราแทบไม่รู้
เลยว่า **ตอนนั้นหน้าเว็บหน้าตาเป็นอย่างไร** — อาจเป็นเพราะ CSS class เปลี่ยนชื่อ
จริง หรืออาจเพราะหน้าเพิ่งแสดง error 500 ที่ไม่มีปุ่มนั้นอยู่เลยตั้งแต่ต้น
Screenshot ที่ถ่ายไว้ ณ วินาทีที่ล้มเหลวคือเครื่องมือ debug ที่ทรงพลังที่สุด
สำหรับปัญหาประเภทนี้

### 637.2 วิธีที่ 1: `unittest`/`TestCase` — Hook เข้ากับ `tearDown`

```python
# blog/tests/selenium_base.py (เพิ่มเข้าไปจากขั้นตอนที่ 633.4)
import datetime
import os
from django.conf import settings
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.chrome.service import Service as ChromeService
from webdriver_manager.chrome import ChromeDriverManager

SCREENSHOT_DIR = os.path.join(settings.BASE_DIR, "test-screenshots")


class SeleniumTestCase(StaticLiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        options = ChromeOptions()
        options.add_argument("--headless=new")
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-dev-shm-usage")
        options.add_argument("--window-size=1920,1080")
        service = ChromeService(ChromeDriverManager().install())
        cls.selenium = webdriver.Chrome(service=service, options=options)
        cls.selenium.implicitly_wait(0)

    @classmethod
    def tearDownClass(cls):
        cls.selenium.quit()
        super().tearDownClass()

    def tearDown(self):
        if self._test_failed():
            self._save_failure_screenshot()
        super().tearDown()

    def _test_failed(self):
        # หมายเหตุ: ใช้ attribute ภายใน (_outcome) ของ unittest ซึ่งเป็น
        # วิธีที่ใช้กันแพร่หลายที่สุดในการรู้ผลลัพธ์ของเทสภายใน tearDown()
        # เอง แต่เป็น private API ที่อาจเปลี่ยนโครงสร้างในเวอร์ชัน Python
        # ใหม่ ๆ ได้ — ถ้าใช้ pytest ล้วน แนะนำวิธีที่ 2 (hook) แทนเสมอ
        outcome = getattr(self, "_outcome", None)
        if outcome is None:
            return False
        result = getattr(outcome, "result", outcome)
        errors = getattr(result, "errors", []) + getattr(result, "failures", [])
        return any(test is self for test, _ in errors)

    def _save_failure_screenshot(self):
        os.makedirs(SCREENSHOT_DIR, exist_ok=True)
        timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"{self.__class__.__name__}.{self._testMethodName}.{timestamp}.png"
        filepath = os.path.join(SCREENSHOT_DIR, filename)
        self.selenium.save_screenshot(filepath)
        print(f"\n[Selenium] บันทึก screenshot ของ test ที่ล้มเหลวไว้ที่: {filepath}")
```

เพิ่ม `test-screenshots/` เข้า `.gitignore` (ทบทวนหลักการจาก Part 001 ข้อ
8.5) เพราะไฟล์เหล่านี้เป็นผลลัพธ์การรัน test ในเครื่อง ไม่ควร commit เข้า Git:

```
# .gitignore (เพิ่มเติม)
test-screenshots/
```

### 637.3 วิธีที่ 2: `pytest` Hook — แม่นยำกว่าและไม่พึ่ง Private API

สำหรับ test ที่เขียนด้วย `pytest` ล้วน (รวมถึง Playwright ในขั้นตอนที่ 635)
pytest มี hook ทางการที่ออกแบบมาเพื่อสิ่งนี้โดยเฉพาะ ไม่ต้องแตะ internal
attribute ของ `unittest` เลย:

```python
# conftest.py
import datetime
import os
import pytest

SCREENSHOT_DIR = "test-screenshots"


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """เก็บผลลัพธ์ของแต่ละ phase (setup/call/teardown) ไว้ใน item เอง
    เพื่อให้ fixture อื่นตรวจสอบได้ทีหลังว่า test นี้ผ่านหรือล้มเหลว"""
    outcome = yield
    report = outcome.get_result()
    setattr(item, f"rep_{report.when}", report)


@pytest.fixture(autouse=True)
def _screenshot_on_failure(request, page):
    yield
    # request.node.rep_call ถูกตั้งค่าโดย hook ด้านบน หลังจากรัน test เสร็จ
    report = getattr(request.node, "rep_call", None)
    if report is not None and report.failed:
        os.makedirs(SCREENSHOT_DIR, exist_ok=True)
        timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"{request.node.name}.{timestamp}.png"
        page.screenshot(path=os.path.join(SCREENSHOT_DIR, filename))
        print(f"\n[Playwright] บันทึก screenshot ไว้ที่: {SCREENSHOT_DIR}/{filename}")
```

### 637.4 ทางลัดของ Playwright: ธงคำสั่งสำเร็จรูป

Playwright (ผ่าน `pytest-playwright`) มีความสามารถถ่าย screenshot, บันทึกวิดีโอ,
และเก็บ **trace** (บันทึกทุก action พร้อม DOM snapshot ที่เล่นย้อนดูได้ทีหลัง
ผ่าน Playwright Trace Viewer) ให้อัตโนมัติโดยไม่ต้องเขียน hook เองเลย
เพียงเพิ่มธงตอนรัน:

```bash
pytest --screenshot=only-on-failure --video=retain-on-failure --tracing=retain-on-failure
```

| ธงคำสั่ง | ผลลัพธ์ |
|---|---|
| `--screenshot=only-on-failure` | ถ่ายภาพหน้าจอ ณ วินาทีที่ล้มเหลวเท่านั้น (ประหยัดพื้นที่) |
| `--video=retain-on-failure` | บันทึกวิดีโอทั้ง test แต่**เก็บไว้เฉพาะ**เทสที่ล้มเหลว (ลบทิ้งอัตโนมัติถ้าผ่าน) |
| `--tracing=retain-on-failure` | เก็บ trace แบบเล่นย้อนดูทุก step ได้ (เปิดด้วย `playwright show-trace trace.zip`) |

นี่คือหนึ่งในเหตุผลที่ตารางในขั้นตอนที่ 635.3 ระบุว่า Playwright มี
"Trace/Video/Screenshot อัตโนมัติ" เป็นจุดแข็ง — สิ่งที่ต้องเขียน hook เองใน
Selenium (ข้อ 637.2-637.3) กลายเป็นแค่ธงคำสั่งบรรทัดเดียวใน Playwright

---

## ขั้นตอนที่ 638: รัน Headless Browser Test ใน CI

### 638.1 Headless Mode คืออะไร

**Headless browser** คือเบราว์เซอร์ที่รันโดยไม่มีหน้าต่าง GUI ให้เห็นบนจอ —
ทำงานเหมือนเบราว์เซอร์ปกติทุกประการ (render CSS, รัน JavaScript, ประมวลผล
DOM) เพียงแต่ไม่วาดภาพขึ้นจอจริง ๆ ซึ่งจำเป็นมากสำหรับเซิร์ฟเวอร์ CI ที่ไม่มี
จอแสดงผล (display) ต่ออยู่เลย

ในโค้ดฐาน (ขั้นตอนที่ 633.4) เราเปิดใช้อยู่แล้วด้วย:

```python
options.add_argument("--headless=new")
```

`--headless=new` คือโหมด headless เวอร์ชันใหม่ของ Chrome (ตั้งแต่ Chrome 109
เป็นต้นมา) ที่ใช้ rendering engine ตัวเดียวกับโหมดปกติทุกประการ (ต่างจาก
`--headless` แบบเก่าที่เป็น engine แยกต่างหากและมีพฤติกรรมบางอย่างต่างจาก
โหมดปกติเล็กน้อย) แนะนำให้ใช้ `--headless=new` เสมอสำหรับ Chrome เวอร์ชัน
ใหม่ เพื่อให้ผลการทดสอบตรงกับสิ่งที่ผู้ใช้จริงเห็นมากที่สุด

### 638.2 ตัวอย่าง GitHub Actions Workflow แบบย่อ

```yaml
# .github/workflows/e2e-tests.yml (ตัวอย่างเบื้องต้น — เจาะลึกเต็มรูปแบบใน Part 088)
name: E2E Tests

on: [push, pull_request]

jobs:
  selenium-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: ติดตั้ง Google Chrome
        run: |
          sudo apt-get update
          sudo apt-get install -y google-chrome-stable

      - name: ติดตั้ง Dependencies
        run: |
          pip install -r requirements.txt -r requirements-test.txt

      - name: รัน Django Migration
        run: python manage.py migrate

      - name: รัน E2E Tests (Selenium, headless)
        run: python manage.py test blog.tests.test_e2e_login blog.tests.test_e2e_post_create

  playwright-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: ติดตั้ง Dependencies
        run: |
          pip install -r requirements.txt -r requirements-test.txt
          playwright install --with-deps chromium

      - name: รัน E2E Tests (Playwright)
        run: pytest -m "not slow" --screenshot=only-on-failure

      - name: อัปโหลด Screenshot เมื่อ Test ล้มเหลว
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failure-screenshots
          path: test-screenshots/
```

### 638.3 ทำไมไม่ต้องใช้ Xvfb อีกต่อไป

ในอดีต (ก่อนที่ headless mode จะพัฒนาสมบูรณ์) ทีมงานจำนวนมากต้องใช้
**Xvfb** (X Virtual Framebuffer) เพื่อจำลอง "จอเสมือน" ให้เบราว์เซอร์ธรรมดา
(ที่ไม่รองรับ headless) เชื่อว่ามีจอแสดงผลอยู่จริง ปัจจุบัน Chrome, Firefox
และเบราว์เซอร์ของ Playwright ทุกตัวรองรับโหมด headless ในตัวโดยตรงแล้ว
**ไม่จำเป็นต้องใช้ Xvfb อีกต่อไป**สำหรับกรณีทั่วไป (ยกเว้นกรณีพิเศษบางอย่าง
เช่น ต้องการทดสอบ WebGL/GPU rendering ที่ headless mode มีข้อจำกัด)

### 638.4 ข้อควรระวังเมื่อรันบน CI ที่ Part 088 จะเจาะลึกเต็มรูปแบบ

| ปัญหาที่พบบ่อยบน CI | สาเหตุ | แนวทางแก้เบื้องต้น |
|---|---|---|
| `session not created: DevToolsActivePort file doesn't exist` | ขาดธง `--no-sandbox`/`--disable-dev-shm-usage` บน container ที่มี resource จำกัด | เพิ่มธงทั้งสองตัวตามขั้นตอนที่ 633.4 (ตั้งไว้ตั้งแต่ต้นแล้ว) |
| Test ผ่านในเครื่อง local แต่ fail บน CI | เครื่อง CI ช้ากว่า/มี CPU น้อยกว่า ทำให้ timing ต่างกัน | เพิ่ม timeout ของ `WebDriverWait` ให้กว้างขึ้นสำหรับ CI โดยเฉพาะ |
| ไม่มี font ภาษาไทยบน CI ทำให้ screenshot ข้อความเพี้ยน | Docker image ของ CI มักไม่มี font ภาษาไทยติดมาด้วย | ติดตั้ง `fonts-thai-tlwg` ผ่าน `apt-get` ใน workflow |
| Test รันช้ามากเมื่อมีหลายสิบเคส | E2E test แต่ละเคสช้าโดยธรรมชาติ (ตามตารางขั้นตอนที่ 631.3) | รันแบบขนาน (parallel), แยก job E2E ออกจาก unit test — Part 065 และ 088 |

Part นี้ให้แค่ภาพรวมพอให้เข้าใจว่า "รันได้จริงบน CI" — การตั้งค่า caching
ของ browser binary, matrix testing หลายเบราว์เซอร์พร้อมกัน, การจัดการ
secret สำหรับ environment variable, และกลยุทธ์ retry ระดับ pipeline จะถูก
สอนอย่างละเอียดครบถ้วนใน **Part 088: CI/CD Pipeline สมบูรณ์แบบ**

---

## ขั้นตอนที่ 639: กลยุทธ์รับมือ Flaky Test

### 639.1 Flaky Test คืออะไร และทำไมมันคือฝันร้ายของทุกทีม

**Flaky Test** คือ test ที่ให้ผลลัพธ์**ไม่คงที่**เมื่อรันซ้ำโดยที่โค้ดไม่ได้
เปลี่ยนแปลงเลย — บางครั้งผ่าน บางครั้งไม่ผ่าน ปัญหาของ flaky test ไม่ใช่แค่
"น่ารำคาญ" แต่มันทำลาย**ความน่าเชื่อถือของ test suite ทั้งหมด**: เมื่อทีมเจอ
flaky test บ่อยเข้า พวกเขาจะเริ่ม**เพิกเฉยต่อผลการทดสอบที่ล้มเหลว** ("อ๋อ
เดี๋ยวรันใหม่เดี๋ยวก็ผ่าน") ซึ่งทำให้ test ที่ล้มเหลวเพราะบั๊กจริง ๆ ถูกมองข้าม
ไปด้วย — นี่คือเหตุผลที่ E2E test (ซึ่งเปราะบางที่สุดตามตารางขั้นตอนที่
631.3) ต้องได้รับการดูแลเป็นพิเศษ

### 639.2 สาเหตุที่พบบ่อยที่สุดของ Flaky Test

| สาเหตุ | ตัวอย่างในบริบท E2E Test | วิธีแก้ |
|---|---|---|
| **Timing/Race Condition** | `find_element` หา element ก่อนที่ HTMX จะ swap เสร็จ | ใช้ `WebDriverWait`/auto-wait ของ Playwright เสมอ (ขั้นตอนที่ 634.5, 635.2) — ห้ามใช้ `time.sleep()` |
| **Test ขึ้นต่อกัน (Test Interdependence)** | Test A สร้าง user ชื่อ `namwan` ทิ้งไว้ Test B รันตามมาสร้างชื่อเดียวกันซ้ำ → `IntegrityError` | ใช้ `factory_boy` (Part 063) หรือ `uuid4()` สร้างชื่อ/อีเมลที่ไม่ซ้ำกันทุกครั้ง |
| **Shared State ข้าม Test** | ลืม cleanup ไฟล์ที่อัปโหลดใน test ก่อนหน้า ทำให้ test ถัดไปเจอไฟล์ค้าง | ใช้ `TestCase`/`TransactionTestCase` ที่ Django ล้างข้อมูลให้อัตโนมัติ + cleanup ไฟล์ด้วย `addCleanup()` |
| **Animation/CSS Transition** | คลิกปุ่มขณะที่ modal กำลังเล่น fade-in animation (จากขั้นตอนที่ 523.3/535.2) ทำให้พิกัดคลิกยังไม่นิ่ง | `EC.element_to_be_clickable` (Selenium) หรือปล่อยให้ Playwright auto-wait จัดการ "stability" ให้ |
| **การพึ่งพา External Service** | test เรียก API จริงที่บางครั้งช้า/ล่ม | Mock external call ด้วย `unittest.mock` (Part 062) แม้แต่ใน E2E test |
| **ลำดับการรันไม่แน่นอน (Order Dependency)** | Test สมมติว่ามี `Post` แค่ 1 รายการในฐานข้อมูล แต่ test อื่นสร้างทิ้งไว้ก่อน | เขียนแต่ละ test ให้ **isolate** เสมอ (สร้างข้อมูลของตัวเองใน `setUp`, ไม่พึ่ง state จากภายนอก) |

### 639.3 หลักการ "Retry ไม่ใช่ทางแก้ที่แท้จริง แต่เป็นตาข่ายนิรภัยชั่วคราว"

เมื่อรู้ว่า test บางตัวเปราะบางแต่ยังไม่มีเวลาแก้ต้นเหตุทันที การตั้งค่าให้
**รันซ้ำอัตโนมัติเมื่อล้มเหลว** เป็นตาข่ายนิรภัยที่ยอมรับได้ในระยะสั้น — แต่
**ต้องไม่ใช่คำตอบถาวร** เพราะมันซ่อนปัญหาที่แท้จริงไว้เท่านั้น:

```bash
pip install pytest-rerunfailures
pytest --reruns 2 --reruns-delay 1
```

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
addopts = --reruns 2 --reruns-delay 1
markers =
    flaky_e2e: test ที่รู้ว่ายังไม่เสถียร 100% กำลังรอการแก้ไข (ติดตามใน issue tracker เสมอ)
```

**กฎของหลักสูตรนี้**: ทุก test ที่ต้องพึ่ง `--reruns` เพื่อผ่าน ต้องถูกมาร์กไว้
ชัดเจน (`@pytest.mark.flaky_e2e`) พร้อมลิงก์ issue ที่ติดตามการแก้ไขต้นเหตุ
เสมอ ห้ามปล่อยให้ `--reruns` กลายเป็น "วิธีทำให้ CI เขียวโดยไม่ต้องแก้อะไร"

### 639.4 Pattern การเขียน Test ที่ลด Flakiness ตั้งแต่ต้น

```python
# ❌ ไม่ดี: ใช้ time.sleep() แบบตายตัว — ช้าเกินจำเป็นและยังไม่การันตีว่าพอ
import time

def test_bad_example(self):
    self.selenium.find_element(By.ID, "like-btn").click()
    time.sleep(3)
    self.assertIn("(1)", self.selenium.find_element(By.CSS_SELECTOR, ".like-button").text)


# ✅ ดี: ใช้ explicit wait ที่รอ "พอดี" ตามเงื่อนไขจริง
def test_good_example(self):
    self.selenium.find_element(By.ID, "like-btn").click()
    WebDriverWait(self.selenium, 10).until(
        EC.text_to_be_present_in_element((By.CSS_SELECTOR, ".like-button"), "(1)")
    )


# ❌ ไม่ดี: ข้อมูลทดสอบชนกันข้าม test เมื่อรันพร้อมกันหรือรันซ้ำ
def test_bad_unique_data(self):
    User.objects.create_user(username="testuser", password="x")  # ชื่อตายตัว


# ✅ ดี: สร้างข้อมูลที่ไม่ซ้ำกันเสมอด้วย factory_boy (Part 063) หรือ uuid
import uuid

def test_good_unique_data(self):
    unique_suffix = uuid.uuid4().hex[:8]
    User.objects.create_user(username=f"testuser_{unique_suffix}", password="x")
```

### 639.5 การติดตาม "ระดับความเปราะบาง" ของ Test Suite อย่างเป็นระบบ

ทีมระดับมืออาชีพไม่ปล่อยให้ flaky test เป็นความรู้สึกลอย ๆ ("รู้สึกว่า test
นี้ชอบพังบ่อย") แต่**วัดออกมาเป็นตัวเลข**จริง เช่น รัน E2E test suite ทั้งหมด
ซ้ำ 20 รอบใน CI (โดยไม่แก้โค้ดเลย) แล้วบันทึกอัตราการล้มเหลวของแต่ละเคส:

| Test | จำนวนที่รัน | จำนวนที่ล้มเหลว | อัตรา Flaky | สถานะ |
|---|---|---|---|---|
| `test_user_can_login_via_real_browser` | 20 | 0 | 0% | ✅ เสถียร |
| `test_clicking_like_button_updates_count` | 20 | 3 | 15% | ⚠️ ต้อง quarantine + สืบสวน |
| `test_authenticated_user_can_create_post` | 20 | 0 | 0% | ✅ เสถียร |

Test ที่มีอัตรา flaky สูงกว่าเกณฑ์ที่ทีมกำหนด (เช่น > 2%) ควรถูก **quarantine**
(ย้ายออกจาก required check ของ CI ชั่วคราว แต่ยังรันเก็บสถิติไว้) จนกว่าจะ
สืบหาต้นเหตุและแก้ไขสำเร็จ ไม่ใช่ปล่อยให้บล็อกการ deploy ของทีมทั้งหมดไปเรื่อย ๆ
เพียงเพราะ "เดี๋ยวรันใหม่ก็ผ่าน" — แนวคิดการวัดและจัดการคุณภาพ test suite
อย่างเป็นระบบแบบนี้จะถูกขยายความต่อใน **Part 065: Continuous Testing และ
Code Quality Tools**

---

## ขั้นตอนที่ 640: สรุปและแบบฝึกหัด

### 640.1 Capstone: E2E Test สมบูรณ์แบบสำหรับ Flow Login → สร้างโพสต์ → เห็นในหน้า List

นำทุกเทคนิคจากขั้นตอนที่ 631-639 มาประกอบกันเป็น E2E test เดียวที่จำลอง
**user journey ที่สำคัญที่สุด**ของบล็อก ตั้งแต่ต้นจนจบ พร้อม screenshot
อัตโนมัติเมื่อล้มเหลว และเขียนด้วยหลักการ "ไม่มี `time.sleep()`" ตลอดทั้งไฟล์:

```python
# blog/tests/test_e2e_full_journey.py
"""
Capstone E2E Test: จำลอง user journey ที่สำคัญที่สุดของบล็อกทั้งระบบ
Flow: สมัคร/มีบัญชีอยู่แล้ว → Login → สร้างโพสต์ใหม่ → เห็นโพสต์ในหน้า List
"""
import uuid

from django.contrib.auth import get_user_model
from django.urls import reverse
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait

from blog.models import Post
from .selenium_base import SeleniumTestCase

User = get_user_model()


class FullUserJourneyE2ETests(SeleniumTestCase):
    def setUp(self):
        # uuid ป้องกันข้อมูลชนกันเมื่อรันซ้ำ/รันขนาน (ทบทวนขั้นตอนที่ 639.4)
        unique_suffix = uuid.uuid4().hex[:8]
        self.username = f"e2e_user_{unique_suffix}"
        self.password = "StrongPass123!"
        self.user = User.objects.create_user(
            username=self.username, password=self.password
        )
        self.post_title = f"โพสต์ E2E ทดสอบ {unique_suffix}"

    def _wait(self, timeout=10):
        return WebDriverWait(self.selenium, timeout)

    def test_login_then_create_post_then_see_it_in_list(self):
        # ------------------------------------------------------------
        # ขั้นตอนที่ 1: เปิดหน้า Login และล็อกอินผ่านฟอร์มจริง
        # ------------------------------------------------------------
        self.selenium.get(f"{self.live_server_url}{reverse('accounts:login')}")

        self.selenium.find_element(By.ID, "id_username").send_keys(self.username)
        password_input = self.selenium.find_element(By.ID, "id_password")
        password_input.send_keys(self.password)
        password_input.send_keys(Keys.RETURN)

        self._wait().until(EC.url_contains("/blog/"))
        self.assertIn(self.username, self.selenium.page_source)

        # ------------------------------------------------------------
        # ขั้นตอนที่ 2: ไปหน้าสร้างโพสต์ และกรอกฟอร์มจนสำเร็จ
        # ------------------------------------------------------------
        self.selenium.get(f"{self.live_server_url}{reverse('blog:post_create')}")

        self._wait().until(
            EC.presence_of_element_located((By.ID, "id_title"))
        ).send_keys(self.post_title)
        self.selenium.find_element(By.ID, "id_content").send_keys(
            "เนื้อหาที่พิมพ์ผ่านเบราว์เซอร์จริงทั้งหมดใน E2E test ของ Part 064"
        )
        self.selenium.find_element(
            By.CSS_SELECTOR, "#post-form button[type=submit]"
        ).click()

        # ------------------------------------------------------------
        # ขั้นตอนที่ 3: ตรวจสอบว่าถูก redirect กลับมาหน้า list และเห็นโพสต์
        # ใหม่ปรากฏอยู่จริง ทั้งฝั่ง UI และฝั่งฐานข้อมูล
        # ------------------------------------------------------------
        self._wait().until(EC.url_matches(rf"{self.live_server_url}/blog/$"))

        post_list_container = self._wait().until(
            EC.presence_of_element_located((By.ID, "post-list"))
        )
        self._wait().until(
            EC.text_to_be_present_in_element((By.ID, "post-list"), self.post_title)
        )
        self.assertIn(self.post_title, post_list_container.text)

        created_post = Post.objects.get(title=self.post_title)
        self.assertEqual(created_post.author, self.user)
        self.assertTrue(created_post.is_published)

        # ------------------------------------------------------------
        # ขั้นตอนที่ 4 (โบนัส): คลิกเข้าไปดูรายละเอียดโพสต์ที่เพิ่งสร้าง
        # เพื่อยืนยันว่า link ในหน้า list ใช้งานได้จริง ไม่ใช่แค่ข้อความปรากฏ
        # ------------------------------------------------------------
        self.selenium.find_element(By.LINK_TEXT, self.post_title).click()

        self._wait().until(EC.url_contains(created_post.slug))
        self.assertIn(self.post_title, self.selenium.page_source)
        self.assertIn(
            "เนื้อหาที่พิมพ์ผ่านเบราว์เซอร์จริง", self.selenium.page_source
        )
```

รันด้วยคำสั่งเดียว และดู screenshot อัตโนมัติ (จากขั้นตอนที่ 637.2) ที่
`test-screenshots/` หาก step ใดล้มเหลว:

```bash
python manage.py test blog.tests.test_e2e_full_journey -v 2
```

Test นี้ครอบคลุมทุกหลักการที่เรียนมาตลอด Part นี้ในไฟล์เดียว: ใช้
`StaticLiveServerTestCase` เป็นฐาน (ขั้นตอนที่ 632-633) ใช้ explicit wait
ทุกจุดแทน `time.sleep()` (ขั้นตอนที่ 634.5, 639.4) ตรวจผลลัพธ์ทั้งฝั่ง UI
และฐานข้อมูล (ขั้นตอนที่ 634.4) ใช้ข้อมูลที่ไม่ซ้ำกันด้วย `uuid` (ขั้นตอนที่
639.2) และมี screenshot-on-failure คอยช่วย debug อัตโนมัติ (ขั้นตอนที่
637.2) หากใครต้องการเวอร์ชัน Playwright ที่ทำสิ่งเดียวกันทุกประการแต่สั้น
กว่าประมาณครึ่งหนึ่ง ให้ลองแปลง test นี้เป็นแบบฝึกหัดที่ 2 ด้านล่าง

### 640.2 สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ✅ เข้าใจ Testing Pyramid: Unit (มากที่สุด, เร็วที่สุด, มั่นใจต่อเคสต่ำสุด)
  → Integration (ปานกลาง) → E2E (น้อยที่สุด, ช้าที่สุด, มั่นใจสูงสุด) และรู้จัก
  กับดัก "Ice Cream Cone" ที่ควรหลีกเลี่ยง
- ✅ ใช้ `LiveServerTestCase`/`StaticLiveServerTestCase` รัน Django server
  จริงสำหรับให้เบราว์เซอร์หรือ HTTP client ภายนอกเชื่อมต่อเข้ามาได้ และเข้าใจ
  ว่าทำไมมันสืบทอดจาก `TransactionTestCase` ไม่ใช่ `TestCase`
- ✅ ติดตั้งและตั้งค่า Selenium กับ `webdriver-manager` สร้าง base test class
  ที่ใช้ซ้ำได้ทั้งโปรเจกต์ พร้อมรู้จัก Selenium Manager ที่มากับ Selenium
  เวอร์ชันใหม่
- ✅ เขียน Browser Automation Test จริงที่คลิกปุ่ม กรอกฟอร์ม และตรวจสอบ
  ผลลัพธ์ทั้งฝั่ง UI และฐานข้อมูล โดยใช้ `WebDriverWait`/`expected_conditions`
  แทน `time.sleep()` เสมอ
- ✅ รู้จัก Playwright เป็นทางเลือกที่ทันสมัยกว่า ด้วย auto-waiting ในตัว
  ทุก action ทำให้โค้ดสั้นกว่าและเปราะบางน้อยกว่า Selenium โดยธรรมชาติ
- ✅ ทดสอบหน้าที่มี HTMX (ปุ่มถูกใจจาก Part 053) และ Alpine.js (Modal จาก
  Part 054) ได้จริง ซึ่งเป็นสิ่งที่ `django.test.Client` ทำไม่ได้เลย
- ✅ ตั้งค่า screenshot อัตโนมัติเมื่อ test ล้มเหลว ทั้งแบบ `unittest`
  (`tearDown` + `_outcome`) และแบบ `pytest` (hook ที่แข็งแรงกว่า) รวมถึง
  ธงคำสั่งสำเร็จรูปของ Playwright
- ✅ เข้าใจภาพรวมการรัน headless browser test ใน CI ผ่าน GitHub Actions
  และรู้ว่าทำไมไม่จำเป็นต้องใช้ Xvfb อีกต่อไป
- ✅ ระบุสาเหตุของ Flaky Test ได้ (timing, shared state, order dependency,
  animation) และรู้จักเครื่องมือรับมือ (`pytest-rerunfailures`, quarantine,
  การวัดอัตราความเปราะบางอย่างเป็นระบบ)
- ✅ เขียน E2E test สมบูรณ์แบบที่จำลอง flow login → สร้างโพสต์ → เห็นโพสต์
  ในหน้า list ได้ทั้งหมดโดยอัตโนมัติ

### 640.3 Checklist ก่อนไป Part ถัดไป

- [ ] อธิบายความแตกต่างระหว่าง Unit/Integration/E2E Test ได้ด้วยคำพูดตัวเอง
      พร้อมยกตัวอย่างจากโปรเจกต์บล็อกของตัวเอง
- [ ] ติดตั้ง `selenium` และ `webdriver-manager` สำเร็จ และรัน smoke test
      แรก (ขั้นตอนที่ 633.5) ผ่านได้จริง
- [ ] เขียน E2E test ที่ login ผ่านฟอร์มจริงได้สำเร็จอย่างน้อย 1 เคส โดยไม่มี
      `time.sleep()` แม้แต่บรรทัดเดียว
- [ ] ติดตั้ง Playwright และเขียน test เดิมซ้ำด้วย Playwright ได้สำเร็จ
      (ทดลองรันด้วย `--headed` ดูเบราว์เซอร์จริงอย่างน้อยหนึ่งครั้ง)
- [ ] เขียน test ที่ทดสอบปุ่ม HTMX หรือ Modal ของ Alpine.js ได้จริงอย่างน้อย
      1 เคส และอธิบายได้ว่าทำไม `self.client` ทำสิ่งนี้ไม่ได้
- [ ] ตั้งค่า screenshot-on-failure สำเร็จ (ลองทำให้ test พังโดยตั้งใจ แล้ว
      ตรวจว่ามีไฟล์ภาพปรากฏใน `test-screenshots/` จริง)
- [ ] เข้าใจว่าทำไม `--headless=new` จำเป็นสำหรับรันบน CI และรู้จักธง
      `--no-sandbox`/`--disable-dev-shm-usage`
- [ ] ระบุสาเหตุของ flaky test ได้อย่างน้อย 3 ข้อ พร้อมวิธีแก้ของแต่ละข้อ
- [ ] รัน capstone test จากขั้อ 640.1 ผ่านสำเร็จในเครื่องตัวเอง

### 640.4 แบบฝึกหัดท้ายบท

**แบบฝึกหัดที่ 1**: เขียน E2E test ใหม่ (ต่อยอดจาก `test_e2e_login.py`)
ที่ทดสอบกรณี**logout**: ล็อกอินสำเร็จก่อน แล้วคลิกปุ่ม "ออกจากระบบ" (ทบทวน
dropdown menu จาก Part 054 ข้อ 535.1) จากนั้นตรวจสอบว่า (1) ถูก redirect
กลับไปหน้าที่ถูกต้อง (2) พยายามเข้าหน้า `/blog/new/` อีกครั้งโดยไม่ล็อกอิน
ใหม่แล้วถูก redirect ไปหน้า login โดยอัตโนมัติ (ทดสอบว่า `@login_required`
ทำงานถูกต้องแม้ในเบราว์เซอร์จริง)

**แบบฝึกหัดที่ 2**: แปลง capstone test จากข้อ 640.1 ให้เป็นเวอร์ชัน
Playwright ทั้งหมด โดยใช้ fixture `live_server`, `page`, `django_user_model`
ตามที่สอนในขั้นตอนที่ 635.5 เปรียบเทียบจำนวนบรรทัดโค้ดระหว่างสองเวอร์ชัน
แล้วบันทึกลงในไฟล์ `notes.md` ว่าแตกต่างกันกี่เปอร์เซ็นต์ และจุดไหนที่ทำให้
เวอร์ชัน Playwright สั้นกว่า

**แบบฝึกหัดที่ 3**: เขียน E2E test สำหรับฟอร์มสร้างโพสต์ที่ **validation
ล้มเหลว** (ส่งฟอร์มโดยเว้น field `title` ว่างไว้) แล้วตรวจสอบว่า (1) หน้า
เว็บไม่ redirect ไปไหน (2) ข้อความ error จาก Django Form ปรากฏบนหน้าจอจริง
ผ่าน `.errorlist` (ทบทวนโครงสร้าง error ของ `ModelForm` จาก Part 025)
(3) ข้อมูลที่ผู้ใช้กรอกไว้ในช่อง `content` (ที่ไม่ได้ผิดพลาด) ยังคงอยู่ในฟอร์ม
ไม่หายไป

**แบบฝึกหัดที่ 4 (ขั้นสูง)**: จำลองสถานการณ์ flaky test ขึ้นมาเองโดยตั้งใจ:
เพิ่ม `time.sleep(random.uniform(0, 0.3))` ในฝั่ง View `post_create_view`
ก่อน `return redirect(...)` (เพื่อจำลอง network latency ที่ไม่แน่นอน) แล้ว
รัน E2E test จากข้อ 640.1 ซ้ำ 10 ครั้งติดต่อกันด้วยคำสั่ง
`for i in {1..10}; do python manage.py test blog.tests.test_e2e_full_journey; done`
สังเกตว่า test ยังผ่านทุกครั้งหรือไม่ (ควรผ่านเพราะใช้ `WebDriverWait` ที่รอ
ตามเงื่อนไขจริงเสมอ) จากนั้นลองแก้ test ให้ใช้ `time.sleep(0.1)` แทน
`WebDriverWait` ชั่วคราว แล้วรันซ้ำ 10 ครั้งอีกที บันทึกผลว่ามีกี่ครั้งที่
ล้มเหลว เพื่อพิสูจน์ด้วยตัวเองว่าทำไมหลักสูตรนี้ถึงห้ามใช้ `time.sleep()`
อย่างเด็ดขาด

### 640.5 คำถามที่พบบ่อย (FAQ)

**Q: ต้องเขียน E2E test ครอบคลุมทุกหน้าของเว็บไซต์เลยหรือไม่?**
A: ไม่ควรทำเด็ดขาด ตามหลัก Testing Pyramid ในขั้นตอนที่ 631 E2E test ควรมี
สัดส่วนน้อยที่สุดในพีระมิด (~10%) และสงวนไว้เฉพาะ **critical user journey**
เท่านั้น เช่น login, สมัครสมาชิก, การชำระเงิน, และ flow ที่ต้องพึ่ง JavaScript
หนัก ๆ ที่ unit test มองไม่เห็น (ขั้นตอนที่ 636) ส่วนหน้าอื่น ๆ ที่ไม่มี
JavaScript ซับซ้อนควรทดสอบด้วย `django.test.Client` (Part 060-061) ซึ่งเร็ว
กว่ามากและให้ผลลัพธ์ที่แม่นยำไม่แพ้กันสำหรับ logic ฝั่ง server

**Q: ควรเลือก Selenium หรือ Playwright สำหรับโปรเจกต์ใหม่?**
A: สำหรับโปรเจกต์ที่เริ่มต้นใหม่ Playwright เป็นตัวเลือกที่แนะนำมากกว่าใน
ปัจจุบัน เพราะ auto-waiting (ขั้อ 635.2) ลดโอกาสเกิด flaky test ได้มากในทาง
ปฏิบัติ และ API สั้นกระชับกว่า อย่างไรก็ตาม Selenium ยังคงเป็นทักษะที่จำเป็น
ต้องอ่านออกเขียนได้ เพราะโปรเจกต์เก่าจำนวนมหาศาลในอุตสาหกรรมยังใช้อยู่ และ
เอกสาร/ชุมชน/Stack Overflow ของ Selenium ยังคงใหญ่กว่ามากในภาพรวม

**Q: ทำไม test ที่รันผ่านในเครื่อง local ถึง fail บน CI บ่อย ๆ?**
A: สาเหตุที่พบบ่อยที่สุดคือ **ความเร็วของเครื่องต่างกัน** — เครื่อง CI มักมี
CPU/RAM จำกัดกว่าเครื่อง dev ทำให้หน้าเว็บ render ช้ากว่า ถ้า test เขียนโดย
ใช้ `WebDriverWait` ที่มี timeout เพียงพอ (เช่น 10 วินาที) ปัญหานี้มักไม่เกิด
แต่ถ้า timeout สั้นเกินไป (เช่น 2 วินาที) หรือแอบใช้ `time.sleep()` แบบตายตัว
ปัญหานี้จะเกิดบ่อยมาก ตรวจสอบให้แน่ใจว่าไม่มี hard-coded timing สั้นเกินไปใน
โค้ด test ของคุณ ตามหลักการในขั้อ 639.4

**Q: ต้องรัน E2E test suite ทุกครั้งที่ commit เหมือน unit test หรือไม่?**
A: ไม่จำเป็น เพราะ E2E test ช้ากว่า unit test มาก (ตามตารางขั้อ 631.3) ทีม
ระดับมืออาชีพส่วนใหญ่จะรัน unit + integration test ทุกครั้งที่ commit/push
(เร็ว ให้ feedback ทันที) แต่รัน E2E test suite เต็มรูปแบบเฉพาะตอนที่จะ merge
เข้า branch หลัก หรือรันเป็นรอบ (เช่น ทุกคืน — "nightly build") กลยุทธ์การ
แบ่งชั้นการรัน test แบบนี้จะถูกอธิบายอย่างละเอียดใน Part 065 และ Part 088

---

## เตรียมตัวสำหรับ Part ถัดไป

**Part 065: Continuous Testing และ Code Quality Tools** จะพาคุณไปดูภาพรวม
ทั้งหมดของ Phase 7 อีกครั้งในมุมของ **กระบวนการ (process)** ไม่ใช่แค่เครื่องมือ
เดี่ยว ๆ — คุณจะได้เรียนรู้การตั้งค่า **pre-commit hooks** ที่รัน linter/
formatter อัตโนมัติก่อน commit ทุกครั้ง เครื่องมือตรวจสอบคุณภาพโค้ดอย่าง
`ruff`, `black`, `mypy` แบบเจาะลึก การจัดกลุ่ม test suite ให้รันแยกชั้น
(unit เร็ว → integration → E2E ช้า ตามที่พูดถึงใน FAQ ข้อสุดท้ายของ Part นี้)
และการวัดคุณภาพของ test suite เองอย่างเป็นระบบ (ไม่ใช่แค่ coverage ตัวเลข
เดียวจาก Part 062 แต่รวมถึงอัตรา flaky ที่เริ่มพูดถึงในขั้อ 639.5 ด้วย) เมื่อ
จบ Phase 7 ทั้งหมด (Part 059-065) คุณจะมี test suite ที่ครบทั้งสามชั้นของ
พีระมิด พร้อมกระบวนการดูแลรักษาคุณภาพที่ยั่งยืน ก่อนที่เราจะย้ายไป **Phase 8:
Performance & Caching** ที่ Part 066 เป็นต้นไป ซึ่งจะใช้ test suite ที่แข็งแรง
นี้เป็นตาข่ายนิรภัยทุกครั้งที่ทำการ optimize โค้ดให้เร็วขึ้น
